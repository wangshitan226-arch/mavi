# 接口

## 模块一 动态结构树的搭建

### 接口一：materialize_structure_tree

#### 阶段接口一：extract_text_blocks

> 将目录图片中的标题经过OCR提取出来，用于结构树骨架的构建

> 经检验，使用Paddle模型识别中文文本准确率较高

**输入：**

content_image_paths: string[]          // 目录图片路径

**说明:**

- 图片数量不限制

- 顺序即目录页顺序（这一点很重要，后面会用）

**输出：**

    TextBlock {
        block_id: string                // 稳定唯一ID，后续所有阶段都用它
        text: string                    // 原始OCR文本，不做任何清洗
        bbox?: BoundingBox              // 坐标OCR
        source_image_id: string         // 对应哪一页目录图
        source_image_index: int         // 第几张目录图 主要用于过程优化追责
    }

**保证：**

- 以行为最小单元作识别

- 只做OCR，不做语义、不猜标题

- 多张目录图全部处理，不因失败中断整体

- 所有输出文本都能追溯到原图

**不保证：**

- 文本是否完整

- OCR 是否准确

**失败处理：**

- 单张图片失败 ： 跳过该图

- 全部失败 ： 返回 []

#### 阶段接口二：build_structure_tree

> 第一个接口输出的文本块也许会有些问题

> 比起强化规则，使用通用LLM直接洗出node结构得到的结果更容易让人满意

**输入：**

TextBlock {}

**输出：**

    ChapterNode {
        node_id: string                //自身id
        title_text: string             //标题文本
        level: int                     // 最终确定的层级
        order_index: int               // 在全目录中的顺序
        parent_id?: string | null      //父节点id
        children?: ChapterNode[]       //嵌套子节点
    }

**保证：**

- 一定有返回值

- 输出是合法树结构

- level 从 1 开始，连续或不连续都允许

**不保证：**

- 树一定是“教材真实结构”

- 不会出现层级压缩或膨胀

**失败处理：**

- 失败返回空

### 接口二：materialize_dynamic_structure_tree

#### 阶段接口一：extract_page_blocks

> 真实的全面抽取教材内容，以段落为单位识别更合适

> 实测MinerU模型识别效果较好

**输入：**

pdf_path              // 教材PDF路径

**输出：**

    PageBlock {
        page_index: int             //以页码为单位
        text_blocks: ContentBlock[]
    }
    ContentBlock {
        block_id: string             //识别块的id
        type: "text" | "formula" | "image" | "table"
        content: string              // text 原文 / latex / image_path / table_repr
        bbox?: BoundingBox           //坐标
        order_index: int             // 在页面内的物理顺序
    }

**保证：**

- 以段落为最小单元作识别

- 每个 ContentBlock 严格来自单一页面

- order_index 单调递增

- 不跨页、不合并

**不保证：**

- 表格、公式解析一定可用

**失败处理：**

- 单页失败 ： 跳过该页

- 全部失败 ： 返回 []

#### 阶段接口二：detect_context_candidates

> 内部接口，输出数据不展示给用户，主要用于识别块的归属优化

**输入：**

- PageBlock

- ChapterNode // 来自接口一，目录结构，用于获取chapter

**输出：**

    ContextCandidate {
        candidate_id: string               //候选id
        block_id: string                   //识别块id
        page_index: int
        type: ContentBlock["type"]
        content: string
        chapter_scores: {
            chapter_id: string             //归属章节标题id
            score: float                   //类似置信度，用于比较候选块所属此标题的可能性大小
            cues?: string[]                //其他提示说明
        }[]                                //一个候选识别块可以对应多个所属标题
        unresolved?: boolean               //合理性判断
    }

**保证：**

- chapter_id 严格来自 ChapterNode

- 不擅自生成新章节

**不保证：**

- score一定准确（score 仅用于相对排序，不具备概率意义）

**失败处理：**

- 构建失败时，返回空 ContextCandidate

- 单 block 无法判断 → unresolved = true，而不是丢弃

#### 阶段接口三：merge_structure_evidence

> 融合结构证据，获得融合节点，可以说是候选到最终结构树之间的中间态，此时已经从候选中选择唯一，但主要还是起解释作用

> 上一个接口是遍历识别块，找每个识别块可能的标题，这里类似于一个表格转置，遍历结构树标题，输出该标题下的所有识别块

