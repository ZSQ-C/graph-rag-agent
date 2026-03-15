


# graphrag_agent/search/tool/reasoning 模块代码理解总结

## 一、核心总览

### 1.1 核心功能与适用场景

**核心功能**：
该模块是 GraphRAG 系统的**推理引擎核心**，实现了 GraphRAG 与 DeepSearch 的深度融合，提供以下核心能力：
- **多轮迭代思考管理**：假设生成、验证、反事实分析、分支推理
- **证据链全流程跟踪**：证据收集、矛盾检测、引用生成、可信度评分
- **社区感知知识检索**：利用知识图谱的聚类结构提供全局视角
- **知识图谱链式探索**：从起始实体开始，自主多步探索图谱
- **自然语言处理工具**：文本提取、知识库格式化、token 估算

**适用场景**：
- 复杂问题深度研究（需要多步推理、多角度分析）
- 知识密集型问答（医学、法律、金融等专业领域）
- 可解释 AI 应用（需要追溯答案来源和推理过程）
- 知识图谱探索（发现隐藏关联、多步路径推理）

---

### 1.2 整体执行流程与关键底层 API

```
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 1：初始化                                                       │
│  ├─ ThinkingEngine.__init__(llm)                         [底层：LLM]    │
│  ├─ EvidenceChainTracker.__init__()                 [底层：hashlib]    │
│  ├─ CommunityAwareSearchEnhancer.__init__(graph, emb, llm)            │
│  └─ ChainOfExplorationSearcher.__init__(graph, llm, emb)               │
└─────────────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 2：思考与推理 (ThinkingEngine)                                   │
│  ├─ initialize_with_query(query)                                       │
│  ├─ generate_initial_thinking()                   [底层：LLM.invoke]    │
│  ├─ generate_hypotheses()              [底层：re, json, LLM.invoke]    │
│  ├─ verify_hypothesis()                  [底层：LLM.invoke]             │
│  ├─ branch_reasoning(branch_name)                                      │
│  ├─ counter_factual_analysis(hypothesis)    [底层：LLM.invoke]          │
│  └─ generate_next_query()                 [底层：re, LLM.invoke]        │
└─────────────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 3：社区感知搜索 (CommunityAwareSearchEnhancer)                   │
│  ├─ enhance_search(query, keywords)                                     │
│  ├─ find_relevant_communities()         [底层：Neo4j.query, cosine]   │
│  ├─ extract_community_knowledge()       [底层：Neo4j.query]            │
│  └─ generate_search_strategy()          [底层：jieba, re, LLM.invoke]  │
└─────────────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 4：链式探索 (ChainOfExplorationSearcher)                        │
│  ├─ explore(query, starting_entities, max_steps, width)              │
│  ├─ _get_neighbors(entities)              [底层：Neo4j.query, pandas]  │
│  ├─ _score_neighbors_enhanced()         [底层：cosine_similarity]      │
│  └─ _decide_next_step_with_memory()      [底层：re, LLM.invoke]        │
└─────────────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 5：证据链管理 (EvidenceChainTracker)                             │
│  ├─ start_new_query(query, keywords)     [底层：hashlib.md5, time]      │
│  ├─ add_reasoning_step()                                               │
│  ├─ add_evidence(step_id, source_id, content, source_type)            │
│  ├─ detect_contradictions(evidence_ids)  [底层：re, LLM.invoke]        │
│  └─ generate_citations(answer)            [底层：_find_matching_evidence]│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、模块拆分（严格按顺序）

### 2.1 初始化模块

| 文件                      | 类/函数                                                  | 底层依赖                             | 功能                 |
| ------------------------- | -------------------------------------------------------- | ------------------------------------ | -------------------- |
| `thinking.py`             | `ThinkingEngine.__init__(llm)`                           | `llm` (LangChain ChatOpenAI)         | 初始化思考引擎状态   |
| `evidence.py`             | `EvidenceChainTracker.__init__()`                        | `hashlib`, `time`                    | 初始化证据链数据结构 |
| `community_enhance.py`    | `CommunityAwareSearchEnhancer.__init__(graph, emb, llm)` | `graph` (Neo4j), `embeddings`, `llm` | 初始化社区搜索组件   |
| `chain_of_exploration.py` | `ChainOfExplorationSearcher.__init__(graph, llm, emb)`   | `graph` (Neo4j), `llm`, `embeddings` | 初始化链式探索器     |

---

### 2.2 核心入口方法模块

| 文件                      | 方法                                                  | 功能                 |
| ------------------------- | ----------------------------------------------------- | -------------------- |
| `thinking.py`             | `think_deeper(query, context)`                        | 启动完整深度思考流程 |
| `thinking.py`             | `generate_next_query()`                               | 生成下一步搜索查询   |
| `evidence.py`             | `start_new_query(query, keywords)`                    | 开始新的查询跟踪     |
| `evidence.py`             | `detect_contradictions(evidence_ids)`                 | 检测证据间的矛盾     |
| `evidence.py`             | `generate_citations(answer)`                          | 为答案生成引用       |
| `community_enhance.py`    | `enhance_search(query, keywords)`                     | 增强搜索过程         |
| `chain_of_exploration.py` | `explore(query, starting_entities, max_steps, width)` | 启动图谱探索         |

---

### 2.3 分支逻辑方法模块

| 文件                   | 方法                                           | 分支逻辑                                            |
| ---------------------- | ---------------------------------------------- | --------------------------------------------------- |
| `thinking.py`          | `branch_reasoning(branch_name, base_branch)`   | 创建/切换推理分支                                   |
| `thinking.py`          | `merge_branches(source_branch, target_branch)` | 合并推理分支                                        |
| `thinking.py`          | `generate_next_query()` 的状态判断             | `has_query` / `no_query` / `answer_ready` / `error` |
| `community_enhance.py` | `find_relevant_communities()` 的评分分支       | 语义相似度 / 关键词匹配 / 社区重要性                |
| `evidence.py`          | `detect_contradictions()` 的类型分支           | 数值矛盾 / 语义矛盾                                 |

---

### 2.4 具体实现方法模块

| 文件                      | 方法                                                         | 具体实现                       |
| ------------------------- | ------------------------------------------------------------ | ------------------------------ |
| `thinking.py`             | `generate_hypotheses(initial_thinking)`                      | JSON 解析 → 正则 fallback      |
| `thinking.py`             | `counter_factual_analysis(hypothesis)`                       | 分支创建 → LLM 分析 → 分支合并 |
| `evidence.py`             | `_extract_numbers_with_context(text)`                        | 正则提取数值 + 上下文          |
| `evidence.py`             | `_detect_semantic_contradiction(c1, c2, e1, e2)`             | LLM 调用分析                   |
| `community_enhance.py`    | `extract_community_knowledge(communities)`                   | Neo4j 查询实体/关系            |
| `chain_of_exploration.py` | `_calculate_adaptive_width(step, query, neighbors, base_width)` | 步骤/邻居/复杂度因子           |
| `chain_of_exploration.py` | `_decide_next_step_with_memory()`                            | 记忆检查 → LLM 决策 → 记忆保存 |

---

### 2.5 辅助方法模块

| 文件                      | 方法                                                  | 辅助功能           |
| ------------------------- | ----------------------------------------------------- | ------------------ |
| `thinking.py`             | `add_reasoning_step(content)`                         | 添加推理步骤记录   |
| `thinking.py`             | `prepare_truncated_reasoning()`                       | 智能截断推理历史   |
| `thinking.py`             | `get_full_thinking()`                                 | 获取完整思考过程   |
| `evidence.py`             | `_extract_key_phrases(content)`                       | 提取关键短语       |
| `evidence.py`             | `_find_matching_evidence(statement)`                  | 为语句查找匹配证据 |
| `community_enhance.py`    | `_extract_temporal_info(text)`                        | 提取时序信息       |
| `chain_of_exploration.py` | `_get_entity_info(entities)`                          | 获取实体详情       |
| `chain_of_exploration.py` | `_rank_content_by_relevance(query_emb, content_list)` | 内容相关性排序     |
| `nlp.py`                  | `extract_between(text, start, end)`                   | 提取标记间内容     |
| `nlp.py`                  | `extract_sentences(text, max_sentences)`              | 提取句子           |
| `prompts.py`              | `kb_prompt(kbinfos, max_tokens)`                      | 格式化知识库提示   |
| `prompts.py`              | `num_tokens_from_string(text)`                        | 估算 token 数      |

---

## 三、方法详细解析（强制 5 要素）

### 【模块】thinking.py - ThinkingEngine

#### 方法 1：`__init__(llm)`

**入参**：
- `llm`: `LangChain ChatOpenAI` - **必填**，大语言模型实例

**核心逻辑**：
1. 保存 `llm` 实例
2. 初始化所有状态变量：
   - `all_reasoning_steps = []`
   - `msg_history = []`
   - `executed_search_queries = []`
   - `hypotheses = []`
   - `verification_chain = []`
   - `reasoning_tree = {}`
   - `current_branch = "main"`

**输出形式**：无（初始化实例变量）

**底层关键依赖**：
- LangChain `ChatOpenAI`

---

#### 方法 2：`think_deeper(query, context)`

**入参**：
- `query`: `str` - **必填**，用户问题
- `context`: `str` - **选填**，默认 `None`，上下文信息

**核心逻辑**：
1. 调用 `initialize_with_query(query)`
2. 如果有 context，调用 `add_reasoning_step()`
3. 调用 `generate_initial_thinking()`
4. 调用 `generate_hypotheses(initial_thinking)`
5. 循环调用 `verify_hypothesis(hypothesis)` 验证每个假设
6. 调用 `update_thinking_based_on_verification(verifications)`
7. 调用 `integrate_thinking_process()` 整合

**输出形式**：`str` - Markdown 格式的完整思考过程

**底层关键依赖**：
- `llm.invoke()`
- `re` (正则表达式)
- `json` (JSON 解析)

**关键代码片段**：
```python
def think_deeper(self, query, context=None):
    self.initialize_with_query(query)
    if context:
        self.add_reasoning_step(f"考虑以下背景信息：\n{context}")
    initial_thinking = self.generate_initial_thinking()
    hypotheses = self.generate_hypotheses(initial_thinking)
    verifications = [self.verify_hypothesis(h) for h in hypotheses]
    updated_thinking = self.update_thinking_based_on_verification(verifications)
    return self.integrate_thinking_process(...)
