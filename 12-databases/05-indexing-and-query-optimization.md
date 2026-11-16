# Indexing and Query Optimization

> **Tier:** T1 (know cold) · **Related:** [PostgreSQL](04-postgresql.md), [SQL practice problems](03-sql-practice-problems.md), [Transactions and concurrency](06-transactions-and-concurrency.md), [ActiveRecord querying and N+1](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Django ORM performance](../10-python-django/03-orm-performance.md)
> **On your resume:** PostgreSQL on CrystalCommerce (700+ stores, 8M+ customers) and Colochousing (16K+ users, 24K+ applications): the scale where a missing index or an N+1 turns into a slow page or a slow background sync.

## Quick revision
- An **index** is a separate sorted structure that maps column values to row locations. It speeds up reads (filter, join, sort) and slows down every write.
- **B-tree** (the default): balanced tree of 8 KB pages, sorted keys in linked leaves, depth 3-4 even for hundreds of millions of rows. Supports `=`, `<`, `>`, `BETWEEN`, `IN`, `IS NULL`, `ORDER BY`, prefix `LIKE 'abc%'`.
- **Index skipped when**: low selectivity (most rows match), tiny table, function or cast on the column (`lower(email)`, `date(created_at)`), leading wildcard (`LIKE '%x'`), type mismatch, `OR` across columns without separate indexes, stale statistics.
- **Composite index** `(a, b, c)`: usable for `a`, `a,b`, `a,b,c` (leftmost prefix). Put **equality columns first, then the range or sort column**. Tenant id (`org_id`, `store_id`) usually goes first.
- **Covering** (`INCLUDE`) enables index-only scans. **Partial** (`WHERE status = 'pending'`) indexes a subset. **Expression** (`lower(email)`) indexes a computed value. **Unique** enforces a rule.
- **GIN**: JSONB, arrays, full text, trigrams. **GiST**: ranges, geometry, exclusion constraints. **BRIN**: huge append-only tables ordered by time. **Hash**: equality only, rarely worth it.
- **Clustered** index = the table itself is stored in index order (InnoDB primary key, SQL Server clustered index). Postgres tables are heaps; every Postgres index is secondary.
- `EXPLAIN` shows the plan and estimates; `EXPLAIN (ANALYZE, BUFFERS)` runs the query and shows actual times, rows and cache hits. Wrap writes in `BEGIN ... ROLLBACK`.
- Plan nodes: Seq Scan, Index Scan, Index Only Scan, Bitmap Heap Scan; joins: Nested Loop (small outer, indexed inner), Hash Join (big unsorted inputs), Merge Join (sorted inputs).
- Estimated rows far from actual rows is the top clue: run `ANALYZE`, add extended statistics, rewrite non-sargable predicates.
- **OFFSET** pagination reads and discards skipped rows and shifts under inserts; **keyset** (`WHERE (created_at, id) < ($1, $2)`) stays fast.
- Find slow queries with `pg_stat_statements` (sort by total time), `log_min_duration_statement`, `auto_explain`, APM.
- ORM habits: avoid N+1 (`includes`, `select_related`), select only needed columns, batch, use `exists?`, index foreign keys.
- Production: `CREATE INDEX CONCURRENTLY`; drop unused indexes (`pg_stat_user_indexes.idx_scan = 0`).

## 1. What an index is

Without an index, finding `customer_id = 4210` means reading every row (a sequential scan). An index is like the index at the back of a book: a sorted list of values with page numbers, so you jump straight to the right place.

The trade-off: every INSERT, every DELETE, and every UPDATE of an indexed column also updates each index; indexes take disk and memory; and the planner has more options to evaluate. Index for the queries you actually run.

## 2. How a B-tree index works

```text
                    [ root: 1000 | 5000 ]
                   /          |           \
      [ 200 | 600 ]    [ 2000 | 3500 ]    [ 7000 | 9000 ]      internal pages: separator keys
        /   |   \          ...                  ...
 [leaf: 1..199] <-> [leaf: 200..599] <-> ...                    leaves: sorted (key, row pointer), linked
```

