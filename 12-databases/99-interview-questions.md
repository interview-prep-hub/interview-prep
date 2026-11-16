# Databases: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## Fundamentals and data modeling (see [DBMS fundamentals](01-dbms-fundamentals.md))

**Q1. What is the difference between a DBMS and an RDBMS?**
A DBMS is any software that stores and manages data with querying, concurrency control and recovery. An RDBMS is a DBMS built on the relational model: tables, keys, constraints and SQL (PostgreSQL, MySQL). MongoDB is a DBMS but not relational.

**Q2. Primary key vs unique key?**
Both reject duplicates. A primary key is NOT NULL, one per table, and the usual target of foreign keys. A unique constraint is an alternate key: you can have several, and it allows NULLs (many NULLs in Postgres by default).

**Q3. What is a foreign key, and what do ON DELETE CASCADE, SET NULL and RESTRICT do?**
A column whose values must exist in another table's key, preventing orphans. When the parent is deleted: CASCADE deletes the children, SET NULL clears their reference, RESTRICT (and the default NO ACTION) blocks the delete.

**Q4. Super key vs candidate key vs composite key?**
A super key is any column set that uniquely identifies rows; a candidate key is a minimal super key; a composite key is any key made of two or more columns, such as `(event_id, user_id)` in a junction table.

**Q5. Surrogate or natural primary key?**
Usually a surrogate (identity bigint or UUID) as the primary key, plus a UNIQUE constraint on the natural key (email, SKU). Natural keys change and are wide, and every change ripples through foreign keys.

**Q6. Explain 1NF, 2NF and 3NF briefly.**
1NF: atomic values, no repeating groups. 2NF: no non-key column depends on only part of a composite key. 3NF: no non-key column depends on another non-key column. Each fact is stored once, removing update, insert and delete anomalies.

**Q7. What does BCNF add over 3NF?**
Every determinant must be a candidate key. A table where `instructor -> course` but instructor is not a key can be in 3NF (course is part of a candidate key) yet violate BCNF.

**Q8. When do you denormalize?**
When a measured read path is too slow and the redundancy is controllable: counter caches, cached order totals, reporting tables, materialized views. Keep a source of truth and a rebuild path.

**Q9. How do you model a many-to-many relationship?**
A junction table with two foreign keys and a composite primary key or unique constraint on the pair, for example `event_attendees(event_id, user_id, rsvp)`.

**Q10. What happens when a CHECK constraint's expression evaluates to NULL?**
The row is accepted. CHECK rejects only FALSE, so `CHECK (age >= 18)` lets a NULL age through unless the column is also NOT NULL.

**Q11. View vs materialized view?**
A view is a saved query that runs every time (always fresh, no storage). A materialized view stores its result (fast, indexable) and is stale until `REFRESH MATERIALIZED VIEW`.

**Q12. OLTP vs OLAP?**
OLTP: many short transactions on few rows (the app database, row storage, normalized). OLAP: large scans and aggregates for analytics (column storage, star schemas, warehouses).

**Q13. Why do many teams keep business logic out of triggers and stored procedures?**
It is hidden from application code, harder to test, review, debug and deploy, and vendor-specific. They are still useful for audit logs, `updated_at`, and invariants every client must obey.

## SQL (see [SQL from zero](02-sql-from-zero.md) and [SQL practice problems](03-sql-practice-problems.md))

**Q14. What is the logical order of SQL clauses?**
FROM/JOIN, WHERE, GROUP BY, HAVING, SELECT (including window functions), DISTINCT, ORDER BY, LIMIT/OFFSET. That is why WHERE cannot use a SELECT alias but ORDER BY can.

**Q15. INNER JOIN vs LEFT JOIN?**
INNER returns only matching pairs. LEFT returns every left row, with NULLs for the right side when nothing matches.

**Q16. WHERE vs HAVING?**
WHERE filters rows before grouping and cannot use aggregates; HAVING filters groups after GROUP BY and can (`HAVING COUNT(*) > 5`).

