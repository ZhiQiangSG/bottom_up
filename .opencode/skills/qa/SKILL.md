---
name: qa
description: Run ordered quality gates (lint to typecheck to migrations to test to coverage to secure) native-first, Docker opt-in. Check-only by default.
---

# QA (Verify + Secure Gates)

Run the repo quality gates in CI-parity order. Check-only by default — no code fixes unless `--fix` is passed.

Source of truth: `CONTRIBUTING.md#agentic-engineering-methodology` (Verify + Secure) and `AGENTS.md## Workflow`. This skill operationalizes the Verify + Secure steps only. Link to those files — do not duplicate them.

## When to use

Use when the user invokes `/qa` (runs in the `qa-reviewer` subagent). Load via `skill({ name: "qa" })`, then follow this workflow.

## Workflow

### 1. Parse flags (check FIRST)

From `$ARGUMENTS`:

- `--docker` — run tests/migrations inside Docker (`just pytest`, `just manage makemigrations --check`). Default is native (`uv run pytest`, `uv run python manage.py makemigrations --check`).
- `--quick` — steps 1-4 only (skip coverage + pre-commit). Default is full (steps 1-6).
- `--fix` — allow `uv run ruff check --fix .`. Without it, run `uv run ruff check .` check-only and never fix.
- `--dry-run` — print the gate plan and make zero `write`/`edit`/`apply_patch` calls and run zero fixing commands.

Strict dry-run gate: if `--dry-run` is present, stop after printing the plan. No file writes, no `ruff --fix`, no commits.

### 2. Gates (in order, stop-and-report on first failure)

Native default; substitute `--docker` variants when flagged:

```bash
uv run ruff check .               # --fix variant: uv run ruff check --fix .
uv run ruff format --check .      # --fix variant: uv run ruff format .
uv run mypy bottom_up
uv run python manage.py makemigrations --check   # --docker: just manage makemigrations --check
uv run pytest -q                  # --docker: just pytest
uv run coverage run -m pytest && uv run coverage html   # skip if --quick
uv run pre-commit run --all-files  # skip if --quick
```

Notes:

- Gate order is `lint → typecheck → test` per `AGENTS.md Commands (verify order)`. Do not reorder.
- Coverage scope is `bottom_up/**` only (see `tool.coverage.run` in `pyproject.toml`).
- CI parity: `linter` = pre-commit; `pytest` = Docker build → `makemigrations --check` → `migrate` → `pytest` (see `.github/workflows/ci.yml`).

### 3. Secrets guard (every run)

- Never read or print secret values. Check status only: `git status --short` + `git diff --stat`.
- Fail the run if `.env`, `.envs/.production/`, or credentials in `.envs/.local/` are staged or modified. Report the paths, do not show contents.

### 4. Report

Summarize per gate: pass/fail + rerun command. Map each gate to CI (`linter` / `pytest` / `pr-guard`).

On failure: report file:line + gate log excerpt and the minimal rerun command. Do NOT auto-fix unless `--fix` was passed. Do NOT commit, push, or open a PR unless explicitly asked.

## Flags ($ARGUMENTS)

- `--docker` — Docker CI-parity path (`just pytest`, `just manage makemigrations --check`).
- `--quick` — fast loop (lint + typecheck + migrations-check + pytest only).
- `--fix` — allow `ruff check --fix` / `ruff format`. Without it, check-only.
- `--dry-run` — print the gate plan, make zero `write`/`edit` calls, run zero fixing commands.

## Boundaries

- NEVER invent endpoints, env vars, ports, or versions — read them.
- NEVER commit secrets (`.env`, `.envs/.production/`, credentials in `.envs/.local/`).
- NEVER run destructive commands (`just prune`, `down -v`, `rm -rf`, `push`) unless explicitly asked.
- NEVER break `toctree` refs, `:ref:` targets, or spec `requires_specs` links.
- Without `--fix`, NEVER call `write`, `edit`, or `apply_patch`. With `--fix`, only Ruff via `bash` (`uv run ruff check --fix .`, `uv run ruff format .`) may touch files — never direct `write`/`edit`.
