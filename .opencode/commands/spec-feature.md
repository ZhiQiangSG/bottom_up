---
description: Draft BDD spec + type contract + subtask checklist for a feature in the main agent. Read-only, stops for approval.
subtask: false
---

Load and execute the `spec-feature` skill using the `skill` tool.

Targets:
- @plans/INDEX.md
- @CONTRIBUTING.md
- @AGENTS.md
- @config/api_router.py
- @pyproject.toml

Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "spec-feature" })` and follow it exactly.
2. Parse `$ARGUMENTS` for `spec=plans/domains/<domain>/NN-<feature>.md` (required). If missing, ask for it and stop.
3. Lazy-load only that spec plus its `requires_specs`. Never load all specs eagerly.
4. Output spec summary + type contract + subtask checklist. Make ZERO `write`/`edit` calls.
5. End with the handoff line for `/implement-feature` and stop for human approval.

Usage:
- `/spec-feature spec=plans/domains/discovery/04-search-filter-jobs.md`
