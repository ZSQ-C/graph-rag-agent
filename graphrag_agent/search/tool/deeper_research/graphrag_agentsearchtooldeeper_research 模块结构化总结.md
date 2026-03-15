# graphrag_agent/search/tool/deeper_research 模块结构化总结

## 一、核心功能与整体流程

### 核心功能
该模块是**增强版深度研究工具的辅助功能集合**，从 `deeper_research_tool.py` 主文件中拆分出的核心子功能，实现了：
- **Chain-of-Exploration (COE) 增强搜索**：结合知识图谱探索的查询增强
- **多分支推理**：基于假设生成多个并行推理路径
- **矛盾检测与解决**：识别并分析证据间的矛盾信息
- **引用生成**：为答案自动添加证据引用标记
- **分支合并**：汇总多分支推理结果

### 整体流程

```
┌──────────────────────────────────────────────────────────────┐
│  用户发起深度研究查询                                         │
│  DeeperResearchTool.thinking() / search()                   │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第一阶段：查询增强 (enhancer.py)                             │
│  1. 提取关键词                                                 │
│  2. 社区感知搜索 (community_search)                           │
│  3. Chain-of-Exploration 探索                                │
│  4. 生成增强搜索策略                                          │
│  输出：community_context (包含探索路径和发现实体)              │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第二阶段：多分支推理 (branching.py)                          │
│  1. 生成多个假设 (hypotheses)                                 │
│  2. 为每个假设创建推理分支                                     │
│  3. 执行反事实分析 (counter-factual)                          │
│  4. 收集各分支证据                                            │
│  输出：Dict[branch_name → hypothesis + evidence]              │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第三阶段：矛盾检测 (branching.py)                            │
│  1. 收集所有证据                                              │
│  2. 检测数值/语义矛盾                                         │
│  3. 记录矛盾分析结果                                          │
│  输出：List[contradictions]                                   │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第四阶段：分支合并与引用生成 (branching.py)                   │
│  1. 汇总各分支推理结果                                        │
│  2. 切换到各分支合并推理                                      │
│  3. 为最终答案生成引用标记                                     │
│  输出：cited_answer + merged_reasoning                        │
└──────────────────────────────────────────────────────────────┘
```

---

## 二、目录结构

```
graphrag_agent/search/tool/deeper_research/
├── __init__.py          # 包初始化，导出核心函数
├── enhancer.py          # 查询增强功能 (COE + 社区搜索)
└── branching.py         # 多分支推理、矛盾检测、引用生成
```

---

## 三、模块拆分与方法详解

### 【模块 1】enhancer.py - 查询增强器

#### 核心方法

**`enhance_search_with_coe(tool, query, keywords)`**

**用途**：使用社区感知搜索结合 Chain-of-Exploration 对查询进行增强

- **入参**：
  - `tool`: `DeeperResearchTool` 实例（复用其依赖对象）
  - `query`: `str` - 原始查询
  - `keywords`: `Dict[str, List[str]]` - 关键词字典（包含 high_level 和 low_level 关键词）

- **核心逻辑**：
  1. **缓存检查**：检查是否已处理过相同查询
  2. **社区感知搜索**：调用 `community_search.enhance_search()` 获取社区上下文
  3. **焦点实体提取**：从搜索策略中提取或从关键词派生焦点实体
  4. **Chain-of-Exploration 探索**：
     - 使用 `chain_explorer.explore()` 沿知识图谱探索
     - 最多探索 3 步，聚焦前 3 个实体
  5. **发现实体记录**：将探索过程中发现的新实体加入搜索策略
  6. **缓存结果**：存储到 `_coe_cache` 避免重复计算

- **输出形式**：`Dict` - 社区上下文字典
  ```python
  {
      "search_strategy": {
          "focus_entities": ["实体 1", "实体 2"],
          "discovered_entities": ["新发现实体 1", "新发现实体 2"]
      },
      "exploration_results": {
          "exploration_path": [
              {"step": 1, "node_id": "实体 A", ...},
              {"step": 2, "node_id": "实体 B", ...}
          ]
      }
  }
  ```

- **特殊处理**：
  - **双重缓存机制**：
    - `_coe_cache`：缓存完整查询结果（key: `coe_search:{query}`）
    - `_specific_coe_cache`：缓存特定实体的探索结果（key: `coe:{query}:{entities}`）
  - **实体数量限制**：只取前 3 个焦点实体进行探索（避免过多分支）
  - **降级策略**：如果无焦点实体，使用 high_level + low_level 关键词

---

### 【模块 2】branching.py - 多分支推理

#### 核心方法 1：创建推理分支

**`create_multiple_reasoning_branches(tool, query_id, hypotheses)`**

**用途**：基于生成的假设创建多个并行推理分支

