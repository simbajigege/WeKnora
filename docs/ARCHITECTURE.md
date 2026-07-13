# 架构说明

本文描述仓库当前代码结构。模块关系均由 import、构造函数或路由注册验证；产品功能细节继续查阅 `docs/` 下已有专题文档。

## 1. 系统定位与运行时

WeKnora 是自托管的企业知识框架，核心能力包括文档摄取、可插拔解析与分块、向量/关键词/图检索、RAG 问答、ReAct Agent、Wiki 生成、外部数据源同步、IM/Embed 接入和多租户 RBAC。

标准部署的主要运行时：

```text
浏览器 / Embed / CLI / IM / MCP
              │
              ▼
Vue SPA ──HTTP/SSE──> Go/Gin API ──GORM──> PostgreSQL/SQLite
                          │  │
                          │  ├── Redis + Asynq（异步任务/流状态/并发控制）
                          │  ├── DocReader（gRPC 或 HTTP）
                          │  ├── LLM / Embedding / Rerank providers
                          │  ├── 向量库 / Neo4j / 对象存储
                          │  └── MCP / Web Search / 外部数据源
                          ▼
                     Langfuse tracing
```

Lite 版复用 Go API 与 Vue 前端，改用 SQLite、内存队列并可将前端嵌入单二进制；桌面端由 Wails 包装同一套前端和后端。

## 2. 仓库模块地图

| 目录 | 职责 | 关键入口/证据 |
|---|---|---|
| `cmd/server/` | 标准/Lite Go 服务入口、生命周期 | `cmd/server/main.go` 调用 `container.BuildContainer` 并从容器取得 `*gin.Engine` |
| `cmd/desktop/` | Wails 桌面端、WebView 与本地服务 | `cmd/desktop/main.go` |
| `internal/container/` | dig 依赖注入与运行时资源装配 | `internal/container/container.go` 注册 repository、service、handler、router |
| `internal/router/` | Gin 路由、API-key policy、RBAC guard、Asynq worker | `internal/router/router.go` |
| `internal/middleware/` | JWT/API Key、租户上下文、RBAC、KB 访问、审计、恢复 | `internal/middleware/auth.go`、`rbac.go`、`kb_access.go` |
| `internal/handler/` | HTTP 协议适配、参数校验、响应 | 例如 `KnowledgeHandler` 依赖 service 接口 |
| `internal/application/service/` | 用例编排、知识处理、会话、Wiki、租户等业务规则 | `knowledge.go`、`knowledge_process.go`、`wiki_ingest.go` |
| `internal/application/repository/` | 关系数据、检索和 memory repository 实现 | `knowledge.go` 使用 GORM；`retriever/` 按后端分包 |
| `internal/types/` | 领域类型、上下文值和接口契约 | `internal/types/interfaces/` |
| `internal/infrastructure/` | DocReader、chunker、Web fetch/search 等外部适配器 | `docparser/`、`chunker/`、`web_search/` |
| `internal/models/` | Chat/embedding/rerank provider 与限流 | provider 子包由容器注册 |
| `internal/agent/` | ReAct 循环、工具、审批和观察 | `engine.go`、`tools/` |
| `internal/datasource/` | Feishu/Notion/Yuque/RSS 等同步连接器 | `connector/`、`scheduler.go` |
| `internal/im/` | WeCom/Feishu/Slack/Telegram 等 IM 适配 | 平台子包由容器导入注册 |
| `frontend/` | Vue 3 SPA、Embed 双入口、Pinia、API client | `src/main.ts`、`src/embed-main.ts`、`src/router/index.ts` |
| `docreader/` | Python 文档解析服务和 parser registry | `main.py`、`parser/registry.py`、`proto/docreader.proto` |
| `client/` | 独立 Go API SDK module | 被 `cli/go.mod` 通过 `replace ../client` 使用 |
| `cli/` | Agent-first Go CLI 与内置 MCP server | `cli/main.go`；细节见 `cli/AGENTS.md` |
| `mcp-server/` | Python MCP-to-WeKnora API 桥接 | `main.py`、`weknora_mcp_server.py` |
| `migrations/` | PostgreSQL/SQLite 等版本化迁移 | `versioned/*.up.sql` / `*.down.sql` |
| `config/` | 系统配置、内置 Agent/模型、Prompt | YAML 与 Prompt templates |
| `docker/`、`helm/` | 容器与 Kubernetes 交付 | Compose/Dockerfiles/Helm chart |

## 3. 后端依赖方向

已验证的主路径：

```text
cmd/server
  → internal/container
    → internal/router + middleware
      → internal/handler
        → internal/types/interfaces 中的 Service
          → internal/application/service
            → internal/types/interfaces 中的 Repository/Provider
              → application/repository 或 infrastructure/models 具体实现
```

证据示例：

