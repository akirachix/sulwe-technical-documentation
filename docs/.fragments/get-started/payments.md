---
title: Payments
description: Monthly farmer payouts through M-Pesa B2C.
---

# :material-cash-fast: Payments

<div class="at-a-glance" markdown>

**In short:** once a month an administrator pays every farmer for their accepted deliveries through M-Pesa. A payment counts as paid only when the provider confirms it, never when we ask.

**Who uses it:** cooperative administrators &nbsp;·&nbsp; **Where:** Admin dashboard → Payments &nbsp;·&nbsp; **API prefix:** `/api/payments`

</div>

## What it does

- Adds up each farmer's accepted deliveries for a month and shows what is owed.
- Creates one payment per farmer, grouped into a monthly **batch**.
- Sends each payment to the farmer's M-Pesa number through **Daraja B2C**.
- Stores the provider's answer: **paid** with a receipt, or **failed** with a reason.
- Blocks double payment of the same farmer for the same month.
- Gives the administrator a run page that follows progress to the end.

## The five steps of a monthly run

1. **Preview the month.** Totals each farmer's accepted deliveries and lists who can be paid. Farmers are excluded when they have nothing to pay, were already paid that month, or have no valid M-Pesa number.
2. **Create the batch.** The administrator confirms; one **pending** payment is created per payable farmer. Nothing is sent yet, so the run can still be abandoned.
3. **Send the payments.** Each payment is submitted once. A submitted payment can never be sent again.
4. **Receive the result.** The provider calls the webhook; the payment becomes **paid** or **failed**.
5. **Follow the run.** The run page refreshes every 5 seconds until nothing is pending.

<div class="mermaid" markdown>
flowchart LR
    A[Preview month] --> B[Create batch]
    B --> C[Send each payment]
    C --> D{Provider answer}
    D -->|success| E[paid + receipt]
    D -->|failure| F[failed + reason]
    D -->|timeout/unknown| G[uncertain]
    G --> H[Reconcile manually]
    H --> E
    H --> F
</div>

!!! tip "One payment per farmer per month"
    A farmer with a payment for that month is skipped automatically, so running the batch twice never pays twice. The database enforces this, not just the code.

## Who gets excluded, and why

| Reason | Message shown | What the administrator does |
| --- | --- | --- |
| No accepted deliveries | *Nothing to pay* | Nothing |
| Already has a payment this month | *Already paid* | Nothing |
| No phone number | *Missing phone number* | Add it on the farmer record, then re-preview |
| Invalid M-Pesa number | *Invalid number* | Correct the number, then re-preview |
| Farmer inactive | *Farmer inactive* | Reactivate or leave out |

## Payment states

Two separate things are tracked: what the farmer is **owed**, and how far the **provider request** got.

| Field | Values | Meaning |
| --- | --- | --- |
| `status` | `pending`, `paid`, `failed` | The financial state |
| `submission_state` | empty, `submitted`, `accepted`, `uncertain`, `failed`, `mocked` | The transport state |

Only the provider's callback can move `status` to `paid` or `failed`. `submission_state` records how the request travelled, which matters when the answer is unclear.

## Modes

`DARAJA_SUBMISSION_MODE` decides whether requests really reach the provider.

=== "Mock (local development)"

    - **Send** records the payment as sent but never contacts Safaricom.
    - The result is simulated with **Mark paid (test)** or **Mark failed (test)**.
    - Use this mode on your own computer.

=== "Sandbox (testing)"

    - **Send** makes a real request to the sandbox. No real money moves.
    - The sandbox only pays its test number `254708374149`.
    - The result arrives through the callback to the public webhook URL.

=== "Production"

    - Refused in code for this MVP release. Any environment other than `sandbox` or `mock` raises at startup.

!!! warning "Production is switched off"
    The MVP cannot send real money. This is a deliberate guard: the flow is proven against the sandbox before anyone can move a farmer's payment.

## Safety rules

??? success "Only administrators of the same cooperative"
    Every endpoint needs a signed-in administrator and shows only their cooperative. Another cooperative's payment answers **404**.

??? success "Two keys before a sandbox payment"
    Sending needs the server setting `DARAJA_ALLOW_SANDBOX_SUBMISSIONS=true` **and** `confirm_sandbox_transfer: true` in the request body.

