# Next.js and TanStack: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. Version-specific answers name the version; check the docs for anything newer.

## Next.js fundamentals (see [01-nextjs-fundamentals.md](01-nextjs-fundamentals.md))

**Q1. What is Next.js in one sentence?**
A React framework that adds file-based routing, server rendering (SSR, SSG, ISR, streaming, Server Components), data fetching and caching, backend endpoints and asset optimization.

**Q2. CSR vs SSR?**
CSR renders in the browser after JavaScript loads (empty HTML first); SSR renders HTML on the server per request, giving faster first paint and crawlable content.

**Q3. SSG vs ISR?**
SSG builds HTML once at build time; ISR serves static HTML and regenerates it in the background after a revalidate period or on demand.

**Q4. What is streaming SSR?**
The server sends the page shell immediately and streams each Suspense boundary's HTML as its data resolves, instead of waiting for all data.

**Q5. App Router vs Pages Router in one line each?**
App Router: `app/`, Server Components by default, nested layouts, streaming, Server Actions. Pages Router: `pages/`, `getServerSideProps`/`getStaticProps`, Client Components only, `pages/api`.

**Q6. When was the App Router stable, and Server Actions?**
App Router in Next.js 13.4; Server Actions in Next.js 14.

**Q7. What replaces `getServerSideProps` in the App Router?**
An async Server Component that awaits its data, rendered dynamically because it uses request data or uncached fetches.

**Q8. What replaces `getStaticPaths`?**
`generateStaticParams`, which returns the params to prebuild.

**Q9. What does `layout.tsx` do?**
Wraps a segment's pages with shared UI that persists across navigations, keeping state; the root layout renders `<html>` and `<body>`. A `template.tsx` wraps the same way but re-mounts on every navigation.

**Q10. What do `loading.tsx` and `error.tsx` provide?**
`loading.tsx` is a Suspense fallback for the segment; `error.tsx` is a Client Component error boundary with `error` and `reset` props.

**Q11. What is a route group?**
A folder in parentheses, like `(marketing)`, that organizes routes or applies a layout without adding a URL segment.

**Q12. `[slug]` vs `[...slug]` vs `[[...slug]]`?**
One segment; one or more segments as an array; zero or more segments (also matches the parent path).

**Q13. What are parallel and intercepting routes for?**
Parallel routes (`@slot`) render several pages in one layout; intercepting routes (`(.)photos/[id]`) show a route in context, such as a modal, during client navigation.

**Q14. Default component type in `app/`?**
Server Component. Add `'use client'` at the top of a module to make it and its imports client code.

**Q15. Three things Server Components cannot do?**
Use state or effects, attach event handlers, use browser APIs (also context).

**Q16. What props can pass from a Server to a Client Component?**
Serializable ones: primitives, plain objects, arrays, dates, maps, sets, promises, JSX and Server Actions; not plain functions or class instances.

**Q17. How can a Client Component render a Server Component?**
By receiving it as `children` or another prop from a Server Component parent, not by importing it.

**Q18. Which `useRouter` do you use in the App Router?**
The one from `next/navigation` (with `usePathname`, `useSearchParams`, `useParams`); `next/router` belongs to the Pages Router.

**Q19. Does `<Link>` prefetch?**
In production, yes: routes are prefetched as links enter the viewport (dynamic routes up to the nearest `loading.tsx`); not in development.

**Q20. How do you set per-page SEO metadata?**
Export a `metadata` object or an async `generateMetadata` function from the page or layout.

**Q21. Why `next/image`?**
On-demand resizing, modern formats, lazy loading and reserved dimensions that prevent layout shift.

**Q22. What changed for `priority` on `next/image` in Next.js 16?**
It was deprecated in favor of `preload`; the docs suggest `loading="eager"` or `fetchPriority="high"` for most hero images.

**Q23. Why `next/font`?**
It self-hosts fonts at build time with size-adjusted fallbacks: no layout shift and no browser request to Google.

