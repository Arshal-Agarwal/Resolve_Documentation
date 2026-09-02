# Resolve — Unified Feature List & Phased Delivery Plan

**Source of truth:** `Resolve_SRS_Consolidated.md` (v1.0)
**Project:** Resolve — Enterprise Case & Workflow Platform (end-semester backend engineering project, 4-developer team)
**Document Status:** Consolidated from four independently drafted phase/feature plans. This document is now the **single bottlenecked reference** for build order — if a feature isn't in here with a phase assigned, it isn't scheduled.

> Every feature line keeps its original SRS ID (`FR-…`, `NFR-…`, `BR-…`) so this plan stays traceable back to the SRS and forward into sprint tickets and test cases, per the SRS §9.4 chain: `SRS Requirement → MoSCoW Priority → Feature → Implementation → Test Case`.

---

## 1. How Phasing Works

```text
Phase 0 → Documentation, Architecture & Contracts     (freeze the shared understanding before parallel work starts)
Phase 1 → Foundational Platform / Bootstrap           (the skeleton every later feature depends on)
Phase 2 → Must Have (M)                                (the system is not "done" without these)
Phase 3 → Should Have (S)                              (important, ships if time allows — usually does)
Phase 4 → Could Have (C)                               (polish / stretch — ships only if capacity remains)
Phase 5 → Won't Have — This Release (W)                (explicitly deferred, documented so it isn't rediscovered later)
Phase 6 → Final Integration, Stabilization & Submission (cross-phase closing work, not a new priority tier)
```

Phase 0 is prerequisite design work, not a runtime feature. Phase 6 is finalization, not a MoSCoW category — it verifies that everything built in Phases 1–4 actually works together.

### Phase Summary

| Phase | Theme | Basis | Approx. Scope |
|---|---|---|---|
| 0 | Docs, architecture, contracts, environment setup | Prerequisite | 21 deliverables |
| 1 | Foundational platform / bootstrap | Prerequisite | ~19 requirements |
| 2 | Core Resolve functionality | MUST HAVE | ~55 requirements |
| 3 | Reliability & extended capabilities | SHOULD HAVE | ~35 requirements |
| 4 | Advanced capabilities | COULD HAVE | ~25 requirements |
| 5 | Explicitly deferred backlog | WON'T HAVE (this release) | 7 FR items + broader out-of-scope list |
| 6 | Final integration, stabilization, submission | Cross-phase | 11 closing activities |

---

## 2. Phase 0 — Documentation, Architecture & Contracts

**Objective:** Freeze the shared understanding of the system before substantial parallel implementation begins. This phase produces no runtime features — it produces the artifacts that let four developers build concurrently without colliding.

### 2.1 Requirements & Traceability
- **Requirements baseline** — Finalize the SRS; freeze functional requirements, NFRs, business rules, and constraints, including the initial and out-of-scope boundaries.
- **MoSCoW feature backlog** — Stand up a backlog with columns for Feature ID, Feature Name, SRS Requirement IDs, MoSCoW Priority, Phase, Owner, Acceptance Criteria, Test Case IDs, and Status, so this document's items become trackable tickets.
- **Requirements traceability convention** — Fix the `Requirement → Feature → Implementation → Test Case` chain (SRS §9.4) so every ticket references an `FR-`/`NFR-`/`BR-` ID from the start.

### 2.2 Domain & Data Model
- **Domain model freeze** — Lock the core entities: `Organization, User, Team, Role, Permission, Case, CaseAssignment, CaseStatusHistory, Task, TaskDependency, Comment, Document, Approval, AuditEvent`, plus the extended entities introduced in later phases: `WorkflowDefinition, WorkflowInstance, WorkflowTask, OutboxEvent, Notification, NotificationPreference, DocumentVersion`.
- **Database schema (ER diagram)** — Design PostgreSQL tables, relationships, constraints, indexes, tenant ownership columns, timestamps, and optimistic-lock version columns for the core table set in SRS §7.
- **Migration strategy** — Freeze migration tooling (Flyway/Liquibase), naming convention, review process, and seed/demo-data strategy (supports FR-QA-05).

### 2.3 Module Boundaries & Architecture
- **Module boundaries** — Freeze the package/module split: `identity, organization, case_management, task, workflow, approval, document, audit, notification, shared` (NFR-ARCH-02); document each module's responsibilities, dependencies, and public interface.
- **Module contracts** — Define the in-process interfaces and DTOs used for module-to-module communication (e.g., `Case Module → Audit Interface → Audit Module`), per NFR-ARCH-03 / SRS §9.2.
- **Architecture diagram** — Produce the modular-monolith diagram and mark which components (PostgreSQL, Redis, Kafka, object storage, search, observability stack) come online in which phase.
- **Architecture Decision Records (ADRs)** — Stand up an ADR template and log the founding decisions: modular monolith first (extract only with justification), PostgreSQL as system of record, tenant isolation approach, RBAC model, case state machine, optimistic locking, API versioning strategy, Redis usage boundary, Kafka + outbox pattern, search architecture.