- **入参**：
  - `tool`: `DeeperResearchTool` 实例
  - `query_id`: `str` - 查询 ID
  - `hypotheses`: `Optional[List[str]]` - 假设列表（如未提供则自动生成）

- **核心逻辑**：
  1. **获取原始查询**：从 `tool.current_query_context` 中查找
  2. **生成假设**（如未提供）：
     - 使用 `query_generator.generate_multiple_hypotheses()` 生成
     - 缓存到 `_hypotheses_cache` 避免重复生成
  3. **创建推理分支**（最多 3 个）：
     - 调用 `thinking_engine.branch_reasoning(branch_name)` 创建分支
     - 记录假设到 `tool.explored_branches`
     - 添加推理步骤到证据链
  4. **反事实分析**（仅第一个分支）：
     - 执行 `counter_factual_analysis("假设 X 不成立")`
     - 缓存结果到 `_counter_cache`
     - 记录为证据
  5. **切换回主分支**：`thinking_engine.switch_branch("main")`

- **输出形式**：`Dict[str, Dict[str, Any]]` - 分支结果字典
  ```python
  {
      "branch_1": {
          "hypothesis": "假设内容 1",
          "counter_analysis": "反事实分析结果"  # 仅 branch_1 有
      },
      "branch_2": {
          "hypothesis": "假设内容 2"
      },
      "branch_3": {
          "hypothesis": "假设内容 3"
      }
  }
  ```

- **特殊处理**：
  - **假设缓存**：避免为同一查询重复生成假设
  - **反事实分析缓存**：key 为 `counter:{query_id}:{hypothesis}`
  - **仅第一个分支执行反事实分析**：减少计算开销
  - **分支命名规范**：`branch_1`, `branch_2`, `branch_3`

---

#### 核心方法 2：矛盾检测

**`detect_and_resolve_contradictions(tool, query_id)`**

**用途**：对已收集证据进行矛盾检测，并记录分析结果

- **入参**：
  - `tool`: `DeeperResearchTool` 实例
  - `query_id`: `str` - 查询 ID

- **核心逻辑**：
  1. **缓存检查**：检查是否已执行矛盾检测（key: `contradiction:{query_id}`）
  2. **收集所有证据**：
     - 从证据链跟踪器获取推理链（`get_reasoning_chain()`）
     - 提取所有证据 ID
  3. **执行矛盾检测**：调用 `evidence_tracker.detect_contradictions()`
  4. **记录矛盾分析**（如发现矛盾）：
     - 添加推理步骤（`add_reasoning_step()`）
     - 对每个矛盾分类处理：
       - **数值矛盾**（`numerical`）：记录冲突值
       - **语义矛盾**（`semantic`）：记录语义分析
       - **其他类型**：记录通用分析
     - 将矛盾分析添加为证据
  5. **缓存结果**：存储到 `_contradiction_detailed_cache`

- **输出形式**：`Dict` - 矛盾分析结果
  ```python
  {
      "contradictions": [
          {
              "type": "numerical",  # 或 "semantic"
              "context": "矛盾上下文",
              "value1": "值 1",
              "value2": "值 2",
              "analysis": "分析结果"
          },
          ...
      ],
      "step_id": "矛盾分析步骤 ID"  # 如无矛盾则为 None
  }
  ```

- **特殊处理**：
  - **详细缓存**：缓存完整的矛盾分析结果（避免重复检测）
  - **矛盾分类**：
    - `numerical`：数值冲突（如不同来源的数据不一致）
    - `semantic`：语义冲突（如陈述相互矛盾）
    - `unknown`：其他类型
  - **日志记录**：详细记录每个矛盾的分析结果

---

#### 核心方法 3：引用生成

**`generate_citations(tool, answer, query_id)`**

**用途**：使用证据链跟踪器为答案生成引用标记

- **入参**：
  - `tool`: `DeeperResearchTool` 实例
  - `answer`: `str` - 原始答案文本
  - `query_id`: `str` - 查询 ID

- **核心逻辑**：
  1. 调用 `evidence_tracker.generate_citations(answer)`
  2. 记录引用数量日志
  3. 返回带引用的答案

- **输出形式**：`str` - 添加了引用标记的答案
  ```
  原始答案："根据研究显示，全球气温正在上升 [1]。"
  引用后：  "根据研究显示，全球气温正在上升 [1][证据 ID: 123]。"
  ```

- **特殊处理**：
  - **委托给证据跟踪器**：具体引用逻辑由 `evidence_tracker` 实现
  - **日志记录**：输出添加的引用数量

---

#### 核心方法 4：分支合并

**`merge_reasoning_branches(tool, query_id)`**

**用途**：汇总多个推理分支的要点，返回 Markdown 格式的推理结果

