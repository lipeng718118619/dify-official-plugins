# Markdown 结构化文档父子分片方案设计

## 1. 核心思路

Markdown 文档的语义边界已由标题层级（`#` `##` `###`）**显式编码**，父分片切分无需依赖 LLM。
最优方案是：**先用 AST 解析器将文档解析为结构树，再基于节点类型进行标题感知父分片 + 段落级子分片**，将代码块、表格、列表等视为不可拆分的原子节点。

---

## 2. 为什么使用 AST 解析而非正则/文本匹配

纯文本正则方案存在根本性缺陷：

| 缺陷场景 | 正则方案 | AST 方案 |
|---------|---------|---------|
| 代码块内的 `# 注释` | ❌ 误识别为标题 | ✅ 节点类型为 `code_block`，内部内容不参与标题识别 |
| 行内代码 `` `# foo` `` | ❌ 可能干扰标题检测 | ✅ 节点类型为 `inline_code` |
| 多层嵌套列表 | ❌ 缩进规则复杂，易误判 | ✅ 列表节点天然有嵌套层级 |
| 表格内含 `\|` 的单元格 | ❌ 列数计算错误 | ✅ 表格节点已完整解析各列 |
| HTML 注释 `<!-- # 假标题 -->` | ❌ 可能误匹配 | ✅ 被解析为 `html_block`，不参与标题提取 |
| 转义字符 `\#` | ❌ 需额外处理 | ✅ 解析器已处理转义，输出真实文本 |

**推荐解析库：`markdown-it-py`**（CommonMark 规范兼容，纯 Python，性能优秀，节点类型丰富）

```python
from markdown_it import MarkdownIt

md = MarkdownIt()
tokens = md.parse(markdown_text)  # 返回 Token 列表（扁平化 AST）
```

---

## 3. 整体架构

```
原始 Markdown 文档
        │
        ▼
┌──────────────────────────────────────────────┐
│  第一步：AST 解析（markdown-it-py）            │
│  输出 Token 流，每个 Token 含：                │
│    · type: heading/fence/table/paragraph/... │
│    · level: 标题层级（1/2/3）                 │
│    · map: [起始行, 结束行]                     │
│    · content: 节点原始文本                     │
└──────────────────────────────────────────────┘
        │ Token 流
        ▼
┌──────────────────────────────────────────────┐
│  第二步：标题感知父分片（Heading Splitter）      │
│  遍历 Token 流，按 H1 > H2 > H3 边界分组        │
│  每个父分片 ≤ 5000 字符                        │
│  过长时递归向下一级标题切分                      │
│  短小节（< 200 字符）自动合并                   │
└──────────────────────────────────────────────┘
        │ 父分片列表（含标题面包屑 + 节点类型信息）
        ▼
┌──────────────────────────────────────────────┐
│  第三步：段落级子分片（Node-aware Splitter）    │
│  · fence/table/list → 直接作为独立子分片        │
│  · paragraph → 按句子边界切分，上限 512 字符    │
│  · 说明段 + 紧跟的 fence → 绑定为同一子分片     │
└──────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────┐
│  最终输出：纯文本，分隔符编码父子结构            │
│  · 父分片间以 <P_SPLIT> 分隔                   │
│  · 每个父分片内，父内容与各子分片以 <C_SPLIT> 分隔│
│                                              │
│  格式：                                       │
│  父1内容<C_SPLIT>子1<C_SPLIT>子2<P_SPLIT>      │
│  父2内容<C_SPLIT>子1<C_SPLIT>子2               │
└──────────────────────────────────────────────┘
```

---

## 4. AST Token 类型与处理规则

`markdown-it-py` 解析后的关键 Token 类型及对应处理策略：

