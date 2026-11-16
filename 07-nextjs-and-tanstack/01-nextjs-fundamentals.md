# Next.js Fundamentals

> **Tier:** T1 (know cold) · **Related:** [React 19 and Server Components](../06-react/09-react-19-and-server-components.md), [Routing in React](../06-react/08-routing.md), [Next.js data fetching and caching](02-nextjs-data-fetching-and-caching.md), [Next.js backend APIs and MongoDB](03-nextjs-backend-apis-and-mongodb.md), [TanStack Start](05-tanstack-start.md), [React performance](../06-react/06-performance.md)
> **On your resume:** the RemoteDev internship, where you built and integrated backend APIs with Next.js (TypeScript) and MongoDB; Next.js is in your skills list and is the natural comparison point for the TanStack Start platform at Devntech.

## Quick revision
- Next.js is a React framework: file-based routing, server rendering (SSR, SSG, ISR, streaming, React Server Components), data fetching and caching, backend endpoints (Route Handlers, Server Actions), image/font/script optimization, and a production build and server.
- Rendering: **CSR** renders in the browser; **SSR** builds HTML per request; **SSG** builds HTML at build time; **ISR** serves static HTML and regenerates it after a revalidate period or on demand; **streaming** sends the shell first and fills Suspense boundaries as data arrives.
- **App Router** (`app/`, stable since 13.4): Server Components by default, nested layouts, streaming, Server Actions. **Pages Router** (`pages/`): `getServerSideProps`, `getStaticProps`, `getStaticPaths`, `pages/api`. Both can coexist; new work goes in `app/`.
- File conventions: `page`, `layout`, `template`, `loading`, `error` (a Client Component), `global-error`, `not-found`, `route`, `default`. Folders: `[id]`, `[...slug]`, `[[...slug]]`, `(group)`, `_private`, `@slot`, `(.)intercepted`.
- Everything in `app/` is a Server Component unless its module (or an importing module) starts with `'use client'`. Mark interactive leaves, not whole pages.
- Server Components cannot use state, effects, event handlers, context or browser APIs. Props passed to Client Components must be serializable; pass Server Components into Client Components as `children`.
- Navigation: `<Link>` (prefetches in production); `useRouter`, `usePathname`, `useSearchParams` from `next/navigation` (not `next/router`); `redirect()` and `notFound()` on the server.
- SEO: export `metadata` or `generateMetadata`; add `sitemap.ts`, `robots.ts`, `opengraph-image.tsx`; set `metadataBase` for absolute URLs.
- `next/image`: resizing, modern formats, lazy loading, required dimensions (or `fill`) to prevent layout shift; remote hosts allowed through `images.remotePatterns`. Next.js 16 deprecated `priority` in favor of `preload`.
- `next/font` self-hosts fonts at build time (no layout shift, no browser request to Google). `next/script` loads third-party scripts with a `strategy`.
- `NEXT_PUBLIC_` variables are inlined into the client bundle **at build time**; all other env vars stay on the server.
- Next.js 15 made `params`, `searchParams`, `cookies()`, `headers()` and `draftMode()` async (`await params`); Next.js 16 removed the temporary synchronous access.
- Next.js 16 (October 2025): Turbopack by default, Cache Components with `"use cache"`, `middleware.ts` renamed to `proxy.ts` (Node.js runtime), `next lint` removed, Node.js 20.9+ required.
- Deploy on Vercel (zero config), with `next build && next start` on any Node host, with `output: 'standalone'` for small Docker images, or with `output: 'export'` as a static site without server features.

## 1. What Next.js adds to React

React renders components; it does not decide URLs, where code runs or how data reaches the page. A plain Vite SPA ships one JavaScript bundle and renders everything in the browser.

| Concern | Vite + React SPA | Next.js |
|---|---|---|
| Routing | add React Router or TanStack Router | file system (`app/`) |
| First paint and SEO | empty HTML until JavaScript runs | HTML from the server (SSR, SSG, ISR) |
| Data fetching | client-side (`useEffect`, TanStack Query) | async Server Components, plus client fetching where needed |
| Backend | a separate API server | Route Handlers and Server Actions in the same app |
| Optimization | manual | `next/image`, `next/font`, `next/script`, automatic code splitting |
| Deployment | static files on a CDN | Node server, serverless, Vercel, Docker, or static export |

