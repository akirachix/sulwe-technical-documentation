---
title: Tasks Management
description: Assigning and tracking work for agents.
---

# :material-checkbox-marked-circle-outline: Tasks Management

<div class="at-a-glance" markdown>

**In short:** administrators create tasks, assign them to an agent and follow them until they are done. Agents see their own tasks in the app and mark them complete.

**Who uses it:** administrators assign, agents complete &nbsp;·&nbsp; **Where:** Admin dashboard → Agent tasks, Agent app → Tasks &nbsp;·&nbsp; **API prefix:** `/tasks`

</div>

## What it does

- Creates a task with a title, description and assigned agent.
- Links a task to a collection point when it concerns one.
- Sets a priority and tracks status to completion or cancellation.
- Lets the assignee update status and add a completion note.
- Lists tasks filtered by agent, collection point or status.

## Why it matters

A cooperative runs on small jobs: check a scale, visit a farmer, collect a signature, follow up a failed payment. Without a shared list these live in someone's head or a WhatsApp thread.

| Without tasks | With tasks |
| --- | --- |
| Work is remembered, not tracked | Every job has an owner and a state |
| Nothing shows what is outstanding | Dashboard shows open work per agent |
| Handover depends on memory | The list is the handover |

## Priorities and statuses

| Priority | Use it for |
| --- | --- |
| `low` | Work that can wait |
| `normal` | Everyday work |
| `high` | Work to do first |

| Status | Meaning | Who can set it |
| --- | --- | --- |
| `open` | Assigned, not started | admin |
| `in_progress` | Being worked on | assignee, admin |
| `completed` | Done | assignee, admin |
| `cancelled` | No longer needed | admin |

## How it works

1. The administrator creates a task, chooses the agent, the priority and optionally the collection point.
2. The agent sees it under **To do**, sorted high first, then oldest.
3. The agent opens it, reads the notes, and moves it to **In progress**.
4. When finished, the agent marks it **Completed** and may add a note.
5. The administrator sees the new status on the dashboard.

## Rules

!!! success "A task belongs to one cooperative"
    The agent and the collection point must both be in the creator's cooperative. Anything else answers **404**.

!!! success "Only the assignee or an administrator moves a task"
    Another agent gets **403 FORBIDDEN**.

!!! success "Completed and cancelled are final for agents"
    An agent cannot reopen a completed task. An administrator can, which records who reopened it and when.

!!! success "Deleting is for mistakes, cancelling is for reality"
    A task with history is cancelled, not deleted. Delete answers **409 TASK_HAS_HISTORY** after any status change.

## Data stored

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | primary key |
| `cooperative_id` | uuid | owning cooperative |
| `title` | text | short summary |
| `description` | text | optional detail |
| `agent_id` | uuid | assignee |
| `collection_point_id` | uuid, nullable | when site-specific |
| `priority` | enum | `low`, `normal`, `high` |
| `status` | enum | `open`, `in_progress`, `completed`, `cancelled` |
| `completed_note` | text | optional |
| `due_date` | date, nullable | optional deadline |
| `created_by` | uuid | who assigned |

## API endpoints

| Method | Path | Who | What it does |
| --- | --- | --- | --- |
| `GET` | `/tasks` | admin, agent | Lists, filtered; agents see their own |
| `POST` | `/tasks` | admin | Creates a task |
| `GET` | `/tasks/{id}` | admin, assignee | One task |
| `PATCH` | `/tasks/{id}` | admin, assignee | Details or status |
| `DELETE` | `/tasks/{id}` | admin | Removes an untouched task |

??? example "Example: create a task"
    ```http
    POST /tasks
    Authorization: Bearer <admin_token>
    Content-Type: application/json

    { "title": "Calibrate scale SLW-2026-00417",
      "description": "Reads 0.4 kg heavy against the test weight.",
      "agent_id": "8b2c...", "priority": "high", "due_date": "2026-11-12" }
    ```

    ```json
    201 Created
    { "id": "f1a9...", "status": "open", "priority": "high" }
    ```

## Common errors

| Status | `code` | Cause |
| --- | --- | --- |
| 403 | `FORBIDDEN` | Another agent touching the task |
| 404 | `TASK_NOT_FOUND` | Unknown, or another cooperative |
| 409 | `TASK_HAS_HISTORY` | Delete after a status change |
| 422 | `INVALID_STATUS_TRANSITION` | e.g. `open` straight to `completed` by an agent |

## Edge cases

- **Agent deactivated with open tasks.** Tasks stay open and are reassigned by an administrator; they do not disappear.
- **Task linked to a closed point.** Still visible, marked *point inactive*.
- **Due date in the past at creation.** Refused with **422 DUE_DATE_IN_PAST**.
- **Two administrators edit at once.** Last write wins, with `updated_at` shown.
- **Agent completes offline.** The change queues and syncs; a conflict is resolved in favour of the later timestamp, with a note on the task.

## Testing notes

- **API:** role checks, status transitions, delete with history.
- **UI:** list sorts high priority first; filters survive a refresh.
- **E2E:** admin creates → agent sees → agent completes → admin sees completed.

## Related

- [User registration](registration.md)
- [Collection points](collection-points.md)

## Where it lives in the code

In the `sulwe-backend` repository:

- **Router:** `routers/task.py`
- **Service:** `services/task.py`
- **Schema:** `schemas/task.py`
- **Tests:** `tests/test_tasks.py`
