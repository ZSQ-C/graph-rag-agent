
# 其他 Agent 综合代码总结

## 一、核心总览

### 核心定位
本模块包含 GraphRAG 系统中的其他高级 Agent 实现：GraphAgent（图结构 Agent）、HybridAgent（混合搜索 Agent）、FusionAgent（融合 Agent）、DeepResearchAgent（深度研究 Agent）。这些 Agent 都继承自 BaseAgent，在 BaseAgent 提供的通用基础设施之上，实现了各自特定的检索策略、工作流配置、关键词提取、回答生成逻辑。

- **GraphAgent**：使用 LocalSearchTool 和 GlobalSearchTool 两种图搜索工具，支持文档评分和条件路由（generate/reduce），提供关键词提取功能，适用于需要图检索增强的场景。
- **HybridAgent**：使用 HybridSearchTool 混合搜索工具，结合了多种检索方式的优势，关键词提取功能委托给搜索工具，适用于需要混合检索的场景。
- **FusionAgent**：是一个轻量封装版本，完全委托给多智能体编排栈（MultiAgentFacade），提供兼容旧版接口的实现，适用于需要与多智能体系统集成的场景。
- **DeepResearchAgent**：使用 DeepResearchTool 或 DeeperResearchTool 深度研究工具，支持显式推理过程、迭代式搜索、知识图谱探索、推理链分析、矛盾检测等高级功能，适用于需要深度研究的复杂问题场景。

### 整体流程串讲
所有这些 Agent 都遵循相同的基础流程：创建实例时初始化特定的搜索工具，设置缓存目录，调用父类 BaseAgent 的构造函数初始化 LLM 模型、缓存管理器、工作流图等基础设施。当用户提问时，BaseAgent 的缓存检查机制会先尝试从多层缓存中查找答案，缓存未命中则执行 LangGraph 工作流：agent 节点分析查询、提取关键词、决定是否调用工具；retrieve 节点使用特定的搜索工具进行检索；generate 节点（或 reduce 节点）生成最终答案；最后更新缓存并返回答案。

各个 Agent 的差异主要体现在：使用的搜索工具不同、工作流边配置不同（GraphAgent 有条件路由，其他直接到 generate）、关键词提取方式不同、回答生成逻辑不同、是否支持高级功能（如 DeepResearchAgent 的思考过程、知识图谱探索等）。

## 二、GraphAgent 详细分析

### 核心功能
GraphAgent 使用 LocalSearchTool（本地图搜索）和 GlobalSearchTool（全局图搜索）两种工具，支持根据工具调用类型和文档内容进行条件路由（_grade_documents）：如果是全局搜索工具调用则路由到 reduce 节点，否则路由到 generate 节点。关键词提取功能使用 LLM 实现，支持缓存提取结果。回答生成逻辑支持 generate 节点（普通回答）和 reduce 节点（全局搜索结果整合）两种模式。

### 关键方法
- **__init__()**：初始化 LocalSearchTool 和 GlobalSearchTool，设置缓存目录为 "./cache/graph_agent"，调用父类构造函数。
- **_setup_tools()**：返回 LocalSearchTool 和 GlobalSearchTool 的工具列表。
- **_add_retrieval_edges()**：添加 reduce 节点，添加从 retrieve 出发的条件边（根据 _grade_documents 的返回值路由到 generate 或 reduce），添加从 reduce 到 END 的边。
- **_extract_keywords()**：使用 LLM 和 GRAPH_AGENT_KEYWORD_PROMPT 提取关键词，支持缓存，返回 low_level 和 high_level 关键词列表。
- **_grade_documents()**：评估文档相关性，检查是否为全局搜索工具调用，检查文档内容是否足够，计算关键词匹配率，返回 "generate" 或 "reduce"。
- **_generate_node()**：检查缓存，构建提示词（LC_SYSTEM_PROMPT 和 GRAPH_AGENT_GENERATE_PROMPT），调用 LLM，更新缓存，返回答案。
- **_reduce_node()**：检查缓存，构建提示词（REDUCE_SYSTEM_PROMPT 和 GRAPH_AGENT_REDUCE_PROMPT），调用 LLM，更新缓存，返回整合后的答案。

