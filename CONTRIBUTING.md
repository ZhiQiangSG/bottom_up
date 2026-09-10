# Contributing to bottom_up

Please read this guide before your first contribution. It covers setup, workflow, code style, and testing so we all work the same way.

## Table of contents

1. [Code of conduct](#code-of-conduct)
2. [Ways to contribute](#ways-to-contribute)
3. [Prerequisites](#prerequisites)
4. [Quickstart](#quickstart)
5. [Option A — Docker (recommended, closest to CI/prod)](#option-a--docker-recommended-closest-to-ciprod)
6. [Option B — uv native (fast iteration)](#option-b--uv-native-fast-iteration)
7. [Pre-commit (required)](#pre-commit-required)
8. [Branching, commits, and pull requests](#branching-commits-and-pull-requests)
9. [Code style](#code-style)
10. [Testing and type checks](#testing-and-type-checks)
11. [Agentic engineering methodology](#agentic-engineering-methodology)
    - [Agentic tooling — which command when](#agentic-tooling--which-command-when)
12. [Database migrations](#database-migrations)
13. [Docs](#docs)

## Code of conduct

- Give constructive reviews; explain the *why* behind requested changes.

## Ways to contribute

- Report bugs and suggest features via GitHub Issues.
- Pick up an existing issue, fix a bug, or add a test.
- Improve docs (`README.md`, `docs/`, or this file).
- Small PRs are better than big ones. If a change will be large, open an issue first so we can agree on the approach.

New to the codebase? Good first tasks: add/extend a test in `tests/` or `bottom_up/*/tests/`, fix a Ruff warning, or clarify docs.

## Prerequisites

| Tool | Version / notes |
|------|-----------------|
| Git | any recent version |
| Python | `3.14` (see `.python-version`) |
| [uv](https://docs.astral.sh/uv/) | latest; manages deps via `pyproject.toml` + `uv.lock` |
| Docker Desktop + Compose v2 | for Postgres, Redis, Mailpit, Django, Celery |
| [just](https://github.com/casey/just) | optional but recommended; wraps `docker compose` (see `justfile`) |
| pre-commit | installed via `uv` dev group (`pre-commit==4.6.1`) |

Editors must respect `.editorconfig` (UTF-8, LF, final newline, trim trailing whitespace; Python 4 spaces; HTML/CSS/YML/TOML 2 spaces).

> [!IMPORTANT]
> Never commit real secrets. `.env`, `.envs/.production/`, and credentials in `.envs/.local/` (Postgres password, Flower password, etc.) must stay local. If a secret is ever pushed, tell the team immediately so we can rotate it.

## Quickstart

```bash
git clone <your-repo-url> bottom_up
cd bottom_up
```

Then follow **Option A** (Docker) or **Option B** (uv). When in doubt, use **Option A** — it is what CI uses.

## Option A — Docker (recommended, closest to CI/prod)

Services are defined in `docker-compose.local.yml`: `django` (`:8000`), `postgres`, `redis`, `mailpit` (`:8025`), `celeryworker`, `celerybeat`, `flower` (`:5555`). Env comes from `.envs/.local/.django` and `.envs/.local/.postgres`.

```bash
# 1. Build the image
just build
# or: docker compose -f docker-compose.local.yml build

# 2. Start everything in the background
just up
# or: docker compose -f docker-compose.local.yml up -d --remove-orphans

# 3. Run migrations + create an admin
just manage migrate
just manage createsuperuser

# 4. Open the app
# http://127.0.0.1:8000
# Emails (dev): http://127.0.0.1:8025  (Mailpit)
# Celery monitor: http://127.0.0.1:5555 (Flower)
```

Useful commands:

```bash
just logs django        # follow logs (any service name works)
just manage <cmd>       # run manage.py, e.g. just manage shell
just pytest             # run test suite inside Docker
just down               # stop containers
just prune              # stop + delete volumes (wipes local DB!)
```

## Option B — uv native (fast iteration)

Use this for quick edit/test loops. You still need Postgres + Redis running (easiest via Docker), because `config/settings/base.py` reads `POSTGRES_DB/USER/PASSWORD/HOST/PORT` and `REDIS_URL`.

```bash
# 1. Install deps (includes dev group: pytest, ruff, mypy, pre-commit, ...)
uv sync --group dev

# 2. Start only the infra in Docker
docker compose -f docker-compose.local.yml up -d postgres redis mailpit

# 3. Point Django at localhost services, then migrate + run
# PowerShell / zsh example (adjust user/password/db to match .envs/.local/.postgres):
export POSTGRES_HOST=localhost POSTGRES_PORT=5432
export POSTGRES_DB=bottom_up POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres
export REDIS_URL=redis://localhost:6379/0 USE_DOCKER=no
export DJANGO_SETTINGS_MODULE=config.settings.local

uv run python manage.py migrate
uv run python manage.py createsuperuser
uv run python manage.py runserver
```

Notes:

- See canonical runbook in `README.md#troubleshooting-runbook` for `USE_DOCKER`, Postgres/Redis, `username/first_name=None`, `api/docs` 403, Celery root.
- `SECRET_KEY`, `EMAIL_HOST=mailpit`, and cache settings all have safe dev defaults; override via env vars only if needed.

## Pre-commit (required)

We use `pre-commit` (`.pre-commit-config.yaml`): general hygiene checks, `django-upgrade --target-version 6.0`, `ruff-check --fix`, `ruff-format`, `pyproject-fmt`, and `djlint` for Django templates. CI runs the same in the `linter` job.

```bash
uv run pre-commit install
uv run pre-commit run --all-files   # run once before your first PR
```

If a hook rewrites files, re-stage and re-commit. Do not skip hooks with `--no-verify` unless you have a documented reason.

## Branching, commits, and pull requests

`main` is protected. Never push directly — use feature branches + PRs:

```bash
git checkout -b feat/short-description
# or: fix/..., docs/..., chore/...
```

Rules:

1. One branch per issue/feature. Keep PRs small and focused.
2. Sync regularly: `git pull --rebase origin main`.
3. Commit messages: short imperative subject (`Add login rate limit`), blank line, then *why* if non-obvious.
4. Before pushing, run (natively or via Docker):
   ```bash
   uv run pre-commit run --all-files
   uv run mypy bottom_up
   uv run pytest
   # or: just pytest
   ```
5. Open a PR against `main`: the PR template (`.github/PULL_REQUEST_TEMPLATE.md`) loads automatically — fill in Spec, Gates evidence, and Test steps; do not delete unchecked checklist items (`pr-guard` blocks merge until all are checked). Add screenshots for UI changes.
6. CI must be green: `linter` (pre-commit) + `pytest` (Docker build, `makemigrations --check`, `migrate`, `pytest`) + `pr-guard` (see `.github/workflows/pr-guard.yml`). Docs-only changes skip `pytest` via `paths-ignore: docs/**`, but `pr-guard` still runs.
7. Get at least **1 review** from a teammate before merging. Address comments or explain why not. Repo admins: mark `linter`, `pytest`, and `pr-guard` as required status checks on `main` (Settings → Branches) and require 1 review + squash-merge.
8. Merge via GitHub (squash preferred for small PRs). Delete the branch after merge.

## Code style

Configured in `pyproject.toml` + `.editorconfig`:

- **Ruff** (lint + format): single-line imports, rules in `tool.ruff`. Line length handled by the formatter (do not add `# noqa` without reason).
- **djLint** (templates): 2-space indent, `profile = "django"`.
- **django-upgrade**: keep code modern for Django 6.0.
- **Type checks**: `mypy` with `mypy_django_plugin` / `mypy_drf_plugin` (`config.settings.test`).

```bash
uv run ruff check --fix .
uv run ruff format .
uv run mypy bottom_up
```

Follow the existing layout: apps in `bottom_up/` (e.g. `bottom_up/users/`), project config in `config/`, shared templates/static in `bottom_up/templates/` + `bottom_up/static/`.

## Testing and type checks

Test config: `tool.pytest` uses `--ds=config.settings.test --reuse-db --import-mode=importlib`; coverage (`tool.coverage.run`) includes `bottom_up/**`, omits migrations/tests.

```bash
# Fastest (native; needs Postgres/Redis up — see Option B)
uv run pytest
uv run pytest tests/test_merge_production_dotenvs_in_dotenv.py -x -q   # single file

# Docker (CI also runs build + makemigrations --check + migrate)
just pytest
# or: docker compose -f docker-compose.local.yml run --rm django pytest

# Coverage (from README)
uv run coverage run -m pytest
uv run coverage html
uv run open htmlcov/index.html   # macOS; use xdg-open on Linux

# Type checks (from README)
uv run mypy bottom_up
```

Write tests for new behavior (pytest + factory-boy are available). Put app tests next to the app or under `tests/`.

## Agentic engineering methodology

Standard way humans and agents build features: BDD → Types → Plan → TDD → Verify → Secure.

- `feat/` branches: follow the full loop below.
- `fix/`/`docs/`/`chore/` branches: the PR checklist at the end is enough, unless behavior changes — then follow the full loop too.

### 1. BDD — draft the spec (what, not how)

1. Start at `plans/INDEX.md`, open one `plans/domains/<domain>/NN-<feature>.md`, then follow only its `requires_specs`.
2. Each feature file must have: Goal / Non-goals, Actors, Gherkin scenarios (`Given/When/Then`), and acceptance criteria observable via API/UI.
3. Example shape:

```gherkin
Feature: Filtered job discovery
  Scenario: Filter by work type
    Given published jobs exist
    When GET /api/jobs/?work_type=internship
    Then only internships are returned
```

### 2. Type-driven — define the contract (shape)

Define in this order, no logic yet:

1. `bottom_up/<app>/models.py` — Django models with types.
2. `bottom_up/<app>/api/serializers.py` — DRF serializers (the API contract).
3. `bottom_up/<app>/api/views.py` + `config/api_router.py` — routes.
4. OpenAPI via `drf-spectacular` (`api/schema/`, `api/docs/`) — machine-checkable contract.

Gate before planning code:

```bash
uv run mypy bottom_up
```

### 3. Plan — decompose into subtasks

Write a small checklist; one Red-Green-Refactor cycle per item. Get human approval before coding. Example:

```md
- [ ] model + migration (`makemigrations`)
- [ ] serializer + validation
- [ ] view + permissions + router entry
- [ ] celery task if async
- [ ] unit → API → acceptance tests
```

### 4. TDD — tests drive implementation

Per subtask:

1. RED: write a failing `pytest` test from the Gherkin scenario (use `factory-boy`, `pytest-django`, `config.settings.test`).
2. GREEN: minimal code to pass.
3. REFACTOR: clean up, then run in this order:

```bash
uv run ruff check --fix .
uv run ruff format .
uv run mypy bottom_up
uv run pytest
```

Put app tests next to the app (`bottom_up/<app>/tests/`) or under `tests/`. See `bottom_up/users/tests/` for the `test_views` / `test_urls` / `test_openapi` pattern.

### 5. Verify — acceptance against the spec

- Every Gherkin scenario maps to at least one API test. No unmapped scenario ships.
- Run the CI-equivalent gates:

```bash
just pytest
just manage makemigrations --check
uv run coverage run -m pytest && uv run coverage html
```

If a scenario fails, go back to step 4 — do not proceed to merge.

### 6. Secure at every gate (not just release)

```bash
uv run pre-commit run --all-files
```

- Ruff `S`/`BLE`/`DJ` rules catch common security bugs; do not add `# noqa` without a reason.
- Never commit `.env`, `.envs/.production/`, or credentials in `.envs/.local/`.
- CI must be green: `linter` (pre-commit) + `pytest` (Docker build → `makemigrations --check` → `migrate` → `pytest`).

### Agentic tooling — which command when

Full flags live in `.opencode/README.md` (canonical index); skills are source of truth on conflict. Every command supports `--dry-run` (plan only, zero writes).

| Phase | Type | Runner |
|---|---|---|
| Plan BDD + Types + Plan | `/spec-feature spec=plans/domains/<domain>/NN-<feature>.md` | main agent, read-only, stops for approval |
| Build TDD + Verify + Secure | `/implement-feature spec=... tasks="<checklist>"` | `feature-builder` subagent |
| Gates | `/qa` | `qa-reviewer` — check-only; `--fix` allows Ruff fixes only |
| Debt / vulns / full | `/code-audit`, `/security-audit`, `/audit-all` | `code-auditor` / `security-auditor`, read-only stdout |
| Docs | `/update-docs` | `docs-maintainer` |

Rules: skill wins on conflict; never commit/push/prune volumes unless explicitly asked; secrets guard fails on staged `.env`, `.envs/.production/`, or credentials in `.envs/.local/` (paths only, never contents).

### Agent prompt snippet

Copy-paste when delegating to an agent:

```text
Follow CONTRIBUTING.md Agentic engineering methodology for <feature>.
Spec: plans/domains/<domain>/NN-<feature>.md (plus its requires_specs only).
Loop per subtask: failing pytest from Gherkin → minimal fix → ruff + mypy + pytest.
Gates before done: mypy clean, just pytest green, makemigrations --check, pre-commit clean.
```

### PR checklist

- [ ] Spec linked (`plans/domains/.../NN-*.md`), scenarios listed
- [ ] Types: `mypy bottom_up` clean; OpenAPI updated if API changed
- [ ] TDD: failing-first tests added; `just pytest` green
- [ ] Verify: every scenario maps to a test; migrations committed
- [ ] Secure: `pre-commit run --all-files` clean; no secrets committed
- [ ] Small PR, 1 review, squash-merge

## Database migrations

- After changing models: `just manage makemigrations` (or `uv run python manage.py makemigrations`).
- Commit the migration files.
- CI enforces `python manage.py makemigrations --check` — it fails if you forgot a migration.

## Docs

- User/dev docs live in `docs/` (Sphinx). Preview with `docker-compose.docs.yml`.
- Update `README.md` / `docs/*.rst` when you add settings, commands, or services.
- Keep this `CONTRIBUTING.md` up to date when the workflow changes.