```

---

#### 方法 3：`generate_next_query()`

**入参**：无

**核心逻辑**：
1. 构建消息历史 `[SystemMessage + self.msg_history]`
2. 调用 `llm.invoke()` 生成思考
3. 清理响应（移除 `<think>` 标签）
4. 调用 `extract_queries()` 提取搜索查询
5. 状态判断分支：
   - 有查询 → `{"status": "has_query"}`
   - 无查询但有关键词 → `{"status": "answer_ready"}`
   - 无查询 → `{"status": "no_query"}`
   - 异常 → `{"status": "error"}`

**输出形式**：`Dict[str, Any]`
```python
{
    "status": "has_query",  # or "no_query", "answer_ready", "error"
    "content": "思考内容",
    "queries": ["查询1", "查询2"]
}
```

**底层关键依赖**：
- `llm.invoke()`
- `re.sub()` (正则替换)
- `extract_queries()` (nlp.py)

**特殊处理**：
- 状态判断分支逻辑
- 异常捕获并返回错误状态
- `<think>` 标签清理

---

#### 方法 4：`branch_reasoning(branch_name, base_branch)`

**入参**：
- `branch_name`: `str` - **必填**，分支名称
- `base_branch`: `str` - **选填**，默认 `"main"`，基础分支

**核心逻辑**：
1. 检查基础分支是否存在，不存在则使用 `"main"`
2. 创建新分支 `self.reasoning_tree[branch_name] = []`
3. 复制基础分支内容
4. 切换到新分支 `self.current_branch = branch_name`
5. 调用 `add_reasoning_step()` 记录分支创建

**输出形式**：无（修改实例变量）

**底层关键依赖**：无（纯内存操作）

**特殊处理**：
- 基础分支不存在时的 fallback
- 分支内容深拷贝

---

### 【模块】evidence.py - EvidenceChainTracker

#### 方法 1：`start_new_query(query, keywords)`

**入参**：
- `query`: `str` - **必填**，用户查询
- `keywords`: `Dict[str, List[str]]` - **必填**，关键词字典

**核心逻辑**：
1. 使用 `hashlib.md5()` 生成查询 ID：`{query}:{time.time()}` 哈希的前 10 位
2. 存储查询上下文到 `self.query_contexts[query_id]`

**输出形式**：`str` - 查询 ID（10 位 MD5）

**底层关键依赖**：
- `hashlib.md5()`
- `time.time()`

**关键代码片段**：
```python
def start_new_query(self, query, keywords):
    query_id = hashlib.md5(f"{query}:{time.time()}".encode()).hexdigest()[:10]
    self.query_contexts[query_id] = {
        "query": query, "keywords": keywords,
        "start_time": time.time(), "step_ids": []
    }
    return query_id
