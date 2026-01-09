# 接口
## 模块一 动态结构树的搭建
## 接口一：materialize_structure_tree
### 阶段接口一：extract_text_blocks
**输入：**
content_image_paths: string[]
**说明:**
- 图片数量不限制
- 顺序即目录页顺序（这一点很重要，后面会用）
**输出：**
''' 
TextBlock {
    block_id: string                // 稳定唯一ID，后续所有阶段都用它
    text: string                    // 原始OCR文本，不做任何清洗
    bbox?: BoundingBox              // 可选，允许无坐标OCR
    source_image_id: string         // 对应哪一页目录图
    source_image_index: int         // 第几张目录图（0-based）
    line_index?: int                // 在该图中的大致行序（可选但强烈建议）
}
''' 
**保证：**
- 只做 OCR，不做语义、不猜标题
- 多张目录图全部处理，不因失败中断整体
- 所有输出文本都能追溯到原图
**不保证：**
- 文本是否完整
- 是否是一行标题
- 是否属于同一章节
- OCR 是否准确
**失败处理：**
- 单张图片失败 → 跳过该图
- 全部失败 → 返回 []

### 阶段接口二：detect_title_candidates
**输入：**
TextBlock {
    block_id: string                // 稳定唯一ID，后续所有阶段都用它
    text: string                    // 原始OCR文本，不做任何清洗
    bbox?: BoundingBox              // 可选，允许无坐标OCR
    source_image_id: string         // 对应哪一页目录图
    source_image_index: int         // 第几张目录图（0-based）
    line_index?: int                // 在该图中的大致行序（可选但强烈建议）
}
**输出：**
TitleCandidate {
    candidate_id: string            // 新ID，不要复用 block_id
    source_block_id: string         // 来自哪个 TextBlock
    text: string                    // 原文，允许和 TextBlock.text 一致
    level_guess?: int | null        // 章节层级猜测（1,2,3...）
    confidence?: float              // 0~1，允许不存在
    cues?: string[]                 // 判断依据，如 "编号前缀", "字体推测", "对齐"
}
**保证：**
- 每一个候选 必须来源于输入 TextBlock
- 不篡改原文，不拼接，不合并
**不保证：**
- 一定是真标题
- 不漏标题
- 层级一定正确
**失败处理：**
- 构建失败时，返回空 TitleCandidate 列表

### 阶段接口三：build_structure_tree
**输入：**
- TitleCandidate {
    candidate_id: string            // 新ID，不要复用 block_id
    source_block_id: string         // 来自哪个 TextBlock
    text: string                    // 原文，允许和 TextBlock.text 一致
    level_guess?: int | null        // 章节层级猜测（1,2,3...）
    confidence?: float              // 0~1，允许不存在
    cues?: string[]                 // 判断依据，如 "编号前缀", "字体推测", "对齐"
}
**输出：**
- ChapterNode {
    node_id: string
    title_text: string
    level: int                     // 最终确定的层级
    order_index: int               // 在全目录中的顺序
    source_candidate_ids: string[] // 可能多个候选合并而来
    parent_id?: string | null
    children?: ChapterNode[]
}
**保证：**
- 一定有返回值（哪怕是 []）
- 输出是合法树结构
- level 从 1 开始，连续或不连续都允许
**不保证：**
- 树一定是“教材真实结构”
- 不会出现层级压缩或膨胀
**失败处理：**
- 无候选 → 返回 []
- 构建异常 → 返回 [] + 日志记录

## 接口二：materialize_dynamic_structure_tree
### 阶段接口一：extract_page_blocks
**输入：**
- pdf_path
**输出：**
PageBlock {
    page_index: int
    text_blocks: ContentBlock[]
}
ContentBlock {
    block_id: string
    type: "text" | "formula" | "image" | "table"
    content: string              // text 原文 / latex / image_path / table_repr
    bbox?: BoundingBox
    order_index: int             // 在页面内的物理顺序
}
**保证：**
- 每个 ContentBlock 严格来自单一页面
- order_index 单调递增
- 不跨页、不合并
**不保证：**
- block 类型一定正确
- 表格、公式解析一定可用
**失败处理：**
- 单页失败 → 跳过该页
- 全部失败 → 返回 []

