# 文档处理模块结构化总结

## 一、核心功能与整体流程

### 核心功能
该模块实现了**多格式文档的读取、清洗、分块**一体化处理流程，为 GraphRAG 系统提供高质量的文本输入。

### 整体流程

```
┌──────────────────────────────────────────────────────────────┐
│                    用户调用入口                               │
│         DocumentProcessor.process_directory()                │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第一阶段：文件读取 (FileReader)                              │
│  1. 遍历目录（递归/非递归）                                   │
│  2. 识别文件类型                                              │
│  3. 调用对应读取方法                                          │
│  4. 处理编码和异常                                            │
│  输出：List[(filepath, content)]                              │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第二阶段：文本分块 (ChineseTextChunker)                      │
│  1. 预处理大文本（分割成段落）                                 │
│  2. 中文分词（HanLP）                                         │
│  3. 智能分块（考虑句子边界）                                   │
│  4. 添加重叠（保持上下文连续性）                               │
│  输出：List[List[tokens]]                                     │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│  第三阶段：结果整合 (DocumentProcessor)                       │
│  1. 组装文件信息和分块结果                                     │
│  2. 计算统计信息                                              │
│  3. 错误处理和日志记录                                         │
│  输出：List[Dict{filepath, content, chunks, stats}]           │
└──────────────────────────────────────────────────────────────┘
```

---

## 二、模块拆分与方法详解

### 【模块 1】FileReader - 文件读取器

#### 初始化方法
```python
__init__(directory_path: str)
```
- **入参**：`directory_path` - 文件目录路径
- **核心逻辑**：保存目录路径到实例变量
- **输出**：无

---

#### 核心入口方法

**1. `read_files(file_extensions, recursive)`**
- **入参**：
  - `file_extensions`: `Optional[List[str]]` - 文件扩展名列表（如 `['.txt', '.pdf']`），`None` 表示读取所有支持格式
  - `recursive`: `bool` - 是否递归读取子目录（默认 `True`）
- **核心逻辑**：
  1. 定义支持的文件扩展名及对应处理方法字典
  2. 根据 `recursive` 参数选择递归或非递归模式
  3. 调用对应方法读取文件
  4. 捕获并记录目录遍历异常
- **输出形式**：`List[Tuple[str, str]]` - `(相对路径，文件内容)` 元组列表
- **特殊处理**：
  - 未指定扩展名时自动使用所有支持的格式
  - 异常时不中断，记录错误后继续处理其他文件

---

#### 分支逻辑方法

**2. `_read_files_recursive(root_dir, file_extensions, supported_extensions)`**
- **入参**：
  - `root_dir`: `str` - 当前处理的目录路径
  - `file_extensions`: `List[str]` - 要处理的文件扩展名列表
  - `supported_extensions`: `Dict` - 扩展名与处理函数的映射
- **核心逻辑**：
  1. 使用 `os.listdir()` 遍历目录内容
  2. 对子目录递归调用自身
  3. 对文件检查扩展名并调用对应读取方法
  4. 使用 `os.path.relpath()` 计算相对路径
- **输出形式**：`List[Tuple[str, str]]` - `(相对路径，文件内容)` 元组列表
- **特殊处理**：
  - 存储相对路径而非文件名，区分不同目录中的同名文件
  - 逐层捕获异常，确保单层目录错误不影响其他层

**3. `_process_files_in_dir(directory, filenames, file_extensions, supported_extensions)`**
- **入参**：
  - `directory`: `str` - 目录路径
  - `filenames`: `List[str]` - 文件名列表
  - 其他参数同上
- **核心逻辑**：
  1. 遍历文件名列表
  2. 检查扩展名并调用对应读取方法
  3. 直接使用文件名（非相对路径）
- **输出形式**：`List[Tuple[str, str]]` - `(文件名，文件内容)` 元组列表
- **特殊处理**：仅处理单层目录，不过滤文件夹（由调用方负责）

