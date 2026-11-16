# ADR-001: PostgreSQL over MongoDB as the system of record

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-003 Multi-tenancy](ADR-003-multi-tenancy.md) · [ADR-004 Drizzle ORM](ADR-004-drizzle-orm.md) · [ADR-006 PO/invoice consistency](ADR-006-po-invoice-consistency.md) · [Database design](../02-database/database-design.md) · [schema.sql](../02-database/schema.sql)

## Context (the problem)

SupplyMesh stores the full procurement chain for many organizations: supplier → purchase request → approval steps → purchase order → goods receipt → invoice → payment. These records point at each other: an `invoice_items` row points to a `purchase_order_items` row, which points to a `purchase_request_items` row, which belongs to a request charged to a department and approved under a specific `approval_workflow_versions` row. Money moves along this chain. A wrong reference or a half-applied update means paying the wrong supplier, paying twice, or paying for goods that never arrived.

We need one primary database for this transactional data. The two realistic candidates are a relational database (PostgreSQL) and a document database (MongoDB). We also looked at MySQL as the other mainstream relational choice.

## Decision drivers

- **Atomic multi-row money updates.** Recording a payment must insert into `payments`, increase `invoices.amount_paid`, move `invoices.status` to `partially_paid` / `paid` and write an `audit_logs` row, all or nothing.
- **Referential integrity across tenants.** A purchase order in Org B must never point at a supplier in Org A. Composite FKs `(organization_id, supplier_id) → suppliers(organization_id, id)` give us this in the schema.
- **Business rules the database can enforce:** `chk_inv_paid` (never pay more than `total`), `chk_poi_received` (never receive more than ordered), `chk_inv_sod` (invoice recorder ≠ approver), `UNIQUE (organization_id, supplier_id, supplier_invoice_number)` (no duplicate invoice), `UNIQUE (organization_id, idempotency_key)` on `payments`.
- **Tenant isolation inside the database**, not only in application code: Row-Level Security with `SET LOCAL app.current_org_id` (ADR-003).
- **Exact decimal money for global suppliers:** `NUMERIC(19,4)` amounts plus `fx_rate_to_base NUMERIC(18,8)` captured per transaction.
- **Reporting on relationships:** spend by department / supplier / category, approval cycle time and supplier on-time rate are JOIN + GROUP BY queries.
- **Small, hot work queues:** "my pending approvals", "invoices due", "documents expiring" need partial indexes (`idx_steps_pending_member`, `idx_inv_unpaid_due`, `idx_supplier_docs_expiry`).
- **Managed backups with point-in-time recovery** for financial data from day one (AWS RDS).

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| **PostgreSQL 16 (RDS)** | 35 normalized tables, composite tenant FKs, CHECK constraints, RLS policies, JSONB only for `organizations.settings`, addresses, audit before/after and outbox payloads | FKs, CHECKs, generated columns, partial indexes, RLS and transactional DDL all exist natively; strong SQL for analytics; mature managed offering with PITR | Schema changes need migrations; vertical write scaling on one primary; joins need index discipline |
| MongoDB (Atlas or self-hosted) | Collections per aggregate, e.g. a `purchaseOrders` document embedding its lines and receipts; invoices reference the PO by id | Flexible documents suit varied supplier attributes; multi-document ACID transactions since 4.0; `$jsonSchema` validation; `Decimal128` for money; built-in sharding | No foreign keys between collections: "invoice line points to an existing PO line of the same tenant" becomes application code. Validation is per document, so cross-row rules (`amount_paid ≤ total` across payments) live in code. No RLS equivalent, so tenant filtering is purely an application concern. Relational reports need `$lookup` pipelines |
| MySQL 8 | Same relational model as PostgreSQL | FKs, CHECK constraints (8.0.16+), generated columns, mature managed offerings | No row-level security, no partial indexes (the work-queue indexes become full indexes), no transactional DDL, weaker JSON indexing |

## Decision

Use **PostgreSQL 16 on AWS RDS** as the single system of record for all tenant data. Locally and in CI it runs as a container. JSONB is allowed only where data is never filtered, joined or summed (see database-design §7).

