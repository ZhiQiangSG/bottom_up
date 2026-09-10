---
domain: identity-access
requires_specs:
  - ARCHITECTURE.md
  - domains/identity-access/domain.md#session-management
---

# 03 — Log Out

| ID | 03 |
| --- | --- |
| Name | Log Out |
| Description | User securely terminates the active session via a confirmation modal and returns to the public landing page. |

## Goal

Invalidate the server session, clear local tokens/cookies, and redirect to the public landing page so private pages require re-authentication.

## Non-goals

- Multi-device session revocation beyond the current browser session.
- Account deletion or deactivation.

## Actors

- Student
- Employer
- School Admin

## Preconditions

- User is logged into an active session in the current browser.

## Postconditions

- Server session invalidated; local tokens/cookies destroyed.
- User redirected to the public landing page; back-button access to private pages requires re-authentication.

## Gherkin scenarios

```gherkin
Feature: Log Out
  Scenario: Confirmed logout ends session
    Given the user is logged in
    When the user clicks "Log Out", confirms in the modal, and the server invalidates the session
    Then local tokens/cookies are cleared
    And a "Successfully logged out" toast is shown
    And the user is redirected to the public landing page

  Scenario: Cancelled logout keeps session
    Given the user is logged in and the confirmation modal is open
    When the user clicks "Cancel"
    Then the modal closes and the existing session is untouched

  Scenario: Network drops mid-logout still clears locally
    Given the user confirmed logout
    When the network drops before the server responds
    Then local auth tokens are still wiped and the user is forced to the landing page
```

## Acceptance criteria

- [ ] "Log Out" button opens a confirmation modal ("Are you sure you want to logout?") preventing accidental sign-out.
- [ ] Confirm invalidates server session, clears local tokens/cookies, shows success toast, redirects to landing page.
- [ ] Cancel closes modal with session untouched; back button cannot reach private pages without re-authentication.
- [ ] Server/network failure still wipes local tokens and lands on the public page with an error message.

## Open questions

- None — contract fully covered by `domains/identity-access/domain.md#session-management`.