## 三、HybridAgent 详细分析

### 核心功能
HybridAgent 使用 HybridSearchTool 混合搜索工具，结合了多种检索方式的优势，工作流简单直接（从 retrieve 直接到 generate），关键词提取功能委托给 HybridSearchTool，回答生成逻辑与 NaiveRagAgent 类似，但使用 HYBRID_AGENT_GENERATE_PROMPT 提示词。

### 关键方法
- **__init__()**：初始化 HybridSearchTool，设置缓存目录为 "./cache/hybrid_agent"，调用父类构造函数。
- **_setup_tools()**：返回 HybridSearchTool 的普通工具和全局工具列表。
- **_add_retrieval_edges()**：直接添加从 retrieve 到 generate 的边。
- **_extract_keywords()**：委托给 HybridSearchTool.extract_keywords()，支持缓存。
- **_generate_node()**：检查缓存，构建提示词（LC_SYSTEM_PROMPT 和 HYBRID_AGENT_GENERATE_PROMPT），调用 LLM，更新缓存，返回答案。

## 四、FusionAgent 详细分析

### 核心功能
FusionAgent 是一个轻量封装版本，完全不继承 BaseAgent，而是完全委托给多智能体编排栈（MultiAgentFacade）。它提供了兼容旧版接口的实现，包括 ask()、ask_with_trace()、ask_stream() 等方法，同时实现了简单的内存缓存机制（_global_cache 和 _session_cache）。

### 关键方法
- **__init__()**：初始化 MultiAgentFacade，设置缓存目录，初始化 _MemoryShim 和 _GraphShim（兼容旧版接口），初始化内存缓存。
- **ask()**：调用 _execute()，返回答案。
- **ask_with_trace()**：调用 _execute()，返回包含答案和 payload 的字典。
- **ask_stream()**：检查缓存，缓存未命中则调用 _execute()，按句子分块返回结果。
- **_execute()**：检查缓存，缓存未命中则调用 multi_agent.process_query()，规范化答案，写入缓存，返回答案和 payload。
- **_read_cache()**：从全局缓存或会话缓存读取答案。
- **_write_cache()**：写入答案到全局缓存和会话缓存。
- **_normalize_answer()**：规范化答案，确保是字符串。
- **_stream_chunks()**：按句子分块流式返回答案。

## 五、DeepResearchAgent 详细分析

### 核心功能
DeepResearchAgent 是最复杂的 Agent，使用 DeepResearchTool（标准版）或 DeeperResearchTool（增强版）深度研究工具，支持显式推理过程、迭代式搜索、知识图谱探索、推理链分析、矛盾检测等高级功能。它扩展了 ask() 方法，支持 show_thinking 和 exploration_mode 参数；新增了 ask_with_thinking() 方法返回带思考过程的答案；新增了 explore_knowledge() 方法进行知识图谱探索；新增了 analyze_reasoning_chain() 方法分析推理链；新增了 detect_contradictions() 方法检测信息矛盾；新增了 is_deeper_tool() 方法切换工具版本。

