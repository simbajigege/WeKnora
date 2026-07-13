# WeKnora — Agent 工作指南

## 这是什么项目
WeKnora 是面向企业知识管理的开源框架，提供文档摄取与解析、RAG 检索问答、ReAct Agent、Wiki、数据源同步和多租户 RBAC。仓库同时包含 Web、CLI、桌面端、小程序及 MCP 接入。

## 快速定向
- **确认目录**：先运行 `pwd`；以下命令默认在仓库根目录执行。
- **技术栈**：Go 1.26 + Gin/GORM/dig；Vue 3 + TypeScript + Vite；Python + gRPC/MCP SDK。
- **主要入口**：`cmd/server/main.go`、`frontend/src/main.ts`、`docreader/main.py`、`cli/main.go`。
- **快速启动**：`cp .env.example .env && docker compose up -d`。
- **开发启动**：`make dev-start`；单独启动后端/前端用 `make dev-app` / `make dev-frontend`。
- **基础测试**：`make test`；前端用 `cd frontend && npm test && npm run build-with-types`；CLI 用 `cd cli && make test && make lint`。

## 知识库地图
修改前按任务阅读相关文档：

| 我想了解…… | 去读这个文件 |
|---|---|
| 业务定位、用户问题和可落地方案 | `docs/business-solution.md` |
| 整体架构、模块边界、数据流 | `docs/ARCHITECTURE.md` |
| 命名、组织和编码约定 | `docs/CONVENTIONS.md` |
| 框架与基础设施选型 | `docs/TECH_DECISIONS.md` |
| 完成定义、测试和审查清单 | `docs/QUALITY.md` |
| 分支、同步与提交工作流 | `docs/git-commit-workflow.md` |
| 当前执行计划 | `docs/exec-plans/active/` |
| 已完成执行计划 | `docs/exec-plans/completed/` |
| 产品待办 | `docs/exec-plans/backlog.md` |
| 技术债务 | `docs/exec-plans/tech-debt-tracker.md` |
| 上游已有开发说明 | `docs/开发指南.md`、`cli/AGENTS.md` |

## 每次工作的固定顺序
1. 运行 `git status --short --branch`，确认工作树和当前分支。
2. 阅读 `docs/git-commit-workflow.md`，选择同步、二开或上游贡献流程。
3. 阅读任务涉及的架构与约定文档；复杂任务在 `docs/exec-plans/active/` 建计划。
4. 小步修改并运行受影响模块的最小测试，再运行提交前质量门禁。
5. 架构、约定或工作流变化时同步更新 `docs/`。
6. **每次提交前必须重读** `docs/git-commit-workflow.md` 的决策表，确认分支起点、目标远端和门禁。
7. 提交前复查 diff；未经用户明确授权，不执行 commit、push、合并或历史改写。

## 关键约束
- `main` 只镜像 `upstream/main`；二开集成在 `custom/main`，功能开发使用 `feat/*` 或 `fix/*`。
- 不在 handler/router 中直接实现持久化；遵循 handler → service → repository/manager 的依赖方向。
- 前端通过 `frontend/src/api/` 访问后端，不在 Vue 组件中散落服务端 URL。
- 数据库结构变化必须增加相应迁移，不能只修改 GORM model。
- 认证、租户、知识库和文件接口必须检查租户隔离、资源归属与 RBAC。
- 不手改 Swagger 生成物、构建产物或依赖锁文件；应通过对应生成/包管理命令更新。
- 不跳过测试或用删除/弱化测试来换取通过。
