# LLM Wiki 生成现状与升级方案

> 阶段性总结：说明 WeKnora 如何从用户文档生成 LLM Wiki、Agent 如何使用 Wiki，以及当前基于 chunk 的证据链可能出现的问题和建议升级路径。
>
> 阅读视角：产品与工程评审
> 源码版本：`5eefa70e`（2026-07-13）
> 状态：调研完成，待排期

---

## 一、要解决的产品问题

LLM Wiki 的目标不是简单地把上传文档改写成若干 Markdown 页面，而是将分散在多个来源中的知识整理成可浏览、可关联、可被 Agent 查询，并且能够追溯到原始证据的知识层。

当前实现采用“文档级理解 + chunk 级证据”的混合思路：系统先从已入库的 chunks 重建文档内容，抽取实体、概念和摘要，再逐批扫描 chunks，为实体和概念页面绑定证据。这个方向是合理的，但语义理解和证据定位之间仍存在上下文损失，尤其容易出现在跨 chunk 指代、Markdown 标题上下文和超长文档中。

本次升级不建议用“完整 Markdown”完全替换 chunks。推荐目标是：

```text
规范化 Markdown 负责结构化理解和 Wiki 写作
                 +
chunk / source span 负责检索、引用、核验和信息撤回
```

---

## 二、用户输入与文档解析

### 2.1 用户可以输入什么

知识摄取入口支持文件、URL、手工 Markdown/文本和 API passages。前端默认上传白名单包括：

- 文档：PDF、TXT、Markdown、DOC/DOCX、PPT/PPTX、EPUB、MHTML。
- 表格：CSV、XLS/XLSX。
- 图片：JPG/JPEG、PNG。
- 音频：MP3、WAV、M4A、FLAC、OGG。

实际可用格式还受所选解析引擎影响。例如内置、WeKnora Cloud、MinerU 和 PaddleOCR-VL 的格式覆盖范围并不完全相同。前端默认文件大小上限为 50 MB，也可以由运行时配置覆盖。

### 2.2 解析与切块流程

`internal/application/service/knowledge_process.go` 编排主流程：

```text
上传文件 / URL / passages
  → 选择 DocReader 解析引擎
  → 输出 MarkdownContent、图片引用和元数据
  → 音频 ASR、图片存储与远程图片改写
  → Split 或 SplitParentChild
  → chunks 入库
  → 可选向量/关键词索引
  → 摘要、问题、Wiki、知识图谱等后处理
```

解析器统一产生 `MarkdownContent`。普通文件随后由 Go chunker 切块；API 直接提供的 `passages` 会跳过文件解析，每个 passage 直接成为待入库内容。

默认 chunk 参数为 512 字符、80 字符 overlap。策略包括 `auto`、`heading`、`heuristic`、`recursive` 和兼容旧行为的 `legacy`。切块器会保护 Markdown 图片、链接、表格、代码块和 LaTeX 块，避免常规递归切分直接破坏这些结构。

开启 parent-child 后，系统默认生成较大的 parent chunks 和较小的 child chunks。child 用于 embedding 和精确召回，parent 用于提供更完整上下文。

---

## 三、现有 LLM Wiki 如何生成

### 3.1 Wiki 与独立知识图谱是两套能力

Wiki 由 `indexing_strategy.wiki_enabled=true` 开启，核心存储是 `wiki_pages`、`wiki_folders` 以及页面的 `in_links`、`out_links`、`source_refs` 和 `chunk_refs`。

独立知识图谱由 `indexing_strategy.graph_enabled=true` 与 `extract_config.enabled=true` 开启，Agent 通过 `query_knowledge_graph` 使用。它不是 Wiki 页面链接图。两种能力可以同时开启，但数据模型、生成任务和 Agent 工具不同。

### 3.2 Wiki Map 阶段：先理解文档，再定位证据

`internal/application/service/wiki_ingest_batch.go` 的单文档 Map 流程为：

1. 从数据库读取该文档的全部 chunks。
2. `reconstructEnrichedContent` 按文档顺序重建文本，并把图片 OCR/caption 子 chunk 合并到内容中。
3. 将重建内容截断到最多 32768 个字符。
4. 第一遍调用 LLM，抽取候选 entity/concept slug、名称、别名、描述和 details。
5. 同时生成文档 summary。
6. 第二遍把文本 chunks 分批发送给 `WikiChunkCitationPrompt`，判断每个候选页面由哪些 chunk 支撑。
7. 将候选实体、概念、摘要和证据引用转成 `SlugUpdate`，进入 Reduce。

因此，“Wiki 基于 chunks 构建”需要更准确地理解为：

