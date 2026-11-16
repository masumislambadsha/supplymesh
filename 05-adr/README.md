# Deliverable 05: Architecture Decision Records

An **Architecture Decision Record (ADR)** is a short document that captures one significant technical decision: the problem, the options we considered, what we chose, why, and what it costs us. ADRs are written when the decision is made and are not rewritten later. If a decision changes, a new ADR supersedes the old one, so the history of reasoning stays visible.

For SupplyMesh, every ADR is tied to concrete product requirements (procurement roles, approval policies, three-way matching, multi-tenant isolation, audit, multi-currency suppliers, month-end invoice peaks) rather than to general technology preferences. Each one also names the alternatives honestly, including where a rejected option is strong, and states a concrete trigger for revisiting the decision.

## Template

Every ADR uses the same sections:

| Section | Purpose |
|---|---|
| Context (the problem) | What forces a decision, in SupplyMesh terms |
| Decision drivers | The specific requirements the decision must satisfy |
| Options considered | A table: option, how it would work here, pros, cons |
| Decision | What we chose, precisely enough to implement |
| Reasons for selection | Each reason linked to a driver |
| Advantages / Disadvantages | Honest consequences of the choice |
| Trade-offs accepted | What we knowingly give up |
| Consequences / follow-ups | Work this creates, and the trigger that would make us revisit it |

**Status values:** Proposed → Accepted → (Deprecated | Superseded by ADR-NNN).

## Index

| # | Title | Status | Decision in one line |
|---|---|---|---|
| [ADR-001](ADR-001-postgresql-over-mongodb.md) | PostgreSQL over MongoDB | Accepted | PostgreSQL 16 on RDS is the system of record, because payment and matching invariants rely on FKs, CHECKs, unique indexes, RLS and multi-row ACID transactions |
| [ADR-002](ADR-002-modular-monolith.md) | Modular monolith | Accepted | One NestJS codebase with enforced module boundaries, deployed as `api` + `worker`; intelligence/analytics is the first extraction candidate |
| [ADR-003](ADR-003-multi-tenancy.md) | Multi-tenancy | Accepted | Shared schema with `organization_id`, PostgreSQL RLS via `SET LOCAL`, and composite tenant FKs |
| [ADR-004](ADR-004-drizzle-orm.md) | Drizzle ORM | Accepted | Drizzle + drizzle-kit, because its SQL-first schema expresses our CHECKs, generated columns, partial indexes and RLS, and keeps tenant context in the same transaction |
| [ADR-005](ADR-005-approval-workflow-storage.md) | Approval workflow storage | Accepted | Policies stored as versioned relational rules; requests snapshot the published version and rule at submission |
| [ADR-006](ADR-006-po-invoice-consistency.md) | PO/invoice consistency | Accepted | Local transactions with row locks, optimistic locking (`version` + `If-Match`), three-way match with per-org tolerances, constraints and idempotency keys; no sagas |
| [ADR-007](ADR-007-transactional-outbox.md) | Transactional outbox | Accepted | Events are written to `outbox_events` in the business transaction and relayed to BullMQ every second |
| [ADR-008](ADR-008-background-jobs.md) | Background jobs | Accepted | BullMQ on Redis, consumed by a dedicated `worker` container, with retries and a dead-letter set |
| [ADR-009](ADR-009-redis.md) | Redis | Accepted | Required at MVP because of BullMQ; also used for permission/analytics caches, rate limits and idempotency responses; never the source of truth |
| [ADR-010](ADR-010-auth-and-rbac.md) | Auth & RBAC | Accepted | 15-min JWT + rotating refresh tokens; permission-based RBAC with department scope, approval limits and segregation-of-duties policies |
| [ADR-011](ADR-011-file-storage.md) | File storage | Accepted | Private S3 bucket, presigned upload/download URLs, virus scan before a document can be reviewed |
| [ADR-012](ADR-012-deployment.md) | Deployment & CI/CD | Accepted | Docker Compose on one EC2 host behind NGINX with RDS/S3/SES/KMS; GitHub Actions CI/CD; ECS later; no Kubernetes now |

## How the decisions fit together

- **Data integrity chain:** ADR-001 (PostgreSQL) → ADR-003 (RLS + composite FKs) → ADR-004 (Drizzle expresses those constraints) → ADR-006 (transactions and locks for money).
- **Asynchronous chain:** ADR-007 (outbox) → ADR-008 (BullMQ worker) → ADR-009 (Redis, never authoritative).
- **Runtime shape:** ADR-002 (modular monolith) → ADR-012 (containers on one host, then ECS).
