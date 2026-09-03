# `document-service` — Schema & Description

**Postgres schema:** `document`
**Owns:** `document.documents`, `document.document_versions`
**SRS coverage:** FR-DOC-01…10

## Purpose

Owns file **metadata** for anything uploaded to a case or attached to a comment. Binary bytes are never stored here — they live in S3-compatible object storage, referenced by `storage_key`. This service also owns the virus-scan status gate and version history for re-uploads.

## Depends On

- `organization.organizations` — tenant ownership.
- `case_management.cases` — parent case (nullable, see below).
- `collaboration.comments` — parent comment, for comment attachments (nullable).
- `identity.users` — uploader.
- **S3-compatible object storage** (external, not Postgres) — actual file bytes.

## Depended On By

- `workflow-service` may trigger on `DocumentUploaded`.
- `search` projection (OpenSearch, outside this schema) is fed from this service's events.

---

## Tables

### `document.documents`
File metadata; binary content lives in object storage (FR-DOC-01…10).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NULL | `NULL` when the document is attached directly to a comment instead. |
| `comment_id` | UUID | FK → `collaboration.comments.id`, NULL | Set for comment attachments (FR-COL-06, Could-have); `NULL` for case-level documents. |
| `uploaded_by` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `filename` | VARCHAR(255) | NOT NULL | Original filename as uploaded. |
| `mime_type` | VARCHAR(100) | NOT NULL | Validated against an allow-list at upload (FR-API-06). |
| `size_bytes` | BIGINT | NOT NULL | Validated against the max-upload-size limit (FR-API-06). |
| `checksum` | VARCHAR(128) | NOT NULL | SHA-256 of file content, detects corruption/tampering (FR-DOC-05). |
| `storage_key` | VARCHAR(500) | NOT NULL | Object key in S3-compatible storage — bytes never touch PostgreSQL (FR-DOC-02). Format: `documents/{tenant_id}/{case_id}/{document_id}/{filename}`. |
| `scan_status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`CLEAN`,`INFECTED`,`FAILED`) | Set by the async Virus Scanner Worker; only `CLEAN` documents are generally downloadable (BR-14). |
| `current_version` | INTEGER | NOT NULL, DEFAULT 1 | Points to the latest row in `document_versions`. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| *(check)* | — | `CHECK (case_id IS NOT NULL OR comment_id IS NOT NULL)` | A document must belong to something. |

### `document.document_versions`
Version history for re-uploaded documents (FR-DOC-07).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `document_id` | UUID | FK → `document.documents.id`, NOT NULL | — |
| `version_number` | INTEGER | NOT NULL | Sequential per document. |
| `storage_key` | VARCHAR(500) | NOT NULL | Object key for this version's bytes. |
| `checksum` | VARCHAR(128) | NOT NULL | — |
| `size_bytes` | BIGINT | NOT NULL | — |
| `uploaded_by` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| *(unique)* | — | UNIQUE(`document_id`, `version_number`) | — |

> OCR-extracted text (FR-DOC-08, Could-have) is deliberately **not** a column here — it flows straight into the OpenSearch projection instead of bloating the relational row.

---

## Business Rules Enforced Here

- **BR-14** — A document is not available for general download until `scan_status = 'CLEAN'`.

## Events Published (via `shared.outbox_events`)

```text
DocumentUploaded
```

## Non-Relational Dependency

**S3-compatible object storage** is the actual file store, keyed as:
```text
documents/{tenant_id}/{case_id}/{document_id}/{filename}
```
Tenant-scoped prefix is non-negotiable — it keeps isolation visible at the storage layer, not just in SQL.

## API Surface (owned endpoints)

```text
POST   /api/v1/cases/{caseId}/documents
GET    /api/v1/cases/{caseId}/documents
GET    /api/v1/documents/{id}/download
POST   /api/v1/documents/{id}/versions
```