- Postgres B-trees are B+tree style: internal pages hold separator keys, leaf pages hold every key with a pointer (TID: block number plus offset) to the row in the table (the heap).
- **Lookup**: start at the root, follow separators down to a leaf: O(log n), with a fanout of hundreds of keys per 8 KB page, so 3-4 page reads find any key among hundreds of millions.
- **Range scan**: find the first matching leaf, then walk the linked leaves in order. This is also why a B-tree can return rows already sorted (`ORDER BY` without a sort step), forward or backward.
- After finding entries, Postgres usually visits the heap to fetch the row and check visibility (MVCC). An **index-only scan** skips that when all needed columns are in the index and the page is marked all-visible.
- The tree stays balanced as rows are inserted (pages split); random keys (UUIDv4) cause splits all over the tree, sequential keys only at the right edge. PostgreSQL 13+ deduplicates repeated keys in leaves, which shrinks indexes on low-cardinality columns.

## 3. Clustered vs non-clustered

| | Clustered | Non-clustered (secondary) |
|---|---|---|
| What it is | The table rows are physically stored in index order; the leaf level is the row data | A separate structure whose leaves point to rows |
| How many | One per table | Many |
| MySQL InnoDB | The primary key is always the clustered index (or the first NOT NULL unique key, or a hidden row id) | Secondary index leaves store the **primary key value**, so a lookup is two B-tree searches |
| SQL Server | You choose one clustered index (default: the primary key) | Point to the clustered key, or to a row id on heap tables |
| PostgreSQL | No maintained clustered index: tables are heaps. `CLUSTER orders USING idx` reorders once, under an exclusive lock, and decays as data changes | Every index, including the primary key |

Practical consequences: in InnoDB keep the primary key small and increasing (a random UUID string primary key bloats every secondary index and splits pages); range queries on the clustered key are very fast. In Postgres the physical order matters only for BRIN indexes and for how many pages a range scan touches.

## 4. When the planner uses or skips an index

The planner estimates the cost of each plan from table statistics and picks the cheapest. An index is skipped when it would be slower or when it cannot be used:

| Situation | Example | Fix |
|---|---|---|
| Low selectivity (matches a large share of rows) | `WHERE active = true` on 95% active rows | Expected; a partial index on the rare value (`WHERE active = false`) |
| Tiny table | 50-row lookup table | Expected; a seq scan is cheaper |
| Function on the column | `WHERE lower(email) = 'a@b.com'`, `WHERE date(ordered_at) = '2025-06-15'` | Expression index, or rewrite as a range on the raw column |
| Leading wildcard | `WHERE name LIKE '%phone'` | pg_trgm GIN index, or full-text search |
| Prefix LIKE in a non-C collation | `WHERE sku LIKE 'AB%'` | Index with `text_pattern_ops` |
| Type mismatch (the column side gets cast) | `WHERE customer_id = 4210.0` (bigint vs numeric); in MySQL `WHERE phone = 3001234567` on a VARCHAR | Pass parameters with the column's type |
| OR across different columns | `WHERE customer_id = 1 OR status = 'pending'` | An index on each column (BitmapOr), or rewrite as `UNION` |
| Leading column of a composite index not filtered | Index `(store_id, status)`, query `WHERE status = 'paid'` | Add an index starting with `status`, or reorder |
| Negations | `WHERE status <> 'cancelled'` | Usually matches most rows; consider a partial index on the positive set |
| Stale statistics | Estimates say 10 rows, reality is 500,000 | `ANALYZE`, tune autovacuum, extended statistics |

Do not "fix" a correct seq scan. If a query really reads half the table, a sequential read is the fastest way to do it. Setting `enable_seqscan = off` in a session is a diagnostic trick to see whether the index plan would be faster, never a production setting.

## 5. Composite indexes and column order

An index on `(a, b, c)` is sorted by `a`, then by `b` within each `a`, then by `c`. Like a phone book sorted by last name then first name: easy to find "Khan, Ali", useless for "everyone named Ali".

| Query | Uses `(store_id, status, created_at)`? |
|---|---|
| `WHERE store_id = 1` | Yes (prefix) |
| `WHERE store_id = 1 AND status = 'paid'` | Yes |
| `WHERE store_id = 1 AND status = 'paid' AND created_at > now() - interval '7 days'` | Yes, fully |
| `WHERE store_id = 1 AND status = 'paid' ORDER BY created_at DESC LIMIT 20` | Yes, including the sort (no Sort node) |
| `WHERE store_id = 1 AND created_at > ...` | Partly: seeks `store_id`, then checks `created_at` entry by entry across all statuses |
| `WHERE status = 'paid'` | Generally no (leading column missing) |

