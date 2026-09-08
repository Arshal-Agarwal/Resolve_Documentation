# Resolve — Kafka Event Schemas & Topic Contracts

**Scope:** every Kafka topic in the system, its producer, its consumer(s), the shared envelope, and the per-event payload schema.
**Status:** Task 0 candidate for sign-off. Previously-open decisions (marked `[x]` below) have been resolved here; ratify or override them as a team, but nothing below is silently ambiguous anymore.
**Nature:** all publishing goes through the transactional outbox; all topics share one envelope, differing only in `metadata`.

---

## 1. Topic Ownership & Producer/Consumer Map

Topic creation (partition counts, DLQ config) is a one-time Task 1 job. Once a topic exists, the **producer's** service owns its schema.

| Topic | Producer | Consumer(s) | Why |
|---|---|---|---|
| `resolve.case.created` | Case Management | Audit, Search Indexer | Audit trail; search projection (FR-SRCH-03) |
| `resolve.case.assigned` | Case Management | Workflow, Notification, Audit | Rule triggers; "case assigned" alert (FR-NOT-01); trail |
| `resolve.case.statuschanged` | Case Management | Workflow, **Notification (filtered)**, Audit, Search Indexer | Workflow's main rule trigger; **Notification alerts only when `newStatus == REJECTED`** (FR-NOT-02); trail; search projection |
| `resolve.case.resolved` | Case Management | Notification, Audit | "Case resolved" alert (FR-NOT-02); trail |
| `resolve.task.created` | Workflow | Notification, Audit | Assignment alert (assignee in payload); trail |
| `resolve.task.completed` | Workflow | Audit | Case Management checks completion synchronously, not via this event |
| `resolve.document.uploaded` | Document | Audit, **Virus Scanner Worker**, **OCR Worker**, **Metadata Processor**, Search Indexer | Trail; async scan (BR-14); OCR text extraction (FR-DOC-08, C); metadata enrichment; search projection |
| `resolve.document.scanned` *(new — see §4.10)* | Document *(publishes on the Virus Scanner Worker's behalf)* | Audit | Gives the system an actual event for the scan result, instead of only a DB column flip |
| `resolve.approval.requested` | Workflow | Notification, Audit | "Approval needed" alert (FR-NOT-02); trail |
| `resolve.approval.completed` | Workflow | Audit | Left audit-only — see §7 item 6 for why this stays as-is |

**The pattern:**
- **Producers** — Case Management, Workflow, Document: the 3 services owning a lifecycle-bearing entity.
- **Audit** — universal subscriber.
- **Notification** — subscribes only to what a human needs to know about, including the `statuschanged→REJECTED` filter above.
- **Workflow** — both producer (task/approval events) and consumer (`case.*`).
- **Search Indexer, OCR Worker, Metadata Processor, Virus Scanner Worker** — decoupled background workers, not among the 9 numbered service tasks; fold each into its owning service (Document/Case Management) or run as its own small consumer — team's choice, but the consumer relationship itself is real per `SRS.md` §1.2 and must be accounted for in Task 1's consumer-group list.
- **RBAC, Authentication, Organization, User & Team** — no Kafka involvement at all.

### Consumer group naming
One group per consuming service (not per topic), so multi-topic consumers stay coordinated:

```text
resolve-audit-service
resolve-notification-service
resolve-workflow-service
resolve-search-indexer
resolve-virus-scanner-worker
resolve-ocr-worker
resolve-metadata-processor
```

---

## 2. Transactional Outbox Mechanism

Business write + `outbox_events` row commit in **one** DB transaction (FR-EVT-02); a background Outbox Publisher relays `PENDING` rows to Kafka with retry (FR-EVT-03), marking `PUBLISHED` on ack.

**`outbox_events`** (owned by the producing service — Case Management, Workflow, Document):

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | **Not** the Kafka key — see §7 item 2. Unique per row, so using it as the key would scatter one aggregate's events across partitions. |
| `organization_id` | UUID | FK → `organizations.id`, NOT NULL | Per-tenant DLQ triage without a join. |
| `aggregate_type` | VARCHAR(50) | NOT NULL | `CASE`, `TASK`, `DOCUMENT`, `APPROVAL`. |
| `aggregate_id` | UUID | NOT NULL | **This is the Kafka partition key.** |
| `event_type` | VARCHAR(100) | NOT NULL | PascalCase, e.g. `CaseCreated` — see §7 item 1. |
| `event_version` | INTEGER | NOT NULL, DEFAULT 1 | FR-EVT-06. |
| `payload` | JSONB | NOT NULL | Full envelope (§3). |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`PUBLISHED`,`FAILED`) | — |
| `retry_count` | INTEGER | NOT NULL, DEFAULT 0 | Drives DLQ routing — 5 attempts, see §5. |
| `available_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Next-retry-eligible time. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `published_at` | TIMESTAMPTZ | NULL | Set on confirmed delivery. |

**Flow:**
```text
1. Producer: business write + outbox_events insert, same transaction
2. Outbox Publisher: polls PENDING rows (available_at <= now())
3. Publish to Kafka, keyed by aggregate_id, topic = resolve.{aggregate_type}.{event_type}
4. On ack: status = PUBLISHED, published_at = now()
5. On failure: retry_count++, available_at = now() + backoff(retry_count)
6. After 5 failures: route to <topic>.dlq
```

Core writes never depend on Kafka (FR-EVT-09, NFR-REL-01) — a stuck row blocks nothing but its own delivery.

---

## 3. Common Event Envelope

```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000001",
  "eventType": "CaseAssigned",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "CASE",
  "resourceId": "c1a2b3d4-0000-4000-8000-000000000042",
  "timestamp": "2026-09-08T10:00:00Z",
  "metadata": {}
}
```

**JSON Schema (envelope):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "resolve.event.envelope.v1",
  "type": "object",
  "required": ["eventId", "eventType", "eventVersion", "tenantId", "resourceType", "resourceId", "timestamp", "metadata"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "eventType": { "type": "string", "description": "PascalCase, e.g. CaseAssigned. Matches outbox_events.event_type." },
    "eventVersion": { "type": "integer", "minimum": 1 },
    "tenantId": { "type": "string", "format": "uuid" },
    "actorId": { "type": ["string", "null"], "format": "uuid" },
    "resourceType": { "type": "string", "enum": ["CASE", "TASK", "DOCUMENT", "APPROVAL"] },
    "resourceId": { "type": "string", "format": "uuid" },
    "timestamp": { "type": "string", "format": "date-time" },
    "metadata": { "type": "object" }
  }
}
```

