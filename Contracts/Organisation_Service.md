# Resolve — Organization Service: Exhaustive API Contract Reference

**Service:** Organization (Tenant) Service — Task 2
**Owns:** `organizations` table only. **Depends on:** nothing (root service; every other tenant-owned table foreign-keys to `organizations.id`).
**Nature:** Synchronous REST only. Publishes **no** Kafka events in the initial scope.
**Sources used:** `General/SRS.md` (§1, §2.3, §2.4, §5, §7, §8), `General/Schemas_High_Level.md` (§2.1 `organizations`), `Contracts/Task1-4.md` (§0–§2), `tasks.md` (Task 2).

> **How this document is organized:** Section 2 is the *frozen* contract — exactly what `Contracts/Task1-4.md` already specifies, reproduced in full with every status code. Section 3 adds endpoints that the SRS/schema/architecture imply are needed but that the frozen contract doesn't yet cover (list, update, internal slug resolution) — each is flagged as a **gap-fill recommendation**, not an already-agreed contract, so your team can ratify or reject them in Task 0. Section 4 documents what's deliberately **not** part of this service, so the surface stays bounded on purpose rather than by omission.

---

## 1. Shared Conventions (apply to every endpoint below)

**Base path:** `/api/v1/organizations` (public), `/internal/v1/organizations` (service-to-service only, never exposed through the gateway).

