# `approval-service` — Schema & Description

**Postgres schema:** `approval`
**Owns:** `approval.approvals`
**SRS coverage:** FR-APR-01…08

## Purpose

Approval is modeled as an explicit, auditable business object rather than an implicit side effect of a case status change. This service owns the full lifecycle of a request-to-decision cycle, including dynamic routing (which role must approve) driven by `workflow-service` rules.

## Depends On

- `organization.organizations` — tenant ownership.
- `case_management.cases` — the case the approval applies to.
- `identity.users` — requester and (once resolved) approver.

## Depended On By

- `case_management-service` blocks certain transitions until this service reports `status = APPROVED` for the relevant approval (BR-04).
- `workflow-service` creates rows here via `REQUIRE_APPROVAL` rule actions.

---

## Tables

### `approval.approvals`
Explicit, auditable approval requests and decisions (FR-APR-01…08).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `requested_by` | UUID | FK → `identity.users.id`, NOT NULL | — |
| `approver_id` | UUID | FK → `identity.users.id`, NULL | Specific assigned approver, once resolved to an individual. |
| `required_role` | VARCHAR(100) | NULL | Role/level required if not yet resolved to a specific user (dynamic routing, FR-APR-06, BR-05) — set by `workflow-service`. |
| `reason` | TEXT | NULL | — |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`APPROVED`,`REJECTED`,`EXPIRED`) | — |
| `decision_notes` | TEXT | NULL | Approver's rationale. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `expires_at` | TIMESTAMPTZ | NULL | Past this, an undecided approval is no longer actionable (FR-APR-05, BR-08). |
| `decision_at` | TIMESTAMPTZ | NULL | — |

> Multi-step approval chains and delegation (FR-APR-07/08, Could-have) can be layered on later via a `parent_approval_id` self-reference and a `delegated_to_user_id` column respectively — not added now, per "don't build mechanism before there's a demonstrated need."

---

## Business Rules Enforced Here

- **BR-04** — A case cannot reach `APPROVED`/`RESOLVED` unless every approval required by workflow rules has `status = APPROVED`.
- **BR-05** — Example rule this service's `required_role` field exists to satisfy: `IF case.amount > 10,00,000 THEN REQUIRE_APPROVAL('SENIOR_MANAGER')`.
- **BR-08** — An approval past `expires_at` without a decision becomes non-actionable; a scheduled job (owned by this service) transitions it to `EXPIRED`.

## Events Published (via `shared.outbox_events`)

```text
ApprovalRequested, ApprovalCompleted
```

## API Surface (owned endpoints)

```text
POST   /api/v1/cases/{caseId}/approvals
GET    /api/v1/cases/{caseId}/approvals
POST   /api/v1/approvals/{id}/decide
GET    /api/v1/approvals/pending           (for the current approver)
```
