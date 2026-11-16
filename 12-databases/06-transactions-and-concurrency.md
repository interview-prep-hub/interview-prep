# Transactions and Concurrency

> **Tier:** T1 (know cold) · **Related:** [PostgreSQL](04-postgresql.md), [Indexing and query optimization](05-indexing-and-query-optimization.md), [MySQL essentials](07-mysql-essentials.md), [ActiveRecord advanced](../05-ruby-on-rails/06-activerecord-advanced.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md)
> **On your resume:** rent payments and split bills in the Coloc resident app, stock and order sync between CrystalCommerce stores and PriceCharting, background jobs on both Devntech projects (Sidekiq on Colochousing): all places where two things happen at once.

## Quick revision
- A **transaction** is all-or-nothing: `BEGIN ... COMMIT` or `ROLLBACK`. **Savepoints** roll back part of one.
- **ACID**: Atomicity (all or nothing), Consistency (constraints hold), Isolation (concurrent transactions do not interfere, to a configurable degree), Durability (committed data survives crashes, via the WAL).
- Anomalies: **dirty read** (see uncommitted data), **non-repeatable read** (same row, different value), **phantom** (same query, new rows), **lost update** (two read-modify-writes, one overwritten), **write skew** (two transactions check the same condition, then write different rows, breaking the rule together).
- Levels: Read Uncommitted < **Read Committed** (Postgres default) < **Repeatable Read** (MySQL InnoDB default) < Serializable.
- Postgres never allows dirty reads. Its Repeatable Read is **snapshot isolation** (no phantoms; concurrent updates of the same row raise a serialization error). Its Serializable uses **SSI** and aborts one transaction with SQLSTATE `40001`: always retry.
- InnoDB Repeatable Read: plain SELECTs read a snapshot, but locking reads and UPDATEs read the latest data and take **next-key (gap) locks**.
- **MVCC**: readers do not block writers. Writers on the same row still block each other.
- `SELECT ... FOR UPDATE` locks rows until commit; `NOWAIT` errors instead of waiting; `SKIP LOCKED` skips locked rows (job queues).
- **Optimistic** locking: a version column, `UPDATE ... WHERE version = ?`, retry on conflict (low contention). **Pessimistic**: lock first (high contention, short critical sections).
- **Deadlock**: two transactions each wait for the other's lock. Postgres detects it (after `deadlock_timeout`, 1 s by default) and aborts one (`40P01`). Prevent with a consistent lock order and short transactions.
- **Advisory locks**: app-defined locks on a number (`pg_advisory_xact_lock`), for "only one sync per store".
- Keep transactions short; never call external APIs inside one; enqueue jobs **after commit**.
- Best concurrency fixes are often in the schema: atomic `UPDATE ... SET x = x - 1 WHERE x >= 1`, unique and partial unique indexes, exclusion constraints.

## 1. Transactions, savepoints and errors

```sql
BEGIN;
UPDATE accounts SET balance_cents = balance_cents - 5000 WHERE id = 1;
UPDATE accounts SET balance_cents = balance_cents + 5000 WHERE id = 2;
COMMIT;                -- both changes become visible together, or neither does (ROLLBACK)

BEGIN;
INSERT INTO orders (customer_id, status) VALUES (1, 'pending');
SAVEPOINT before_items;
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (999, 1, 1, 10);  -- fails (FK)
ROLLBACK TO SAVEPOINT before_items;   -- undo just the failed part
COMMIT;                               -- the order row is kept
```

- Without `BEGIN`, each statement runs in its own transaction (autocommit).
- In Postgres, any error inside a transaction marks it aborted: every further statement fails with "current transaction is aborted, commands ignored until end of transaction block" until you `ROLLBACK` (or roll back to a savepoint).
- Rails nested `transaction(requires_new: true)` blocks become savepoints; without `requires_new`, inner blocks simply join the outer transaction.

## 2. ACID in depth

