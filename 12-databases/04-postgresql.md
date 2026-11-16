# PostgreSQL

> **Tier:** T1 (know cold) · **Related:** [SQL from zero](02-sql-from-zero.md), [Indexing and query optimization](05-indexing-and-query-optimization.md), [Transactions and concurrency](06-transactions-and-concurrency.md), [Drizzle ORM](13-drizzle-orm.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)
> **On your resume:** PostgreSQL is the database behind both Devntech client projects (CrystalCommerce and Colochousing) and the new multi-tenant TypeScript platform (Drizzle), so expect Postgres-specific follow-ups.

## Quick revision
- Open source, standards-focused, extensible: rich types (JSONB, arrays, ranges, enums, UUID), **transactional DDL**, MVCC, mature extensions (PostGIS, pg_trgm, pgvector).
- **Process per connection**: each client gets its own backend process, so connections are expensive. Use a pooler (PgBouncer) and keep app pools small.
- **shared_buffers** caches 8 KB pages; **WAL** (write-ahead log) records every change before data files are touched and powers crash recovery, replication and point-in-time recovery.
- Types: `bigint` ids, `numeric` or integer cents for money (never float), `text` (same speed as varchar), **`timestamptz`** (not timestamp), `uuid` with `gen_random_uuid()` (core since 13), **`jsonb`** (not json).
- Prefer `GENERATED ALWAYS AS IDENTITY` (SQL standard, Postgres 10+) over `serial` for new tables.
- JSONB: `->` returns jsonb, `->>` returns text, `#>>` path as text, `@>` containment, `?` key exists. Index with **GIN** (`jsonb_path_ops` is smaller but supports fewer operators) or a B-tree expression index on one key.
- Full-text search: `tsvector` + `tsquery` + GIN; `pg_trgm` for fuzzy matching and fast `ILIKE '%x%'`.
- Upsert: `INSERT ... ON CONFLICT (cols) DO UPDATE SET x = EXCLUDED.x`; `RETURNING` gives rows back without a second query.
- **MVCC**: updates write a new row version (`xmin`/`xmax`); readers and writers do not block each other; dead versions are cleaned by **VACUUM/autovacuum**. Long transactions block cleanup and cause **bloat**.
- **RLS** policies filter rows per session setting (for example the current org id), a defense-in-depth layer for multi-tenancy.
- `LISTEN/NOTIFY`: lightweight pub/sub, delivered on commit, not durable.
- Declarative **partitioning** (range, list, hash) for huge time-based tables and instant retention.
- **PgBouncer transaction mode** breaks session state: `SET`, session advisory locks, `LISTEN`, temp tables, and (historically) prepared statements.
- Backups: `pg_dump`/`pg_restore` (logical), base backup + WAL archiving for **PITR**. Test restores.

## 1. Why Postgres

| Strength | What it means for you |
|---|---|
| Transactional DDL | A failed migration rolls back completely; no half-applied schema (MySQL commits each DDL statement) |
| Rich types | JSONB, arrays, ranges, UUID, enums, network types, full-text search built in |
| Correctness | Strict typing, CHECK constraints always enforced, standard SQL (CTEs, window functions, `MERGE`) |
| Concurrency | MVCC with row-level locking; Serializable isolation via SSI |
| Extensibility | Extensions add types, indexes and functions: PostGIS, pg_trgm, pgvector, pg_stat_statements |
| Ecosystem | First-class in Rails, Django, Drizzle, Prisma; offered by every cloud and by Supabase and Neon |

Trade-offs: connection handling needs a pooler at scale, VACUUM needs attention on write-heavy tables, and major-version upgrades need planning (pg_upgrade or logical replication).

## 2. Architecture: processes, memory and the WAL

```text
client --> postmaster --fork--> backend process (one per connection)
                                   | reads and changes 8 KB pages in
                                   v
                          shared_buffers (shared memory page cache)
                                   | every change is first appended to
                                   v
                          WAL (pg_wal) --> crash recovery, replicas, archive for PITR
background processes: checkpointer, background writer, WAL writer, autovacuum, archiver
```

- **Process per connection.** The postmaster forks a backend for every connection. Each costs memory (more with big sorts: `work_mem` applies per sort or hash step, per query), so 2,000 direct connections hurt even if most are idle. Default `max_connections` is 100.
- **shared_buffers** holds recently used pages; the OS page cache is a second layer. A common starting point is around a quarter of RAM.
- **Write-ahead logging.** A change is written to the WAL first; `COMMIT` returns once the WAL is flushed to disk. Dirty data pages are written later by the background writer and at checkpoints. After a crash, Postgres replays WAL from the last checkpoint. The same WAL stream feeds streaming replicas and point-in-time recovery.

