---
description: Read-only Django vulnerability audit (deploy posture, authz, injection, secrets, pip-audit) via the security-auditor subagent
agent: security-auditor
subtask: true
---

Load and execute the `security-audit` skill using the `skill` tool.

Targets:
- @config/settings/base.py
- @config/settings/production.py
- @config/api_router.py
- @pyproject.toml
- @uv.lock

Context overrides: $ARGUMENTS

Instructions:

1. Call `skill({ name: "security-audit" })` and follow it exactly.
2. Parse `$ARGUMENTS` for optional flags (all optional, defaults shown):
   - `--quick` — Gates A + B + D only (settings + authz + secrets; skips pip-audit + deep grep).
   - `--full` — default, all gates A–E.
   - `--dry-run` — print the gate plan, make zero `write`/`edit` calls, run zero fixing commands.
   - No args → full read-only run.
3. Run gates in order: deploy posture (`check --deploy`, report-only) → authz → injection/XSS grep + `S/BLE/DJ` triage → secrets guard → `pip-audit --desc` (unless `--quick`).
4. On finding, report Severity + file:line + evidence (redacted) + fix hint with handoff to `/implement-feature`. Do NOT fix, do NOT upgrade deps.
5. Report to stdout only. Summarize counts per severity, top-3 next actions, and what was skipped (`--quick`). Do NOT commit/push unless explicitly asked.

Usage:
- `/security-audit`
- `/security-audit --quick`
- `/security-audit --dry-run`
