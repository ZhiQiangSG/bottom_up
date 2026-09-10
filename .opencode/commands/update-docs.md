---
description: Audit repo and sync README + AGENTS + ARCHITECTURE + docs via update-docs skill
agent: docs-maintainer
subtask: false
---

Load and execute the `update-docs` skill using the `skill` tool.

Targets:
- @README.md
- @AGENTS.md
- @plans/ARCHITECTURE.md
- @docs/index.rst
- @CONTRIBUTING.md
- @plans/INDEX.md

Manifests: @pyproject.toml @justfile @docker-compose.local.yml @docker-compose.docs.yml
Router: @config/api_router.py
Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "update-docs" })` and follow it exactly.
2. Parse `$ARGUMENTS` for optional flags:
   - `target=readme|agents|arch|docs|all` (default `all`) — file scope.
   - `focus=<section>` (e.g. `focus=quickstart`, `focus=env`, `focus=status`) — scoped rewrite only.
   - `path=docs/<file>.rst` or `path=plans/ARCHITECTURE.md` — single-file scope.
   - `branch=<name>` — use that branch for history.
   - `--dry-run` — print proposals, make zero write/edit calls.
   - No args → full audit + sync of core targets (`README`, `AGENTS`, `ARCHITECTURE`, `docs/*.rst`).
3. Conditional targets: touch `CONTRIBUTING.md` only on workflow/command/CI changes; touch `plans/INDEX.md` only when a new domain/spec file was added; `plans/VISION.md` is proposal-only, never direct rewrite.
4. Prefer `read`/`glob`/`grep` tools and `@file` refs above. Use shell injection ONLY for git (no pipes):
   Recent commits: !`git log -n 20 --oneline`
   Status: !`git status --short`
   Change stat: !`git diff --stat main...HEAD`
5. If git commands fail (no repo), continue with filesystem audit and note it.
6. After rewrite (or dry-run output), summarize what changed per file and what custom content was preserved.

Usage:
- `/update-docs`
- `/update-docs target=readme focus=api`
- `/update-docs target=arch --dry-run`
- `/update-docs target=docs path=docs/users.rst`
- `/update-docs branch=feat/auth --dry-run`