**Q17. Write a query for the second highest salary.**
```sql
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);
```
Alternatives: `DENSE_RANK() ... WHERE rnk = 2`, or `SELECT DISTINCT salary ... ORDER BY salary DESC OFFSET 1 LIMIT 1` wrapped in a scalar subquery so "none" returns NULL.

**Q18. Delete duplicate emails, keeping the lowest id.**
```sql
DELETE FROM leads a USING leads b WHERE a.email = b.email AND a.id > b.id;
```
Then add a unique index so duplicates cannot come back.

**Q19. UNION vs UNION ALL?**
UNION removes duplicates (an extra sort or hash); UNION ALL keeps every row and is faster. Use UNION ALL unless you need deduplication.

**Q20. For salaries 100, 90, 90, 80, what do ROW_NUMBER, RANK and DENSE_RANK return?**
ROW_NUMBER: 1, 2, 3, 4. RANK: 1, 2, 2, 4. DENSE_RANK: 1, 2, 2, 3.

**Q21. What does `SELECT NULL = NULL;` return?**
NULL (unknown), not TRUE. Use `IS NULL` or `IS NOT DISTINCT FROM` to compare with NULL.

**Q22. Ten employees, one with a NULL `manager_id`. What do `COUNT(*)` and `COUNT(manager_id)` return?**
10 and 9: `COUNT(column)` skips NULLs. Other aggregates (`SUM`, `AVG`) ignore NULLs too.

**Q23. Spot the bug: `SELECT name FROM employees WHERE id NOT IN (SELECT manager_id FROM employees);` returns no rows.**
The subquery contains a NULL (the CEO has no manager), and `x NOT IN (..., NULL)` is never TRUE. Use `NOT EXISTS` or filter `WHERE manager_id IS NOT NULL` in the subquery.

**Q24. Spot the bug: `SELECT c.name, o.id FROM customers c LEFT JOIN orders o ON o.customer_id = c.id WHERE o.status = 'paid';`**
Customers without paid orders disappear because `o.status` is NULL for them, so the LEFT JOIN acts like an INNER JOIN. Move `o.status = 'paid'` into the ON clause.

**Q25. DELETE vs TRUNCATE vs DROP?**
DELETE removes chosen rows (WHERE, row triggers). TRUNCATE empties the whole table fast without row triggers and can reset identities. DROP removes the table itself. All are transactional in Postgres; TRUNCATE and DROP commit implicitly in MySQL.

**Q26. What is a correlated subquery?**
A subquery that references the outer row, so it logically runs once per outer row: `WHERE salary > (SELECT AVG(salary) FROM employees x WHERE x.department_id = e.department_id)`. A window function often does the same in one pass.

**Q27. EXISTS vs IN?**
EXISTS checks whether any related row exists and stops at the first match; IN tests membership in a list or subquery result. Planners often treat them alike; for "not exists" prefer NOT EXISTS because NOT IN breaks on NULLs.

**Q28. What are CTEs and recursive CTEs for?**
A CTE (`WITH name AS (...)`) names a subquery to make multi-step queries readable. A recursive CTE walks hierarchies (org charts, category trees, threaded comments) with an anchor query, `UNION ALL` and a step that joins back to the CTE.

**Q29. How do you get the top 3 earners in each department?**
```sql
SELECT * FROM (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS r
  FROM employees e
) t WHERE r <= 3;
```
Use ROW_NUMBER instead if ties must not produce extra rows.

**Q30. How do you compute a running total?**
`SUM(amount) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`; add `PARTITION BY` to restart it per customer or per month.

**Q31. What does `SELECT 5 / 2, 5 / 2.0;` return in Postgres?**
2 and 2.5: dividing two integers truncates. Cast one side (`5::numeric / 2`) or multiply by `100.0` when computing percentages.

**Q32. How do you find customers who never placed an order?**
`SELECT c.* FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);` or a LEFT JOIN with `WHERE o.id IS NULL`.