**Q24. What is special about `NEXT_PUBLIC_` variables?**
They are inlined into the client bundle at build time, so they are public and do not change at runtime without a rebuild.

**Q25. What became async in Next.js 15?**
`params`, `searchParams`, `cookies()`, `headers()` and `draftMode()`: you `await` them.

**Q26. Name four Next.js 16 changes.**
Turbopack by default, Cache Components with `"use cache"`, `middleware.ts` renamed to `proxy.ts` (Node.js runtime), and `next lint` removed (plus Node.js 20.9+).

**Q27. What does `output: 'standalone'` do?**
Produces a minimal server (`server.js`) with only the needed dependencies, ideal for Docker; you copy `.next/static` and `public` alongside it.

## Data fetching, caching and Server Actions (see [02-nextjs-data-fetching-and-caching.md](02-nextjs-data-fetching-and-caching.md))

**Q28. How do you fetch data in the App Router?**
In async Server Components, with `fetch` or a server-only data access layer that checks authorization and returns DTOs.

**Q29. What is request memoization?**
Identical GET `fetch` calls in one render run once; React `cache()` gives the same deduplication for ORM calls.

**Q30. Name the four caching layers of the previous model.**
Request memoization, Data Cache, Full Route Cache (server) and Router Cache (browser).

**Q31. Are `fetch` calls cached by default in Next.js 15?**
No. Opt in with `cache: 'force-cache'` or `next: { revalidate }`, but static routes still run fetches at build time and bake results into the HTML. GET Route Handlers and client page navigations also stopped being cached by default.

**Q32. What makes a route dynamic?**
`cookies()`, `headers()`, `searchParams`, `connection()`, uncached data after those, or `dynamic = 'force-dynamic'` in the previous model.

**Q33. How do you cache a database query across requests (previous model)?**
`unstable_cache(fn, keyParts, { revalidate, tags })`.

**Q34. What is `"use cache"`?**
A Next.js 16 Cache Components directive that marks a file, component or function as cacheable, keyed by its arguments, with `cacheLife` and `cacheTag` controlling lifetime and invalidation.

**Q35. Can a `"use cache"` function call `cookies()`?**
No. Read request data outside and pass values in as arguments.

**Q36. What happens to uncached dynamic parts under Cache Components?**
They render at request time inside `<Suspense>` boundaries while the static shell is prerendered (partial prerendering).

**Q37. `revalidatePath` vs `revalidateTag`?**
By route path vs by tag across every cached entry that carries it.

**Q38. What changed for `revalidateTag` in Next.js 16?**
It takes a cache-life profile as a second argument (`revalidateTag('posts', 'max')`); `updateTag` gives read-your-writes inside Server Actions.

**Q39. What is a Server Action?**
An async function marked `'use server'` that runs on the server and can be called from forms or client code through an automatic POST endpoint.

**Q40. Does `'use server'` mark Server Components?**
No. Server Components need no directive; `'use server'` marks Server Functions (Actions).

**Q41. What is progressive enhancement with Server Actions?**
A `<form action={serverAction}>` rendered by a Server Component submits as a normal HTML form even before JavaScript loads.

**Q42. What does `useActionState` return?**
`[state, formAction, isPending]`: the action's most recent result, a wrapped action for the form, and a pending flag.

**Q43. Why must `redirect()` stay outside try/catch?**
It works by throwing a special error; catching it cancels the redirect (rethrow with `unstable_rethrow` if needed).

**Q44. Five rules for secure Server Actions?**
Validate input with Zod, authenticate inside the action, authorize the specific resource, derive org and user from the session, return minimal data (and rate-limit expensive actions).

**Q45. Which protections does Next.js add to Server Actions?**
Encrypted non-deterministic action ids, removal of unused actions, encrypted closures, POST-only with an Origin vs Host check, and a 1 MB default body limit.

**Q46. Does an auth check in the page protect its Server Actions?**
No. Each action is its own entry point and must check the session itself.

**Q47. How do you avoid fetch waterfalls?**
Start independent requests together with `Promise.all`, preload with `cache()`, and give slow sections their own Suspense boundaries.

