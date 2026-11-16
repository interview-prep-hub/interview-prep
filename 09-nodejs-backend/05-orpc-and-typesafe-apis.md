# oRPC and Type-Safe APIs

> **Tier:** T1 (know cold) · **Related:** [Hono](04-hono.md), [Production APIs](06-production-apis.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [Drizzle ORM](../12-databases/13-drizzle-orm.md), [Data fetching with TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [GraphQL](../04-web-fundamentals/05-graphql.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)
> **On your resume:** the multi-tenant TypeScript platform at Devntech (oRPC, TanStack Start, Drizzle), where you contributed Rails sync APIs, the marketplace page migration and org-scoped access control; oRPC is also listed under Node.js in your skills.

> **Version note:** code follows the oRPC v1 style (`@orpc/server`, `@orpc/client`, `@orpc/tanstack-query`, `@orpc/contract`, `@orpc/openapi`); check the docs. oRPC's docs site has started documenting a v2 line published under the npm `beta` tag, with some renamed options (OpenAPI route metadata, link and generator options), so confirm which major your `package.json` uses before copying exact imports.

## Quick revision
- End-to-end type safety: one source of truth for the API's shape, so client calls and server handlers are checked against each other at compile time; runtime validation at the boundary keeps those types honest.
- The spectrum: REST + OpenAPI codegen (any language, generated clients), GraphQL + codegen (typed schema, flexible queries), tRPC and oRPC (TypeScript only, types inferred from the server, no codegen).
- An oRPC procedure: `os.input(schema).output(schema).handler(async ({ input, context }) => ...)`. Zod, Valibot or any Standard Schema library works.
- Routers are plain nested objects (`{ marketplace: { list, get } }`), and the client mirrors them: `client.marketplace.list(input)`.
- **Initial context** is passed in by whoever calls the router (`handler.handle(request, { context })`), typed with `os.$context<T>()`. **Execution context** (the docs also say injected context) is added by middleware through `next({ context })`.
- `.use()` middleware authenticates, loads the org membership and injects `{ session, org }`; every procedure built on that base is protected automatically.
- Errors: `throw new ORPCError('NOT_FOUND')` (codes map to HTTP statuses); typed error maps with `.errors({...})` and `throw errors.CONFLICT({ data })`; clients narrow with `safe()` and `isDefinedError()`.
- Invalid input becomes `BAD_REQUEST`; output that fails its schema becomes `INTERNAL_SERVER_ERROR`, so output schemas catch leaked fields.
- Serving: `new RPCHandler(router)` from `@orpc/server/fetch`, then `handler.handle(request, { prefix, context })` in Hono middleware, a TanStack Start server route, Next.js, Workers, Bun, or Node (`@orpc/server/node`).
- Client: `createORPCClient(new RPCLink({ url }))`, typed as `RouterClient<typeof router>`; import the router type with `import type` so no server code ships.
- The RPC protocol keeps `Date`, `BigInt`, `Map`, `Set` and `File`/`Blob` intact across the wire.
- TanStack Query: `createTanstackQueryUtils(client)`, then `useQuery(orpc.x.y.queryOptions({ input }))`, `useMutation(orpc.x.y.mutationOptions())`, and invalidate with `orpc.x.key()`.
- Contract-first: define the API with `oc` from `@orpc/contract`, share it, implement it with `implement(contract)`.
- OpenAPI: give procedures HTTP routes, serve them with `OpenAPIHandler`, generate a spec with `OpenAPIGenerator`; Rails, mobile and partner clients can integrate without TypeScript.
- oRPC vs tRPC: oRPC has no query/mutation split, has first-class OpenAPI and contract-first support, typed error maps and native types without a transformer; tRPC is older with a larger community.
- Testing: `call(procedure, input, { context })` or `createRouterClient(router, { context })` runs procedures in-process with a fake context.

## 1. What end-to-end type safety means

Without it, the backend renames `price` to `priceCents`, the frontend keeps reading `price`, and the bug surfaces in production as `NaN`. With it, the rename breaks the frontend's build. The chain looks like this:

```text
Drizzle table -> row types -> procedure output -> typed client -> TanStack Query hook -> component props
```

Two halves make it work:
- **Static:** the client's types are derived from the server (inferred from the router, or from a shared contract), so mismatches are compile errors.
- **Runtime:** types are erased, and anyone can `curl` your endpoint with garbage. Input schemas reject bad requests on the server; output schemas stop the server from returning more (or other) data than promised.

Type safety reduces a class of integration bugs; it does not replace validation, authorization or tests.

## 2. The type-safety spectrum

| Approach | Source of truth | Client types | Non-TypeScript clients | Trade-offs |
|---|---|---|---|---|
| REST with hand-written types | docs and memory | manual, drift | yes | cheap to start, drift bugs later |
| REST + OpenAPI | an OpenAPI spec (code-first or design-first) | generated (for example openapi-typescript) | yes, any language | codegen step, spec upkeep; great HTTP caching and tooling |
| GraphQL | the schema (SDL) | generated (GraphQL Code Generator) | yes | flexible queries; server complexity, N+1 risk, weaker HTTP caching |
| tRPC | the TypeScript router type | inferred, no codegen | poor | TypeScript monorepos only |
| oRPC | the router type, or a shared contract | inferred, no codegen | yes, through generated OpenAPI | younger ecosystem |
| Hono RPC | chained route types | inferred | yes, routes are plain HTTP | types depend on route chaining |
| Framework functions (Next.js Server Actions, TanStack Start server functions) | the function itself | inferred inside one app | no | coupled to the framework |

## 3. Procedures and routers

```ts
// src/server/orpc.ts: shared base builders (oRPC v1 style)
import { os } from '@orpc/server';
import type { DB } from './db/client';

export type Session = { userId: string; activeOrgId: string | null };
export type InitialContext = { db: DB; session: Session | null; headers: Headers };

export const base = os.$context<InitialContext>(); // callers must provide this context
```

```ts
// src/server/routers/health.ts
import { z } from 'zod';
import { base } from '../orpc';

export const ping = base
  .input(z.object({ name: z.string().min(1).optional() }))
  .output(z.object({ message: z.string(), at: z.date() }))
  .handler(async ({ input }) => ({ message: `pong ${input.name ?? ''}`.trim(), at: new Date() }));

// src/server/router.ts
export const router = { health: { ping } /*, marketplace: marketplaceRouter */ };
export type AppRouter = typeof router;
```

- Procedures are values and routers are objects. Nesting is just nesting objects; there is no registration step.
- `.input()` validates before the handler runs. `.output()` validates the return value; Zod objects strip unknown keys, so a stray `passwordHash` never leaves the server.
- The handler receives `input`, `context` and `errors` (for typed errors), among other fields.
- There is one kind of procedure. Whether a call behaves like a query or a mutation is decided by how the client uses it.

## 4. Context and middleware: authentication and org scoping

```ts
// src/server/orpc.ts (continued)
import { ORPCError } from '@orpc/server';
import { and, eq } from 'drizzle-orm';
import { memberships } from './db/schema';

export const authed = base.use(async ({ context, next }) => {
  if (!context.session) throw new ORPCError('UNAUTHORIZED');
  return next({ context: { session: context.session } });   // downstream sees a non-null session
});

export const orgProcedure = authed.use(async ({ context, next }) => {
  const orgId = context.session.activeOrgId;
  if (!orgId) throw new ORPCError('FORBIDDEN', { message: 'No active organization' });
  const [membership] = await context.db
    .select({ orgId: memberships.orgId, role: memberships.role })
    .from(memberships)
    .where(and(eq(memberships.userId, context.session.userId), eq(memberships.orgId, orgId)));
  if (!membership) throw new ORPCError('FORBIDDEN');          // stale or forged org id
  return next({ context: { org: { id: membership.orgId, role: membership.role } } });
});

export const adminProcedure = orgProcedure.use(async ({ context, next }) => {
  if (context.org.role !== 'owner' && context.org.role !== 'admin') throw new ORPCError('FORBIDDEN');
  return next();
});
```

| | Initial context | Execution (injected) context |
|---|---|---|
| Who provides it | the caller: the HTTP adapter, a test, a server-side client | middleware, during the call |
| How | `handler.handle(req, { context })`, `call(proc, input, { context })` | `return next({ context: { org } })` |
| Typed by | `os.$context<T>()` (callers are forced to pass it) | inferred from what middleware returns |
| Holds | environment values: db, headers, the resolved session | derived values: user, org, role, loaders |

Multi-tenant rules that interviewers like to hear:
- Never authorize with an `orgId` taken from input. Derive it from the session, verify the membership on each request, and put it in context.
- Every tenant query filters by `context.org.id`, ideally through a scoped repository ([Drizzle scoping](../12-databases/13-drizzle-orm.md)).
- Return `NOT_FOUND` for another org's ids, so attackers cannot probe which ids exist.

## 5. Errors

| Code | HTTP status | Typical cause |
|---|---|---|
| `BAD_REQUEST` | 400 | input failed validation |
| `UNAUTHORIZED` | 401 | no or invalid session |
| `FORBIDDEN` | 403 | authenticated but not allowed |
| `NOT_FOUND` | 404 | missing (or another org's) resource |
| `CONFLICT` | 409 | unique constraint, version conflict |
| `TOO_MANY_REQUESTS` | 429 | rate limited |
| `INTERNAL_SERVER_ERROR` | 500 | unexpected error, output validation failure |

Unknown errors thrown in handlers are converted into a generic `INTERNAL_SERVER_ERROR`, so log them centrally (an `onError` interceptor on the handler). For failures the client should handle, declare **typed errors**:

```ts
export const createListing = adminProcedure
  .errors({
    CONFLICT: { message: 'Slug already taken', data: z.object({ slug: z.string() }) },
  })
  .input(z.object({ title: z.string().min(3), slug: z.string().min(3), priceCents: z.number().int().positive() }))
  .handler(async ({ input, context, errors }) => {
    if (await slugExists(context.db, context.org.id, input.slug)) {
      throw errors.CONFLICT({ data: { slug: input.slug } });   // data is checked against the schema
    }
    return insertListing(context.db, context.org.id, input);
  });
```

```ts
// client side
import { isDefinedError, safe } from '@orpc/client';

const [error, listing] = await safe(client.marketplace.create({ title, slug, priceCents }));
if (isDefinedError(error) && error.code === 'CONFLICT') {
  setFieldError('slug', `"${error.data.slug}" is already taken`); // typed from the error map
} else if (error) {
  toast.error('Could not create the listing');
}
```

## 6. Serving the router

The fetch adapter turns a standard `Request` into a `Response`, so it mounts anywhere. In Hono:

```ts
import { onError } from '@orpc/server';
import { RPCHandler } from '@orpc/server/fetch';
import { Hono } from 'hono';
import { getSession } from './auth';
import { db } from './db/client';
import { router } from './router';

const rpc = new RPCHandler(router, {
  interceptors: [onError((error) => console.error(error))], // central logging of unexpected errors
});

export const app = new Hono();
app.use('/rpc/*', async (c, next) => {
  const headers = c.req.raw.headers;
  const { matched, response } = await rpc.handle(c.req.raw, {
    prefix: '/rpc',
    context: { db, session: await getSession(headers), headers },
  });
  if (matched) return c.newResponse(response.body, response);
  await next();
});
```

In a TanStack Start server route (the shape the oRPC TanStack Start adapter docs show; check both libraries' docs for your versions):

```ts
// src/routes/api/rpc.$.ts
import { RPCHandler } from '@orpc/server/fetch';
import { createFileRoute } from '@tanstack/react-router';
import { getSession } from '@/server/auth';
import { db } from '@/server/db/client';
import { router } from '@/server/router';

const rpc = new RPCHandler(router);

export const Route = createFileRoute('/api/rpc/$')({
  server: {
    handlers: {
      ANY: async ({ request }) => {
        const { response } = await rpc.handle(request, {
          prefix: '/api/rpc',
          context: { db, session: await getSession(request.headers), headers: request.headers },
        });
        return response ?? new Response('Not found', { status: 404 });
      },
    },
  },
});
```

On plain Node, `RPCHandler` from `@orpc/server/node` takes `(req, res, { prefix, context })`. Resolving the session in the adapter keeps procedures easy to test; the alternative is middleware that reads cookies from `context.headers` through your auth library.

## 7. The client

```ts
// src/lib/orpc-client.ts (browser)
import { createORPCClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { RouterClient } from '@orpc/server';
import type { AppRouter } from '@/server/router'; // type-only: erased at build time

const link = new RPCLink({
  url: `${window.location.origin}/api/rpc`,
  headers: () => ({ 'x-request-id': crypto.randomUUID() }),          // evaluated per request
  // cross-origin cookies: fetch: (request, init) => fetch(request, { ...init, credentials: 'include' }),
});

export const client: RouterClient<AppRouter> = createORPCClient(link);
// const { items, total } = await client.marketplace.list({ page: 1 }); fully typed
```

During SSR, calling your own server over HTTP is wasteful. `createRouterClient(router, { context })` from `@orpc/server` gives the same typed interface but runs procedures in-process; in TanStack Start, `createIsomorphicFn` can pick the server client during SSR and the `RPCLink` client in the browser (the oRPC TanStack Start docs show this pattern).

## 8. TanStack Query integration

```ts
// src/lib/orpc.ts
import { createTanstackQueryUtils } from '@orpc/tanstack-query';
import { client } from './orpc-client';

export const orpc = createTanstackQueryUtils(client);
```

```tsx
const listings = useQuery(orpc.marketplace.list.queryOptions({ input: { page: 1 } }));

const queryClient = useQueryClient();
const createListing = useMutation(
  orpc.marketplace.create.mutationOptions({
    onSuccess: () => queryClient.invalidateQueries({ queryKey: orpc.marketplace.key() }), // all marketplace queries
  }),
);
// createListing.mutate({ title, slug, priceCents });
```

- `queryOptions({ input })` builds the query key and `queryFn` from the procedure path and input, so keys never drift from the API.
- `.key()` is a partial-match key for invalidation; `.queryKey({ input })` is the exact key for `setQueryData`; `infiniteOptions({ input: (pageParam) => ..., initialPageParam, getNextPageParam })` drives infinite lists.
- Older releases shipped a React-specific package with different helper names; check the docs for your version. More on queries, keys and caching in [TanStack Query](../06-react/04-data-fetching-tanstack-query.md).

## 9. Contract-first development

By default oRPC is router-first: the implementation is the source of truth. Contract-first separates the API definition from the implementation:

```ts
// packages/contract/src/index.ts: shared by the server and every client
import { oc } from '@orpc/contract';
import { z } from 'zod';

export const ListingDto = z.object({ id: z.string(), title: z.string(), priceCents: z.number().int(), createdAt: z.date() });

export const contract = {
  marketplace: {
    list: oc
      .input(z.object({ page: z.number().int().min(1).default(1), category: z.string().optional() }))
      .output(z.object({ items: z.array(ListingDto), total: z.number().int() })),
  },
};
```

```ts
// apps/api/src/router.ts: the compiler forces every procedure to match the contract
import { implement } from '@orpc/server';
import { contract } from '@acme/contract';

const impl = implement(contract).$context<InitialContext>();

export const router = impl.router({
  marketplace: {
    list: impl.marketplace.list.handler(async ({ input, context }) => listPublished(context, input)),
  },
});
```

Use it when teams or repositories are separate, when frontend and backend work in parallel from an agreed API, or when you publish an SDK; clients type themselves from the contract (`ContractRouterClient<typeof contract>`) without importing server code. In a single monorepo owned by one team, router-first is simpler.

## 10. OpenAPI generation

A Ruby on Rails app cannot import TypeScript types. It needs documented HTTP endpoints, which oRPC can expose and describe:

```ts
// a procedure that is also a REST endpoint (oRPC v1 style: .route(); the v2 beta moves this metadata)
export const listPublic = base
  .route({ method: 'GET', path: '/marketplace/{orgSlug}/listings', summary: 'Published listings of an org' })
  .input(z.object({ orgSlug: z.string(), page: z.coerce.number().int().min(1).default(1) }))
  .output(z.object({ items: z.array(ListingDto), total: z.number().int() }))
  .handler(async ({ input, context }) => listPublishedBySlug(context.db, input));
```

```ts
import { OpenAPIGenerator } from '@orpc/openapi';
import { OpenAPIHandler } from '@orpc/openapi/fetch';
import { ZodToJsonSchemaConverter } from '@orpc/zod/zod4'; // '@orpc/zod' for Zod 3

export const openapiHandler = new OpenAPIHandler(router); // serves REST-style routes, mounted like RPCHandler

const generator = new OpenAPIGenerator({ schemaConverters: [new ZodToJsonSchemaConverter()] });
export const spec = await generator.generate(router, { info: { title: 'Platform API', version: '1.0.0' } });
// serve `spec` as JSON and point Swagger UI or Scalar at it; Ruby or Kotlin clients can be generated from it
```

## 11. End-to-end example: an org-scoped `marketplace.list`

**Schema** (Drizzle; every tenant table has `org_id`, and indexes start with it):

```ts
// src/server/db/schema.ts (excerpt)
import { index, integer, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core';

export const listings = pgTable('listings', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull(),
  title: text('title').notNull(),
  category: text('category').notNull(),
  status: text('status', { enum: ['draft', 'published'] }).notNull().default('draft'),
  priceCents: integer('price_cents').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [index('listings_org_status_created_idx').on(t.orgId, t.status, t.createdAt)]);
```

**Procedure:**

```ts
// src/server/routers/marketplace.ts
import { and, count, desc, eq } from 'drizzle-orm';
import { z } from 'zod';
import { listings } from '../db/schema';
import { orgProcedure } from '../orpc';

const ListInput = z.object({
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(50).default(20),
  category: z.string().optional(),
});
const ListingDto = z.object({
  id: z.string(), title: z.string(), category: z.string(), priceCents: z.number().int(), createdAt: z.date(),
});

export const marketplaceRouter = {
  list: orgProcedure
    .input(ListInput)
    .output(z.object({ items: z.array(ListingDto), total: z.number().int() }))
    .handler(async ({ input, context }) => {
      const where = and(
        eq(listings.orgId, context.org.id),            // tenant scope from verified context, never from input
        eq(listings.status, 'published'),
        input.category ? eq(listings.category, input.category) : undefined, // and() skips undefined
      );
      const [items, totals] = await Promise.all([
        context.db
          .select({ id: listings.id, title: listings.title, category: listings.category,
                    priceCents: listings.priceCents, createdAt: listings.createdAt })
          .from(listings).where(where)
          .orderBy(desc(listings.createdAt))
          .limit(input.pageSize).offset((input.page - 1) * input.pageSize),
        context.db.select({ total: count() }).from(listings).where(where),
      ]);
      return { items, total: totals[0]?.total ?? 0 };
    }),
};
```

**Component** (the router is served at `/api/rpc` as in section 6; `orpc` comes from section 8):

```tsx
import { keepPreviousData, useQuery } from '@tanstack/react-query';
import { useState } from 'react';
import { orpc } from '@/lib/orpc';

const PAGE_SIZE = 20;

export function MarketplaceList({ category }: { category?: string }) {
  const [page, setPage] = useState(1);
  const { data, isPending, isError, error, isPlaceholderData } = useQuery({
    ...orpc.marketplace.list.queryOptions({ input: { page, pageSize: PAGE_SIZE, category } }),
    placeholderData: keepPreviousData,      // keep the current page on screen while the next loads
    staleTime: 30_000,
  });

  if (isPending) return <p>Loading listings...</p>;
  if (isError) return <p role="alert">Could not load listings: {error.message}</p>;

  const lastPage = Math.max(1, Math.ceil(data.total / PAGE_SIZE));
  return (
    <section>
      <ul>
        {data.items.map((l) => (
          <li key={l.id}>{l.title}: {(l.priceCents / 100).toFixed(2)} (listed {l.createdAt.toLocaleDateString()})</li>
        ))}
      </ul>
      <button onClick={() => setPage((p) => p - 1)} disabled={page === 1}>Previous</button>
      <span> Page {page} of {lastPage} </span>
      <button onClick={() => setPage((p) => p + 1)} disabled={isPlaceholderData || page >= lastPage}>Next</button>
    </section>
  );
}
```

`l.createdAt` is a real `Date` on the client because the RPC protocol preserves it. Rename `priceCents` in the Drizzle schema and the procedure, the DTO and this component all fail to compile until they agree.

## 12. Testing procedures

```ts
import { call } from '@orpc/server';
import { describe, expect, it } from 'vitest';
import { seedOrgWithListings, testDb } from '../test/db';   // helpers over a real test database
import { router } from './router';

const ctx = (session: { userId: string; activeOrgId: string } | null) =>
  ({ db: testDb, session, headers: new Headers() });

describe('marketplace.list', () => {
  it('rejects anonymous callers', async () => {
    await expect(call(router.marketplace.list, { page: 1 }, { context: ctx(null) }))
      .rejects.toMatchObject({ code: 'UNAUTHORIZED' });
  });

  it('only returns listings from the active org', async () => {
    const orgA = await seedOrgWithListings(testDb, 3);
    const orgB = await seedOrgWithListings(testDb, 2);
    const result = await call(router.marketplace.list, { page: 1 }, {
      context: ctx({ userId: orgA.ownerId, activeOrgId: orgA.orgId }),
    });
    expect(result.total).toBe(3);
    expect(result.items.map((l) => l.id)).not.toContain(orgB.listingIds[0]);
  });
});
```

- `call` runs the full middleware chain and schemas in-process; `createRouterClient(router, { context })` gives the client-shaped API for the same purpose.
- Run query-heavy tests against a real Postgres (Testcontainers or a test database), not mocks.
- Add a few HTTP-level tests through the adapter (for example Hono's `app.request('/rpc/...')`) to cover wiring, prefixes and auth headers.

## 13. oRPC vs tRPC, and when to use RPC at all

| | tRPC | oRPC |
|---|---|---|
| Procedure kinds | `.query()`, `.mutation()`, `.subscription()` | one kind; the client chooses query or mutation usage |
| OpenAPI / REST | community adapters | first-class: route metadata, `OpenAPIHandler`, spec generator |
| Contract-first | not built in | `@orpc/contract` with `implement()` |
| Typed errors | `TRPCError` with codes | `ORPCError` plus typed error maps with schema-checked data |
| Rich types over the wire | needs a transformer such as superjson | `Date`, `BigInt`, `Map`, `Set`, `File`/`Blob` built in |
| Schema libraries | Zod and others | any Standard Schema library |
| TanStack Query | official integration | `@orpc/tanstack-query` |
| Maturity | older, larger community | newer, smaller community |

| Situation | Good fit |
|---|---|
| You own client and server in a TypeScript monorepo | oRPC or tRPC |
| Public or partner API, many languages, mobile clients you do not control | REST + OpenAPI (oRPC can generate it) |
| Many clients needing different shapes, aggregation across services | GraphQL |
| Mostly simple CRUD inside one Next.js or TanStack Start app | framework server functions may be enough |

RPC's costs: tighter coupling between client and server releases, weaker HTTP caching (POST by default), less discoverability without OpenAPI, and the type benefits only reach TypeScript clients.

## Common pitfalls
- Authorizing with an `orgId` or `userId` from input instead of the verified context.
- Importing the router as a value in client code instead of `import type`, which can pull server code (and secrets) into the bundle.
- Skipping output schemas and leaking columns such as password hashes or internal notes.
- Throwing plain `Error` for expected failures, which clients only see as generic 500s.
- Treating static types as runtime protection and loosening input schemas.
- A `prefix` that does not match the mount path, so nothing matches and every call 404s.
- Hono middleware that reads the body before the oRPC handler ("body already used").
- Over-broad or too-narrow cache invalidation; use `orpc.<namespace>.key()` deliberately.
- Mixing different versions of `@orpc/*` packages; upgrade them together.
- Breaking a procedure's input or output while old mobile or Rails clients still call it.

## How to talk about this in an interview
- Tell the request story on the Devntech platform: session cookie, adapter builds the initial context, middleware verifies the org membership and injects `org`, the procedure runs an org-scoped Drizzle query, and the typed client feeds TanStack Query in a React component rendered by TanStack Start.
  > **Fill in:** which procedures or middleware you wrote or changed (Rails sync endpoints, marketplace queries, access-control checks) and one bug the types or schemas caught.
  > **Fill in:** how the Rails monolith talked to the new platform (an oRPC client, OpenAPI routes, webhooks, a shared database during transition) and how calls were authenticated.
- Explain trade-offs, not just features: why procedures over REST inside the app, why OpenAPI matters for the Rails side, why not GraphQL.
- Be honest about scope: you contributed to a platform others designed, so say "the platform used oRPC; I worked on X and Y" rather than claiming the architecture.
- Deeper context: [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md).

## Interview questions

**Q1. What does end-to-end type safety mean, and does it replace runtime validation?**
The client's view of the API is derived from the server (router type or shared contract), so mismatched inputs, outputs or renamed fields fail at compile time across the boundary. It does not replace runtime validation: types are erased and any HTTP client can send anything, so the server still validates input (and ideally output) with schemas.
*Follow-up:* Where does the chain start? Often at the database schema (Drizzle infers row types) and ends at component props.

**Q2. What is oRPC?**
A TypeScript RPC framework: you define procedures with input and output schemas and a handler, group them into plain-object routers, and get a fully typed client without code generation. It adds middleware with typed context, typed errors, contract-first support, OpenAPI generation and TanStack Query utilities, and runs on any fetch-compatible runtime.
*Follow-up:* What is a procedure? A function plus its input schema, output schema and middleware chain.

**Q3. How do you define a procedure and a router?**
Start from a builder (`os`, or `os.$context<T>()`), chain `.input(schema)`, optionally `.output(schema)`, and `.handler(async ({ input, context }) => result)`. A router is a nested object of procedures, and its `typeof` is what clients import as a type.
*Follow-up:* How do you share auth across procedures? Build them from a base that has `.use(authMiddleware)` applied.

**Q4. What is the difference between initial and execution context?**
Initial context is supplied by whoever invokes the router (the HTTP adapter, a test, a server-side client) and is typed with `$context<T>()`, so callers must provide it. Execution context is produced during the call by middleware via `next({ context })` and merged in for everything downstream, such as the authenticated user or org.
*Follow-up:* What belongs in initial context? Environment values: the db client, request headers, a resolved session.

**Q5. How would you implement authentication and org scoping with oRPC middleware?**
An `authed` middleware throws `UNAUTHORIZED` when there is no session and narrows `session` to non-null. An `orgProcedure` middleware reads the active org from the session, verifies membership in the database, throws `FORBIDDEN` if missing, and injects `{ org: { id, role } }`. Procedures built on it filter every query by `context.org.id`; role checks stack as further middleware.
*Follow-up:* Why verify membership every time? Memberships change, and a session's org id could be stale or tampered with.

**Q6. How do errors work in oRPC?**
Throw `ORPCError` with a code such as `NOT_FOUND`, which maps to an HTTP status; unknown errors become a generic `INTERNAL_SERVER_ERROR`. Typed error maps (`.errors({ CONFLICT: { data: schema } })`) let handlers `throw errors.CONFLICT({ data })`, and clients use `safe()` plus `isDefinedError()` to get the code and typed data.
*Follow-up:* What happens when output fails its schema? The client gets an internal server error instead of malformed data.

**Q7. How does the client get its types without code generation?**
The client imports only the router's type (`import type { AppRouter }`); `RouterClient<AppRouter>` maps every procedure path to a typed async function using TypeScript inference. The link (for example `RPCLink`) handles the actual HTTP transport. Nothing from the server's runtime code is bundled.
*Follow-up:* What if client and server live in different repos? Share a contract package (contract-first) or publish OpenAPI.

**Q8. How does oRPC compare with tRPC?**
Both give inferred end-to-end types. tRPC splits procedures into queries, mutations and subscriptions and relies on community adapters for OpenAPI; oRPC has one procedure kind (the client chooses query or mutation usage), first-class OpenAPI and contract-first development, typed error maps, and native support for `Date`, `File` and similar types without a transformer. tRPC has the larger community.
*Follow-up:* Why might a team migrating from Rails prefer oRPC? The Rails side needs documented HTTP endpoints, which OpenAPI generation provides.

**Q9. When would you choose RPC over REST or GraphQL?**
RPC (oRPC, tRPC) when one team owns a TypeScript client and server, typically in a monorepo, and wants speed and safety. REST with OpenAPI for public or partner APIs and many languages. GraphQL when many clients need flexible shapes or you aggregate many backends. oRPC narrows the gap by generating OpenAPI for selected procedures.
*Follow-up:* Biggest RPC risk? Coupling client and server releases, especially with mobile apps that lag behind.

**Q10. What is contract-first development, and when would you use it?**
You define the API (procedure names, input and output schemas) with `oc` from `@orpc/contract` in a shared package, and the server implements it via `implement(contract)`, with the compiler enforcing a match. Useful when teams or repos are separate, work proceeds in parallel, or you publish an SDK. Router-first is simpler when one team owns both sides.
*Follow-up:* How does a client type itself from a contract? `ContractRouterClient<typeof contract>`.

**Q11. How do you expose an oRPC API to a non-TypeScript client such as a Rails app?**
Add HTTP route metadata to the procedures it needs, serve them with `OpenAPIHandler`, and generate an OpenAPI document with `OpenAPIGenerator` and a schema converter. The Rails team can generate or hand-write a client from the spec; authenticate it as a service (API key or HMAC) and keep those endpoints backward compatible.
*Follow-up:* Why not let Rails call the RPC protocol directly? It is an internal wire format; OpenAPI routes are stable and documented.

**Q12. How do you serve oRPC inside Hono or TanStack Start?**
Create `new RPCHandler(router)` from the fetch adapter. In Hono, add middleware on `/rpc/*` that calls `handler.handle(c.req.raw, { prefix: '/rpc', context })` and returns the response if matched. In TanStack Start, a catch-all server route (`/api/rpc/$`) calls `handler.handle(request, { prefix: '/api/rpc', context })` and returns the response or a 404.
*Follow-up:* What must match? The prefix and the mount path; otherwise no procedure matches.

**Q13. How do you use oRPC with TanStack Query, and how do you invalidate after mutations?**
Wrap the client with `createTanstackQueryUtils(client)`; use `orpc.x.list.queryOptions({ input })` with `useQuery` and `orpc.x.create.mutationOptions()` with `useMutation`. In `onSuccess`, `invalidateQueries({ queryKey: orpc.x.key() })` refetches every query under that namespace; `queryKey({ input })` targets one entry for `setQueryData`.
*Follow-up:* Why not write keys by hand? Generated keys stay in sync with procedure paths and inputs.

**Q14. How do you test procedures?**
Call them in-process with `call(procedure, input, { context })` (or `createRouterClient`), passing fake sessions to test auth and org isolation; run them against a real test database for query behavior; add a few HTTP tests through the adapter for wiring. Assert error codes with `rejects.toMatchObject({ code: 'FORBIDDEN' })`.
*Follow-up:* The most valuable multi-tenant test? Two orgs, and every procedure proving org A cannot read or change org B's data.

**Q15. What are the downsides of RPC-style APIs?**
Clients and servers are coupled, so breaking changes need coordinated releases; HTTP caching is weaker because calls are usually POSTs; discoverability needs extra docs; and the type benefits stop at TypeScript. Mitigations: additive changes, OpenAPI for outside consumers, and query-level caching on the client.
*Follow-up:* Does oRPC support GET for cacheable reads? The link can be configured to use GET for chosen procedures; check the docs.

**Q16. How do you evolve a procedure without breaking deployed clients?**
Make additive changes: new optional input fields with defaults, new output fields. Never remove or rename fields that old clients use; add a new procedure (`listV2`) for breaking changes, keep the old one until usage metrics drop to zero, then remove it. Mobile apps and the Rails sync code can lag behind a web deploy, so assume old clients exist.
*Follow-up:* How do you know old clients are gone? Log procedure usage by client version and watch it.

## Further reading
- [oRPC: Getting started](https://orpc.unnoq.com/docs/getting-started)
- [oRPC: Middleware](https://orpc.unnoq.com/docs/middleware)
- [oRPC: Error handling](https://orpc.unnoq.com/docs/error-handling)
- [oRPC: TanStack Query integration](https://orpc.unnoq.com/docs/integrations/tanstack-query)
- [oRPC: OpenAPI specification](https://orpc.unnoq.com/docs/openapi/openapi-specification)