Rules of thumb:
1. **Equality columns first, then one range or sort column last.** After a range condition, later columns can only filter, not narrow the scan.
2. **Put the column every query filters on first**: in a multi-tenant app that is `org_id` or `store_id`.
3. Among equality columns, order for prefix reuse: if some queries filter only `store_id` and others `store_id + status`, `(store_id, status)` serves both.
4. Match sort directions for mixed orders: `ORDER BY created_at DESC, id ASC` needs an index `(created_at DESC, id ASC)` (or the exact reverse).
5. One well-chosen composite index often replaces several single-column ones. A single-column index on `a` is redundant once `(a, b)` exists.

Recent PostgreSQL versions (18) can sometimes use a multicolumn B-tree with a "skip scan" when the leading column is not filtered; check the release notes, and do not design around it. MySQL has a similar optimization.

## 6. Covering, partial, expression and unique indexes

```sql
-- Covering: extra payload columns at the leaf level enable an Index Only Scan (Postgres 11+)
CREATE INDEX orders_customer_covering ON orders (customer_id) INCLUDE (status, ordered_at);

-- Partial: index only the rows queries care about (small, cheap to maintain)
CREATE INDEX orders_pending_idx ON orders (ordered_at) WHERE status = 'pending';

-- Partial unique: soft deletes without blocking re-registration
CREATE UNIQUE INDEX users_email_active_uq ON users (email) WHERE deleted_at IS NULL;

-- Expression: the query must use the same expression
CREATE INDEX users_lower_email_idx ON users (lower(email));
SELECT * FROM users WHERE lower(email) = lower($1);

-- Unique composite: one listing per item per channel (and a fast lookup)
CREATE UNIQUE INDEX listings_item_channel_uq ON channel_listings (inventory_item_id, channel_id);
```

- A partial index is used only when the query's WHERE implies the index predicate (`status = 'pending'` literally, not a parameter the planner cannot see at plan time).
- Index-only scans need the visibility map to be current (VACUUM); watch `Heap Fetches` in EXPLAIN.
- Unique indexes treat NULLs as distinct by default (`NULLS NOT DISTINCT` since PostgreSQL 15).

## 7. Other index types

| Type | Best for | Example |
|---|---|---|
| B-tree | Equality, ranges, sorting: the default for scalar columns | `CREATE INDEX ON orders (customer_id, ordered_at)` |
| GIN (inverted index) | Values containing many elements: JSONB keys, array items, words, trigrams | `USING gin (payload)`, `USING gin (tags)`, `USING gin (name gin_trgm_ops)` |
| GiST | Overlap and nearness: ranges, geometry (PostGIS), exclusion constraints | `EXCLUDE USING gist (room_id WITH =, stay WITH &&)` |
| SP-GiST | Space-partitioned data: points, IP ranges, some text searches | Specialized |
| BRIN | Very large tables whose physical order follows the column (append-only logs) | `CREATE INDEX ON events USING brin (created_at)`: tiny, coarse |
| Hash | Equality only | Rarely better than B-tree; crash-safe only since PostgreSQL 10 |

GIN indexes are slower to update than B-trees (Postgres buffers pending entries with `fastupdate`), so they fit read-heavy columns. BRIN stores only min/max per block range, so it is useless if rows are not physically ordered by that column.

## 8. What indexes cost

- **Writes**: each index adds work to every INSERT and DELETE, and to UPDATEs touching indexed columns. Indexing a column that changes constantly (`updated_at`, `last_seen_at`) also prevents HOT updates.
- **Storage and memory**: indexes compete with table data for `shared_buffers` and the OS cache.
- **Unused and duplicate indexes**: pure cost. Find them:

```sql
SELECT relname AS table_name, indexrelname AS index_name, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```
Check the stats window (counters reset), replicas (their scans are not counted on the primary), and keep indexes that back unique or primary key constraints.

