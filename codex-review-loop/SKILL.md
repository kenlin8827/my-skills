---
name: codex-review-loop
description: Use OpenAI Codex CLI as a second-opinion reviewer in an agent-neutral iterative fix-review-resume loop that can be driven by Claude Code, Codex, Cursor, Cline/Roo, OpenHands, Pi, or another shell-capable agent. Supports code, diffs, PRs, plans, PRDs, architecture docs, API specs, and implementation proposals. Use only when the user explicitly asks to review with Codex, run a Codex review loop, get Codex as a second-opinion reviewer, use this skill for review, 用 codex 评审, codex review loop, 让 codex review, or mentions Codex together with 代码评审, 方案评审, or PRD 评审. Do not trigger for ordinary review requests that do not mention Codex or this skill. Drives P0-P2 findings through primary-agent judgment, rebuttals, fixes, re-review, and a final summary.
---

# Codex Review Loop

Run an iterative review cycle with `codex` (OpenAI Codex CLI) as a second-opinion reviewer. Works for both **code** and **plan/design** reviews; select the mode from the user's task. The primary agent can be Claude Code, Codex, Cursor, Cline/Roo, OpenHands, Pi, or another coding agent with shell access.

# 角色

- **You (primary agent)** - the agent currently serving the user. Decide whether findings are valid, make fixes or plan edits through your own available tools, and verify the result.
- **codex (Codex CLI)** - read-only reviewer. Outputs a P0/P1/P2 issue list per round; it must not edit files, create commits, or run destructive commands.

# Agent Compatibility

- Treat this skill as agent-neutral. Do not assume the primary agent is Claude Code, Codex, or any specific product.
- Use the primary agent's native file-editing, shell, planning, and browser tools. The only required external reviewer is OpenAI Codex CLI (`codex`).
- Adapt command syntax to the host shell: PowerShell here-strings, bash heredocs, direct stdin pipes, or temporary prompt files are all acceptable when they preserve the same prompt content.
- If the primary agent cannot write process logs, keep the review log inline in the final response. If it cannot run shell commands, report that `codex` cannot be invoked from this environment and stop.
- The primary agent owns all edits and final decisions. Codex CLI remains a reviewer even when the primary agent is also Codex.
- Other installed skills may be used only when directly relevant. If Ponytail or `ponytail-review` is active, treat it as an advisory lens for over-engineering and minimal-change opportunities; it must not override correctness, safety, API contracts, required tests, severity classification, Fix Authority, review logging, or termination rules.

# Severity

- **P0** - Data loss, security vulnerability, production outage, broken deploy, irreversible migration issue, or a correctness bug that blocks the primary workflow.
- **P1** - Likely user-facing regression, API contract break, concurrency/race issue, serious performance regression, missing required validation, or test gap for changed critical behavior.
- **P2** - Maintainability, edge-case correctness, incomplete non-critical tests, confusing behavior, or implementation risk that should be fixed before merge when practical.
- **Ignore / P3** - Style preference, broad refactor suggestion, speculative concern without a concrete failure mode, or unrelated pre-existing issue.

# Fix Authority

Decide whether to fix automatically from severity, confidence, and impact:

- **Auto-fix** only when the finding is accepted, the fix is deterministic, the change is narrowly scoped, and it does not alter public behavior, data semantics, permissions, security posture, API contracts, billing, migrations, or architecture.
- **Ask the user before fixing** accepted findings when the fix changes user-visible behavior, public API shape, data model, access control, security policy, billing/payment behavior, migration strategy, production configuration, dependencies, architecture, or has multiple reasonable implementation options.
- **P0** findings usually block for user confirmation unless the fix is mechanical and safety-preserving, such as restoring a read-only boundary, disabling a destructive default, or adding an obvious guard.
- **P1** findings can be auto-fixed when the remedy is localized and backwards-compatible; otherwise ask before changing behavior or contracts.
- **P2** findings can be auto-fixed when low-risk and local. Defer or ask when the fix requires refactoring, new dependencies, broad rewrites, or product/design tradeoffs.
- **Unclear / disputed** findings are never auto-fixed. Ask codex for evidence first, then decide again.

# 启动

