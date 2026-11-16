# Data Fetching and TanStack Query

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [State management](03-state-management.md), [Forms](05-forms-and-validation.md), [Routing](08-routing.md), [Testing](10-testing.md), [Next.js data fetching](../07-nextjs-and-tanstack/02-nextjs-data-fetching-and-caching.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md)
> **On your resume:** Fetching and caching server data sits behind every screen you build on the Colochousing REST API (web and the Coloc resident app), on the oRPC-based TanStack Start platform at Devntech, in the DRF-backed LMS at PureLogics and against the Next.js APIs at RemoteDev.

## Quick revision
- Fetching in `useEffect` needs loading and error state, an `AbortController` (or ignore flag) against race conditions, and a `res.ok` check, because `fetch` rejects only on network errors.
- Hand-rolled fetching does not scale: no cache, duplicate requests, no retries, no background refresh, waterfalls, stale data after mutations.
- TanStack Query is a server-state cache: `useQuery({ queryKey, queryFn })`. The key is the cache identity and must include every variable the `queryFn` uses.
- Design keys from general to specific (`['bills', 'list', { houseId }]`) and centralize them in key factories or `queryOptions` helpers.
- `staleTime` (default 0): how long data counts as fresh, so it is not refetched. `gcTime` (default 5 minutes): how long an unused query stays in memory.
- Stale queries refetch when a component mounts them, when the window regains focus, on reconnect, or on an interval you set.
- `isPending`: no data yet. `isFetching`: a request in flight, including background refetches. `isLoading` = `isPending && isFetching`.
- Dependent queries: `enabled: !!userId`. Pagination: page in the key plus `placeholderData: keepPreviousData`. Infinite lists: `useInfiniteQuery` with `initialPageParam` and `getNextPageParam`.
- Mutations: `useMutation({ mutationFn, onSuccess })`, then `invalidateQueries({ queryKey })` (prefix match) or `setQueryData` with the response.
- Optimistic update: `onMutate` (cancel, snapshot, write) -> `onError` (roll back) -> `onSettled` (invalidate).
- Prefetch with `prefetchQuery` on hover, or `ensureQueryData` in route loaders.
- `useSuspenseQuery`: `data` is always defined; loading goes to `<Suspense>`, errors to an error boundary.
- Retries: 3 with exponential backoff on the client by default; skip retries for 4xx.
- SSR: prefetch on the server, `dehydrate`, wrap the client in `HydrationBoundary`; one `QueryClient` per request.
- The same hooks work in React Native; wire `focusManager` and `onlineManager` to `AppState` and NetInfo.

## 1. Fetching in useEffect, done correctly

```tsx
type User = { id: number; name: string };

function UserCard({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    setUser(null);
    setError(null);

    async function load() {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch does not reject on 404 or 500
        setUser((await res.json()) as User);
      } catch (err) {
        if (controller.signal.aborted) return; // userId changed or component unmounted
        setError(err instanceof Error ? err.message : 'Unknown error');
      }
    }
    load();

    return () => controller.abort(); // cancel the request for the previous userId
  }, [userId]);

  if (error) return <p role="alert">{error}</p>;
  if (!user) return <p>Loading...</p>;
  return <h2>{user.name}</h2>;
}
```

**The race condition:** the user opens profile 1, then quickly profile 2. Both requests are in flight; if the response for 1 arrives last, the screen shows user 1 under the URL for user 2. Aborting in the cleanup (or ignoring stale responses) guarantees only the latest request can update state. In development, StrictMode's extra mount aborts the first request immediately, which is expected.

## 2. Why hand-rolled fetching does not scale

- **No cache:** navigating back refetches and shows the spinner again.
- **No deduplication:** three components that need the current user send three requests.
- **No retries or backoff** for flaky mobile networks.
- **No background refresh:** data stays stale until a reload; nothing refetches on focus or reconnect.
- **Waterfalls:** a parent fetches, renders, and only then a child starts its own fetch; each level adds a round trip.
- **Mutations:** after creating a bill you must remember every list and counter that shows bills.
- **Boilerplate:** loading, error, pagination, cancellation and SSR handoff, repeated per screen.

