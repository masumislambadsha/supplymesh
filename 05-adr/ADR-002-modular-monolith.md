# ADR-002: Modular monolith (NestJS) with a separate worker process

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-007 Transactional outbox](ADR-007-transactional-outbox.md) · [ADR-008 Background jobs](ADR-008-background-jobs.md) · [ADR-012 Deployment](ADR-012-deployment.md) · [Application architecture](../03-architecture/application-architecture.md) · [Risk and scalability](../06-risk/risk-and-scalability.md)

## Context (the problem)

SupplyMesh has seven business modules (Supplier Management, Procurement Requests, Approval Workflow Engine, Purchase Orders, Invoices & Payments, Supplier Intelligence, Organization Analytics) plus cross-cutting concerns (identity, tenancy, RBAC, audit, notifications, files). We must decide how to split this into deployable units.

Our assumption: a **small team of 3–6 engineers** builds and runs the product, targets about 200 organizations in year 1, and has no dedicated platform team. The most important workflows cross module boundaries and must be strongly consistent. For example, the final approval decision must change `approval_steps.status`, insert `approval_decisions`, set `purchase_requests.status = 'approved'` and write an outbox event in one transaction.

## Decision drivers

- Final approval (approval module) and request status (procurement module) must commit together, or a request can be "approved" with a pending step.
- Recording a goods receipt (purchasing) and three-way matching an invoice (payables) both update `purchase_order_items.quantity_received` / `quantity_invoiced` under row locks (ADR-006).
- 3–6 engineers: every extra deployable adds pipelines, dashboards, on-call surface and versioned contracts.
- Load is uneven: month-end invoice batches, Monday-morning approval bursts and nightly intelligence recomputes. Heavy background work must not slow down the API.
- Boundaries must still be real, so that a module can be extracted later without a rewrite.

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| **Modular monolith** | One NestJS codebase with 14 modules; deployed as `api` (HTTP) and `worker` (BullMQ) from the same image; one PostgreSQL database with module-owned tables | Cross-module invariants use one local ACID transaction; one build, one deploy, one log stream; refactoring boundaries is cheap; in-process calls have no network latency | Boundaries are enforced by tooling, not by the network; all modules share one release cadence; one runtime and one database to scale |
| Microservices | Separate services, e.g. supplier, procurement, approval, purchasing, payables, intelligence, notification, each with its own database, talking over HTTP and a broker | Independent deploy and scaling per service; fault isolation; team autonomy at larger headcount | Approval → request status and receipt → invoice match become distributed workflows (sagas with compensations); duplicated tenant/RBAC logic in every service; a broker (e.g. Kafka) and service discovery to operate; tracing becomes mandatory; far more infrastructure than 3–6 engineers can run |
| Service-oriented architecture (SOA) | A few coarse services (e.g. "Procurement" = requests + approvals + POs, "Finance" = invoices + payments, "Insights") sharing an integration layer or ESB | Fewer, larger services than microservices; aligns with departments | Still splits the PO ↔ invoice consistency boundary between Procurement and Finance; integration layer becomes a central bottleneck; most of the operational cost of microservices with fewer benefits |

## Decision

Build a **modular monolith**: one NestJS (TypeScript) codebase, deployed as two processes from the same Docker image: `api` (stateless HTTP) and `worker` (BullMQ consumers and schedulers). The Next.js frontend is a separate `web` container. Modules: identity, organization, access-control, supplier, procurement, approval, purchasing, payables, intelligence, analytics, notification, audit, files, and platform/shared.

## Reasons for selection

1. **Our consistency boundaries span modules.** Approval → request status and receipt → invoice matching are single transactions in a monolith. In microservices or SOA they become sagas, and a half-finished saga in payables is a financial error.
2. **Team size.** With 3–6 engineers, the operational cost of microservices (per-service pipelines, versioned contracts, broker, tracing) would take time away from the features in the brief.
3. **Uneven load is solved by the process split, not the service split.** Running `worker` in its own container keeps nightly recomputes and month-end notification bursts away from API latency.
4. **The domain is still being learned.** Boundaries between procurement and purchasing, or intelligence and analytics, may move. Moving code between modules is a refactor; moving it between services is a migration.

## How we keep the monolith modular

| Rule | How it is enforced |
|---|---|
| Each module exposes only a public service interface (`index.ts` exports); internals stay private | Import-boundary rules (dependency-cruiser or eslint-plugin-boundaries) fail the CI build on deep imports |
| Each table has exactly one owning module (e.g. payables owns `invoices`, `invoice_items`, `invoice_discrepancies`, `payments`) | Repositories live inside the owning module; other modules never query those tables directly |
| Cross-module calls go through public services and accept the caller's transaction handle | `withTenantTx` passes one Drizzle transaction, so approval can call `procurement.markApproved(tx, id)` |
| Side effects that may be delayed use events | In-process domain events for same-transaction reactions; the transactional outbox for async work (ADR-007) |
| Layering inside each module: domain / application / infrastructure | Domain classes have no NestJS or Drizzle imports (checked by the same lint rules) |
| Shared contracts are explicit | `packages/contracts` (zod schemas, DTO types) is the only shared code between web, api and worker |
| Each module has its own test suite | Integration tests run per module against Testcontainers PostgreSQL |

## Extraction path

Modules are extracted only when a measured trigger fires, in this order:

1. **intelligence + analytics** (first). They are read-heavy, already run asynchronously and only read other modules' data. Trigger: analytics or nightly `supplier_performance_records` recomputes push OLTP p95 latency above target, or the recompute no longer fits its nightly window. First step: point them at an RDS read replica; second step: a separate service with its own reporting store fed by outbox events.
2. **notification** (second). It is stateless, already behind the `notifications` queue, and only does external I/O (SES). Trigger: new channels (SMS, supplier portal messaging) or email volume that needs its own scaling and provider failover.
3. **Not candidates:** purchasing and payables stay together, because three-way matching needs one transaction over PO lines and invoices.

## Advantages

- One deploy pipeline, one image, one place to debug a request end to end.
- Strong consistency for every financial workflow without distributed transactions.
- The worker can scale separately from the API even though the code is shared.

## Disadvantages

- A bad deploy affects every module at once.
- Boundary rules are only as strong as the lint configuration and code review.
- The whole backend must use one runtime (Node.js) and one dependency set.

## Trade-offs accepted

- We give up independent deployability per module in exchange for local transactions and a much smaller operational footprint.
- We accept a shared database, but not shared tables: ownership is per module, so a later extraction moves whole tables.

## Consequences / follow-ups

- Architecture lint rules and the table-ownership map are part of the repository from the first commit.
- Request IDs and structured logs carry the module name, so a later split does not lose observability.
- **Revisit when:** the team grows beyond about 3 independent squads that block each other on releases, or one of the extraction triggers above is measured in production.
