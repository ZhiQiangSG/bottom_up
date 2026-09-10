# Identity & Access — Domain Contract

> Roles, credentials, sessions, and verification shared by all features in this domain.
> Owns who a user is and what they may do. Does not own jobs, search, or tracking.

## Ubiquitous language

| Term | Definition |
| --- | --- |
| Role | One of `student`, `employer`, `school_admin`; fixed at registration. |
| Unverified account | Newly registered account with `is_verified=False`; must confirm email before full access. |
| Suspended account | Account flagged `is_suspended=True` by an admin; login is blocked. |
| Temporary session | Authenticated but unverified session issued at registration; limited to onboarding/verification pages. |
| Verification email | Email carrying a single-use token that flips `is_verified` to `True`. |

## Entities & aggregates

| Entity | Key fields (type) | Relations | Notes |
| --- | --- | --- | --- |
| `users.User` (exists) | `email: EmailField unique`, `name: CharField`, `password: hashed`, `USERNAME_FIELD=email`, `username=None`, `first_name=None`, `last_name=None` | — | See `ARCHITECTURE.md` §5; Argon2 hasher. |
| `Role` (proposed) | `user: FK settings.AUTH_USER_MODEL`, `role: enum(student, employer, school_admin)`, `is_verified: bool=False`, `is_suspended: bool=False` | One-to-one with user | Open decision: `User.role` field vs separate profile models (see `## Open decisions`). |
| Student profile (proposed) | `university: str`, `graduation_year: int`, `major: str` | FK to user | Conditional fields from `01-register.md`. |
| Employer profile (proposed) | `company_name: str`, `website_url: URL`, `company_approved: bool=False`, `company_verified: bool=False` | FK to user | Gates posting (see `domains/postings/domain.md#company-verification`). |
| School Admin profile (proposed) | `university: str`, `department: str`, `job_title: str` | FK to user | Requires approved (non-public) email domain. |

## RBAC Roles

Student, Employer, and School Admin roles. Registration assigns a role; login redirects by role; posting and approval actions require Employer or School Admin roles.

Role-dashboard route map (proposed): `student` → student onboarding dashboard, `employer` → employer dashboard, `school_admin` → career-center dashboard. Exact routes are an open decision (see `02-login.md`).

## Password Policy

Mandatory fields populated, passwords match and meet complexity rules, email format valid. Inline errors preserve input on failure.

Concrete rules:

- Minimum length 8; at least one letter and one digit (Django `MinimumLengthValidator(8)` + custom check; see `ARCHITECTURE.md` §5 allauth `ACCOUNT_SIGNUP_FIELDS=[email*, password1*, password2*]`).
- `password1` must equal `password2`; mismatch is a field error, no record created.
- Email must parse as a valid address and be unique (case-insensitive); duplicates return an account-conflict error without revealing which field matched beyond the email itself.

## Session Management

Login issues a session cookie/token. Logout invalidates the server session and clears local tokens/cookies, then redirects to the public landing page. Network failure mid-logout still wipes local tokens.

Concrete rules:

- Browser UI uses allauth session cookie; API clients use DRF `Token` via `POST /api/auth-token/` (see `ARCHITECTURE.md` §5).
- `03-logout.md` confirmation modal is required; cancel leaves the session untouched.
- Scoped to current browser session only; multi-device revocation is out of scope.

## Email Verification

New accounts start unverified; a verification email is dispatched at registration. Suspended or unverified accounts are blocked or flagged at login.

Concrete rules (`ACCOUNT_EMAIL_VERIFICATION=mandatory`, see `ARCHITECTURE.md` §5):

- `is_verified=False` at creation; single-use token emailed via Mailpit locally (`:8025`).
- Unverified login attempt: blocked or flagged with an account-status message, no dashboard redirect.
- Suspended login attempt: blocked with a generic account-status message (no existence leak beyond the attempting account).
- Registration email/token failure surfaces a server-timeout error with no half-created authenticated session (see `01-register.md` timeout scenario).

## State machine

| State | Meaning | Allowed next states |
| --- | --- | --- |
| `unverified` | Registered, email not confirmed | `active`, `suspended` |
| `active` | Verified, may log in fully | `suspended` |
| `suspended` | Admin-flagged, login blocked | `active` (unsuspend) |

## Invariants

- [ ] INV-01: No `User` row exists without exactly one role assignment.
- [ ] INV-02: `is_verified=False` accounts never reach a role dashboard beyond onboarding.
- [ ] INV-03: Failed registration (validation, duplicate, timeout) creates no user row and no session.
- [ ] INV-04: Logout always clears local tokens/cookies even when the server is unreachable.

## Authorization matrix

| Role | Action | Rule |
| --- | --- | --- |
| Anonymous | Register (`01`) | Allow when logged out and email unused |
| Anonymous | Log in (`02`) | Allow; generic error on bad credentials |
| student, employer, school_admin | Log out (`03`) | Allow own current session only |
| school_admin | Suspend/unsuspend account | Allow (admin only; proposed) |
| employer | Post job | Allow only when `company_approved` and `company_verified` |
| school_admin | Approve listing | Allow when approval queue is active |

## Domain events & side effects

| Event | Payload (JSON-serializable) | Produced by | Consumed by |
| --- | --- | --- | --- |
| `user.registered` | `{"user_id": "int", "role": "str"}` | Registration view | Verification-email Celery task |
| `verification.email_sent` | `{"user_id": "int"}` | Email task | Mailpit locally / Mailgun in prod |
| `user.logged_in` | `{"user_id": "int", "role": "str"}` | Login view / OAuth callback | Analytics log (no PII beyond id/role) |
| `user.logged_out` | `{"user_id": "int"}` | Logout view | Session invalidation |

## Error contract

| Condition | Observable outcome |
| --- | --- |
| Missing field / mismatched / weak password / bad email | 400 with inline field errors; input preserved; no record created |
| Duplicate email | Account-conflict error; no duplicate record |
| Missing Terms/Privacy checkboxes | Agreements-required error; no record created |
| Restricted domain for Student/School Admin | Domain-restriction error; no record created |
| Wrong email or password at login | Generic "Invalid email or password"; password cleared; stay on page |
| Unverified/suspended at login | Account-status message; no dashboard redirect |
| OAuth cancel/failure | OAuth error shown; no session issued |
| DB/email timeout mid-registration | Server-timeout error; no half-created session |

## Open decisions

- Role field on `User` vs separate profile models — see `ARCHITECTURE.md` §12; record in `plans/decisions/NNNN-role-storage.md` once decided.
- Google OAuth scope and account-matching rule (email match vs explicit link) — see `02-login.md` and `ARCHITECTURE.md` §12.
- Approved email-domain list owner and storage location — see `01-register.md`.
