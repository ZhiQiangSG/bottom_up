# Applications — Domain Contract

> What counts as applying.
> Owns the external-redirect flow, outbound tracking events, and the student
> dashboard status. Does not own job creation (`domains/postings/domain.md#job-lifecycle`)
> or search (`domains/discovery/domain.md#search-contract`).

## Ubiquitous language

| Term | Definition |
| --- | --- |
| External apply | Opening the employer's career page in a new tab; the application itself completes off-platform. |
| Tracking overlay | BottomUp modal kept open after redirect ("We've opened the job in a new tab! Did you complete your application?") with a "Track Application" button. |
| Outbound redirect event | Analytics record that Apply was clicked for a job by a student at a timestamp. |
| Dashboard status | Per-student-per-job label; canonical values in `## State machine`. |
| Internal apply | Deferred on-platform submission (resume/profile); not implemented — see `## Internal Apply (deferred)`. |

## Entities & aggregates

| Entity | Key fields (type) | Relations | Notes |
| --- | --- | --- | --- |
| `ApplicationEvent` (proposed) | `student: FK settings.AUTH_USER_MODEL`, `job: FK postings.Job`, `outcome: enum(redirected, expired_blocked, link_blocked)`, `created_at: datetime` | FKs to user and job | Append-only; one row per Apply click (idempotent per click, not deduped). |
| Dashboard status (derived, proposed) | `status: enum(not_applied, external_redirect_applied, expired)` | Derived from latest event + job liveness | Displayed on student dashboard; `external_redirect_applied` renders as "External Redirect / Applied". |

## Outbound Tracking

Applying opens the employer's external career page in a new tab; the BottomUp tab stays open with a tracking overlay modal. Every outbound redirect is logged for analytics and the student's dashboard status updates to "External Redirect / Applied". Expired or malformed links disable apply with a toast.

Concrete rules:

- "Apply Now" button carries click-tracking; a leaving-platform disclaimer is shown on the detail page.
- Click opens `external_url` in a new tab (`rel=noopener`), keeps BottomUp open, shows the overlay modal + "Track Application" button, logs the event, and sets dashboard status to `external_redirect_applied`.
- `external_url` must be absolute HTTPS; malformed URLs or employer pages returning 404/500 block apply with an explanatory toast and record `link_blocked` (never `redirected`).
- Listings deactivated or past deadline disable the button with "Expired" status + toast and record `expired_blocked`.
- Repeat clicks append repeat `redirected` events (no dedupe); status stays `external_redirect_applied`.

## State machine

Dashboard status per student per job:

| State | Meaning | Allowed next states |
| --- | --- | --- |
| `not_applied` | No successful redirect yet (initial) | `external_redirect_applied`, `expired` |
| `external_redirect_applied` | At least one `redirected` event logged | `expired` (job later expires) |
| `expired` | Job deactivated or past deadline | — (terminal) |

## Internal Apply (deferred)

`06-post-job.md` offers an Application Method toggle (Internal vs External URL).
`internal` is deferred: no submission endpoint, resume upload, autofill, or
employer-side applicant tracking exists in this spec. Selecting `internal` at
posting time is rejected per `domains/postings/domain.md#error-contract` until
a future applications spec defines its contract.

## Invariants

- [ ] INV-01: A `redirected` event is logged only when a new tab was actually opened with a valid URL.
- [ ] INV-02: Blocked applies (`expired_blocked`, `link_blocked`) never set `external_redirect_applied`.
- [ ] INV-03: Expired/deactivated listings always disable Apply with "Expired" status + toast, even if the page loaded while live (deadline-vs-expiry race).
- [ ] INV-04: Tracking calls are authenticated as the clicking student; students cannot log events for other users.

## Authorization matrix

| Role | Action | Rule |
| --- | --- | --- |
| student (logged in) | Click "Apply Now" on a live job | Allow; logs event as self |
| Anonymous | Apply / tracking endpoint | Deny — login required |
| employer | Log apply events for students | Deny |
| school_admin | Read aggregate redirect counts | Allow (analytics only; proposed) |

Permission shape follows `ARCHITECTURE.md` §5: `IsAuthenticated`, queryset scoped to `request.user`.

## Domain events & side effects

| Event | Payload (JSON-serializable) | Produced by | Consumed by |
| --- | --- | --- | --- |
| `application.redirected` | `{"student_id": "int", "job_id": "int", "url": "str", "timestamp": "str"}` | Apply click / tracking endpoint | Analytics store; dashboard status update |
| `application.blocked` | `{"student_id": "int", "job_id": "int", "reason": "expired\|link_invalid"}` | Apply click guard | Toast; analytics store (unsuccessful) |

Tracking endpoint shape is open (see `05-apply-job.md`); payload above is the
proposed contract. No emails or Celery tasks required today (fire-and-forget log write).

## Error contract

| Condition | Observable outcome |
| --- | --- |
| Listing expired/deactivated at click time | Button disabled; "Expired" status + toast; `expired_blocked` logged; no new tab |
| Malformed `external_url` or employer page 404/500 | Apply blocked with explanatory toast; `link_blocked` logged; no `redirected` event |
| Anonymous tracking call | 401/403; no event recorded |
| Tracking call for another user | 403; no event recorded |

## Open decisions

- Tracking API shape (event payload + dashboard status enum value) — proposed above; confirm in implementation plan.
- Deadline-vs-expiry race handling when the window closes mid-read (guard evaluates liveness at click time; see INV-03).