```

---

#### 方法 2：`add_evidence(step_id, source_id, content, source_type)`

**入参**：
- `step_id`: `str` - **必填**，步骤 ID
- `source_id`: `str` - **必填**，来源 ID（如块 ID）
- `content`: `str` - **必填**，证据内容
- `source_type`: `str` - **必填**，来源类型

**核心逻辑**：
1. 生成证据 ID：`hashlib.md5(f"{source_id}:{content[:50]}".encode()).hexdigest()[:10]`
2. 创建证据记录字典
3. 存储到 `self.evidence_items[evidence_id]`
4. 遍历 `self.reasoning_steps`，找到对应 step 并添加证据 ID

**输出形式**：`str` - 证据 ID（10 位 MD5）

**底层关键依赖**：
- `hashlib.md5()`

**特殊处理**：
- 内容取前 50 字符用于哈希（避免过长字符串）
- 避免重复添加（检查 evidence_id 是否已在 step.evidence_ids）

---

#### 方法 3：`detect_contradictions(evidence_ids)`

**入参**：
- `evidence_ids`: `List[str]` - **必填**，证据 ID 列表

**核心逻辑**：
1. 如果证据数 < 2，返回空列表
2. **分支 1：数值矛盾检测**（使用正则）
   - 调用 `_extract_numbers_with_context()` 提取数值+上下文
   - 使用 `_context_similarity()` 计算上下文相似度（Jaccard）
   - 如果相似度 > 0.7 且数值差异 > 0.1%，则标记矛盾
3. **分支 2：语义矛盾检测**（使用 LLM）
   - 调用 `_detect_semantic_contradiction()`
   - 避免重复检测已发现的数值矛盾
4. 保存矛盾信息到 `self.contradictions`

**输出形式**：`List[Dict]` - 矛盾列表
```python
[
    {
        "type": "numerical",  # or "semantic"
        "evidence1": "e1", "evidence2": "e2",
        "context": "上下文", "value1": 1.0, "value2": 1.1
    },
    ...
]
```

**底层关键依赖**：
- `re` (正则表达式)
- `_context_similarity()` (Jaccard 相似度)
- `llm.invoke()` (语义矛盾检测)

**特殊处理**：
- 避免重复检测（数值矛盾已发现则跳过语义检测）
- 上下文相似度阈值（0.7）
- 数值差异阈值（0.1%）

---

### 【模块】community_enhance.py - CommunityAwareSearchEnhancer

#### 方法 1：`find_relevant_communities(query, keywords, top_k)`

**入参**：
- `query`: `str` - **必填**，查询
- `keywords`: `Dict[str, List[str]]` - **必填**，关键词
- `top_k`: `int` - **选填**，默认 3，返回社区数量

**核心逻辑**：
1. 调用 `embeddings.embed_query(query)` 获取查询向量
2. Neo4j 查询社区摘要：`MATCH (c:__Community__) WHERE c.summary IS NOT NULL ...`
3. 对每个社区计算综合得分：
   - **语义相似度**（0.6 权重）：`cosine_similarity(query_emb, comm_emb)`
   - **关键词匹配**（0.3 权重）：
     - high_level 关键词每个 +2.0
     - low_level 关键词每个 +0.5
     - 归一化到 [0, 1]
   - **社区重要性**（0.1 权重）：`c.community_rank` 归一化
4. 排序返回 top_k

**输出形式**：`List[Dict]` - 相关社区列表

**底层关键依赖**：
- `Neo4j.query()` (Cypher 查询)
- `embeddings.embed_query()` (向量嵌入)
- `cosine_similarity()` (sklearn)

**特殊处理**：
- 三个维度的加权综合得分
- 关键词匹配加权（high_level 权重更高）

---

### 【模块】chain_of_exploration.py - ChainOfExplorationSearcher

#### 方法 1：`explore(query, starting_entities, max_steps, exploration_width)`

**入参**：
- `query`: `str` - **必填**，查询
- `starting_entities`: `List[str]` - **必填**，起始实体
- `max_steps`: `int` - **选填**，默认 5，最大探索步数
- `exploration_width`: `int` - **选填**，默认 3，基础探索宽度

**核心逻辑**：
1. 重置状态 `visited_nodes = set(starting_entities)`
2. 调用 `_generate_exploration_strategy()` 生成策略
3. 多步探索循环（共 `max_steps` 步）：
   - Step 1: `_get_neighbors(current_entities)` → 获取邻居
   - Step 2: `_calculate_adaptive_width()` → 动态宽度
   - Step 3: `_score_neighbors_enhanced()` → 邻居评分
   - Step 4: `_decide_next_step_with_memory()` → LLM 决策
   - Step 5: 更新 `visited_nodes` 和 `exploration_path`
   - Step 6: 收集实体/关系/内容/社区信息
4. 调用 `_rank_content_by_relevance()` 排序内容
5. 返回完整结果

**输出形式**：`Dict` - 探索结果
```python
{
    "entities": [...], "relationships": [...], "content": [...],
    "communities": [...], "exploration_path": [...],
    "statistics": {...}, "performance_metrics": {...}
}
```

**底层关键依赖**：
- `Neo4j.query()` (Cypher)
- `llm.invoke()` (LLM 决策)
- `cosine_similarity()` (sklearn)
- `time.time()` (性能计时)

**特殊处理**：
- 动态宽度控制（避免指数爆炸）
- 记忆机制（避免重复决策）
- 内容相关性排序（最终结果优化）

---

#### 方法 2：`_calculate_adaptive_width(step, query, neighbors, base_width)`

**入参**：
- `step`: `int` - **必填**，当前步骤
- `query`: `str` - **必填**，查询
- `neighbors`: `List` - **必填**，邻居列表
- `base_width`: `int` - **选填**，默认 3，基础宽度

**核心逻辑**：
1. **步骤因子**：`step_factor = max(0.5, 1.0 - step * 0.2)` → 步骤越深，宽度越小
2. **邻居因子**：`neighbor_factor = min(1.5, len(neighbors) / 10)` → 邻居越多，宽度越大
3. **复杂度因子**：调用 `_estimate_query_complexity(query)` → 复杂查询增加宽度
4. 最终宽度：`int(base_width * step_factor * neighbor_factor * complexity_factor)`
5. 边界限制：`max(1, min(5, adjusted_width))`

**输出形式**：`int` - 调整后的宽度（1-5）

**底层关键依赖**：
- `_estimate_query_complexity()` (查询复杂度估算)

**特殊处理**：
- 宽度在 [1, 5] 范围内
- 步骤因子防止指数爆炸（越探索越窄）

---

### 【模块】nlp.py

| 方法                                     | 入参                                        | 核心逻辑                             | 输出        | 底层依赖 |
| ---------------------------------------- | ------------------------------------------- | ------------------------------------ | ----------- | -------- |
| `extract_between(text, start, end)`      | `text: str`, `start: str`, `end: str`       | `re.findall(start + r"(.*?)" + end)` | `List[str]` | `re`     |
| `extract_sentences(text, max_sentences)` | `text: str`, `max_sentences: Optional[int]` | 正则分割句子                         | `List[str]` | `re`     |

---

### 【模块】prompts.py

| 方法                             | 入参                               | 核心逻辑                                       | 输出        | 底层依赖                         |
| -------------------------------- | ---------------------------------- | ---------------------------------------------- | ----------- | -------------------------------- |
| `num_tokens_from_string(text)`   | `text: str`                        | 调用 `count_tokens()`，失败时 `len(text) // 4` | `int`       | `count_tokens()` (get_models.py) |
| `kb_prompt(kbinfos, max_tokens)` | `kbinfos: Dict`, `max_tokens: int` | 按文档分组 chunks，限制 token 数               | `List[str]` | `defaultdict`                    |