- 文档理解输入是由 chunks 重建的近似全文。
- 页面证据和后续合并输入优先使用被引用的具体 chunks。
- 如果引用阶段没有找到 chunk，系统会退回第一遍抽取出的 `Description/Details`，页面仍可能生成，但证据粒度会下降。

### 3.3 Wiki Reduce 与收尾阶段

Reduce 按 slug 聚合来自多个文档的更新，并用 per-slug 锁避免并发覆盖：

- summary 页面按文档创建，slug 为 `summary/<knowledge-id>`，只保留文档级来源，不记录 chunk refs。
- entity/concept 页面可以聚合多个来源，优先把被引用 chunk 的原文交给页面编辑模型。
- 页面保留 `source_refs` 和 `chunk_refs`；新增、冲突和撤回通过 Wiki 页面修改提示词合并。
- taxonomy 阶段为新页面规划最多两层的文件夹结构。
- finalize 阶段重建首页介绍、清理死链，并注入 `[[slug|display]]` 形式的 Wiki 交叉链接。

最终页面类型包括 summary、entity、concept、index、log、synthesis 和 comparison。自动摄取主要生成 summary、entity、concept 和结构页；synthesis/comparison 主要由 Agent 写入工具创建。

---

## 四、Wiki 关系与 Agent 使用方式

### 4.1 Wiki 内部有三类关系

1. 页面关系：Markdown 中的 `[[slug|display]]` 被解析成 `in_links/out_links`，构成 Wiki 页面链接图。
2. 文档来源关系：`source_refs` 指向贡献该页面的 knowledge ID。
3. 证据关系：`chunk_refs` 指向支撑页面事实的具体 chunk UUID。

`/wiki/graph` 展示的是第一类页面链接图，不等同于独立知识图谱。

### 4.2 Agent 如何使用 Wiki 回答问题

内部 Agent 可使用 `wiki_search`、`wiki_read_page` 和 `wiki_read_source_doc`：

```text
用户问题
  → wiki_search 找候选页面
  → wiki_read_page 读取页面正文、页面关系和来源
  → 需要核验或补充细节时，从 <sources> 取得 knowledge_id
  → wiki_read_source_doc 搜索或展开原文 chunks
  → 综合 Wiki 页面与原始证据回答
```

Agent 连接的是 Wiki 页面系统，而不是默认连接独立知识图谱。只有配置图谱能力并允许 `query_knowledge_graph` 时，Agent 才会额外查询实体关系图。

Python MCP server 目前提供只读的 `wiki_search`、`wiki_read_page` 和 `wiki_index_view`，外部 Agent 可以通过这些工具接入 Wiki；如需回到源文档的 chunk 级证据，仍需扩展对应 MCP 工具或调用 WeKnora API。

---

## 五、当前可能出现的问题

### 5.1 跨 chunk 语义关系可能丢失

典型场景：

```markdown
# 实体 A

## A1 主题
实体 A 支持能力 X。

## A2 主题
它还支持能力 Y，默认值为 10。
```

如果 A1 和 A2 被切成两个 chunks，后一个 chunk 只有“它”，引用模型可能无法确定“它”指向实体 A。当前有三种可能结果：

1. 两个 chunks 在同一引用批次内，模型利用相邻内容正确归因。
2. Wiki 页面 A 仍由第一遍全文抽取生成，但 A2 对应 chunk 没有进入 `chunk_refs`，页面只能使用第一遍的压缩 details。
3. 第一遍也没有保留 A2，引用阶段又无法归因，A2 从 Wiki 页面中消失。

默认 80 字符 overlap、候选实体描述和同批次相邻 chunks 可以缓解这个问题，但无法覆盖实体名离切分边界较远、跨引用批次或长距离指代的情况。

### 5.2 已生成的标题上下文没有进入 Wiki 引用阶段

heading-aware chunker 会给 chunk 生成类似以下 breadcrumb：

```text
# 实体 A
## A2 主题
```

该信息保存在内存字段 `ContextHeader`，embedding 时会拼到 chunk 正文前。但 `internal/types/chunk.go` 将它声明为 `gorm:"-"`，因此不会持久化。

Wiki 引用任务稍后从数据库重新读取 chunks，并且 `renderChunksXML` 只发送 `chunk.Content`。结果是：向量检索在首次索引时知道 chunk 属于“实体 A > A2”，Wiki 引用模型却看不到这层上下文。

这是当前最明确、最值得优先修复的证据归因缺口。

### 5.3 引用批次之间没有连续上下文

引用阶段按总字符预算把 chunks 切成多个 batch，各 batch 并行调用 LLM。batch 内保留文档顺序，但 batch 之间不共享前一个 chunk，也不会传递前一批的阅读状态。

