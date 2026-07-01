# 父子分片方案设计文档（LLM 父分片版）

## 1. 背景与目标

在 RAG（检索增强生成）场景中，文档分片策略对最终检索质量至关重要。传统的基于分隔符或固定长度的分片方式存在以下局限：

- 语义割裂：按字符数或换行符切分，容易将同一语义段落强行打断；
- 上下文丢失：子分片过小导致召回的内容缺乏背景；
- 检索粒度单一：无法同时兼顾精确命中（细粒度）与上下文完整性（粗粒度）。

**父子分片（Parent-Child Chunking）** 方案通过两层结构解决上述问题：
- **父分片（Parent Chunk）**：语义完整的较大文本块，提供上下文；
- **子分片（Child Chunk）**：父分片的细粒度切分，用于向量检索命中。

本方案的核心创新在于：**使用 LLM 对父分片进行语义感知切分**，最大程度保留段落语义完整性，并将父分片最大长度限制在 **5000 字符**。

---

## 2. 整体架构

```
原始文档
    │
    ▼
┌─────────────────────────────────────┐
│   文本预处理（CleanProcessor）  │
│  · 去除多余空格/换行/制表符（可选）      │
│  · 去除 URL 和邮件地址（可选）          │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│    LLM 父分片（LLM Parent Splitter）  │
│  · 输入：清洗后的全文                  │
│  · 模型：用户指定的 LLM               │
│  · 约束：每个父分片 ≤ 5000 字符        │
│  · 策略：语义感知切分，保持段落完整      │
└─────────────────────────────────────┘
    │ 父分片列表
    ▼
┌─────────────────────────────────────┐
│    规则子分片（Fixed Child Splitter）  │
│  · 输入：每个父分片                    │
│  · 分隔符：可配置（默认 \n）           │
│  · 最大长度：可配置（默认 512 字符）    │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│    ParentChildStructureChunk 输出   │
│  · parent_content: str              │
│  · child_contents: list[str]        │
│  · parent_mode: "llm"               │
└─────────────────────────────────────┘
```

---

## 3. 父分片策略：LLM 语义切分

### 3.1 切分原则

LLM 在切分时遵循以下优先级：

| 优先级 | 原则 |
|--------|------|
| 1 | **语义完整性**：每个父分片应围绕一个完整的主题或论点 |
| 2 | **长度约束**：每个父分片不超过 **5000 字符** |
| 3 | **边界对齐**：优先在自然段落边界（空行、标题）处切分 |
| 4 | **最小冗余**：避免同一内容出现在多个父分片中 |

### 3.2 LLM Prompt 设计

```
你是一个专业的文档分析助手，负责将长文本切分为语义完整的段落（父分片）。

**切分规则：**
1. 每个父分片必须围绕一个完整的主题或论点，保持语义自洽；
2. 每个父分片的字符长度不得超过 5000 个字符；
3. 优先在自然段落边界（空行、章节标题、逻辑转折处）切分；
4. 不得将一个完整的句子切断，切分点必须在句子结束处；
5. 切分结果用 <SPLIT> 标记分隔，不要添加任何额外说明或修改原文内容。

**输出格式示例：**
第一段内容...<SPLIT>第二段内容...<SPLIT>第三段内容...

**待切分文本：**
{input_text}
```

### 3.3 超长文本处理（滑动窗口预分割）

**模型上下文规格：** 采用 256k 上下文窗口的模型，有效输入上下文为 128k（约 **100,000 汉字/字符**，含 Prompt 占用）。

基于此，处理逻辑分为两条路径：

```
原始文本
    │
    ├─── len ≤ 100,000 字符 ──→ 直接整体提交 LLM（单次调用）
    │
    └─── len > 100,000 字符 ──→ 滑动窗口预分割
                                      │
                                      ▼
                            按 80,000 字符分割为若干"窗口块"
                            （相邻窗口保留 500 字符重叠，
                              确保边界处语义连续性）
                                      │
                                      ▼
                            对每个窗口块独立调用 LLM 进行语义切分
                                      │
                                      ▼
                            合并所有窗口的切分结果
                            （去除重叠区域产生的重复父分片）
                                      │
                                      ▼
                                最终父分片列表
```

**窗口参数：**
- 单次提交阈值：**100,000 字符**（不超过此值无需预分割）
- 窗口大小：**80,000 字符**（留出约 20,000 字符给 Prompt 和输出）
- 重叠大小：**500 字符**（确保切割点附近的语义不丢失）
- 重叠片段合并规则：相邻窗口边界处若出现内容高度重复的父分片，保留前者并丢弃后者

### 3.4 父分片后处理

LLM 返回结果后执行以下后处理步骤：

1. 按 `<SPLIT>` 标记切分为列表；
2. 过滤空片段（`strip()` 后长度为 0）；
3. **长度溢出兜底**：若某个父分片仍超过 5000 字符，则退化为固定文本分割器（`FixedRecursiveCharacterTextSplitter`）进行补充切分；
4. 去除首尾多余标点（`. 。`）。