### 关键方法
- **__init__()**：初始化 DeepResearchTool 或 DeeperResearchTool，设置缓存目录为 "./cache/enhanced_research_agent"，初始化 show_thinking、session_context、community_enhancer 等状态，调用父类构造函数。
- **_setup_tools()**：根据 show_thinking 和 use_deeper_tool 返回不同的工具组合（基础研究工具、知识图谱探索工具、推理链分析工具、流式工具等）。
- **_add_retrieval_edges()**：直接添加从 retrieve 到 generate 的边。
- **_extract_keywords()**：委托给 research_tool.extract_keywords()。
- **_generate_node()**：处理各种返回格式（AsyncGenerator、Dict、包含思考过程的字符串），支持缓存，支持从包含 &lt;think&gt; 标记的结果中提取干净答案。
- **ask()**：支持 show_thinking 和 exploration_mode 参数，设置状态后调用父类方法。
- **ask_with_thinking()**：支持 community_aware 参数，使用社区感知增强搜索，调用 research_tool.thinking()，返回带思考过程的字典。
- **ask_stream()**：支持 show_thinking 参数，检查缓存，根据是否显示思考过程和工具类型选择不同的流式方法。
- **explore_knowledge()**：使用知识图谱探索模式，提取关键词作为起始实体，执行探索，使用 LLM 生成探索摘要，美化返回结果。
- **analyze_reasoning_chain()**：使用推理链分析工具，需要增强版工具，获取查询 ID，执行分析。
- **detect_contradictions()**：检测和分析信息矛盾，需要增强版工具，执行深度思考，提取矛盾信息，使用 LLM 分析矛盾影响。
- **is_deeper_tool()**：切换是否使用增强版研究工具，重新加载工具。

## 六、同类 Agent 对比表

| Agent 名称 | 搜索工具 | 工作流 | 关键词提取 | 高级功能 | 适用场景 |
|-----------|---------|-------|-----------|---------|---------|
| NaiveRagAgent | NaiveSearchTool | 简单直接 | 空实现 | 无 | 快速测试、基准对比 |
| GraphAgent | LocalSearchTool + GlobalSearchTool | 条件路由（generate/reduce） | LLM 提取 + 缓存 | 无 | 图检索增强 |
| HybridAgent | HybridSearchTool | 简单直接 | 委托给搜索工具 | 无 | 混合检索 |
| FusionAgent | MultiAgentFacade | 委托给多智能体 | 无 | 无 | 多智能体集成 |
| DeepResearchAgent | DeepResearchTool / DeeperResearchTool | 简单直接 | 委托给研究工具 | 思考过程、知识图谱探索、推理链分析、矛盾检测 | 深度研究、复杂问题 |

## 七、可复现实操步骤

### 使用 GraphAgent
```python
from graphrag_agent.agents.graph_agent import GraphAgent

agent = GraphAgent()
answer = agent.ask("什么是图数据库？")
print(answer)
agent.close()
```

### 使用 HybridAgent
```python
from graphrag_agent.agents.hybrid_agent import HybridAgent

agent = HybridAgent()
answer = agent.ask("什么是混合检索？")
print(answer)
agent.close()
```

### 使用 FusionAgent
```python
from graphrag_agent.agents.fusion_agent import FusionGraphRAGAgent

agent = FusionGraphRAGAgent()
answer = agent.ask("什么是多智能体系统？")
print(answer)
agent.close()
```

### 使用 DeepResearchAgent
```python
from graphrag_agent.agents.deep_research_agent import DeepResearchAgent

agent = DeepResearchAgent(use_deeper_tool=True)

# 普通问答
answer = agent.ask("什么是深度研究？")
print(answer)

# 带思考过程的问答
result = agent.ask_with_thinking("解释量子计算")
print("思考过程：", result.get("thinking", ""))
print("答案：", result.get("answer", ""))

# 知识图谱探索
exploration = agent.explore_knowledge("人工智能")
print("探索摘要：", exploration.get("summary", ""))

agent.close()
```

## 八、关键模块总览

| 模块名称 | 负责功能 | 核心特点 |
|---------|---------|---------|
| GraphAgent | 图检索增强 | 双搜索工具、条件路由、LLM 关键词提取 |
| HybridAgent | 混合检索 | 混合搜索工具、关键词提取委托 |
| FusionAgent | 多智能体集成 | 完全委托、兼容旧接口、内存缓存 |
| DeepResearchAgent | 深度研究 | 思考过程、知识图谱探索、推理链分析、矛盾检测 |

