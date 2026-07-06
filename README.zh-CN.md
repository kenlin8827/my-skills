# 我的技能库

个人 agent skills 的源码仓库，作为本地安装副本的维护源。

## 技能

- `codex-review-loop` - 使用 Codex CLI 作为只读 second-opinion reviewer，驱动 agent-neutral 的代码/方案评审循环。

## 安装位置

把本仓库作为可编辑源码。推荐的本地 canonical 安装位置是：

```text
%USERPROFILE%\.agents\skills
```

Pi 可以读取共享的 `.agents\skills` 位置；除非做特定测试，不要再维护 `.pi\skills` 下的私有副本。

Claude Code 如有需要也可以使用自己的 skill 目录：

```text
%USERPROFILE%\.claude\skills
```

## 约定

- 详细 agent 行为说明放在各 skill 的 `SKILL.md`。
- 如需 UI 元数据，放在 `agents/openai.yaml`。
- 不在单个 skill 目录下添加 `README.md`，避免和 `SKILL.md` 漂移。
- 生成的 review log 和过程状态默认视为本地产物，不提交；除非明确需要审计记录。
