# AGENTS.md — bottom_up

Cookiecutter Django (Django 6 + DRF + Celery + Redis + Postgres). Python `3.14` (`.python-version`), deps via `pyproject.toml` + `uv.lock`. Never use bare `python`/`pip` — always `uv run` or `just`.

## Layout

- Apps: `bottom_up/` (e.g. `bottom_up/users/`). Project config: `config/` (`settings/`, `urls.py`, `api_router.py`, `celery_app.py`). Shared templates/static: `bottom_up/templates/`, `bottom_up/static/`.
- API at `api/` (`config/api_router.py`); schema/docs at `api/schema/`, `api/docs/`.
- Settings: `config/settings/{local,test,production}.py` all import `base.py`. Tests always use `config.settings.test` (already in pytest `addopts`).

## Commands (verify order: lint → typecheck → test)

```bash
uv sync --group dev                                    # install incl. pytest/ruff/mypy/pre-commit
uv run pytest                                          # native; needs Postgres+Redis up
uv run pytest tests/test_merge_production_dotenvs_in_dotenv.py -x -q  # single file
just pytest                                            # Docker; CI also runs build + makemigrations --check + migrate
uv run ruff check --fix . && uv run ruff format .      # lint+format
uv run mypy bottom_up                                  # typecheck (django/drf plugins, test settings)
uv run pre-commit run --all-files                      # required before push; CI linter runs same
uv run coverage run -m pytest && uv run coverage html  # coverage over bottom_up/** only
```

Docker (closest to CI/prod, `justfile` wraps `docker-compose.local.yml`):

```bash
just build && just up                  # django :8000, mailpit :8025, flower :5555
just manage migrate                    # just manage <cmd> runs manage.py in container
just manage makemigrations             # after model changes; must commit files
just logs django                       # any service name works
just down                              # stop; just prune wipes DB volumes
```

## Env / services gotchas

See canonical runbook: `README.md#troubleshooting-runbook` for `USE_DOCKER`, Postgres/Redis, `username/first_name=None`, `api/docs` 403, Celery root.

- Native runs still need Postgres+Redis up; `USE_DOCKER=no` natively, `yes` in containers.
- Never commit secrets: `.env`, `.envs/.production/`, credentials in `.envs/.local/`.

## Conventions

- Ruff: line-length 119 but `E501` ignored (formatter owns it); single-line imports (`force-single-line`); rule set in `tool.ruff` — don't add `# noqa` without reason.
- Templates: djLint, 2-space indent, `profile=django`. Keep code valid for `django-upgrade --target-version 6.0`.
- mypy Tier3 ignores `*.migrations/admin/apps/urls` (see overrides in `pyproject.toml`); `.editorconfig` (LF, final newline, trim WS; Python 4sp, HTML/CSS/YML/TOML 2sp).
- Typing tiers (per-module strictness, 100% not required — rule, not file list):
-   Tier1 always annotate `->` + args: any function with branching/return across DB/API/Celery/auth boundary (`*.models`, `*.managers`, `*.services`, `*.selectors`, `*.tasks`, `*.api.views`, `*.api.serializers`, `*.adapters`, `*.views` logic).
-   Tier2 opportunistic: `conftest/factories/context_processors/apps.py/celery_app/tests` — add if editing.
-   Tier3 exempt: `class Meta`, forms pass-through, admin fieldsets, `urls/api_router/wsgi/settings/migrations` (pure declaration for Django to consume).
- Tests: pytest `--ds=config.settings.test --reuse-db --import-mode=importlib`. Fixtures in `bottom_up/conftest.py` (`user` via `UserFactory`, autouse media `tmpdir`). Put app tests next to app or under `tests/`. factory-boy available.
- CI (`.github/workflows/ci.yml`, docs-only changes skip): `linter` = pre-commit; `pytest` = Docker build → `makemigrations --check` (fails if migration missing) → `migrate` → `pytest`.

## Workflow — BDD → Types → Plan → TDD → Verify → Secure

- `main` protected — feature branches (`feat/|fix/|docs/|chore/`), small PRs, `git pull --rebase origin main`, squash-merge preferred, delete branch after.
- Full loop in `CONTRIBUTING.md#agentic-engineering-methodology`; commands summary in `README.md`; slash-commands index in `.opencode/README.md`.
- 1. BDD: read `plans/INDEX.md`, then one `plans/domains/*/NN-*.md`; follow its `requires_specs` only. Do not load all specs eagerly.
- 2. Types: models → serializers → views/router (`config/api_router.py`) → OpenAPI (`drf-spectacular`). Gate: `uv run mypy bottom_up`.
- 3. Plan: decompose into subtask checklist, one Red-Green-Refactor per subtask. Human approves plan before code.
- 4. TDD: RED failing pytest (`factory-boy`) from Gherkin → GREEN minimal → REFACTOR `ruff + mypy`. Tests next to app (`bottom_up/<app>/tests/`) or under `tests/`.
- 5. Verify: each Gherkin scenario maps to `test_views`/`test_urls`/`test_openapi` style tests. Gate: `just pytest` (Docker; CI also runs build + makemigrations --check + migrate) + `makemigrations --check`.
- 6. Secure every gate: `uv run pre-commit run --all-files`; never commit `.env`, `.envs/.production/`, or credentials in `.envs/.local/`.
