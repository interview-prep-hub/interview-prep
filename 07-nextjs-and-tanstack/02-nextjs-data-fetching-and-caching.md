# Next.js Data Fetching, Caching and Server Actions

> **Tier:** T1 (know cold) · **Related:** [Next.js fundamentals](01-nextjs-fundamentals.md), [Next.js backend APIs and MongoDB](03-nextjs-backend-apis-and-mongodb.md), [React 19 and Server Components](../06-react/09-react-19-and-server-components.md), [TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [Forms and validation](../06-react/05-forms-and-validation.md), [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md), [Web security](../04-web-fundamentals/07-web-security-owasp.md)
> **On your resume:** the Next.js (TypeScript) and MongoDB work at RemoteDev: how pages load data, how mutations refresh what users see, and how to debug "why is this page stale?"

## Quick revision
- In the App Router, fetch data inside async Server Components: `await fetch(...)` or call the database directly through a server-only data access layer.
- Request memoization: identical GET `fetch` calls in one render run once; wrap ORM calls in React `cache()` for the same deduplication.
- Previous caching model (no Cache Components): the **Data Cache** (fetch results across requests; opt in with `cache: 'force-cache'` or `next: { revalidate }`), the **Full Route Cache** (prerendered HTML and RSC payload of static routes) and the **Router Cache** (RSC payloads in the browser).
- Next.js 15 changed the defaults: `fetch`, GET Route Handlers and client navigations are **not cached** by default. Static routes are still prerendered at build, so their data is captured then.
- A route turns dynamic when it reads `cookies()`, `headers()` or `searchParams`, calls `connection()`, or sets `dynamic = 'force-dynamic'` (previous model).
- Time-based revalidation: `next: { revalidate: 60 }` or `export const revalidate = 60`. On-demand: `revalidatePath()` and `revalidateTag()` from Server Actions or Route Handlers.
- `unstable_cache(fn, keyParts, { revalidate, tags })` caches non-fetch work (database queries) in the previous model.
- Next.js 16 **Cache Components** (`cacheComponents: true`): nothing is cached unless marked with `"use cache"` (file, component or function) plus `cacheLife()` and `cacheTag()`; uncached request-time parts sit inside `<Suspense>`; the static shell is prerendered.
- Next.js 16 APIs: `revalidateTag(tag, 'max')` takes a cache-life profile (stale-while-revalidate); `updateTag(tag)` gives read-your-writes inside Server Actions; `refresh()` re-renders uncached data.
- Server Actions are async functions marked `'use server'`; Next.js exposes each as a POST endpoint callable from `<form action={fn}>` or client code.
- Forms bound to Server Actions work before JavaScript loads (progressive enhancement).
- `useActionState(action, initialState)` returns `[state, formAction, isPending]`; `useFormStatus` drives submit buttons; `useOptimistic` shows changes instantly.
- After a mutation, revalidate (`revalidatePath`, `revalidateTag`/`updateTag`) and optionally `redirect()`, which throws, so keep it outside try/catch.
- Security: every Server Action is a public endpoint. Validate input with Zod, authenticate and authorize inside each action, derive user and org from the session, return minimal data. Next.js adds unguessable action ids, encrypted closures, an Origin vs Host check against CSRF and a 1 MB default body limit.
- Streaming: `loading.tsx` and `<Suspense>` let fast parts render first; pass promises to Client Components and unwrap them with `use()`.
- Avoid waterfalls: start independent requests together (`Promise.all`), preload with `cache()` plus `void getX()`, and await data in the component that needs it.
- TanStack Query still fits client-heavy parts: infinite scroll, polling, optimistic interfaces, client-side filters, data from a separate API.
- Expected errors are returned as values (form state) or handled with `notFound()`; unexpected ones throw to `error.tsx`.

## 1. Fetching in Server Components

```tsx
// app/marketplace/page.tsx: an async Server Component
import { getPublishedListings } from '@/data/listings';

export default async function MarketplacePage() {
  const listings = await getPublishedListings();   // runs on the server; queries and secrets never reach the browser
  return <ListingGrid listings={listings} />;
}
```

```ts
// src/data/listings.ts: a data access layer (DAL)
import 'server-only';
import { cache } from 'react';
import { requireUser } from '@/lib/auth';
import { connectToDatabase } from '@/lib/mongodb';
import { Listing } from '@/models/listing';

export const getPublishedListings = cache(async () => {
  const user = await requireUser();                 // authorization lives next to the data
  await connectToDatabase();
  const docs = await Listing.find({ orgId: user.orgId, status: 'published' })
    .sort({ createdAt: -1 }).limit(50).lean();
  return docs.map((d) => ({ id: String(d._id), title: d.title, priceCents: d.priceCents })); // a DTO
});
```

Next.js's security guidance describes three approaches: call an existing HTTP API from Server Components (treat it as zero trust), build a data access layer (recommended for new projects: server-only, authorization inside, DTOs out), or query directly in components (fine for prototypes). Pick one and stay consistent.

Never fetch your own Route Handler from a Server Component: it adds an HTTP hop to the same server. Call the DAL function directly.

## 2. Static vs dynamic rendering

A route is **static** (prerendered at build or revalidation time) unless something needs request-time information:

| Makes a route dynamic | Example |
|---|---|
| Request APIs | `await cookies()`, `await headers()` |
| Search params | reading `await searchParams` in a page |
| An explicit opt-in | `await connection()` from `next/server`, or `export const dynamic = 'force-dynamic'` (previous model) |
| Uncached data fetched after a request API | `fetch(url)` with no cache option after reading cookies |

`next build` prints each route with a symbol: static (prerendered), SSG (prerendered via `generateStaticParams`) or dynamic (rendered per request). When a page is unexpectedly static or dynamic, that output is the first thing to check.

## 3. The caching layers (previous model)

This model applies when Cache Components is not enabled (all of Next.js 15, and Next.js 16 by default).

| Layer | What it stores | Where | Lifetime | Opt out or invalidate |
|---|---|---|---|---|
| Request memoization | results of identical GET `fetch` calls (and functions wrapped in React `cache`) during one render | server memory | one request | not needed |
| Data Cache | `fetch` responses that opted into caching | server (file system, or a shared cache handler) | until revalidated; can survive deploys | `cache: 'no-store'`; `next.revalidate`; `revalidateTag`, `revalidatePath` |
| Full Route Cache | HTML and RSC payload of static routes | server | until revalidated or redeployed | make the route dynamic; revalidate |
| Router Cache | RSC payloads of visited and prefetched segments | browser memory | the session, subject to `staleTimes` | `router.refresh()`; revalidation or cookie changes inside a Server Action |

## 4. What Next.js 15 changed

- `fetch` requests are **not cached** in the Data Cache unless they opt in.
- GET Route Handlers are dynamic by default (they were static before).
- The client Router Cache no longer reuses page segments on navigation by default (`staleTimes.dynamic` is 0); layouts and back/forward navigation still reuse cached payloads.

```ts
await fetch('https://api.example.com/categories', { cache: 'force-cache' });            // cache until revalidated
await fetch('https://api.example.com/rates', { next: { revalidate: 3600 } });           // refresh at most hourly
await fetch(`https://api.example.com/listings?org=${orgId}`, {
  cache: 'force-cache',
  next: { tags: ['listings'] },                                                         // invalidate with revalidateTag
});
await fetch('https://api.example.com/me', { cache: 'no-store' });                       // always fresh
```

```ts
// database queries, which do not go through fetch (previous model)
import { unstable_cache } from 'next/cache';

