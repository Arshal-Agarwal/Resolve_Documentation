# Resolve — User & Team Service: Exhaustive Database & Contract Schema

**Document type:** Service-level database/schema design  
**Service:** User & Team Service  
**Status:** Proposed implementation specification, reconciled against Resolve SRS, high-level schema, tasks, and current service contracts  
**Primary responsibility:** Tenant-scoped user profiles, team management, team membership, internal user lookup, credential-operation façade  
**PostgreSQL ownership:** `users`, `teams`, `team_members`  
**Redis:** None required by this service for the current scope  
**Kafka:** None  
**Critical dependency:** Organization Service for tenant validation  
**Authentication dependency:** Authentication Service invokes internal credential endpoints; this service remains the physical table owner.

---

# 1. Purpose

The User & Team Service owns the human identity/profile and team domain inside a tenant.

It answers:

- Who is this user?
- Which organization does the user belong to?
- Is the user active?
- What is the user's profile?
- Which teams exist?
- Who belongs to each team?
- What user data may Authentication use to verify credentials?

It does **not** own:

- Organizations
- Roles
- Permissions
- Cases
- Tasks
- Sessions/tokens

---

# 2. Database Ownership

## 2.1 PostgreSQL database

Recommended logical database:

```text
resolve_user_team
```

Owned tables:

```text
users
teams
team_members
```

The service must be the **only service that directly connects to this database**.

---

# 3. Cross-Service Tenant Boundary

The original modular-monolith schema uses PostgreSQL foreign keys such as:

```text
users.organization_id → organizations.id
```

Once services have separate databases, a PostgreSQL FK cannot cross database boundaries.

Therefore the extracted-service implementation changes the physical constraint to:

```text
users.organization_id UUID NOT NULL
```

and validates the tenant through:

```text
User & Team Service
        |
        v
Organization Service
GET /internal/v1/organizations/{id}
        |
        v
ACTIVE?
```

### Important

`organization_id` remains a **logical foreign key** even though it is not a physical PostgreSQL FK.

This is mandatory for a real multi-database service architecture.

---

# 4. `users`

## 4.1 Purpose

Represents one authenticated human identity within exactly one organization.

## 4.2 PostgreSQL schema

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    organization_id UUID NOT NULL,

    email VARCHAR(255) NOT NULL,
    username VARCHAR(100) NOT NULL,

    password_hash VARCHAR(255),

    full_name VARCHAR(255) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_users_status
        CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED')),

    CONSTRAINT uq_users_org_email
        UNIQUE (organization_id, email),

    CONSTRAINT uq_users_org_username
        UNIQUE (organization_id, username)
);
```

## 4.3 Column contract

| Column | Type | Null | Constraint | Meaning |
|---|---|---:|---|---|
| `id` | UUID | No | PK | Stable public user identifier |
| `organization_id` | UUID | No | Logical FK | Tenant owner |
| `email` | VARCHAR(255) | No | Unique within tenant | Login identifier |
| `username` | VARCHAR(100) | No | Unique within tenant | Human-friendly identifier |
| `password_hash` | VARCHAR(255) | Yes | Auth-managed | bcrypt/Argon2 hash |
| `full_name` | VARCHAR(255) | No | — | Display name |
| `status` | VARCHAR(20) | No | CHECK | Account lifecycle |
| `created_at` | TIMESTAMPTZ | No | Default now | Creation time |
| `updated_at` | TIMESTAMPTZ | No | Default now | Last update |

---

# 5. User Status Model

Canonical service-contract values:

```text
ACTIVE
INACTIVE
SUSPENDED
```

### Meaning

| Status | Login | Read profile | Normal operations |
|---|---|---|---|
| `ACTIVE` | Yes | Yes | Yes |
| `INACTIVE` | No | Admin-controlled | No |
| `SUSPENDED` | No | Admin-controlled | No |

Authentication should map any non-`ACTIVE` user to an authentication denial.

### Source inconsistency

The high-level schema used `DISABLED`, while the User & Team contract uses `INACTIVE`/`SUSPENDED`.

**Final service schema uses `ACTIVE`, `INACTIVE`, `SUSPENDED` because this is the more complete current User & Team contract.**

Do not introduce `DISABLED` as a fourth value without a contract revision.

---

# 6. User Constraints

## 6.1 Email normalization

Before persistence:

```text
trim
↓
lowercase
↓
validate syntax
↓
store normalized form
```

Recommended:

```text
Jane.Doe@Example.com
        ↓
