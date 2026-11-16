# MySQL Essentials

> **Tier:** T3 (essentials) · **Related:** [PostgreSQL](04-postgresql.md), [Indexing and query optimization](05-indexing-and-query-optimization.md), [Transactions and concurrency](06-transactions-and-concurrency.md), [SQL from zero](02-sql-from-zero.md)
> **On your resume:** MySQL in your skills list. Many Pakistani software houses and legacy PHP or Rails apps run MySQL, so expect "how is it different from Postgres?" questions even when your projects used Postgres.

## Quick revision
- Use the **InnoDB** engine (default since MySQL 5.5): transactions, row locks, foreign keys, crash recovery. MyISAM is legacy.
- InnoDB stores each table **clustered by its primary key**; secondary indexes store the primary key value, so a secondary lookup is two B-tree searches. Keep primary keys small and increasing.
- Default isolation is **Repeatable Read** (Postgres: Read Committed), with **next-key/gap locks** on locking reads.
- Upsert: `INSERT ... ON DUPLICATE KEY UPDATE` fires on **any** unique key; no `RETURNING` (use `LAST_INSERT_ID()`).
- **CTEs and window functions exist since 8.0.** Older 5.7 servers lack them.
- `utf8` means 3-byte utf8mb3 (no emoji). Always use **utf8mb4**.
- Default 8.0 collation `utf8mb4_0900_ai_ci` is case- and accent-insensitive: `'ali' = 'ALI'` is true, and unique indexes treat them as duplicates.
- **Strict SQL mode** turns silent truncation into errors; **ONLY_FULL_GROUP_BY** rejects ambiguous GROUP BY queries. Both are on by default in modern versions.
- DDL commits implicitly: a failed multi-step migration can leave the schema half-changed.
- Replication ships the **binary log** from a source to replicas (asynchronous by default, so replicas lag).

## 1. MySQL vs PostgreSQL

| Topic | MySQL 8 (InnoDB) | PostgreSQL |
|---|---|---|
| Default isolation | Repeatable Read | Read Committed |
| Auto-increment ids | `id BIGINT AUTO_INCREMENT PRIMARY KEY` | Identity columns or sequences |
| Upsert | `INSERT ... ON DUPLICATE KEY UPDATE` (any unique key) | `INSERT ... ON CONFLICT (cols) DO UPDATE` (explicit target) |
| Returning inserted data | `LAST_INSERT_ID()`; no `RETURNING` | `RETURNING` |
| JSON | `JSON` type (binary), `->` / `->>`, `JSON_EXTRACT`; index through generated columns or functional indexes; newer 8.0 releases add multi-valued indexes for arrays | `jsonb`, containment operators, GIN indexes |
| CTEs and window functions | Since 8.0 | Since 8.4 (2009) |
| String comparison | Collation-dependent; the default is case-insensitive | Case-sensitive (`ILIKE`, `lower()`, `citext` for insensitive) |
| Identifier quoting | Backticks; table-name case sensitivity depends on OS and `lower_case_table_names` | Double quotes; unquoted names fold to lowercase |
| DDL in transactions | Implicit commit per DDL statement (each statement is atomic in 8.0) | Transactional DDL |
| Booleans | `BOOLEAN` is `TINYINT(1)` | Native `boolean` |
| Partial indexes | Not supported | `CREATE INDEX ... WHERE ...` |
| CHECK constraints | Enforced since 8.0.16; older versions parsed and ignored them | Always enforced |
| Concurrency internals | MVCC through undo logs, purged in the background | MVCC through row versions in the table, cleaned by VACUUM |
| Full-text search | `FULLTEXT` index with `MATCH ... AGAINST` | `tsvector` / `tsquery` |
| License | GPL (Oracle); MariaDB is a community fork | PostgreSQL license (permissive) |

