# 技术决策记录

本文区分“代码可确认的用途”和“根据结构推断的理由”。没有 ADR/讨论记录支持的历史原因明确标记为待补充。

## 1. Go 作为主服务语言

**用途**：HTTP/SSE API、Agent/RAG 编排、provider 适配、任务 worker、CLI 和桌面后端。
**可确认事实**：根 module 与 CLI/client 子 module 使用 Go 1.26；主入口可构建为标准、Lite 和桌面版本。
**选择原因（推断）**：单二进制交付、并发流处理和强类型接口适合自托管集成平台。
**替代方案**：Python/Node 单体；仓库没有记录当时的对比结论。
**注意事项**：CGO、SQLite/vector 扩展和多平台桌面构建提高了构建复杂度。

## 2. Gin + go.uber.org/dig

**用途**：Gin 提供路由和中间件；dig 负责 repository/service/handler/provider 的构造。
**代码证据**：`cmd/server/main.go` 获取 `*gin.Engine`；`internal/container/container.go` 集中 `Provide/Invoke`。
**选择原因（推断）**：系统 provider 数量大，构造函数注入能将可替换实现集中装配。
**替代方案**：手写 composition root、Wire/Fx；原始取舍待补充。
**注意事项**：新增依赖必须更新容器，并避免形成隐式循环；启动测试要覆盖装配失败。

## 3. 接口驱动的 repository/provider 架构

**用途**：隔离业务与数据库、向量库、对象存储、LLM、DocReader、队列。
**代码证据**：`internal/types/interfaces/` 定义契约，容器把具体实现绑定到 service。
**选择原因（确认）**：README 明确支持多模型、多向量库、多存储与私有化部署。
**替代方案**：在业务服务中直接调用 SDK；会放大替换成本并削弱测试隔离。
**注意事项**：不要创建只转发调用、没有稳定边界意义的接口。

## 4. GORM + golang-migrate

**用途**：GORM 承担关系数据访问，golang-migrate 管理显式 up/down schema 迁移。
**选择原因（推断）**：ORM 提高大量领域实体的 CRUD 效率，独立迁移工具保证升级顺序和 dirty-state 可观察。
**替代方案**：GORM AutoMigrate、sqlc、手写 SQL；代码显示生产 schema 不依赖 AutoMigrate。
**注意事项**：model 与 migration 必须同步；tenant filter 是安全边界；SQLite 和标准版 migration 需要保持语义一致。

## 5. Redis + Asynq 的异步任务模型

**用途**：文档解析、enrichment、维护/同步和 Wiki 任务；不同 worker pool 隔离容量。
**代码证据**：容器在 Redis 可用时注册 core/enrichment/maintenance/wiki Asynq server。
**选择原因（推断）**：摄取链路长且包含外部 I/O，需要重试、队列隔离、取消和 DLQ。
**替代方案**：Kafka/Celery/数据库队列；历史选择原因待补充。
**注意事项**：enqueue 与状态变更要考虑原子性、幂等、重复执行和 worker 崩溃恢复；Lite 无 Redis 时需维持等价语义。

## 6. 独立 Python DocReader + gRPC/HTTP 双传输

**用途**：PDF、Office、图片、网页、OCR 等解析。
**代码证据**：Go 的 `DocumentReader` 接口由 `GRPCDocumentReader`/`HTTPDocumentReader` 实现；Python `main.py` 注册解析服务。
**选择原因（推断）**：Python 文档与 OCR 生态更完整，进程隔离可避免把重依赖塞进 Go 主服务；HTTP 支持云解析，gRPC 支持自托管高效传输。
**替代方案**：Go 原生解析、仅 HTTP；原始性能数据待补充。
**注意事项**：proto 是跨语言契约；最大消息、超时、TLS/token 和旧 payload 兼容必须同时测试两端。

## 7. Vue 3 + Pinia + TypeScript + Vite

**用途**：管理 SPA、问答 UI、Embed 双入口以及 Wails WebView。
**代码证据**：`main.ts` 注册 Vue/Pinia/router/i18n，Vite 配置 `main` 与 `embed` 两个 input。
**选择原因（推断）**：Composition API 与 Pinia 适合复杂交互状态，Vite 支持 SPA/Embed 构建拆分。
**替代方案**：React/Next/Nuxt；原始选型原因待补充。
**注意事项**：主 SPA 与 Embed 的认证/体积/安全上下文不同；类型检查不能由普通 Vite build 代替。

## 8. 多检索后端与 registry/factory

**用途**：pgvector、Elastic/OpenSearch、Milvus、Weaviate、Qdrant、Doris、Tencent VectorDB、Neo4j 等。
**选择原因（确认）**：产品目标包含本地、私有云和既有基础设施兼容。
**替代方案**：固定单一向量库；会简化实现但降低部署适配性。
**注意事项**：跨 store 批处理必须按 KB/store 分组；能力差异不能用最低公分母悄悄吞掉。

## 9. ReAct Agent + MCP + sandbox skill

**用途**：在知识检索之外编排 Web Search、MCP 工具与沙箱技能。
**选择原因（确认）**：README 将多步自主推理列为核心能力，代码提供 tool approval 和 sandbox validator。
**替代方案**：只提供固定 RAG pipeline；无法覆盖多步任务。
**注意事项**：工具权限、人工审批、secret 隔离、输出净化、循环上限和可观测性属于必须门禁。

## 10. Langfuse 作为 tracing 后端

**用途**：Agent、模型调用、token、工具与文档处理 trace。
**可确认事实**：README/CHANGELOG 声明 Langfuse 为当前唯一 tracing backend；Gin middleware 在未配置时 no-op。
**选择原因**：历史迁移理由未形成 ADR，待补充。
**注意事项**：trace 不得泄露凭证、私有文档原文或跨租户数据。

## 11. 标准版、Lite 与 Desktop 共用核心

**用途**：同一代码支持 Compose/Kubernetes、单机 SQLite 和桌面应用。
**选择原因（确认）**：降低本地体验门槛，同时保留企业部署形态。
**注意事项**：edition、队列、数据库、静态前端和文件路径的条件分支必须做至少标准/Lite 两种验证。

## 待补充

- [ ] Go/Gin/dig、Vue、Asynq、GORM 和 Langfuse 的原始选型会议或 ADR。
- [ ] DocReader gRPC 与 HTTP 的性能/兼容基准。
- [ ] 各向量库的官方支持等级、能力矩阵和版本升级策略。
- [ ] 标准/Lite/Desktop 的长期兼容承诺与弃用政策。