### 2.4 API & Event Contracts
- **REST API inventory** — Freeze the initial API surface listed in SRS §8.
- **OpenAPI contract** — Define every endpoint's method, request/response schema, authentication, authorization, validation rules, status codes, and error format (FR-API-05, SRS §9.1).
- **Common API conventions** — Freeze `/api/v1` versioning, error-response format, pagination, filtering, sorting, date/time format, UUID format, the authentication header, `Idempotency-Key` handling, and HTTP status-code conventions.
- **Domain event catalog** — Define the initial event set (`CaseCreated, CaseAssigned, CaseStatusChanged, CaseResolved, TaskCreated, TaskCompleted, DocumentUploaded, ApprovalRequested, ApprovalCompleted`), and for each, its producer, consumers, payload shape, version, and correlation metadata.
- **Event JSON Schemas** — Write versioned JSON Schema contracts for the events above, supporting backward-compatible evolution (FR-EVT-06).

### 2.5 Team & Delivery Setup
- **Tenant isolation design** — Document the tenant boundary end-to-end: `JWT → tenant_id → Authorization → Repository/Query Tenant Filter → Tenant-owned Resource`.
- **Repository & Git workflow** — Freeze branch strategy, pull-request process, code review rules, and naming conventions.
- **Development environment** — Prepare the toolchain: Java, Spring Boot, PostgreSQL, Docker, Docker Compose, migrations, JUnit, Mockito, Testcontainers.
- **CI skeleton** — Stand up GitHub Actions for build, lint/format, and unit tests.
- **Coding standards** — Agree on package, DTO, entity, service, repository, exception, logging, and testing conventions (supports NFR-MAINT-01).
- **Definition of Done** — A feature is "done" only once it has its requirement ID, acceptance criteria, contract, implementation, tests, authorization/tenant checks, documentation, code review, and a green CI run.
- **Scale & scope confirmation** — Reconfirm the SRS §5.4 capacity targets and the §5.6 out-of-scope list with the whole team so later phases don't silently creep beyond them.

---

## 3. Phase 1 — Foundational Platform / Bootstrap

**Objective:** Stand up the skeleton every feature phase is built inside — project structure, persistence, containerization, CI, and the minimum identity/tenancy/organization model needed before a single Case can exist.

### 3.1 Project & Environment
- **Modular monolith skeleton** (NFR-ARCH-01) — Stand up the initial codebase as a single, well-modularized deployable.
- **Module package structure** (NFR-ARCH-02) — Organize code by domain boundary, not technical layer.
- **In-process module contracts** (NFR-ARCH-03) — Establish the interface/DTO conventions between modules agreed in Phase 0.
- **Toolchain pin** (NFR-COMPAT-01) — Pin the JVM/Spring Boot version and CI toolchain.
- **Reproducible environment** (NFR-PORT-01, FR-QA-06) — Docker/Docker Compose setup that runs the full stack with no hard-coded configuration.
- **Migration-only schema management** (FR-QA-05, NFR-DATA-03) — Database schema changes go exclusively through versioned migrations, from the very first table.
- **CI pipeline** (FR-QA-04) — Build, test, and lint running automatically on GitHub Actions for every push/PR.

### 3.2 Organizations, Users & Tenancy Skeleton
- **Organization/tenant onboarding** (FR-ORG-01) — Create and provision an isolated organization.
- **User management** (FR-ORG-02) — Create and manage users within a tenant.
- **Team management** (FR-ORG-03) — Create and manage teams within a tenant, so cases/tasks can later be assigned to a group.
- **Tenant context association** (FR-ORG-04) — Every user is associated with exactly one organization; requests are evaluated in that organization's context.
- **Tenant ownership column** (FR-TEN-01) — Every tenant-owned entity carries an `organization_id`/`tenant_id`.
- **Repository-level tenant scoping** (FR-TEN-02) — Enforce tenant filtering at the query layer, not only at the API boundary.
- **Tenant ID in JWT** (FR-TEN-04) — `tenant_id` is present in JWT claims and validated on every authenticated request.

### 3.3 Identity & Authentication Skeleton
- **User provisioning** (FR-IAM-01) — Allow a user to be registered within an organization.
- **Login & JWT issuance** (FR-IAM-02) — Authenticate via email/username + password; issue a signed JWT on success; reject invalid attempts without issuing a token.
- **Logout / session end** (FR-IAM-04) — Invalidate the current session/refresh token.
- **Password hashing** (FR-IAM-05) — Hash all passwords (bcrypt/argon2); plaintext credentials are never persisted.
- **Authentication middleware** (FR-IAM-06) — Reject any request lacking a valid, non-expired token to a protected endpoint.

> **Phase 1 exit criteria:** a user can be provisioned into an organization, authenticate, and receive a tenant-scoped JWT — with the whole stack reproducible via `docker compose up`. This unblocks every Phase 2 feature.

