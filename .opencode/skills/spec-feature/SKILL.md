---
name: spec-feature
description: Draft BDD spec + type contract + subtask checklist for a feature. Read-only planning, no code edits.
---

# Spec Feature (Planning: BDD + Types + Plan)

Produce a BDD spec review, type contract, and subtask checklist for one feature. Make ZERO code edits. Stop for human approval before any implementation.

Source of truth: `CONTRIBUTING.md#agentic-engineering-methodology` and `AGENTS.md## Workflow`. This skill operationalizes steps 1-3 only. Link to those files — do not duplicate them.

## When to use

Use when the user invokes `/spec-feature` or asks to plan a feature. Load via `skill({ name: "spec-feature" })`, then follow this workflow.

## Workflow

### 1. Discover spec (read-only)

- Require `spec=plans/domains/<domain>/NN-<feature>.md` from `$ARGUMENTS`. If missing, ask for it — do not guess.
- Read `plans/INDEX.md` first to locate the feature, then read ONLY that one spec file.
- Follow ONLY its `requires_specs` frontmatter transitively. Do not load all specs eagerly.
- Verify the spec has: Goal / Non-goals, Actors, Gherkin scenarios (`Given/When/Then`), acceptance criteria observable via API/UI. If any piece is missing, flag it and propose minimal additions — do not write code.

### 2. Type contract (shape, no logic)

Inspect on disk — do not assume:

- Apps: `bottom_up/` apps, `config/api_router.py` (`router.register()` entries), `config/settings/test.py`, existing serializers/views patterns (see `bottom_up/users/` for the `test_views` / `test_urls` / `test_openapi` pattern).
- Manifests: `pyproject.toml` (`tool.pytest`, `tool.mypy` with `mypy_django_plugin` / `mypy_drf_plugin`), `justfile`, `docker-compose.local.yml`.

Output the contract in this order (names only, no implementation):

1. `bottom_up/<app>/models.py` — models/fields with types.
2. `bottom_up/<app>/api/serializers.py` — DRF serializers (the API contract).
3. `bottom_up/<app>/api/views.py` + `config/api_router.py` — routes.
4. OpenAPI via `drf-spectacular` (`api/schema/`, `api/docs/`) — machine-checkable contract.

State the gate: `uv run mypy bottom_up` must pass before TDD begins.

### 3. Decompose into subtasks

Write a small checklist; one Red-Green-Refactor cycle per item. Example:

```md
- [ ] model + migration (`makemigrations`)
- [ ] serializer + validation
- [ ] view + permissions + router entry
- [ ] celery task if async
- [ ] unit → API → acceptance tests
```

Each item must name its target test file (`bottom_up/<app>/tests/` or `tests/`) and the Gherkin scenario(s) it covers.

### 4. Output and stop

Output three sections and stop:

1. Spec summary (scenarios listed, gaps flagged).
2. Type contract (file order above).
3. Subtask checklist (per-item tests + scenarios).

End with exactly: `Awaiting approval — run /implement-feature spec=<same-path> tasks=<checklist> next.`

## Flags ($ARGUMENTS)

- `spec=plans/domains/<domain>/NN-<feature>.md` — required, single-file scope.
- `--dry-run` — same as default (this skill never writes).

## Boundaries

- NEVER call `write`, `edit`, or `apply_patch` under any circumstance.
- NEVER invent endpoints, env vars, ports, or versions — read them.
- NEVER commit secrets (`.env`, `.envs/.production/`, credentials in `.envs/.local/`).
- NEVER run destructive commands (`just prune`, `down -v`, `rm`, `push`, `commit`).
