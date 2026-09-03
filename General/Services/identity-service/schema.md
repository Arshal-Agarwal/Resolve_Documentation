# `identity-service` — Schema & Description

**Postgres schema:** `identity`
**Owns:** `identity.users`, `identity.roles`, `identity.permissions`, `identity.user_roles`, `identity.role_permissions`
**SRS coverage:** FR-IAM-01…11 (Authentication & Session Management), FR-AUZ-01…07 (Authorization & RBAC)

## Purpose

Identity is the single place in the system that answers two questions: **"who is making this request?"** and **"what is that identity allowed to do?"**. It owns authentication (login, tokens, password handling) and authorization (roles, permissions, and their assignment). No other service stores credentials, roles, or permissions — they call into `identity-service` (in-process, via its module interface) to check access, exactly as required by FR-AUZ-02 ("centralized authorization evaluation... not scattered checks").

## Depends On

- `organization.organizations` — every `identity.users` row belongs to exactly one organization (FR-TEN-04).

## Depended On By

Every other service depends on `identity.users` for `created_by`, `assigned_to_user_id`, `actor_id`, and similar actor references. Those are **read-only foreign keys** — no other service ever writes to the `identity` schema directly.

---

## Tables

### `identity.users`
Authenticatable identities, always scoped to exactly one organization (FR-IAM-01, SRS §1.3).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | User identifier, referenced by every other service as an actor/assignee pointer. |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | A user belongs to exactly one tenant. |
| `email` | VARCHAR(255) | NOT NULL | Login identifier. |
| `username` | VARCHAR(100) | NULL | Optional separate login handle. |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt/argon2 hash — plaintext is never stored (FR-IAM-05). |
| `full_name` | VARCHAR(255) | NULL | Display name. |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'`, CHECK IN (`ACTIVE`,`DISABLED`) | A `DISABLED` user fails authentication (FR-IAM-06). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| *(unique)* | — | UNIQUE(`organization_id`, `email`) | Email is unique within a tenant, not globally. |

### `identity.roles`
Named collections of permissions; system-defined or tenant-custom (FR-AUZ-01, FR-AUZ-05).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NULL | `NULL` = platform-wide system role (e.g., `ORG_ADMIN`); non-null = tenant-scoped custom role. |
| `name` | VARCHAR(100) | NOT NULL | E.g. `SENIOR_MANAGER`. |
| `description` | TEXT | NULL | — |
| `is_system_role` | BOOLEAN | NOT NULL, DEFAULT `false` | System roles cannot be edited/deleted by tenant admins. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

### `identity.permissions`
Atomic, platform-global permission codes (FR-AUZ-01).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `code` | VARCHAR(100) | UNIQUE, NOT NULL | E.g. `CASE_READ`, `CASE_APPROVE`. |
| `description` | TEXT | NULL | — |

**Seed catalog** (finalize exact set with the team):
```text
CASE_READ, CASE_CREATE, CASE_UPDATE, CASE_ASSIGN, CASE_APPROVE
TASK_READ, TASK_CREATE, TASK_UPDATE
DOCUMENT_READ, DOCUMENT_UPLOAD
COMMENT_READ, COMMENT_CREATE
AUDIT_READ
USER_MANAGE, TEAM_MANAGE, ROLE_MANAGE
```

### `identity.user_roles`
Many-to-many Users ↔ Roles (FR-AUZ-04).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `user_id` | UUID | FK → `identity.users.id`, PK (part 1) | — |
| `role_id` | UUID | FK → `identity.roles.id`, PK (part 2) | — |
| `assigned_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Grant timestamp — useful for audit. |

### `identity.role_permissions`
Many-to-many Roles ↔ Permissions (FR-AUZ-04).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `role_id` | UUID | FK → `identity.roles.id`, PK (part 1) | — |
| `permission_id` | UUID | FK → `identity.permissions.id`, PK (part 2) | — |

---

## Non-Relational State Owned by This Service

| Store | Key/Structure | Purpose |
|---|---|---|
| Redis | `session:revoked:{token_id}` → `"1"`, TTL = remaining token life | Immediate logout invalidation (FR-IAM-08). |
| Redis | `mfa:otp:{user_id}` → hashed 6-digit code, TTL ~5 min | MFA (FR-IAM-10, Could-have). |
| Redis | `cache:permissions:{user_id}` → JSON array of permission codes, short TTL | Avoids recomputing effective permissions per request (FR-RED-01, read by every service via the shared authorization check). |

## Business Rules Enforced Here

- **BR-15** — Effective permissions are the union of all permissions from all roles assigned to a user (directly; team-based inheritance reads `organization.team_members` and applies the same union).
- **BR-02** — Any centralized permission check other services call into (e.g., "does this user have `CASE_APPROVE`?") is answered exclusively by this service.

## API Surface (owned endpoints)

```text
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
POST   /api/v1/auth/password-reset
GET    /api/v1/roles
POST   /api/v1/roles
GET    /api/v1/permissions
POST   /api/v1/users/{id}/roles
```