---

## 四、疑惑解答（示例）

### 疑惑 1：为什么 `branch_reasoning` 需要复制基础分支内容？

**解答**：
推理分支设计目的是**并行探索多个假设**，每个分支需要独立的推理历史。如果不复制基础分支内容，分支间会共享同一份推理历史，导致分支切换时互相干扰。复制确保每个分支有自己的上下文，支持独立的推理路径。

**底层逻辑**：`self.reasoning_tree[branch_name] = []` 后逐步骤复制，使用 `step.copy()` 避免引用问题。

---

### 疑惑 2：为什么 `_calculate_adaptive_width` 随着步骤增加而减小宽度？

**解答**：
这是为了**避免探索的指数爆炸问题**。在图谱探索中：
- 第 0 步：N 个实体
- 第 1 步：N * 平均邻居数（可能爆炸）
- 第 2 步：指数增长

通过 `step_factor = max(0.5, 1.0 - step * 0.2)`，每步宽度最多减少 20%，早期步骤可以广泛探索，后期步骤聚焦重要路径。

---

### 疑惑 3：为什么 `detect_contradictions` 同时使用数值正则和 LLM 语义分析？

**解答**：
**双层矛盾检测策略**：
- **数值矛盾**：使用正则表达式，速度快、成本低、准确率高
- **语义矛盾**：使用 LLM，成本高但能处理复杂语义冲突