## 2. Rendering strategies

| Strategy | When HTML is produced | Freshness | Pages Router | App Router | Good for |
|---|---|---|---|---|---|
| CSR | in the browser after JS loads | live | fetch in effects or TanStack Query | a Client Component that fetches | dashboards behind login, heavy widgets |
| SSR | on each request | per request | `getServerSideProps` | a dynamic route (uses cookies, headers, searchParams or uncached data) | personalized pages that need SEO or a fast first paint |
| SSG | at build time | as of the build | `getStaticProps` (+ `getStaticPaths`) | a static route (the default when nothing is dynamic), `generateStaticParams` | marketing pages, docs, blogs |
| ISR | at build, regenerated in the background | at most N seconds old, or on demand | `revalidate` in `getStaticProps`, `res.revalidate()` | `revalidate`, `revalidatePath`, `revalidateTag`, or `"use cache"` with `cacheLife` | catalogs and listings |
| Streaming | shell immediately, slow parts later | per request | limited | `loading.tsx` and `<Suspense>` | pages with one slow section |
| Partial prerendering (16, Cache Components) | static shell at build, dynamic holes streamed per request | mixed | none | `cacheComponents: true` | mostly static pages with personal parts |

You choose per route, not per app. A marketplace can serve its landing page as SSG, listing pages with ISR and the account area with SSR. The theory behind these modes (HTML vs RSC payload, hydration) is in [React 19 and Server Components](../06-react/09-react-19-and-server-components.md).

## 3. App Router vs Pages Router

| | Pages Router (`pages/`) | App Router (`app/`) |
|---|---|---|
| Components | Client Components only (rendered on the server for the first load) | Server Components by default, `'use client'` for interactive parts |
| Data loading | `getServerSideProps`, `getStaticProps`, `getStaticPaths` per page | `async` components fetch where the data is used |
| Layouts | `_app.tsx`, per-page layout patterns | nested `layout.tsx` files that persist across navigation |
| Loading and errors | manual | `loading.tsx` (Suspense) and `error.tsx` (error boundary) per segment |
| Mutations | API routes plus client fetch | Server Actions (and Route Handlers) |
| API endpoints | `pages/api/*.ts` with `(req, res)` | `app/**/route.ts` with Web `Request`/`Response` |
| Streaming | limited | built in |

```tsx
// pages/listings/[id].tsx (Pages Router)
import type { GetServerSideProps } from 'next';

type Props = { listing: { id: string; title: string } };

export const getServerSideProps: GetServerSideProps<Props, { id: string }> = async ({ params }) => {
  const listing = await getListing(params!.id);
  if (!listing) return { notFound: true };
  return { props: { listing } };      // props must be JSON-serializable: no Date objects
};

export default function ListingPage({ listing }: Props) {
  return <h1>{listing.title}</h1>;
}
```

```tsx
// app/listings/[id]/page.tsx (App Router): the component fetches its own data on the server
import { notFound } from 'next/navigation';

export default async function ListingPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const listing = await getListing(id);  // direct database call or fetch; never shipped to the browser
  if (!listing) notFound();
  return <h1>{listing.title}</h1>;
}
```

The routers can live side by side, so teams migrate page by page. The Pages Router is still supported, but new features land in the App Router.

## 4. File and folder conventions

| File | Purpose |
|---|---|
| `page.tsx` | the UI of a route; makes the segment publicly reachable |
| `layout.tsx` | wraps children and persists across navigations (state kept, not re-rendered); the root layout is required and renders `<html>` and `<body>` |
| `template.tsx` | like a layout but re-mounts on every navigation (fresh state, effects re-run) |
| `loading.tsx` | instant loading UI; wraps the page in `<Suspense>` |
| `error.tsx` | error boundary for the segment; must be a Client Component; receives `error` and `reset` |
| `global-error.tsx` | catches errors in the root layout; renders its own `<html>` and `<body>` |
| `not-found.tsx` | UI for `notFound()` and unmatched URLs |
| `route.ts` | a Route Handler (API endpoint); cannot share a segment with `page.tsx` |
| `default.tsx` | fallback for a parallel route slot (required for every slot in Next.js 16) |

