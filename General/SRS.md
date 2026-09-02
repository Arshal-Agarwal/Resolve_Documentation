# Software Requirements Specification (SRS)
## Resolve — Enterprise Case & Workflow Platform

**Version:** 1.0 (Consolidated)
**Date:** August 2026
**Status:** Draft — merged from three team drafts into a single source of truth
**Project Type:** Backend-first, multi-tenant case & workflow management platform (end-semester backend engineering project, 4-developer team)

> This document reconciles three independently authored SRS drafts. Where drafts disagreed on wording, scope, or detail level, the more precise or more complete version was kept, and gaps in one draft were filled using detail from the other two. Nothing described here as "Should/Could" has been silently promoted to "Must" — priority calls from the original drafts are preserved.

---

## Document Structure

```text
SRS
 │
 ├── 0. Introduction (Purpose, Scope, Problem Statement, Definitions, Conventions, Project Context)
 ├── 1. Actors / Roles
 ├── 2. Functional Requirements
 ├── 3. Non-Functional Requirements
 ├── 4. Business Rules
 ├── 5. Constraints (Technical, Architectural, Delivery, Scale, Regulatory, Out-of-Scope)
 ├── 6. External Integrations
 ├── 7. Data Model Overview
 ├── 8. Initial API Surface
 ├── 9. Contracts & Traceability
 └── 10. Acceptance Criteria
```

---

## 0. Introduction

### 0.1 Purpose
This SRS defines the functional and non-functional requirements, actors, business rules, constraints, data model, API surface, and external integrations for **Resolve**, a backend-first, multi-tenant enterprise case and workflow management platform. It is intended for engineering, QA, and technical/academic review during design and implementation.

### 0.2 Scope
Resolve models work as a **Case**: a central entity that can be assigned, moved through a controlled workflow, carry tasks and documents, require approvals, support internal collaboration, and generate a complete audit trail — for multiple tenant organizations sharing one platform.

In scope for the core system:
- User, team, role, and permission management within an organization (tenant)
- Case creation, assignment, lifecycle/workflow control, and history
- Task management, including dependencies
- Approval requests and decisions
- Comments and internal collaboration
- Document metadata and (in the extended build) binary storage
- Audit trail and human-readable activity timelines
- Tenant isolation enforced at the data-access layer
- Authentication and permission-based authorization
- Documented REST APIs, versioned and validated
- PostgreSQL as the system of record

The initial implementation is a well-modularized **monolith**; components are extracted into independent services only when there is a clear, documented engineering justification — extraction is never the default.

The following are treated as **extensions**, layered on once the core is stable, and are not all mandatory for a minimum viable build: Redis (caching, rate limiting, idempotency), object storage (S3-compatible), Apache Kafka with the transactional outbox pattern, dedicated search (OpenSearch/Elasticsearch), OCR, advanced/tenant-customizable workflow rules, notifications, and outbound webhooks.

### 0.3 Problem Statement
Organizations often manage complex operational work using disconnected tools — spreadsheets, email, chat, and ad-hoc document repositories. This creates recurring problems: no centralized case information, unclear task ownership, manual and inconsistent approval processes, difficulty reconstructing case history, weak visibility into who did what and when, fragmented document management, and difficulty enforcing organizational permissions.

Resolve addresses this by representing operational work as a **Case** and connecting it to its tasks, documents, approvals, comments, workflow state, and activity history in one auditable system.

**Target use cases** are intentionally domain-agnostic: banking dispute management, insurance claims, legal matters, compliance investigations, customer escalations, procurement approvals, financial operations, and internal employee cases. Example flow — a bank employee files a suspicious-transaction case:

```text
Created → Assigned to Investigator → Investigation (Tasks, Documents, Comments)
        → Submitted for Review → Manager Approval → Resolved
```

### 0.4 Definitions & Abbreviations

| Term | Meaning |
|---|---|
| Tenant / Organization | An isolated customer account on the shared platform |
| Case | The central business entity representing a unit of work |
| RBAC | Role-Based Access Control |
| SLA | Service Level Agreement / target |
| Outbox Pattern | Persisting an event in the same DB transaction as the business change, for reliable publishing |
| Idempotency Key | Client-supplied token that lets the server recognize and de-duplicate a retried request |
| DLQ | Dead-Letter Queue — holds events/messages that exhausted retries so they don't block the system |

### 0.5 Requirement ID Convention
`FR-<MODULE>-<NN>` for functional requirements, `NFR-<CATEGORY>-<NN>` for non-functional requirements, `BR-<NN>` for business rules. IDs are stable across document revisions for traceability into test cases and the project's MoSCoW backlog (**M**ust / **S**hould / **C**ould / **W**on't-this-release).

### 0.6 Project Context
Resolve is being built by a 4-developer team against a **20-week phased roadmap**: Foundation → Core Domain → Production Engineering → Event Architecture → Documents → Search & Workflows → Productionization. Scope per phase is bounded accordingly; later-phase capabilities (Kafka, search, advanced workflow execution) are deliberately deferred rather than parallelized from day one. As an academic/team engineering exercise, the project prioritizes a **complete, coherent, testable, and explainable core system** over the sheer number of technologies used — every major piece of infrastructure introduced must have an identifiable problem it solves, documented as such.

---

## 1. Actors / Roles

### 1.1 Human Actors

