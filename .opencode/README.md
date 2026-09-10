# .opencode — agents, commands, skills

> Main agent plans (`/spec-feature`), subagents execute.
> All commands support `--dry-run` (plan only, zero writes).
> Never commit/push/prune volumes unless explicitly asked.

## Skill vs command vs agent (read this first)

- **Skill** (`skills/<name>/SKILL.md`) is the workflow — the source of truth
  for gate order, flags, and boundaries. Loaded via `skill({ name: "<name>" })`.
- **Command** (`commands/<name>.md`) is the thin wrapper you type
  (`/spec-feature`, `/qa`, …). It parses `$ARGUMENTS`, lists `@file` targets,
  and delegates to exactly one skill (or, for `/audit-all`, fans out to three).
- **Agent** (`agents/<name>.md`) is the permissioned runner for
  `subtask: true` commands. It sets `edit`/`bash` permissions, temperature,
  and the build-vs-verify boundary. `subtask: false` commands run in the
  main agent instead.

Rule: if skill and command disagree, the skill wins. Commands never add
gates the skill does not define.

## Commands (what to type)

| Command | Runner | Skill | Read/write | Use when |
|---|---|---|---|---|
| `/spec-feature spec=plans/domains/<d>/NN-*.md` | main (`subtask: false`, read-only) | `spec-feature` | zero writes | Draft BDD spec review + type contract + subtask checklist; stops for approval |
| `/implement-feature spec=... tasks="<checklist>" [--dry-run] [--docker] [--quick]` | `feature-builder` (`subtask: true`) | `implement-feature` (+ `qa` for gates) | writes | Implement approved spec via TDD Red-Green-Refactor |
| `/qa [--quick] [--docker] [--fix] [--dry-run]` | `qa-reviewer` (`subtask: true`) | `qa` | check-only (writes only with `--fix`, ruff only) | Ordered gates: `ruff → mypy → makemigrations --check → pytest → coverage → pre-commit` |
| `/code-audit [scope=debt\|smells\|arch\|all] [path=...] [--quick] [--dry-run]` | `code-auditor` (`subtask: true`) | `code-audit` | strict read-only, rejects `--fix` | Tech-debt / smells / arch drift; stdout report + top-5 paydown |
| `/security-audit [--quick] [--full] [--dry-run]` | `security-auditor` (`subtask: true`) | `security-audit` | strict read-only | Vulns: deploy → authz → injection/XSS → secrets → `pip-audit` |
| `/audit-all [--quick] [--docker] [scope=...] [path=...] [--dry-run]` | main (`subtask: false`, fans out) | `code-audit` + `security-audit` + `qa` | check-only, rejects `--fix` | Full read-only merge of the three auditors; stdout only, no report files |
| `/update-docs [target=readme\|agents\|arch\|docs\|all] [focus=...] [path=...] [branch=...] [--dry-run]` | `docs-maintainer` (`subtask: false`) | `update-docs` | writes | Sync `README/AGENTS/ARCHITECTURE/docs/*.rst`; `VISION.md` proposal-only |

## Agents (permissioned runners)

- `feature-builder` — the only builder. Requires `spec=` + approved `tasks=`;
  lazy-loads only that spec + `requires_specs`. Never plans from scratch.
- `qa-reviewer` — verifies, never builds. Without `--fix`, zero writes;
  with `--fix`, only Ruff (`check --fix`, `format`) may touch files.
- `code-auditor` — audits, never builds. Never runs `pytest`/`coverage`/
  migrations/`pre-commit`. `--fix` is rejected (→ `/qa --fix` or `/implement-feature`).
- `security-auditor` — verifies vulns, never fixes. Never upgrades deps,
  never `pip-audit --fix`, never `ruff --fix`. Never reads/prints secrets.
- `docs-maintainer` — syncs docs. Keeps `AGENTS.md` <~60 lines,
  `ARCHITECTURE.md` §numbers stable, `docs/index.rst` toctree valid.

Non-overlap: `/qa` owns gates pass/fail, `/code-audit` owns debt/smells,
`/security-audit` owns vulnerabilities, `/implement-feature` owns fixes.

## Examples

```text
/spec-feature spec=plans/domains/discovery/04-search-filter-jobs.md
/implement-feature spec=plans/domains/discovery/04-search-filter-jobs.md tasks="<approved checklist>"
/qa --quick
/qa --docker
/qa --fix
/security-audit --quick
/code-audit scope=smells path=bottom_up/users/
/audit-all --quick
/audit-all path=bottom_up/users/ scope=smells
/update-docs target=arch --dry-run
/update-docs target=readme focus=api
```

## Flags convention

- `--dry-run` (every command): print the plan, make zero `write`/`edit` calls,
  invoke zero subagents (`/audit-all`), run zero gate/fixing commands.
- `--quick`: skip the slow part (`/qa`: coverage + pre-commit;
  `/security-audit`: pip-audit + deep grep; `/code-audit`: smells only;
  `/audit-all` passes through to all three).
- `--docker` (`/qa`, `/audit-all`, `/implement-feature` verify): Docker
  CI-parity path (`just pytest`, `just manage makemigrations --check`);
  default is native (`uv run pytest`).
- `--fix` (`/qa` only): allow Ruff fixes. Rejected by `/code-audit`,
  `/security-audit`, and `/audit-all` — use `/implement-feature` for
  refactors/vulns instead.

## Permissions (`opencode.json` + agent frontmatter)

- Global deny: `edit .env`, `.envs/.production/**`, `.envs/.local/.django`,
  `.envs/.local/.postgres`; ask on `just prune*`, `docker *prune*`, `*down -v*`.
- Auditors (`code/security`) + QA (`qa-reviewer`): `edit: deny`.
  Builder/docs (`feature-builder`/`docs-maintainer`): `edit: allow` within scope.
- Secrets guard (every `/qa`, `/security-audit`, `/audit-all` run): fail if
  `.env`, `.envs/.production/`, or credentials in `.envs/.local/` are
  staged/modified. Report paths only, never contents.