export const getCategoryCounts = unstable_cache(
  async (orgId: string) => countListingsByCategory(orgId),
  ['category-counts'],                               // key prefix; arguments are added to the key
  { revalidate: 300, tags: ['listings'] },
);
```

The trap: "not cached by default" refers to the Data Cache. A static route still runs its fetches once at build time and bakes the results into the prerendered HTML, which then stays until the route is revalidated or redeployed.

## 5. Next.js 16 Cache Components and "use cache"

Next.js 16 introduced an explicit model, enabled with `cacheComponents: true` in `next.config.ts`. Hedge the details in interviews and check the docs:

```tsx
// app/marketplace/page.tsx (cacheComponents: true)
import { cacheLife, cacheTag } from 'next/cache';
import { Suspense } from 'react';

export default function MarketplacePage() {
  return (
    <>
      <CategoryList />                                    {/* cached: part of the prerendered static shell */}
      <Suspense fallback={<p>Loading your saved listings...</p>}>
        <SavedListings />                                 {/* reads cookies(): rendered per request and streamed in */}
      </Suspense>
    </>
  );
}

async function CategoryList() {
  'use cache';
  cacheLife('hours');                                     // a built-in profile; custom profiles live in next.config
  cacheTag('categories');                                 // invalidate with revalidateTag or updateTag
  const categories = await getCategories();
  return <ul>{categories.map((c) => <li key={c.id}>{c.name}</li>)}</ul>;
}
```

- `"use cache"` works at the top of a file, a component or an async function. The cache key includes the build id, the function and its serializable arguments (and captured closure variables).
- Cached scopes cannot call `cookies()`, `headers()` or read `searchParams`: read them outside and pass the values in as arguments.
- Everything uncached runs at request time; anything that needs request data must sit under a `<Suspense>` boundary so the static shell can still prerender (partial prerendering).
- Without `cacheLife`, a default profile applies (5 minutes stale on the client, 15 minutes revalidation on the server, no time-based expiry).
- The default cache handler is in memory per instance, so serverless instances may not share entries; remote cache handlers exist for that.
- With Cache Components on, the `dynamic`, `revalidate` and `fetchCache` segment options are not available.

## 6. Revalidation

```ts
export const revalidate = 600;                   // page-level ISR (previous model): regenerate at most every 10 minutes

