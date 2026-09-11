# Resolve — User & Team Service: Exhaustive API Contract Reference

**Service:** User & Team Service — Task 3
**Owns:** `users` (profile fields — `password_hash` is written here but owned/managed by Authentication), `teams`, `team_members`
**Depends on:** Task 2 (Organization Service) — a user cannot exist without a valid `organization_id`.
**Nature:** Synchronous REST only. Publishes no Kafka events in the initial scope.
**Sources used:** `General/SRS.md` (§1, §2.3, §2.4, §5, §7, §8), `General/Schemas_High_Level.md` (§2.2 users, §2.3 teams, §2.4 team_members), `tasks.md` (Task 3), and the frozen `Organization Service` contract (for the shared conventions and the cross-service coupling it flags at its §3.4).

**How this document is organized:** Section 2 covers the endpoints `tasks.md` already names explicitly (`GET/POST /users`, `PATCH /users/{id}`) — treated as the closest thing to a frozen baseline this service has, since `tasks.md` states them directly rather than leaving them fully open. Section 3 covers everything `tasks.md` defers to Task 0 ("team CRUD endpoints — define exact paths in Task 0") plus internal endpoints other services structurally require — each flagged as a **proposal**, not an already-agreed contract. Section 4 documents what's deliberately excluded, so the surface is bounded by decision rather than omission.

---

## 1. Shared Conventions (apply to every endpoint below)

**Base path:** `/api/v1` (public), `/internal/v1` (service-to-service only, never exposed through the gateway).

**Auth header:** `Authorization: Bearer {jwt}` on every public endpoint — this service has no unauthenticated endpoints.

**Internal auth:** mTLS client cert or a signed service-to-service JWT carrying a `service` claim. The caller's forwarded end-user token is never used for internal calls.

**Tenant scoping:** every request is scoped by the `organization_id` claim embedded in the caller's JWT. A request can never read or write a user/team/membership belonging to a different organization — attempting to do so returns `404`, not `403`, so callers cannot distinguish "exists in another tenant" from "doesn't exist" (avoids leaking cross-tenant existence).

**Pagination (list endpoints):**
Request: `?page=0&size=20&sort=createdAt,desc`
Response:
```json
{ "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 }
```

**Standard error shape** (every non-2xx response unless noted otherwise):
```json
{
  "timestamp": "2026-09-08T10:30:00Z",
  "status": 400,
  "code": "USER_VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/api/v1/users",
  "requestId": "req_7c1a2f",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "field": "email", "code": "INVALID_FORMAT", "message": "email must be a valid email address" }
  ]
}
```
`errors[]` only appears on `400` field-validation failures. All `code` values are namespaced `USER_*` or `TEAM_*` depending on the resource. Below, each endpoint's non-2xx examples only show the fields that change; assume `timestamp`, `requestId`, `traceId` are always present.

**Idempotency:** every resource-creating `POST` requires an `Idempotency-Key` header. A retry with the same key returns the original response verbatim; a retry with the same key but a different body returns:
```json
{ "status": 409, "code": "IDEMPOTENCY_KEY_REUSED", "message": "Idempotency-Key was already used with a different request body" }
```

**Authorization model note:** most endpoints below are gated on an `IAM_USER_MANAGE` / `IAM_TEAM_MANAGE`-style permission code (resolved via RBAC), scoped to the caller's own organization — not a hardcoded role name. A user with sufficient permission may manage users/teams within their own org only; no endpoint here grants cross-tenant access regardless of permission level. `GET` endpoints for a user's own profile are the one exception, additionally allowed for the resource owner themself (a user can always read their own record).

**Events:** none. No `UserCreated` / `TeamCreated` events exist in the SRS's domain-event catalog (`FR-EVT-01` lists only Case/Task/Document/Approval events) — consistent with this service being purely synchronous. If a future requirement needs other services to react to user/team changes, that's a new FR, not something silently already covered.

---

## 2. Endpoints Named Directly in `tasks.md`

### 2.1 `POST /api/v1/users`
Provisions a new user within the caller's tenant (`FR-ORG-02`, `FR-IAM-01`).

**Auth:** `IAM_USER_MANAGE` permission. **Idempotency-Key:** required.

