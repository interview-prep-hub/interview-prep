# React 19 and Server Components

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [Data fetching](04-data-fetching-tanstack-query.md), [Forms](05-forms-and-validation.md), [Routing](08-routing.md), [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [Next.js data fetching](../07-nextjs-and-tanstack/02-nextjs-data-fetching-and-caching.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [React on Rails](../05-ruby-on-rails/16-react-on-rails.md)
> **On your resume:** React 19 (December 2024) is what new Expo SDKs and frameworks ship with, so it matters for the Coloc resident app and the TanStack Start platform at Devntech; Server Components and Server Functions come up with Next.js (RemoteDev), and SSR and hydration apply to React on Rails pages rendered on the server.

## Quick revision
- React 19 stable (December 2024): Actions, `useActionState`, `useOptimistic`, `useFormStatus` (from `react-dom`), `use`, `ref` as a prop, `<Context>` as a provider, document metadata tags, resource preloading APIs, and Server Components and Server Functions stable for frameworks.
- An Action is a function run inside a transition (it may be async): React tracks its pending state, routes thrown errors to error boundaries, and coordinates optimistic updates.
- `<form action={fn}>` calls `fn(formData)` and resets uncontrolled fields after success.
- `useActionState(action, initialState)` returns `[state, formAction, isPending]`; the action receives `(previousState, formData)`.
- `useOptimistic(value, updateFn)` shows a temporary value while an Action runs and falls back to the real value when it ends.
- `useFormStatus()` reads the parent `<form>`'s `pending`, `data`, `method` and `action`; it must be called in a component rendered inside that form.
- `use(promise)` suspends until the promise resolves; `use(context)` reads context. Unlike other hooks, `use` may be called conditionally.
- Removed: `ReactDOM.render` and `hydrate`, `findDOMNode`, string refs, legacy context, `propTypes` and `defaultProps` on function components; `act` now comes from `react`.
- Server Components run only on the server (at build or request time), can be `async` and read data directly, and send no JavaScript for themselves; they cannot use state, effects, event handlers or browser APIs.
- `'use client'` marks a module as a client entry point: it and everything it imports become client code. Client Components still render to HTML on the server and then hydrate.
- `'use server'` marks Server Functions that the client can call (a POST under the hood). They are public endpoints: validate input and check authorization in every one.
- Props passed from a Server to a Client Component must be serializable: no functions (except Server Functions) and no class instances.
- A Server Component can be passed as `children` to a Client Component; a Client Component cannot import a Server Component.
- Streaming SSR sends the HTML shell first and streams each `<Suspense>` boundary's content as its data resolves; selective hydration hydrates parts independently.
- Hydration attaches React to server-rendered HTML; the first client render must produce the same markup or you get a hydration error.
- Common hydration mismatches: dates, `Math.random()`, locale formatting, `typeof window` branches, invalid HTML nesting, browser extensions.
- React 19.2 (hedge, check the release notes) added `<Activity>` and `useEffectEvent`.

## 1. Actions and async transitions

In React 18, transitions had to be synchronous. React 19 lets `startTransition` take an async function. React calls these "Actions": while one runs, `isPending` is true, the UI stays responsive, thrown errors reach the nearest error boundary, and optimistic state can be shown.

```tsx
import { useState, useTransition } from 'react';

function RenameHouse({ houseId }: { houseId: string }) {
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  function handleSave() {
    startTransition(async () => {
      const err = await renameHouse(houseId, name); // resolves to an error message or null
      startTransition(() => {
        setError(err); // updates after an await need their own startTransition (current limitation)
      });
    });
  }

  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button onClick={handleSave} disabled={isPending}>{isPending ? 'Saving...' : 'Save'}</button>
      {error && <p role="alert">{error}</p>}
    </>
  );
}
```

Actions also plug into forms: pass a function to `<form action={fn}>` (or `formAction` on a button) and React calls it with the form's `FormData` inside a transition, then resets the form's uncontrolled fields when it succeeds. By convention, props that accept Actions are named `action` or end in `Action`.

## 2. useActionState

`useActionState` wraps an Action and keeps its last result as state, which is what most forms need: pending flag, errors, success message.

```tsx
import { useActionState } from 'react';

type SaveState = { error: string | null; saved: boolean };

async function saveProfile(_prev: SaveState, formData: FormData): Promise<SaveState> {
  const name = String(formData.get('name') ?? '').trim();
  if (name.length < 2) return { error: 'Name is too short', saved: false };
  const res = await fetch('/api/profile', { method: 'PATCH', body: formData });
  return res.ok ? { error: null, saved: true } : { error: 'Could not save, try again', saved: false };
}

export function ProfileForm() {
  const [state, formAction, isPending] = useActionState(saveProfile, { error: null, saved: false });
  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>{isPending ? 'Saving...' : 'Save'}</button>
      {state.error && <p role="alert">{state.error}</p>}
      {state.saved && <p>Saved</p>}
    </form>
  );
}
```

- Signature: `const [state, formAction, isPending] = useActionState(action, initialState, permalink?)`. The action receives the previous state first, then the form data, and returns the next state (sync or async).
- Repeated submissions are queued and run in order, each receiving the previous result.
- With Server Functions in a framework, the form can submit before JavaScript has loaded (progressive enhancement); the optional `permalink` supports that case.
- It was called `useFormState` in `react-dom` during the canary period; the stable hook comes from `react`.

## 3. useOptimistic

`useOptimistic` shows an immediate, temporary value while an Action is in flight. When the Action finishes, React switches back to the real value passed in (which should by then include the server's result).

```tsx
import { useOptimistic } from 'react';

type Message = { id: string; text: string; sending?: boolean };

function Chat({ messages, send }: { messages: Message[]; send: (text: string) => Promise<void> }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (current: Message[], text: string) => [...current, { id: `temp-${current.length}`, text, sending: true }],
  );

  async function formAction(formData: FormData) {
    const text = String(formData.get('text'));
    addOptimistic(text); // appears instantly, marked as sending
    await send(text);    // the parent refreshes `messages`; the optimistic copy is then dropped
  }

  return (
    <>
      <ul>
        {optimisticMessages.map((m) => (
          <li key={m.id}>{m.text} {m.sending && <small>(sending...)</small>}</li>
        ))}
      </ul>
      <form action={formAction}>
        <input name="text" required />
        <button>Send</button>
      </form>
    </>
  );
}
```

If `send` throws, the optimistic item disappears automatically when the Action ends, and the error goes to the nearest error boundary unless you catch it and show a message. The optimistic setter must be called inside an Action or transition. TanStack Query's `onMutate` pattern solves the same problem at the cache level ([data fetching](04-data-fetching-tanstack-query.md)).

## 4. useFormStatus

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton({ label }: { label: string }) {
  const { pending } = useFormStatus(); // status of the nearest parent <form>
  return <button type="submit" disabled={pending}>{pending ? 'Working...' : label}</button>;
}

// <form action={formAction}>
//   <input name="email" type="email" />
//   <SubmitButton label="Subscribe" />
// </form>
```

It reads the status of the form it is rendered inside, which makes reusable submit buttons possible without prop drilling. Calling it in the same component that renders the `<form>` does not work: that component is not inside the form, so `pending` stays `false`.

## 5. use()

`use` reads a resource during render: a promise or a context.

```tsx
import { Suspense, use } from 'react';

type PostComment = { id: string; body: string }; // not "Comment", which is a built-in DOM type

// Client Component (in its own 'use client' file in a real app)
function Comments({ commentsPromise }: { commentsPromise: Promise<PostComment[]> }) {
  const comments = use(commentsPromise); // suspends until resolved; a rejection goes to the error boundary
  return <ul>{comments.map((c) => <li key={c.id}>{c.body}</li>)}</ul>;
}

// Server Component: starts the request without awaiting it, so the page streams while comments load
export default function PostPage({ postId }: { postId: string }) {
  const commentsPromise = getComments(postId);
  return (
    <Suspense fallback={<p>Loading comments...</p>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

- It can be called inside conditions and loops, unlike other hooks, but not inside `try/catch` (use an error boundary).
- In a Client Component the promise must be stable (created by a Server Component, a cache or a data library). Creating a new promise during each client render makes the component suspend forever.
- `use(ThemeContext)` works like `useContext`, but may appear after an early return.
- For client-side fetching, a Suspense-ready library (`useSuspenseQuery`) is still more practical than hand-managed promises.

## 6. Other React 19 improvements

```tsx
// ref is a normal prop for function components: no forwardRef needed
function TextInput({ ref, ...props }: ComponentProps<'input'>) {
  return <input ref={ref} {...props} />;
}

// Ref callbacks may return a cleanup function
<div
  ref={(node) => {
    if (!node) return;
    const observer = new ResizeObserver(() => console.log(node.getBoundingClientRect().width));
    observer.observe(node);
    return () => observer.disconnect(); // runs when the node is removed
  }}
/>;

// The context itself can be rendered as the provider
const ThemeContext = createContext<'light' | 'dark'>('light');
<ThemeContext value="dark"><App /></ThemeContext>; // replaces <ThemeContext.Provider value="dark">
```

**Document metadata.** `<title>`, `<meta>` and `<link>` rendered anywhere in the tree are hoisted into `<head>`, on the client and in SSR:

```tsx
function BillPage({ bill }: { bill: Bill }) {
  return (
    <article>
      <title>{`${bill.title} | Bills`}</title> {/* one string; mixing text and expressions creates an array */}
      <meta name="description" content={`Bill details for ${bill.title}`} />
      <h1>{bill.title}</h1>
    </article>
  );
}
```

**Stylesheets and scripts.** `<link rel="stylesheet" href="..." precedence="default" />` lets React deduplicate stylesheets, order them by precedence and hold rendering until they load; `<script async src="...">` is deduplicated as well.

**Resource preloading** through functions from `react-dom`:

```tsx
import { preconnect, prefetchDNS, preinit, preload } from 'react-dom';

function AppShell() {
  preinit('https://cdn.example.com/widget.js', { as: 'script' });                                  // load and run now
  preload('/fonts/inter.woff2', { as: 'font', type: 'font/woff2', crossOrigin: 'anonymous' }); // fetch early
  preconnect('https://api.example.com');                                                          // open the connection
  prefetchDNS('https://images.example.com');                                                      // resolve DNS only
  return <Layout />;
}
```

**Also new:** clearer hydration error messages with a diff, full support for custom elements (Web Components), more tolerance for third-party scripts and extensions that inject tags into `<head>` and `<body>`, and `createRoot`/`hydrateRoot` options `onCaughtError`, `onUncaughtError` and `onRecoverableError` for central error reporting.

## 7. Removed and deprecated APIs

| Removed in React 19 | Use instead |
|---|---|
| `ReactDOM.render`, `ReactDOM.hydrate` | `createRoot`, `hydrateRoot` from `react-dom/client` |
| `unmountComponentAtNode` | `root.unmount()` |
| `findDOMNode` | refs |
| string refs (`ref="input"`) | `useRef` or callback refs |
| legacy context (`contextTypes`, `getChildContext`) | `createContext` |
| `propTypes` and `defaultProps` on function components | TypeScript and default parameters |
| `act` from `react-dom/test-utils` | `act` from `react` |
| `react-test-renderer/shallow` | Testing Library |
| UMD builds | ES module CDNs or a bundler |

TypeScript changes in the React 19 types: `useRef` requires an argument, the global `JSX` namespace moved to `React.JSX`, and ref callbacks must not implicitly return a value. `forwardRef` and `<Context.Provider>` still work but are expected to be deprecated. The official upgrade guide lists codemods for most of this.

## 8. React Server Components: what runs where

Server Components (RSC) are components that run only on the server, either at build time or per request. Their output is serialized into an "RSC payload" (a description of the rendered tree with placeholders for Client Components), which the framework streams to the browser and also uses to produce HTML. Their own code never ships to the client.

```text
Server (build time or per request)                     Browser
Server Components run: await DB queries, read files,   receives HTML (fast first paint) and
use secrets, import heavy libraries; output becomes    the RSC payload; React rebuilds the tree,
HTML + RSC payload, with slots for Client Components   loads JS only for Client Components and
                                                       hydrates them so they become interactive
```

```tsx
// app/houses/page.tsx: a Server Component (the default in the Next.js App Router)
import { db } from '@/db';
import { FavoriteButton } from './favorite-button';

export default async function HousesPage() {
  const houses = await db.query.houses.findMany(); // runs on the server; credentials never reach the browser
  return (
    <ul>
      {houses.map((h) => (
        <li key={h.id}>
          {h.name} <FavoriteButton houseId={h.id} /> {/* a client island receiving serializable props */}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// app/houses/favorite-button.tsx
'use client';

import { useState } from 'react';

export function FavoriteButton({ houseId }: { houseId: string }) {
  const [saved, setSaved] = useState(false);
  return (
    <button aria-pressed={saved} onClick={() => setSaved((s) => !s)} data-house={houseId}>
      {saved ? 'Saved' : 'Save'}
    </button>
  );
}
```

What Server Components buy you: zero client JavaScript for non-interactive UI (heavy markdown or date libraries stay on the server), data access without an API layer or client waterfalls, and secrets that stay server-side. What they cannot do: hold state, run effects, attach event handlers or touch browser APIs. RSC is a React architecture implemented by frameworks (Next.js App Router is the main one; others are adding support). You do not use it in a plain Vite SPA.

### Going deeper: navigation with Server Components

On the first request the framework sends HTML plus the RSC payload. On later client-side navigations it fetches only the RSC payload for the new route, and React merges it into the existing tree: shared layouts keep their client state (an open sidebar, a scroll position) while the changed segment re-renders. Server Functions can trigger the same refresh after a mutation (`revalidatePath` in Next.js).

## 9. 'use client', the serialization boundary and composition

`'use client'` at the top of a file marks a boundary in the module graph: that module and everything it imports become client code. Mark the leaves (a button, a form, a chart), not whole pages, to keep the client bundle small.

Props crossing from a Server Component to a Client Component must be serializable:
- **Allowed:** strings, numbers, bigints, booleans, `null`, `undefined`, plain objects and arrays of these, `Date`, `Map`, `Set`, typed arrays, promises, JSX elements, and Server Functions.
- **Not allowed:** regular functions (event handlers), class instances, symbols not registered with `Symbol.for`.

```tsx
// ClientShell is 'use client' (it holds sidebar state). ServerSidebar stays a Server Component,
// because the Server Component parent creates it and passes it through as children.
export default function Layout() {
  return (
    <ClientShell>
      <ServerSidebar />
    </ClientShell>
  );
}
```

Rules of composition:
- Server Components can import and render Client Components.
- Client Components cannot import Server Components (the import would turn them into client code), but they can receive them as `children` or other JSX props.
- `'use client'` does not mean "client-only": Client Components are still rendered to HTML on the server for the first load, then hydrated. Browser-only code belongs in effects or handlers.
- The `server-only` package (`import 'server-only'`) makes the build fail if a server module (database access, secrets) is imported into client code.

## 10. Server Functions and 'use server'

`'use server'` marks functions that run on the server but can be called from the client. The framework gives each one an endpoint; calling it from the browser sends a POST request with serialized arguments. When used with `action` props they are often called Server Actions.

```tsx
// app/houses/actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const noteSchema = z.object({ houseId: z.string().min(1), note: z.string().min(1).max(500) });

export async function addNote(formData: FormData) {
  const session = await getSession();             // your auth helper
  if (!session) throw new Error('Unauthorized');   // a public endpoint: always authenticate
  const input = noteSchema.parse({ houseId: formData.get('houseId'), note: formData.get('note') });
  await assertCanAccessHouse(session.userId, input.houseId); // and authorize
  await db.insert(notes).values({ ...input, authorId: session.userId });
  revalidatePath(`/houses/${input.houseId}`);       // refresh the server-rendered page
}
```

```tsx
// Used from a Server or Client Component
<form action={addNote}>
  <input type="hidden" name="houseId" value={house.id} />
  <textarea name="note" required />
  <SubmitButton label="Add note" />
</form>
```

- `'use server'` goes at the top of a file (every export becomes a Server Function) or at the top of an async function body. It does **not** mark Server Components; those need no directive.
- Treat every Server Function as an exposed API endpoint: validate inputs, authenticate, authorize (especially in multi-tenant apps), and never trust hidden form fields.
- Arguments and return values must be serializable.
- TanStack Start offers a similar idea with `createServerFn`, and oRPC procedures are another way to get typed server calls; see [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md).

## 11. Server Components vs Client Components

| | Server Component | Client Component |
|---|---|---|
| Marker | default in RSC frameworks | `'use client'` at the top of the module (or imported by one) |
| Runs | on the server only (build or request time) | on the server for the first HTML, then in the browser |
| Can be `async` | yes | no (use `use()` or a data library) |
| State, effects, event handlers | no | yes |
| Browser APIs | no | yes, in effects and handlers |
| Direct database, filesystem, secrets | yes | no |
| JavaScript sent to the browser | none for itself | its code and its dependencies |
| Imports the other kind | can render Client Components | cannot import Server Components; can receive them as props |
| Typical use | data fetching, layout, static or content-heavy UI | forms, modals, tabs, charts, anything interactive |

## 12. Streaming SSR with Suspense

Classic SSR is all-or-nothing: the server waits for every piece of data, sends the full HTML, the browser downloads all JavaScript, and hydration covers the whole page at once. Streaming SSR, built on `<Suspense>`, breaks that up.

```tsx
export default function HousePage({ houseId }: { houseId: string }) {
  return (
    <>
      <HouseHeader houseId={houseId} /> {/* fast: part of the first HTML chunk */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews houseId={houseId} /> {/* slow: streamed in when its data resolves */}
      </Suspense>
    </>
  );
}
```

- The server (`renderToPipeableStream` in Node, `renderToReadableStream` with Web Streams) sends the shell immediately, with fallbacks for pending boundaries.
- When a boundary's data is ready, the server streams its HTML plus a small inline script that swaps it into place, without waiting for the rest of the page.
- Selective hydration hydrates boundaries independently and prioritizes the part the user is interacting with.
- Frameworks wire this up for you: in Next.js, a `loading.tsx` file is a Suspense boundary for a route segment.

## 13. Hydration and hydration errors

Hydration is React attaching to HTML that the server already rendered: it renders the same tree on the client, reuses the existing DOM nodes and attaches event handlers (`hydrateRoot(container, <App />)` from `react-dom/client`). The first client render must produce the same markup as the server; otherwise React reports a hydration error and re-renders that part on the client, which costs performance and can cause flicker.

| Cause | Fix |
|---|---|
| `Date.now()`, `new Date()`, `Math.random()` in render | render a stable value first, update after mount; pass the value from the server |
| Locale or timezone formatting differs between server and browser | format with an explicit locale and time zone, or format on the client after mount |
| `typeof window !== 'undefined'` branches or `localStorage` reads during render | read browser-only values in an effect or with `useSyncExternalStore` and a server snapshot |
| Invalid HTML nesting (`<div>` inside `<p>`, `<a>` inside `<a>`) | fix the markup: the browser "repairs" it differently from React's tree |
| Browser extensions injecting attributes or elements | usually harmless; React 19 tolerates more of it in `<head>` and `<body>` |
| Different data on server and client | hydrate the client cache with the same data (for example TanStack Query's `HydrationBoundary`) |

```tsx
// Bug: the server's time and locale differ from the browser's, so the text does not match
function LastSynced() {
  return <p>Last synced at {new Date().toLocaleTimeString()}</p>;
}

// Fix: identical first render on both sides, then update after hydration
function LastSyncedFixed() {
  const [time, setTime] = useState<string | null>(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <p>Last synced at {time ?? '...'}</p>;
}
```

`suppressHydrationWarning` on an element silences the warning for its own text content (one level deep) when a mismatch is unavoidable, such as a timestamp. For components that cannot render on the server at all (a map widget), load them only on the client, for example with `next/dynamic` and `ssr: false` from a Client Component in Next.js. React 18 reported mismatches as separate warnings; React 19 logs a single error with a diff of the mismatch.

## 14. CSR vs SSR vs SSG vs RSC

| Strategy | Where and when HTML is produced | Where data is fetched | JavaScript shipped | Good for |
|---|---|---|---|---|
| CSR (client-side rendering) | in the browser, after the bundle loads | browser, after load | the whole app | dashboards behind login, internal tools (a Vite SPA) |
| SSR (server-side rendering) | on the server, per request | server, per request | the whole app, which hydrates | personalized pages that need SEO or a fast first paint |
| SSG (static generation) | at build time | at build time | the whole app, which hydrates | marketing pages, docs, blogs |
| ISR (Next.js) | at build time, regenerated in the background | periodically or on demand | same as SSG | large catalogs that change occasionally |
| RSC | on the server (build or request) as an RSC payload, usually plus SSR HTML | server, inside components | only Client Components | content-heavy apps with islands of interactivity |

SSR and RSC are different axes: SSR is about producing HTML on the server for the first load; RSC is about which components ever ship JavaScript. A Next.js App Router page usually uses both. React on Rails can server-render components into Rails views (classic SSR with hydration); see [React on Rails](../05-ruby-on-rails/16-react-on-rails.md).

## 15. React 19.2 additions (hedged)

React 19.2 (October 2025) added, among other things (check the release notes for the full and current list):

```tsx
// <Activity>: hide a subtree while keeping its state; its effects are cleaned up and
// its updates deprioritized while hidden, then restored when it becomes visible
<Activity mode={tab === 'chat' ? 'visible' : 'hidden'}>
  <ChatPanel /> {/* keeps scroll position and draft text when the user switches tabs */}
</Activity>
```

```tsx
// useEffectEvent: read the latest props or state inside an effect without making them dependencies
function ChatRoom({ roomId, theme }: { roomId: string; theme: string }) {
  const onConnected = useEffectEvent(() => {
    showToast(`Connected to ${roomId}`, theme); // always sees the current theme
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => onConnected());
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // changing the theme no longer reconnects
  // ...
}
```

The same release also added performance tracks for Chrome DevTools and server-side APIs such as `cacheSignal` and partial pre-rendering. Treat exact APIs as subject to change and verify in the docs before using them in an interview answer.

## Common pitfalls
- Putting `'use client'` at the top of every page, which ships everything to the browser and loses the benefits of RSC.
- Believing `'use client'` components are not server-rendered, then accessing `window` during render.
- Passing functions or class instances from Server to Client Components.
- Treating Server Functions as private: skipping validation and authorization.
- Using `'use server'` to mark Server Components.
- Calling `useFormStatus` in the component that renders the `<form>`.
- Creating promises during client renders and passing them to `use()`, causing endless suspense.
- Rendering dates, random values or `localStorage` data during the first render of SSR pages.
- Invalid HTML nesting that breaks hydration.
- Using removed APIs (`ReactDOM.render`, `defaultProps` on functions, string refs) when upgrading old code.

## How to talk about this in an interview
- Be precise with vocabulary: Actions (async transitions), Server Components (run only on the server, no client JS), Client Components (`'use client'`, still SSR'd and hydrated), Server Functions (`'use server'`, public endpoints). Precision here signals real understanding.
- Explain RSC by its benefits and limits, and add that it requires framework support; most of your React work so far may be client components, which is normal.
- Connect to your experience honestly: your Next.js work at RemoteDev was backend APIs with TypeScript and MongoDB, and TanStack Start (the framework under the new platform) offers SSR and server functions. Describe what the projects actually use, then show you know how RSC differs.

> **Fill in:** which React version each of your projects runs (the Colochousing React on Rails code, the Coloc resident app, the TanStack Start platform) and whether you used any React 19 features such as Actions, `useOptimistic` or ref as a prop.

> **Fill in:** whether any React on Rails components in the Colochousing app are server-rendered, and whether you have debugged a hydration mismatch.

- If you have not shipped Server Components in production, say so and walk through how you would split a page into server and client parts; reasoning beats pretending.

## Interview questions

**Q1. What are Actions in React 19?**
Functions run inside a transition, now allowed to be async. React tracks their pending state (`isPending` from `useTransition` or `useActionState`), keeps the UI responsive, sends thrown errors to error boundaries and coordinates optimistic updates. Passing a function to `<form action>` makes it an Action that receives `FormData` and resets uncontrolled fields after success.
*Follow-up:* Any limitation? State updates after an `await` inside the Action must be wrapped in another `startTransition` to count as part of it.

**Q2. Explain useActionState. What does the action receive?**
`const [state, formAction, isPending] = useActionState(action, initialState)`. The action receives the previous state and the submitted `FormData` and returns the next state, possibly asynchronously. You pass `formAction` to `<form action>`, render errors or success from `state`, and disable the button with `isPending`. Submissions queue in order.
*Follow-up:* What was it called before? `useFormState` in `react-dom` during the canary period.

**Q3. How does useOptimistic work, and what happens when the request fails?**
`const [optimistic, addOptimistic] = useOptimistic(value, updateFn)` returns `value` normally; calling `addOptimistic(input)` inside an Action renders `updateFn(current, input)` immediately. When the Action finishes, React renders the real `value` again. On failure there is nothing to undo: the optimistic value simply disappears, and you show the error.
*Follow-up:* How is that different from TanStack Query's optimistic updates? Those write to the shared cache and need manual rollback in `onError`.

**Q4. What is useFormStatus, and why must it live in a child of the form?**
A `react-dom` hook that returns `pending`, `data`, `method` and `action` for the nearest parent `<form>`, so a reusable `SubmitButton` can disable itself without props. It reads the form above it in the tree; a component that renders the `<form>` itself is not inside it, so the hook reports nothing.
*Follow-up:* Does it work with onSubmit handlers? It tracks forms submitted through Actions (`action` prop).

**Q5. What does use() do, and how is it different from other hooks?**
`use(promise)` suspends the component until the promise resolves (rejections go to error boundaries) and `use(context)` reads context. Unlike other hooks it can be called conditionally and in loops, though not in `try/catch`. Promises passed from a Client Component must be stable, typically created by a Server Component or a cache.
*Follow-up:* Why not create the promise inside the client component? Each render would create a new promise and the component would suspend forever.

**Q6. What changed about refs in React 19?**
Function components receive `ref` as a regular prop, so `forwardRef` is no longer needed (and is expected to be deprecated). Ref callbacks can return a cleanup function that runs when the element detaches. In TypeScript, `useRef` now requires an argument and ref callbacks must not implicitly return values.
*Follow-up:* What about `<Context.Provider>`? You can render `<Context value={...}>` directly; the old form still works for now.

**Q7. What are React Server Components, and what problem do they solve?**
Components that run only on the server, at build time or per request, can be `async` and read databases or files directly, and send no JavaScript for themselves; their output is streamed as an RSC payload and HTML. They cut client bundle size, remove client-side fetch waterfalls and API boilerplate for reads, and keep secrets on the server. They need a framework such as the Next.js App Router.
*Follow-up:* What can they not do? Use state, effects, event handlers or browser APIs.

**Q8. Server Components vs Client Components?**
Server Components are the default in RSC frameworks, run only on the server, may be async and access server resources, and ship no JavaScript. Client Components are marked with `'use client'`, render on the server for the first HTML and then hydrate in the browser, and can use state, effects and handlers. Server Components can render Client Components and pass Server Components to them as children, not the other way around.
*Follow-up:* How do you decide? Start on the server and move only interactive leaves to the client.

**Q9. What does 'use client' actually mean? Is that component not server-rendered?**
It marks a boundary in the module graph: this file and everything it imports are bundled for the client and may use client features. Client Components are still pre-rendered to HTML on the server for the initial load and then hydrated, so code that touches `window` during render still breaks.
*Follow-up:* Where should you put it? At the leaves, such as a button or a form, to keep the client bundle small.

**Q10. What does 'use server' do, and what are the security concerns?**
It marks Server Functions: server-side async functions the client can call, which the framework exposes as endpoints invoked with POST requests. Anyone can call them with any arguments, so validate input with a schema, authenticate, authorize the specific resource (tenant and ownership checks) and rate-limit where needed.
*Follow-up:* Does 'use server' mark Server Components? No; Server Components need no directive.

**Q11. What can you pass from a Server Component to a Client Component?**
Serializable values: primitives, plain objects and arrays, `Date`, `Map`, `Set`, typed arrays, promises, JSX elements (including Server Components as children) and Server Functions. Not regular functions such as event handlers, class instances or unregistered symbols. Define handlers inside the Client Component instead.
*Follow-up:* How do you give a Client Component data from the database? Fetch in the Server Component and pass plain data as props.

**Q12. How can a Client Component render a Server Component?**
It cannot import one, but it can receive one as `children` or another JSX prop: the Server Component parent renders `<ClientShell><ServerSidebar /></ClientShell>`. The server renders `ServerSidebar`, and the client component just places the result.
*Follow-up:* Why does importing not work? The import would pull the module into the client bundle, making it a Client Component.

**Q13. What is streaming SSR, and how does Suspense enable it?**
The server sends the HTML shell right away with fallbacks for unresolved `<Suspense>` boundaries, then streams each boundary's HTML (with a small script that swaps it in) as its data resolves. Combined with selective hydration, users see and use the fast parts of the page before the slow parts finish.
*Follow-up:* Which APIs implement it? `renderToPipeableStream` in Node and `renderToReadableStream` for Web Streams; frameworks call them for you.

**Q14. What is hydration? What causes hydration errors, and how do you fix them?**
Hydration is React attaching to server-rendered HTML by rendering the same tree on the client and wiring up event handlers. Errors happen when the first client render differs: times and random values, locale formatting, `window` or `localStorage` checks during render, invalid HTML nesting, extensions. Fix by rendering identical output first and updating in an effect, using explicit locales, fixing markup, or rendering browser-only widgets on the client only.
*Follow-up:* What does `suppressHydrationWarning` do? It silences the warning for one element's own text, for unavoidable cases such as timestamps.

**Q15. CSR vs SSR vs SSG vs RSC?**
CSR renders in the browser after the bundle loads (simple hosting, slow first paint, weak SEO). SSR renders HTML per request on the server and hydrates (fast first paint, personalized). SSG renders at build time (fastest, for static content). RSC decides which components run only on the server and ship no JavaScript; it is usually combined with SSR or static rendering.
*Follow-up:* Which would you pick for a logged-in dashboard? CSR or SSR with client components is fine; SEO does not matter there.

**Q16. Which legacy APIs did React 19 remove?**
`ReactDOM.render` and `ReactDOM.hydrate` (use `createRoot` and `hydrateRoot`), `unmountComponentAtNode`, `findDOMNode`, string refs, legacy context, `propTypes` and `defaultProps` on function components, `react-dom/test-utils`' `act` (now in `react`), the shallow renderer and the UMD builds. Codemods cover most migrations.
*Follow-up:* What replaces `defaultProps` on functions? Default values in destructuring.

**Q17. What are Activity and useEffectEvent?**
Both arrived in React 19.2 (verify details in the release notes). `<Activity mode="hidden">` hides a subtree while preserving its state, cleaning up its effects and deprioritizing its updates, which suits tabs and back navigation. `useEffectEvent` creates a function that reads the latest props and state from inside an effect without becoming a dependency, so non-reactive logic stops triggering re-subscriptions.
*Follow-up:* What did people do before `useEffectEvent`? Stored the latest callback in a ref updated by an effect.

## Further reading
- [React 19 release post](https://react.dev/blog/2024/12/05/react-19)
- [Server Components reference](https://react.dev/reference/rsc/server-components)
- ['use client' reference](https://react.dev/reference/rsc/use-client)
- ['use server' reference](https://react.dev/reference/rsc/use-server)