---

## 4. Phase 2 — Must Have (M): Core Resolve Functionality

**Objective:** Every requirement in this phase is required for the platform to meet its own Definition of Done. Nothing here is optional. By the end of Phase 2, the SRS §10 end-to-end acceptance scenario should pass in full.

### 4.1 Authorization & RBAC
- **Atomic permission model** (FR-AUZ-01) — Roles are named collections of discrete permissions (e.g., `CASE_READ`, `CASE_APPROVE`).
- **Centralized policy evaluation** (FR-AUZ-02) — Authorization is evaluated in one policy layer, not scattered across controllers.
- **Deny-by-default enforcement** (FR-AUZ-03) — Any action lacking the required permission in the caller's effective set is denied.
- **Many-to-many role mapping** (FR-AUZ-04) — Users↔Roles and Roles↔Permissions both support many-to-many assignment.

### 4.2 Multi-Tenancy
- **Cross-tenant rejection** (FR-TEN-03) — A request referencing another tenant's resource is rejected or returns 404.

### 4.3 Case Management
- **Case creation** (FR-CASE-01) — Create a case with title, description, priority, and initial status `OPEN`, auto-scoped to the creator's tenant.
- **Case retrieval & listing** (FR-CASE-02) — Fetch a single case and a paginated, filterable, sortable, tenant-scoped list.
- **Case update** (FR-CASE-03) — Update permitted case fields, subject to permission checks.
- **Case assignment** (FR-CASE-04) — Assign or reassign a case to a user and/or team.
- **State machine enforcement** (FR-CASE-05) — Enforce the exact lifecycle `OPEN → ASSIGNED → INVESTIGATION → REVIEW → {APPROVED, REJECTED} → {RESOLVED, REOPENED}`; no skipped states.
- **Transition validation** (FR-CASE-06) — Reject a transition if current state, permissions, required tasks, or required approvals aren't satisfied.
- **Status history** (FR-CASE-07) — Persist every status change to `case_status_history`.
- **Optimistic locking** (FR-CASE-08) — A version column on the case entity rejects conflicting concurrent updates.
- **Transactional mutations** (FR-CASE-09) — Multi-step case changes execute inside a single DB transaction.

### 4.4 Task Management
- **Task creation** (FR-TASK-01) — Create a task under a case (title, description, assignee, priority, due date).
- **Task assignment** (FR-TASK-02) — Assign or reassign a task to a user or team.
- **Task update/completion** (FR-TASK-03) — Update and complete a task, recording `completed_at`.
- **Task listing** (FR-TASK-04) — List tasks for a case, filterable by status, assignee, and due date.

### 4.5 Approval System
- **Approval request** (FR-APR-01) — Submit an approval request capturing requester, approver, and reason.
- **Approval decision** (FR-APR-02) — The designated approver approves or rejects, recording `decision_at`.
- **Approval-gated transitions** (FR-APR-03) — A transition requiring approval stays blocked until an `APPROVED` decision exists.
- **Approval lifecycle recording** (FR-APR-04) — Record the full lifecycle (`created_at`, `expires_at`, `decision_at`, `status`).

### 4.6 Document Management
- **Document metadata** (FR-DOC-01) — Store filename, MIME type, size, checksum, and storage key in PostgreSQL on upload.
- **Binary storage separation** (FR-DOC-02) — File bytes go to S3-compatible object storage, never into the relational database.
- **Document listing & download** (FR-DOC-03) — List and download a case's documents, subject to permission checks.
- **Upload event** (FR-DOC-04) — Publish a `document.uploaded` domain event on successful upload.

### 4.7 Collaboration
- **Comment creation** (FR-COL-01) — Add a comment to a case.
- **Comment listing** (FR-COL-02) — List comments chronologically, scoped by permission.

### 4.8 Audit System
- **Append-only audit events** (FR-AUD-01) — Write an `audit_events` record for every significant state-changing action.
- **Audit event schema** (FR-AUD-02) — Capture `actor_id, tenant_id, action, resource_type, resource_id, old_value, new_value, timestamp`.
- **Correlation identifiers** (FR-AUD-03) — Include `request_id`/`trace_id` where available.
- **Authorized audit API** (FR-AUD-04) — Expose a queryable `GET /audit`, restricted to authorized roles.

### 4.9 Workflow Engine (Baseline)
- **Declarative rules** (FR-WF-01) — Support `IF <condition> THEN <action>` rule definitions.
- **Event-triggered evaluation** (FR-WF-02) — Evaluate applicable rules on relevant case events and execute their actions.

