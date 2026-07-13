# A. LLM Wiki 更新链路

> LLM Wiki 更新不是一次同步改写，而是把文档变化转成异步队列操作，再由 Map/Reduce/Finalize 三段流水线合并到 Wiki 页面。
>
> 阅读视角：产品经理
> 源码版本：2026-07-13 本地工作区

---

## 一、这个模块解决什么问题

LLM Wiki 要解决的是“知识库里的文档变了以后，已有 Wiki 如何继续可信地演进”。

最简单的做法是每次上传、删除、重解析都重建整个 Wiki。但 WeKnora 没有这样做，因为 Wiki 页面会跨文档聚合，一个实体页可能同时来自 10 篇文档。全量重建成本高，也容易覆盖用户或 Agent 后续编辑出的页面结构。

当前实现选择了更细的更新模型：

```text
文档新增 / 重解析 / 删除
  -> 写入 task_pending_ops 里的 wiki:ingest 操作
  -> wiki worker 按知识库批量消费
  -> Map：把每篇文档变成 slug 更新
  -> Reduce：按 slug 合并到 Wiki 页面
  -> Finalize：统一更新首页、清死链、补交叉链接
```

一句话总结：**LLM Wiki 更新是“文档级事件输入、页面级合并输出”的异步增量更新系统。**

---

## 二、更新入口：文档处理完成后才进入 Wiki

文档上传或重解析不会立刻写 Wiki。Wiki 更新挂在后处理阶段。

`internal/application/service/knowledge_post_process.go` 先判断这篇知识是否有可用文本 chunk，以及知识库是否开启 Wiki：

```go
willSpawnWiki := kb.IndexingStrategy.WikiEnabled && len(textChunks) > 0
```

如果会生成 Wiki，后处理会把知识状态切到 `finalizing`，并把 Wiki 当成一个后处理子任务计数。这样用户看到的解析状态不会在 Wiki 还没生成时提前变成完成。

随后调用：

```go
EnqueueWikiIngest(ctx, s.taskEnqueuer, s.pendingRepo, payload.TenantID, payload.KnowledgeBaseID, payload.KnowledgeID)
```

这段逻辑的产品含义是：**Wiki 是文档解析完成后的增强产物，不阻塞基础 chunks 入库，但会被纳入“解析最终完成”的状态管理。**

---

## 三、队列层：更新先变成 pending op

`internal/application/service/wiki_ingest.go` 的 `EnqueueWikiIngest` 不直接调用 LLM。它做两件事：

1. 在 `task_pending_ops` 插入一行 `task_type="wiki:ingest"`、`scope="knowledge_base"`、`scope_id=kbID`、`dedup_key=knowledgeID`。
2. 向 asynq 投递一个延迟 30 秒的 `wiki:ingest` 触发任务。

这个 30 秒延迟是 debounce：连续上传很多文档时，不需要每篇文档都马上触发一轮完整更新。

pending op 的核心字段在 `internal/types/task_pending_op.go`：

```go
TaskType string
Scope    string
ScopeID  string
Op       string
DedupKey string
Payload  json.RawMessage
FailCount int
ClaimedAt *time.Time
```

对 Wiki 来说，`DedupKey` 就是 `knowledgeID`。这意味着同一篇文档的多次操作可以被系统识别为同一个更新对象。

---

## 四、Worker 主流程：一次处理一批文档操作

`internal/application/service/wiki_ingest_batch.go` 的 `ProcessWikiIngest` 是更新主干。

它的执行顺序是：

```text
1. 读取 Wiki KB 和模型配置
2. 按 KB 的并发上限占用一个 inflight slot
3. 从 task_pending_ops claim 一批文档操作
4. Map：每篇文档并行生成 SlugUpdate
5. Reduce：按 slug 并行改写 Wiki 页面
6. 发布 draft 页面
7. 写 wiki_log_entries
8. 把首页/死链/交叉链接工作写入 wiki:finalize 队列
9. 删除成功的 pending rows，失败的留下重试或进 dead letter
```

这里有两个关键并发保护：

`ClaimBatch` 保证同一 `dedup_key` 的操作不会被不同 worker 拆开处理。也就是说，同一篇文档的 ingest 和 retract 不会并发互相打架。

`withSlugLock` 保证同一个 Wiki 页面 slug 的读改写是串行的。比如两个文档同时贡献 `entity/acme`，它们可以并行 Map，但真正合并到 `entity/acme` 时要排队。

---

## 五、新增文档：Map 生成候选，Reduce 合并页面

新增文档走 `mapOneDocument`。

第一步，它用该文档 chunks 重建近似全文：

```go
chunks, err := s.chunkRepo.ListChunksByKnowledgeID(ctx, payload.TenantID, knowledgeID)
content := reconstructEnrichedContent(ctx, s.chunkRepo, payload.TenantID, chunks)
```

然后做三类 LLM 工作：

1. 抽取候选实体和概念 slug。
2. 生成该文档 summary 页面内容。
3. 分批扫描 chunks，判断哪些 chunk 支撑哪些 slug。

Map 的输出不是页面，而是 `SlugUpdate`：

```text
summary/<knowledgeID> -> 文档摘要更新
entity/foo            -> 实体页新增贡献
concept/bar           -> 概念页新增贡献
```

Reduce 阶段再按 slug 聚合。`reduceSlugUpdates` 会读取当前页面，如果不存在就创建 draft 页面；如果存在，就把本批新增信息和原页面内容一起交给 `WikiPageModifyPrompt`，由 LLM 生成改写后的页面正文。