| Actor | Description | Representative Permissions |
|---|---|---|
| **System Administrator** | Platform-level operator, not scoped to any single organization; manages tenants at the infrastructure level. | Provision/suspend organizations, view platform health |
| **Organization / Tenant Admin** | Owns a tenant; manages that tenant's users, teams, roles, and configuration, including custom workflow rules. | `USER_MANAGE`, `TEAM_MANAGE`, `ROLE_MANAGE`, all `CASE_*` |
| **Case Creator / Case Worker / Requester** | Any authenticated user who opens a case, works assigned tasks, uploads documents, and comments (e.g., a bank employee filing a suspicious-transaction case). | `CASE_CREATE`, `CASE_READ` (own/assigned), `DOCUMENT_UPLOAD`, `COMMENT_CREATE` |
| **Investigator / Assignee / Operational User** | User or team member responsible for progressing a case: completing tasks, uploading documents, commenting, performing permitted workflow transitions. | `CASE_READ`, `CASE_UPDATE`, `CASE_ASSIGN`, `DOCUMENT_UPLOAD`, `COMMENT_CREATE`, `TASK_*` |
| **Reviewer / Manager / Approver** | Reviews a case submitted for review and records an approval decision (e.g., financial limits, legal reviews, final resolutions). | `CASE_READ`, `CASE_APPROVE`, `COMMENT_CREATE` |
| **Team** | A named group of users that a case or task can be assigned to collectively. | Inherits effective permissions of its members' roles |
| **Auditor / Compliance Officer** | Read-only access to audit trails and complete case histories for compliance reporting. | `AUDIT_READ`, `CASE_READ` |
| **Guest / Unauthenticated User** | Any caller without a valid session. | None — all endpoints except `/auth/*` are denied |

### 1.2 System (Non-Human) Actors

| Actor | Role in the System |
|---|---|
| **Outbox Publisher** | Background process that relays persisted `outbox_events` rows to Kafka, retrying on failure with exponential backoff |
| **Virus Scanner Worker** | Kafka consumer (or ICAP-style sidecar, e.g. ClamAV) that scans newly uploaded documents |
| **OCR Worker** | Kafka consumer that extracts text from uploaded documents |
| **Metadata Processor** | Kafka consumer that enriches document metadata post-upload |
| **Search Indexer** | Kafka consumer that projects case/comment/document data into the search engine |
| **Notification Service** | Kafka consumer that delivers notifications (email / in-app) on relevant domain events |
| **Audit Consumer** | Kafka consumer that persists audit events derived from domain events |
| **Workflow Rule Engine** | Internal component that evaluates configured, tenant-customizable `IF/THEN` rules against case events |
| **Rate Limiter** | Redis-backed component enforcing per-user/per-tenant request quotas |
| **CI/CD Pipeline** | GitHub Actions automation that builds, tests, lints, and (eventually) deploys the system |
| **Outbound Webhook Dispatcher** | Optional component that pushes signed event payloads to external third-party endpoints |

### 1.3 Access & Role Rules
- A user belongs to exactly **one** Organization (Tenant) and cannot see, modify, or search data outside that tenant's boundary.
- Users are assigned one or more named Roles; every API endpoint checks whether the caller's effective (role-derived) permissions include the required atomic permission before allowing access.
- Role permissions are additive: a user's effective permissions are the union of the permissions of every role assigned to them, directly or via team membership.

---

## 2. Functional Requirements