### 4.10 Event-Driven Architecture & Transactional Outbox
- **Domain event publishing** (FR-EVT-01) — Publish events for key state changes (`CaseCreated, CaseAssigned, CaseStatusChanged, CaseResolved, TaskCreated, TaskCompleted, DocumentUploaded, ApprovalRequested, ApprovalCompleted`).
- **Transactional outbox write** (FR-EVT-02) — Write the business update and its outbox row in the same DB transaction.
- **Reliable outbox publisher** (FR-EVT-03) — Relay persisted events to Kafka, retrying on failure.
- **Idempotent consumers** (FR-EVT-04) — Kafka consumers tolerate redelivered/duplicate events without side effects.
- **Kafka independence** (FR-EVT-09) — Core transactional writes keep working even if Kafka is temporarily unavailable.

### 4.11 Redis-Backed State
- **Idempotency-key storage** (FR-RED-03) — Store idempotency-key state in Redis as `idempotency:{key}`.
- **Redis-outage fallback** (FR-RED-04) — Non-cache-dependent writes keep functioning if Redis is unavailable.

### 4.12 Idempotency & Concurrency Control
- **Idempotency-Key support** (FR-IDC-01) — State-changing endpoints accept an `Idempotency-Key` header and return the original result on retry.
- **Optimistic-lock conflict handling** (FR-IDC-02) — Stale writes on the case entity return HTTP 409.

### 4.13 Search (Baseline)
- **PostgreSQL-backed search** (FR-SRCH-01) — Case search/filter (ID, title, status) works directly against PostgreSQL initially.
- **Search endpoint** (FR-SRCH-02) — Expose `GET /search/cases`, scoped to the caller's tenant.

### 4.14 REST API, Contracts & Documentation
- **REST interface** (FR-API-01) — Expose core functionality through REST endpoints.
- **API versioning** (FR-API-02) — Version the API explicitly (`/api/v1/...`).
- **Request validation** (FR-API-03) — Validate incoming payloads; reject malformed requests.
- **Consistent error responses** (FR-API-04) — Return structured, consistent error bodies for invalid/unauthorized requests.
- **OpenAPI/Swagger documentation** (FR-API-05) — Document the full public API surface.
- **Upload guardrails** (FR-API-06) — Enforce maximum file-size limits and block unsafe file types on document upload.

### 4.15 Observability (Baseline)
- **Correlation IDs** (FR-OBS-01) — Attach `request_id`/`trace_id` to every request and propagate through logs, events, and downstream calls.
- **Structured logging** (FR-OBS-02) — JSON logs include `user_id, tenant_id, endpoint, latency, status`.
- **Core metrics** (FR-OBS-03) — Expose Prometheus-compatible metrics (req/sec, P50/P95/P99 latency, error rate).

### 4.16 Testing, CI/CD & Documentation
- **Unit tests** (FR-QA-01) — Cover domain logic: state machine, authorization decisions, rule evaluation, permissions.
- **Integration tests** (FR-QA-02) — Run via Testcontainers against real PostgreSQL, Redis, and Kafka.
- **OpenAPI coverage verification** (FR-QA-03) — Confirm the spec covers every public endpoint.

### 4.17 Cross-Cutting Non-Functional Requirements (Must)
- **Secure secrets handling** (NFR-SEC-01) — Credentials and inter-service secrets are hashed at rest and sent over TLS/HTTPS.
- **Data-layer tenant isolation** (NFR-SEC-02) — Tenant isolation is enforced at the data-access layer, not only the API layer.
- **Auth + permission gate on writes** (NFR-SEC-03) — Every state-changing endpoint requires authentication and permission-based authorization.
- **Input validation** (NFR-SEC-04) — External input is validated before processing; malformed payloads are rejected.
- **Transaction boundaries** (NFR-DATA-01) — Multi-step DB changes use correct transaction scoping.
- **Referential integrity** (NFR-DATA-02) — Database relationships use appropriate constraints.
- **Graceful degradation** (NFR-REL-01) — Core writes don't depend on Kafka, Redis, or search being available.
- **At-least-once delivery** (NFR-REL-02) — Events are delivered at-least-once with idempotent consumers; nothing is silently lost.
- **DB as source of truth** (NFR-REL-03) — PostgreSQL is authoritative; search/cache are eventually-consistent projections.
- **Safe failure & recoverability** (NFR-REL-04) — Mid-operation failures fail safely; backups preserve committed data with zero loss.
- **Deliberate concurrency handling** (NFR-CONC-01) — Concurrent updates are handled deliberately; no silent lost updates.

> **Phase 2 exit criteria:** the full SRS §10 scenario passes — authenticate → operate in org context → create case → assign → complete tasks → transition through workflow states → request/decide approval → resolve → see it in the audit trail — and unauthorized access, cross-tenant access, and invalid transitions are all provably rejected.

---

## 5. Phase 3 — Should Have (S): Reliability & Extended Capabilities

**Objective:** Meaningfully strengthens the platform — reliability, tenant self-service, and richer collaboration — but the system is coherent and demoable without these. Build once Phase 2 is stable.