- **Building on a live table**: plain `CREATE INDEX` blocks writes for the whole build. Use `CREATE INDEX CONCURRENTLY` (slower, cannot run in a transaction block, leaves an INVALID index to drop and retry if it fails).

```ruby
class AddPendingIndexToOrders < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!                       # CONCURRENTLY cannot run inside a transaction

  def change
    add_index :orders, :ordered_at, where: "status = 'pending'",
              name: "orders_pending_idx", algorithm: :concurrently
  end
end
```

## 9. EXPLAIN and EXPLAIN ANALYZE

`EXPLAIN` prints the chosen plan with estimates without running the query. `EXPLAIN (ANALYZE, BUFFERS)` runs it and adds actual timings, row counts and buffer usage (recent versions include buffer counts by default). Because ANALYZE executes the statement, wrap INSERT, UPDATE and DELETE in `BEGIN; ... ROLLBACK;`.

How to read a plan:
- It is a tree: the most indented nodes run first and feed their parent. Read inside out.
- `cost=0.43..39.21`: startup cost and total cost in planner units (not milliseconds); only compare costs between plans.
- `rows=10` (estimate) vs `actual ... rows=12`: big mismatches (10x or more) are the main sign of bad statistics or a non-sargable predicate.
- `actual time=0.027..0.058` is in milliseconds **per loop**; multiply by `loops`.
- `Buffers: shared hit=5 read=120`: pages found in cache vs read from disk or the OS.

The following plans are illustrative, from an `orders` table with about 2 million rows.

**Seq Scan** (reads the whole table):
```text
Seq Scan on orders  (cost=0.00..41274.00 rows=19870 width=48) (actual time=0.031..182.442 rows=20113 loops=1)
  Filter: (status = 'pending'::text)
  Rows Removed by Filter: 1979887
Planning Time: 0.090 ms
Execution Time: 183.560 ms
```
Two million rows read to return 20,000 (1%): a partial or regular index on `status` would help.

**Index Scan** (walks the index, fetches each row from the heap):
```text
Limit  (cost=0.43..39.21 rows=10 width=48) (actual time=0.028..0.061 rows=10 loops=1)
  ->  Index Scan Backward using orders_customer_id_ordered_at_idx on orders  (cost=0.43..78.01 rows=20 width=48) (actual time=0.027..0.058 rows=10 loops=1)
        Index Cond: (customer_id = 4210)
Planning Time: 0.120 ms
Execution Time: 0.081 ms
```
`WHERE customer_id = 4210 ORDER BY ordered_at DESC LIMIT 10`: the composite index serves filter and sort (read backward), and the scan stops after 10 rows.

**Index Only Scan** (never touches the table):
```text
Index Only Scan using orders_customer_id_ordered_at_idx on orders  (cost=0.43..4.79 rows=20 width=16) (actual time=0.019..0.024 rows=18 loops=1)
  Index Cond: (customer_id = 4210)
  Heap Fetches: 0
  Buffers: shared hit=4
```
Only indexed columns were selected and the visibility map said all pages are visible.

**Bitmap Heap Scan** (medium selectivity):
```text
Bitmap Heap Scan on orders  (cost=1015.39..18342.12 rows=46810 width=48) (actual time=4.812..38.950 rows=47233 loops=1)
  Recheck Cond: ((ordered_at >= '2025-06-01 00:00:00+00'::timestamp with time zone) AND (ordered_at < '2025-06-08 00:00:00+00'::timestamp with time zone))
  Heap Blocks: exact=4120
  ->  Bitmap Index Scan on orders_ordered_at_idx  (cost=0.00..1003.69 rows=46810 width=0) (actual time=4.301..4.302 rows=47233 loops=1)
        Index Cond: ((ordered_at >= '2025-06-01 00:00:00+00'::timestamp with time zone) AND (ordered_at < '2025-06-08 00:00:00+00'::timestamp with time zone))
```
The index produces a bitmap of matching pages, then pages are read in physical order (fewer random reads than an index scan). Bitmaps from several indexes can be combined with BitmapAnd / BitmapOr.

