---
title: Produce Records
description: Recording each farmer's tea delivery and the amount owed.
---

# :material-clipboard-text: Produce Records

<div class="at-a-glance" markdown>

**In short:** an agent records each tea delivery — who brought it, on which scale, how much it weighed, and whether it was accepted. The amount owed is calculated automatically from the current price.

**Who uses it:** agents record, administrators review &nbsp;·&nbsp; **Where:** Agent app → New delivery &nbsp;·&nbsp; **API prefix:** `/produce-records`

</div>

## What it does

- Records a delivery with farmer, weighing device, collection point and recording agent.
- Captures the **gross weight in kilograms**, the **status** (accepted or rejected) and optional **agent notes**.
- Calculates the amount payable: weight × current price per kilogram.
- Checks that every part of the record belongs together before saving.
- Sends the farmer an [SMS](sms-notifications.md) when the delivery is accepted.
- Lists and filters deliveries; allows correction and deletion under rules.

## The most important rule

!!! abstract "Amount owed"
    **amount = gross_weight_kg × price_per_kg**, with the price being the [tea price](tea-pricing.md) in force on the delivery date.
    A **rejected** delivery is always **0**, whatever the weight.

Rounding: weights are stored to two decimals, money to zero decimals (whole francs), rounding half up at the end of the calculation, never per step.

## Checks before a delivery is saved

| # | Check | If it fails |
| --- | --- | --- |
| 1 | Price, collection point and agent are in the **same cooperative** | **422 CROSS_COOPERATIVE_REFERENCE** |
| 2 | Device is registered at **that collection point** | **422 DEVICE_NOT_AT_COLLECTION_POINT** |
| 3 | Device is **active** | **422 DEVICE_INACTIVE** |
| 4 | Collection point is **active** | **422 COLLECTION_POINT_INACTIVE** |
| 5 | Farmer is **active** | **422 FARMER_INACTIVE** |
| 6 | Gross weight is greater than 0 and at most 100 kg | **422 WEIGHT_OUT_OF_RANGE** |
| 7 | A price exists for the delivery date | **422 NO_ACTIVE_PRICE** |

The record is rejected as a whole. Nothing partial is written, and the sequence number is not consumed.

## How an agent records a delivery

1. Open **New delivery** in the agent app.
2. Scan or pick the **farmer**. Unregistered farmers must be registered first (see [User registration](registration.md)).
3. Pick the **device**; only active devices at the agent's point are listed.
4. Put the leaf on the scale. The weight appears; if the scale is offline, type the weight shown on its display.
5. Choose **Accepted** or **Rejected**, and add a note when rejecting (leaf too wet, foreign matter, wilted).
6. Save. The app shows the weight and the amount owed before confirming.
7. On save the record is written; if accepted, the SMS goes out.

!!! tip "Offline first"
    The agent app queues the record when there is no network and syncs when there is. The amount is calculated locally with the cached price and recalculated on the server; a price change is flagged to the agent rather than silently changing the amount.

## Why it matters

- **Accurate payments:** amounts come from verified weights and official prices.
- **Traceability:** every record names farmer, agent, device and place.
- **Fewer disputes:** a clear record of what was delivered, when, where, by whom.
- **The basis for payments:** monthly payouts are calculated from accepted deliveries only.

## Data stored

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | primary key |
| `cooperative_id` | uuid | owning cooperative |
| `collection_point_id` | uuid | where it happened |
| `weighing_device_id` | uuid | which scale produced the weight |
| `farmer_id` | uuid | who delivered |
| `agent_id` | uuid | who recorded |
| `gross_weight_kg` | numeric(8,2) | greater than 0 |
| `status` | enum | `accepted`, `rejected` |
| `price_per_kg` | numeric(10,2) | copied at save time |
| `amount` | numeric(12,0) | calculated, 0 when rejected |
| `notes` | text | optional, required when rejected |
| `delivered_at` | timestamptz | defaults to now, editable by admin |
| `created_at`, `updated_at` | timestamptz | audit |

