---
domain: identity-access
requires_specs:
  - ARCHITECTURE.md
  - domains/identity-access/domain.md#rbac-roles
  - domains/identity-access/domain.md#session-management
  - domains/identity-access/domain.md#email-verification
---

# 02 — Log In

| ID | 02 |
| --- | --- |
| Name | Log In |
| Description | User logs into an existing account with email/password or Google OAuth and lands on the role dashboard. |

## Goal

Authenticate a registered user via email/password or Google OAuth, issue a session cookie/token, and redirect to the respective role dashboard.

## Non-goals

- Password recovery page implementation beyond linking to it from the login form.
- Registration and email verification (covered by `01-register.md`).
- MFA flows (allauth `mfa` package present but out of scope for this spec).

## Actors

- Student
- Employer
- School Admin

## Preconditions

- User has a registered account.
- User has no active session in the current browser.

## Postconditions

- Active session cookie/token is issued.
- User is redirected to the respective role dashboard.

## Gherkin scenarios

```gherkin
Feature: Log In
  Scenario: Successful email and password login
    Given a verified user exists with a known email and password
    When the user submits the login form with correct credentials
    Then an active session cookie/token is issued
    And the user is redirected to the respective role dashboard

  Scenario: Invalid credentials show generic error
    Given a user account exists
    When the user submits with an unknown email or wrong password
    Then a generic "Invalid email or password" error is shown
    And the password field is cleared
    And the user stays on the login page with retry and Forgot Password options

  Scenario: Missing fields blocked
    Given the user is on the login page
    When the user submits with an empty email or password field
    Then a required-field error is shown and no session is issued

  Scenario: Suspended or unverified account blocked
    Given the account is unverified or flagged suspended
    When the user submits correct credentials
    Then login is blocked or flagged with an account-status message
    And the user is not redirected to the dashboard

  Scenario: Successful Google OAuth login
    Given the user has a Google-linked account
    When the user completes Google authentication and Google returns an identity token
    Then the token is matched to the user record, a session is initialised, and the user is redirected to the role dashboard

  Scenario: OAuth failure or cancellation
    Given the user starts Google login
    When the third-party authentication fails or the user cancels mid-process
    Then an OAuth error is shown and the user stays on the login page with no session issued
```

## Acceptance criteria

- [ ] Login form shows email field, password field, Forgot Password link, Login button, and Log in with Google button.
- [ ] Correct credentials issue a session and redirect by role; wrong credentials show a generic error, clear password, stay on page.
- [ ] Unverified/suspended accounts are blocked or flagged at login per domain contract.
- [ ] OAuth cancel/failure shows an error with no session issued; auth service timeout shows a server-error message.

## Open questions

- Google OAuth scope and matching rule (email match vs explicit link) — see `plans/ARCHITECTURE.md` §12 open decisions.
- Role-dashboard route map per role.