- **入参**：
  - `tool`: `DeeperResearchTool` 实例
  - `query_id`: `str` - 查询 ID

- **核心逻辑**：
  1. **初始化 Markdown 输出**：添加标题 `## 多分支推理结果`
  2. **遍历所有分支**：
     - 获取分支信息（假设、步骤 ID）
     - 获取该分支的证据（`get_step_evidence()`）
     - 格式化输出：
       - 分支名称
       - 假设内容
       - 主要发现（最多 3 条证据，每条截断到 200 字符）
       - 反事实分析（如有，截断到 200 字符）
  3. **切换分支并合并**：
     - 切换到每个分支（`switch_branch(branch_name)`）
     - 合并到主分支（`merge_branches(branch_name, "main")`）
  4. **切换回主分支**：`switch_branch("main")`

- **输出形式**：`str` - Markdown 格式的推理结果
  ```markdown
  ## 多分支推理结果
  
  ### 分支：branch_1
  假设：基于假设 1 的内容
  
  主要发现:
  - 证据内容 1...
  - 证据内容 2...
  - 证据内容 3...
  
  反事实分析：如果假设 1 不成立，则...
  
  ### 分支：branch_2
  ...
  ```

- **特殊处理**：
  - **内容截断**：证据和反事实分析超过 200 字符时截断
  - **空分支处理**：如无分支返回空字符串
  - **分支切换**：使用 `thinking_engine` 的分支管理功能

---

## 四、缓存机制对比

| 缓存名称                        | 缓存键格式                        | 缓存内容                | 所属方法                             |
| ------------------------------- | --------------------------------- | ----------------------- | ------------------------------------ |
| `_coe_cache`                    | `coe_search:{query}`              | 完整的 COE 增强搜索结果 | `enhance_search_with_coe`            |
| `_specific_coe_cache`           | `coe:{query}:{entities}`          | 特定实体的探索结果      | `enhance_search_with_coe`            |
| `_hypotheses_cache`             | `{query_id}`                      | 生成的假设列表          | `create_multiple_reasoning_branches` |
| `_counter_cache`                | `counter:{query_id}:{hypothesis}` | 反事实分析结果          | `create_multiple_reasoning_branches` |
| `_contradiction_detailed_cache` | `contradiction:{query_id}`        | 矛盾检测结果            | `detect_and_resolve_contradictions`  |

---

## 五、关键技术概念解析

### 1. **Chain-of-Exploration (COE)**
**定义**：一种沿知识图谱进行多步探索的技术，类似思维链（Chain-of-Thought）但在图结构上执行。

**工作流程**：
```
查询 → 提取焦点实体 → 沿图谱探索 (最多 3 步) → 记录发现实体 → 增强搜索策略
```

**示例**：
```python
query = "人工智能在医疗领域的应用"
focus_entities = ["人工智能", "医疗"]

# 探索路径
Step 0: "人工智能"
Step 1: "机器学习" (人工智能的子领域)
Step 2: "医学影像分析" (机器学习的应用)
Step 3: "癌症检测" (医学影像分析的具体应用)

# 发现实体：["机器学习", "医学影像分析", "癌症检测"]
```

### 2. **多分支推理 (Multi-Branch Reasoning)**
**定义**：为同一问题创建多个并行推理路径，每条路径基于不同假设进行探索。

**架构**：
```
主分支 (main)
├── branch_1: 假设 A 成立 → 收集证据 → 反事实分析
├── branch_2: 假设 B 成立 → 收集证据
└── branch_3: 假设 C 成立 → 收集证据

最后：合并所有分支到主分支
```

**优势**：
- 避免单一推理路径的偏见
- 探索多种可能性
- 通过反事实分析增强鲁棒性

### 3. **反事实分析 (Counter-Factual Analysis)**
**定义**：假设某个命题不成立，分析可能的结果，用于验证推理的可靠性。

**示例**：
```python
假设："疫苗有效"
反事实假设："假设疫苗无效"
反事实分析：
  - 如果疫苗无效，感染率应该持平或上升
  - 实际数据显示感染率下降
  - 结论：反事实假设不成立，支持原假设
```

### 4. **矛盾检测 (Contradiction Detection)**
**定义**：识别不同证据源之间的冲突信息。

**矛盾类型**：
- **数值矛盾**（numerical）：不同来源的数据不一致
  ```
  来源 A："全球气温上升 1.5°C"
  来源 B："全球气温上升 2.0°C"
  ```
- **语义矛盾**（semantic）：陈述相互冲突
  ```
  来源 A："疫苗有效率 95%"
  来源 B："疫苗几乎没有效果"
  ```

---

## 六、使用示例

