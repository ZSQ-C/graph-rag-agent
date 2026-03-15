# Tool 子模块代码理解总结

## 一、核心总览（带逻辑关系）

### 核心定位

Search/Tool 子模块是 GraphRAG 项目中搜索工具的封装层，它基于抽象基类 `BaseSearchTool` 构建了一套完整的搜索工具体系，包括 `LocalSearchTool`（本地搜索）、`GlobalSearchTool`（全局搜索）、`HybridSearchTool`（混合搜索）、`NaiveSearchTool`（简单搜索）、`DeepResearchTool`（深度研究）、`DeeperResearchTool`（增强版深度研究）等，同时提供了专用工具如 `ChainOfExplorationTool`、`HypothesisGeneratorTool`、`AnswerValidationTool`。该模块解决了不同复杂度问题的搜索需求：从简单事实查询到复杂多步推理，从精确实体检索到跨社区概念整合，通过统一的接口设计和工具注册表机制，使得多 Agent 系统能够灵活选择和组合不同的搜索策略。

### 整体流程串讲

用户查询首先进入工具层，通过继承 `BaseSearchTool` 抽象基类，所有搜索工具共享一套初始化流程：首先调用 `BaseSearchTool.__init__()` 初始化 LLM 和 Embeddings 模型，建立 Neo4j 数据库连接，配置上下文感知的缓存管理器，设置性能监控指标；然后每个具体工具调用自己的 `_setup_chains()` 抽象方法，配置专属的 LLM 处理链和提示模板，这一步是各工具差异化的关键。

初始化完成后，进入核心入口方法模块：首先调用 `extract_keywords()` 从查询中提取关键词，这一步有缓存优化，避免重复 LLM 调用；然后执行 `search()` 或 `structured_search()` 方法启动搜索。在搜索过程中，根据工具类型进入不同的分支逻辑：`LocalSearchTool` 使用历史感知检索器进行社区内精确搜索，`GlobalSearchTool` 使用 Map-Reduce 模式进行跨社区搜索，`NaiveSearchTool` 直接进行纯向量搜索，深度研究工具则执行多步思考-搜索-推理。

分支逻辑确定后，进入具体实现方法模块：`LocalSearchTool` 调用 `_normalize_input()` 规范化输入，使用 `rag_chain.invoke()` 执行完整的 RAG 流程；`GlobalSearchTool` 调用 `_get_community_data()` 获取社区数据，通过 `_process_communities()` 执行 Map 阶段批处理，最后用 `_reduce_results()` 执行 Reduce 阶段整合；`NaiveSearchTool` 直接查询带 embedding 的 Chunk 节点，用 `VectorUtils.rank_by_similarity()` 进行相似度排序。

在整个流程中，辅助方法模块提供支持：`vector_search()` 优先使用向量相似度搜索，失败时降级到 `text_search()` 文本匹配；`semantic_search()` 和 `filter_by_relevance()` 对已有结果进行重排序；`_log_performance()` 记录性能指标；`get_tool()` 和 `get_structured_tool()` 将搜索工具封装为 LangChain 的 `BaseTool`，便于在 Agent 系统中使用；上下文管理器 `__enter__()` 和 `__exit__()` 确保资源正确释放。所有工具的输出最终通过检索适配器统一为标准格式，返回给用户或下游 Agent。

***

## 二、模块拆分（固定顺序 + 关系说明）

### 2.1 初始化模块

该模块的作用是为所有搜索工具准备统一的基础设施，位于整体流程的最前端，是所有具体搜索工具的基础，通过抽象基类 `BaseSearchTool` 为子类提供共性能力，子类只需实现差异化的抽象方法。

核心初始化由 `BaseSearchTool.__init__()` 完成：首先调用 `get_llm_model()` 和 `get_embeddings_model()` 加载大语言模型和嵌入模型，设置默认的向量搜索、文本搜索、语义搜索和相关性过滤参数；然后初始化缓存管理器，使用 `ContextAndKeywordAwareCacheKeyStrategy` 作为键策略，`MemoryCacheBackend` 作为存储后端，支持上下文和关键词感知的缓存；接下来设置性能监控指标字典，记录查询时间、LLM 处理时间和总时间；最后调用 `_setup_neo4j()` 建立 Neo4j 连接，获取图实例和驱动对象。在此基础上，各具体工具的 `__init__()` 调用父类构造函数后，进行差异化配置：`LocalSearchTool` 创建 `LocalSearch` 实例和检索器，设置聊天历史；`GlobalSearchTool` 配置社区层级；`NaiveSearchTool` 设置向量搜索的 top\_k 参数；所有工具最后都调用 `_setup_chains()` 抽象方法配置专属处理链。

### 2.2 核心入口方法模块

该模块的作用是接收用户查询并启动完整的搜索流程，位于初始化模块之后，是外部调用的主要入口，负责协调后续分支逻辑和具体实现模块的执行，同时提供缓存优化避免重复计算。

核心入口包括三类方法：所有工具都实现的 `search(query: Any) -> str`，它接收查询并返回纯文本答案，通常是对 `structured_search()` 的封装；各工具特有的 `structured_search(query_input: Any) -> Dict[str, Any]`，它返回包含查询、关键词、答案、检索结果、原始上下文等完整信息的结构化数据；以及所有工具都实现的 `extract_keywords(query: str) -> Dict[str, List[str]]` 抽象方法，它从查询中提取关键词，通常分为低级关键词（具体实体）和高级关键词（主题概念）。入口方法的执行流程统一：首先规范化输入，处理字符串和字典两种格式；然后检查缓存，命中则直接返回缓存结果，避免重复计算；未命中则执行实际搜索流程；最后缓存结果并返回。这些入口方法确保了所有工具具有统一的调用接口，便于在多 Agent 系统中统一使用。

### 2.3 分支逻辑方法模块

该模块的作用是根据查询特征和执行状态选择不同的搜索策略，位于核心入口方法模块之后、具体实现方法模块之前，起到策略选择和容错降级的关键作用，确保在各种情况下都能提供可用的搜索结果。

