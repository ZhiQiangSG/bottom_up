# Architecture — bottom_up (Agent Entrypoint)

> Stable technical truth for agentic work. Companion files:
> - Workflow/commands: `AGENTS.md`, `CONTRIBUTING.md`, `README.md`
> - Product why: `plans/VISION.md` (three-sided: students/employers/schools; Discover → Apply → Hire → Connect → Assist)
> - Spec index: `plans/INDEX.md`
>
> **Lazy-load rule:** start at `INDEX.md` → read ONE `domains/<domain>/domain.md`
> or `NN-*.md` → follow only its `requires_specs` transitively.
> Do NOT load all specs eagerly. Do NOT duplicate this file into specs; link here.

## 1. System Overview

```text
                ┌─────────────┐  :8000  ┌──────────────┐
Browser ───────▶│ django      ├────────▶│ postgres     │
                │ (config/    │         └──────────────┘
                │  urls.py)   │ Redis   ┌──────────────┐
                │             ├────────▶│ redis :6379  │
                │             │         └──────┬───────┘
                │             │                │ broker/backend
                │             │         ┌──────▼───────┐
                │             │         │ celeryworker │
                │             │         │ celerybeat   │ (DatabaseScheduler)
                │             │         │ flower :5555 │
                └──────┬──────┘         └──────────────┘
                       │ SMTP :1025
                ┌──────▼──────┐
                │ mailpit     │ :8025 UI
                └─────────────┘
```

Request routing (`config/urls.py`):

- `/` → `pages/home.html`, `about/` → `pages/about.html`
- `users/` → `bottom_up.users.urls`, `accounts/` → allauth, `admin/` (or `ADMIN_URL` in prod)
- `api/` → `config.api_router` (`DefaultRouter if DEBUG else SimpleRouter`)
- `api/auth-token/`, `api/schema/` (`SpectacularAPIView`), `api/docs/` (`SpectacularSwaggerView`)

## 2. Tech Stack (Pinned)

Python `3.14` (`.python-version`, `requires-python==3.14.*`).
Backend: `Django==6.0.7`, `DRF==3.17.1`, `drf-spectacular==0.30.0`,
`django-cors-headers`, `allauth[mfa]==65.18.0`.
Async: `celery==5.6.3` + `django-celery-beat==2.9.0`, `redis==8.1.0` client on `redis:7.2`.
DB/cache: `psycopg[c]==3.3.4`, `django-redis==7.0.0`, WhiteNoise, `gunicorn==26.0.0`.
Manage via `pyproject.toml` + `uv.lock`, never bare `python/pip` — always `uv run` or `just`.
Dev: `pytest`, `ruff`, `mypy` (+ `django/drfs stubs`), `pre-commit`, `factory-boy`, `debug-toolbar`, `django-extensions`.

## 3. Repo Map — Where Code Goes

```text
config/
  settings/{base,local,test,production}.py  # all import base.py; tests use test
  urls.py api_router.py celery_app.py wsgi.py
bottom_up/<app>/
  models.py managers.py admin.py
  api/{views.py,serializers.py}  # ViewSet + Serializer
  urls.py forms.py adapters.py tasks.py apps.py
  tests/{test_*.py,factories.py} | templates/<app>/ | static/
bottom_up/{templates/,static/,contrib/,conftest.py}
plans/{INDEX.md,VISION.md,ARCHITECTURE.md,domains/<domain>/{domain.md,NN-*.md}}
compose/{local,production}/ docker-compose.{local,production}.yml justfile
```

Only app today is `bottom_up/users/` — use it as the template.
New product domains (`discovery`, `applications`, `postings` in `plans/`)
each become a new Django app under `bottom_up/` + entry in `LOCAL_APPS`
(`config/settings/base.py`) + `router.register()` + migrations.

## 4. Settings & Env Matrix

