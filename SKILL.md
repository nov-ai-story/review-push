---
name: review-push
description: 用户主动触发的审查与提交推送流程：先按 merge-review 方法审查并分类当前变更，确认没有阻塞问题后生成 Conventional Commits 提交信息，提交本地变更，fetch 当前分支并检查远端变化，再安全推送。
---

# Review Push

这是 `merge-review` 的写入型衍生流程。它只用于用户明确要求“审查通过后提交并推送”的本地 Git 仓库。

## 安全门槛

- 只操作用户明确指定的本地仓库、当前分支和远程；不自行切换分支。
- 审查发现 `Critical`、`Warning` 或未解决的 `Question` 时，停止在报告阶段，不提交、不 fetch、不 push。
- 不执行项目代码、测试、构建、安装脚本或未知仓库脚本。可以运行 Git 只读命令和用户明确指定的 Git 检查命令。
- 提交前检查工作区和暂存区，发现与本次变更无关的用户修改时停止并说明，不替用户清理或覆盖。
- 不使用 `git reset --hard`、`git clean`、`git checkout --`、强制 push 或自动覆盖远端历史。
- push 是外部写操作。完成审查和本地提交后，在真正 push 前展示目标远程、分支、commit SHA 和提交摘要，并获得用户对本次 push 的确认；用户已在当前请求中明确授权本次具体 push 时可视为已确认。

## 前置审查：复用 merge-review 方法

先读取同目录的 `../merge-review/SKILL.md`，执行其只读审查流程。审查必须完成以下事项：

1. 确认当前分支、工作区状态、变更范围和关联需求。
2. 枚举全部变更文件，不把某个 diff 锚点当成完整变更。
3. 将文件分类为：逻辑、数据结构、常量/枚举、展示/文案、测试/配置。
4. 深审逻辑主变更及其直接依赖，检查边界条件、异常、并发、兼容性和副作用。
5. 输出简短结论、重点问题、影响模块和人工复核清单。

审查结束条件：所有变更文件已经分类；高优先级风险有明确结论；没有未解决的阻塞问题；用户提供的需求或 spec 已完成对照。审查结论不满足这些条件时不得进入 Git 写入阶段。

## Git 写入流程

### 1. 检查仓库状态

依次读取：

```bash
git rev-parse --show-toplevel
git branch --show-current
git status --short
git diff --stat
git diff --cached --stat
git remote -v
git branch -vv
```

确认：

- 当前目录是目标仓库。
- 当前分支不是 detached HEAD。
- 工作区和暂存区的变更都属于本次审查范围。
- 当前分支存在明确的 upstream；没有 upstream 时先停止并说明目标不明确。
- 没有未解决的 merge/rebase 冲突。

### 2. 生成提交信息

根据实际改动选择 Conventional Commits 类型：

- `feat`：新增用户可见能力
- `fix`：修复错误或不正确行为
- `refactor`：不改变外部行为的结构调整
- `test`：只增加或调整测试
- `docs`：只修改文档
- `chore`：构建、配置、依赖或工具链

标题格式：

```text
<type>(<optional-scope>): <imperative short summary>
```

标题使用英文、动词开头、说明目的而不是罗列文件，尽量不超过 72 个字符。正文使用 Markdown 无头列表，一条描述一个实际变更，说明行为或影响：

```text
fix(three-point): classify station request failures

- Return structured errors for business, HTTP, network, timeout, and invalid responses
- Update each station independently as its request settles
- Show station-specific retry feedback and suppress duplicate image error toasts
```

不要在 commit message 中写未经验证的结论，不要把审查意见、临时调试信息或完整 diff 粘进去。生成后先展示完整 commit message，让用户能检查标题和列表。

### 3. 创建本地提交

提交前再次读取 `git diff` 和 `git diff --cached`，确认没有敏感信息和无关文件。只暂存属于本次变更的文件；若用户已经明确暂存了本次变更，可以保留暂存状态。

使用非交互命令创建提交，例如：

```bash
git add -- <explicit-file-list>
git commit -m "<subject>" -m "- item one\n- item two"
```

提交后记录新 commit SHA，并读取：

```bash
git show --stat --oneline --summary HEAD
git status --short
```

如果提交钩子失败，停止并报告钩子输出；不要跳过钩子，除非用户明确要求。

### 4. fetch 当前分支并检查远端变化

先解析 upstream，得到远程名和分支名，然后只 fetch 当前分支：

```bash
git fetch <remote> <branch>
git rev-list --left-right --count HEAD...<remote>/<branch>
```

解释结果：

- `0 0`：本地和远端一致，说明刚才的 commit 可能已经被某种流程同步，先停止并检查。
- `1 0` 或本地领先：可以继续 push，但仍需 push 前确认。
- `0 N`：远端有本地没有的提交。优先执行 `git rebase <remote>/<branch>` 将本地提交放到最新远端之上；如果出现冲突，停止并交给用户处理，不自动猜测解决方案。
- 两边都大于 0：发生分叉，停止并要求用户选择 rebase 或 merge；不要自动创建 merge commit。

fetch 只更新远程跟踪引用，不能单独防止冲突；必须比较提交图，并在远端前进时整合后再 push。

### 5. 推送

push 前展示：

- remote 名称和 URL（隐藏任何 token）
- 目标分支
- 将要推送的 commit SHA 和标题
- `git log <remote>/<branch>..HEAD --oneline` 的提交列表
- 工作区是否干净

获得确认后执行：

```bash
git push <remote> HEAD:<branch>
```

禁止 `--force`、`--force-with-lease` 和推送到未确认的分支。push 失败时保留本地提交，报告失败原因和下一步，不重复盲推。

## 输出格式

使用简体中文，保持主次清晰：

```markdown
## 审查结论
[通过 / 阻塞；只列最高优先级原因]

## 变更分类
- 逻辑主变更：...
- 数据结构：...
- 常量/枚举：...
- 展示/文案：...
- 测试/配置：...

## Commit 草案
```text
<conventional subject>

- <change>
- <change>
```

## Git 状态
- 当前分支：...
- 工作区：...
- fetch 结果：...
- 是否需要整合远端：...

## 推送结果
- 提交 SHA：...
- remote/branch：...
- 结果：已推送 / 等待确认 / 因冲突停止 / 推送失败
```

没有通过审查时，输出风险和人工复核清单，不输出“提交成功”或“已推送”。