- `internal/router/router.go` 导入 `handler`、`middleware` 和 `interfaces`，`RouterParams` 由 dig 注入。
- `internal/handler/knowledge.go` 的 `KnowledgeHandler` 依赖 `KnowledgeService`、`KnowledgeBaseService` 等接口。
- `internal/application/service/knowledge.go` 依赖 `KnowledgeRepository`、`ChunkRepository`、`TaskPendingOpsRepository` 等接口。
- `internal/application/repository/knowledge.go` 实现 repository，并直接使用 `*gorm.DB`。
- `internal/container/container.go` 统一注册上述实现，业务代码不自行 new 基础设施。

必须维持的规则：

1. router/handler 不直接访问 GORM、Redis 或外部 provider；否则协议层会承载业务和持久化规则。
2. service 依赖 `internal/types/interfaces` 的抽象；具体 repository/provider 由容器装配，保证存储和模型可替换。
3. repository 查询默认携带 `tenant_id`；只有权限解析所需的显式 `ByIDOnly` 方法可以越过租户过滤，并必须在上层补权限检查。
4. 新 service/handler/provider 必须在 `BuildContainer` 注册，并通过启动或容器测试验证依赖图。
5. 新 API 必须经过全局 Auth、API-key 默认拒绝策略以及对应 RBAC/资源 guard；公开 IM/Embed 路由使用各自签名或 publish-token 机制。

## 4. 前端边界

`frontend/src/main.ts` 实际导入 `App.vue`、router、Pinia、i18n 和全局 UI；`router/index.ts` 实际导入 auth store 与 auth API，并通过动态 import 加载页面。`views/knowledge/KnowledgeBase.vue` 实际导入 `@/api/knowledge-base`、chat/wiki API、多个 store 与组件，证明主要方向为：

```text
main/embed-main → router/App → views → components/composables/stores → api → utils/request.ts → /api/v1
```

约束：

- 页面和组件通过 `frontend/src/api/**` 调用后端，共享 Axios 实例位于 `utils/request.ts`。
- 跨页面状态进入 Pinia `stores/`；局部可复用行为进入 `composables/` 或 `hooks/`。
- 主 SPA 与 Embed 是两个 Vite 入口；Embed 认证、origin allowlist 和文件 URL 逻辑不可直接套用普通 SPA 假设。
- 新路由需设置 `requiresAuth`、`requiresInit`，系统管理页面还需 `requiresSystemAdmin`；前端守卫只改善 UX，真正授权仍由后端执行。

## 5. 核心数据流

### 5.1 知识摄取与解析

1. Web/CLI/API 将文件、URL 或 Markdown 发到 `/api/v1/knowledge-bases/:id/knowledge/*`。
2. Auth、API-key capability、角色与 KB access guard 建立 tenant/principal 上下文。
3. `KnowledgeHandler` 调用 `KnowledgeService`，保存知识记录并提交处理任务。
4. Redis 可用时由 Asynq 的 core/enrichment/maintenance/wiki worker pool 消费；Lite 使用替代的内存实现。
5. `DocumentReader` 接口由 `GRPCDocumentReader` 或 `HTTPDocumentReader` 实现，调用 Python DocReader parser registry。
6. 解析结果进入 chunker、embedding、问题/图谱等后处理，再写关系库、向量库和对象存储。
7. processing span、任务状态和 Langfuse trace 向前端提供可观测进度。

### 5.2 RAG / Agent 问答

1. Web、Embed、CLI 或 IM 创建/读取 session，并发起 chat 请求。
2. Session/Agent service 根据 Agent 配置选择快速 RAG 或 ReAct engine。
3. retriever 从可见 KB 的向量/关键词/图索引召回；Agent 还可调用知识搜索、Web Search、MCP 与 sandbox skill。
4. 模型输出经 event bus/stream manager 转为 SSE 或 IM 平台消息，同时记录引用、token 和 tracing。
5. 前端 `useChatStreamHandler`、chat store 和消息组件投影流事件。

### 5.3 外部数据源同步

连接器实现统一 datasource contract，scheduler/maintenance queue 执行增量或全量同步，最终复用知识摄取管线。新增连接器先读 `internal/datasource/CONNECTOR_IMPLEMENTATION_GUIDE.md`。

## 6. 数据与迁移

- 关系模型以 GORM 为主，但 schema 变更以 `migrations/versioned/` 和 `migrations/sqlite/` 为准。
- 标准版迁移使用 `golang-migrate`；启动时记录版本和 dirty 状态。
- 向量数据可落 PostgreSQL/pgvector、Elasticsearch/OpenSearch、Milvus、Weaviate、Qdrant、Doris、Tencent VectorDB 等。
- 对象存储和外部 URL 属于安全边界，必须复用现有 allowlist、presign 和 SSRF 防护工具。

## 7. 进一步阅读

- 开发与部署：`docs/开发指南.md`、`docs/LITE.md`
- 检索分块：`docs/CHUNKING.md`
- 权限：`docs/RBAC说明.md`、`docs/共享空间说明.md`
- Agent/MCP：`docs/agent-skills.md`、`docs/MCP功能使用说明.md`
- API：`docs/api/README.md`、`docs/swagger.yaml`

## 待补充

- [ ] 原始架构评审记录和各模块 owner 无法从仓库确定，需维护者补充。
- [ ] 标准部署的容量目标、SLO 与支持矩阵未在代码中形成单一权威来源。