### 示例 1：查询增强
```python
from graphrag_agent.search.tool.deeper_research import enhance_search_with_coe

# 准备关键词
keywords = {
    "high_level": ["人工智能", "医疗"],
    "low_level": ["机器学习", "医学影像"]
}

# 执行增强搜索
community_context = enhance_search_with_coe(
    tool=deep_research_tool,
    query="人工智能在医疗领域的应用",
    keywords=keywords
)

# 获取探索结果
exploration_path = community_context["exploration_results"]["exploration_path"]
for step in exploration_path:
    print(f"Step {step['step']}: {step['node_id']}")
```

### 示例 2：多分支推理
```python
from graphrag_agent.search.tool.deeper_research import create_multiple_reasoning_branches

# 创建推理分支
branch_results = create_multiple_reasoning_branches(
    tool=deep_research_tool,
    query_id="query_123",
    hypotheses=[
        "人工智能可以提高诊断准确率",
        "人工智能可以降低医疗成本",
        "人工智能可以改善患者体验"
    ]
)

# 访问分支结果
for branch_name, info in branch_results.items():
    print(f"{branch_name}: {info['hypothesis']}")
    if 'counter_analysis' in info:
        print(f"  反事实分析：{info['counter_analysis']}")
```

### 示例 3：矛盾检测
```python
from graphrag_agent.search.tool.deeper_research import detect_and_resolve_contradictions

# 执行矛盾检测
result = detect_and_resolve_contradictions(
    tool=deep_research_tool,
    query_id="query_123"
)

# 分析矛盾
if result['contradictions']:
    print(f"发现 {len(result['contradictions'])} 个矛盾:")
    for contradiction in result['contradictions']:
        print(f"  类型：{contradiction['type']}")
        print(f"  分析：{contradiction['analysis']}")
else:
    print("未发现矛盾")
```

### 示例 4：分支合并
```python
from graphrag_agent.search.tool.deeper_research import merge_reasoning_branches

# 合并所有分支
merged_reasoning = merge_reasoning_branches(
    tool=deep_research_tool,
    query_id="query_123"
)

# 输出 Markdown 格式
print(merged_reasoning)
```

---

## 七、关键设计亮点

### 1. **模块化拆分**
```python
# 旧版：所有逻辑在 deeper_research_tool.py (单文件过长)
# 新版：拆分为独立模块
deeper_research/
├── enhancer.py      # 查询增强
├── branching.py     # 多分支推理
└── __init__.py      # 统一导出
```
- **优势**：提高可维护性，职责清晰

### 2. **工具类依赖注入**
```python
# 不直接实例化，而是接收 tool 实例
def enhance_search_with_coe(tool, query, keywords):
    # 复用 tool 中的 community_search, chain_explorer 等
    community_context = tool.community_search.enhance_search(query, keywords)
```
- **优势**：避免重复实例化，复用已有资源

### 3. **多层缓存策略**
```python
# 5 种不同的缓存，覆盖不同粒度的计算结果
tool._coe_cache                    # 完整查询结果
tool._specific_coe_cache           # 特定实体探索
tool._hypotheses_cache             # 生成的假设
tool._counter_cache                # 反事实分析
tool._contradiction_detailed_cache # 矛盾检测
```
- **优势**：避免重复计算，提高性能

### 4. **分支管理机制**
```python
# 使用 thinking_engine 的分支管理
thinking_engine.branch_reasoning("branch_1")   # 创建分支
thinking_engine.switch_branch("branch_1")      # 切换分支
thinking_engine.merge_branches("branch_1", "main")  # 合并分支
```
- **优势**：支持并行推理，保持上下文隔离

---

## 八、总结

`graphrag_agent/search/tool/deeper_research` 模块是 GraphRAG 系统**深度研究功能的核心实现**，具有以下特点：

### 功能特点
1. **智能化查询增强**：结合 COE 和社区感知搜索，发现隐藏关联
2. **多路径推理**：避免单一推理偏见，探索多种可能性
3. **矛盾识别**：自动检测证据冲突，提高答案可靠性
4. **可追溯引用**：为答案添加证据标记，增强可信度

### 技术亮点
1. **模块化架构**：职责分离，易于维护
2. **缓存优化**：5 层缓存避免重复计算
3. **分支管理**：支持并行推理和合并
4. **依赖注入**：复用工具实例，减少资源消耗

### 典型使用场景
- **复杂问题研究**：需要多角度分析的开放性问题
- **证据冲突处理**：不同来源信息存在矛盾
- **深度知识探索**：沿知识图谱发现隐藏关联
- **可解释 AI**：需要追溯答案来源和推理过程

**与其他模块的关系**：
- **上游**：依赖 `community_search`、`chain_explorer`、`evidence_tracker`
- **下游**：被 `DeeperResearchTool` 主类调用
- **核心地位**：实现 GraphRAG 的高级推理能力