A server-state library solves these once. TanStack Query (formerly React Query) is the most common choice; SWR and RTK Query are alternatives, and router loaders (React Router, TanStack Router, Next.js) solve part of the problem at the route level.

## 3. Setup

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000 }, // treat data as fresh for 30 s app-wide
  },
});

createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <App />
    <ReactQueryDevtools initialIsOpen={false} /> {/* left out of production builds by default */}
  </QueryClientProvider>,
);
```

Create the client once, outside components: `new QueryClient()` in a component body creates a new, empty cache on every render. In SSR frameworks use `const [queryClient] = useState(() => new QueryClient())` inside a client-side provider component.

## 4. useQuery and its status flags

```tsx
import { useQuery } from '@tanstack/react-query';

export async function fetchJson<T>(url: string, signal?: AbortSignal): Promise<T> {
  const res = await fetch(url, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`); // must throw, or the error page is cached as data
  return res.json() as Promise<T>;
}

function Bills({ houseId }: { houseId: string }) {
  const { data, isPending, isError, error, isFetching, refetch } = useQuery({
    queryKey: ['bills', 'list', { houseId }],
    queryFn: ({ signal }) => fetchJson<Bill[]>(`/api/houses/${houseId}/bills`, signal),
  });

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage message={error.message} onRetry={() => refetch()} />;

  return (
    <>
      {isFetching && <small>Refreshing...</small>} {/* background refetch: keep showing data */}
      <BillList bills={data} />
    </>
  );
}
```

The `queryFn` receives a context with the `queryKey` and an abort `signal`; pass the signal to `fetch` so requests are cancelled when no component needs them anymore.

| Flag | Meaning |
|---|---|
| `isPending` (`status: 'pending'`) | no data in the cache yet (first load, or a disabled query with no data) |
| `isError` (`status: 'error'`) | the last fetch failed after retries; `error` holds it |
| `isSuccess` (`status: 'success'`) | `data` is available |
| `isFetching` (`fetchStatus: 'fetching'`) | the query function is running, initial or background |
| `fetchStatus: 'paused'` | it wants to fetch, but the device is offline |
| `isLoading` | `isPending && isFetching`: the first load is actually in progress |
| `isPlaceholderData` | showing `placeholderData`, for example the previous page |

`status` answers "do I have data?", `fetchStatus` answers "is the query function running?". Version note: v4 called the first state `isLoading` and the memory option `cacheTime`; v5 renamed them to `isPending` and `gcTime` and accepts only a single options object.

### Going deeper: structural sharing, tracked properties and select

After a refetch, TanStack Query compares the new JSON with the old and keeps the old references for parts that did not change (structural sharing, on by default), so memoized children and effects that depend on unchanged objects do not re-run. Each `useQuery` also tracks which result properties the component reads and re-renders only when one of those changes. The `select` option derives or narrows the data (`select: (bills) => bills.filter((b) => !b.paid)`), and the component re-renders only when the selected result changes.

## 5. Query keys

Keys are arrays, hashed deterministically (object key order does not matter, array order does). A key is both the cache identity and the dependency list of the query: when it changes, TanStack Query reads or fetches the data for the new key.

- Include every variable the `queryFn` uses (ids, filters, page); a missing one shows wrong data. `@tanstack/eslint-plugin-query` checks this.
- Go from general to specific so prefix invalidation works: `['bills']` > `['bills', 'list', filters]` > `['bills', 'detail', id]`.
- Keep keys serializable: strings, numbers, plain objects.

```ts
import { queryOptions } from '@tanstack/react-query';

type BillFilters = { houseId: string; status?: 'paid' | 'unpaid' };

export const billKeys = {
  all: ['bills'] as const,
  lists: () => [...billKeys.all, 'list'] as const,
  list: (filters: BillFilters) => [...billKeys.lists(), filters] as const,
  details: () => [...billKeys.all, 'detail'] as const,
  detail: (id: string) => [...billKeys.details(), id] as const,
};

// queryOptions bundles key, function and options with full type inference
export const billQueries = {
  list: (filters: BillFilters) =>
    queryOptions({
      queryKey: billKeys.list(filters),
      queryFn: ({ signal }) => fetchJson<Bill[]>(`/api/houses/${filters.houseId}/bills`, signal),
    }),
  detail: (id: string) =>
    queryOptions({
      queryKey: billKeys.detail(id),
      queryFn: ({ signal }) => fetchJson<Bill>(`/api/bills/${id}`, signal),
      staleTime: 60_000,
    }),
};

// useQuery(billQueries.detail(id)); queryClient.prefetchQuery(billQueries.detail(id));
// queryClient.getQueryData(billQueries.detail(id).queryKey) is typed as Bill | undefined
// Invalidate every bill list, whatever the filters: invalidateQueries({ queryKey: billKeys.lists() })
```

## 6. staleTime vs gcTime, and refetch triggers

```text
fetch succeeds -> FRESH (for staleTime) -> STALE (still shown, refetched on triggers)
last observer unmounts -> INACTIVE (kept in the cache) -> garbage collected after gcTime
```

| Option | Default | Meaning |
|---|---|---|
| `staleTime` | `0` | how long data is fresh; fresh data is served from cache without refetching |
| `gcTime` | 5 minutes | how long an inactive query (no component using it) stays in memory |
| `refetchOnMount` | `true` | refetch stale data when a new component subscribes |
| `refetchOnWindowFocus` | `true` | refetch stale data when the tab regains focus |
| `refetchOnReconnect` | `true` | refetch stale data when the network comes back |
| `refetchInterval` | off | poll every N milliseconds |
| `retry` | 3 on the client, 0 on the server | retries with exponential backoff before the query errors |

- They are independent: `staleTime` controls refetching, `gcTime` controls memory. Stale data is still displayed instantly while a background refetch runs (stale-while-revalidate).
- With the default `staleTime: 0`, every mount and focus refetches. Choose per resource: reference data such as countries or plans `Infinity`, a bill list perhaps 30 to 60 seconds, live data a low value plus polling.
- A longer `gcTime` keeps data around for back navigation.

## 7. Dependent and parallel queries

```tsx
const { data: user } = useQuery({
  queryKey: ['user', email],
  queryFn: () => getUserByEmail(email),
});

const userId = user?.id;
const { data: projects } = useQuery({
  queryKey: ['projects', userId],
  queryFn: () => getProjectsByUser(userId!), // only runs while enabled
  enabled: !!userId,                          // waits until the first query provides the id
});
```

- A disabled query with no data reports `isPending: true` with `fetchStatus: 'idle'`, so base spinners on `isLoading` or on the condition itself. Recent v5 versions also offer `skipToken` as a type-safe alternative to `enabled` (check the docs).
- Separate `useQuery` calls in one component run in parallel. For a dynamic number of queries use `useQueries({ queries: ids.map((id) => billQueries.detail(id)) })`.
- Dependent queries are a deliberate waterfall; avoid them when one endpoint can return both pieces.

## 8. Pagination and infinite queries

```tsx
import { keepPreviousData, useQuery } from '@tanstack/react-query';

function BillPages({ houseId }: { houseId: string }) {
  const [page, setPage] = useState(1);
  const { data, isPlaceholderData } = useQuery({
    queryKey: ['bills', 'list', { houseId, page }],
    queryFn: () => fetchJson<{ items: Bill[]; hasMore: boolean }>(`/api/houses/${houseId}/bills?page=${page}`),
    placeholderData: keepPreviousData, // keep page 1 on screen while page 2 loads: no spinner flash
  });

  return (
    <>
      <BillList bills={data?.items ?? []} />
      <button onClick={() => setPage((p) => p - 1)} disabled={page === 1}>Previous</button>
      <button onClick={() => setPage((p) => p + 1)} disabled={isPlaceholderData || !data?.hasMore}>Next</button>
    </>
  );
}
```

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

type PostPage = { items: Post[]; nextCursor: number | null };

function Feed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isPending } = useInfiniteQuery({
    queryKey: ['posts', 'feed'],
    queryFn: ({ pageParam, signal }) => fetchJson<PostPage>(`/api/posts?cursor=${pageParam}`, signal),
    initialPageParam: 0,                                  // required in v5
    getNextPageParam: (lastPage) => lastPage.nextCursor, // null or undefined: no more pages
  });

  if (isPending) return <Spinner />;
  const posts = data?.pages.flatMap((p) => p.items) ?? [];

  return (
    <>
      {posts.map((p) => <PostCard key={p.id} post={p} />)}
      <button onClick={() => fetchNextPage()} disabled={!hasNextPage || isFetchingNextPage}>
        {isFetchingNextPage ? 'Loading...' : hasNextPage ? 'Load more' : 'No more posts'}
      </button>
    </>
  );
}
```

`data.pages` holds one entry per fetched page and `data.pageParams` their params. For infinite scroll, call `fetchNextPage` from an `IntersectionObserver` (see the live-coding exercise in the [question bank](99-interview-questions.md)) and virtualize very long lists ([performance](06-performance.md)). Cursor pagination is safer than offsets when items are inserted while the user scrolls.

## 9. Mutations and cache updates

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function AddBillButton({ houseId }: { houseId: string }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: async (input: { title: string; amountCents: number }) => {
      const res = await fetch(`/api/houses/${houseId}/bills`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(input),
      });
      if (!res.ok) throw new Error('Could not create the bill');
      return (await res.json()) as Bill;
    },
    onSuccess: (created) => {
      queryClient.setQueryData(billKeys.detail(created.id), created); // seed the detail cache, no request
      return queryClient.invalidateQueries({ queryKey: billKeys.lists() }); // lists refetch; awaited
    },
  });

  return (
    <button disabled={mutation.isPending} onClick={() => mutation.mutate({ title: 'Internet', amountCents: 3000 })}>
      {mutation.isPending ? 'Saving...' : 'Add bill'}
    </button>
  );
}
```

