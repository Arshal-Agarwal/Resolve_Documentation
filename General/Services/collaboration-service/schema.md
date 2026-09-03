# `collaboration-service` — Schema & Description

**Postgres schema:** `collaboration`
**Owns:** `collaboration.comments`
**SRS coverage:** FR-COL-01…06

## Purpose

Internal, case-scoped discussion — intentionally not a general-purpose chat product. Owns threaded comments and the parsing of `@mentions` that trigger notifications.

## Depends On

- `organization.organizations` — tenant ownership.
- `case_management.cases` — parent case.
- `identity.users` — author, and mentioned users.

## Depended On By

- `document-service` — `document.documents.comment_id` optionally points here for comment attachments.
- `notification-service` — consumes mention events to deliver notifications.

---

## Tables

### `collaboration.comments`
Internal, case-scoped discussion (FR-COL-01…06).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `author_id` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `parent_comment_id` | UUID | FK → `collaboration.comments.id`, NULL | Set for threaded replies (FR-COL-03); `NULL` for top-level comments. |
| `body` | TEXT | NOT NULL | May contain `@mentions` parsed at write time (FR-COL-04). |
| `is_edited` | BOOLEAN | NOT NULL, DEFAULT `false` | — |
| `edited_at` | TIMESTAMPTZ | NULL | Preserves an edit marker rather than silently overwriting history (FR-COL-05). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

---

## Business Rules Enforced Here

- Permission check before any write: caller must hold `COMMENT_CREATE`/`COMMENT_READ` (checked via `identity-service`), and the parent case must belong to the caller's tenant.

## Events Published (via `shared.outbox_events`)

```text
CommentCreated, CommentMentioned   (metadata: mentionedUserIds[])
```

## API Surface (owned endpoints)

```text
POST   /api/v1/cases/{caseId}/comments
GET    /api/v1/cases/{caseId}/comments
PATCH  /api/v1/comments/{id}
POST   /api/v1/comments/{id}/replies
```
