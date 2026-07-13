# 技术债务追踪

扫描日期：2026-07-13。每条只记录可由代码/配置验证的现状；优先级表示风险和影响面，不代表已排期。

## 当前债务

- [优先级: 高] 根 Go 服务、Vue 前端和 DocReader 缺少普通 PR 的统一 CI 门禁 — `.github/workflows/` 当前主要覆盖 CLI、镜像和 release，本地漏测可能直接进入主线。
- [优先级: 高] Wiki 后处理 enqueue 是 fire-and-forget，代码明确标注 crash window 会让知识停在 finalizing — `internal/application/service/knowledge_post_process.go`。
- [优先级: 中] 多向量 store 的 batch delete 尚未按每个 KB 的 store fan-out — `internal/application/service/knowledge_delete.go` 已标 TODO，可能影响混合存储租户。
- [优先级: 中] 多个核心 UI/业务文件超过 2,000 行，最高业务 Vue 文件超过 6,000 行，修改和回归风险集中 — 例如 `WikiBrowser.vue`、`FAQEntryManager.vue`、`AgentEditorModal.vue`、`knowledge_process.go`、`im/service.go`。
- [优先级: 中] Python 运行与工具版本声明不一致 — `mcp-server` 要求 Python >=3.10，但 mypy 配置为 3.8；DocReader package description 仍为默认文本，削弱发布和工具元数据可信度。
- [优先级: 低] CLI MCP tools 尚未声明 `OutputSchema` — `cli/internal/mcp/tools.go` 已标 TODO，Agent 无法获得完整结构化输出契约。
- [优先级: 低] Yuque 用户组抓取仍串行 — `internal/datasource/connector/yuque/connector.go` 已标性能 TODO；当前注释判断常见规模较小。
- [优先级: 低] 前端组件/API 文件命名存在 PascalCase、小写、kebab-case 和目录 index 多种历史风格 — 影响检索与一致性，但不应通过无关大规模重命名处理。

## 已解决

- 暂无。解决时将条目移动到这里，并附 commit/PR 与验证方式。

## 维护规则

- 产品新功能不放这里，写入 `backlog.md`。
- 仅凭“大文件”不能直接启动重构；先定义边界、回归测试和可度量目标。
- 与 upstream 同步后复查条目；上游已解决的债务应标记对应提交，不重复二开。
