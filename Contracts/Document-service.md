# Resolve — Document Service: Corrected API Contract & Build Plan

**Service:** Document Service — Task 8
**Owns:** `documents`, `document_versions`, `outbox_events`; owns S3-compatible object storage
**Depends on (per `tasks.md`):** Tasks 2–5 (org context, permission checks). *(Note: `case_id` is a foreign key to `cases`, owned by Case Management/Task 6, which `tasks.md` does not list as a dependency — flagged as a real gap in §5.)*
**Nature:** Synchronous REST + Kafka producer, via the shared transactional outbox.
**Owned Kafka topics (produces):** `resolve.document.uploaded` now; `resolve.document.scanned` once virus scanning is implemented (Phase 3).

**This version corrects the first draft against the actual repo** (`Contracts/kafka.md`, `General/Schemas_High_Level.md`, `General/SRS.md`, `tasks.md`, `Contracts/Authentication_Service.md`, `Contracts/Organisation_Service.md`, `Contracts/RBAC_Service.md`). Where the repo's own docs disagree with each other, that's flagged explicitly rather than silently picking a side.

---

## 1. Shared Conventions (corrected)

**Base path:** `/api/v1` (public), `/internal/v1` (service-to-service only).

**Auth header:** `Authorization: Bearer {jwt}`.

**⚠️ Tenant claim name — unresolved conflict in Resolve's own docs:** `Contracts/Authentication_Service.md` §3 and `Contracts/Organisation_Service.md` both specify the JWT claim as **`tenantId`**. `Contracts/User_Team_Service.md`'s prose instead calls it "the `organization_id` claim." These can't both be the real field name. Since two of the three identity contracts and the entire Kafka envelope (`tenantId` throughout) agree, **this contract uses `tenantId`** — confirm against Authentication's actual token-issuing code before other services (including yours) build against the other name.

**Tenant scoping:** every request scoped by `tenantId`. A document belongs to a case (or, later, a comment), which belongs to an organization — so isolation is checked two hops deep once Case Management exists. A cross-tenant request returns `404`, not `403` (anti-enumeration, same rule as every other frozen contract).

**Pagination:** `?page=0&size=20&sort=createdAt,desc` → `{ "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 }`

**Standard error shape:**
```json
{
  "timestamp": "2026-09-26T10:30:00Z",
  "status": 400,
  "code": "DOCUMENT_VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/api/v1/cases/{caseId}/documents",
  "requestId": "req_9a2b3c",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [{ "field": "file", "code": "FILE_TOO_LARGE", "message": "file exceeds the maximum upload size" }]
}
```

**Idempotency:** `POST` upload endpoints require `Idempotency-Key`, same rule as User & Team Service.

**⚠️ Authorization model — genuinely undefined upstream.** `Contracts/RBAC_Service.md` is not a real contract; it's a gap-flag document stating outright that whether the permission check is a REST call or an embedded library, and its request/response shape, is **undefined** — and it names Document Service by name as blocked by this. The naming convention it *does* confirm (from `roles`/`permissions` examples) is `{RESOURCE}_{ACTION}` — e.g. `CASE_READ`, `CASE_APPROVE`. **This contract uses `DOCUMENT_READ`, `DOCUMENT_UPLOAD`, `DOCUMENT_MANAGE`** — not the `IAM_*` style the User & Team contract used, which appears to be that contract's own invention, not an RBAC-confirmed standard.

---

## 2. Endpoint Named Directly in `tasks.md`

### 2.1 `POST /api/v1/cases/{caseId}/documents`
Uploads a document (FR-DOC-01/02).

**Auth:** `DOCUMENT_UPLOAD` permission. **Idempotency-Key:** required.

**Request:** `multipart/form-data` — `file` (binary, required), `description` (optional, ≤1000 chars). `filename`, `mimeType`, `sizeBytes` are derived server-side from the actual upload, not trusted from client-supplied fields.