| Property | Meaning | How Postgres provides it | What can still go wrong |
|---|---|---|---|
| Atomicity | All statements commit, or none do | Transaction status plus MVCC row versions; aborted versions are never visible | App code that commits halfway (two separate transactions for one business action) |
| Consistency | Every committed state satisfies the constraints | PK, FK, UNIQUE, CHECK, exclusion constraints, triggers | Rules that exist only in app code (validations) are not guaranteed |
| Isolation | Concurrent transactions do not see each other's partial work | Snapshots (MVCC), row locks, SSI | The default level allows some anomalies (sections 3 and 4) |
| Durability | Once COMMIT returns, the data survives a crash | WAL flushed to disk at commit; replicas for disk loss | `synchronous_commit = off` trades a small window of loss for speed |

## 3. Concurrency anomalies

Each table shows two sessions over time; in Postgres, each statement outside `BEGIN` commits immediately.

### 3.1 Dirty read: reading uncommitted data
| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN; UPDATE accounts SET balance_cents = 0 WHERE id = 1;` | |
| 2 | | `SELECT balance_cents FROM accounts WHERE id = 1;` sees 0 under Read Uncommitted |
| 3 | `ROLLBACK;` | B acted on a value that never existed |

Postgres never shows uncommitted data at any level (B sees the old committed balance).

### 3.2 Non-repeatable read: the same row changes mid-transaction
| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN; SELECT price FROM products WHERE id = 1;` returns 100 | |
| 2 | | `UPDATE products SET price = 120 WHERE id = 1;` |
| 3 | `SELECT price FROM products WHERE id = 1;` returns 120 under Read Committed, 100 under Repeatable Read | |

### 3.3 Phantom read: new rows appear in a repeated query
| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN; SELECT count(*) FROM bookings WHERE room_id = 7;` returns 2 | |
| 2 | | `INSERT INTO bookings (room_id, stay) VALUES (7, ...);` |
| 3 | Same count returns 3 under Read Committed (a phantom), 2 under Postgres Repeatable Read | |

### 3.4 Lost update: two read-modify-write cycles
| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN; SELECT stock FROM products WHERE id = 1;` returns 10 | |
| 2 | | `BEGIN; SELECT stock FROM products WHERE id = 1;` returns 10 |
| 3 | App computes 10 - 1: `UPDATE products SET stock = 9 WHERE id = 1; COMMIT;` | |
| 4 | | App computes 10 - 1: `UPDATE products SET stock = 9 WHERE id = 1; COMMIT;` |
| Result | Two units sold, stock went down by one | |

Under Read Committed, B's UPDATE waits for A, then overwrites. Under Postgres Repeatable Read, B's UPDATE fails with "could not serialize access due to concurrent update". Fixes: compute in SQL (`SET stock = stock - 1`), lock with `FOR UPDATE` at step 2, or use a version column.

### 3.5 Write skew: two valid decisions that are invalid together
| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;` checks for bookings of room 7 overlapping July: none | |
| 2 | | Same isolation, same check: none |
| 3 | Inserts a booking for room 7, 1 to 31 July | |
| 4 | | Inserts a booking for room 7, 10 July to 9 August |
| 5 | `COMMIT;` succeeds | `COMMIT;` succeeds: the room is double booked |

Neither transaction modified a row the other read, so snapshot isolation sees no conflict. Prevent it with Serializable (one transaction fails and retries), an exclusion constraint, or by locking a shared parent row (the room) first. Same shape: "at least one manager must stay on duty" with two managers going off duty at once.

## 4. Isolation levels

The SQL standard defines levels by the anomalies they forbid:

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible (per the standard) |
| Serializable | Prevented | Prevented | Prevented |

Real engines differ from the standard, and lost update and write skew matter as much:

| Level | PostgreSQL | MySQL InnoDB |
|---|---|---|
| Read Uncommitted | Behaves like Read Committed | Dirty reads possible |
| Read Committed | **Default.** Each statement sees a fresh snapshot of committed data | Each consistent read takes a fresh snapshot; gap locking mostly disabled |
| Repeatable Read | Snapshot isolation: one snapshot for the whole transaction, no phantoms; updating a row changed by a concurrent transaction raises a serialization error; write skew still possible | **Default.** Plain SELECTs reuse the first read's snapshot; locking reads, UPDATE and DELETE act on the latest committed rows with next-key locks; read-then-write lost updates are possible |
| Serializable | SSI: tracks read/write dependencies and aborts one transaction of a dangerous cycle with `40001`; no extra blocking | Plain SELECTs become shared locking reads when autocommit is off; more blocking and deadlocks |

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;      -- or: SET TRANSACTION ISOLATION LEVEL ... as the first statement
-- ...
COMMIT;
SHOW default_transaction_isolation;      -- 'read committed' unless changed
```