**Q33. How do you write an upsert in Postgres?**
`INSERT ... ON CONFLICT (unique_cols) DO UPDATE SET col = EXCLUDED.col` (or `DO NOTHING`). It is atomic, unlike "select, then insert if missing".

**Q34. Spot the bug: `SELECT customer_id, ordered_at, COUNT(*) FROM orders GROUP BY customer_id;`**
`ordered_at` is neither grouped nor aggregated, so Postgres raises an error (MySQL too, with ONLY_FULL_GROUP_BY). Use `MAX(ordered_at)` or group by it.

## PostgreSQL (see [PostgreSQL](04-postgresql.md))

**Q35. JSON vs JSONB?**
`json` stores the exact text and re-parses it on every read. `jsonb` stores a parsed binary form (no duplicate keys, no key order), supports containment operators and GIN indexes. Use JSONB.

**Q36. Which JSONB operators return text and which return JSON?**
`->` returns jsonb, `->>` returns text, `#>` / `#>>` do the same for paths. `@>` tests containment and `?` tests whether a key exists.

**Q37. How do you index a JSONB column?**
A GIN index (`USING gin (payload)`) for `@>`, `?`, `?|`, `?&`; `jsonb_path_ops` for a smaller containment-only index; a B-tree expression index on one key: `CREATE INDEX ON t ((payload ->> 'type'))`.

**Q38. timestamptz vs timestamp?**
`timestamptz` stores an absolute instant and converts using the session time zone; `timestamp` stores a wall-clock value with no zone, which becomes ambiguous across servers and DST. Use timestamptz for events.

**Q39. Why not use float for money?**
Floats are binary approximations, so sums drift (0.1 + 0.2 is not exactly 0.3). Use `numeric(12,2)` or integer cents.

**Q40. serial vs identity columns?**
Identity (`GENERATED ALWAYS AS IDENTITY`, Postgres 10+) is the SQL-standard replacement for serial: the sequence is bound to the column, `ALWAYS` rejects accidental manual ids, and privileges are simpler.

**Q41. What is MVCC in Postgres?**
Updates create new row versions (`xmin`/`xmax`) instead of overwriting; each transaction reads a consistent snapshot. Readers never block writers and vice versa; writers on the same row still wait for each other.

**Q42. What does VACUUM do, and what causes bloat?**
VACUUM makes space from dead row versions reusable, updates the visibility map and freezes old rows (preventing transaction-id wraparound). Bloat comes from dead rows accumulating, typically because long or idle-in-transaction sessions hold back cleanup, or huge batch updates.

**Q43. What is the WAL?**
The write-ahead log: every change is logged and flushed at commit before data pages are written. It provides durability and crash recovery and feeds replication and point-in-time recovery.

**Q44. What is row-level security, and when would you use it?**
Policies that add a hidden filter to every query on a table, such as `org_id = current_setting('app.current_org_id', true)::uuid`. Use it as defense in depth for multi-tenant data, setting the org per transaction with `set_config(..., true)`.

**Q45. What breaks behind PgBouncer in transaction mode?**
Anything tied to a session: `SET` (use `SET LOCAL`), session advisory locks, `LISTEN`, temporary tables, and server-side prepared statements unless your PgBouncer version supports them.

**Q46. How do you make `ILIKE '%term%'` fast?**
Enable `pg_trgm` and create a GIN trigram index: `CREATE INDEX ON customers USING gin (name gin_trgm_ops)`. For natural-language search, use full-text search (`tsvector`, `tsquery`).

## Indexing and performance (see [Indexing and query optimization](05-indexing-and-query-optimization.md))

**Q47. What is an index?**
A separate sorted structure mapping column values to row locations so lookups, joins and sorts avoid full scans. It speeds reads and slows every write.

**Q48. How does a B-tree index work?**
A balanced tree of pages: internal pages hold separator keys, leaves hold sorted keys with row pointers and are linked. Lookups go root to leaf in O(log n), a few page reads even for huge tables; range scans walk the leaves in order.