### 5.1 Identity & Auth
- **Refresh tokens** (FR-IAM-03) — `POST /auth/refresh` issues a new access token without re-entering credentials.
- **Password reset/change** (FR-IAM-07) — Self-service password reset/change workflow.
- **Redis-backed session revocation** (FR-IAM-08) — Immediate token/session invalidation via Redis.

### 5.2 Authorization & RBAC
- **Custom tenant roles** (FR-AUZ-05) — Organization-scoped, admin-defined custom roles.

### 5.3 Multi-Tenancy
- **Cross-tenant test suite** (FR-TEN-05) — Automated tests that specifically assert cross-tenant access is impossible.

### 5.4 Advanced Case & Task Features
- **Close/reopen with reason** (FR-CASE-10) — Formal close/reopen flow with an auditable reason.
- **Human-readable timeline** (FR-CASE-11, cross-ref FR-AUD-05) — A readable, chronological case history/timeline endpoint.
- **Case-endpoint idempotency** (FR-CASE-12) — State-changing case endpoints accept idempotency keys.
- **Bulk operations** (FR-CASE-13) — Bulk reassignment, status updates, and export for cases/tasks.
- **Soft delete/archive** (FR-CASE-14) — Deleting a case archives it rather than erasing the historical record.
- **Task dependencies** (FR-TASK-05) — Explicit precedence (Task A before Task B).
- **Task-gated workflow advancement** (FR-TASK-06) — Block progression while required tasks for the current stage are incomplete.

### 5.5 Approval System
- **Approval expiry** (FR-APR-05) — Approvals past `expires_at` without a decision become non-actionable and must be re-requested.
- **Dynamic approver routing** (FR-APR-06) — Required approver role/level is determined dynamically from workflow rules (e.g., amount-based routing).

### 5.6 Document Management
- **Checksum verification** (FR-DOC-05) — Detect corrupted/tampered uploads via checksum comparison.
- **Asynchronous virus scanning** (FR-DOC-06, BR-14) — Route uploads through malware scanning before they're downloadable.
- **Document versioning** (FR-DOC-07) — Track document revisions via `document_versions`.

### 5.7 Collaboration
- **Threaded replies** (FR-COL-03) — Support threaded comment replies.
- **@Mentions** (FR-COL-04) — @mentions trigger a notification to the mentioned user.

### 5.8 Audit System
- **Human-readable activity timeline** (FR-AUD-05) — Render a per-case activity timeline from audit/domain events, kept logically distinct from the raw technical audit log.
- **Immutable audit API** (FR-AUD-06) — No update/delete API exists for audit records — immutability is enforced, not just documented.

### 5.9 Workflow Engine
- **Structured rule configuration** (FR-WF-03) — Represent rules as structured config/data, not hard-coded conditionals.
- **Standard rule actions** (FR-WF-04) — Support at minimum `REQUIRE_APPROVAL`, `CREATE_TASK`, `SEND_NOTIFICATION` actions.
- **Tenant-customizable workflow rules** (FR-WF-05) — Allow each tenant to configure its own rules rather than one hardcoded lifecycle for everyone.

### 5.10 Event-Driven Architecture & Outbox
- **Dead-letter queue** (FR-EVT-05) — Repeatedly failing events route to a DLQ instead of blocking the pipeline.
- **Event schema versioning** (FR-EVT-06) — Version event schemas for backward-compatible evolution.
- **Consumer groups** (FR-EVT-07) — Organize consumers into groups aligned to responsibility.

### 5.11 Redis-Backed State
- **Read-through caching** (FR-RED-01) — Cache frequently read, rarely changed data (permission lists, case summaries) with TTL and invalidation.
- **Redis-backed rate limiting** (FR-RED-02) — Enforce per-user/per-tenant request quotas.

### 5.12 Idempotency & Concurrency
- **Documented isolation levels** (FR-IDC-03) — Choose and document the DB isolation level per critical write path.
- **Concurrency test suite** (FR-IDC-05) — Automated tests simulate concurrent updates and assert correct conflict handling.

### 5.13 Search
- **Dedicated search indexing** (FR-SRCH-03) — Kafka-driven indexing of case/comment/document-metadata content into OpenSearch/Elasticsearch, once justified.

### 5.14 Notifications
- **Assignment notifications** (FR-NOT-01) — Notify on case/task assignment.
- **Approval & resolution notifications** (FR-NOT-02) — Notify when an approval is requested and when a case reaches a final state.

### 5.15 Observability
- **Health endpoints** (FR-OBS-04) — Expose `/health/liveness` and `/health/readiness`.
- **Infrastructure metrics** (FR-OBS-05) — Expose Kafka consumer lag, DB connection-pool usage, and Redis hit ratio.
- **Grafana dashboards** (FR-OBS-06) — Visualize the metrics above.
- **Distributed tracing** (FR-OBS-07) — OpenTelemetry tracing spans the API and async consumers.

### 5.16 Testing, CI/CD & Documentation
- **Failure-mode test suite** (FR-QA-07) — Documented tests for DB/Redis/Kafka down, consumer crash, duplicate event, concurrent update, and client retry scenarios.
- **Architecture documentation** (FR-QA-08) — Maintain architecture and key design-decision documentation.

