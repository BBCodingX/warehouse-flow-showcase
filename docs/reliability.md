# Reliability & distributed-systems patterns

This is the part of the project that matters. WarehouseFlow exists to show how the hard problems of
an event-driven system are handled — not with a library that hides them, but explicitly.

## 1. The dual-write problem → Transactional Outbox

**Problem.** When the API must both change domain state *and* publish a message, doing them as two
separate writes is unsafe: if the process crashes between the DB commit and the broker publish, the
message is lost; if it crashes the other way, a message describes a change that never happened.

**Solution.** The domain change and an `outbox_messages` row are written in **one PostgreSQL
transaction**. A separate polling publisher (`OutboxBatchProcessor`) reads unprocessed rows in
bounded batches with `SELECT … FOR UPDATE SKIP LOCKED`, publishes to RabbitMQ, and only then sets
`processed_at_utc`. Failed publishes increment `attempt_count` and fill `last_error`, so a poison
row is observable rather than silent.

**Consequence.** Message loss is eliminated. The cost is that delivery becomes **at-least-once** —
a publish can succeed while the `processed_at_utc` update fails, causing a redelivery. That is why
the Inbox exists.

> ADR-008 (Outbox), ADR-018 (polling publisher, and why `LISTEN/NOTIFY` was deferred).

## 2. At-least-once delivery → Inbox / `ProcessedMessage`

Every consumer records `(MessageId, ConsumerName)` in `processed_messages` **in the same
transaction as its domain write**. A redelivered message is recognised and treated as a no-op. The
domain effect happens exactly once even though the transport guarantees only at-least-once.

Redis is checked first as a fast path, but it is not required — if Redis is down, the PostgreSQL
`processed_messages` table is authoritative.

> ADR-009.

## 3. Stock accuracy under concurrency → a layered ladder

Two operators removing the last units of a SKU at the same moment must not both succeed. Stock
correctness does **not** rest on one mechanism; it rests on several working together:

1. **PostgreSQL transaction boundaries** — every stock write is inside one.
2. **Optimistic concurrency** — an application-managed numeric `row_version` on `inventory_items`,
   incremented on every update. A stale update throws `DbUpdateConcurrencyException`.
3. **Bounded retry** — the Order + Reservation Worker retries an optimistic conflict a fixed number
   of times, re-reading rows each pass.
4. **Row-level locks as a second layer** — if the retry budget is exhausted under genuine high
   contention, the reservation re-runs with `SELECT … FOR UPDATE`. This does *not* replace the
   optimistic path; it is a fallback so retries cannot starve.
5. **Check constraints** — `physical_quantity >= 0`, `reserved_quantity >= 0`,
   `reserved_quantity <= physical_quantity` are enforced by the database. Even a logic bug cannot
   persist a negative or over-reserved row.
6. **Unique constraints** — `stock_reservations.idempotency_key`, and a partial unique index
   `picking_tasks (order_id) WHERE status IN ('Created','InProgress')` for "one active picking task
   per order".

The Redis distributed lock is explicitly **not** part of correctness — it only reduces wasted
optimistic retries, and the code proceeds correctly when it cannot be acquired.

> ADR-017.

### The actual code (from the private repo)

```csharp
// ReserveStockConsumer — optimistic first, FOR UPDATE only as a fallback
private const int MaxOptimisticAttempts = 3;

ReservationAttempt attempt;
try
{
    attempt = await TryApplyReservationAsync(reservation, targetItems, useRowLock: false, ct);
}
catch (DbUpdateConcurrencyException)
{
    // Optimistic retry budget exhausted under contention — escalate to row locks.
    attempt = await TryApplyReservationAsync(reservation, targetItems, useRowLock: true, ct);
}
```

```csharp
// inside TryApplyReservationAsync
var items = useRowLock
    ? await db.InventoryItems
        .FromSqlInterpolated($"SELECT * FROM inventory_items WHERE id = ANY({ids}) FOR UPDATE")
        .ToListAsync(ct)
    : await db.InventoryItems.Where(i => ids.Contains(i.Id)).ToListAsync(ct);

// ... plan the allocation, apply deltas, write the ledger rows ...

try
{
    await db.SaveChangesAsync(ct);
    return ReservationAttempt.Succeeded;
}
catch (DbUpdateConcurrencyException) when (!useRowLock && attempt < MaxOptimisticAttempts)
{
    // Bounded retry (ADR-017); rows are re-read on the next pass.
}
```

## 4. Multi-location, all-or-nothing reservation

A single location may not hold enough of a SKU while the warehouse total does. The reservation
selects and locks the product's `Available` inventory rows in a **deterministic order** (location
code, then id), and may spread one order line across several rows. If *any* line of the order
cannot be fully satisfied, **no** reservation is committed. Short-pick release and dispatch later
consume these exact allocations rather than picking rows at random.

> ADR-028.

## 5. Failures that must not vanish → durable async lifecycle + DLQ

`202 Accepted` only proves the request reached the Outbox. So every `ICommand` also writes an
`AsyncOperation` row in that same transaction, with a lifecycle:

`Accepted` → `Published` (broker confirm) → `Succeeded` (Inbox/domain commit) → `Failed` (last
error from the central retry policy).

On terminal failure — only after the real MassTransit retry chain is exhausted — a single
`OperationalException` is created with the correlation id, and it surfaces in an operator queue and
a personal notification feed. Nothing fails silently.

> ADR-026.

## 6. Cycle-count reconciliation safety

Stock can move while a physical count is in progress. When a count opens, the system snapshots each
inventory row's physical quantity and `row_version`. The operator counts **blind** (expected
quantity is shown only after submit). Approval locks the affected rows `FOR UPDATE` in deterministic
order and applies nothing if the snapshot drifted or if the counted quantity would drop below
reserved. A valid variance is applied together with a `CycleCount`-reason ledger row in one
transaction.

> ADR-027.

## 7. Append-only inventory ledger

Every movement writes an `inventory_transactions` row: before/after physical and reserved
quantities, a reason code (`Receive`, `Adjust`, `TransferIn`/`TransferOut`, `Reservation`,
`ReservationRelease`, `ShipmentDispatch`, `CycleCount`, …), source/destination location, a business
reference (`ReferenceType` + `ReferenceId`), and the initiating user. The ledger is never updated
or deleted — the data-retention sweep explicitly skips it. Current stock is always reconstructable
from it.

## 8. What is deliberately *not* here

- No Saga / process manager — the flows are short and a partial unique index expresses the one
  "single active instance" rule more simply.
- No Event Sourcing — the ledger gives auditability without rebuilding aggregates from events.
- No `LISTEN/NOTIFY` — polling latency is negligible at this scale and far easier to debug; noted
  as a future option if metrics ever justify it.
- No MassTransit 9.x — it requires a paid runtime license key, which violates the project's
  "no paid services" constraint. 8.x is Apache-2.0.

Every one of these is a written decision, not an omission — see the ADRs in
[engineering.md](engineering.md#architecture-decision-records).
