# 质量标准

## 1. Definition of Done

一个任务只有同时满足以下条件才算完成：

- [ ] 行为符合需求，正常路径、边界和至少一个失败路径已验证。
- [ ] 改动位于正确层级，没有从 handler/router 直接穿透到数据库或 provider。
- [ ] 新行为有自动化测试；修 bug 先增加能复现问题的回归测试。
- [ ] 受影响模块的格式、lint、类型检查、构建和测试通过。
- [ ] 跨模块契约（API/proto/event/migration）在生产者和消费者两端验证。
- [ ] 鉴权、租户隔离、RBAC、API-key capability、资源归属均按适用范围检查。
- [ ] 文件/URL/HTML/SQL/tool 输入经过既有路径安全、SSRF、XSS 或注入防护。
- [ ] 数据库变化包含 up/down migration、Lite/标准版影响和 dirty-state/回滚考虑。
- [ ] 并发任务考虑幂等、重试、重复投递、取消、部分失败和 worker 重启。
- [ ] API、配置、架构、约定或工作流变化同步文档与示例。
- [ ] diff 中没有密钥、token、个人数据、构建产物、调试输出和无关改动。
- [ ] 分支、commit 和同步方式符合 `docs/git-commit-workflow.md`。

## 2. 按模块执行的检查

### Go 主模块

```bash
make fmt
make lint
make test
go build ./cmd/server
```

最小定向测试示例：

```bash
go test ./internal/middleware -run TestRequireRole
go test ./internal/application/service -run TestKnowledge
go test ./internal/router
```

### Vue 前端

```bash
cd frontend
npm ci
npm test
npm run build-with-types
```

涉及主 SPA/Embed 或桌面时，至少打开对应入口做一次关键路径 smoke test。普通 `npm run build` 不包含完整 `vue-tsc` 门禁。

### CLI

```bash
cd cli
make build
make test
make lint
```

CLI CI 还在 Linux/macOS/Windows 执行 `go test -race`、coverage 和 `go vet`；协议变化需更新 contract golden，live E2E 由 `acceptance-e2e` label 或手工 workflow 触发。遵循 `cli/AGENTS.md`。

### DocReader

```bash
cd docreader
uv sync
uv run python -m unittest discover -s tests -p 'test_*.py'
```

若修改 `docreader.proto`，运行 `docreader/scripts/generate_proto.sh`，并验证 Go/Python 生成物和客户端测试。仓库尚未给 DocReader 定义统一根级 CI lint/type-check 命令，见“待补充”。

### Python MCP server

```bash
cd mcp-server
python -m pip install -e ".[test]"
python -m pytest
```

pyproject 声明 Black、flake8 和 mypy dev 依赖；在版本配置不一致解决前，不把单一 mypy Python 版本视为权威。

### Migration

```bash
make migrate-version
# 新建：make migrate-create name=short_description
```

不要在共享/生产数据库上为了测试随意执行 down/force。迁移测试应使用可丢弃数据库，并验证 fresh install 与已有版本 upgrade。

## 3. 代码审查清单

### 正确性

- [ ] 输入空值、分页、超时、取消、部分失败和重复请求是否处理？
- [ ] repository 查询是否带正确 tenant/KB scope？
- [ ] handler 返回的 not-found/forbidden 是否避免资源枚举和信息泄露？
- [ ] 流事件在断开、重连和模型中途失败时是否有终态？
- [ ] 队列任务是否可能重复执行或永久停在 pending/finalizing？

### 安全

- [ ] 新路由是否经过 Auth、API-key 默认拒绝策略和正确 RBAC guard？
- [ ] `ByIDOnly`、跨租户共享、系统管理员 bypass 是否有显式理由和测试？
- [ ] 外部 URL 是否复用 `internal/utils/security.go` 等 SSRF 防护？
- [ ] 文件路径是否 canonicalize 并限制在允许前缀？
- [ ] HTML/Markdown 是否净化；secret 是否在日志、响应、trace 中 redact？
- [ ] Agent/MCP/sandbox 工具是否遵循审批与权限边界？

### 可维护性

- [ ] 命名和文件组织符合 `docs/CONVENTIONS.md`？
- [ ] 大文件是否继续膨胀，是否能沿已有域边界拆分？
- [ ] 新 provider/connector 是否通过 registry/factory 扩展，而非散落 switch？
- [ ] 注释是否解释并发、兼容或安全原因，而非复述代码？

### 兼容性

- [ ] 标准/Lite/Desktop、PostgreSQL/SQLite是否受影响？
- [ ] API、CLI JSON envelope、SSE event、proto、配置字段是否向后兼容？
- [ ] 前端所有语言、Embed、IM 和外部 client 是否需要同步？

## 4. 测试布局快照

2026-07-13 扫描到约 487 个 Go `_test.go`、28 个前端测试文件、15 个 Python 测试文件。测试重点覆盖 middleware/RBAC、安全工具、repository/service、Agent/IM、chunker 和 CLI contract。

当前 GitHub Actions 仅有 CLI、CLI E2E、镜像和 Lite release workflow；普通根模块 PR 缺少统一 Go/前端/DocReader 门禁。因此本地检查不能假设会被 CI 补做，此项已记录到技术债务。

## 5. 提交前最小门禁选择

| 改动 | 最低检查 |
|---|---|
| 纯 Markdown 文档 | 链接/路径检查、`git diff --check` |
| Go 单包 | `gofmt` + 该包测试；共享接口再跑依赖包 |
| handler/router/RBAC | 对应测试 + `go test ./internal/router ./internal/middleware` |
| 前端工具/组件 | `npm test` + `npm run build-with-types` |
| DocReader/parser | 定向 pytest + Go docparser/client contract 测试 |
| migration/model | 可丢弃 DB 的 up/down/upgrade + repository tests |
| CLI wire contract | CLI 全测试、vet、golden/skill vocabulary 检查 |
| 跨栈功能 | 各层单测 + 一条端到端 smoke path |

## 待补充

- [ ] 根 Go、前端、DocReader 的官方 CI 必选命令与覆盖率阈值。
- [ ] 标准/Lite/Desktop 的性能基线、容量目标和 SLO。
- [ ] DocReader 官方 lint/type-check 命令。
