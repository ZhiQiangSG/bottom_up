# Bottom_Up

E-Capstone project — Django 6 + DRF + Celery + Redis + Postgres (cookiecutter-django). License: MIT.

## Tech Stack & Architecture

- Python `3.14` (see `.python-version`), deps via `pyproject.toml` + `uv.lock` (managed with `uv`)
- Backend: `Django==6.0.7`, `djangorestframework==3.17.1`, `drf-spectacular==0.30.0`, `django-cors-headers`, `django-allauth[mfa]==65.18.0`
- Async: `celery==5.6.3` + `django-celery-beat==2.9.0`, broker/backend `redis` (`redis:7.2` image, `redis==8.1.0` client)
- DB/cache: Postgres (`psycopg[c]==3.3.4`), `django-redis==7.0.0`, WhiteNoise, `gunicorn==26.0.0`
- Dev services (`docker-compose.local.yml`): `django` (`:8000`), `postgres`, `redis`, `mailpit` (`:8025`), `celeryworker`, `celerybeat`, `flower` (`:5555`)
- Layout:
  - Apps: `bottom_up/` (currently `bottom_up/users/` only). Project config: `config/` (`settings/`, `urls.py`, `api_router.py`, `celery_app.py`)
  - Shared templates/static: `bottom_up/templates/`, `bottom_up/static/` (`staticfiles/` build output)
  - API base: `api/` (see `config/api_router.py`); schema/docs: `api/schema/`, `api/docs/`
  - Settings: `config/settings/{local,test,production}.py` all import `base.py`. Tests use `config.settings.test` via `tool.pytest` `addopts`

## Prerequisites