---

## 4. 子分片策略：规则切分

子分片使用现有的 `FixedRecursiveCharacterTextSplitter` 进行固定规则切分。

### 4.1 切分参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `subchunk_separator` | `\n` | 子分片分隔符 |
| `subchunk_max_length` | `512` | 子分片最大字符数 |
| `subchunk_chunk_overlap` | `0` | 子分片重叠长度 |

### 4.2 分隔符优先级（Recursive）

子分片按以下优先级递归切分：

```
\n\n  →  。  →  .（空格）  →  （空字符）
```

### 4.3 子分片与父分片的关系

- 每个子分片必须完全属于唯一一个父分片；
- 子分片的内容是父分片的真子集（无跨父分片内容）；
- 检索时命中子分片 → 回溯到对应父分片 → 将父分片作为上下文送入 LLM。

---

## 5. 数据模型

### 5.1 输入参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `model` | model-selector | ✅ | - | 用于父分片切分的 LLM |
| `input_text` | string | ✅ | - | 待切分的原始文本 |
| `prompt` | string | ✅ | - | 父分片切分的系统提示词（可自定义） |
| `max_length` | number | ❌ | 5000 | 父分片最大字符数（硬约束） |
| `subchunk_separator` | string | ❌ | `\n` | 子分片分隔符 |
| `subchunk_max_length` | number | ❌ | 512 | 子分片最大字符数 |
| `remove_extra_spaces` | boolean | ❌ | false | 是否清除多余空格/换行/制表符 |
| `remove_urls_emails` | boolean | ❌ | false | 是否清除 URL 和邮件地址 |

### 5.2 输出数据结构

```python
class ParentChildChunk(BaseModel):
    parent_content: str                   # 父分片文本（≤5000字符）
    child_contents: list[str]             # 子分片文本列表
    parent_mode: Literal["llm"]           # 父分片模式标识

class ParentChildStructureChunk(BaseModel):
    parent_child_chunks: list[ParentChildChunk]
    parent_mode: Literal["llm"] = "llm"
```

### 5.3 输出示例

```json
{
  "parent_mode": "llm",
  "parent_child_chunks": [
    {
      "parent_mode": "llm",
      "parent_content": "人工智能（AI）是计算机科学的一个分支，致力于构建能够执行通常需要人类智能的任务的系统。这些任务包括语音识别、决策制定、视觉感知以及语言翻译等。近年来，随着深度学习技术的突破，AI 在图像识别、自然语言处理等领域取得了革命性进展...",
      "child_contents": [
        "人工智能（AI）是计算机科学的一个分支，致力于构建能够执行通常需要人类智能的任务的系统。",
        "这些任务包括语音识别、决策制定、视觉感知以及语言翻译等。",
        "近年来，随着深度学习技术的突破，AI 在图像识别、自然语言处理等领域取得了革命性进展..."
      ]
    },
    {
      "parent_mode": "llm",
      "parent_content": "机器学习是 AI 的核心子领域，通过算法让系统从数据中自动学习和改进，无需显式编程...",
      "child_contents": [
        "机器学习是 AI 的核心子领域，通过算法让系统从数据中自动学习和改进，无需显式编程..."
      ]
    }
  ]
}
```

---

## 6. 实现流程

### 6.1 主处理流程

```python
def transform(input_text: str, rules: Rule, model_config: dict, prompt: str) -> ParentChildStructureChunk:
    # Step 1: 文本预处理
    cleaned_text = CleanProcessor.clean(input_text, rules=rules)

    # Step 2: 判断是否需要预分割（文本超长）
    # 模型有效输入上下文约 128k，对应约 100,000 字符（含 Prompt 开销）
    LLM_INPUT_THRESHOLD = 100_000
    WINDOW_SIZE = 80_000   # 单窗口大小，留余量给 Prompt
    WINDOW_OVERLAP = 500   # 窗口重叠，保证边界语义连续
    if len(cleaned_text) > LLM_INPUT_THRESHOLD:
        windows = sliding_window_split(cleaned_text, WINDOW_SIZE, overlap=WINDOW_OVERLAP)
    else:
        windows = [cleaned_text]  # 不超过阈值，整体一次提交

    # Step 3: 对每个窗口调用 LLM 切分父分片
    all_parent_chunks = []
    for window in windows:
        parent_texts = llm_split(window, model_config, prompt)
        all_parent_chunks.extend(parent_texts)

    # Step 4: 父分片后处理（溢出兜底、去重、清洗）
    parent_chunks = postprocess_parents(all_parent_chunks, max_length=rules.segmentation.max_tokens)

    # Step 5: 对每个父分片进行子分片切分
    result = ParentChildStructureChunk(parent_child_chunks=[], parent_mode="llm")
    for parent_text in parent_chunks:
        child_texts = fixed_split(parent_text, rules.subchunk_segmentation)
        result.parent_child_chunks.append(
            ParentChildChunk(
                parent_content=parent_text,
                child_contents=child_texts,
                parent_mode="llm"
            )
        )

    return result
```

