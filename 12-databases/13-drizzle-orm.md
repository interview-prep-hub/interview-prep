# Drizzle ORM

> **Tier:** T1 (know cold) · **Related:** [ORMs compared](12-orms-compared.md), [PostgreSQL](04-postgresql.md), [Transactions and concurrency](06-transactions-and-concurrency.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [Multi-tenant platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md)
> **On your resume:** you contributed to migrating Colochousing onto a new multi-tenant TypeScript platform built with oRPC, TanStack Start and Drizzle: Rails sync APIs, the marketplace page migration and org-scoped access control.

## Quick revision
- Drizzle = **`drizzle-orm`** (a TypeScript schema plus a SQL-shaped query builder and a relational query API) + **`drizzle-kit`** (migrations CLI). No generated client and no separate engine: a thin layer over your driver (node-postgres, postgres.js and others).
- Schema: `pgTable("listings", { id: uuid("id").primaryKey().defaultRandom(), ... }, (t) => [index(...).on(...)])`, `pgEnum`, `.notNull()`, `.default()`, `.defaultNow()`, `.references(() => other.id, { onDelete })`, `uniqueIndex`, `primaryKey({ columns })`.
- Types come from the schema: `typeof table.$inferSelect` (row) and `typeof table.$inferInsert` (insert payload).
- Queries read like SQL: `db.select().from(t).where(and(eq(t.orgId, id), ilike(t.title, q))).orderBy(desc(t.createdAt)).limit(20)`; `insert().values().returning()`, `update().set().where()`, `delete().where()`, joins with `innerJoin`/`leftJoin`.
- Upsert: `` .onConflictDoUpdate({ target, set: { col: sql`excluded.db_column` } }) `` or `.onConflictDoNothing()`.
- Relational API: `db.query.organizations.findMany({ with: { listings: true } })`, needs `relations()` definitions and the schema passed to `drizzle()`; one SQL statement per call. Relations were reworked in the **1.0 line** (`defineRelations`, object-style filters): check the docs for your version.
- Transactions: `db.transaction(async (tx) => ...)`; `tx.rollback()` throws to abort; nested `tx.transaction` uses savepoints; `.for("update")` adds row locks.
- drizzle-kit: **`generate`** (SQL migration files from schema diffs), **`migrate`** (apply them), **`push`** (sync schema directly, prototypes only), **`pull`** (introspect an existing database), **`studio`** (data browser), configured in `drizzle.config.ts`.
- Multi-tenancy: an `org_id` on every tenant table, indexes and unique constraints that start with it, and a repository helper that always adds `eq(table.orgId, ctx.orgId)` with the org taken from the verified session; RLS as a second layer.
- `numeric` columns come back as strings (precision); timestamps should use `{ withTimezone: true }`.

## 1. What Drizzle is

Drizzle is a TypeScript ORM designed so that knowing SQL is most of what you need to use it. You declare tables in TypeScript, and both the database migrations and the static types come from those declarations. Queries are built with functions that mirror SQL clauses, so the generated SQL is predictable, and results are plain objects (no model instances, callbacks or lazy loading).

Why teams pick it: full type inference without a code-generation step, a small runtime that works well in serverless and edge environments, SQL-level control (joins, CTEs, `sql` fragments), and plain SQL migration files you can review.

## 2. Setup with PostgreSQL

```bash
npm i drizzle-orm pg
npm i -D drizzle-kit @types/pg
# or the postgres.js driver: npm i drizzle-orm postgres
```

```ts
// src/db/client.ts (node-postgres)
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });
export const db = drizzle({ client: pool, schema, logger: process.env.NODE_ENV === "development" });
export type DB = typeof db;
```

```ts
// Alternative: postgres.js
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "./schema";

const client = postgres(process.env.DATABASE_URL!, { max: 10, prepare: false }); // prepare: false behind transaction-mode poolers
export const db = drizzle({ client, schema });
```

Older Drizzle versions pass the client as the first argument (`drizzle(pool, { schema })`); recent versions also accept just a connection URL. Passing `schema` is what enables the relational `db.query` API.

## 3. Defining the schema

```ts
// src/db/schema.ts
import {
  pgTable, pgEnum, uuid, text, varchar, integer, boolean, timestamp, jsonb,
  index, uniqueIndex, primaryKey,
} from "drizzle-orm/pg-core";

export const memberRole = pgEnum("member_role", ["owner", "admin", "member"]);

const timestamps = {
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow()
    .$onUpdate(() => new Date()),                       // filled in by Drizzle when an update omits it
};

export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),           // DEFAULT gen_random_uuid()
  name: text("name").notNull(),
  slug: varchar("slug", { length: 64 }).notNull().unique(),
  ...timestamps,
});

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name"),
  ...timestamps,
});

export const memberships = pgTable(
  "memberships",
  {
    orgId: uuid("org_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    role: memberRole("role").notNull().default("member"),
  },
  (t) => [primaryKey({ columns: [t.orgId, t.userId] })]   // composite primary key
);

export const listings = pgTable(
  "listings",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id").notNull().references(() => organizations.id),
    createdById: uuid("created_by_id").references(() => users.id, { onDelete: "set null" }),
    railsId: integer("rails_id"),                         // id in the legacy Rails app, for idempotent sync
    title: text("title").notNull(),
    priceCents: integer("price_cents").notNull(),
    status: text("status", { enum: ["draft", "published", "archived"] }).notNull().default("draft"),
    featured: boolean("featured").notNull().default(false),
    metadata: jsonb("metadata").$type<{ tags?: string[] }>(),
    ...timestamps,
  },
  (t) => [
    index("listings_org_status_created_idx").on(t.orgId, t.status, t.createdAt),
    uniqueIndex("listings_org_rails_id_uq").on(t.orgId, t.railsId),
  ]
);

export const auditLogs = pgTable("audit_logs", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organizations.id),
  actorId: uuid("actor_id").references(() => users.id),
  action: text("action").notNull(),
  targetId: uuid("target_id"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Notes:
- The first argument of each column builder is the database column name (`"org_id"`); the object key (`orgId`) is the TypeScript name.
- `text("status", { enum: [...] })` narrows the TypeScript type without creating a Postgres enum; `pgEnum` creates a real enum type.
- Other column types: `bigint("id", { mode: "number" })` (pick `number` or `bigint` mode), `numeric("amount", { precision: 12, scale: 2 })` (returned as a string by default to avoid float rounding), `date`, `serial`, and identity columns via `.generatedAlwaysAsIdentity()`.
- The third argument of `pgTable` returns an array of indexes and constraints in recent versions; older versions returned an object.
- `.references()` creates a real foreign key; relations for the query API are separate (section 6).

## 4. Type inference

```ts
export type Listing = typeof listings.$inferSelect;     // what a select returns
export type NewListing = typeof listings.$inferInsert;  // what insert accepts: defaults and nullable columns optional

function toCard(l: Listing) {
  return { id: l.id, title: l.title, price: l.priceCents / 100, status: l.status };  // status: "draft" | "published" | "archived"
}
```

Partial selects infer their own shapes, and left-joined columns become nullable automatically. For request validation, generate Zod schemas from tables with the `drizzle-zod` package (`createInsertSchema(listings)`); the validator integrations are being reorganized in recent versions, so check the docs for the current package and import path.

## 5. Queries

```ts
import { and, or, eq, inArray, ilike, desc, sql, count } from "drizzle-orm";

// SELECT with filters, ordering and offset pagination
const page = await db
  .select()
  .from(listings)
  .where(and(eq(listings.orgId, orgId), eq(listings.status, "published")))
  .orderBy(desc(listings.createdAt))
  .limit(20)
  .offset(40);

// Partial select with search (values are bound as parameters, never interpolated)
const matches = await db
  .select({ id: listings.id, title: listings.title })
  .from(listings)
  .where(and(eq(listings.orgId, orgId), or(ilike(listings.title, `%${q}%`), eq(listings.featured, true))));

// INSERT ... RETURNING
const [created] = await db
  .insert(listings)
  .values({ orgId, title: "Room in Ixelles", priceCents: 65000 })
  .returning();

// UPDATE and DELETE (always scoped)
const archived = await db
  .update(listings)
  .set({ status: "archived" })
  .where(and(eq(listings.orgId, orgId), inArray(listings.id, ids)))
  .returning({ id: listings.id });

await db.delete(listings).where(and(eq(listings.orgId, orgId), eq(listings.id, id)));

// JOIN: creatorEmail is typed string | null because of the LEFT JOIN
const rows = await db
  .select({ id: listings.id, title: listings.title, creatorEmail: users.email })
  .from(listings)
  .leftJoin(users, eq(users.id, listings.createdById))
  .where(eq(listings.orgId, orgId));

// GROUP BY with an aggregate
const byStatus = await db
  .select({ status: listings.status, total: count() })
  .from(listings)
  .where(eq(listings.orgId, orgId))
  .groupBy(listings.status);
```

**Upserts** (idempotent sync from the legacy Rails app):
```ts
await db
  .insert(listings)
  .values(batchFromRails)                                // NewListing[] including orgId and railsId
  .onConflictDoUpdate({
    target: [listings.orgId, listings.railsId],          // must match a unique index or constraint
    set: {
      title: sql`excluded.title`,
      priceCents: sql`excluded.price_cents`,             // excluded.<database column name>, not the TS key
      status: sql`excluded.status`,
    },
  });

await db.insert(memberships).values({ orgId, userId }).onConflictDoNothing();
```
Deduplicate the batch first: Postgres rejects an upsert that hits the same key twice in one statement.

**Keyset pagination:**
```ts
const nextPage = await db
  .select()
  .from(listings)
  .where(and(
    eq(listings.orgId, orgId),
    cursor ? sql`(${listings.createdAt}, ${listings.id}) < (${cursor.createdAt}, ${cursor.id})` : undefined,
  ))                                                     // and() ignores undefined conditions
  .orderBy(desc(listings.createdAt), desc(listings.id))
  .limit(20);
```

**Raw SQL and inspection:**
```ts
const result = await db.execute(sql`SELECT count(*) FROM listings WHERE org_id = ${orgId}`);  // result shape depends on the driver
const { sql: text, params } = db.select().from(listings).where(eq(listings.id, id)).toSQL();  // see the generated SQL
```
`sql` template values are parameters; `sql.raw()` inserts text unescaped, so never pass user input to it.

## 6. The relational query API

Relations describe how tables connect for `db.query`. In the long-standing (v1) style they are declared with `relations()`:

```ts
// src/db/schema.ts (continued): relations must be exported from the module passed as `schema`
import { relations } from "drizzle-orm";

export const organizationsRelations = relations(organizations, ({ many }) => ({
  listings: many(listings),
  memberships: many(memberships),
}));

export const listingsRelations = relations(listings, ({ one }) => ({
  organization: one(organizations, { fields: [listings.orgId], references: [organizations.id] }),
  createdBy: one(users, { fields: [listings.createdById], references: [users.id] }),
}));

export const membershipsRelations = relations(memberships, ({ one }) => ({
  organization: one(organizations, { fields: [memberships.orgId], references: [organizations.id] }),
  user: one(users, { fields: [memberships.userId], references: [users.id] }),
}));
```

```ts
const org = await db.query.organizations.findFirst({
  where: eq(organizations.slug, slug),
  columns: { id: true, name: true },
  with: {
    listings: {
      where: eq(listings.status, "published"),
      orderBy: [desc(listings.createdAt)],
      limit: 10,
      with: { createdBy: { columns: { name: true } } },
    },
  },
});
// org: { id, name, listings: { ...listing, createdBy: { name } | null }[] } | undefined
```

- Each `findMany`/`findFirst` call compiles to a single SQL statement, so nested `with` does not cause N+1.
- Relations are a TypeScript-level description; they do not create foreign keys (`.references()` does).
- Use the relational API for nested reads (pages, API responses) and the SQL-like builder for filters across joins, aggregates and anything report-like.
- **1.0 rework:** the 1.0 line introduces `defineRelations` and an object-based syntax for filters and ordering in relational queries, with the older API kept for migration. Check the docs for the version your project uses before copying examples.

## 7. Transactions

```ts
const listing = await db.transaction(async (tx) => {
  const [row] = await tx
    .insert(listings)
    .values({ orgId, title, priceCents, createdById: userId })
    .returning();
  await tx.insert(auditLogs).values({ orgId, actorId: userId, action: "listing.created", targetId: row.id });
  return row;                                           // db.transaction resolves to this value
});
```

- Any thrown error rolls the transaction back and rejects the promise; `tx.rollback()` aborts deliberately (it throws, so catch it if you want a normal return).
- Nested `tx.transaction(async (sp) => ...)` uses a savepoint.
- Options such as `{ isolationLevel: "serializable" }` go in the second argument; serializable transactions need a retry loop ([06](06-transactions-and-concurrency.md)).
- Row locks: `tx.select().from(listings).where(eq(listings.id, id)).for("update")`.
- Use `tx`, not `db`, inside the callback; a `db` call would run outside the transaction.

## 8. drizzle-kit and migrations

```ts
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "postgresql",
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

| Command | What it does | When |
|---|---|---|
| `npx drizzle-kit generate` | Diffs the TS schema against the last snapshot and writes SQL migration files to `out` | Every schema change; commit the files and review the SQL |
| `npx drizzle-kit migrate` | Applies pending migration files and records them in a migrations table | CI/CD and production deploys |
| `npx drizzle-kit push` | Applies the schema diff directly, without files | Local prototyping only |
| `npx drizzle-kit pull` | Introspects an existing database into a TS schema | Adopting Drizzle on a database created by another tool (such as Rails) |
| `npx drizzle-kit studio` | Opens a local browser UI for the data | Inspecting and editing data in development |

Migrations can also run from code at startup or in a deploy script:
```ts
import { migrate } from "drizzle-orm/node-postgres/migrator";
await migrate(db, { migrationsFolder: "./drizzle" });
```

Review generated SQL like any code: a renamed column can come out as drop-and-add (drizzle-kit asks interactively when it detects possible renames), and production changes still need the usual care (additive first, `CREATE INDEX CONCURRENTLY` written by hand where needed, batched backfills). Older drizzle-kit versions used different config keys (for example a `driver` option), so match the docs to your installed version.

## 9. Multi-tenant scoping

On a shared-schema multi-tenant platform, every tenant-owned table carries `org_id`, every composite index and unique constraint starts with it, and **every query must filter by the current org**, taken from the verified session, never from the request body or URL alone. Make the safe path the easy path with a scoped repository:

```ts
// src/db/scoped.ts
import { and, desc, eq, type SQL } from "drizzle-orm";
import { db } from "./client";
import { listings, type NewListing } from "./schema";

export type OrgContext = { orgId: string; userId: string; role: "owner" | "admin" | "member" };

export function listingsFor(ctx: OrgContext) {
  const inOrg = (extra?: SQL) => and(eq(listings.orgId, ctx.orgId), extra);

  return {
    list: (extra?: SQL) =>
      db.select().from(listings).where(inOrg(extra)).orderBy(desc(listings.createdAt)),
    get: async (id: string) => {
      const [row] = await db.select().from(listings).where(inOrg(eq(listings.id, id)));
      return row ?? null;                     // another org's listing is indistinguishable from "not found"
    },
    create: (input: Omit<NewListing, "id" | "orgId" | "createdById">) =>
      db.insert(listings).values({ ...input, orgId: ctx.orgId, createdById: ctx.userId }).returning(),
    update: (id: string, patch: Partial<Omit<NewListing, "id" | "orgId">>) =>
      db.update(listings).set(patch).where(inOrg(eq(listings.id, id))).returning(),
    remove: (id: string) => db.delete(listings).where(inOrg(eq(listings.id, id))),
  };
}
```

- Role checks (only admins may delete) belong next to this layer, in middleware or the procedure.
- Joins need the scope on every tenant table involved, not just the first one.
- Test it: create two orgs and assert that every repository method used by org A returns nothing (or "not found") for org B's ids.
- **RLS as a second layer**: run each request's queries in a transaction that sets the org, and let Postgres policies filter ([PostgreSQL RLS](04-postgresql.md)). Recent Drizzle versions can also declare policies in the schema (`pgPolicy`); check the docs.

```ts
const rows = await db.transaction(async (tx) => {
  await tx.execute(sql`SELECT set_config('app.current_org_id', ${ctx.orgId}, true)`);  // transaction-local
  return tx.select().from(listings);          // the RLS policy limits rows to the current org
});
```

## 10. Drizzle vs Prisma vs ActiveRecord

| | Drizzle | Prisma | ActiveRecord |
|---|---|---|---|
| Language | TypeScript | TypeScript | Ruby |
| Schema lives in | TS files (`pgTable`) | `schema.prisma` | Migrations; models read columns at runtime |
| Query style | SQL-shaped builder plus a relational API | Nested object API (`findMany({ where, include })`) | Chainable relations (`where`, `joins`, `includes`) |
| Types | Inferred from the schema, no codegen | Generated client (`prisma generate`) | Dynamic |
| Migrations | drizzle-kit: SQL files (`generate`/`migrate`), `push` | Prisma Migrate | Ruby DSL migrations |
| Runtime | Thin layer over your driver | Generated client and query engine (architecture changed in recent versions) | Part of Rails |
| Model behavior | None: plain data (validate with Zod) | None | Validations, callbacks, scopes built in |
| Avoiding N+1 | Joins or `with` (one statement) | `include` / nested `select` | `includes`, `preload`, `eager_load` |
| Raw SQL | `sql` tagged template | `$queryRaw` tagged template | `find_by_sql`, `connection.select_all` |

Coming from ActiveRecord, the big shifts are: no callbacks or validations on models (put rules in services and Zod schemas), no lazy loading (you always state what to load), and explicit tenant scoping instead of `default_scope` or `acts_as_tenant`-style magic.

## 11. Drizzle in oRPC procedures and TanStack Start server functions

**oRPC** procedures get typed input and context; middleware turns the session into an org context and the procedure calls the scoped repository. Package names and some APIs have changed between oRPC releases, so check [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md) and the docs.

```ts
import { os, ORPCError } from "@orpc/server";
import { z } from "zod";
import { eq } from "drizzle-orm";
import { listings } from "./db/schema";
import { listingsFor, type OrgContext } from "./db/scoped";

const base = os.$context<{ session: OrgContext | null }>();

const authed = base.use(({ context, next }) => {
  if (!context.session) throw new ORPCError("UNAUTHORIZED");
  return next({ context: { session: context.session } });   // downstream context has a non-null session
});

export const listingRouter = {
  list: authed
    .input(z.object({ status: z.enum(["draft", "published", "archived"]).optional() }))
    .handler(async ({ input, context }) =>
      await listingsFor(context.session).list(input.status ? eq(listings.status, input.status) : undefined)
    ),
  create: authed
    .input(z.object({ title: z.string().min(3), priceCents: z.number().int().positive() }))
    .handler(async ({ input, context }) => {
      const [row] = await listingsFor(context.session).create(input);
      return row;
    }),
};
```

**TanStack Start** server functions run only on the server, so they can import the Drizzle client directly; derive the org from the session inside the function. Start's APIs (package name, input-validation method) changed during its beta and release-candidate phase, so confirm against [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md) and the docs.

```ts
import { createServerFn } from "@tanstack/react-start";
import { listingsFor } from "./db/scoped";

export const getDashboardListings = createServerFn({ method: "GET" }).handler(async () => {
  const session = await requireSession();      // your auth helper: reads the session cookie, throws if missing
  return listingsFor(session).list();          // returns plain, serializable rows
});
```

Keep the Drizzle client in server-only modules so it never ends up in a client bundle, and return plain data rather than driver objects.

## Common pitfalls
- Forgetting the org filter on one query (or one joined table): a cross-tenant data leak. Centralize scoping and test it.
- Trusting an `orgId` sent by the client instead of the one in the verified session.
- Using `push` against shared or production databases instead of reviewed `generate` + `migrate` files.
- Writing `` sql`excluded.priceCents` `` (the TS key) instead of the database column name `excluded.price_cents`.
- Upserting batches that contain the same conflict key twice.
- Calling `db` instead of `tx` inside a transaction callback.
- Expecting `numeric` columns to be numbers; they are strings unless you choose a number mode or convert deliberately.
- `timestamp` without `withTimezone: true`, mixing zones.
- Prepared statements behind transaction-mode PgBouncer (set `prepare: false` with postgres.js).
- Copying relational-query examples from a different major version (pre-1.0 vs 1.0 relations syntax).
- Interpolating user input with `sql.raw()`.

## How to talk about this in an interview
- Your story: on the Colochousing platform migration you worked on the new multi-tenant TypeScript stack (oRPC, TanStack Start, Drizzle): Rails sync APIs feeding the new platform, migrating the marketplace page, and org-scoped access control. Explain how a request becomes an org context and how every query is scoped.
- Contrast with ActiveRecord honestly: what you gained (types end to end, explicit SQL, no hidden callbacks) and what you had to rebuild (validation in Zod, explicit eager loading, tenant scoping).
- If asked about migrations, describe the generate, review, migrate flow and how the legacy Rails database and the new schema coexist during the migration.

> **Fill in:** how the Rails sync APIs move data into the new platform (push from Rails, pull from the TypeScript side, a background job), and how duplicates are avoided (for example a unique legacy id with upserts).

> **Fill in:** how org-scoped access is enforced on the platform: middleware, scoped helpers, RLS, or a mix; and one bug or edge case you handled.

> **Fill in:** which Drizzle features you used most (relational queries, joins, transactions, drizzle-kit commands) and whether the project is on the pre-1.0 or 1.0 relations API.

## Interview questions

**Q1. What is Drizzle, and how is it different from Prisma?**
Drizzle is a TypeScript ORM where you define tables in TypeScript and query with a SQL-shaped builder (plus a relational API); types are inferred directly from the schema and there is no code generation or separate engine. Prisma uses its own schema language and generates a client with a nested object API. Drizzle gives SQL control and a thin runtime; Prisma gives a schema-first workflow and a very ergonomic generated client.

**Q2. What are `$inferSelect` and `$inferInsert`?**
Type helpers on a table: `typeof listings.$inferSelect` is the row type a select returns; `typeof listings.$inferInsert` is the payload `insert().values()` accepts, where columns with defaults or that are nullable are optional. They keep API types in sync with the schema without hand-written interfaces.

**Q3. What is the difference between `drizzle-kit generate`, `migrate`, `push`, `pull` and `studio`?**
`generate` diffs your TS schema against the last snapshot and writes SQL migration files; `migrate` applies pending files and records them; `push` applies the diff straight to a database without files (prototyping only); `pull` introspects an existing database into a TS schema; `studio` opens a local data browser. Production uses generate (reviewed, committed) plus migrate.

**Q4. When do you use the relational query API instead of joins?**
The relational API (`db.query.x.findMany({ with })`) is best for nested reads like "an organization with its 10 latest listings and each listing's creator", returning typed nested objects in a single SQL statement. The SQL-like builder is better for filtering across joins, aggregates, grouping, and anything report-like. Relations must be defined and the schema passed to `drizzle()`.
*Follow-up:* Do relations create foreign keys? No; `.references()` on columns does. Relations only describe joins for the query API.

**Q5. How do you write an upsert in Drizzle, and what is the common mistake?**
`` insert(table).values(rows).onConflictDoUpdate({ target: [cols], set: { col: sql`excluded.column_name` } }) ``, or `onConflictDoNothing()`. The target must match a unique index or constraint. The common mistakes are using the TypeScript property name instead of the database column name after `excluded.`, and sending the same key twice in one batch.

**Q6. How do transactions work in Drizzle, and how do you roll back?**
`db.transaction(async (tx) => { ... })` commits when the callback resolves and rolls back when it throws; `tx.rollback()` throws on purpose. The call returns the callback's value. Nested `tx.transaction` creates savepoints, the second argument sets options like the isolation level, and every query inside must use `tx`.

**Q7. How do you make sure every query is scoped to the current tenant?**
Put `org_id` on every tenant table, derive the org from the authenticated session in middleware, and route all data access through scoped helpers (`listingsFor(ctx)`) that always add `eq(table.orgId, ctx.orgId)`, including on updates, deletes and joined tables. Return "not found" for other orgs' ids, write cross-tenant tests, and add Postgres RLS as a safety net.

**Q8. How would you add row-level security with Drizzle?**
Enable RLS with policies comparing `org_id` to `current_setting('app.current_org_id', true)`, connect as a role that does not bypass RLS, and wrap each request's queries in a transaction that first runs `SELECT set_config('app.current_org_id', <org>, true)`. Transaction-local settings stay safe with connection pooling. Recent Drizzle versions can declare policies in the schema.

**Q9. Why does a `numeric` column come back as a string?**
Postgres `numeric` can hold more precision than a JavaScript number, so drivers return it as a string to avoid silent rounding, and Drizzle types it as `string` by default. Convert deliberately (a decimal library for money math), use a number mode where precision allows, or store money as integer cents.

**Q10. How do you avoid N+1 queries with Drizzle?**
Drizzle has no lazy loading, so N+1 only appears if you query inside a loop yourself. Load related data with a join, an `inArray` query for all ids at once, or the relational API's `with`, which compiles to one SQL statement.

**Q11. How do you see the SQL Drizzle runs?**
Enable the `logger` option when creating the client, or call `.toSQL()` on a query to get the SQL string and parameters without running it. In Postgres, `pg_stat_statements` and the slow-query log show the same statements in production.

**Q12. What do you configure for Drizzle with PgBouncer or serverless?**
Use a pooled connection string, keep the client's pool small (each instance has its own), disable prepared statements for transaction-mode poolers (`prepare: false` in postgres.js), avoid session-level state (use `set_config(..., true)` inside transactions), and create the client once per process.

**Q13. How would you introduce Drizzle on a database that Rails already manages?**
Run `drizzle-kit pull` to introspect the existing tables into a TS schema, then decide which side owns migrations: during a migration period usually one owner (often still Rails) while the TypeScript side reads and writes through the introspected schema; later, move ownership to drizzle-kit with a baseline migration. Sync data with idempotent upserts keyed on legacy ids.

**Q14. How do you validate input when Drizzle models have no validations?**
At the API boundary with Zod: oRPC `.input(schema)`, a server-function validator, or schemas generated from tables with `drizzle-zod`. Database constraints (NOT NULL, CHECK, unique indexes, foreign keys) remain the final guarantee, and unique-violation errors are mapped to friendly messages.

**Q15. How do you implement keyset pagination in Drizzle?**
Order by a unique combination such as `createdAt DESC, id DESC`, return the last row's values as an opaque cursor, and on the next request add a row-comparison condition: ``sql`(${t.createdAt}, ${t.id}) < (${c.createdAt}, ${c.id})` `` together with the org filter and `limit`. Back it with an index on `(org_id, created_at, id)`.

**Q16. How do you define reusable columns like timestamps?**
Put the column builders in a plain object (`const timestamps = { createdAt: ..., updatedAt: ... }`) and spread it into each `pgTable` definition. Use `defaultNow()` for creation time and `$onUpdate(() => new Date())` so Drizzle sets `updatedAt` on updates it performs (raw SQL updates bypass it).

## Further reading
- Drizzle ORM docs: https://orm.drizzle.team/docs/overview
- Drizzle relational queries: https://orm.drizzle.team/docs/rqb
- drizzle-kit overview: https://orm.drizzle.team/docs/kit-overview
