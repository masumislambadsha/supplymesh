# ADR-006: Consistency of purchase orders, receipts, invoices and payments

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-001 PostgreSQL](ADR-001-postgresql-over-mongodb.md) · [ADR-004 Drizzle ORM](ADR-004-drizzle-orm.md) · [ADR-007 Transactional outbox](ADR-007-transactional-outbox.md) · [ADR-009 Redis](ADR-009-redis.md) · [API structure](../03-architecture/api-structure.md) · [System design](../04-system-design/system-design.md)

## Context (the problem)

The PO → goods receipt → invoice → payment chain is where SupplyMesh moves money. Concurrency is normal, not exceptional: at month-end a supplier sends a batch of invoices for one PO while the warehouse records the last receipts, and two Finance Officers work through the same payment run. Typical failures we must prevent:

- paying an invoice twice (double click, network retry, two people);
- paying more than the invoice total in total across several partial payments;
- invoicing or paying for goods that were never received (the point of three-way matching);
- receiving more than was ordered;
- two people editing the same PO or invoice and silently overwriting each other.

## Decision drivers

- Recording a payment must insert into `payments` and update `invoices.amount_paid` and `invoices.status` atomically.
- `purchase_order_items.quantity_received` and `quantity_invoiced` are running counters; concurrent receipts and invoices must not lose updates.
- Match tolerances are per organization: `organizations.settings` holds e.g. `{ "price_tolerance_pct": 2, "qty_tolerance_pct": 0 }`.
- Segregation of duties: the invoice recorder cannot approve it (`chk_inv_sod`); the PO creator cannot record or approve its payment; requester ≠ approver.
- Retried API calls on `POST /payments`, `POST /purchase-orders/{id}/issue` and `POST /invoices` must be safe.
- Everything lives in one PostgreSQL database (ADR-001, ADR-002).

## Options considered

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| **Local ACID transactions + row locks + optimistic locking + DB constraints** | Each command is one `withTenantTx`; counters updated under `SELECT … FOR UPDATE`; user edits use `version` + `If-Match`; CHECK and UNIQUE constraints as the last line of defence | Uses guarantees PostgreSQL already gives; simple to reason about and test; no extra infrastructure | Lock contention on hot rows at month-end; developers must follow a lock order |
| SERIALIZABLE isolation for every transaction | Let PostgreSQL detect conflicts; retry on serialization failure | No explicit locks to forget | Retry storms during month-end batches; every write path needs retry logic; harder to predict latency |
| Optimistic locking only | `UPDATE … WHERE version = $v` everywhere, including counters | No blocking | Under contention most counter updates fail and retry; user-facing 412 errors for server-side work that should simply wait |
| Sagas / distributed transactions (2PC) | Separate steps with compensations, or XA across resources | Needed when data spans services or databases | Our data is in one database, so this adds compensation logic and failure modes with no benefit |

## Decision

Use **local database transactions** with three complementary techniques, plus database constraints and idempotency keys:

| Technique | Where | Why this one |
|---|---|---|
| Pessimistic row locks (`SELECT … FOR UPDATE`) | Server-side counter and balance updates: `purchase_order_items` lines during receipt and invoice approval; the `invoices` row during payment; the `organization_sequences` row for document numbers | Short transactions that must wait, not fail |
| Optimistic locking (`version` + `ETag` / `If-Match`) | Human edits with think time: PR, PO, invoice and supplier headers; invoice approval | Locks cannot be held across HTTP requests; a stale edit gets `412`, a missing header `428` |
| Constraints | `chk_inv_paid`, `chk_poi_received`, `chk_inv_sod`, `chk_pay_amount`, `UNIQUE (organization_id, supplier_id, supplier_invoice_number)`, `UNIQUE (organization_id, idempotency_key)` | Catch any code path that forgets a rule |
| Idempotency keys | `Idempotency-Key` header: Redis response cache (24 h) for all three endpoints; `payments.idempotency_key` UNIQUE in the database | Retries return the first result; the DB constraint holds even if Redis is empty |

Isolation level is PostgreSQL's default **READ COMMITTED** plus the explicit locks above. Lock order is fixed to avoid deadlocks: invoice → PO header → PO lines in ascending `id` → `organization_sequences` (taken last, so the per-organization number row is held as briefly as possible).

### Flows

**Goods receipt.** Lock the PO lines being received; insert `goods_receipts` and `goods_receipt_items`; increase `quantity_received`. `chk_poi_received` rejects over-receipt. Set PO status to `partially_received` or `received`.