revalidatePath('/marketplace');                  // one path
revalidatePath('/marketplace/[slug]', 'page');   // every page of a dynamic route
revalidatePath('/', 'layout');                   // everything under the root layout
revalidateTag('listings', 'max');                // Next.js 16: stale-while-revalidate for tagged entries
updateTag('listings');                           // Next.js 16, Server Actions only: expire now and read fresh data
```

In Next.js 15, `revalidateTag('listings')` takes only the tag; Next.js 16 deprecated that form in favor of the two-argument version.

On-demand revalidation from another system, for example a Rails app or CMS that changes data the Next.js site displays:

```ts
// app/api/revalidate/route.ts
import { timingSafeEqual } from 'node:crypto';
import { revalidateTag } from 'next/cache';
import { z } from 'zod';

const Body = z.object({ tag: z.enum(['listings', 'categories']) });
const sameSecret = (a: string, b: string) => a.length === b.length && timingSafeEqual(Buffer.from(a), Buffer.from(b));

export async function POST(request: Request) {
  const secret = request.headers.get('x-revalidate-secret') ?? '';
  if (!sameSecret(secret, process.env.REVALIDATE_SECRET ?? '')) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }
  const parsed = Body.safeParse(await request.json().catch(() => null));
  if (!parsed.success) return Response.json({ error: 'Invalid body' }, { status: 400 });
  revalidateTag(parsed.data.tag, 'max');         // Next.js 15: revalidateTag(parsed.data.tag)
  return Response.json({ revalidated: true });
}
```

On the client, `router.refresh()` re-requests the current route's Server Components without losing client state.

## 7. Server Actions

```ts
// app/marketplace/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { requireUser } from '@/lib/auth';
import { connectToDatabase } from '@/lib/mongodb';
import { Listing } from '@/models/listing';

const CreateListing = z.object({
  title: z.string().trim().min(3, 'Title is too short').max(120),
  priceCents: z.coerce.number().int().positive(),
});

export type CreateListingState = { errors?: Record<string, string[] | undefined>; message?: string };