| Token type | 含义 | 父分片处理 | 子分片处理 |
|-----------|------|-----------|-----------|
| `heading_open` + `inline` | 标题节点（H1/H2/H3） | 触发父分片边界 | 归入父分片头部 |
| `fence` | 代码块（` ``` `） | 原子节点，不切割 | **独立子分片** |
| `table_open...table_close` | 表格 | 原子节点，不切割 | **独立子分片** |
| `bullet_list` / `ordered_list` | 无序/有序列表 | 原子节点，同列表不跨父分片 | **独立子分片** |
| `paragraph_open` + `inline` | 普通段落 | 按长度归入当前父分片 | 按句子切分，≤512 字符 |
| `html_block` | HTML 块 | 原子节点 | **独立子分片** |
| `front_matter` | YAML front matter | 单独作为元数据父分片 | 不参与检索（可配置跳过） |
| `hr` | 分隔线（`---`） | 可选：作为额外的父分片边界 | 忽略 |
| `blockquote` | 引用块 | 原子节点 | **独立子分片** |

---

## 5. 父分片构建算法

### 5.1 核心逻辑

```python
from markdown_it import MarkdownIt
from markdown_it.token import Token

def build_parent_chunks(markdown_text: str, max_length: int = 5000) -> list[ParentChunk]:
    md = MarkdownIt()
    tokens = md.parse(markdown_text)

    # 提取 front matter（若有）
    front_matter = extract_front_matter(tokens)

    # 将 Token 流按标题边界分组为"节"
    sections = split_by_headings(tokens)  # list[Section(heading, tokens, level)]

    # 对每个节递归处理：若超长则按下一级标题继续切分
    parent_chunks = []
    for section in sections:
        chunks = section_to_chunks(section, max_length)
        parent_chunks.extend(chunks)

    return parent_chunks


def build_output(
    markdown_text: str,
    max_length: int = 5000,
    max_child_length: int = 512,
) -> str:
    """将 Markdown 文档切分并序列化为带分隔符的纯文本。

    输出格式：
        父1内容<C_SPLIT>子1<C_SPLIT>子2<P_SPLIT>父2内容<C_SPLIT>子1<C_SPLIT>子2

    下游解析：
        units = output.split("<P_SPLIT>")
        for unit in units:
            parts = unit.split("<C_SPLIT>")
            parent_content  = parts[0]
            child_contents  = parts[1:]
    """
    parent_chunks = build_parent_chunks(markdown_text, max_length)
    units = []
    for chunk in parent_chunks:
        children = build_child_chunks(chunk, max_child_length)   # list[str]
        unit_parts = [chunk.content] + children
        units.append("<C_SPLIT>".join(unit_parts))
    return "<P_SPLIT>".join(units)


def section_to_chunks(section: Section, max_length: int) -> list[ParentChunk]:
    text = tokens_to_text(section.tokens)

    if len(text) <= max_length:
        # 节本身未超长，直接作为一个父分片
        return [ParentChunk(
            content=text,
            breadcrumb=section.breadcrumb,
            heading_level=section.level,
        )]
    else:
        # 超长：尝试按下一级标题切分
        sub_sections = split_by_headings(section.tokens, min_level=section.level + 1)
        if len(sub_sections) > 1:
            result = []
            for sub in sub_sections:
                result.extend(section_to_chunks(sub, max_length))
            return result
        else:
            # 无下一级标题：按段落边界切分（保护原子块）
            return split_by_paragraphs(section, max_length)
```

### 5.2 标题面包屑追踪

```python
class BreadcrumbTracker:
    """维护当前解析位置的完整标题路径。"""
    def __init__(self):
        self._crumbs: dict[int, str] = {}  # {level: heading_text}

    def update(self, level: int, text: str):
        self._crumbs[level] = text
        # 清除所有更深层级的历史标题
        for l in list(self._crumbs):
            if l > level:
                del self._crumbs[l]

    @property
    def path(self) -> str:
        return " > ".join(
            self._crumbs[l] for l in sorted(self._crumbs)
        )
        # 示例输出："# 产品手册 > ## 安装指南 > ### Linux 安装"