**Q49. When will the database not use an index?**
Low selectivity, tiny tables, functions or casts on the column, leading-wildcard LIKE, type mismatches, OR across columns without separate indexes, a missing leading column of a composite index, or stale statistics.

**Q50. Which queries can use an index on `(a, b)`?**
`WHERE a = ?`, `WHERE a = ? AND b = ?`, `WHERE a = ? ORDER BY b`, and `WHERE a = ? AND b > ?`. Generally not `WHERE b = ?` alone.

**Q51. How do you order columns in a composite index?**
Equality-filtered columns first, then the single range or sort column; put the column every query shares (the tenant id) first.

**Q52. Clustered vs non-clustered index?**
A clustered index stores the table rows in index order (InnoDB primary key, SQL Server clustered index); there is one per table. Non-clustered indexes point to rows. Postgres tables are heaps, so all its indexes are non-clustered.

**Q53. What is a covering index?**
An index containing every column a query needs (key columns or `INCLUDE` columns), so the database answers from the index alone (an index-only scan).

**Q54. Give an example of a partial index.**
`CREATE INDEX ON orders (created_at) WHERE status = 'pending';` or a soft-delete-aware unique email: `CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL;`.

**Q55. What does `EXPLAIN ANALYZE` show that `EXPLAIN` does not?**
It actually runs the query and reports real timings, actual row counts and loop counts (and buffer usage with `BUFFERS`), so you can compare estimates with reality. Wrap writes in `BEGIN ... ROLLBACK`.

**Q56. The plan shows a Seq Scan although an index exists. Is that a bug?**
Not necessarily: if the query returns a large share of the table, a sequential scan is cheaper than many random index lookups. It is a problem when few rows match; then check selectivity estimates, statistics and predicate shape.

**Q57. OFFSET vs keyset pagination?**
OFFSET reads and discards all skipped rows (slow deep pages) and shifts when rows are inserted. Keyset filters after the last seen sort key, `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20`, which stays fast via an index but cannot jump to an arbitrary page.

**Q58. What is the N+1 query problem?**
One query loads N records, then one extra query per record loads an association: N + 1 round trips. Fix with eager loading (`includes`, `select_related`, `Include`, Drizzle `with`) or a join.

**Q59. How do you find slow queries in Postgres?**
`pg_stat_statements` sorted by total execution time, `log_min_duration_statement` for the slow-query log, `auto_explain` for plans, and APM traces; then `EXPLAIN (ANALYZE, BUFFERS)`.

**Q60. Does Postgres index foreign key columns automatically?**
No. Only primary keys and unique constraints get indexes. Index FK columns used in joins and filters (Rails `references` adds one; MySQL InnoDB creates them automatically).

**Q61. Spot the bug: `WHERE date(created_at) = '2025-06-15'` is slow despite an index on `created_at`.**
The function hides the column from the index. Use a half-open range: `created_at >= '2025-06-15' AND created_at < '2025-06-16'`, or an expression index.

## Transactions and concurrency (see [Transactions and concurrency](06-transactions-and-concurrency.md))

**Q62. What does ACID stand for?**
Atomicity (all or nothing), Consistency (constraints hold), Isolation (concurrent transactions do not interfere, per the isolation level), Durability (committed data survives crashes).

**Q63. Name the isolation levels and what each prevents.**
Read Uncommitted (nothing), Read Committed (dirty reads), Repeatable Read (also non-repeatable reads; Postgres also prevents phantoms), Serializable (all anomalies, including write skew).

**Q64. What are the default isolation levels in Postgres and MySQL?**
Postgres: Read Committed. MySQL InnoDB: Repeatable Read.

**Q65. Dirty read, non-repeatable read, phantom read: what are they?**
Dirty: reading another transaction's uncommitted data. Non-repeatable: reading the same row twice and getting different values. Phantom: rerunning a query and seeing new or vanished rows.

**Q66. What is a lost update, and how do you prevent it?**
Two transactions read the same value, compute in the app, and the second write overwrites the first. Prevent with atomic SQL (`SET stock = stock - 1`), `SELECT ... FOR UPDATE`, or optimistic locking with a version column.

