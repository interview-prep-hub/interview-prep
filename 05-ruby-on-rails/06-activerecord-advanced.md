# ActiveRecord Advanced: Transactions, Locking, Constraints and Scale

> **Tier:** T1 (know cold) · **Related:** [ActiveRecord basics](03-activerecord-basics.md), [Querying and N+1](05-activerecord-querying-n-plus-1.md), [Background jobs](09-background-jobs-sidekiq.md), [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md), [Indexing](../12-databases/05-indexing-and-query-optimization.md), [PostgreSQL](../12-databases/04-postgresql.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md)
> **On your resume:** PostgreSQL and background jobs in CrystalCommerce (stock, prices and orders synced with PriceCharting), rent payments and split bills in Colochousing, and the multi-tenant platform migration work at Devntech.

## Quick revision
- `transaction do ... end`: any exception rolls back and re-raises; `raise ActiveRecord::Rollback` rolls back silently. Use bang methods inside so failures actually raise.
- Nested `transaction` blocks join the outer transaction by default; `requires_new: true` creates a savepoint so the inner block can roll back alone.
- Transactions cover one database connection only: not Redis, emails or HTTP calls. Never make slow external calls while holding a transaction open.
- A job enqueued inside a transaction can run before COMMIT (`RecordNotFound`) or after a rollback (a job for data that never existed). Enqueue after commit (`after_commit`, or after the block).
- **Optimistic locking:** a `lock_version` column; a stale update raises `ActiveRecord::StaleObjectError`. Best when conflicts are rare (edit forms).
- **Pessimistic locking:** `lock` / `lock!` / `with_lock` issue `SELECT ... FOR UPDATE` inside a transaction; other writers wait. Best for short, hot critical sections (stock, balances).
- Overselling fix: an atomic conditional UPDATE (`WHERE stock >= qty`), or `with_lock` plus a re-check, plus a `CHECK (stock >= 0)` constraint as a safety net.
- Double-payment fix: idempotency key with a unique index, claim the row with a lock and a status check, call the provider outside the transaction with its idempotency key.
- `FOR UPDATE SKIP LOCKED` lets several workers take different rows without waiting (job queues, work distribution).
- `insert_all` / `upsert_all(unique_by:)` write thousands of rows in one statement but skip validations and callbacks.
- Validations give friendly errors; **constraints** (`null: false`, unique indexes, foreign keys, `add_check_constraint`) guarantee integrity even for bulk writes, races and raw SQL. Use both.
- Big-table index: `algorithm: :concurrently` with `disable_ddl_transaction!`. The strong_migrations gem flags unsafe migrations.
- Connection pool >= threads per process (Puma threads, Sidekiq concurrency); total connections must fit under Postgres `max_connections`.
- Replicas: `connects_to database: { writing: :primary, reading: :primary_replica }` and `connected_to(role: :reading) { ... }`; watch replication lag.
- JSONB + `store_accessor` for flexible data, GIN indexes for querying it; `encrypts :iban` (7.0) for encryption at rest, `deterministic: true` when you must query by it.
- Avoid `default_scope`; prefer explicit scopes. Soft delete with the discard gem (explicit `kept` scope) or paranoia (default scope).

## 1. Transactions

A transaction makes several writes all-or-nothing. Rails already wraps every `save` and `destroy` (including their callbacks) in one; you add your own when one business action touches several rows.

```ruby
ActiveRecord::Base.transaction do
  order.update!(status: :paid)
  Payment.create!(order: order, amount_cents: order.total_cents)
  order.customer.increment!(:paid_orders_count)   # atomic UPDATE ... SET col = col + 1
end   # COMMIT here; any exception above means ROLLBACK, and the exception propagates
```

- `Order.transaction`, `order.transaction` and `ActiveRecord::Base.transaction` all use the same connection (with one database).
- Exceptions roll back and re-raise. `raise ActiveRecord::Rollback` rolls back and is swallowed by the block.
- Non-bang methods return false instead of raising, so `order.update(...)` failing inside the block does **not** roll anything back.
- Pass an isolation level when needed: `transaction(isolation: :serializable)` (Postgres defaults to Read Committed; see [transactions and concurrency](../12-databases/06-transactions-and-concurrency.md)).