**Topic naming:** `resolve.{aggregate_type}.{event_type}`, lowercase aggregate, e.g. `resolve.case.assigned`. Fixed mapping — update both this doc and Task 1's topic list together.

---

## 4. Per-Topic Event Contracts

### 4.1 `resolve.case.created` — `CaseCreated`
**Producer:** Case Management. **Consumers:** Audit, Search Indexer.

**Metadata schema:**
```json
{ "type": "object", "required": ["caseId"], "properties": { "caseId": { "type": "string", "format": "uuid" } } }
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000010",
  "eventType": "CaseCreated",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "CASE",
  "resourceId": "c1a2b3d4-0000-4000-8000-000000000042",
  "timestamp": "2026-09-08T10:00:00Z",
  "metadata": { "caseId": "c1a2b3d4-0000-4000-8000-000000000042" }
}
```

---

### 4.2 `resolve.case.assigned` — `CaseAssigned`
**Producer:** Case Management. **Consumers:** Workflow (may trigger rules), Notification (FR-NOT-01), Audit.

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["caseId"],
  "properties": {
    "caseId": { "type": "string", "format": "uuid" },
    "previousAssignee": { "type": ["string", "null"], "format": "uuid", "description": "Null on first assignment." },
    "newAssignee": { "type": "string", "format": "uuid" }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000011",
  "eventType": "CaseAssigned",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "CASE",
  "resourceId": "c1a2b3d4-0000-4000-8000-000000000042",
  "timestamp": "2026-09-08T10:05:00Z",
  "metadata": {
    "caseId": "c1a2b3d4-0000-4000-8000-000000000042",
    "previousAssignee": null,
    "newAssignee": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f"
  }
}
```

---

### 4.3 `resolve.case.statuschanged` — `CaseStatusChanged`
**Producer:** Case Management. **Consumers:** Workflow (primary rule-evaluation trigger), **Notification (filtered)**, Audit, Search Indexer.

> Notification only acts on this topic when `newStatus == "REJECTED"` — this is the fix for FR-NOT-02, which requires notifying when a case reaches *either* `RESOLVED` (covered by `resolve.case.resolved`) or `REJECTED` (only ever emitted here, and previously not consumed by Notification at all).

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["caseId", "newStatus"],
  "properties": {
    "caseId": { "type": "string", "format": "uuid" },
    "previousStatus": { "type": ["string", "null"], "enum": ["OPEN","ASSIGNED","INVESTIGATION","REVIEW","APPROVED","REJECTED","RESOLVED","REOPENED", null] },
    "newStatus": { "type": "string", "enum": ["OPEN","ASSIGNED","INVESTIGATION","REVIEW","APPROVED","REJECTED","RESOLVED","REOPENED"] }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000012",
  "eventType": "CaseStatusChanged",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "CASE",
  "resourceId": "c1a2b3d4-0000-4000-8000-000000000042",
  "timestamp": "2026-09-08T11:00:00Z",
  "metadata": { "caseId": "c1a2b3d4-0000-4000-8000-000000000042", "previousStatus": "REVIEW", "newStatus": "REJECTED" }
}
```