如果实体定义在上一批末尾，代词或省略主语的细节位于下一批开头，后一批可能漏掉归因。

### 5.4 超长文档第一遍只读取前 32768 个字符

候选实体和文档 summary 的第一遍输入会被截断。引用阶段虽然能继续分批扫描全部 chunks，并允许发现 `new_slugs`，但它更擅长证据分类，不完全等价于对后半篇文档做一次具有完整结构的候选抽取。

因此，长文档可能出现：

- 后半篇独有实体召回不足。
- 文档摘要偏向前半篇。
- 前半篇定义、后半篇展开的概念关系不完整。

### 5.5 规范化 Markdown 没有作为稳定产物持久化

解析器产生的 `MarkdownContent` 在完成图片处理后立即进入 chunker。`Knowledge` 保存原文件路径和元数据，但没有保存一份可供后续任务稳定读取的规范化 Markdown。

如果直接把 Wiki 改为基于 Markdown 生成，目前只能选择：

- 继续从 chunks 重建，无法真正摆脱切块后的信息损失。
- 重新解析原文件，增加成本，并可能因解析器版本或外部服务变化产生不一致结果。

### 5.6 引用缺失会退化为摘要文本

候选页面没有匹配到任何 chunk 时，Reduce 不会拒绝生成，而是使用第一遍抽取的 `Description/Details`。这保证了可用性，却也意味着页面中可能存在无法精确回溯的压缩或改写内容。

当前已经记录 `uncited` 数量用于观测，但尚未形成质量门禁、用户提示或自动重试策略。

---

## 六、建议的目标架构

### 6.1 不做 Markdown 与 chunk 二选一

推荐的职责分工为：

```text
原文件
  → 解析并持久化规范化 Markdown
      ├── 按标题/章节做分层 Map-Reduce
      │     → 实体、概念、摘要、跨章节关系
      └── 切成 chunks
            → embedding、关键词检索、Agent 原文读取
            → Wiki 证据对齐和引用
```

Markdown 负责回答“这篇文档整体在讲什么、章节之间是什么关系”；chunk 或 source span 负责回答“这个事实具体来自哪里”。

### 6.2 P1：先修复上下文证据链

这是改动较小、收益最明确的一阶段：

1. 将 `ContextHeader` 持久化，或在 Wiki 任务中根据规范化 Markdown 与 `StartAt/EndAt` 稳定重建。
2. `renderChunksXML` 同时发送 context 和 content，但 citation 仍只绑定目标 chunk ID。
3. 每个引用 batch 增加只读的前一个/后一个 chunk 上下文，明确禁止将上下文 chunk 当作目标证据误引。
4. 为引用批次边界增加 overlap，或使用滑动窗口分类。
5. 增加 `uncited candidate rate`、`citation coverage` 和跨 chunk 指代失败率指标。

建议提示结构：

```xml
<previous>实体 A 支持能力 X。</previous>
<target id="c002">
  <context># 实体 A > ## A2 主题</context>
  <content>它还支持能力 Y，默认值为 10。</content>
</target>
<next>后续限制条件……</next>
```

验收标准：在目标 chunk 不出现实体名、只通过标题或上一 chunk 建立指代的测试集中，目标 chunk 能稳定绑定到正确实体；同时不得把仅在上下文中出现、目标正文未讨论的实体误绑定到目标 chunk。

### 6.3 P2：持久化规范化 Markdown 并按章节生成

1. 将解析器最终输出的规范化 Markdown 保存为独立对象或受版本管理的正文产物。
2. 记录 parser engine、parser version、内容 hash 和生成时间，保证重解析可审计。
3. 根据 Markdown heading tree 或解析器结构信息切成 section，而不是按固定 32768 字符直接截断。
4. 对 section 做候选抽取，再在文档级 Reduce 中合并同一实体、跨章节关系和摘要。
5. 使用 `StartAt/EndAt` 将 section 结论对齐到 chunks；长期可引入 `source_span(document_id, start, end, hash)`，减少 chunk 重切后引用 UUID 全量失效的问题。

验收标准：超过 32768 字符的文档不再只依赖前缀生成候选和摘要；重新执行 Wiki 任务不需要再次调用 DocReader；相同规范化 Markdown 和配置能够得到可比较的生成结果。

### 6.4 P2：建立 Wiki 质量评测与回归集

至少覆盖以下文档模式：

- 实体名在上一 chunk，目标 chunk 使用“它/该产品/此机制”。
- 实体名只存在于 Markdown 父标题。
- 实体定义与细节恰好跨 citation batch 边界。
- 超过 32768 字符，关键实体只在后半篇出现。
- parent-child chunking 下，child 需要 parent 标题才能解释。
- 多文档对同一实体给出冲突事实。
- 图片 OCR/caption 是唯一事实来源。
- 删除一个来源后，页面应保留其他来源内容并移除失效引用。