**Invoice and three-way match.** Insert the `invoices` row (the duplicate-invoice UNIQUE rejects a re-entered supplier invoice), then lock the referenced PO lines (`SELECT … FOR UPDATE`, sorted by `id`), then insert `invoice_items`. The order matters: inserting `invoice_items` before locking would take FK share locks on the same PO lines, and two concurrent invoices for one PO would deadlock when both try to upgrade to `FOR UPDATE`. Then compare per line:

| Check | Compared values | Discrepancy type |
|---|---|---|
| Price | `invoice_items.unit_price` vs `purchase_order_items.unit_price`, within `price_tolerance_pct` | `price` |
| Quantity | invoiced quantity vs `quantity_received − quantity_invoiced`, within `qty_tolerance_pct` | `quantity` |
| Nothing received yet | `quantity_received = 0` | `missing_receipt` |
| Header | `invoices.currency` vs PO currency; `total` vs sum of lines + tax | `currency`, `total`, `tax` |

No issues: status `matched`, `matched_at` set. Issues: `invoice_discrepancies` rows, status `discrepancy`, and an `invoice.discrepancy_detected` outbox event. Matching only reads `quantity_invoiced`; it does not change it. There is no CHECK that stops `quantity_invoiced` exceeding `quantity_received`, on purpose: over-invoicing is a business event that is recorded and reviewed, not a SQL error.

**Invoice approval.** Requires `If-Match`, no `open` discrepancies (`idx_disc_open`), and a different member from `recorded_by_member_id` (`chk_inv_sod`). Under the lock on the invoice and its PO lines, the quantity rule is checked again (another invoice for the same PO may have been approved since matching) and `quantity_invoiced` is increased. It is reduced again only if the approved invoice is voided.

**Payment.** In one transaction: lock the invoice row; require status `approved` or `partially_paid`; check the recorder did not create the PO; compute `outstanding = total − amount_paid − (sum of this invoice's payments in scheduled or processing)`; reject if `amount > outstanding`; insert the payment. `amount_paid` and the invoice status change **only when a payment becomes `completed`**, in the same transaction as that transition. A payment that becomes `failed` or `cancelled` never touched `amount_paid`, so nothing has to be reversed. A second request with the same key hits `UNIQUE (organization_id, idempotency_key)` and returns the existing payment.

## Reasons for selection

1. **Single database, single transaction.** Sagas solve consistency across services; our payment, invoice and PO rows share one PostgreSQL transaction, so a saga would add compensation code for failures that cannot happen.
2. **Month-end batches favour waiting over failing.** Under READ COMMITTED with row locks, a second receipt on the same PO line waits a few milliseconds; under SERIALIZABLE or optimistic-only it would fail and retry.
3. **Humans need conflict detection, not locks.** A Procurement Manager editing a PO for five minutes cannot hold a lock; `version` + `If-Match` reports a conflict instead of overwriting.
4. **Defence in depth.** If a code path forgets the outstanding-amount check, `chk_inv_paid` still stops `amount_paid > total` on completion.

## Advantages

- Every financial invariant is either a constraint or a check under a lock in one transaction, and both are testable with concurrent integration tests.
- No extra infrastructure; consistency does not depend on Redis or the queue.
- Tolerances are configurable per organization without code changes.

## Disadvantages

- Hot rows serialize work: one large PO receiving many invoices, and the per-organization `organization_sequences` row at month-end.
- Developers must follow the lock order; a mistake shows up as a deadlock under load.
- Two concurrency styles (pessimistic and optimistic) must be explained to new engineers.

## Trade-offs accepted

- We accept brief waits on hot rows in exchange for no lost updates and no retry storms.
- We accept that over-invoicing is allowed into the database as a discrepancy, because Finance needs to see and resolve it.
- Reversing a `completed` payment (bank recall) is out of MVP scope; it would need a reversal record, since `chk_pay_amount` requires amounts > 0.

## Consequences / follow-ups

- A shared `lockPoLines(tx, ids)` helper sorts ids before locking; code review rejects ad-hoc `FOR UPDATE`.
- Concurrency tests: two parallel payments for the full amount → exactly one succeeds; parallel receipts never exceed the ordered quantity.
- Transactions get a `lock_timeout` (e.g. 5 s) and return a retryable `409` instead of hanging.
- **Revisit when:** lock waits on `organization_sequences` or PO lines exceed ~100 ms p95 during month-end (first shorten the transactions that hold them; pre-allocated number blocks are faster but break gap-free numbering, so only with Finance's agreement), or payments integrate with an external bank API, at which point the external call moves to the worker via the outbox and the payment stays `processing` until the bank confirms.
