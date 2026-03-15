# NaiveRagAgent 代码总结

## 一、核心总览

### 核心定位

NaiveRagAgent 是 GraphRAG 系统中最简单的 RAG Agent 实现，主要解决了快速搭建基础 RAG 系统的问题。它继承自 BaseAgent，使用简单的向量检索（NaiveSearchTool）进行文档检索，工作流简单直接（从 retrieve 直接到 generate），关键词提取简化处理，回答生成逻辑清晰易懂。该模块适用于需要快速测试 RAG 系统的场景，以及作为其他复杂 Agent 的基准对比。

### 整体流程串讲

当创建 NaiveRagAgent 实例时，首先初始化 NaiveSearchTool 搜索工具，设置缓存目录为 "./cache/naive\_agent"，然后调用父类 BaseAgent 的构造函数初始化 LLM 模型、缓存管理器、工作流图等基础设施。

当用户调用 ask() 方法提问时，BaseAgent 的缓存检查机制会先尝试从全局缓存、快速缓存、常规缓存中查找答案，如果找到则直接返回。如果缓存未命中，则执行 LangGraph 工作流：首先进入 agent 节点，分析用户查询，调用 \_extract\_keywords() 提取关键词（简化版本，返回空列表），然后决定是否需要调用工具；如果需要调用工具，则进入 retrieve 节点，使用 NaiveSearchTool 进行向量检索；检索完成后，直接进入 generate 节点（因为 \_add\_retrieval\_edges() 直接添加了从 retrieve 到 generate 的边），调用 \_generate\_node() 生成最终答案；最后更新缓存并返回答案。

## 二、模块拆分

### 初始化模块

该模块的作用是初始化 NaiveRagAgent 的特定组件，位于整个模块的最底层。它主要通过 __init__() 构造函数实现，负责初始化 NaiveSearchTool 搜索工具，设置缓存目录，调用父类构造函数。

### 工具设置模块

该模块的作用是设置 Agent 使用的工具，位于初始化模块的上层。它主要通过 \_setup\_tools() 方法实现，负责返回 NaiveSearchTool 的工具实例。

### 工作流边配置模块

该模块的作用是配置 LangGraph 工作流的边，位于工具设置模块的上层。它主要通过 \_add\_retrieval\_edges() 方法实现，负责添加从 retrieve 到 generate 的直接边。

### 关键词提取模块

该模块的作用是提取查询关键词，位于工作流边配置模块的上层。它主要通过 \_extract\_keywords() 方法实现，是简化版本，直接返回空列表。

### 回答生成模块

该模块的作用是生成最终答案，位于关键词提取模块的上层，是整个模块的核心。它主要通过 \_generate\_node() 方法实现，负责检查缓存、构建提示词、调用 LLM、更新缓存、返回答案。

## 三、方法详细解析

### NaiveRagAgent.__init__()

#### 方法文字流程串讲

__init__() 是 NaiveRagAgent 的构造函数，用于初始化 Agent 的特定组件。方法开始时，首先创建 NaiveSearchTool 实例，用于简单的向量检索。接着，设置缓存目录为 "./cache/naive\_agent"。最后，调用父类 BaseAgent 的构造函数，传入缓存目录，初始化 LLM 模型、缓存管理器、工作流图等基础设施。

#### 强制 5 要素

- **入参**：无
- **核心逻辑**：初始化 NaiveSearchTool → 设置缓存目录 → 调用父类构造函数
- **输出形式**：无（构造函数）
- **底层关键依赖**：NaiveSearchTool、BaseAgent.__init__()
- **关键代码片段**：

```python
def __init__(self):
    self.search_tool = NaiveSearchTool()
    self.cache_dir = "./cache/naive_agent"
    super().__init__(cache_dir=self.cache_dir)
```

### NaiveRagAgent.\_setup\_tools()

#### 方法文字流程串讲

\_setup\_tools() 是 NaiveRagAgent 的工具设置方法，用于返回 Agent 使用的工具列表。方法直接返回一个包含 NaiveSearchTool 工具实例的列表，供 LangGraph 工作流使用。

#### 强制 5 要素

