# Postings — Domain Contract

> How jobs come to exist and go live.
> Owns the `Job` aggregate, drafts, validation, and the approval queue.
> Does not own search ranking (`domains/discovery/domain.md#search-contract`)
> or application tracking (`domains/applications/domain.md#outbound-tracking`).

## Ubiquitous language

| Term | Definition |
| --- | --- |
| Job | An internship or entry-level opening created by an employer; searchable only when `Live`. |
| Draft | Partially filled job saved without validation; invisible to students. |
| Pending Review | Fully validated job awaiting School Career Center approval. |
| Live | Approved and searchable job with a future deadline. |
| Expired | Past-deadline or deactivated job; apply is disabled. |
| Application Method | `internal` (deferred) or `external_url`; today only `external_url` is implemented. |

## Entities & aggregates

| Entity | Key fields (type) | Relations | Notes |
| --- | --- | --- | --- |
| `Job` (proposed) | `title: str(200)`, `description: rich text`, `requirements: text`, `job_type: enum(internship, entry_level)`, `target_roles: str[]`, `location_type: enum(remote, hybrid, onsite)`, `location: str`, `salary_min/salary_max: int nullable`, `duration: str nullable`, `deadline: datetime`, `status: JobStatus`, `application_method: enum(external_url, internal)`, `external_url: URL nullable`, `posted_by: FK settings.AUTH_USER_MODEL` | FK to user; never import `User` directly | New app `bottom_up/postings/` + `LOCAL_APPS` + migration (see `ARCHITECTURE.md` §3, §6). |
| Approval item (proposed) | `job: FK Job`, `reviewed_by: FK settings.AUTH_USER_MODEL nullable`, `decision: enum(approved, rejected) nullable` | FKs to job and reviewer | Only when school approval is active. |

Mandatory-field matrix for Publish: title, description, requirements, target roles,
deadline (future), location type, salary/stipend, application method. Draft skips validation.

## Job Lifecycle

Draft → Live (or Pending Review) → Active/Searchable → Expired. Publish requires title, description, requirements, target roles, deadline, location type, salary/stipend, and application method. Validation failures halt submission, highlight fields, and preserve input. Drafts save partial input.

## State machine

| State | Meaning | Allowed next states |
| --- | --- | --- |
| `draft` | Partial input saved via "Save as Draft" | `pending_review`, `live` |
| `pending_review` | Validated; queued for Career Center | `live`, `draft` (returned), `expired` |
| `live` | Searchable by eligible students | `expired` |
| `expired` | Past deadline or deactivated | — (terminal) |

`Active/Searchable` = `status == live AND deadline > now()`. Expiry is evaluated at
read time; a background beat task may flip `live` → `expired` (see `ARCHITECTURE.md` §7).

## Approval Queue

Listings may route to the School Career Center approval queue before becoming searchable by eligible students.

Concrete rules:

- Queue is active only when the school-approval flag is on; otherwise validated Publish goes straight to `live`.
- Queue owner: School Career Center (`school_admin` role); eligibility filter for which students see approved jobs is open (see `## Open decisions`).
- Employer sees `Pending Review` status + notification; students never see queued items (INV-02).

## Company Verification

Posting requires a logged-in, approved, verified company account. Student/School Admin registration with unapproved or generic public email domains is rejected.

Concrete rules:

- `company_approved AND company_verified` must both be true to open or submit the posting form; otherwise reject with a verification-required message (see `06-post-job.md` scenario 4).
- Approved email-domain list owner/storage is open (see `01-register.md` and `## Open decisions`).

## Invariants

- [ ] INV-01: Only `live` jobs with a future deadline are searchable (drafts, queued, expired never leak).
- [ ] INV-02: `pending_review` jobs are visible only to the posting employer and `school_admin` reviewers.
- [ ] INV-03: Publish validates all mandatory fields; failure halts with field-level errors and preserved input; no state change.
- [ ] INV-04: `external_url` application method requires a valid absolute HTTPS URL; `internal` is rejected as deferred until its contract exists.
- [ ] INV-05: Deadline must be in the future at Publish time; past deadlines are a validation error, not silent expiry.

## Authorization matrix

| Role | Action | Rule |
| --- | --- | --- |
| employer (verified company) | Create draft / Publish | Allow |
| employer (unverified company) | Open or submit posting form | Deny with verification-required message |
| student | Post or edit job | Deny |
| school_admin | Approve / return / reject queued job | Allow when queue is active |
| school_admin | Post job | Deny (admins review, they do not post) |

## Domain events & side effects

| Event | Payload (JSON-serializable) | Produced by | Consumed by |
| --- | --- | --- | --- |
| `job.draft_saved` | `{"job_id": "int", "employer_id": "int"}` | "Save as Draft" view | Dashboard toast "Draft Saved Successfully" |
| `job.published` | `{"job_id": "int", "status": "str"}` | Publish view | Success notification; optional employer email |
| `job.queued_for_review` | `{"job_id": "int"}` | Publish view (approval on) | Career Center queue; reviewer notification |
| `job.expired` | `{"job_id": "int"}` | Deadline check / beat task | Search-index removal; detail page shows "Expired" |

## Error contract

| Condition | Observable outcome |
| --- | --- |
| Publish with missing mandatory field or over-limit rich text | Submission halts; field-level errors with instructions; input preserved |
| Past or missing deadline | Validation error; no state change |
| Unverified/unapproved company attempts posting | Verification-required message; form blocked |
| `external_url` malformed or non-HTTPS | Field error; publish blocked |
| `internal` method selected today | Deferred-method error pointing to future applications spec |

## Open decisions

- Approval queue owner and eligibility filter for searchable students — see `ARCHITECTURE.md` §12; record in `plans/decisions/NNNN-approval-queue.md` once decided.
- Internal-apply method contract (deferred to a future applications spec) — see `06-post-job.md`.
- Approved email-domain list owner and storage location (shared with identity-access).
