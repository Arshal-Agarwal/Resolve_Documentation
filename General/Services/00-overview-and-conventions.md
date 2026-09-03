# Resolve — Schema Documentation Index & Service Decomposition

**Source of truth:** `Resolve_Database_Schema_Consolidated.md` (final schema) · `Resolve_SRS_Consolidated.md` · `Resolve_Feature_List_and_Phases_Consolidated.md`
**Purpose of this folder (`General/`):** this is the **documentation input**, not the codebase. Every `.md` file here fully specifies one bounded-context service — its owned tables, its dependencies on other services, and the FR/NFR/BR requirements it satisfies. Nothing here is implementation code; it's the contract the codebase gets generated from.

---

## 1. How the Monolith Was Split Into Services

Resolve remains a **modular monolith** (NFR-ARCH-01 — no premature microservice extraction). "Service" here means a **bounded context / module**, matching the domain package layout already named in the SRS (NFR-ARCH-02): `identity`, `organization`, `case_management`, `task`, `document`, `approval`, `workflow`, `audit`, `notification`, plus `collaboration` (comments) and `shared` (cross-cutting infrastructure) added for completeness.

**Database decision:** all services still run against **one PostgreSQL instance**, but each service owns its own **Postgres schema (namespace)** rather than dumping every table into `public`:

```sql
CREATE SCHEMA identity;
CREATE SCHEMA organization;
CREATE SCHEMA case_management;
CREATE SCHEMA task;
CREATE SCHEMA document;
CREATE SCHEMA approval;
CREATE SCHEMA collaboration;
CREATE SCHEMA audit;
CREATE SCHEMA workflow;
CREATE SCHEMA notification;
CREATE SCHEMA shared;
```

**Why this, and not 11 separate databases:** the SRS is explicit that the platform is one monolith until there's a demonstrated reason to split it (NFR-ARCH-01, NFR-MAINT-02). Separate databases would mean no cross-service foreign keys, no cross-service transactions, and distributed-transaction complexity the project doesn't need yet. A schema-per-module namespace gives genuine logical separation — clear ownership, `service.table` naming that matches the module in code, the ability to grant/revoke DB permissions per module later, and a straight path to physical extraction if that's ever justified — without paying the distributed-systems tax today.

**Cross-schema foreign keys are allowed and expected** (e.g., `case_management.cases.organization_id → organization.organizations.id`). Postgres supports FKs across schemas within the same database natively; each service's `.md` file documents exactly which of its columns point outward, and which other services point into it.

---

## 2. Service → Schema → Table Map

| # | Service (folder) | Postgres Schema | Owned Tables | SRS Module |
|---|---|---|---|---|
| 1 | `identity-service` | `identity` | `users`, `roles`, `permissions`, `user_roles`, `role_permissions` | Identity, Auth & RBAC (SRS §1, §2 of the requirements doc) |
| 2 | `organization-service` | `organization` | `organizations`, `teams`, `team_members` | Multi-Tenancy & Org Structure |
| 3 | `case-management-service` | `case_management` | `cases`, `case_assignments`, `case_status_history` | Case Management |
| 4 | `task-service` | `task` | `tasks`, `task_dependencies` | Task Management |
| 5 | `document-service` | `document` | `documents`, `document_versions` | Document Management |
| 6 | `approval-service` | `approval` | `approvals` | Approval System |
| 7 | `collaboration-service` | `collaboration` | `comments` | Collaboration |
| 8 | `audit-service` | `audit` | `audit_events` | Audit System |
| 9 | `workflow-service` | `workflow` | `workflow_definitions`, `workflow_instances`, `workflow_tasks` | Workflow Engine |
| 10 | `notification-service` | `notification` | `notifications`, `notification_preferences` | Notifications |
| 11 | `shared-infrastructure` | `shared` | `outbox_events` | Event-Driven Architecture (cross-cutting) |

Every table in the consolidated schema is accounted for exactly once — no table is owned by two services.

---

## 3. Migration / Build Order

Because cross-schema FKs create real dependencies, schemas (and the services that own them) must be created in this order:

```text
1. organization        (no dependencies)
2. identity             (users.organization_id → organization.organizations)
3. shared               (outbox_events.organization_id → organization.organizations; no dependents needed yet)
4. case_management      (depends on organization, identity)
5. task                 (depends on case_management, organization, identity)
6. collaboration        (depends on case_management, organization, identity)
7. document             (depends on case_management, collaboration, organization, identity)
8. approval             (depends on case_management, organization, identity)
9. audit                (depends on organization, identity — resource_id is polymorphic, no hard FK)
10. workflow            (depends on organization, case_management, task, approval, notification-triggering)
11. notification         (depends on organization, identity)
```

This is also the recommended order for scaffolding the codebase itself: get `organization` and `identity` fully working first (this is the Phase 1 foundation from the Feature List), then build outward.

---

## 4. Folder Arrangement — Documentation (this repo, today)

```text
./
└── General/
    ├── 00-overview-and-conventions.md      ← this file
    ├── identity-service/
    │   └── schema.md
    ├── organization-service/
    │   └── schema.md
    ├── case-management-service/
    │   └── schema.md
    ├── task-service/
    │   └── schema.md
    ├── document-service/
    │   └── schema.md
    ├── approval-service/
    │   └── schema.md
    ├── collaboration-service/
    │   └── schema.md
    ├── audit-service/
    │   └── schema.md
    ├── workflow-service/
    │   └── schema.md
    ├── notification-service/
    │   └── schema.md
    └── shared-infrastructure/
        └── schema.md
```

## 5. Folder Arrangement — Codebase (target, to be generated)

Once the code is scaffolded (see the Antigravity prompt at the repo root), `./` should end up looking like this, with `General/` staying put as living documentation and each service becoming a real Maven/Gradle module directory alongside it:

```text
./
├── General/                          ← unchanged, stays as documentation
├── identity-service/
│   ├── src/main/java/com/resolve/identity/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── entity/
│   │   ├── dto/
│   │   └── config/
│   ├── src/main/resources/
│   │   └── db/migration/             ← Flyway scripts, schema "identity"
│   ├── src/test/java/com/resolve/identity/
│   └── build.gradle (or pom.xml)
├── organization-service/             ← same internal shape
├── case-management-service/
├── task-service/
├── document-service/
├── approval-service/
├── collaboration-service/
├── audit-service/
├── workflow-service/
├── notification-service/
├── shared-infrastructure/
│   └── src/main/java/com/resolve/shared/   ← common DTOs, outbox publisher, base entities, exception types
├── docker-compose.yml                ← Postgres, Redis, Kafka, OpenSearch, Kafka UI
├── docs/
│   └── architecture/                 ← ADRs, non-schema architecture notes
├── openapi/                          ← per-service OpenAPI specs
└── README.md
```

Each service module is a normal Spring Boot module inside the single monolith build (multi-module Gradle/Maven project) — still one deployable application, per NFR-ARCH-01, just cleanly separated in code and in the database the same way.

---

*Read this file first, then every `*/schema.md` file in this folder, before generating anything.*
