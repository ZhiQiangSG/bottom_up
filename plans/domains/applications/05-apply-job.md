---
domain: applications
requires_specs:
  - ARCHITECTURE.md
  - domains/applications/domain.md#outbound-tracking
  - domains/postings/domain.md#job-lifecycle
---

# 05 — Apply for a Job

| ID | 05 |
| --- | --- |
| Name | Apply for a Job |
| Description | Student reviews full job details on BottomUp and applies on the employer's external career page. |

## Goal

Open the employer's external application page in a new tab while keeping BottomUp open with a tracking overlay modal, logging the outbound redirect and updating the student dashboard status.

## Non-goals

- Internal (on-platform) application submission — this spec covers external-redirect only.
- Resume upload or profile autofill.
- Employer-side applicant tracking.

## Actors

- Student

## Preconditions

- Student is logged in.
- Job listing is active and stores a valid external application URL.

## Postconditions

- Employer page opened in a separate tab/window.
- Outbound redirect event logged for analytics.
- Dashboard status for the job updated to "External Redirect / Applied".

## Gherkin scenarios

```gherkin
Feature: Apply for a Job
  Scenario: External apply opens new tab with tracking overlay
    Given the student is viewing an active listing with a valid external URL
    When the student clicks "Apply Now"
    Then the employer career page opens in a new tab
    And the BottomUp tab shows an overlay modal ("We've opened the job in a new tab! Did you complete your application?") with a "Track Application" button
    And an outbound redirect event is logged
    And the dashboard status becomes "External Redirect / Applied"

  Scenario: Expired listing disables apply
    Given the listing was deactivated after the page loaded
    When the student clicks "Apply Now"
    Then the button is disabled, status shows "Expired", and a toast explains the role is no longer accepting submissions

  Scenario: Malformed or dead external link handled
    Given the stored external URL is malformed or the employer page returns 404/500
    When the student clicks "Apply Now"
    Then apply is blocked with an explanatory toast and no tracking event is recorded as successful
```

## Acceptance criteria

- [ ] Detail page shows "Apply Now" button, leaving-platform disclaimer, and click-tracking behind the button.
- [ ] Click opens external URL in a new tab, keeps BottomUp open with overlay modal + "Track Application" button, logs redirect, updates dashboard to "External Redirect / Applied".
- [ ] Expired/deactivated listing disables the button with "Expired" status + toast; malformed/dead links are blocked with an explanatory toast.
- [ ] OpenAPI schema exposes the tracking endpoint (`GET /api/schema/`) once implemented.

## Open questions

- Tracking API shape (event payload + dashboard status enum value).
- Deadline-vs-expiry race handling when the window closes mid-read.
