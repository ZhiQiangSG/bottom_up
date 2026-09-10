# Domain Contract Template — Full DDD

> Copy this file to `plans/domains/<domain>/domain.md` when creating a new
> domain. Keep every `##` heading below (add domain-specific subsections
> underneath) so `requires_specs` anchors in `NN-*.md` files keep resolving
> (see `plans/INDEX.md` validator). Paths in this template are relative to
> `plans/`. Link `ARCHITECTURE.md`; do not duplicate it.

# <Domain> — Domain Contract

> One paragraph: what this domain owns and what it explicitly does not own.

## Ubiquitous language

| Term | Definition |
| --- | --- |
| <Term> | <One-sentence, test-observable meaning.> |

Rules:

- Use these exact terms in `NN-*.md` Gherkin scenarios and in code
  (`models.py`, `serializers.py` field names).
- Every status/state word used in a `Then` step must appear here or in
  `## State machine`.

## Entities & aggregates

| Entity | Key fields (type) | Relations | Notes |
| --- | --- | --- | --- |
| <Entity> | `<field>: <type>` | FK to `settings.AUTH_USER_MODEL`; never import `User` directly | <Proposed until migrated.> |

Rules:

- Mark proposed models/fields with `(proposed)` until the migration is committed.
- New apps live under `bottom_up/<app>/`; register in `LOCAL_APPS`
  (`config/settings/base.py`); expose via `router.register()` in
  `config/api_router.py` (see `ARCHITECTURE.md` §3, §5, §6).

## State machine

| State | Meaning | Allowed next states |
| --- | --- | --- |
| <State> | <When an entity is in this state.> | <Transitions.> |

Rules:

- State names are the canonical enum values used in API responses and
  dashboard UI. Every transition must be exercised by at least one Gherkin
  scenario in an `NN-*.md` file.
- If the domain has no lifecycle, write "No lifecycle — stateless." and
  delete the table.

## Invariants

- [ ] INV-01: <Always-true business rule, API/UI-observable.>
- [ ] INV-02: <E.g. Draft jobs are never returned by the search endpoint.>

Rules:

- Number invariants (`INV-01`, …). Each one maps to at least one pytest.

## Authorization matrix

| Role | Action | Rule |
| --- | --- | --- |
| <Role> | <Action> | Allow / Deny + condition |

Roles are Student, Employer, School Admin unless the domain narrows them.

## Domain events & side effects

| Event | Payload (JSON-serializable) | Produced by | Consumed by |
| --- | --- | --- | --- |
| <event.name> | `{"<field>": "<type>"}` | <View/task> | <Celery task / email / analytics> |

Rules:

- Celery task args must be JSON-serializable; test with
  `CELERY_TASK_ALWAYS_EAGER=True` (see `ARCHITECTURE.md` §7).
- Emails go via Mailpit locally (`:8025`), Mailgun in production
  (see `ARCHITECTURE.md` §4).

## Error contract

| Condition | Observable outcome |
| --- | --- |
| <Invalid/expired/unauthorized condition> | <HTTP status or UI message; no partial state change.> |

Rules:

- Every row maps to a Gherkin error scenario and a pytest.
- Never leak existence details on auth failures (generic messages).

## Open decisions

- <Unresolved contract choice — link to `plans/decisions/NNNN-*.md` once decided.>
