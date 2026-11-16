# Routing in React

> **Tier:** T1 (know cold) · **Related:** [Data fetching](04-data-fetching-tanstack-query.md), [Performance](06-performance.md), [Patterns](07-patterns-and-architecture.md), [React 19 and RSC](09-react-19-and-server-components.md), [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [TanStack Router](../07-nextjs-and-tanstack/04-tanstack-router.md), [Expo Router](../08-react-native-expo/03-navigation-expo-router.md), [Authentication](../04-web-fundamentals/06-authentication-authorization.md)
> **On your resume:** Routing spans your stack: Rails routes serving React on Rails pages in the Colochousing app, the TanStack Start platform (built on TanStack Router), where you contributed to the marketplace page migration and org-scoped access control, navigation in the Coloc resident app, and Next.js file-based routing (including API routes) at RemoteDev.

## Quick revision
- Client-side routing: intercept link clicks, update the URL with the History API (`history.pushState`), and render the matching component without a page load; `popstate` fires on back and forward.
- SPAs need a server fallback: unknown paths must return `index.html`, or deep links 404 on refresh.
- React Router has three modes: declarative (`<BrowserRouter>` + `<Routes>`), data (`createBrowserRouter` + `RouterProvider`, with loaders and actions, since v6.4) and framework (v7, the successor of Remix).
- Route objects: `path`, `element` or `Component`, `children`, `index`, `loader`, `action`, `errorElement`, `lazy`.
- Nested routes render inside the parent's `<Outlet />`; layout routes share UI; `index: true` marks the default child.
- Hooks: `useParams` (always strings), `useSearchParams`, `useNavigate`, `useLocation`, `useLoaderData`, `useNavigation`, `useRouteError`.
- `<Link>`/`<NavLink>` for user navigation (real anchors), `navigate()` after actions, `<Navigate>` to redirect during render. Redirects use `replace`.
- Loaders fetch before the route renders (in parallel for nested routes), removing fetch-on-render waterfalls; actions handle `<Form>` submissions and then revalidate loaders.
- `errorElement` catches errors thrown in a route's loader, action or render.
- Protected routes: a guard layout route that renders `<Navigate to="/login" replace />` or `<Outlet />`, or a loader that throws `redirect()`. Real authorization happens on the server.
- 404s: a `path: '*'` route plus 404 responses thrown from loaders. `<ScrollRestoration />` handles scroll positions.
- TanStack Router: type-safe paths, params and schema-validated search params, loaders, file-based routing; it is the router inside TanStack Start.
- Next.js derives routes from the `app/` directory; Expo Router brings file-based routing to React Native.

## 1. How client-side routing works

In a multi-page app every link is a full request: the server renders a new HTML page. In a single-page app (SPA) the browser loads the app once, and a router swaps views in JavaScript while keeping the URL meaningful.

```ts
// The core idea behind every router (simplified)
function navigate(path: string) {
  window.history.pushState({}, '', path); // update the address bar, no request to the server
  renderRoute(path);                      // match the path and render that view
}

window.addEventListener('popstate', () => renderRoute(window.location.pathname)); // back and forward

// Routers render <Link> as a real <a>, intercept the click, call preventDefault(), then navigate()
```

| | Multi-page app (classic Rails, Django templates) | Single-page app |
|---|---|---|
| Navigation | full request, server renders HTML | JavaScript swaps views, data comes from an API |
| State between pages | lost unless stored | kept in memory |
| First load | fast HTML | waits for the JS bundle (unless SSR) |
| SEO for public pages | easy | needs SSR or static generation |
| Server's routing job | all routes | API plus `index.html` for every app route |

**Server fallback.** If a user refreshes `/houses/42` on a static host, the server looks for a file at that path and returns 404. Configure it to serve `index.html` for unknown paths (`try_files $uri /index.html;` in nginx, a rewrite rule on static hosts) so the router can take over. Hash routing (`/#/houses/42`, `createHashRouter`) avoids this because the part after `#` never reaches the server, at the cost of uglier URLs and weaker SEO.

**With Rails.** When React is embedded through React on Rails, Rails routes usually decide which page loads and React handles UI inside that page. If a section uses client-side routing, add a catch-all Rails route for HTML requests that renders the page hosting the React app, so deep links work ([React on Rails](../05-ruby-on-rails/16-react-on-rails.md)).

## 2. React Router: modes and setup

| Mode | Setup | What you get |
|---|---|---|
| Declarative | `<BrowserRouter>` + `<Routes>` / `<Route>` | matching, links, params, nested routes |
| Data (v6.4+) | `createBrowserRouter([...])` + `<RouterProvider>` | plus loaders, actions, pending UI, error elements, lazy routes |
| Framework (v7) | Vite plugin, route config in `app/routes.ts` | plus SSR, type generation, automatic code splitting |

In v7 everything is imported from `react-router` (`react-router-dom` remains as a compatibility re-export); in v6 you import the same names from `react-router-dom`.

```tsx
// main.tsx: data router
import { createRoot } from 'react-dom/client';
import { createBrowserRouter, RouterProvider } from 'react-router';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <Home /> },                                       // matches "/"
      { path: 'houses', element: <HouseList />, loader: housesLoader },
      { path: 'houses/:houseId', element: <HouseDetail />, loader: houseLoader, action: renameHouseAction },
      { path: 'reports', lazy: () => import('./routes/reports') },              // code-split route
      { path: '*', element: <NotFound /> },                                     // catch-all 404
    ],
  },
]); // created once at module level, never inside a component

createRoot(document.getElementById('root')!).render(<RouterProvider router={router} />);
```

```tsx
// Declarative mode, no loaders or actions
<BrowserRouter>
  <Routes>
    <Route path="/" element={<RootLayout />}>
      <Route index element={<Home />} />
      <Route path="houses/:houseId" element={<HouseDetail />} />
      <Route path="*" element={<NotFound />} />
    </Route>
  </Routes>
</BrowserRouter>
```

## 3. Nested routes, layouts and Outlet

```tsx
import { NavLink, Outlet, ScrollRestoration } from 'react-router';

function RootLayout() {
  return (
    <div className="app">
      <nav>
        <NavLink to="/houses" className={({ isActive }) => (isActive ? 'active' : undefined)}>
          Houses
        </NavLink>
        <NavLink to="/reports">Reports</NavLink>
      </nav>
      <main>
        <Outlet /> {/* the matched child route renders here */}
      </main>
      <ScrollRestoration />
    </div>
  );
}
```

- Child paths are relative to the parent (`houses/:houseId` under `/`).
- A route with `element` but no `path` is a pathless layout route: it wraps its children with shared UI or logic (an auth guard, a tenant switcher) without adding a URL segment.
- `index: true` is the child shown at the parent's exact path.
- Layouts persist across child navigation, so their state (an open sidebar, a search box) survives.

## 4. Params, search params and navigation

```tsx
import { Link, useNavigate, useParams, useSearchParams } from 'react-router';

function HouseDetail() {
  const { houseId } = useParams(); // string | undefined: params are always strings
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();
  const tab = searchParams.get('tab') ?? 'overview';

  return (
    <>
      <h1>House {houseId}</h1>
      <button onClick={() => setSearchParams({ tab: 'residents' })} aria-pressed={tab === 'residents'}>
        Residents
      </button>
      <Link to={`/houses/${houseId}/events`}>Events</Link>
      <button onClick={() => navigate(-1)}>Back</button>
    </>
  );
}
```

- **`<Link>`** renders a real `<a href>`: users can open it in a new tab, copy it, and screen readers announce it as a link. Use it for anything the user clicks to go somewhere.
- **`navigate(to, { replace, state })`** for programmatic navigation after something happens (a successful save, a login). `replace: true` replaces the history entry, so Back does not return to a form or a redirecting page.
- **`<Navigate to replace />`** redirects during render (guards); calling `navigate()` during render is not allowed.
- `useLocation()` gives `pathname`, `search`, `hash` and the `state` passed during navigation.
- Parse and validate params and search params: `Number(houseId)` can be `NaN`, and the URL is user input.

## 5. Loaders and actions (data routers)

A loader runs before its route renders and provides data; an action handles a non-GET submission. When an action finishes, the router revalidates the loaders on the page so the UI shows fresh data.

```tsx
import {
  Form,
  redirect,
  useLoaderData,
  useNavigation,
  type ActionFunctionArgs,
  type LoaderFunctionArgs,
} from 'react-router';

type House = { id: string; name: string; city: string };

export async function houseLoader({ params }: LoaderFunctionArgs) {
  const res = await fetch(`/api/houses/${params.houseId}`);
  if (res.status === 404) throw new Response('House not found', { status: 404 }); // goes to errorElement
  if (!res.ok) throw new Error('Failed to load the house');
  return (await res.json()) as House;
}

export async function renameHouseAction({ request, params }: ActionFunctionArgs) {
  const formData = await request.formData();
  const res = await fetch(`/api/houses/${params.houseId}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name: formData.get('name') }),
  });
  if (!res.ok) return { error: 'Could not rename the house' };
  return redirect(`/houses/${params.houseId}`);
}