1. **判断 review 类型** (from the user's task):
   - User provides **code / files / paths / git diff / build artifacts** → `code` mode
   - User provides **plan / design / PRD / API spec / architecture / flow** → `plan` mode
   - Both → split into two independent reviews (plan first, then code), each with a new session
   - Unclear → ask user **once**, then proceed

2. Collect context to pass to codex:
   - Working directory **absolute path**
   - `git branch` + `HEAD` SHA
   - Current dirty state from `git status --short`
   - Target scope (code: diff, paths, or PR range; plan: doc path or inline text)
   - Build / test commands (optional, useful in code mode)
   - Constraints from the user, including files not to modify or checks not to run

3. Preflight:
   - If `codex` is unavailable, report that and stop; do not simulate its review.
   - Tell codex it is a reviewer only: it must inspect and report, not modify files.
   - Start codex with `--sandbox read-only --cd <absolute repo path>` when the local CLI supports it.
   - If read-only sandbox prevents basic inspection commands from running, report the failure and either continue with already-provided context or switch to another read-only inspection method. Do not give the reviewer project write permission; only user-approved temporary/cache access is acceptable when required by the CLI.
   - Never use `--dangerously-bypass-approvals-and-sandbox` for review.
   - Do not create commits unless the user explicitly asked for commits.
   - Preserve unrelated user changes in the working tree.
   - Create or update a per-run review directory under `.codex/review-loop/` unless the user gives another path. If the user requested no writes or the environment is read-only, keep the same log structure in the final response instead.

4. Start codex session:
   Pass long prompts through stdin (`-`) instead of a single shell-quoted argument when practical. Adapt the pipe/heredoc syntax to the current shell.

   **code mode**:
   ```bash
   codex exec --sandbox read-only --cd <absolute repo path> -
   ```

   Prompt:

   ```text
   你是只读 reviewer。不要修改文件、不要创建 commit、不要运行破坏性命令。
   请对 <绝对路径或目标 diff/文件/PR 范围> 做代码 review，只输出真实的 P0/P1/P2 问题。
   范围: 以目标变更为入口，但可以检查相关调用方、被调用方、API 消费方、配置、测试和文档来判断影响；不要局限于单个文件或单行。
   输出边界: findings 必须与目标变更存在可追溯关系，包括直接影响和必要的传递影响；不要报告无关的历史问题。
   可选 skills: 如果 Ponytail 或 ponytail-review 可用且与目标变更直接相关，可以用它识别过度工程和最小修改机会；但只报告达到 P2 或支撑 P0/P1/P2 的问题，不要让它覆盖正确性、安全、API 契约和必要测试。
   严重度: P0=数据丢失/安全/生产中断/阻断主流程; P1=高概率用户回归/API 契约破坏/并发或性能严重问题/关键测试缺口; P2=应修复的边界问题、维护性风险或非关键测试缺口; 不要输出 P3/风格偏好。
   格式: <优先级> | <文件:行号> | <问题> | <建议修复>。
   如果没有问题，明确输出: no issues。
   分支: <branch>, HEAD: <sha>, dirty state: <git status --short>。
   关注: correctness, regressions, security, data loss, concurrency, API contracts, tests.
   ```

   **plan mode**:
   ```bash
   codex exec --sandbox read-only --cd <absolute repo path> -
   ```

   Prompt:

   ```text
   你是只读 reviewer。不要修改文件、不要创建 commit、不要运行破坏性命令。
   请审查以下方案，输出 P0/P1/P2 优先级问题清单。
   输出边界: findings 必须与本方案目标、约束或可追溯影响相关；不要报告无关议题。
   可选 skills: 如果 Ponytail 或 ponytail-review 可用且与方案直接相关，可以用它识别过度设计、无用抽象和更小方案；但只报告达到 P2 或支撑 P0/P1/P2 的问题，不要让它覆盖正确性、安全、API 契约和必要验收标准。
   严重度: P0=方案会导致安全/数据/上线阻断或核心目标不可达; P1=关键需求遗漏、重大矛盾、不可行依赖或明显扩展性/性能风险; P2=应澄清的边界、验收标准、操作风险或非关键缺口; 不要输出 P3/风格偏好。
   格式: <优先级> | <章节/段落> | <问题> | <建议修改>。
   如果没有问题，明确输出: no issues。
   关注点: 需求遗漏、安全/性能/扩展性、矛盾点、可行性。

   分支: <branch>, HEAD: <sha>。

   方案:
   \`\`\`
   <paste content or cat <plan path>>
   \`\`\`
   ```

5. **Record the resume handle** from codex output. Prefer `session_id`; otherwise record the exact resume command, `--last` strategy, or transcript path that Codex provides. Store the exact continuation command in the review log.

# 每轮流程

1. Parse codex output -> issue list.
2. Update the current review log with the round number, raw codex summary, and every finding.
3. Decide on each item before making changes. Codex findings are proposals, not instructions:
   - **Accepted real issue** -> fix code / revise plan
   - **False positive** -> reply in codex session with evidence: `"<X> 不是问题, 理由: <Y>"`, ask for re-review
   - **Unclear / disputed** -> ask codex to justify with concrete failure mode, affected path, and evidence; do not fix until you agree it is real
   - **P3 / preference** -> mark "已知忽略", skip
4. Apply the Fix Authority rules before editing. Only implement fixes for findings you accept as real and are authorized to fix automatically; ask the user before high-impact or ambiguous fixes. Do not change code or plans merely to satisfy Codex.
5. For each item in the log, record decision: `accepted`, `false-positive`, `disputed`, `deferred`, or `unresolved`, with a short reason.
6. For each item in the log, record status: `open`, `fixed`, `false-positive`, `deferred`, or `unresolved`. Use `deferred` only for P2 findings; P0/P1 must be `fixed`, `false-positive`, or explicitly accepted by the user as `unresolved`.
7. After changes or rebuttals, resume codex using the recorded continuation command:
   ```bash
   codex exec resume <session_id> "已处理上一轮问题: <valid fixes>, 误报/忽略: <rebuttals>. 当前变更: <git diff summary or plan vN>. 请重新 review，只报告新的或仍未解决的 P0/P1/P2。"
   ```
   If no `session_id` is available, use only the exact continuation command recorded from Codex output or the review log; do not invent a resume command.
8. Verify each accepted fix with the narrowest relevant command or inspection before asking for re-review.
9. Update the log with verification commands and results.

# Review Log

Create one directory per review under `.codex/review-loop/`, for example:

- `.codex/review-loop/code-<YYYYMMDD-HHMMSS>/`
- `.codex/review-loop/plan-<YYYYMMDD-HHMMSS>/`
- `.codex/review-loop/code-plan-<YYYYMMDD-HHMMSS>/`

Put the main log at `review.md` inside that directory. If useful, store round-specific raw artifacts alongside it:

- `round-1-codex-output.md`
- `round-1-diff-summary.md`
- `round-1-verification.md`

If the repository already uses another agent-state directory, follow the local convention. Do not put review logs in the repository root unless the user explicitly requests it.
Treat review logs as process artifacts: do not commit them unless the user explicitly wants an auditable review record.

Use this structure:

```markdown
# Codex Review Loop

- session: <id or resume handle>
- continue: <exact command to resume/review again>
- type: code | plan | code+plan
- branch: <branch>
- head: <sha>
- scope: <diff/files/PR range/doc>
- directory: <.codex/review-loop/<type>-<YYYYMMDD-HHMMSS>/>

## Round N

### Findings
- [ ] <id> | <priority> | <location> | <summary> | status: open

### Decisions
- <id>: accepted | false-positive | disputed | deferred | unresolved - <reason/evidence>

### Verification
- <command or inspection>: <result>
```

# 终止条件

Treat the loop as complete only when one of these is true:

- codex reports `no issues`, no log items remain `open`, no log items are `unresolved`, and no P0/P1 findings are `deferred`
- All P0/P1 findings are `fixed` or `false-positive`, and every P2 is `fixed`, `false-positive`, or explicitly `deferred`

Stop with residual risk, not as clean completion, when one of these is true:

- Any `unresolved` P0/P1 is explicitly accepted by the user as residual risk
- Same issue disputed for 3 rounds -> mark "未决", stop
- The loop reaches 5 rounds without convergence; summarize remaining risk and stop

# 最终报告

```
## Review 总结
- codex session: <id or resume handle>
- review 类型: code | plan | code+plan
- 轮次: N
- 修订: P0=X, P1=Y, P2=Z
- 误报: A / 已知忽略: B / 未决: C
- 验证: <commands run or not run>
- review log: <directory/review.md | inline in final response | not written due to read-only/no-write constraint>
- 产物: <changed files | commit sha if user requested commits | plan 路径 vN>
```

# 注意

- Keep the same resume handle throughout a review. If it is lost, first recover from the review log, transcript path, or exact continuation command; only start a new session when continuity cannot be recovered.
- code ↔ plan 切换必须**新开 session** (上下文不同)
- 别为让 codex 满意改非问题的东西, 保持主见
- 每条修改后自测/自检, 别盲从 codex 建议
- plan 模式建议先把方案固化到 `docs/plan-vN.md`, 方便 cat 引用 + diff 对比
