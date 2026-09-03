# `case-management-service` — Schema & Description

**Postgres schema:** `case_management`
**Owns:** `case_management.cases`, `case_management.case_assignments`, `case_management.case_status_history`
**SRS coverage:** FR-CASE-01…15 (Case Management)

## Purpose

The central business entity of the whole platform. `case_management-service` owns the Case lifecycle: creation, assignment, the validated state machine, and the append-only history of every status change. Nearly every other service (`task`, `document`, `approval`, `collaboration`, `workflow`, `audit`) exists to attach behavior or data to a case, so this service's contract is the most-consumed one in the system.

## Depends On

- `organization.organizations` — tenant ownership (FR-TEN-01).
- `identity.users` — `created_by`, `assigned_to_user_id`.
- `organization.teams` — `assigned_to_team_id`.

## Depended On By

`task-service`, `document-service`, `approval-service`, `collaboration-service`, `workflow-service`, and `audit-service` all hold a `case_id` foreign key pointing here.

---

## Tables

### `case_management.cases`
The central business entity (FR-CASE-01…15).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | Every query filters on this (FR-TEN-02). |
| `case_number` | VARCHAR(50) | UNIQUE, NOT NULL | Human-friendly reference (e.g. `CASE-2026-00042`) for UI/notifications. |
| `title` | VARCHAR(255) | NOT NULL | — |
| `description` | TEXT | NULL | — |
| `status` | VARCHAR(30) | NOT NULL, DEFAULT `'OPEN'`, CHECK IN (`OPEN`,`ASSIGNED`,`INVESTIGATION`,`REVIEW`,`APPROVED`,`REJECTED`,`RESOLVED`,`REOPENED`) | Must follow the state machine (FR-CASE-05, BR-01). |
| `priority` | VARCHAR(20) | NOT NULL, DEFAULT `'MEDIUM'`, CHECK IN (`LOW`,`MEDIUM`,`HIGH`) | Drives rules like BR-06 (auto-create senior task on `HIGH`) — evaluated by `workflow-service`. |
| `amount` | NUMERIC(15,2) | NULL | Monetary value where relevant; drives threshold-based approval routing (BR-05). |
| `created_by` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `assigned_to_user_id` | UUID | FK → `identity.users.id`, NULL | Current individual assignee (mutually optional with team assignment). |
| `assigned_to_team_id` | UUID | FK → `organization.teams.id`, NULL | Current team assignee. |
| `version` | INTEGER | NOT NULL, DEFAULT 0 | Optimistic-locking counter (FR-CASE-08, FR-IDC-02). |
| `is_archived` | BOOLEAN | NOT NULL, DEFAULT `false` | Soft-delete flag (FR-CASE-14). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `resolved_at` | TIMESTAMPTZ | NULL | Set when status reaches `RESOLVED`. |
| `closed_at` | TIMESTAMPTZ | NULL | Set on explicit close (FR-CASE-10), distinct from resolution. |

### `case_management.case_assignments`
Full history of who a case was assigned to and by whom — distinct from the current pointer on `cases` (FR-CASE-04).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `assigned_to_user_id` | UUID | FK → `identity.users.id`, NULL | — |
| `assigned_to_team_id` | UUID | FK → `organization.teams.id`, NULL | — |
| `assigned_by` | UUID | FK → `identity.users.id`, NOT NULL | Must hold `CASE_ASSIGN` permission, checked via `identity-service` (BR-17). |
| `assigned_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

### `case_management.case_status_history`
Append-only log of every state transition (FR-CASE-07).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `from_status` | VARCHAR(30) | NULL | `NULL` for the initial `OPEN` record. |
| `to_status` | VARCHAR(30) | NOT NULL | — |
| `changed_by` | UUID | FK → `identity.users.id`, NULL | `NULL` if system-triggered (e.g., `workflow-service` auto-transition). |
| `reason` | TEXT | NULL | Required for close/reopen (FR-CASE-10). |
| `changed_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

---

## Business Rules Enforced Here

- **BR-01** — Case status may only follow the defined lifecycle: `OPEN → ASSIGNED → INVESTIGATION → REVIEW → {APPROVED → RESOLVED, REJECTED → REOPENED}`.
- **BR-02** — A transition is valid only if the caller holds the permission required for it (checked via `identity-service`).
- **BR-03** — A case cannot advance past a stage with incomplete required tasks (checked via `task-service`).
- **BR-04** — A case cannot reach `APPROVED`/`RESOLVED` unless all required approvals are `APPROVED` (checked via `approval-service`).
- **BR-10** — Concurrent updates to the same case: the second write is rejected with HTTP 409 if based on a stale `version`.

## Events Published (via `shared.outbox_events`, same transaction)

```text
CaseCreated, CaseAssigned, CaseStatusChanged, CaseResolved
```

## API Surface (owned endpoints)

```text
POST   /api/v1/cases
GET    /api/v1/cases
GET    /api/v1/cases/{id}
PATCH  /api/v1/cases/{id}
POST   /api/v1/cases/{id}/assign
POST   /api/v1/cases/{id}/transition
GET    /api/v1/cases/{id}/history
POST   /api/v1/cases/{id}/close
POST   /api/v1/cases/{id}/reopen
```