```

### 5.3 短小节合并

```python
def merge_short_sections(sections: list[Section], merge_threshold: int = 200) -> list[Section]:
    """将相邻的短小同级节合并，避免产生过小的父分片。"""
    merged = []
    buffer = None
    for sec in sections:
        if buffer is None:
            buffer = sec
        elif (sec.level == buffer.level
              and len(tokens_to_text(buffer.tokens)) < merge_threshold
              and len(tokens_to_text(sec.tokens)) < merge_threshold
              and len(tokens_to_text(buffer.tokens + sec.tokens)) <= MAX_PARENT_LENGTH):
            buffer = buffer.merge(sec)
        else:
            merged.append(buffer)
            buffer = sec
    if buffer:
        merged.append(buffer)
    return merged
```

---

## 6. 子分片构建算法

```python
def build_child_chunks(parent: ParentChunk, max_child_length: int = 512) -> list[str]:
    children = []
    pending_paragraph = None  # 用于"说明段 + 代码块"绑定

    for node in parent.nodes:  # nodes 为 AST 节点列表
        if node.type in ("fence", "table", "bullet_list", "ordered_list",
                         "html_block", "blockquote"):
            # 原子节点：若有待绑定的说明段，与其合并
            if pending_paragraph and len(pending_paragraph + node.text) <= max_child_length:
                children.append(with_breadcrumb(pending_paragraph + "\n\n" + node.text, parent))
                pending_paragraph = None
            else:
                if pending_paragraph:
                    children.append(with_breadcrumb(pending_paragraph, parent))
                    pending_paragraph = None
                children.append(with_breadcrumb(node.text, parent))  # 原子块不限长度

        elif node.type == "paragraph":
            if pending_paragraph:
                children.append(with_breadcrumb(pending_paragraph, parent))
            # 段落可能需要按句子进一步切分
            if len(node.text) <= max_child_length:
                pending_paragraph = node.text  # 暂存，等待可能的后续代码块绑定
            else:
                for sentence_chunk in split_by_sentences(node.text, max_child_length):
                    children.append(with_breadcrumb(sentence_chunk, parent))
                pending_paragraph = None

    if pending_paragraph:
        children.append(with_breadcrumb(pending_paragraph, parent))

    return children


def with_breadcrumb(text: str, parent: ParentChunk) -> str:
    """向量化时在子分片前拼接标题面包屑，提升检索相关性。"""
    return f"[{parent.breadcrumb}] {text}"
```

---

## 7. 完整切分示例

### 7.1 基础示例（两级标题，含代码块）

**输入：**
```markdown
# 产品手册

## 安装指南

### Linux 安装

运行以下命令安装依赖：

```bash
# 这里的 # 是 shell 注释，不是 Markdown 标题
apt-get install python3
```

安装完成后验证版本。

### Windows 安装

下载安装包后双击运行。
```

**正则方案的问题：** `# 这里的 # 是 shell 注释` 会被误识别为 H1 标题，导致父分片边界错误。

**AST 方案的输出（纯文本，使用分隔符）：**

```
### Linux 安装

运行以下命令安装依赖：

```bash
# 这里的 # 是 shell 注释，不是 Markdown 标题
apt-get install python3
```

安装完成后验证版本。
<C_SPLIT>
[# 产品手册 > ## 安装指南 > ### Linux 安装] 运行以下命令安装依赖：
<C_SPLIT>
[# 产品手册 > ## 安装指南 > ### Linux 安装] ```bash
# 这里的 # 是 shell 注释
apt-get install python3
```
<C_SPLIT>
[# 产品手册 > ## 安装指南 > ### Linux 安装] 安装完成后验证版本。
<P_SPLIT>
### Windows 安装

下载安装包后双击运行。
<C_SPLIT>
[# 产品手册 > ## 安装指南 > ### Windows 安装] 下载安装包后双击运行。
```

**下游解析：**
```python
units = output.split("<P_SPLIT>")
for unit in units:
    parts = unit.split("<C_SPLIT>")
    parent_content = parts[0].strip()   # 第一段：完整父分片文本
    child_contents = [p.strip() for p in parts[1:]]  # 后续：各子分片
```