### 6.2 LLM 调用模块

```python
def llm_split(text: str, model_config: dict, prompt: str) -> list[str]:
    """
    调用 LLM 对文本进行语义切分，返回父分片列表。
    LLM 返回用 <SPLIT> 分隔的文本。
    """
    full_prompt = prompt.replace("{input_text}", text)
    response = invoke_llm(model_config, full_prompt)
    
    # 解析 <SPLIT> 分隔结果
    raw_chunks = response.split("<SPLIT>")
    return [chunk.strip() for chunk in raw_chunks if chunk.strip()]
```

### 6.3 溢出兜底处理

```python
def postprocess_parents(parent_texts: list[str], max_length: int) -> list[str]:
    """
    对超过 max_length 的父分片进行二次固定切分（兜底策略）。
    """
    result = []
    fallback_splitter = FixedRecursiveCharacterTextSplitter.from_encoder(
        chunk_size=max_length,
        chunk_overlap=0,
        fixed_separator="\n\n",
        separators=["\n\n", "。", ". ", " ", ""],
    )
    for text in parent_texts:
        if len(text) <= max_length:
            result.append(clean_page_content(text))
        else:
            # 超长则退化到固定切分
            sub_chunks = fallback_splitter.split_text(text)
            result.extend([clean_page_content(c) for c in sub_chunks if c.strip()])
    return result
```

---

## 7. 与现有方案的对比

| 特性 | paragraph 模式 | full_doc 模式 | **LLM 模式（本方案）** |
|------|---------------|--------------|----------------------|
| 父分片切分方式 | 固定分隔符 + 长度 | 整文档作为父块 | **LLM 语义感知切分** |
| 语义完整性 | 中 | 高（但粒度太粗） | **高** |
| 父分片最大长度 | 可配置 | 无限制 | **5000 字符（可配置）** |
| 适用文档类型 | 结构清晰的文档 | 短文档 | **任意文档（尤其混排文档）** |
| LLM 依赖 | 无 | 无 | **需要 LLM（成本更高）** |
| 处理速度 | 快 | 最快 | 较慢（LLM 调用） |
| 检索上下文质量 | 中 | 高但不精确 | **最高** |

---

## 8. 适用场景

| 场景 | 推荐程度 | 说明 |
|------|---------|------|
| 法律合同、政策文件 | ⭐⭐⭐⭐⭐ | 段落语义边界不固定，LLM 能正确识别条款边界 |
| 学术论文、研究报告 | ⭐⭐⭐⭐⭐ | 章节结构复杂，LLM 能按研究问题切分 |
| 新闻文章、博客 | ⭐⭐⭐⭐ | 叙事逻辑清晰，LLM 切分效果好 |
| 结构化技术文档（Markdown/reStructuredText） | ⭐⭐⭐ | 标题层级已是显式语义边界，**规则切分是首选**；但 LLM 在以下场景有规则无法替代的优势：①单节超过 5000 字符时，LLM 能找到语义断点（如两个示例之间）而非机械截断；②识别并合并 `### Note`/`### Warning` 等语义上从属于上节的细碎小节；③将代码块与其上方说明段绑定在同一父分片；④处理混用 HTML 标签或格式不规范的文档。格式规整的文档优先用 paragraph 模式，存在上述问题时再启用本方案 |
| 短文本（< 1000 字符） | ⭐⭐ | 文本本身已很短，父子结构收益有限 |

---

## 9. 关键约束与限制

1. **父分片硬上限**：无论 LLM 如何切分，父分片最终长度不超过 **5000 字符**（兜底固定切分保证）；
2. **LLM 不可修改原文**：Prompt 中明确要求 LLM 不得修改或摘要文本内容，仅做切分；
3. **子分片不跨父分片**：子分片内容严格来源于单个父分片；
4. **空片过滤**：所有空白片段在输出前被过滤；
5. **Token 成本**：LLM 父切分会产生额外 token 消耗，长文档建议使用较小的快速模型（如 GPT-4o-mini）。

---

## 10. 配置推荐

### 通用长文档
```yaml
max_length: 5000
subchunk_max_length: 512
subchunk_separator: "\n"
remove_extra_spaces: true
remove_urls_emails: false
```

### 法律/合同文档
```yaml
max_length: 3000          # 条款较短，适当减小
subchunk_max_length: 300
subchunk_separator: "\n"
remove_extra_spaces: false  # 保留格式空格
remove_urls_emails: false
```

### 技术手册/API 文档
```yaml
max_length: 4000
subchunk_max_length: 400
subchunk_separator: "\n\n"  # 代码块用双换行分隔
remove_extra_spaces: false
remove_urls_emails: false   # 保留 URL（接口地址重要）
```