**Auth header:** `Authorization: Bearer {jwt}` on every public endpoint — the Organization Service has no unauthenticated endpoints (unlike Auth's login/refresh).

**Internal auth:** mTLS client cert or a signed service-to-service JWT carrying a `service` claim. The caller's forwarded end-user token is never used for internal calls.

**Pagination** (list endpoints):
Request: `?page=0&size=20&sort=createdAt,desc`
Response:
```json
{ "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 }
```

**Standard error shape** (every non-2xx response unless noted otherwise):
```json
{
  "timestamp": "2026-09-05T10:30:00Z",
  "status": 400,
  "code": "ORG_VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/api/v1/organizations",
  "requestId": "req_9f2a1c",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "field": "slug", "code": "INVALID_FORMAT", "message": "Slug must be lowercase alphanumeric with hyphens" }
  ]
}
```
`errors[]` only appears on `400` field-validation failures. All `code` values are namespaced `ORG_*`. Below, each endpoint's non-2xx examples only show the fields that change; assume `timestamp`, `requestId`, `traceId` are always present.

**Idempotency:** `POST /api/v1/organizations` is a resource-creating `POST` and requires an `Idempotency-Key` header. A retry with the same key returns the original response verbatim; a retry with the same key but a different body returns:
```json
{ "status": 409, "code": "IDEMPOTENCY_KEY_REUSED", "message": "Idempotency-Key was already used with a different request body" }
```

**Authorization model note:** every endpoint in this service is gated on the **System Administrator** actor (SRS §1.1) — a platform-level role not scoped to any tenant — rather than an org-scoped permission code like `ORG_MANAGE`. That's intentional: nothing in the permission catalog (`roles`/`permissions`/`role_permissions`) is tenant-scoped enough to *create* a tenant in the first place. `GET /api/v1/organizations/{id}` is the one exception that also accepts a caller who is simply a member of that org.

**Events:** none. `Contracts/Task1-4.md` §1 states explicitly that none of Tasks 2–4 publish Kafka events in the initial scope, and the SRS's domain-event catalog (FR-EVT-01) lists only Case/Task/Document/Approval events — no `OrganizationCreated`/`OrganizationSuspended` events exist today. If a downstream consumer later needs to react to tenant suspension, that's a new FR, not something silently already covered.

---

## 2. Canonical Endpoints (frozen — from `Contracts/Task1-4.md`)

### 2.1 `POST /api/v1/organizations`
Provisions a new tenant.

**Auth:** System Administrator role.
**Idempotency-Key:** required.

**Request**
```json
{ "name": "Acme Bank", "slug": "acme-bank" }
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | required, ≤255 chars |
| `slug` | string | required, unique globally, lowercase alphanumeric + hyphens |

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

### 2.2 `GET /api/v1/organizations/{id}`
Retrieves a single tenant.

**Auth:** System Administrator, **or** an authenticated user whose JWT `tenant_id` claim matches `{id}`.

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

### 2.3 `PATCH /api/v1/organizations/{id}/status`
Suspends or reactivates a tenant (platform-level lifecycle control, SRS §2.1 `organizations.status`).

**Auth:** System Administrator role.

**Request**
```json
{ "status": "SUSPENDED" }
```
`status` must be one of `ACTIVE`, `SUSPENDED`.

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

**Downstream effect (not this service's job, but relevant):** a `SUSPENDED` org should cause Authentication to reject logins and existing sessions for its users. Nothing in the current contracts wires that up automatically since no event is published — flagged again in §3.3 below.

---

## 3. Gap-Fill Recommendations (not yet in the frozen contract)

The SRS models `organizations` as a full CRUD-capable entity with a list-friendly admin surface (FR-ORG-01, NFR-PERF-03's pagination requirement, the System Administrator actor's "view platform health" responsibility), but the frozen Task 1–4 contract only specifies create/get-one/set-status. The four gaps below are what a System Administrator realistically needs and what other services structurally require; propose these in your Task 0 sign-off rather than treating the list above as exhaustive on its own.

### 3.1 `GET /api/v1/organizations` — list/search tenants
**Why:** the System Administrator actor is described as managing tenants "at the infrastructure level" (SRS §1.1) and viewing "platform health" — that's not possible with only get-by-id. Every other list endpoint in the frozen contracts (`GET /users`, `GET /teams`) follows the same pagination convention; organizations shouldn't be the one resource you can only look up one at a time.

**Auth:** System Administrator role only (not "member of org" — a tenant admin has no business browsing other tenants).

**Request:** `?name=&slug=&status=&page=0&size=20&sort=createdAt,desc`

**`200 OK`**
```json
{
  "content": [
    { "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "name": "Acme Bank", "slug": "acme-bank", "status": "ACTIVE",
      "createdAt": "2026-09-05T10:30:00Z", "updatedAt": "2026-09-05T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/organizations" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "ORG_FORBIDDEN", "message": "Only System Administrators may list organizations", "path": "/api/v1/organizations" }
```

---

### 3.2 `PATCH /api/v1/organizations/{id}` — update profile fields
**Why:** the status-only PATCH in §2.3 has no counterpart for correcting `name` or `slug` (a typo at provisioning time, a tenant rebrand). Without this, the only fix is direct DB access, which conflicts with FR-QA-05 (migrations-only schema changes still allow data fixes via API, but there's currently no API path at all).

**Auth:** System Administrator role. *(Open decision: could `slug` changes be blocked or require extra confirmation, since the slug is embedded in login flows and possibly cached client-side? Recommend allowing but documenting the disruption.)*

**Request** (partial update — send only fields to change)
```json
{ "name": "Acme Banking Corp" }
```

**`200 OK`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Acme Banking Corp",
  "slug": "acme-bank",
  "status": "ACTIVE",
  "createdAt": "2026-09-05T10:30:00Z",
  "updatedAt": "2026-09-05T12:00:00Z"
}
```

**`400 Bad Request`**
```json
{ "status": 400, "code": "ORG_VALIDATION_ERROR", "message": "Request validation failed", "path": "/api/v1/organizations/{id}",
  "errors": [{ "field": "slug", "code": "INVALID_FORMAT", "message": "Slug must be lowercase alphanumeric with hyphens" }] }
```

**`401 Unauthorized`**
```json
{ "status": 401, "code": "AUTH_MISSING_TOKEN", "message": "Authentication required", "path": "/api/v1/organizations/{id}" }
```

**`403 Forbidden`**
```json
{ "status": 403, "code": "ORG_FORBIDDEN", "message": "Only System Administrators may update organization details", "path": "/api/v1/organizations/{id}" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "ORG_NOT_FOUND", "message": "Organization not found", "path": "/api/v1/organizations/{id}" }
```

**`409 Conflict`**
```json
{ "status": 409, "code": "ORG_SLUG_TAKEN", "message": "An organization with this slug already exists", "path": "/api/v1/organizations/{id}" }
```

---

### 3.3 `GET /internal/v1/organizations/{id}` — service-to-service lookup
**Why:** any service that writes an `organization_id` foreign key (User & Team on user creation, Case Management on case creation, etc.) needs to confirm the tenant exists and is `ACTIVE` before writing — otherwise `organization_id` referential integrity relies purely on the DB FK constraint with no pre-check, and a request against a `SUSPENDED` tenant has no defined rejection path anywhere in the current contracts (see the note at the end of §2.3).

**Auth:** service identity (mTLS/service JWT).

**`200 OK`**
```json
{ "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "name": "Acme Bank", "slug": "acme-bank", "status": "ACTIVE" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "ORG_NOT_FOUND", "message": "Organization not found", "path": "/internal/v1/organizations/{id}" }
```

---

### 3.4 `GET /internal/v1/organizations/by-slug/{slug}` — service-to-service slug resolution
**Why:** this is the clearest actual gap. Authentication's `POST /api/v1/auth/login` (`Contracts/Task1-4.md` §4) accepts `organizationSlug` and resolves it by calling **User & Team's** `GET /internal/v1/users/by-email?email=&organizationSlug=` — meaning User & Team is silently doing slug→tenant resolution today, even though it doesn't own `organizations`. That's a hidden cross-service coupling the Open Decisions register (§5 of `Contracts/Task1-4.md`) doesn't call out. The clean fix is for Authentication (and/or User & Team) to resolve the slug via the Organization Service directly, and reject login early with a clear error if the tenant is `SUSPENDED` — instead of that check happening nowhere.

**Auth:** service identity (mTLS/service JWT).

**`200 OK`**
```json
{ "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "name": "Acme Bank", "slug": "acme-bank", "status": "ACTIVE" }
```

**`404 Not Found`**
```json
{ "status": 404, "code": "ORG_NOT_FOUND", "message": "Organization not found", "path": "/internal/v1/organizations/by-slug/{slug}" }
```

---

## 4. Explicitly Out of Scope

Called out so the surface is bounded by decision, not by oversight:

| Not provided | Why |
|---|---|
| `DELETE /api/v1/organizations/{id}` (hard delete) | The schema gives `organizations` only `ACTIVE`/`SUSPENDED` — there's no `is_archived` column here (unlike `cases`, FR-CASE-14). Tenant "removal" is modeled as `SUSPENDED`, not deletion, so historical data for a suspended tenant is preserved by construction. |
| Public self-service org signup (unauthenticated `POST`) | SRS §1.1 frames tenant provisioning as a System Administrator action ("Provision/suspend organizations"), not open registration — consistent with FR-ORG-01's "onboarding" framing rather than a public signup flow. |
| Org-level settings/branding/billing endpoints | SRS §5.6 explicitly excludes billing/payments this release; no FR describes tenant branding or configurable settings beyond `name`/`slug`/`status`. |
| Tenant-editable workflow rule configuration via this service | Workflow rules (FR-WF-*) belong to the Workflow & Work Execution Service (`workflow_definitions`), not `organizations` — this service only owns the tenant boundary itself. |
| Schema-per-tenant provisioning options | FR-TEN-07 explicitly marks schema-per-tenant as Won't-Have this release; all tenants share one schema, isolated by `organization_id`. |

---

## 5. Data Owned

`organizations` (from `Schemas_High_Level.md` §2.1) — the only table this service owns or writes to.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | Tenant identifier, referenced by every tenant-owned table across all services. |
| `name` | VARCHAR(255) | NOT NULL | Display name. |
| `slug` | VARCHAR(100) | UNIQUE, NOT NULL | URL-safe identifier; used for login-time tenant disambiguation. |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'`, CHECK IN (`ACTIVE`,`SUSPENDED`) | Platform-level lifecycle state. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

No `version` column — optimistic locking is scoped only to `cases` per the schema doc's merge notes (§0), and nothing in the FRs identifies concurrent organization updates as a contention risk worth the extra column.

---

## 6. Error Code Catalog (this service)

| Code | HTTP Status | Meaning |
|---|---|---|
| `ORG_VALIDATION_ERROR` | 400 | Request body failed field validation. |
| `ORG_INVALID_STATUS` | 400 | `status` value outside `ACTIVE`/`SUSPENDED`. |
| `AUTH_MISSING_TOKEN` | 401 | No/invalid bearer token (shared platform-wide code). |
| `ORG_FORBIDDEN` | 403 | Caller lacks System Administrator role, or isn't a member of the requested org. |
| `ORG_NOT_FOUND` | 404 | No organization with the given `id`/`slug`. |
| `ORG_SLUG_TAKEN` | 409 | `slug` already in use by another organization. |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Same `Idempotency-Key`, different body, on a retried `POST`. |

---

## 7. Requirement Traceability

| Endpoint | Traces to |
|---|---|
| `POST /api/v1/organizations` | FR-ORG-01, NFR-SCAL-03 (baseline: 5+ tenants) |
| `GET /api/v1/organizations/{id}` | FR-ORG-01, FR-TEN-03 (cross-tenant 403/404), NFR-SEC-02 |
| `PATCH /api/v1/organizations/{id}/status` | SRS §2.1 actor table ("Provision/suspend organizations") |
| `GET /api/v1/organizations` *(§3.1, proposed)* | FR-ORG-01, NFR-PERF-03 (pagination on list endpoints) |
| `PATCH /api/v1/organizations/{id}` *(§3.2, proposed)* | FR-ORG-01 (onboarding correction), NFR-DATA-03 (no manual DDL fixes) |
| `GET /internal/v1/organizations/{id}` *(§3.3, proposed)* | FR-TEN-01/02 (every tenant-owned write needs a valid, active tenant), NFR-SEC-02 |
| `GET /internal/v1/organizations/by-slug/{slug}` *(§3.4, proposed)* | FR-IAM-02 (login), closes the hidden coupling noted in `Contracts/Task1-4.md` §5 open decisions |

---

*Compiled from `Resolve_Documentation/{General/SRS.md, General/Schemas_High_Level.md, Contracts/Task1-4.md, tasks.md}`. Section 2 is frozen and should not be reinterpreted; Section 3 requires Task 0 sign-off before implementation.*