- `invalidateQueries` marks matching queries stale (prefix match; `exact: true` for one key) and refetches the active ones; inactive ones refetch on next use. Returning its promise from `onSuccess` keeps the mutation pending until the refetch finishes.
- `setQueryData(key, dataOrUpdater)` writes to the cache directly; updaters must return new objects, never mutate.
- Prefer invalidation for correctness (the server computes totals, ordering and permissions); use `setQueryData` when the response contains exactly what the cache needs.
- `mutate` reports errors through state and callbacks; `mutateAsync` returns a promise you must `try/catch`. Mutation state: `isPending`, `isError`, `isSuccess`, `error`, `variables`, `reset()`.
- Callbacks passed to `mutate(vars, { onSuccess })` run after those in `useMutation`, and not at all if the component unmounted. Keep cache logic in `useMutation`, UI reactions (close a modal, toast) in `mutate`.

## 10. Optimistic updates with rollback

```tsx
function useToggleBillPaid(filters: BillFilters) {
  const queryClient = useQueryClient();
  const listKey = billKeys.list(filters);

  return useMutation({
    mutationFn: (bill: Bill) => patchBill(bill.id, { paid: !bill.paid }),
    onMutate: async (bill) => {
      await queryClient.cancelQueries({ queryKey: listKey }); // an in-flight refetch must not overwrite us
      const previous = queryClient.getQueryData<Bill[]>(listKey); // snapshot for rollback
      queryClient.setQueryData<Bill[]>(listKey, (old) =>
        old?.map((b) => (b.id === bill.id ? { ...b, paid: !b.paid } : b)),
      );
      return { previous }; // handed to onError and onSettled
    },
    onError: (_error, _bill, context) => {
      queryClient.setQueryData(listKey, context?.previous); // roll back
    },
    onSettled: () => queryClient.invalidateQueries({ queryKey: listKey }), // resync with the server
  });
}
```