Each requirement carries a stable ID and MoSCoW priority (**M**ust / **S**hould / **C**ould / **W**on't-this-release).

### 2.1 Identity, Authentication & Session Management

| ID | Requirement | Priority |
|---|---|---|
| FR-IAM-01 | The system shall allow a user to be registered/provisioned within an organization (tenant). | M |
| FR-IAM-02 | The system shall authenticate users via email/username and password, issuing a signed JWT access token on success; invalid login attempts shall be rejected without issuing a token. | M |
| FR-IAM-03 | The system shall issue a refresh token and expose `POST /auth/refresh` to obtain a new access token without re-entering credentials. | S |
| FR-IAM-04 | The system shall provide `POST /auth/logout`, invalidating the current session/refresh token. | M |
| FR-IAM-05 | The system shall hash all stored passwords (bcrypt/argon2) and never persist plaintext credentials. | M |
| FR-IAM-06 | The system shall reject any request lacking a valid, non-expired, non-revoked token to any protected endpoint via authentication middleware. | M |
| FR-IAM-07 | The system shall support a password reset/change workflow. | S |
| FR-IAM-08 | The system shall support Redis-backed session/token revocation for immediate invalidation. | S |
| FR-IAM-09 | The system should support configurable access-token TTL per tenant or role. | C |
| FR-IAM-10 | The system could support multi-factor authentication (OTP) via Redis-backed short-lived codes. | C |
| FR-IAM-11 | The system will not implement third-party SSO/OAuth identity providers in the initial release. | W |

### 2.2 Authorization & RBAC

| ID | Requirement | Priority |
|---|---|---|
| FR-AUZ-01 | The system shall model Roles as named collections of discrete, atomic Permissions (e.g., `CASE_READ`, `CASE_APPROVE`). | M |
| FR-AUZ-02 | The system shall evaluate authorization centrally (policy layer / method security) rather than via ad hoc checks scattered in controllers. | M |
| FR-AUZ-03 | The system shall deny any action for which the caller's effective permissions do not include the required permission, using permission checks rather than relying solely on hard-coded role checks. | M |
| FR-AUZ-04 | The system shall support many-to-many assignment: Users↔Roles and Roles↔Permissions. | M |
| FR-AUZ-05 | The system shall support organization-scoped custom roles, so appropriate users can assign roles to other users. | S |
| FR-AUZ-06 | The system should support field- or resource-level authorization. | C |
| FR-AUZ-07 | The system could support attribute-based access control (ABAC) layered on RBAC. | C |

### 2.3 Multi-Tenancy & Tenant Isolation

| ID | Requirement | Priority |
|---|---|---|
| FR-TEN-01 | Every tenant-owned entity (Case, Task, Document, Comment, Approval, Activity) shall carry an `organization_id`/`tenant_id`. | M |
| FR-TEN-02 | Tenant scoping shall be enforced at the repository/query layer on every read and write, not only at the API boundary — data-access queries filter by `tenant_id` by default. | M |
| FR-TEN-03 | Requests referencing a resource belonging to a different tenant than the caller's shall be rejected or return 404. | M |
| FR-TEN-04 | `tenant_id` shall be present in JWT claims and validated on every authenticated request, so requests are evaluated in the context of the caller's organization. | M |
| FR-TEN-05 | Automated tests shall specifically assert cross-tenant access is impossible. | S |
| FR-TEN-06 | The system could evaluate PostgreSQL Row-Level Security as defense-in-depth. | C |
| FR-TEN-07 | Schema-per-tenant deployment will not be supported in the initial release. | W |

### 2.4 Organizations, Users & Teams

| ID | Requirement | Priority |
|---|---|---|
| FR-ORG-01 | The system shall support creation and onboarding of organizations (tenants), each operating inside its own isolated data boundary. | M |
| FR-ORG-02 | Authorized users shall be able to create and manage organization users. | M |
| FR-ORG-03 | Authorized users shall be able to create and manage teams within a tenant; cases and tasks may be assigned to either individual users or entire teams. | M |
| FR-ORG-04 | The system shall associate every user with an organization and evaluate authenticated requests in that organization's context. | M |

### 2.5 Case Management

| ID | Requirement | Priority |
|---|---|---|
| FR-CASE-01 | The system shall allow creating a case with title, description, priority, and initial status `OPEN`, automatically associated with the creator's tenant. | M |
| FR-CASE-02 | The system shall support retrieving a single case and a paginated, filterable, sortable, tenant-scoped case list. | M |
| FR-CASE-03 | The system shall allow updating case fields subject to permission checks. | M |
| FR-CASE-04 | The system shall support assignment and reassignment of a case to a team and/or user, performable only by an appropriately authorized user. | M |
| FR-CASE-05 | The system shall enforce a validated state machine: `OPEN → ASSIGNED → INVESTIGATION → REVIEW → {APPROVED, REJECTED} → {RESOLVED, REOPENED}`; any transition outside this exact machine is blocked. | M |
| FR-CASE-06 | A transition shall be rejected if current state, permissions, required tasks, or required approvals are not satisfied. | M |
| FR-CASE-07 | Every status change shall be persisted to `case_status_history`. | M |
| FR-CASE-08 | The case entity shall use optimistic locking (a version column) to reject conflicting concurrent updates. | M |
| FR-CASE-09 | Multi-step case mutations shall execute inside a single database transaction. | M |
| FR-CASE-10 | The system shall support closing/reopening a case with an auditable reason, where applicable business conditions are satisfied. | S |
| FR-CASE-11 | The system shall expose a human-readable case history/timeline endpoint. | S |
| FR-CASE-12 | State-changing case endpoints should support idempotency keys. | S |
| FR-CASE-13 | The system should allow bulk case/task operations (bulk reassignment, bulk status updates, bulk export). | S |
| FR-CASE-14 | Deleting a case should archive it (soft-delete) rather than permanently erasing it, preserving the historical record. | S |
| FR-CASE-15 | A tenant-editable visual state-machine designer will not be supported initially. | W |

### 2.6 Task Management

| ID | Requirement | Priority |
|---|---|---|
| FR-TASK-01 | The system shall allow creating a task under a case with title, description, assignee, priority, due date. | M |
| FR-TASK-02 | The system shall support assignment/reassignment of a task to users or teams. | M |
| FR-TASK-03 | The system shall support updating and completing a task, recording `completed_at`. | M |
| FR-TASK-04 | The system shall list tasks for a case, filterable by status, assignee, and due date. | M |
| FR-TASK-05 | The system shall support task dependencies (Task A precedes Task B). | S |
| FR-TASK-06 | The system shall block workflow advancement while tasks required for the current stage are incomplete. | S |
| FR-TASK-07 | The system could support recurring/templated tasks generated by workflow rules. | C |
| FR-TASK-08 | The system could support task-level SLA timers and overdue notifications. | C |

### 2.7 Approval System

| ID | Requirement | Priority |
|---|---|---|
| FR-APR-01 | The system shall allow submitting an approval request capturing `requested_by`, `approver`, `reason`. | M |
| FR-APR-02 | The designated approver shall be able to approve or reject a pending request, recording `decision_at`. | M |
| FR-APR-03 | A case transition requiring approval shall be blocked until an `APPROVED` decision exists for that requirement. | M |
| FR-APR-04 | Full approval lifecycle (`created_at`, `expires_at`, `decision_at`, `status`) shall be recorded. | M |
| FR-APR-05 | The system shall support approval expiry (`expires_at`); an approval past its expiry without a decision is no longer actionable and must be re-requested. | S |
| FR-APR-06 | Required approver role/level shall be determined dynamically from workflow rules (e.g., amount-based routing to a senior approver). | S |
| FR-APR-07 | The system could support multi-step/multi-approver chains. | C |
| FR-APR-08 | The system could support delegation of approval authority. | C |

### 2.8 Document Management

| ID | Requirement | Priority |
|---|---|---|
| FR-DOC-01 | The system shall support uploading a file for a case, storing metadata (filename, MIME type, size, checksum, storage key) in PostgreSQL. | M |
| FR-DOC-02 | File bytes shall be stored in S3-compatible object storage, never in the relational database, once the extended implementation is in place. | M |
| FR-DOC-03 | The system shall support listing and downloading a case's documents, subject to authentication, authorization, and tenant-isolation rules. | M |
| FR-DOC-04 | A `document.uploaded` domain event shall be published on successful upload. | M |
| FR-DOC-05 | Checksums shall be verified to detect corrupted/tampered uploads. | S |
| FR-DOC-06 | Uploaded documents shall be routed to asynchronous virus scanning before being available for general download. | S |
| FR-DOC-07 | The system shall support document versioning via `document_versions`. | S |
| FR-DOC-08 | The system could support OCR extraction for downstream search indexing. | C |
| FR-DOC-09 | The system could support short-lived, pre-signed download URLs. | C |
| FR-DOC-10 | In-browser document editing/annotation will not be supported initially. | W |

### 2.9 Collaboration (Comments)

| ID | Requirement | Priority |
|---|---|---|
| FR-COL-01 | The system shall allow adding a comment to a case, remaining associated with that case. | M |
| FR-COL-02 | The system shall list comments chronologically, scoped by permission. | M |
| FR-COL-03 | The system shall support threaded replies. | S |
| FR-COL-04 | The system shall support @mentions that trigger a notification. | S |
| FR-COL-05 | The system shall support editing a comment with a preserved edit marker. | C |
| FR-COL-06 | The system could support attachments directly on a comment. | C |

### 2.10 Audit System & Activity Timeline

| ID | Requirement | Priority |
|---|---|---|
| FR-AUD-01 | An append-only `audit_events` record shall be written for every significant state-changing action. | M |
| FR-AUD-02 | Each record shall capture `actor_id`, `tenant_id`, `action`, `resource_type`, `resource_id`, `old_value`, `new_value`, `timestamp`. | M |
| FR-AUD-03 | Records shall include `request_id` and `trace_id` for correlation with logs/traces, where available. | M |
| FR-AUD-04 | A queryable `GET /audit` endpoint shall exist, restricted to authorized roles. | M |
| FR-AUD-05 | A human-readable, chronological per-case activity timeline shall be rendered from audit/domain events, kept logically distinct from the raw technical audit log. | S |
| FR-AUD-06 | Audit records shall be immutable after creation (no update/delete API) — no user, including a System Administrator, may edit or delete an existing entry. | S |
| FR-AUD-07 | The system could support exporting audit trails (CSV/PDF) for a case or date range. | C |
| FR-AUD-08 | A tamper-proof (e.g., blockchain-backed) audit ledger will not be implemented initially — PostgreSQL append-only storage is sufficient. | W |

### 2.11 Workflow Engine

| ID | Requirement | Priority |
|---|---|---|
| FR-WF-01 | The system shall support declarative rules of the form `IF <condition> THEN <action>`. | M |
| FR-WF-02 | Applicable rules shall be evaluated on relevant case events and their actions executed. | M |
| FR-WF-03 | Rules shall be represented as structured configuration/data, not hard-coded conditionals. | S |
| FR-WF-04 | At minimum `REQUIRE_APPROVAL`, `CREATE_TASK`, `SEND_NOTIFICATION` actions shall be supported. | S |
| FR-WF-05 | The system should support custom workflow rules per tenant, rather than forcing a single hardcoded lifecycle across every organization. | S |
| FR-WF-06 | The system could persist `workflow_instances` tracking runtime progress against a case. | C |
| FR-WF-07 | A visual, drag-and-drop workflow designer UI will not ship initially — workflows are configured via developer config files/JSON. | W |

### 2.12 Event-Driven Architecture & Transactional Outbox

| ID | Requirement | Priority |
|---|---|---|
| FR-EVT-01 | Domain events shall be defined/published for key state changes (`CaseCreated`, `CaseAssigned`, `CaseStatusChanged`, `CaseResolved`, `TaskCreated`, `TaskCompleted`, `DocumentUploaded`, `ApprovalRequested`, `ApprovalCompleted`). | M |
| FR-EVT-02 | The business-entity update and the outbox event row shall be written in the same DB transaction. | M |
| FR-EVT-03 | An outbox publisher shall reliably relay persisted events to Kafka, retrying on failure with exponential backoff. | M |
| FR-EVT-04 | Kafka consumers shall be idempotent against redelivered/duplicate events (tracking processing IDs so events are handled exactly once at the app level). | M |
| FR-EVT-05 | Repeatedly failing events shall route to a dead-letter queue (DLQ) so they don't block downstream processing. | S |
| FR-EVT-06 | Event schemas shall be versioned for backward-compatible evolution. | S |
| FR-EVT-07 | Consumers shall be organized into consumer groups aligned to responsibility. | S |
| FR-EVT-08 | The system could preserve per-case event ordering (e.g., partition by `case_id`). | C |
| FR-EVT-09 | Core transactional operations shall remain functional if Kafka is temporarily unavailable. | M |

### 2.13 Caching, Rate Limiting & Redis-Backed State

| ID | Requirement | Priority |
|---|---|---|
| FR-RED-01 | Frequently read, rarely changed data (permission lists, case summary views) shall be cached in Redis with TTL and invalidation. | S |
| FR-RED-02 | Per-user and per-tenant API rate limiting shall be enforced via a Redis-backed algorithm, so a single noisy tenant cannot degrade performance for others. | S |
| FR-RED-03 | Idempotency-key state shall be stored in Redis as `idempotency:{key}`. | M |
| FR-RED-04 | Non-cache-dependent write operations shall continue functioning if Redis is unavailable (graceful fallback to reading directly from PostgreSQL). | M |
| FR-RED-05 | The system could use Redis for short-lived workflow state/temporary locks. | C |
| FR-RED-06 | The system could use Redis-backed OTPs for MFA. | C |

### 2.14 Idempotency & Concurrency Control

| ID | Requirement | Priority |
|---|---|---|
| FR-IDC-01 | State-changing endpoints shall accept an `Idempotency-Key` header and return the original result on retry (e.g., a case must not be created twice, an approval must not be double-recorded). | M |
| FR-IDC-02 | Optimistic locking on the case entity shall return HTTP 409 on stale-write attempts, forcing the client to re-fetch and retry. | M |
| FR-IDC-03 | The appropriate DB isolation level shall be selected and documented per critical write path. | S |
| FR-IDC-04 | Pessimistic locking could be applied to specific high-contention operations, with justification documented. | C |
| FR-IDC-05 | Automated tests shall simulate concurrent updates and assert correct conflict handling. | S |

### 2.15 Search

| ID | Requirement | Priority |
|---|---|---|
| FR-SRCH-01 | Case search/filter (ID, title, status, description, comments) shall work directly against PostgreSQL initially. | M |
| FR-SRCH-02 | `GET /search/cases` shall be exposed, scoped to the caller's tenant. | M |
| FR-SRCH-03 | Case/comment/document-metadata content shall be indexed into OpenSearch/Elasticsearch via a Kafka-driven indexer once advanced search needs justify it — not synchronously from the request path. | S |
| FR-SRCH-04 | The system could index OCR-extracted document text. | C |
| FR-SRCH-05 | Full faceted/analytics-style search UI will not be implemented initially. | W |

### 2.16 Notifications

| ID | Requirement | Priority |
|---|---|---|
| FR-NOT-01 | The system shall send notifications (email or in-app) when a case or task is assigned to a user/team. | S |
| FR-NOT-02 | The system shall send a notification when an approval is requested, and when a case reaches a final state (`RESOLVED` or `REJECTED`). | S |
| FR-NOT-03 | The system could support per-user notification preferences (`notification_preferences`). | C |

### 2.17 REST API, Contracts & Documentation

| ID | Requirement | Priority |
|---|---|---|
| FR-API-01 | The system shall expose core functionality through REST APIs. | M |
| FR-API-02 | APIs shall use an explicit versioning strategy (e.g., `/api/v1/...`). | M |
| FR-API-03 | The API shall validate incoming request data and reject malformed/invalid payloads. | M |
| FR-API-04 | The API shall return structured, consistent error responses for invalid or unauthorized requests. | M |
| FR-API-05 | The REST API shall be documented using OpenAPI/Swagger, covering endpoint, method, request/response schema, auth, validation, and status/error codes. | M |
| FR-API-06 | The system shall enforce maximum file-size upload limits and block unsafe file types on document upload. | M |

### 2.18 Observability

| ID | Requirement | Priority |
|---|---|---|
| FR-OBS-01 | `request_id`/`trace_id` shall be attached to every request and propagated through logs, events, and downstream calls, including across async background events. | M |
| FR-OBS-02 | Structured (JSON) logs shall include `user_id`, `tenant_id`, `endpoint`, `latency`, `status`. | M |
| FR-OBS-03 | Prometheus-compatible metrics (req/sec, P50/P95/P99 latency, error rate) shall be exposed. | M |
| FR-OBS-04 | Health endpoints (`/health/liveness`, `/health/readiness`) shall be exposed. | S |
| FR-OBS-05 | Kafka consumer lag, DB connection-pool usage, Redis hit ratio shall be exposed as metrics. | S |
| FR-OBS-06 | Grafana dashboards shall visualize the above metrics. | S |
| FR-OBS-07 | Distributed tracing via OpenTelemetry shall span the API and async consumers. | S |
| FR-OBS-08 | The system could raise alerts when error rate/consumer lag exceed thresholds. | C |

### 2.19 Testing, CI/CD & Documentation

| ID | Requirement | Priority |
|---|---|---|
| FR-QA-01 | Unit tests shall cover domain logic (state machine, authorization decisions, rule evaluation, permissions). | M |
| FR-QA-02 | Integration tests shall use Testcontainers against real PostgreSQL, Redis, Kafka. | M |
| FR-QA-03 | An OpenAPI/Swagger spec shall be exposed for all public endpoints. | M |
| FR-QA-04 | CI (build, test, lint) shall run automatically on every pull request/push via GitHub Actions. | M |
| FR-QA-05 | Database schema changes shall use versioned migrations exclusively. | M |
| FR-QA-06 | The system shall be fully reproducible locally via Docker/Docker Compose. | M |
| FR-QA-07 | Documented failure-mode tests shall exist (DB down, Redis down, Kafka down, consumer crash, duplicate event, concurrent update, client retry). | S |
| FR-QA-08 | Architecture and key design-decision documentation shall be maintained. | S |
| FR-QA-09 | The system could add load/performance testing. | C |
| FR-QA-10 | The system could add a formal security review/pen-test pass before public release. | C |

---

## 3. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-ARCH-01 | Architecture | The initial architecture shall be a well-modularized monolith; components are extracted into independent services only with clear, documented engineering justification. |
| NFR-ARCH-02 | Architecture | Module boundaries shall follow domain-driven design: `identity`, `organization`, `user`, `case_management`, `task`, `document`, `approval`, `workflow`, `audit`, `notification`, `shared` — mirroring the domain, not technical layers alone. |
| NFR-ARCH-03 | Architecture | Business logic, API handling, persistence, and infrastructure concerns shall remain appropriately separated; internal module communication shall use in-process interfaces/DTOs rather than unnecessary HTTP calls. |
| NFR-API-01 | Compatibility | Public APIs shall be versioned and shall evolve in a backward-compatible manner once published. |
| NFR-SEC-01 | Security | All authentication credentials and inter-service secrets shall be stored and transmitted securely (hashed at rest, TLS/HTTPS in transit). |
| NFR-SEC-02 | Security | Tenant isolation is a security boundary and shall be enforced at the data-access layer, not only the API layer. |
| NFR-SEC-03 | Security | All state-changing endpoints shall require authentication and permission-based authorization. |
| NFR-SEC-04 | Security | External input shall be validated before being processed; malformed payloads are rejected. |
| NFR-DATA-01 | Data Integrity | Multi-step operations requiring related database changes shall use appropriate transaction boundaries. |
| NFR-DATA-02 | Data Integrity | Database relationships shall use appropriate referential-integrity constraints. |
| NFR-DATA-03 | Data Integrity | Schema changes shall be managed exclusively via version-controlled database migrations, never manual DDL. |
| NFR-PERF-01 | Performance | Cached read endpoints shall achieve materially lower P95 latency than their uncached equivalents; exact latency SLOs are defined and measured during load-testing. |
| NFR-PERF-02 | Performance | Kafka consumer lag shall be monitored and kept within an operationally acceptable bound under normal load. |
| NFR-PERF-03 | Performance | Frequently accessed data shall use appropriate database indexes and query patterns; large-collection APIs shall support pagination, sorting, and filtering. |
| NFR-REL-01 | Reliability | Core transactional business operations shall not depend on the availability of Kafka, Redis, or the search engine to complete (graceful degradation). |
| NFR-REL-02 | Reliability | Event delivery shall be at-least-once with idempotent consumers, so no event is silently lost, and consumers account for redelivered/duplicate events. |
| NFR-REL-03 | Reliability | The database shall be the single source of truth for business state; downstream systems (search, cache) are eventually-consistent projections. |
| NFR-REL-04 | Reliability | Mid-operation failures shall fail safely without corrupting data; database state shall be recoverable from backups with zero lost committed data. |
| NFR-CONC-01 | Concurrency | Concurrent updates to the same resource shall be handled deliberately (optimistic locking or an equivalent mechanism), avoiding unintended lost updates. |
| NFR-SCAL-01 | Scalability | The API layer shall be stateless so it can scale horizontally behind a load balancer. |
| NFR-SCAL-02 | Scalability | Redis and Kafka usage shall be introduced only for workloads that justify the added operational complexity. |
| NFR-SCAL-03 | Scalability | The architecture shall be built and sized for a baseline capacity of 5+ organizations, 100 concurrent active users, and 10,000 total cases; handling scale beyond these targets is explicitly deferred to future releases. |
| NFR-OBS-01 | Observability | Every request shall be traceable end-to-end via correlation identifiers across logs, metrics, and traces. |
| NFR-MAINT-01 | Maintainability | Code shall follow SOLID principles, consistent module boundaries, and agreed coding/naming conventions to keep the monolith navigable as it grows. |
| NFR-MAINT-02 | Maintainability | Every infrastructure component shall have a documented reason to exist; technology shall not be added purely for its own sake. |
| NFR-TEST-01 | Testability | Business logic shall be unit-testable in isolation from infrastructure (DB, Kafka, Redis) via clear boundaries/interfaces; authentication, authorization, tenant-isolation, and workflow-transition behavior shall be specifically tested. |
| NFR-PORT-01 | Portability | The entire stack shall run reproducibly via Docker/Docker Compose on a developer machine or CI runner; environment-specific configuration shall never be hard-coded. |
| NFR-USAB-01 | Usability (API) | All public endpoints shall be discoverable and self-describing via an OpenAPI/Swagger specification. |
| NFR-COMPAT-01 | Compatibility | The system shall run on the JVM (Java + Spring Boot) targeting a version compatible with the declared testing/CI toolchain. |

---

## 4. Business Rules

| ID | Rule |
|---|---|
| BR-01 | A case may only transition between states following the defined lifecycle: `OPEN → ASSIGNED → INVESTIGATION → REVIEW → {APPROVED → RESOLVED, REJECTED → REOPENED}`. Skipping states is not permitted, regardless of who requests it. |
| BR-02 | A case transition is only valid if the acting user holds the permission required for that transition (e.g., only a user with `CASE_APPROVE` may move a case from `REVIEW` to `APPROVED`). |
| BR-03 | A case cannot advance past a workflow stage while it has incomplete tasks marked as required for that stage. |
| BR-04 | A case cannot be marked `APPROVED`/`RESOLVED` unless every approval required by workflow rules for that case has a status of `APPROVED`; the status change remains blocked until an authorized approver records an explicit decision. |
| BR-05 | If a case's monetary amount exceeds a configured threshold, approval must be routed to a `SENIOR_MANAGER`-level approver (example rule: `IF case.amount > 10,00,000 THEN REQUIRE_APPROVAL("SENIOR_MANAGER")`). |
| BR-06 | If a case's priority is `HIGH`, a "Senior Investigation" task is automatically created upon assignment. |
| BR-07 | When a case reaches `RESOLVED`, a notification is automatically sent to the case's creator and current assignee. |
| BR-08 | An approval request that reaches its `expires_at` timestamp without a decision is no longer actionable and must be re-requested. |
| BR-09 | A user may only read or modify resources (cases, tasks, documents, comments, approvals) that belong to their own tenant (`organization_id` match), with no exceptions — data from one tenant must never be exposed to users of another tenant under any circumstance. |
| BR-10 | Two users attempting to update the same case concurrently: the second write is rejected (HTTP 409) if based on a stale version, forcing the client to re-fetch and retry. |
| BR-11 | A client retrying a state-changing request with the same `Idempotency-Key` must receive the original result, not a duplicate operation. |
| BR-12 | Every state-changing action on a case, task, document, approval, or comment must produce a corresponding audit event; an action without an audit trail is considered incomplete. |
| BR-13 | Audit events, once written, are never updated or deleted — including by a System Administrator. Corrections are made via new, subsequent audit events, not by mutating history. |
| BR-14 | A document is not available for general download until it has passed virus scanning (status other than "pending scan" / "infected"). |
| BR-15 | Role permissions are additive: a user's effective permissions are the union of the permissions of all roles assigned to them (directly or via team membership). |
| BR-16 | A workflow rule's action executes at most once per triggering event, even if the event is redelivered (idempotent rule execution). |
| BR-17 | Only appropriately authorized users may assign or reassign a case. |
| BR-18 | AI and automated algorithms may offer recommendations (e.g., suggested case routing or triage priority) but may never serve as an authoritative decider or automatically approve a case — a human approval decision is always required. |

---

## 5. Constraints

### 5.1 Technology Constraints
- Backend language and framework are fixed: **Java** with **Spring Boot**; security via **Spring Security**.
- Primary datastore is **PostgreSQL** accessed via **JPA/Hibernate**; schema changes must go through migrations, never manual DDL.
- Caching, rate limiting, and idempotency state must use **Redis**.
- Asynchronous messaging must use **Apache Kafka**, with the **transactional outbox pattern** as the only sanctioned way to bridge DB writes and event publishing.
- Binary file storage must use **S3-compatible object storage** (AWS S3 or a local-object-store equivalent); PostgreSQL must never store file bytes.
- Search, when introduced, must use **OpenSearch/Elasticsearch**, fed via Kafka — not a bolt-on full-text hack on the primary DB beyond the initial `LIKE`/`ILIKE`-level implementation.
- Observability stack is fixed: **Prometheus** (metrics), **Grafana** (dashboards), **OpenTelemetry** (tracing), structured logging.
- CI/CD must run on **GitHub Actions**.
- API contracts must be documented via **OpenAPI/Swagger**.
- Testing stack is fixed: **JUnit**, **Mockito**, **Testcontainers**.
- All components must be containerized via **Docker**.

### 5.2 Architectural Constraints
- The system must start as a **modular monolith**; extraction of any component into a separate service requires an explicit, documented engineering justification — it is not the default.
- Module boundaries must mirror the domain (`identity`, `case_management`, `task`, `document`, `approval`, `workflow`, `audit`, `notification`), not technical layers alone.
- New infrastructure (a cache, a queue, a search engine) may only be introduced when a specific, demonstrated need exists — not preemptively.
- Because the architecture is a modular monolith, internal module communication should use in-process interfaces/DTOs rather than unnecessary HTTP calls between modules (e.g., Case Module → Audit Interface → Audit Module).

### 5.3 Delivery / Project Constraints
- The system is developed by a 4-developer team against a **20-week phased roadmap** (Foundation → Core Domain → Production Engineering → Event Architecture → Documents → Search & Workflows → Productionization); scope for each phase is bounded accordingly, and later-phase capabilities (Kafka, search, workflow execution) are deferred rather than parallelized from day one.
- The team shall establish shared contracts (REST/OpenAPI, module interfaces, event schemas) before parallel implementation begins (see Section 9).
- The project's Definition of Done requires authentication, permission-based authorization, safe multi-tenancy, controlled state transitions, deliberate concurrency handling, working tasks/approvals/documents, justified Redis/Kafka usage, the outbox pattern, idempotent consumers, an audit trail, documented API, migrations, automated tests, containerized reproducibility, CI, metrics/logging, tested failure scenarios, and architecture documentation — a release is not considered complete without all of these.

### 5.4 Scale / Capacity Constraints
- The system is sized for a baseline capacity of **5+ organizations (tenants)**, **100 concurrent active users**, and **10,000 total cases**; scale beyond these targets is explicitly deferred to future releases.
- Individual tenants are rate-limited via Redis so a single noisy tenant cannot degrade platform performance for others.
- Target deployment is optimized for single-provider cloud hosting (AWS); multi-cloud packaging or on-premises installers are out of scope.

### 5.5 Regulatory / Compliance Constraints
- Because example use cases include banking, insurance, and compliance domains, the audit trail (Section 2.10) must be complete and immutable enough to support compliance review, even though the platform itself does not implement domain-specific regulatory logic (e.g., AML rules) — those remain the responsibility of the configured workflow rules.
- The application follows general security standards (encryption, access checks), but formal SOC2/HIPAA (or similar) audit sign-offs are not part of this release.

### 5.6 Out of Scope (Won't Have, This Release)
- Third-party SSO/OAuth identity providers.
- Schema-per-tenant deployment.
- A tenant-editable visual case-workflow designer, and a visual, drag-and-drop workflow-designer UI generally — workflows are configured via developer config files/JSON.
- Native mobile apps — the deliverable is an API backend and web client only.
- Autonomous AI approvals — all approval decisions require an authorized human (see BR-18).
- Billing/payments functionality (no payment gateway or invoicing).
- Blockchain-backed audit ledgers — append-only logging is handled via PostgreSQL.
- Native enterprise ERP/CRM connectors (Salesforce/SAP, etc.) — third-party systems integrate via standard outbound webhooks only.
- Proprietary/commercial BPM engine dependencies — the workflow engine is custom-built and modular.
- Full faceted/analytics-style search UI.
- In-browser document editing/annotation.
- Formal SOC2/HIPAA certification.
- Large-scale microservice deployment, multi-region architecture, and production-scale infrastructure orchestration.
- Complex analytics platform and advanced AI features beyond the advisory role in BR-18.

---

## 6. External Integrations

| Integration | Purpose | Direction | Notes |
|---|---|---|---|
| **S3-compatible object storage** | Persist uploaded document binaries | Outbound (write) / Inbound (read on download) | PostgreSQL stores only metadata + storage key (FR-DOC-01/02) |
| **Apache Kafka** | Domain event bus for asynchronous processing (audit, search indexing, notifications, document pipeline) | Outbound (publish via outbox) / Inbound (consume) | Publishing mediated exclusively by the transactional outbox pattern (FR-EVT-02/03) |
| **OpenSearch / Elasticsearch** | Full-text and structured search across cases, comments, and document metadata once introduced | Outbound (index) / Inbound (query) | Fed via a Kafka-driven indexer, not written synchronously from the request path (FR-SRCH-03) |
| **Redis** | Caching, rate limiting, idempotency-key storage, short-lived state, session revocation | Bidirectional | Treated as a non-critical dependency: its unavailability must not block core writes (FR-RED-04) |
| **Virus Scanning Service** (e.g., ClamAV / ICAP sidecar) | Scans uploaded documents for malware before they are downloadable | Inbound trigger via Kafka (`document.uploaded`) | External or containerized scanning engine; outcome updates document status (BR-14) |
| **OCR Engine** | Extracts text from uploaded documents for indexing | Inbound trigger via Kafka | Optional/Could-have integration (FR-DOC-08) |
| **Prometheus** | Scrapes application metrics | Inbound (pull) | Application exposes a `/metrics` endpoint |
| **Grafana** | Visualizes metrics scraped by Prometheus | Reads from Prometheus | Dashboards for latency, error rate, consumer lag, cache hit ratio |
| **OpenTelemetry Collector** | Collects distributed traces across the API and async consumers | Outbound (traces/spans) | Correlated via `trace_id` propagated per NFR-OBS-01 |
| **GitHub Actions** | CI/CD: build, test, lint, and (eventually) deploy on every push/PR | N/A (build-time) | Also runs Testcontainers-based integration tests |
| **Notification Delivery Channel** (email / push — specific provider TBD) | Delivers notifications triggered by domain events (mentions, approvals, resolutions) | Outbound | Provider intentionally left unspecified pending a later-phase decision |
| **Outbound Webhooks** (optional) | Pushes signed event payload notifications to external third-party endpoints when case events occur | Outbound | Standard integration point in place of native ERP/CRM connectors |

---

## 7. Data Model Overview

The initial database shall support at least the following core tables:

```text
organizations

users
teams
roles
permissions
user_roles
role_permissions

cases
case_assignments
case_status_history

tasks
task_dependencies

comments

documents

approvals

audit_events
```

Additional tables introduced as corresponding extended features are implemented:

```text
workflow_definitions
workflow_instances
workflow_tasks
outbox_events
notifications
notification_preferences
document_versions
```

---

## 8. Initial API Surface

```text
# Authentication
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout

# Organizations
POST /api/v1/organizations
GET  /api/v1/organizations/{id}

# Users
GET   /api/v1/users
POST  /api/v1/users
PATCH /api/v1/users/{id}

# Cases
POST  /api/v1/cases
GET   /api/v1/cases
GET   /api/v1/cases/{id}
PATCH /api/v1/cases/{id}
POST  /api/v1/cases/{id}/assign
POST  /api/v1/cases/{id}/transition
POST  /api/v1/cases/{id}/close
POST  /api/v1/cases/{id}/reopen

# Tasks
POST  /api/v1/cases/{id}/tasks
GET   /api/v1/cases/{id}/tasks
PATCH /api/v1/cases/{id}/tasks/{taskId}

# Documents
POST /api/v1/cases/{id}/documents
GET  /api/v1/cases/{id}/documents

# Comments
POST /api/v1/cases/{id}/comments
GET  /api/v1/cases/{id}/comments

# Approvals
POST /api/v1/cases/{id}/approvals
GET  /api/v1/cases/{id}/approvals

# Search
GET /api/v1/search/cases        # PostgreSQL-backed initially; dedicated engine may be introduced later

# Audit
GET /api/v1/audit                # access must be appropriately authorized
```

Example domain event payload (see FR-EVT-01, FR-EVT-06):

```json
{
  "eventType": "CASE_ASSIGNED",
  "tenantId": "uuid",
  "actorId": "uuid",
  "resourceId": "uuid",
  "timestamp": "2026-09-01T10:00:00Z",
  "metadata": {
    "previousAssignee": "uuid",
    "newAssignee": "uuid"
  }
}
```

---

## 9. Contracts & Traceability

### 9.1 REST Contracts
REST APIs shall be defined using OpenAPI. Each contract must define: endpoint, HTTP method, request schema, response schema, authentication, authorization, validation rules, status codes, and error responses. Contracts are established up front so the team can implement modules in parallel.

### 9.2 Module Contracts
Because the initial architecture is a modular monolith, internal module communication should use in-process interfaces and DTOs rather than unnecessary HTTP calls (e.g., `Case Module → Audit Interface → Audit Module`).

### 9.3 Event Contracts
Where asynchronous events are implemented, event payloads shall have explicitly defined schemas — recommended format is JSON plus JSON Schema — and shall be versioned for backward-compatible evolution (FR-EVT-06).

### 9.4 Traceability
Functional requirement IDs in Section 2 are consistent across the companion prioritization backlog, so items can be cross-referenced directly into test case IDs and sprint backlogs. Every feature should maintain traceability through the chain:

```text
SRS Requirement → MoSCoW Priority → Feature → Implementation → Test Case
```

Example:

```text
FR-CASE-05 (Valid Case State Transitions)
        ↓
Case State Machine
        ↓
CaseWorkflowService
        ↓
Unit + Integration Tests
```

---

## 10. Acceptance Criteria

The project is considered to have achieved its minimum scope when the following end-to-end scenario works:

```text
1. User authenticates
2. User operates within an organization
3. User creates a case
4. Case is assigned
5. Tasks are created
6. Tasks are completed
7. Case moves through valid workflow states
8. Approval is requested
9. Authorized reviewer approves/rejects
10. Case is resolved where appropriate
11. Important actions appear in activity/audit records
```

The team must also demonstrate:

```text
✓ Unauthorized operations are rejected
✓ Cross-tenant access is rejected
✓ Invalid workflow transitions are rejected
✓ Important business logic is tested
✓ The API is documented
✓ The database schema is reproducible
✓ The application can be run using the documented environment (Docker/Docker Compose)
```

**Final scope principle:** Resolve should ultimately demonstrate the team's ability to analyze a backend requirement, model the domain, define contracts, implement the system, secure it, test it, document it, and make informed architectural decisions — prioritizing a complete, coherent, testable, and explainable core system over the number of technologies used.

---

*End of document. This consolidated SRS supersedes the three individual drafts (`Claud_Resolve_SRS.md`, `Resolve_SRS_AD_.md`, `resolve_srs__1_.md`) as the team's single source of truth.*