jane.doe@example.com
```

The system should not rely only on case-sensitive PostgreSQL equality for login uniqueness.

## 6.2 Username normalization

Use a documented policy, preferably:

```text
trim
lowercase
```

unless the UI explicitly requires case-preserving usernames.

## 6.3 Password hash

The service stores the hash but must never expose it in:

- REST responses
- logs
- events
- audit payloads
- internal lookup responses

Authentication supplies plaintext only through the protected credential endpoint; User & Team performs the hash operation.

---

# 7. User Indexes

Recommended indexes:

```sql
CREATE INDEX idx_users_org
    ON users (organization_id);

CREATE INDEX idx_users_org_status
    ON users (organization_id, status);

CREATE INDEX idx_users_org_email
    ON users (organization_id, email);

CREATE INDEX idx_users_org_username
    ON users (organization_id, username);

CREATE INDEX idx_users_created_at
    ON users (organization_id, created_at DESC);
```

The unique constraints already create indexes for email and username.

---

# 8. `teams`

## 8.1 Purpose

Represents a named group of users inside one organization.

## 8.2 PostgreSQL schema

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    organization_id UUID NOT NULL,

    name VARCHAR(255) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_teams_org_name
        UNIQUE (organization_id, name)
);
```

## 8.3 Column contract

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | UUID | No | Team identifier |
| `organization_id` | UUID | No | Tenant |
| `name` | VARCHAR(255) | No | Team name |
| `created_at` | TIMESTAMPTZ | No | Creation time |
| `updated_at` | TIMESTAMPTZ | No | Last update |

---

# 9. Team Indexes

```sql
CREATE INDEX idx_teams_org
    ON teams (organization_id);

CREATE INDEX idx_teams_org_created_at
    ON teams (organization_id, created_at DESC);
```

The unique `(organization_id, name)` constraint creates the main lookup index.

---

# 10. `team_members`

## 10.1 Purpose

Many-to-many relationship between users and teams.

## 10.2 PostgreSQL schema

```sql
CREATE TABLE team_members (
    team_id UUID NOT NULL,
    user_id UUID NOT NULL,

    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (team_id, user_id)
);
```

Because this is a separate service database:

- `team_id → teams.id` is a physical FK.
- `user_id → users.id` is a physical FK.

Recommended:

```sql
ALTER TABLE team_members
    ADD CONSTRAINT fk_team_members_team
    FOREIGN KEY (team_id)
    REFERENCES teams(id)
    ON DELETE CASCADE;

ALTER TABLE team_members
    ADD CONSTRAINT fk_team_members_user
    FOREIGN KEY (user_id)
    REFERENCES users(id)
    ON DELETE CASCADE;
```

---

# 11. Team Membership Security Invariants

The service must enforce:

```text
team.organization_id == user.organization_id
```

before inserting a membership.

A user from Organization A must never be added to a team belonging to Organization B.

This cannot be solved by the composite primary key alone.

Recommended transaction:

```text
BEGIN
  SELECT team.organization_id
  SELECT user.organization_id

  IF organizations differ:
      reject

  INSERT team_members
COMMIT
```

---

# 12. Team Membership Indexes

The primary key handles:

```text
(team_id, user_id)
```

Add:

```sql
CREATE INDEX idx_team_members_user
    ON team_members (user_id, team_id);
```

This is required for:

- "list all teams for user"
- permission inheritance
- team-based assignment
- user deactivation checks

---

# 13. User Deletion Policy

Do not hard-delete users by default.

Cases, audit events, comments, assignments, and historical records can continue to reference a user identifier.

Therefore:

```text
ACTIVE → INACTIVE/SUSPENDED
```

