# Resolve — Organization Service: Exhaustive Database & Contract Schema

**Document type:** Service-level database/schema design  
**Service:** Organization Service  
**Status:** Proposed implementation specification, reconciled against Resolve SRS, high-level schema, tasks, and current Organization Service contract  
**Primary responsibility:** Tenant lifecycle and canonical organization identity  
**PostgreSQL ownership:** `organizations`  
**Redis:** None required for current scope  
**Kafka:** None in current contract  
**Security authority:** System Administrator for platform-level tenant administration  
**Downstream role:** Canonical tenant authority for every other service.

---

# 1. Purpose

The Organization Service owns the tenant boundary of Resolve.

An organization represents one isolated customer account.

Every tenant-scoped resource elsewhere in the platform ultimately contains:

```text
organization_id
```

The Organization Service is authoritative for:

- organization existence
- organization UUID
- organization name
- organization slug
- organization lifecycle state

It does not own:

- users
- teams
- roles
- permissions
- cases
- tasks
- documents
- authentication sessions

---

# 2. Database Ownership

Recommended database:

```text
resolve_organization
```

Owned table:

```text
organizations
```

Only Organization Service may directly connect to this database.

---

# 3. `organizations` Table

## 3.1 PostgreSQL schema

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    name VARCHAR(255) NOT NULL,

    slug VARCHAR(100) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_organizations_slug
        UNIQUE (slug),

    CONSTRAINT chk_organizations_status
        CHECK (status IN ('ACTIVE', 'SUSPENDED')),

    CONSTRAINT chk_organizations_name_not_blank
        CHECK (length(trim(name)) > 0),

    CONSTRAINT chk_organizations_slug_format
        CHECK (slug ~ '^[a-z0-9]+([a-z0-9-]*[a-z0-9])?$')
);
```

---

# 4. Column Contract

| Column | Type | Null | Constraint | Meaning |
|---|---|---:|---|---|
| `id` | UUID | No | PK | Canonical tenant ID |
| `name` | VARCHAR(255) | No | Non-empty | Display name |
| `slug` | VARCHAR(100) | No | Globally unique | Login/routing identifier |
| `status` | VARCHAR(20) | No | `ACTIVE/SUSPENDED` | Tenant lifecycle |
| `created_at` | TIMESTAMPTZ | No | Default now | Provisioning time |
| `updated_at` | TIMESTAMPTZ | No | Default now | Last change |

No `version` column is required under the current project decision.

---

# 5. Organization Identifier Rules

`id` is the canonical immutable identifier.

Rules:

1. Generated using UUID.
2. Never reused.
3. Never changed after creation.
4. Safe to expose in URLs.
5. Used as `organization_id` by other services.
6. Must not be accepted from an untrusted client as a way to bypass tenant context.

---

# 6. Organization Slug

## 6.1 Purpose

The slug provides a human-readable tenant identifier:

```text
acme-bank
```

It is used by login:

```json
{
  "email": "jane.doe@acmebank.com",
  "password": "...",
  "organizationSlug": "acme-bank"
}
```

## 6.2 Rules

Recommended:

```text
lowercase
ASCII
numbers
hyphen
1–100 characters
no leading hyphen
no trailing hyphen
no consecutive spaces
```

Example valid:

```text
acme
acme-bank
acme-bank-2026
```

Invalid:

```text
Acme Bank
-acme
acme-
acme_bank
```

The final regex must match the OpenAPI validation contract.

---

# 7. Organization Status

Canonical values:

```text
ACTIVE
SUSPENDED
```

## 7.1 State machine

```text
ACTIVE
  |
  v
SUSPENDED
  |
  v
