# TanStack Start

> **Tier:** T1 (know cold) · **Related:** [TanStack Router](04-tanstack-router.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [Drizzle ORM](../12-databases/13-drizzle-orm.md), [TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [Next.js fundamentals](01-nextjs-fundamentals.md), [React 19 and Server Components](../06-react/09-react-19-and-server-components.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md)
> **On your resume:** the multi-tenant TypeScript platform at Devntech (oRPC, TanStack Start, Drizzle), where you contributed Rails sync APIs, the marketplace page migration and org-scoped access control; TanStack Start is in your skills list.

> **Version note:** TanStack Start reached release-candidate status in September 2025 and was announced as stable (1.0) in October 2026. Along the way it moved from Vinxi to a Vite plugin (Rsbuild is also supported) and renamed several APIs. Code below follows the shape of the current docs; pin exact versions in `package.json` and check the docs and migration guides before copying.

## Quick revision
- TanStack Start = TanStack Router + a server: full-document SSR, streaming, server functions, server routes and middleware, built with Vite (or Rsbuild) and deployable to many hosts.
- Mental model: a client-first, fully typed app that also renders on the server. It is not RSC-first like Next.js (React Server Components are experimental in Start).
- Key files: `vite.config.ts` (`tanstackStart()` before the React plugin), `src/router.tsx` (exports `getRouter()`), `src/routes/__root.tsx` (the HTML document with `<HeadContent />` and `<Scripts />`), route files, and the generated `routeTree.gen.ts`.
- Server functions: `createServerFn({ method: 'GET' | 'POST' })`, `.validator(schema)` (some release candidates called it `.inputValidator`), `.handler(async ({ data, context }) => ...)`; call them as `fn({ data })` from loaders, components (`useServerFn`) or other server code.
- Server functions only run on the server; the client bundle gets an RPC stub, so they can use the database and secrets. Their input is untrusted: validate it.
- Request helpers live in `@tanstack/react-start/server`: `getRequest()`, `getRequestHeader()`, `getCookie()`, `setCookie()`, `setResponseHeader()`, `setResponseStatus()` (names changed between versions).
- Throw `redirect()` or `notFound()` from server functions and loaders.
- **Route loaders are isomorphic:** they run on the server for the first request and in the browser on client navigation. Server-only work belongs in server functions.
- Server routes: `createFileRoute('/api/x')({ server: { handlers: { GET, POST } } })` for webhooks, health checks, REST endpoints or a mounted oRPC handler.
- Middleware: request middleware (`createMiddleware()`, every request) and function middleware (`createMiddleware({ type: 'function' })`, attached with `.middleware([...])`); `next({ context })` passes typed context; global middleware is registered in `src/start.ts` with `createStart`.
- `beforeLoad` guards improve the experience; authorization must also happen in server functions, server routes and API procedures.
- Data: loaders prefetch with TanStack Query and components use `useSuspenseQuery`; `@tanstack/react-router-ssr-query` dehydrates, hydrates and streams the cache; create a new `QueryClient` per request.
- SEO: route `head()` returns title, meta tags and links (canonical); SSR delivers full HTML; selective SSR can disable SSR per route.
- Env vars: `process.env` in server code (read it per request, not at module scope); `import.meta.env.VITE_*` in client code, which is public.
- Deployment: Node servers, Docker, Vercel, Netlify, Cloudflare and more, configured through Vite and Nitro integrations; follow the hosting guide.
- vs Next.js: Start is client-first with explicit loaders, server functions and typed routing on Vite; Next.js is server-first with Server Components, built-in caching layers, Server Actions and a larger ecosystem.
- Multi-tenant pattern: session cookie, then middleware resolves the user and active org, then server functions and oRPC procedures filter every Drizzle query by `org_id`, then typed results flow into TanStack Query and SSR.

## 1. What TanStack Start is

[TanStack Router](04-tanstack-router.md) gives you typed routes, search params, loaders and preloading in the browser. Start adds the server half:
- **SSR and streaming:** the first request renders full HTML on the server (good for SEO and first paint), streams Suspense boundaries, then hydrates into a client-side app.
- **Server functions:** typed RPC between client and server without writing an API layer by hand.
- **Server routes:** plain HTTP endpoints in the same project.
- **Middleware:** request and function middleware with typed context that can cross from client to server.
- **Bundling and deployment:** Vite (or Rsbuild) builds client and server bundles that deploy to many hosts.

The philosophy differs from Next.js: Start assumes most of your UI is an interactive client application and makes server rendering and server calls explicit, while Next.js defaults to Server Components and implicit framework behavior.

## 2. Project structure

```text
vite.config.ts            tanstackStart() plugin, then the React plugin
src/
  router.tsx              getRouter(): creates the router (and QueryClient) per request
  routeTree.gen.ts        generated by the router plugin; never edited by hand
  start.ts                optional: global request and function middleware (createStart)
  routes/
    __root.tsx            HTML document: <HeadContent />, <Outlet />, <Scripts />
    index.tsx             /
    marketplace/index.tsx /marketplace
    marketplace/$slug.tsx /marketplace/:slug
    _authed.tsx           pathless layout with an auth guard
    api/rpc.$.ts          server route mounting the oRPC handler
  server/                 server functions, middleware, Drizzle client and repositories
```

```ts
// vite.config.ts
import { tanstackStart } from '@tanstack/react-start/plugin/vite';
import viteReact from '@vitejs/plugin-react';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [tanstackStart(), viteReact()],   // React's plugin must come after Start's
});
```

```tsx
// src/router.tsx
import { QueryClient } from '@tanstack/react-query';
import { createRouter } from '@tanstack/react-router';
import { setupRouterSsrQueryIntegration } from '@tanstack/react-router-ssr-query';
import { routeTree } from './routeTree.gen';

export function getRouter() {
  const queryClient = new QueryClient();                 // per request on the server: no data shared between users
  const router = createRouter({ routeTree, context: { queryClient }, defaultPreload: 'intent' });
  setupRouterSsrQueryIntegration({ router, queryClient }); // dehydrate on the server, hydrate in the browser
  return router;
}
```

```tsx
// src/routes/__root.tsx
import type { QueryClient } from '@tanstack/react-query';
import { createRootRouteWithContext, HeadContent, Outlet, Scripts } from '@tanstack/react-router';
import type { ReactNode } from 'react';

export const Route = createRootRouteWithContext<{ queryClient: QueryClient }>()({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'Marketplace' },
    ],
  }),
  component: () => (
    <RootDocument>
      <Outlet />
    </RootDocument>
  ),
});

function RootDocument({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <head>
        <HeadContent />          {/* tags from head() of every matched route */}
      </head>
      <body>
        {children}
        <Scripts />              {/* client entry and hydration data */}
      </body>
    </html>
  );
}
```

## 3. Server functions

```ts
// src/server/listings.functions.ts
import { notFound } from '@tanstack/react-router';
import { createServerFn } from '@tanstack/react-start';
import { z } from 'zod';
import { orgMiddleware } from './middleware';
import { findPublishedListingBySlug, toggleSaved } from './listings.repository'; // Drizzle code, server only

export const getListingBySlug = createServerFn({ method: 'GET' })
  .validator(z.object({ slug: z.string().min(1) }))       // .inputValidator() in some RC versions
  .handler(async ({ data }) => {
    const listing = await findPublishedListingBySlug(data.slug);
    if (!listing) throw notFound();
    return listing;                                        // returned values must be serializable
  });

export const toggleSavedListing = createServerFn({ method: 'POST' })
  .middleware([orgMiddleware])                             // session and org in context (section 5)
  .validator(z.object({ listingId: z.string().uuid() }))
  .handler(async ({ data, context }) => toggleSaved(context.session.userId, context.org.id, data.listingId));
```

```tsx
// Calling them
export const Route = createFileRoute('/marketplace/$slug')({
  loader: ({ params }) => getListingBySlug({ data: { slug: params.slug } }), // direct call during SSR, RPC in the browser
  component: ListingPage,
});

function SaveButton({ listingId }: { listingId: string }) {
  const toggleSaved = useServerFn(toggleSavedListing);    // also handles redirect() and notFound() thrown on the server
  return <button onClick={() => toggleSaved({ data: { listingId } })}>Save</button>;
}
```

- **Under the hood:** the build replaces the handler in the client bundle with a stub. Calling it sends an HTTP request to an internal endpoint (GET puts the data in the URL, POST in the body); the server runs middleware, validation and the handler, then serializes the result back.
- **GET vs POST:** GET for reads (cache-friendly, idempotent), POST for writes.
- **Validation is required, not optional:** TypeScript types disappear at runtime and anyone can call the endpoint with any payload.
- **Request access:** read cookies and headers only on the server, inside handlers or middleware:

```ts
import { createServerFn } from '@tanstack/react-start';
import { getCookie } from '@tanstack/react-start/server';

export const getCurrentUser = createServerFn({ method: 'GET' }).handler(async () => {
  const token = getCookie('session');
  return token ? verifySessionToken(token) : null;         // { userId, activeOrgId, role } or null
});
```

Related helpers: `createServerOnlyFn` and `createClientOnlyFn` throw if called in the wrong environment; `createIsomorphicFn().server(...).client(...)` picks an implementation per environment.

## 4. Server routes

```ts
// src/routes/api/health.ts
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/api/health')({
  server: {
    handlers: {
      GET: async () => Response.json({ ok: true }),
    },
  },
});
```

```ts
// src/routes/api/webhooks/payments.ts
export const Route = createFileRoute('/api/webhooks/payments')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const raw = await request.text();                 // exact bytes for the HMAC check
        if (!isValidSignature(Buffer.from(raw), request.headers.get('x-signature-256') ?? undefined, process.env.WEBHOOK_SECRET ?? '')) {
          return new Response('Invalid signature', { status: 401 });
        }
        await recordPaymentEvent(JSON.parse(raw));
        return Response.json({ received: true });
      },
    },
  },
});
```

Handlers receive `{ request, params, context }` and return a `Response`, so anything fetch-based mounts here, including an oRPC `RPCHandler` on a catch-all route such as `api/rpc.$.ts` (code in [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md)). Earlier versions used a separate `createServerFileRoute` API; check which shape your version uses.

## 5. Middleware

```ts
// src/server/middleware.ts
import { createMiddleware } from '@tanstack/react-start';
import { getCookie } from '@tanstack/react-start/server';

export const authMiddleware = createMiddleware({ type: 'function' }).server(async ({ next }) => {
  const session = await readSession(getCookie('session'));   // verify and decode the session cookie
  if (!session) throw new Error('UNAUTHENTICATED');           // or throw redirect({ to: '/login' })
  return next({ context: { session } });                      // typed for everything downstream
});

export const orgMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])                               // dependencies run first
  .server(async ({ next, context }) => {
    const membership = await findMembership(context.session.userId, context.session.activeOrgId);
    if (!membership) throw new Error('FORBIDDEN');            // stale or forged org id
    return next({ context: { org: { id: membership.orgId, role: membership.role } } });
  });
```

```ts
// src/start.ts: global middleware (current docs shape; check your version)
import { createMiddleware, createStart } from '@tanstack/react-start';

const requestLogger = createMiddleware().server(async ({ next, request }) => {
  const started = Date.now();
  const result = await next();
  console.log(request.method, new URL(request.url).pathname, `${Date.now() - started} ms`);
  return result;
});

export const startInstance = createStart(() => ({ requestMiddleware: [requestLogger] }));
```

- **Request middleware** runs for every request: SSR, server routes and server functions. Use it for logging, request ids and security headers.
- **Function middleware** wraps server functions; it can also validate input and run client-side logic (`.client()`), for example attaching headers before the RPC is sent.
- Global `functionMiddleware` applies to every server function. Recent docs note that Start installs CSRF protection for server functions by default only when `src/start.ts` does not exist; if you create one, add the CSRF middleware yourself.

## 6. SSR, hydration and streaming

```text
1. GET /marketplace/room-in-brussels reaches the Start server
2. request middleware runs; getRouter() creates a router and QueryClient for this request
3. routes match; beforeLoad runs parent to child; loaders run, on the server this time
4. React renders HTML as a stream; slow Suspense boundaries arrive later in the same response
5. loader data and the dehydrated query cache are serialized into the page
6. the browser loads the JavaScript and hydrates with the same data (no refetch)
7. later navigations: loaders run in the browser and server functions become RPC calls
```

Because of step 7, a loader must never touch the database or secrets directly:

```ts
// Bug: on client navigation this loader runs in the browser
loader: async () => db.select().from(listings),
// Fix: delegate server-only work to a server function
loader: () => listPublishedListings({ data: {} }),
```

- **Streaming:** queries read with `useSuspenseQuery` run during SSR when needed and stream to the client as they resolve; a loader can also start a query without awaiting it so it streams instead of blocking.
- **Selective SSR:** routes can opt out of server rendering (`ssr: false`) or render data on the server but the component in the browser (`ssr: 'data-only'`); check the docs for your version.
- **Hydration mismatches** follow React's rules: no `Date.now()`, random values or `window` checks in the first render ([hydration](../06-react/09-react-19-and-server-components.md)); `ClientOnly` renders a fallback on the server and its children after hydration.

## 7. Loading data with TanStack Query

```tsx
// src/routes/marketplace/index.tsx
import { queryOptions, useSuspenseQuery } from '@tanstack/react-query';
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';
import { listPublishedListings } from '@/server/listings.functions';

const SearchSchema = z.object({
  page: z.number().int().min(1).default(1),
  category: z.string().optional(),
});
type MarketplaceSearch = z.infer<typeof SearchSchema>;

const listingsQuery = (search: MarketplaceSearch) =>
  queryOptions({
    queryKey: ['marketplace', 'list', search],
    queryFn: () => listPublishedListings({ data: search }),   // a server function (or an oRPC call)
    staleTime: 30_000,
  });

export const Route = createFileRoute('/marketplace/')({
  validateSearch: SearchSchema,
  loaderDeps: ({ search }) => search,
  loader: ({ context, deps }) => context.queryClient.ensureQueryData(listingsQuery(deps)),
  component: MarketplacePage,
});

function MarketplacePage() {
  const { data } = useSuspenseQuery(listingsQuery(Route.useSearch())); // SSR-rendered, hydrated, then reactive
  return <ListingsGrid items={data.items} total={data.total} />;
}
```

- The loader fills the cache during SSR; the integration dehydrates it into the HTML and hydrates the browser's `QueryClient`, so the client does not refetch on load.
- After a mutation, `queryClient.invalidateQueries({ queryKey: ['marketplace'] })` refetches affected queries.
- With oRPC, swap the options for `orpc.marketplace.list.queryOptions({ input: deps })`; during SSR an isomorphic client can call procedures in-process ([oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md)).
- Recent TanStack Query releases deprecate `ensureQueryData` in favor of `queryClient.query(...)`; check your version ([TanStack Router](04-tanstack-router.md)).

## 8. Auth patterns

```tsx
// src/routes/_authed.tsx: guards every nested route
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router';
import { getCurrentUser } from '@/server/auth.functions';

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUser();          // a server function: the session check runs on the server
    if (!user) throw redirect({ to: '/login', search: { redirect: location.href } });
    return { user };                              // available in child routes' context
  },
  component: () => <Outlet />,
});
```

- Sessions live in HttpOnly cookies, read with `getCookie` on the server (the docs show iron-session for encrypted cookie sessions); auth libraries such as Better Auth, Clerk, WorkOS or Auth.js integrate with Start.
- The guard decides what the user sees. It is not security: every server function and server route that returns or changes private data must check the session itself, typically through `authMiddleware` and `orgMiddleware`.
- Login and logout are POST server functions that set or clear the cookie (`setCookie`), then `throw redirect(...)`.

## 9. Environment variables

- Server code reads any variable from `process.env`. Read it inside handlers (per request), not at module scope in shared files, where it can be inlined into the client bundle or be undefined on edge runtimes.
- Client code reads only prefixed variables from `import.meta.env` (`VITE_` with Vite, `PUBLIC_` with Rsbuild); unprefixed ones are `undefined` in the browser.
- Anything with the public prefix ships in the JavaScript bundle: never put secrets there.
- Pass runtime values the client needs through loaders, and validate server env with Zod at startup.

## 10. Deployment

Start builds a client bundle and a server bundle. Hosting moved from Start's own adapters to Vite and Nitro integrations, so the same app can target a Node server (often in Docker), Vercel, Netlify, Cloudflare Workers and other platforms; follow the hosting guide for your provider and check which runtime features (Node APIs, TCP database drivers) it supports. For a multi-tenant platform with Postgres and Drizzle, a Node server in a container keeps database pooling simple.

## 11. TanStack Start vs Next.js

| | TanStack Start | Next.js (App Router) |
|---|---|---|
| Foundation | TanStack Router plus Vite (or Rsbuild) | React plus the Next.js compiler (Turbopack by default in 16) |
| Mental model | client-first app with SSR; explicit loaders and server functions | server-first: Server Components by default |
| Server Components | experimental | core model |
| Routing types | fully inferred: paths, params, search, loader data | file-based; typed routes in recent versions |
| Search params | validated and typed | strings via `searchParams` |
| Data loading | isomorphic loaders plus TanStack Query | async Server Components plus framework caching |
| Mutations | server functions, oRPC, TanStack Query mutations | Server Actions |
| Caching | explicit: router cache and TanStack Query | framework layers (changed in 15; Cache Components in 16) |
| API endpoints | server routes | Route Handlers |
| Middleware | request and function middleware with typed context | middleware, renamed proxy in 16 |
| Maturity | younger; APIs changed through beta and RC | mature, very large ecosystem |
| Good fit | interactive SaaS and dashboards that need SSR for some pages | content-heavy and SEO-heavy sites, teams invested in RSC and Vercel |

## 12. Case study: a multi-tenant platform with Start, oRPC and Drizzle

This is a typical setup to adapt, not a description of any specific system:

```text
Browser
  -> TanStack Start: SSR, routes, loaders, TanStack Query
       -> server functions (page-specific work)             -> Drizzle -> PostgreSQL
       -> /api/rpc/* server route -> oRPC RPCHandler
            -> middleware: session -> membership -> context.org
            -> procedures (shared API surface)               -> Drizzle, every query filtered by org_id
Rails monolith <-> sync APIs (documented HTTP endpoints, retries, idempotency) while the migration runs
```

**Org context from the session.** The session cookie identifies the user and their active org. Middleware (Start function middleware or oRPC middleware) loads the membership on every request, rejects stale or forged org ids, and puts `{ org: { id, role } }` in context. Switching orgs is a POST that updates the session after checking membership. The org never comes from a URL or request body alone.

**Org-scoped queries.** Every tenant table has `org_id`; composite indexes and unique constraints start with it; repositories take the org from context and add `eq(table.orgId, org.id)` to every query; tests create two orgs and prove that each procedure cannot read or change the other's data. Row-level security can back this up ([Drizzle](../12-databases/13-drizzle-orm.md), [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)).

| | Server functions | oRPC procedures |
|---|---|---|
| Callers | this Start app | the Start app, other services, Rails (through OpenAPI), mobile |
| Contract | inferred inside one app | a router type or shared contract, optional OpenAPI |
| Good for | page-specific loaders and mutations, session handling | the platform's shared, documented API |

**Migrating a marketplace page from Rails without losing SEO:**
- Inventory existing URLs from the sitemap, analytics and search console, and keep paths identical where possible (`/marketplace/:slug` becomes `src/routes/marketplace/$slug.tsx`).
- Server-render every indexable page so crawlers get full HTML with content and links, and return real 404s for missing listings (`notFound()`), not soft 404s.
- Reach metadata parity through `head()`: title, description, canonical URL, Open Graph tags, structured data, and `hreflang` if localized.
- Use permanent (301) redirects for any changed URL, ideally at the reverse proxy or CDN, from a maintained redirect map.
- Regenerate the sitemap and robots rules from the new app (a server route can output XML).
- Match or beat the old pages' Core Web Vitals (LCP, CLS, INP).
- Roll out with the strangler fig pattern: a reverse proxy sends `/marketplace/*` to Start and everything else to Rails; compare rendered HTML and meta tags between versions; watch search console errors, 404s and traffic; keep a fast rollback switch. Sync APIs keep Rails and the new database consistent until Rails is retired ([platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md)).

```tsx
// src/routes/marketplace/$slug.tsx: metadata from loader data
export const Route = createFileRoute('/marketplace/$slug')({
  loader: ({ params }) => getListingBySlug({ data: { slug: params.slug } }),
  head: ({ loaderData }) => ({
    meta: [
      { title: `${loaderData?.title ?? 'Listing'} | Marketplace` },
      { name: 'description', content: loaderData?.summary ?? '' },
      { property: 'og:title', content: loaderData?.title ?? '' },
    ],
    links: [{ rel: 'canonical', href: `https://www.example.com/marketplace/${loaderData?.slug ?? ''}` }],
  }),
  component: ListingPage,
});
```

## Common pitfalls
- Querying the database or reading secrets in a route loader, which also runs in the browser.
- Reading `process.env` at module scope in shared code, or prefixing secrets with `VITE_`.
- Treating `beforeLoad` as security and leaving server functions or server routes unprotected.
- Skipping `.validator()` on server functions because "the client is typed".
- Sharing one `QueryClient` across server requests, leaking one user's data into another's page.
- Mixing API shapes from different versions (`inputValidator` vs `validator`, `createServerFileRoute` vs `server.handlers`, Vinxi-era config).
- Returning non-serializable values (class instances, database driver objects) from server functions.
- Hydration mismatches from time, random values or browser-only checks in the first render.
- Defining `src/start.ts` without re-adding CSRF protection for server functions, on versions where that applies.
- Migrating SEO pages to client-only rendering, or changing URLs without 301 redirects and canonical tags.

## How to talk about this in an interview
- Your resume says you contributed to migrating Colochousing onto a new multi-tenant TypeScript platform built with oRPC, TanStack Start and Drizzle, covering Rails sync APIs, the marketplace page migration and org-scoped access control. Tell it as a request story: SSR route, loader, server function or oRPC procedure, middleware with org context, scoped Drizzle query, hydrated TanStack Query cache.
  > **Fill in:** what you built on the marketplace page (routes, filters in search params, listing detail, metadata) and how SEO was preserved (same URLs, SSR, redirects, head tags).
  > **Fill in:** which part of org-scoped access control you implemented (middleware, membership checks, scoped repositories, tests with two orgs).
  > **Fill in:** how the Rails sync APIs fit in: which data synced, in which direction, and how failures and retries were handled.
- Be honest about scope: "I contributed to a platform the team designed; my parts were X and Y." Then show you understand the whole flow.
- Have a crisp answer to "Why TanStack Start and not Next.js?": typed routing and search params end to end, explicit data loading with TanStack Query, Vite tooling, and a client-first model that suits an interactive multi-tenant app; while acknowledging Next.js's maturity and RSC.
- Deep dives: [platform migration project](../17-projects-deep-dive/04-multi-tenant-platform-migration.md), [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md).

## Interview questions

**Q1. What is TanStack Start, and how does it relate to TanStack Router?**
A full-stack React framework built on TanStack Router: Router provides typed routing, search params, loaders and preloading; Start adds SSR and streaming, server functions, server routes, middleware and Vite-based client and server builds. A Start app is a Router app that also runs on a server.
*Follow-up:* What did Start move away from during its RC phase? Vinxi, in favor of a Vite plugin.

**Q2. What are server functions, and how do they work?**
Functions created with `createServerFn` that always execute on the server. The build swaps their implementation in the client bundle for an RPC stub, so calling them from the browser sends a GET or POST to an internal endpoint, runs middleware, validation and the handler on the server, and returns the serialized result with full type inference.
*Follow-up:* How do you pass input? `fn({ data: {...} })`, validated by `.validator(schema)`.

**Q3. Why validate server function input when the client is typed?**
Types are erased at build time, and the RPC endpoint is reachable by anyone with any payload. A Zod schema in `.validator()` enforces the contract at runtime, rejects bad input and gives the handler parsed, typed `data`.
*Follow-up:* What about authorization? Also inside: middleware or the handler checks the session and org on every call.

**Q4. Where do loaders run in TanStack Start, and why can't a loader query the database directly?**
Loaders are isomorphic: on the server for the initial SSR request and in the browser for client-side navigations. Database access in a loader would ship server code to the client or fail there; call a server function from the loader instead.
*Follow-up:* And `beforeLoad`? Also isomorphic, so it should call server functions for anything secret.

**Q5. How do you read cookies and headers in a server function?**
With helpers from `@tanstack/react-start/server`, such as `getCookie('session')`, `getRequestHeader('user-agent')` and `getRequest()`, called inside the handler or middleware where a request exists. Responses can be adjusted with `setCookie`, `setResponseHeader` and `setResponseStatus`.
*Follow-up:* Why do names vary between tutorials? The helpers were renamed during beta and RC; follow the docs for your version.

**Q6. Request middleware vs function middleware: how would you inject org context?**
Request middleware runs for every request (SSR, server routes, server functions), good for logging and headers. Function middleware wraps server functions and passes typed context with `next({ context })`. An `authMiddleware` resolves the session; an `orgMiddleware` depending on it verifies membership of the active org and adds `{ org }`; server functions that need it list `.middleware([orgMiddleware])`.
*Follow-up:* Where are global middlewares registered? In `src/start.ts` via `createStart`.

**Q7. How do you protect routes and server functions?**
For the experience, a pathless layout's `beforeLoad` calls a `getCurrentUser` server function and redirects to login. For security, every server function and server route returning or changing private data runs auth and org middleware, and queries are scoped to the org. Route guards alone are not security.
*Follow-up:* Why? Server functions and routes can be called directly, bypassing the UI.

**Q8. How do SSR and hydration work in TanStack Start?**
A request creates a router per request, runs `beforeLoad` and loaders on the server, streams the rendered HTML, and serializes loader data and the dehydrated query cache into the page. The browser hydrates with that same data, then behaves as a client-side app whose later navigations run loaders in the browser.
*Follow-up:* What causes hydration mismatches? Different output between server and first client render: dates, random values, browser checks.

**Q9. How do you use TanStack Query with SSR in Start?**
Create the `QueryClient` inside `getRouter()` (per request), put it in router context, and call `setupRouterSsrQueryIntegration`. Loaders call `ensureQueryData` (or `queryClient.query` in recent versions) with shared `queryOptions`; components read with `useSuspenseQuery`, so data renders on the server, hydrates without refetching and stays reactive.
*Follow-up:* Why not a module-level `QueryClient` on the server? Requests from different users would share cached data.

**Q10. Server functions vs oRPC procedures: when do you use each?**
Server functions for page-specific work inside the Start app (loaders, form mutations, session handling) where inferred types are enough. oRPC procedures for the platform's shared API surface that other callers use, such as other services, a mobile app or the Rails monolith through OpenAPI, with typed errors and a contract.
*Follow-up:* Can both coexist? Yes: the oRPC handler mounts in a server route next to server functions.

**Q11. When do you need server routes?**
For plain HTTP endpoints: webhooks (raw body and signature checks), health checks, REST endpoints for external clients, file downloads, or mounting another fetch handler such as oRPC. Handlers receive `{ request, params, context }` and return a `Response`.
*Follow-up:* Can one file have a server handler and a page component? Yes, both on the same route.

**Q12. TanStack Start vs Next.js: how do you choose?**
Start: client-first with SSR, end-to-end typed routing and search params, explicit loaders and TanStack Query, server functions, Vite; younger and its APIs moved during RC. Next.js: server-first with Server Components, built-in caching and Server Actions, a mature ecosystem and Vercel integration. Interactive SaaS dashboards fit Start well; content-heavy SEO sites often fit Next.js.
*Follow-up:* Does Start support React Server Components? Experimentally; its core model does not depend on them.

**Q13. How do environment variables work in Start?**
Server code reads `process.env` (inside handlers, not at module scope in shared code). Client code sees only public-prefixed variables through `import.meta.env` (`VITE_` with Vite), and those values are in the bundle. Secrets stay unprefixed and server-only; runtime values reach the client through loaders.
*Follow-up:* What happens if you prefix a secret with `VITE_`? It is embedded in the public JavaScript bundle.

**Q14. How would you migrate a Rails page to Start while keeping SEO?**
Keep URLs identical (or 301-redirect changed ones), server-render the page, match metadata through `head()` (title, description, canonical, Open Graph, structured data), return real 404s, regenerate sitemaps, and check Core Web Vitals. Roll out behind a reverse proxy with the strangler fig pattern, compare old and new HTML, and monitor search console and traffic with a rollback switch.
*Follow-up:* Why not render the new page client-side only? Crawlers and link previews need the content in the initial HTML.

**Q15. How do you keep tenant data isolated in a Start, oRPC and Drizzle stack?**
Resolve the user from the session and the active org from a verified membership in middleware, put the org in context, and scope every Drizzle query with `org_id` through repositories; never take the org from request input alone. Index and constrain by `org_id`, optionally add row-level security, and test with two orgs.
*Follow-up:* What about caches? Include the org id in every cache and query key.

**Q16. What risks come with adopting a framework that was in release candidate, and how do you manage them?**
APIs changing between releases (validator names, server route shapes, build tooling), thinner documentation and fewer community answers. Mitigate by pinning versions, upgrading deliberately with release notes and migration guides, keeping framework-specific code thin (business logic in plain modules and oRPC procedures), and covering critical flows with tests.
*Follow-up:* Is the risk gone now? Start was declared stable in October 2026; what remains is normal upgrade effort, which a thin framework layer keeps small.

## Further reading
- [TanStack Start: Overview](https://tanstack.com/start/latest/docs/framework/react/overview)
- [TanStack Start: Server functions](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)
- [TanStack Start: Middleware](https://tanstack.com/start/latest/docs/framework/react/guide/middleware)
- [TanStack Start: Execution model](https://tanstack.com/start/latest/docs/framework/react/guide/execution-model)
- [TanStack Start: Server routes](https://tanstack.com/start/latest/docs/framework/react/guide/server-routes)