先做低成本的数值矛盾检测，已发现矛盾的证据对不再用 LLM 重复检测，兼顾效率和准确性。

---

## 五、规范修正

| 原术语/笔误       | 规范术语                        | 说明                                     |
| ----------------- | ------------------------------- | ---------------------------------------- |
| `knowledge graph` | `知识图谱` 或 `knowledge graph` | 中英文一致                               |
| `contradictions`  | `矛盾` 或 `contradictions`      | 保持统一                                 |
| `citations`       | `引用` 或 `citations`           | 保持统一                                 |
| `hypothesises`    | `hypotheses`                    | 修正拼写（hypothesis 复数为 hypotheses） |
| `counter_factual` | `counterfactual`                | 规范拼写（无连字符）                     |

---

## 六、同类逻辑对比

### 6.1 不同的矛盾检测方法

| 功能名称         | 入参                         | 底层依赖 API                         | 输出格式                                    | 差异点                                 |
| ---------------- | ---------------------------- | ------------------------------------ | ------------------------------------------- | -------------------------------------- |
| **数值矛盾检测** | `evidence_ids`               | `re` (正则)、`_context_similarity()` | `{"type": "numerical", "value1", "value2"}` | 快速、低成本、准确率高，仅处理数值冲突 |
| **语义矛盾检测** | `content1, content2, e1, e2` | `llm.invoke()`                       | `{"type": "semantic", "analysis"}`          | 成本高、能处理复杂语义冲突             |

