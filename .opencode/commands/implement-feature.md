---
description: Implement an approved feature spec via TDD loop in the feature-builder subagent
agent: feature-builder
subtask: true
---

Load and execute the `implement-feature` skill using the `skill` tool.

Targets:
- @plans/INDEX.md
- @CONTRIBUTING.md
- @AGENTS.md
- @config/api_router.py
- @pyproject.toml

Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "implement-feature" })` and follow it exactly.
2. Parse `$ARGUMENTS` for `spec=plans/domains/<domain>/NN-<feature>.md` (required) and `tasks=<approved-checklist>` (required). If either is missing, stop and ask. Optional pass-through flags: `--docker` / `--quick` (forwarded to the `qa` skill for Verify + Secure gates), `--dry-run` (plan only, zero writes).
3. Lazy-load only that spec plus its `requires_specs`. Confirm the type contract before writing code; if missing, send the user back to `/spec-feature`.
4. Loop per subtask: failing pytest from Gherkin → minimal fix → `ruff + mypy + pytest`. Then verify via the `qa` skill (`--docker` selects `just pytest` + `just manage makemigrations --check`; `--quick` skips coverage + pre-commit) and secure (`pre-commit run --all-files` unless `--quick`).
5. If `--dry-run` is present, print the plan + first failing test sketch and make zero `write`/`edit` calls.
6. Summarize tests added, scenarios covered, and gates run. Do NOT commit/push unless explicitly asked.

Usage:
- `/implement-feature spec=plans/domains/discovery/04-search-filter-jobs.md tasks="<paste approved checklist>"`
- `/implement-feature spec=plans/domains/discovery/04-search-filter-jobs.md tasks="<checklist>" --dry-run`
- `/implement-feature spec=plans/domains/discovery/04-search-filter-jobs.md tasks="<checklist>" --docker --quick`
