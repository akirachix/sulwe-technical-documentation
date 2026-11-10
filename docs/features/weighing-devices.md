---
title: Weighing Devices
description: A registry of the scales used at collection points.
---

# Weighing Devices

<div class="at-a-glance" markdown>

**In short:** administrators register every scale used at a collection point, each with a unique serial number. A weight can only be recorded on an active, registered scale, so every kilogram in the system can be traced to a physical device.

**Who uses it:** administrators register, agents use &nbsp;·&nbsp; **Where:** Admin dashboard → Devices &nbsp;·&nbsp; **API prefix:** `/weighing-devices`

</div>

## What it does

- Registers a scale with a **unique serial number** and the **collection point** where it stands.
- Marks each scale **active** or **inactive**.
- Lists and filters scales by collection point or status.
- Updates a serial number or status.
- Removes a scale that was registered by mistake and has never weighed anything.
- Blocks deliveries from inactive scales.

## Why it matters

| Risk without a device registry | How the registry prevents it |
| --- | --- |
| A weight with no source | Every delivery names the device that produced it |
| A scale used at the wrong site | The device must belong to the delivery's collection point |
| A faulty scale keeps being used | Deactivate it; new deliveries are refused |
| Two records claim the same hardware | Serial numbers are unique |
| Disputes about a weight | The record shows device, time, agent and point |

!!! warning "One registration per physical scale"
    A serial number can only be registered once in the cooperative. The second attempt answers **409 DEVICE_SERIAL_TAKEN**.

!!! note "Only active devices at active points"
    A delivery needs an **active** device at an **active** collection point. Both conditions are checked on save, so deactivating either one stops new deliveries without touching history.

## The three states of a device

| State | Meaning | New deliveries |
| --- | --- | --- |
| Active | In service, calibrated | Allowed |
| Inactive | Broken, removed, or being calibrated | Refused with **422 DEVICE_INACTIVE** |
| Deleted | Registered by mistake, never used | Not applicable |

## How an administrator registers a scale

1. Open **Devices → Register device**.
2. Pick the **collection point** from the dropdown — only your own cooperative's points appear.
3. Type the **serial number** exactly as printed on the scale, e.g. `SLW-2026-00417`.
4. Save. The device is created **active**.
5. The collector at that point can now select it when recording deliveries.
6. If the scale is sent for repair, set it **inactive** until it returns.

## Rules the backend enforces

!!! success "Serial numbers are unique per cooperative"
    Duplicate answers **409 DEVICE_SERIAL_TAKEN**. Case and surrounding spaces are ignored, so `slw-2026-00417` and `SLW-2026-00417` collide.

!!! success "The point must belong to your cooperative"
    Any other id answers **404 COLLECTION_POINT_NOT_FOUND**.

!!! success "Devices belong to a collection point, permanently"
    A device can be moved to another point by update, but past deliveries keep the device id and their own point id, so history does not move.

!!! success "Only administrators manage devices"
    Agents may read the devices at their own point, and nothing else. Writes from an agent answer **403 FORBIDDEN**.

!!! success "A used device cannot be deleted"
    `DELETE` answers **409 DEVICE_HAS_RECORDS** if any delivery references it. Deactivate instead.

## Data stored

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | primary key |
| `cooperative_id` | uuid | owning cooperative |
| `collection_point_id` | uuid | foreign key, not null |
| `serial_number` | text | unique per cooperative, normalised to upper case |
| `is_active` | boolean | defaults to `true` |
| `created_at`, `updated_at` | timestamptz | audit |

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `GET` | `/weighing-devices` | admin, agent | Lists devices, filtered by point or status |
| `POST` | `/weighing-devices` | admin | Registers a device |
| `GET` | `/weighing-devices/{id}` | admin, agent | One device |
| `PATCH` | `/weighing-devices/{id}` | admin | Serial number, status or point |
| `DELETE` | `/weighing-devices/{id}` | admin | Removes an unused device |

??? example "Example: register a device"
    ```http
    POST /weighing-devices
    Authorization: Bearer <admin_token>
    Content-Type: application/json

    { "serial_number": "SLW-2026-00417", "collection_point_id": "0b9f..." }
    ```

    ```json
    201 Created
    { "id": "3a71...", "serial_number": "SLW-2026-00417",
      "collection_point_id": "0b9f...", "is_active": true }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 403 | `FORBIDDEN` | Agent tried to write |
| 404 | `DEVICE_NOT_FOUND` | Unknown id, or another cooperative |
| 404 | `COLLECTION_POINT_NOT_FOUND` | Point not in your cooperative |
| 409 | `DEVICE_SERIAL_TAKEN` | Serial already registered |
| 409 | `DEVICE_HAS_RECORDS` | Delete attempted with history |
| 422 | `DEVICE_INACTIVE` | Delivery attempted on a stopped scale |

## Edge cases

- **Serial number printed with a prefix sticker.** Enter the engraved number, not the sticker, and record the sticker in the notes of the first task if the team needs it.
- **Scale replaced on the same spot.** Register the new serial, deactivate the old one. History stays with the old device.
- **Scale shared between two points.** Not supported in the MVP: one device, one point. Register it at the point that uses it most.
- **Offline delivery.** The agent app queues the record; the device is validated again when the queue syncs, and a device deactivated in between is rejected with a clear message.
- **Device id typo in a bulk import.** Import fails row by row, reporting the serial that could not be matched.

## Testing notes

- **API:** duplicate serial with different case, foreign point, delete with records, delivery on inactive device.
- **UI:** the list shows point and status; the register form prevents saving without a point.
- **E2E:** register device → record delivery → deactivate → second delivery refused.

## Related

- [Produce records](produce-records.md): deliveries recorded on these devices.
- [Collection points](collection-points.md)
- [Architecture](../architecture.md): how the scale is used by the system.

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/weighing_device.py`
- **Service:** `services/weighing_device.py`
- **Schema:** `schemas/weighing_device.py`
- **Tests:** `tests/test_weighing_device.py`
