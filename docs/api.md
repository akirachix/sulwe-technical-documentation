---
title: API Reference
description: Every endpoint Umucyo exposes, who may call it, and what it returns.
---

# API Reference

<div class="at-a-glance" markdown>

**In short:** this page is the map of the backend's endpoints, grouped by module. Every path here is live. Where we show a request and a response, those are real shapes you can paste into Postman.

**Who uses it:** backend developers, frontend and PWA developers, the USSD gateway integrator, and testers writing API tests.

**Base URL:** development `http://localhost:8000`, staging and production from your deployment settings.

</div>

!!! info "The authoritative list"
    Swagger is generated from the code, so it is never out of date. Open `/docs` on any running backend to see every route, its schema and a **Try it out** button. This page explains the rules behind those routes; Swagger gives you the exact fields.

!!! info "Authentication"
    Protected endpoints need `Authorization: Bearer <access_token>`. Get a token by logging in and verifying the emailed code. See [User registration](features/registration.md#logging-in).

---

## Conventions

Read this once and every endpoint below makes sense.

| Convention | Rule |
| --- | --- |
| Base path | Modules are mounted at their own prefix: `/auth`, `/users`, `/collection-points`, `/tasks`, `/tea-prices`, `/api/payments` |
| Content type | `application/json` for every request with a body. The USSD gateway is the one exception — see [USSD](features/ussd.md) |
| Field naming | `snake_case` everywhere, including query parameters and JSON bodies |
| Dates | ISO 8601 with time: `2026-11-09T14:03:00+02:00`. Dates without time are `YYYY-MM-DD` |
| Money | Integer **RWF**, never a float and never a decimal string. `45000` means 45,000 RWF |
| Weights | Decimal kilograms, two decimal places, as a JSON number: `12.75` |
| IDs | UUID v4 strings, never sequential integers |
| Pagination | `?limit=` and `?offset=`. Default `limit=50`, maximum `200` |
| Sorting | `?sort=` with a leading `-` for descending: `?sort=-created_at` |
| Filtering | Plain query parameters that match column names: `?status=pending&collection_point_id=<uuid>` |
| Idempotency | Writes that move money or send a message accept `Idempotency-Key`. Repeat calls with the same key return the first result rather than acting twice |

!!! warning "Never send money as a decimal"
    A float `45000.0` may serialize as `45000.00000001` and fail validation, or worse, round wrongly inside a payment batch. Send integers. The database stores `numeric(12,2)`; the API contract is integer RWF.

---

## Response envelope

Successful responses return the object directly. Collections return the array directly. There is no wrapper object — one less layer to unwrap in the frontend.

```json
{
  "id": "8f14e45f-ceea-467f-a1b2-9c0a3b7c2d11",
  "name": "Nyamasheke Shed A",
  "status": "active",
  "created_at": "2026-11-09T14:03:00+02:00"
}
```

A paginated collection:

```json
{
  "items": [
    { "id": "…", "name": "Nyamasheke Shed A" },
    { "id": "…", "name": "Rusizi Shed B" }
  ],
  "total": 12,
  "limit": 50,
  "offset": 0
}
```

Errors use one shape everywhere:

```json
{
  "detail": "A tea price already covers this collection point for that date.",
  "code": "PRICE_OVERLAP",
  "field": "effective_date"
}
```

| Field | Meaning |
| --- | --- |
| `detail` | A sentence safe to show a user. Never contains a stack trace |
| `code` | A stable machine string your client can switch on. Codes never change meaning |
| `field` | Present on `422` only, naming the offending input |

---

## Status codes

| Code | Meaning | What the client should do |
| --- | --- | --- |
| `200` | Success | — |
| `201` | Created | Use the returned `id` |
| `204` | Success, no body | — |
| `400` | The request breaks a business rule | Show `detail` to the user |
| `401` | Not signed in, or the token expired | Refresh the token, then retry once |
| `403` | Signed in, but not allowed | Hide the action; do not retry |
| `404` | Not found, or not in your cooperative | Treat as gone |
| `409` | Conflicts with the current state | Re-read the record, then decide |
| `422` | The request body is invalid | Highlight `field` |
| `429` | Too many attempts | Wait for `Retry-After` seconds |
| `500` | Server fault | Report with the request ID; do not retry blindly |

!!! tip "404 is also a permission answer"
    We return `404` rather than `403` for records that exist in another cooperative. Telling a caller "this exists but is not yours" leaks information. If a cooperative admin gets `404` on a record they can see in the UI, the record has moved tenants.

---

## Authentication

=== "Authentication"

    | Method | Path | Access |
    | --- | --- | --- |
    | `POST` | `/auth/register/cooperative` | Public |
    | `POST` | `/auth/bootstrap-admin` | Bootstrap secret |
    | `POST` | `/auth/login` | Public |
    | `POST` | `/auth/resend-otp` | Public |
    | `POST` | `/auth/verify-otp` | Public |
    | `POST` | `/auth/set-password` | Setup token |
    | `GET`, `PATCH`, `DELETE` | `/auth/me` | Signed in |
    | `GET`, `PUT` | `/auth/me/notifications` | Signed in |
    | `POST` | `/auth/logout` | Signed in |

**Sign-in flow.** Credentials, then a code by email, then a token.

1. `POST /auth/login` with `email` and `password`. Returns `{"otp_required": true}`.
2. An email arrives with six digits.
3. `POST /auth/verify-otp` with `email` and `code`. Returns the token pair.

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs…",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs…",
  "token_type": "bearer",
  "expires_in": 900
}
```

| Token | Lifetime | Use |
| --- | --- | --- |
| Access | 15 minutes | Every protected call |
| Refresh | 7 days | Exchange for a new access token |

!!! warning "Codes expire and lock out"
    An OTP is valid for 10 minutes and burns on first use. Five wrong codes for one email in 15 minutes returns `429 TOO_MANY_ATTEMPTS`. `POST /auth/resend-otp` is rate limited to three sends per hour per address.

=== "People and places"

    | Method | Path | Access |
    | --- | --- | --- |
    | `GET`, `POST` | `/collection-points/` | Administrator |
    | `DELETE` | `/collection-points/{id}` | Administrator |
    | `GET` | `/users/farmers` | Staff |
    | `GET`, `POST` | `/users/agents` | Administrator |
    | `DELETE` | `/users/agents/{id}` | Administrator |
    | `POST` | `/users/agents/{id}/resend-invitation` | Administrator |

**Creating a collection point.**

```http
POST /collection-points/
Authorization: Bearer <access_token>
Content-Type: application/json
```

```json
{
  "name": "Nyamasheke Shed A",
  "district": "Nyamasheke",
  "sector": "Bushekeri",
  "capacity_kg": 4000
}
```

Returns `201` with the created record. A duplicate name inside the same cooperative returns:

```json
{ "detail": "A collection point with this name already exists in your cooperative.", "code": "DUPLICATE_POINT_NAME" }
```

!!! note "Delete is a soft delete"
    `DELETE /collection-points/{id}` sets `status` to `inactive`. Records that reference the point keep working. A point with produce records in the current open period cannot be deleted — you get `409 POINT_HAS_OPEN_RECORDS`.

=== "Operations"

    | Method | Path | Access |
    | --- | --- | --- |
    | `GET`, `POST` | `/tasks` | Staff |
    | `GET`, `PATCH`, `DELETE` | `/tasks/{id}` | Staff |
    | `GET`, `POST` | `/tea-prices` | Administrator |
    | `GET`, `PATCH` | `/tea-prices/{id}` | Administrator |
    | `POST` | `/tea-prices/{id}/deactivate` | Administrator |

**Tasks** support filtering, which is how the dashboard builds its columns:

```http
GET /tasks?status=open&priority=high&assignee_id=<uuid>&sort=-created_at
```

```json
{
  "items": [
    {
      "id": "3c9f…",
      "title": "Re-weigh delivery for farmer 1024",
      "status": "open",
      "priority": "high",
      "assignee_id": "a71b…",
      "linked_record_id": "bb02…",
      "created_at": "2026-11-09T09:12:00+02:00"
    }
  ],
  "total": 1, "limit": 50, "offset": 0
}
```

**Tea prices** are the other half of this module. Creating one is where most conflicts come from — see the overlap rule below.

```json
{
  "collection_point_id": "8f14…",
  "price_per_kg": 350,
  "effective_from": "2026-11-01",
  "effective_to": null
}
```

| Error | When |
| --- | --- |
| `409 PRICE_OVERLAP` | Another price already covers these dates for this point |
| `409 PRICE_IN_PAST` | `effective_from` is before the last closed period |
| `422 INVALID_PRICE` | `price_per_kg` is zero, negative or above the sanity ceiling |

=== "Weighing and produce"

    | Method | Path | Access |
    | --- | --- | --- |
    | `POST` | `/deliveries/` | Agent |
    | `GET` | `/deliveries/` | Staff |
    | `GET`, `PATCH` | `/deliveries/{id}` | Staff |
    | `POST` | `/deliveries/{id}/void` | Administrator |
    | `GET`, `POST` | `/weighing-devices/` | Administrator |
    | `POST` | `/weighing-devices/{id}/bind` | Administrator |
    | `GET` | `/weighing-devices/{id}/readings` | Staff |

These routes are documented in Swagger under their own tags. The shape that matters for clients is the delivery create call, because the price is resolved on the server and not sent by the device:

```json
{
  "farmer_id": "c4d2…",
  "collection_point_id": "8f14…",
  "device_id": "9a01…",
  "weight_kg": 12.75,
  "captured_at": "2026-11-09T07:41:00+02:00",
  "client_reference": "offline-7f3a"
}
```

```json
{
  "id": "ee51…",
  "weight_kg": 12.75,
  "price_per_kg": 350,
  "amount_rwf": 4462,
  "status": "accepted",
  "sms_status": "queued"
}
```

!!! warning "The device never sends a price"
    If a client sends `price_per_kg` it is ignored, not trusted. The server looks up the price in force for that point and date. `client_reference` is what makes an offline retry safe: the same reference twice returns the first record instead of a duplicate delivery.

=== "Payments"

    | Method | Path | Access |
    | --- | --- | --- |
    | `POST` | `/api/payments/preview` | Administrator |
    | `POST` | `/api/payments/batch` | Administrator |
    | `GET` | `/api/payments/batch/{batch_id}` | Administrator |
    | `GET` | `/api/payments/` | Administrator |
    | `GET` | `/api/payments/{id}` | Administrator |
    | `GET` | `/api/payments/dashboard/{cooperative_id}` | Administrator |
    | `POST` | `/api/payments/{id}/transfer` | Administrator |
    | `POST` | `/api/payments/{id}/mock-result` | Administrator, mock mode |
    | `POST` | `/api/payments/webhook` | Provider callback token |
    | `POST` | `/api/payments/webhook/timeout` | Provider callback token |

Payments follow a two-step pattern on purpose: **preview, then commit.**

1. `POST /api/payments/preview` with a date range. Returns totals and a per-farmer list. Writes nothing.
2. An admin reviews the list.
3. `POST /api/payments/batch` with the same range and an `Idempotency-Key`. Creates the batch and its payments.

```json
{
  "period_start": "2026-10-01",
  "period_end": "2026-10-31"
}
```

```json
{
  "batch_id": "77ab…",
  "farmer_count": 214,
  "total_rwf": 8841000,
  "excluded": [
    { "farmer_id": "c4d2…", "reason": "NO_MOMO_NUMBER" },
    { "farmer_id": "d81e…", "reason": "ALREADY_PAID" }
  ]
}
```

| Error | When |
| --- | --- |
| `409 BATCH_ALREADY_RUN` | This period already has a batch in `pending` or `processing` |
| `409 SELF_APPROVAL_BLOCKED` | The requester created the batch and is trying to approve it |
| `402 INSUFFICIENT_FLOAT` | The cooperative's wallet cannot cover the batch |
| `403 NOT_ADMIN` | Any non-administrator calling a payments route |

!!! note "Webhooks are not user calls"
    `/api/payments/webhook` is called by the mobile money provider, not by our clients. It authenticates with a callback token in the header, not a user bearer token, and it must return `200` quickly — the provider retries on anything else.

=== "Health"

    | Method | Path | Access |
    | --- | --- | --- |
    | `GET` | `/health` | Public |

`GET /health` returns `{"status": "ok", "database": "ok", "redis": "ok"}`. Any dependency down returns `503`. Point your uptime monitor at this and nothing else.

---

## Rate limits

| Route group | Limit | Window | Scope |
| --- | --- | --- | --- |
| `/auth/login`, `/auth/verify-otp` | 5 | 15 min | Per email |
| `/auth/resend-otp` | 3 | 1 hour | Per email |
| Authenticated reads | 600 | 1 min | Per user |
| Authenticated writes | 120 | 1 min | Per user |
| `/api/payments/batch` | 3 | 1 hour | Per cooperative |
| `/api/payments/webhook` | 300 | 1 min | Per provider |

Exceeding a limit returns `429` with a `Retry-After` header in seconds. The frontend should back off, not loop.

---

## Versioning

The API is versioned by path prefix only when a breaking change is unavoidable. Today every route is unversioned and stable.

A change is **safe** and needs no version bump: adding an optional request field, adding a response field, adding an endpoint, adding a new error code to a route that already errors.

A change is **breaking**: removing or renaming a field, changing a type (`integer` to `string`), making an optional field required, changing a status code for an existing case, changing what `code` means.

When a break is unavoidable we mount the new shape under `/v2`, keep `/v1` running for one full quarter, and log every call to the old path so we know who is still on it.

---

## Error codes by module

Keep this table in sync with the code. Testers work from it.

| Code | Module | Meaning |
| --- | --- | --- |
| `VALIDATION_ERROR` | All | `422`; `field` names the input |
| `UNAUTHENTICATED` | All | Token missing or expired |
| `TOO_MANY_ATTEMPTS` | Auth | Rate limit hit |
| `OTP_EXPIRED` | Auth | Code older than 10 minutes |
| `OTP_INVALID` | Auth | Wrong digits |
| `DUPLICATE_POINT_NAME` | Collection points | Name taken in this cooperative |
| `POINT_HAS_OPEN_RECORDS` | Collection points | Cannot deactivate with open records |
| `PRICE_OVERLAP` | Tea pricing | Dates already covered |
| `PRICE_IN_PAST` | Tea pricing | Date before the last closed period |
| `INVALID_PRICE` | Tea pricing | Zero, negative, or above the ceiling |
| `DUPLICATE_DELIVERY` | Deliveries | `client_reference` already used |
| `NO_PRICE_IN_FORCE` | Deliveries | No price covers the delivery date |
| `DEVICE_NOT_BOUND` | Weighing | Device is not bound to that point |
| `BATCH_ALREADY_RUN` | Payments | Open batch exists for the period |
| `SELF_APPROVAL_BLOCKED` | Payments | Creator cannot approve |
| `INSUFFICIENT_FLOAT` | Payments | Wallet cannot cover the batch |
| `NOT_ADMIN` | Payments | Role check failed |
| `SMS_OPTED_OUT` | SMS | Farmer declined messages |

---

## Trying it out

```bash
# 1. Log in and read the code from the inbox
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@nyamasheke.rw","password":"…"}'

# 2. Exchange the code for a token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@nyamasheke.rw","code":"123456"}' | jq -r .access_token)

# 3. Use the token
curl -s http://localhost:8000/collection-points/ \
  -H "Authorization: Bearer $TOKEN" | jq
```

!!! tip "Faster than curl"
    Open `/docs`, click **Authorize**, paste the access token, and every endpoint gets a **Try it out** form with the schema filled in. That is the fastest way to test a new route.

---

## Related pages

- [System architecture](architecture.md) — where these modules sit in the backend
- [Database design](database.md) — the tables behind each endpoint
- [Security](security.md) — token lifetimes, roles and the password rules
- [User registration](features/registration.md) — the sign-in flow in full
- [Payments](features/payments.md) — preview, batch and approval from the user's side
- [Weighing devices](features/weighing-devices.md) — the delivery device requirements
