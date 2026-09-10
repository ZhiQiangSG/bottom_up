---
description: Read-only tech-debt and code-smell audit (complexity, duplication, typing, best practices, arch drift) via the code-auditor subagent
agent: code-auditor
subtask: true
---

Load and execute the `code-audit` skill using the `skill` tool.

Targets:
- @AGENTS.md
- @plans/ARCHITECTURE.md
- @config/api_router.py
- @pyproject.toml

Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "code-audit" })` and follow it exactly.
2. Parse `$ARGUMENTS` for optional flags (all optional, defaults shown):
   - `scope=debt|smells|arch|all` (default `all`) — finding category.
   - `path=bottom_up/<app>/` — single-app scope; must stay inside `bottom_up/`, `config/`, or `tests/`.
   - `--quick` — smells only (same as `scope=smells`).
   - `--dry-run` — print the audit plan, make zero `write`/`edit` calls.
   - No args → full read-only audit.
3. Audit via `read`/`glob`/`grep` in order: smells → debt/best-practices → arch drift (report-only). Never run `pytest`, `coverage`, migrations, or `pre-commit`.
4. On finding, report Category + Severity + Effort (S/M/L) + file:line + fix hint with handoff to `/implement-feature`. Reject `--fix` (direct to `/qa --fix` or `/implement-feature`).
5. Report to stdout only. Summarize counts per category/severity, top-5 paydown list, and what was skipped (`scope=`/`--quick`/`path=`). Do NOT commit/push unless explicitly asked.

Usage:
- `/code-audit`
- `/code-audit scope=smells path=bottom_up/users/`
- `/code-audit scope=arch`
- `/code-audit --quick`
- `/code-audit --dry-run`
