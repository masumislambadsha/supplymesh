# ADR-005: Approval workflows stored as versioned relational rules

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-006 PO/invoice consistency](ADR-006-po-invoice-consistency.md) · [ADR-008 Background jobs](ADR-008-background-jobs.md) · [ADR-010 Auth & RBAC](ADR-010-auth-and-rbac.md) · [Database design §4.4–4.5](../02-database/database-design.md) · [System design](../04-system-design/system-design.md)

## Context (the problem)

Each organization defines its own approval policy for purchase requests. The brief's example: under $500 needs the Department Manager, $500–5,000 needs the Procurement Manager, over $5,000 needs multiple approvals. Real customers will also want rules by department or category, parallel approvals, quorums ("2 of the Finance Officers") and SLA escalation.

Two things make this hard. First, policies change while requests are in flight: an Owner may lower a threshold on Tuesday while 40 requests are waiting. Second, auditors later ask "which rule approved PR-2026-000123, and who signed it?". We need to decide where and how the policy is stored, and how a running request relates to it.

## Decision drivers

- Organization Owners and Procurement Managers configure policies in the UI, without a deployment.
- A policy edit must not change the routing of requests already submitted.
- The audit trail must show which policy version and which rule applied to each request.
- References in a policy (roles, departments, categories, members) must stay valid; deleting a role used by an active policy must fail.
- The "My approvals" inbox is the most frequently used query and must be one indexed SQL query.
- Approver checks need `member_roles.approval_limit_amount` and segregation of duties (requester ≠ approver).

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| (a) Hard-coded if/else | `if (amount < 500) deptManager() else if …` in the approval service | Fastest to write; easy to test | Every customer change is a deploy; one policy for all tenants contradicts "configurable per organization"; no versioning |
| (b) JSON workflow definition | `approval_workflows.definition JSONB` holding rules and steps; versioned by copying the blob | Simple to save from a UI editor; flexible shape; versioning is "store a copy" | No FKs: a step naming a deleted role fails only at runtime. "Which workflows use the Finance Officer role?" needs a JSON scan. Shape validated only in code |
| **(c) Relational versioned rules** | `approval_workflows` → `approval_workflow_versions` → `approval_rules` → `approval_step_templates`; runtime in `approval_steps` + `approval_decisions` | FKs and CHECKs on every reference (`chk_approver_target`, `chk_rule_range`); immutable published versions; indexed inbox; SQL-queryable for audit | More tables; the UI editor saves a tree of rows; expressiveness limited to the columns we model |
| (d) External engine (Camunda / Temporal) | BPMN models (Camunda) or workflow code (Temporal) per tenant; the engine holds the running state | Rich modelling (gateways, timers, escalations); durable execution; built-in versioning; visual editors | A second system of record: inbox and request status live partly in the engine, so "my pending approvals" is no longer one SQL query. Extra infrastructure and licensing or hosting cost. Tenant-specific policies must be generated as BPMN or code. Consistency between engine state and `purchase_requests.status` becomes a distributed problem |

## Decision

Store policies as **relational, versioned rules** (option c). A published `approval_workflow_versions` row is **immutable**. Editing a policy creates a new `draft` version; publishing it retires the previous one (`uq_one_published_version` allows one published version per workflow). Requests **snapshot** the version at submission.

### The brief's example as rows

Rules are evaluated by ascending `priority`; the first match wins. Ranges are half-open: `min_amount` is inclusive, `max_amount` is exclusive, both in base currency.

| `approval_rules` (priority) | `min_amount` | `max_amount` | `approval_step_templates` |
|---|---|---|---|
| 1 | 0 | 500 | step 1: `approver_type = department_manager` |
| 2 | 500 | 5000 | step 1: `approver_type = role`, `approver_role_id` → `procurement_manager` |
| 3 | 5000 | NULL | step 1: `department_manager` → step 2: `role` = `procurement_manager` → step 3: `role` = `finance_officer` |