| Folder pattern | Example | Matches |
|---|---|---|
| `segment` | `app/about/page.tsx` | `/about` |
| `[param]` | `app/listings/[id]/page.tsx` | `/listings/42` |
| `[...slug]` | `app/docs/[...slug]/page.tsx` | `/docs/a` and `/docs/a/b`, not `/docs` |
| `[[...slug]]` | `app/shop/[[...slug]]/page.tsx` | `/shop` and `/shop/a/b` |
| `(group)` | `app/(marketing)/pricing/page.tsx` | `/pricing` (the group is not in the URL) |
| `_folder` | `app/_components/` | nothing: private, never routed |
| `@slot` | `app/dashboard/@stats/page.tsx` | rendered by the parent layout as the `stats` prop |
| `(.)folder` | `app/@modal/(.)photos/[id]/page.tsx` | intercepts `/photos/[id]` during client navigation |

```tsx
// app/marketplace/layout.tsx: shared by every /marketplace/* page
export default function MarketplaceLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-[240px_1fr]">
      <CategorySidebar />   {/* stays mounted, keeps scroll and state while children change */}
      <main>{children}</main>
    </div>
  );
}
```

```tsx
// app/marketplace/loading.tsx
export default function Loading() {
  return <p aria-busy="true">Loading listings...</p>;
}

// app/marketplace/error.tsx
'use client';

export default function MarketplaceError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  return (
    <div role="alert">
      <p>Something went wrong loading the marketplace.</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

In production, error messages thrown on the server are replaced by a generic message plus a `digest` you can match in server logs, so internals do not leak.

**Parallel and intercepting routes, briefly.** Slots (`@stats`, `@activity`) let one layout render several independent pages side by side, each with its own loading and error states. Interception lets `/photos/42` open as a modal over the feed during client navigation while a direct visit or refresh renders the full page; it combines a slot (`@modal`) with an intercepting folder (`(.)photos/[id]`).

## 5. Dynamic segments and static params

```tsx
// app/marketplace/[slug]/page.tsx
type Props = { params: Promise<{ slug: string }>; searchParams: Promise<{ tab?: string }> };

export async function generateStaticParams() {
  const listings = await getFeaturedListings();          // prebuilt at build time
  return listings.map((l) => ({ slug: l.slug }));
}

export default async function ListingPage({ params, searchParams }: Props) {
  const [{ slug }, { tab = 'overview' }] = await Promise.all([params, searchParams]);
  const listing = await getListingBySlug(slug);
  if (!listing) notFound();
  return <ListingView listing={listing} tab={tab} />;
}
```

- `generateStaticParams` prebuilds the listed paths; others render on first request (or 404 with `dynamicParams = false` in the previous caching model).
- Reading `searchParams` makes the page dynamic, so it renders per request.
- Catch-all params are arrays: `[...slug]` gives `{ slug: string[] }`.
- In Client Components, read params with `useParams()` or unwrap the promise prop with React's `use()`.

## 6. Server and Client Components in Next.js

The full model is in [React 19 and Server Components](../06-react/09-react-19-and-server-components.md). The Next.js-specific rules:
- Files in `app/` are Server Components by default. `'use client'` marks a module and everything it imports as client code.
- Server Components can be `async`, read the database, use secrets and import heavy libraries without shipping them. They cannot use `useState`, `useEffect`, context, event handlers or browser APIs.
- Client Components still render to HTML on the server for the first load, then hydrate. Browser-only code goes in effects or event handlers.
- Data flows down through serializable props; mutations flow back through Server Actions or API calls.
- `import 'server-only'` at the top of database and auth modules makes the build fail if a Client Component imports them.

```tsx
// app/providers.tsx: context providers must be Client Components
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState, type ReactNode } from 'react';