**输入：**

ChapterNode[]

ContextCandidate[]  

**输出：**

    MergedNode {
        merged_id: string                  //融合节点的id
        chapter_id: string                  //归属章节的id
        title_text: string                  //标题文本
        level: int                         //标题等级
        page_start: int                    //开始页
        page_end: int                      //结束页
        context_block_ids: string[]        //对应该标题下的所有识别块id
        confidence: float                  //置信度
        conflicts?: {                      //可能的冲突，用于解释失败的原因
            type: string                   //冲突类型，想要极致优化可以细化分类
            detail: string                 //细节说明
        }[]                                //冲突可能不止一个，因为这里是识别块列表
    }

**保证：**

- page_start ≤ page_end

- chapter_id 唯一

**不保证：**

- 章节边界完全正确

- 冲突一定可解

**失败处理：**

- 构建失败时，返回空

#### 阶段接口四：materialize_structure_tree

> 可以很明显的看出阶段接口三的输出的节点是没有明显上下层级关系的，我们这里使用chapter_id的层级关系来对应节点的层级关系，返回最终动态结构树

**输入：**

MergedNode

**输出:**

    MaterializedChapterNode {          //最终章节节点
        id: string                     // 稳定、可引用
        title_text: string
        level: int
        page_start: int
        page_end: int
        parent_id?: string              //从chapter_id中获取的上级节点id
        children_ids?: string[]         //从chapter_id中获取的上级节点id
        context_block_ids: string[]    // 正文锚点
        confidence: float              // 冻结后的综合置信度
    }

    MaterializedStructureTree {
        version_id: string            //版本id
        generated_at: timestamp        //树生成时间
        chapters: MaterializedChapterNode[]
        root_ids: string[]             //根节点id，可能不止一个
    }

**保证:**

- 所有 id 在当前版本内唯一且稳定

- page_start ≤ page_end

- 树结构无环

- 所有 parent_id 必然存在于同一输出中

**不保证:**

- 章节层级完全符合教材原意

- 页码边界完全精确

- 所有目录标题都被成功实体化

- confidence 具备概率意义

**失败处理：**

- 返回空 chapters

- 保留 version_id 以便排错

---

## 模块二 教材内容语义单元化

### 接口： SemanticStructureTree

> 很明显，我们要做的识别教材并不是只要独立的文本块、图片、表格、公式这些

> 真正的数学教辅资料的独立单元应该是知识点、例题、习题，各类识别块是这些单元的组成部分

**输入：**

MaterializedStructureTree

**输出：**

    SemanticStructureTree {              //语义单元化后的结构树
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

    SemanticChapterNode {               //语义单元化后的节点
        id: string                      // 保证可追溯
        title_text: string
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]
        object_ids: string[]            // 语义对象锚点（替代 block）
        confidence: float
    }

    ContentObject {
        id: string                      //object的id
        chapter_id: string              // 出生即绑定章节，不可更改
        block_ids: string[]             // 构成该对象的最小单元
        type: "knowledge_text" | "example" | "exercise"
        content: string                 // 已合并、顺序整理后的内容
        order_index: number              // 在章节内的物理顺序
        confidence: float
    }

**保证：**

- 输出的树结构不变

- 所有block都应该被引用不能拉下，哪怕独立成一个object，保证教材的完整性

- 每个对象可能包含多个类型的block_id，但是一个对象的组成单元一定来自同一个最小标题分组下

**不保证：**

- 对象里的组成（图片、表格、题目）一定是搭配的

**失败处理**

- 返回空

- 保留 version_id 以便排错

---

## 模块三 题目理解与知识点映射

### 接口一：QuestionAnalysis

> 对题目进行识别分析，方便挂载

**输入：**

question_image_path:string

**输出：**

    QuestionAnalysis {
        question_id: string                    //问题id
        question_text: string                   //问题文本
        analysis_units: {
            relat_knowledge: string[]           // 题目涉及到的知识点，细粒度大，用于挂载时的匹配
            search_knowledge: string[]          //题目涉及到的知识点，细粒度可小，主要展现题目“特质”
            semantic_cues: string[]             // 关键词、公式、概念线索
            reasoning_steps?: string[]          // 可选，给用户看的推理
        }[]
    }


**保证**