export async function createListing(_prev: CreateListingState, formData: FormData): Promise<CreateListingState> {
  const user = await requireUser();                                // authenticate inside the action
  const parsed = CreateListing.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { errors: parsed.error.flatten().fieldErrors }; // Zod 4 also offers z.flattenError
  await connectToDatabase();
  const doc = await Listing.create({ ...parsed.data, orgId: user.orgId, ownerId: user.userId }); // org from the session
  revalidatePath('/marketplace');
  redirect(`/marketplace/${doc.id}`);                              // throws a special error: outside try/catch
}
```

```tsx
// app/marketplace/new/create-listing-form.tsx
'use client';

import { useActionState } from 'react';
import { createListing, type CreateListingState } from '../actions';

export function CreateListingForm() {
  const [state, formAction, isPending] = useActionState<CreateListingState, FormData>(createListing, {});
  return (
    <form action={formAction}>
      <input name="title" required />
      {state.errors?.title && <p role="alert">{state.errors.title[0]}</p>}
      <input name="priceCents" type="number" required />
      <button disabled={isPending}>{isPending ? 'Saving...' : 'Create listing'}</button>
    </form>
  );
}
```

- How it works: the build gives each action an id; calling it sends a POST with the serialized arguments; Next.js runs it, applies revalidation, and returns the result plus the updated RSC payload in one round trip.
- A `<form action={serverAction}>` rendered by a Server Component submits as a normal HTML form before hydration: progressive enhancement.
- Pass extra arguments with `bind`: `<form action={updateListing.bind(null, listing.id)}>` (bound values are still client-controlled input).
- Call actions from event handlers too (`onClick={() => startTransition(() => toggleSaved(id))}`), and pair them with `useOptimistic` for instant feedback. The React side of these hooks is in [React 19](../06-react/09-react-19-and-server-components.md) and [forms](../06-react/05-forms-and-validation.md).

## 8. Securing Server Actions

Every exported action is reachable by a direct POST, whether or not your UI shows the button.

- **Validate** every argument with Zod; TypeScript types are not enforced at runtime, and hidden form fields are user input.
- **Authenticate inside the action.** A redirect in the page that renders the form protects the UI, not the action.
- **Authorize the specific resource**: the listing must belong to the caller's org. Derive `orgId` and `userId` from the session, never from form data.
- **Return minimal data**: a success flag or a DTO, never a raw database document.
- **Rate limit** expensive actions (emails, AI calls, payments).
- Built-in protections help but do not replace the above: encrypted, non-deterministic action ids; unused actions removed from the client bundle; closed-over variables encrypted; actions accept only POST and compare `Origin` with `Host` (or `X-Forwarded-Host`), configurable through `serverActions.allowedOrigins`; request bodies capped at 1 MB by default (`serverActions.bodySizeLimit`).

```ts
'use server';
// imports as in actions.ts above, plus: import { isValidObjectId } from 'mongoose';

export async function deleteListing(listingId: string) {
  const user = await requireUser();
  if (!isValidObjectId(listingId)) return { ok: false as const, error: 'Invalid id' };
  await connectToDatabase();
  const result = await Listing.deleteOne({ _id: listingId, orgId: user.orgId }); // scoped: no IDOR
  if (result.deletedCount === 0) return { ok: false as const, error: 'Not found' };
  revalidatePath('/marketplace');
  return { ok: true as const };
}
```

## 9. Streaming with Suspense and loading.tsx

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <>
      <h1>Dashboard</h1>                             {/* sent immediately */}
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />                                    {/* async Server Component: streams in when ready */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />                             {/* independent: does not wait for Stats */}
      </Suspense>
    </>
  );
}
```

`loading.tsx` is a Suspense boundary around the whole page; explicit `<Suspense>` boundaries are more granular. To start a fetch on the server but read it in a Client Component, pass the unawaited promise as a prop and unwrap it with `use()`:

```tsx
// Server Component
const reviewsPromise = getReviews(listingId);       // not awaited
return <Suspense fallback={<p>Loading reviews...</p>}><Reviews reviewsPromise={reviewsPromise} /></Suspense>;

// Client Component
'use client';
import { use } from 'react';
export function Reviews({ reviewsPromise }: { reviewsPromise: Promise<{ id: string; text: string }[]> }) {
  const reviews = use(reviewsPromise);
  return <ul>{reviews.map((r) => <li key={r.id}>{r.text}</li>)}</ul>;
}
```