export function Providers({ children }: { children: ReactNode }) {
  const [queryClient] = useState(() => new QueryClient()); // one client per browser session
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
// app/layout.tsx renders <Providers>{children}</Providers>; the children can still be Server Components
```

Errors you will meet:

| Error | Cause | Fix |
|---|---|---|
| "useState only works in a Client Component" | a hook in a Server Component | move the interactive part into a small `'use client'` component |
| "Functions cannot be passed directly to Client Components" | passing an `onClick` handler from a Server Component | define the handler inside the Client Component, or pass a Server Action |
| `createContext` errors | a provider rendered as a Server Component | wrap providers in a Client Component as above |
| Secrets or DB code in the browser bundle | a client module imported a server module | `server-only`, and keep data access in server files |
| Hydration mismatch | `Date.now()`, locale formatting or `window` checks during render | identical first render; browser-only values in effects |

## 7. Navigation and prefetching

```tsx
'use client';

import Link from 'next/link';
import { usePathname, useRouter, useSearchParams } from 'next/navigation';

export function CategoryNav({ categories }: { categories: string[] }) {
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const router = useRouter();

  const select = (category: string) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set('category', category);
    params.delete('page');
    router.push(`${pathname}?${params.toString()}`);     // soft navigation: layouts keep their state
  };

  return (
    <nav>
      <Link href="/marketplace" aria-current={pathname === '/marketplace' ? 'page' : undefined}>All</Link>
      {categories.map((c) => <button key={c} onClick={() => select(c)}>{c}</button>)}
    </nav>
  );
}
```

- Client-side (soft) navigation fetches only the RSC payload of the changed segments; shared layouts stay mounted.
- In production, `<Link>` prefetches routes as they enter the viewport (static routes fully, dynamic routes up to the nearest `loading.tsx`). Next.js 16 reworked prefetching to download shared layouts once and fetch only missing parts. There is no prefetching in development.
- `useRouter()` gives `push`, `replace`, `refresh` (re-fetch server data for the current route), `back`, `forward` and `prefetch`.
- On the server, `redirect('/login')` and `notFound()` work by throwing special errors, so do not wrap them in try/catch.
- A statically rendered page that calls `useSearchParams()` needs a `<Suspense>` boundary around that component, or the build fails.

## 8. Metadata and SEO

```tsx
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://www.example.com'),       // relative URLs below become absolute
  title: { default: 'Marketplace', template: '%s | Marketplace' },
  description: 'Rooms and services in shared houses.',
};
```

```tsx
// app/marketplace/[slug]/page.tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const listing = await getListingBySlug(slug);           // wrap in React cache() so the page reuses the result
  if (!listing) return { title: 'Listing not found' };
  return {
    title: listing.title,                                  // "Room in Brussels | Marketplace"
    description: listing.summary,
    alternates: { canonical: `/marketplace/${slug}` },
    openGraph: { title: listing.title, images: [listing.coverUrl] },
  };
}
```

- `app/sitemap.ts` and `app/robots.ts` generate `sitemap.xml` and `robots.txt` from code; `opengraph-image.tsx` renders share images.
- Structured data: render a `<script type="application/ld+json">` with `JSON.stringify(data)`, escaping `<`.
- Content must be in the server-rendered HTML for reliable indexing: keep SEO-critical content in Server Components, not behind client fetches.

## 9. next/image, next/font and next/script

```tsx
import Image from 'next/image';

<Image
  src={listing.coverUrl}
  alt={listing.title}
  width={640}
  height={420}                                  // reserves space: no layout shift
  sizes="(max-width: 768px) 100vw, 640px"       // lets the browser pick the right srcset entry
/>
```

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  images: { remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com' }] },
  output: 'standalone',
};
export default nextConfig;
```

- `next/image` resizes on demand, serves WebP or AVIF, lazy-loads by default and requires dimensions (or `fill` with a sized parent). For the largest above-the-fold image, use `loading="eager"` or `fetchPriority="high"` (the old `priority` prop is deprecated in Next.js 16 in favor of `preload`). Next.js 16 also changed some image defaults, such as cache TTL and allowed qualities.
- `next/font/google` and `next/font/local` download fonts at build time and self-host them, with fallback metrics that prevent layout shift.

