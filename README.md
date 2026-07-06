# My Skills

Personal agent skills maintained as source-of-truth files.

## Skills

- `codex-review-loop` - Run an agent-neutral Codex CLI review loop for code and plan reviews.

## Install Locations

Use this repository as the editable source. The canonical local install location is:

```text
%USERPROFILE%\.agents\skills
```

Pi can read the shared `.agents\skills` location, so do not maintain a separate private copy under `.pi\skills` unless a specific test requires it.

Claude Code can use its own skill directory when needed:

```text
%USERPROFILE%\.claude\skills
```

## Conventions

- Keep detailed agent instructions in each skill's `SKILL.md`.
- Keep UI metadata in `agents/openai.yaml` when useful.
- Do not add per-skill `README.md` files; they tend to drift from `SKILL.md`.
- Treat generated review logs and process state as local artifacts, not committed source, unless an audit record is explicitly required.