**Q48. When is TanStack Query still useful in Next.js?**
Infinite scroll, polling, client-side filters, optimistic updates with rollback, data from a separate API, or hooks shared with React Native.

**Q49. How do you pass server-started data to a Client Component without blocking?**
Pass the unawaited promise as a prop and unwrap it with `use()` inside a Suspense boundary.

**Q50. First checks for "the page shows stale data"?**
Is the route static? Is the data cached? Did the mutation revalidate the right path or tag? Is the Router Cache or a CDN serving an old version? Test with `next build && next start`.

## Backend APIs and MongoDB (see [03-nextjs-backend-apis-and-mongodb.md](03-nextjs-backend-apis-and-mongodb.md))

**Q51. Where do Route Handlers live, and what do they export?**
In `route.ts` files under `app/`, exporting functions named after HTTP methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`).

**Q52. How do you read a dynamic param in a Route Handler (Next.js 15+)?**
`export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) { const { id } = await params; }`

**Q53. How do you read the query string in a Route Handler?**
`request.nextUrl.searchParams.get('page')` on a `NextRequest`.

**Q54. How do you return 201 with a Location header?**
`NextResponse.json(body, { status: 201, headers: { Location: `/api/listings/${id}` } })`.

**Q55. How do you return 204?**
`new Response(null, { status: 204 })`, since a 204 response must not have a body.

**Q56. Server Actions vs Route Handlers?**
Actions for mutations from your own UI; Route Handlers for webhooks, mobile or partner clients, non-POST methods and streaming.

**Q57. When does a separate backend beat both?**
When several clients or teams share the API, or you need long-running jobs, websockets or independent scaling.

**Q58. What is middleware (proxy in Next.js 16) good for?**
Cheap, cross-cutting work before routes: redirects, rewrites, headers and optimistic auth checks.

**Q59. Why is middleware never enough for auth?**
It can be bypassed or misconfigured (CVE-2025-29927 let attackers skip it in self-hosted apps), so pages, handlers and actions must verify sessions too.

**Q60. Why cache the Mongoose connection on `globalThis`?**
Hot reload re-runs modules in development and warm serverless instances keep globals, so caching the connection promise reuses one pool instead of opening new connections per request.

**Q61. Why reset the cached promise when connecting fails?**
Otherwise every later request reuses the rejected promise and fails forever.

**Q62. What does `models.Listing || model('Listing', schema)` prevent?**
`OverwriteModelError` when a module re-runs and tries to compile the same model again.

**Q63. What does `.lean()` return?**
Plain JavaScript objects instead of Mongoose documents: faster and lighter, without `save()`, getters or virtuals.

**Q64. Why use Zod if Mongoose validates?**
Zod checks the request contract at the boundary (types, coercion, stripped unknown keys, clear 422s) before any database work; Mongoose protects stored data from all code paths.

**Q65. Offset vs cursor pagination in MongoDB?**
`skip/limit` gives page numbers but slows on deep pages and shifts with inserts; a cursor on `_id` (or sort key plus `_id`) is fast and stable.

**Q66. How do you map MongoDB errors to HTTP?**
`CastError` to 400 or 404, `ValidationError` to 422, duplicate key `11000` to 409, unknown errors to 500 with a generic message.

**Q67. How do you prevent IDOR in a Mongo-backed API?**
Query with both the id and the caller's org (`findOne({ _id, orgId })`), with the org from the session, and return 404 on no match.

**Q68. Can Mongoose run on the Edge runtime?**
No. It needs TCP sockets and Node APIs; use the Node.js runtime (recent docs also mark the route-level Edge option as deprecated).

**Q69. How do you test a Route Handler?**
Import the exported function and call it with a `NextRequest`, mocking auth and the database layer, or use `mongodb-memory-server` for integration tests.

**Q70. Name three serverless limits that shape Next.js APIs.**
Maximum duration, connection counts across instances, and request body size limits (also cold starts and no persistent disk).

**Q71. What does `after()` do?**
Runs work after the response is sent (stable since Next.js 15.1), suitable for short tasks like logging; real jobs belong in a queue.

## TanStack Router (see [04-tanstack-router.md](04-tanstack-router.md))

**Q72. What makes TanStack Router different?**
Full type inference for paths, params, search params, loader data and context, plus validated search params and built-in loader caching.

**Q73. What is `routeTree.gen.ts`?**
The route tree generated by the bundler plugin from `src/routes`; never edit it by hand.

**Q74. What does the `Register` declaration do?**
Registers your router's type so `Link`, `useNavigate`, `redirect` and hooks are checked against your routes.

**Q75. What is `__root.tsx`?**
The root route, created with `createRootRoute` or `createRootRouteWithContext<Ctx>()`, rendering the app shell and `<Outlet />`.

**Q76. What does `$` mean in a route file name?**
A path param (`$listingId`); a file named just `$` is a splat whose value is `params._splat`.

**Q77. What is a pathless layout?**
A route like `_authed.tsx` that adds no URL segment but wraps its children, often with a `beforeLoad` guard.

**Q78. How do you validate search params?**
`validateSearch` with a schema: a Zod 4 schema directly, or Zod 3 wrapped with `zodValidator` from `@tanstack/zod-adapter`.

**Q79. What type does `?page=2` produce?**
The number 2, because search values are JSON-parsed before validation.

**Q80. How do you update one search param and keep the rest?**
`navigate({ search: (prev) => ({ ...prev, page: prev.page + 1 }) })`.

**Q81. Why do loaders need `loaderDeps`?**
Loaders do not receive search params; declared deps are passed in and become part of the cache key, so cached data always matches the URL.

**Q82. Default loader `staleTime`?**
0: cached data shows immediately on revisit and reloads in the background; preloaded data stays fresh for 30 seconds.

**Q83. `beforeLoad` vs `loader`?**
`beforeLoad` runs serially parent to child before loaders (guards, context); loaders run in parallel to fetch data.

**Q84. How do you redirect unauthenticated users?**
`throw redirect({ to: '/login', search: { redirect: location.href } })` in `beforeLoad`.

**Q85. Is a `beforeLoad` guard security?**
No. It controls the UI; APIs and server functions must enforce authorization themselves.

**Q86. What is router context?**
Typed dependencies (QueryClient, auth, API clients) provided to `createRouter` and available in every `beforeLoad` and loader.

**Q87. How do loaders and TanStack Query work together?**
The loader calls `queryClient.ensureQueryData(options)`; the component calls `useSuspenseQuery(options)` with the same options, and `defaultPreloadStaleTime` is set to 0.

**Q88. What does `defaultPreload: 'intent'` do?**
Preloads a route's code and data when a link is hovered or focused.

## TanStack Start (see [05-tanstack-start.md](05-tanstack-start.md))

**Q89. What is TanStack Start?**
A full-stack React framework on TanStack Router and Vite (or Rsbuild) adding SSR, streaming, server functions, server routes and middleware; stable (1.0) since October 2026 after an RC phase from September 2025.

**Q90. How do you define a server function?**
`createServerFn({ method: 'POST' }).validator(schema).handler(async ({ data, context }) => ...)` (some RC versions named it `inputValidator`), called as `fn({ data })`.

**Q91. Where does a server function run when called from the browser?**
On the server: the client bundle contains an RPC stub that sends an HTTP request to an internal endpoint.

**Q92. Where do Start route loaders run?**
Both: on the server for the first SSR request and in the browser on client navigation.

**Q93. How do you read a cookie in a server function?**
`getCookie('session')` from `@tanstack/react-start/server`, inside the handler or middleware.

**Q94. Request middleware vs function middleware?**
Request middleware runs for every request (SSR, server routes, server functions); function middleware wraps server functions and passes typed context with `next({ context })`.

**Q95. What is a server route?**
A route with `server: { handlers: { GET, POST } }` that returns a `Response`, used for webhooks, health checks, REST endpoints or mounting oRPC.

**Q96. How does SSR data reach the client without refetching?**
Loader data and the dehydrated query cache are serialized into the HTML, and the client hydrates with them.

**Q97. Why create the `QueryClient` inside `getRouter()`?**
So each server request gets its own cache and users never see each other's data.

**Q98. How do you set the page title and canonical URL in Start?**
Return them from the route's `head()` (meta and links), optionally using `loaderData`; `<HeadContent />` renders them.

**Q99. How do env vars differ between server and client in Start?**
Server code uses `process.env`; client code only sees public-prefixed `import.meta.env` variables, which ship in the bundle.

**Q100. Server functions vs oRPC procedures?**
Server functions for page-specific work inside one Start app; oRPC procedures for a shared, documented API used by other services, mobile apps or Rails through OpenAPI.

**Q101. TanStack Start vs Next.js in one line?**
Start is client-first with explicit loaders, server functions and typed routing on Vite; Next.js is server-first with Server Components, built-in caching and Server Actions.

**Q102. How do you keep SEO when migrating a Rails page to Start?**
Same URLs (or 301s), SSR, metadata parity through `head()`, real 404s, regenerated sitemaps, good Core Web Vitals and a monitored strangler-fig rollout.

## Spot the bug

**Q103. What's wrong?**
```tsx
'use client';
import { db } from '@/lib/db';