## 5. PostgreSQL specifics

- **Read Committed** (default): good for most web requests. Each statement sees data committed before it started. An `UPDATE` that finds its target row locked waits, then re-checks its WHERE clause against the newest version, so `UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0` is safe here.
- **Repeatable Read**: a stable snapshot for reports and multi-step reads that must agree with each other. Write conflicts on the same row raise `40001`, so callers must be ready to retry.
- **Serializable (SSI)**: the strongest guarantee (results equal some serial order) without lock-based blocking. Every caller needs a retry loop for `40001` errors.

### Going deeper: how SSI decides
Serializable Snapshot Isolation runs transactions on snapshots like Repeatable Read, but also records lightweight predicate locks on what each transaction read. When it sees a dangerous pattern of read/write dependencies between concurrent transactions (the shape behind write skew), it aborts one of them. It can produce false positives (aborting transactions that would have been fine), all participating transactions should use Serializable for the guarantee to hold, and declaring read-only work `READ ONLY` reduces aborts.

## 6. MySQL InnoDB specifics

- **Consistent reads** (plain SELECT) read from a snapshot built from undo logs; under Repeatable Read the snapshot is fixed at the first read.
- **Locking reads** (`SELECT ... FOR UPDATE`, `FOR SHARE`) and `UPDATE`/`DELETE` read the latest committed version (a "current read") and lock index records plus the **gaps** between them (next-key locks), so other transactions cannot insert into a range you scanned. That is how InnoDB avoids phantoms for locking reads.
- Gap locks on ranges or non-unique indexes cause surprising lock waits and deadlocks, for example two transactions inserting into the same gap. Some teams switch to Read Committed to reduce gap locking.
- Because UPDATEs act on the latest version, a stale value read earlier by a plain SELECT can silently overwrite a newer one: InnoDB Repeatable Read does not raise an error for lost updates the way Postgres does. Use atomic updates or `FOR UPDATE`.

## 7. MVCC vs locking

| | MVCC (Postgres, InnoDB, Oracle) | Pure two-phase locking |
|---|---|---|
| Reads | Read a snapshot; never wait for writers | Take shared locks; wait for writers |
| Writes | Lock the rows they change; wait for other writers of the same rows | Exclusive locks; block readers too |
| Cost | Old versions must be cleaned (VACUUM, undo purge) | Lower throughput, more deadlocks under mixed read/write load |

MVCC solves read/write contention, not write/write contention. Two transactions updating the same row still serialize on its lock; that is where explicit locking strategies come in.

## 8. Row locks: FOR UPDATE, FOR SHARE, NOWAIT, SKIP LOCKED

| Clause (Postgres) | Effect |
|---|---|
| `FOR UPDATE` | Exclusive row lock until the transaction ends; blocks other locking reads and writes on those rows |
| `FOR NO KEY UPDATE` | Slightly weaker; what a normal UPDATE takes; does not block foreign-key checks |
| `FOR SHARE` | Shared lock: others may also share-lock, nobody may update |
| `FOR KEY SHARE` | Weakest; what foreign-key checks take |
| `NOWAIT` | Error immediately ("could not obtain lock on row") instead of waiting |
| `SKIP LOCKED` | Silently skip rows another transaction has locked |