**Nesting and savepoints.** An inner `transaction` block joins the outer one by default, which leads to a famous surprise:

```ruby
User.transaction do
  User.create!(name: "Ayesha")
  User.transaction do
    User.create!(name: "Bilal")
    raise ActiveRecord::Rollback    # swallowed by the inner block; no savepoint exists
  end
end
# Both users are committed!

User.transaction do
  User.create!(name: "Ayesha")
  User.transaction(requires_new: true) do   # SAVEPOINT
    User.create!(name: "Bilal")
    raise ActiveRecord::Rollback            # ROLLBACK TO SAVEPOINT: only Bilal is undone
  end
end
```

Keep transactions **short**: they hold row locks and a pooled connection until COMMIT. An HTTP call to a payment provider or a slow API inside a transaction blocks other writers and can exhaust the pool.

## 2. Side effects inside transactions

Redis, email and HTTP are outside the database transaction, so they cannot be rolled back and they can observe uncommitted state:

```ruby
ActiveRecord::Base.transaction do
  order = Order.create!(order_params)
  OrderConfirmationJob.perform_later(order.id)   # pushed to Redis immediately, before COMMIT
  reserve_stock!(order)                          # raises: ROLLBACK, but the job is already queued
end
```

Two failure modes: a fast Sidekiq thread runs the job before the COMMIT and gets `RecordNotFound` (it cannot see uncommitted rows), or the transaction rolls back and the job emails a customer about an order that does not exist. Fixes:
- Enqueue after the transaction block returns, or in `after_commit` / `after_create_commit` callbacks, which run only after a successful COMMIT.
- Newer Rails versions can make Active Job defer enqueues until the surrounding transaction commits; the setting and its default changed between 7.2 and 8.0, so check your version's release notes before relying on it. Plain `Sidekiq::Job` classes know nothing about database transactions.
- Make the job tolerate it anyway: look the record up with `find_by` and exit quietly, or retry, if it is missing.

### Going deeper: the transactional outbox
When an event must never be lost (for example "order paid" must reach an external channel), write an `outbox_events` row in the **same** transaction as the business change, and let a separate job or poller publish unsent rows and mark them sent. The database commit then guarantees the event exists if and only if the change happened. See [background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md).

## 3. Locking: optimistic vs pessimistic

**Optimistic locking** assumes conflicts are rare. Add an integer `lock_version` column and Active Record does the rest:

```ruby
add_column :listings, :lock_version, :integer, null: false, default: 0   # migration

# Every update becomes:
# UPDATE listings SET price_cents = 1299, lock_version = 6 WHERE id = 42 AND lock_version = 5
# If no row matched (someone else saved first), Rails raises ActiveRecord::StaleObjectError.

def update
  @listing = current_store.listings.find(params[:id])
  @listing.update!(listing_params)            # permit :lock_version; the form sends it in a hidden field
  redirect_to @listing
rescue ActiveRecord::StaleObjectError
  @listing.reload
  flash.now[:alert] = "Someone else changed this listing. Review the latest values and save again."
  render :edit, status: :conflict
end
```

**Pessimistic locking** takes a row lock up front, so concurrent writers wait in line:

```ruby
Listing.transaction do
  listing = Listing.lock.find(id)            # SELECT ... FOR UPDATE
  listing.update!(stock: listing.stock - 1)
end

listing.with_lock do                         # opens a transaction, reloads with FOR UPDATE, yields
  listing.update!(stock: listing.stock - 1)
end

Listing.lock("FOR UPDATE NOWAIT").find(id)   # fail immediately instead of waiting
Task.where(status: :queued).lock("FOR UPDATE SKIP LOCKED").limit(10)  # take rows nobody else holds
```

| | Optimistic | Pessimistic |
|---|---|---|
| Mechanism | Version check at UPDATE time | Row lock at read time (`FOR UPDATE`) |
| Conflict handling | Raises `StaleObjectError`; you reload, merge or retry | Others block until COMMIT |
| Best for | Rare conflicts, long human edits (forms) | Frequent conflicts on hot rows, short critical sections |
| Costs | Retries and UX for conflicts | Lock waits, possible deadlocks, needs a transaction |