---

### 7.2 四级标题 + 多级超长的完整推演

**文档结构（H1/H2/H3 内容块均超过 5000 字符，H4 未超长）：**

```
# 产品手册                      ← 超长（含全部子内容）
  引言段落...（500字）
  ## 架构设计                   ← 超长（含全部子内容）
    架构概述...（800字）
    ### 核心模块                ← 超长（含全部子内容）
      模块说明...（600字）
      #### 模块A                ← 正常（2000字）
      #### 模块B                ← 正常（1800字）
    ### 扩展机制                ← 正常（1200字）
  ## 部署指南                   ← 正常（3000字）
```

**递归切分过程（`section_to_chunks` 调用链）：**

```
第1层：整文档按 H1 边界切分
└── "# 产品手册" 节 → 超长
        │ 尝试按 H2（min_level=2）切分
        ▼
第2层：H1 节内按 H2 边界切分，得到：
├── [前置内容] "引言段落..." (500字，无子标题)
│     → ≤5000，直接成父分片 ✅
│     breadcrumb: "# 产品手册"
│
├── "## 架构设计" 节 → 超长
│       │ 尝试按 H3（min_level=3）切分
│       ▼
│   第3层：H2 节内按 H3 边界切分，得到：
│   ├── [前置内容] "架构概述..." (800字，无子标题)
│   │     → ≤5000，直接成父分片 ✅
│   │     breadcrumb: "# 产品手册 > ## 架构设计"
│   │
│   ├── "### 核心模块" 节 → 超长
│   │       │ 尝试按 H4（min_level=4）切分
│   │       ▼
│   │   第4层：H3 节内按 H4 边界切分，得到：
│   │   ├── [前置内容] "模块说明..." (600字，无子标题)
│   │   │     → ≤5000，直接成父分片 ✅
│   │   │     breadcrumb: "# 产品手册 > ## 架构设计 > ### 核心模块"
│   │   │
│   │   ├── "#### 模块A" 节 (2000字) → ≤5000 ✅
│   │   │     breadcrumb: "# 产品手册 > ## 架构设计 > ### 核心模块 > #### 模块A"
│   │   │
│   │   └── "#### 模块B" 节 (1800字) → ≤5000 ✅
│   │         breadcrumb: "# 产品手册 > ## 架构设计 > ### 核心模块 > #### 模块B"
│   │
│   └── "### 扩展机制" 节 (1200字) → ≤5000 ✅
│         breadcrumb: "# 产品手册 > ## 架构设计 > ### 扩展机制"
│
└── "## 部署指南" 节 (3000字) → ≤5000 ✅
      breadcrumb: "# 产品手册 > ## 部署指南"
```

**最终产生的父分片列表（共 6 个）：**

| 序号 | 父分片来源 | breadcrumb | 大小 |
|------|-----------|-----------|------|
| 1 | H1 前置引言段 | `# 产品手册` | 500字 |
| 2 | H2 前置架构概述 | `# 产品手册 > ## 架构设计` | 800字 |
| 3 | H3 前置模块说明 | `# 产品手册 > ## 架构设计 > ### 核心模块` | 600字 |
| 4 | #### 模块A 整节 | `# 产品手册 > ## 架构设计 > ### 核心模块 > #### 模块A` | 2000字 |
| 5 | #### 模块B 整节 | `# 产品手册 > ## 架构设计 > ### 核心模块 > #### 模块B` | 1800字 |
| 6 | ### 扩展机制 整节 | `# 产品手册 > ## 架构设计 > ### 扩展机制` | 1200字 |
| 7 | ## 部署指南 整节 | `# 产品手册 > ## 部署指南` | 3000字 |