**201 Created**
```json
{
  "id": "b1e2c3d4-1111-4a1b-8c3d-5e6f7a8b9c0d",
  "caseId": "5c8e9f10-2222-4a1b-8c3d-5e6f7a8b9c0d",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "filename": "incident-report.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 482913,
  "checksum": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "scanStatus": "PENDING",
  "currentVersion": 1,
  "uploadedBy": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "createdAt": "2026-09-26T10:30:00Z"
}
```
*(no `updatedAt` — the frozen `documents` table doesn't have that column.)*

**400 Bad Request**
```json
{ "status": 400, "code": "DOCUMENT_FILE_TOO_LARGE", "message": "Request validation failed", "path": "/api/v1/cases/{caseId}/documents",
  "errors": [{ "field": "file", "code": "FILE_TOO_LARGE", "message": "file exceeds the maximum upload size" }] }
```
```json
{ "status": 400, "code": "DOCUMENT_BLOCKED_MIME_TYPE", "message": "Request validation failed", "path": "/api/v1/cases/{caseId}/documents",
  "errors": [{ "field": "mimeType", "code": "MIME_TYPE_BLOCKED", "message": "this file type is not permitted" }] }
```

**401 / 403 / 404** — `AUTH_MISSING_TOKEN` / `DOCUMENT_FORBIDDEN` / `CASE_NOT_FOUND`, same shapes as the User & Team contract.

---

### 2.2 `GET /api/v1/cases/{caseId}/documents`
Lists documents for a case (FR-DOC-03).

**Auth:** `DOCUMENT_READ`.

**Request:** `?filename=&mimeType=&page=0&size=20&sort=createdAt,desc`

**200 OK**
```json
{
  "content": [
    { "id": "b1e2c3d4-1111-4a1b-8c3d-5e6f7a8b9c0d", "caseId": "5c8e9f10-2222-4a1b-8c3d-5e6f7a8b9c0d",
      "filename": "incident-report.pdf", "mimeType": "application/pdf", "sizeBytes": 482913,
      "scanStatus": "PENDING", "currentVersion": 1, "uploadedBy": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "createdAt": "2026-09-26T10:30:00Z" }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

---

## 3. Gap-Fill Proposals

### 3.1 `GET /api/v1/cases/{caseId}/documents/{documentId}` — metadata
**Auth:** `DOCUMENT_READ`. Same shape as §2.1's `201` body.

### 3.2 `GET /api/v1/cases/{caseId}/documents/{documentId}/download`
**Auth:** `DOCUMENT_READ`. **Per BR-14: a document with `scanStatus` other than `CLEAN` is not downloadable** — return `403 DOCUMENT_NOT_YET_SCANNED` (or `INFECTED`/`FAILED` equivalents) instead of the file, *unless* your team has explicitly deferred virus scanning past this pass (see §5) — in which case document this exception, don't silently ignore BR-14. Otherwise: presigned GET URL, short expiry, `302` redirect.

### 3.3–3.5 Versioning (`POST/GET .../versions`, `GET .../versions/{n}/download`)
**Why:** `document_versions` exists structurally (FR-DOC-07), though it's a Phase-3/Should-have item per `Features_Phases.md` — see §5 on whether it's in scope for this pass. Request/response fields use `versionNumber` (matches the real column `version_number`, not `version`).

### 3.6 `GET /internal/v1/documents/{id}` — service-to-service lookup
Same structural need as User & Team's `GET /internal/v1/users/{id}`.

---

## 4. Storage & Event Contracts (corrected)

### 4.1 Object storage key — from `Schemas_High_Level.md` §4.4, verbatim
```
documents/{tenant_id}/{case_id}/{document_id}/{filename}
```
**⚠️ Open gap, not just mine:** this convention has no segment for version number, yet `document_versions.storage_key` must be a distinct key per version. The source docs don't resolve this either — pick one (e.g. append `/v{version_number}` before `{filename}`) and treat it as a team decision, not an already-settled fact.

### 4.2 Transactional outbox → Kafka (corrected to match `Contracts/kafka.md` exactly)

**`outbox_events`** columns (owned by Document, same shape Case Management/Workflow use):
`id`, `organization_id`, `aggregate_type` (`'DOCUMENT'`), `aggregate_id` (= `document_id` — **this is the Kafka partition key**, never `id`), `event_type` (PascalCase, e.g. `'DocumentUploaded'`), `event_version` (default `1`), `payload` (JSONB, the full envelope), `status` (`PENDING`/`PUBLISHED`/`FAILED`), `retry_count`, `available_at`, `created_at`, `published_at`.

**Publisher retry backoff (per `Contracts/kafka.md` §7, resolved):** base delay 1s, ×2 multiplier, cap 5 minutes; after 5 failed attempts route to `resolve.document.uploaded.dlq`.

**Common envelope — real shape, not the simplified one from the first draft:**
```json
{
  "eventId": "b2e1a4c0-1111-4a2b-9c3d-000000000016",
  "eventType": "DocumentUploaded",
  "eventVersion": 1,
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "actorId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "resourceType": "DOCUMENT",
  "resourceId": "e1f2a3b4-0000-4000-8000-000000000077",
  "timestamp": "2026-09-08T12:00:00Z",
  "metadata": {
    "documentId": "e1f2a3b4-0000-4000-8000-000000000077",
    "caseId": "c1a2b3d4-0000-4000-8000-000000000042",
    "storageKey": "documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/c1a2b3d4-0000-4000-8000-000000000042/e1f2a3b4-0000-4000-8000-000000000077/statement.pdf"
  }
}
```

**Second topic you'll eventually own — `resolve.document.scanned` (`DocumentScanned`, new, §4.10):** published once virus scanning reports a result, same envelope shape, `metadata: { documentId, caseId, scanStatus }`. Deferred until Phase 3 unless your team pulls it forward.

**Consumers of `resolve.document.uploaded`** (so you know who breaks if you change the shape): Audit, Virus Scanner Worker, OCR Worker (Could-have), Metadata Processor, Search Indexer.

---

## 5. Real Gaps to Flag to Your Team (not silently resolved here)

1. **Case Management dependency isn't listed.** `documents.case_id` FKs to `cases`, owned by Task 6 — but Task 8's stated dependency list is only "Tasks 2–5." No `Contracts/Case_Management_Service.md` exists yet either, so there's no internal lookup endpoint to call even if the dependency were listed. Practical effect: you cannot *really* validate `case_id` against another service yet — see §6's standalone strategy.
2. **RBAC's request/response shape is undefined** (§1 above) — same practical effect, handled in §6.
3. **Phase scope ambiguity:** `tasks.md`'s Task 8 checklist only lists FR-DOC-01–04. FR-DOC-05 (checksum), FR-DOC-06 (virus scan, BR-14), FR-DOC-07 (versioning) are `Features_Phases.md`'s next tier. But the frozen schema already requires `checksum`/`scan_status` columns. Recommendation: build the columns now (default `scan_status = 'PENDING'`), decide as a team whether scanning/versioning endpoints ship in this pass or the next.
4. **Version storage-key convention** — see §4.1.

---

## 6. Standalone Development Strategy — building without Tasks 2–6 actually running

Since Organization, RBAC, and Case Management either don't exist yet or aren't reachable from your machine, every external check is implemented as a small, clearly-labeled **stub module** — same shape as the interface you'll eventually call, so swapping in the real HTTP call later means editing one file, not every route.

| Real dependency | Stub for now | Swap-in later |
|---|---|---|
| Authentication verifies the JWT, issues `tenantId`/`userId` | `src/middleware/auth.js` reads a **fake, hardcoded** `{ tenantId, userId }` (same `permitAll`-style stopgap you're already using in user-team-service) | Replace the stub body with real JWT verification once Authentication is reachable |
| RBAC checks `DOCUMENT_UPLOAD`/`DOCUMENT_READ`/`DOCUMENT_MANAGE` | `src/services/permissionClient.js` — `checkPermission()` always returns `true` | Replace the function body with the real call once Task 5's shape is defined — every route already calls this function, not RBAC directly |
| Case Management confirms `case_id` exists and belongs to the tenant | `src/services/caseClient.js` — `assertCaseExists()` always resolves (optionally checks a hardcoded in-memory list of "known" fake case IDs for slightly more realistic 404 testing) | Replace with a real internal call to Case Management once it exists |

This means **all of Phase 1–2 (upload, list, metadata, download) can be built, run, and tested completely standalone right now** — nothing blocks on another team's service. Only real Kafka *consumers* (Audit, Search Indexer, etc.) and the real permission/case checks are deferred; your synchronous request/response paths are fully testable today.

---

## 7. Build Plan — Setup, Then 4 Phases

*(Setup and Docker Compose already done — Postgres on `5433`, SeaweedFS S3 on `8333`, Redpanda on `9092`.)*

### Corrected migration SQL
```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL,
  case_id UUID,
  comment_id UUID,
  uploaded_by UUID NOT NULL,
  filename VARCHAR(255) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  size_bytes BIGINT NOT NULL,
  checksum VARCHAR(128) NOT NULL,
  storage_key VARCHAR(500) NOT NULL,
  scan_status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
    CHECK (scan_status IN ('PENDING','CLEAN','INFECTED','FAILED')),
  current_version INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (case_id IS NOT NULL OR comment_id IS NOT NULL)
);