---

### 6.2 不同的探索策略生成方法

| 功能名称         | 入参                         | 底层依赖 API                  | 输出格式                                  | 差异点                     |
| ---------------- | ---------------------------- | ----------------------------- | ----------------------------------------- | -------------------------- |
| **社区搜索策略** | `query, community_knowledge` | `jieba`, `re`, `llm.invoke()` | `{"focus_entities", "follow_up_queries"}` | 利用社区知识，偏向全局视角 |
| **图谱探索策略** | `query, starting_entities`   | `re`, `json`, `llm.invoke()`  | `{"focus_relations", "relation_weights"}` | 利用实体关系，偏向路径探索 |

---

### 6.3 不同的文本提取方法（nlp.py）

| 功能名称                 | 入参                     | 底层依赖 API   | 输出格式    | 差异点             |
| ------------------------ | ------------------------ | -------------- | ----------- | ------------------ |
| `extract_between`        | `text, start, end`       | `re.findall()` | `List[str]` | 提取标记间的内容   |
| `extract_from_templates` | `text, templates, regex` | `re.findall()` | `List[str]` | 基于模板或正则提取 |
| `extract_sentences`      | `text, max_sentences`    | `re.split()`   | `List[str]` | 分割句子           |

---

## 七、可复现实操步骤

### 步骤 1：环境准备与依赖安装

**操作内容**：安装项目依赖

**依赖的 API / 模块**：
- Python 3.10+
- `requirements.txt` 中的依赖（langchain, neo4j, PyPDF2, python-docx 等）

**最简代码示例**：
```bash
# 创建虚拟环境
python -m venv .venv

# 激活虚拟环境
# Windows
.venv\Scripts\activate
# Linux/Mac
source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

**注意事项**：
- 需要配置 `.env` 文件（复制 `.env.example`）
- 需要 Neo4j 数据库（本地或云端）
- 需要 OpenAI API Key（或兼容 API）

---

### 步骤 2：初始化核心模块

**操作内容**：创建并初始化 reasoning 模块的核心类

**依赖的 API / 模块**：
- `ThinkingEngine` (thinking.py)
- `EvidenceChainTracker` (evidence.py)
- `get_llm_model()` (get_models.py)

**最简代码示例**：
```python
from graphrag_agent.search.tool.reasoning.thinking import ThinkingEngine
from graphrag_agent.search.tool.reasoning.evidence import EvidenceChainTracker
from graphrag_agent.models.get_models import get_llm_model