---

### 4.4 `resolve.case.resolved` — `CaseResolved`
**Producer:** Case Management. **Consumers:** Notification (FR-NOT-02 — alerts creator + assignee, BR-07), Audit.

**Metadata schema:**
```json
{ "type": "object", "required": ["caseId"], "properties": { "caseId": { "type": "string", "format": "uuid" } } }
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000013",
  "eventType": "CaseResolved",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "CASE",
  "resourceId": "c1a2b3d4-0000-4000-8000-000000000042",
  "timestamp": "2026-09-08T15:00:00Z",
  "metadata": { "caseId": "c1a2b3d4-0000-4000-8000-000000000042" }
}
```
*(Notification is expected to look up the case's `created_by` and current assignee itself — the event doesn't carry either.)*

---

### 4.5 `resolve.task.created` — `TaskCreated`
**Producer:** Workflow. **Consumers:** Notification (assignment alert), Audit.

**Metadata schema (fixed — added `assignedToUserId`/`assignedToTeamId`, previously missing despite `kafka.md` stating Notification relies on the assignee being in this payload):**
```json
{
  "type": "object",
  "required": ["taskId", "caseId"],
  "properties": {
    "taskId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "assignedToUserId": { "type": ["string", "null"], "format": "uuid" },
    "assignedToTeamId": { "type": ["string", "null"], "format": "uuid" }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000014",
  "eventType": "TaskCreated",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": null,
  "resourceType": "TASK",
  "resourceId": "d1e2f3a4-0000-4000-8000-000000000099",
  "timestamp": "2026-09-08T11:05:00Z",
  "metadata": {
    "taskId": "d1e2f3a4-0000-4000-8000-000000000099",
    "caseId": "c1a2b3d4-0000-4000-8000-000000000042",
    "assignedToUserId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
    "assignedToTeamId": null
  }
}
```
`actorId: null` here because this example is a workflow-rule-generated task (BR-06), not directly user-created.

---

### 4.6 `resolve.task.completed` — `TaskCompleted`
**Producer:** Workflow. **Consumers:** Audit only — Case Management checks task completion synchronously when gating a transition, not via this event.

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["taskId", "caseId", "completedAt"],
  "properties": {
    "taskId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "completedAt": { "type": "string", "format": "date-time" }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000015",
  "eventType": "TaskCompleted",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "TASK",
  "resourceId": "d1e2f3a4-0000-4000-8000-000000000099",
  "timestamp": "2026-09-08T14:00:00Z",
  "metadata": { "taskId": "d1e2f3a4-0000-4000-8000-000000000099", "caseId": "c1a2b3d4-0000-4000-8000-000000000042", "completedAt": "2026-09-08T14:00:00Z" }
}
```

---

### 4.7 `resolve.document.uploaded` — `DocumentUploaded`
**Producer:** Document. **Consumers:** Audit, Virus Scanner Worker, OCR Worker (Could-have), Metadata Processor, Search Indexer.

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["documentId", "caseId", "storageKey"],
  "properties": {
    "documentId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "storageKey": { "type": "string", "description": "S3-compatible object key; documents/{tenant_id}/{case_id}/{document_id}/{filename}" }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000016",
  "eventType": "DocumentUploaded",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "DOCUMENT",
  "resourceId": "e1f2a3b4-0000-4000-8000-000000000077",
  "timestamp": "2026-09-08T12:00:00Z",
  "metadata": {
    "documentId": "e1f2a3b4-0000-4000-8000-000000000077",
    "caseId": "c1a2b3d4-0000-4000-8000-000000000042",
    "storageKey": "documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/c1a2b3d4-0000-4000-8000-000000000042/e1f2a3b4-0000-4000-8000-000000000077/statement.pdf"
  }
}
```