1. `onMutate` runs before the request: cancel outgoing refetches for the key, snapshot the data, write the optimistic value, return the snapshot.
2. `onError` restores the snapshot and tells the user.
3. `onSettled` runs after success or failure and invalidates, so the cache ends up matching the server.

Version note: recent v5 releases pass an extra context argument to these callbacks and the docs rename the value returned from `onMutate`; the three-argument pattern above still works, but check the docs for your version. For a UI-only optimistic display, render `mutation.variables` as a pending row while `mutation.isPending`: no cache writes, nothing to roll back. React 19's `useOptimistic` covers the same need for Actions ([React 19](09-react-19-and-server-components.md)).

## 11. Prefetching and useSuspenseQuery

```tsx
function BillLink({ id }: { id: string }) {
  const queryClient = useQueryClient();
  const prefetch = () => queryClient.prefetchQuery(billQueries.detail(id)); // warm the cache
  return <Link to={`/bills/${id}`} onMouseEnter={prefetch} onFocus={prefetch}>Open bill</Link>;
}
```

- `prefetchQuery` fetches only if the data is missing or stale, and never throws.
- In route loaders, `await queryClient.ensureQueryData(billQueries.detail(id))` returns cached data or fetches it; the page's `useQuery` with the same options then reads the cache. This "render as you fetch" pattern removes waterfalls ([routing](08-routing.md), [TanStack Router](../07-nextjs-and-tanstack/04-tanstack-router.md)).

```tsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';
import { useSuspenseQuery } from '@tanstack/react-query';

function BillPage({ id }: { id: string }) {
  return (
    <ErrorBoundary fallback={<p role="alert">Could not load the bill.</p>}>
      <Suspense fallback={<Spinner />}>
        <BillDetails id={id} />
      </Suspense>
    </ErrorBoundary>
  );
}

function BillDetails({ id }: { id: string }) {
  const { data } = useSuspenseQuery(billQueries.detail(id)); // data is Bill, never undefined
  return <h1>{data.title}</h1>;
}
```

With `useSuspenseQuery` there are no pending branches: the nearest `<Suspense>` shows the fallback and errors go to the nearest error boundary. It has no `enabled` option, and several suspense queries in one component run one after another, so prefetch or use `useSuspenseQueries`. Reset errored queries together with the boundary through `QueryErrorResetBoundary` or `useQueryErrorResetBoundary`.

## 12. Errors and retries

```ts
import { QueryCache, QueryClient } from '@tanstack/react-query';

class HttpError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        if (error instanceof HttpError && error.status >= 400 && error.status < 500) return false; // 4xx will not heal
        return failureCount < 3;
      },
    },
  },
  queryCache: new QueryCache({
    onError: (error) => showToast(error.message), // global, once per failed query; any toast library
  }),
});
```

- Defaults: 3 retries with exponential backoff (`retryDelay`) on the client, none on the server. During retries the query stays pending and `failureCount` grows.
- Show errors inline with a retry button (`refetch`), or opt into error boundaries with `throwOnError: true` (renamed from `useErrorBoundary` in v5).
- A failed background refetch keeps the previous `data`, so show stale data with a small warning instead of wiping the screen.
- Handle 401 centrally (refresh the session or redirect to login) in your fetch wrapper or in `QueryCache` `onError`.

## 13. SSR and hydration (overview)

On the server, create a new `QueryClient` per request (never share one between users), prefetch what the page needs, and serialize the cache with `dehydrate`. On the client, `HydrationBoundary` merges that state into the browser cache, so `useQuery` with the same keys has data on the first render.

```tsx
// Next.js App Router sketch: a Server Component prefetches, a Client Component calls useQuery
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

export default async function BillsPage() {
  const queryClient = new QueryClient();
  await queryClient.prefetchQuery({ queryKey: ['bills', 'list'], queryFn: getBills });
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <BillsClient /> {/* 'use client' component using useQuery({ queryKey: ['bills', 'list'], ... }) */}
    </HydrationBoundary>
  );
}
```