---

#### 具体实现方法（文件读取）

**4. `_read_txt(file_path)` - TXT 文件读取**
- **入参**：`file_path: str` - 文件路径
- **核心逻辑**：
  1. 尝试使用 UTF-8 编码读取
  2. 失败时使用 `chardet` 检测编码
  3. 使用检测到的编码重新读取
- **输出形式**：`str` - 文件内容
- **特殊处理**：
  - **编码降级策略**：UTF-8 → chardet 检测 → GBK 默认
  - 使用 `errors='replace'` 处理无法解码的字符
  - 读取前 10KB 用于编码检测（足够识别编码特征）

**5. `_read_pdf(file_path)` - PDF 文件读取**
- **入参**：`file_path: str`
- **核心逻辑**：
  1. 使用 `PyPDF2.PdfReader` 加载 PDF
  2. 逐页提取文本
  3. 用 `\n\n` 连接各页内容
- **输出形式**：`str` - 所有页面文本
- **特殊处理**：
  - 单页失败时插入占位符 `[第 X 页无法读取]`
  - 页面间用双换行分隔

**6. `_read_docx(file_path)` - Word 文档读取**
- **入参**：`file_path: str`
- **核心逻辑**：
  1. 使用 `python-docx` 加载文档
  2. 遍历所有段落提取文本
  3. 用 `\n` 连接段落
- **输出形式**：`str` - 段落文本
- **特殊处理**：
  - 仅提取段落，不提取表格、图片等非文本元素
  - 过滤空段落（由 `para.text` 自动处理）

**7. `_read_doc(file_path)` - 旧版 Word 文档读取**
- **入参**：`file_path: str`
- **核心逻辑**：三层降级策略
  1. 尝试 `win32com.client`（Windows 专用，调用 Word COM 接口）
  2. 尝试 `textract`（跨平台文本提取库）
  3. 尝试 `python-docx`（部分兼容）
- **输出形式**：`str` - 文件内容或警告信息
- **特殊处理**：
  - **平台差异化处理**：优先使用 Windows 原生接口
  - 所有方法失败时返回警告信息而非抛出异常

**8. `_read_csv(file_path)` - CSV 文件读取（纯文本模式）**
- **入参**：`file_path: str`
- **核心逻辑**：
  1. 使用 `csv.reader` 读取 CSV
  2. 将每行转为逗号分隔的字符串
  3. 用 `\n` 连接所有行
- **输出形式**：`str` - CSV 文本
- **特殊处理**：
  - 支持编码检测和降级（类似 TXT）
  - **注意**：丢失结构化信息，仅用于文本向量化

**9. `_read_json(file_path)` - JSON 文件读取**
- **入参**：`file_path: str`
- **核心逻辑**：
  1. 使用 `json.load()` 加载为 Python 对象
  2. 使用 `json.dumps()` 转为格式化字符串
- **输出形式**：`str` - 格式化的 JSON 文本（缩进 2 空格）
- **特殊处理**：
  - `ensure_ascii=False` 保留中文字符
  - 格式化输出提高可读性

**10. `_read_yaml(file_path)` - YAML 文件读取**
- **入参**：`file_path: str`
- **核心逻辑**：
  1. 使用 `yaml.load()` 加载为 Python 对象
  2. 使用 `yaml.dump()` 转为标准化 YAML 文本
- **输出形式**：`str` - 标准化的 YAML 文本
- **特殊处理**：
  - 使用 `CLoader` 加速解析
  - `default_flow_style=False` 使用块级格式

---

#### 辅助方法

**11. `read_csv_as_dicts(file_path)` / `read_json_as_dict(file_path)` / `read_yaml_as_dict(file_path)`**
- **用途**：返回结构化数据而非纯文本
- **输出形式**：`List[Dict]` 或 `Dict`
- **特殊处理**：异常时返回空集合而非抛出异常

