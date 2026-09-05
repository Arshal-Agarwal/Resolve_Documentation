# Resolve — Service Assignment Task List (v3: One Task = One Service)

**Source of truth:** `Resolve_Documentation/General/{SRS.md, Features_Phases.md, Schemas_High_Level.md, System_Architecture.png}`
**Supersedes:** v2. Same 9 functional services and same architectural reasoning (sync vs. async, who calls whom) — but no task now bundles more than one service. The previous "Owner A gets 4 services in one task" is now 4 separate tasks; a person can still be assigned several tasks in sequence, but each task itself is atomic.

---

## Task Sequence

```text
Task 0  → All 4, together — API contracts & event schema freeze
Task 1  → All 4, together — shared infrastructure (Kafka, outbox, Docker, CI, observability)
Task 2  → Organization Service              (1 service)
Task 3  → User & Team Service                (1 service)
Task 4  → Authentication Service             (1 service)
Task 5  → RBAC Service                       (1 service)
Task 6  → Case Management Service            (1 service)
Task 7  → Workflow & Work Execution Service  (1 service)
Task 8  → Document Service                   (1 service)
Task 9  → Audit Service                      (1 service)
Task 10 → Notification Service               (1 service)
Task 11 → All 4, together — cross-service integration & reliability pass
Task 12 → All 4, together — final submission
```

