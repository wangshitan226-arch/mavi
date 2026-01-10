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


    TextBlock {
        block_id: string                // 稳定唯一ID，后续所有阶段都用它
        text: string                    // 原始OCR文本，不做任何清洗
        bbox?: BoundingBox              // 可选，允许无坐标OCR
        source_image_id: string         // 对应哪一页目录图
        source_image_index: int         // 第几张目录图（0-based）
        line_index?: int                // 在该图中的大致行序（可选但强烈建议）
    }


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


---
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


---
### 阶段接口三：build_structure_tree

**输入：**

    TitleCandidate {
        candidate_id: string            // 新ID，不要复用 block_id
        source_block_id: string         // 来自哪个 TextBlock
        text: string                    // 原文，允许和 TextBlock.text 一致
        level_guess?: int | null        // 章节层级猜测（1,2,3...）
        confidence?: float              // 0~1，允许不存在
        cues?: string[]                 // 判断依据，如 "编号前缀", "字体推测", "对齐"
    }

**输出：**

    ChapterNode {
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

pdf_path

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


---
### 阶段接口二：detect_context_candidates

**输入：**

- PageBlock[]

  - 每页的 TextBlock 列表

- ChapterNode[] // 来自接口一，目录结构

**输出：**

    ContextCandidate {
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
        unresolved?: boolean
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


---
### 阶段接口三：classify_context_by_chapter

**输入：**

ContextCandidate[] 

**输出：**

    ChapterContext {
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


---
### 阶段接口四：merge_structure_evidence

**输入：**

ChapterContext[] 

ChapterNode[]

ContextCandidate[]  

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


---
### 阶段接口五：materialize_structure_tree

**输入：**

    materialize_structure_tree(
        merged_nodes: MergedNode[],
        options?: {
            strategy?: "auto" | "conservative" | "aggressive"
            min_confidence?: number
            allow_manual_override?: boolean
        }
    )

**输出:**

    MaterializedChapterNode {
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

    MaterializedStructureTree {
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

- 所有 id 在当前版本内唯一且稳定

- page_start ≤ page_end

- 树结构无环

- 所有 parent_id 必然存在于同一输出中

- 输出结果可直接用于笔记挂载与可视化

**不保证:**

- 章节层级完全符合教材原意

- 页码边界完全精确

- 所有目录标题都被成功实体化

- confidence 具备概率意义

**失败处理：**

- 返回空 chapters

- 填充 unresolved_nodes

- 保留 version_id 以便排错

---

## 模块二 教材内容语义单元化

## 接口： SemanticStructureTree

**输入：**

    MaterializedChapterNode {
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

    MaterializedStructureTree {
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

**输出：**

    SemanticStructureTree {
        version_id: string
        generated_at: timestamp
        derived_from_version: string     // 指向输入 structure version
        chapters: SemanticChapterNode[]
        root_ids: string[]
        unresolved_nodes?: {
            id: string
            reason: string
        }[]
    }

    SemanticChapterNode {
        id: string                      // 与结构节点一致，保证可追溯
        title_text: string
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]

        object_ids: string[]            // 语义对象锚点（替代 block）
        confidence: float

        provenance: {
            source_titles: string[]
            source_pages: number[]
            source_block_ids: string[]  // 明确来自哪些 block
        }
    }

    ContentObject {
        id: string
        chapter_id: string              // 出生即绑定章节，不可更改

        block_ids: string[]             // 构成该对象的最小单元
        type: "knowledge_text" | "example" | "exercise"

        content: string                 // 已合并、顺序整理后的内容
        bbox?: BoundingBox[]             // 可选，按 block 聚合
        order_index: number              // 在章节内的物理顺序

        confidence: float
    }

    ObjectCandidate {
        id: string
        block_ids: string[]              // 单 block 或弱聚合
        page_index: number

        predicted_type?: ContentObject["type"]

        assignment_trace: {
            chapter_id: string
            score: number
            cues?: string[]
        }[]

        resolved_object_id?: string     // 若被吸收进某 ContentObject
        unresolved: boolean             // 未能稳定归并
    }


**保证：**

- 输出的树结构不变

- 每个对象可能包含多个类型的block_id，但是一个对象的组成单元一定来自同一个最小标题分组下

**不保证：**

- 对象里的组成（图片、表格、题目）一定是搭配的

**失败处理**

- 返回空 chaptersxia

- 填充 unresolved_nodes

- 保留 version_id 以便排错

---

## 模块三 题目理解与知识点映射

## 接口一：QuestionAnalysis

**输入：**

question_image_path:string

**输出：**

    QuestionAnalysis {
        question_id: string
        question_text: string

        analysis_units: {
            inferred_chapter_level: int        // 预期匹配的章节层级（如 1 / 2）
            semantic_cues: string[]             // 关键词、公式、概念线索
            reasoning_steps?: string[]          // 可选，给人看的推理
        }[]
    }


**保证**

- 为应对综合类数学题，question_analysis最小单元要以章节树一级标题为细粒度

- 只分析，不匹配

**不保证**

- 分析结果一定准确

**失败返回**

- 空QuestionAnalysis

## 接口二：KnowledgeMatchCandidate

**输入：**

QuestionAnalysis

SemanticStructureTree

**输出：**

    KnowledgeMatchCandidate {
        question_id: string

        candidates: {
            chapter_id: string
            score: float
            cues: string[]          // 命中依据：关键词、公式、语义线索
            conflict_notes?: string[] // 为什么可能不匹配
        }[]

        guidance: {
            summary: string          // 给用户的一句话说明
            suggestion?: string      // 是否建议新建章节
        }
    }


**保证**

- 只匹配，不确认

- 当候选为空时，保证匹配说明中有明确解释以及匹配建议（供用户自定义新开一节）

**不保证**

- 匹配结果一定准确

**失败返回**

- 空KnowledgeMatchCandidate

## 接口三：KnowledgeMatchResult

**输入：**

KnowledgeMatchCandidate

SemanticStructureTree

user_choice

**输出：**

    KnowledgeMatchStructureTree{
        version_id: string
        generated_at: timestamp
        derived_from_version: string     // 指向输入 structure version
        chapters: SemanticChapterNode[]
        root_ids: string[]
        unresolved_nodes?: {
            id: string
            reason: string
        }[]
    }

    SemanticChapterNode {
        id: string                      // 与结构节点一致，保证可追溯
        title_text: string
        question_ids?:string[]               //挂载在node层级下
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]

        object_ids: string[]            // 语义对象锚点（替代 block）
        confidence: float

        provenance: {
            source_titles: string[]
            source_pages: number[]
            source_block_ids: string[]  // 明确来自哪些 block
        }
    }

**保证**

- 最终挂载位置由用户确定

- 输出的树结构不变，只做挂载

**不保证**

- 挂载位置一定存在

**失败返回**

- 原样SemanticStructureTree

---

## 模块四 解题与认知价值提取

## 接口一：SolveQuestion

**输入：**

QuestionAnalysis        //目的是获得question_text，减少OCR频率

**输出：**

    SolveQuestion {
        question_id: string

        steps: {
            step_id: string
            content: string          // 本步骤做了什么
            formulas?: string[]      // 用到的公式
            reasoning_type?: "algebra" | "geometry" | "logic" | "transformation"
            order_index: number
        }[]

        final_answer: string
    }

**保证**

- 只解题，不做价值提取

- 答案流程规范

**不保证**

- unknow

**失败处理**

- 输出空

## 接口二：ValueExtration

**输入：**

SolveQuestion        //获取answer

**输出：**

    StepValueAnnotation {
        question_id: string

        step_annotations: {
            step_id: string
            is_valuable: boolean
            difficulty_note?: string
            confidence: float
            value_type?: "common_trick" | "key_transformation" | "error_prone"
        }[]
    }


**保证**

- 难点说明要对应于步骤，方便step作分割资产记录

- 加一些可信度

**不保证**

- 可信度一定准确

**失败处理**

- 输出空

## 接口三 StepsMatchCandidate

**输入：**

ValueExtration

**输出：**

    StepsMatchCandidate {
        question_id: string
        steps_id:string
        candidates: {
            chapter_id: string
            score: float
            cues: string[]          // 命中依据：关键词、公式、语义线索
            conflict_notes?: string[] // 为什么可能不匹配
            relation_type: "belongs_to" | "applies_concept"
        }[]

        guidance: {
            summary: string          // 给用户的一句话说明
            suggestion?: string      // 是否建议新建章节
        }
    }

**保证**

- 只匹配，不确认

**不保证**

- 匹配一定准确

**失败处理**

- 输出空


## 接口四： StepsMatchStructureTree

**输入：**

StepsMatchCandidate

KnowledgeMatchStructureTree

user_choice

**输出：**

    StepsMatchStructureTree{
        version_id: string
        generated_at: timestamp
        derived_from_version: string     // 指向输入 structure version
        chapters: SemanticChapterNode[]
        root_ids: string[]
        unresolved_nodes?: {
            id: string
            reason: string
        }[]
    }

    SemanticChapterNode {
        id: string                      // 与结构节点一致，保证可追溯
        title_text: string
        question_ids?:string[]
        attached_steps?: {
            step_id: string
            source_question_id: string
            confidence: float
        }[]
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]

        object_ids: string[]            // 语义对象锚点（替代 block）
        confidence: float

        provenance: {
            source_titles: string[]
            source_pages: number[]
            source_block_ids: string[]  // 明确来自哪些 block
        }
    }

**保证**

- 最终挂载位置由用户确定

- 输出的树结构不变，只做挂载

**不保证**

- 挂载位置一定存在

**失败返回**

- 原样KnowledgeMatchStructureTree