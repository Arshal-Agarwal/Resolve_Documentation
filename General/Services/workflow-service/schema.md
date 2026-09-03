# `workflow-service` — Schema & Description

**Postgres schema:** `workflow`
**Owns:** `workflow.workflow_definitions`, `workflow.workflow_instances`, `workflow.workflow_tasks`
**SRS coverage:** FR-WF-01…07, BR-05, BR-06, BR-07, BR-16

## Purpose

The rule engine. Owns declarative `IF <condition> THEN <action>` rule sets (system default and tenant-custom), and the execution log of every rule-triggered action, so re-delivered events never cause a rule to fire twice (idempotent execution, BR-16).

## Depends On

- `organization.organizations` — tenant ownership (nullable, for tenant-custom rules).
- `case_management.cases` — the case a rule is evaluated against.
- `task.tasks` — the task a `CREATE_TASK` action may have produced.

## Depended On By

Nothing depends on `workflow-service` structurally — it's a **consumer/orchestrator** that calls outward into `task-service` (create task), `approval-service` (require approval), and `notification-service` (send notification) in reaction to events published by `case-management-service` and others.

---

## Tables

### `workflow.workflow_definitions`
Declarative `IF <condition> THEN <action>` rule sets (FR-WF-01…07).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `organization_id` | UUID | FK → `organization.organizations.id`, NULL | `NULL` = platform default ruleset; non-null = tenant-custom rules (FR-WF-05). |
| `name` | VARCHAR(255) | NOT NULL | — |
| `description` | TEXT | NULL | — |
| `rules` | JSONB | NOT NULL | E.g. `[{"if": "case.amount > 1000000", "then": "REQUIRE_APPROVAL(SENIOR_MANAGER)"}]` (FR-WF-03). |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT `true` | Inactive definitions are not evaluated. |
| `version` | INTEGER | NOT NULL, DEFAULT 1 | Incremented on rule changes, for traceability. |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |

### `workflow.workflow_instances`
Runtime progress of a definition against a specific case (FR-WF-06, Could-have).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `workflow_definition_id` | UUID | FK → `workflow.workflow_definitions.id`, NOT NULL | — |
| `case_id` | UUID | FK → `case_management.cases.id`, NOT NULL | — |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'RUNNING'`, CHECK IN (`RUNNING`,`COMPLETED`,`FAILED`) | — |
| `context` | JSONB | NULL | Snapshot of relevant case data at evaluation time, for debugging/replay. |
| `started_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | — |
| `completed_at` | TIMESTAMPTZ | NULL | — |

### `workflow.workflow_tasks`
The execution log of individual rule-triggered actions — **not** a duplicate of `task.tasks`. Satisfies both action-tracking (FR-WF-04, BR-16) and, via `resulting_task_id`, traceability between a workflow action and the case task it produced.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PK | — |
| `workflow_instance_id` | UUID | FK → `workflow.workflow_instances.id`, NOT NULL | — |
| `action_type` | VARCHAR(50) | NOT NULL, CHECK IN (`REQUIRE_APPROVAL`,`CREATE_TASK`,`SEND_NOTIFICATION`) | — |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `'PENDING'`, CHECK IN (`PENDING`,`EXECUTED`,`FAILED`) | — |
| `payload` | JSONB | NULL | Action-specific parameters (e.g., which task template, which approver role). |
| `resulting_task_id` | UUID | FK → `task.tasks.id`, NULL | Set only when `action_type = 'CREATE_TASK'` and the task was actually created. |
| `executed_at` | TIMESTAMPTZ | NULL | — |
| `idempotency_key` | VARCHAR(255) | NOT NULL | Derived from the triggering event ID, so a redelivered event doesn't re-execute the action (FR-EVT-04, BR-16). |
| *(unique)* | — | UNIQUE(`workflow_instance_id`, `idempotency_key`) | Enforces exactly-once execution at the DB level. |

---

## Business Rules Enforced Here

- **BR-05** — Amount-threshold approval routing example rule.
- **BR-06** — Auto-create a "Senior Investigation" task when a case's priority is `HIGH`.
- **BR-07** — Auto-notify the creator and assignee when a case reaches `RESOLVED`.
- **BR-16** — A workflow rule's action executes at most once per triggering event, even if the event is redelivered.

## Events Consumed

```text
CaseCreated, CaseStatusChanged   (from case_management)
```

## Events Published (via `shared.outbox_events`)

```text
WorkflowRuleTriggered, WorkflowActionExecuted
```

## API Surface (owned endpoints)

```text
GET    /api/v1/workflows
POST   /api/v1/workflows                     (tenant-custom rule authoring)
GET    /api/v1/cases/{caseId}/workflow-instances
```
