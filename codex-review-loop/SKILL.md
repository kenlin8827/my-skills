---
name: codex-review-loop
description: Use OpenAI Codex CLI as a second-opinion reviewer in an iterative fix-resume loop. Auto-detects whether the user wants code review (files, git diff) or plan/design review (PRD, architecture, API spec, flow) from the task description. Drives a P0-P2 issue cycle until both sides reach "no issues" consensus, then outputs a summary. Triggers on "review with codex", "用 codex 评审", "codex review loop", "代码评审", "方案评审", "PRD 评审", "让 codex review".
---

# Codex Review Loop

Run an iterative review cycle with `codex` (OpenAI Codex CLI) as a second-opinion reviewer. Works for both **code** and **plan/design** reviews; you pick the mode from the user's task.

# 角色

- **You (Claude Code)** — primary developer / architect. Decide, fix code or revise plans.
- **codex (Codex CLI)** — reviewer. Outputs a P0/P1/P2 issue list per round.

# 启动

1. **判断 review 类型** (from the user's task):
   - User provides **code / files / paths / git diff / build artifacts** → `code` mode
   - User provides **plan / design / PRD / API spec / architecture / flow** → `plan` mode
   - Both → split into two independent reviews (plan first, then code), each with a new session
   - Unclear → ask user **once**, then proceed

2. Collect context to pass to codex:
   - Working directory **absolute path**
   - `git branch` + `HEAD` SHA
   - Target scope (code: paths; plan: doc path or inline text)
   - Build / test commands (optional, useful in code mode)

3. Start codex session:

   **code mode**:
   ```bash
   codex exec "请对 <绝对路径> 做代码 review，输出 P0/P1/P2 优先级 BUG 清单。
   格式: <文件:行号> | <问题> | <建议修复>。
   分支: <branch>, HEAD: <sha>。"
   ```

   **plan mode**:
   ```bash
   codex exec "请审查以下方案，输出 P0/P1/P2 优先级问题清单。
   格式: <章节/段落> | <问题> | <建议修改>。
   关注点: 需求遗漏、安全/性能/扩展性、矛盾点、可行性。

   分支: <branch>, HEAD: <sha>。

   方案:
   \`\`\`
   <paste content or cat <plan path>>
   \`\`\`"
   ```

4. **Record `session_id`** from codex output. Reuse it for every round.

# 每轮流程

1. Parse codex output → issue list
2. Per item, classify:
   - **Real issue** → fix code (commit) / revise plan (bump to v2)
   - **False positive** → reply in codex session: `"<X> 不是问题, 理由: <Y>"`, ask for re-review
   - **P3 / preference** → mark "已知忽略", skip
3. After changes, resume codex:
   ```bash
   codex resume <session_id> "已修改 (commit <sha> / plan v2), 请重新 review。"
   ```

# 终止条件 (any one)

- codex reports no new P0/P1 for 2 consecutive rounds
- P0/P1 = 0, remaining P2s both sides agree to defer
- Same issue disputed for 3 rounds → mark "未决", stop

# 最终报告

```
## Review 总结
- codex session: <id>
- review 类型: code | plan | code+plan
- 轮次: N
- 修订: P0=X, P1=Y, P2=Z
- 误报: A / 已知忽略: B / 未决: C
- 产物: <commit sha> | <plan 路径 vN>
```

# 注意

- `session_id` 全程不变; 丢失即重建, 前功尽弃
- code ↔ plan 切换必须**新开 session** (上下文不同)
- 别为让 codex 满意改非问题的东西, 保持主见
- 每条修改后自测/自检, 别盲从 codex 建议
- plan 模式建议先把方案固化到 `docs/plan-vN.md`, 方便 cat 引用 + diff 对比