A database-backed job queue: many workers grab different jobs without blocking each other.

```sql
WITH next_job AS (
  SELECT id FROM jobs
  WHERE status = 'queued' AND run_at <= now()
  ORDER BY run_at
  LIMIT 1
  FOR UPDATE SKIP LOCKED                 -- a job locked by another worker is invisible here
)
UPDATE jobs SET status = 'running', locked_at = now()
FROM next_job
WHERE jobs.id = next_job.id
RETURNING jobs.*;
```

Database-backed queues such as Rails 8's Solid Queue are built on this pattern. `SKIP LOCKED` gives an intentionally incomplete view, so use it for work distribution, not for reads that must see every row.

## 9. Optimistic vs pessimistic locking

**Optimistic**: assume conflicts are rare; detect them at write time.
```sql
SELECT id, title, lock_version FROM listings WHERE id = 7;     -- lock_version = 3
UPDATE listings
SET title = 'Sunny room near the station', lock_version = lock_version + 1
WHERE id = 7 AND lock_version = 3;                             -- 0 rows updated = someone else won
```
Rails does this automatically when a table has an integer `lock_version` column and raises `ActiveRecord::StaleObjectError` on conflict. In HTTP APIs the same idea appears as `ETag` / `If-Match`.

**Pessimistic**: lock first, then read and write.
```sql
BEGIN;
SELECT * FROM listings WHERE id = 7 FOR UPDATE;   -- other writers wait here
UPDATE listings SET title = '...' WHERE id = 7;
COMMIT;
```

| | Optimistic | Pessimistic |
|---|---|---|
| Mechanism | Version check on write | Row lock on read (`FOR UPDATE`) |
| Best when | Conflicts are rare; long think time (a user editing a form) | Conflicts are frequent; short critical sections (stock, balances) |
| Cost of conflict | Retry or show "someone else changed this" | Waiting; risk of deadlocks; blocked connections |
| Holds a connection while the user thinks | No | Never do this with pessimistic locks |

## 10. Deadlocks

| Step | Session A | Session B |
|---|---|---|
| 1 | `BEGIN; UPDATE products SET stock = stock - 1 WHERE id = 1;` (locks row 1) | |
| 2 | | `BEGIN; UPDATE products SET stock = stock - 1 WHERE id = 2;` (locks row 2) |
| 3 | `UPDATE products SET stock = stock - 1 WHERE id = 2;` waits for B | |
| 4 | | `UPDATE products SET stock = stock - 1 WHERE id = 1;` waits for A: a cycle |
| 5 | After `deadlock_timeout`, Postgres aborts one: "ERROR: deadlock detected" (`40P01`) | The other proceeds |

This is exactly what happens when two carts containing the same two products check out in different item orders. Prevention:
- **Lock in a consistent order**: sort ids, or lock everything up front: `SELECT id FROM products WHERE id = ANY($1) ORDER BY id FOR UPDATE;`. A single `UPDATE ... WHERE id IN (...)` locks rows in whatever order it scans them, so it can still deadlock against another such statement.
- Keep transactions short and touch fewer rows.
- Set `lock_timeout` so waits fail fast, and **retry** the aborted transaction (deadlocks are normal under load; the bug is not handling them).

## 11. Advisory locks

Advisory locks lock an application-chosen number rather than a row: cooperative mutual exclusion inside Postgres.

```sql
SELECT pg_try_advisory_lock(17);       -- session-level: true if acquired; held until unlock or disconnect
SELECT pg_advisory_unlock(17);

BEGIN;
SELECT pg_advisory_xact_lock(1, 17);   -- transaction-level, two-key form: (feature 1 = "price sync", store 17)
-- ... sync this store's prices; a second worker for store 17 waits here ...
COMMIT;                                -- released automatically
```

