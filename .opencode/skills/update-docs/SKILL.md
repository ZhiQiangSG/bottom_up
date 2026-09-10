---
name: update-docs
description: Audit codebase and sync README.md, AGENTS.md, plans/ARCHITECTURE.md, and docs/*.rst after major changes
---

# Update Docs

Audit the live repo state and sync user-facing + agent-facing docs to match reality. Never invent commands, versions, endpoints, or features.

Replaces the removed `update-readme` skill — README logic below is backward-compatible with it.

## When to use

Use when the user invokes `/update-docs` or asks to refresh docs after a major modification. Load via `skill({ name: "update-docs" })`, then follow this workflow.

Major modification means any of: new app under `bottom_up/`, model/API change, dependency or version bump, port/env change, command/CI/workflow change, new spec in `plans/domains/`.

## Workflow

### 1. Discover (read-only)

Inspect on disk — do not assume:

- Structure: `bottom_up/` apps, `config/settings/`, `config/urls.py`, `config/api_router.py`, `config/celery_app.py`, `manage.py`, `bottom_up/templates/`, `bottom_up/static/`
- Manifests: `pyproject.toml` (`[project]` name/version/description/dependencies, `[dependency-groups]` dev), `uv.lock`, `.python-version` (Python 3.14), `justfile`, `docker-compose.local.yml`, `docker-compose.docs.yml`, `.envs/.local/.postgres`
- Config: `config/settings/base.py` vs `local/test/production.py`, API prefix in `config/api_router.py`, `api/schema/` + `api/docs/` if present
- Core targets (always read before writing): `README.md`, `AGENTS.md`, `plans/ARCHITECTURE.md`, `docs/index.rst`, `docs/howto.rst`, `docs/users.rst`, `docs/conf.py`
- Conditional targets (read only if relevant change detected): `CONTRIBUTING.md` (workflow/commands/pre-commit/CI changed), `plans/INDEX.md` (new `domains/<d>/NN-*.md` added), `plans/VISION.md` (proposal-only, never auto-rewrite)
- History: `git log -n 20 --oneline` + `git status --short` + `git diff --stat main...HEAD`. If not a git repo, skip and note "history unavailable".

Preserve pass (always run before writing):

- Read every target file you intend to touch first via the `read` tool.
- Extract and carry over custom content: badges, logos (`<img>`/![logo]), mermaid/architecture diagrams, hand-written notes, ADRs, manual warnings — unless directly contradicted by live code truth.
- Never drop these silently. List kept items in your final summary.

Scoped update (`target=`, `focus=`, `path=`):

- `target=` selects files: `readme|agents|arch|docs|all` (default `all`). `target=docs` without `path=` covers all `docs/*.rst`; with `path=docs/<file>.rst` covers that file only.
- `focus=<section>` rewrites ONLY that section within the selected target(s). Preserve all other sections byte-for-byte from the version you just read.
- If the target section is missing, insert it in the template position without touching neighbors.
- Valid `focus` for `readme`: `title`, `stack`, `prerequisites`, `quickstart`, `config`, `commands`, `api`, `features`, `deployment`.
- Valid `focus` for `agents`: `layout`, `commands`, `env`, `conventions`, `workflow`.
- Valid `focus` for `arch`: `overview`, `stack`, `repomap`, `env`, `auth`, `data`, `celery`, `quality`, `workflow`, `recipes`, `pitfalls`, `status`.
- `path=` must stay inside `docs/` or equal `plans/ARCHITECTURE.md`. Reject anything else.

### 2. Synthesize

- Prerequisites: Python from `.python-version`, `uv`, Docker + `just`, Postgres + Redis for native runs (`POSTGRES_HOST/PORT/DB/USER/PASSWORD`, `REDIS_URL=redis://localhost:6379/0`, `USE_DOCKER=no`, `DJANGO_SETTINGS_MODULE=config.settings.local`)
- Setup truth: `uv sync --group dev`, `docker compose -f docker-compose.local.yml up -d postgres redis mailpit`, `just build && just up`, `just manage migrate`, `uv run pytest` / `just pytest`
- Stack: Django 6, DRF + drf-spectacular, Celery + beat, Redis, Postgres (psycopg), uv, Docker, Mailpit `:8025`, Flower `:5555` — pin exact versions from `pyproject.toml`/`uv.lock` only
- Features/status: only what exists on disk + recent commits. Mark gaps as `> Status: not yet implemented`.
- ARCHITECTURE truth (§§1-12 in `plans/ARCHITECTURE.md`): system diagram + routing from `config/urls.py`, pinned stack, repo map + `LOCAL_APPS` + `router.register()`, env matrix (container vs native), auth/API conventions (`USERNAME_FIELD=email`, `IsAuthenticated` default, spectacular `IsAdminUser`, self-scoped ViewSets), migration rule (`makemigrations` + commit, CI `--check`), Celery (`config/celery_app.py`, tasks in `<app>/tasks.py`), quality-gate order `lint → typecheck → test`, agentic rules (`opencode.json` perms, specs `requires_specs`, lazy-load via `plans/INDEX.md`), recipes A/B/C, pitfalls, §12 status. Link to specs — never duplicate this file into specs.
- Sphinx truth: `docs/index.rst` toctree (`maxdepth: 2`), `howto.rst` build via `docker-compose.docs.yml`, per-app `<app>.rst` in `users.rst` `automodule` + `:members:` + `:noindex:` style, Napoleon Google/Numpy docstrings only.

### 3. Rewrite (or propose)

Strict dry-run gate (check FIRST):

- If `--dry-run` is present in `$ARGUMENTS`: DO NOT call `write`, `edit`, or `apply_patch` under any circumstance.
- Output one ```markdown or ```rst fenced block per touched file with `path:` header, followed by a bullet changelog.
- End response there. No file writes, no commits.

Full rewrite (no `focus=`, no `path=`, no `--dry-run`, `target=all` or omitted):

- `README.md` — write complete file using this structure:

```markdown
# <name from pyproject.toml>
<concise description> + badges + License
## Tech Stack & Architecture
## Prerequisites
## Quickstart (native + Docker)
## Configuration / Env
## Basic Commands (users, test, lint, typecheck, celery)
## API / Usage Examples
## Key Features & Current Status
## Deployment
## Contributing / License
```

- `AGENTS.md` — concise ops reference only (<~60 lines): Layout, Commands (verify order: lint → typecheck → test), Env/services gotchas, Conventions, Workflow. Update only sections whose truth changed. Never mirror README prose.
- `plans/ARCHITECTURE.md` — update §§1-12 in place, preserving section numbers and lazy-load rule. Never rename sections without updating cross-refs in specs.
- `docs/*.rst` — keep `index.rst` toctree valid; update `howto.rst` build cmds only from `docker-compose.docs.yml`/`Makefile`; for a new Django app `bottom_up/<app>/`, create `docs/<app>.rst` following `users.rst` pattern and insert into toctree alphabetically.
- Conditional: `CONTRIBUTING.md` only when workflow/commands/CI changed; `plans/INDEX.md` only when a new domain/spec file was added (add row to Domains or Features table); `plans/VISION.md` proposal-only via `--dry-run`, never direct rewrite.
- Keep code blocks copy-pasteable. Prefer `uv run` / `just <cmd>` over bare `python`.
- Do NOT commit unless explicitly asked.

## Flags ($ARGUMENTS)

- `target=readme|agents|arch|docs|all` — file scope (default `all`).
- `focus=<section>` — scoped rewrite within target, preserve rest byte-for-byte (see Step 1).
- `path=docs/<file>.rst` or `path=plans/ARCHITECTURE.md` — single-file scope; must stay inside allowlist above.
- `branch=<name>` — use `git log <name> -n 20 --oneline` for history.
- `--dry-run` — proposal only, zero file tool calls (see Step 3).

## Boundaries

- NEVER hallucinate endpoints, env vars, ports, or versions — read them.
- NEVER delete License/badges/diagrams/notes without replacement.
- NEVER commit secrets (`.env`, `.envs/.production/`, credentials in `.envs/.local/`).
- NEVER run destructive commands (`just prune`, `down -v`, `rm`, `push`, `commit`).
- NEVER convert `.rst` to `.md` (or reverse) unasked.
- NEVER break `toctree` refs, `:ref:` targets, or spec `requires_specs` links.
- NEVER auto-rewrite `plans/VISION.md` — proposal only.
- NEVER duplicate `plans/ARCHITECTURE.md` into specs or create a second root `ARCHITECTURE.md` — link to the canonical file.