### 5.17 Cross-Cutting Non-Functional Requirements (Should)
- **Cache performance target** (NFR-PERF-01) — Cached endpoints are materially faster than uncached equivalents; SLOs defined via load testing.
- **Consumer-lag monitoring** (NFR-PERF-02) — Kafka consumer lag is tracked and kept within acceptable bounds.
- **Indexing & pagination** (NFR-PERF-03) — Appropriate indexes plus pagination/sorting/filtering on large-collection APIs.
- **Stateless API layer** (NFR-SCAL-01) — The API scales horizontally behind a load balancer.
- **End-to-end correlation** (NFR-OBS-01) — Correlation IDs traceable across logs, metrics, and traces.
- **Consistent conventions** (NFR-MAINT-01) — SOLID principles and agreed module/coding conventions.
- **Security-relevant test coverage** (NFR-TEST-01) — Auth, authz, tenant-isolation, and workflow-transition behavior specifically tested.

> **Phase 3 exit criteria:** the platform tolerates partial infrastructure failure gracefully, tenants can self-configure roles and workflow rules, and operational visibility (dashboards, tracing, health checks) is in place.

---

## 6. Phase 4 — Could Have (C): Advanced Capabilities

**Objective:** Genuine value-adds with lower urgency. Pursue only once Phases 2 and 3 are complete and stable, and time/capacity remains. Nothing here blocks the Definition of Done.

### 6.1 Identity & Auth
- **Configurable token TTL** (FR-IAM-09) — Per-tenant or per-role access-token expiry configuration.
- **MFA / OTP** (FR-IAM-10) — Multi-factor authentication via Redis-backed one-time codes.

### 6.2 Authorization & RBAC
- **Field/resource-level authorization** (FR-AUZ-06) — Finer-grained checks below the endpoint level.
- **ABAC layer** (FR-AUZ-07) — Attribute-based access control layered on top of RBAC.

### 6.3 Multi-Tenancy
- **Row-Level Security evaluation** (FR-TEN-06) — Evaluate PostgreSQL RLS as a defense-in-depth layer.

### 6.4 Task Management
- **Recurring/templated tasks** (FR-TASK-07) — Workflow-rule-generated recurring or templated tasks.
- **SLA timers** (FR-TASK-08) — Task-level SLA timers with overdue notifications.

### 6.5 Approval System
- **Multi-step approval chains** (FR-APR-07) — Sequential multi-approver chains.
- **Approval delegation** (FR-APR-08) — Delegate approval authority to another user.

### 6.6 Document Management
- **OCR extraction** (FR-DOC-08) — Extract text from documents for downstream search indexing.
- **Pre-signed download URLs** (FR-DOC-09) — Short-lived, secure S3 pre-signed URLs.

### 6.7 Collaboration
- **Comment editing** (FR-COL-05) — Edit a comment with a preserved edit marker.
- **Comment attachments** (FR-COL-06) — Attach files directly to a comment.

### 6.8 Audit System
- **Audit export** (FR-AUD-07) — Export audit trails (CSV/PDF) for a case or date range.

### 6.9 Workflow Engine
- **Workflow runtime instances** (FR-WF-06) — Persist `workflow_instances` tracking runtime progress against a case.

### 6.10 Event-Driven Architecture
- **Per-case event ordering** (FR-EVT-08) — Preserve ordering by partitioning Kafka on `case_id`.

### 6.11 Redis-Backed State
- **Workflow state / temporary locks** (FR-RED-05) — Short-lived workflow state or locks in Redis.
- **Redis-backed OTP state** (FR-RED-06) — Store MFA one-time codes in Redis.

### 6.12 Idempotency & Concurrency
- **Pessimistic locking** (FR-IDC-04) — Apply pessimistic locks to specific high-contention operations, with justification documented.

### 6.13 Search
- **OCR-backed search** (FR-SRCH-04) — Index OCR-extracted document text for search.

### 6.14 Notifications
- **Notification preferences** (FR-NOT-03) — Per-user configurable notification preferences.

### 6.15 Observability
- **Threshold alerting** (FR-OBS-08) — Alert when error rate or consumer lag exceeds configured thresholds.

### 6.16 Testing, CI/CD & Documentation
- **Load/performance testing** (FR-QA-09) — Formal load/stress testing pass.
- **Security review / pen-test** (FR-QA-10) — Formal security review or penetration-test pass before public release.

> **Phase 4 exit criteria:** none — this phase is opportunistic. Ship whichever items fit before the deadline; none of them block the Definition of Done.

---

## 7. Phase 5 — Won't Have — This Release (Deferred Backlog)

**Objective:** Explicitly documented, not silently dropped. These are candidates for a *future* release, not oversights — each maps back to an explicit constraint in SRS §5 (scale targets, regulatory boundaries, or the "modular monolith first" architectural discipline).