**Nested Loop** (for each outer row, probe the inner side):
```text
Nested Loop  (cost=0.85..1250.40 rows=150 width=40) (actual time=0.040..1.900 rows=162 loops=1)
  ->  Index Scan using orders_pending_idx on orders o  (cost=0.42..310.10 rows=150 width=16) (actual time=0.025..0.700 rows=162 loops=1)
        Index Cond: (ordered_at > (now() - '1 day'::interval))
  ->  Index Scan using customers_pkey on customers c  (cost=0.43..6.27 rows=1 width=32) (actual time=0.006..0.006 rows=1 loops=162)
        Index Cond: (id = o.customer_id)
```
Ideal when the outer side is small and the inner side is indexed. The same shape with `loops=500000` because of a bad estimate is a classic slow query.

**Hash Join** (build a hash table on the smaller input, probe with the larger):
```text
HashAggregate  (cost=71236.00..71236.50 rows=50 width=16) (actual time=905.301..905.322 rows=48 loops=1)
  Group Key: c.city
  ->  Hash Join  (cost=6345.00..61236.00 rows=2000000 width=8) (actual time=60.104..710.412 rows=2000000 loops=1)
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on orders o  (cost=0.00..36274.00 rows=2000000 width=8) (actual time=0.010..150.210 rows=2000000 loops=1)
        ->  Hash  (cost=3845.00..3845.00 rows=200000 width=16) (actual time=59.810..59.811 rows=200000 loops=1)
              Buckets: 262144  Batches: 1  Memory Usage: 11768kB
              ->  Seq Scan on customers c  (cost=0.00..3845.00 rows=200000 width=16) (actual time=0.008..22.400 rows=200000 loops=1)
```
Seq scans are right here: the query reads every order. `Batches` above 1 means the hash spilled to disk (raise `work_mem` for that query or reduce the input).

**Merge Join** walks two inputs already sorted on the join key (often from indexes, or after explicit Sort nodes) in step; good for large joins on indexed keys. A `Sort` node reporting `Sort Method: external merge  Disk: 51200kB` spilled to disk, while `top-N heapsort` means a LIMIT let Postgres keep only the best rows.

| Join | Picked when | Watch out for |
|---|---|---|
| Nested Loop | Outer side small, inner side indexed on the join key | Underestimated outer rows causing huge loop counts |
| Hash Join | Large inputs, equality join, no useful order | Memory spills (`Batches` > 1) |
| Merge Join | Both inputs sorted on the join key (or cheap to sort) | Expensive sorts if no index provides the order |

## 10. Statistics and ANALYZE

The planner estimates row counts from per-column statistics: null fraction, distinct count, most common values and a histogram, gathered by `ANALYZE` from a sample (autovacuum runs it automatically).

```sql
ANALYZE orders;                                               -- after bulk loads or big deletes
SELECT attname, n_distinct, most_common_vals FROM pg_stats WHERE tablename = 'orders';
```

Typical estimation failures: just after a large import (stale stats), correlated predicates (`city = 'Lahore' AND zip = '54000'` estimated as independent), expressions and JSONB keys without expression indexes, and parameters in generic prepared-statement plans.

### Going deeper: tuning the statistics
```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;    -- larger sample for a skewed column
CREATE STATISTICS addr_city_zip (dependencies) ON city, zip FROM addresses;  -- teach the planner about correlation
ANALYZE addresses;                                            -- extended statistics are filled by ANALYZE
```
The default sample (`default_statistics_target = 100`) is fine for most columns; raise it only for skewed columns whose estimates are measurably wrong.

## 11. Pagination: OFFSET vs keyset

```sql
-- OFFSET: the database still produces and throws away the first 100,000 rows
SELECT id, ordered_at, status FROM orders ORDER BY ordered_at DESC, id DESC LIMIT 20 OFFSET 100000;

-- Keyset (cursor): continue after the last row the client saw
SELECT id, ordered_at, status FROM orders
WHERE (ordered_at, id) < ($1, $2)              -- row comparison: the last page's (ordered_at, id)
ORDER BY ordered_at DESC, id DESC
LIMIT 20;

CREATE INDEX orders_ordered_at_id_idx ON orders (ordered_at, id);  -- scanned backward for DESC
```