**Q67. What is write skew?**
Two transactions read overlapping data, make individually valid decisions and write different rows, together breaking a rule (two bookings for the same room). Snapshot isolation misses it; Serializable, constraints or locking a shared row prevent it.

**Q68. Optimistic vs pessimistic locking?**
Optimistic: no lock while reading; the update checks a version (`WHERE lock_version = ?`) and retries on conflict, good for rare conflicts. Pessimistic: lock rows first (`FOR UPDATE`), good for frequent conflicts with short critical sections.

**Q69. What is a deadlock, and how do you prevent it?**
Two transactions each wait for a lock the other holds. The database detects it and aborts one. Prevent with a consistent lock order (sort ids), short transactions and `lock_timeout`, and retry the victim.

**Q70. What does `SELECT ... FOR UPDATE SKIP LOCKED` do?**
Locks matching rows but skips rows other transactions have locked, so many workers can claim different jobs from one table without blocking. Database-backed queues use it.

**Q71. How do you prevent overselling stock?**
`UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock >= 1 RETURNING stock;` and treat "no row" as out of stock; add `CHECK (stock >= 0)` as a safety net.

**Q72. Why enqueue background jobs after commit?**
A worker can start before the transaction commits and fail to find the new row, or run a job for a transaction that later rolls back. Use `after_commit` or enqueue after the transaction block.

**Q73. What is a savepoint?**
A marker inside a transaction you can roll back to (`ROLLBACK TO SAVEPOINT`) without abandoning the whole transaction. Rails uses them for `transaction(requires_new: true)`.

**Q74. What happens to a Postgres transaction after a statement fails?**
It becomes aborted: every further statement errors with "current transaction is aborted" until you ROLLBACK (or roll back to a savepoint).

## MySQL (see [MySQL essentials](07-mysql-essentials.md))

**Q75. InnoDB vs MyISAM?**
InnoDB: transactions, row locks, foreign keys, crash recovery, MVCC; the default. MyISAM: table locks, no transactions or foreign keys; legacy.

**Q76. utf8 vs utf8mb4 in MySQL?**
MySQL's `utf8` stores at most 3 bytes per character, so emoji fail. `utf8mb4` is full UTF-8 and the MySQL 8 default.

**Q77. How do you upsert in MySQL?**
`INSERT ... ON DUPLICATE KEY UPDATE col = new.col` (with a row alias, or `VALUES(col)` on older versions). It fires on any unique key conflict, and there is no RETURNING.

**Q78. Why are InnoDB secondary index lookups more expensive?**
The table is clustered by primary key, and secondary index entries store the primary key value, so a lookup searches the secondary index and then the clustered index. Covering indexes avoid the second search.

**Q79. What does ONLY_FULL_GROUP_BY do?**
It rejects selecting columns that are neither grouped, aggregated nor functionally dependent on the GROUP BY columns, instead of returning arbitrary values.

**Q80. Is DDL transactional in MySQL?**
No: each DDL statement commits implicitly, so a multi-step migration that fails midway leaves partial changes. Postgres DDL is transactional.

## MongoDB (see [MongoDB fundamentals](08-mongodb-fundamentals.md) and [schema design, aggregation and Mongoose](09-mongodb-schema-design-aggregation-mongoose.md))

**Q81. SQL vs NoSQL: when would you choose MongoDB?**
When data is naturally document-shaped and read as a unit (catalogs, content, profiles, events), fields vary, and you want horizontal scaling. Choose SQL when relationships are many-to-many, integrity must be enforced by the database, or reporting is heavy.

**Q82. What is an ObjectId?**
MongoDB's default 12-byte `_id`: a 4-byte timestamp, a 5-byte random value and a 3-byte counter, generated by the driver and roughly sorted by creation time.

**Q83. Embed or reference?**
Embed data that is owned, bounded and read with the parent (order lines, addresses). Reference data that is unbounded, shared, accessed independently or updated by others (reviews, enrollments).