---

### 4.8 `resolve.approval.requested` — `ApprovalRequested`
**Producer:** Workflow. **Consumers:** Notification (FR-NOT-02 — "approval needed" alert), Audit.

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["approvalId", "caseId"],
  "properties": {
    "approvalId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "approverId": { "type": ["string", "null"], "format": "uuid", "description": "Null if only required_role is resolved, not yet a specific approver (FR-APR-06)." }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000017",
  "eventType": "ApprovalRequested",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": null,
  "resourceType": "APPROVAL",
  "resourceId": "f1a2b3c4-0000-4000-8000-000000000055",
  "timestamp": "2026-09-08T11:10:00Z",
  "metadata": { "approvalId": "f1a2b3c4-0000-4000-8000-000000000055", "caseId": "c1a2b3d4-0000-4000-8000-000000000042", "approverId": null }
}
```

---

### 4.9 `resolve.approval.completed` — `ApprovalCompleted`
**Producer:** Workflow. **Consumers:** Audit only (kept as-is — see §7 item 6).

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["approvalId", "caseId", "decision"],
  "properties": {
    "approvalId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "decision": { "type": "string", "enum": ["APPROVED", "REJECTED", "EXPIRED"] }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000018",
  "eventType": "ApprovalCompleted",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "APPROVAL",
  "resourceId": "f1a2b3c4-0000-4000-8000-000000000055",
  "timestamp": "2026-09-08T13:00:00Z",
  "metadata": { "approvalId": "f1a2b3c4-0000-4000-8000-000000000055", "caseId": "c1a2b3d4-0000-4000-8000-000000000042", "decision": "APPROVED" }
}
```

---

### 4.10 `resolve.document.scanned` — `DocumentScanned` *(new)*
**Producer:** Document (published once the Virus Scanner Worker reports a result — the worker calls back into Document Service, or Document Service subscribes to the scanner's own completion signal and republishes via its outbox). **Consumers:** Audit.

This closes a real gap: previously `documents.scan_status` was updated out-of-band with no event at all, so nothing downstream could react to a scan finishing except by polling the column.

**Metadata schema:**
```json
{
  "type": "object",
  "required": ["documentId", "caseId", "scanStatus"],
  "properties": {
    "documentId": { "type": "string", "format": "uuid" },
    "caseId": { "type": "string", "format": "uuid" },
    "scanStatus": { "type": "string", "enum": ["CLEAN", "INFECTED", "FAILED"] }
  }
}
```

**Example:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000020",
  "eventType": "DocumentScanned",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": null,
  "resourceType": "DOCUMENT",
  "resourceId": "e1f2a3b4-0000-4000-8000-000000000077",
  "timestamp": "2026-09-08T12:01:30Z",
  "metadata": { "documentId": "e1f2a3b4-0000-4000-8000-000000000077", "caseId": "c1a2b3d4-0000-4000-8000-000000000042", "scanStatus": "CLEAN" }
}
```

---

## 5. Delivery Semantics & Reliability

- **Delivery guarantee:** at-least-once (NFR-REL-02) — every consumer must be idempotent against redelivery (FR-EVT-04).
- **Consumer-side idempotency mechanism:** Workflow's `workflow_tasks.idempotency_key`, derived from the triggering `eventId`, with `UNIQUE(workflow_instance_id, idempotency_key)` (BR-16). Audit, Notification, Search Indexer, and the new document-pipeline workers should track processed `eventId`s per consumer and skip on replay — same shape, no dedicated table documented for them.
- **Ordering (resolved — §7 item 2):** partition key is `aggregate_id`/`resourceId`, never `outbox_events.id`.
- **Retry/backoff (resolved — was unspecified):** base delay 1s, ×2 multiplier, capped at 5 minutes, **5 attempts** before DLQ.
- **DLQ naming (resolved):** `{topic}.dlq`, e.g. `resolve.case.assigned.dlq`.
- **Core-path independence:** none of Case Management's, Workflow's, or Document's synchronous write paths depend on Kafka being reachable (FR-EVT-09/NFR-REL-01) — an outage delays outbox publishing, not the underlying transaction.

---

## 6. Related Redis State (not Kafka, but adjacent)

| Key Pattern | Used by | Purpose |
|---|---|---|
| `lock:workflow:{case_id}` | Workflow consumer | Serializes rule evaluation for a case so redelivered/concurrent events don't double-fire an action (FR-RED-05, Could-have). |
| `idempotency:{idempotency_key}` | HTTP layer, not Kafka | Retried *client* requests, not redelivered *events* — kept separate from consumer idempotency above to avoid conflating the two mechanisms. |

---

## 7. Decisions (previously open — now resolved)

```text
[x] eventType casing → PascalCase, everywhere. SRS §8 and Schemas_High_Level §4.2's own
    SCREAMING_SNAKE_CASE example ("CASE_ASSIGNED") are documentation defects in the source
    docs, not a real alternative — every per-event contract and outbox_events.event_type
    example already committed to PascalCase ("CaseCreated", "ApprovalCompleted").

