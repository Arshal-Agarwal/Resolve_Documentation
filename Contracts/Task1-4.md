# Resolve — Task 1–4 Contracts (Single Source of Truth)

**Scope of this document:** Task 1 (Shared Infrastructure conventions), Task 2 (Organization Service), Task 3 (User & Team Service), Task 4 (Authentication Service). Tasks 5–10 (RBAC, Case Management, Workflow, Document, Audit, Notification) are **not** included here — see `Resolve_Service_Contracts_Consolidated.md` for those.
**Supersedes (for this scope):** the Task 1–4 portions of all four prior submissions.
**What's new versus the last consolidated pass:** every endpoint below now has a **full example body for every status code it can return**, not just the success case — this is what each owner should build directly against.

> Conflict resolutions (pagination field name, error shape, JWT claim casing, login field name, JWT verification strategy, refresh-token semantics, the `users`/`password_hash` split) carry over unchanged from the full consolidation — restated in §0 for convenience, not re-litigated.

---

## 0. Conflict Resolutions (Recap, Tasks 1–4 Only)

| Conflict | Resolved as |
|---|---|
| `users` / `password_hash` ownership | User & Team Service owns the entire table. Authentication has zero direct DB access — it calls internal endpoints instead. |
| Pagination field name | `content` (matches Spring Data's default `Page` serialization — no custom mapping needed). |
| Error response shape | Flat top-level fields (matches Spring Boot's default error-attribute shape), with a service-namespaced `code` and a field-level `errors[]` array. |
| JWT tenant claim casing | `tenant_id` (snake_case) in the token specifically; body/response JSON stays camelCase everywhere else. |
| Login request field | `identifier` (accepts email or username) + `organizationSlug` (disambiguates tenant before the per-tenant email lookup). |
| JWT verification strategy | Local verification via a published JWKS endpoint, RS256, no roles/permissions embedded. |
| Refresh-token representation | Opaque token, Redis-backed, rotated on every use. |

---

## 1. Task 1 — Shared Conventions Every Endpoint Below Follows

### Base path & auth
- Public endpoints: `/api/v1/...`. Internal, service-to-service-only endpoints: `/internal/v1/...` — never exposed through the gateway.
- `Authorization: Bearer {jwt}` required on everything except `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/password-reset/request`, and `GET /.well-known/jwks.json`.
- Internal endpoints require a service-identity credential (mTLS client cert or a signed service-to-service JWT with a `service` claim), never the end user's forwarded token.

### Pagination (list endpoints)
Request: `?page=0&size=20&sort=createdAt,desc`

Response:
```json
{ "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 }
```

### Standard error shape
Used for every non-2xx response below unless otherwise noted:
```json
{
  "timestamp": "2026-09-05T10:30:00Z",
  "status": 400,
  "code": "USER_VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/api/v1/users",
  "requestId": "req_9f2a1c",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "field": "email", "code": "INVALID_FORMAT", "message": "Invalid email address" }
  ]
}
```
`errors[]` is present only for `400` field-validation failures; omitted for other status codes. `code` is namespaced per service: `ORG_*`, `USER_*`, `TEAM_*`, `AUTH_*`. Below, each endpoint's error responses only show the fields that change (`status`, `code`, `message`, `path`) — assume `timestamp`, `requestId`, `traceId` are always present as shown above.

### Idempotency
Any `POST` that creates a resource or triggers a side effect requires `Idempotency-Key`. Retried requests with the same key return the original response verbatim. A retry with the **same key but a different request body** returns:
```json
{ "status": 409, "code": "IDEMPOTENCY_KEY_REUSED", "message": "Idempotency-Key was already used with a different request body" }
```

### Kafka / events in this scope
None of the four services in this document produce Kafka events in the initial scope. Task 1's infrastructure work (Kafka cluster, outbox library, topic provisioning) exists to serve Tasks 6–10; Tasks 2–4 don't publish anything themselves.

---

## 2. Task 2 — Organization Service

**Owns:** `organizations`. **Depends on:** nothing.

### `POST /api/v1/organizations`
Provisions a new tenant. **Auth:** System Admin role.

**Request**
```json
{ "name": "Acme Bank", "slug": "acme-bank" }
```

**`201 Created`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Acme Bank",
  "slug": "acme-bank",
  "status": "ACTIVE",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T10:30:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "ORG_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/organizations",
  "errors": [{ "field": "slug", "code": "INVALID_FORMAT", "message": "Slug must be lowercase alphanumeric with hyphens" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/organizations" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "ORG_FORBIDDEN", "message": "Only System Administrators may provision organizations", "path": "/api/v1/organizations" }
```

**`409 Conflict`**
```json
{ "status": 409, "code": "ORG_SLUG_TAKEN", "message": "An organization with this slug already exists", "path": "/api/v1/organizations" }
```

---

### `GET /api/v1/organizations/{id}`
**Auth:** System Admin, or a member of that org.

**`200 OK`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Acme Bank",
  "slug": "acme-bank",
  "status": "ACTIVE",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T10:30:00Z"
}
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/organizations/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "ORG_FORBIDDEN", "message": "Not a member of this organization", "path": "/api/v1/organizations/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "ORG_NOT_FOUND", "message": "Organization not found", "path": "/api/v1/organizations/{id}" }
```

---

### `PATCH /api/v1/organizations/{id}/status`
**Auth:** System Admin role.

**Request**
```json
{ "status": "SUSPENDED" }
```

**`200 OK`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Acme Bank",
  "slug": "acme-bank",
  "status": "SUSPENDED",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T11:15:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "ORG_INVALID_STATUS", "message": "Request validation failed", "path": "/api/v1/organizations/{id}/status",
  "errors": [{ "field": "status", "code": "INVALID_ENUM_VALUE", "message": "status must be one of ACTIVE, SUSPENDED" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/organizations/{id}/status" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "ORG_FORBIDDEN", "message": "Only System Administrators may change organization status", "path": "/api/v1/organizations/{id}/status" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "ORG_NOT_FOUND", "message": "Organization not found", "path": "/api/v1/organizations/{id}/status" }
```

---

## 3. Task 3 — User & Team Service

**Owns:** `users` (entire table, including `password_hash`), `teams`, `team_members`. **Depends on:** Organization Service.

### 3.1 Users

#### `POST /api/v1/users`
**Auth:** `USER_MANAGE` permission.

**Request**
```json
{ "email": "jane.doe@acmebank.com", "username": "jane.doe", "fullName": "Jane Doe", "initialPassword": "TempPass!2026" }
```

**`201 Created`**
```json
{
  "id": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@acmebank.com",
  "username": "jane.doe",
  "fullName": "Jane Doe",
  "status": "ACTIVE",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T10:30:00Z"
}
```
*(never includes `passwordHash`)*

**`400 Bad Request`**
```json
{ "status": 400, "code": "USER_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/users",
  "errors": [{ "field": "initialPassword", "code": "TOO_WEAK", "message": "Password must be at least 8 characters" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Missing USER_MANAGE permission", "path": "/api/v1/users" }
```

**`409 Conflict`**
```json
{ "status": 409, "code": "USER_EMAIL_TAKEN", "message": "A user with this email already exists in this organization", "path": "/api/v1/users" }
```

---

#### `GET /api/v1/users`
**Auth:** any authenticated user. Supports `?status=&email=&username=&page=&size=`.

**`200 OK`**
```json
{
  "content": [
    { "id": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "email": "jane.doe@acmebank.com", "username": "jane.doe", "fullName": "Jane Doe", "status": "ACTIVE",
      "createdAt": "2026-09-05T10:30:00Z", "updatedAt": "2026-09-05T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users" }
```

---

#### `GET /api/v1/users/{id}`
**Auth:** same tenant.

**`200 OK`** — single user object, same shape as `POST /users`'s `201`.

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Cannot access users outside your organization", "path": "/api/v1/users/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/users/{id}" }
```

---

#### `PATCH /api/v1/users/{id}`
**Auth:** `USER_MANAGE`, or the user themself for their own profile fields (not `status`, not `password_hash`).

**Request**
```json
{ "fullName": "Jane R. Doe" }
```

**`200 OK`**
```json
{
  "id": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@acmebank.com",
  "username": "jane.doe",
  "fullName": "Jane R. Doe",
  "status": "ACTIVE",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T11:20:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "USER_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/users/{id}",
  "errors": [{ "field": "fullName", "code": "TOO_LONG", "message": "fullName must be 255 characters or fewer" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Cannot update another user's profile without USER_MANAGE", "path": "/api/v1/users/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/users/{id}" }
```

**`409 Conflict`** *(only if the team adopts a `version` column on `users` — open item, see §5)*
```json
{ "status": 409, "code": "USER_STALE_UPDATE", "message": "User was modified by another request; refetch and retry", "path": "/api/v1/users/{id}" }
```

---

#### `PATCH /api/v1/users/{id}/status`
**Auth:** `USER_MANAGE`.

**Request**
```json
{ "status": "DISABLED" }
```

**`200 OK`**
```json
{
  "id": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@acmebank.com",
  "status": "DISABLED",
  "updatedAt": "2026-09-05T11:25:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "USER_INVALID_STATUS", "message": "Request validation failed", "path": "/api/v1/users/{id}/status",
  "errors": [{ "field": "status", "code": "INVALID_ENUM_VALUE", "message": "status must be one of ACTIVE, DISABLED" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/users/{id}/status" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "USER_FORBIDDEN", "message": "Missing USER_MANAGE permission", "path": "/api/v1/users/{id}/status" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/users/{id}/status" }
```

---

### 3.2 Teams

#### `GET /api/v1/teams`
**Auth:** any authenticated user.

**`200 OK`**
```json
{
  "content": [
    { "id": "a1b2c3d4-0000-4000-8000-000000000001", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "name": "AML Investigations", "createdAt": "2026-09-05T10:30:00Z", "updatedAt": "2026-09-05T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams" }
```

---

#### `POST /api/v1/teams`
**Auth:** `TEAM_MANAGE`.

**Request**
```json
{ "name": "AML Investigations" }
```

**`201 Created`**
```json
{
  "id": "a1b2c3d4-0000-4000-8000-000000000001",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "AML Investigations",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T10:30:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams",
  "errors": [{ "field": "name", "code": "REQUIRED", "message": "name is required" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Missing TEAM_MANAGE permission", "path": "/api/v1/teams" }
```

**`409 Conflict`** *(only if team names are unique per tenant — open item, see §5)*
```json
{ "status": 409, "code": "TEAM_NAME_TAKEN", "message": "A team with this name already exists in this organization", "path": "/api/v1/teams" }
```

---

#### `GET /api/v1/teams/{id}`
**Auth:** any authenticated user.

**`200 OK`** — single team object, same shape as `POST /teams`'s `201`.

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Cannot access teams outside your organization", "path": "/api/v1/teams/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

---

#### `PATCH /api/v1/teams/{id}`
**Auth:** `TEAM_MANAGE`.

**Request**
```json
{ "name": "Senior AML Investigations" }
```

**`200 OK`**
```json
{
  "id": "a1b2c3d4-0000-4000-8000-000000000001",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Senior AML Investigations",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T11:30:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams/{id}",
  "errors": [{ "field": "name", "code": "TOO_LONG", "message": "name must be 255 characters or fewer" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Missing TEAM_MANAGE permission", "path": "/api/v1/teams/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

---

#### `DELETE /api/v1/teams/{id}`
**Auth:** `TEAM_MANAGE`.

**`204 No Content`** *(empty body)*

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Missing TEAM_MANAGE permission", "path": "/api/v1/teams/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}" }
```

**`409 Conflict`** *(if the team is currently assigned to open cases — recommend soft-delete instead, see §5)*
```json
{ "status": 409, "code": "TEAM_HAS_ACTIVE_ASSIGNMENTS", "message": "Team is assigned to one or more open cases and cannot be deleted", "path": "/api/v1/teams/{id}" }
```

---

#### `GET /api/v1/teams/{id}/members`
**Auth:** any authenticated user.

**`200 OK`**
```json
{
  "content": [
    { "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f", "joinedAt": "2026-09-05T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}/members" }
```

---

#### `POST /api/v1/teams/{id}/members`
**Auth:** `TEAM_MANAGE`.

**Request**
```json
{ "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f" }
```

**`201 Created`**
```json
{
  "teamId": "a1b2c3d4-0000-4000-8000-000000000001",
  "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "joinedAt": "2026-09-05T11:35:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "TEAM_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/teams/{id}/members",
  "errors": [{ "field": "userId", "code": "INVALID_FORMAT", "message": "userId must be a valid UUID" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Missing TEAM_MANAGE permission", "path": "/api/v1/teams/{id}/members" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}/members" }
```
or
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/api/v1/teams/{id}/members" }
```

**`409 Conflict`**
```json
{ "status": 409, "code": "TEAM_MEMBER_ALREADY_EXISTS", "message": "User is already a member of this team", "path": "/api/v1/teams/{id}/members" }
```

---

#### `DELETE /api/v1/teams/{id}/members/{userId}`
**Auth:** `TEAM_MANAGE`.

**`204 No Content`** *(empty body)*

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/teams/{id}/members/{userId}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "TEAM_FORBIDDEN", "message": "Missing TEAM_MANAGE permission", "path": "/api/v1/teams/{id}/members/{userId}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "TEAM_NOT_FOUND", "message": "Team not found", "path": "/api/v1/teams/{id}/members/{userId}" }
```
or
```json
{ "status": 404, "code": "TEAM_MEMBER_NOT_FOUND", "message": "User is not a member of this team", "path": "/api/v1/teams/{id}/members/{userId}" }
```

---

### 3.3 Internal Endpoints (Authentication Service is the only caller)

#### `GET /internal/v1/users/by-email?email=&organizationSlug=`

**`200 OK`**
```json
{ "id": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "status": "ACTIVE" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "No user with this email in this organization", "path": "/internal/v1/users/by-email" }
```

---

#### `POST /internal/v1/users/{id}/verify-credentials`

**Request**
```json
{ "plaintextPassword": "TempPass!2026" }
```

**`200 OK`** (valid credentials)
```json
{ "valid": true, "status": "ACTIVE", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" }
```

**`200 OK`** (wrong password — still `200`, not `401`, so Authentication decides the client-facing error)
```json
{ "valid": false, "status": "ACTIVE" }
```

**`200 OK`** (disabled account)
```json
{ "valid": false, "status": "DISABLED" }
```

**`404 Not Found`** (should not normally occur if the caller already resolved `id` via `by-email` first)
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/{id}/verify-credentials" }
```

---

#### `PATCH /internal/v1/users/{id}/password-hash`

**Request**
```json
{ "newPlaintextPassword": "NewPass!2027" }
```

**`204 No Content`** *(empty)* — hashed and stored internally; Authentication never sees or computes the hash.

**`400 Bad Request`**
```json
{ "status": 400, "code": "USER_VALIDATION_ERROR", "message": "Request validation failed", "path": "/internal/v1/users/{id}/password-hash",
  "errors": [{ "field": "newPlaintextPassword", "code": "TOO_WEAK", "message": "Password must be at least 8 characters" }] }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "USER_NOT_FOUND", "message": "User not found", "path": "/internal/v1/users/{id}/password-hash" }
```

---

## 4. Task 4 — Authentication Service

**Owns:** no Postgres table; Redis session/security state only. **Depends on:** User & Team Service.

### `POST /api/v1/auth/login`
**Auth:** none (public).

**Request**
```json
{ "identifier": "jane.doe@acmebank.com", "organizationSlug": "acme-bank", "password": "TempPass!2026" }
```

**`200 OK`**
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "8b2c1e4a-9f3d-4b2a-8c1d-...",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "AUTH_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/auth/login",
  "errors": [{ "field": "organizationSlug", "code": "REQUIRED", "message": "organizationSlug is required" }] }
```

**`401 Unauthorized`** *(returned for both "no such user" and "wrong password" — never reveal which, to avoid user enumeration)*
```json
{ "status": 401, "code": "AUTH_INVALID_CREDENTIALS", "message": "Invalid email/username or password", "path": "/api/v1/auth/login" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "AUTH_ACCOUNT_DISABLED", "message": "This account has been disabled", "path": "/api/v1/auth/login" }
```

**Internal calls made:** `GET /internal/v1/users/by-email` → `POST /internal/v1/users/{id}/verify-credentials` (both on User & Team Service).

---

### `POST /api/v1/auth/refresh`
**Auth:** none — refresh token carried in the body.

**Request**
```json
{ "refreshToken": "8b2c1e4a-9f3d-4b2a-8c1d-..." }
```

**`200 OK`** — new `accessToken` and a **rotated** `refreshToken` (the old one is invalidated in Redis)
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "c3d4e5f6-a7b8-4c9d-0e1f-...",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "AUTH_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/auth/refresh",
  "errors": [{ "field": "refreshToken", "code": "REQUIRED", "message": "refreshToken is required" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_INVALID_REFRESH_TOKEN", "message": "Refresh token is invalid, expired, or already used", "path": "/api/v1/auth/refresh" }
```

---

### `POST /api/v1/auth/logout`
**Auth:** valid access token. Empty body; the token to revoke is read from the `Authorization` header.

**`204 No Content`** *(empty)* — writes `session:revoked:{tokenId}` to Redis.

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/auth/logout" }
```
or
```json
{ "status": 401, "code": "AUTH_INVALID_TOKEN", "message": "Token is invalid or already expired", "path": "/api/v1/auth/logout" }
```

---

### `POST /api/v1/auth/password-reset/request`
**Auth:** none (public).

**Request**
```json
{ "identifier": "jane.doe@acmebank.com", "organizationSlug": "acme-bank" }
```

**`202 Accepted`** — **always** returned with this exact generic message, even if no such account exists, to avoid leaking whether an email is registered
```json
{ "message": "If an account matches, a reset link has been sent." }
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "AUTH_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/auth/password-reset/request",
  "errors": [{ "field": "organizationSlug", "code": "REQUIRED", "message": "organizationSlug is required" }] }
```

---

### `POST /api/v1/auth/password-reset/confirm`
**Auth:** none — reset token carried in the body.

**Request**
```json
{ "resetToken": "a1b2c3d4-0000-4000-8000-000000000099", "newPassword": "NewPass!2027" }
```

**`204 No Content`** *(empty)* — internally calls `PATCH /internal/v1/users/{id}/password-hash` on User & Team Service.

**`400 Bad Request`**
```json
{ "status": 400, "code": "AUTH_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/auth/password-reset/confirm",
  "errors": [{ "field": "newPassword", "code": "TOO_WEAK", "message": "Password must be at least 8 characters" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_INVALID_RESET_TOKEN", "message": "Reset token is invalid or expired", "path": "/api/v1/auth/password-reset/confirm" }
```

---

### `GET /.well-known/jwks.json`
**Auth:** none (public). Published so every other service can verify JWT signatures locally.

**`200 OK`**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "2026-09-key-1",
      "alg": "RS256",
      "n": "sXch...base64url-modulus...",
      "e": "AQAB"
    }
  ]
}
```

---

## 5. Open Decisions Register (Tasks 1–4 Scope)

```text
[ ] Does `users` get a `version` column for optimistic locking on PATCH, matching `cases`? (Not required by SRS, but flagged above as a possible 409 case.)
[ ] Are team names unique per tenant? (Flagged as a possible 409 on team creation.)
[ ] Team deletion: hard delete vs. soft delete (is_archived) — recommend soft, for consistency with cases.
[ ] Organization name/slug uniqueness scope beyond the `slug` UNIQUE constraint.
[ ] Access-token TTL — no numeric value frozen anywhere (examples use 3600s/1hr).
[ ] Password-reset token TTL and whether it's single-use (recommended: yes, single-use, short TTL — not yet explicitly frozen).
```

---

*Scoped to Tasks 1–4 only, per request. Tasks 5–10 (RBAC, Case Management, Workflow, Document, Audit, Notification) remain covered in `Resolve_Service_Contracts_Consolidated.md`.*