export function ListingCount() {
  const count = db.listings.count();
  return <span>{count}</span>;
}
```
A Client Component imports the database client, which would ship server code and credentials to the browser (and cannot work there). Query in a Server Component or server function and pass the result down; mark the db module with `import 'server-only'`.

**Q104. What's wrong?**
```ts
'use server';
export async function deleteListing(id: string) {
  await Listing.deleteOne({ _id: id });
  revalidatePath('/marketplace');
}
```
No authentication or authorization: anyone can POST to this action and delete any listing. Check the session, scope the delete by `orgId` (and role), and validate `id`.

**Q105. What's wrong?**
```ts
export async function GET() {
  await mongoose.connect(process.env.MONGODB_URI!);
  return Response.json(await Listing.find().lean());
}
```
It connects on every request (exhausting connections) and returns every tenant's listings. Use a cached connection on `globalThis` and filter by the caller's org.

**Q106. What's wrong?**
```ts
// src/routes/marketplace/index.tsx (TanStack Start)
export const Route = createFileRoute('/marketplace/')({
  loader: () => db.select().from(listings),
});
```
Loaders also run in the browser on client navigation, so the database code and its secrets would be bundled for the client. Move the query into a server function and call it from the loader.

**Q107. What's wrong?**
```ts
export const Route = createFileRoute('/marketplace/')({
  validateSearch: MarketplaceSearch,
  loader: ({ location }) => fetchListings((location.search as { page: number }).page),
});
```
The loader reads search params without declaring them, so cached or preloaded data can belong to a different page. Declare `loaderDeps: ({ search }) => ({ page: search.page })` and use `deps.page`.

**Q108. What's wrong?**
```ts
try {
  const listing = await getListing(id);
  if (!listing) notFound();
  redirect(`/marketplace/${listing.slug}`);
} catch (err) {
  return { error: 'Something went wrong' };
}
```
`notFound()` and `redirect()` throw special errors, which this catch swallows. Call them outside the try block, or rethrow with `unstable_rethrow(err)` first.

**Q109. What's wrong?**
```bash
NEXT_PUBLIC_PAYMENTS_SECRET_KEY=replace-me
```
Anything prefixed with `NEXT_PUBLIC_` is inlined into the browser bundle, so the secret key is public. Keep secrets unprefixed and use them only in server code.

**Q110. What's wrong?**
```ts
export async function DELETE(_req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  await Listing.deleteOne({ _id: id });
  return NextResponse.json(null, { status: 204 });
}
```
No auth or org scoping (IDOR), no id validation, and a JSON body on a 204. Require the session, delete with `{ _id: id, orgId }`, return 404 when nothing was deleted, and respond with `new Response(null, { status: 204 })`.
