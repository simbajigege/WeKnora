# Fork 同步、分支与 Git 提交工作流

本文是本 fork 的 Git 操作权威指南。每次修改、提交、同步或发起 PR 前，先从下面的决策表选择流程，不凭印象操作。

## 1. 远端和长期分支

| 名称 | 指向/用途 | 规则 |
|---|---|---|
| `upstream` | `https://github.com/Tencent/WeKnora.git` | 只拉取腾讯原仓库，禁止推送 |
| `origin` | `https://github.com/simbajigege/WeKnora.git` | 推送个人 fork |
| `main` | 镜像 `upstream/main` | 不放二开提交，不 squash/rebase 上游历史 |
| `custom/main` | 二开长期集成分支 | 接收上游同步和二开功能 |
| `feat/*`、`fix/*` | 短期二开分支 | 默认从 `custom/main` 创建 |
| 上游贡献分支 | 准备提交腾讯的独立分支 | 必须从最新干净 `main` 创建 |

当前本地已配置：`main` track `upstream/main`，默认 push remote 为 `origin`，`upstream` push URL 为 `DISABLED`，`pull.ff=only`。

## 2. 先选流程

| 你的目标 | 起点 | 使用流程 |
|---|---|---|
| 只把腾讯更新同步到 fork | `main` | A：同步镜像主线 |
| 把腾讯更新带入二开代码 | `custom/main` | 先 A，再 B：合并上游到二开主线 |
| 开发只属于本 fork 的功能 | `custom/main` | C：二开功能开发 |
| 修复二开版本的线上问题 | `custom/main` | D：二开 hotfix |
| 给腾讯原仓库贡献通用修复 | `main` | E：上游贡献 |
| 只有文档/知识库变化 | 按归属选择 `custom/main` 或上游贡献分支 | C 或 E，commit type 用 `docs` |

不要从带有二开提交的 `custom/main` 创建上游 PR 分支，否则 PR 会夹带私有改动。

## 3. 所有工作开始前：Preflight

```bash
pwd
git status --short --branch
git remote -v
git branch -vv
```

确认：

- 当前位于 WeKnora 仓库根目录。
- 没有覆盖用户未提交修改；若工作树不干净，先判断它们是否属于当前任务。
- `origin`/`upstream` 指向正确仓库。
- 已阅读 `AGENTS.md`、相关架构文档和 `docs/QUALITY.md`。

Agent 执行 `commit`、`push`、merge、rebase 或删除分支前，应确认当前任务明确授权该动作；不要因为“完成即提交”自行扩大权限。

## A. 同步镜像主线

目标：让本地 `main` 和 fork 的 `origin/main` 精确跟随腾讯 `upstream/main`。

```bash
git fetch upstream --prune
git switch main
git status --short
git merge --ff-only upstream/main
git push origin main
```

检查：

```bash
git rev-list --left-right --count upstream/main...main
git log --oneline upstream/main..main
```

期望输出分别是 `0  0` 和空列表。如果 `--ff-only` 失败，说明 `main` 已有本地/二开提交：停止，不要强推；先创建保护分支并人工判断这些提交的归属。

```bash
git branch rescue/main-YYYYMMDD main
```

未经确认不要用 `reset --hard` 或 `push --force` 修复镜像分支。

## B. 将上游更新合入二开主线

先完成流程 A，然后：

```bash
git switch custom/main
git status --short
git merge main
```

冲突处理原则：

1. 先理解上游变更意图，再恢复二开行为；不要机械选择 ours/theirs。
2. API、migration、配置和生成文件冲突要检查双方消费者。
3. 解决后运行受影响模块测试及 `docs/QUALITY.md` 的门禁。
4. 审查 `git diff --cc` 和最终 merge diff，再提交 merge。

```bash
git push -u origin custom/main   # 首次
git push                         # 后续
```

长期共享的 `custom/main` 使用 merge 接收 `main`，避免周期性改写团队共享历史。只有尚未共享的个人短期分支才考虑 rebase。

