---
title: Tea Pricing
description: Managing the price paid per kilogram of green tea leaves.
---

# Tea Pricing

<div class="at-a-glance" markdown>

**In short:** administrators set the price paid per kilogram of green leaf, with a start date and an optional end date. Periods may not overlap, so at any moment exactly one price applies and no delivery is ever in doubt.

**Who uses it:** administrators &nbsp;·&nbsp; **Where:** Admin dashboard → Tea pricing &nbsp;·&nbsp; **API prefix:** `/tea-prices`

</div>

## What it does

- Creates a price with **price per kilogram**, **effective from** and optional **effective to**.
- Ends a price on a given date.
- Keeps every past price, so the price history is auditable.
- Gives the [produce record](produce-records.md) feature exactly one price for any date.

## Why it matters

The price is the multiplier in every payment. If the system can answer "which price applied on 12 October?" with more than one answer, two farmers with identical weights could be paid differently.

| Rule | Consequence if broken |
| --- | --- |
| One price per date | Payments would depend on query order |
| Prices are never edited after use | A farmer's amount could change after being told |
| Currency is fixed per cooperative | Mixed units would corrupt every total |

## A price record

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | uuid | primary key |
| `cooperative_id` | uuid | owning cooperative |
| `price_per_kg` | numeric(10,2) | amount paid for each kilogram |
| `effective_from` | date | first day the price applies |
| `effective_to` | date, nullable | last day it applies; `null` means open-ended |
| `is_active` | boolean | whether the price can still be used |
| `created_by` | uuid | who set it |

## Rules

!!! warning "No overlapping periods"
    Two active periods cannot overlap. A new price must start **after** the current one ends. Overlap answers **409 PRICE_PERIOD_OVERLAP**.

!!! success "Backdating is refused"
    A price may start today or later, but not in the past, because deliveries already recorded would need rewriting.

!!! success "Open periods close before a new one starts"
    Setting a new price while the current one is open-ended asks you to set its end date first, or does it for you in the same action: the day before the new price starts.

!!! success "Amounts are copied onto deliveries"
    Editing a price never changes an amount already recorded. A correction is a new price plus an adjustment, never an edit of history.

## How an administrator sets a price

1. Open **Tea pricing** and choose **New price**.
2. Enter the **price per kilogram**, e.g. `320`.
3. Choose **effective from** — today or a future date.
4. Leave **effective to** empty for an open-ended price.
5. Save. The system checks for overlap and refuses with a clear message if there is one.
6. From that date, every new delivery uses this price. Past deliveries keep theirs.

!!! tip "Announce the change"
    When a price changes, tell farmers through the [SMS](sms-notifications.md) channel and the [USSD](ussd.md) menu before the effective date. A price no one knows about causes disputes even when the calculation is right.

## Currency

All prices are in the cooperative's currency, set once at cooperative creation. The MVP fixes it to the local currency used for payouts, so the price, the delivery amount and the payment are always the same unit.

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `GET` | `/tea-prices` | admin, agent | Lists prices, newest first |
| `POST` | `/tea-prices` | admin | Creates a price |
| `GET` | `/tea-prices/{id}` | admin | One price |
| `PATCH` | `/tea-prices/{id}` | admin | Changes price or end date |
| `POST` | `/tea-prices/{id}/deactivate` | admin | Ends a price on a date |
| `GET` | `/tea-prices/current` | agent | The price in force today |

??? example "Example: set a new price"
    ```http
    POST /tea-prices
    Authorization: Bearer <admin_token>
    Content-Type: application/json

    { "price_per_kg": 320, "effective_from": "2026-11-01" }
    ```

    ```json
    201 Created
    { "id": "9f2a...", "price_per_kg": 320,
      "effective_from": "2026-11-01", "effective_to": null, "is_active": true }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 403 | `FORBIDDEN` | Agent tried to write |
| 409 | `PRICE_PERIOD_OVERLAP` | New period touches an existing one |
| 409 | `PRICE_IN_USE` | Deleting a price already applied |
| 422 | `PRICE_BACKDATED` | Start date before today |
| 422 | `PRICE_INVALID` | Zero, negative, or more than two decimals |
| 422 | `END_BEFORE_START` | End date earlier than start |

## Edge cases

- **Two price changes on the same day.** The second is refused as an overlap; end the first period and restart, or set a single new price.
- **Price change with no end date on the old price.** The system closes the old period the day before the new one starts.
- **Farmer asks about an old price.** The `GET /tea-prices` list is the history; each delivery also stores the price it used.
- **A price is entered with decimals.** Accepted to two decimals; the total is rounded once, at the end.
- **Cooperative changes currency.** Not supported in the MVP; it would invalidate history.

## Testing notes

- **API:** overlap at the boundary dates, backdating, zero, end before start.
- **UI:** the list shows a clear "current" badge on exactly one row.
- **E2E:** set price → deliver → amount matches → set new price → old delivery unchanged.

## Related

- [Produce records](produce-records.md): where the price is applied.
- [Payments](payments.md): what the amounts become.

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/produce_price.py`
- **Service:** `services/produce_price.py`
- **Schema:** `schemas/produce_price.py`
- **Tests:** `tests/test_produce_price.py`