### 阶段接口二：detect_context_candidates
**输入：**
- PageBlock[]
  - 每页的 TextBlock 列表
- ChapterNode[] // 来自接口一，目录结构
**输出：**
- ContextCandidate {
    candidate_id: string
    block_id: string
    page_index: int
    type: ContentBlock["type"]
    content: string

    chapter_scores: {
        chapter_id: string
        score: float
        cues?: string[]
    }[]
    unresolved?: boolean          // 明确标记：我真的不知道
}
**保证：**
- chapter_id 严格来自 ChapterNode
- 不擅自生成新章节
**不保证：**
- score一定准确（score 仅用于相对排序，不具备概率意义）
- top-1 一定正确
**失败处理：**
- 构建失败时，返回空 ContextCandidate
- 单 block 无法判断 → unresolved = true，而不是丢弃

### 阶段接口三：classify_context_by_chapter
**输入：**
- ContextCandidate[] 
**输出：**
- ChapterContext {
    chapter_id: string
    page_start: int
    page_end: int

    blocks: {
        text: BlockRef[]
        formula: BlockRef[]
        image: BlockRef[]
        table: BlockRef[]
    }
}
BlockRef {
    block_id: string
    content: string
    page_index: int
}
**保证：**
- 以chapter_id为主键归类
- page_start ≤ page_end
- block 不重复归属到同一 chapter 的多个类型中
**不保证：**
- 文本、公式、图像、表格分离一定准确
- 页码边界精确
**失败处理：**
- 构建失败时，返回空

### 阶段接口四：merge_structure_evidence
**输入：**
- ChapterContext[] 
- ChapterNode[]
- ContextCandidate[]  
**输出：**
MergedNode {
    merged_id: string
    chapter_id: string
    title_text: string
    level: int
    page_start: int
    page_end: int
    context_block_ids: string[]
    confidence: float
    conflicts?: {
        type: string
        detail: string
    }[]
}
**保证：**
- page_start ≤ page_end
- chapter_id 唯一
**不保证：**
- 章节边界完全正确
- 冲突一定可解
**失败处理：**
- 构建失败时，返回空

### 阶段接口五：materialize_structure_tree
**输入：**
- materialize_structure_tree(
  merged_nodes: MergedNode[],
  options?: {
    strategy?: "auto" | "conservative" | "aggressive"
    min_confidence?: number
    allow_manual_override?: boolean
  }
)
**参数解释**
merged_nodes
    来自 merge_structure_evidence
    允许存在冲突、重叠、低置信度
strategy
    "auto"（默认）：平衡覆盖率与准确率
    "conservative"：宁缺毋滥，低置信度节点不入树
    "aggressive"：尽量保留，适合探索阶段
min_confidence
    默认例如 0.6
    小于该值的节点不会进入最终树
    allow_manual_override
    是否允许后续人工调整（只是标记，不是实现）
**输出:**
- MaterializedChapterNode {
  id: string                     // 稳定、可引用
  title_text: string
  level: int
  page_start: int
  page_end: int
  parent_id?: string
  children_ids?: string[]
  context_block_ids: string[]    // 正文锚点
  confidence: float              // 冻结后的综合置信度
  provenance: {
    source_titles: string[]      // 来自哪些目录/标题证据
    source_pages: number[]       // 来自哪些页面
  }
}
- MaterializedStructureTree {
  version_id: string
  generated_at: timestamp
  strategy: string
  chapters: MaterializedChapterNode[]
  root_ids: string[]
  unresolved_nodes?: {
    id: string
    reason: string
  }[]
}
**保证:**
    所有 id 在当前版本内唯一且稳定
    page_start ≤ page_end
    树结构无环
    所有 parent_id 必然存在于同一输出中
    输出结果可直接用于笔记挂载与可视化
**不保证:**
    章节层级完全符合教材原意
    页码边界完全精确
    所有目录标题都被成功实体化
    confidence 具备概率意义
**失败处理：**
    返回空 chapters
    填充 unresolved_nodes
    保留 version_id 以便排错
