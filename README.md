# SupplyMesh — Global Procurement & Supplier Intelligence Platform

**Technical product blueprint**: a design-first software engineering assessment. This repository contains no application code. It covers product design, data design, architecture, system design and the reasoning behind each decision.

> Author: Masum Islam Badsha · Date: November 2026
>
> **Presentation video:** [watch on Google Drive](https://drive.google.com/file/d/1R6ohqt0NAe-HltmK0i_G2I2fmq98SJfq/view?usp=drivesdk)

---

## How to read this repository

| # | Deliverable | Document |
|---|---|---|
| 01 | Product Feature Specification (features, business rules, MoSCoW, dependencies, limitations) | [01-product/feature-specification.md](01-product/feature-specification.md) |
| 01 | User Journey Diagrams (supplier onboarding, request approval, PO → invoice → payment) | [01-product/user-journeys.md](01-product/user-journeys.md) |
| 01 | MVP vs Future Scope | [01-product/mvp-vs-future-scope.md](01-product/mvp-vs-future-scope.md) |
| 02 | Database Design: ER diagram, table definitions, constraints, indexing, normalization, multi-tenancy | [02-database/database-design.md](02-database/database-design.md) |
| 02 | Reference schema (PostgreSQL 16 DDL, loaded and tested) | [02-database/schema.sql](02-database/schema.sql) |
| 03 | Application Architecture + high-level diagram | [03-architecture/application-architecture.md](03-architecture/application-architecture.md) |
| 03 | API Structure Proposal | [03-architecture/api-structure.md](03-architecture/api-structure.md) |
| 04 | System Design: 4 critical workflows, bottlenecks, scaling | [04-system-design/system-design.md](04-system-design/system-design.md) |
| 05 | Architecture Decision Records (12 ADRs) | [05-adr/README.md](05-adr/README.md) |
| 06 | Risk and Scalability Analysis | [06-risk/risk-and-scalability.md](06-risk/risk-and-scalability.md) |
| 07 | Engineering Practices: OOP, SOLID, Clean Architecture, security, testing | [07-engineering/engineering-practices.md](07-engineering/engineering-practices.md) |
| ▶ | Technical presentation video | [Watch on Google Drive](https://drive.google.com/file/d/1R6ohqt0NAe-HltmK0i_G2I2fmq98SJfq/view?usp=drivesdk) |

All diagrams are written in [Mermaid](https://mermaid.js.org/), so GitHub renders them directly and they are version-controlled as text. SVG exports are in [`diagrams/`](diagrams/).

---

## The problem in one paragraph

A mid-to-large company buys from hundreds of suppliers in several countries. Today, requests arrive by email, approvals depend on who happens to be in the office, the purchase order lives in a spreadsheet, and the invoice is paid without anyone checking that the goods actually arrived. The result is **maverick spending** (purchases outside policy), **duplicate or inflated invoices**, **no supplier accountability**, and **no audit trail** when something goes wrong. SupplyMesh enforces one controlled path: **approved supplier → approved request → purchase order → goods received → matched invoice → payment**. Every step is permissioned, configurable per organization and audited. The data collected along the way is used to measure supplier performance and risk.

## Key decisions at a glance

| Area | Decision | ADR |
|---|---|---|
| Database | PostgreSQL 16 (relational integrity + ACID for financial flows) | [ADR-001](05-adr/ADR-001-postgresql-over-mongodb.md) |
| Architecture style | **Modular monolith** (NestJS) + separate worker process from the same codebase | [ADR-002](05-adr/ADR-002-modular-monolith.md) |
| Multi-tenancy | Shared schema, `organization_id` on every row, **PostgreSQL RLS** + composite tenant FKs | [ADR-003](05-adr/ADR-003-multi-tenancy.md) |
| Data access | Drizzle ORM (SQL-first, type-safe, supports RLS session settings and partial indexes) | [ADR-004](05-adr/ADR-004-drizzle-orm.md) |
| Approval engine | Workflows stored as **versioned relational rules**; requests snapshot the version at submission | [ADR-005](05-adr/ADR-005-approval-workflow-storage.md) |
| Financial consistency | DB transactions + row locks + optimistic locking + three-way match + idempotency keys | [ADR-006](05-adr/ADR-006-po-invoice-consistency.md) |
| Events & notifications | **Transactional outbox** → BullMQ | [ADR-007](05-adr/ADR-007-transactional-outbox.md) |
| Background jobs | BullMQ on Redis, in a dedicated worker container | [ADR-008](05-adr/ADR-008-background-jobs.md) |
| Redis | Required: queues, permission/analytics cache, rate limiting. **Never** the source of truth. | [ADR-009](05-adr/ADR-009-redis.md) |
| AuthN / AuthZ | Short-lived JWT + rotating refresh tokens; RBAC + department scope + approval limits + segregation-of-duties policies | [ADR-010](05-adr/ADR-010-auth-and-rbac.md) |
| Files | Private S3 bucket, presigned upload/download URLs, virus scan before use | [ADR-011](05-adr/ADR-011-file-storage.md) |
| Deployment | Docker multi-container behind NGINX; single VPS-style EC2 + managed RDS for MVP → ECS later; GitHub Actions CI/CD | [ADR-012](05-adr/ADR-012-deployment.md) |

## Technology map: what is used, and when

| Technology | Role in SupplyMesh | Stage |
|---|---|---|
| TypeScript | One language across frontend, backend, worker and shared contracts | MVP |
| Next.js (App Router) | Web frontend: server-rendered dashboards, role-aware UI | MVP |
| NestJS (on Node.js) | Backend API + worker. Its module system enforces the monolith's internal boundaries. | MVP |
| Express.js | **Not used directly.** It is NestJS's default HTTP adapter underneath; a separate Express service would duplicate NestJS. | — |
| PostgreSQL 16 | System of record | MVP |
| Drizzle ORM + drizzle-kit | Type-safe queries + versioned SQL migrations | MVP |
| Redis 7 | BullMQ queues, caches, rate-limit counters | MVP |
| Docker / Docker Compose | Identical dev, CI and prod images; multi-container stack | MVP |
| NGINX | TLS termination, reverse proxy (`/` → web, `/api` → api), gzip, rate limiting, upload size limits | MVP |
| AWS (RDS, S3, SES, KMS) | Managed DB with PITR backups, document storage, email, key management | MVP |
| VPS (EC2 instance) | Hosts the Docker Compose stack for the pilot | MVP |
| GitHub Actions | CI (lint, type-check, tests) and CD (build images, migrate, deploy) | MVP |
| AWS ECS Fargate + ALB, ElastiCache, RDS read replica | Horizontal scaling once one host is not enough | Growth |
| Kubernetes, Kafka, microservices, Elasticsearch | **Not needed at this stage.** Each one is justified in ADR-002 / ADR-008 / Risk doc. | Not planned |

## Repository conventions

- Table, column and status names in every document match [`schema.sql`](02-database/schema.sql) exactly. The materialized views and the `supplymesh_platform` and `supplymesh_migrator` roles are created by deployment migrations and are described, not defined, in the schema file.
- "Member" means a user's membership in one organization (`organization_members`). All business records refer to members, not users.
- Amounts are stored in transaction currency **and** organization base currency. Policies and analytics use the base currency.