主要分支逻辑由 `BaseSearchTool` 提供的通用方法实现：`vector_search(query: str, limit: int)` 优先使用向量相似度搜索，通过 `db.index.vector.queryNodes()` Neo4j 存储过程查询向量索引，如果抛出异常（如索引不存在、向量维度不匹配等），则捕获异常并打印错误信息，然后自动调用 `text_search()` 作为备选方案；`text_search(query: str, limit: int)` 使用 Cypher 的 `CONTAINS` 条件进行文本匹配搜索，查询实体 ID 或描述中包含查询词的实体，作为向量搜索失败时的降级策略；`semantic_search(query: str, entities: List[Dict], ...)` 对已有的实体列表进行语义相似度排序，不需要重新查询数据库；`filter_by_relevance(query: str, docs: List, ...)` 根据相关性过滤文档列表。这些分支逻辑方法确保了搜索功能的鲁棒性，即使在向量索引不可用时也能提供基本的搜索能力。

### 2.4 具体实现方法模块

该模块的作用是执行具体的搜索操作和数据处理，位于分支逻辑方法模块之后，是实际产生搜索结果的核心部分，其输出经过辅助方法模块处理后返回给用户，各工具的差异化主要体现在这一模块。

对于 `LocalSearchTool`，具体实现包括：`_normalize_input()` 规范化输入，处理字符串和字典两种格式，提取 query 和 keywords；`structured_search()` 是核心实现，调用 `rag_chain.invoke()` 执行完整的 RAG 流程，传入 input、response\_type 和 chat\_history，获取 chain\_output 后提取 answer、context，用 `results_from_documents()` 和 `results_to_payload()` 转换为标准格式，最后缓存并返回结构化结果。对于 `GlobalSearchTool`，具体实现包括：`_get_community_data(keywords)` 使用关键词过滤查询社区数据，构建 Cypher 查询，按 community\_rank 和 weight 排序，限制返回 20 个社区；`_process_community_batch(query, batch)` 处理社区批次，合并批次内的社区数据，一次性调用 LLM 处理整个批次以提高效率；`_process_communities(query, communities)` 执行 Map 阶段，按批次大小分批处理社区，收集中间结果；`_reduce_results(query, intermediate_results)` 执行 Reduce 阶段，调用 reduce\_chain 整合所有中间结果生成最终答案；`_community_results_to_retrieval(communities)` 将社区数据转换为标准 RetrievalResult 格式。对于 `NaiveSearchTool`，具体实现是 `search()` 方法，直接查询带 embedding 的 Chunk 节点，用 `VectorUtils.rank_by_similarity()` 对候选集进行相似度排序，取 top\_k 个结果，格式化上下文后调用 LLM 生成答案。这些具体实现方法直接与 Neo4j、LLM 等底层系统交互，产生实际的搜索结果。

### 2.5 辅助方法模块

该模块的作用是为上述所有模块提供通用的支持功能，贯穿整个搜索流程的始终，负责工具封装、性能监控、资源管理、结果转换等共性任务，确保各模块之间的协同工作，提高代码复用性。

辅助方法主要分布在 `BaseSearchTool` 中：`db_query(cypher: str, params: Dict)` 是数据库查询的统一入口，通过 `get_db_manager().execute_query()` 执行 Cypher 查询；`_setup_neo4j()` 设置 Neo4j 连接，获取图实例和驱动对象；`_log_performance(operation: str, start_time: float)` 记录性能指标，计算操作耗时并打印；`get_tool()` 创建动态的 `DynamicSearchTool` 类，继承自 LangChain 的 `BaseTool`，封装 `search()` 方法，便于在 Agent 系统中使用；`get_structured_tool()`（各具体工具实现）创建可返回结构化结果的工具版本；`close()` 关闭资源连接，调用 Neo4jGraph 的 close 方法（如果存在）；`__enter__()` 和 `__exit__()` 实现上下文管理器协议，确保资源正确释放。此外，各具体工具还有自己的辅助方法：`LocalSearchTool._filter_documents_by_relevance()` 调用基类的 `filter_by_relevance()`；`GlobalSearchTool._normalize_input()` 规范化输入格式。这些辅助方法将共性功能抽取出来，提高了代码的复用性和可维护性，确保资源正确管理和性能有效监控。

***

## 三、方法详细解析（强制5要素 + 文字流程串讲）

### 3.1 BaseSearchTool.**init**()

#### 方法文字流程串讲

该方法是所有搜索工具的基础初始化方法，为整个搜索流程准备必要的基础设施。方法开始时，首先接收可选的 `cache_dir` 参数（默认值为 `"./cache/search"`），用于指定缓存存储目录。接下来初始化大语言模型和嵌入模型，调用 `get_llm_model()` 获取 LLM 实例，调用 `get_embeddings_model()` 获取嵌入模型实例，然后从 `BASE_SEARCH_CONFIG` 中读取默认参数：`default_vector_limit`（向量搜索默认结果数）、`default_text_limit`（文本搜索默认结果数）、`default_semantic_top_k`（语义搜索默认 top\_k）、`default_relevance_top_k`（相关性过滤默认 top\_k）。

然后初始化缓存管理器，创建 `CacheManager` 实例，使用 `ContextAndKeywordAwareCacheKeyStrategy` 作为键生成策略（支持上下文和关键词感知的缓存），使用 `MemoryCacheBackend` 作为存储后端（内存缓存，设置最大容量为 `BASE_SEARCH_CONFIG["cache_max_size"]`），传入 `cache_dir` 参数。接下来初始化性能监控指标字典 `performance_metrics`，包含三个键：`query_time`（数据库查询时间）、`llm_time`（大语言模型处理时间）、`total_time`（总处理时间），初始值都为 0。最后调用 `_setup_neo4j()` 私有方法建立 Neo4j 数据库连接，完成整个初始化过程。整个方法没有复杂的分支判断，按顺序执行初始化步骤，确保所有基础设施准备就绪。

#### 强制5要素

**入参**：

- `cache_dir: str = "./cache/search"` - 缓存目录，选填，默认值为 `"./cache/search"`

**核心逻辑**：
初始化 LLM 和 Embeddings 模型 → 配置默认搜索参数 → 初始化缓存管理器（上下文和关键词感知）→ 设置性能监控指标 → 建立 Neo4j 连接

**输出形式**：

- 无返回值，初始化实例属性

**底层关键依赖**：

- `get_llm_model()` - 获取 LLM 模型
- `get_embeddings_model()` - 获取嵌入模型
- `CacheManager` - 缓存管理器
- `ContextAndKeywordAwareCacheKeyStrategy` - 缓存键策略
- `MemoryCacheBackend` - 内存缓存后端
- `get_db_manager()` - 数据库连接管理器
- `_setup_neo4j()` - 建立 Neo4j 连接

**关键代码片段**：