Choosing: both run serious production workloads. Postgres wins on types, constraints and SQL features; MySQL has a long history of simple replication and huge web deployments, and is often what an existing stack already runs. In an interview, prefer "it depends on the team and existing stack" with concrete differences over brand loyalty.

## 2. InnoDB vs MyISAM

| | InnoDB | MyISAM |
|---|---|---|
| Transactions | Yes (ACID) | No |
| Locking | Row-level | Table-level (one writer blocks the table) |
| Foreign keys | Enforced | Not supported |
| Crash recovery | Automatic (redo and undo logs) | Tables can be corrupted (`REPAIR TABLE`) |
| MVCC | Yes | No |
| Status | Default since 5.5 | Legacy; avoid for application tables |

If you inherit an old schema, `SHOW TABLE STATUS` reveals the engine; converting is `ALTER TABLE t ENGINE=InnoDB` (a table rebuild, so plan it).

## 3. Clustered primary key and secondary index lookups

```text
Clustered index (the table):  PK 1 -> full row | PK 2 -> full row | PK 3 -> full row ...
Secondary index on customer_id:  (customer_id 5, PK 2) | (customer_id 5, PK 9) | (customer_id 7, PK 3) ...
```

- `SELECT * FROM orders WHERE customer_id = 5` searches the secondary index, collects primary keys 2 and 9, then searches the clustered index for each row.
- A covering index avoids the second step: with `INDEX (customer_id, status)`, `SELECT status FROM orders WHERE customer_id = 5` is answered from the index alone (EXPLAIN shows `Using index`).
- Every secondary index stores the primary key, so a wide primary key (a `CHAR(36)` UUID) inflates every index. Random keys also insert all over the clustered index, causing page splits. Prefer `BIGINT AUTO_INCREMENT`, or store ordered UUIDs as `BINARY(16)` (`UUID_TO_BIN(uuid, 1)` reorders time-based UUIDs).
- Without a primary key, InnoDB uses the first `UNIQUE NOT NULL` index, or a hidden 6-byte row id. Always declare one.
- InnoDB requires an index on foreign key columns and creates one if missing (Postgres does not).

## 4. utf8 vs utf8mb4

MySQL's `utf8` is an alias for **utf8mb3**: at most 3 bytes per character, so emoji and some rare characters (4 bytes in UTF-8) fail with "Incorrect string value" or are mangled. `utf8mb4` is real UTF-8 and the default in MySQL 8.

```sql
CREATE DATABASE app CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
ALTER TABLE messages CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;  -- rebuilds the table
```

Legacy detail: older row formats limited index keys to 767 bytes, and `VARCHAR(255)` in utf8mb4 needs 1020, which is why old Rails apps used `VARCHAR(191)` for indexed strings. Modern defaults (DYNAMIC row format) allow 3072 bytes. The connection must also be utf8mb4 (`encoding: utf8mb4` in Rails `database.yml`).

## 5. Strict SQL mode and ONLY_FULL_GROUP_BY

```sql
SELECT @@sql_mode;   -- typically includes ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_DATE, ...
```

**Strict mode** (`STRICT_TRANS_TABLES`): invalid data raises an error. Without it, MySQL "fixes" values with only a warning: `'abcdef'` into `VARCHAR(3)` becomes `'abc'`, an impossible date becomes `'0000-00-00'`, out-of-range numbers are clipped. Old servers configured without strict mode are a common source of quietly corrupted data.

**ONLY_FULL_GROUP_BY**:
```sql
-- Rejected: ordered_at is neither grouped, aggregated, nor determined by customer_id
SELECT customer_id, ordered_at, COUNT(*) FROM orders GROUP BY customer_id;

-- Fixes
SELECT customer_id, MAX(ordered_at) AS last_order_at, COUNT(*) FROM orders GROUP BY customer_id;
SELECT customer_id, ANY_VALUE(status) AS some_status, COUNT(*) FROM orders GROUP BY customer_id;  -- any value is fine
```
Old MySQL silently returned an arbitrary row's value, producing nondeterministic bugs that surfaced when the mode was enabled during an upgrade.