`SKIP LOCKED` lets many workers pull from the same table without stepping on each other; database-backed queues such as Solid Queue are built on it.

### Preventing overselling stock

Two buyers read `stock = 1`, both pass the check and both decrement: stock becomes -1 and two orders ship one item. Fixes, from simplest to most flexible:

```ruby
# 1. Atomic conditional UPDATE: no read-modify-write window at all
rows = Listing.where(id: listing_id).where("stock >= ?", qty)
              .update_all(["stock = stock - ?", qty])
raise OutOfStockError if rows.zero?

# 2. Pessimistic lock + re-check inside the lock (when you need Ruby logic)
listing.with_lock do
  raise OutOfStockError if listing.stock < qty
  listing.update!(stock: listing.stock - qty)
end

# 3. Safety net in the schema, whatever the code does
add_check_constraint :listings, "stock >= 0", name: "listings_stock_non_negative"
```

When stock also changes from an external channel sync, the same rules apply to the sync job: decrement atomically and treat "out of stock" as a normal outcome, not a crash.

### Preventing double payments

A double click, a mobile retry on a flaky network or a re-run job can charge twice. Combine an idempotency key, a short locked claim and the provider's own idempotency support:

```ruby
# rent_payments: status enum (pending, processing, paid, failed), unique index on idempotency_key
def pay!(rent_payment, idempotency_key:)
  claimed = rent_payment.with_lock do
    next false unless rent_payment.pending?               # someone already claimed or paid it
    rent_payment.update!(status: :processing, idempotency_key: idempotency_key)
    true
  end
  return rent_payment unless claimed

  result = PaymentGateway.charge(amount_cents: rent_payment.amount_cents,
                                 idempotency_key: idempotency_key)   # outside the transaction
  rent_payment.update!(status: result.success? ? :paid : :failed, provider_ref: result.id)
end
```