export function HouseDetailPage() {
  const house = useLoaderData() as House; // v7 can infer this from the loader's type
  const navigation = useNavigation();
  return (
    <>
      <h1>{house.name}</h1>
      <Form method="post"> {/* submits to this route's action, no fetch code needed */}
        <input name="name" defaultValue={house.name} />
        <button disabled={navigation.state === 'submitting'}>Rename</button>
      </Form>
    </>
  );
}
```

- Loaders for all matched nested routes start in parallel when navigation begins, so a layout and its child do not wait on each other (no fetch-on-render waterfall).
- `useNavigation().state` is `'idle'`, `'loading'` or `'submitting'`, for global pending UI.
- `useFetcher()` submits or loads without navigating (a "mark as paid" toggle in a list row).
- `useActionData()` reads what an action returned, such as validation errors.
- Loaders pair well with TanStack Query: the loader warms the cache, the component reads it.

```tsx
export const houseLoaderWithCache = (queryClient: QueryClient) => async ({ params }: LoaderFunctionArgs) =>
  queryClient.ensureQueryData(houseQueries.detail(params.houseId!)); // cached data or one fetch

// Route: { path: 'houses/:houseId', loader: houseLoaderWithCache(queryClient), element: <HouseDetail /> }
// HouseDetail calls useQuery(houseQueries.detail(houseId)) and gets the cached data immediately
```

### Going deeper: revalidation

After every successful action (and on some navigations, such as a search-param change), the router re-runs the loaders of the routes on screen so their data reflects the change. A route can opt out or narrow this with a `shouldRevalidate` function, for example to skip an expensive loader when only an unrelated search param changed. With TanStack Query behind the loaders, revalidation is a cheap cache read unless the query was invalidated.

## 6. Error handling with errorElement

```tsx
import { isRouteErrorResponse, useRouteError } from 'react-router';