# 1. 获取 LLM 模型
llm = get_llm_model()

# 2. 初始化思考引擎
thinking_engine = ThinkingEngine(llm)

# 3. 初始化证据链跟踪器
evidence_tracker = EvidenceChainTracker()
```

**注意事项**：
- 确保 `.env` 中的 `OPENAI_API_KEY` 已配置
- `get_llm_model()` 会自动从配置读取参数

---

### 步骤 3：执行完整思考流程

**操作内容**：启动深度思考，生成假设并验证

**依赖的 API / 模块**：
- `ThinkingEngine.think_deeper()`
- `ThinkingEngine.initialize_with_query()`
- `ThinkingEngine.generate_initial_thinking()`

**最简代码示例**：
```python
# 定义查询和上下文
query = "人工智能对就业市场的影响"
context = "考虑到技术发展速度和自动化趋势"

# 执行深度思考
final_thinking = thinking_engine.think_deeper(query, context)

# 打印结果
print(final_thinking)
```

**注意事项**：
- 这会调用多次 LLM，可能需要一些时间
- 上下文可选，没有可以传 `None`

---

### 步骤 4：证据链跟踪

**操作内容**：创建证据链，添加推理步骤和证据

**依赖的 API / 模块**：
- `EvidenceChainTracker.start_new_query()`
- `EvidenceChainTracker.add_reasoning_step()`
- `EvidenceChainTracker.add_evidence()`

**最简代码示例**：
```python
# 开始新查询
keywords = {
    "high_level": ["人工智能", "就业"],
    "low_level": ["自动化", "技术"]
}
query_id = evidence_tracker.start_new_query(query, keywords)

# 添加推理步骤
step_id = evidence_tracker.add_reasoning_step(
    query_id,
    search_query="查找就业数据",
    reasoning="需要了解历史就业趋势"
)

# 添加证据
evidence_id = evidence_tracker.add_evidence(
    step_id,
    source_id="doc_001",
    content="2023年自动化岗位同比增长15%",
    source_type="chunk"
)

print(f"添加证据成功，ID: {evidence_id}")
```

**注意事项**：
- 每个证据必须关联到一个推理步骤
- `source_type` 可以是 "chunk"、"document"、"community" 等

---

### 步骤 5：矛盾检测

**操作内容**：检测证据间的矛盾

**依赖的 API / 模块**：
- `EvidenceChainTracker.detect_contradictions()`
- `_extract_numbers_with_context()`
- `_detect_semantic_contradiction()`

**最简代码示例**：
```python
# 添加更多证据（用于检测矛盾）
evidence_id_2 = evidence_tracker.add_evidence(
    step_id,
    source_id="doc_002",
    content="2023年自动化岗位同比增长5%",
    source_type="chunk"
)

# 检测矛盾
contradictions = evidence_tracker.detect_contradictions([evidence_id, evidence_id_2])

# 打印结果
print(f"发现 {len(contradictions)} 个矛盾:")
for c in contradictions:
    print(f"类型: {c['type']}")
    if c['type'] == 'numerical':
        print(f"值1: {c['value1']}, 值2: {c['value2']}")
```

**注意事项**：
- 需要至少 2 个证据才能检测矛盾
- 数值矛盾需要数值在相似上下文中

---

### 步骤 6：引用生成

**操作内容**：为答案生成引用标记

**依赖的 API / 模块**：
- `EvidenceChainTracker.generate_citations()`
- `_extract_key_statements()`
- `_find_matching_evidence()`

**最简代码示例**：
```python
# 假设的答案文本
answer = "根据研究显示，2023年自动化岗位同比增长显著。"

# 生成引用
citation_result = evidence_tracker.generate_citations(answer)

# 打印带引用的答案
print("带引用的答案:")
print(citation_result["cited_answer"])

print("\n引用列表:")
for i, citation in enumerate(citation_result["citations"]):
    print(f"[{i+1}] {citation['source_id']}")