```python
def __init__(self, cache_dir: str = "./cache/search"):
    self.llm = get_llm_model()
    self.embeddings = get_embeddings_model()
    self.default_vector_limit = BASE_SEARCH_CONFIG["vector_limit"]
    self.default_text_limit = BASE_SEARCH_CONFIG["text_limit"]
    
    self.cache_manager = CacheManager(
        key_strategy=ContextAndKeywordAwareCacheKeyStrategy(),
        storage_backend=MemoryCacheBackend(
            max_size=BASE_SEARCH_CONFIG["cache_max_size"]
        ),
        cache_dir=cache_dir
    )
    
    self.performance_metrics = {
        "query_time": 0,
        "llm_time": 0,
        "total_time": 0
    }
    
    self._setup_neo4j()
```

***

### 3.2 BaseSearchTool.vector\_search()

#### 方法文字流程串讲

该方法是基于向量相似度的搜索方法，优先使用向量搜索，失败时自动降级到文本搜索。方法开始时，首先接收 `query`（搜索查询）和可选的 `limit`（最大返回结果数）两个参数。整个方法被包裹在 try-except 块中，确保异常时能安全降级。

在 try 块内，首先确定结果数量：如果 `limit` 参数不为 None 则使用传入值，否则使用 `self.default_vector_limit` 默认值。接下来生成查询的嵌入向量，调用 `self.embeddings.embed_query(query)` 将查询文本转换为向量表示，这是向量搜索的基础。然后构建 Neo4j 向量搜索查询，使用 `db.index.vector.queryNodes()` 存储过程，传入索引名称 `'vector'`、结果数量 `$limit` 和查询向量 `$embedding`，YIELD 返回 node 和 score，按 score 降序排序，返回 node.id 和 score。

然后执行查询，调用 `self.db_query(cypher, params)` 传入 Cypher 查询语句和参数字典。检查查询结果是否为空：如果 results 不为空，则提取 `'id'` 列转换为列表返回；如果为空，则返回空列表 `[]`。

如果 try 块中抛出任何异常（如索引不存在、向量维度不匹配、数据库连接失败等），则进入 except 块：首先打印错误信息 `"向量搜索失败: {e}"`，然后自动调用 `self.text_search(query, limit)` 作为备选方案，返回文本搜索的结果。这种降级策略确保了即使向量搜索不可用时，仍能提供基本的搜索能力，保证了搜索功能的鲁棒性。

#### 强制5要素

**入参**：

- `query: str` - 搜索查询，必填，无默认值
- `limit: int = None` - 最大返回结果数，选填，默认使用 `self.default_vector_limit`

**核心逻辑**：
确定结果数量 → 生成查询嵌入向量 → 构建并执行 Neo4j 向量索引查询 → 提取并返回实体 ID 列表 → 异常时降级到文本搜索

**输出形式**：

- 返回值：`List[str]` - 匹配的实体 ID 列表
- 异常处理：向量搜索失败时自动调用 `text_search()` 作为备选
- 空结果：返回空列表 `[]`

**底层关键依赖**：

- `self.embeddings.embed_query()` - 生成查询嵌入向量
- `self.db_query()` - 执行 Cypher 查询
- `db.index.vector.queryNodes()` - Neo4j 向量索引查询存储过程
- `self.text_search()` - 备选文本搜索方法

**关键代码片段**：

```python
def vector_search(self, query: str, limit: int = None) -> List[str]:
    try:
        limit = limit or self.default_vector_limit
        query_embedding = self.embeddings.embed_query(query)
        cypher = """
        CALL db.index.vector.queryNodes('vector', $limit, $embedding)
        YIELD node, score
        RETURN node.id AS id, score
        ORDER BY score DESC
        """
        results = self.db_query(cypher, {
            "embedding": query_embedding,
            "limit": limit
        })
        if not results.empty:
            return results['id'].tolist()
        else:
            return []
    except Exception as e:
        print(f"向量搜索失败: {e}")
        return self.text_search(query, limit)
```

**特殊处理标注**：

- **异常捕获**：完整 try-except 包裹，确保向量搜索失败时不影响整体流程
- **降级策略**：异常时自动调用 text\_search() 作为备选
- **默认值处理**：limit 为 None 时使用 self.default\_vector\_limit

***

### 3.3 LocalSearchTool.structured\_search()

#### 方法文字流程串讲

该方法是 LocalSearchTool 的核心实现，执行本地搜索并返回结构化结果。方法开始时，首先记录 `overall_start` 开始时间，用于计算总耗时。接下来调用 `_normalize_input(query_input)` 规范化输入：如果输入是字典，则提取 `query`（或 `input`）和 `keywords`；如果输入是其他类型，则转换为字符串作为 query，keywords 设为空列表。拿到规范化后的 `query` 和 `keywords` 后，检查 query 是否为空，如果为空则抛出 `ValueError("query不能为空")`。

然后构建缓存键：如果没有 keywords，则缓存键为 query；如果有 keywords，则缓存键为 `f"{query}||{','.join(sorted(keywords))}"`。再构建结构化缓存键为 `f"{cache_key}::structured"`。首先检查结构化缓存：调用 `self.cache_manager.get(structured_cache_key)`，如果返回值是字典类型，则直接返回缓存的结构化结果，避免重复计算。

如果结构化缓存未命中，则检查普通答案缓存 `self.cache_manager.get(cache_key)`，但不直接返回，继续执行搜索流程。然后进入 try 块执行实际搜索：调用 `self.rag_chain.invoke()` 执行完整的 RAG 流程，传入字典参数：`input` 为查询、`response_type` 为 `"多个段落"`、`chat_history` 为 `self.chat_history`。

拿到 `chain_output` 后，提取答案：`answer = chain_output.get("answer") or "抱歉，我无法回答这个问题。"`；提取上下文文档：`documents = chain_output.get("context") or []`。然后将文档转换为标准格式：调用 `results_from_documents(documents, source="local_search")` 转换为 RetrievalResult 列表，再调用 `results_to_payload()` 转换为可序列化的字典列表。

然后构建结构化结果字典 `structured_result`，包含：`query`、`keywords`、`answer`、`retrieval_results`、`raw_context`（原始上下文列表，包含 page\_content 和 metadata）。接下来缓存结果：首先缓存结构化结果 `self.cache_manager.set(structured_cache_key, structured_result)`；然后如果普通答案缓存为 None，则缓存普通答案 `self.cache_manager.set(cache_key, answer)`。