- **入参**：无
- **核心逻辑**：返回包含 NaiveSearchTool 的工具列表
- **输出形式**：返回工具列表
- **底层关键依赖**：NaiveSearchTool.get\_tool()
- **关键代码片段**：

```python
def _setup_tools(self) -&gt; List:
    return [
        self.search_tool.get_tool(),
    ]
```

### NaiveRagAgent.\_add\_retrieval\_edges()

#### 方法文字流程串讲

\_add\_retrieval\_edges() 是 NaiveRagAgent 的工作流边配置方法，用于添加从 retrieve 到 generate 的边。方法直接调用 workflow\.add\_edge() 添加一条从 "retrieve" 到 "generate" 的边，实现简单直接的工作流。

#### 强制 5 要素

- **入参**：
  - workflow: StateGraph（必填），LangGraph 工作流对象
- **核心逻辑**：添加从 retrieve 到 generate 的边
- **输出形式**：无（修改 workflow 对象）
- **底层关键依赖**：workflow\.add\_edge()
- **关键代码片段**：

```python
def _add_retrieval_edges(self, workflow):
    workflow.add_edge("retrieve", "generate")
```

### NaiveRagAgent.\_extract\_keywords()

#### 方法文字流程串讲

\_extract\_keywords() 是 NaiveRagAgent 的关键词提取方法，是一个简化版本。方法直接返回一个包含空的 low\_level 和 high\_level 关键词列表的字典，不进行实际的关键词提取。

#### 强制 5 要素

- **入参**：
  - query: str（必填），查询字符串
- **核心逻辑**：返回空的关键词字典
- **输出形式**：返回关键词字典
- **底层关键依赖**：无
- **关键代码片段**：

```python
def _extract_keywords(self, query: str) -&gt; Dict[str, List[str]]:
    return {"low_level": [], "high_level": []}
```

### NaiveRagAgent.\_generate\_node()

#### 方法文字流程串讲

\_generate\_node() 是 NaiveRagAgent 的回答生成方法，是整个模块的核心。方法开始时，首先从状态中获取 messages 列表，尝试安全地获取问题（倒数第三条消息）和检索结果（最后一条消息），如果获取失败则使用默认值。

接着，首先尝试从全局缓存中查找答案，如果找到则直接返回缓存结果，并记录执行日志。如果全局缓存未命中，则获取当前会话 ID，然后尝试从会话缓存中查找答案，如果找到则将结果同步到全局缓存，并返回缓存结果。

如果所有缓存都未命中，则创建提示模板（包含系统提示 NAIVE\_PROMPT 和用户提示 NAIVE\_RAG\_HUMAN\_PROMPT），构建 RAG 链（提示模板 → LLM → StrOutputParser），调用 rag\_chain.invoke() 传入上下文、问题、响应类型，获取 LLM 的响应。

然后，如果响应有效且长度大于 10，则同时更新会话缓存和全局缓存。最后，记录执行日志，返回包含响应消息的字典。整个过程中，如果任何步骤出现异常，会捕获异常并返回错误消息。

#### 强制 5 要素

- **入参**：
  - state: AgentState（必填），工作流状态
- **核心逻辑**：获取问题和检索结果 → 检查全局缓存 → 检查会话缓存 → 构建提示词 → 调用 LLM → 更新缓存 → 返回答案
- **输出形式**：返回包含 messages 的字典
- **底层关键依赖**：global\_cache\_manager.get()、cache\_manager.get()、ChatPromptTemplate、llm、StrOutputParser、global\_cache\_manager.set()、cache\_manager.set()
- **关键代码片段**：