is preferred to deletion.

If hard deletion is ever required, it must be introduced as a new explicit data-retention decision because downstream records depend on stable user IDs.

---

# 14. Team Deletion Policy

The current contract includes:

```text
DELETE /api/v1/teams/{id}
```

Recommended behavior:

1. Verify tenant.
2. Verify authorization.
3. Remove memberships.
4. Prevent future assignments to the deleted team.
5. Preserve historical case/task references if other services already hold the team UUID.

Because teams are referenced by other services, a production-grade implementation should prefer an archive/deactivation model if historical integrity requires it.

If physical delete is retained for this capstone, do not allow deletion when the team is actively referenced by business workflows unless the downstream contract explicitly supports it.

---

# 15. Public User API

## 15.1 Create

```text
POST /api/v1/users
```

Required:

```json
{
  "email": "jane.doe@example.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "organizationId": "..."
}
```

### Security

A normal tenant user cannot choose an arbitrary `organizationId`.

The server derives tenant from the authenticated JWT unless the caller is a System Administrator provisioning a user for a specific tenant.

---

## 15.2 List

```text
GET /api/v1/users
```

Supported filters:

```text
status
email
username
search
page
size
sort
```

Tenant comes from JWT.

Never accept:

```text
?organizationId=another-tenant
```

as an authorization override.

---

## 15.3 Get user

```text
GET /api/v1/users/{id}
```

Query must effectively be:

```sql
SELECT ...
FROM users
WHERE id = :id
  AND organization_id = :jwtTenantId;
```

---

## 15.4 Update user

```text
PATCH /api/v1/users/{id}
```

Permitted profile fields:

```text
fullName
email
username
status
```

Do not permit:

```text
passwordHash
organizationId
id
createdAt
```

through the public endpoint.

Password updates belong to the Authentication internal contract.

Tenant changes should not be a normal PATCH operation.

---

# 16. Public Team API

```text
POST   /api/v1/teams
GET    /api/v1/teams
GET    /api/v1/teams/{id}
PATCH  /api/v1/teams/{id}
DELETE /api/v1/teams/{id}

POST   /api/v1/teams/{id}/members
GET    /api/v1/teams/{id}/members
DELETE /api/v1/teams/{id}/members/{userId}
```

All routes are tenant-scoped.

---

# 17. Internal Authentication Contracts

## 17.1 User lookup

```text
GET /internal/v1/users/lookup
```

Parameters:

```text
organizationId
email
```

Takes an already-**resolved** `organizationId`, never a slug — Organization Service owns slug resolution (`GET /internal/v1/organizations/by-slug/{slug}`), and Authentication calls that first. This service never resolves a slug itself.

Response:

```json
{
  "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "status": "ACTIVE",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

Never return `password_hash`.

---

# 18. Credential Verification

```text
POST /internal/v1/users/{id}/verify-credentials
```

Request:

```json
{
  "plaintextPassword": "TempPass!2026"
}
```

Response:

```json
{
  "valid": true,
  "status": "ACTIVE",
  "organizationId": "..."
}
```

### Rules

- Compare plaintext against stored password hash.
- Never return the hash.
- Never log the password.
- Apply constant-time/safe password verification through the selected password-hashing library.
- Return `valid: false` for wrong credentials rather than creating a distinguishable user-enumeration response.

---

# 19. Password Hash Update

```text
PATCH /internal/v1/users/{id}/password-hash
```

Request:

```json
{
  "newPlaintextPassword": "NewPass!2027"
}
```

The name is slightly misleading because Authentication sends plaintext to the endpoint.

The User & Team Service must:

```text
receive plaintext
↓
validate password policy
↓
hash with Argon2id/bcrypt
↓
replace password_hash
↓
update updated_at
```

It must never return the hash.

### Better future contract

A cleaner name is:

```text
POST /internal/v1/users/{id}/credentials/password
```

but this should only be adopted with a contract revision.

---

# 20. Tenant Validation

Before creating a user or team:

```text
organizationId
    ↓
Organization Service
    ↓