然后记录总耗时：`self.performance_metrics["total_time"] = time.time() - overall_start`，最后返回 `structured_result`。如果 try 块中抛出异常，则进入 except 块：打印错误信息 `"本地搜索失败: {e}"`，构建错误结果字典，包含 query、keywords、answer（错误信息）、retrieval\_results（空列表）、raw\_context（空列表）、error（异常字符串），记录总耗时后返回错误结果。整个流程有完善的缓存优化和异常处理，确保高效和稳定。

#### 强制5要素

**入参**：

- `query_input: Any` - 搜索输入，可以是字符串或包含 query/keywords 的字典，必填

**核心逻辑**：
规范化输入 → 检查缓存（结构化优先）→ 执行 RAG 链 → 提取答案和上下文 → 转换为标准格式 → 构建并缓存结构化结果 → 返回

**输出形式**：

- 返回值：`Dict[str, Any]` - 结构化结果，包含：
  - `query`: 原始查询
  - `keywords`: 关键词列表
  - `answer`: 最终答案
  - `retrieval_results`: 标准检索结果列表
  - `raw_context`: 原始上下文列表
- 异常返回：包含 `error` 字段的错误字典

**底层关键依赖**：

- `self._normalize_input()` - 规范化输入
- `self.rag_chain.invoke()` - RAG 链执行
- `results_from_documents()` - 文档转换
- `results_to_payload()` - 结果序列化
- `self.cache_manager` - 缓存管理器

**关键代码片段**：

```python
def structured_search(self, query_input: Any) -> Dict[str, Any]:
    overall_start = time.time()
    parsed = self._normalize_input(query_input)
    query = parsed["query"]
    keywords = parsed["keywords"]
    
    cache_key = query if not keywords else f"{query}||{','.join(sorted(keywords))}"
    structured_cache_key = f"{cache_key}::structured"
    
    cached_structured = self.cache_manager.get(structured_cache_key)
    if isinstance(cached_structured, dict):
        return cached_structured
    
    try:
        chain_output = self.rag_chain.invoke({
            "input": query,
            "response_type": "多个段落",
            "chat_history": self.chat_history,
        })
        
        answer = chain_output.get("answer") or "抱歉，我无法回答这个问题。"
        documents = chain_output.get("context") or []
        retrieval_results = results_to_payload(
            results_from_documents(documents, source="local_search")
        )
        
        structured_result = {
            "query": query,
            "keywords": keywords,
            "answer": answer,
            "retrieval_results": retrieval_results,
            "raw_context": [...],
        }
        
        self.cache_manager.set(structured_cache_key, structured_result)
        self.performance_metrics["total_time"] = time.time() - overall_start
        return structured_result
    except Exception as e:
        return {
            "query": query,
            "keywords": keywords,
            "answer": f"搜索过程中出现问题: {str(e)}",
            "retrieval_results": [],
            "raw_context": [],
            "error": str(e),
        }
```

**特殊处理标注**：

- **缓存优化**：双重缓存（结构化优先，普通答案备用）
- **输入规范化**：兼容字符串和字典两种输入格式
- **异常处理**：完整 try-except 包裹，确保异常时返回错误信息而非崩溃

***

### 3.4 GlobalSearchTool.\_process\_communities()

#### 方法文字流程串讲

该方法是 GlobalSearchTool 的 Map 阶段实现，处理社区数据生成中间结果，使用批处理提高效率。方法开始时，接收 `query`（搜索查询）和 `communities`（社区数据列表）两个参数。首先从 `GLOBAL_SEARCH_SETTINGS` 中读取 `community_batch_size` 批处理大小，该参数控制每批处理的社区数量，通过批处理减少 LLM 调用次数，提高效率。

然后初始化空的 `results` 列表用于收集中间结果。接下来使用 for 循环按批次处理社区：从 0 开始，步长为 `batch_size`，遍历 `communities` 列表。在每轮循环中，提取当前批次 `batch = communities[i:i+batch_size]`。

然后将批次处理包裹在 try-except 块中：在 try 块内，调用 `_process_community_batch(query, batch)` 处理当前批次，传入查询和批次数据；检查批次处理结果 `batch_result` 是否非空且去除空白后长度大于 0，如果是则将 `batch_result` 添加到 `results` 列表中。如果 try 块中抛出异常，则进入 except 块，打印错误信息 `"批处理失败: {e}"`，跳过当前批次继续处理下一批。

这种批处理策略的目的是减少 LLM 调用次数：假设共有 20 个社区，批处理大小为 5，则只需调用 4 次 LLM，而非 20 次，大大提高了处理效率。同时，单个批次失败不会影响其他批次的处理，保证了整体流程的健壮性。最后返回收集到的 `results` 中间结果列表，供后续 Reduce 阶段使用。

#### 强制5要素

**入参**：

- `query: str` - 搜索查询字符串，必填
- `communities: List[dict]` - 社区数据列表，必填

**核心逻辑**：
读取批处理大小 → 按批次遍历社区 → 调用 \_process\_community\_batch 处理每批 → 收集有效中间结果 → 异常时跳过当前批次继续

**输出形式**：

- 返回值：`List[str]` - 中间结果列表
- 空结果：返回空列表
- 异常处理：单个批次失败不影响其他批次

**底层关键依赖**：

- `GLOBAL_SEARCH_SETTINGS["community_batch_size"]` - 批处理大小配置
- `self._process_community_batch()` - 批次处理方法
- `self.map_chain` - Map 阶段 LLM 处理链

**关键代码片段**：

```python
def _process_communities(self, query: str, communities: List[dict]) -> List[str]:
    batch_size = GLOBAL_SEARCH_SETTINGS["community_batch_size"]
    results = []
    
    for i in range(0, len(communities), batch_size):
        batch = communities[i:i+batch_size]
        try:
            batch_result = self._process_community_batch(query, batch)
            if batch_result and len(batch_result.strip()) > 0:
                results.append(batch_result)
        except Exception as e:
            print(f"批处理失败: {e}")
    
    return results
```

**特殊处理标注**：

- **批处理优化**：按批次处理社区，减少 LLM 调用次数
- **异常隔离**：单个批次失败不影响其他批次，使用 try-except 包裹每批处理

***

### 3.5 NaiveSearchTool.search()

#### 方法文字流程串讲

该方法是 NaiveSearchTool 的核心实现，执行纯向量搜索，不依赖知识图谱的社区结构或实体关系。方法开始时，记录 `overall_start` 开始时间。接下来解析输入：如果输入是字典且包含 `query` 键，则 `query` 为字典的 `query` 值；否则将输入转换为字符串作为 `query`。