**12. `read_txt_files()`**
- **用途**：快捷方法，仅读取 TXT 文件
- **实现**：调用 `read_files(['.txt'])`

**13. `list_all_files(recursive)`**
- **入参**：`recursive: bool` - 是否递归
- **核心逻辑**：
  - 递归模式：使用 `os.walk()` 遍历所有子目录
  - 非递归模式：使用 `os.listdir()` 仅读取当前目录
- **输出形式**：`List[str]` - 相对路径列表
- **特殊处理**：
  - 递归模式返回相对路径
  - 非递归模式返回文件名（包含文件夹）

---

### 【模块 2】ChineseTextChunker - 中文文本分块器

#### 初始化方法

**`__init__(chunk_size, overlap, max_text_length)`**
- **入参**：
  - `chunk_size`: `int` - 每个文本块的目标大小（tokens 数）
  - `overlap`: `int` - 相邻文本块的重叠大小
  - `max_text_length`: `int` - HanLP 处理的最大文本长度
- **核心逻辑**：
  1. 验证 `chunk_size > overlap`
  2. 加载 HanLP 中文分词模型
- **输出**：无
- **特殊处理**：参数不合法时抛出 `ValueError`

---

#### 核心入口方法

**`chunk_text(text)`**
- **入参**：`text: str` - 要分割的文本
- **核心逻辑**：
  1. 处理空文本或过短文本（直接分词返回）
  2. 预处理大文本（调用 `_preprocess_large_text`）
  3. 对每个段落调用 `_chunk_single_segment`
  4. 合并所有分块结果
- **输出形式**：`List[List[str]]` - 分块列表，每块是 token 列表
- **特殊处理**：
  - 文本长度 `< chunk_size/10` 时不分块，直接返回单个 token 列表

---

#### 分支逻辑方法

**`_preprocess_large_text(text)`**
- **入参**：`text: str`
- **核心逻辑**：
  1. 检查是否需要预分割（`len(text) > max_text_length`）
  2. 按段落分割（`\n\n` → `\n`）
  3. 重新组合段落，确保每段不超过目标大小
  4. 超长段落调用 `_split_long_paragraph`
- **输出形式**：`List[str]` - 分割后的段落列表
- **特殊处理**：
  - 动态计算 `target_segment_size`（避免过小分段）
  - 保留段落结构（优先按自然段落分割）

**`_split_long_paragraph(text, max_size)`**
- **入参**：
  - `text: str` - 超长段落
  - `max_size: int` - 最大分割大小
- **核心逻辑**：
  1. 按句子边界分割（正则 `r'([。！？.!?])'`）
  2. 重组句子和标点
  3. 无句子边界时按固定长度分割
  4. 重新组合句子，确保不超过 `max_size`
- **输出形式**：`List[str]` - 分割后的段落列表
- **特殊处理**：
  - 保留标点符号到句子末尾
  - 单个超长句子强制按固定长度分割

---

#### 具体实现方法

**`_chunk_single_segment(text)`**
- **入参**：`text: str` - 单个段落
- **核心逻辑**：
  1. 对整个段落分词（调用 `_safe_tokenize`）
  2. 滑动窗口分块：
     - 确定结束位置（`start_pos + chunk_size`）
     - 尝试在句子边界结束（调用 `_find_next_sentence_end`）
     - 提取当前块
     - 计算下一块起始位置（考虑重叠，调用 `_find_previous_sentence_end`）
  3. 防止无限循环（`start_pos >= end_pos` 时强制推进）
- **输出形式**：`List[List[str]]` - 分块结果
- **特殊处理**：
  - **句子边界优化**：允许略微超出 `chunk_size`（+100 tokens）以保持句子完整性
  - **重叠策略**：在句子边界处寻找重叠起始点

**`_safe_tokenize(text)`**
- **入参**：`text: str`
- **核心逻辑**：
  1. 检查文本长度（超过 `max_text_length` 时按字符分割）
  2. 调用 HanLP 分词
  3. 异常时按字符分割