Give client queries a `staleTime` above 0 so hydrated data is not refetched immediately. TanStack Start and TanStack Router integrate the query client with route loaders and streaming SSR; their APIs moved quickly, so check the current docs. Details: [Next.js data fetching](../07-nextjs-and-tanstack/02-nextjs-data-fetching-and-caching.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md).

## 14. React Native, REST and oRPC

**React Native.** The same `@tanstack/react-query` hooks work unchanged. There is no browser focus or online event, so connect them yourself:

```ts
import { useEffect } from 'react';
import { AppState, Platform, type AppStateStatus } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { focusManager, onlineManager } from '@tanstack/react-query';

onlineManager.setEventListener((setOnline) =>
  NetInfo.addEventListener((state) => setOnline(!!state.isConnected)), // returns the unsubscribe function
);

export function useAppFocusRefetch() {
  useEffect(() => {
    const sub = AppState.addEventListener('change', (status: AppStateStatus) => {
      if (Platform.OS !== 'web') focusManager.setFocused(status === 'active'); // refetch stale data on resume
    });
    return () => sub.remove();
  }, []);
}
```

Persisting the cache for offline starts uses TanStack's persist-client plugins with an AsyncStorage persister (check the docs for package names). More in [RN state, storage and offline](../08-react-native-expo/04-state-storage-auth-offline.md).

**REST.** Wrap the API client per resource in `queryOptions` factories and mutation hooks (`useBills`, `useCreateBill`), so screens never build URLs or keys. With a Rails or DRF API, keys mirror resources and params: `['bills', 'list', { houseId, page }]`.

**oRPC.** oRPC ships a TanStack Query integration that derives `queryOptions`, `mutationOptions` and keys from your typed router client, so inputs and outputs are inferred end to end. Roughly (package and helper names vary by version, so check the oRPC docs):

```tsx
// Illustrative: `orpc` is the utilities object created from your oRPC client
const { data } = useQuery(orpc.bill.list.queryOptions({ input: { houseId } }));

const createBill = useMutation(
  orpc.bill.create.mutationOptions({
    onSuccess: () => queryClient.invalidateQueries({ queryKey: orpc.bill.key() }),
  }),
);
```

See [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md).

## Common pitfalls
- Not throwing on `!res.ok`, so error responses are cached as successful data.
- A query key missing a variable the `queryFn` uses: wrong data, and no refetch when the variable changes.
- Creating the `QueryClient` in a component body, or sharing one across SSR requests (data leaks between users).
- Copying query data into `useState`, where it goes stale; derive from `data` or use the `select` option.
- Leaving `staleTime: 0` and wondering why every focus refetches, or setting it huge and forgetting to invalidate after mutations.
- Treating a disabled query's `isPending` as "loading" and showing a spinner forever.
- Optimistic updates without `cancelQueries` (an in-flight refetch overwrites the optimistic value) or without rollback.
- Retrying 401, 403 and 404 responses three times.
- Mutating cached data in place inside `setQueryData` updaters.
- Invalidating with `exact: true` or an overly specific key, leaving related lists stale.

## How to talk about this in an interview
- Frame the shift: "I separate server state from client state. Server data lives in a cache keyed by query key; staleness, background refresh and invalidation are handled by TanStack Query, so components just declare what they need."
- Show the fundamentals first (abort, race conditions, `res.ok`), then why a library wins. That order proves you understand what the library does for you.
- Your projects supply examples: lists and details on the Colochousing REST API (houses, applications, bills, events) and an oRPC client on the TanStack Start platform, where query options can be generated from typed procedures.

> **Fill in:** how data fetching works in the codebases you touch at Devntech (TanStack Query, router loaders, oRPC query utilities, plain fetch in effects, or a mix) and why.

> **Fill in:** one mutation flow you built (for example creating or paying a split bill in the resident app) and how the UI refreshed afterwards: invalidation, `setQueryData` or an optimistic update.

- For PureLogics, the LMS frontend consumed DRF endpoints (courses, enrollments, subscriptions); explain how you would key and invalidate those queries today, without claiming the original code did so.

## Interview questions