### 7.1 Deferred Functional Requirements (W-tagged in the SRS)
- **No third-party SSO/OAuth** (FR-IAM-11) — External identity providers (Okta, Google, Azure AD) are not integrated this release.
- **No schema-per-tenant deployment** (FR-TEN-07) — Physical multi-database/schema isolation per customer is out of scope.
- **No visual state-machine designer** (FR-CASE-15) — A tenant-editable, drag-and-drop case-workflow designer will not ship.
- **No in-browser document editing** (FR-DOC-10) — Interactive online document editing/annotation is not built.
- **No blockchain audit ledger** (FR-AUD-08) — Tamper-proofing relies on PostgreSQL append-only storage, not a distributed ledger.
- **No visual workflow designer UI** (FR-WF-07) — Workflows stay defined in code/JSON configuration, not a drag-and-drop builder.
- **No faceted/analytics search UI** (FR-SRCH-05) — Advanced, faceted, analytics-style search is not implemented.

### 7.2 Broader Out-of-Scope Items (SRS §5.6)
- **No native mobile apps** — deliverable is an API backend + web client only.
- **No autonomous AI approvals** (BR-18) — AI may only recommend; a human decision is always required.
- **No billing/payments** — no payment gateway or invoicing functionality.
- **No native ERP/CRM connectors** — third-party systems integrate via standard outbound webhooks, not bespoke Salesforce/SAP connectors.
- **No proprietary/commercial BPM engine dependency** — the workflow engine remains custom-built and modular.
- **No formal SOC2/HIPAA certification** — general security practices apply, but formal audit sign-off is out of scope.
- **No large-scale microservice/multi-region deployment** — production-scale infrastructure orchestration is deferred.
- **No complex analytics platform** or advanced AI features beyond BR-18's advisory role.

---

## 8. Phase 6 — Final Integration, Stabilization & Submission

Phase 6 is not a new feature priority tier — it is the closing work that proves everything built in Phases 1–4 actually functions as one coherent system.

- **Full system integration** — Verify every implemented module and infrastructure piece works together, not just in isolation.
- **End-to-end business flow** — Walk the full path: `Login → Create Case → Assign → Create Tasks → Complete Tasks → Investigation → Review → Approval → Resolution → Audit/Activity`.
- **Security verification** — Re-test authentication, authorization, tenant isolation, invalid tokens, invalid permissions, cross-tenant access, input validation, and file-type/size restrictions.
- **Failure verification** — Exercise the failure modes relevant to whichever infrastructure was actually implemented (DB down, Redis down, Kafka down, consumer crash, duplicate event, concurrent update, client retry).
- **API contract verification** — Check the implementation against the OpenAPI contract: endpoints, requests, responses, errors, status codes, auth, authorization.
- **Database verification** — Confirm migrations, indexes, constraints, foreign keys, tenant fields, and seed/demo data are all correct.
- **Documentation finalization** — README, architecture diagram, ER diagram, API documentation, module documentation, setup instructions, testing instructions, ADRs, known limitations, and future scope.
- **Docker reproducibility check** — Validate the complete documented setup from a clean environment.
- **CI verification** — Final build/test/lint pipeline passes cleanly.
- **Demo dataset** — Prepare realistic seed data spanning multiple organizations, users, teams, roles, cases, tasks, approvals, comments, documents, and audit history.
- **Final presentation** — Cover problem statement, requirements, MoSCoW, domain model, architecture, database, API contracts, RBAC, tenant isolation, workflow, concurrency, reliability, testing, observability, failure scenarios, and future scope.

---

## 9. Phase Dependency Diagram

```text
PHASE 0
Requirements + Architecture + Contracts
              │
              ▼
PHASE 1
Foundational Platform / Bootstrap
              │
              ▼
PHASE 2 — MUST HAVE
Core Resolve
              │
              ├── Identity / RBAC
              ├── Tenancy
              ├── Cases
              ├── Tasks
              ├── Approvals
              ├── Comments
              ├── Documents
              ├── Audit
              ├── Workflow (baseline)
              ├── Events / Outbox
              ├── Idempotency / Concurrency
              ├── Search (baseline)
              ├── REST APIs
              └── Testing / Observability (baseline)
              │
              ▼
PHASE 3 — SHOULD HAVE
Reliability + Extended Capabilities
              │
              ├── Redis (caching, rate limiting)
              ├── Kafka reliability (DLQ, versioning, consumer groups)
              ├── Document processing (virus scan, versioning)
              ├── Notifications
              ├── Advanced workflow (tenant rules, structured config)
              ├── Dedicated search indexing
              └── Production observability (health checks, dashboards, tracing)
              │
              ▼
PHASE 4 — COULD HAVE
Advanced Capabilities
              │
              ├── MFA / advanced AuthZ (ABAC, field-level)
              ├── OCR
              ├── Advanced approvals (chains, delegation)
              ├── Advanced collaboration (edit, attachments)
              ├── Advanced search (OCR-backed)
              └── Performance / security extensions
              │
              ▼
PHASE 6
Integration + Stabilization + Submission

(PHASE 5 — Won't Have — runs alongside as a documented backlog, not a build step.)
```