然后检查缓存：构建缓存键 `f"naive:{query}"`，调用 `self.cache_manager.get(cache_key)` 查询缓存，如果缓存命中则打印 `"缓存命中: {query[:30]}..."` 并直接返回缓存结果。如果缓存未命中，则进入 try 块执行实际搜索。

在 try 块内，首先记录 `search_start` 搜索开始时间，生成查询的嵌入向量 `query_embedding = self.embeddings.embed_query(query)`。接下来查询带 embedding 的 Chunk 节点：构建 Cypher 查询 `MATCH (c:__Chunk__) WHERE c.embedding IS NOT NULL RETURN c.id, c.text, c.embedding LIMIT 100`，获取 100 个候选 Chunk 节点。

然后使用工具类对候选集进行排序：调用 `VectorUtils.rank_by_similarity(query_embedding, chunks_with_embedding, "embedding", self.top_k)`，传入查询向量、候选集、embedding 字段名和 top\_k 参数，返回按相似度排序的 Chunk 列表。取 top\_k 个结果 `results = scored_chunks[:self.top_k]`。计算搜索时间 `search_time = time.time() - search_start`，记录到 `self.performance_metrics["query_time"]`。

检查结果是否为空：如果 `results` 为空，则返回提示信息 `"没有找到与'{query}'相关的信息。\n\n{'data': {'Chunks':[] }}"`。如果结果非空，则格式化检索到的文档片段：初始化 `chunks_content` 和 `chunk_ids` 两个空列表，遍历 `results` 中的每个 item，提取 `chunk_id`（item.get("id", "unknown")）和 `text`（item.get("text", "")），如果 text 非空，则添加到 `chunks_content`（格式为 `"Chunk ID: {chunk_id}\n{text}"`），并将 `chunk_id` 添加到 `chunk_ids`。使用 `"\n\n---\n\n"` 连接 `chunks_content` 构建 `context`。

然后生成回答：记录 `llm_start` LLM 开始时间，调用 `self.query_chain.invoke()` 传入 `query`、`context`、`response_type`，获取 `answer`。计算 LLM 时间 `llm_time = time.time() - llm_start`，记录到 `self.performance_metrics["llm_time"]`。

确保回答中包含 Chunk ID：检查 answer 中是否包含 `"{'data': {'Chunks':"`，如果不包含，则添加引用信息：取前 5 个 chunk\_id，用逗号连接，格式化为 `f"\n\n{{'data': {{'Chunks':[{chunk_references}] }} }}"` 追加到 answer。然后缓存结果 `self.cache_manager.set(cache_key, answer)`。记录总耗时 `total_time = time.time() - overall_start`，记录到 `self.performance_metrics["total_time"]`。最后返回 answer。

如果 try 块中抛出异常，则进入 except 块：构建错误信息 `error_msg = f"搜索过程中出现错误: {str(e)}"`，打印错误信息，返回错误结果 `f"搜索过程中出错: {str(e)}\n\n{{'data': {{'Chunks':[] }} }}"`。整个流程简单直接，纯向量搜索，不依赖复杂的图谱结构，适合作为备选方案。

#### 强制5要素

**入参**：

- `query_input: Any` - 用户查询或包含查询的字典，必填

**核心逻辑**：
解析输入 → 检查缓存 → 生成查询嵌入 → 查询候选 Chunk → 相似度排序 → 格式化上下文 → LLM 生成答案 → 追加引用 → 缓存并返回

**输出形式**：

- 返回值：`str` - 基于检索结果生成的回答
- 空结果：返回提示信息，包含空 Chunk 列表
- 异常返回：返回错误信息，包含空 Chunk 列表

**底层关键依赖**：

- `self.embeddings.embed_query()` - 生成查询嵌入
- `self.graph.query()` - 查询 Chunk 节点
- `VectorUtils.rank_by_similarity()` - 相似度排序
- `self.query_chain.invoke()` - LLM 生成答案
- `self.cache_manager` - 缓存管理器

**关键代码片段**：

```python
def search(self, query_input: Any) -> str:
    overall_start = time.time()
    
    if isinstance(query_input, dict) and "query" in query_input:
        query = query_input["query"]
    else:
        query = str(query_input)
    
    cache_key = f"naive:{query}"
    cached_result = self.cache_manager.get(cache_key)
    if cached_result:
        return cached_result
    
    try:
        query_embedding = self.embeddings.embed_query(query)
        
        chunks_with_embedding = self.graph.query("""
        MATCH (c:__Chunk__)
        WHERE c.embedding IS NOT NULL
        RETURN c.id AS id, c.text AS text, c.embedding AS embedding
        LIMIT 100
        """)
        
        scored_chunks = VectorUtils.rank_by_similarity(
            query_embedding,
            chunks_with_embedding,
            "embedding",
            self.top_k
        )
        
        results = scored_chunks[:self.top_k]
        
        if not results:
            return f"没有找到与'{query}'相关的信息。\n\n{{'data': {{'Chunks':[] }} }}"
        
        chunks_content = []
        chunk_ids = []
        for item in results:
            chunk_id = item.get("id", "unknown")
            text = item.get("text", "")
            if text:
                chunks_content.append(f"Chunk ID: {chunk_id}\n{text}")
                chunk_ids.append(chunk_id)
        
        context = "\n\n---\n\n".join(chunks_content)
        
        answer = self.query_chain.invoke({
            "query": query,
            "context": context,
            "response_type": response_type
        })
        
        if "{'data': {'Chunks':" not in answer:
            chunk_references = ", ".join([f"'{id}'" for id in chunk_ids[:5]])
            answer += f"\n\n{{'data': {{'Chunks':[{chunk_references}] }} }}"
        
        self.cache_manager.set(cache_key, answer)
        return answer
    except Exception as e:
        return f"搜索过程中出错: {str(e)}\n\n{{'data': {{'Chunks':[] }} }}"
```

**特殊处理标注**：

- **缓存优化**：首先检查缓存，避免重复计算
- **引用强制**：确保答案中包含 Chunk ID 引用
- **候选集限制**：先取 100 个候选再排序，平衡效率和效果

***

## 四、同类逻辑对比表