| | OFFSET | Keyset |
|---|---|---|
| Cost of page N | Grows with N (reads every skipped row) | Constant (index seek) |
| Consistency | New rows shift pages: duplicates or skipped items | Stable: continues after a fixed point |
| Jump to page 37 | Yes | No (next/previous only) |
| Requirements | Any ORDER BY | A unique, indexed sort key (add `id` as tie-breaker) |
| Fits | Small admin tables, page numbers | Feeds, infinite scroll, mobile lists, API sync endpoints |

```ruby
Order.where("(ordered_at, id) < (?, ?)", cursor_at, cursor_id)
     .order(ordered_at: :desc, id: :desc)
     .limit(20)
```

Related trap: `SELECT count(*)` on a huge table is slow in Postgres (MVCC means it must check row visibility). For "about 2.1M results", use the planner's estimate: `SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders';`.

## 12. Finding slow queries

```sql
-- Top queries by total time (pg_stat_statements extension; column names as of Postgres 13+)
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 1) AS mean_ms,
       rows, left(query, 80) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

- Sort by **total** time first: a 5 ms query called 2 million times a day costs more than a 3-second report run twice.
- `log_min_duration_statement = 500` (milliseconds) logs every slower statement; the `auto_explain` module also logs their plans.
- App side: Rails logs every query with its duration; Rails 7 query log tags (`config.active_record.query_log_tags_enabled = true`) append the controller, action or job as a SQL comment, so slow statements in Postgres logs point back to code. The `bullet` gem flags N+1 queries in development; `rack-mini-profiler` shows per-request SQL. APM tools (New Relic, Datadog, Scout, Skylight) aggregate it in production. Drizzle has a `logger` option; Django has Debug Toolbar.

## 13. Optimization checklist

1. Measure: find the query (pg_stat_statements, slow log, APM) and how often it runs.
2. Reproduce with production-like data volume; `EXPLAIN (ANALYZE, BUFFERS)`.
3. Compare estimated vs actual rows; `ANALYZE` if stale.
4. Large Seq Scan with a big "Rows Removed by Filter"? Add an index that matches the filter.
5. Make predicates sargable: no functions or casts on indexed columns, matching parameter types, no leading wildcards.
6. Order composite index columns: equality first, then range or sort.
7. Select only the columns you need; consider a covering index for hot read paths.
8. Remove round trips: fix N+1, batch writes, push aggregation into SQL.
9. Replace deep OFFSET pagination with keyset.
10. For dashboards, pre-aggregate (rollup tables, materialized views) or cache.
11. Re-measure, check write overhead, and drop indexes nothing uses.

## 14. ORM-side habits

| Habit | Rails (ActiveRecord) | Django | Drizzle |
|---|---|---|---|
| Avoid N+1 | `includes`, `preload`, `eager_load`; `strict_loading` to raise on lazy loads | `select_related` (FK joins), `prefetch_related` (second query) | Explicit joins, or relational queries with `with` (one SQL statement) |
| Fetch fewer columns | `select(:id, :name)`, `pluck(:id)`, `pick` | `only()`, `values()`, `values_list()` | `db.select({ id: t.id })` |
| Iterate big tables | `find_each`, `in_batches` | `iterator(chunk_size=...)` | Keyset loop with `limit` |
| Bulk writes | `insert_all`, `upsert_all`, `update_all`, `delete_all` (skip callbacks) | `bulk_create`, `bulk_update`, `update()` | `.values([...])` arrays, `onConflictDoUpdate` |
| Existence checks | `exists?` (and `any?` on an unloaded relation); `present?` loads every row | `.exists()` | `select ... limit 1` |
| Counting | `count` always queries, `size` uses loaded records or a counter cache, `length` loads everything | `.count()` | `count()` helper |
| Atomic increments | `update_counters(id, stock: -1)` or `increment!` (both issue `SET col = col + n`) | `F("stock") - 1` | ``sql`${t.stock} - 1` `` |

Details and examples live in [ActiveRecord querying and N+1](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Django ORM performance](../10-python-django/03-orm-performance.md) and [Drizzle ORM](13-drizzle-orm.md).