---

## 10. Four-Developer Ownership Suggestion

Ownership should be finalized **after Phase 0 contracts are frozen** — this is a starting allocation, not a permanent silo. Integration, code review, testing, architecture, and contract changes remain whole-team responsibilities.

| Developer | Owns |
|---|---|
| **Developer 1 — Identity & Organization** | Authentication, Users, Organizations, Teams, Roles, Permissions, RBAC, Tenant context |
| **Developer 2 — Case Management** | Cases, Assignments, State machine, Case history, Concurrency, Search |
| **Developer 3 — Workflows & Work Execution** | Tasks, Task dependencies, Approvals, Workflow rules, Workflow execution |
| **Developer 4 — Platform & Supporting Systems** | Documents, Comments, Audit, Notifications, Async infrastructure, Observability |

---

## 11. Phase Completion Gates

### Phase 0
```text
✓ SRS frozen
✓ MoSCoW frozen
✓ Feature backlog created
✓ Domain model approved
✓ Schema approved
✓ Module boundaries approved
✓ API contracts approved
✓ Event contracts approved
✓ Git workflow established
✓ Development environment reproducible
```

### Phase 1
```text
✓ Modular-monolith skeleton stood up
✓ User can be provisioned, authenticated, and receives a tenant-scoped JWT
✓ Docker Compose environment runs end-to-end
✓ CI pipeline runs on every push/PR
```

### Phase 2
```text
✓ MUST features implemented
✓ Core end-to-end workflow works
✓ Authentication/authorization tested
✓ Tenant isolation tested
✓ API documented
✓ Core tests passing
✓ CI passing
```

### Phase 3
```text
✓ Selected SHOULD features implemented
✓ Reliability mechanisms tested
✓ Redis/Kafka behaviour documented
✓ Observability improved
✓ Integration tests passing
```

### Phase 4
```text
✓ Selected COULD features implemented
✓ No optional feature destabilizes core functionality
✓ Advanced features documented
```

### Phase 6
```text
✓ Final end-to-end demo passes
✓ CI passes
✓ Docker setup works
✓ Documentation complete
✓ Architecture matches actual implementation
✓ Known limitations documented
✓ Final presentation ready
```

---

## 12. Cross-Reference: Business Rules (BR-*) by Phase

Business rules are not phase-bound features in their own right — they are constraints the relevant phase's implementation must satisfy as it lands.

| Business Rule | Enforced starting in |
|---|---|
| BR-01, BR-02, BR-03, BR-04, BR-17 (state machine, permission checks, task/approval gating, assignment authorization) | Phase 2 (Case Management, Approval System) |
| BR-05, BR-06, BR-07, BR-16 (workflow rule examples, idempotent rule execution) | Phase 2 (Workflow Engine baseline) |
| BR-08 (approval expiry) | Phase 3 (FR-APR-05) |
| BR-09 (tenant isolation, no exceptions) | Phase 1 (foundation) — verified with automated tests in Phase 3 (FR-TEN-05) |
| BR-10 (optimistic-lock conflicts) | Phase 2 (FR-CASE-08, FR-IDC-02) |
| BR-11 (idempotent retries) | Phase 2 (FR-IDC-01) |
| BR-12, BR-13 (audit completeness & immutability) | Phase 2 (FR-AUD-01–04) — immutability enforced in Phase 3 (FR-AUD-06) |
| BR-14 (virus-scan gate on downloads) | Phase 3 (FR-DOC-06) |
| BR-15 (additive role permissions) | Phase 1/2 (FR-AUZ-04) |
| BR-18 (no autonomous AI approvals) | Permanent constraint — applies to Phase 4/5 if any AI-assisted recommendation feature is ever added |

---

## 13. Final Rule — Implementation Sequence

```text
SRS
 ↓
MoSCoW
 ↓
Feature List (this document)
 ↓
Phase Allocation
 ↓
Phase 0 Contracts
 ↓
Parallel Implementation (Phases 1–4)
 ↓
Integration & Testing
 ↓
Final Submission (Phase 6)
```

Before the four developers begin parallel implementation, the team must freeze: requirements, the domain model, the database schema, module boundaries, REST/OpenAPI contracts, event contracts, API conventions, and the Definition of Done.

The consolidated SRS (`Resolve_SRS_Consolidated.md`) remains the **single source of truth for requirements**; this document is the derived, bottlenecked implementation backlog and phase plan — if a feature isn't listed here with a phase, it does not get built yet.

---

*End of document. Consolidated from four independently drafted phase/feature plans. Every checkbox-worthy item traces back to a specific FR/NFR/BR ID in `Resolve_SRS_Consolidated.md`.*
