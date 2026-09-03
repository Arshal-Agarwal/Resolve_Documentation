# `organization-service` — Schema & Description

**Postgres schema:** `organization`
**Owns:** `organization.organizations`, `organization.teams`, `organization.team_members`
**SRS coverage:** FR-ORG-01…04, FR-TEN-01…07 (Multi-Tenancy & Tenant Isolation)

## Purpose

This is the root of tenancy. Every other service's tenant-owned tables carry an `organization_id` that ultimately points here — `organization-service` is the one place that defines what a tenant *is* and how it's structured internally (teams). It has no dependencies on any other service; it's the first thing that must exist and the first thing every other schema migration depends on.

## Depends On

Nothing. This is the root of the dependency graph.

## Depended On By

Every single other service — directly (`organization_id` FK) or, in `identity-service`'s case, indirectly through `users.organization_id`.

---

## Tables

### `organization.organizations`
Root tenant boundary (FR-ORG-01).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | Tenant identifier, referenced by every tenant-owned table across all services. |
| `name` | VARCHAR(255) | NOT NULL | Display name of the organization. |
| `slug` | VARCHAR(100) | UNIQUE, NOT NULL | URL-safe identifier (e.g., for tenant-aware routing). |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'`, CHECK IN (`ACTIVE`,`SUSPENDED`) | Platform-level lifecycle state, managed by System Administrators. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Provisioning timestamp. |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Last metadata change. |

### `organization.teams`
Named groups a case or task can be assigned to collectively (FR-ORG-03).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | Owning tenant. |
| `name` | VARCHAR(255) | NOT NULL | Team display name. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

### `organization.team_members`
Join table for team membership; drives permission inheritance (BR-15).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `team_id` | UUID | FK → `organization.teams.id`, PK (part 1) | — |
| `user_id` | UUID | FK → `identity.users.id`, PK (part 2) | **Cross-schema FK** — points into `identity`, the only outward reference this service has. |
| `joined_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

---

## Business Rules Enforced Here

- **FR-TEN-01 / FR-TEN-02** — This service defines the `organization_id` convention every other schema inherits; it is also where tenant provisioning and suspension happen.
- **FR-TEN-03** — Cross-tenant resource checks ultimately validate against this table (does `resource.organization_id` match the caller's `organization_id`?), but the check itself is enforced at each owning service's repository layer, not centrally re-queried here on every call.

## API Surface (owned endpoints)

```text
POST   /api/v1/organizations                 (System Admin only)
GET    /api/v1/organizations/{id}
PATCH  /api/v1/organizations/{id}/status
GET    /api/v1/teams
POST   /api/v1/teams
POST   /api/v1/teams/{id}/members
DELETE /api/v1/teams/{id}/members/{userId}
```

## Notes for Implementation

- `organization-service` and `identity-service` are circularly related in concept (a user needs an org; team membership needs a user) but **not circularly dependent at the schema level** — `organization.team_members.user_id` is the only place this schema reaches into `identity`, and it's a read-only reference (never written from this service's own business logic without an identity check).
- This must be the **first** schema migrated and the **first** service scaffolded (see `00-overview-and-conventions.md` §3).
