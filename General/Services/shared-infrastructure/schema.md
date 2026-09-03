# `shared-infrastructure` — Schema & Description

**Postgres schema:** `shared`
**Owns:** `shared.outbox_events`
**Also documents:** Redis key conventions, Kafka event envelope, S3 object-key convention — used platform-wide
**SRS coverage:** FR-EVT-01…09 (Event-Driven Architecture & Transactional Outbox)

## Purpose

This is not a business-domain service — it's the cross-cutting plumbing every other service uses to publish events reliably. Every service writes its own business row **and** a row here, in the same database transaction (the transactional outbox pattern), so an event is never "lost" between a committed business change and a Kafka publish that might fail.

## Depends On

- `organization.organizations` — tenant ownership, so a stuck DLQ can be triaged per tenant without a join.

## Depended On By

**Every** service that publishes domain events (`case_management`, `task`, `document`, `approval`, `collaboration`, `workflow`) writes into `shared.outbox_events`. A background **Outbox Publisher** process (owned by this module, not any business service) reads pending rows and relays them to Kafka.

---

## Table

### `shared.outbox_events`
Transactional outbox — written in the same transaction as the business change it describes (FR-EVT-01…09).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | Also usable as the Kafka message key for per-aggregate ordering. |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | Lets the team filter/triage a stuck DLQ per tenant without a join. |
| `aggregate_type` | VARCHAR(50) | NOT NULL | E.g. `CASE`, `TASK`, `DOCUMENT`, `APPROVAL` — identifies the owning service's entity. |
| `aggregate_id` | UUID | NOT NULL | ID of the entity the event is about (in its owning service's schema). |
| `event_type` | VARCHAR(100) | NOT NULL | E.g. `CaseCreated`, `ApprovalCompleted`. |
| `event_version` | INTEGER | NOT NULL, DEFAULT 1 | Schema version for backward-compatible evolution (FR-EVT-06). |
| `payload` | JSONB | NOT NULL | Full event body (see envelope below). |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`PUBLISHED`,`FAILED`) | — |
| `retry_count` | INTEGER | NOT NULL, DEFAULT 0 | Feeds DLQ routing after repeated failures (FR-EVT-05). |
| `available_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Next-retry-eligible time, for exponential backoff (FR-EVT-03). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `published_at` | TIMESTAMPTZ | NULL | Set once the Outbox Publisher confirms delivery to Kafka. |

---

## Kafka — Event Envelope Convention (used by every service)

```json
{
  "eventId": "uuid",
  "eventType": "CASE_ASSIGNED",
  "eventVersion": 1,
  "tenantId": "uuid",
  "actorId": "uuid",
  "resourceType": "CASE",
  "resourceId": "uuid",
  "timestamp": "2026-09-01T10:00:00Z",
  "metadata": { "previousAssignee": "uuid", "newAssignee": "uuid" }
}
```

| Field | Description |
|---|---|
| `eventId` | Unique per event — consumers use it to de-duplicate (FR-EVT-04). |
| `eventType` | Matches `outbox_events.event_type`. |
| `eventVersion` | Matches `outbox_events.event_version` (FR-EVT-06). |
| `tenantId` | Always present — consumers must scope any downstream write by this. |
| `actorId` | Who/what triggered the event (absent/`null` for system-generated). |
| `resourceType` / `resourceId` | The aggregate the event is about. |
| `timestamp` | Event creation time (not publish time). |
| `metadata` | Event-type-specific payload fields. |

**Topic naming:** `resolve.{aggregate_type}.{event_type}` (e.g. `resolve.case.assigned`), partitioned by `resourceId` where per-case ordering matters (FR-EVT-08, Could-have).

**Initial event catalog:**
```text
CaseCreated          → { caseId }
CaseAssigned         → { caseId, previousAssignee, newAssignee }
CaseStatusChanged    → { caseId, previousStatus, newStatus }
CaseResolved         → { caseId }
TaskCreated          → { taskId, caseId }
TaskCompleted        → { taskId, caseId, completedAt }
DocumentUploaded     → { documentId, caseId, storageKey }
ApprovalRequested    → { approvalId, caseId, approverId }
ApprovalCompleted    → { approvalId, caseId, decision }
```

---

## Redis — Platform-Wide Key Conventions

Redis holds non-authoritative state only (NFR-REL-03 — Postgres remains the source of truth).

| Key Pattern | Value | TTL | Owning Concern |
|---|---|---|---|
| `idempotency:{idempotency_key}` | JSON: `{status_code, response_body, request_hash}` | ~24h | Any state-changing endpoint (FR-RED-03, FR-IDC-01). |
| `session:revoked:{token_id}` | `"1"` | Matches token TTL | `identity-service` (FR-IAM-08). |
| `ratelimit:{tenant_id}:{window}` / `ratelimit:{user_id}:{window}` | Integer counter | Matches window | Rate limiting middleware, applied platform-wide (FR-RED-02). |
| `cache:permissions:{user_id}` | JSON array of permission codes | Short, invalidated on role change | `identity-service` (FR-RED-01). |
| `cache:case:{case_id}:summary` | JSON case summary | Short, invalidated on case update | `case-management-service` (FR-RED-01). |
| `mfa:otp:{user_id}` | Hashed 6-digit code | ~5 min | `identity-service` (FR-IAM-10). |
| `lock:workflow:{case_id}` | `"1"` | Seconds | `workflow-service` (FR-RED-05). |

Every service must degrade gracefully if Redis is unavailable — non-cache-dependent writes continue functioning (FR-RED-04, NFR-REL-01).

## S3-Compatible Object Storage — Key Convention

Owned in practice by `document-service`, documented here because it's a shared external dependency, not a Postgres table:

```text
documents/{tenant_id}/{case_id}/{document_id}/{filename}
```

The tenant-scoped prefix is non-negotiable — it keeps tenant isolation visible even at the storage layer.

## OpenSearch / Elasticsearch — Search Projection

An eventually-consistent projection, not the authoritative source (FR-SRCH-03, Should-have). Fed by a Kafka-driven indexer that consumes the events above; never written synchronously from any service's request path.

```json
{
  "caseId": "uuid",
  "tenantId": "uuid",
  "title": "string",
  "description": "string",
  "status": "INVESTIGATION",
  "priority": "HIGH",
  "assignedUserId": "uuid",
  "assignedTeamId": "uuid",
  "comments": [],
  "documents": [],
  "ocrText": "string",
  "updatedAt": "timestamp"
}
```

---

## Implementation Note

Code-wise, `shared-infrastructure` is where the **Outbox Publisher** background process, the **AuditLogger** interface consumed by every service (see `audit-service/schema.md`), base entity classes, common DTOs/exception types, and the Redis/Kafka client configuration all live — it's a dependency of every other module, and depends on none of them (other than `organization` for the outbox's tenant column).
