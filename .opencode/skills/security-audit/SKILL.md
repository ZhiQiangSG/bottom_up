---
name: security-audit
description: Read-only Django 6 + DRF + Celery vulnerability audit (deploy posture, authz, injection, secrets, deps via pip-audit). Stdout report, zero writes.
---

# Security Audit (Vulnerabilities Only)

Read-only vulnerability audit for the `bottom_up` Django project. Reports findings to stdout — never writes code, never upgrades deps, never prints secrets.

Source of truth: `AGENTS.md` (layout, env gotchas, secure gate) and `CONTRIBUTING.md#agentic-engineering-methodology` (Secure step). This skill operationalizes vulnerability-finding only. Link to those files — do not duplicate them.

Non-overlap contract: `/qa` owns gates pass/fail. `/code-audit` owns debt/smells. This skill owns vulnerabilities. If a finding needs a code change, hand off to `/implement-feature` (or `/qa --fix` for ruff-only fixes) — do not fix here.

## When to use

Use when the user invokes `/security-audit` (runs in the `security-auditor` subagent). Load via `skill({ name: "security-audit" })`, then follow this workflow.

## Workflow

### 1. Parse flags (check FIRST)

From `$ARGUMENTS`:

- `--quick` — Gates A + B + D only (settings, authz, secrets). Skips pip-audit + deep grep.
- `--full` — default. All gates A–E.
- `--dry-run` — print the gate plan and make zero `write`/`edit`/`apply_patch` calls and run zero fixing commands.

Strict dry-run gate: if `--dry-run` is present, stop after printing the plan. No file writes, no dep upgrades, no `ruff --fix`.

### 2. Gates (in order, report-only)

Inspect on disk — do not assume. Prefer `read`/`grep` + `@file` refs.

Gate A — Django deploy posture:

- `read config/settings/base.py`, `config/settings/local.py`, `config/settings/production.py`.
- Check: `DEBUG`, `ALLOWED_HOSTS`, `SECRET_KEY` source (`env()`, never hardcoded), `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS/INCLUDE_SUBDOMAINS/PRELOAD`, `SECURE_CONTENT_TYPE_NOSNIFF`, `SESSION_COOKIE_SECURE/HTTPONLY/SAMESITE`, `CSRF_COOKIE_SECURE/HTTPONLY`, `CORS_URLS_REGEX`, `ADMIN_URL` (non-default in production), `EMAIL_BACKEND` (no console backend in production).
- Run `uv run python manage.py check --deploy` (report-only; needs env vars per `AGENTS.md` — if it fails on missing env, note it and continue with static reads).

Gate B — AuthN/Z (DRF + allauth):

- `read config/api_router.py` + `bottom_up/*/api/views.py` + `bottom_up/*/api/serializers.py`.
- Check: `REST_FRAMEWORK DEFAULT_PERMISSION_CLASSES` includes `IsAuthenticated`; every ViewSet declares explicit `permission_classes`; object-level scoping (queryset filtered to request user, no unscoped `Model.objects.all()` on user data); `SPECTACULAR_SETTINGS SERVE_PERMISSIONS` is `IsAdminUser`.
- Check allauth: `ACCOUNT_EMAIL_VERIFICATION`, `ACCOUNT_LOGIN_METHODS`, `ACCOUNT_ADAPTER` / `SOCIALACCOUNT_ADAPTER` paths exist.

Gate C — Injection / XSS / shell:

- `grep` for `mark_safe|format_html|extra\(|RawSQL|shell=True|subprocess|pickle\.loads|yaml\.load\(|eval\(|exec\(` across `bottom_up/**` and `config/**`.
- Triage existing Ruff `S`/`BLE`/`DJ` hits (interpret them — do not re-run the lint gate; `/qa` owns that).
- Each hit: confirm reachability (user input → sink) before rating High.

Gate D — Secrets guard (every run):

- Never read or print secret values. Check status only: `git status --short` + `git diff --stat` (if not a git repo, note "history unavailable" and continue).
- Fail the run if `.env`, `.envs/.production/`, or credentials in `.envs/.local/` are staged or modified. Report the paths, do not show contents.

Gate E — Dependencies (skipped if `--quick`):

- Run `uv run pip-audit --desc` (report-only; never `--fix` — dep upgrades are out of scope).
- If pip-audit is unreachable offline, note `pip-audit unavailable` and fall back to `read pyproject.toml + uv.lock` for unpinned/loose pins.
- Only report CVEs/GHSAs that pip-audit (or lock inspection) actually shows — never invent CVE IDs.

### 3. Report (stdout only)

Per finding, one block:

```markdown
- **Severity**: High | **Gate**: B-authz | **Location**: `bottom_up/users/api/views.py:42`
  - **Finding**: UserViewSet queryset not scoped to request user.
  - **Evidence**: `queryset = User.objects.all()` (1-line excerpt, redacted).
  - **Fix hint**: scope to `request.user` or add object permission; implement via `/implement-feature`.
```

End with: counts per severity, top-3 next actions, what was skipped (`--quick`) and handoff suggestions. Do NOT commit, push, or open a PR unless explicitly asked.

## Flags ($ARGUMENTS)

- `--quick` — Gates A + B + D only.
- `--full` — default, Gates A–E.
- `--dry-run` — print the gate plan, make zero `write`/`edit` calls, run zero fixing commands.

## Boundaries

- NEVER call `write`, `edit`, or `apply_patch` under any circumstance — strict read-only.
- NEVER run `--fix`, upgrade deps, or apply `pip-audit --fix`.
- NEVER read or print secret values (`.env`, `.envs/.production/`, credentials in `.envs/.local/`).
- NEVER invent CVE/GHSA IDs, endpoints, env vars, ports, or versions — read them.
- NEVER run destructive commands (`just prune`, `down -v`, `rm -rf`, `push`) unless explicitly asked.
- NEVER write report files — stdout only.