??? success "No manual paid or failed"
    `PATCH /api/payments/{id}/status` always answers **405 METHOD_NOT_ALLOWED**. Only the provider's callback may settle a payment.

??? success "Unclear answers block a resend"
    A timeout or an unknown answer sets `uncertain`, and the payment cannot be resent until it is reconciled. Sending twice is worse than paying late.

??? success "Protected callbacks"
    Webhooks need no login but check a secret token in the URL, compared in constant time and stripped from logs.

??? success "Batch creation is atomic"
    A batch is created in one transaction. A failure halfway leaves no partial batch.

## Data stored

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | primary key |
| `batch_id` | uuid | the monthly run |
| `cooperative_id` | uuid | owning cooperative |
| `farmer_id` | uuid | payee |
| `month`, `year` | int | the paid period |
| `amount` | numeric(12,0) | whole currency units |
| `status` | enum | `pending`, `paid`, `failed` |
| `submission_state` | enum | transport state |
| `provider_reference` | text | provider's id |
| `receipt_number` | text | M-Pesa receipt when paid |
| `failure_reason` | text | when failed |

Unique constraint: `(farmer_id, month, year)` — the database itself refuses a second payment.

## API endpoints

All routes start with `/api/payments`.

| Method | Path | What it does |
| --- | --- | --- |
| `POST` | `/preview` | Amount per farmer for a month; saves nothing |
| `POST` | `/batch` | Creates the month's pending payments |
| `GET` | `/batch/{batch_id}` | One batch and its payments |
| `GET` | `/?month=&year=` | Lists your cooperative's payments |
| `GET` | `/{payment_id}` | One payment |
| `GET` | `/dashboard/{cooperative_id}` | Paid, pending and failed totals |
| `POST` | `/{payment_id}/transfer` | Sends one payment to the provider |
| `POST` | `/{payment_id}/mock-result` | Mock mode only |
| `POST` | `/webhook` | Provider result callback |
| `POST` | `/webhook/timeout` | Provider timeout callback |

??? example "Example: preview a month"
    ```http
    POST /api/payments/preview
    Authorization: Bearer <admin_token>
    Content-Type: application/json

    { "month": 10, "year": 2026 }
    ```

    ```json
    200 OK
    { "total": 1846000, "payable_farmers": 62,
      "excluded": [ { "farmer": "Jean M.", "reason": "invalid_number" } ] }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 403 | `FORBIDDEN` | Not an administrator |
| 404 | `PAYMENT_NOT_FOUND` | Unknown or another cooperative |
| 405 | `METHOD_NOT_ALLOWED` | Trying to set paid/failed by hand |
| 409 | `PAYMENT_ALREADY_PAID` | Farmer already paid this month |
| 409 | `PAYMENT_ALREADY_SUBMITTED` | Second send attempt |
| 409 | `PAYMENT_UNCERTAIN` | Must be reconciled first |
| 422 | `SANDBOX_CONFIRMATION_REQUIRED` | Missing `confirm_sandbox_transfer` |

## Edge cases

- **Provider timeout.** Payment becomes `uncertain`; the operator checks the provider portal, then resolves by hand. It cannot be resent meanwhile.
- **Duplicate callback.** Handled idempotently: the same provider reference is applied once.
- **Farmer changes number after the batch.** The payment keeps the number it was created with; the next month uses the new one.
- **Partial batch failure.** Payments that succeeded stay paid; the rest remain pending and are retried individually.
- **Cooperative has 600 farmers.** Sending is queued, not synchronous; the run page shows progress.
- **Month with no deliveries.** Preview returns an empty list and the batch is refused with **422 NOTHING_TO_PAY**.

## Testing notes

- **API:** double batch, double send, manual status, uncertain resend, webhook replay.
- **Integration:** mock provider returns success, failure and timeout.
- **E2E:** deliver → preview → batch → send → paid → visible by USSD.

## Related

- [Produce records](produce-records.md): the deliveries payments come from.
- [USSD reporting](ussd.md): how the farmer checks the payment.
- [Configuration](../configuration.md): Daraja settings.
- [Security](../security.md): access control and callbacks.

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/payments.py`
- **Service:** `services/payment_service.py`
- **Provider:** `services/daraja_client.py`
- **Tests:** `tests/test_payment.py`
