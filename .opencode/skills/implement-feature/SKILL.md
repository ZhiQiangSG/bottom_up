---
name: implement-feature
description: Implement an approved feature spec via TDD Red-Green-Refactor, then verify and secure. Requires approved /spec-feature output.
---

# Implement Feature (Building: TDD + Verify + Secure)

Implement one approved feature spec using executable tests. Requires the approved `/spec-feature` output (spec path + subtask checklist) before starting.

Source of truth: `CONTRIBUTING.md#agentic-engineering-methodology` and `AGENTS.md## Workflow`. This skill operationalizes steps 4-6 only. Link to those files — do not duplicate them.

## When to use

Use when the user invokes `/implement-feature` (runs in the `feature-builder` subagent). Load via `skill({ name: "implement-feature" })`, then follow this workflow.

## Prerequisites (check FIRST)

- `spec=plans/domains/<domain>/NN-<feature>.md` present in `$ARGUMENTS`, plus the approved subtask checklist (`tasks=`). If either is missing, stop and ask — do not guess the spec.
- Read ONLY that spec plus its `requires_specs`. Do not load all specs eagerly.
- Confirm the type contract from `/spec-feature` (models → serializers → views/router → OpenAPI). If the contract is missing, stop and direct the user back to `/spec-feature`.

## Workflow

### 1. TDD loop (per subtask)

For each checklist item, one Red-Green-Refactor cycle:

1. RED: write a failing `pytest` test derived from the Gherkin scenario (use `factory-boy`, `pytest-django`, `config.settings.test`). Tests live next to the app (`bottom_up/<app>/tests/`) or under `tests/`. Follow the `bottom_up/users/tests/` `test_views` / `test_urls` / `test_openapi` pattern.
2. GREEN: minimal implementation to pass. Typing: annotate Tier1 boundaries (`models`, `services`, `tasks`, `api.views`, `api.serializers`) per `AGENTS.md Conventions`. Docstrings: Google-style docstring on every new/changed public module/class/function/method (one-line summary + `Args:`/`Returns:`/`Raises:` where applicable). Private `_`-prefixed helpers exempt unless non-obvious. Example:

  ```python
  def get_absolute_url(self) -> str:
      """Get URL for user's detail view.

      Returns:
          str: URL for user detail.
      """
  ```
3. REFACTOR: clean up, then run in this order:

```bash
uv run ruff check --fix .
uv run ruff format .
uv run mypy bottom_up
uv run pytest
```

Single-file fast loop: `uv run pytest tests/test_merge_production_dotenvs_in_dotenv.py -x -q` (replace with the target file).

### 2. Verify (acceptance against the spec)

- Every Gherkin scenario maps to at least one API test. No unmapped scenario ships. If a scenario fails, return to the TDD loop — do not proceed to merge.
- For CI-equivalent gates, load `skill({ name: "qa" })` and follow it exactly (native by default; pass through `--docker` / `--quick` if present in `$ARGUMENTS`). Do not re-list gate commands here — `qa` is the single source of truth for gate order and commands.
- Commit migration files when models change (`just manage makemigrations`; native: `uv run python manage.py makemigrations`).

### 3. Secure (at this gate and every gate)

- Follow `skill({ name: "qa" })` secrets guard + pre-commit gate exactly. Do not duplicate gate commands here.
- Ruff `S`/`BLE`/`DJ` rules catch common security bugs; do not add `# noqa` without a reason.
- Never commit `.env`, `.envs/.production/`, or credentials in `.envs/.local/`.
- CI must be green: `linter` (pre-commit) + `pytest` (Docker build → `makemigrations --check` → `migrate` → `pytest`).

### 4. Report

Summarize per subtask: tests added (file paths), scenarios covered, gates run (`ruff`, `mypy`, `pytest`/`just pytest`, `makemigrations --check`, `pre-commit`), and any intentionally skipped items. Do NOT commit, push, or open a PR unless explicitly asked.

## Flags ($ARGUMENTS)

- `spec=plans/domains/<domain>/NN-<feature>.md` — required, single-file scope.
- `tasks=<approved-checklist>` — required; paste the approved `/spec-feature` checklist.
- `--dry-run` — print the subtask plan and first failing test sketch, make zero `write`/`edit` calls.
- `--docker` / `--quick` — passed through to the `qa` skill for the Verify + Secure gates.

## Boundaries

- NEVER invent endpoints, env vars, ports, or versions — read them.
- NEVER commit secrets (`.env`, `.envs/.production/`, credentials in `.envs/.local/`).
- NEVER run destructive commands (`just prune`, `down -v`, `rm -rf`, `push`) unless explicitly asked.
- NEVER break `toctree` refs, `:ref:` targets, or spec `requires_specs` links.
- Keep PRs small (one `feat/` branch per feature); 1 review + squash-merge per `CONTRIBUTING.md`.