Tasks 2–10 are 9 independent, single-service tasks. Suggested sequential build order within each is noted per task (e.g., Organization before User, since a user can't exist without a tenant to belong to) — but nothing stops two people building two different services at the same time once Tasks 0–1 are done.

---

## Task 0 — API Contracts & Event Schema Freeze

**Assign to:** All 4, together.
- [ ] OpenAPI spec for every REST endpoint across all 9 services.
- [ ] JSON Schema for every Kafka event topic (list is in Task 1).
- [ ] Common conventions: `/api/v1` prefix, pagination/error format, `Idempotency-Key`, service-to-service auth.
- [ ] Confirm table ownership — one table, one owning service, no exceptions.
- [ ] Decide database-per-service vs. shared instance with schema separation.

**Exit criteria:** every endpoint and event has a written contract.

---

## Task 1 — Shared Infrastructure Setup

**Assign to:** All 4, together (or one infra-lead + reviewers).
- [ ] Kafka (or Redpanda) via Docker Compose; topic list:
  ```text
  resolve.case.created / .assigned / .statuschanged / .resolved
  resolve.task.created / .completed
  resolve.document.uploaded
  resolve.approval.requested / .completed
  ```
- [ ] Partitioning by `resourceId`; consumer groups per consuming service; DLQ topics.
- [ ] Shared outbox library/pattern for every producing service.
- [ ] Common event envelope (`eventId`, `eventType`, `eventVersion`, `tenantId`, `actorId`, `resourceType`, `resourceId`, `timestamp`, `metadata`).
- [ ] Docker Compose: Postgres (per-service), Redis, Kafka, MinIO/S3-compatible storage.
- [ ] Shared logging/metrics/tracing config (structured logs, Prometheus scrape config, OpenTelemetry propagated through both REST and Kafka headers).
- [ ] Per-service CI workflow + a top-level workflow that runs the full Compose stack for integration tests.

**Exit criteria:** `docker compose up` brings up everything; a test event published to any topic is consumed by a dummy consumer; all 4 developers run the stack locally.

---

## Task 2 — Organization (Tenant) Service

**Owner:** ______________________
**Build order note:** build first within the identity group — everything else references a tenant.

**Endpoints:** `POST /api/v1/organizations`, `GET /api/v1/organizations/{id}`
**Owned table:** `organizations`
**Nature:** Synchronous REST only.

**Deliverables:**
- [ ] Tenant provisioning/onboarding with an isolated data boundary (FR-ORG-01)
- [ ] Tenant lifecycle state (`ACTIVE`/`SUSPENDED`), managed by System Administrators
- [ ] `organization_id` is the value every other service's tenant-owned table foreign-keys to — get this contract stable before Task 3 starts

---

## Task 3 — User & Team Service

**Owner:** ______________________
**Depends on:** Task 2 (a user needs an org to belong to).

**Endpoints:** `GET/POST /api/v1/users`, `PATCH /api/v1/users/{id}`, plus team CRUD endpoints (define exact paths in Task 0)
**Owned tables:** `users` (profile fields — `password_hash` is written here but *owned/managed* by Auth), `teams`, `team_members`
**Nature:** Synchronous REST only.

**Deliverables:**
- [ ] User provisioning within a tenant (FR-ORG-02, FR-IAM-01)
- [ ] Team creation and management, team membership (FR-ORG-03)
- [ ] Every user associated with exactly one organization (FR-ORG-04)
- [ ] Note the shared-table wrinkle with Authentication below and agree the exact read/write split in Task 0.

---

## Task 4 — Authentication Service

**Owner:** ______________________
**Depends on:** Task 3 (needs the `users` table to exist).

**Endpoints:** `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`
**Owned data:** `users.password_hash` (shares the `users` table with Task 3 — agree in Task 0 whether Auth owns that one column exclusively or calls User Service for it), Redis session state
**Nature:** Synchronous REST only — called by every other service indirectly (via token validation).

**Deliverables:**
- [ ] Login issuing a signed JWT on valid credentials, rejecting invalid attempts without issuing a token (FR-IAM-02)
- [ ] Logout / session invalidation (FR-IAM-04)
- [ ] Password hashing (bcrypt/argon2), plaintext never stored (FR-IAM-05)
- [ ] A reusable authentication-middleware artifact (library or documented pattern) every other service imports to validate tokens **without** a network call back to this service on every request — decide the exact mechanism (local JWT signature verification vs. introspection endpoint) in Task 0.
- [ ] `tenant_id` claim embedded and validated (FR-TEN-04)

---

## Task 5 — RBAC (Access Control) Service

**Owner:** ______________________
**Can build in parallel with Task 4** — roles/permissions don't depend on the login mechanism itself, only on `users` existing (Task 3).

**Interface:** internal permission-check contract other services call (define the exact shape — REST endpoint or embedded library — in Task 0)
**Owned tables:** `roles`, `permissions`, `user_roles`, `role_permissions`
**Nature:** Synchronous — called on the hot path of nearly every write across every other service.

**Deliverables:**
- [ ] Atomic permission model, roles as named permission collections (FR-AUZ-01)
- [ ] Many-to-many Users↔Roles and Roles↔Permissions (FR-AUZ-04)
- [ ] Centralized permission evaluation, deny-by-default (FR-AUZ-02/03)
- [ ] Given this is called constantly, benchmark it early — a slow permission check becomes everyone's problem, not just yours.

---

## Task 6 — Case Management Service

**Owner:** ______________________
**Depends on:** Tasks 2–5 (needs org context, users, and permission checks working end-to-end).

**Endpoints:**
```text
POST/GET/PATCH /api/v1/cases, /api/v1/cases/{id}
POST /api/v1/cases/{id}/assign, /transition, /close, /reopen
GET  /api/v1/search/cases
POST/GET /api/v1/cases/{id}/comments
```
**Owned tables:** `cases`, `case_assignments`, `case_status_history`, `comments`
**Nature:** Synchronous REST + Kafka producer.

**Owned Kafka topics (produces):** `resolve.case.created`, `.assigned`, `.statuschanged`, `.resolved`

**Deliverables:**
- [ ] Case CRUD, assignment, state machine, status history, optimistic locking (FR-CASE-01…09)
- [ ] Comments — create + chronological listing (FR-COL-01/02)
- [ ] Postgres-backed case search (FR-SRCH-01/02)
- [ ] Events written via the shared outbox pattern from Task 1
- [ ] Synchronous call to Task 5 (permission gate) and Task 7 (task/approval completion check before allowing certain transitions)

---

## Task 7 — Workflow & Work Execution Service

**Owner:** ______________________
**Depends on:** Tasks 2–5; consumes events from Task 6.

**Endpoints:**
```text
POST/GET/PATCH /api/v1/cases/{id}/tasks, /tasks/{taskId}
POST/GET /api/v1/cases/{id}/approvals
```
**Owned tables:** `tasks`, `task_dependencies`, `approvals`, `workflow_definitions`, `workflow_instances`, `workflow_tasks`
**Nature:** Synchronous REST + Kafka producer + Kafka consumer.

**Owned Kafka topics:**
- Produces: `resolve.task.created`, `.completed`, `resolve.approval.requested`, `.completed`
- Consumes: `resolve.case.*`

**Deliverables:**
- [ ] Task CRUD, assignment, completion (FR-TASK-01…04)
- [ ] Approval request/decision, approval-gated blocking (FR-APR-01…04)
- [ ] Rule evaluation on consumed case events (FR-WF-01/02/04)
- [ ] Idempotent rule execution via `workflow_tasks.idempotency_key` (BR-16)
- [ ] Synchronous endpoint Task 6 calls to check "is this case's required work complete"

---

## Task 8 — Document Service

**Owner:** ______________________
**Depends on:** Tasks 2–5 (org context, permission checks).

**Endpoints:** `POST/GET /api/v1/cases/{id}/documents`
**Owned tables:** `documents`, `document_versions`; owns S3-compatible object storage
**Nature:** Synchronous REST + Kafka producer.

**Owned Kafka topics (produces):** `resolve.document.uploaded`

**Deliverables:**
- [ ] Metadata to Postgres, binary bytes to S3-compatible storage (FR-DOC-01/02)
- [ ] Listing/download with permission + tenant checks (FR-DOC-03)
- [ ] Upload guardrails: max size, blocked unsafe MIME types (FR-API-06)
- [ ] Publish `document.uploaded` via the shared outbox pattern (FR-DOC-04)

---

## Task 9 — Audit Service

**Owner:** ______________________
**Depends on:** Task 1 (Kafka) only — this is the most decoupled task in the list.

**Endpoints:** `GET /api/v1/audit`
**Owned table:** `audit_events`
**Nature:** **Pure async consumer.** No writes come in via REST from other services — it subscribes to Kafka.

**Owned Kafka topics (consumes):** every topic across every other service — the one universal subscriber.

**Deliverables:**
- [ ] Consume every domain event and translate it into an `audit_events` row (FR-AUD-01/02)
- [ ] Preserve `request_id`/`trace_id` from the event envelope for correlation (FR-AUD-03)
- [ ] Authorized, queryable `GET /audit` (FR-AUD-04)
- [ ] Idempotent consumption — a redelivered event must not create a duplicate row (FR-EVT-04)
- [ ] Document explicitly that this audit trail is now *eventually consistent* with the source of truth (a few hundred ms to seconds behind), not written in the same transaction as the action — this is a real behavior change from a monolith design and should be a stated, agreed decision, not a surprise.

---

## Task 10 — Notification Service

**Owner:** ______________________
**Depends on:** Task 1 (Kafka) only — same decoupling as Audit.

**Endpoints:** `GET /api/v1/notifications` (new — add to the Task 0 contract, not in the original SRS list but needed for an inbox)
**Owned tables:** `notifications`, `notification_preferences`
**Nature:** Pure async consumer, plus its own small read API.

**Owned Kafka topics (consumes):** `resolve.case.assigned`, `resolve.approval.requested`, `resolve.case.resolved` (+ any `@mention` event defined in Task 0)

**Deliverables:**
- [ ] Notify on case/task assignment, approval request, and case resolution (FR-NOT-01/02)
- [ ] Per-user notification preferences by event type + channel (FR-NOT-03)
- [ ] Idempotent consumption — no duplicate notifications on redelivery (FR-EVT-04)

---

## Task 11 — Cross-Service Integration & Reliability Pass

**Assign to:** All 4, together, once Tasks 2–10 individually pass their own tests.
- [ ] Wire and test the two real synchronous calls: Case→RBAC, Case→Workflow.
- [ ] End-to-end Kafka test: publish `CaseCreated`, confirm it lands in Audit and (where relevant) Notification.
- [ ] Full SRS §10 scenario as a genuine cross-process integration test.
- [ ] Chaos-test the async path: kill Kafka mid-flow, confirm sync services keep accepting writes, confirm Audit/Notification catch up after recovery.
- [ ] Confirm rejection paths (unauthorized, cross-tenant, invalid transition) still work with Auth as a real network hop.

## Task 12 — Final Integration, Documentation & Submission

**Assign to:** All 4, together.
- [ ] Failure-mode verification per service, plus "one service down while others stay up."
- [ ] Contract verification against the Task 0 freeze.
- [ ] Architecture diagram updated to the real 9-service topology; ADR for the monolith→multi-service shift.
- [ ] `docker compose up` from clean, bringing up all 9 services + Kafka + Postgres(es) + Redis + MinIO.
- [ ] Demo dataset + rehearsed presentation.

---

## Quick Reference

| Task | Service | Owner | Depends On |
|---|---|---|---|
| 0 | *(shared)* Contracts & Event Schema | All 4 | — |
| 1 | *(shared)* Shared Infrastructure | All 4 | Task 0 |
| 2 | Organization Service | ______ | Task 1 |
| 3 | User & Team Service | ______ | Task 2 |
| 4 | Authentication Service | ______ | Task 3 |
| 5 | RBAC Service | ______ | Task 3 |
| 6 | Case Management Service | ______ | Tasks 2–5 |
| 7 | Workflow & Work Execution Service | ______ | Tasks 2–5, consumes Task 6 events |
| 8 | Document Service | ______ | Tasks 2–5 |
| 9 | Audit Service | ______ | Task 1 only |
| 10 | Notification Service | ______ | Task 1 only |
| 11 | *(shared)* Integration & Reliability | All 4 | Tasks 2–10 |
| 12 | *(shared)* Final Submission | All 4 | Task 11 |

---

*Derived from `Resolve_Documentation/General/{SRS.md, Features_Phases.md, Schemas_High_Level.md, System_Architecture.png}`. Each task now maps to exactly one service — assign multiple sequential tasks to the same person where it matches their earlier cluster (e.g., Tasks 2–5 to one person), or spread them differently; the dependency column tells you what has to exist first either way.*