CREATE TABLE document_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  storage_key VARCHAR(500) NOT NULL,
  checksum VARCHAR(128) NOT NULL,
  size_bytes BIGINT NOT NULL,
  uploaded_by UUID NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (document_id, version_number)
);

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL,
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  event_version INTEGER NOT NULL DEFAULT 1,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','PUBLISHED','FAILED')),
  retry_count INTEGER NOT NULL DEFAULT 0,
  available_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ
);
```

### Phase 1 — Upload, fully standalone
Build §2.1 end to end using the three stubs from §6. Compute `checksum` (SHA-256 of the buffer) and `storage_key` (§4.1's convention) in code; insert `documents` + `document_versions` (version 1) + `outbox_events` in one transaction.
**Test:** everything from the original Phase-1 Postman checklist, plus: confirm `scanStatus` comes back `"PENDING"`, confirm `checksum` is a real SHA-256 hex string, confirm the outbox row's `payload` matches §4.2's envelope exactly.

### Phase 2 — Listing, metadata, download
Build §2.2, §3.1, §3.2 — including the BR-14 scan-status gate on download (decide with your team whether to enforce it yet, per §5 item 3, and document whichever you choose).

### Phase 3 — Versioning (if pulled into this pass) + checksum verification on read
Build §3.3–3.5. Add a re-hash-and-compare step on download to satisfy FR-DOC-05 if in scope.

### Phase 4 — Outbox relay + Kafka, using the corrected envelope
Same relay logic as before, but publish the §4.2 envelope shape, keyed by `aggregate_id` (= `documentId`), to `resolve.document.uploaded`.

---

**Compiled from** `Contracts/kafka.md`, `General/Schemas_High_Level.md`, `General/SRS.md`, `General/Features_Phases.md`, `tasks.md`, `Contracts/Authentication_Service.md`, `Contracts/Organisation_Service.md`, `Contracts/RBAC_Service.md`, and `Contracts/User_Team_Service.md`.
