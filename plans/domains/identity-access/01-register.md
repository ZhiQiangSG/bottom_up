---
domain: identity-access
requires_specs:
  - ARCHITECTURE.md
  - domains/identity-access/domain.md#rbac-roles
  - domains/identity-access/domain.md#password-policy
  - domains/identity-access/domain.md#email-verification
  - domains/postings/domain.md#company-verification
---

# 01 — Register New Account

| ID | 01 |
| --- | --- |
| Name | Register New Account |
| Description | User creates a new account as Student, Employer, or School Admin using email and password. |

## Goal

Let a new user select a role, submit role-specific details, and get an unverified account with a verification email dispatched, then enter a temporary session routed to role onboarding.

## Non-goals

- Social Sign-On buttons (Google / Microsoft OAuth) — optional scope, only if timeframe permits.
- Password recovery / reset flows (covered by login spec `02-login.md`).
- Company approval workflow beyond email-domain rejection at registration.

## Actors

- Student
- Employer
- School Admin

## Preconditions

- No pre-existing account exists for the provided email.
- User is not currently logged in.

## Postconditions

- New user + profile record created with the designated role; password securely hashed.
- Account starts unverified; verification email dispatched.
- User is authenticated into a temporary session and redirected to the role dashboard onboarding wizard.

## Gherkin scenarios

```gherkin
Feature: Register New Account
  Scenario: Successful role-aware registration
    Given the user is logged out and the email is not registered
    When the user selects role "Student" and submits email, password, matching confirmation, University Name, Expected Graduation Year, Major, plus Terms and Privacy checkboxes
    Then a user record with role "Student" is created unverified
    And a verification email is dispatched
    And the user is authenticated and redirected to the student onboarding dashboard

  Scenario: Validation failure preserves input
    Given the user is on the registration page
    When the user submits with a missing mandatory field, mismatched passwords, a weak password, or an invalid email format
    Then submission halts with inline field errors
    And entered input is preserved for correction
    And no user record is created

  Scenario: Duplicate email rejected
    Given the email is already registered
    When the user submits the registration form with that email
    Then submission halts with an account-conflict error
    And no duplicate record is created

  Scenario: Missing policy agreements rejected
    Given the user filled all fields correctly
    When the user submits without checking Terms and Privacy boxes
    Then submission halts with an agreements-required error

  Scenario: Unapproved domain rejected for School Admin and Student
    Given the user selects role "School Admin"
    When the user submits with a generic public email domain (e.g. gmail.com)
    Then submission halts with a domain-restriction error

  Scenario: Email service timeout surfaces safely
    Given the database or email service is unresponsive mid-transaction
    When the user submits a valid form
    Then a server-timeout error is shown and no half-created session is left authenticated
```

## Acceptance criteria

- [ ] Role radio buttons Student / Employer / School Admin toggle conditional fields (Student: University, Graduation Year, Major; Employer: Company Name, Website URL; School Admin: University, Department, Job Title).
- [ ] Valid submit creates unverified user with hashed password, dispatches verification email, authenticates temporary session, redirects to role dashboard.
- [ ] Invalid submit (bad email, weak/mismatched password, missing field, unchecked boxes, duplicate email, restricted domain) halts with inline errors and preserves input.
- [ ] OpenAPI schema exposes the registration endpoint (`GET /api/schema/`) once implemented.

## Open questions

- Exact registration route (allauth vs DRF endpoint) — see `plans/ARCHITECTURE.md` auth conventions.
- Approved email-domain list owner and storage location.