exists?
    ↓
ACTIVE?
    ↓
create
```

For authenticated tenant operations:

```text
JWT tenantId
      |
      v
requested resource.organization_id
      |
      +-- mismatch → reject
      |
      +-- match → continue
```

---

# 21. Authorization Matrix

| Operation | Required permission |
|---|---|
| Create user | `USER_MANAGE` |
| List users | `USER_MANAGE` or approved self-read scope |
| Update user | `USER_MANAGE` or self-profile scope |
| Disable/suspend user | `USER_MANAGE` |
| Create team | `TEAM_MANAGE` |
| Update team | `TEAM_MANAGE` |
| Delete team | `TEAM_MANAGE` |
| Add team member | `TEAM_MANAGE` |
| Remove team member | `TEAM_MANAGE` |
| Internal credential verification | Service identity only |
| Internal password update | Service identity only |

RBAC remains the authority for permission decisions.

---

# 22. Error Codes

| Code | HTTP | Meaning |
|---|---:|---|
| `USER_VALIDATION_ERROR` | 400 | Invalid user input |
| `USER_INVALID_STATUS` | 400 | Invalid lifecycle state |
| `TEAM_VALIDATION_ERROR` | 400 | Invalid team input |
| `AUTH_MISSING_TOKEN` | 401 | Authentication required |
| `USER_FORBIDDEN` | 403 | User operation not permitted |
| `TEAM_FORBIDDEN` | 403 | Team operation not permitted |
| `USER_NOT_FOUND` | 404 | Tenant-scoped user not found |
| `TEAM_NOT_FOUND` | 404 | Tenant-scoped team not found |
| `TEAM_MEMBER_NOT_FOUND` | 404 | Membership not found |
| `USER_EMAIL_TAKEN` | 409 | Email already exists in tenant |
| `TEAM_NAME_TAKEN` | 409 | Team name already exists in tenant |
| `TEAM_MEMBER_ALREADY_EXISTS` | 409 | Duplicate membership |
| `ORGANIZATION_NOT_FOUND` | 404 | Tenant does not exist |
| `ORGANIZATION_SUSPENDED` | 403 | Tenant cannot accept new operations |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Same key used with different request |

---

# 23. Transaction Boundaries

## User creation

```text
BEGIN
  validate organization
  normalize email/username
  check uniqueness
  hash password if this operation includes initial credentials
  INSERT users
COMMIT
```

No Kafka event is required by the current contract.

## Team membership

```text
BEGIN
  load team
  load user
  verify same tenant
  verify membership absent
  INSERT team_members
COMMIT
```

---

# 24. Concurrency and Race Conditions

Database uniqueness constraints are authoritative.

For two concurrent requests creating:

```text
same organization + same email
```

both may pass an application-level existence check, but only one insert succeeds.

The database must return the unique-constraint conflict and the service maps it to:

```text
409 USER_EMAIL_TAKEN
```

Likewise:

```text
organization_id + team.name
```

maps to:

```text
409 TEAM_NAME_TAKEN
```

---

# 25. Tenant Isolation Query Rule

Every tenant-owned query must contain tenant scope.

Bad:

```sql
SELECT * FROM users WHERE id = :id;
```

Correct:

```sql
SELECT *
FROM users
WHERE id = :id
  AND organization_id = :tenantId;
```

Bad:

```sql
SELECT * FROM teams WHERE id = :id;
```

Correct:

```sql
SELECT *
FROM teams
WHERE id = :id
  AND organization_id = :tenantId;
```

For team members, scope must be inherited through the team:

```sql
SELECT tm.user_id
FROM team_members tm
JOIN teams t ON t.id = tm.team_id
WHERE tm.team_id = :teamId
  AND t.organization_id = :tenantId;