## 6. Replication basics

- The source writes every change to the **binary log** (binlog). Each replica copies it into a relay log and applies it.
- Binlog formats: statement-based, **row-based** (the default in modern versions, safest), or mixed.
- **Asynchronous** by default: replicas lag behind (usually milliseconds, sometimes much more), and a failover can lose the last transactions. Semi-synchronous replication waits until at least one replica has received each transaction.
- **GTIDs** (global transaction ids) make failover and re-pointing replicas simpler. Group Replication and InnoDB Cluster provide automatic failover.
- Read replicas create a **read-your-writes** problem: a user saves a form, the next page reads from a lagging replica and shows old data. Rails multiple-database support (`connects_to` with writing and reading roles, plus automatic role switching that keeps a user on the primary for a short delay after a write) addresses this.

## 7. Syntax differences cheat sheet

| Task | MySQL | PostgreSQL |
|---|---|---|
| Upsert | `INSERT ... VALUES (...) AS new ON DUPLICATE KEY UPDATE price = new.price` | `ON CONFLICT (...) DO UPDATE SET price = EXCLUDED.price` |
| Update with a join | `UPDATE orders o JOIN customers c ON c.id = o.customer_id SET o.city = c.city` | `UPDATE orders o SET city = c.city FROM customers c WHERE c.id = o.customer_id` |
| Date math | `NOW() - INTERVAL 7 DAY`, `DATE_FORMAT(d, '%Y-%m')` | `now() - interval '7 days'`, `to_char(d, 'YYYY-MM')` |
| String aggregation | `GROUP_CONCAT(name ORDER BY name SEPARATOR ', ')` | `string_agg(name, ', ' ORDER BY name)` |
| Null default | `IFNULL(x, 0)` or `COALESCE(x, 0)` | `COALESCE(x, 0)` |
| Concatenation | `CONCAT(a, ' ', b)` (the double-pipe operator means OR by default) | `CONCAT(a, ' ', b)` or the double-pipe operator |
| Regex | `col REGEXP '^A'` | `col ~ '^A'` |

On older 8.0 releases the upsert form is `ON DUPLICATE KEY UPDATE price = VALUES(price)`; newer releases deprecate `VALUES()` in favor of the row alias shown above. The statement reports 1 affected row for an insert, 2 for an update and 0 when nothing changed.

## Common pitfalls
- Creating tables or connections in `utf8` instead of `utf8mb4`, then failing on the first emoji in a chat message or review.
- Forgetting that the default collation is case-insensitive: a unique index on `code` rejects `'abc'` when `'ABC'` exists, and `WHERE code = 'abc'` matches both.
- Implicit type conversion: `WHERE phone = 3001234567` on a VARCHAR column converts every row to a number, ignoring the index and even matching strings like `'3001234567x'`. Quote the value.
- Error 1093: you cannot UPDATE or DELETE a table while selecting from it in a subquery. Wrap the subquery in a derived table (`SELECT id FROM (SELECT ...) AS t`).
- `UPDATE t SET a = b, b = a` does not swap in MySQL; assignments run left to right.
- `ON DUPLICATE KEY UPDATE` reacting to a different unique key than you intended, on tables with several unique indexes.
- Multi-statement migrations failing halfway because DDL is not transactional; keep one change per migration.
- `TRUNCATE` cannot be rolled back (implicit commit).
- Gap locks under Repeatable Read causing lock waits and deadlocks for concurrent inserts into the same range.
- `LIMIT` inside an `IN (subquery)` is not supported; rewrite with a join or derived table.
- Long `ALTER TABLE` on big tables blocking writes: use online DDL (`ALGORITHM=INSTANT` or `INPLACE` where supported) or tools like gh-ost and pt-online-schema-change.