## 3. Data types that matter

| Need | Use | Avoid |
|---|---|---|
| Ids, counters | `bigint` (8 bytes); `integer` (4 bytes, about 2.1 billion max) for small tables | Running out of `integer` ids on a busy table |
| Money | `numeric(12,2)` or integer cents (`bigint`) | `real`/`double precision` (binary rounding), the `money` type (locale dependent) |
| Strings | `text`; `varchar(n)` only when the limit is a business rule | `char(n)` (pads with spaces) |
| Points in time | `timestamptz` | `timestamp` without time zone for real events |
| Calendar dates | `date` | Timestamps for birthdays or rent periods |
| Unique ids across systems | `uuid` + `gen_random_uuid()` | UUIDs stored as `text` (bigger, slower) |
| Semi-structured data | `jsonb` | `json`, unless you must keep the exact input text |
| Lists of scalars | Arrays (`text[]`) | Arrays of ids that should be foreign keys |
| Fixed value sets | `text` + CHECK, or an `ENUM` type | Magic integers |
| Intervals | Range types: `daterange`, `tstzrange`, `int4range` | Two columns when you need overlap checks |

**text vs varchar.** In Postgres they are stored and processed the same way; `varchar(n)` just adds a length check. `text` with a CHECK constraint is easier to change later.

**timestamptz vs timestamp.** `timestamptz` stores an absolute instant (internally UTC) and converts on input and output using the session `TimeZone`. `timestamp` stores a wall-clock reading with no zone, so "10:00" is ambiguous. Rails creates `timestamp` columns for `datetime` by default and stores UTC by convention; Rails 7+ can switch the default with `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.datetime_type = :timestamptz`. In Drizzle, write `timestamp("created_at", { withTimezone: true })`.

**UUIDs.** `gen_random_uuid()` (v4, random) is built in since PostgreSQL 13. Random UUIDs scatter inserts across the B-tree; time-ordered UUIDv7 keeps inserts near the end of the index. Recent versions add a built-in `uuidv7()` function; check the release notes for your version, or generate v7 in the app.

**Enums.**
```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'paid';  -- adding is easy
-- Removing a value is not supported: you recreate the type and migrate the column.
```
Many teams prefer `text` + CHECK because changing allowed values is a simple constraint swap.

**Arrays.**
```sql
ALTER TABLE products ADD COLUMN tags text[] NOT NULL DEFAULT '{}';
CREATE INDEX products_tags_gin ON products USING gin (tags);
SELECT name FROM products WHERE tags @> ARRAY['sale'];   -- GIN-indexable containment
SELECT name FROM products WHERE 'sale' = ANY (tags);     -- same result, cannot use the GIN index
```

**Ranges with an exclusion constraint** (no double booking, enforced by the database):
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;           -- lets GiST handle "room_id WITH ="
CREATE TABLE room_bookings (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id bigint NOT NULL,
  stay    daterange NOT NULL,                        -- '[2025-07-01,2025-08-01)' by default
  EXCLUDE USING gist (room_id WITH =, stay WITH &&)  -- no two overlapping stays per room
);
```
A second overlapping insert fails with an exclusion violation, even under concurrency. See the booking example in [06](06-transactions-and-concurrency.md).

## 4. Identity vs serial

```sql
CREATE TABLE a (id bigserial PRIMARY KEY);                           -- old shorthand
CREATE TABLE b (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY); -- standard, preferred