ACTIVE
```

Suspension does not delete tenant data.

---

# 8. Meaning of Suspension

When an organization is `SUSPENDED`:

### Authentication

Must reject new login attempts.

### User & Team

Should reject new tenant provisioning/membership-changing operations.

### Case/Task/Document/etc.

Should reject normal write operations unless a future platform policy explicitly permits them.

### Read behavior

The current contracts do not define a universal suspended-tenant read policy. Recommended:

- System Administrator: may inspect.
- Tenant users: denied normal application access.
- Historical data: retained.

This policy should be frozen in the shared authorization contract.

---

# 9. Indexes

The unique slug constraint creates the primary lookup index.

Recommended:

```sql
CREATE INDEX idx_organizations_status
    ON organizations (status);

CREATE INDEX idx_organizations_created_at
    ON organizations (created_at DESC);
```

For administrator search:

```sql
CREATE INDEX idx_organizations_name
    ON organizations (lower(name));
```

---

# 10. Public API

## 10.1 Create organization

```text
POST /api/v1/organizations
```

Auth:

```text
System Administrator only
```

Required:

```text
Idempotency-Key
```

Request:

```json
{
  "name": "Acme Bank",
  "slug": "acme-bank"
}
```

Response:

```json
{
  "id": "...",
  "name": "Acme Bank",
  "slug": "acme-bank",
  "status": "ACTIVE",
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

# 11. Organization Creation Transaction

```text
BEGIN
  validate request
  normalize name/slug
  check slug format
  INSERT organizations
COMMIT
```

The unique database constraint remains authoritative for concurrent slug creation.

If two requests create:

```text
acme-bank
```

simultaneously:

```text
one → 201
one → 409 ORG_SLUG_TAKEN
```

---

# 12. Idempotency

Organization creation requires:

```text
Idempotency-Key
```

Recommended persistence:

Because the current Organization Service owns only `organizations`, the project needs an explicit decision for idempotency storage.

### Preferred option

Use Redis:

```text
resolve:organization:idempotency:{key}
```

Value:

```json
{
  "requestHash": "...",
  "status": 201,
  "responseBody": {
    "id": "...",
    "name": "...",
    "slug": "..."
  }
}
```

### Invariant

Same key + same request:

```text
return original response
```

Same key + different request:

```text
409 IDEMPOTENCY_KEY_REUSED
```

Without this Redis record, the stated `Idempotency-Key` contract cannot be made robust across retries.

---

# 13. Get Organization

```text
GET /api/v1/organizations/{id}
```

Allowed:

- System Administrator
- authenticated user whose JWT tenant matches `{id}`

Tenant comparison:

```text
JWT tenantId == requested organization id
```

Otherwise reject.

---

# 14. Update Organization Status

```text
PATCH /api/v1/organizations/{id}/status
```

Request:

```json
{
  "status": "SUSPENDED"
}
```

Only System Administrators may perform this operation.

---

# 15. Recommended Organization Profile Update

The existing contract identifies a gap:

```text
PATCH /api/v1/organizations/{id}
```

Recommended supported fields:

```text
name
slug
```

Do not allow:

```text
id
status
createdAt
```

through this endpoint.

Status has its dedicated lifecycle endpoint.

---

# 16. Slug Change Safety

Changing:

```text
acme-bank
→
acme-banking
```

affects:

- login
- bookmarks
- client configuration
- tenant URLs if used
- caches

Therefore:

1. Validate uniqueness.
2. Update atomically.
3. Invalidate relevant caches.
4. Existing JWTs should remain valid because JWT tenant identity uses UUID, not slug.
5. New login requests use the new slug.

---

# 17. Internal Organization Lookup

## 17.1 By ID

```text
GET /internal/v1/organizations/{id}
```

Response:

```json
{
  "id": "...",
  "name": "Acme Bank",
  "slug": "acme-bank",
  "status": "ACTIVE"
}
```

Purpose:

- validate tenant existence
- validate tenant lifecycle
- support cross-service provisioning

Auth:

```text
service-to-service identity
```

---

# 18. Internal Lookup by Slug

```text
GET /internal/v1/organizations/by-slug/{slug}
```

Purpose:

Authentication login.

This closes an architectural gap in the current contracts: slug belongs to Organization Service and should not be resolved by User & Team Service.

---

# 19. Canonical Service-to-Service Rule

Other services must not:

```text
connect directly to organization_db
```

They must call:

```text
Organization Service
```

for authoritative tenant state.

They may cache the result for performance, but cache must never become the source of truth.

---

# 20. Tenant Validation Contract

For creation of a tenant-owned resource:

```text
organization_id
       |
       v
Organization Service
       |
       +-- NOT FOUND → reject
       |
       +-- SUSPENDED → reject
       |
       +-- ACTIVE → continue
```

For authenticated requests:

```text
JWT tenantId
       |
       v
requested resource organization_id
       |
       +-- mismatch → reject
       |
       +-- match → continue
```

---

# 21. Error Catalog

| Code | HTTP | Meaning |
|---|---:|---|
| `ORG_VALIDATION_ERROR` | 400 | Invalid organization request |
| `ORG_INVALID_STATUS` | 400 | Invalid lifecycle status |
| `AUTH_MISSING_TOKEN` | 401 | Authentication required |
| `ORG_FORBIDDEN` | 403 | Caller lacks permission |
| `ORG_NOT_FOUND` | 404 | Organization does not exist |
| `ORG_SLUG_TAKEN` | 409 | Slug already exists |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Same key used for another request |
| `ORG_SUSPENDED` | 403 | Tenant is suspended |
| `SERVICE_UNAVAILABLE` | 503 | Dependency unavailable |

---

# 22. No Physical Cross-Database Foreign Keys

Other services logically reference:

```text
organization_id → organizations.id
```

but cannot have PostgreSQL FK constraints if each service owns a separate database.

Therefore:

```text
Organization DB
└── organizations.id

User DB
└── users.organization_id

Case DB
└── cases.organization_id

Task DB
└── tasks.organization_id
```

The Organization Service is the canonical authority.

---

# 23. Tenant Deletion

Hard deletion is **not supported**.

The current schema intentionally has no:

```text
DELETE /organizations/{id}
```

and no:

```text
is_archived
```

column.

Tenant removal is represented by:

```text
ACTIVE → SUSPENDED
```

This preserves:

- users
- cases
- tasks
- documents
- approvals
- audit history

and avoids catastrophic cascading deletion across service databases.

---

# 24. No Kafka Events in Current Release

The current event catalog contains no:

```text
OrganizationCreated
OrganizationSuspended
OrganizationReactivated
```

events.

Therefore this service remains synchronous.

### Important consequence

Suspension does not automatically propagate through Kafka.

The platform must therefore handle suspension using one of:

1. authentication checks Organization Service on login and optionally session verification;
2. short-lived cached organization state;
3. a future organization lifecycle event.

For the current capstone contract, option 1 is the safest baseline.

---

# 25. Cache Guidance

Redis is not required as a primary dependency.

If introduced later:

```text
resolve:org:{organizationId}
resolve:org:slug:{slug}
```

Cache values must include:

```json
{
  "id": "...",
  "slug": "acme-bank",
  "status": "ACTIVE"
}
```

Never cache indefinitely.

Suspension must invalidate the organization cache immediately.

---

# 26. Concurrency

The service does not currently require an optimistic `version` field.

Concurrent updates are protected by:

- database uniqueness constraints
- transactional writes
- explicit status transitions
- serialized conflict handling where necessary

If organization profile editing becomes a real contention point, add:

```text
version INTEGER
```

through a new migration and contract revision.

Do not add it speculatively.

---

# 27. Security Model

## System Administrator

Can:

```text
create organization
list organizations
get organization
update organization
suspend organization
reactivate organization
```

## Tenant User

Can:

```text
GET own organization
```

subject to tenant match.

Cannot:

```text
read another tenant
suspend tenant
change another tenant
create arbitrary tenant
```

---

# 28. API Tenant-Safety Rule

Never authorize this:

```text
GET /organizations/{id}
```

based solely on:

```text
"id exists"
```

Correct:

```text
if systemAdmin:
    allow
else if jwt.tenantId == id:
    allow
else:
    reject
```

---

# 29. Data Lifecycle

## Provisioning

```text
POST /organizations
        ↓
validate
        ↓
create ACTIVE tenant
        ↓
return organization UUID
        ↓
User/Team provisioning can begin
```

## Suspension

```text
ACTIVE
  ↓
PATCH /status
  ↓
SUSPENDED
  ↓
new access/provisioning denied
```

## Reactivation

```text
SUSPENDED
  ↓
PATCH /status
  ↓
ACTIVE
```

No historical data is deleted.

---

# 30. Complete Database Schema

```text
resolve_organization PostgreSQL
│
└── organizations
    ├── id UUID PK
    ├── name VARCHAR(255) NOT NULL
    ├── slug VARCHAR(100) UNIQUE NOT NULL
    ├── status VARCHAR(20) NOT NULL
    ├── created_at TIMESTAMPTZ NOT NULL
    └── updated_at TIMESTAMPTZ NOT NULL
```

Optional supporting Redis:

```text
resolve:organization:idempotency:{key}
resolve:org:{id}
resolve:org:slug:{slug}
```

Redis is supporting state only.

---

# 31. Edge-Case Matrix

| Scenario | Expected behavior |
|---|---|
| Duplicate slug | 409 |
| Slug invalid | 400 |
| Blank name | 400 |
| Unknown organization ID | 404 |
| Tenant user requests another org | 403/404 according to shared security policy |
| Non-admin creates organization | 403 |
| Non-admin suspends org | 403 |
| Suspend nonexistent org | 404 |
| Suspend already suspended org | Idempotent success or defined no-op |
| Reactivate active org | Idempotent success or defined no-op |
| Concurrent same slug creation | One succeeds, one gets 409 |
| Same idempotency key + same request | Original result |
| Same idempotency key + different request | 409 |
| Organization DB unavailable | 503 |
| Other service asks about suspended org | Return `SUSPENDED` |
| Slug changes | Existing UUID-based JWTs remain structurally valid |
| Hard-delete requested | Not supported |
| Tenant data cleanup requested | Requires separate retention/deletion project |

---

# 32. Implementation Checklist

- [ ] Create `resolve_organization` PostgreSQL database.
- [ ] Add `organizations` migration.
- [ ] Add slug uniqueness constraint.
- [ ] Add status check constraint.
- [ ] Add normalized slug validation.
- [ ] Implement organization provisioning.
- [ ] Implement get-by-ID.
- [ ] Implement status lifecycle.
- [ ] Implement list/search for System Administrators.
- [ ] Implement profile update.
- [ ] Implement internal lookup by ID.
- [ ] Implement internal lookup by slug.
- [ ] Secure internal endpoints.
- [ ] Implement idempotency storage.
- [ ] Add concurrent slug tests.
- [ ] Add cross-tenant authorization tests.
- [ ] Add suspension behavior tests.
- [ ] Add migration tests.
- [ ] Confirm no Kafka dependency is introduced without a new requirement.

---

# 33. Traceability

| Requirement/Decision | Covered by |
|---|---|
| FR-ORG-01 | Organization lifecycle and APIs |
| FR-ORG-04 | Canonical tenant identity consumed by User/Team |
| FR-TEN-01 | Organization is root tenant boundary |
| FR-TEN-02 | Tenant validation contract |
| FR-TEN-03 | Tenant-aware access checks |
| FR-TEN-04 | JWT tenant identity integration |
| NFR-SEC-02 | Cross-tenant isolation |
| NFR-DATA-03 | Migration-only schema management |
| NFR-SCAL-03 | Multi-tenant organization capacity |
| BR-09 | Absolute tenant isolation |