**关键结论：**
- H1/H2/H3 节超长时，**不会**被强行截断，而是递归向下寻找子标题作为自然边界
- 每个标题节内"子标题之前的前置段落"（如引言、概述）会被**单独切出**成一个父分片，继承父级 breadcrumb
- 算法最多支持到 H6，H4 之后若仍超长，自动退化到段落→句子→固定截断的兜底链路
- 所有父分片的 breadcrumb 保留完整的层级路径，子分片检索命中后可准确还原文档位置

---

## 8. 输出格式规范

工具输出为**纯文本字符串**，使用两种分隔符编码父子层级结构：

| 分隔符 | 作用 | 位置 |
|--------|------|------|
| `<P_SPLIT>` | 父分片边界 | 相邻父分片之间 |
| `<C_SPLIT>` | 父/子分片边界 | 父内容与第一个子分片之间，以及各子分片之间 |

**完整格式：**

```
父分片1内容<C_SPLIT>子分片1-1<C_SPLIT>子分片1-2<C_SPLIT>子分片1-N
<P_SPLIT>
父分片2内容<C_SPLIT>子分片2-1<C_SPLIT>子分片2-2
<P_SPLIT>
父分片3内容<C_SPLIT>子分片3-1
```

**解析规则：**
1. 先按 `<P_SPLIT>` 切分 → 得到每个父分片的"完整单元"
2. 每个单元再按 `<C_SPLIT>` 切分 → `parts[0]` 为父内容，`parts[1:]` 为该父分片的所有子分片

```python
units = result.split("<P_SPLIT>")
for unit in units:
    parts = [p.strip() for p in unit.split("<C_SPLIT>")]
    parent_content = parts[0]
    child_contents = parts[1:]
```

**注意事项：**
- 父内容本身即完整的 Markdown 原文片段（含标题、代码块、表格等，格式保留）
- 子分片文本前缀携带标题面包屑，格式为 `[# H1 > ## H2] 内容`
- `<P_SPLIT>` 和 `<C_SPLIT>` 为保留字符串，文档内容不应出现这两个字符串（如有冲突需转义）

---

## 9. 解析库选型

| 库 | CommonMark 兼容 | AST 质量 | Python | 推荐度 |
|----|----------------|---------|--------|-------|
| `markdown-it-py` | ✅ 完整支持 | Token 流 + 节点树，类型齐全 | 纯 Python | ⭐⭐⭐⭐⭐ |
| `mistletoe` | ✅ 支持 | 标准 AST，可扩展 | 纯 Python | ⭐⭐⭐⭐ |
| `commonmark` | ✅ 严格标准 | AST 完整 | 纯 Python | ⭐⭐⭐ |
| 正则/文本匹配 | ❌ 存在误判 | 无 AST | - | ⭐ |

**选用 `markdown-it-py`**，理由：
1. CommonMark 规范严格兼容，避免边缘案例
2. Token 类型最丰富（含 front matter 插件、footnote 插件等）
3. 社区活跃，是 `mdit-py-plugins` 生态的基础
4. Dify 生态中已有使用，依赖不新增

安装：
```bash
pip install markdown-it-py
```

---

## 10. 与其他方案的对比总结

| 维度 | 正则文本匹配 | AST 解析（本方案） | LLM 切分 |
|------|------------|-----------------|---------|
| 代码块内 `#` 误判 | ❌ 存在 | ✅ 不存在 | ✅ 不存在 |
| 嵌套结构解析 | ❌ 脆弱 | ✅ 精确 | ✅ 理解 |
| 处理速度 | 最快 | 快（毫秒级） | 慢（秒级） |
| Token 消耗 | 0 | 0 | 高 |
| 切分确定性 | 中（有误判） | 高（完全确定） | 低（模型相关） |
| 格式不规范文档 | 差 | 中（依赖解析器鲁棒性） | 最好 |
| 节内语义断点 | 机械 | 机械 | 语义感知 |

**结论：AST 解析是 Markdown 文档切分的最优基础方案**，相比正则方案消除了所有语法歧义，相比 LLM 方案零成本且结果可重现。仅在文档质量极差（大量自定义语法、格式混乱）时考虑引入 LLM 作为补充。
