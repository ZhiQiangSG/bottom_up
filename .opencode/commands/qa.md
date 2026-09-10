---
description: Run ordered QA gates (lint to typecheck to test to secure) via the qa-reviewer subagent
agent: qa-reviewer
subtask: true
---

Load and execute the `qa` skill using the `skill` tool.

Targets:
- @pyproject.toml
- @justfile
- @.pre-commit-config.yaml
- @.github/workflows/ci.yml

Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "qa" })` and follow it exactly.
2. Parse `$ARGUMENTS` for optional flags (all optional, defaults shown):
   - `--quick` — fast loop (lint + typecheck + migrations-check + pytest only).
   - `--docker` — Docker CI-parity path (`just pytest`, `just manage makemigrations --check`); default is native (`uv run pytest`).
   - `--fix` — allow `ruff check --fix` / `ruff format`; default is check-only (zero writes).
   - `--dry-run` — print the gate plan, make zero `write`/`edit` calls, run zero fixing commands.
   - No args → full native check-only run (all gates, coverage + pre-commit included).
3. Run gates in order: `ruff check` → `ruff format --check` → `mypy bottom_up` → `makemigrations --check` → `pytest` → `coverage` (unless `--quick`) → `pre-commit` (unless `--quick`), plus the secrets guard.
4. On failure, report file:line + gate log + minimal rerun command mapped to CI (`linter` / `pytest` / `pr-guard`). Do NOT auto-fix without `--fix`.
5. Summarize per-gate pass/fail and what was skipped (`--quick`) or substituted (`--docker`). Do NOT commit/push unless explicitly asked.

Usage:
- `/qa`
- `/qa --quick`
- `/qa --docker`
- `/qa --fix`
- `/qa --dry-run`