```tsx
import { Inter } from 'next/font/google';
import Script from 'next/script';

const inter = Inter({ subsets: ['latin'], display: 'swap' }); // apply with className={inter.className}

<Script src="https://analytics.example.com/script.js" strategy="afterInteractive" />
// strategies: beforeInteractive (root layout only), afterInteractive (default), lazyOnload
```

## 10. Environment variables

| File | Loaded when |
|---|---|
| `.env` | always |
| `.env.development`, `.env.production`, `.env.test` | for that `NODE_ENV` |
| `.env.local`, `.env.development.local`, and so on | local overrides, not committed (`.env.local` is skipped in tests) |

Real environment variables win over files.

- Server code reads any variable through `process.env`. Client code sees only `NEXT_PUBLIC_*` variables, and their values are **inlined at `next build`**.
- Consequence: a Docker image built once cannot pick up a new `NEXT_PUBLIC_API_URL` at runtime. Rebuild per environment, or read the value on the server and pass it down.
- Never put secrets in `NEXT_PUBLIC_` variables. Server-only variables read in a Client Component are simply `undefined`.
- Validate env with Zod at startup ([TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md)).

## 11. Next.js 15 and 16 changes to know

**Next.js 15 (October 2024):**
- Async request APIs: `await params`, `await searchParams`, `await cookies()`, `await headers()`, `await draftMode()`. A codemod handles most upgrades. Dynamic data is requested only when actually awaited, which lets more of a page be prepared ahead of the request.
- Caching defaults: `fetch` requests, GET Route Handlers and client-side navigations are no longer cached by default (details in [data fetching and caching](02-nextjs-data-fetching-and-caching.md)).
- React 19 support and a stable Turbopack dev server.

**Next.js 16 (October 2025), hedged; read the upgrade guide:**
- Turbopack is the default bundler for dev and build (`--webpack` to opt out).
- Cache Components (`cacheComponents: true`) with `"use cache"`, `cacheLife` and `cacheTag`; caching becomes explicit and opt-in, and partial prerendering is the default model.
- `middleware.ts` becomes `proxy.ts` with an exported `proxy` function on the Node.js runtime; `middleware.ts` remains only for Edge use and is deprecated.
- `revalidateTag(tag, profile)` takes a cache-life profile; `updateTag()` and `refresh()` were added for Server Actions.
- Removed: synchronous access to request APIs, AMP, `next lint` (run ESLint or Biome directly). Node.js 20.9+ and TypeScript 5.1+ required. Parallel route slots need `default.tsx`.

## 12. Deployment

| Option | How | Notes |
|---|---|---|
| Vercel | push to Git | zero config: ISR, image optimization, preview deployments |
| Node server | `next build` then `next start` | any VM or PaaS; supports every feature |
| Docker | `output: 'standalone'` | copies only the files and dependencies the server needs |
| Static export | `output: 'export'` | plain files on a CDN; no per-request SSR, Server Actions, ISR or default image optimization |

```text
# Dockerfile with output: 'standalone'
FROM node:24-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Self-hosting several instances: the default cache lives on each instance's disk or memory, so configure a shared cache handler for consistent ISR; set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` so every instance can decrypt Server Action payloads; image optimization runs on your server.

## Common pitfalls
- `'use client'` at the top of every page, shipping the whole tree as client JavaScript.
- Importing `useRouter` from `next/router` inside the App Router (use `next/navigation`).
- Fetching your own Route Handler from a Server Component: an unnecessary HTTP hop; call the data function directly.
- Secrets in `NEXT_PUBLIC_` variables, or expecting `NEXT_PUBLIC_` values to change at runtime in Docker.
- Forgetting `await params` after upgrading to Next.js 15.
- Hydration mismatches from dates, random values or `window` checks during render.
- `useSearchParams()` without a Suspense boundary in a static route.
- A plain `<img>` (or missing dimensions) for the hero image, hurting LCP and CLS.
- Auth checks only in middleware/proxy, with nothing in pages, actions or route handlers.
- `error.tsx` without `'use client'`, or wrapping `redirect()` in try/catch.
- Context providers rendered directly in a Server Component layout.

