# `audit-service` — Schema & Description

**Postgres schema:** `audit`
**Owns:** `audit.audit_events`
**SRS coverage:** FR-AUD-01…08, BR-12, BR-13

## Purpose

An append-only, immutable log of every significant action across the entire platform. This is intentionally the one service allowed to reference "any resource in any other service" **without a hard foreign key** — `resource_type` + `resource_id` is a polymorphic pointer, because audit must be able to log an event about a case, a task, a document, or a role change without taking a schema dependency on all of them.

## Depends On

- `organization.organizations` — tenant ownership.
- `identity.users` — actor (nullable for system-triggered events).
- **No hard FK to any business entity** — `resource_type`/`resource_id` are deliberately unconstrained (see above).

## Depended On By

- Every other service **writes** here (via a shared `AuditLogger` interface exposed by this module — in-process call, same transaction as the business change it's logging, per BR-12).
- Compliance/auditor-facing UI reads exclusively from here.

---

## Tables

### `audit.audit_events`
Append-only, immutable log of every significant action (FR-AUD-01…08, BR-12, BR-13).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | — |
| `actor_id` | UUID | FK → `identity.users.id`, NULL | `NULL` for system-triggered actions (`workflow-service`, expiry jobs). |
| `action` | VARCHAR(100) | NOT NULL | E.g. `CASE_ASSIGNED`, `DOCUMENT_UPLOADED`. |
| `resource_type` | VARCHAR(50) | NOT NULL | E.g. `CASE`, `TASK`, `APPROVAL` — polymorphic, no FK. |
| `resource_id` | UUID | NOT NULL | ID of the affected resource in its owning service's schema. |
| `old_value` | JSONB | NULL | — |
| `new_value` | JSONB | NULL | — |
| `request_id` | VARCHAR(100) | NULL | Correlates to the originating HTTP request (FR-AUD-03). |
| `trace_id` | VARCHAR(100) | NULL | Correlates to the distributed trace (NFR-OBS-01). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Immutable — no `updated_at`, no delete path, ever. |

---

## Business Rules Enforced Here

- **BR-12** — Every state-changing action across the platform must produce a corresponding audit event; an action without one is considered incomplete (enforced by convention/interceptor, not by this schema alone).
- **BR-13** — Audit events, once written, are never updated or deleted. Corrections are new, subsequent audit events.

## API Surface (owned endpoints)

```text
GET    /api/v1/audit?resourceType=&resourceId=&from=&to=
GET    /api/v1/cases/{caseId}/timeline        (human-readable projection of this + case_status_history)
```

## Implementation Note

Because every other service calls into this one on nearly every write, `audit-service`'s in-process interface (e.g., `AuditLogger.log(action, resourceType, resourceId, oldValue, newValue)`) should be treated as **shared infrastructure**, available to all modules — similar in spirit to `shared-infrastructure`, even though its data lives in its own schema.
