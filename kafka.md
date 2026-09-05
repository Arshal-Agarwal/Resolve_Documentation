Here's the producer/consumer map for every topic, plus who owns topic *creation* (that's an infra concern, separate from who publishes/subscribes to it).

## Topic Ownership

**Topic creation itself belongs to Task 1 (Shared Infrastructure)** — provisioning topics, setting partition counts, and configuring DLQs is a one-time setup step done by the team together, not something each service owner does independently. Once topics exist, each service's *producer* code is responsible for that topic's schema staying correct (per the Task 0 contract).

## Producer → Consumer Map

| Topic | Producer (writes to it) | Consumer(s) (read from it) | Why they need it |
|---|---|---|---|
| `resolve.case.created` | Case Management (Task 6) | Audit (Task 9) | Audit trail entry |
| `resolve.case.assigned` | Case Management (Task 6) | Workflow (Task 7), Notification (Task 10), Audit (Task 9) | Workflow: may trigger rules on assignment. Notification: "case assigned" alert (FR-NOT-01). Audit: trail entry |
| `resolve.case.statuschanged` | Case Management (Task 6) | Workflow (Task 7), Audit (Task 9) | Workflow: this is the main trigger for rule evaluation — e.g. entering `REVIEW` may require checking/creating an approval. Audit: trail entry |
| `resolve.case.resolved` | Case Management (Task 6) | Notification (Task 10), Audit (Task 9) | Notification: "case resolved" alert to creator/assignee (FR-NOT-02). Audit: trail entry |
| `resolve.task.created` | Workflow (Task 7) | Notification (Task 10), Audit (Task 9) | Notification: task-assignment alert, since there's no separate `task.assigned` topic — assignee is carried in this event's payload. Audit: trail entry |
| `resolve.task.completed` | Workflow (Task 7) | Audit (Task 9) | Trail entry only — Case Management checks task completion via a **synchronous** call, not this event, since it needs the answer immediately during a transition, not eventually |
| `resolve.document.uploaded` | Document (Task 8) | Audit (Task 9) | Trail entry. *(Not in the current 9-service list, but worth flagging: your SRS also names a Virus Scanner Worker as a consumer of this exact event — if you want async virus scanning, that's a 10th consumer to add, either as its own tiny service or a worker inside Document Service)* |
| `resolve.approval.requested` | Workflow (Task 7) | Notification (Task 10), Audit (Task 9) | Notification: "approval needed" alert (FR-NOT-02). Audit: trail entry |
| `resolve.approval.completed` | Workflow (Task 7) | Audit (Task 9) | Trail entry only — FR-NOT-02 only requires notifying on the *request*, not the decision. Case Management again checks the decision synchronously when gating a transition, not via this event. |

## The Pattern, Simplified

- **Producers:** Case Management, Workflow, Document — exactly the 3 services that own a business entity with a lifecycle. Each only produces events for its own aggregate.
- **Consumers:**
  - **Audit subscribes to literally everything** — every topic above lists it. That's the "universal subscriber" shape from before.
  - **Notification subscribes selectively** — only the events a human actually needs to be told about (assignment, approval-requested, resolution).
  - **Workflow is the one service that's both** — it produces its own events (task/approval) *and* consumes `case.*` to know when to evaluate rules.
  - **RBAC, Auth, Organization, User, Case (as a consumer)** — none of them touch Kafka at all. They're purely synchronous.

One thing worth deciding explicitly in Task 0 rather than leaving implicit: whether `resolve.task.completed` and `resolve.approval.completed` staying **Audit-only** is intentional, or whether you actually want Notification to alert on those too (e.g., "your approval request was approved/rejected" is a pretty natural notification most tools send, even though the current FR-NOT items don't strictly require it).