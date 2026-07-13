# B. 新增关系与冲突检测

> 解释新增文档如何补充已有实体，以及系统目前能否发现并提醒事实冲突
>
> 阅读视角：产品经理
> 源码版本：2026-07-13

---

## 一、这两个问题的产品本质

新增文档进入 Wiki 时，系统需要完成两个不同层次的任务：

1. **知识合并**：识别文档中的实体与概念，把关于同一对象的新事实并入同一个页面。
2. **知识治理**：发现新旧材料互相矛盾时，保留证据并把问题暴露给用户处理。

当前实现对第一个任务支持较完整；对第二个任务具备页面内冲突处理和 issue 基础设施，但两者还没有自动串起来。

---

## 二、新文档如何补充原有实体

### 第一步：从新文档抽取实体与概念

`internal/application/service/wiki_ingest_cite.go` 先调用候选页面抽取提示词，得到实体或概念的名称、slug、别名和简述。

### 第二步：把新实体对齐到已有页面

`internal/application/service/wiki_ingest.go` 中的 `deduplicateExtractedBatch` 会按名称和别名搜索相似的已有页面，再通过 `WikiDeduplicationPrompt` 判断它们是否是同一个真实对象。确认是同一对象后，新条目的 slug 会被改成已有页面的 slug。

这一步决定了“腾讯”“腾讯公司”“Tencent”是否会落到同一个实体页，而不是生成三个页面。

### 第三步：按 slug 合并新旧信息

`internal/application/service/wiki_ingest_batch.go` 中的 `reduceSlugUpdates` 读取已有页面，把原正文和新文档中支持该实体的 chunks 一起交给 `WikiPageModifyPrompt`。模型被要求保留仍有效的旧信息，并加入有引用的新事实。

因此，如果新增文档同时实质性描述了新实体 B 和旧实体 A，例如“B 是 A 推出的产品”，这段 chunk 可以同时被引用到 A、B 两个候选页面，关系事实有机会同时进入两个实体页。

但这里的“关系”本质上是自然语言事实和 Wiki 链接，不是结构化的 `A --推出--> B` 关系记录。系统不会单独抽取关系类型、方向、有效时间等字段。

### 第四步：补 Wiki 链接

Finalize 阶段的 `injectCrossLinks` 会在本批受影响页面中，根据页面标题和别名把文本提及替换成 `[[slug|名称]]` 链接。

它只重写本批受影响的页面，候选链接主要来自本批新写入页面和页面已有的出链。因此，新文档生成或更新的 A、B 页面通常能互相补链接；很久以前生成、此次未受影响的其他页面不会被全库重写。

---

## 三、第一种情况的完整例子

假设原 Wiki 已有“腾讯”页面，新上传文档写着：

> 元宝是腾讯推出的 AI 助手，接入了混元模型。

系统可能执行：

1. 抽取 `元宝`、`腾讯`、`混元模型` 三个实体。
2. 去重阶段把新抽取的“腾讯”对齐到已有 `entity/tencent`。
3. citation 阶段把这段原文 chunk 同时分配给三个实体。
4. Reduce 更新腾讯页，加入“推出元宝”这一有引用事实；同时创建或更新元宝页和混元模型页。
5. Finalize 尝试把页面正文中的实体名称变成 Wiki 链接。

所以答案是：**会尝试补充与原实体有关的内容，但成功率依赖实体抽取、同实体去重和 chunk 归属判断；它不是一个有确定 schema 的关系抽取系统。**

---

## 四、新旧内容冲突时会发生什么

`internal/agent/prompts_wiki.go` 的 `WikiPageModifyPrompt` 已经明确要求：

```text
If the new chunks clearly and directly supersede or contradict existing content,
update the main text to reflect the newer cited information AND add a brief
"Contradictions / Updates" section summarizing the change.

If the conflict is ambiguous, unresolved, or not directly supported by the
provided chunks, do not overwrite the existing content; instead, add only a
"Contradictions / Updates" section describing the conflict with citations.
```

这意味着，只要新旧事实落到同一个实体页，编辑模型有机会发现矛盾：

- 新资料明确取代旧资料：正文采用新事实，并增加冲突/更新说明。
- 冲突无法判定：保留原正文，在冲突/更新小节并列说明，且附 chunk 引用。

这是“页面生成时的局部冲突识别”，不是独立、确定性的冲突检测任务。它只比较当前页面正文与本次新增材料，也没有结构化比较日期、数值、谓词和来源可信度。

---

## 五、为什么现在还不能算自动提醒用户

项目已经有 `wiki_page_issues`、`contradictory_facts` 类型、待处理数量和前端问题抽屉。用户在页面标题旁可以看到问题标记，并执行忽略或修复。

但自动 ingest 流程发现冲突后，只要求 LLM 把冲突写进页面正文，没有调用 `CreateIssue`。真正创建 `contradictory_facts` issue 的入口是 `wiki_flag_issue` 工具，通常由 Wiki research agent 在阅读页面、确认问题后主动调用。

因此当前状态是：

- **能在合并时发现一部分冲突**：是。
- **能把冲突保留下来供人查看**：是，主要通过正文中的 `Contradictions / Updates`。
- **每次上传后自动生成红色冲突提醒**：否。
- **系统具备承载提醒和人工处理的 UI/数据模型**：是，但需要把 ingest 冲突结果结构化并写入 issue 表。

---

## 六、产品视角：下一步最值得补的能力

- **把“编辑”和“裁决”分开**：页面合并模型不应一边改正文、一边悄悄决定谁正确；更稳妥的方式是输出结构化冲突候选。
- **冲突必须保留来源双方**：issue 至少记录页面 slug、旧 chunk、新 chunk、冲突字段、来源文档和时间。
- **默认不自动判真伪**：除非存在明确版本时间或权威来源规则，否则提示用户选择“采用新值、保留旧值、并列保留”。
- **只对可比较事实检测**：优先处理日期、数值、状态、职位、版本等高置信冲突，避免把观点差异误报为事实矛盾。

---

## 附：涉及的核心文件

| 文件 | 角色 |
|---|---|
| `internal/application/service/wiki_ingest_cite.go` | 抽取候选实体，并把文档 chunks 分配给实体或概念 |
| `internal/application/service/wiki_ingest.go` | 实体去重、已有页面对齐和跨页链接注入 |
| `internal/application/service/wiki_ingest_batch.go` | 按 slug 合并页面的新旧内容 |
| `internal/agent/prompts_wiki.go` | 定义冲突处理和页面编辑规则 |
| `internal/agent/tools/wiki_flag_issue.go` | 由 agent 创建 `contradictory_facts` 等人工待处理问题 |
| `internal/types/wiki_page.go` | 定义 `WikiPageIssue` 数据模型 |
| `frontend/src/views/knowledge/wiki/WikiBrowser.vue` | 展示冲突问题标签和待处理抽屉 |
