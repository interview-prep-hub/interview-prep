# TanStack Router

> **Tier:** T1 (know cold) · **Related:** [Routing in React](../06-react/08-routing.md), [TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [TanStack Start](05-tanstack-start.md), [Next.js fundamentals](01-nextjs-fundamentals.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [Generics and utility types](../03-languages/typescript/02-generics-utility-types.md)
> **On your resume:** the TanStack Start platform at Devntech, which is built on TanStack Router; you worked on the marketplace page migration and org-scoped access control there.

## Quick revision
- TanStack Router (v1 since December 2023) is a fully type-safe React router: paths, params, search params, loader data and context are inferred, so a wrong `<Link to>` or a missing param is a compile error.
- File-based routing: the bundler plugin (`tanstackRouter()` from `@tanstack/router-plugin/vite`) scans `src/routes` and generates `routeTree.gen.ts`; each file exports `Route = createFileRoute('/path')({...})`.
- `__root.tsx` is the root route (`createRootRoute`, or `createRootRouteWithContext<Ctx>()`), rendering the app shell and an `<Outlet />`.
- Register the router type once (`declare module '@tanstack/react-router' { interface Register { router: typeof router } }`) so every hook and link is typed.
- Code-based routing (`createRoute({ getParentRoute, path })` plus `addChildren`) is the alternative to files.
- File naming: `$param` for path params, `index.tsx` for index routes, `_layout` for pathless layouts, `(group)` folders, `$.tsx` for splats (`params._splat`), a trailing `_` to un-nest, a `-` prefix to exclude files.
- Search params are typed state: `validateSearch` with a schema (a Zod 4 schema directly; Zod 3 through `@tanstack/zod-adapter`), read with `Route.useSearch()`, update with `navigate({ search: (prev) => ({ ...prev, page: 2 }) })`. Values are JSON-parsed, so numbers stay numbers.
- Loaders run before rendering, in parallel across matched routes. They do not receive search params: declare the ones they need with `loaderDeps`, which also become part of the cache key.
- Built-in stale-while-revalidate cache: `staleTime` 0 by default (reload in the background on revisit), `preloadStaleTime` 30 seconds, `gcTime` 5 minutes in recent docs; `shouldReload` for finer control.
- `beforeLoad` runs serially, parent to child, before any loader: the place for auth guards (`throw redirect({ to: '/login', search: { redirect: location.href } })`) and for adding context.
- Router context (`createRootRouteWithContext`) injects the `QueryClient`, auth state and API clients into every `beforeLoad` and loader.
- Route UI states: `pendingComponent` (after `pendingMs`, 1 second by default, then shown at least `pendingMinMs`, 500 ms), `errorComponent`, `notFoundComponent`, and `throw notFound()` from loaders.
- Typed navigation: `<Link to="/listings/$listingId" params={{ listingId }} search={{ tab: 'reviews' }}>`, `useNavigate({ from: Route.fullPath })`, `redirect()`.
- Preloading: `defaultPreload: 'intent'` loads route code and data on hover or focus; links can also preload on viewport entry.
- Code splitting: `autoCodeSplitting: true` in the plugin, or `.lazy.tsx` files with `createLazyFileRoute`.
- With TanStack Query: loaders call `queryClient.ensureQueryData(options)`, components call `useSuspenseQuery(options)`, and the router's `defaultPreloadStaleTime` is set to 0. Recent Query releases deprecate `ensureQueryData` in favor of `queryClient.query(...)`.
- vs React Router: inferred types everywhere, validated search params, built-in loader caching. vs Next.js: a client router (SSR comes from TanStack Start), explicit loaders instead of Server Components.

## 1. Why a type-safe router

In most routers, URLs are strings. `navigate('/listngs/' + id)` compiles, a renamed route breaks links silently, `useParams()` returns `string | undefined` for everything, and search params are untyped strings that every component parses differently. TanStack Router treats routes as typed values:
- **Paths:** `<Link to>` only accepts routes that exist.
- **Params:** required params must be passed, with the right names.
- **Search params:** validated once at the route, then typed everywhere (`page` is a `number`).
- **Loader data and context:** `Route.useLoaderData()` returns exactly what the loader returned.

The URL becomes a reliable piece of state: filters and pagination live in the URL, so pages are shareable, bookmarkable and survive refreshes, which matters for a marketplace.

## 2. Setup

```ts
// vite.config.ts
import { tanstackRouter } from '@tanstack/router-plugin/vite';
import react from '@vitejs/plugin-react';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    tanstackRouter({ target: 'react', autoCodeSplitting: true }), // must come before the React plugin
    react(),
  ],
});
```

```tsx
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { createRouter, RouterProvider } from '@tanstack/react-router';
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { useAuth } from './auth';
import { routeTree } from './routeTree.gen';          // generated by the plugin: never edit it by hand

const queryClient = new QueryClient();

const router = createRouter({
  routeTree,
  context: { queryClient, auth: undefined! },         // auth is supplied at render time below
  defaultPreload: 'intent',                           // preload code and data on hover or focus
  defaultPreloadStaleTime: 0,                         // TanStack Query decides freshness, not the router
});

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;                            // types every Link, hook and redirect in the app
  }
}

function App() {
  const auth = useAuth();
  return <RouterProvider router={router} context={{ auth }} />;
}

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}><App /></QueryClientProvider>
  </StrictMode>,
);
```

Plugin and option names have changed across v1 releases (the plugin was previously exported as `TanStackRouterVite`), so match the docs to your installed version.

## 3. File-based and code-based routes

| File in `src/routes` | URL | Notes |
|---|---|---|
| `__root.tsx` | every page | root route: shell, providers, `<Outlet />` |
| `index.tsx` | `/` | |
| `marketplace.tsx` or `marketplace/route.tsx` | `/marketplace` | layout for its children (renders `<Outlet />`) |
| `marketplace/index.tsx` | `/marketplace` exactly | route id `/marketplace/` |
| `marketplace/$listingId.tsx` | `/marketplace/42` | `$` marks a path param; dots and folders are interchangeable (`marketplace.$listingId.tsx`) |
| `_authed.tsx` | no URL segment | pathless layout: wraps children, ideal for guards |
| `_authed/dashboard.tsx` | `/dashboard` | protected by `_authed`'s `beforeLoad` |
| `(admin)/users.tsx` | `/users` | route group folder: organization only |
| `files/$.tsx` | `/files/a/b/c` | splat; the rest of the path is `params._splat` |
| `marketplace_.$listingId.edit.tsx` | `/marketplace/42/edit` | trailing `_` un-nests: not rendered inside the `marketplace` layout |
| `-components/Card.tsx` | none | `-` prefix: ignored by the generator |

```tsx
// Code-based alternative (no plugin)
import { createRootRoute, createRoute, createRouter } from '@tanstack/react-router';

const rootRoute = createRootRoute({ component: RootLayout });
const listingsRoute = createRoute({ getParentRoute: () => rootRoute, path: '/listings', component: Listings });
const listingRoute = createRoute({ getParentRoute: () => listingsRoute, path: '$listingId', component: ListingPage });

const routeTree = rootRoute.addChildren([listingsRoute.addChildren([listingRoute])]);
export const router = createRouter({ routeTree });
```

File-based routing is the recommended default: less boilerplate, automatic code splitting and the same type inference.

## 4. Root route and router context

```tsx
// src/routes/__root.tsx
import type { QueryClient } from '@tanstack/react-query';
import { createRootRouteWithContext, Link, Outlet } from '@tanstack/react-router';
import type { AuthState } from '../auth';

type RouterContext = { queryClient: QueryClient; auth: AuthState };

export const Route = createRootRouteWithContext<RouterContext>()({
  component: () => (
    <>
      <nav>
        <Link to="/" activeOptions={{ exact: true }} activeProps={{ className: 'font-bold' }}>Home</Link>
        <Link to="/marketplace" activeProps={{ className: 'font-bold' }}>Marketplace</Link>
      </nav>
      <Outlet />                                       {/* the matched child route renders here */}
    </>
  ),
  notFoundComponent: () => <p>Page not found</p>,
});
```

Router context is dependency injection for routes: whatever you pass to `createRouter({ context })` (and `RouterProvider context`) is available as `context` in every `beforeLoad` and loader, fully typed. A `beforeLoad` can return an object that is merged into the context of its child routes, which is how a guard hands a non-null `user` (or the active org) down the tree.

## 5. Path params

```tsx
// src/routes/marketplace/$listingId.tsx
export const Route = createFileRoute('/marketplace/$listingId')({
  loader: ({ params }) => fetchListing(params.listingId),   // params.listingId: string
  component: ListingPage,
});

function ListingPage() {
  const { listingId } = Route.useParams();
  const listing = Route.useLoaderData();                    // typed as fetchListing's resolved value
  return <h1>{listing.title} ({listingId})</h1>;
}
```

Outside the route file, `useParams({ from: '/marketplace/$listingId' })` gives the same typing. A route's `params` option can parse and stringify params (for example, numeric ids); check the docs for its current shape.

## 6. Search params with validation

```tsx
// src/routes/marketplace/index.tsx
import { createFileRoute, Link, useNavigate } from '@tanstack/react-router';
import { z } from 'zod';

const SORTS = ['newest', 'price_asc', 'price_desc'] as const;

const MarketplaceSearch = z.object({
  page: z.number().int().min(1).default(1),
  category: z.string().optional(),
  sort: z.enum(SORTS).default('newest'),
});

export const Route = createFileRoute('/marketplace/')({
  validateSearch: MarketplaceSearch,                 // Zod 4: pass the schema directly (Standard Schema)
  component: MarketplacePage,
});

function MarketplacePage() {
  const { page, category, sort } = Route.useSearch(); // page: number, sort: 'newest' | 'price_asc' | 'price_desc'
  const navigate = useNavigate({ from: Route.fullPath });

  return (
    <>
      {SORTS.map((s) => (
        <button key={s} aria-pressed={s === sort}
          onClick={() => navigate({ search: (prev) => ({ ...prev, sort: s, page: 1 }) })}>
          {s}
        </button>
      ))}
      <ListingsGrid page={page} category={category} sort={sort} />
      <Link from={Route.fullPath} search={(prev) => ({ ...prev, page: prev.page + 1 })}>Next page</Link>
    </>
  );
}
```

```ts
// Zod 3: wrap with the adapter so link inputs keep correct optional types
// import { fallback, zodValidator } from '@tanstack/zod-adapter';
// const MarketplaceSearch = z.object({ page: fallback(z.number().int().min(1), 1).default(1), ... });
// validateSearch: zodValidator(MarketplaceSearch)
```

- Search values are serialized as JSON-like values: `?page=2` arrives as the number `2`, and arrays or objects survive round trips.
- `.default()` makes the param optional for links; `.catch()` (Zod 4) or `fallback()` (Zod 3 adapter) turns garbage in a hand-edited URL into a default instead of an error.
- The functional `search: (prev) => ...` form updates one key without dropping the others.
- Search middlewares (`retainSearchParams`, `stripSearchParams`) can keep chosen params across links or remove params equal to their defaults for cleaner URLs.

## 7. Loaders, loaderDeps and caching

```tsx
export const Route = createFileRoute('/marketplace/')({
  validateSearch: MarketplaceSearch,
  loaderDeps: ({ search: { page, category, sort } }) => ({ page, category, sort }), // what the loader depends on
  loader: ({ deps, abortController }) => fetchListings(deps, { signal: abortController.signal }),
  component: MarketplacePage,
});
// in the component: const { items, total } = Route.useLoaderData();
```

- **Why loaders:** data starts loading as soon as navigation begins (or on hover with preloading), in parallel for every matched route, instead of waterfalls of `useEffect` calls after rendering.
- **Why `loaderDeps`:** loaders do not receive search params directly. If they read them anyway, cached or preloaded data could belong to a different URL. Declared deps join the path params in the cache key.
- **Caching:** results are cached per route plus params plus deps. With the default `staleTime` of 0, revisiting a route shows cached data immediately and reloads in the background; preloaded data counts as fresh for 30 seconds (`preloadStaleTime`); unused entries are garbage-collected after `gcTime`. `shouldReload` and per-route `staleTime` tune this.
- `router.invalidate()` marks loader data stale and reloads the active routes, for example after a mutation.
- Loaders run on navigation in the browser for a plain SPA; in TanStack Start they also run on the server for the first request ([TanStack Start](05-tanstack-start.md)).

## 8. beforeLoad and auth guards

```tsx
// src/routes/_authed.tsx: a pathless layout guarding every route nested under it
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router';

export const Route = createFileRoute('/_authed')({
  beforeLoad: ({ context, location }) => {
    if (!context.auth.user) {
      throw redirect({ to: '/login', search: { redirect: location.href } }); // /login validates `redirect`
    }
    return { user: context.auth.user };              // children get a non-null user in context
  },
  component: () => <Outlet />,
});

// src/routes/_authed/admin.tsx
export const Route = createFileRoute('/_authed/admin')({
  beforeLoad: ({ context }) => {
    if (context.user.role !== 'admin') throw redirect({ to: '/dashboard' });
  },
  component: AdminPage,
});
```

- `beforeLoad` runs before any loader, from parent to child, so a failing guard prevents child loaders from fetching.
- After login, navigate to `search.redirect` (validated as a relative path) to return the user where they started.
- Guards improve the experience; they do not secure data. Every API, server function or oRPC procedure must check the session and org itself, because anyone can call it directly.

## 9. Pending, error and not-found states

```tsx
import { createFileRoute, notFound, useRouter, type ErrorComponentProps } from '@tanstack/react-router';

export const Route = createFileRoute('/marketplace/$listingId')({
  loader: async ({ params }) => {
    const listing = await fetchListing(params.listingId);
    if (!listing) throw notFound();                  // renders notFoundComponent
    return listing;
  },
  pendingComponent: ListingSkeleton,                 // appears only if loading exceeds pendingMs (1 s by default)
  errorComponent: ListingError,
  notFoundComponent: () => <p>This listing does not exist.</p>,
  component: ListingPage,
});

function ListingError({ error }: ErrorComponentProps) {
  const router = useRouter();
  return (
    <div role="alert">
      <p>Could not load the listing: {error.message}</p>
      <button onClick={() => router.invalidate()}>Retry</button>  {/* re-runs the loaders */}
    </div>
  );
}
```

Router-level defaults (`defaultPendingComponent`, `defaultErrorComponent`, `defaultNotFoundComponent`) cover routes without their own. `pendingMs` avoids flashing spinners on fast loads, and `pendingMinMs` stops a spinner from flickering when it does appear.

## 10. Typed navigation

```tsx
<Link to="/marketplace/$listingId" params={{ listingId: listing.id }} search={{ tab: 'reviews' }}>
  {listing.title}
</Link>
// <Link to="/marketplce/$listingId" ...>  compile error: no such route
// <Link to="/marketplace/$listingId">      compile error: params.listingId is required

const navigate = useNavigate();
await navigate({ to: '/marketplace/$listingId', params: { listingId }, replace: true });
```

- `from` (on `useNavigate`, `Link`, hooks) makes relative navigation and search updates type-safe for a specific route.
- `activeProps` and `activeOptions` style active links; `redirect()` is thrown from `beforeLoad` or loaders.
- `useRouterState()` exposes pending status and location for global loading bars.

## 11. Preloading and code splitting

- `defaultPreload: 'intent'` preloads a route's code and loader data when the user hovers or focuses a link (after a short delay), so the click feels instant. Links can override with `preload="viewport"`, `"render"` or `false`.
- `autoCodeSplitting: true` lets the plugin split route components into separate chunks automatically. The manual alternative is a `.lazy.tsx` file with `createLazyFileRoute` that holds only the components, while critical options (search validation, loaders, `beforeLoad`) stay in the main route file.

## 12. TanStack Query integration

```ts
// src/queries/listings.ts
import { queryOptions } from '@tanstack/react-query';

type ListingsSearch = { page: number; category?: string; sort: string };

export const listingsQueryOptions = (search: ListingsSearch) =>
  queryOptions({
    queryKey: ['listings', search],
    queryFn: async ({ signal }) => {
      const params = new URLSearchParams({ page: String(search.page), sort: search.sort });
      if (search.category) params.set('category', search.category);
      const res = await fetch(`/api/listings?${params}`, { signal });
      if (!res.ok) throw new Error(`Failed to load listings (${res.status})`);
      return (await res.json()) as { items: { id: string; title: string }[]; total: number };
    },
    staleTime: 30_000,
  });
// with oRPC instead: orpc.marketplace.list.queryOptions({ input: search })
```

```tsx
export const Route = createFileRoute('/marketplace/')({
  validateSearch: MarketplaceSearch,
  loaderDeps: ({ search: { page, category, sort } }) => ({ page, category, sort }),
  loader: ({ context: { queryClient }, deps }) => queryClient.ensureQueryData(listingsQueryOptions(deps)),
  component: MarketplacePage,
});

function MarketplacePage() {
  const search = Route.useSearch();
  const { data } = useSuspenseQuery(listingsQueryOptions(search)); // already cached by the loader: no spinner
  return <ListingsGrid items={data.items} total={data.total} />;
}
```

- The loader warms the cache before rendering; the component subscribes to the same query, so background refetches, invalidation and mutations update it.
- Set the router's `defaultPreloadStaleTime` to 0 so TanStack Query, not the router cache, decides when data is stale.
- After a mutation, `queryClient.invalidateQueries({ queryKey: ['listings'] })` refetches active listing queries.
- Recent TanStack Query v5 releases deprecate `ensureQueryData` and `fetchQuery` in favor of `queryClient.query(...)` (`ensureQueryData` maps to `query({ ...options, staleTime: 'static' })`); the old methods keep working until the next major, so check the version in your project.

## 13. TanStack Router vs React Router vs Next.js

| | TanStack Router | React Router (v7) | Next.js App Router |
|---|---|---|---|
| Kind | client router; SSR through TanStack Start | library (declarative or data mode) or framework mode | full framework |
| Route definitions | files (plugin) or code | JSX, objects, or `routes.ts` in framework mode | folders in `app/` |
| Type safety | inferred everywhere: paths, params, search, loader data, context | generated types in framework mode | typed routes and generated helpers in recent versions |
| Search params | validated, typed, JSON-serialized | `URLSearchParams` strings | `searchParams` prop and `useSearchParams` (strings) |
| Data loading | loaders with SWR cache, `loaderDeps`, preloading; pairs with TanStack Query | loaders and actions, no built-in cache | async Server Components and Next.js caching |
| Mutations | your choice: TanStack Query, server functions, oRPC | actions | Server Actions |
| Rendering | CSR, or SSR and streaming with Start | CSR, SSR, prerendering in framework mode | SSR, SSG, ISR, RSC, streaming |

## Common pitfalls
- Editing `routeTree.gen.ts` by hand, or forgetting to commit or regenerate it.
- Skipping the `Register` declaration, so links and hooks lose their types.
- Reading search params inside a loader without `loaderDeps`, so cached data does not match the URL.
- Putting every search param in `loaderDeps`, causing reloads for params the loader ignores.
- Treating `beforeLoad` guards as security instead of enforcing auth on the server.
- Using plain `useQuery` after a loader prefetch and showing an unnecessary loading state instead of `useSuspenseQuery`.
- Leaving the router's preload cache in charge while also using TanStack Query (set `defaultPreloadStaleTime: 0`).
- Search schemas without defaults or fallbacks, so a hand-edited URL renders an error page.
- Replacing the whole search object on navigation and dropping filters (use the `(prev) => ({ ...prev })` form).
- Mixing plugin or adapter versions from different eras of the docs.

## How to talk about this in an interview
- Connect it to the Devntech platform: TanStack Start is TanStack Router plus a server, so the marketplace pages you migrated were Router routes with typed params, validated search params for filters and pagination, and loaders that prefetched data.
  > **Fill in:** which marketplace routes you built or migrated, which search params they validated (category, sort, page, location?), and how their loaders fetched data (oRPC queries, server functions, TanStack Query).
  > **Fill in:** how org-scoped access showed up in routing (a guarded pathless layout, an org switcher in context, redirects for users without access).
- Explain the value in one line: "Routes are typed values, so broken links and wrong params fail the build, and search params become validated state that makes marketplace filters shareable."
- Be ready to contrast with Next.js routing and with React Router, which you know from React work: see [Routing in React](../06-react/08-routing.md) and the [platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md).

## Interview questions

**Q1. Why choose TanStack Router over React Router?**
Full type inference for paths, params, search params, loader data and context (the plugin generates the route tree; you never write route types by hand), first-class validated search params, and built-in loader caching with preloading. React Router is more widespread and its framework mode (formerly Remix) is a full framework.
*Follow-up:* What is the cost? A build plugin, a generated route tree and a steeper learning curve.

**Q2. How does file-based routing work in TanStack Router, and what is `routeTree.gen.ts`?**
The bundler plugin watches `src/routes`, derives the route hierarchy from file and folder names, and generates `routeTree.gen.ts`, which wires every `createFileRoute` export into one typed tree passed to `createRouter`. It is generated output: never edit it by hand.
*Follow-up:* Can you skip the plugin? Yes, with code-based routes built from `createRoute` and `addChildren`.

**Q3. What does the `Register` interface declaration do?**
It tells the library's types which router your app uses (`interface Register { router: typeof router }`), so `Link`, `useNavigate`, `redirect` and route hooks everywhere are checked against your actual route tree.
*Follow-up:* Symptom of forgetting it? Links accept any string and hooks return loosely typed data.

**Q4. How do you validate search params, and why keep state in them?**
Give the route a `validateSearch` schema (a Zod 4 schema directly, or Zod 3 through `zodValidator`), with defaults and fallbacks for bad input; read with `Route.useSearch()` and update with `navigate({ search: (prev) => ... })`. URL state is shareable, bookmarkable, survives refreshes and works with back and forward, which is ideal for filters, sorting and pagination.
*Follow-up:* What type does `?page=2` produce? The number 2, because search values are JSON-parsed before validation.

**Q5. Why can't a loader read search params directly, and what is `loaderDeps`?**
Loader results are cached by route, params and declared dependencies. If a loader read search params freely, a cached or preloaded result could be served for a different URL. `loaderDeps` declares exactly which search values the loader uses; they are passed in as `deps` and included in the cache key.
*Follow-up:* What if you include too many? The loader reloads when irrelevant params change.

**Q6. How does loader caching work?**
Results are cached per route, params and deps with stale-while-revalidate semantics: `staleTime` defaults to 0, so revisits show cached data and reload in the background; preloaded data stays fresh for `preloadStaleTime` (30 seconds); unused entries expire after `gcTime`. `router.invalidate()` forces reloads.
*Follow-up:* What changes with TanStack Query? Set `defaultPreloadStaleTime: 0` and let Query's `staleTime` decide.

**Q7. `beforeLoad` vs `loader`: what goes where?**
`beforeLoad` runs serially from parent to child before any loader: authentication and authorization redirects, and computing context for children. Loaders then run in parallel to fetch data. A failing `beforeLoad` stops child loaders from running.
*Follow-up:* Is a `beforeLoad` guard enough to protect data? No: the API must enforce access, because it can be called directly.

**Q8. What is router context, and how do you inject the `QueryClient` and auth?**
A typed object declared with `createRootRouteWithContext<Ctx>()` and provided through `createRouter({ context })` and `RouterProvider context`. Every `beforeLoad` and loader receives it, so they can call `context.queryClient.ensureQueryData(...)` or check `context.auth.user`. `beforeLoad` return values extend it for child routes.
*Follow-up:* Why inject instead of importing a singleton? Easier testing, and in SSR each request needs its own client.

**Q9. How do you integrate TanStack Query with loaders?**
Define `queryOptions` factories; in the loader, call `queryClient.ensureQueryData(options)` to warm the cache before rendering; in the component, call `useSuspenseQuery(options)` with the same options, so data is available immediately and stays reactive to invalidation. Recent Query versions prefer `queryClient.query(...)` over `ensureQueryData`.
*Follow-up:* Why `useSuspenseQuery` instead of `useQuery`? Its data is never undefined, and the loader has already resolved it.

**Q10. How do pending, error and not-found components work?**
Routes declare `pendingComponent` (shown only if loading exceeds `pendingMs`, then for at least `pendingMinMs`), `errorComponent` (receives the error; retry with `router.invalidate()`) and `notFoundComponent` (rendered when a loader or `beforeLoad` throws `notFound()`). Router-level defaults cover the rest.
*Follow-up:* Why the `pendingMs` delay? To avoid flashing spinners on fast navigations.

**Q11. How do typed `Link` and `useNavigate` work, and what happens on a typo?**
`to` accepts only paths in the registered route tree, and the required `params` and `search` are inferred from that route, so a misspelled path or a missing param fails type checking. `from` makes relative navigation and functional search updates typed for a specific route.
*Follow-up:* How do you update one search param? `search: (prev) => ({ ...prev, page: prev.page + 1 })`.

**Q12. What does preloading on intent do?**
With `defaultPreload: 'intent'`, hovering or focusing a link starts loading that route's code chunk and loader data, so by the time the user clicks, the page is often ready. Preloaded data is cached for `preloadStaleTime`.
*Follow-up:* Risk? Extra requests on pages with many links; tune per link or route.

**Q13. How do you code split routes?**
Enable `autoCodeSplitting` in the plugin so route components load as separate chunks, or move components into `.lazy.tsx` files created with `createLazyFileRoute` while keeping critical options such as search validation, loaders and `beforeLoad` in the main route file.
*Follow-up:* Why keep loaders out of lazy files? The router needs them to start fetching before the component code arrives.

**Q14. Pathless layouts vs route groups?**
A pathless layout (`_authed.tsx`) is a real route without a URL segment: it can render a layout and run `beforeLoad` for all nested routes, which suits auth guards. A route group folder (`(admin)`) only organizes files and affects neither URLs nor nesting.
*Follow-up:* How do you render a child outside its parent's layout? Un-nest it with a trailing underscore, like `marketplace_.$listingId.edit.tsx`.

**Q15. How does TanStack Router differ from Next.js routing?**
TanStack Router is a client router with inferred types, validated search params and loaders; SSR, streaming and server functions come from TanStack Start. Next.js is a server-first framework with Server Components, file conventions such as `layout` and `loading`, framework caching and Server Actions. TanStack keeps data loading explicit; Next.js puts it inside components on the server.
*Follow-up:* Which suits a dashboard-heavy SaaS? TanStack Router or Start often fits well; content-heavy SEO sites lean toward Next.js.

**Q16. How do you update one search param without losing the others, and keep URLs clean?**
Use the functional form `navigate({ search: (prev) => ({ ...prev, sort: 'price_asc', page: 1 }) })`, resetting pagination when filters change. To keep URLs short, strip params equal to their defaults with the `stripSearchParams` middleware, and retain global params across links with `retainSearchParams`.
*Follow-up:* Why reset `page` when the filter changes? Page 7 of the old results may not exist in the new ones.

## Further reading
- [TanStack Router: Overview](https://tanstack.com/router/latest/docs/framework/react/overview)
- [TanStack Router: File-based routing](https://tanstack.com/router/latest/docs/framework/react/routing/file-based-routing)
- [TanStack Router: Search params](https://tanstack.com/router/latest/docs/framework/react/guide/search-params)
- [TanStack Router: Data loading](https://tanstack.com/router/latest/docs/framework/react/guide/data-loading)
- [TanStack Router: TanStack Query integration](https://tanstack.com/router/latest/docs/integrations/query)
