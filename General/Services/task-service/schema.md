# `task-service` — Schema & Description

**Postgres schema:** `task`
**Owns:** `task.tasks`, `task.task_dependencies`
**SRS coverage:** FR-TASK-01…08

## Purpose

Owns units of work performed under a case. Tasks can gate case-stage advancement (`is_required_for_stage`) and can depend on one another. Tasks may be created by a human or by `workflow-service` (via a `CREATE_TASK` rule action).

## Depends On

- `organization.organizations` — tenant ownership.
- `case_management.cases` — parent case.
- `identity.users` — assignee/creator.
- `organization.teams` — team assignee.

## Depended On By

- `case_management-service` reads task completion state to decide whether a stage can advance (BR-03).
- `workflow-service` creates rows here via `CREATE_TASK` actions and records the resulting `task.tasks.id` back on its own `workflow.workflow_tasks.resulting_task_id`.

---

## Tables

### `task.tasks`
Units of work under a case (FR-TASK-01…08).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NOT NULL | Denormalized for tenant-scoped queries without joining through `case_management.cases`. |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `title` | VARCHAR(255) | NOT NULL | — |
| `description` | TEXT | NULL | — |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`IN_PROGRESS`,`COMPLETED`,`CANCELLED`) | — |
| `priority` | VARCHAR(20) | NOT NULL, DEFAULT `'MEDIUM'`, CHECK IN (`LOW`,`MEDIUM`,`HIGH`) | — |
| `is_required_for_stage` | BOOLEAN | NOT NULL, DEFAULT `false` | If `true`, blocks case-stage advancement until complete (FR-TASK-06, BR-03). |
| `assigned_to_user_id` | UUID | FK → `identity.users.id`, NULL | — |
| `assigned_to_team_id` | UUID | FK → `organization.teams.id`, NULL | — |
| `due_date` | TIMESTAMPTZ | NULL | — |
| `completed_at` | TIMESTAMPTZ | NULL | Set on completion (FR-TASK-03). |
| `created_by` | UUID | FK → `identity.users.id`, NOT NULL | May reference a system/service user for workflow-generated tasks (BR-06). |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

> Recurring/templated tasks (FR-TASK-07, Could-have) don't require a schema change to start — a workflow rule can issue repeated `CREATE_TASK` actions. Add a nullable `template_id` self-reference only if templating becomes a demonstrated need.

### `task.task_dependencies`
Task-precedes-task relationships (FR-TASK-05).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `task_id` | UUID | FK → `task.tasks.id`, PK (part 1) | The dependent task. |
| `depends_on_task_id` | UUID | FK → `task.tasks.id`, PK (part 2) | Must reach `COMPLETED` before `task_id` can complete (app-enforced; no self-reference). |

---

## Business Rules Enforced Here

- **BR-03** — Exposes a "does this case have incomplete required tasks?" check that `case-management-service` calls before allowing certain transitions.
- **BR-06** — Consumed target of the workflow rule: "if case priority is `HIGH`, create a Senior Investigation task."

## Events Published (via `shared.outbox_events`)

```text
TaskCreated, TaskCompleted
```

## API Surface (owned endpoints)

```text
POST   /api/v1/cases/{caseId}/tasks
GET    /api/v1/cases/{caseId}/tasks
PATCH  /api/v1/tasks/{id}
POST   /api/v1/tasks/{id}/complete
POST   /api/v1/tasks/{id}/dependencies
```
