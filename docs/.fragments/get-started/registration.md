---
title: User Registration and Accounts
description: Controlled registration of administrators, agents and farmers, and secure login.
---

# :material-account-multiple-plus: User Registration and Accounts

<div class="at-a-glance" markdown>

**In short:** a cooperative and its first administrator are created by public sign-up or a protected bootstrap endpoint. Administrators register agents, and agents register farmers. Every staff login needs a password and a code sent by email.

**Who uses it:** administrators and agents &nbsp;·&nbsp; **Where:** Admin dashboard, agent app, login pages &nbsp;·&nbsp; **API prefix:** `/auth`, `/users`

</div>

## What it does

- Creates the cooperative and its first administrator.
- Lets administrators add other administrators.
- Registers agents and ties each one to a collection point.
- Activates agent accounts through a one-time, expiring invitation link.
- Lets agents register farmers, identified by phone number.
- Protects every login with a password plus a 6-digit code sent by email.
- Records registration and activation events in the audit log.

## The four kinds of account

| Account | Created by | Signs in with | Sees |
| --- | --- | --- | --- |
| Administrator | Public sign-up, bootstrap endpoint, or another administrator | Email + password + emailed code | Everything in their cooperative |
| Agent | Administrator | Email + password + emailed code | Their collection point only |
| Farmer | Agent | No web login — phone number for USSD | Their own records |
| System job | Deployment | Service credentials | Background work only |

!!! info "Farmers have no password on purpose"
    A farmer does not need an account to be paid. Their phone number is their identity: it receives the SMS, it opens the USSD menu, and it is what the payment is sent to. This keeps the system usable for people without smartphones or email.

## Setting up a cooperative

1. **Sign up** at `POST /auth/register/cooperative` with cooperative name, email, first name, last name and password.
2. The backend creates the cooperative and the first administrator in one transaction.
3. The administrator signs in with the emailed code.
4. **Or use the bootstrap endpoint**, `POST /auth/bootstrap-admin`, with the same details plus the secret in the `X-Bootstrap-Key` header. Use it for a new server or a restore.

!!! info "Two doors, one room"
    Public sign-up is rate-limited to **5 attempts per 15 minutes per IP**. The bootstrap endpoint is off entirely when `BOOTSTRAP_SECRET` is empty, which is the default in production.

## Registering an agent

1. The administrator opens **Agents → New agent**, enters first name, last name, email and **collection point**.
2. The backend checks the email is unused and the point belongs to the cooperative, then creates the account as **pending**.
3. The agent receives an invitation email with a setup link, valid for **24 hours**.
4. The agent opens the link and sets a password (minimum 8 characters).
5. The backend checks the token is valid, unexpired and unused; hashes the password; activates the account; marks the email verified; clears the token; writes the audit entry.
6. The agent can now log in with email, password and emailed code.

!!! warning "Invitations expire"
    An expired link answers **410 INVITATION_EXPIRED**. The administrator resends it from **Agents → Resend invitation**, which invalidates the old link.

## Registering a farmer

1. The agent opens **Farmers → New farmer** in the agent app.
2. Enter first name, last name, **phone number** in the form `+2507XXXXXXXX` or `07XXXXXXXX`.
3. The phone number is normalised to E.164 before saving, so `0788123456` and `+250788123456` are the same farmer.
4. A farmer with the same normalised number in the same cooperative is refused: **409 FARMER_PHONE_ALREADY_REGISTERED**.
5. The farmer is created **active**, attached to the agent's collection point, with no password.

## Logging in

1. `POST /auth/login` with email and password. A wrong password answers **401 INVALID_CREDENTIALS** and is rate-limited after 5 failures.
2. A 6-digit code arrives by email, valid **10 minutes**.
3. `POST /auth/verify-otp` exchanges the code for an access token, valid **30 minutes**.
4. Every later request sends `Authorization: Bearer <token>`.
5. `POST /auth/resend-otp` sends a fresh code; the old one stops working.