| Key | Container | Native (`uv run`) |
|---|---|---|
| `POSTGRES_HOST/PORT` | `postgres/5432` | `localhost/5432` (match `.envs/.local/.postgres`) |
| `REDIS_URL` | `redis://redis:6379/0` | `redis://localhost:6379/0` |
| `USE_DOCKER` | `yes` (toolbar Docker IPs) | `no` |
| `DJANGO_SETTINGS_MODULE` | `config.settings.local` | `config.settings.local` (tests: `config.settings.test` via pytest `addopts`) |
| Email | `mailpit:1025` | same via forwarded `mailpit` service |
| `TIME_ZONE` | `Asia/Singapore` | same |

`local.py`: `DEBUG=True`, `LocMemCache`. `test.py`: MD5 hasher, locmem email,
`MEDIA_URL=http://media.testserver/`. `production.py`: Mailgun via anymail,
`django-redis`, `CompressedManifestStaticFilesStorage`, `ADMIN_URL` from env.
Infra must be up even natively:
`docker compose -f docker-compose.local.yml up -d postgres redis mailpit`.
Never commit `.env`, `.envs/.production/`, secrets in `.envs/.local/`.

## 5. Auth / Users / API Conventions

- `AUTH_USER_MODEL=users.User` (`bottom_up/users/models.py`):
  `USERNAME_FIELD=email`, `email unique`, `username=None`,
  `first_name/last_name=None`, extra `name CharField`.
  `objects: UserManager`. `get_absolute_url()` → `users:detail`.
- allauth: `ACCOUNT_LOGIN_METHODS={email}`,
  `ACCOUNT_SIGNUP_FIELDS=[email*,password1*,password2*]`,
  `ACCOUNT_EMAIL_VERIFICATION=mandatory`,
  adapters `bottom_up.users.adapters.{Account,SocialAccount}Adapter`,
  forms `bottom_up.users.forms.User{,Social}SignupForm`.
- DRF (`base.py:REST_FRAMEWORK`): `Session+Token`, default `IsAuthenticated`,
  `DEFAULT_SCHEMA_CLASS=AutoSchema`. `CORS_URLS_REGEX=r"^/api/.*$"`.
  Spectacular: `TITLE bottom_up API v1.0.0`, `SCHEMA_PATH_PREFIX=/api/`,
  `SERVE_PERMISSIONS=[IsAdminUser]` (docs 403 for non-staff — expected).
- Current resources: `GET /api/users/` (self-scoped list), `GET/PATCH/PUT
  /api/users/{pk}/` (self only), `GET /api/users/me/`, `POST /api/auth-token/`.
- Convention for new resources: `Model → Serializer (name,url style minimal) →
  ViewSet (scope queryset to request.user/role, IsAuthenticated) →
  router.register("things", ThingViewSet)` in `api_router.py` → schema check
  via `curl /api/schema/`.

## 6. Data Model & Migrations

Current: only `users.User` (+ `sites` override in `bottom_up.contrib.sites`).
`DEFAULT_AUTO_FIELD=BigAutoField`, `ATOMIC_REQUESTS=True`.
Future models (roles, postings, applications) go in new apps, FK to
`settings.AUTH_USER_MODEL`, never import `User` directly in new apps.

Rule: after ANY model change run `just manage makemigrations`
(or `uv run python manage.py makemigrations`) and COMMIT the file.
CI runs `makemigrations --check` → fails if missing, then `migrate` → `pytest`.

## 7. Celery / Async

`config/celery_app.py`, `CELERY_BROKER_URL/RESULT_BACKEND=REDIS_URL`,
JSON serializers, `TIME_LIMIT=300`, `SOFT=60`,
`BEAT_SCHEDULER=DatabaseScheduler`, `WORKER_SEND_TASK_EVENTS=True`.
Tasks live in `bottom_up/<app>/tasks.py` (see `users/tasks.py`).
Run from repo root (same dir as `manage.py`):
`uv run celery -A config.celery_app worker -l info` (+ `beat` separately).
In Docker: `celeryworker` / `celerybeat` / `flower` services.

## 8. Testing & Quality Gates (Verify Order: lint → typecheck → test)