## C. 二开功能开发

```bash
git switch custom/main
git pull --ff-only origin custom/main   # 远端分支创建后使用
git switch -c feat/short-topic
```

实现期间小步检查；提交前：

```bash
git status --short
git diff --check
git diff --stat
git diff
```

按改动范围运行测试，然后提交：

```bash
git add <明确列出的文件>
git diff --cached --check
git diff --cached
git commit -m "feat(scope): describe the outcome"
git push -u origin feat/short-topic
```

通过 PR 合入 `origin/custom/main`。不要把二开功能 PR 的 base 设为腾讯 `main`。

## D. 二开 Hotfix

从最新 `custom/main` 创建：

```bash
git switch custom/main
git pull --ff-only origin custom/main
git switch -c fix/short-topic
```

先写回归测试，再提交 `fix(scope): ...`。修复若也适用于上游，不直接把二开分支 PR 给腾讯；完成后按流程 E 从干净 `main` 单独移植最小提交（必要时 cherry-pick 后解决二开依赖）。

## E. 向 Tencent/WeKnora 贡献

```bash
git fetch upstream --prune
git switch main
git merge --ff-only upstream/main
git switch -c fix/upstream-short-topic   # 或 feat/upstream-short-topic
```

保持提交只包含可公开、可通用的最小修改。提交前验证没有二开历史：

```bash
git log --oneline upstream/main..HEAD
git diff --stat upstream/main...HEAD
```

然后：

```bash
git push -u origin fix/upstream-short-topic
```

在 GitHub 创建 PR：`base repository = Tencent/WeKnora`，`base = main`，`head = simbajigege:fix/upstream-short-topic`。遵循 `.github/pull_request_template.md`，说明测试、兼容性和文档影响。

## 4. Commit 规范

格式见 `docs/CONVENTIONS.md`，统一为：

```text
type(scope): imperative summary
```

- `feat` 新功能；`fix` bug；`docs` 文档；`refactor` 无行为重构；`test` 测试；`chore`/`build`/`ci` 工程维护。
- scope 用稳定模块名，不用人名、日期或 issue 标题。
- 一个 commit 一个可回滚意图；不要混入格式化、依赖升级或无关文件。
- 使用 `git add <paths>`，避免未检查的 `git add .`。
- 不提交 `.env`、token、凭证、数据库、日志、coverage、二进制或构建目录。
- 已提交但未推送的个人分支可修正；共享分支不 amend/rebase，除非协作者明确同意。

## 5. 提交后与合并后检查

```bash
git status --short --branch
git show --stat --oneline HEAD
git log --oneline --decorate -5
```

合并进 `custom/main` 后删除已合并的短期分支：

```bash
git branch -d feat/short-topic
git push origin --delete feat/short-topic
```

删除远端分支、push、force push 都会改变共享状态；Agent 必须获得对应授权。永远不要删除 `main` 或 `custom/main`。

## 6. 初始化/修复远端配置

仅当新 clone 尚未配置时执行：

```bash
git remote add upstream https://github.com/Tencent/WeKnora.git
git fetch upstream --prune
git branch --set-upstream-to=upstream/main main
git config branch.main.pushRemote origin
git config remote.pushDefault origin
git config pull.ff only
git remote set-url --push upstream DISABLED
git switch -c custom/main
git config branch.custom/main.pushRemote origin
```

首次发布二开主线：`git push -u origin custom/main`。

## 7. 禁止操作

- 不在 `main` 直接开发或合入二开 PR。
- 不向 `upstream` push；不修改其 push URL 绕过保护。
- 不在未检查工作树时切分支、merge、rebase 或清理文件。
- 不用 `reset --hard`、`clean -fd`、`push --force` 处理普通同步冲突。
- 不为“保持整洁”改写已共享的 `custom/main` 历史。
- 不跳过测试、权限检查或 migration 验证后直接合并。