| Tool | Version / notes |
| ------ | ----------------- |
| Python | `3.14` (`.python-version`, `requires-python ==3.14.*`) |
| [uv](https://docs.astral.sh/uv/) | latest; `uv sync --group dev` installs dev group (pytest, ruff, mypy, pre-commit, factory-boy, etc.) |
| Docker + Compose v2 | for Postgres, Redis, Mailpit, Django, Celery |
| [just](https://github.com/casey/just) | recommended; wraps `docker compose` (`justfile`, `COMPOSE_FILE=docker-compose.local.yml`) |
| Postgres + Redis | required even for native runs (run via Docker) |

## Quickstart (native + Docker)

```bash
git clone <your-repo-url> bottom_up
cd bottom_up
uv sync --group dev
```

Option A — Docker (recommended, closest to CI/prod):

```bash
just build
just up
just manage migrate
just manage createsuperuser
# open:
# http://127.0.0.1:8000
# http://127.0.0.1:8025  # Mailpit
# http://127.0.0.1:5555  # Flower
```

Option B — uv native (fast iteration, still needs infra):

```bash
docker compose -f docker-compose.local.yml up -d postgres redis mailpit
export POSTGRES_HOST=localhost POSTGRES_PORT=5432
export POSTGRES_DB=bottom_up POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres
# match values in `.envs/.local/.postgres` if customized
export REDIS_URL=redis://localhost:6379/0 USE_DOCKER=no
export DJANGO_SETTINGS_MODULE=config.settings.local
uv run python manage.py migrate
uv run python manage.py createsuperuser
uv run python manage.py runserver
```

Useful Docker helpers:

```bash
just logs django        # any service name works
just manage <cmd>       # e.g. just manage shell, just manage makemigrations
just pytest             # test suite inside Docker (same as CI)
just down               # stop; just prune wipes volumes/DB!
```

## Configuration / Env

- Settings: `config/settings/base.py` base; `local.py` (`DEBUG=True`, `EMAIL_HOST=mailpit:1025`, `LocMemCache`, debug-toolbar, extensions), `test.py` (MD5 hasher, locmem email, `MEDIA_URL=http://media.testserver/`), `production.py` (Mailgun via `django-anymail`, `django-redis` cache, HSTS/SSL, `ADMIN_URL` from env).
- Key env vars (see `base.py` + `.envs/.local/`):
  - `POSTGRES_DB/USER/PASSWORD/HOST/PORT` (default `HOST=postgres`, `PORT=5432`; use `localhost` natively)
  - `REDIS_URL` (default `redis://redis:6379/0`; native: `redis://localhost:6379/0`)
  - `DJANGO_SETTINGS_MODULE=config.settings.local`, `USE_DOCKER=no` natively / `yes` in containers, `DJANGO_DEBUG`, `DJANGO_SECRET_KEY`, `DJANGO_READ_DOT_ENV_FILE`
  - `EMAIL_HOST=mailpit` (dev), `CELERY_*` derived from `REDIS_URL`, `TIME_ZONE=Asia/Singapore`
- Do NOT commit secrets: `.env`, `.envs/.production/`, credentials in `.envs/.local/` (e.g. `.postgres` user/password).

## Basic Commands (users, test, lint, typecheck, celery)

```bash
# install
uv sync --group dev

# users / DB
just manage migrate
just manage makemigrations   # after model changes; must commit files
just manage shell

# tests (config: --ds=config.settings.test --reuse-db --import-mode=importlib)
uv run pytest
uv run pytest tests/test_merge_production_dotenvs_in_dotenv.py -x -q
just pytest                  # Docker; CI also runs build + makemigrations --check + migrate

# lint + format + typecheck (run in this order)
uv run ruff check --fix .
uv run ruff format .
uv run mypy bottom_up
uv run pre-commit run --all-files  # required before push; CI linter runs same

# coverage (includes bottom_up/** only)
uv run coverage run -m pytest && uv run coverage html

# celery (from repo root, same dir as manage.py)
uv run celery -A config.celery_app worker -l info
uv run celery -A config.celery_app beat -l info
```

## API / Usage Examples

Base prefix `api/` (`config/urls.py` → `config.api_router`):

- `GET /api/users/` — list (scoped to `request.user` only, `IsAuthenticated`)
- `GET /api/users/{pk}/` — retrieve (scoped to self)
- `PATCH/PUT /api/users/{pk}/` — update (scoped to self)
- `GET /api/users/me/` — current user (`@action(detail=False)`)
- `POST /api/auth-token/` — obtain DRF token (`rest_framework.authtoken.views.obtain_auth_token`)
- `GET /api/schema/` — OpenAPI schema (`SpectacularAPIView`, title `bottom_up API v1.0.0`)
- `GET /api/docs/` — Swagger UI (`SpectacularSwaggerView`, admin-only by `SERVE_PERMISSIONS`)

Non-API:

- `/` (`pages/home.html`), `about/` (`pages/about.html`), `users/` (`bottom_up.users.urls`), `accounts/` (allauth, email-only login, mandatory verification), `admin/` (or `settings.ADMIN_URL` in prod)

Example:

```bash
# schema
curl http://127.0.0.1:8000/api/schema/ | head -n 20
# token + me
curl -X POST -d "username=<email>&password=<pass>" http://127.0.0.1:8000/api/auth-token/
curl -H "Authorization: Token <token>" http://127.0.0.1:8000/api/users/me/
```

## Key Features & Current Status

- Custom user (`AUTH_USER_MODEL=users.User`, Argon2 hasher): login via email (allauth), registration allowed, mandatory email verification — present
- `UserViewSet` (retrieve/list/update + `me`, self-scoped queryset) + `UserSerializer[name,url]` — present
- Admin, crispy-bootstrap5 pages (`home`, `about`), Mailpit dev email, Flower/Celery beat scheduler — present
- CORS limited to `^/api/.*$`, Token + Session auth, `IsAuthenticated` default — present
- > Status: not yet implemented — no domain apps beyond `users`; no additional API resources, custom Celery tasks, or frontend beyond cookiecutter pages verified on disk.

## Troubleshooting Runbook

Canonical runbook for common pitfalls. `AGENTS.md`, `CONTRIBUTING.md`, and `plans/ARCHITECTURE.md` link here — do not duplicate.

| Symptom | Cause | Fix |
| ------- | ----- | --- |
| Debug toolbar missing / wrong IPs, env confusion | `USE_DOCKER` wrong (`config/settings/local.py`). `yes` only in containers, `no` natively | Natively: `export USE_DOCKER=no`. Containers use `USE_DOCKER=yes` via `.envs/.local/.django` |
| `OperationalError` / connection refused to Postgres/Redis, `migrate` / `pytest` / `runserver` fails | Infra not up. Native runs still need Docker Postgres+Redis | `docker compose -f docker-compose.local.yml up -d postgres redis mailpit`, then `POSTGRES_HOST=localhost POSTGRES_PORT=5432` (match `.envs/.local/.postgres`), `REDIS_URL=redis://localhost:6379/0`, `DJANGO_SETTINGS_MODULE=config.settings.local` |
| `AttributeError` on `user.username` / `user.first_name` | Custom user has `username=None`, `first_name/last_name=None` (`bottom_up/users/models.py`, `USERNAME_FIELD=email`) | Use `user.email` for login and `user.name` for display. Never assume `username` exists |
| `GET /api/docs/` returns `403 Forbidden` | Expected. `SERVE_PERMISSIONS=[IsAdminUser]` in `config/settings/base.py` | Log in as staff/admin to view docs. Do not expose publicly |
| `ModuleNotFoundError: No module named config` running Celery | Celery started outside repo root | Run from repo root (same dir as `manage.py`): `uv run celery -A config.celery_app worker -l info` (and `beat` separately) |

## Deployment

- Production settings (`config/settings/production.py`): `SECRET_KEY`, `DJANGO_ALLOWED_HOSTS` (default `bottomup.com`), `DATABASE_URL` or `POSTGRES_*`, `REDIS_URL`, `MAILGUN_API_KEY/DOMAIN`, `DJANGO_ADMIN_URL`, `DJANGO_SECURE_SSL_REDIRECT`, Anymail Mailgun backend, `CompressedManifestStaticFilesStorage`.
- Docker prod uses `compose/production/` images; local uses `compose/local/django/Dockerfile` + `/start`, `/start-celeryworker`, `/start-celerybeat`, `/start-flower`.
- CI (`.github/workflows/ci.yml`, docs-only skip): `linter` = pre-commit; `pytest` = Docker build → `makemigrations --check` → `migrate` → `pytest`.

## Contributing / License

- `main` protected — branches `feat/|fix/|docs/|chore/`, `git pull --rebase origin main`, small PRs, 1 review, squash-merge preferred. Full workflow in `CONTRIBUTING.md`.
- Agents: slash-commands index in `.opencode/README.md`; workflow in `CONTRIBUTING.md#agentic-engineering-methodology`.
- MIT — see `LICENSE`. Contact from `pyproject.toml`: `team hired`.