**Q1. How do you fetch data correctly in useEffect, and what is the race condition?**
Track data, loading and error; create an `AbortController` in the effect, pass its signal to `fetch`, throw when `!res.ok`, and abort in the cleanup. Without that, two quick parameter changes can resolve out of order and the older response overwrites the newer one. Ignore abort errors so they do not show as failures.
*Follow-up:* Why check `res.ok`? `fetch` only rejects on network failure; 404 and 500 resolve normally.

**Q2. Why use TanStack Query instead of useEffect plus useState?**
It provides a shared cache keyed by query key, request deduplication, background refresh on mount, focus and reconnect, retries with backoff, pagination and infinite queries, mutations with invalidation, optimistic updates, SSR hydration and devtools. Writing all that per screen is a lot of buggy boilerplate.
*Follow-up:* When would you skip it? A tiny app with a couple of requests, or when route loaders already cover your needs.

**Q3. What is the difference between staleTime and gcTime?**
`staleTime` (default 0) is how long data is considered fresh: fresh data is served from cache without refetching; stale data is shown and refetched on triggers. `gcTime` (default 5 minutes, `cacheTime` before v5) is how long a query with no active observers stays in memory before garbage collection. One controls refetching, the other memory.
*Follow-up:* Settings for reference data? `staleTime: Infinity`, so it is fetched once per session.

**Q4. When does TanStack Query refetch automatically?**
For stale queries: when a new component subscribes, when the window regains focus, when the network reconnects, and on `refetchInterval`. Also after `invalidateQueries` for active queries and on manual `refetch`. Fresh queries ignore those triggers.
*Follow-up:* How do you stop focus refetching for one query? `refetchOnWindowFocus: false`, or a longer `staleTime`.

**Q5. isPending vs isFetching vs isLoading?**
`isPending` means no data yet. `isFetching` means the query function is running, including background refetches while data is shown. `isLoading` is `isPending && isFetching`, the first load actually in progress. Show a full spinner for the first, a subtle indicator for background fetching.
*Follow-up:* Why could `isPending` stay true forever? The query is disabled and has no data.

**Q6. How do you design query keys? What is a key factory?**
Keys are arrays containing every variable the `queryFn` depends on, ordered from general to specific (`['bills', 'list', { houseId, status }]`), so prefix invalidation can target a whole group. A key factory builds them in one place (`billKeys.list(filters)`), often paired with `queryOptions` so the key, function and options are reused for queries, prefetching and cache reads.
*Follow-up:* Does object key order inside a key matter? No, keys are hashed deterministically; array order does matter.

**Q7. How does invalidateQueries work, and when would you use setQueryData instead?**
`invalidateQueries({ queryKey })` marks every query whose key starts with that prefix as stale and refetches the ones on screen; the others refetch on next use. `setQueryData` writes into the cache directly, useful when the mutation response already contains the new entity. Invalidation is the safer default because the server recomputes derived data.
*Follow-up:* How do you make the mutation wait for the refetch? Return the invalidation promise from `onSuccess` or `onSettled`.

**Q8. Walk me through an optimistic update with rollback.**
In `onMutate`: cancel in-flight queries for the key, snapshot the current data with `getQueryData`, write the optimistic version with `setQueryData`, return the snapshot. In `onError`: restore the snapshot and notify the user. In `onSettled`: invalidate the key so the cache matches the server either way.
*Follow-up:* Why cancel queries first? A refetch that started earlier could resolve later and overwrite the optimistic value with old data.

**Q9. How do you implement dependent queries?**
Use `enabled`: the second query has `enabled: !!userId` and includes `userId` in its key, so it starts as soon as the first query provides the id. While disabled it reports pending with `fetchStatus: 'idle'`. Recent v5 versions also accept `skipToken` in place of the `queryFn`.
*Follow-up:* Downside? A sequential waterfall; prefer one endpoint or a loader when you can.