- **输出形式**：`List[str]` - token 列表
- **特殊处理**：
  - 三重保护：长度检查 → 分词模型 → 字符级 fallback

---

#### 辅助方法

**`_is_sentence_end(token)`**
- **用途**：判断 token 是否为句子结束符
- **判断标准**：`['。', '！', '？']`

**`_find_next_sentence_end(tokens, start_pos)`**
- **用途**：从指定位置向后查找句子结束位置
- **输出**：句子结束位置的索引（`i + 1`）

**`_find_previous_sentence_end(tokens, start_pos)`**
- **用途**：从指定位置向前查找句子结束位置
- **输出**：句子结束位置的索引

**`process_files(file_contents)`**
- **用途**：批量处理多个文件
- **入参**：`List[Tuple[str, str]]` - `(filename, content)` 列表
- **输出**：`List[Tuple[str, str, List[List[str]]]]` - `(filename, content, chunks)` 列表

**`get_text_stats(text)`**
- **用途**：获取文本统计信息
- **输出**：`Dict` - 包含文本长度、分块预估、段落数等

---

### 【模块 3】DocumentProcessor - 文档处理器

#### 初始化方法

**`__init__(directory_path, chunk_size, overlap)`**
- **入参**：
  - `directory_path`: `str` - 文件目录路径
  - `chunk_size`: `int` - 分块大小
  - `overlap`: `int` - 分块重叠大小
- **核心逻辑**：
  1. 创建 `FileReader` 实例
  2. 创建 `ChineseTextChunker` 实例
- **输出**：无

---

#### 核心入口方法

**`process_directory(file_extensions, recursive)`**
- **入参**：
  - `file_extensions`: `Optional[List[str]]` - 文件扩展名列表
  - `recursive`: `bool` - 是否递归处理子目录
- **核心逻辑**：
  1. 调用 `FileReader.read_files()` 读取文件
  2. 遍历每个文件：
     - 创建文件结果字典（路径、内容、扩展名等）
     - 调用 `ChineseTextChunker.chunk_text()` 分块
     - 计算分块统计信息（数量、平均长度）
     - 捕获分块异常
  3. 返回所有文件处理结果
- **输出形式**：`List[Dict[str, Any]]` - 每个文件一个字典
  ```python
  {
      "filepath": "subdir/file.txt",      # 相对路径
      "filename": "file.txt",             # 仅文件名
      "extension": ".txt",
      "content": "文件内容",
      "content_length": 1234,
      "chunks": [["token1", "token2"], ...],  # 分块结果
      "chunk_count": 5,
      "chunk_lengths": [200, 210, ...],
      "average_chunk_length": 205.5
  }
  ```
- **特殊处理**：
  - 分块失败时记录错误但不中断其他文件处理
  - 打印调试信息（文件数量、类型）

---

#### 辅助方法

**`get_file_stats(file_extensions, recursive)`**
- **入参**：同 `process_directory`
- **核心逻辑**：
  1. 读取文件
  2. 统计扩展名分布
  3. 统计子目录信息
  4. 计算总文本长度和平均长度
- **输出形式**：`Dict[str, Any]`
  ```python
  {
      "total_files": 10,
      "extension_counts": {".txt": 5, ".pdf": 3, ...},
      "total_content_length": 50000,
      "average_file_length": 5000.0,
      "directories": ["subdir1", "subdir2"],
      "directory_count": 2
  }
  ```

**`get_extension_type(extension)`**
- **入参**：`extension: str` - 文件扩展名（如 `'.pdf'`）
- **核心逻辑**：查表返回文档类型描述
- **输出形式**：`str` - 类型描述（如 `'PDF 文档'`）

---

## 三、不同格式文件读取方法对比