- 为应对综合类数学题，relat_knowledge最小单元要以章节树一级标题为细粒度（最大标题）

- 只分析，不匹配

**不保证**

- 分析结果一定准确

**失败返回**

- 空QuestionAnalysis

### 接口二：KnowledgeMatchCandidate

> 开始根据分析结果匹配候选章节id，即挂载位置

**输入：**

QuestionAnalysis

SemanticStructureTree

**输出：**

    KnowledgeMatchCandidate {
        question_id: string
        candidates: {
            chapter_id: string      //候选章节id
            score: float            //排名指标
            cues: string[]          // 命中依据，主要用于解释
            conflict_notes?: string[] // 为什么可能不匹配
        }[]
        summary: string          // 给用户的一句话说明
    }


**保证**

- 只匹配，不确认

- 当候选为空时，保证匹配说明中有明确解释以及匹配建议（供用户自定义新开一节）

**不保证**

- 匹配结果一定准确

**失败返回**

- 空KnowledgeMatchCandidate

### 接口三：KnowledgeMatchResult

> 这里前端将显示候选，我们接收用户的选择之后，将其挂载到动态结构树中

> 也许你会在此发现又出现了一个全新版本的结构树，难道每次输入题目都要重构一个全新的结构树吗？这么做的目的是为了方便独立模块优化，简单的CRUD会导致牵一发而动全身，可以在业务侧用CRUD实现，规避数据冗余

**输入：**

KnowledgeMatchCandidate

SemanticStructureTree

user_choice

**输出：**

    KnowledgeMatchStructureTree{         //新的包含题目挂载的语义结构树
        version_id: string
        generated_at: timestamp
        derived_from_version: string     // 指向输入的结构树的version
        chapters: SemanticChapterNode[]
        root_ids: string[]
    }

    SemanticChapterNode {
        id: string
        title_text: string
        question_ids?:string[] //节点内新增了问题id进行引用、溯源，挂载在node层级下
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]
        object_ids: string[]
        confidence: float
        source_block_ids: string[]
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

### 接口一：SolveQuestion

> 这里调用LLM解题，返回结构化步骤，并打价值标签

> 灵魂接口，比较开放，优化需思考

**输入：**

QuestionAnalysis        //目的是获得question_text，减少OCR频率

**输出：**

    SolveQuestion {
        question_id: string
        steps: {
            step_id: string          //步骤id
            content: string          // 本步骤做了什么
            formulas?: string[]      // 用到的公式
            reasoning_type?: "algebra" | "geometry" | "logic" | "transformation"   //本步骤的类型
            is_valuable: boolean     //本步骤是否有记录价值
            value_type?: "common_trick" | "key_transformation" | "error_prone"     //价值类型
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

### 接口二 StepsMatchCandidate

> 核心功能，我们并不是简单的挂载，而是将每个题目的知识点拆解出来，补充到各自对应的结构树的树枝下

> 输出结构是一个step对应多个候选chapter，后续让用户选择

**输入：**

ChapterNode

SolveQuestion                     //获得这个题目的所有step

**输出：**

    StepsMatchCandidate {         //候选挂载章节
        question_id: string
        steps_id:string
        candidates: {
            chapter_id: string
            score: float            //排序依据
            cues: string[]          // 解释说明，命中依据
            conflict_notes?: string[] // 为什么可能不匹配
            relation_type: "belongs_to" | "applies_concept"
        }[]
        summary: string          // 给用户的一句话说明
    }

**保证**

- 只匹配，不确认

**不保证**

- 匹配一定准确

**失败处理**

- 输出空


### 接口四： StepsMatchStructureTree

> 在用户做出选择后构建挂载有价值步骤的新结构树

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
    }

    SemanticChapterNode {
        id: string
        title_text: string
        question_ids?:string[]
        attached_steps?: {                //该节点下挂载的步骤列表
            step_id: string
            source_question_id: string    //溯源步骤来源
        }[]
        level: int
        page_start: int
        page_end: int
        parent_id?: string
        children_ids?: string[]
        object_ids: string[]
        confidence: float
        source_block_ids: string[]
    }

**保证**

- 最终挂载位置由用户确定

- 输出的树结构不变，只做挂载

**不保证**

- 挂载位置一定存在

**失败返回**

- 原样KnowledgeMatchStructureTree