[x] Partition key mismatch → resolved to aggregate_id/resourceId, never outbox_events.id.
    outbox_events.id is unique per row and would scatter one case's events across
    partitions, defeating the per-aggregate ordering FR-EVT-08 asks for.

[x] resolve.task.created's metadata → assignedToUserId/assignedToTeamId added (§4.5),
    matching what kafka.md already claimed Notification relies on.

[x] DLQ topic naming → `{topic}.dlq`. Retry ceiling → 5 attempts before routing to DLQ.

[x] Outbox Publisher backoff params → base delay 1s, ×2 multiplier, cap 5 minutes,
    5 max attempts (aligned with the DLQ ceiling above).

[x] Virus Scanner Worker, OCR Worker, Metadata Processor, Search Indexer → added as real
    consumers per SRS §1.2, even though none is one of the 9 numbered service tasks in
    tasks.md. Recommend folding Virus Scanner Worker + Metadata Processor into Document
    Service, OCR Worker as its own Could-have worker (FR-DOC-08), Search Indexer as its
    own Should-have worker (FR-SRCH-03).

[x] DocumentScanned event → added (§4.10). Document Service publishes it once a scan
    completes, so scan_status changes are observable as an event, not only a DB column.

[x] Should Notification also consume resolve.task.completed / resolve.approval.completed?
    → Left Audit-only. Neither FR-NOT-01 nor FR-NOT-02 requires it, so adding it isn't
    needed to satisfy any written requirement — revisit only as a Phase 4 nice-to-have
    ("your approval was approved/rejected").

[x] FR-NOT-02 also requires a notification when a case reaches REJECTED, not only
    RESOLVED. REJECTED is only ever emitted via resolve.case.statuschanged, which
    Notification previously didn't consume at all. Fixed: Notification now consumes
    resolve.case.statuschanged, filtered to newStatus == "REJECTED" (§4.3).

[ ] Still open: FR-SRCH-03 wants comment content indexed too, but no comment domain
    event exists in the frozen FR-EVT-01 catalog — Search Indexer currently has nothing
    to consume for comment content. Either add a CommentCreated event to the catalog, or
    have Search Indexer pull comments synchronously after a case event. Not resolved
    here since it requires expanding the event catalog itself, which is a Task 0 team
    decision, not a documentation fix.
```

---

## 8. Requirement Traceability

| Concern | Traces to |
|---|---|
| Event catalog (10 event types, incl. DocumentScanned) | FR-EVT-01 |
| Same-transaction outbox write | FR-EVT-02 |
| Reliable relay + backoff | FR-EVT-03 |
| Idempotent consumers | FR-EVT-04, BR-16 |
| DLQ routing | FR-EVT-05 |
| Versioned event schemas | FR-EVT-06 |
| Consumer groups per responsibility | FR-EVT-07 |
| Per-aggregate ordering | FR-EVT-08 |
| Core paths independent of Kafka | FR-EVT-09, NFR-REL-01 |
| Notification triggers (assignment/approval-request/resolution/**rejection**) | FR-NOT-01, FR-NOT-02, BR-07 |
| Idempotent rule execution | BR-16, `workflow_tasks.idempotency_key` |
| Virus scan gating downloads, scan-result event | BR-14 |
| OCR text extraction | FR-DOC-08 |
| Kafka-driven search indexing | FR-SRCH-03 |

---

*Compiled from `Resolve_Documentation/{kafka.md, General/Schemas_High_Level.md, General/SRS.md, tasks.md}`. All items in §7 are resolved except the last (comment indexing), which needs a Task 0 team decision to expand the event catalog before it can be closed here.*