## Common pitfalls
- Assuming Postgres indexes foreign keys. It does not; every `*_id` you join or filter on needs its own index (Rails `references` adds one).
- Wrapping indexed columns in functions (`WHERE date(created_at) = ...`, `WHERE lower(email) = ...` without an expression index).
- Composite index in the wrong order (range column first), or an index whose leading column queries never filter.
- Indexing low-cardinality booleans alone; use a partial index on the rare value instead.
- Adding an index on a busy production table without `CONCURRENTLY`, blocking writes for minutes.
- Trusting `EXPLAIN` estimates without `ANALYZE` actuals, or testing on a dev database with 100 rows where everything is a seq scan.
- Running `EXPLAIN ANALYZE` on a DELETE outside a transaction (it really deletes).
- Deep OFFSET pagination in APIs and infinite scroll.
- `present?` / `length` on large ActiveRecord relations, loading thousands of rows to answer a yes/no or count question.
- Piling up indexes "just in case" on write-heavy tables.

## How to talk about this in an interview
- Frame performance work as measure, explain, fix, re-measure. Interviewers want the method more than a memorized list.
- Tie it to scale you have worked at: CrystalCommerce serves 700+ stores and 8M+ customers, and its background jobs sync listings, prices, stock and orders with PriceCharting, so query cost per sync batch matters. Colochousing has 16K+ users, 24K+ rental applications and a REST API next to the Rails web app.
- If your direct experience is mainly fixing N+1 queries and adding indexes in Rails, say exactly that and show you know the next steps (EXPLAIN ANALYZE, composite order, keyset pagination).

> **Fill in:** a slow page, endpoint or job you investigated on a Devntech project: how you found it (logs, APM, bullet), what the plan or query count showed, and the fix (index, includes, batching).

> **Fill in:** one index you added: the columns, why that order, and whether it was built concurrently.

## Interview questions

**Q1. What is an index, and what does it cost?**
A separate sorted data structure (usually a B-tree) mapping column values to row locations, so the database can find, join and sort rows without scanning the whole table. It costs extra work on every insert, delete and update of indexed columns, plus disk and cache space. Index for real query patterns, not every column.

**Q2. How does a B-tree index work?**
It is a balanced tree of fixed-size pages: internal pages hold separator keys that route the search, and leaf pages hold all keys in sorted order with pointers to the rows, linked to their neighbours. A lookup descends from root to leaf in O(log n) (3-4 page reads for huge tables thanks to the high fanout), and range scans walk the linked leaves, which also yields sorted output.
*Follow-up:* Why are random UUID keys bad for B-trees? Inserts land on random leaves, causing page splits and poor cache locality everywhere.

**Q3. When will Postgres not use an index that exists?**
When it estimates a seq scan is cheaper (low selectivity, small table), or when the predicate cannot use the index: a function or cast on the column, a leading wildcard, a type mismatch, OR across columns, or the composite index's leading column not being filtered. Stale statistics can also lead to the wrong choice.
*Follow-up:* How do you confirm? `EXPLAIN (ANALYZE, BUFFERS)`, and compare estimates with actual rows.

**Q4. How do you choose the column order of a composite index?**
Follow the query: equality-filtered columns first, then the single range or ORDER BY column last, because after a range condition later columns cannot narrow the scan. Put the column every query shares first (tenant id), so the index also serves prefix queries. Example: `WHERE store_id = ? AND status = ? AND created_at > ? ORDER BY created_at` gets `(store_id, status, created_at)`.

**Q5. What is the difference between a clustered and a non-clustered index?**
A clustered index stores the table rows themselves in index order, so there is only one per table (InnoDB's primary key, SQL Server's clustered index). Non-clustered indexes are separate structures pointing to rows; in InnoDB they store the primary key, so lookups go through the clustered index again. Postgres tables are heaps: all indexes are non-clustered, and `CLUSTER` only reorders once.

**Q6. What is a covering index, and what is an index-only scan?**
A covering index contains every column the query needs, either as key columns or as `INCLUDE` payload. Postgres can then answer from the index alone (Index Only Scan) without visiting the table, provided the visibility map marks the pages all-visible (`Heap Fetches: 0`).

**Q7. What is a partial index? Give an example.**
An index with a WHERE clause, covering only some rows. `CREATE INDEX ON orders (ordered_at) WHERE status = 'pending'` keeps a tiny index for the job that scans pending orders. `CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL` enforces uniqueness only among non-deleted users.