| 功能名称                                  | 核心流程                                    | 入参                | 底层依赖 API                                           | 输出格式            | 差异场景                     |
| :------------------------------------ | :-------------------------------------- | :---------------- | :------------------------------------------------- | :-------------- | :----------------------- |
| LocalSearchTool.search()              | 规范化输入 → RAG链执行 → 返回答案                   | query\_input: Any | rag\_chain.invoke()                                | str             | 社区内精确搜索，支持聊天历史           |
| GlobalSearchTool.search()             | 获取社区数据 → Map阶段批处理 → Reduce阶段整合          | query\_input: Any | map\_chain, reduce\_chain                          | List\[str]      | 跨社区广泛搜索，概念性问题            |
| NaiveSearchTool.search()              | 查询候选Chunk → 相似度排序 → LLM生成答案             | query\_input: Any | VectorUtils.rank\_by\_similarity()                 | str             | 纯向量搜索，不依赖图谱结构，备选方案       |
| LocalSearchTool.structured\_search()  | 规范化输入 → RAG链执行 → 转换为标准格式 → 返回结构化结果      | query\_input: Any | results\_from\_documents(), results\_to\_payload() | Dict\[str, Any] | 需要完整检索结果和原始上下文的场景        |
| GlobalSearchTool.structured\_search() | 获取社区数据 → Map/Reduce → 转换为标准格式 → 返回结构化结果 | query\_input: Any | \_community\_results\_to\_retrieval()              | Dict\[str, Any] | 需要Map/Reduce中间结果和社区数据的场景 |

| 功能名称               | 核心流程                           | 入参                              | 底层依赖 API                         | 输出格式        | 差异场景                |
| :----------------- | :----------------------------- | :------------------------------ | :------------------------------- | :---------- | :------------------ |
| vector\_search()   | 生成查询嵌入 → Neo4j向量索引查询 → 返回实体ID  | query: str, limit: int          | db.index.vector.queryNodes()     | List\[str]  | 优先方法，精度高，依赖向量索引     |
| text\_search()     | 构建CONTAINS条件 → 文本匹配搜索 → 返回实体ID | query: str, limit: int          | MATCH ... WHERE ... CONTAINS     | List\[str]  | 备选方法，向量搜索失败时使用      |
| semantic\_search() | 生成查询嵌入 → 计算每个候选相似度 → 按相似度排序    | query: str, entities: List, ... | VectorUtils.cosine\_similarity() | List\[Dict] | 对已有实体重排序，不需要重新查询数据库 |

| 功能名称                      | 核心流程                               | 入参         | 底层依赖 API                              | 输出格式                   | 差异场景                |
| :------------------------ | :--------------------------------- | :--------- | :------------------------------------ | :--------------------- | :------------------ |
| extract\_keywords(Local)  | 检查缓存 → LLM提取 → JSON解析 → 标准化格式 → 缓存 | query: str | keyword\_chain.invoke(), json.loads() | Dict\[str, List\[str]] | 提取低级和高级关键词，用于本地搜索   |
| extract\_keywords(Global) | 检查缓存 → LLM提取 → JSON解析 → 格式转换 → 缓存  | query: str | keyword\_chain.invoke(), json.loads() | Dict\[str, List\[str]] | 主要关注高级关键词，用于全局搜索    |
| extract\_keywords(Naive)  | 直接返回空关键词字典                         | query: str | 无                                     | Dict\[str, List\[str]] | Naive RAG不需要复杂关键词提取 |

***

## 五、疑惑解答

**疑惑1：为什么需要BaseSearchTool抽象基类？所有工具都继承它有什么好处？**

从底层API来看，所有搜索工具都需要：初始化LLM和Embeddings模型、建立Neo4j连接、配置缓存、执行向量搜索/文本搜索、封装为LangChain工具。如果每个工具都独立实现这些功能，会有大量重复代码。通过`BaseSearchTool`抽象基类，将这些共性功能抽取出来：`__init__()`统一初始化基础设施，`vector_search()`和`text_search()`提供通用搜索方法，`get_tool()`提供统一的工具封装，`_setup_chains()`、`extract_keywords()`、`search()`作为抽象方法要求子类实现。从业务目标来看，这种设计保证了所有工具具有统一的接口，便于在多Agent系统中统一调用和替换，同时大大提高了代码复用性和可维护性。

**疑惑2：LocalSearchTool和GlobalSearchTool都有structured\_search()，它和search()有什么区别？**

从代码流程来看，`search(query_input: Any) -> str`是兼容旧接口的方法，它内部调用`structured_search()`，然后只返回`structured.get("answer", "未找到相关信息")`纯文本答案；而`structured_search(query_input: Any) -> Dict[str, Any]`返回完整的结构化结果，包含query、keywords、answer、retrieval\_results、raw\_context等所有信息。从业务目标来看，`search()`适用于只需要最终答案的简单场景，而`structured_search()`适用于需要检索结果、原始上下文、引用来源等完整信息的复杂场景，比如需要展示证据链、调试搜索过程、后续进一步处理等。

**疑惑3：GlobalSearchTool为什么要使用批处理？批处理大小如何选择？**

从底层API来看，如果不使用批处理，有20个社区就需要调用20次LLM，每次调用都有网络延迟和Token开销，效率很低。使用批处理后，假设批处理大小为5，则只需调用4次LLM，大大减少了调用次数。从代码流程来看，`_process_communities()`按`community_batch_size`分批处理，`_process_community_batch()`合并批次内的社区数据，一次性调用LLM处理。从业务目标来看，批处理大小需要平衡：太小则调用次数多效率低，太大则单次输入的Token过多可能超过LLM上下文限制。`GLOBAL_SEARCH_SETTINGS["community_batch_size"]`提供了合理的默认值，用户也可以根据自己的LLM上下文限制调整。

**疑惑4：NaiveSearchTool看起来功能最简单，它存在的意义是什么？**

从代码流程来看，NaiveSearchTool确实最简单：直接查询带embedding的Chunk节点，相似度排序，LLM生成答案，不依赖知识图谱的社区结构、实体关系、社区摘要等。从业务目标来看，它有两个重要意义：第一，作为备选方案，当知识图谱构建不完整、社区结构不合理、向量索引有问题时，NaiveSearchTool仍能提供基本的搜索能力；第二，作为基准方案，可以对比其他更复杂搜索工具的效果，验证GraphRAG带来的增益。根据微软的GraphRAG实现，项目已经有`__Chunk__`节点，NaiveSearchTool直接在这些节点上做向量化检索，是最基础的RAG方案。

***

## 六、规范修正

### 6.1 术语统一

| 原术语（代码中出现）        | 标准术语   | 说明                   |
| :---------------- | :----- | :------------------- |
| lc\_search\_tool  | 本地搜索工具 | 统一使用全称               |
| global\_retriever | 全局检索工具 | 统一使用"检索工具"或"搜索工具"    |
| naive\_retriever  | 简单检索工具 | 统一说明这是Naive RAG      |
| chunk             | 文本块    | 统一说明这是文档分块后的单元       |
| **Community**     | 社区节点   | 统一说明这是Neo4j中的社区节点标签  |
| **Chunk**         | 文本块节点  | 统一说明这是Neo4j中的文本块节点标签 |
| chat\_history     | 聊天历史   | 统一使用"聊天历史"           |

