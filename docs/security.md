---
title: Security
description: How Umucyo protects accounts, data and payments.
---

# Security

<div class="at-a-glance" markdown>

**In short:** logins take two steps, protected endpoints check the role and cooperative of the user, and a payment that was already sent is refused if sent again.

**Who this page is for:** anyone touching authentication, permissions, money or farmer data — and any reviewer checking that Umucyo can be trusted with a cooperative's records.

**The full requirements document:** [Umucyo — Security and Privacy Requirements](https://docs.google.com/document/d/1Hi9y_FUT5iKZv-uhqkLoFgt1UAKnOAEoXYKoV_XotXE/edit?usp=sharing). That document is the source of truth for scope, glossary, requirements and risk. This page explains how the system meets it.

</div>

## Security architecture

![Umucyo security architecture](assets/architecture/security-architecture.png)

The picture above shows the shape of the system: a farmer on a basic phone, an agent on the PWA, an administrator on the dashboard, and a backend that is the only thing allowed to talk to the database, the SMS gateway and the mobile-money provider.

That shape is itself a control. The web apps hold no data of their own, so there is exactly one place where permissions are enforced, and exactly one place to audit.

!!! note "Where the requirements live"
    Requirement identifiers, acceptance criteria and the risk register are kept in the [Security and Privacy Requirements document](https://docs.google.com/document/d/1Hi9y_FUT5iKZv-uhqkLoFgt1UAKnOAEoXYKoV_XotXE/edit?usp=sharing). When this page and that document disagree, the document wins — tell us so we can fix the page.

## What we are protecting

| What | Why it matters | Sensitivity |
| --- | --- | --- |
| Staff and agent credentials | An agent account can move produce records and trigger payouts | High |
| Farmer names, phone numbers and IDs | Real people; a leaked list is a safeguarding problem, not just a privacy one | High |
| Weight measurements | These decide what a farmer is paid | High |
| Delivery and produce records | The historical record of a season; if it can be edited quietly, it is worthless as evidence | High |
| Price periods | A wrong or back-dated price changes every payment that follows | Medium |
| Payment references and statuses | Money either left the account or it did not; there is no "probably" | High |
| Complaints and message history | Contains personal circumstances and expectations of a reply | High |
| Offline records and sync queues | The only copy of a delivery may sit on a phone until it syncs | High |
| Credentials for Safaricom, SMS and hosting | A leaked key can spend money or send messages as the cooperative | Critical |
| Database and backups | Everything above, in one place | Critical |

## Accounts and login

- **Two-step login:** a password, then a 6-digit code sent by email.
- **Controlled registration:** cooperative signup or a bootstrap secret creates the first administrator; administrators create agents; agents create farmers.
- **Hashed secrets:** passwords and password-setup tokens are stored only as hashes.
- **One-time invitations:** an agent's setup link expires and cannot be reused.
- **Rate limits:** repeated login, code and password-setup attempts are slowed down (`429`).
- **Short-lived tokens:** access tokens expire after 30 minutes by default.

### What happens when someone signs in

1. The user submits an email address and password.
2. The backend looks up the account and compares the password against the stored hash. It never stores or logs the password itself.
3. If the password matches, a 6-digit code is generated, stored against the user with a short expiry, and sent to the account's email address.
4. The user submits the code. A correct, unexpired code returns an access token.
5. The token travels with every following request. When it expires, the user signs in again.

Nothing about this flow reveals whether an email address exists. A wrong password and an unknown address get the same answer.

### Why there is no self-registration

There is no "sign up" screen, and that is deliberate.

A stranger who could create an account would need a cooperative to attach to, and any cooperative they picked would gain a fake member. Instead, accounts are **invited**: a cooperative is created with its first administrator, that administrator creates agents, and agents register farmers in the field. Every account therefore has a person who is accountable for it.

### Rate limits in practice

| Action | Limit | What the user sees |
| --- | --- | --- |
| Password attempts | A small number per account, then a cooldown | "Too many attempts. Try again shortly." |
| Code attempts | A few per issued code | The code is invalidated; request a new one |
| Password-setup links | One use, short expiry | "This link has expired. Ask an administrator for a new one." |
| Requests in general | Per-route limits | `429 Too Many Requests` |

The point of a rate limit is not to punish a typo. It is to make guessing a 6-digit code or a password impractically slow.

## Who can see what

| Role | Access |
| --- | --- |
| Administrator | Their own cooperative's data |
| Agent | Their collection point's farmers, deliveries and tasks |
| Farmer | Their own information, by SMS and USSD |

The backend checks the role on every request (`require_admin`, `require_agent`, `require_staff`) and filters data to the user's cooperative. Data from another cooperative answers `404`, so its existence isn't revealed.

### Roles side by side

| Action | Administrator | Agent | Farmer |
| --- | --- | --- | --- |
| View farmers at their collection point | Yes | Yes | No |
| Register a farmer | Yes | Yes | No |
| Record a delivery | Yes | Yes | No |
| Correct a delivery | Yes, with an audit entry | No | No |
| Set a price period | Yes | No | No |
| Create or send a payment | Yes | No | No |
| Create an agent account | Yes | No | No |
| See their own deliveries | Yes | Yes | Yes, by USSD or SMS |

An agent's boundary is the collection point, not the cooperative. Two agents at two sheds in the same cooperative cannot see each other's farmers.

### The three refusals

The status code is part of the design, not an accident:

| Code | Meaning | Why we chose it |
| --- | --- | --- |
| `401 Unauthorized` | Not signed in, or the token expired | The client should stop and ask for a sign-in |
| `403 Forbidden` | Signed in, but this role may not do this | The client should tell the user plainly; retrying will not help |
| `404 Not Found` | The record is not yours, or does not exist | We do not confirm that another cooperative's record exists |

That last row is why a record belonging to someone else is not a `403`. Answering `403` there would leak that the record exists.

## Payments

- Only administrators can create or send payments, and only for their cooperative.
- Real-money (production) payouts are switched off in code.
- A sandbox payment needs two confirmations: a server setting and a flag in the request.
- Nobody can mark a payment paid or failed by hand; only Safaricom's callback can.
- Callbacks are checked with a secret token, which is removed from the logs.

### Why the callback is the only authority

A payment screen that could mark a payment as paid would eventually mark one paid that never was — a double click, a dropped connection, an optimistic refresh. So the rule is absolute: the status changes only when the provider says so.

| Payment status | Set by | Can a person change it? |
| --- | --- | --- |
| Created | The administrator, on the dashboard | Not after it is sent |
| Sent | The backend, after the provider accepts the request | No |
| Paid | The provider, through the callback | No |
| Failed | The provider, through the callback or a timeout | No |

### Stopping the double payout

Sending a payment twice is the most expensive mistake in the system. Three separate things prevent it:

1. **The preview step** shows the amount and the recipient count before anything is sent, so a wrong number is caught while it is still free to fix.
2. **The idempotency key** means a retried request — from a flaky network, or an impatient click — is recognised as the same request and answered, not repeated.
3. **The callback** is matched to the payment it refers to. A callback that arrives twice updates the same payment to the same status.

## Farmer data over USSD and SMS

A farmer does not have an account, a password or an app. Three rules protect them anyway:

- **They are identified, not authenticated.** A phone number that is registered and known to the system can ask about its own deliveries. A number that is not registered gets a generic answer, not a list of what exists.
- **Nothing reusable is spoken or sent.** No password, no PIN, no code that could be overheard and used later.
- **Only their own information.** A USSD session cannot be steered to another farmer's record, because the session is tied to the number that dialled in.

Impersonation is still the hardest risk to close on this channel. A phone can be borrowed, lent or stolen. The controls are: limited attempts per number, restricted disclosure, and an audit trail so a pattern of unusual queries is visible.

## Designing a change to a sensitive area

If you are adding a feature that touches accounts, money or personal data, answer these before writing code:

| Question | Why it is asked |
| --- | --- |
| Who may call this, and how is that enforced? | A check in the router only, with none in the service, is the usual hole |
| Which cooperative or collection point does the query filter on? | A missing filter is a cross-cooperative leak, not a performance issue |
| What does an unauthorised caller get back? | Wrong answer: a `200` with an empty list, which tells them nothing — and also tells them nothing is there |
| Is the input validated in the service? | Routers change; services are called by everything |
| What is written to the log, and could it hold a secret? | Logs outlive tokens, and are read by more people |
| Can this be retried safely? | Any write that can be retried needs an idempotency story |
| What test proves the refusal works? | If it is not tested, it is not enforced |

## Secrets

!!! danger "Keep secrets out of the code"
    Keys, passwords and tokens live only in `.env` files or the hosting platform's settings. Never commit them, paste them in chats or show them in screenshots. If one leaks, replace it at its source straight away.

### Rotating a leaked secret

| Secret | Where it lives | What rotation costs |
| --- | --- | --- |
| `SECRET_KEY` | Backend `.env`, hosting settings | Every signed-in user is signed out |
| Database password | Backend `.env`, database provider | A short outage while both sides are updated |
| Safaricom consumer key and secret | Backend `.env` | Payouts fail until it is replaced |
| SMS provider key | Backend `.env` | Notifications queue and fail until replaced |
| Bootstrap secret | Backend `.env` | Nothing; it is only used to create a first administrator |

Rotate first, ask questions second. A leaked key does not wait for a tidy rollback plan.

## Logging and audit

Two different things, often confused:

- **Logs** help us debug: a request failed, a job retried, a callback was rejected. They are technical, they rotate, and they must never contain passwords, tokens or full phone numbers.
- **The audit trail** answers "who did this, and when": a delivery corrected, a price period set, a payment sent. It is part of the record and it is not deleted.

| Event | Logged | In the audit trail |
| --- | --- | --- |
| Failed sign-in | Yes | Repeated failures only |
| Agent invited | Yes | Yes |
| Farmer registered | Yes | Yes |
| Delivery recorded | Yes | Yes |
| Delivery corrected, with reason | Yes | Yes, with before and after |
| Price period created | Yes | Yes |
| Payment sent | Yes | Yes |
| Callback received | Yes, with the token removed | Payment status change only |

## What is not done yet

Honest limits, so nobody assumes a control that is not there:

- **No encryption at rest that we manage ourselves.** We rely on the database provider's disk encryption and on encrypted transport. A cooperative with a stricter requirement needs this stated and addressed.
- **No hardware-backed authentication.** There is no hardware token or passkey yet; the second step is an emailed code.
- **No automated intrusion detection.** Rate limits and audit review are manual.
- **No data-retention automation.** Records are kept; a retention policy exists in the requirements document but is not enforced by a job.
- **No pen test completed.** The risk register is documented and the controls are tested by unit and integration tests, but the system has not had an external security review.

## Good practice for contributors

- Use the role checks on every new protected endpoint.
- Filter every query by the user's cooperative or collection point.
- Validate input in services, not routers.
- Add tests for "not signed in" (`401`), "wrong role" (`403`) and "another cooperative" (`404`).
- Never return a database object directly; return a response model that names every field it exposes.
- Prefer `404` over `403` when the record belongs to someone else.
- Ask before adding a new field that holds personal data.

## Reporting a problem

If you find a hole, do not open a public issue. Tell an administrator privately, with the steps to reproduce and what you were able to see. If a real key has been exposed, treat it as leaked: rotate it first, then write it up.

## Where to go next

- [Security and Privacy Requirements document](https://docs.google.com/document/d/1Hi9y_FUT5iKZv-uhqkLoFgt1UAKnOAEoXYKoV_XotXE/edit?usp=sharing) — scope, glossary, requirements, risks
- [Architecture](architecture.md) — how the pieces fit together
- [Database](database.md) — the tables that hold farmer and delivery data
- [Backend](backend.md) — where the checks are enforced
- [Glossary](glossary.md) — OTP, idempotency, audit trail, tenant isolation