```bash
uv sync --group dev
uv run ruff check --fix . && uv run ruff format .
uv run mypy bottom_up          # plugins django/drf, test settings; ignores *.migrations.*
uv run pytest                  # --ds=config.settings.test --reuse-db --import-mode=importlib
uv run pytest tests/test_merge_production_dotenvs_in_dotenv.py -x -q   # single file
just pytest                    # Docker; CI also runs build + makemigrations --check + migrate
uv run pre-commit run --all-files  # required before push; CI linter runs same
uv run coverage run -m pytest && uv run coverage html  # covers bottom_up/** only
```

Fixtures: `bottom_up/conftest.py` (`user` via `UserFactory`, autouse media `tmpdir`).
Put app tests in `bottom_up/<app>/tests/` or `tests/`. Use `factory-boy`.
Templates: djLint 2-space, `profile=django`; keep valid for `django-upgrade --target-version 6.0`.
Ruff: `line-length 119` (E501 ignored), single-line imports.

## 9. Agentic Workflow Rules

1. Branch: `feat/|fix/|docs/|chore/short-name`, `git pull --rebase origin main`, small PRs, 1 review, squash-merge, delete branch.
2. Specs: new feature = `plans/domains/<domain>/NN-*.md` with `requires_specs`
   frontmatter listing `domain.md` + `ARCHITECTURE.md`; update `plans/INDEX.md` table.
3. Permissions (`opencode.json`): `edit:* allow` except `.env` + `.envs/.production/** deny`;
   `bash:* allow` except `rm -rf * deny`, ask on `just prune*`, `docker *prune*`, `docker *down -v*`.
4. CI (`.github/workflows/ci.yml`, docs-only skip): `linter` = pre-commit;
   `pytest` = Docker build → `makemigrations --check` → `migrate` → `pytest`.

## 10. Recipes

### A. Add new app/domain (e.g. `postings`)

1. `uv run django-admin startapp postings bottom_up/postings` (or copy `users/` skeleton).
2. Add `bottom_up.postings` to `LOCAL_APPS`, add `models.py` + `admin.py` + `tests/factories.py`.
3. `just manage makemigrations postings && just manage migrate`.
4. Add `tests/test_postings.py` using `UserFactory` + new factory; `uv run pytest <file> -x -q`.
5. Run verify order (§8). Add spec `plans/domains/postings/NN-*.md`, update `INDEX.md`.

### B. Add API endpoint

1. `serializers.py` + `views.py: ThingViewSet(ModelViewSet, IsAuthenticated, scoped get_queryset)`.
2. `router.register("things", ThingViewSet)` in `config/api_router.py`.
3. `curl /api/schema/ | grep things`, `curl -H "Authorization: Token <t>" /api/things/`.
4. Spectacular + permission check; `ruff/mypy/pytest`.

### C. Add Celery task

1. Def in `bottom_up/<app>/tasks.py` with `@shared_task`, JSON-serializable args only.
2. Call from view/serializer, test with `CELERY_TASK_ALWAYS_EAGER=True` in test.
3. Verify worker + beat locally via `just logs celeryworker`.

## 11. Pitfalls — Do Not Do

See canonical runbook: `README.md#troubleshooting-runbook`.

- Bare `python/pip`, running Celery outside repo root, `cd x && cmd` (use `workdir`).
- Forgetting `USE_DOCKER=no` natively or Postgres/Redis not up.
- Assuming `user.username` / `first_name` exist — they are `None` on this `User`.
- Exposing `api/docs/` publicly — admin-only is intentional.
- Adding `# noqa` without reason, committing migrations-less models, committing secrets.
- Loading all `plans/domains/*` at once — follow `INDEX.md` lazy-load.

## 12. Current Status & Open Decisions

Done: custom user + Argon2, `UserViewSet` + `me`, admin, crispy `home/about`,
Mailpit, Flower/beat. Missing: everything in `discovery/applications/postings`
— contracts only, no models/APIs/tasks. Next: decide role field on `User`
vs profile models, Google OAuth scope (`02-login.md`), approval queue owner.
