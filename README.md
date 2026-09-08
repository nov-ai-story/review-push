# Review Push

用于审查当前 Git 变更，并在审查通过后创建提交、同步远端和推送当前分支的 Agent Skill。

## 全局安装

仓库现位于 `nov-ai-story` 组织。全局安装到 Codex 时，请显式指定 Codex：

```bash
npx skills add nov-ai-story/review-push --global --yes --agent codex
```

显式指定 `--agent codex` 很重要。`skills` 安装器会根据当前目录自动检测 Agent；如果目录中存在 `.promptscript/` 或 `promptscript.yaml`，非交互安装可能会自动选择 PromptScript。PromptScript 不支持全局 Skill 安装，因此使用未指定 Agent 的全局命令可能出现：

```text
PromptScript does not support global skill installation
```

安装完成后，Skill 位于 Codex 的全局 Skill 目录，默认路径为：

```text
~/.codex/skills/review-push
```

如果设置了 `CODEX_HOME`，则路径为 `$CODEX_HOME/skills/review-push`。

## PromptScript

PromptScript 目前只能安装到项目目录。需要在 PromptScript 项目中使用时，不要传入 `--global`：

```bash
npx skills add nov-ai-story/review-push --yes --agent promptscript
```

## 使用

在需要提交并推送的 Git 仓库中明确调用 `review-push`，并说明本轮需要推送。Skill 会保留无关的本地修改，只提交本次范围内的文件；逻辑变更会先经过审查。