**Q10. How do you paginate without flashing a spinner? And infinite scroll?**
Put the page in the key and set `placeholderData: keepPreviousData`, so the previous page stays visible while the next loads; `isPlaceholderData` disables the Next button meanwhile. For infinite scroll use `useInfiniteQuery` with `initialPageParam` and `getNextPageParam`, render `data.pages.flatMap(...)`, and call `fetchNextPage` when a sentinel element becomes visible.
*Follow-up:* Cursor or offset? Cursor, when items can be inserted or removed while the user scrolls.

**Q11. What are request waterfalls, and how do you avoid them?**
Requests that run one after another because each starts only after a previous render or response, such as a parent fetching, rendering a child, and the child fetching. Fix them by fetching in parallel (several `useQuery` calls or `useQueries`), hoisting fetches to route loaders with `ensureQueryData`, prefetching on hover, or adding an endpoint that returns the combined data.
*Follow-up:* How do you spot them? A staircase of requests in the browser's Network panel.

**Q12. How does TanStack Query handle errors and retries, and what would you change?**
A failed query retries 3 times with exponential backoff on the client, then sets `status: 'error'` with `error`; background refetch failures keep the old `data`. I would skip retries for 4xx responses with a `retry` function, show inline errors with a retry button, add a global `QueryCache` `onError` for toasts or 401 handling, and use `throwOnError` where an error boundary should take over.
*Follow-up:* Why no retries on the server? So SSR fails fast instead of delaying the response.

**Q13. What does useSuspenseQuery change?**
The component suspends until data exists, so `data` is always defined and there are no loading branches; the nearest `<Suspense>` shows the fallback and errors go to the nearest error boundary. It has no `enabled` option, and several suspense queries in one component run serially, so prefetch or use `useSuspenseQueries`.
*Follow-up:* How do you retry after an error? Reset the boundary together with `QueryErrorResetBoundary`.

**Q14. How do you prefetch data?**
Call `queryClient.prefetchQuery(options)` on hover or focus of a link, or `ensureQueryData` in a route loader, with the same `queryOptions` the destination component uses. When the page renders, its `useQuery` finds the data in the cache. `prefetchQuery` skips fresh data and never throws.
*Follow-up:* What does `ensureQueryData` return? The cached data if present, otherwise the result of fetching it.

**Q15. How does SSR hydration work with TanStack Query?**
On the server, create a `QueryClient` per request, prefetch the page's queries and `dehydrate` the cache into serializable state. The client wraps the tree in `HydrationBoundary` with that state, so `useQuery` with matching keys renders with data immediately and matches the server HTML. A `staleTime` above 0 avoids an instant refetch.
*Follow-up:* Why a new client per request? A shared client would leak one user's cached data to another.

**Q16. Server state vs client state: why keep them separate?**
Server state is owned by the backend, asynchronous, shared with other users and can go stale; it needs caching, refetching and invalidation. Client state (a modal, a theme, a draft) is synchronous and fully yours. Mixing them forces a client store to imitate a cache badly.
*Follow-up:* Example of mixing them badly? Copying `/me` into Redux and forgetting to update it after the profile is edited.

**Q17. Does the same code work in React Native? What is different?**
Yes, the hooks are identical. Differences: no window focus, so connect `focusManager` to `AppState`; online detection through `onlineManager` and NetInfo; devtools through React Native-specific tooling; and optional persistence of the cache to AsyncStorage for offline starts.
*Follow-up:* Why care about focus on mobile? Users return to the app after minutes or hours, and stale data should refresh on resume.

**Q18. How would you use TanStack Query with oRPC or another typed RPC client?**
Through the client's TanStack Query integration, which generates `queryOptions` and `mutationOptions` per procedure with inferred input and output types, plus key helpers for invalidation. Screens call `useQuery(orpc.bill.list.queryOptions({ input }))` and get typed data without hand-written fetchers or keys. I would confirm the exact helper names in the current oRPC docs.
*Follow-up:* What do you gain over REST plus fetch? End-to-end types: changing a procedure's output breaks the client at compile time.

## Further reading
- [TanStack Query docs (React)](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Fetching data with effects](https://react.dev/reference/react/useEffect#fetching-data-with-effects)
- [oRPC docs](https://orpc.unnoq.com/)