### 6.2 代码笔误与不规范修正

1. **BaseSearchTool.get\_tool()**：动态类名`DynamicSearchTool`建议改为更具体的名称，如`{self.__class__.__name__}Tool`，便于调试时区分不同工具。
2. **LocalSearchTool.extract\_keywords()**：JSON解析失败时返回`{"low_level": [], "high_level": []}`，建议同时记录警告日志，便于排查问题。
3. **GlobalSearchTool.\_get\_community\_data()**：LIMIT 20是硬编码，建议改为配置项`GLOBAL_SEARCH_SETTINGS["max_communities"]`，提高可配置性。
4. **变量命名一致性**：`LocalSearchTool`中使用`self.chat_history`，建议其他工具如果需要聊天历史也使用相同的变量名，提高一致性。

***

## 七、可复现实操步骤（傻瓜式落地）

### 步骤1：环境准备与依赖导入

**操作内容**：设置Python环境，导入tool子模块的必要类
**依赖API/模块**：`graphrag_agent.search.tool`、`graphrag_agent.models.get_models`
**执行目标**：确保所有依赖可用，为后续操作做好准备

**最简代码**：

```python
from graphrag_agent.search.tool import (
    BaseSearchTool,
    LocalSearchTool,
    GlobalSearchTool,
    NaiveSearchTool,
    HybridSearchTool,
    DeepSearchTool
)
from graphrag_agent.models.get_models import get_llm_model, get_embeddings_model
```

**注意事项**：

- 确保已正确配置`.env`文件中的Neo4j和LLM访问信息
- 确保Neo4j数据库已启动并包含已构建的索引数据（**Chunk**、**Entity**、\_\_Community\_\_节点）

***

### 步骤2：初始化搜索工具

**操作内容**：创建各种搜索工具实例
**依赖API/模块**：`LocalSearchTool()`、`GlobalSearchTool()`、`NaiveSearchTool()`
**执行目标**：创建可用的搜索工具实例，建立与Neo4j的连接

**最简代码**：

```python
# 创建LocalSearchTool实例（本地精确搜索）
local_tool = LocalSearchTool()

# 创建GlobalSearchTool实例（跨社区搜索，可指定level）
global_tool = GlobalSearchTool(level=2)

# 创建NaiveSearchTool实例（纯向量搜索，备选方案）
naive_tool = NaiveSearchTool()
```

**注意事项**：

- GlobalSearchTool的`level`参数指定社区层级，通常从0开始（最细粒度）
- 工具初始化时会自动加载LLM和Embeddings模型，建立Neo4j连接，初始化缓存
- 如果连接失败，检查`.env`配置和数据库状态

***

### 步骤3：使用search()方法执行简单搜索

**操作内容**：调用各工具的search()方法，获取纯文本答案
**依赖API/模块**：`LocalSearchTool.search()`、`GlobalSearchTool.search()`、`NaiveSearchTool.search()`
**执行目标**：验证基础搜索功能，获取搜索结果

**最简代码**：

```python
# 测试查询
query = "什么是机器学习？"

# LocalSearchTool示例（本地精确搜索）
local_answer = local_tool.search(query)
print("=" * 50)
print("LocalSearchTool结果：")
print(local_answer)

# GlobalSearchTool示例（跨社区搜索）
global_answer = global_tool.search(query)
print("=" * 50)
print("GlobalSearchTool结果：")
print(global_answer)

# NaiveSearchTool示例（纯向量搜索）
naive_answer = naive_tool.search(query)
print("=" * 50)
print("NaiveSearchTool结果：")
print(naive_answer)
```

**注意事项**：

- search()方法返回纯文本答案，兼容旧接口
- 搜索结果会自动缓存，重复查询相同问题会直接返回缓存
- 如果搜索结果为空，检查知识图谱中是否有相关数据

***

### 步骤4：使用structured\_search()获取完整结果

**操作内容**：调用structured\_search()方法，获取包含检索结果的完整结构化数据
**依赖API/模块**：`LocalSearchTool.structured_search()`、`GlobalSearchTool.structured_search()`
**执行目标**：获取完整的搜索信息，包括答案、检索结果、原始上下文等

**最简代码**：

```python
# LocalSearchTool结构化搜索
local_structured = local_tool.structured_search(query)
print("=" * 50)
print("LocalSearchTool结构化结果：")
print("查询：", local_structured["query"])
print("答案：", local_structured["answer"])
print("检索结果数量：", len(local_structured["retrieval_results"]))
print("原始上下文数量：", len(local_structured["raw_context"]))

# 查看前2个检索结果
print("\n前2个检索结果：")
for i, result in enumerate(local_structured["retrieval_results"][:2]):
    print(f"\n结果 {i+1}:")
    print(f"  证据：{result['evidence'][:100]}...")
    print(f"  来源：{result['source']}")
    print(f"  分数：{result['score']}")

# GlobalSearchTool结构化搜索
global_structured = global_tool.structured_search(query)
print("=" * 50)
print("GlobalSearchTool结构化结果：")
print("查询：", global_structured["query"])
print("中间结果数量：", len(global_structured["intermediate_results"]))
print("最终答案：", global_structured["final_answer"][:200], "...")
```

**注意事项**：

- structured\_search()返回更详细的信息，适合调试和高级用法
- retrieval\_results是标准格式，可直接用于多Agent系统
- raw\_context保留了原始的LangChain Document格式

***

### 步骤5：测试关键词提取功能

**操作内容**：调用extract\_keywords()方法，从查询中提取关键词
**依赖API/模块**：`LocalSearchTool.extract_keywords()`、`GlobalSearchTool.extract_keywords()`
**执行目标**：验证关键词提取功能，了解各工具的关键词策略

**最简代码**：

```python
# 测试关键词提取
keyword_query = "请介绍一下人工智能和机器学习的关系"

# LocalSearchTool关键词提取
local_keywords = local_tool.extract_keywords(keyword_query)
print("=" * 50)
print("LocalSearchTool关键词：")
print("低级关键词：", local_keywords["low_level"])
print("高级关键词：", local_keywords["high_level"])

# GlobalSearchTool关键词提取
global_keywords = global_tool.extract_keywords(keyword_query)
print("=" * 50)
print("GlobalSearchTool关键词：")
print("关键词列表：", global_keywords["keywords"])
print("低级关键词：", global_keywords["low_level"])
print("高级关键词：", global_keywords["high_level"])

# NaiveSearchTool关键词提取（返回空）
naive_keywords = naive_tool.extract_keywords(keyword_query)
print("=" * 50)
print("NaiveSearchTool关键词：", naive_keywords)
```

