# RBAC (Access Control) Service — Documentation Gap Flag

> ⚠️ **This is not a contract.** This file exists only to make a real gap visible and trackable, not to invent RBAC Service's API on its owning developer's behalf. Every "Undefined" entry below is genuinely undefined in the current documentation set — nothing here should be treated as agreed or implementable until whoever owns Task 5 writes the real contract (ideally in the same format as `Authentication_Service.md`, `Organisation_Service.md`, and `User_Team_Service.md`) and it goes through Task 0 sign-off.

**Service:** RBAC (Access Control) Service — `tasks.md` Task 5
**Status:** **No `Contracts/RBAC_Service.md` or `Schemas/*_RBAC_Service_Schema.md` exists.** This is the only one of the three identity-cluster services (Organization, User & Team, Authentication, RBAC) with zero dedicated documentation, despite being described in `tasks.md` as "called on the hot path of nearly every write across every other service."

---

## 1. What's already implied elsewhere (safe to treat as settled)

Pulled from cross-references in other services' contracts and `tasks.md` — these are the only RBAC-related facts that currently have any backing:

- **Owned tables** (`tasks.md` Task 5): `roles`, `permissions`, `user_roles`, `role_permissions`.
- **Model:** atomic permission model (FR-AUZ-01) — roles are named collections of discrete permissions (e.g., `CASE_READ`, `CASE_APPROVE`). Many-to-many both ways: Users↔Roles and Roles↔Permissions (FR-AUZ-04).
- **Evaluation:** centralized, deny-by-default (FR-AUZ-02/03) — authorization logic lives in one place, not scattered across every service's controllers.
- **Nature:** synchronous only, no Kafka involvement (`Contracts/kafka.md` §32 explicitly confirms this).
- **Dependency direction:** depends only on Task 3 (User & Team Service) existing — roles/permissions attach to users that already exist. Can be built in parallel with Authentication (Task 4).

## 2. What's genuinely undefined

| Question | Why it matters | Who's currently blocked by it |
|---|---|---|
| **Is the permission check a REST endpoint or an embedded library?** `tasks.md` Task 5 explicitly leaves this open ("define the exact shape — REST endpoint or embedded library — in Task 0"). | Every other service's authorization code depends on knowing whether it makes a network call or imports a package. | Case Management, Workflow, Document services (all listed as calling RBAC "on the hot path of nearly every write") |
| **Exact request/response shape of the permission check**, whichever form it takes (e.g., `POST /internal/v1/authz/check { userId, permission, resourceId? }` → `{ allowed: boolean }`, or some other shape entirely) | Nobody can write a client against an undefined shape | Same as above |
| **Org Admin permission code for Authentication's token-settings endpoint** — `Contracts/Authentication_Service.md` §10 currently says "`ROLE_MANAGE` or a dedicated `AUTH_SETTINGS_MANAGE` permission — confirm with RBAC's permission catalog in Task 0," i.e., it's guessing at a name that may not exist | `GET`/`PATCH /api/v1/auth/token-settings` cannot be built until this exists — currently blocked, explicitly, in the Authentication Service's own README | Authentication Service (Phase 4 item, currently on hold for this reason) |
| **Custom tenant roles** (FR-AUZ-05, Phase 3, Should-have) — how org-scoped custom roles interact with the presumably-global `roles`/`permissions` tables | Affects the owned-table design itself, not just an endpoint | RBAC Service's own implementation |
| **Caching contract** — `Authentication_Service.md` §4 mentions permissions are "queried (with local caching)" at authorization time, implying callers cache RBAC responses locally, but no TTL, invalidation strategy, or cache-busting-on-permission-change mechanism is documented anywhere | A revoked permission could remain effective for an undefined amount of time if caching is added without an invalidation story | Every calling service |

## 3. Recommended next step

Whoever is assigned Task 5 should write `Contracts/RBAC_Service.md` in the same structure as the three existing service contracts (shared conventions, canonical endpoints with every status code, data owned, error catalog, requirement traceability) — using this file's §2 table as the starting checklist of decisions that specifically need to be made, not skipped past. Once that contract exists, this file should be deleted; it's a placeholder, not a permanent fixture.