核心指标建议：

| 指标 | 含义 |
|---|---|
| Entity/Concept Recall | 应生成的页面中实际生成比例 |
| Citation Recall | 应关联到页面的证据 chunk 中实际关联比例 |
| Citation Precision | 已关联 chunk 中真正讨论该页面主题的比例 |
| Unsupported Claim Rate | 页面事实无法由 source/chunk 证据支持的比例 |
| Long-document Coverage | 长文各章节在页面和摘要中的覆盖率 |
| Retraction Accuracy | 删除来源后，应删除与应保留事实的准确率 |

---

## 七、待办清单

- [ ] **P1：持久化或可重建 Wiki 所需的 chunk `ContextHeader`。** 明确数据库迁移、SQLite 兼容和旧 chunk 回填策略。
- [ ] **P1：升级 Wiki citation prompt 输入。** 为目标 chunk 提供标题 breadcrumb 和受控前后文，引用结果仍绑定目标 chunk。
- [ ] **P1：增加跨 citation batch 上下文。** 使用滑动窗口或边界 overlap，并验证不会扩大误引用。
- [ ] **P1：补充跨 chunk 指代自动化测试。** 覆盖标题继承、代词、batch 边界和 parent-child 四类场景。
- [ ] **P1：增加 citation 质量观测。** 暴露 uncited candidate rate、每页证据数和批次失败率。
- [ ] **P2：设计并持久化规范化 Markdown 产物。** 包含 parser/version/hash，并定义存储配额、生命周期和权限边界。
- [ ] **P2：将候选抽取与摘要改成章节级 Map-Reduce。** 消除 32768 字符前缀截断造成的长文覆盖缺口。
- [ ] **P2：设计 section/source span 到 chunk 的证据对齐。** 评估继续使用 chunk UUID 与引入稳定 source span 的取舍。
- [ ] **P2：建立 Wiki 生成离线评测集和回归指标。** 先形成基线，再决定模型、prompt 和分块策略升级。
- [ ] **P3：扩展只读 MCP 的源文档证据工具。** 让外部 Agent 能从 Wiki 页面继续读取具体来源 chunks。

---

## 八、阶段性决策

1. 保留 chunk 作为检索和证据层，不采用“Wiki 完全绕过 chunk”的方案。
2. Wiki 语义生成逐步转向持久化的规范化 Markdown 和章节结构。
3. 第一阶段优先解决 `ContextHeader` 丢失和引用批次边界问题，因为它们已有明确代码证据且改动范围可控。
4. 在大规模调整生成管线前，先建立跨 chunk 语义与引用质量基线，避免只凭页面观感判断升级效果。
5. Wiki 页面链接图与独立知识图谱继续保持能力边界，不在本次升级中合并数据模型。

---

## 附：涉及的核心文件

| 文件 | 角色 |
|---|---|
| `internal/application/service/knowledge_process.go` | 文档解析、图片/音频处理、切块、入库和后处理编排 |
| `internal/infrastructure/docparser/engine_registry.go` | 解析引擎及支持格式注册 |
| `internal/infrastructure/chunker/splitter.go` | 默认递归切块、overlap 和保护区间处理 |
| `internal/infrastructure/chunker/strategy.go` | auto/heading/heuristic/legacy 策略选择 |
| `internal/types/chunk.go` | chunk 数据模型、ContextHeader 与 embedding 输入规则 |
| `internal/application/service/wiki_ingest.go` | Wiki 摄取常量、内容重建和通用逻辑 |
| `internal/application/service/wiki_ingest_batch.go` | Wiki Map/Reduce/finalize 主流程 |
| `internal/application/service/wiki_ingest_cite.go` | 候选抽取、chunk 引用批次和证据合并 |
| `internal/application/service/wiki_ingest_taxonomy.go` | Wiki 文件夹 taxonomy 规划 |
| `internal/agent/prompts_wiki.go` | Wiki 候选、引用、摘要和页面修改 prompts |
| `internal/types/wiki_page.go` | Wiki 页面、source refs、chunk refs 和配置模型 |
| `internal/agent/tools/wiki_tools.go` | Agent 的 Wiki 搜索与页面读取工具 |
| `internal/agent/tools/wiki_read_source_doc.go` | Agent 按 knowledge ID 回读源文档 chunks |
| `mcp-server/weknora_mcp_server.py` | 外部 Agent 的只读 Wiki MCP 工具 |
| `frontend/src/utils/tool-capabilities.ts` | Agent 工具与 KB 能力开关映射 |
