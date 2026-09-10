---
domain: <domain-name>
requires_specs:
  - ARCHITECTURE.md
  - domains/<domain-name>/domain.md#<anchor>
---

# NN — <Feature Name>

| ID | NN |
| --- | --- |
| Name | <Feature Name> |
| Description | <One sentence: who does what.> |

## Goal

<What this feature achieves for the user. 1–3 sentences.>

## Non-goals

- <Explicitly out of scope.>
- <E.g. Social Sign-On deferred if timeframe does not permit.>

## Actors

- <Role 1>
- <Role 2>

## Preconditions

- <Must be true before the flow starts.>
- <E.g. User has no pre-existing account for the email.>

## Postconditions

- <Must be true after the ideal flow.>
- <E.g. Account created unverified, verification email dispatched.>

## Gherkin scenarios

```gherkin
Feature: <Feature Name>
  Scenario: <Ideal path>
    Given <context>
    When <action via API/UI>
    Then <observable outcome>

  Scenario: <Validation / error path>
    Given <context>
    When <invalid action>
    Then <inline error, input preserved, no state change>

  Scenario: <Second error / edge case>
    Given <context>
    When <action>
    Then <observable outcome>
```

Rules:

- Every scenario must be observable via API response or UI change.
- Every `Then` must map to at least one pytest (`test_views` / `test_urls` / `test_openapi` style).
- Keep API paths as actually routed in `config/api_router.py`; mark proposed paths with `(proposed)` until implemented.

## Acceptance criteria

- [ ] <API/UI-observable check, e.g. `POST /api/...` returns 201 + verification email queued.>
- [ ] <Validation check, e.g. missing field returns 400 with field errors, input preserved.>
- [ ] <State check, e.g. dashboard status updates to "...".>
- [ ] OpenAPI schema exposes the endpoint (`GET /api/schema/`).

## Open questions

- <Unresolved contract choice, e.g. exact router path or role-field location.>
- <Link to `plans/decisions/NNNN-*.md` once decided.>

## Requires specs

Paths below are relative to `plans/` (see `plans/INDEX.md` lazy-load rule). Follow only the `requires_specs` frontmatter above transitively.