## How to talk about this in an interview
- Anchor on RemoteDev: you built and integrated backend APIs with Next.js (TypeScript) and MongoDB. Explain where the API code lived, how pages consumed it, and which rendering mode the pages used.
  > **Fill in:** App Router or Pages Router; which resources or features you built; whether pages were SSR, static or client-rendered; how the app was deployed.
  > **Fill in:** one problem you hit (hydration error, caching surprise, environment variable issue) and how you fixed it.
- Show judgment: "I pick the rendering mode per route: static or ISR for public SEO pages, dynamic for personalized ones, client fetching for highly interactive parts."
- Contrast with TanStack Start, which you see at Devntech: Next.js is server-first with Server Components; Start is client-first with isomorphic loaders and server functions. See [TanStack Start](05-tanstack-start.md) and the [RemoteDev deep dive](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md).
- Keep version claims pinned: "Next.js 15 made request APIs async; Next.js 16 renamed middleware to proxy and introduced Cache Components."

## Interview questions

**Q1. What does Next.js add on top of React?**
File-based routing, server rendering (SSR, SSG, ISR, streaming, Server Components), data fetching with caching, backend endpoints (Route Handlers, Server Actions), image, font and script optimization, and a production build and server. React alone only renders components.
*Follow-up:* When would you not use Next.js? An internal SPA behind login with a separate API, where Vite plus a router is simpler.

**Q2. Explain CSR, SSR, SSG and ISR, and how you choose.**
CSR renders in the browser after JavaScript loads; SSR builds HTML per request; SSG builds it once at build time; ISR serves static HTML and regenerates it after a revalidate period or on demand. Choose per route: SSG or ISR for public, cacheable pages; SSR for personalized or always-fresh pages that need SEO; CSR for interactive areas behind login.
*Follow-up:* Where does streaming fit? It is SSR that sends the shell first and fills Suspense boundaries as data resolves.

**Q3. App Router vs Pages Router?**
The App Router uses Server Components by default, nested persistent layouts, `loading`/`error` files, streaming and Server Actions, and fetches data inside components. The Pages Router uses page-level `getServerSideProps`/`getStaticProps`, Client Components only and `pages/api` endpoints. They coexist, so migration can be incremental.
*Follow-up:* What replaces `getServerSideProps`? An async Server Component that awaits its data, rendered dynamically.

**Q4. Layout vs template?**
A layout wraps its segment's pages and persists across navigations, keeping state and not re-rendering. A template wraps the same way but re-mounts on every navigation, resetting state and re-running effects (useful for enter animations or per-page logging).
*Follow-up:* Which is required? The root layout, which renders `<html>` and `<body>`.

**Q5. How do `loading.tsx` and `error.tsx` work?**
`loading.tsx` wraps the segment's page in a Suspense boundary, so its UI shows instantly while the page streams. `error.tsx` is a Client Component error boundary for the segment, receiving `error` and `reset`; errors in the root layout need `global-error.tsx`.
*Follow-up:* Why must `error.tsx` be a Client Component? Error boundaries rely on client-side React state and the `reset` callback.

**Q6. What can Server Components not do?**
Use state or effects, attach event handlers, read context or touch browser APIs. They can be async, access databases and secrets, and render Client Components, whose props must be serializable.
*Follow-up:* How do you give a Server Component interactivity? Extract the interactive part into a small `'use client'` component.

**Q7. Where do you put `'use client'`?**
At the top of the smallest modules that need interactivity (buttons, forms, charts, providers), because everything those modules import also becomes client code. Pages and layouts stay Server Components and pass data down.
*Follow-up:* Can a Client Component render a Server Component? Not by importing it, but it can receive one as `children` or another prop.

**Q8. How does data get from server to client components, and what cannot be passed?**
As props: strings, numbers, booleans, plain objects and arrays, dates, maps, sets, promises (unwrapped with `use()`), JSX and Server Actions. Not regular functions, class instances or symbols. Pass minimal DTOs, never raw database documents.
*Follow-up:* Why minimal DTOs? Everything passed is serialized into the page and visible to users.

