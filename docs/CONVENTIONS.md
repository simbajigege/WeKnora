# 代码约定

以下规则来自当前代码、lint 配置和最近 100 条提交；“建议”表示仓库存在不一致，新增代码应向该方向收敛。

## 1. Go

- package 和文件名使用小写，复合文件名常用 snake_case：`knowledge_process.go`、`web_search_provider.go`。
- 测试与实现同包放置，文件以 `_test.go` 结尾：`knowledge_process_test.go`。
- 导出类型/函数用 PascalCase：`KnowledgeService`、`NewKnowledgeService`；非导出标识符用 camelCase：`knowledgeService`。
- 构造函数通常为 `NewXxx`，返回接口或具体 handler：`NewKnowledgeRepository`、`NewKnowledgeHandler`。
- 接口集中在 `internal/types/interfaces/`，实现通常用非导出 struct，并用构造函数暴露。
- import 按 gofmt 分组；提交前运行 `make fmt`。`.golangci.yml` 启用 `gofmt`、`gofumpt`、`govet`、`revive` 和 120 列 `lll`。
- error 要增加语义上下文并保留 `%w`；领域级 not-found 使用可判定 sentinel，不用字符串比较。
- 接收请求链路的函数以 `context.Context` 传播 tenant、principal、trace 和取消信号。

## 2. Vue / TypeScript

- 页面和大型组件通常为 PascalCase：`KnowledgeBase.vue`、`AgentEditorModal.vue`。
- 仓库仍有历史小写或连字符组件，如 `doc-content.vue`、`knowledge-processing-timeline.vue`；新增组件建议统一 PascalCase，不做无关批量重命名。
- 组合式函数使用 `useXxx`：`useChatStreamHandler.ts`；Pinia store 使用 `useXxxStore`。
- API 按业务域组织在 `frontend/src/api/<domain>/index.ts` 或少量顶层 kebab-case 文件中。当前两种组织并存，扩展既有域时沿用该域现状。
- 使用 `@/` 指向 `frontend/src`；页面不得自行创建另一套 Axios client，复用 `utils/request.ts`。
- 路由页面使用动态 import；路由 meta 明确认证、初始化和系统管理员要求。
- 用户可见文本进入 i18n，不只修改单一语言；当前语言文件较大，键名应按功能域分组。
- 安全渲染复用现有 DOMPurify/markdown 工具，不能用未净化 HTML 替代。
- TypeScript 修改至少执行 `npm run build-with-types`；纯工具函数优先使用 Node test 的 `*.test.ts`/`*.test.mjs`。

## 3. Python

- 模块、函数与变量用 snake_case，类用 PascalCase：`chain_parser.py`、`BaseParser`。
- DocReader parser 放在 `docreader/parser/` 并通过 registry/chain 选择，不把格式分支堆入服务入口。
- 测试以 `test_*.py` 为主；DocReader 依赖由 `pyproject.toml` + `uv.lock` 锁定。
- `mcp-server/pyproject.toml` 声明 Black 88 列和严格 mypy；实际 Python 版本声明存在不一致，见技术债务。

## 4. 目录与边界

- HTTP 参数/响应放 handler；用例编排放 application/service；关系查询放 application/repository；外部系统适配放 infrastructure、models、datasource 或 im。
- 共享领域契约放 `internal/types`，不要为绕过依赖方向从 repository 反向导入 handler。
- Vue 页面放 `views/`，可复用 UI 放 `components/`，状态放 `stores/`，无 UI 的可复用逻辑放 `composables/`/`utils/`。
- SQL schema 修改必须添加递增的 up/down migration，并同步 SQLite 变体（若 Lite 受影响）。
- 生成文件包括 `docs/swagger.*`、`docs/docs.go`、`docreader/proto/*pb*`；修改源注解/proto 后运行生成命令，不手工修生成物。

## 5. 注释与文档

- 注释解释约束、风险和“为什么”，例如 repository 对并发字段的 omit 原因；不要逐行翻译代码。
- 导出 Go 标识符遵循 Go doc 风格，以标识符开头。
- 安全、并发、兼容性 workaround 必须说明失效条件，并尽可能附测试。
- API/架构/工作流变化同步更新 `docs/`；CLI 特有契约同步 `cli/AGENTS.md` 和 `cli/CHANGELOG.md`。

## 6. Git Commit 格式

最近 100 条提交中，84 条为 scoped Conventional Commit，5 条为无 scope typed commit，11 条为普通句子。因此新增提交统一采用：

```text
type(scope): imperative summary
```

常用 type：`feat`、`fix`、`docs`、`refactor`、`test`、`chore`、`perf`、`build`、`ci`。scope 使用稳定业务域，如 `knowledge`、`wiki`、`router`、`frontend`、`cli`、`docreader`、`rbac`。

示例：

```text
feat(knowledge): support custom extraction instructions
fix(router): restrict KB-scoped file proxy to exports
docs(harness): document fork synchronization workflow
```

- 标题描述结果，避免 “update code” 等无信息文本。
- 一个 commit 只承载一个可回滚意图；测试和对应文档应与实现同 commit。
- 破坏性 CLI 变更还要更新 `cli/CHANGELOG.md` 的 `BREAKING` 说明。
- 分支选择、同步和提交前检查见 `docs/git-commit-workflow.md`。

## 待补充

- [ ] 仓库未发现统一 ESLint/Prettier 配置；前端最终格式规则需维护者确认。
- [ ] Python DocReader 没有在根级 CI 中形成明确的 lint/type-check 门禁，需确认期望命令。
