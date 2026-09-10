# Discovery — Domain Contract

> How students find jobs.
> Owns querying, filtering, sorting, pagination, and shareable URLs over
> `live` jobs. Does not own job creation (`domains/postings/domain.md#job-lifecycle`)
> or apply tracking (`domains/applications/domain.md#outbound-tracking`).

## Ubiquitous language

| Term | Definition |
| --- | --- |
| Active listing | A job with `status == live AND deadline > now()`; the only rows this domain ever returns. |
| Query | Free-text search over title, company, description (`q`). |
| Filter | Multi-select predicate that narrows results without changing their order. |
| Sort | Result ordering: `newest`, `salary`, or `deadline`. |
| Shareable URL | Page URL carrying `q` + filters + sort so results reproduce on load. |
| Empty state | UI shown when zero listings match; suggests broader criteria. |

## Entities & aggregates

| Entity | Key fields (type) | Relations | Notes |
| --- | --- | --- | --- |
| `Job` read model (proposed) | `title: str`, `company_name: str`, `location: str`, `location_type: enum`, `deadline: datetime`, `salary_min/salary_max: int nullable`, `posted_at: datetime` | Reads `postings.Job`; no writes from this domain | Card shows title, company, location, deadline; click opens full details. |

This domain is read-only. Writes belong to postings; clicks belong to applications.

## Search Contract

Listings displayed newest-first by default. Queries and multi-select filters (role, location, arrangement, industry, salary, duration, timeframe) plus sorting (newest, salary, deadline) resolve against active listings. Page URL carries query parameters so results are shareable. Zero matches render an empty state suggesting broader criteria.

Canonical query parameters (proposed `GET /api/jobs/`; mark `(proposed)` until routed in `config/api_router.py`):

| Param | Type | Meaning |
| --- | --- | --- |
| `q` | `str`, max 200 chars | Free-text search; trimmed; special chars escaped, never raw SQL |
| `role` | `str[]` multi-select | Target-role match |
| `location` | `str[]` multi-select | Location match |
| `arrangement` | `enum(remote, hybrid, onsite)[]` | Location-type match |
| `industry` | `str[]` multi-select | Industry match |
| `salary_min`, `salary_max` | `int` | Salary overlap; bucket definitions open (see `## Open decisions`) |
| `duration` | `str` | Duration match |
| `timeframe` | `str` | Posting window match |
| `work_type` | `enum(internship, entry_level)` | See `04` work-type scenario |
| `sort` | `enum(newest, salary, deadline)`, default `newest` | `newest` = `posted_at DESC`; `salary` = `salary_max DESC`; `deadline` = soonest first |
| `page`, `page_size` | `int`, defaults `1` / `20`, max `100` | Pagination; out-of-range pages return empty list, not an error |

## State machine

No lifecycle — stateless. Activeness is derived at query time from
`domains/postings/domain.md#job-lifecycle` (`live` + future deadline).

## Invariants

- [ ] INV-01: Results contain only active listings; drafts, queued, and expired rows never appear.
- [ ] INV-02: Default sort is `newest` when no `sort` param is given.
- [ ] INV-03: The URL always reflects the active `q` + filters + sort (copy-paste reproduces results).
- [ ] INV-04: Oversized input (over 200 chars), special characters, or unknown enum values are sanitised; the database never errors (400 or sanitised 200, never 500).
- [ ] INV-05: Out-of-range pages return an empty list with intact filter state, not an error.

## Authorization matrix

| Role | Action | Rule |
| --- | --- | --- |
| student (logged in) | Search / filter / sort | Allow over active listings |
| Anonymous | Search | Deny — login required per `04-search-filter-jobs.md` preconditions (redirect to login) |
| employer, school_admin | Search | Allow (same active-listing scope; no privileged rows) |

Permission shape follows `ARCHITECTURE.md` §5: `IsAuthenticated`, queryset scoped to active listings.

## Domain events & side effects

| Event | Payload (JSON-serializable) | Produced by | Consumed by |
| --- | --- | --- | --- |
| `search.performed` | `{"user_id": "int", "q": "str", "filters": "object"}` | Search view | Analytics log (optional; no PII beyond id) |

No emails or Celery tasks in this domain.

## Error contract

| Condition | Observable outcome |
| --- | --- |
| Zero matches | Empty state with broaden-search guidance; filters preserved |
| Oversized query (>200 chars) or special characters | Validated/sanitised; no DB error; results or empty state returned |
| Unknown `sort` or enum value | Falls back to default (`newest`) or 400 with field error; never 500 |
| Anonymous access | Redirect to login (UI) or 403/401 (API) |

## Open decisions

- Final router path and query-param names for `GET /api/jobs/` (proposed) — confirm in implementation plan.
- Salary-range bucket definitions and deadline semantics for internships.