The lock is held only for the quick status flip; the slow HTTP call happens after COMMIT, and a retried request with the same key either finds the payment already claimed or is de-duplicated by the provider. (`PaymentGateway` stands in for your provider's SDK.)

## 4. Bulk operations

Creating 50,000 listings with `create!` in a loop means 50,000 INSERTs, validations and callbacks. Bulk methods send one statement:

```ruby
rows = api_items.map do |item|
  { store_id: store.id, product_id: item.product_id, condition: item.condition, price_cents: item.price_cents }
end

Listing.insert_all(rows)       # INSERT ... ON CONFLICT DO NOTHING: duplicates are skipped silently
Listing.insert_all!(rows)      # raises on a duplicate
Listing.upsert_all(rows, unique_by: %i[store_id product_id condition])  # INSERT ... ON CONFLICT (...) DO UPDATE
Listing.upsert_all(rows, unique_by: %i[store_id product_id condition], update_only: [:price_cents])

Listing.where(store_id: store.id, active: false).delete_all
Listing.where(store_id: store.id).in_batches(of: 5_000).update_all(needs_sync: true)
```

What you give up: no validations, no callbacks, no model instances, and every row hash must have the same keys. `unique_by:` names the columns (or index) of a **unique index** that decides what counts as a conflict (honored on PostgreSQL and SQLite; MySQL uses any unique key). Rails 7.0+ fills `created_at` / `updated_at` automatically for these methods. Because validations are skipped, clean and validate the data in Ruby first and let database constraints reject anything that slips through. `upsert_all` keyed on an external ID is the backbone of an idempotent sync job: running it twice produces the same rows.

## 5. Database constraints vs validations

| | Validations (Ruby) | Constraints (database) |
|---|---|---|
| Where they run | In the model before SQL is sent | In the database on every write |
| Failure | Friendly `errors` for forms and API responses | Exceptions: `RecordNotUnique`, `NotNullViolation`, `InvalidForeignKey`, other `StatementInvalid` subclasses |
| Bypassed by | `update_column(s)`, `update_all`, `insert_all`, raw SQL, other apps sharing the DB, race conditions | Nothing short of dropping the constraint |
| Concurrency-safe | No (uniqueness is check-then-insert) | Yes |
| Good for | User feedback and rules needing Ruby logic or context | Invariants that must never break |

Use both: validations for the user experience, constraints for integrity.

```ruby
class HardenListings < ActiveRecord::Migration[7.1]
  def change
    change_column_null :listings, :price_cents, false
    add_index :listings, [:store_id, :product_id, :condition], unique: true
    add_foreign_key :channel_listings, :listings                        # no orphans
    add_check_constraint :listings, "stock >= 0", name: "listings_stock_non_negative"
    add_check_constraint :listings, "price_cents >= 0", name: "listings_price_non_negative"
  end
end
```

`add_check_constraint` arrived in Rails 6.1. Adding constraints to an existing big table can fail on dirty data or lock the table, so clean the data first; Postgres can add some constraints as `NOT VALID` and validate them later, which strong_migrations recommends.

## 6. Indexes and safe migrations

```ruby
add_index :orders, [:store_id, :created_at]                            # composite
add_index :listings, :sku, unique: true, where: "archived_at IS NULL"  # partial (Postgres)
add_index :users, "lower(email)", unique: true                         # expression index
add_index :products, :metadata, using: :gin                            # JSONB containment queries
```

- **Composite column order matters** (leftmost prefix): `(store_id, created_at)` serves `WHERE store_id = ?`, `WHERE store_id = ? ORDER BY created_at`, but not `WHERE created_at > ?` alone.
- **Partial indexes** index only some rows: smaller, and a partial unique index lets an archived row and a live row share a SKU.
- Every index slows writes and uses disk; index foreign keys and real query patterns, not every column.

On a large, busy table, a normal `CREATE INDEX` blocks writes until it finishes. Build it concurrently, which Postgres cannot do inside a transaction:

```ruby
class AddIndexToListingsSku < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def change
    add_index :listings, :sku, algorithm: :concurrently
  end
end
```

If a concurrent build fails it leaves an invalid index behind that you must drop before retrying. The **strong_migrations** gem catches dangerous operations at development time (non-concurrent indexes, removing a column the app still reads, changing a column type, adding a foreign key that locks both tables) and prints the safe recipe, such as adding `self.ignored_columns += ["old_column"]` to the model and deploying before removing the column. More in [legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md).

## 7. Connection pool sizing

Each Ruby process keeps a pool of database connections (`pool:` in `database.yml`). A thread checks one out while it talks to the database; when none is free it waits, then raises `ActiveRecord::ConnectionTimeoutError` (about 5 seconds by default).

| Process | Threads needing connections | Pool per process |
|---|---|---|
| Puma worker | `RAILS_MAX_THREADS` (for example 5) | >= 5 |
| Sidekiq process | `concurrency` (recent versions default to 5) | >= concurrency (plus a few if jobs spawn threads) |

Total connections = web servers x Puma workers x threads + Sidekiq processes x concurrency + consoles, cron and migrations. Example: 3 servers x 2 workers x 5 threads = 30, plus 2 Sidekiq processes x 10 = 20, so 50 connections, which must stay under Postgres `max_connections` (100 by default) with headroom. If it does not fit, reduce threads or add PgBouncer (with transaction pooling, watch out for features that need a session, such as prepared statements and session-level advisory locks). Threads you create yourself must return their connection: wrap their work in `ActiveRecord::Base.connection_pool.with_connection { ... }`.

## 8. Multiple databases and read replicas

Rails 6+ supports several databases per app, including replicas:

```yaml
# config/database.yml
production:
  primary:
    url: <%= ENV["DATABASE_URL"] %>
  primary_replica:
    url: <%= ENV["DATABASE_REPLICA_URL"] %>
    replica: true
```

```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary_replica }
end

ActiveRecord::Base.connected_to(role: :reading) do
  SalesReport.build_for(store)        # heavy reads go to the replica
end
```

Rails can also switch roles automatically per request (reads to the replica for GET requests, unless the user wrote recently) through the database selector middleware that `bin/rails g active_record:multi_db` sets up. The classic replica bug is **replication lag**: a user saves, gets redirected, and the replica has not caught up, so the change "disappears". Read your own writes from the primary for a short window.

### Going deeper: sharding
Horizontal sharding (`connects_to shards:`, Rails 6.1) splits one logical table across several databases, for example by tenant. It is rarely needed at 1-3 YOE scale; multi-tenancy is usually a `tenant_id` column with scoped queries (see [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)).

## 9. JSONB, arrays, encryption, default_scope and soft delete

**JSONB and arrays (PostgreSQL):**

```ruby
create_table :products do |t|
  t.jsonb  :metadata, null: false, default: {}
  t.string :tags, array: true, null: false, default: []
end

class Product < ApplicationRecord
  store_accessor :metadata, :region, :release_year     # product.region = "PAL"
end

Product.where("metadata ->> 'region' = ?", "PAL")
Product.where("metadata @> ?", { region: "PAL" }.to_json)   # containment, uses a GIN index
Product.where("? = ANY(tags)", "rpg")
```

JSONB suits flexible or external data: raw API payloads kept for debugging a sync, per-store settings, attributes that vary by product type. Avoid it for data you filter, join or constrain constantly: you lose types, foreign keys and simple validations. `store_accessor` values are whatever JSON holds (often strings), so cast them or declare types with the attribute API.

**Encrypted attributes (Rails 7.0):**

```ruby
class Resident < ApplicationRecord
  encrypts :iban                          # non-deterministic: strongest, cannot be queried
  encrypts :email, deterministic: true    # same input -> same ciphertext, so where(email:) works
end
```

Run `bin/rails db:encryption:init` and store the generated keys in credentials. Values are encrypted before they reach the database and decrypted on read, which protects backups and SQL dumps; deterministic mode trades some secrecy (equal values are recognizable) for queryability.

**default_scope, and why to avoid it:**

```ruby
class Listing < ApplicationRecord
  default_scope { where(archived_at: nil) }   # applied to every query
end
```

It silently changes every query, including associations and joins; its `where` values become defaults for `Listing.new`; and escaping it with `unscoped` removes **all** conditions, including association scoping, so `store.listings.unscoped` returns listings from every store. Once added, it is very hard to remove from a legacy app. Prefer an explicit `scope :live, -> { where(archived_at: nil) }`.

**Soft delete** keeps rows but marks them deleted, for audit trails, undo or legal reasons:
- **discard** gem: `include Discard::Model` with a `discarded_at` column gives `record.discard`, `undiscard`, and explicit `kept` / `discarded` scopes, without a default scope.
- **paranoia** gem: `acts_as_paranoid` overrides `destroy` to set `deleted_at` and hides deleted rows with a default scope (`really_destroy!` deletes for real).
- Either way: make unique indexes partial (`WHERE deleted_at IS NULL`), exclude deleted rows from counts and associations, and remember that privacy laws may still require real deletion.

## Common pitfalls
- Using non-bang `save` / `update` inside a transaction, so a failure commits half the work.
- Expecting `raise ActiveRecord::Rollback` in a nested block to roll back without `requires_new: true`.
- Enqueuing jobs, sending emails or calling APIs inside a transaction.
- Read-check-write on stock or balances (`if listing.stock >= qty then update`) without a lock or an atomic UPDATE.
- Holding a row lock while calling a slow external API.
- Calling `lock!` outside a transaction (the lock is released immediately).
- Assuming `insert_all` / `upsert_all` run validations or callbacks.
- Building a big index without `algorithm: :concurrently` and blocking writes during a deploy.
- Sizing the pool from Puma alone and forgetting Sidekiq processes and consoles.
- Adding `default_scope` to implement soft delete or tenancy in a legacy app.

## How to talk about this in an interview
- CrystalCommerce syncs stock, prices and orders with PriceCharting: perfect ground for atomic stock updates, idempotent upserts keyed on external IDs, unique indexes, and enqueuing sync jobs only after commit.
- Colochousing rent payments and split bills are the natural double-payment story: idempotency keys, a status machine and short locked claims. Describe the pattern; claim only what you actually built.
- For the multi-tenant platform migration, connection limits, data integrity constraints and safe migrations are reasonable topics if they came up.
- Show judgment: "I would start with an atomic UPDATE and a check constraint, and move to explicit locking only when the logic needs Ruby."

> **Fill in:** how stock or order state was kept consistent between CrystalCommerce and PriceCharting (locks, atomic updates, reconciliation), if you know.

> **Fill in:** how the Colochousing app prevented duplicate rent payments or duplicate bill shares, and whether you touched that code.

## Interview questions

**Q1. How do transactions work in Rails, and what triggers a rollback?**
`transaction do ... end` wraps the block in BEGIN/COMMIT on the current connection. Any exception raised in the block rolls back and propagates; `ActiveRecord::Rollback` rolls back silently. Non-bang methods return false without raising, so use bang methods inside.
*Follow-up:* Does a transaction cover Redis or emails? No, only that database connection.

**Q2. What happens with nested transactions?**
By default the inner block joins the outer transaction, so an inner `ActiveRecord::Rollback` is swallowed and nothing is undone; an inner exception that propagates rolls back everything. `transaction(requires_new: true)` creates a savepoint, so the inner block can roll back on its own.
*Follow-up:* What SQL does `requires_new` use? `SAVEPOINT` and `ROLLBACK TO SAVEPOINT`.

**Q3. Why is enqueuing a job inside a transaction a problem, and how do you fix it?**
The job is pushed to Redis immediately, so a worker may run before COMMIT and not find the row, or the transaction may roll back after the job was queued. Enqueue after the block, use `after_commit` callbacks, or rely on transaction-aware enqueueing where your Rails version supports it, and make jobs tolerate missing records.
*Follow-up:* What if the event must never be lost? A transactional outbox: write the event row in the same transaction and publish it from a separate worker.

**Q4. What is the difference between optimistic and pessimistic locking?**
Optimistic locking adds a `lock_version` check to each UPDATE and raises `StaleObjectError` when someone saved first; it suits rare conflicts like edit forms. Pessimistic locking takes a `SELECT ... FOR UPDATE` row lock inside a transaction so others wait; it suits hot rows such as stock or balances.
*Follow-up:* What does `with_lock` do? Opens a transaction, reloads the record with `FOR UPDATE`, and yields.

**Q5. How do you prevent overselling stock?**
Remove the read-modify-write window: an atomic `UPDATE ... SET stock = stock - qty WHERE id = ? AND stock >= qty` and check the affected row count, or `with_lock` and re-check stock inside the lock. Add `CHECK (stock >= 0)` as a final safety net.
*Follow-up:* Why not just validate `stock >= 0`? Two requests can both validate against stale values; only the database sees the latest state atomically.

**Q6. How would you prevent a rent payment from being charged twice?**
Require an idempotency key stored with a unique index, claim the payment by flipping its status from pending to processing inside a short `with_lock`, call the provider after COMMIT with the same idempotency key, then record the result. Retries find the payment already claimed.
*Follow-up:* Why not hold the lock during the provider call? It keeps a transaction and a connection open for a slow network call and blocks other work on that row.

**Q7. How do you handle `ActiveRecord::StaleObjectError`?**
In a form flow, rescue it, reload the record, tell the user someone else changed it and render the form with status 409 so they can review. In a job, reload and retry a few times, since the conflict is usually transient.
*Follow-up:* What must the form include? The `lock_version` hidden field, permitted in strong params.

**Q8. What is `FOR UPDATE SKIP LOCKED` used for?**
It locks the matching rows that are free and skips rows other transactions hold, so many workers can each grab different rows from a work table without waiting. Database-backed job queues like Solid Queue rely on it.
*Follow-up:* And `NOWAIT`? It raises immediately instead of waiting for a lock.

**Q9. When would you use `insert_all` or `upsert_all`, and what do you give up?**
For bulk imports and syncs, where one statement replaces thousands of INSERTs. You give up validations, callbacks and model instances, so validate data first and rely on constraints. `upsert_all(..., unique_by:)` turns repeated syncs into idempotent updates keyed on a unique index.
*Follow-up:* How do you skip duplicates instead of updating them? `insert_all`, which uses `ON CONFLICT DO NOTHING` on Postgres.

**Q10. Validations vs database constraints: which do you use?**
Both. Validations produce friendly errors but run only in Ruby and can be bypassed by bulk methods, raw SQL, other apps and race conditions. Constraints (`NOT NULL`, unique indexes, foreign keys, check constraints) are enforced on every write, atomically. Validations are for UX; constraints are for integrity.
*Follow-up:* What happens when a constraint fails? Rails raises an exception such as `RecordNotUnique` or `InvalidForeignKey`, which you can rescue and turn into an error message.

**Q11. How do you add an index to a large production table without downtime?**
Use `add_index ... algorithm: :concurrently` with `disable_ddl_transaction!`, because Postgres cannot build concurrently inside a transaction. A normal index build blocks writes. strong_migrations flags the unsafe version.
*Follow-up:* What if the concurrent build fails? It leaves an invalid index; drop it and retry.

**Q12. Does column order matter in a composite index? What is a partial index?**
Yes: an index on `(store_id, created_at)` helps queries filtering on `store_id` (optionally sorting by `created_at`), but not queries on `created_at` alone. A partial index covers only rows matching a condition, such as `WHERE archived_at IS NULL`, which keeps it small and can make uniqueness apply only to live rows.
*Follow-up:* What does `explain` show when an index is missing? A sequential scan on a large table.

**Q13. How do you size the database connection pool?**
Per process, the pool must cover the threads that use the database: Puma threads per worker, or Sidekiq concurrency per process. Then add up all processes and keep the total under Postgres `max_connections`, with PgBouncer if needed.
*Follow-up:* Symptom of a too-small pool? `ActiveRecord::ConnectionTimeoutError` under load.

**Q14. How do read replicas work in Rails, and what can go wrong?**
Declare a replica in `database.yml`, call `connects_to database: { writing: :primary, reading: :primary_replica }`, and route reads with `connected_to(role: :reading)` or the automatic database selector middleware. Replication lag can hide a user's own fresh write, so read from the primary right after writes.
*Follow-up:* What can you not do on a replica connection? Writes; Rails raises if you try in the reading role.

**Q15. When would you use a JSONB column instead of normal columns?**
For flexible or external data with no fixed shape: raw API payloads, settings, attributes that vary by product type. Use normal columns for data you filter, join, constrain or validate often. Query JSONB with `->>` and `@>` and index it with GIN.
*Follow-up:* What does `store_accessor` do? Defines attribute-like readers and writers for keys inside a JSON column.

**Q16. Why avoid `default_scope`, and how would you implement soft delete?**
It silently affects every query, association and `new`, and `unscoped` removes all conditions, including association scoping. For soft delete, use a `discarded_at` column with explicit scopes (the discard gem), make unique indexes partial, and keep real deletion for privacy requests.
*Follow-up:* How does paranoia differ? It overrides `destroy` and hides deleted rows with a default scope.

**Q17. How do encrypted attributes work, and what is deterministic encryption?**
`encrypts :iban` (Rails 7.0) encrypts the value before saving and decrypts it on read, with keys stored in credentials. Non-deterministic encryption produces different ciphertext each time and cannot be queried; `deterministic: true` produces the same ciphertext for the same input, which allows `where(email: ...)` at the cost of revealing equal values.
*Follow-up:* Is this a substitute for hashing passwords? No: passwords should be hashed (`has_secure_password`), not reversibly encrypted.

## Further reading
- [Active Record Transactions API](https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html) and [Locking::Pessimistic API](https://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html)
- [Multiple Databases with Active Record](https://guides.rubyonrails.org/active_record_multiple_databases.html)
- [Active Record Encryption](https://guides.rubyonrails.org/active_record_encryption.html) and [Active Record and PostgreSQL](https://guides.rubyonrails.org/active_record_postgresql.html)