**Q9. How does navigation work in the App Router?**
`<Link>` and `useRouter().push` perform client-side navigation: Next.js fetches the RSC payload for the changed segments and merges it, keeping shared layouts mounted. Links prefetch in production as they enter the viewport; `router.refresh()` refetches server data for the current route.
*Follow-up:* Which package do the hooks come from? `next/navigation` in the App Router; `next/router` is the Pages Router API.

**Q10. How do you handle SEO in Next.js?**
Render content on the server; export `metadata` or `generateMetadata` (title templates, descriptions, canonical URLs, Open Graph); generate `sitemap.xml` and `robots.txt` with `sitemap.ts` and `robots.ts`; add JSON-LD structured data; keep URLs stable; optimize LCP and CLS with `next/image` and `next/font`.
*Follow-up:* Why `metadataBase`? It turns relative canonical and Open Graph URLs into absolute ones.

**Q11. Why use `next/image` and `next/font`?**
`next/image` resizes images on demand, serves modern formats, lazy-loads and reserves space to avoid layout shift. `next/font` self-hosts fonts at build time with size-adjusted fallbacks, removing layout shift and third-party font requests.
*Follow-up:* What is needed for remote images? An `images.remotePatterns` entry in `next.config`.

**Q12. How do environment variables work in Next.js, and what is the `NEXT_PUBLIC_` gotcha?**
Server code can read any variable; only `NEXT_PUBLIC_*` reaches the browser, and those values are inlined at build time. Changing them later in a container does nothing until you rebuild, and anything with that prefix is public.
*Follow-up:* How do you give the client a runtime-configurable value? Read it on the server and pass it down as a prop.

**Q13. What changed in Next.js 15 around `params` and `cookies()`, and why?**
They became asynchronous (`await params`, `await cookies()`), along with `searchParams`, `headers()` and `draftMode()`. Making request data async lets Next.js render and prepare everything that does not depend on the request before awaiting it. Next.js 16 removed the temporary synchronous access.
*Follow-up:* How do you upgrade a large codebase? Run the official codemod, then fix the remaining cases.

**Q14. How do dynamic routes, catch-all routes and `generateStaticParams` work?**
`[id]` captures one segment, `[...slug]` one or more as an array, `[[...slug]]` zero or more. `generateStaticParams` returns the params to prebuild at build time; other values render on demand.
*Follow-up:* What makes such a page dynamic anyway? Reading cookies, headers, searchParams or uncached data.

**Q15. What are route groups, parallel routes and intercepting routes for?**
Route groups `(name)` organize files or apply different layouts without changing URLs. Parallel routes (`@slot`) render several pages in one layout with independent loading and error states. Intercepting routes (`(.)photos/[id]`) show a route in the current context, such as a modal, during client navigation, while a direct visit shows the full page.
*Follow-up:* What did Next.js 16 change for slots? Every slot needs an explicit `default.tsx`.

**Q16. How do you deploy Next.js outside Vercel?**
Run `next build` and `next start` on a Node host, or build with `output: 'standalone'` and copy `.next/standalone`, `.next/static` and `public` into a slim Docker image. With several instances, configure a shared cache handler and a fixed Server Actions encryption key. `output: 'export'` produces a static site without server features.
*Follow-up:* What is lost with static export? Per-request SSR, Server Actions, ISR, Route Handlers beyond static GETs, and default image optimization.

**Q17. What changed in Next.js 16?**
Turbopack became the default bundler; Cache Components introduced explicit caching with `"use cache"`; `middleware.ts` was renamed `proxy.ts` and runs on Node.js; `revalidateTag` gained a cache-life argument and `updateTag`/`refresh` arrived for Server Actions; `next lint` and synchronous request APIs were removed; Node.js 20.9+ is required. Check the upgrade guide for details.
*Follow-up:* Is the old caching model gone? No: it still applies when Cache Components is not enabled.

## Further reading
- [Next.js: App Router documentation](https://nextjs.org/docs/app)
- [Next.js: Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Next.js: Project structure and file conventions](https://nextjs.org/docs/app/getting-started/project-structure)
- [Next.js 16 release post](https://nextjs.org/blog/next-16)
- [Next.js: Upgrading to version 16](https://nextjs.org/docs/app/guides/upgrading/version-16)
