---
name: code-audit
description: Read-only tech-debt and code-smell audit against AGENTS.md conventions + ARCHITECTURE. Stdout report, zero writes.
---

# Code Audit (Tech Debt / Smells / Best Practices)

Read-only codebase health audit for the `bottom_up` Django project. Reports findings to stdout — never writes code, never runs test gates, never scores CVEs.

Source of truth: `AGENTS.md` (layout, conventions incl. typing tiers, workflow) and `plans/ARCHITECTURE.md` (canonical technical truth §§1-12). This skill operationalizes best-practice-finding only. Link to those files — do not duplicate them.

Non-overlap contract: `/qa` owns gates pass/fail. `/security-audit` owns vulnerabilities. This skill owns debt/smells/drift. If a finding needs a code change, hand off to `/implement-feature` (or `/qa --fix` for ruff-only fixes) — do not fix here. Arch doc fixes belong to `docs-maintainer` via `/update-docs`.

## When to use

Use when the user invokes `/code-audit` (runs in the `code-auditor` subagent). Load via `skill({ name: "code-audit" })`, then follow this workflow.

## Workflow

### 1. Parse flags (check FIRST)

From `$ARGUMENTS`:

- `scope=debt|smells|arch|all` — finding category (default `all`).
- `path=bottom_up/<app>/` — single-app scope; must stay inside `bottom_up/`, `config/`, or `tests/`. Default is whole codebase.
- `--quick` — same as `scope=smells` only.
- `--dry-run` — print the audit plan and make zero `write`/`edit`/`apply_patch` calls.

Strict dry-run gate: if `--dry-run` is present, stop after printing the plan. No file writes.

### 2. Audit (in order, report-only)

Inspect on disk via `read`/`glob`/`grep` — do not assume. Exemplar patterns live in `bottom_up/users/` (`test_views` / `test_urls` / `test_openapi`).

Smells (`scope=smells`):

- Complexity: `C90`-style long functions/files, `PLR0912/0913/0915` (branches/args/statements), deep nesting, `SIM/RET/PERF` anti-patterns, `EM/TRY` (string exceptions, broad except).
- Surface: `TODO/FIXME/XXX` count with file:line; commented-out code blocks.
- Templates: `djLint` drift (4-space indent inside Django templates, non-`profile=django` constructs).

Debt (`scope=debt`):

- Dead code: unused imports/vars/functions, unreachable branches, orphaned modules (imported nowhere).
- Duplication: copy-pasted blocks across apps (name the files, do not refactor here).
- Typing: Tier1 missing `->` + arg annotations per `AGENTS.md` typing tiers (`*.models`, `*.managers`, `*.services`, `*.selectors`, `*.tasks`, `*.api.views`, `*.api.serializers`, `*.adapters`, logic in `*.views`) and `tool.mypy` overrides in `pyproject.toml`.
- Modernity: `django-upgrade --target-version 6.0` staleness, old DRF/Celery idioms.
- Tests: files not following the `test_views` / `test_urls` / `test_openapi` pattern; Gherkin scenarios in `plans/domains/*/NN-*.md` with no mapped test (report-only).

Best practices (part of `scope=debt` unless `--quick`):

- Django ORM: list views missing `select_related`/`prefetch_related` (N+1 risk), multi-write paths missing `transaction.atomic` (note `ATOMIC_REQUESTS` is on — still flag non-request contexts like Celery/tasks/management commands).
- Celery: tasks without retries/timeouts/`acks_late` where appropriate (`config/celery_app.py`, `<app>/tasks.py`).
- DRF: serializers missing validation (`validate_*`/`extra_kwargs`), views missing explicit `permission_classes` (note only — `/security-audit` scores the vuln).
- Coverage: read-only note of gaps over `bottom_up/**` (do not re-run coverage here; `/qa` owns that).

Arch drift (`scope=arch`, report-only):

- Contract chain: `models → serializers → views → config/api_router.py → OpenAPI` — flag unregistered views, stale schema, missing router entries.
- `plans/ARCHITECTURE.md §§1-12` vs disk (apps, `LOCAL_APPS`, ports, commands); invalid `requires_specs` anchors in `plans/domains/*/NN-*.md`.
- Never rewrite ARCHITECTURE here — hand off to `/update-docs`.

### 3. Report (stdout only)

Per finding, one block:

```markdown
- **Category**: Smell | **Severity**: Med | **Effort**: S | **Location**: `bottom_up/users/services.py:88`
  - **Finding**: Function with 6 branches; consider extracting helper.
  - **Fix hint**: extract pure helper + unit test; implement via `/implement-feature`.
```

End with: counts per category/severity, top-5 paydown list ordered by severity × effort (small effort first), what was skipped (`scope=`/`--quick`/`path=`), and handoff suggestions. Do NOT commit, push, or open a PR unless explicitly asked.

## Flags ($ARGUMENTS)

- `scope=debt|smells|arch|all` — finding category (default `all`).
- `path=bottom_up/<app>/` — single-app scope.
- `--quick` — smells only.
- `--dry-run` — print the audit plan, make zero `write`/`edit` calls.

## Boundaries

- NEVER call `write`, `edit`, or `apply_patch` under any circumstance — strict read-only.
- NEVER accept `--fix` — reject it and direct to `/qa --fix` (ruff-only) or `/implement-feature` (refactors).
- NEVER run `pytest`, `coverage`, `migrate`/`makemigrations`, or `pre-commit` — `/qa` owns gates.
- NEVER score CVEs or re-audit secrets — `/security-audit` owns those.
- NEVER rewrite `plans/ARCHITECTURE.md` or docs — `/update-docs` owns that.
- NEVER invent endpoints, env vars, ports, or versions — read them.
- NEVER break `toctree` refs, `:ref:` targets, or spec `requires_specs` links.
- NEVER write report files — stdout only.