**Q84. What is the 16 MB limit, and why do unbounded arrays hurt before reaching it?**
The maximum document size. Growing arrays make every update rewrite a bigger document, inflate multikey indexes and slow reads; bound them or move items to their own collection.

**Q85. `$set` vs `replaceOne`?**
`$set` updates only the listed fields; `replaceOne` replaces the whole document except `_id`, dropping fields you omit.

**Q86. What does `$elemMatch` do?**
It requires one array element to match all conditions. Separate dot-notation conditions (`"items.sku"`, `"items.qty"`) may be satisfied by different elements.

**Q87. What is the ESR rule?**
Order compound index fields as Equality, Sort, Range, so equality narrows the scan, the sort comes from the index, and the range filters within it.

**Q88. What is a TTL index?**
An index on a Date field with `expireAfterSeconds`; a background task deletes expired documents about once a minute. Used for sessions, tokens and temporary data.

**Q89. What does `w: "majority"` mean?**
The write is acknowledged only after a majority of replica set members have it, so it survives a primary failover. It is the default write concern in modern versions.

**Q90. How does a replica set elect a new primary?**
When members lose contact with the primary for the election timeout, an eligible secondary with recent data calls an election and becomes primary with a majority of votes. Drivers discover it and retry eligible writes.

**Q91. How do you choose a shard key?**
High cardinality, even write distribution (avoid monotonically increasing ranged keys) and present in most queries so they target one shard; for tenants, something like `{ storeId: 1, createdAt: 1 }`.

**Q92. Does MongoDB support transactions?**
Yes: multi-document ACID transactions since 4.0 on replica sets and 4.2 on sharded clusters. They cost more, so design schemas where single-document atomic updates cover most cases.

**Q93. `$lookup` vs Mongoose `populate`?**
`$lookup` joins inside the database in an aggregation pipeline. `populate` runs one extra `$in` query per populated path and merges in Node. Populating inside a loop is a true N+1.

**Q94. How do you order aggregation stages for performance?**
`$match` first (and `$sort` + `$limit` early) so indexes are used, shrink documents before `$unwind` and `$lookup`, limit before joining, and precompute frequent dashboards with `$merge`.

**Q95. Why didn't my Mongoose `pre("save")` hook run on `findOneAndUpdate`?**
Save middleware runs only for `save()` and `create()`; query methods trigger query middleware. Validators on updates also need `runValidators: true`.

**Q96. What does `lean()` do?**
Returns plain JavaScript objects instead of Mongoose documents: faster and lighter, but no `save()`, instance methods or virtuals by default.

## Redis (see [Redis](10-redis.md))

**Q97. What is Redis typically used for?**
Caching, sessions, rate limiting, job queues (Sidekiq, BullMQ), pub/sub, distributed locks, leaderboards (sorted sets), counters and approximate unique counts (HyperLogLog).

**Q98. Why is Redis fast if commands run on a single thread?**
Data lives in memory, commands take microseconds, there is no locking overhead, and structures are compact. Each command is atomic, but one slow command blocks everyone.

**Q99. What are common cache invalidation strategies?**
TTL expiry, deleting keys after the database commit, write-through updates, versioned keys (Rails `cache_key_with_version`), and bumping a namespace version.

**Q100. What is a cache stampede, and how do you prevent it?**
A hot key expires and many requests recompute it at once, overloading the database. Prevent it with a recompute lock, serving stale data briefly (`race_condition_ttl`), early refresh and TTL jitter.

**Q101. Which eviction policy should Sidekiq's Redis use?**
`noeviction`: evicting queue or retry keys loses jobs. Caches use `allkeys-lru` or `allkeys-lfu`, ideally on a separate instance.

**Q102. RDB vs AOF?**
RDB takes periodic snapshots (compact, may lose minutes of writes); AOF logs every write (with `everysec`, at most about a second lost). Durable data uses AOF, often with RDB.

**Q103. How do you rate limit with Redis?**
Fixed window: `INCR` a per-user, per-window key with `EXPIRE`. Sliding window: a sorted set of request timestamps, trimming old entries and counting. Token buckets in Lua for high volume.

