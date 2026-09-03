# `notification-service` — Schema & Description

**Postgres schema:** `notification`
**Owns:** `notification.notifications`, `notification.notification_preferences`
**SRS coverage:** FR-NOT-01…03

## Purpose

Delivers (or queues) notifications to users in reaction to domain events from other services — case assignment, approval requests, case resolution, comment mentions — and lets each user opt in/out per event type and channel.

## Depends On

- `organization.organizations` — tenant ownership.
- `identity.users` — recipient.

## Depended On By

Nothing structurally — this is a terminal consumer of events from `case_management`, `approval`, `collaboration`, and `workflow`.

---

## Tables

### `notification.notifications`
Delivered (or queued) notifications for a user (FR-NOT-01, FR-NOT-02).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | — |
| `recipient_user_id` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `type` | VARCHAR(50) | NOT NULL | E.g. `CASE_ASSIGNED`, `APPROVAL_REQUESTED`, `CASE_RESOLVED`, `MENTION`. |
| `title` | VARCHAR(255) | NOT NULL | — |
| `body` | TEXT | NULL | — |
| `related_resource_type` | VARCHAR(50) | NULL | E.g. `CASE`, `APPROVAL` — polymorphic, no hard FK (same rationale as `audit_events`). |
| `related_resource_id` | UUID | NULL | Deep-link target. |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT `false` | — |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `read_at` | TIMESTAMPTZ | NULL | — |

### `notification.notification_preferences`
Per-user opt-in/out per event type and channel (FR-NOT-03, Could-have).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `event_type` | VARCHAR(50) | NOT NULL | Matches `notifications.type`. |
| `channel` | VARCHAR(20) | NOT NULL, CHECK IN (`EMAIL`,`IN_APP`) | — |
| `enabled` | BOOLEAN | NOT NULL, DEFAULT `true` | — |
| *(unique)* | — | UNIQUE(`user_id`, `event_type`, `channel`) | — |

---

## Business Rules Enforced Here

- **BR-07** — Notify creator and assignee when a case reaches `RESOLVED` (triggered by `workflow-service`, delivered by this service).

## Events Consumed

```text
CaseAssigned, ApprovalRequested, CaseResolved, CommentMentioned
```

## API Surface (owned endpoints)

```text
GET    /api/v1/notifications
POST   /api/v1/notifications/{id}/read
GET    /api/v1/notification-preferences
PUT    /api/v1/notification-preferences
```
