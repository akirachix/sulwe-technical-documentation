---
title: USSD Reporting
description: Farmers' access to Umucyo from any basic phone.
---

# USSD Reporting

<div class="at-a-glance" markdown>

**In short:** a farmer dials a short code on any basic phone, gets a menu in English or Kinyarwanda, and can check deliveries, payments and their own details — no smartphone, no internet, no app.

**Who uses it:** farmers &nbsp;·&nbsp; **Gateway:** Africa's Talking &nbsp;·&nbsp; **API prefix:** `/ussd`

</div>

## What it does

- Shows an interactive menu on basic mobile phones.
- Offers **English** and **Kinyarwanda**.
- Identifies the farmer by the phone number the session arrives from.
- Gives access to delivery history and payment reports.
- Refuses restricted information to numbers that are not registered farmers.
- Handles wrong choices and sessions that end without a response.

## The menu

```text
Umucyo
1. My deliveries
2. My payments
3. My details
4. Language
```

```text
My payments
1. Last payment
2. This year
0. Back
```

Every screen ends with a way back, and `0` always goes up one level, so a farmer is never trapped.

## How a session works

1. The farmer dials the cooperative's USSD code.
2. The gateway opens a session and forwards it to the backend with the caller's number.
3. The backend normalises the number and looks up the farmer.
4. If no farmer matches, it returns a short message asking them to register at their collection point, then ends.
5. If matched, it returns the menu in the farmer's stored language.
6. Each choice returns the next screen or the requested information.
7. The session ends with `END`, or when the gateway times out.

<div class="mermaid" markdown>
sequenceDiagram
    Farmer->>Gateway: dials short code
    Gateway->>API: session, phone number, input
    API->>DB: find farmer by phone
    DB-->>API: farmer + language
    API-->>Gateway: CON or END + screen
    Gateway-->>Farmer: menu on screen
</div>

## Response rules

| Rule | Reason |
| --- | --- |
| Always answer `CON` or `END` | The gateway expects exactly one of these prefixes |
| Keep screens under 182 characters | Long screens are cut by some networks |
| MSISDN is normalised before lookup | `078...`, `25078...` and `+25078...` must match the same farmer |
| No PII beyond the caller's own records | A shared phone must not expose another farmer |
| Session state stored per `sessionId` | Retries from the gateway must land on the same screen |

!!! note "Registered farmers only"
    Payment reports are only shown to numbers that belong to a registered farmer. Anyone else gets a short message and the session ends.

## What a farmer sees

| Menu item | Content |
| --- | --- |
| My deliveries | Last five deliveries: date, weight, amount, point |
| My payments | Last payment or the year's total, with month and receipt |
| My details | Name, phone, collection point |
| Language | Switch between English and Kinyarwanda |

## Why it matters

- **Inclusion:** works on any phone, including the cheapest handset.
- **No internet:** USSD runs on the signalling channel.
- **Their language:** English and Kinyarwanda, switchable from the menu.
- **Less travel:** farmers check without walking to the collection point.
- **Fewer disputes:** the same numbers the office sees are in the farmer's hand.

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `POST` | `/ussd` | gateway | Handles one session step |
| `GET` | `/ussd/simulate` | developer | Local menu preview, mock mode only |

??? example "Example: first request of a session"
    ```http
    POST /ussd
    Content-Type: application/x-www-form-urlencoded

    sessionId=ATUid_9f2a&phoneNumber=%2B250788123456&text=&serviceCode=*384*1234#
    ```

    ```text
    CON Umucyo
    1. My deliveries
    2. My payments
    3. My details
    4. Language
    ```

## Common errors

| Situation | What the farmer sees |
| --- | --- |
| Number not registered | *This number is not registered. Ask your collection point.* then `END` |
| Wrong choice | *Invalid choice. Try again.* then the menu |
| Session expired | A fresh menu on the next dial |
| Gateway retry | The same screen, not a duplicated step |

## Edge cases

- **Feature phone with a small screen.** Screens stay under 182 characters; lists show five items.
- **Farmer dials in Kinyarwanda but the record says English.** The menu still opens; language can be changed on screen 4.
- **Shared handset.** Only the caller's own records are shown, keyed by number.
- **Network retry sends the same input twice.** Session state is keyed on `sessionId` + input, so the second call returns the same screen.
- **Farmer has no deliveries yet.** An explicit *You have no deliveries this month.* rather than an empty list.
- **Very long farmer name.** Truncated on the details screen so the screen stays within limits.

## Testing notes

- **API:** unregistered number, wrong input, language switch, session replay.
- **Integration:** mock gateway posts realistic payloads with `*` and `#` encoded.
- **E2E:** dial the sandbox short code and walk the full menu.

## Related

- [SMS notifications](sms-notifications.md)
- [Payments](payments.md)
- [Registration](registration.md)

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/ussd.py`
- **Service:** `services/ussd_service.py`
- **Menus:** `services/ussd_menus.py`
- **Tests:** `tests/test_ussd.py`