```

**注意事项**：
- 需要先添加证据才能生成引用
- 引用标记为 `[1]`、`[2]` 格式

---

### 步骤 7：获取完整推理链

**操作内容**：获取完整的推理链和统计信息

**依赖的 API / 模块**：
- `EvidenceChainTracker.get_reasoning_chain()`
- `EvidenceChainTracker.summarize_reasoning()`

**最简代码示例**：
```python
# 获取完整推理链
reasoning_chain = evidence_tracker.get_reasoning_chain(query_id)

print("推理链信息:")
print(f"查询: {reasoning_chain['query']}")
print(f"步骤数: {len(reasoning_chain['steps'])}")
print(f"矛盾数: {reasoning_chain['contradiction_count']}")

# 获取推理摘要
summary = evidence_tracker.summarize_reasoning(query_id)
print("\n推理摘要:")
print(f"总步数: {summary['steps_count']}")
print(f"总证据数: {summary['evidence_count']}")
print(f"耗时: {summary['duration_seconds']:.2f}秒")
```

**注意事项**：
- 必须提供正确的 `query_id`
- 摘要包含关键统计信息

---

## 八、关键模块总览

| 模块文件                  | 核心类/函数                                | 底层 API/依赖                                          | 负责功能                                     |
| ------------------------- | ------------------------------------------ | ------------------------------------------------------ | -------------------------------------------- |
| `thinking.py`             | `ThinkingEngine`                           | `llm.invoke()`, `re`, `json`                           | 多轮思考、假设生成验证、分支推理、反事实分析 |
| `evidence.py`             | `EvidenceChainTracker`                     | `hashlib.md5()`, `time`, `re`, `llm.invoke()`          | 证据收集、矛盾检测、引用生成、可信度评分     |
| `community_enhance.py`    | `CommunityAwareSearchEnhancer`             | `Neo4j.query()`, `cosine_similarity()`, `jieba`        | 社区搜索、社区知识提取、搜索策略生成         |
| `chain_of_exploration.py` | `ChainOfExplorationSearcher`               | `Neo4j.query()`, `llm.invoke()`, `cosine_similarity()` | 图谱探索、动态宽度、邻居评分、LLM 决策       |
| `nlp.py`                  | `extract_between()`, `extract_sentences()` | `re`                                                   | 文本提取、句子分割                           |
| `prompts.py`              | `kb_prompt()`, `num_tokens_from_string()`  | `defaultdict`, `count_tokens()`                        | 知识库格式化、token 估算                     |

---

## 九、完整功能复现示例

```python
"""
完整功能复现示例
需要先配置好 .env 文件和 Neo4j 数据库
"""

# 1. 导入模块
from graphrag_agent.search.tool.reasoning.thinking import ThinkingEngine
from graphrag_agent.search.tool.reasoning.evidence import EvidenceChainTracker
from graphrag_agent.models.get_models import get_llm_model

# 2. 初始化
llm = get_llm_model()
thinking_engine = ThinkingEngine(llm)
evidence_tracker = EvidenceChainTracker()

# 3. 深度思考
query = "人工智能对医疗行业的影响"
context = "考虑到诊断准确率、医疗成本等因素"
final_thinking = thinking_engine.think_deeper(query, context)
print("=== 完整思考过程 ===")
print(final_thinking)

# 4. 证据链跟踪
keywords = {
    "high_level": ["人工智能", "医疗"],
    "low_level": ["诊断", "成本"]
}
query_id = evidence_tracker.start_new_query(query, keywords)

step_id = evidence_tracker.add_reasoning_step(
    query_id,
    search_query="查找医疗 AI 数据",
    reasoning="需要了解 AI 在医疗中的应用"
)

evidence_tracker.add_evidence(
    step_id,
    source_id="study_2023",
    content="AI辅助诊断准确率提升20%",
    source_type="document"
)

# 5. 生成引用
answer = "AI技术显著提升了医疗诊断准确率，改善了患者预后。"
citation_result = evidence_tracker.generate_citations(answer)
print("\n=== 带引用的答案 ===")
print(citation_result["cited_answer"])

# 6. 获取推理链
chain = evidence_tracker.get_reasoning_chain(query_id)
print(f"\n=== 推理链统计 ===")
print(f"步骤数: {len(chain['steps'])}")
print(f"矛盾数: {chain['contradiction_count']}")
```

---

**总结**：
该模块通过将 GraphRAG 的图结构知识与 DeepSearch 的多步思考深度融合，实现了一个功能强大、可追溯、自适应的推理引擎。通过严格按模板要求的步骤操作，零基础用户也可以顺利复现完整功能。