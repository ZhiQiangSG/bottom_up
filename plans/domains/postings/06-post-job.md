---
domain: postings
requires_specs:
  - ARCHITECTURE.md
  - domains/identity-access/domain.md#rbac-roles
  - domains/postings/domain.md#company-verification
  - domains/postings/domain.md#job-lifecycle
  - domains/postings/domain.md#approval-queue
---

# 06 — Post a New Job Opening

| ID | 06 |
| --- | --- |
| Name | Post a New Job Opening |
| Description | An employer creates, configures, and publishes a new internship or entry-level job listing. |

## Goal

Let a verified employer draft, validate, and publish a job (directly Live or via Pending Review) so it becomes searchable by eligible students.

## Non-goals

- Student discovery ranking (covered by `04-search-filter-jobs.md`).
- External application tracking (covered by `05-apply-job.md`).
- Bulk import or scraper ingestion.

## Actors

- Employer
- School Career Center (approver, only when school approval is active)

## Preconditions

- Employer is logged in with an approved and verified company account.

## Postconditions

- Job is saved as Draft, Live, or Pending Review.
- Live jobs become searchable by eligible students; Pending Review jobs route to the Career Center approval queue.

## Gherkin scenarios

```gherkin
Feature: Post a New Job Opening
  Scenario: Successful publish goes live or to review
    Given the employer is logged in with a verified company account
    When the employer completes Title, Description, Requirements, Target Roles, Deadline, Location Type, Salary/Stipend, and Application Method, then clicks "Publish"
    Then the record is saved as "Live" (or "Pending Review" when school approval is active)
    And a success notification is shown
    And the job becomes searchable once active

  Scenario: Save as draft preserves partial input
    Given the employer started the multi-step form
    When the employer clicks "Save as Draft"
    Then partial input is saved as Draft
    And the employer returns to the dashboard with a "Draft Saved Successfully" toast

  Scenario: Publish with missing or over-limit fields blocked
    Given the employer clicks "Publish" with missing mandatory metadata or over-limit rich text
    When validation runs
    Then submission halts with field-level errors
    And entered input is preserved on the page

  Scenario: Unverified company cannot post
    Given the employer account is not approved/verified
    When the employer attempts to open or submit the posting form
    Then posting is rejected with a verification-required message
```

## Acceptance criteria

- [ ] "Post a Job" multi-step form collects Job Title, Description, Job Type, Location Type, Deadline, Salary/Stipend, Target Audience selectors, and Application Method toggle (Internal vs External URL).
- [ ] "Publish" validates all mandatory fields; success saves Live (or Pending Review with school approval on) with success notification; optional employer email sent.
- [ ] "Save as Draft" persists partial input and toasts on return to dashboard.
- [ ] Validation failures halt, highlight fields with instructions, preserve input; unverified companies are rejected.
- [ ] OpenAPI schema exposes the postings endpoint (`GET /api/schema/`) once implemented.

## Open questions

- Approval queue owner and eligibility filter for searchable students — see `plans/ARCHITECTURE.md` §12.
- Internal-apply method contract (deferred to a future applications spec).
