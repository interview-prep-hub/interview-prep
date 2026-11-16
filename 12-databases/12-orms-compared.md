# ORMs Compared

> **Tier:** T2 (working knowledge) · **Related:** [Drizzle ORM](13-drizzle-orm.md), [ActiveRecord querying and N+1](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Django models and ORM](../10-python-django/02-models-and-orm.md), [Entity Framework and Dapper](../11-dotnet/02-entity-framework-and-dapper.md), [MongoDB and Mongoose](09-mongodb-schema-design-aggregation-mongoose.md), [Indexing and query optimization](05-indexing-and-query-optimization.md)
> **On your resume:** ActiveRecord (Rails at Devntech and Visnext), the Django ORM (the PureLogics LMS with DRF), Drizzle (the new multi-tenant TypeScript platform), Entity Framework and Dapper (.NET skills), and MongoDB from Next.js at RemoteDev. Few candidates can compare this many from experience.

## Quick revision
- An **ORM** maps tables to classes and rows to objects so you query and persist with your language instead of hand-written SQL. Every ORM still emits SQL: read it.
- **Impedance mismatch**: objects form graphs with identity, inheritance and navigation; relational data is sets of rows linked by keys. Lazy loading, N+1 queries and awkward inheritance mappings come from that gap.
- Styles: **Active Record** (the object saves itself: ActiveRecord, Django ORM), **Data Mapper / Unit of Work** (a context tracks plain objects: EF Core, Hibernate, SQLAlchemy ORM), **query builder** (composable SQL: Drizzle's core API, Knex, Kysely), **micro-ORM** (you write SQL, it maps rows: Dapper), **schema-first generated client** (Prisma).
- N+1 fixes: Rails `includes`/`preload`/`eager_load`; Django `select_related`/`prefetch_related`; EF Core `Include`/`ThenInclude` (+ `AsSplitQuery`); Dapper: write the JOIN (multi-mapping); Drizzle joins or `with`; Prisma `include`; Mongoose `populate` or `$lookup`.
- Migrations: handwritten DSL (Rails), generated from models (Django, EF Core, Drizzle `generate`, Prisma Migrate), separate tools for micro-ORMs, none for Mongoose.
- Drop to raw SQL for reports, window functions, bulk upserts, DB-specific features and measured hot paths, always with **bind parameters**.
- Pitfalls: lazy-loading surprises, leaky abstractions (generated SQL, callbacks skipped by bulk methods), huge transactions, ORM-only uniqueness checks.
- Choose by framework default, team, type-safety needs, SQL control and runtime (serverless, edge).

## 1. What an ORM is, and the impedance mismatch

An ORM (object-relational mapper) lets you write `Order.where(status: "paid").includes(:customer)` instead of SQL, maps result rows to objects, tracks changes and writes them back, and usually manages schema migrations. You gain productivity, safer parameter handling and portable code; you pay with abstraction leaks and generated SQL you did not choose.

Where objects and tables disagree:

| Object world | Relational world | Resulting friction |
|---|---|---|
| Navigate `order.customer.address` | Join tables by keys | Lazy loading triggers hidden queries (N+1) |
| Object identity (two references, one object) | Primary keys (two loaded copies of one row) | Stale copies, identity maps in Data Mapper ORMs |
| Inheritance | No inheritance | Single-table inheritance (Rails `type` column), class-table or concrete-table mappings |
| Rich value types (Money, Address) | Scalar columns | Composed or embedded mappings, JSON columns |
| Loop over objects and call `save` | Set-based statements | N separate UPDATEs where one `UPDATE ... WHERE` would do |
| Language types | SQL types (numeric, timestamptz) | Decimals returned as strings in JS drivers, time-zone surprises |

## 2. Four (and a half) styles

| Style | Idea | Examples |
|---|---|---|
| Active Record | A class per table; instances wrap rows and persist themselves (`order.save`) | Rails ActiveRecord, Django ORM, Laravel Eloquent, Sequelize |
| Data Mapper + Unit of Work | Plain entity classes; a separate context loads them, tracks changes and flushes them in one transaction | EF Core (`DbContext`), Hibernate, SQLAlchemy ORM, TypeORM (data mapper mode) |
| Query builder | Compose SQL with typed functions; results are plain rows or objects | Drizzle's core API, Knex, Kysely, Arel (underneath ActiveRecord) |
| Micro-ORM | You write the SQL; the library maps result rows onto objects | Dapper |
| Schema-first generated client | A schema file generates a typed client and migrations | Prisma |

Active Record is quick and intuitive but couples domain objects to persistence (callbacks, validations and queries live on the model). Data Mapper keeps domain classes persistence-ignorant and batches writes through the unit of work, at the cost of more concepts. Query builders and micro-ORMs give full SQL control with less magic.

## 3. The comparison table

| | ActiveRecord | Django ORM | EF Core | Dapper | Drizzle | Prisma | Mongoose |
|---|---|---|---|---|---|---|---|
| Language | Ruby | Python | C# | C# | TypeScript | TypeScript | JS/TS |
| Style | Active Record | Active Record style | Data Mapper + Unit of Work | Micro-ORM | SQL-like builder + relational API | Schema-first client | ODM for MongoDB |
| Schema source of truth | Migrations; `schema.rb` or `structure.sql` reflects the DB | Model classes | Entity classes + configuration | Your database and SQL | TS schema files | `schema.prisma` | Schemas in code |
| Migrations | Handwritten Ruby DSL | Generated by `makemigrations` | Generated by `dotnet ef migrations add` | None (use a separate tool) | `drizzle-kit generate` / `migrate` / `push` | `prisma migrate dev` / `deploy` | None built in |
| Type safety | Dynamic (optional RBS/Sorbet) | Runtime (type stubs exist) | Strong, LINQ checked by the compiler | Typed mapping, SQL in strings | Strong, inferred from schema | Strong, generated | Inferred with `InferSchemaType` |
| N+1 fix | `includes`, `preload`, `eager_load`, `strict_loading` | `select_related`, `prefetch_related` | `Include`, `ThenInclude`, `AsSplitQuery` | Write the JOIN, multi-mapping | Joins, or `with` in relational queries | `include` / nested `select` | `populate`, or `$lookup` |

Briefly: **Sequelize** is an older Active Record-style JavaScript ORM (`include` for eager loading, sequelize-cli migrations); **TypeORM** uses decorators on entity classes and supports both Active Record and Data Mapper styles (`relations` or `leftJoinAndSelect` for eager loading).

## 4. The N+1 fix in each

The N+1 problem: one query loads N parents, then one more query per parent loads its association.

```ruby
# Rails: 1 + 50 queries
Order.limit(50).each { |o| puts o.customer.name }
# Fixed: 2 queries (preload), or one JOIN (eager_load) when you also filter on customers
Order.includes(:customer).limit(50).each { |o| puts o.customer.name }
```

```python
# Django: JOIN for foreign keys and one-to-one
for order in Order.objects.select_related("customer")[:50]:
    print(order.customer.name)
# Second query with IN (...) for reverse foreign keys and many-to-many
for order in Order.objects.prefetch_related("items")[:50]:
    print(len(order.items.all()))
```

```csharp
// EF Core
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items).ThenInclude(i => i.Product)
    .AsSplitQuery()        // one SQL statement per collection instead of one wide, duplicated join
    .AsNoTracking()        // read-only: skip change tracking
    .Take(50)
    .ToListAsync();
```

```csharp
// Dapper: you write the join; multi-mapping splits each row into two objects at the second "id" column
const string sql = @"
    SELECT o.id, o.status, c.id, c.name
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    ORDER BY o.id DESC
    LIMIT 50";
var orders = await conn.QueryAsync<Order, Customer, Order>(
    sql,
    (order, customer) => { order.Customer = customer; return order; },
    splitOn: "id");
```

```ts
// Drizzle: an explicit join, or the relational API (one SQL statement either way)
const rows = await db
  .select({ orderId: orders.id, status: orders.status, customerName: customers.name })
  .from(orders)
  .innerJoin(customers, eq(customers.id, orders.customerId))
  .limit(50);
const nested = await db.query.orders.findMany({ with: { customer: true }, limit: 50 });

// Prisma
const withCustomer = await prisma.order.findMany({ include: { customer: true }, take: 50 });

// Mongoose: one extra $in query for all customers
const docs = await OrderModel.find().populate("customer", "name").limit(50).lean();
```

## 5. Migrations compared

| Tool | Workflow |
|---|---|
| Rails | `bin/rails g migration AddStatusToOrders` -> edit the Ruby DSL -> `bin/rails db:migrate` (updates `schema.rb`) |
| Django | Edit models -> `python manage.py makemigrations` (generated, review it) -> `migrate` |
| EF Core | Edit entities -> `dotnet ef migrations add AddStatus` -> `dotnet ef database update` (or generate an SQL script for production) |
| Drizzle | Edit the TS schema -> `drizzle-kit generate` (SQL files) -> `drizzle-kit migrate`; `push` syncs directly for prototypes ([13](13-drizzle-orm.md)) |
| Prisma | Edit `schema.prisma` -> `prisma migrate dev` locally -> `prisma migrate deploy` in CI/production |
| Dapper, raw SQL | A separate tool: FluentMigrator, DbUp, Flyway, Liquibase |
| Mongoose | No schema migrations: backfill scripts or jobs, plus a `schemaVersion` field |

Whatever generates them, review migrations like code: generated diffs can drop and recreate a renamed column (data loss), and every tool needs the same production habits: additive changes first (expand, migrate, contract), concurrent index builds, batched backfills, and code that works with both the old and new schema during the deploy.

## 6. When to drop to raw SQL

Good reasons: reports with window functions, CTEs or recursive queries; bulk operations (`INSERT ... SELECT`, multi-row upserts, `UPDATE ... FROM`); database-specific features (JSONB operators, full-text search, `FOR UPDATE SKIP LOCKED`, `LATERAL`); hot paths where the generated SQL measured badly; one-off data migrations.

Do it safely, with bound parameters, in each stack:

```ruby
Order.find_by_sql(["SELECT * FROM orders WHERE customer_id = ? AND ordered_at > ?", customer_id, 30.days.ago])
Order.where("ordered_at > ?", 30.days.ago)          # safe fragment
Order.where("status = '#{params[:status]}'")         # SQL injection: never interpolate input
```

```python
Order.objects.raw("SELECT * FROM shop_order WHERE customer_id = %s", [customer_id])
with connection.cursor() as cur:
    cur.execute("UPDATE shop_order SET status = %s WHERE id = %s", ["paid", order_id])
```

```csharp
var recent = await db.Orders
    .FromSql($"SELECT * FROM orders WHERE customer_id = {customerId}")   // interpolation becomes a parameter
    .ToListAsync();
var rows = await conn.QueryAsync<Order>(
    "SELECT * FROM orders WHERE customer_id = @customerId", new { customerId });   // Dapper
```

```ts
await db.execute(sql`SELECT * FROM orders WHERE customer_id = ${customerId}`);  // Drizzle: values are bound
await prisma.$queryRaw`SELECT * FROM "Order" WHERE "customerId" = ${customerId}`; // Prisma: values are bound
// Escape hatches that are NOT parameterized: Drizzle sql.raw(), Prisma $queryRawUnsafe, EF Core FromSqlRaw with concatenation
```

## 7. ORM pitfalls in depth

**Lazy-loading surprises.** Associations load on first access, so a serializer, view or GraphQL resolver that touches `order.customer` inside a loop silently fires one query per row. Some stacks fail loudly instead: a lazy load after the context is disposed (EF Core proxies), or synchronous ORM access from async code in Django. Fixes: eager load explicitly in the query (DRF: in `get_queryset`), turn on `strict_loading` in Rails to raise on lazy loads, and watch query counts in tests and logs.

**Leaky abstractions.** You still need to know what SQL runs:
- `relation.present?` loads every record; `exists?` runs `SELECT 1 ... LIMIT 1`. `count` always queries; `length` loads everything.
- Bulk methods skip the model layer: `update_all`, `delete_all`, `insert_all`, `update_column` in Rails, `QuerySet.update()` and `bulk_create` in Django, and EF Core's `ExecuteUpdate` / `ExecuteDelete` bypass validations, callbacks or change tracking.
- Rails `first` adds `ORDER BY id`; default scopes leak into every query and join.
- Loading a whole table with `Model.all.each` instead of batching (`find_each`).
- Type mapping: Postgres `numeric` comes back as a string in node-postgres and Drizzle (to avoid float precision loss), and `timestamp` vs `timestamptz` changes what time zone you see.
- EF Core change tracking grows with every loaded entity; use `AsNoTracking` for reads.

**Huge transactions.** Wrapping a 100,000-row import, a long backfill or a sync of a whole store in one transaction holds locks for minutes, bloats the table, can exhaust memory (change trackers, identity maps) and hits timeouts; one bad row rolls back everything. Process in batches (`find_each`, `in_batches`, `insert_all` per chunk), commit per batch, make the import idempotent so it can resume, and move side effects (emails, API calls, job enqueues) after commit.

**Callbacks with hidden side effects.** `after_save` hooks that send emails or call APIs run inside the transaction, fire on every save (including admin fixes and seeds) and make behavior depend on hook order. Prefer explicit service objects or `after_commit` for side effects.

## 8. How to choose
- **Use the framework's default** unless there is a strong reason: Rails -> ActiveRecord, Django -> Django ORM, ASP.NET Core -> EF Core. Ecosystem, docs and hiring all assume it.
- **.NET**: EF Core for most code plus Dapper for reports and hot paths is a common, sensible combination.
- **TypeScript**: Drizzle when you want SQL-shaped queries, inferred types without a generation step and a light runtime that suits serverless and edge; Prisma for a schema-first workflow and a polished generated client (its architecture changed in recent versions, so check current docs); Kysely for a pure type-safe query builder.
- **MongoDB**: Mongoose for schemas, validation and middleware; the native driver for thin, performance-sensitive services.
- Questions to ask: How much SQL control do we need? How important is compile-time type safety? Which database features will we use? How will migrations run in CI? Where does the code run (long-lived servers or serverless)?

## Common pitfalls
- Enforcing uniqueness only in the ORM (`validates :email, uniqueness: true`): two concurrent requests both pass. Add a unique index and handle the violation.
- Interpolating user input into SQL fragments in `where`, `order` or raw queries.
- Mass assignment: passing request params straight into `create` or `update` (use strong parameters, DRF serializers, DTOs).
- Assuming the ORM adds indexes for foreign keys (Postgres does not; Rails `references` does).
- Migrations that rewrite or lock large tables during peak traffic.
- Django querysets evaluated several times by accident (each `len()` or iteration on a new queryset runs the query again), or EF Core queries that cannot be translated to SQL (EF Core 3+ throws instead of silently evaluating in memory).
- Choosing an ORM by benchmark charts instead of by fit with the framework and team.

## How to talk about this in an interview
- You have used several of these on real work: ActiveRecord daily at Devntech on legacy Rails monoliths, the Django ORM with DRF for the PureLogics LMS, Drizzle on the new multi-tenant TypeScript platform, MongoDB with Next.js at RemoteDev, plus Entity Framework and Dapper from .NET. Use that breadth: "the N+1 problem looks like X in Rails and Y in Drizzle".
- Show you read generated SQL: logs, `to_sql`, `.query` in Django, Drizzle's logger. Interviewers love "I checked the SQL and saw...".
- When asked "which ORM is best", answer with trade-offs and context, not a favorite.

> **Fill in:** an N+1 or slow query you fixed in ActiveRecord or Django, and how you spotted it.

> **Fill in:** what moving from ActiveRecord to Drizzle felt like on the platform migration: what you missed, what you liked.

## Interview questions

**Q1. What is an ORM, and what is the object-relational impedance mismatch?**
An ORM maps tables to classes and rows to objects so you can query and persist data with your language's constructs, generating the SQL for you. The mismatch is the conceptual gap between object graphs (identity, inheritance, navigation, encapsulation) and relational sets (rows, keys, joins, set-based updates); it causes N+1 queries, awkward inheritance mappings and per-object updates where one set-based statement would do.

**Q2. What is the difference between the Active Record and Data Mapper patterns?**
In Active Record, each model object wraps a row and knows how to load and save itself (`order.save`), as in Rails and Django: simple and fast to build with, but domain logic and persistence are coupled. In Data Mapper, entities are plain objects and a separate mapper or context (EF Core's DbContext) loads them, tracks changes and writes them in a unit of work: more structure, better separation, more concepts.

**Q3. How do you fix N+1 queries in Rails, Django, EF Core, Drizzle and Mongoose?**
Rails: `includes` (or `preload`/`eager_load`). Django: `select_related` for foreign keys, `prefetch_related` for reverse and many-to-many relations. EF Core: `Include`/`ThenInclude`, optionally `AsSplitQuery`, or project with `Select`. Drizzle: an explicit join or `db.query.x.findMany({ with })`. Mongoose: `populate` (one extra query per path) or an aggregation `$lookup`.

**Q4. What is the difference between `includes`, `preload` and `eager_load` in Rails?**
`preload` always runs a separate query per association with `IN (...)`. `eager_load` always uses one LEFT OUTER JOIN query. `includes` picks for you: preload by default, eager_load when you reference the association's table in conditions or ordering (or call `references`).

**Q5. What is the difference between `select_related` and `prefetch_related` in Django?**
`select_related` follows foreign keys and one-to-one relations with a SQL JOIN in the same query. `prefetch_related` runs a separate query per relation and joins the results in Python, which works for reverse foreign keys and many-to-many relations; `Prefetch` objects let you filter or order the prefetched set.

**Q6. When would you write raw SQL instead of using the ORM, and how do you keep it safe?**
For reports with window functions or CTEs, bulk inserts and upserts, database-specific features (JSONB, full-text search, SKIP LOCKED) and hot paths where the generated SQL is measurably poor. Keep it safe with bind parameters (`find_by_sql` with `?`, Django params lists, EF Core `FromSql` interpolation, Dapper anonymous objects, Drizzle `sql` templates), never string interpolation of input.

**Q7. Drizzle or Prisma?**
Drizzle defines the schema in TypeScript, queries look like SQL, types are inferred with no generation step, and the runtime is light, which suits serverless and edge. Prisma uses its own schema language, generates a client with a very ergonomic nested API and an integrated migration workflow. Pick Drizzle for SQL control and a thin layer; Prisma for schema-first developer experience.

**Q8. EF Core or Dapper, and can you use both?**
EF Core gives LINQ, change tracking, migrations and relationship handling: productive for CRUD and domain logic. Dapper maps the results of SQL you write, with minimal overhead: ideal for reports, complex queries and hot paths. Many .NET teams use EF Core for writes and most reads, and Dapper for the heavy read queries, sharing the same database.

**Q9. Why can `validates :email, uniqueness: true` still allow duplicates?**
The validation runs a SELECT before the INSERT; two concurrent requests can both see no existing row and both insert. Only a unique index in the database makes it impossible. Keep the validation for friendly messages, add the index, and rescue `ActiveRecord::RecordNotUnique`.

**Q10. What problems do huge transactions cause in ORM code?**
Long-held locks block other requests, dead rows pile up because VACUUM cannot clean them, memory grows with tracked objects, statements time out, and a single failure rolls back hours of work. Batch the work, commit per batch, make it resumable and idempotent, and run side effects after commit.

**Q11. What is a micro-ORM?**
A thin library that executes the SQL you write and maps result rows to objects, without change tracking, relationship management or migrations. Dapper is the classic example. You get control and speed and give up the conveniences of a full ORM.

**Q12. What is lazy loading, and why is it risky?**
Lazy loading fetches an association only when it is first accessed. It is convenient, but loops that touch associations trigger one query each (N+1), and access after the context or transaction ends can fail. Eager load what you need, and use guards like Rails `strict_loading` to make accidental lazy loads raise.

**Q13. How do generated migrations differ from handwritten ones?**
Generated migrations (Django, EF Core, Drizzle, Prisma) diff your model or schema against the last state and emit changes automatically; handwritten ones (Rails) describe each change explicitly. Generated diffs are fast but can misread intent (a rename as drop and add), so review them; either way, production safety depends on additive, batched, lock-aware changes.

## Further reading
- Rails guides, Active Record query interface: https://guides.rubyonrails.org/active_record_querying.html
- Django docs, database access optimization: https://docs.djangoproject.com/en/stable/topics/db/optimization/
- EF Core, loading related data: https://learn.microsoft.com/en-us/ef/core/querying/related-data/
- Drizzle ORM docs: https://orm.drizzle.team/docs/overview