INSERT INTO b (id) VALUES (5);                           -- error: ALWAYS rejects manual ids
INSERT INTO b (id) OVERRIDING SYSTEM VALUE VALUES (5);   -- explicit escape hatch
```

| | `serial` / `bigserial` | Identity column |
|---|---|---|
| Standard | Postgres-specific shorthand | SQL standard (Postgres 10+) |
| Implementation | A sequence plus `DEFAULT nextval(...)` | A sequence bound to the column |
| Manual ids | Silently allowed, later causing duplicate-key errors | Rejected with `ALWAYS`; allowed with `BY DEFAULT` |
| Privileges | Separate grants on the sequence | Table privileges are enough |

Both leave gaps (rolled-back inserts still consume values). After importing rows with explicit ids, move the sequence: `SELECT setval(pg_get_serial_sequence('users', 'id'), (SELECT max(id) FROM users));` (Rails: `connection.reset_pk_sequence!("users")`). Rails migrations still create `bigint` keys backed by a sequence default; Drizzle offers `.generatedAlwaysAsIdentity()`.

## 5. JSONB in practice

```sql
CREATE TABLE webhook_events (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source     text  NOT NULL,
  payload    jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO webhook_events (source, payload) VALUES
  ('payments', '{"type": "rent.paid", "amount_cents": 65000,
                 "user": {"id": 42, "city": "Brussels"}, "tags": ["card", "auto"]}');

SELECT payload -> 'user'                  AS user_json,  -- jsonb
       payload ->> 'type'                 AS type,       -- text
       payload #>> '{user,city}'          AS city,       -- text at a path
       (payload ->> 'amount_cents')::int  AS amount      -- cast text before doing math
FROM webhook_events
WHERE payload @> '{"type": "rent.paid"}'                 -- containment: GIN-indexable
  AND payload ? 'tags'                                   -- top-level key exists
  AND (payload -> 'tags') ? 'card';                      -- string is an element of the array

UPDATE webhook_events SET payload = jsonb_set(payload, '{user,city}', '"Ghent"') WHERE id = 1;
UPDATE webhook_events SET payload = payload || '{"processed": true}' WHERE id = 1;  -- merge keys
UPDATE webhook_events SET payload = payload - 'tags' WHERE id = 1;                  -- drop a key
```

| | `json` | `jsonb` |
|---|---|---|
| Storage | Exact input text | Parsed binary |
| Duplicate keys, key order, whitespace | Preserved | Last duplicate wins; order and whitespace not kept |
| Write cost | Cheaper (no parsing) | Slightly more (parse on write) |
| Read and query | Re-parsed every time | Fast; supports `@>`, `?` and indexing |

Indexing:
```sql
CREATE INDEX webhook_payload_gin  ON webhook_events USING gin (payload);                 -- @>, ?, ?|, ?&
CREATE INDEX webhook_payload_path ON webhook_events USING gin (payload jsonb_path_ops);  -- smaller, mainly @>
CREATE INDEX webhook_type_idx     ON webhook_events ((payload ->> 'type'));             -- B-tree on one key
```

When JSONB fits: variable attributes (product specs that differ per category), raw webhook or third-party API payloads, per-user settings, audit snapshots. When it does not: anything you join on, constrain, filter or sort constantly, or that must reference another table. Promote those to real columns. Other caveats: the planner has weak statistics inside JSONB (estimates can be poor), and updating one key rewrites the whole value. PostgreSQL 12 added SQL/JSON path queries (`@?`, `jsonb_path_query`).

In Rails: `t.jsonb :settings, null: false, default: {}` and `where("settings @> ?", { theme: "dark" }.to_json)`. The `?` operator collides with ActiveRecord's positional `?` binds, so prefer `@>` or named binds there. In Drizzle: `jsonb("settings").$type<Settings>()`.

## 6. Full-text search and pg_trgm

```sql
-- Assume products also has a description column
ALTER TABLE products ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;                                                   -- stored generated columns: Postgres 12+
CREATE INDEX products_search_idx ON products USING gin (search);

SELECT name, ts_rank(search, q) AS rank
FROM products, websearch_to_tsquery('english', 'wireless mouse -gaming') AS q
WHERE search @@ q
ORDER BY rank DESC
LIMIT 20;
```

A `tsvector` is a sorted list of normalized lexemes (stemmed words: "running" becomes "run"); a `tsquery` is the search expression; `@@` matches them. `websearch_to_tsquery` accepts search-box syntax (quotes, `or`, `-word`).

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX customers_name_trgm ON customers USING gin (name gin_trgm_ops);

SELECT name FROM customers WHERE name ILIKE '%ary%';          -- the trigram index makes this fast
SELECT name, similarity(name, 'Mariam') AS score
FROM customers WHERE name % 'Mariam'                           -- typo-tolerant match
ORDER BY score DESC;
```

| Approach | Good for | Limits |
|---|---|---|
| `LIKE 'abc%'` + B-tree | Prefix search | Needs `text_pattern_ops` or C collation; no infix |
| `ILIKE '%abc%'` + pg_trgm GIN | Substring and fuzzy search on names, SKUs, emails | No linguistic stemming or ranking |
| Full-text search | Words in articles, descriptions, listings, with ranking | Language config needed; not for partial words |
| External engine (Elasticsearch, OpenSearch, Meilisearch, Typesense) | Facets, typo tolerance at scale, relevance tuning | Extra infrastructure and a sync pipeline |

In Rails, the `pg_search` gem wraps both FTS and trigram search.

## 7. Upsert with ON CONFLICT and RETURNING

```sql
-- A price-sync job re-running safely: insert new prices, update changed ones, skip identical ones
INSERT INTO market_prices (product_id, source, captured_on, price_cents)
VALUES (101, 'pricing_api', '2025-06-15', 4599),
       (102, 'pricing_api', '2025-06-15', 1250)
ON CONFLICT (product_id, source, captured_on)                 -- must match a unique index/constraint
DO UPDATE SET price_cents = EXCLUDED.price_cents              -- EXCLUDED = the row you tried to insert
WHERE market_prices.price_cents IS DISTINCT FROM EXCLUDED.price_cents
RETURNING product_id, price_cents;
```

Rules that bite in sync jobs:
- The same conflict key twice in one statement fails with "ON CONFLICT DO UPDATE command cannot affect row a second time". Deduplicate the batch first.
- `DO NOTHING` plus `RETURNING` returns nothing for skipped rows; select them separately if you need their ids.
- For a partial unique index, repeat its predicate: `ON CONFLICT (email) WHERE deleted_at IS NULL DO NOTHING`.
- Sequences advance even when the row conflicts, so expect id gaps.
- Upsert is atomic under concurrency, unlike "SELECT, then INSERT if missing", which races.

Rails: `MarketPrice.upsert_all(rows, unique_by: %i[product_id source captured_on])` (one statement, skips validations and callbacks); `insert_all` is the DO NOTHING flavor. Drizzle: `.onConflictDoUpdate({ target, set })` ([Drizzle ORM](13-drizzle-orm.md)). PostgreSQL 15's `MERGE` covers more complex match-and-modify logic.

## 8. MVCC, VACUUM and bloat

Postgres implements MVCC (multi-version concurrency control) by keeping **row versions** in the table itself. Every version carries hidden system columns: `xmin`, the transaction that created it, and `xmax`, the transaction that deleted or locked it.

```sql
CREATE TABLE t (id int PRIMARY KEY, v text);
INSERT INTO t VALUES (1, 'a');
SELECT xmin, xmax, * FROM t;    -- xmin = inserting transaction id, xmax = 0
UPDATE t SET v = 'b' WHERE id = 1;
SELECT xmin, xmax, * FROM t;    -- new xmin: you are now looking at a new version of the row
```

- An `UPDATE` writes a new version and stamps the old one with `xmax`; a `DELETE` only stamps `xmax`. Each transaction reads through a **snapshot** and sees only versions committed before it.
- Result: readers never block writers and writers never block readers. Writers still block writers on the same row.
- Old versions become **dead tuples** once no running transaction can see them. **VACUUM** marks their space reusable (it rarely shrinks the file) and updates the visibility map (needed for index-only scans).
- **autovacuum** runs VACUUM and ANALYZE when dead rows pass a threshold (by default 50 rows plus 20% of the table). For large, busy tables, lower it per table: `ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.02);`.
- **Bloat** is table or index space full of dead or empty slots. Causes: long-running or `idle in transaction` sessions (they hold back the cleanup horizon), abandoned replication slots, massive batch updates, autovacuum that cannot keep up. Fixes: kill long transactions, tune autovacuum, batch big updates, rebuild with `VACUUM FULL` (takes an exclusive lock) or the `pg_repack` extension (online).

### Going deeper: freezing and HOT updates
- Transaction ids are 32-bit and wrap around, so VACUUM also **freezes** old rows (marks them visible to everyone). If freezing falls far behind, Postgres eventually refuses new writes to protect data, which is why autovacuum must never be disabled.
- **HOT updates** (heap-only tuples): if no indexed column changes and the page has room, the new version stays on the same page and indexes are not touched. A lower `fillfactor` leaves room; indexing a frequently updated column (like `updated_at`) prevents HOT.

```sql
-- Tables with the most dead rows
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- Oldest open transactions (the usual bloat culprit)
SELECT pid, state, now() - xact_start AS xact_age, left(query, 60) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_age DESC LIMIT 10;
```

## 9. Roles and privileges

A **role** is both a user and a group: roles with `LOGIN` can connect, and roles can be members of other roles.

```sql
CREATE ROLE app_rw NOLOGIN;                                     -- a group role
GRANT CONNECT ON DATABASE app_production TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public                       -- tables created later too
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

CREATE ROLE web_app LOGIN PASSWORD 'change-me' IN ROLE app_rw;  -- the app's login
CREATE ROLE analyst LOGIN PASSWORD 'change-me';
GRANT SELECT ON orders, order_items TO analyst;                 -- read-only reporting
```

Least privilege in practice: the application never connects as a superuser or as the tables' owner (owners can drop tables and bypass RLS by default); migrations run as an owner role, the app runs as a DML-only role; reporting gets read-only access, ideally on a replica. Since PostgreSQL 15, ordinary users can no longer create objects in the `public` schema by default.

## 10. Row-level security

RLS adds a hidden `WHERE` to every query on a table, based on policies. For multi-tenant apps it is a safety net under application-level scoping: if a query forgets `WHERE org_id = ...`, the database still filters it.

```sql
ALTER TABLE listings ENABLE ROW LEVEL SECURITY;
ALTER TABLE listings FORCE ROW LEVEL SECURITY;      -- apply policies to the table owner as well

CREATE POLICY org_isolation ON listings
  USING (org_id = NULLIF(current_setting('app.current_org_id', true), '')::uuid)       -- read/update/delete
  WITH CHECK (org_id = NULLIF(current_setting('app.current_org_id', true), '')::uuid); -- insert/update

-- Per request, inside the transaction that serves it
BEGIN;
SELECT set_config('app.current_org_id', '0b3f5a2e-8c1d-4e6f-9a7b-2c4d6e8f0a1b', true);  -- true = transaction-local
SELECT * FROM listings;                              -- only that org's rows
COMMIT;
```

- `current_setting(name, true)` returns NULL instead of erroring when the setting is missing; `NULLIF(..., '')` handles the empty string a setting can hold after an earlier transaction used it. If nothing is set, `org_id = NULL` matches no rows, so the policy **fails closed**.
- Transaction-local settings (`set_config(..., true)` or `SET LOCAL`) work with PgBouncer transaction pooling; plain session `SET` would leak between clients.
- Superusers, roles with `BYPASSRLS`, and table owners (unless `FORCE`) skip policies. Policies are permissive (combined with OR) unless declared `AS RESTRICTIVE`.
- Index `org_id` (usually as the first column of composite indexes), since the policy predicate is added to every query.

Design choices for tenancy (shared tables with `org_id`, schema per tenant, database per tenant) are compared in [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md).

## 11. LISTEN/NOTIFY

```sql
LISTEN order_events;                                            -- session A subscribes
NOTIFY order_events, '{"order_id": 42, "status": "paid"}';      -- session B publishes
SELECT pg_notify('order_events', json_build_object('order_id', 42)::text);  -- usable in triggers
```

- Delivered only to sessions listening at that moment, and only when the sending transaction commits (rolled-back notifications never arrive).
- Not durable: a listener that was disconnected misses messages. Payloads are small (under 8000 bytes by default), so send ids, not documents.
- Uses: waking job workers instead of polling, cache-invalidation signals, live dashboards. Rails Action Cable ships a `postgresql` adapter built on it.
- A listener needs a dedicated session connection: it does not work through PgBouncer transaction pooling. For durable messaging use a jobs table (`FOR UPDATE SKIP LOCKED`, see [06](06-transactions-and-concurrency.md)), Redis Streams or a broker.

## 12. Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;     -- Rails migration: enable_extension "pg_trgm"
SELECT extname, extversion FROM pg_extension;
```

| Extension | What it adds |
|---|---|
| `pg_stat_statements` | Per-query execution statistics; the first stop for slow-query hunting (needs `shared_preload_libraries`) |
| `pgcrypto` | Hashing (`crypt`, `gen_salt`, `digest`), symmetric encryption (`pgp_sym_encrypt`); provided `gen_random_uuid()` before 13 |
| `pg_trgm` | Trigram similarity, fast `LIKE/ILIKE '%x%'` |
| `btree_gist` | B-tree operators in GiST, needed for exclusion constraints mixing `=` and `&&` |
| `citext` | Case-insensitive text type (emails, usernames) |
| `uuid-ossp` | Older UUID generators |
| `PostGIS` | Geospatial types, distance and containment queries, spatial indexes |
| `pgvector` (`vector`) | Vector columns and similarity search for embeddings |
| `postgres_fdw` | Query tables in another Postgres server |
| `tablefunc` | `crosstab()` pivots |

Managed services (RDS, Cloud SQL, Supabase, Neon) allow a curated list, so check before designing around one.

## 13. Partitioning

Declarative partitioning splits one logical table into child tables by a key.

```sql
CREATE TABLE audit_events (
  id         uuid NOT NULL DEFAULT gen_random_uuid(),
  org_id     bigint NOT NULL,
  action     text NOT NULL,
  created_at timestamptz NOT NULL,
  PRIMARY KEY (id, created_at)                   -- unique keys must include the partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_events_2025_06 PARTITION OF audit_events
  FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');
CREATE TABLE audit_events_2025_07 PARTITION OF audit_events
  FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');

-- Retention without a giant DELETE (no bloat, near instant)
ALTER TABLE audit_events DETACH PARTITION audit_events_2025_06;
DROP TABLE audit_events_2025_06;
```

- Strategies: **RANGE** (time), **LIST** (region, tenant tier), **HASH** (spread evenly). Queries that filter on the partition key touch only matching partitions (**partition pruning**).
- Caveats: primary and unique keys must include the partition key; queries without it scan every partition; thousands of partitions slow planning; future partitions must be created ahead of time (a scheduled job or the `pg_partman` extension).
- Use it for very large, append-heavy tables with time-based access and retention (events, logs, audit trails). It is not a first fix for slow queries; indexes and query design come first.

## 14. Connection pooling with PgBouncer

Every Puma thread, Sidekiq thread and serverless function instance wants its own connection, and every connection is a Postgres process. A pooler multiplexes many client connections over a few server connections.

| Mode | Server connection is held for | Notes |
|---|---|---|
| Session | The whole client session | Safe for everything, smallest gain |
| Transaction | One transaction | The usual choice for web apps |
| Statement | One statement | Multi-statement transactions are not allowed |

Transaction-mode caveats (the next transaction may run on a different server connection):
- Session `SET` leaks or disappears: use `SET LOCAL` or `set_config(..., true)` inside the transaction.
- Session-level advisory locks: use `pg_advisory_xact_lock`.
- `LISTEN`, temporary tables and `WITH HOLD` cursors do not work.
- Server-side prepared statements historically broke; recent PgBouncer versions can track protocol-level prepared statements (`max_prepared_statements`), so check your version. Otherwise disable them: Rails `prepared_statements: false` in `database.yml`, postgres.js `prepare: false`.
- Rails migrations take an advisory lock; run them against a direct connection or set `advisory_locks: false`.

Sizing: Rails `pool` should be at least the Puma thread count, and Sidekiq `concurrency` should not exceed its pool; total connections are roughly processes times pool size. Managed platforms ship their own poolers (RDS Proxy, Supabase and Neon poolers).

## 15. Backups, recovery and replication

```bash
pg_dump -Fc -d app_production -f app.dump               # logical backup, custom format, compressed
pg_restore -d app_restore --jobs 4 app.dump             # parallel restore
pg_dumpall --globals-only > globals.sql                 # roles and tablespaces
```

- **Logical** (`pg_dump`): a consistent snapshot of one database without blocking writes; portable across major versions; slow for very large databases. Scrub personal data before restoring production into staging.
- **Physical + WAL archiving**: `pg_basebackup` (or tools like pgBackRest and WAL-G) plus continuously archived WAL enables **point-in-time recovery**: restore the base backup, replay WAL up to `recovery_target_time` (for example just before a bad `DELETE`).
- Managed services automate snapshots and PITR windows; know the retention period and practise a restore.
- **Streaming replication** ships WAL to read-only replicas (high availability, read scaling; replicas lag slightly, so read-your-own-writes needs care). **Logical replication** (Postgres 10+) replicates selected tables and is used for zero-downtime major upgrades and migrations.

## 16. psql essentials

```bash
psql "postgresql://app@localhost:5432/app_development"   # password from ~/.pgpass or a prompt
bin/rails dbconsole                                       # psql with database.yml settings
```

| Command | Does |
|---|---|
| `\l`, `\c dbname` | List databases, connect to one |
| `\dt`, `\d orders`, `\d+ orders` | List tables, describe a table (columns, indexes, FKs), with sizes |
| `\di`, `\df`, `\dn`, `\du` | Indexes, functions, schemas, roles |
| `\x auto` | Expanded output for wide rows |
| `\timing on` | Show each query's duration |
| `\e`, `\i file.sql` | Edit the last query in an editor, run a file |
| `\copy (SELECT ...) TO 'out.csv' CSV HEADER` | Client-side CSV export or import |
| `\watch 5` | Re-run the last query every 5 seconds |
| `\?`, `\h CREATE INDEX` | psql help, SQL syntax help |

## Common pitfalls
- Opening too many connections (serverless functions, high Sidekiq concurrency) and hitting `max_connections`. Pool, and size pools deliberately.
- `idle in transaction` sessions from code that opens a transaction and then waits on an HTTP call. They hold locks and block VACUUM. Set `idle_in_transaction_session_timeout`.
- Using `timestamp` without time zone and mixing zones; using `float` for money.
- Assuming foreign keys are indexed. They are not in Postgres.
- Session `SET` behind PgBouncer transaction mode, so settings leak between tenants. Use transaction-local settings.
- JSONB for everything: no constraints, poor statistics, painful queries. Promote hot keys to columns.
- Upserting a batch containing the same key twice.
- `ALTER TYPE ... ADD VALUE` and other DDL in a long migration holding locks on a busy table. Keep DDL short and set `lock_timeout`.
- Never testing restores, then discovering the backups were incomplete.

## How to talk about this in an interview
- Both Devntech client projects run on PostgreSQL: CrystalCommerce (a legacy Rails e-commerce platform with background sync jobs) and Colochousing (Rails 7 with Sidekiq). The new multi-tenant TypeScript platform uses Postgres through Drizzle, with org-scoped access control.
- Good stories to prepare: an upsert-based sync (external IDs plus unique constraints), a slow query you fixed with an index, a JSONB column you used or would use, how tenant scoping is enforced.
- Be accurate about depth: you use Postgres as an application developer (schema, queries, indexes, migrations); for operations topics (replication, PITR, PgBouncer tuning) say what you know conceptually and that the infrastructure was managed by others or by the hosting provider, if that is the case.

> **Fill in:** where Postgres is hosted for CrystalCommerce and Colochousing (managed service or self-hosted) and whether a pooler sits in front of it.

> **Fill in:** one Postgres-specific feature you used on a client project (JSONB, arrays, upsert, a partial index, full-text search) and why.

> **Fill in:** how org-scoped access is enforced on the new platform (app-level filters, RLS, or both).

## Interview questions

**Q1. Why would you choose PostgreSQL?**
It is a correct, standards-focused relational database with rich types (JSONB, arrays, ranges, UUID), transactional DDL that makes migrations safe, strong concurrency through MVCC, and an extension ecosystem (PostGIS, pg_trgm, pgvector). It is well supported by Rails, Django and TypeScript ORMs and by every cloud. For most business applications it is a safe default.
*Follow-up:* Any downsides? Connections are expensive (needs pooling) and write-heavy tables need VACUUM tuning.

**Q2. What is the difference between JSON and JSONB, and when would you use JSONB instead of columns?**
`json` stores the exact text and re-parses it on every read; `jsonb` stores a parsed binary form (no duplicate keys, no key order), supports containment and existence operators and GIN indexes. Use JSONB for variable or external data: webhook payloads, per-category product attributes, settings. Use columns for anything you join, constrain, filter or sort often.
*Follow-up:* How do you query a nested key? `payload #>> '{user,city}'`, or `payload -> 'user' ->> 'city'`.

**Q3. How do you index JSONB?**
A GIN index on the whole column (`USING gin (payload)`) supports `@>`, `?`, `?|` and `?&`; `jsonb_path_ops` is smaller and faster for containment only. For equality or sorting on one specific key, a B-tree expression index: `CREATE INDEX ON t ((payload ->> 'type'))`. The query must use the same expression or operator to hit the index.

**Q4. timestamptz or timestamp?**
`timestamptz` for anything that happened at a moment in time: it stores an absolute instant and converts using the session time zone. `timestamp` has no zone, so values are ambiguous when servers, users or DST differ. Use `date` for calendar concepts like a rent period.

**Q5. Which type do you use for money, and why not float?**
`numeric(12,2)` or integer cents. Floats are binary approximations (0.1 + 0.2 is not exactly 0.3), so totals drift. Keep the currency in its own column if you handle several, and round with explicit rules.

**Q6. serial vs identity, and bigint vs UUID primary keys?**
Identity columns are the SQL-standard replacement for `serial`: the sequence is bound to the column, `ALWAYS` blocks accidental manual ids, and privileges are simpler. bigint keys are compact and ordered; UUIDs can be generated anywhere, merge cleanly across systems and do not leak counts, at 16 bytes. Random v4 UUIDs fragment B-tree inserts, which time-ordered v7 avoids.

**Q7. Explain MVCC in Postgres.**
Each row version stores the transaction that created it (`xmin`) and that deleted or locked it (`xmax`). An UPDATE writes a new version instead of overwriting, and every transaction reads from a snapshot that shows only versions committed before it. So readers never block writers and vice versa; only writers on the same row wait for each other. The cost is dead versions that VACUUM must clean.

**Q8. What does VACUUM do, and why does bloat happen?**
VACUUM makes space from dead row versions reusable, updates the visibility map for index-only scans, and freezes old rows to prevent transaction-id wraparound; autovacuum runs it automatically. Bloat happens when dead rows pile up faster than cleanup: long or idle-in-transaction sessions hold back the cleanup horizon, huge batch updates create millions of dead rows, or autovacuum thresholds are too lax for a large table.
*Follow-up:* VACUUM vs VACUUM FULL? FULL rewrites and shrinks the table under an exclusive lock; plain VACUUM runs alongside normal traffic.

**Q9. What is the WAL used for?**
Every change is appended to the write-ahead log and flushed at commit before data pages are written. That gives durability and crash recovery (replay from the last checkpoint), feeds streaming replicas, and, with archiving, enables point-in-time recovery.

**Q10. How would you enforce tenant isolation with row-level security?**
Enable RLS on tenant tables, add a policy like `USING (org_id = current_setting('app.current_org_id', true)::uuid)` with a matching `WITH CHECK`, and at the start of each request transaction call `set_config('app.current_org_id', ..., true)`. Connect as a non-owner role without BYPASSRLS (or FORCE RLS). Keep app-level scoping too; RLS is the safety net.
*Follow-up:* Why transaction-local? So the setting cannot leak to another client through a pooled connection.

**Q11. Why do you need PgBouncer, and what breaks in transaction mode?**
Postgres uses a process per connection, so thousands of client connections waste memory and hit `max_connections`. PgBouncer in transaction mode lends a server connection only for the duration of a transaction. Anything session-scoped breaks: `SET`, session advisory locks, `LISTEN`, temp tables, and server-side prepared statements unless your PgBouncer version supports them.

**Q12. Full-text search, LIKE or pg_trgm?**
`LIKE 'abc%'` works with a B-tree for prefixes. `ILIKE '%abc%'` and typo-tolerant matching need pg_trgm with a GIN index (names, SKUs, emails). Full-text search (`tsvector`, `tsquery`, ranking, stemming) fits natural-language content. Move to a search engine when you need facets and relevance tuning at scale.

**Q13. How does `ON CONFLICT` work, and what goes wrong in batch syncs?**
On a unique-key collision, `DO UPDATE` updates the existing row using `EXCLUDED` (the proposed row), atomically, or `DO NOTHING` skips. In batches, duplicate keys within one statement raise "cannot affect row a second time", `RETURNING` omits skipped rows, and a `WHERE ... IS DISTINCT FROM` clause avoids rewriting unchanged rows (less bloat).

**Q14. When would you partition a table?**
When a table is very large and access is naturally sliced by a key, usually time: logs, events, audit trails. You get partition pruning and cheap retention (drop old partitions). Costs: unique keys must include the partition key, cross-partition queries are slower, and partitions must be created ahead. Indexes and query fixes come first.

**Q15. How do you back up Postgres and recover from an accidental DELETE?**
Logical dumps with `pg_dump` for portability and smaller databases; physical base backups plus WAL archiving for large ones. To undo a bad DELETE, restore to a new instance with point-in-time recovery just before the statement, then copy the lost rows back. Managed services provide snapshots and PITR. Practise restores.

**Q16. What are LISTEN/NOTIFY good for, and what are the limits?**
Lightweight signaling between sessions: wake workers, invalidate caches, push updates to Action Cable. Messages are delivered on commit to currently connected listeners only, are not stored, and have a small payload limit. Listeners need a dedicated connection, so not through transaction pooling.

**Q17. What is transactional DDL, and why does it matter?**
Schema changes run inside transactions like data changes, so a migration that fails halfway rolls back completely. In MySQL each DDL statement commits implicitly, so a failed multi-step migration can leave the schema half-changed. Exceptions in Postgres: `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block.

**Q18. How do you find what is running or blocking right now?**
`pg_stat_activity` shows sessions, states, transaction start times and queries; `pg_blocking_pids(pid)` shows who blocks whom; `pg_locks` has the details. `pg_cancel_backend(pid)` cancels a query, `pg_terminate_backend(pid)` ends the session. For history, use `pg_stat_statements` and the slow-query log ([05](05-indexing-and-query-optimization.md)).

## Further reading
- PostgreSQL docs, data types: https://www.postgresql.org/docs/current/datatype.html
- JSON types and operators: https://www.postgresql.org/docs/current/datatype-json.html
- Routine vacuuming: https://www.postgresql.org/docs/current/routine-vacuuming.html
- Row security policies: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