**Q8. A query filters on `lower(email)`. How do you index it?**
Create an expression index: `CREATE INDEX ON users (lower(email))`, and write the query with exactly `lower(email) = lower($1)`. Alternatives: store emails normalized (Rails 7.1 `normalizes`) and use a plain unique index, or use the `citext` type.

**Q9. When do you use GIN, GiST, BRIN or hash indexes?**
GIN for values with many elements: JSONB containment and keys, arrays, full-text search, trigram LIKE search. GiST for overlap and proximity: ranges, geometry, exclusion constraints. BRIN for huge append-only tables physically ordered by the indexed column (time-series logs), at a tiny size. Hash for pure equality, rarely better than B-tree.

**Q10. Walk me through reading an EXPLAIN ANALYZE plan.**
Read from the innermost node outward. For each node compare estimated rows with actual rows, look at actual time per loop times the loop count, and check buffers (hit vs read). Large seq scans with many removed rows suggest a missing index; huge loop counts in a Nested Loop suggest a bad estimate; Sort or Hash nodes spilling to disk suggest memory limits or a missing index for the order.

**Q11. When does the planner choose a Seq Scan, Index Scan or Bitmap Heap Scan?**
Index Scan for few rows (high selectivity), especially with LIMIT and a matching order. Bitmap Heap Scan for a moderate number of rows: it collects matching pages first and reads them in physical order, and can combine indexes. Seq Scan when a large fraction of the table matches or the table is small, since sequential reads beat many random ones.

**Q12. Nested Loop vs Hash Join vs Merge Join?**
Nested Loop probes the inner side once per outer row: great for a small outer set with an indexed inner key. Hash Join builds a hash table from the smaller input and probes it with the larger: good for big unsorted equality joins. Merge Join walks two inputs sorted on the key: good when indexes already provide the order.

**Q13. The estimated row count is 10 but the actual is 500,000. What do you do?**
Run `ANALYZE` on the table (stats may be stale after a bulk load), check that the predicate is sargable, raise the column's statistics target for skewed data, and add extended statistics for correlated columns. Then re-check the plan; the bad estimate usually caused a wrong join type or scan.

**Q14. What is wrong with OFFSET pagination, and how does keyset pagination fix it?**
OFFSET makes the database read and discard all skipped rows, so deep pages get slower, and rows inserted between requests shift the pages (duplicates or misses). Keyset pagination filters past the last seen sort key, `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20`, which is an index seek with constant cost. The trade-off is no random page jumps.

**Q15. How do you find slow queries in production?**
`pg_stat_statements` sorted by total execution time, the slow query log (`log_min_duration_statement`), `auto_explain` for plans, and APM traces linking queries to endpoints and jobs. Then reproduce with EXPLAIN ANALYZE on realistic data.

**Q16. Why not just index every column?**
Each index slows writes, uses storage and cache memory, can block HOT updates, and most would never be used. Single-column indexes also rarely match multi-column queries as well as one composite index does. Index the actual query patterns and drop indexes with zero scans.

**Q17. How do you add an index to a large production table safely?**
`CREATE INDEX CONCURRENTLY`, which builds without blocking writes. It cannot run inside a transaction (Rails: `disable_ddl_transaction!` plus `algorithm: :concurrently`), takes longer, and leaves an INVALID index if it fails, which you drop and retry. Schedule it off-peak and monitor.

**Q18. What is the N+1 query problem, and how do you detect it?**
Loading N parent records with one query and then running one more query per record for an association: 1 + N round trips. Detect it in logs (the same query repeated with different ids), with the bullet gem, `strict_loading`, or APM traces. Fix it with eager loading (`includes`, `select_related`, Drizzle `with`) or a join.

**Q19. Does Postgres create indexes on foreign keys automatically?**
No. It indexes primary keys and unique constraints, not the referencing columns. Without an index on `orders.customer_id`, joins and "orders for customer X" scan the table, and deleting a customer scans `orders` to check the constraint. MySQL InnoDB requires and creates them; Rails `t.references` adds one by default.

## Further reading
- PostgreSQL docs, indexes: https://www.postgresql.org/docs/current/indexes.html
- Using EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- pg_stat_statements: https://www.postgresql.org/docs/current/pgstatstatements.html