| Control | Value | Why |
| --- | --- | --- |
| Password minimum | 8 characters | weak passwords are the cheapest attack |
| Login attempts | 5 per 15 minutes | slows brute force |
| Code lifetime | 10 minutes | short enough to be useless if leaked |
| Code attempts | 5 | stops guessing a 6-digit code |
| Token lifetime | 30 minutes | limits damage from a stolen token |
| Invitation lifetime | 24 hours | enough to act, short enough to expire |

## Security rules

- Passwords and setup tokens are stored **only as hashes** (bcrypt), never logged.
- Every request checks role: `require_admin`, `require_agent` or `require_staff`.
- Agents are scoped to their collection point at the query level, not the UI.
- A user from another cooperative answers **404**, never 403 — the system does not confirm the record exists.
- Emails are unique across the whole system; phone numbers unique per cooperative.

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `POST` | `/auth/register/cooperative` | public | Cooperative + first administrator |
| `POST` | `/auth/bootstrap-admin` | secret holder | Same, without the public page |
| `POST` | `/auth/login` | public | Password check, emails a code |
| `POST` | `/auth/verify-otp` | public | Code → access token |
| `POST` | `/auth/resend-otp` | public | New code, old one void |
| `POST` | `/auth/set-password` | invitation holder | Activates an agent |
| `GET` | `/auth/me` | any signed-in | Own account and role |
| `POST` | `/auth/logout` | any signed-in | Ends the session |
| `GET` | `/users/agents` | admin | Lists agents |
| `POST` | `/users/agents` | admin | Registers an agent |
| `POST` | `/users/agents/{id}/resend-invitation` | admin | Sends the setup email again |
| `GET` | `/users/farmers` | admin, agent | Lists farmers, scoped |
| `POST` | `/users/farmers` | agent | Registers a farmer |

??? example "Example: register a farmer"
    ```http
    POST /users/farmers
    Authorization: Bearer <agent_token>
    Content-Type: application/json

    { "first_name": "Alice", "last_name": "Uwase", "phone_number": "0788123456" }
    ```

    ```json
    201 Created
    { "id": "6c1d...", "first_name": "Alice", "last_name": "Uwase",
      "phone_number": "+250788123456", "is_active": true,
      "collection_point_id": "0b9f..." }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 401 | `OTP_INVALID` / `OTP_EXPIRED` | Wrong or stale code |
| 403 | `FORBIDDEN` | Role not allowed to call the endpoint |
| 409 | `EMAIL_ALREADY_REGISTERED` | Email used by another account |
| 409 | `FARMER_PHONE_ALREADY_REGISTERED` | Same phone in this cooperative |
| 410 | `INVITATION_EXPIRED` | Setup link older than 24 hours |
| 429 | `RATE_LIMITED` | Too many attempts |

## Edge cases

- **Same person, two cooperatives.** An agent email is global, so the account belongs to one cooperative. If they move, the administrator creates a new account or transfers ownership by admin action.
- **Phone entered with spaces or dashes.** Normalised, then checked; the stored value is always E.164.
- **Agent leaves.** Deactivate the account; their deliveries keep the agent id.
- **Administrator locked out.** Another administrator can reset; with only one administrator, use the bootstrap endpoint.
- **Email bounces.** The administrator sees the invitation as *pending* and can correct the address and resend.

## Testing notes

- **API:** duplicate email, duplicate phone, expired invitation, wrong OTP, rate limit.
- **UI:** inline validation, password strength hint, "keep me logged in" behaviour.
- **E2E:** sign up → invite agent → agent sets password → agent registers farmer.

## Related

- [Collection points](collection-points.md)
- [Security](../security.md)
- [Configuration](../configuration.md)

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/auth.py`, `routers/users.py`
- **Service:** `services/auth_service.py`
- **Schema:** `schemas/user.py`
- **Tests:** `tests/test_auth.py`, `tests/test_users.py`