Uses: one sync job per store at a time, singleton cron work across several servers, serializing work on a resource that has no single row to lock. Rails takes an advisory lock while running migrations. Behind PgBouncer transaction pooling, use only the `_xact_` variants.

## 12. Long transactions and timeouts

Long transactions hold row locks (others queue behind them), hold back VACUUM (bloat, see [PostgreSQL](04-postgresql.md)), raise conflict rates under Repeatable Read and Serializable, and pin pooled connections.

- Never call external services (a payment provider, the PriceCharting API, email) inside a database transaction. Do the DB work, commit, then call out; or record intent in a table and let a job do the call.
- Process large backfills in batches, each in its own short transaction.
- Guard rails: `statement_timeout` (kill runaway queries), `lock_timeout` (fail fast when waiting for locks, great for migrations), `idle_in_transaction_session_timeout` (kill sessions that opened a transaction and went quiet).
- **Enqueue jobs after commit.** A Sidekiq job enqueued inside a transaction can start before the commit and fail to find the record (or see old data). Use `after_commit` / `after_create_commit` callbacks, or enqueue after the transaction block. Recent Rails versions can defer Active Job enqueues until commit; check your version's configuration. For guaranteed delivery, write an outbox row in the same transaction and publish it from a job ([background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md)).

## 13. Worked examples

### 13.1 Inventory decrement (no overselling)
```sql
-- Best default: one atomic, conditional statement
UPDATE products
SET stock = stock - 2
WHERE id = 42 AND stock >= 2
RETURNING stock;                         -- no row returned = not enough stock

-- When app code must read and decide first: lock, check, write
BEGIN;
SELECT stock FROM products WHERE id = 42 FOR UPDATE;   -- concurrent buyers wait here
UPDATE products SET stock = stock - 2 WHERE id = 42;     -- after the app checks stock >= 2
COMMIT;

-- Safety net that turns any remaining bug into an error instead of negative stock
ALTER TABLE products ADD CONSTRAINT stock_not_negative CHECK (stock >= 0);
```

```ruby
updated = Product.where(id: product_id).where("stock >= ?", qty)
                 .update_all(["stock = stock - ?", qty])   # returns the number of rows changed
raise OutOfStock if updated.zero?
```