页面上会保留两层来源：

- `source_refs`：哪些 knowledge 文档贡献了这个页面。
- `chunk_refs`：哪些 chunk 是具体证据。

---

## 六、重解析：不是先删 Wiki，而是在新内容到来时做替换

重解析入口在 `internal/application/service/knowledge_process.go`。如果 KB 开启 Wiki，它先调用 `prepareWikiForReparse`。

`prepareWikiForReparse` 做得很克制：只清理该 knowledge 还没执行的旧 `ingest` pending op，不删除已有 Wiki 页面，也不写 tombstone。

原因在 `knowledge_delete.go` 的注释里说得很清楚：重解析不是“这篇文档没了”，而是“这篇文档的贡献要换成新版本”。真正的替换要等新 chunks 解析完成后，在 `mapOneDocument` 里完成。

`mapOneDocument` 会先查这篇文档之前贡献过哪些页面：

```go
oldPageSlugs := s.getExistingPageSlugsForKnowledge(ctx, payload.KnowledgeBaseID, knowledgeID)
```

然后把旧页面集合和新抽取集合做对比：

```text
旧 slug 仍在新结果里：
  -> 同时发 retract + addition
  -> 页面编辑模型执行“替换旧贡献，不是追加重复内容”

旧 slug 不在新结果里：
  -> 发 retractStale
  -> 从这个页面中撤回该文档的贡献

summary 页面：
  -> 直接整体覆盖
```

所以重解析的产品语义是：**旧贡献不会在 reparse 开始时立即消失，而是在新 Wiki ingest 成功时被替换或撤回。**

这有一个体验取舍：重解析过程中，用户可能短暂看到旧 Wiki；但如果新解析失败，旧 Wiki 不会被提前清空。

---

## 七、删除文档：先阻止复活，再撤回贡献

删除路径比重解析更强，因为文档真的不存在了。

`cleanupWikiOnKnowledgeDelete` 做三件事：

1. 写 Redis tombstone：防止已排队或正在 LLM 调用中的 ingest 在最后写出幽灵页面。
2. 删除该 knowledge 的 pending ingest op。
3. 立即处理已有页面，并且无条件排一个 retract op。

已有页面的处理规则是：

```text
页面只来自这篇文档：
  -> 删除页面

页面还来自其他文档：
  -> 移除这个 knowledge 的 source_refs
  -> 移除对应 chunk_refs
  -> 后续 retract 让 LLM 清理正文里的旧贡献
```

为什么还要“无条件”排 retract？因为存在竞态：删除函数查看页面时页面可能还没生成，但另一个 ingest worker 可能马上就要写页面。retract worker 会在运行时重新查 `ListPagesBySourceRef`，所以即使第一次快照为空，也能兜底撤回后来冒出来的页面。

---

## 八、Finalize：把页面局部更新收敛成知识库整体状态

Reduce 完成后，页面内容已经能看到了，但首页介绍、死链清理、交叉链接这些是 KB 级工作。

这些工作不在每个 ingest 批次末尾同步做，而是写入 `wiki:finalize` lane，并用 `wiki-finalize-<kbID>` task id 合并触发。

`ProcessWikiFinalize` 负责：

1. 读取 finalize rows。
2. 根据本批 added/removed 文档增量更新 index intro。
3. 对受影响页面清理死链。
4. 对受影响页面注入新交叉链接。
5. 如果还有 finalize rows，继续 reschedule。

这个设计的重点是规模：批量导入几万篇文档时，不能每 5 篇就重建一次首页。

---

## 九、产品视角：最有价值的设计决策

- **异步队列让上传体验和 Wiki 生成解耦**：chunks 可以先可用，Wiki 作为增强能力后台完成。
- **按文档 dedup，按页面加锁**：系统既能让同一 KB 多批并行，又避免同一文档或同一页面的竞态写入。
- **重解析采用“成功后替换”**：避免新解析失败时提前清空旧 Wiki，但会带来短暂旧内容窗口。
- **删除采用 tombstone + retract 双保险**：防止 queued/in-flight ingest 把已删除文档重新写回 Wiki。
- **Finalize 从批次尾部拆出去**：把 KB 级收尾从高频 O(batch) 变成 debounce 后的低频收敛，适合大规模导入。

---

## 附：涉及的核心文件

| 文件 | 角色 |
|---|---|
| `internal/application/service/knowledge_post_process.go` | 文档解析后的后处理入口，决定是否 enqueue Wiki ingest |
| `internal/application/service/wiki_ingest.go` | Wiki pending op 入队、retract 入队、锁、重试、finalize enqueue |
| `internal/application/service/wiki_ingest_batch.go` | Wiki 更新主流程：ProcessWikiIngest、mapOneDocument、reduceSlugUpdates、ProcessWikiFinalize |
| `internal/application/service/wiki_ingest_cite.go` | chunk 引用分类，把候选 slug 对齐到具体 chunk |
| `internal/application/service/knowledge_delete.go` | 删除和重解析时的 Wiki 队列清理、tombstone、retract |
| `internal/application/repository/task_queue.go` | task_pending_ops 的 ClaimBatch/Delete/Retry 实现 |
| `internal/application/service/wiki_page.go` | Wiki 页面创建、更新、出入链维护 |
| `internal/types/wiki_page.go` | WikiPage 数据模型：source_refs、chunk_refs、links、folder 等字段 |