With this convention, $499.99 goes to rule 1, exactly $500.00 to rule 2, and exactly $5,000.00 to rule 3. An organization that wants $5,000.00 to stay with the Procurement Manager only moves the boundary of rules 2 and 3. "Multiple approvals" can also be expressed as two templates with the same `step_order` (parallel) or as `required_approvals = 2` (quorum).

### Snapshot at submission

`POST /purchase-requests/{id}/submit` runs in one transaction:

1. Convert the total: `estimated_total_base = estimated_total × fx_rate_to_base`. Rules match on `estimated_total_base`, never on the transaction-currency amount.
2. Load the published version of the active workflow; evaluate `approval_rules` by priority against amount, `department_id`, `category_id` and `min_priority`.
3. Store `purchase_requests.workflow_version_id` and `matched_rule_id`, set `status = 'in_approval'` and `submitted_at`.
4. Copy each template into an `approval_steps` row (name, `approver_type`, role, `required_approvals`). For `department_manager` steps, resolve `approver_member_id` from `departments.manager_member_id` now.
5. Steps with the lowest `step_order` become `pending` with `due_at = now() + sla_hours`; later steps stay `waiting`.
6. Write the `approval.requested` outbox event and an audit row.

A decision (`POST /approval-steps/{id}/decisions`) locks the step row, checks permission, the approver's `approval_limit_amount` and requester ≠ approver, inserts into `approval_decisions` (one vote per member per step) and increments `approvals_count`. When `approvals_count = required_approvals` the step is `approved`; when all steps of the current `step_order` are approved, the next order becomes `pending`; after the last step the request becomes `approved`. A rejection sets the request to `rejected` and the remaining steps to `skipped`. A `returned` decision sets the request to `returned` and the remaining steps to `skipped`; when the requester resubmits, a fresh snapshot is taken against the version published at that moment, and the old steps and decisions stay for audit.

## Reasons for selection

1. **Configurable per organization without code** (rules out a).
2. **References stay valid.** The FK from `approval_step_templates.approver_role_id` to `roles` means a role used by a policy cannot be deleted; with JSON (b) this fails silently on the next submission.
3. **In-flight requests are stable and auditable.** The request points to its version and rule; published versions never change, so the answer to "why did this need three approvals?" is a JOIN.
4. **The inbox stays one query.** `idx_steps_pending_member` and `idx_steps_pending_role` serve "my approvals" directly, which an external engine (d) would split across two systems.
5. **The SLA job is cheap.** The `approvals` queue scans `idx_steps_overdue` every 15 minutes.

## Advantages

- Policies, runtime state and decisions live in one database and one transaction.
- Database constraints check policy shape (`chk_approver_target`, `chk_rule_range`, `chk_required`).
- Versions give a complete history of policy changes for auditors.

## Disadvantages

- Expressiveness is limited to modelled columns (amount, department, category, priority). Conditions like "supplier risk is high" need a schema change.
- The policy editor must save a tree of rows and validate gaps or overlaps between ranges.
- A department manager resolved at submission stays the approver even if the manager changes later; reassignment is an explicit admin action.

## Trade-offs accepted

- We give up arbitrary BPMN-style branching in exchange for integrity, auditability and one-query inboxes.
- We accept that policy changes only apply to new submissions; that is the behavior auditors expect.

## Consequences / follow-ups

- The policy editor validates on publish that the rule ranges cover 0 to infinity for the default case, and warns about overlaps.
- If the resolved department manager is the requester, the engine routes the step to the manager of the parent department (`departments.parent_department_id`) to keep requester ≠ approver. If there is no parent department, the step becomes a `role` step for `procurement_manager`.
- `approval_workflows.subject_type` already allows invoice approval later; that adds an `invoice_id` column to `approval_steps`, not a new engine.
- **Revisit when:** customers need conditions we cannot express in rule columns (for example supplier risk level or budget remaining) in more than a few cases, which would lead to a typed `approval_rule_conditions` table; or long-running, multi-party workflows across many document types appear, which is the point where Temporal becomes worth its operational cost.