**Q104. Pub/sub vs Streams?**
Pub/sub delivers only to currently connected subscribers, with no storage (fine for realtime fan-out like Action Cable). Streams store messages with consumer groups, acknowledgements and replay.

**Q105. How does Sidekiq use Redis?**
Jobs are JSON pushed onto per-queue lists; worker threads pop them with blocking reads. Scheduled, retry and dead jobs live in sorted sets keyed by time. Retries mean jobs must be idempotent.

**Q106. Redis vs Memcached?**
Memcached is a multi-threaded cache of opaque strings with no persistence or replication. Redis adds data structures, persistence, replication, pub/sub, streams and scripting.

## Firebase (see [Firebase](11-firebase.md))

**Q107. Firestore vs Realtime Database?**
Firestore: collections and documents, compound indexed queries, web and mobile offline support, per-operation pricing. Realtime Database: one JSON tree, simpler queries, very low latency, bandwidth pricing.

**Q108. Are Firestore Security Rules filters?**
No. A query must be constrained so every possible result is allowed; otherwise the whole query is denied.

**Q109. How does your backend trust a Firebase-authenticated user?**
Verify the ID token on every request: Admin SDK `verifyIdToken` in Node, or check the JWT's RS256 signature against Google's keys plus `aud`, `iss` and `exp` in other stacks. Use the token's `uid`, never one sent in the body.

**Q110. Why can Firestore bills surprise you?**
Billing is per document read, write and delete: busy listeners, large result sets, offsets, re-subscribing listeners and rule `get()` calls multiply reads.

**Q111. Where does FCM fit when an Expo app sends push notifications?**
Expo's push service relays Android notifications through FCM (using credentials you upload) and iOS notifications through APNs. You can also call FCM directly with native device tokens.

## ORMs and Drizzle (see [ORMs compared](12-orms-compared.md) and [Drizzle ORM](13-drizzle-orm.md))

**Q112. Active Record vs Data Mapper?**
Active Record objects wrap rows and save themselves (Rails, Django): simple but coupled to persistence. Data Mapper keeps plain entities and a separate context tracks and flushes changes (EF Core, Hibernate).

**Q113. `includes` vs `preload` vs `eager_load` in Rails?**
`preload` runs a separate IN query per association; `eager_load` uses one LEFT JOIN query; `includes` picks preload unless you reference the association's table in conditions or ordering.

**Q114. `select_related` vs `prefetch_related` in Django?**
`select_related` follows foreign keys and one-to-one relations with a JOIN; `prefetch_related` runs a second query per relation and joins in Python (reverse foreign keys, many-to-many).

**Q115. When would you drop to raw SQL in an ORM project?**
Reports with window functions or CTEs, bulk upserts, database-specific features (JSONB, full-text search, SKIP LOCKED) and measured hot paths. Always with bind parameters.

**Q116. Why can `validates :email, uniqueness: true` still allow duplicates?**
It checks with a SELECT before inserting, so two concurrent requests both pass. Only a unique index guarantees uniqueness; rescue `ActiveRecord::RecordNotUnique`.

**Q117. Drizzle vs Prisma?**
Drizzle: schema in TypeScript, SQL-shaped queries, types inferred without code generation, thin runtime. Prisma: its own schema language, a generated client with a nested API, integrated migrations.

**Q118. `drizzle-kit generate` vs `push`?**
`generate` writes reviewed SQL migration files that `migrate` applies in CI and production; `push` applies the schema diff directly with no files, for local prototyping only.

**Q119. What are `$inferSelect` and `$inferInsert`?**
Drizzle type helpers on a table: the row type a select returns, and the payload type insert accepts (defaulted and nullable columns optional).

**Q120. How do you scope Drizzle queries to the current tenant?**
`org_id` on every tenant table, the org taken from the verified session, and scoped helpers that always add `eq(table.orgId, ctx.orgId)` to selects, updates and deletes; Postgres RLS as a second layer, and cross-tenant tests.
