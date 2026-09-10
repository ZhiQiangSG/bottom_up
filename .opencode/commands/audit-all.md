---
description: Fan out to code-audit + security-audit + qa (read-only) and merge stdout reports. No fixes, no report files.
subtask: false
---

Load and merge the existing auditors. No new tools, no fixes at this layer.

Targets:
- @AGENTS.md
- @CONTRIBUTING.md
- @pyproject.toml
- @justfile
- @.pre-commit-config.yaml
- @.github/workflows/ci.yml
- @config/api_router.py

Context overrides: $ARGUMENTS

Instructions:

1. Parse `$ARGUMENTS` (all optional, defaults shown):
   - `--quick` — pass through: `/code-audit --quick` (smells only) + `/security-audit --quick` (Gates A+B+D, skip pip-audit) + `/qa --quick` (skip coverage + pre-commit).
   - `--docker` — pass to `/qa` only (`just pytest`, `just manage makemigrations --check`); audits always run native read-only.
   - `path=bottom_up/<app>/` — pass to `/code-audit` only; must stay inside `bottom_up/`, `config/`, or `tests/`.
   - `scope=debt|smells|arch|all` (default `all`) — pass to `/code-audit` only.
   - `--dry-run` — print the fan-out plan and stop. Make zero `write`/`edit` calls, invoke zero subagents, run zero gate commands.
   - No args → full read-only merge (`code-audit scope=all` + `security-audit --full` + `qa` full native check-only).
2. Fan out in order via subagents (stop-and-report on `--dry-run` before this step):
   1. `code-auditor` via `/code-audit` skill — smells → debt/best-practices → arch drift. Never `pytest`/`coverage`/`pre-commit`.
   2. `security-auditor` via `/security-audit` skill — Gates A→E in order. Never upgrade deps, never `pip-audit --fix`.
   3. `qa-reviewer` via `/qa` skill — `ruff check` → `ruff format --check` → `mypy bottom_up` → `makemigrations --check` → `pytest` → `coverage` (skip if `--quick`) → `pre-commit` (skip if `--quick`).
3. Enforce at this layer (no delegation of judgment):
   - Check-only: reject `--fix` here; direct to `/qa --fix` (ruff-only) or `/implement-feature` (refactors/vulns).
   - Secrets guard: fail if `.env`, `.envs/.production/`, or credentials in `.envs/.local/` are staged/modified. Never print secret values.
   - Never commit, push, prune volumes (`just prune`, `down -v`), or run destructive commands.
   - Never write report files — stdout only.
4. Merge to stdout:
   - Section 1 `code-audit`: counts per category/severity + top-5 paydown (severity × effort, small first).
   - Section 2 `security-audit`: counts per severity + top-3 next actions (redacted evidence).
   - Section 3 `qa`: per-gate pass/fail + rerun command, mapped to CI (`linter` / `pytest` / `pr-guard`).
   - Footer: what was skipped (`--quick` / `--docker` / `path=` / `scope=`), combined next-action list (qa failures first, then High vulns, then top debt), handoff hints (`/implement-feature spec=...` or `/qa --fix`).
   - Do NOT commit/push unless explicitly asked.

Usage:
- `/audit-all`
- `/audit-all --quick`
- `/audit-all --docker`
- `/audit-all path=bottom_up/users/ scope=smells`
- `/audit-all --dry-run`