function RootError() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    // A Response thrown from a loader or action, e.g. 404 or 403
    return <h1>{error.status} {error.statusText || String(error.data)}</h1>;
  }
  return <h1>Something went wrong</h1>;
}
```

- Errors thrown in a loader, action or during rendering bubble to the nearest route with an `errorElement`, replacing that route's element. Put error elements on nested layout routes so a failing child does not blank the whole app.
- Throw `Response` objects for expected HTTP outcomes (404, 403) and `Error` for unexpected failures.

## 7. Lazy routes

```tsx
// Route definition: { path: 'reports', lazy: () => import('./routes/reports') }

// routes/reports.tsx exports route properties by name
export async function loader() {
  return fetchJson<Report[]>('/api/reports');
}

export function Component() {
  const reports = useLoaderData() as Report[];
  return <ReportTable reports={reports} />;
}
```

The route's code downloads the first time it matches, and the router fetches the chunk and runs the loader as part of the navigation. Route-level splitting is usually the biggest bundle win ([performance](06-performance.md)).

## 8. Protected routes and auth guards

```tsx
import { Navigate, Outlet, useLocation } from 'react-router';

function RequireAuth() {
  const { user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <FullPageSpinner />; // do not flash the login page while the session loads
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;
  return <Outlet />;
}

// Routes: { element: <RequireAuth />, children: [{ path: 'billing', element: <Billing /> }, ...] }
```

```tsx
// Data-router alternative: the guard runs before anything renders
export async function protectedLoader() {
  const user = await getCurrentUser(); // e.g. GET /api/me with the session cookie
  if (!user) throw redirect('/login');
  return user;
}

// Login page: after a successful login, send the user back where they were going
function LoginPage() {
  const location = useLocation(); // the router's location, which carries `state`
  const navigate = useNavigate();
  const from = (location.state as { from?: { pathname: string } } | null)?.from?.pathname ?? '/';

  async function handleLogin(credentials: Credentials) {
    await login(credentials);
    navigate(from, { replace: true });
  }
  return <LoginForm onSubmit={handleLogin} />;
}
```

- Role checks work the same way (`<RequireRole role="admin" />` or a loader that throws a 403 response).
- Client-side guards are UX only: they hide screens, but anyone can call the API directly. Every endpoint must check authentication and authorization (Rails `before_action` and policies, DRF permissions, oRPC middleware). In a multi-tenant app, the server checks that the requested organization belongs to the user; the route guard only keeps the UI honest ([authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md)).

## 9. 404s, scroll restoration and focus

- A `path: '*'` child renders for unmatched URLs; loaders throw `404` responses for missing records ("house 999 does not exist"), handled by `errorElement`.
- A pure SPA always returns HTTP 200 with `index.html`, so search engines cannot see a real 404 status; SSR frameworks can send the proper status.
- `<ScrollRestoration />` (data routers, rendered once in the root layout) scrolls to the top on new navigations and restores the previous position on Back. With declarative routing you write it yourself: an effect that calls `window.scrollTo(0, 0)` when `pathname` changes.
- Accessibility: after navigation, move focus to the main heading or announce the new page, and update the document title (React 19 can render `<title>` from any component).

## 10. React Router v7 framework mode

Remix v2 merged into React Router v7. Framework mode adds a Vite plugin, a route config (`app/routes.ts`, or file-system conventions through an add-on package), server rendering, static pre-rendering or SPA mode, server-side loaders and actions, and generated types per route. Conventions changed across releases, so treat this as a sketch and check reactrouter.com:

```tsx
// app/routes/house.tsx (framework mode sketch)
import type { Route } from './+types/house';

export async function loader({ params }: Route.LoaderArgs) {
  return getHouse(params.houseId); // runs on the server; can talk to the database directly
}

export default function House({ loaderData }: Route.ComponentProps) {
  return <h1>{loaderData.name}</h1>; // loaderData is typed from the loader's return value
}
```

## 11. TanStack Router

TanStack Router is fully type-safe: route paths, params, search params and loader data are inferred, so `<Link to="/houses/$houseId">` fails to compile if the route does not exist or a param is missing. It supports file-based routing (a bundler plugin generates the route tree), loaders with built-in caching, search params validated by a schema, and it powers TanStack Start. APIs evolved quickly, so check tanstack.com for current details:

```tsx
// src/routes/houses.$houseId.tsx (file-based route sketch)
import { createFileRoute } from '@tanstack/react-router';

type HouseSearch = { tab: 'overview' | 'residents' };

export const Route = createFileRoute('/houses/$houseId')({
  validateSearch: (search: Record<string, unknown>): HouseSearch => ({
    tab: search.tab === 'residents' ? 'residents' : 'overview', // untrusted URL input becomes a typed value
  }),
  loader: ({ params }) => fetchHouse(params.houseId),
  component: HouseDetail,
});

function HouseDetail() {
  const house = Route.useLoaderData();
  const { tab } = Route.useSearch(); // typed as 'overview' | 'residents'
  return <h1>{house.name} ({tab})</h1>;
}

// <Link to="/houses/$houseId" params={{ houseId: '42' }} search={{ tab: 'residents' }}>Open</Link>
```

Schema libraries such as Zod can replace the hand-written `validateSearch` function. Deep dive: [TanStack Router](../07-nextjs-and-tanstack/04-tanstack-router.md) and [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md).

| | React Router (data / framework) | TanStack Router | Next.js App Router | Expo Router |
|---|---|---|---|---|
| Route definition | objects or config file (file conventions optional) | file-based (or code-based) | `app/` file system | `app/` file system |
| Type safety | generated route types in framework mode | inferred end to end, including search params | typed via generated types and helpers | typed routes option |
| Data loading | loaders and actions | loaders with caching, often with TanStack Query | Server Components, Server Actions | client-side fetching |
| SSR | framework mode | through TanStack Start | built in | not applicable (native) |

## 12. Next.js and Expo Router pointers

- **Next.js App Router:** folders in `app/` are URL segments (`app/houses/[houseId]/page.tsx`), with `layout.tsx` for nested layouts, `loading.tsx` (a Suspense boundary), `error.tsx` (an error boundary) and `not-found.tsx`. Navigate with `<Link>` from `next/link` and `useRouter`, `usePathname`, `useSearchParams` from `next/navigation`. Since Next.js 15, `params` and `searchParams` are async in pages. See [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md).
- **Expo Router:** file-based routing for React Native on top of React Navigation, with stacks, tabs and deep links derived from the `app/` folder. See [navigation and Expo Router](../08-react-native-expo/03-navigation-expo-router.md).

## Common pitfalls
- Plain `<a href>` for internal links, causing full reloads and lost state.
- No SPA fallback on the server, so refreshing a deep link returns 404.
- Treating params as numbers without parsing, or trusting search params without validation.
- Redirects without `replace`, so Back bounces the user into a redirect loop.
- Guards that flash the login page while the session is still loading.
- Relying on client-side guards for security.
- Creating the router inside a component, so it is rebuilt on every render.
- Calling `navigate()` during render instead of rendering `<Navigate>`.
- One error boundary at the root only, so a failing child route blanks the whole layout.
- Fetching in child components after render when a loader could fetch in parallel.

## How to talk about this in an interview
- Explain the mechanism first (History API, intercepted clicks, server fallback), then the library features. Interviewers often probe whether you know what a router actually does.
- Tie it to your work: the TanStack Start platform uses TanStack Router, so you can discuss type-safe routes, loaders and search params; the Colochousing app mixes Rails routing with React on Rails components.

> **Fill in:** how routing and access control worked in the marketplace page migration on the TanStack Start platform (route structure, loaders, how org-scoped access was checked in the route and on the server).

> **Fill in:** how the Colochousing Rails routes and the React components divide responsibility (which pages are Rails views with embedded components, and whether any section uses client-side routing).

- For mobile, explain how the same ideas map to React Native navigation (stacks and tabs instead of URLs, deep links that open a screen from a push notification) and relate it to the Coloc resident app; see [navigation and Expo Router](../08-react-native-expo/03-navigation-expo-router.md).
- Security framing wins points: "route guards are for UX; authorization is enforced by the API".

## Interview questions

**Q1. How does client-side routing work under the hood?**
The router renders links as anchors but intercepts clicks, calls `preventDefault()`, updates the URL with `history.pushState` and renders the component that matches the new path, without requesting a new page. It listens to `popstate` to handle back and forward. The server only has to deliver the app shell and the API.
*Follow-up:* What is `replaceState` for? Changing the URL without adding a history entry, as with redirects or filter updates.

**Q2. Why does refreshing a deep link return 404 on a deployed SPA, and how do you fix it?**
On refresh the browser requests `/houses/42` from the server, which has no such file. Configure a fallback that returns `index.html` for unknown paths (nginx `try_files`, rewrite rules on static hosts), so the client router can render the route. Alternatively use hash routing, or SSR.
*Follow-up:* What about real 404s? A catch-all route shows a not-found page; only SSR can also send a 404 status.

**Q3. What are nested routes, and what does Outlet do?**
Routes can have child routes whose paths extend the parent's. The parent renders shared UI (navigation, a layout, a tenant header) and an `<Outlet />` where the matched child renders. Layouts persist while moving between children, so their state survives, and pathless layout routes group children without adding a URL segment.
*Follow-up:* What is an index route? The default child rendered at the parent's exact path.

**Q4. How do you read URL params and query strings?**
`useParams()` returns dynamic segments such as `houseId` (always strings). `useSearchParams()` returns a `URLSearchParams` object plus a setter for the query string. Parse and validate both: they come from the user. TanStack Router can validate search params with a schema and type them.
*Follow-up:* Why keep filters in search params? Refresh, Back and shared links restore the same view.

**Q5. useNavigate vs Link vs Navigate?**
`<Link>` renders a real anchor for user-initiated navigation (accessible, open-in-new-tab works). `useNavigate()` returns a function for programmatic navigation in handlers and effects, such as after a successful save. `<Navigate>` is a component that redirects while rendering, typical in guards. Use `replace` for redirects.
*Follow-up:* Why not navigate during render? It is a side effect; React Router warns and `<Navigate>` exists for that case.

**Q6. What are loaders and actions, and what problem do they solve?**
A loader fetches a route's data before it renders, and loaders of nested routes run in parallel, which removes fetch-on-render waterfalls and loading-state boilerplate. An action handles a form submission (`<Form method="post">`) and afterwards the router revalidates loaders, so the page shows fresh data. Errors from either go to the route's `errorElement`.
*Follow-up:* How do they combine with TanStack Query? The loader calls `ensureQueryData`; the component reads the same query from the cache.

**Q7. How do you implement protected routes, and is that enough for security?**
Wrap protected routes in a guard layout route that renders `<Navigate to="/login" replace state={{ from: location }} />` when there is no user and `<Outlet />` otherwise, showing a spinner while the session loads; or check in a loader and `throw redirect('/login')`. It is not enough: guards only hide UI, and the API must authenticate and authorize every request.
*Follow-up:* How do you return the user to the original page after login? Read `location.state.from` and `navigate(from, { replace: true })`.

**Q8. How do you handle 404s and errors per route?**
A `path: '*'` route renders a not-found page for unmatched URLs. Loaders throw a `Response` with status 404 for missing records, and `errorElement` with `useRouteError` plus `isRouteErrorResponse` renders the right message. Placing error elements on nested layouts keeps the rest of the page alive when one section fails.
*Follow-up:* Which errors reach `errorElement`? Errors thrown in loaders, actions and rendering of that route subtree.

**Q9. How do you code split by route?**
Use the route's `lazy` property (`lazy: () => import('./routes/reports')`, with the module exporting `Component`, `loader` and so on), or wrap route components in `React.lazy` with `Suspense` in declarative mode. Framework modes (React Router v7, TanStack Router with its plugin, Next.js) split routes automatically.
*Follow-up:* Why split by route first? Users rarely visit every page, so most route code is wasted on first load.

**Q10. How do you restore scroll position between pages?**
In data routers, render `<ScrollRestoration />` once in the root layout: it scrolls to the top on new navigation and restores positions on Back and Forward. In declarative mode, add an effect that scrolls to the top when `pathname` changes, and accept that Back will not restore the old position unless you store it.
*Follow-up:* Anything else to handle on navigation for accessibility? Move focus to the main heading and update the document title.

**Q11. What is React Router v7 framework mode, and how does it relate to Remix?**
Remix v2's features merged into React Router v7, and framework mode is that merged framework: a Vite plugin, a route config, server rendering or pre-rendering, server-side loaders and actions, and generated route types. Apps can adopt it gradually from data mode. Details changed across releases, so I would check the docs before quoting conventions.
*Follow-up:* Can you still use React Router as a plain library? Yes, declarative and data modes remain.

**Q12. What does TanStack Router offer over React Router?**
End-to-end type safety for paths, params, search params and loader data (invalid links fail to compile), first-class search param validation with schemas, loaders with built-in caching, file-based routing via a plugin, and the foundation for TanStack Start. React Router has a larger ecosystem and the Remix lineage.
*Follow-up:* What does validated search params buy you? Typed, defaulted values instead of parsing strings in every component.

**Q13. How do you redirect a user back to the page they wanted after login?**
When the guard redirects to `/login`, pass the current location in navigation state (or as a `?redirectTo=` param). After a successful login, read it and `navigate(from, { replace: true })`, falling back to a default page. Validate a `redirectTo` param so it only allows internal paths (no open redirects).
*Follow-up:* Why `replace`? So Back does not return to the login page.

**Q14. Hash router vs browser router?**
A browser router uses clean paths through the History API and needs a server fallback to `index.html`. A hash router keeps the route after `#`, which never reaches the server, so it works on any static host without configuration, at the cost of uglier URLs and poorer SEO.
*Follow-up:* When is a hash router reasonable? Static hosting without rewrite support, or apps embedded in an environment you do not control.

**Q15. How do routing and data fetching interact?**
Fetch-on-render (components fetch in effects after mounting) creates waterfalls as each nested level waits for its parent. Render-as-you-fetch starts all requests when navigation begins, through loaders or prefetching, so data and code load in parallel. With TanStack Query, loaders warm the cache and components read it, keeping caching and invalidation in one place.
*Follow-up:* How does Next.js handle this? Server Components fetch on the server as part of rendering the route.

## Further reading
- [React Router docs](https://reactrouter.com/)
- [TanStack Router docs](https://tanstack.com/router/latest)
- [History API on MDN](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
