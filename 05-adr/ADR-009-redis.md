# ADR-009: Redis for queues, caching and rate limiting, never as the source of truth

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-006 PO/invoice consistency](ADR-006-po-invoice-consistency.md) · [ADR-007 Transactional outbox](ADR-007-transactional-outbox.md) · [ADR-008 Background jobs](ADR-008-background-jobs.md) · [ADR-010 Auth & RBAC](ADR-010-auth-and-rbac.md) · [ADR-012 Deployment](ADR-012-deployment.md)

## Context (the problem)

The brief lists Redis in the technology ecosystem. We must decide whether SupplyMesh actually needs it at the MVP stage, what it is used for, and what it must never be used for. Procurement data is financial: a stale cached status could let someone approve a request that was already rejected, or pay an invoice that is already `paid`.

## Is Redis required?

**Yes, at MVP, because BullMQ needs it** (ADR-008). Without the job system, Redis would be optional for the first months: one `api` instance could cache in memory and NGINX could rate-limit alone. Once BullMQ is in place, Redis is already running, so we also use it for the shared, short-lived data listed below. Those uses become necessary, not optional, as soon as `api` runs more than one replica.

## Decision drivers

- BullMQ queues, retries and repeatable jobs (ADR-008).
- Permission checks on every request without querying `role_permissions` and `member_roles` each time, while revoking access within seconds.
- Per-member rate limiting that works across several `api` replicas.
- Safe retries of `POST /payments`, `POST /purchase-orders/{id}/issue` and `POST /invoices` (`Idempotency-Key`).
- Dashboard queries for Organization Analytics are expensive and repeated.
- Correctness of money and status must never depend on Redis.

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| **Redis 7 container on the host (MVP), ElastiCache later** | One Redis for BullMQ, caches, throttler counters and idempotency responses; AOF persistence | One component covers queues, cache and counters; very fast; good NestJS support | One more stateful service; data can be lost on a crash; memory policy must suit both queues and caches |
| No Redis (PostgreSQL only) | pg-boss for jobs; an `idempotency_keys` table; in-process LRU for permissions; NGINX-only rate limiting | Fewer moving parts; everything durable | Queue and cache churn on the OLTP primary during month-end; in-process caches cannot be invalidated across replicas; per-member limits impossible with NGINX alone |
| Memcached for caching + another queue | Memcached for caches; SQS or pg-boss for jobs | Simple cache | Two systems instead of one; no data structures for BullMQ or atomic counters |
| ElastiCache from day one | Managed Redis (or Valkey) in AWS | Managed failover and backups | Extra monthly cost before there is load; MVP runs on one host anyway (ADR-012) |

## Decision

Run **Redis 7** as a container on the MVP host, password-protected and reachable only on the internal Docker network, with **AOF persistence** (`appendfsync everysec`). Move to ElastiCache at the growth stage. All keys are prefixed `org:{orgId}:` (platform keys excepted).

| Use | Key / TTL | Invalidation |
|---|---|---|
| BullMQ queues | BullMQ-managed | n/a |
| Permission sets | `org:{orgId}:member:{memberId}:perms`, 10 min | Deleted on any `member_roles` or `role_permissions` change |
| Org settings + reference data (categories, departments) | 15 min | Deleted on write |
| Analytics dashboard results | 5 min; key includes org + filters | TTL only |
| Rate-limit counters (`@nestjs/throttler`, per member) | Window length | TTL |
| Idempotency responses | 24 h | TTL |

**Never cached:** purchase request, PO and invoice status, `invoices.amount_paid`, approval step state, or any amount used in a decision. These are always read from PostgreSQL inside the transaction that acts on them.

## Never the source of truth

| Redis failure | Effect | Why it is safe |
|---|---|---|
| Cache keys lost | Cache misses; queries hit PostgreSQL | Every cached value is derived from PostgreSQL |
| Idempotency cache lost | A retried `POST /payments` reaches the database | `UNIQUE (organization_id, idempotency_key)` on `payments` returns the original; PO issue is guarded by the `draft → issued` status check plus `version`; invoices by the duplicate-invoice UNIQUE |
| Redis down | Rate limiting fails open (NGINX per-IP limit still applies); permission checks fall back to PostgreSQL; outbox rows wait | API writes still commit; events stay in `outbox_events` |
| Redis loses jobs already published | Jobs whose outbox row has `published_at` set are gone | Re-drive: reset `published_at` for the affected window (ADR-007); consumers are idempotent |

## Reasons for selection

1. **BullMQ makes Redis a hard dependency anyway**, so using it for caches and counters adds no new component.
2. **Permissions are not in the JWT** (ADR-010), so they must be loaded per request; a 10-minute cache with explicit invalidation keeps that cheap while a role change takes effect immediately.
3. **Rate limits per member** need a shared counter once `api` has two replicas.
4. **Keeping queue churn off PostgreSQL** protects the primary during month-end invoice peaks.

## Advantages

- Fast permission checks and dashboards; less load on the primary.
- One shared place for counters and short-lived state across `api` replicas.
- Losing Redis degrades performance and delays notifications, but never corrupts data.

## Disadvantages

- One more stateful service to run, secure, back up and monitor.
- On a single host, Redis is a single point of failure for async work.
- Cache invalidation code must be maintained for permissions and settings.

## Trade-offs accepted

- **Memory policy conflict.** BullMQ requires `maxmemory-policy noeviction`; caches would normally use LRU eviction. At MVP we run one instance with `noeviction`, put a TTL on every cache key and alert at 70% memory. At growth we split into two instances: queues (`noeviction`) and cache (`allkeys-lru`).
- We accept that analytics may be up to 5 minutes stale; decision data is never cached.

## Consequences / follow-ups

- A shared key builder enforces the `org:{orgId}:` prefix (ADR-003).
- Monitoring: memory use, evicted keys (must stay 0), AOF rewrite errors, connected clients.
- **What we would do without Redis:** pg-boss for jobs, a PostgreSQL `idempotency_keys` table, in-process caches with a single `api` instance, NGINX-only rate limiting. This is the fallback if Redis ever becomes a reliability problem.
- **Revisit when:** memory use exceeds ~70% of the host allocation, `api` scales beyond one host (move to ElastiCache, split queue and cache instances), or the cache hit rate for permissions drops below ~90%.