**注意事项**：

- LocalSearchTool提取低级和高级两类关键词
- GlobalSearchTool主要关注高级关键词（主题概念）
- NaiveSearchTool不需要关键词提取，返回空字典
- 关键词提取结果会自动缓存，避免重复LLM调用

***

### 步骤6：测试向量搜索和文本搜索

**操作内容**：直接调用vector\_search()和text\_search()方法，测试底层搜索能力
**依赖API/模块**：`BaseSearchTool.vector_search()`、`BaseSearchTool.text_search()`
**执行目标**：验证底层搜索功能，了解降级策略

**最简代码**：

```python
# 测试底层搜索方法（以local_tool为例，继承自BaseSearchTool）
search_term = "机器学习"

# 向量搜索（优先）
vector_results = local_tool.vector_search(search_term, limit=5)
print("=" * 50)
print("向量搜索结果：", vector_results)

# 文本搜索（备选）
text_results = local_tool.text_search(search_term, limit=5)
print("=" * 50)
print("文本搜索结果：", text_results)

# 测试语义搜索（对已有结果重排序）
# 首先获取一些实体
mock_entities = [
    {"id": "人工智能", "description": "计算机科学的一个分支", "embedding": [0.1, 0.2, 0.3]},
    {"id": "机器学习", "description": "人工智能的子领域", "embedding": [0.4, 0.5, 0.6]},
    {"id": "深度学习", "description": "机器学习的子领域", "embedding": [0.7, 0.8, 0.9]},
]
semantic_results = local_tool.semantic_search(search_term, mock_entities, top_k=2)
print("=" * 50)
print("语义搜索结果：")
for item in semantic_results:
    print(f"  {item['id']}: score={item.get('score', 'N/A')}")
```

**注意事项**：

- vector\_search()优先使用向量索引，失败时自动降级到text\_search()
- text\_search()使用CONTAINS条件进行文本匹配
- semantic\_search()对已有实体进行重排序，不需要重新查询数据库

***

### 步骤7：使用上下文管理器确保资源释放

**操作内容**：使用with语句创建搜索工具，确保资源正确释放
**依赖API/模块**：`BaseSearchTool.__enter__()`、`BaseSearchTool.__exit__()`
**执行目标**：学习正确的资源管理方式，避免资源泄漏

**最简代码**：

```python
# 使用with语句（上下文管理器）确保资源正确释放
query = "什么是深度学习？"

print("=" * 50)
print("使用上下文管理器：")

with LocalSearchTool() as tool:
    answer = tool.search(query)
    print("搜索结果：", answer[:200], "...")
    
# with块结束后，资源会自动释放，无需手动调用close()

# 对比：不使用with语句，需要手动调用close()
print("=" * 50)
print("不使用上下文管理器（手动关闭）：")

tool = LocalSearchTool()
try:
    answer = tool.search(query)
    print("搜索结果：", answer[:200], "...")
finally:
    tool.close()  # 确保资源释放
```

**注意事项**：

- 推荐使用with语句（上下文管理器），自动确保资源正确释放
- 如果不使用with语句，需要在finally块中手动调用close()
- close()主要关闭Neo4j连接，释放相关资源

***

## 八、关键模块总览

| 模块名称                                         | 负责功能                                | 在流程中的核心作用                   |
| :------------------------------------------- | :---------------------------------- | :-------------------------- |
| `BaseSearchTool`                             | 搜索工具抽象基类，提供通用功能（初始化、缓存、向量搜索、工具封装等）  | 工具层基础，所有具体搜索工具的父类，统一接口和共性能力 |
| `LocalSearchTool`                            | 本地搜索工具，基于向量检索实现社区内部精确查询，支持聊天历史      | 精确搜索，处理明确的事实性问题，利用社区内的实体和关系 |
| `GlobalSearchTool`                           | 全局搜索工具，基于知识图谱和Map-Reduce模式实现跨社区广泛查询 | 广泛搜索，处理概念性、总结性问题，利用跨社区知识整合  |
| `NaiveSearchTool`                            | 简单的Naive RAG搜索工具，只使用embedding进行向量搜索 | 备选方案，不依赖复杂图谱结构，提供最基础的RAG能力  |
| `HybridSearchTool`                           | 混合搜索工具，结合低级实体和高级社区的双级检索             | 综合搜索，同时提供细节和全局视角            |
| `DeepResearchTool`                           | 深度研究工具，多步骤思考-搜索-推理                  | 复杂推理，处理最复杂的问题               |
| `DeeperResearchTool`                         | 增强版深度研究工具，含社区感知、CoE、证据跟踪            | 高级推理，提供最全面的深度研究能力           |
| `ChainOfExplorationTool`                     | 链式探索工具，封装ChainOfExplorationSearcher | 关系链探索，从起始实体沿关系逐步扩展          |
| `HypothesisGeneratorTool`                    | 假设生成工具，生成多种分析假设                     | 假设生成，从多个角度分析问题              |
| `AnswerValidationTool`                       | 答案验证工具，评估答案质量                       | 质量保障，验证答案的相关性和可用性           |
| `CacheManager`                               | 缓存管理器，上下文和关键词感知                     | 性能优化，避免重复计算和LLM调用           |
| `VectorUtils`                                | 向量工具类，余弦相似度计算等                      | 数学基础，提供向量操作的核心功能            |
| `get_llm_model()` / `get_embeddings_model()` | 模型工厂，创建LLM和Embeddings               | 模型层基础，提供语言模型和嵌入模型           |
| `get_db_manager()`                           | 数据库连接管理器                            | 数据层基础，提供与Neo4j的连接           |

***

**总结**：Search/Tool 子模块通过抽象基类 `BaseSearchTool` 构建了一套完整的搜索工具体系，从简单的 `NaiveSearchTool` 到复杂的 `DeeperResearchTool`，覆盖了不同复杂度的搜索需求。所有工具共享统一的初始化流程、缓存机制、搜索方法和工具封装，通过工具注册表机制便于在多 Agent 系统中灵活选择和组合。该模块设计清晰、职责分明、可扩展性强，是 GraphRAG 项目中搜索功能的核心封装层。
