---
title: SMS Notifications
description: Automatic delivery confirmations sent to farmers by SMS.
---

# :material-message-text: SMS Notifications

<div class="at-a-glance" markdown>

**In short:** when an agent records an accepted delivery, the farmer receives an SMS confirming the weight and the amount, on any phone, with no internet and no app.

**Who receives it:** farmers &nbsp;·&nbsp; **Sent through:** SMS provider (SMS Leopard in the MVP) &nbsp;·&nbsp; **Trigger:** accepted produce record

</div>

## What the SMS contains

- the farmer's name
- the gross weight delivered
- the amount to be paid
- the collection point and the cooperative
- the delivery date

A sample message:

```text
Umucyo: Alice, 14.5 kg accepted at Nyabihu AA on 12/10/2026.
Amount: 4,640. Cooperative: Sulwe.
```

## Why it matters

The SMS is the farmer's **proof**. It is the only part of the system a farmer holds in their hand, so it has to arrive, be readable on a basic phone, and be true.

| Problem without SMS | Effect of the SMS |
| --- | --- |
| Farmer does not know what was recorded | They see weight and amount minutes after delivery |
| Disputes surface at payment time | Disputes surface the same day, when it can still be fixed |
| Trust depends on the agent's word | Trust depends on a message from the cooperative |
| No record for the farmer | The message can be kept as a reference |

## Message rules

!!! info "One message per accepted delivery"
    Rejected deliveries send nothing. Deliveries later edited do not resend automatically; an administrator can resend from the record page.

!!! info "160 characters, no emoji, no links"
    Messages are plain GSM-7 text. Longer messages are truncated at the amount, never mid-number.

!!! info "Language follows the farmer"
    The message uses the farmer's preferred language — English or Kinyarwanda — stored on the farmer record, defaulting to the collection point's language.

## How it works

1. An agent records a delivery with status **accepted**.
2. The record is committed.
3. A background job picks up the record and builds the message text.
4. The job sends to the farmer's phone number through the SMS provider.
5. The provider's response is stored on the record as `sms_status`: `queued`, `sent`, `delivered` or `failed`.
6. A failure is retried three times with backoff, then surfaced on the admin dashboard.

<div class="mermaid" markdown>
flowchart LR
    A[Agent saves accepted record] --> B[Job queued]
    B --> C[Build message in farmer's language]
    C --> D[SMS provider]
    D -->|accepted| E[sms_status = sent]
    D -->|rejected| F[retry x3 with backoff]
    F -->|still failing| G[sms_status = failed, shown to admin]
</div>

!!! note "Sending never blocks the agent"
    The agent's save returns immediately. SMS delivery is asynchronous, so a slow provider cannot stop a queue of farmers at the scale.

## Failure handling

| Situation | What happens |
| --- | --- |
| Provider timeout | Retry after 30 s, then 2 min, then 10 min |
| Provider rejects the number | `failed` with reason `INVALID_NUMBER`, no retry |
| Insufficient credit | `failed` with reason `NO_CREDIT`, alert on dashboard |
| Farmer has no phone | Record saved, `sms_status = skipped` |
| Duplicate trigger | Idempotency key on the record; one message only |

## Settings

| Setting | Purpose |
| --- | --- |
| `SMS_LEOPARD_API_KEY` | Provider key |
| `SMS_LEOPARD_API_SECRET` | Provider secret |
| `SMS_LEOPARD_SENDER_ID` | Sender name shown to the farmer |
| `SMS_MODE` | `mock` in development, `live` in production |

See [Configuration](../configuration.md). In `mock` mode messages are written to the log and never sent, so development never spends credit.

## Data stored

SMS itself is not a table of its own; its state lives on the delivery record:

| Field | Meaning |
| --- | --- |
| `sms_status` | `queued`, `sent`, `delivered`, `failed`, `skipped` |
| `sms_provider_id` | provider's message id, for support queries |
| `sms_sent_at` | when the provider accepted it |
| `sms_error` | reason when failed |

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `POST` | `/produce-records/{id}/resend-sms` | admin | Sends the confirmation again |
| `GET` | `/notifications/sms` | admin | Lists recent messages and their status |
| `POST` | `/webhooks/sms/delivery` | provider | Delivery receipt callback |

## Common errors

| Situation | Visible result |
| --- | --- |
| Provider rejects the number | Record shows `failed / INVALID_NUMBER` |
| No credit | Dashboard banner; records show `failed / NO_CREDIT` |
| Resend on a rejected delivery | **422 SMS_NOT_APPLICABLE** |
| Resend within 60 seconds | **429 SMS_TOO_SOON** |

## Edge cases

- **Farmer shares a phone with a relative.** The message names the farmer, so the reader knows whose delivery it is.
- **Late sync from offline.** The SMS is sent at sync time, not delivery time, and says the delivery date, not the send date.
- **Network outage at the provider.** Messages queue; the provider's own retry handles the rest.
- **Farmer changes number.** The new number gets future messages; old messages are not resent.
- **Very long cooperative or point name.** Truncated so the amount is always intact.

## Testing notes

- **API:** accepted sends, rejected does not, retry path, duplicate suppression.
- **Integration:** mock provider records exactly one call per delivery.
- **E2E:** record delivery → message appears on a test handset.

## Related

- [Produce records](produce-records.md)
- [USSD reporting](ussd.md)
- [Configuration](../configuration.md)

## Where it lives in the code

In the `sulwe-backend` repository:

- **Service:** `services/sms_service.py`
- **Templates:** `templates/sms/`
- **Tests:** `tests/test_sms.py`