### Requirement → PostgreSQL feature

| SupplyMesh requirement | PostgreSQL feature used | Where in `schema.sql` |
|---|---|---|
| Never pay more than the invoice total | CHECK + row lock in one transaction | `chk_inv_paid`, `invoices.amount_paid` |
| No double payment on retry | Unique index | `UNIQUE (organization_id, idempotency_key)` on `payments` |
| No duplicate supplier invoice | Unique index | `UNIQUE (organization_id, supplier_id, supplier_invoice_number)` |
| No cross-tenant references | Composite foreign keys | e.g. `invoices (organization_id, purchase_order_id)` |
| Tenant isolation even if code forgets a filter | Row-Level Security | `tenant_isolation` policy on every tenant table |
| Line totals can never disagree with quantity × price | Generated columns | `purchase_order_items.line_subtotal`, `invoice_items.line_subtotal` |
| Fast approval inbox and payment run | Partial indexes | `idx_steps_pending_member`, `idx_inv_unpaid_due` |
| Gap-free PO and request numbers per organization | `SELECT … FOR UPDATE` on a counter row | `organization_sequences` |
| Audit trail that grows ~3M rows/month platform-wide | Declarative range partitioning | `audit_logs` monthly partitions |
| Sparse per-org configuration (match tolerances) | JSONB | `organizations.settings` |

## Reasons for selection

1. **The payment invariant is a database invariant.** `chk_inv_paid` plus a row lock on the invoice (ADR-006) guarantees that the sum of completed payments never exceeds the invoice total, even with two Finance Officers clicking at the same time. In MongoDB we would rely on application code getting this right in every path.
2. **Cross-tenant references are impossible by construction.** The composite FK pattern used on every child table turns a cross-tenant bug into a constraint violation. This was tested against the schema (database-design §8.4). MongoDB has no equivalent.
3. **Duplicate-invoice and double-payment protection is a unique index**, not a read-then-write check that can race during month-end invoice batches.
4. **RLS gives a second, independent isolation layer** for a multi-tenant SaaS holding commercial data. That is the main reason MySQL was rejected as well.
5. **Partial indexes match our workload.** Approval inboxes, unpaid invoices and expiring documents are a small, hot slice of large tables.
6. **Organization Analytics is relational.** Monthly spend by department and supplier is a GROUP BY over POs and invoices; we serve it from materialized views that PostgreSQL can refresh concurrently.

## Advantages

- Most business rules from the brief are enforced in one place, close to the data, and covered by constraint tests.
- One technology for OLTP, reporting views, JSONB configuration and fuzzy supplier search (`pg_trgm` index on `suppliers.legal_name`).
- RDS gives PITR (7–35 days), Multi-AZ and read replicas without us operating them.

## Disadvantages

- Every schema change is a migration that must be backward compatible during a rolling deploy (expand/contract, ADR-012).
- One primary handles all writes. Month-end invoice spikes hit the same instance as approvals.
- Flexible per-country supplier data (addresses, bank formats) is less natural; we use JSONB for `suppliers.address` and keep it out of queries.

## Trade-offs accepted

- We accept migration discipline in exchange for constraints that protect money.
- We accept that a document database would be faster to iterate on for supplier attributes. Supplier custom fields are future scope and will use validated JSONB, not a second database.
- We accept vertical scaling of the primary for year 1 (planning assumption: ~200 organizations, ~90,000 invoices per year for a large tenant; see Risk §3.1), with read replicas for analytics as the next step.

## Consequences / follow-ups

- All amounts use `NUMERIC(19,4)`, never floating point; `*_base` amounts and `fx_rate_to_base` are captured once per transaction.
- `pg_stat_statements` is enabled from the start; indexes are added from evidence, not guesses.
- Analytics read from `supplier_performance_records` and materialized views, later from an RDS read replica.
- **Revisit when:** (a) write throughput on the primary stays above ~70% CPU after query tuning and PgBouncer, which points to partitioning or a distributed PostgreSQL (e.g. Citus) rather than a different database family; or (b) full-text search over supplier documents becomes a must-have, which would add a search index next to PostgreSQL, not replace it.
