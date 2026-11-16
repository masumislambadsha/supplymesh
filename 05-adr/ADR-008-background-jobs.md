# ADR-008: Background jobs with BullMQ in a dedicated worker container

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-002 Modular monolith](ADR-002-modular-monolith.md) · [ADR-007 Transactional outbox](ADR-007-transactional-outbox.md) · [ADR-009 Redis](ADR-009-redis.md) · [ADR-011 File storage](ADR-011-file-storage.md) · [System design](../04-system-design/system-design.md)

## Context (the problem)

A lot of SupplyMesh work should not run inside an HTTP request:

| Work | Trigger | Why not inline |
|---|---|---|
| Approval / PO / invoice / payment emails (AWS SES) and in-app notifications | Outbox events | External I/O; must retry on SES errors |
| Virus scan of uploaded supplier documents (ClamAV) | Upload confirmed | Seconds of CPU per file |
| Document-expiry check → risk flag | Nightly | Scans all organizations |
| Approval SLA escalation (`approval.reminder`) | Every 15 minutes | Time-based, not request-based |
| Supplier performance recompute (`supplier_performance_records`, `risk_level`) | Nightly | Aggregates POs, receipts and invoices per supplier per month |
| Analytics materialized view refresh | Hourly | Heavy reads |
| Next `audit_logs` partition, purge expired `user_sessions` | Monthly / daily | Maintenance |

Load is spiky: month-end invoice batches create bursts of discrepancy notifications, and Monday-morning approval bursts create many `approval.requested` emails. We need a job system with retries, scheduling and separation from API latency.

## Decision drivers

- Retries with backoff for SES and scan failures; visibility into permanently failed jobs.
- Repeatable (cron-like) schedules that run exactly once across replicas.
- Heavy jobs must not compete with API requests for the Node.js event loop.
- One language and codebase (TypeScript, NestJS modules reused in the worker).
- Small operational footprint for a 3–6 person team; must run on one EC2 host for MVP (ADR-012).

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| In-process timers / `@nestjs/schedule` in the API | Cron decorators and `setTimeout` inside `api` | Zero infrastructure | Lost on restart; runs N times with N API replicas; no retries; heavy recomputes slow down HTTP responses |
| **BullMQ on Redis, separate `worker` process** | `@nestjs/bullmq` queues; `worker` container runs consumers and repeatable jobs from the same image | Mature Node.js library; retries with exponential backoff; repeatable jobs; rate limiting and concurrency per queue; job ids for de-duplication; dashboards available | Requires Redis (ADR-009); Redis is not durable storage, so the outbox must stay the source of events |
| pg-boss (PostgreSQL-backed queue) | Jobs stored in a PostgreSQL schema; `SKIP LOCKED` polling | No Redis; jobs can be created in the same transaction as business data | Adds write and vacuum load to the same primary that handles month-end OLTP peaks; smaller ecosystem; we would still want Redis for caching and rate limits later |
| AWS SQS + EventBridge Scheduler (+ Lambda or worker polling) | Managed queues and schedules | Fully managed and durable; scales without ops | Harder to run locally and in CI; ties MVP to AWS-specific wiring; no native repeatable jobs inside the app; per-job Lambda cold starts for ClamAV |
| Kafka / RabbitMQ | A message broker with consumer groups | High throughput (Kafka), flexible routing (RabbitMQ) | Another stateful system to operate; our volume (~1M requests/year platform-wide) does not need it |

## Decision

Use **BullMQ on Redis**, consumed by a dedicated **`worker` container** built from the same image as `api` (different entrypoint). Queues:

| Queue | Jobs | Schedule / trigger |
|---|---|---|
| `outbox-relay` | Publish `outbox_events` to the queues below | Every 1 s |
| `notifications` | SES email + `notifications` rows | Outbox events |
| `documents` | `documents.scan` (ClamAV); nightly expiry check → risk flag | Upload confirmation; nightly |
| `approvals` | SLA escalation using `idx_steps_overdue` | Repeatable, every 15 min (built from the start; reminders switched on in Release 1.1) |
| `intelligence` | Recompute `supplier_performance_records` and supplier `risk_level` | Nightly, one job per organization |
| `analytics` | Refresh materialized views | Hourly |
| `maintenance` | Create next `audit_logs` partition; purge expired sessions | Monthly / daily |

**Retry policy:** exponential backoff, 5 attempts. After that the job stays in BullMQ's failed set, which we treat as the dead-letter queue, and an alert fires. **Idempotency:** every job is keyed by the outbox event id (or by `org + period` for scheduled jobs), and handlers check whether the effect already exists before acting, e.g. a `notifications` row for that event and recipient.

## Reasons for selection

1. **API latency is protected.** A nightly recompute over ~2,000 suppliers for a large tenant, or a ClamAV scan, runs in `worker`, not on the event loop that serves the approval inbox.
2. **Retries and dead letters are built in.** SES throttling during a month-end burst becomes delayed delivery, not lost email.
3. **Repeatable jobs run once.** BullMQ coordinates schedules through Redis, so adding a second worker does not send duplicate SLA reminders.
4. **Same code, same modules.** The worker imports the same NestJS modules and `withTenantTx`, so a job uses the same domain rules as an API call.
5. **pg-boss was the strongest alternative.** We prefer not to put queue churn on the OLTP primary during month-end peaks, and Redis is needed anyway for shared caches and rate limiting once `api` scales out (ADR-009).

## Advantages

- Clear split: `api` handles requests, `worker` handles everything asynchronous; each scales separately.
- Per-queue concurrency limits stop one tenant's batch from starving others (nightly jobs fan out one job per organization).
- Local development and CI run the same stack with a Redis container.

## Disadvantages

- Redis becomes a required runtime dependency.
- Jobs in Redis can be lost if Redis loses data; we rely on the outbox and re-drive for recovery.
- Another process to monitor (queue depth, failed set, job duration).

## Trade-offs accepted

- We accept running Redis in exchange for a mature, Node-native job system with scheduling.
- We accept at-least-once execution; correctness comes from idempotent handlers, not from the queue.

## Consequences / follow-ups

- Graceful shutdown: on deploy, the worker stops taking jobs and finishes running ones before exit.
- Metrics per queue: waiting count, failed count, oldest job age; alert on failed-set growth.
- An internal admin view of the failed set (e.g. Bull Board) behind platform-operator authentication (not a tenant role), for retrying jobs after a fix.
- **Revisit when:** we move to ECS and want fully managed durability (SQS for notifications), or Redis operations cost more than its value; pg-boss remains the fallback if we ever want to drop Redis, since producers only depend on the outbox.