**Request**
```json
{
  "email": "jane.doe@resolve.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "status": "ACTIVE"
}
```

| Field | Type | Rules |
|---|---|---|
| `email` | string | required, valid email format, unique within the organization |
| `username` | string | required, unique within the organization |
| `fullName` | string | required, ≤255 chars |
| `status` | string | optional, one of `ACTIVE`, `INACTIVE`, `SUSPENDED`; defaults to `ACTIVE` |

Note: `password_hash` is never accepted or returned by this endpoint — it is written exclusively by the Authentication Service (see the shared-table note in §5).

**201 Created**
```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@resolve.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "status": "ACTIVE",
  "createdAt": "2026-09-08T10:30:00Z",
  "updatedAt": "2026-09-08T10:30:00Z"
}
```

**400 Bad Request**
```json
{ "status": 400, "code": "USER_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/users",
  "errors": [{ "field": "email", "code": "INVALID_FORMAT", "message": "email must be a valid email address" }] }
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Caller lacks permission to provision users", "path": "/api/v1/users" }
```

**409 Conflict**
```json
{ "status": 409, "code": "USER_EMAIL_TAKEN", "message": "A user with this email already exists in this organization", "path": "/api/v1/users" }
```

---

### 2.2 `GET /api/v1/users`
Lists users within the caller's tenant (`FR-ORG-02`).

**Auth:** `IAM_USER_MANAGE` permission, or `IAM_USER_VIEW` for read-only listing.

**Request:** `?email=&username=&status=&teamId=&page=0&size=20&sort=createdAt,desc`

**200 OK**
```json
{
  "content": [
    { "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "email": "jane.doe@resolve.com", "username": "jane.doe", "fullName": "Jane Doe", "status": "ACTIVE",
      "createdAt": "2026-09-08T10:30:00Z", "updatedAt": "2026-09-08T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Caller lacks permission to list users", "path": "/api/v1/users" }
```

---

### 2.3 `GET /api/v1/users/{id}`
Retrieves a single user's profile. *(Implied by the SRS's per-resource read pattern; not explicitly listed in `tasks.md` alongside the list endpoint, but every other frozen contract in this project pairs a list endpoint with a get-by-id endpoint — flagged here rather than silently assumed.)*

**Auth:** `IAM_USER_VIEW` permission, or the resource owner reading their own record.

**200 OK** — same shape as §2.1's `201` body.

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users/{id}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Caller may not view this user", "path": "/api/v1/users/{id}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/users/{id}" }
```

---

### 2.4 `PATCH /api/v1/users/{id}`
Updates a user's profile fields, including status changes (`FR-ORG-02`).

**Auth:** `IAM_USER_MANAGE` permission. The resource owner may update a limited subset of their own fields (`fullName`); only a permission-holder may change `status` or `email`.

**Request** (partial update — send only fields to change)
```json
{ "status": "SUSPENDED" }
```

| Field | Type | Rules |
|---|---|---|
| `email` | string | optional, valid email format, unique within the organization |
| `username` | string | optional, unique within the organization |
| `fullName` | string | optional, ≤255 chars |
| `status` | string | optional, one of `ACTIVE`, `INACTIVE`, `SUSPENDED` |

**200 OK**
```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@resolve.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "status": "SUSPENDED",
  "createdAt": "2026-09-08T10:30:00Z",
  "updatedAt": "2026-09-08T11:15:00Z"
}
```

**400 Bad Request**
```json
{ "status": 400, "code": "USER_INVALID_STATUS", "message": "Request validation failed", "path": "/api/v1/users/{id}",
  "errors": [{ "field": "status", "code": "INVALID_ENUM_VALUE", "message": "status must be one of ACTIVE, INACTIVE, SUSPENDED" }] }
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users/{id}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Caller lacks permission to update this field", "path": "/api/v1/users/{id}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/users/{id}" }
```

**409 Conflict**
```json
{ "status": 409, "code": "USER_EMAIL_TAKEN", "message": "A user with this email already exists in this organization", "path": "/api/v1/users/{id}" }
```

---

## 3. Gap-Fill Recommendations (not yet in a frozen contract)

`tasks.md` explicitly defers exact team endpoint paths to Task 0 ("plus team CRUD endpoints (define exact paths in Task 0)"). The proposals below follow the same shape and conventions as the endpoints in Section 2, so your team can ratify or amend them rather than starting from nothing.

### 3.1 `POST /api/v1/teams` — create a team
**Why:** `FR-ORG-03` requires team creation; no endpoint exists yet.

**Auth:** `IAM_TEAM_MANAGE` permission. **Idempotency-Key:** required.

**Request**
```json
{ "name": "Investigations Team" }
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | required, ≤255 chars, unique within the organization |