| 方法             | 支持格式     | 核心库                              | 编码处理              | 特殊处理                       | 返回内容         |
| ---------------- | ------------ | ----------------------------------- | --------------------- | ------------------------------ | ---------------- |
| `_read_txt`      | `.txt`       | `codecs`, `chardet`                 | UTF-8 → chardet → GBK | 编码自动检测                   | 纯文本           |
| `_read_pdf`      | `.pdf`       | `PyPDF2`                            | 二进制读取            | 逐页提取，单页失败不影响其他页 | 所有页面文本     |
| `_read_markdown` | `.md`        | `codecs`                            | UTF-8                 | 保留 Markdown 语法             | Markdown 文本    |
| `_read_docx`     | `.docx`      | `python-docx`                       | 库内部处理            | 仅提取段落，忽略表格/图片      | 段落文本         |
| `_read_doc`      | `.doc`       | `win32com`/`textract`/`python-docx` | 库内部处理            | 三层降级策略                   | 段落文本或警告   |
| `_read_csv`      | `.csv`       | `csv`, `chardet`                    | UTF-8 → chardet → GBK | 转为纯文本，丢失结构化         | CSV 文本         |
| `_read_json`     | `.json`      | `json`                              | UTF-8                 | 格式化输出（缩进 2 空格）      | 格式化 JSON 文本 |
| `_read_yaml`     | `.yaml/.yml` | `yaml`                              | UTF-8                 | 标准化 YAML 格式               | 标准化 YAML 文本 |

---

## 四、关键设计亮点

### 1. **编码容错机制**
```
UTF-8 编码读取
    ↓ 失败
二进制读取前 10KB → chardet 检测编码
    ↓ chardet 不可用
默认使用 GBK 编码
    ↓ 仍失败
返回错误信息（不中断流程）
```

### 2. **智能分块策略**
- **句子边界感知**：不在句子中间切断
- **重叠设计**：保持上下文连续性（`overlap` 参数控制）
- **大文本预处理**：避免超过分词模型限制

### 3. **异常处理哲学**
- **局部失败不影响全局**：单个文件/页面/段落失败不影响其他部分
- **降级策略**：多种方法尝试，实在失败返回友好提示
- **详细日志**：记录每个错误便于调试

### 4. **路径管理**
- **递归模式**：使用相对路径区分不同目录的同名文件
- **非递归模式**：使用文件名（简洁但不唯一）

---

## 五、术语规范

| 术语                   | 含义             | 备注                        |
| ---------------------- | ---------------- | --------------------------- |
| **Token**              | 分词后的最小单位 | 中文中通常是一个词          |
| **Chunk**              | 文本分块         | 由多个 token 组成           |
| **Segment**            | 预处理后的段落   | 大文本分割后的中间产物      |
| **Overlap**            | 重叠大小         | 相邻 chunk 共享的 token 数  |
| **Recursive**          | 递归遍历         | 处理所有子目录              |
| **Encoding Detection** | 编码检测         | 使用 chardet 库识别文件编码 |

---

## 六、使用示例

```python
# 1. 创建文档处理器
processor = DocumentProcessor('documents/', chunk_size=512, overlap=50)

# 2. 获取文件统计
stats = processor.get_file_stats()
print(f"总文件数：{stats['total_files']}")

# 3. 处理所有文件
results = processor.process_directory(recursive=True)

# 4. 访问处理结果
for file_result in results:
    print(f"文件：{file_result['filepath']}")
    print(f"分块数：{file_result['chunk_count']}")
    print(f"第一块内容：{''.join(file_result['chunks'][0])}")
```

---

## 七、总结

该模块实现了**从文件读取到文本分块的完整 ETL 流程**，具有以下特点：

1. **多格式支持**：覆盖 9 种常见文件格式
2. **健壮性**：多层异常处理和编码降级策略
3. **智能化**：句子边界感知分块，保持语义完整性
4. **可扩展性**：模块化设计，易于添加新格式或分块策略
5. **易用性**：高层接口简洁，底层细节封装完善

适合用于 GraphRAG 系统的文档预处理、向量化前的文本准备等场景。