## How to talk about this in an interview
- Be straightforward: your Devntech client projects (CrystalCommerce, Colochousing) use PostgreSQL, and MySQL is a database you know and can work in. Then show the differences that matter in practice: isolation default, upsert syntax, utf8mb4, case-insensitive collations, non-transactional DDL.
- Connect it to Rails: ActiveRecord hides most differences, but raw SQL, upserts (`upsert_all` maps to `ON DUPLICATE KEY UPDATE` on MySQL), JSON queries and migrations expose them.

> **Fill in:** where you used MySQL (university labs or the Databases lab you assisted with at PUCIT, personal projects, an internship) and what you built with it.

## Interview questions

**Q1. What are the main differences between MySQL and PostgreSQL?**
Postgres defaults to Read Committed, has transactional DDL, `RETURNING`, partial indexes, JSONB with GIN, richer types and always-enforced CHECK constraints. MySQL defaults to Repeatable Read with gap locking, uses `ON DUPLICATE KEY UPDATE`, clusters tables by primary key, and its default collation is case-insensitive. Both are solid; the existing stack and team experience often decide.

**Q2. InnoDB vs MyISAM?**
InnoDB supports transactions, row-level locking, foreign keys, MVCC and crash recovery, and is the default. MyISAM has table-level locks, no transactions and no foreign keys, and can be corrupted by crashes. Use InnoDB for application data.

**Q3. How do clustered and secondary indexes work in InnoDB?**
The table itself is a B+tree ordered by the primary key, with full rows in the leaves. Secondary indexes store the indexed columns plus the primary key, so a lookup by a secondary index finds the primary key, then searches the clustered index for the row. Hence: small, increasing primary keys, and covering indexes for hot queries.
*Follow-up:* Why is a random UUID primary key costly here? It widens every secondary index and inserts into random pages, causing splits.

**Q4. What is the difference between utf8 and utf8mb4?**
MySQL's `utf8` (utf8mb3) stores at most 3 bytes per character, so 4-byte characters like emoji cannot be stored. `utf8mb4` is complete UTF-8 and the MySQL 8 default. Set it for the database, tables and connection.

**Q5. What does ONLY_FULL_GROUP_BY do?**
It rejects queries that select non-aggregated columns that are neither in GROUP BY nor functionally dependent on it, because their value would be arbitrary. Fix with an aggregate, by grouping by the column, or `ANY_VALUE()` when any value is acceptable.

**Q6. How do you write an upsert in MySQL, and what are the gotchas?**
`INSERT ... ON DUPLICATE KEY UPDATE col = new.col` (row alias) or `VALUES(col)` on older versions. It fires on a conflict with any unique index or the primary key, so tables with several unique keys can update an unexpected row. There is no RETURNING, and affected-row counts (1 insert, 2 update, 0 unchanged) tell you what happened.

**Q7. What is strict SQL mode?**
A mode where invalid or out-of-range values cause errors instead of being silently truncated or converted with a warning. It is on by default in modern MySQL; legacy servers without it often contain truncated strings and zero dates.

**Q8. How does MySQL replication work, and what problems does replica lag cause?**
The source records changes in the binary log; replicas fetch and replay them, asynchronously by default. Lag means reads from a replica can be stale, causing read-your-writes bugs right after a user saves something. Route reads that must be fresh to the primary (Rails role switching does this for a short window after writes).

**Q9. Why do concurrent inserts deadlock more often in MySQL than in Postgres?**
Under Repeatable Read, InnoDB's locking reads and some inserts take gap and next-key locks on index ranges, not just rows. Two transactions that lock overlapping gaps and then insert into them can deadlock. Mitigations: Read Committed isolation, consistent access order, unique-key lookups instead of ranges, and retrying deadlocked transactions.

## Further reading
- MySQL Reference Manual, InnoDB locking and transaction model: https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html
- MySQL Reference Manual, server SQL modes: https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html