!!! warning "The price is copied, not looked up"
    `price_per_kg` is written onto the record at save time. Changing the price later never changes past deliveries. This is deliberate: a farmer's amount must not move after they were told it.

## Finding deliveries

Deliveries can be filtered by farmer, collection point, weighing device, status and date range. Agents automatically see only their own collection point; administrators see the whole cooperative.

## Updating and deleting

| Action | Who | Rule |
| --- | --- | --- |
| Change notes | admin, recording agent | Always allowed |
| Change weight or status | admin | Allowed while the delivery is not in a paid month |
| Change weight or status | agent | Only within 24 hours of recording |
| Delete | admin | Only if no payment references it |

!!! warning "Paid months are frozen"
    Once a payment for a month exists, deliveries in that month refuse edits with **409 RECORD_IN_PAID_PERIOD**. Corrections are made through an adjustment, not by editing history.

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `GET` | `/produce-records` | admin, agent | Lists with filters, scoped |
| `POST` | `/produce-records` | agent, admin | Records a delivery |
| `GET` | `/produce-records/{id}` | admin, agent | One record |
| `PATCH` | `/produce-records/{id}` | admin, agent | Notes, weight, status |
| `DELETE` | `/produce-records/{id}` | admin | Removes an unpaid record |
| `GET` | `/produce-records/summary` | admin | Totals per farmer for a period |

??? example "Example: record a delivery"
    ```http
    POST /produce-records
    Authorization: Bearer <agent_token>
    Content-Type: application/json

    { "farmer_id": "6c1d...", "weighing_device_id": "3a71...",
      "collection_point_id": "0b9f...", "gross_weight_kg": 14.5,
      "status": "accepted", "notes": "Morning delivery" }
    ```

    ```json
    201 Created
    { "id": "d4e2...", "gross_weight_kg": 14.5, "price_per_kg": 320,
      "amount": 4640, "status": "accepted", "sms_status": "queued" }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 403 | `FORBIDDEN` | Agent editing another point's record |
| 404 | `RECORD_NOT_FOUND` | Unknown id, or another cooperative |
| 409 | `RECORD_IN_PAID_PERIOD` | Month already paid |
| 422 | `WEIGHT_OUT_OF_RANGE` | Zero, negative or over 100 kg |
| 422 | `NO_ACTIVE_PRICE` | No price for the delivery date |
| 422 | `DEVICE_NOT_AT_COLLECTION_POINT` | Device from another site |
| 422 | `FARMER_INACTIVE` | Farmer deactivated |

## Edge cases

- **Two agents record the same farmer at the same minute.** Allowed; both are separate deliveries. Duplicate detection uses a 60-second window with the same farmer, device and weight, and warns the agent before saving.
- **Scale reads 0.00.** Refused; a zero-weight delivery is almost always a scale error.
- **Weight over 100 kg.** Refused; a bag of leaf that heavy signals a unit error or a faulty scale.
- **Price changes at noon.** Deliveries before the change keep the old price, after it get the new one.
- **Farmer deactivated after delivering.** The delivery stays valid and payable.
- **Rejected leaf is still collected** and disposed of by the cooperative; it simply carries an amount of 0.

## Testing notes

- **API:** every check in the table above, plus rounding at 0.005.
- **UI:** amount preview updates as the weight is typed; rejection requires a note.
- **E2E:** register farmer → record delivery → SMS received → appears in the monthly preview.

## Related

- [SMS notifications](sms-notifications.md)
- [Weighing devices](weighing-devices.md)
- [Tea pricing](tea-pricing.md)
- [Payments](payments.md)

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/produce_record.py`
- **Service:** `services/produce_record.py`
- **Schema:** `schemas/produce_record.py`
- **Tests:** `tests/test_produce_record.py`