```python
def _generate_node(self, state):
    messages = state["messages"]
    
    try:
        question = messages[-3].content if len(messages) &gt;= 3 else "未找到问题"
    except Exception:
        question = "无法获取问题"
        
    try:
        docs = messages[-1].content if messages[-1] else "未找到相关信息"
    except Exception:
        docs = "无法获取检索结果"

    global_result = self.global_cache_manager.get(question)
    if global_result:
        self._log_execution("generate", 
                        {"question": question, "docs_length": len(docs)}, 
                        "全局缓存命中")
        return {"messages": [AIMessage(content=global_result)]}

    thread_id = state.get("configurable", {}).get("thread_id", "default")
        
    cached_result = self.cache_manager.get(question, thread_id=thread_id)
    if cached_result:
        self._log_execution("generate", 
                        {"question": question, "docs_length": len(docs)}, 
                        "会话缓存命中")
        self.global_cache_manager.set(question, cached_result)
        return {"messages": [AIMessage(content=cached_result)]}

    prompt = ChatPromptTemplate.from_messages([
        ("system", NAIVE_PROMPT),
        ("human", NAIVE_RAG_HUMAN_PROMPT),
    ])

    rag_chain = prompt | self.llm | StrOutputParser()
    try:
        response = rag_chain.invoke({
            "context": docs, 
            "question": question, 
            "response_type": response_type
        })
        
        if response and len(response) &gt; 10:
            self.cache_manager.set(question, response, thread_id=thread_id)
            self.global_cache_manager.set(question, response)
        
        self._log_execution("generate", 
                        {"question": question, "docs_length": len(docs)}, 
                        response)
        
        return {"messages": [AIMessage(content=response)]}
    except Exception as e:
        error_msg = f"生成回答时出错: {str(e)}"
        self._log_execution("generate_error", 
                        {"question": question, "docs_length": len(docs)}, 
                        error_msg)
        return {"messages": [AIMessage(content=f"抱歉，我无法回答这个问题。技术原因: {str(e)}")]}
```

## 四、同类逻辑对比表

| 功能名称                    | 核心流程                     | 入参       | 底层依赖 API                    | 输出格式                       | 差异场景      |
| ----------------------- | ------------------------ | -------- | --------------------------- | -------------------------- | --------- |
| \_setup\_tools          | 返回 NaiveSearchTool 列表    | 无        | NaiveSearchTool.get\_tool() | List\[Tool]                | 简单向量检索场景  |
| \_add\_retrieval\_edges | 直接添加 retrieve→generate 边 | workflow | workflow\.add\_edge()       | 无                          | 简单工作流场景   |
| \_extract\_keywords     | 返回空关键词字典                 | query    | 无                           | Dict\[str, List\[str]]     | 不需要关键词的场景 |
| \_generate\_node        | 检查缓存 → 调用 LLM → 更新缓存     | state    | ChatPromptTemplate, llm     | Dict\[str, List\[Message]] | 简单回答生成场景  |

## 五、疑惑解答

（本模块代码逻辑清晰，暂无可标注的疑问）

## 六、规范修正

（本模块代码符合 PEP 8 规范，命名清晰，无需修正）

## 七、可复现实操步骤

### 步骤 1：导入并初始化 NaiveRagAgent

- **操作内容**：导入 NaiveRagAgent 并创建实例
- **依赖 API / 模块**：NaiveRagAgent
- **最简代码**：

```python
from graphrag_agent.agents.naive_rag_agent import NaiveRagAgent

agent = NaiveRagAgent()
```

- **注意事项**：确保环境变量配置正确
- **执行目标**：获得可使用的 NaiveRagAgent 实例

### 步骤 2：使用 ask() 方法提问

- **操作内容**：调用 agent.ask() 进行同步问答
- **依赖 API / 模块**：BaseAgent.ask()
- **最简代码**：

```python
answer = agent.ask("什么是 RAG？")
print("回答：", answer)
```

- **注意事项**：首次提问可能较慢，后续会利用缓存
- **执行目标**：获得查询的答案

### 步骤 3：关闭资源

- **操作内容**：调用 agent.close() 关闭资源
- **依赖 API / 模块**：BaseAgent.close()
- **最简代码**：

```python
agent.close()
```

- **注意事项**：确保所有缓存写入完成
- **执行目标**：安全关闭 Agent 资源

## 八、关键模块总览

| 模块名称                    | 负责功能                | 在流程中的核心作用      |
| ----------------------- | ------------------- | -------------- |
| __init__                | 初始化 NaiveSearchTool | 提供简单的向量检索工具    |
| \_setup\_tools          | 设置工具列表              | 为工作流提供检索工具     |
| \_add\_retrieval\_edges | 配置工作流边              | 实现简单直接的工作流     |
| \_extract\_keywords     | 提取关键词               | 简化版本，不做实际提取    |
| \_generate\_node        | 生成最终答案              | 整合检索结果和 LLM 生成 |

