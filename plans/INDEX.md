# Plans Index

> Read this first. Locate the domain or feature you need, load only that file, then follow its `requires_specs` frontmatter transitively. Do not load all specs eagerly.

Paths below are relative to `plans/`.

## Domains

| Domain | Contract | What it covers |
| --- | --- | --- |
| identity-access | domains/identity-access/domain.md | Roles, password policy, sessions, and email verification for all three roles. |
| discovery | domains/discovery/domain.md | Job search, filters, sorting, and shareable result URLs. |
| applications | domains/applications/domain.md | External-apply redirect with outbound tracking and dashboard status. |
| postings | domains/postings/domain.md | Job lifecycle, drafts, approval queue, and company verification. |

## Features

| ID | File | Description |
| --- | --- | --- |
| 01 | domains/identity-access/01-register.md | Role-aware registration with conditional fields, policy checkboxes, and email verification. |
| 02 | domains/identity-access/02-login.md | Email/password plus Google OAuth login with role-based dashboard redirect. |
| 03 | domains/identity-access/03-logout.md | Confirmed logout that invalidates the server session and clears local tokens. |
| 04 | domains/discovery/04-search-filter-jobs.md | Filtered and sorted job discovery with shareable URLs and an empty state. |
| 05 | domains/applications/05-apply-job.md | External “Apply Now” redirect that stays open with a tracking overlay modal. |
| 06 | domains/postings/06-post-job.md | Multi-step job posting with draft/publish flow and optional school approval. |

## Lazy-loading workflow

1. Start here to find the domain or feature file.
2. Read that single file.
3. Follow only the `requires_specs` paths listed in its frontmatter.

## Spec validator (Definition of Ready)

New `domains/<domain>/domain.md` files must copy `domains/TEMPLATE_DOMAIN.md`:

- Sections: Ubiquitous language, Entities & aggregates, State machine,
  Invariants (`INV-NN`), Authorization matrix, Domain events & side effects,
  Error contract, Open decisions. Keep `##` headings stable once `NN-*.md`
  files anchor to them.
- Mark proposed models/fields/endpoints with `(proposed)` until migrated/routed.

New `domains/<domain>/NN-*.md` files must copy `domains/TEMPLATE.md`:

- Frontmatter: `domain`, `requires_specs[]` (paths relative to `plans/`, always include `ARCHITECTURE.md` plus the domain contract anchor, e.g. `domains/discovery/domain.md#search-contract`).
- Sections: Goal, Non-goals, Actors, Preconditions, Postconditions, Gherkin `Feature:` with `Given/When/Then` scenarios, Acceptance criteria (checkboxes, API/UI-observable), Open Questions.
- Reject if: no Gherkin block, no acceptance checkboxes, `requires_specs` missing `ARCHITECTURE.md`, or anchor does not match a `##` heading slug in the target `domain.md`.