```

---

# 26. API Response Safety

Never return:

```text
password_hash
organization internals
security secrets
internal service credentials
```

A user response should be approximately:

```json
{
  "id": "...",
  "organizationId": "...",
  "email": "jane.doe@example.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "status": "ACTIVE",
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

# 27. Referential Integrity Across Services

Because separate databases cannot enforce:

```text
users.organization_id → organization_db.organizations.id
```

the service must treat the Organization Service as the canonical authority.

However, historical records should retain IDs even if an organization is later suspended.

Therefore:

```text
Organization ACTIVE
    ↓
new user/team creation allowed

Organization SUSPENDED
    ↓
new user/team creation denied

Existing records
    ↓
retained
```

---

# 28. No Kafka Contract

The current architecture intentionally defines User & Team Service as synchronous.

Therefore this service does not produce:

```text
UserCreated
UserUpdated
TeamCreated
TeamMemberAdded
```

events in the current release.

If downstream systems later need these events, add them through the transactional outbox pattern and update the event contract before implementation.

Do not introduce Kafka merely because the service is separate.

---

# 29. Full Schema Summary

```text
resolve_user_team PostgreSQL
│
├── users
│   ├── id PK
│   ├── organization_id logical FK → Organization Service
│   ├── email
│   ├── username
│   ├── password_hash
│   ├── full_name
│   ├── status
│   ├── created_at
│   └── updated_at
│
├── teams
│   ├── id PK
│   ├── organization_id logical FK → Organization Service
│   ├── name
│   ├── created_at
│   └── updated_at
│
└── team_members
    ├── team_id FK → teams.id
    ├── user_id FK → users.id
    └── joined_at
```

---

# 30. Edge-Case Matrix

| Edge case | Expected result |
|---|---|
| Create user in nonexistent org | Reject |
| Create user in suspended org | Reject |
| Create user in another tenant | Reject |
| Duplicate tenant email | 409 |
| Duplicate tenant username | 409 |
| Duplicate team name | 409 |
| Add nonexistent user to team | 404 |
| Add user from another tenant | 403/409 security rejection |
| Add same member twice | 409 |
| Remove non-member | 404 |
| Update another tenant's user | 403/404, never data disclosure |
| List users without tenant scope | Never allowed |
| Authentication asks for unknown user | 404 internally; generic failure externally |
| Password hash included in response | Never |
| Concurrent duplicate creation | DB constraint decides winner |
| Organization suspended after user exists | User retained; new tenant operations denied |
| User becomes inactive | Authentication must reject future login |
| Team deleted while referenced elsewhere | Must follow explicit downstream retention policy |
| Internal endpoint called by public client | Reject |
| Organization Service unavailable | Controlled `503`; do not assume tenant exists |

---

# 31. Implementation Checklist

- [ ] Create `resolve_user_team` PostgreSQL database.
- [ ] Add migrations for `users`.
- [ ] Add migrations for `teams`.
- [ ] Add migrations for `team_members`.
- [ ] Add tenant-scoped indexes.
- [ ] Normalize email and username.
- [ ] Implement user CRUD.
- [ ] Implement team CRUD.
- [ ] Implement membership CRUD.
- [ ] Implement tenant validation.
- [ ] Implement internal user lookup.
- [ ] Implement credential verification.
- [ ] Implement password-hash update.
- [ ] Protect internal endpoints with service authentication.
- [ ] Add authorization checks through RBAC.
- [ ] Add unique-constraint conflict mapping.
- [ ] Add cross-tenant security tests.
- [ ] Add concurrent creation tests.
- [ ] Add password-hash non-disclosure tests.
- [ ] Verify no Kafka dependency is introduced.

---

# 32. Traceability

| Requirement | Schema/feature |
|---|---|
| FR-ORG-02 | `users` + user APIs |
| FR-ORG-03 | `teams` + `team_members` |
| FR-ORG-04 | `users.organization_id` |
| FR-IAM-01 | User provisioning + credentials |
| FR-TEN-01 | Tenant ownership column |
| FR-TEN-02 | Repository-level tenant filtering |
| FR-TEN-04 | JWT tenant binding consumed by this service |
| BR-09 | Tenant isolation |
| BR-15 | Team membership contributes to effective RBAC |
| NFR-SEC-02 | Tenant isolation and internal endpoint security |
| NFR-PERF-03 | Pagination and indexes |