For a multi-item order, lock or update the products in id order. When the same item sells on several channels (a store's website and a marketplace), the decrement must be this atomic statement, and imported channel orders should carry a unique external order id so a re-run sync cannot apply the same sale twice.

### 13.2 Preventing a double rent payment
Threats: a double tap on "Pay", a mobile retry after a timeout, two devices, a webhook delivered twice. Defend in layers:

```sql
CREATE UNIQUE INDEX rent_payments_idempotency_uq ON rent_payments (idempotency_key);
CREATE UNIQUE INDEX one_paid_rent_per_period
  ON rent_payments (lease_id, period) WHERE status = 'succeeded';
```

```ruby
def start_rent_payment(lease, period, key)
  lease.with_lock do                                       # BEGIN; SELECT ... FOR UPDATE on the lease
    lease.rent_payments.find_by(period: period, status: %w[pending succeeded]) ||
      lease.rent_payments.create!(period: period, amount_cents: lease.rent_cents,
                                  status: "pending", idempotency_key: key)
  end
rescue ActiveRecord::RecordNotUnique
  RentPayment.find_by!(idempotency_key: key)               # the same request already created it
end
```

1. The client generates an **idempotency key** when the payment screen opens and resends it on retries; the unique index turns duplicates into a lookup.
2. Locking the lease row serializes concurrent attempts for that lease and period.
3. The payment provider is called **after** this transaction commits, passing the same idempotency key (most providers support one), so a retried request cannot charge twice.
4. The webhook handler is idempotent (unique provider event id), and the partial unique index guarantees at most one succeeded payment per lease per month even if every other layer fails.

### 13.3 Booking the last room
Two residents try to book the same room for overlapping dates at the same moment (the write skew from 3.5).

| Approach | How | Notes |
|---|---|---|
| Exclusion constraint (best) | `EXCLUDE USING gist (room_id WITH =, stay WITH &&)` on `room_bookings` | The database rejects the overlap at any isolation level; catch SQLSTATE `23P01` (`PG::ExclusionViolation` in Ruby) and show "just booked" |
| Lock the parent row | `SELECT ... FROM rooms WHERE id = 7 FOR UPDATE;` then check overlaps, insert, commit | Works under the default Read Committed (the check after the wait sees the other booking); keep it short |
| Serializable + retry | Run check and insert as `SERIALIZABLE` | One transaction gets `40001`; its retry sees the booking and refuses |
| Capacity counter (events, seats) | `UPDATE events SET seats_taken = seats_taken + 1 WHERE id = $1 AND seats_taken < capacity RETURNING seats_taken;` | Atomic; no row returned = full |

## 14. App-level equivalents

**Rails (ActiveRecord)**
```ruby
Order.transaction do                                  # BEGIN ... COMMIT; any exception rolls back
  order.update!(status: "paid")
  order.payments.create!(amount_cents: order.total_cents)
end

Order.transaction(requires_new: true) { ... }         # nested: a SAVEPOINT
Order.transaction(isolation: :serializable) { ... }   # set the level for this transaction
product.with_lock { product.update!(stock: product.stock - 1) }   # BEGIN; SELECT ... FOR UPDATE (reload)
Product.lock.find(id)                                 # FOR UPDATE inside an open transaction
Job.lock("FOR UPDATE SKIP LOCKED").where(status: "queued").order(:run_at).first

def with_retry(attempts: 3)
  ActiveRecord::Base.transaction(isolation: :serializable) { yield }
rescue ActiveRecord::SerializationFailure, ActiveRecord::Deadlocked
  attempts -= 1
  retry if attempts.positive?                         # re-runs the whole method body
  raise
end
```
Gotcha: raising `ActiveRecord::Rollback` inside a nested block without `requires_new` rolls back nothing (the inner block swallows it and the outer transaction commits).

**Django**
```python
from django.db import transaction
from django.db.models import F

with transaction.atomic():
    product = Product.objects.select_for_update().get(pk=product_id)   # FOR UPDATE
    if product.stock < qty:
        raise OutOfStock()
    product.stock = F("stock") - qty                                   # computed in SQL
    product.save(update_fields=["stock"])
    transaction.on_commit(lambda: send_receipt.delay(order_id))        # enqueue after commit

with transaction.atomic():
    job = Job.objects.select_for_update(skip_locked=True).filter(status="queued").first()
```

**Drizzle** (check the docs for your version's exact options)
```ts
import { eq, sql } from "drizzle-orm";

await db.transaction(async (tx) => {
  const [product] = await tx.select().from(products)
    .where(eq(products.id, productId))
    .for("update");                                     // SELECT ... FOR UPDATE
  if (!product || product.stock < qty) tx.rollback();   // throws and rolls back
  await tx.update(products)
    .set({ stock: sql`${products.stock} - ${qty}` })
    .where(eq(products.id, productId));
}, { isolationLevel: "read committed" });
```
`db.transaction` resolves with the callback's return value and rejects if it throws (including `tx.rollback()`); nested `tx.transaction(...)` uses savepoints. See [Drizzle ORM](13-drizzle-orm.md).

## Common pitfalls
- Read-modify-write in app code (`product.stock -= 1; product.save!`) without a lock or version: lost updates under load.
- `validates :email, uniqueness: true` without a unique index: two concurrent requests both pass.
- Calling a payment API or sending email inside a transaction, then rolling back after the side effect happened.
- Enqueuing Sidekiq jobs inside a transaction: the job runs before commit and cannot find the row.
- Not retrying `40001` serialization failures and `40P01` deadlocks; treating them as fatal bugs.
- Locking rows in different orders in different code paths.
- Holding a pessimistic lock across user think time or an HTTP call.
- Assuming Repeatable Read prevents everything: write skew still happens.
- Using session-level advisory locks or `SET` behind transaction-mode PgBouncer.
- Expecting `ActiveRecord::Rollback` in a nested block to undo anything.

## How to talk about this in an interview
- Use your domains: rent payments and split bills in the Coloc app (double-submit and idempotency), CrystalCommerce stock that can sell on a store's site and through PriceCharting (atomic decrements, idempotent order import), Sidekiq jobs that must be safe to run twice.
- Lead with schema-level guarantees (unique and partial unique indexes, CHECK, exclusion constraints, atomic updates), then locks, then isolation levels. That ordering shows judgment.
- If you have not personally debugged a deadlock in production, say so, then explain how you would: read the deadlock log entry, find the two statements, and enforce a lock order.

> **Fill in:** how rent payments in the Coloc app guard against double submission (idempotency keys, unique constraints, provider features) as far as you know the implementation.

> **Fill in:** whether CrystalCommerce's PriceCharting order sync uses unique external ids or locks to avoid applying an order twice.

> **Fill in:** a race condition or duplicate-record bug you saw on a Devntech project and how it was fixed.

## Interview questions

**Q1. What is a transaction, and what does ACID mean?**
A transaction groups statements into one unit that commits entirely or not at all. Atomicity: all or nothing. Consistency: constraints hold before and after. Isolation: concurrent transactions do not see each other's partial work (to the degree the isolation level promises). Durability: committed data survives a crash, because the WAL is flushed at commit.

**Q2. Explain the isolation levels and the anomalies each prevents.**
Read Uncommitted allows dirty reads. Read Committed prevents dirty reads but allows non-repeatable reads and phantoms. Repeatable Read also prevents non-repeatable reads (and in Postgres, phantoms) but allows write skew. Serializable prevents all of them: results match some serial order. Higher levels mean more blocking or more aborted transactions to retry.
*Follow-up:* What is Postgres's default? Read Committed. MySQL InnoDB's? Repeatable Read.

**Q3. How does Repeatable Read differ between Postgres and MySQL?**
Postgres gives true snapshot isolation: one snapshot per transaction, no phantoms, and an UPDATE of a row changed concurrently fails with a serialization error. InnoDB uses a snapshot only for plain SELECTs; UPDATEs and locking reads see the latest data and take next-key locks, so phantoms are blocked for locking reads but lost updates from stale reads are possible and gap locks cause extra waits and deadlocks.

**Q4. What is a lost update, and how do you prevent it?**
Two transactions read the same value, each computes a new value in app code, and the second write overwrites the first. Prevent it with an atomic SQL update (`SET stock = stock - 1`), a row lock (`SELECT ... FOR UPDATE`), optimistic locking with a version column, or Repeatable Read/Serializable plus retries in Postgres.

**Q5. What is write skew? Give an example.**
Two transactions read the same data, each makes a decision that is valid alone, and they write different rows, so together they break an invariant. Example: two bookings for the same room each check "no overlap", then both insert. Snapshot isolation does not catch it; Serializable, an exclusion constraint, or locking a common parent row does.

**Q6. Optimistic or pessimistic locking: when do you use each?**
Optimistic (version column, retry on conflict) when conflicts are rare or when a user holds data for a long time, like editing a listing in a form. Pessimistic (`FOR UPDATE`) when conflicts are frequent and the critical section is short, like decrementing stock or balances. Rails supports both: `lock_version` and `lock` / `with_lock`.

**Q7. What is a deadlock, and how do you prevent it?**
Two or more transactions each hold a lock the other needs, so none can proceed. Postgres detects the cycle and aborts one with "deadlock detected". Prevent with a consistent lock order (sort ids, lock up front), short transactions, fewer locked rows and `lock_timeout`; and always retry the aborted transaction.

**Q8. What does `SELECT ... FOR UPDATE SKIP LOCKED` do, and where is it useful?**
It locks matching rows but skips rows already locked by other transactions instead of waiting. Several workers can then pull different jobs from the same table without blocking, which is how database-backed queues (like Solid Queue) work. Do not use it for reads that must see every row.

**Q9. How do you prevent overselling inventory?**
Make the decrement a single conditional statement: `UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1 RETURNING stock` and treat "no row" as out of stock. Add `CHECK (stock >= 0)` as a safety net. For multi-item orders, lock rows in id order. If decisions need app logic, lock with `FOR UPDATE` in a short transaction.

**Q10. How do you prevent a double payment when a user taps "Pay" twice?**
An idempotency key generated by the client and stored under a unique index (retries become lookups), a row lock or unique constraint for "one payment per lease per period", calling the provider outside the transaction with the same idempotency key, and idempotent webhook handling keyed by the provider's event id.

**Q11. What are savepoints, and how do nested transactions work in Rails?**
A savepoint marks a point inside a transaction you can roll back to without abandoning the whole transaction. In Rails, nested `transaction` blocks join the outer transaction by default; `requires_new: true` creates a savepoint, so an error inside rolls back only that block.
*Follow-up:* What happens if you raise `ActiveRecord::Rollback` in a plain nested block? Nothing is rolled back; the inner block swallows it and the outer transaction commits.

**Q12. Why are long-running transactions harmful?**
They hold locks that block other writers, prevent VACUUM from removing dead rows (bloat), increase serialization conflicts, and tie up pooled connections. Typical cause: doing slow work (HTTP calls, file processing) while a transaction is open. Keep transactions short and set `idle_in_transaction_session_timeout`.

**Q13. What are advisory locks?**
Locks on application-defined numbers, managed by Postgres but meaningful only to your code. `pg_advisory_xact_lock(key)` holds one until the transaction ends; `pg_try_advisory_lock` returns immediately with true or false. Use them for "only one sync per store at a time" or singleton scheduled jobs.

**Q14. Why must background jobs be enqueued after commit?**
Sidekiq workers run in other processes and can pick the job up before your transaction commits; they then cannot find the new record, or see stale data, and the job fails or does the wrong thing. If the transaction rolls back, the job should not run at all. Use `after_commit` callbacks or enqueue after the transaction block (or a transactional outbox).

**Q15. How do you handle serialization failures?**
Treat SQLSTATE `40001` (and deadlocks, `40P01`) as retryable: rerun the whole transaction a few times with a small backoff, and make sure it has no external side effects inside it. In Rails, rescue `ActiveRecord::SerializationFailure` and `ActiveRecord::Deadlocked`.

**Q16. MVCC vs locking: what is the difference?**
With MVCC, writers create new row versions and readers see a consistent snapshot, so reads never wait for writes. Pure lock-based schemes make readers take shared locks that conflict with writers. MVCC still locks rows for writes, so two writers on the same row wait for each other.

**Q17. How do you prevent two people from booking the same room for overlapping dates?**
Best: a range column with an exclusion constraint (`EXCLUDE USING gist (room_id WITH =, stay WITH &&)`), so the database rejects overlaps whatever the isolation level. Alternatives: lock the room row with `FOR UPDATE` before checking (under Read Committed), or run the check and insert as Serializable with retries.

**Q18. Does Read Committed make `UPDATE ... SET stock = stock - 1` safe?**
Yes. The second UPDATE waits for the first to commit, then re-evaluates its WHERE clause on the new row version and applies the decrement to the latest value. What Read Committed does not protect is reading a value in one statement and writing a computed value in another.

## Further reading
- PostgreSQL docs, transaction isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- Explicit locking (row locks, advisory locks, deadlocks): https://www.postgresql.org/docs/current/explicit-locking.html
- Rails API, ActiveRecord transactions: https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html