**201 Created**
```json
{
  "id": "9b2d1e4a-1234-4a1b-8c3d-5e6f7a8b9c0d",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Investigations Team",
  "createdAt": "2026-09-08T10:30:00Z",
  "updatedAt": "2026-09-08T10:30:00Z"
}
```

**400 Bad Request**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams",
  "errors": [{ "field": "name", "code": "REQUIRED", "message": "name is required" }] }
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to create teams", "path": "/api/v1/teams" }
```

**409 Conflict**
```json
{ "status": 409, "code": "TEAM_NAME_TAKEN", "message": "A team with this name already exists in this organization", "path": "/api/v1/teams" }
```

---

### 3.2 `GET /api/v1/teams` — list teams
**Why:** pairs with §3.1 the same way `GET /users` pairs with `POST /users`; other services (e.g. Workflow, when assigning work to a team) need a way to enumerate teams.

**Auth:** `IAM_TEAM_VIEW` permission.

**Request:** `?name=&page=0&size=20&sort=createdAt,desc`

**200 OK**
```json
{
  "content": [
    { "id": "9b2d1e4a-1234-4a1b-8c3d-5e6f7a8b9c0d", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "name": "Investigations Team", "createdAt": "2026-09-08T10:30:00Z", "updatedAt": "2026-09-08T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to list teams", "path": "/api/v1/teams" }
```

---

### 3.3 `GET /api/v1/teams/{id}` — retrieve a single team
**Why:** standard get-by-id pairing, consistent with every other resource in this contract and the Organization Service's own pattern.

**Auth:** `IAM_TEAM_VIEW` permission, or membership on the team itself.

**200 OK** — same shape as §3.1's `201` body, plus a `members` count or summary is a reasonable future extension (not included here to keep the contract minimal; propose separately if needed).

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller may not view this team", "path": "/api/v1/teams/{id}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

---

### 3.4 `PATCH /api/v1/teams/{id}` — rename a team
**Why:** mirrors `PATCH /users/{id}`; a team name typo at creation time otherwise has no correction path other than direct DB access, which conflicts with `FR-QA-05` (migrations-only schema changes — data fixes must go through the API).

**Auth:** `IAM_TEAM_MANAGE` permission.

**Request**
```json
{ "name": "Fraud Investigations Team" }
```

**200 OK**
```json
{
  "id": "9b2d1e4a-1234-4a1b-8c3d-5e6f7a8b9c0d",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Fraud Investigations Team",
  "createdAt": "2026-09-08T10:30:00Z",
  "updatedAt": "2026-09-08T12:00:00Z"
}
```

**400 Bad Request**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams/{id}",
  "errors": [{ "field": "name", "code": "REQUIRED", "message": "name is required" }] }
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to update this team", "path": "/api/v1/teams/{id}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

**409 Conflict**
```json
{ "status": 409, "code": "TEAM_NAME_TAKEN", "message": "A team with this name already exists in this organization", "path": "/api/v1/teams/{id}" }
```

---

### 3.5 `DELETE /api/v1/teams/{id}` — delete a team
**Why:** the schema's `team_members` foreign keys cascade on team deletion (`Schemas_High_Level.md` §2.4), implying deletion is a supported operation; without this endpoint an empty/disbanded team can never be removed.

**Auth:** `IAM_TEAM_MANAGE` permission.

**204 No Content** — empty body.

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to delete this team", "path": "/api/v1/teams/{id}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

---

### 3.6 `POST /api/v1/teams/{id}/members` — add a member
**Why:** `FR-ORG-03` requires team membership management; this is the core operation the whole table exists for.

**Auth:** `IAM_TEAM_MANAGE` permission. **Idempotency-Key:** required.

**Request**
```json
{ "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7" }
```

**201 Created**
```json
{
  "teamId": "9b2d1e4a-1234-4a1b-8c3d-5e6f7a8b9c0d",
  "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "joinedAt": "2026-09-08T10:30:00Z"
}
```

**400 Bad Request**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams/{id}/members",
  "errors": [{ "field": "userId", "code": "REQUIRED", "message": "userId is required" }] }
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to manage this team's membership", "path": "/api/v1/teams/{id}/members" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}/members" }
```
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/teams/{id}/members" }
```

**409 Conflict**
```json
{ "status": 409, "code": "TEAM_MEMBER_ALREADY_EXISTS", "message": "User is already a member of this team", "path": "/api/v1/teams/{id}/members" }
```

---

### 3.7 `GET /api/v1/teams/{id}/members` — list a team's members
**Why:** without this, there's no way to see who's on a team short of listing every user and cross-referencing manually.

**Auth:** `IAM_TEAM_VIEW` permission, or membership on the team itself.

**Request:** `?page=0&size=20`

**200 OK**
```json
{
  "content": [
    { "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "fullName": "Jane Doe", "email": "jane.doe@resolve.com", "joinedAt": "2026-09-08T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller may not view this team's membership", "path": "/api/v1/teams/{id}/members" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}/members" }
```

---

### 3.8 `DELETE /api/v1/teams/{id}/members/{userId}` — remove a member
**Why:** membership management is incomplete without a removal path; symmetric with §3.6.

**Auth:** `IAM_TEAM_MANAGE` permission.

**204 No Content** — empty body.

**401 Unauthorized**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members/{userId}" }
```

**403 Forbidden**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Caller lacks permission to manage this team's membership", "path": "/api/v1/teams/{id}/members/{userId}" }
```

**404 Not Found**
```json
{ "status": 404, "code": "TEAM_MEMBER_NOT_FOUND", "message": "User is not a member of this team", "path": "/api/v1/teams/{id}/members/{userId}" }
```

---

### 3.9 `GET /internal/v1/users/lookup` — service-to-service lookup by email, within a resolved tenant
**Why:** Authentication's `POST /api/v1/auth/login` needs to turn an `(organizationId, email)` pair into a `userId` + `status` before it can call §3.11's credential-verification endpoint. **`passwordHash` is never returned by this service, on any endpoint, public or internal** — see §3.11 for why the hash never has to leave this service at all. Tenant-slug resolution belongs to Organization Service, not here: this endpoint takes an already-resolved `organizationId`, never a slug (closing the coupling `Contracts/Organisation_Service.md` §3.4 flags — Authentication calls `GET /internal/v1/organizations/by-slug/{slug}` on Organization Service first, and only passes this service the resolved id).

**Auth:** service identity (mTLS/service JWT).

**Request:** `?organizationId=3fa85f64-5717-4562-b3fc-2c963f66afa6&email=jane.doe@resolve.com`

**200 OK**
```json
{
  "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "status": "ACTIVE"
}
```

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/lookup" }
```
Authentication maps this to the same generic `AUTH_INVALID_CREDENTIALS` shown to the client, never a distinguishable 404 (prevents user enumeration).

---

### 3.10 `GET /internal/v1/users/{id}` — service-to-service lookup by id
**Why:** any service that writes a `user_id`/`assignee_id` foreign key (Case Management on assignment, Workflow on task/approval assignment) needs to confirm the user exists and is `ACTIVE` before writing — the same structural need the Organization Service's own §3.3 documents for `organization_id`.

**Auth:** service identity (mTLS/service JWT).

**200 OK**
```json
{ "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "fullName": "Jane Doe", "status": "ACTIVE" }
```

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/{id}" }
```

---

### 3.11 `POST /internal/v1/users/{id}/verify-credentials` — service-to-service credential check
**Why:** `password_hash` physically lives in this service's `users` table (§5's shared-table note), but Authentication owns credential logic. Rather than exposing the hash to Authentication at all — which would mean two services independently implementing hash comparison, and a hash value crossing a network boundary — this service accepts a plaintext password over the (mTLS-protected, internal-only) network and does the comparison itself, returning only a boolean.

**Auth:** service identity (mTLS/service JWT).

**Request**
```json
{ "plaintextPassword": "TempPass!2026" }
```

**200 OK**
```json
{ "valid": true, "status": "ACTIVE", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" }
```
`valid: false` is a normal `200`, not an error — it lets Authentication decide the client-facing response shape (generic `AUTH_INVALID_CREDENTIALS`) itself, rather than this service leaking a distinction between "wrong password" and any other failure mode.

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/{id}/verify-credentials" }
```

**Security requirements:** never log the plaintext password; never return `passwordHash` under any circumstance, on this or any other endpoint; use a constant-time/safe comparison via the chosen password-hashing library (bcrypt/Argon2id).

---

### 3.12 `PATCH /internal/v1/users/{id}/password-hash` — service-to-service password update
**Why:** symmetric with §3.11 — Authentication needs to set a new password (initial set, password-reset confirm) without ever computing or holding a hash itself, since this service is the sole writer of its own `users` table (§5).

**Auth:** service identity (mTLS/service JWT).

**Request**
```json
{ "newPlaintextPassword": "NewPass!2027" }
```
The field name is intentionally explicit that this is plaintext in transit (over mTLS, internal-only) — this service hashes it before writing.

**204 No Content** — `password_hash` updated, `updated_at` bumped. Never returns the new hash.

**404 Not Found**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/{id}/password-hash" }
```

---

## 4. Explicitly Out of Scope

Called out so the surface is bounded by decision, not by oversight:

| Not provided | Why |
|---|---|
| `DELETE /api/v1/users/{id}` (hard delete) | Users are deactivated via `status: INACTIVE`/`SUSPENDED`, never hard-deleted — consistent with the audit trail requirement (`FR-AUD-*`) that historical actor references must remain resolvable. |
| Password set/change on any **public** endpoint | `password_hash` is written only via the internal §3.12 endpoint, callable by Authentication's service identity only — never by an end-user request, and never accepted in any `/api/v1/*` request body. |
| Self-service signup (unauthenticated `POST /users`) | Mirrors the Organization Service's own exclusion — user provisioning is an administrative action gated on `IAM_USER_MANAGE`, not open registration. |
| Bulk import/CSV upload of users | No FR describes bulk provisioning; out of scope unless a future requirement adds it. |
| Nested/hierarchical teams (sub-teams, team-of-teams) | The schema (`teams`, `team_members`) models a flat structure only — no `parent_team_id` column exists. |
| Cross-organization team membership | `team_members` only ever links a user and team within the same organization; the FK constraints don't support cross-tenant membership, and no FR asks for it. |

---

## 5. Data Owned

### `users` (from `Schemas_High_Level.md` §2.2)

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | User identifier, referenced by every service that assigns work to a person. |
| `organization_id` | UUID | NOT NULL, FK → `organizations.id` | Tenant this user belongs to (`FR-ORG-04`: every user in exactly one organization). |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE per organization | Login identifier. |
| `username` | VARCHAR(100) | NOT NULL, UNIQUE per organization | Display handle. |
| `password_hash` | VARCHAR(255) | nullable | **Written and owned by Authentication**, not this service — see the shared-table note below. |
| `full_name` | VARCHAR(255) | NOT NULL | — |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'`, CHECK IN (`ACTIVE`,`INACTIVE`,`SUSPENDED`) | Account lifecycle state. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | — |

**Shared-table note (open decision for Task 0):** `password_hash` lives in this service's table but must be written exclusively by Authentication. Two options to ratify in Task 0: **(a)** Authentication writes directly to this column via a shared database connection (simplest, but blurs table ownership across services), or **(b)** Authentication calls a small internal endpoint on this service (e.g. `PUT /internal/v1/users/{id}/password-hash`) to set it, keeping this service the sole writer of its own table. Option (b) is more consistent with this project's "one table, one owning service" principle and is the recommended default absent a Task 0 objection.

### `teams` (from `Schemas_High_Level.md` §2.3)

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | Team identifier. |
| `organization_id` | UUID | NOT NULL, FK → `organizations.id` | Tenant this team belongs to. |
| `name` | VARCHAR(255) | NOT NULL, UNIQUE per organization | — |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | — |

### `team_members` (from `Schemas_High_Level.md` §2.4)

| Column | Type | Constraints | Description |
|---|---|---|---|
| `team_id` | UUID | PK (composite), FK → `teams.id` ON DELETE CASCADE | — |
| `user_id` | UUID | PK (composite), FK → `users.id` ON DELETE CASCADE | — |
| `joined_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | — |

No `version` column on any of the three tables — optimistic locking is scoped only to `cases` per the schema doc's merge notes; nothing in the FRs identifies concurrent user/team updates as a contention risk worth the extra column.

---

## 6. Error Code Catalog (this service)

| Code | HTTP Status | Meaning |
|---|---|---|
| `USER_VALIDATION_ERROR` | 400 | User request body failed field validation. |
| `USER_INVALID_STATUS` | 400 | `status` value outside `ACTIVE`/`INACTIVE`/`SUSPENDED`. |
| `TEAM_VALIDATION_ERROR` | 400 | Team request body failed field validation. |
| `AUTH_MISSING_TOKEN` | 401 | No/invalid bearer token (shared platform-wide code). |
| `USER_FORBIDDEN` | 403 | Caller lacks the required user-management permission, or may not view this user. |
| `TEAM_FORBIDDEN` | 403 | Caller lacks the required team-management permission, or may not view this team. |
| `USER_NOT_FOUND` | 404 | No user with the given id/email in the caller's organization. |
| `TEAM_NOT_FOUND` | 404 | No team with the given id in the caller's organization. |
| `TEAM_MEMBER_NOT_FOUND` | 404 | The specified user is not a member of the specified team. |
| `USER_EMAIL_TAKEN` | 409 | `email` already in use by another user in the same organization. |
| `TEAM_NAME_TAKEN` | 409 | `name` already in use by another team in the same organization. |
| `TEAM_MEMBER_ALREADY_EXISTS` | 409 | The specified user is already a member of the specified team. |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Same `Idempotency-Key`, different body, on a retried `POST`. |

---

## 7. Requirement Traceability

| Endpoint | Traces to |
|---|---|
| `POST /api/v1/users` | `FR-ORG-02`, `FR-IAM-01` |
| `GET /api/v1/users` | `FR-ORG-02`, `NFR-PERF-03` (pagination on list endpoints) |
| `GET /api/v1/users/{id}` (§2.3, implied) | `FR-ORG-02` |
| `PATCH /api/v1/users/{id}` | `FR-ORG-02` |
| `POST /api/v1/teams` (§3.1, proposed) | `FR-ORG-03` |
| `GET /api/v1/teams` (§3.2, proposed) | `FR-ORG-03`, `NFR-PERF-03` |
| `GET /api/v1/teams/{id}` (§3.3, proposed) | `FR-ORG-03` |
| `PATCH /api/v1/teams/{id}` (§3.4, proposed) | `FR-ORG-03`, `NFR-DATA-03` (no manual DDL fixes) |
| `DELETE /api/v1/teams/{id}` (§3.5, proposed) | `FR-ORG-03` |
| `POST /api/v1/teams/{id}/members` (§3.6, proposed) | `FR-ORG-03` |
| `GET /api/v1/teams/{id}/members` (§3.7, proposed) | `FR-ORG-03` |
| `DELETE /api/v1/teams/{id}/members/{userId}` (§3.8, proposed) | `FR-ORG-03` |
| `GET /internal/v1/users/lookup` (§3.9, proposed — but already relied upon) | `FR-IAM-02` (login), closes the coupling the Organization Service's own §3.4 flags |
| `GET /internal/v1/users/{id}` (§3.10, proposed) | `FR-ORG-04`, `NFR-SEC-02` |
| `POST /internal/v1/users/{id}/verify-credentials` (§3.11, proposed — but already relied upon) | `FR-IAM-02`, `FR-IAM-05` (password_hash never leaves this service) |
| `PATCH /internal/v1/users/{id}/password-hash` (§3.12, proposed — but already relied upon) | `FR-IAM-05`, `FR-IAM-07` (password reset) |

**Compiled from** `Resolve_Documentation/{General/SRS.md, General/Schemas_High_Level.md, tasks.md}` and the frozen Organization Service contract. Section 2 reflects the closest thing to an agreed baseline `tasks.md` provides; Section 3 requires Task 0 sign-off before implementation.