## 10. Avoiding waterfalls

```tsx
// Waterfall: reviews wait for the listing although they do not depend on it
const listing = await getListing(id);
const reviews = await getReviews(id);

// Parallel: total time is the slower call, not the sum
const [listing, reviews] = await Promise.all([getListing(id), getReviews(id)]);
```

- **Component waterfalls:** a parent awaits, renders a child, and only then does the child start its own fetch. Start fetches higher up, or give each slow child its own Suspense boundary so they load in parallel.
- **Preloading:** with the fetcher wrapped in React `cache()`, call `void getListing(id)` early (before an unrelated await); the later call reuses the in-flight promise.
- **Dependent data** (needs the listing's owner id) is legitimately sequential; stream it behind Suspense so it does not block the rest of the page.

## 11. When TanStack Query still belongs in a Next.js app

| Need | Server Components | TanStack Query in Client Components |
|---|---|---|
| Initial page data, SEO content | best fit | possible with prefetch and hydration |
| Infinite scroll, polling, live updates | awkward | best fit (`useInfiniteQuery`, `refetchInterval`) |
| Client-side filters without navigation | needs URL changes and refetches | best fit |
| Optimistic updates with rollback | `useOptimistic` covers simple cases | full control (`onMutate`, rollback) |
| Data from a separate API (Rails, an oRPC service) | fetch on the server | client cache with background refetch |
| Code shared with a React Native app | not shareable | hooks shared across web and mobile |

A common pattern: prefetch on the server, dehydrate, and hydrate into `useQuery` on the client ([TanStack Query SSR](../06-react/04-data-fetching-tanstack-query.md)).

## 12. Error handling

| Kind | Example | Handling |
|---|---|---|
| Expected, form-level | validation failed, duplicate title | return it as action state and render it next to the field |
| Expected, resource-level | listing does not exist | `notFound()`, rendering `not-found.tsx` |
| Unexpected | database down, a bug | throw; the nearest `error.tsx` renders with `reset`; log on the server |
| Root layout errors | provider crashes | `global-error.tsx` |

```ts
import { notFound, unstable_rethrow } from 'next/navigation';

export async function loadListing(id: string) {
  try {
    const listing = await getListing(id);
    if (!listing) notFound();
    return listing;
  } catch (err) {
    unstable_rethrow(err);   // re-throws Next.js's own notFound/redirect signals instead of swallowing them
    console.error(err);
    throw new Error('Could not load the listing');
  }
}
```

## 13. Debugging "my page shows stale data"

1. Check the build output: is the route static? Then its data dates from build time or the last revalidation.
2. Check the fetch or function: `force-cache`, `revalidate`, `unstable_cache` or `"use cache"` keep data until revalidated.
3. Check the mutation: does it call `revalidatePath` for the exact route (dynamic routes need the `'page'` type) or the matching tag?
4. Check the Router Cache: soft navigations and back/forward reuse payloads; revalidating inside the Server Action or `router.refresh()` updates them.
5. Check external caches: a CDN or proxy caching HTML because of your `Cache-Control` headers.
6. Self-hosted with several instances: each instance has its own cache unless you configure a shared handler.
7. Reproduce with `next build && next start`: in development, pages render on every request and hide caching behavior.

## Common pitfalls
- Assuming `fetch` is cached in Next.js 15+, or assuming it is not when the route is static and prerendered.
- Wrapping per-user data in shared caches (`force-cache`, `unstable_cache`, `"use cache"`) without the user or org in the key.
- Forgetting to revalidate after a mutation, or revalidating the wrong path.
- Authorizing only at the page level and not inside the Server Action.
- Trusting `orgId` or `userId` from form data.
- Returning raw database documents from actions or passing them to Client Components.
- Calling `redirect()` or `notFound()` inside try/catch without rethrowing.
- Sequential awaits for independent data, and Suspense boundaries placed so high that the whole page waits.
- Fetching your own Route Handlers from Server Components.
- Testing caching only in development mode.

## How to talk about this in an interview
- Use RemoteDev as the anchor: explain how pages got their data (Server Components calling the database, or client fetches to your Next.js APIs) and how writes refreshed the UI.
  > **Fill in:** whether you used Server Components, Route Handlers, Server Actions or client fetching at RemoteDev, and one caching or stale-data issue you debugged.
- Lead with the version distinction: "In Next.js 15 fetches are not cached by default, but static routes are still prerendered. Next.js 16 adds Cache Components, where caching is explicit with `"use cache"`." That separates you from candidates repeating Next.js 13 advice.
- On security, sound like someone who has reviewed code: "Each Server Action is a public POST endpoint, so I validate with Zod, check the session and the resource's org inside the action, and return a minimal result."
- For the Devntech comparison, TanStack Start uses loaders plus TanStack Query instead of framework caching layers: [TanStack Start](05-tanstack-start.md).

## Interview questions

**Q1. How do you fetch data in the App Router?**
In async Server Components: `await` a `fetch` or call the database through a server-only data access layer that checks authorization and returns DTOs. Client Components fetch only for interactive, client-driven data, often with TanStack Query.
*Follow-up:* Why not call your own API route from a Server Component? It adds an HTTP hop to the same server; call the data function directly.

**Q2. Explain the Next.js caching layers.**
In the previous model: request memoization dedupes identical fetches within one render; the Data Cache stores opted-in fetch results across requests; the Full Route Cache stores HTML and RSC payloads of static routes; the Router Cache keeps RSC payloads in the browser for navigation. Next.js 16's Cache Components replace implicit caching with explicit `"use cache"`.
*Follow-up:* Which of these lives in the browser? Only the Router Cache.

**Q3. What changed about caching in Next.js 15, and why?**
`fetch` requests, GET Route Handlers and client page navigations stopped being cached by default, because implicit caching surprised developers with stale data. You opt in with `cache: 'force-cache'`, `next.revalidate`, `unstable_cache` or segment config.
*Follow-up:* Can a Next.js 15 page still show build-time data? Yes, if the route is static: its fetches ran during prerendering.

**Q4. What makes a route dynamic?**
Reading `cookies()`, `headers()` or `searchParams`, calling `connection()`, uncached data fetched after a request API, or forcing it with `dynamic = 'force-dynamic'` in the previous model. Otherwise it is prerendered as static.
*Follow-up:* How do you check? The `next build` output marks each route as static or dynamic.

**Q5. Time-based vs on-demand revalidation?**
Time-based regenerates cached data at most every N seconds (`next.revalidate`, `export const revalidate`, `cacheLife`), trading freshness for simplicity. On-demand invalidates exactly when data changes, with `revalidatePath`, `revalidateTag` or `updateTag` called from a Server Action, a Route Handler or a webhook.
*Follow-up:* Typical combination? A long revalidate period as a safety net plus on-demand invalidation on writes.

**Q6. `revalidatePath` vs `revalidateTag`?**
`revalidatePath` invalidates by route (a path, or every page of a dynamic route with the `'page'` type). `revalidateTag` invalidates every cached entry tagged with that name across routes, which suits data shown in many places. Next.js 16 adds a cache-life profile argument to `revalidateTag` and `updateTag` for read-your-writes.
*Follow-up:* Which for "a listing changed, and it appears on the home page, category pages and its own page"? Tag it and revalidate the tag.

**Q7. What are Server Actions, and how do they work under the hood?**
Async functions marked `'use server'` that run on the server but can be called from forms or client code. The build assigns each an id; a call becomes a POST with serialized arguments; Next.js runs the function and returns its result plus refreshed RSC payload in one round trip.
*Follow-up:* What can arguments and return values be? Serializable values only: plain objects, arrays, primitives, dates, FormData.

**Q8. How do you secure Server Actions?**
Treat each as a public API endpoint: validate inputs with Zod, authenticate and authorize inside the action (page checks do not protect it), derive org and user from the session, scope queries to them, return minimal data and rate-limit expensive actions. Built-ins (unguessable ids, encrypted closures, Origin checks, body limits) are extra layers, not substitutes.
*Follow-up:* Are hidden form fields safe for ids? They are user input; verify ownership server-side.

**Q9. How do you show validation errors and pending state with Server Actions?**
Wrap the action with `useActionState` in a Client Component: the action receives the previous state and FormData and returns new state (field errors), and the hook exposes `[state, formAction, isPending]`. `useFormStatus` gives pending state inside a submit button component.
*Follow-up:* Old name for `useActionState`? `useFormState` from `react-dom`, which was replaced.

**Q10. How does streaming work in Next.js?**
The server sends the page shell immediately with fallbacks for pending Suspense boundaries, then streams each boundary's HTML as its data resolves; React swaps it in and hydrates selectively. `loading.tsx` adds a boundary around a page; explicit `<Suspense>` gives finer control.
*Follow-up:* What does streaming do for SEO? Content still arrives in the HTML response, just in chunks.

**Q11. How do you avoid data-fetching waterfalls?**
Start independent requests together with `Promise.all`, preload with `cache()`-wrapped functions, give slow independent sections their own Suspense boundaries, and avoid parent-then-child sequential fetching when the child does not depend on the parent's data.
*Follow-up:* When is sequential correct? When the second request needs the first one's result.

**Q12. How do you cache a database query rather than a fetch?**
React `cache()` deduplicates within one request only. Across requests, use `unstable_cache` with key parts, tags and revalidate in the previous model, or `"use cache"` with `cacheLife` and `cacheTag` under Cache Components.
*Follow-up:* What must be in the cache key for tenant data? The org (and user, if per user), passed as arguments.

**Q13. When would you still use TanStack Query in a Next.js app?**
For client-driven data: infinite scrolling, polling or live updates, client-side filters, optimistic updates with rollback, data from a separate backend, or hooks shared with a React Native app. Initial data can be prefetched on the server and hydrated.
*Follow-up:* Does it replace Server Components? No: it complements them for interactive parts.

**Q14. A user updates a listing, but the page still shows the old title. How do you debug it?**
Check whether the route is static, whether the data source is cached (`force-cache`, `unstable_cache`, `"use cache"`), whether the mutation revalidated the right path or tag, whether the client Router Cache served an old payload, whether a CDN caches the HTML, and whether several instances have separate caches. Reproduce with a production build.
*Follow-up:* Quick fix in the action? Revalidate the correct path or tag, or call `updateTag` in Next.js 16.

**Q15. What are Cache Components and `"use cache"` in Next.js 16?**
An opt-in model (`cacheComponents: true`) where nothing is cached implicitly. `"use cache"` marks files, components or functions as cacheable, with keys derived from their arguments, `cacheLife` setting lifetime and `cacheTag` enabling invalidation. Uncached request-time parts render under Suspense while the static shell is prerendered (partial prerendering).
*Follow-up:* Can a cached function read cookies? No: read them outside and pass values as arguments.

**Q16. How do you handle errors in Server Components and Server Actions?**
Return expected errors as values (field errors in action state), call `notFound()` for missing resources, and throw unexpected errors so `error.tsx` renders, logging them on the server. Do not swallow `redirect()` or `notFound()` in try/catch; rethrow them with `unstable_rethrow`.
*Follow-up:* What do users see for a server error in production? A generic message and a digest that maps to server logs.

## Further reading
- [Next.js: Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data)
- [Next.js: Caching (Cache Components)](https://nextjs.org/docs/app/getting-started/caching)
- [Next.js: Caching and revalidating (previous model)](https://nextjs.org/docs/app/guides/caching-without-cache-components)
- [Next.js: Mutating data with Server Actions](https://nextjs.org/docs/app/getting-started/mutating-data)
- [Next.js: Data security](https://nextjs.org/docs/app/guides/data-security)
