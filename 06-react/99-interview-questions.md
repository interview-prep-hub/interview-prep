# React: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. The last section has eight live-coding exercises with complete TSX solutions; practice typing them from memory.

## Fundamentals (see [01-fundamentals.md](01-fundamentals.md))

**Q1. What is React in one sentence?**
A JavaScript library for building user interfaces from components, where the UI is a function of state and React updates the DOM to match whenever state changes.

**Q2. What does JSX compile to?**
Function calls: `jsx(type, props, key)` from `react/jsx-runtime` with the automatic runtime (React 17+), or `React.createElement(type, props, ...children)` with the classic runtime. Each call returns a plain element object.

**Q3. Why must component names be capitalized?**
JSX treats lowercase tags as DOM element strings (`'div'`) and capitalized tags as variable references, so `<myButton />` renders an unknown HTML element instead of calling your component.

**Q4. Props vs state?**
Props are read-only inputs from the parent; state is private data the component owns and changes with a setter, which triggers a re-render. Values derivable from either are neither: compute them during render.

**Q5. What is the virtual DOM?**
The in-memory tree of React elements, backed by React's Fiber tree. On each update React diffs the new tree against the previous one and applies only the differences to the real DOM.

**Q6. What heuristics does reconciliation use?**
A different element type at the same position replaces the whole subtree (state is lost); the same type keeps the DOM node or component instance and updates props; list children are matched by key. That makes diffing O(n).

**Q7. Why are keys needed, and why not use the array index?**
Keys identify list items across renders so each item keeps its own state. Index keys shift when items are inserted, removed or reordered, so state such as input text or focus ends up on the wrong row. Use stable ids from the data.

**Q8. What's the output?**
```tsx
function Badge({ count }: { count: number }) {
  return <div>{count && <span>{count} new</span>}</div>;
}
// <Badge count={0} />
```
`<div>0</div>`: `0 && ...` evaluates to `0`, and React renders numbers (React Native would crash on raw text). Write `count > 0 && ...`.

**Q9. Spot the bug.**
```tsx
<button onClick={handleDelete(bill.id)}>Delete</button>
```
`handleDelete` runs during every render and `onClick` receives its return value. Pass a function: `onClick={() => handleDelete(bill.id)}`.

**Q10. What is lifting state up?**
Moving shared state to the closest common parent of the components that need it, passing the value down as props and changes up through callback props.

**Q11. What are synthetic events, and what changed in React 17?**
React's cross-browser wrappers around native events, passed to camelCase handler props. React 17 attached listeners to the root container instead of `document` and removed event pooling.

**Q12. What does StrictMode do?**
In development only: renders components and state initializers twice, mounts-unmounts-remounts components (effects run setup, cleanup, setup) and warns about deprecated APIs, to expose impure renders and missing cleanups.

**Q13. How do you reset a component's state when an id changes?**
Render it with `key={id}`. A new key creates a new instance with fresh state, without any effect code.

**Q14. Spot the bug.**
```tsx
function EditName({ initialName }: { initialName: string }) {
  const [name, setName] = useState(initialName);
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
// The parent later passes a different initialName, but the input keeps the old text.
```
`useState` uses its argument only on the first render. Remount with a `key` from the parent when the record changes, or make the input fully controlled by the parent.

**Q15. How would you start a new React project today?**
Vite with the `react-ts` template for an SPA, or a framework when you need routing, data loading or SSR (Next.js, React Router framework mode, TanStack Start; Expo for mobile). Create React App is deprecated.

## Hooks (see [02-hooks.md](02-hooks.md))

**Q16. What are the rules of hooks, and why do they exist?**
Call hooks only at the top level of components or custom hooks (never in conditions, loops or after early returns) and only from React functions. React identifies each hook's state by its call order, which must be identical on every render.

**Q17. What's logged after one click?**
```tsx
const [count, setCount] = useState(0);
function handleClick() {
  setCount(count + 1);
  console.log(count);
}
```
`0`. State is a snapshot for the current render; the new value only appears in the next render.

**Q18. What is `count` after one click, starting from 0?**
```tsx
function handleClick() {
  setCount((c) => c + 1);
  setCount((c) => c + 1);
  setCount(count + 1);
}
```
`1`. The two updaters take the queued value to 2, then `setCount(count + 1)` replaces it with `0 + 1` from the render's snapshot.

**Q19. What is automatic batching in React 18?**
React groups state updates from the same tick into one render everywhere: event handlers, promises, `setTimeout` and native listeners. Before React 18 only updates inside React event handlers were batched. It requires `createRoot`; `flushSync` opts out.

**Q20. Explain the useEffect dependency array.**
No array: after every render. `[]`: once after mount, cleanup on unmount. `[a, b]`: after mount and whenever `a` or `b` changes by `Object.is`. List every reactive value the effect reads.

**Q21. When does an effect's cleanup run?**
Before the effect re-runs (with the previous values) and when the component unmounts. In development StrictMode also runs one extra cleanup and setup right after mount.

**Q22. Spot the bug.**
```tsx
useEffect(() => {
  const id = setInterval(() => setSeconds(seconds + 1), 1000);
  return () => clearInterval(id);
}, []);
```
Stale closure: `seconds` is frozen at its first value, so the display stops at 1. Use `setSeconds((s) => s + 1)`.

**Q23. Spot the bug.**
```tsx
function Results({ status, page }: { status: string; page: number }) {
  const [results, setResults] = useState<Result[]>([]);
  const params = { status, page };
  useEffect(() => {
    fetchResults(params).then(setResults);
  }, [params]);
  return <ResultList results={results} />;
}
```
`params` is a new object on every render, so the effect runs after every render, and each `setResults` causes another render: an endless fetch loop. Depend on `[status, page]`, build the object inside the effect, and ignore stale responses.

**Q24. In what order are these logged on mount, and on a re-render?**
```tsx
function Demo() {
  console.log('render');
  useEffect(() => {
    console.log('effect');
    return () => console.log('cleanup');
  });
  useLayoutEffect(() => {
    console.log('layout effect');
  });
  return null;
}
```
Mount: `render`, `layout effect`, `effect`. Re-render: `render`, `layout effect`, `cleanup`, `effect`. Layout effects run synchronously during the commit; passive effect cleanups and setups run afterwards. StrictMode adds extra calls in development.

**Q25. useRef vs useState?**
Changing state re-renders and each render sees a snapshot; changing a ref does not re-render and always holds the latest value. Use refs for DOM nodes, timer ids and flags that do not affect what is shown.

**Q26. useMemo vs useCallback vs React.memo?**
`useMemo` caches a computed value, `useCallback` caches a function identity (it equals `useMemo(() => fn, deps)`), and `React.memo` makes a component skip re-rendering when its props are shallow-equal. The first two mainly exist to keep identities stable for `memo` children and effect dependencies, or to skip expensive work.

**Q27. When do you prefer useReducer over useState?**
When several values change together, transitions are complex enough to be a small state machine, or you want the logic in a pure, testable reducer with a stable `dispatch` to pass down.

**Q28. What is a custom hook, and why write one?**
A function starting with `use` that calls other hooks to package reusable stateful logic (fetching, subscriptions, debouncing). It keeps components declarative and testable. Each caller gets its own state: hooks share logic, not state.

**Q29. Spot the bug.**
```tsx
function Profile({ user }: { user: User | null }) {
  if (!user) return <Login />;
  const [tab, setTab] = useState('posts');
  return <ProfileTabs value={tab} onChange={setTab} />;
}
```
`useState` runs after an early return, so it is skipped on some renders and breaks the rules of hooks. Move the hook above the `if`.

**Q30. Why can't the effect callback itself be async?**
An async function returns a promise, but React expects the effect to return nothing or a cleanup function. Define the async function inside the effect and call it.

**Q31. What is useLayoutEffect for?**
Measuring or adjusting the DOM after React updates it but before the browser paints, such as positioning a tooltip without flicker. It blocks painting, so `useEffect` remains the default.

**Q32. What is useId for?**
Generating unique ids that match between server and client, for `htmlFor`, `aria-describedby` and similar attributes. Never for list keys.

**Q33. useTransition vs useDeferredValue?**
`useTransition` marks a state update you trigger as non-urgent and gives `isPending`; `useDeferredValue` gives a lagging copy of a value. Both keep input responsive during expensive renders; neither debounces network requests.

**Q34. What is useSyncExternalStore for?**
Subscribing to stores outside React (browser APIs, state libraries) safely under concurrent rendering, with a server snapshot for SSR. Redux and Zustand use it internally.

**Q35. When do you not need an effect?**
For values derived from props or state (compute during render), for logic caused by a user action (put it in the event handler), and for resetting state on a prop change (use a `key`). Effects are for synchronizing with external systems.

## State management (see [03-state-management.md](03-state-management.md))

**Q36. What kinds of state exist in a React app?**
Local UI, shared client, server cache, URL and form state. Natural homes: component state, context or a store, TanStack Query, search params, and a form library.

**Q37. What is prop drilling, and how do you fix it?**
Passing props through components that only forward them. Fix it with composition first (pass elements as children or slots), then context, then a store.

**Q38. Lifting state vs Context vs Redux or Zustand: when do you use each?**
Lift state when a few nearby components share it. Use context for rarely changing values needed across the tree (theme, user, locale, current org). Use a store when shared client state changes often or has many writers: Zustand for simplicity, Redux Toolkit for large apps and strict conventions. Server data goes to TanStack Query in every case.

**Q39. What is the main performance pitfall of Context?**
Every consumer re-renders when the value changes, there are no selectors, and `memo` does not block it. A new object literal as the value re-renders all consumers on every provider render.

**Q40. Spot the bug.**
```tsx
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  return <AuthContext.Provider value={{ user, setUser }}>{children}</AuthContext.Provider>;
}
```
A new value object is created on every render of `AuthProvider`, re-rendering every consumer even when `user` did not change. Memoize it: `useMemo(() => ({ user, setUser }), [user])`.

**Q41. When is Redux Toolkit worth adding?**
For large shared client state updated from many places, flows that benefit from explicit actions, middleware and time-travel devtools, big teams that want one convention, or an existing Redux codebase.

**Q42. What does createSlice give you?**
Action creators and a reducer generated from one object of case reducers, with Immer so you can write "mutating" updates that produce immutable state.

**Q43. Why use createSelector?**
`useSelector` re-renders when the selected value changes by reference, and selectors that filter or map return new arrays every time. `createSelector` memoizes derived data so the reference stays stable until its inputs change.

**Q44. How does Zustand limit re-renders?**
Components subscribe with a selector (`useStore((s) => s.count)`) and re-render only when that value changes; `useShallow` compares multi-field selections shallowly. No provider is needed.

**Q45. Where should server data live?**
In a server-state cache such as TanStack Query or RTK Query, keyed by request, with staleness, refetching and invalidation built in. Not copied into `useState` or a client store.

**Q46. What belongs in the URL?**
State users expect to survive refresh, sharing and the back button: filters, sort order, pagination, tabs, search terms, the selected item.

## Data fetching and TanStack Query (see [04-data-fetching-tanstack-query.md](04-data-fetching-tanstack-query.md))

**Q47. Spot the bug.**
```ts
async function getBills(): Promise<Bill[]> {
  const res = await fetch('/api/bills');
  return res.json();
}
```
`fetch` resolves for 404 and 500 too, so error bodies are treated as data (and cached as such by a query library). Check `if (!res.ok) throw new Error(...)` before parsing.

**Q48. How do you prevent race conditions when fetching in an effect?**
Abort the previous request in the cleanup with an `AbortController`, or set an `ignore` flag, so a slow response for old parameters cannot overwrite newer data.

**Q49. What problems does TanStack Query solve?**
Caching, request deduplication, background refetching, retries, pagination and infinite queries, mutations with invalidation, optimistic updates, SSR hydration and loading or error state, all keyed by query key.

**Q50. staleTime vs gcTime?**
`staleTime` (default 0) is how long data counts as fresh, which suppresses refetches. `gcTime` (default 5 minutes) is how long unused data stays cached before garbage collection.

**Q51. What are TanStack Query's notable defaults?**
`staleTime: 0`, `gcTime` of 5 minutes, 3 retries with exponential backoff on the client, and refetching stale queries on mount, window focus and reconnect.

**Q52. isPending vs isFetching?**
`isPending`: no data yet. `isFetching`: the query function is running, including background refetches while old data stays on screen.

**Q53. Spot the bug.**
```ts
const { data } = useQuery({
  queryKey: ['bills'],
  queryFn: () => fetchBills(houseId),
});
```
`houseId` is missing from the key, so every house shares one cache entry and switching houses shows stale data without refetching. Use `queryKey: ['bills', houseId]`.

**Q54. invalidateQueries vs setQueryData?**
`invalidateQueries` marks matching queries stale (prefix match) and refetches active ones; `setQueryData` writes data into the cache directly. Invalidate for correctness; set data when the response already contains the exact new value.

**Q55. What are the steps of an optimistic update?**
`onMutate`: cancel queries for the key, snapshot the data, write the optimistic value, return the snapshot. `onError`: restore the snapshot. `onSettled`: invalidate to resync with the server.

**Q56. How do you make one query wait for another?**
Set `enabled: !!dependency` on the second query and include the dependency in its key; it runs as soon as the value exists.

**Q57. What does useInfiniteQuery require in v5?**
`initialPageParam` and `getNextPageParam` (return `undefined` or `null` when there are no more pages). Data arrives as `data.pages`, and you get `fetchNextPage`, `hasNextPage` and `isFetchingNextPage`.

**Q58. How do you avoid request waterfalls?**
Fetch in parallel, hoist fetching into route loaders (`ensureQueryData`), prefetch on hover, or add an endpoint that returns the combined data, instead of letting each component fetch after its parent renders.

## Forms (see [05-forms-and-validation.md](05-forms-and-validation.md))

**Q59. Controlled vs uncontrolled inputs?**
Controlled: React state holds the value (`value` + `onChange`). Uncontrolled: the DOM holds it and you read it with `FormData` or a ref, using `defaultValue` for the initial value.

**Q60. Spot the bug.**
```tsx
const [email, setEmail] = useState<string>();
return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
```
The initial value is `undefined`, so the input starts uncontrolled and becomes controlled on the first keystroke (React warns). Initialize with `useState('')`.

**Q61. Why use React Hook Form?**
It registers inputs through refs (uncontrolled), so typing does not re-render the form, and it provides schema validation through resolvers, error state, field arrays, focus management and server error helpers.

**Q62. When do you need React Hook Form's Controller?**
For components without a native input and ref: UI-library selects, date pickers, rich text editors and every React Native input.

**Q63. How do Zod and React Hook Form work together?**
`zodResolver(schema)` validates the form values with the schema, and `z.infer<typeof schema>` types the form, so rules and types come from one source that a TypeScript backend can reuse.

**Q64. Spot the bug.**
```tsx
<form onSubmit={handleSubmit(save)}>
  {fields.map((f, i) => <input key={f.id} {...register(`items.${i}.name`)} />)}
  <button onClick={() => append({ name: '' })}>Add item</button>
  <button type="submit">Save</button>
</form>
```
The "Add item" button has no `type`, so it defaults to `submit` and submits the form on every click. Add `type="button"`.

**Q65. How do you show server-side validation errors?**
Map the API's field errors onto the form with `setError(field, { type: 'server', message })`, and put general errors under `root` with `setError('root.server', ...)`.

**Q66. How do you prevent double submission?**
Disable the button while submitting (`isSubmitting`, `isPending`), guard the handler with a ref for very fast double clicks, and make the server idempotent.

**Q67. What makes form errors accessible?**
Visible labels tied to inputs, `aria-invalid` on invalid fields, `aria-describedby` linking the message, `role="alert"` so it is announced, and focus moved to the first invalid field.

## Performance (see [06-performance.md](06-performance.md))

**Q68. What triggers a re-render?**
A state change in the component (including a subscribed store value), a re-render of its parent, or a change in a context it reads.

**Q69. Does a component re-render when its props change?**
It re-renders because its parent re-rendered; new props come along with that. Props only decide for `memo` components, which skip rendering when props are shallow-equal.

**Q70. What does React.memo do, and what defeats it?**
It skips re-rendering when every prop is `Object.is`-equal to last time. New objects, arrays, inline functions or JSX passed on every render defeat it.

**Q71. Spot the bug.**
```tsx
const Row = memo(function Row({ item, onSelect }: RowProps) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }: { items: Item[] }) {
  const [selected, setSelected] = useState<string | null>(null);
  return <ul>{items.map((item) => <Row key={item.id} item={item} onSelect={() => setSelected(item.id)} />)}</ul>;
}
```
Every row gets a new `onSelect` function on every render, so `memo` never skips anything. Pass `setSelected` (stable) or a `useCallback` function that takes the id, which `Row` already supplies.

**Q72. What are "move state down" and "lift content up"?**
Keep state in the smallest component that uses it, and pass expensive subtrees as `children` so a wrapper's own state changes do not re-render them.

**Q73. How do you code split in React?**
`const Page = lazy(() => import('./Page'))` rendered inside `<Suspense fallback>`, split by route first, plus dynamic `import()` for heavy libraries inside the handlers that need them.

**Q74. How do you render a huge list efficiently?**
Virtualize it (TanStack Virtual, react-window; `FlatList` or FlashList in React Native) so only visible rows exist in the DOM, or paginate.

**Q75. Spot the bug.**
```tsx
{messages.map((m) => <Message key={Math.random()} message={m} />)}
```
A new key on every render makes React unmount and remount every message: state is lost and all DOM work is redone. Use `key={m.id}`.

**Q76. How do you find what is slow?**
Record the interaction in the React DevTools Profiler (with "record why each component rendered" enabled), check the browser Performance panel for long tasks, and confirm with a production build.

**Q77. What is the React Compiler?**
A build-time compiler, stable since late 2025, that automatically memoizes components and hooks that follow the Rules of React, making most manual `useMemo`, `useCallback` and `memo` unnecessary. It does not fix structural problems such as huge lists.

## Patterns and architecture (see [07-patterns-and-architecture.md](07-patterns-and-architecture.md))

**Q78. What are compound components?**
Components that share implicit state through context, such as `<Tabs>`, `<Tabs.List>`, `<Tabs.Tab>` and `<Tabs.Panel>`: callers control structure and markup, the logic stays inside.

**Q79. What is a render prop?**
A function prop that a component calls to render part of its output with its internal data, such as `render={({ field }) => <input {...field} />}`. Hooks replaced most uses.

**Q80. Why did hooks replace most higher-order components?**
HOCs cause wrapper hell, prop name collisions, unclear prop origins and awkward typing; hooks share the same logic with a plain function call inside the component.

**Q81. What do error boundaries catch, and what not?**
Errors thrown while rendering their children (render, lifecycle methods, constructors). Not errors in event handlers, async callbacks, server rendering or the boundary itself.

**Q82. How do events behave inside portals?**
They bubble through the React tree, not the DOM tree: React ancestors of the portal receive them even though its DOM node lives elsewhere. Context works normally.

**Q83. How do you pass a ref to a function component in React 19, and in React 18?**
React 19: `ref` is a regular prop you destructure and forward. React 18: wrap the component in `forwardRef`.

**Q84. How would you organize a large React codebase?**
By feature (`features/bills` with components, hooks, API layer and tests), a shared UI kit and helpers, thin route files, one-way dependencies and a public `index.ts` per feature.

**Q85. Spot the bug.**
```tsx
function Page() {
  const [query, setQuery] = useState('');
  const Filters = () => <input placeholder="City" />;
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Filters />
    </>
  );
}
```
`Filters` is a new component type on every render, so its input is destroyed and recreated on each keystroke in the search box, losing its text. Define `Filters` outside `Page`.

## Routing (see [08-routing.md](08-routing.md))

**Q86. How does client-side routing work?**
The router intercepts link clicks, updates the URL with `history.pushState`, and renders the matching component without a page load; it listens to `popstate` for back and forward.

**Q87. Why does refreshing a deep link return 404 after deployment?**
The server has no file at that path. Configure a fallback that serves `index.html` for unknown routes, or use hash routing or SSR.

**Q88. Why use route loaders?**
They fetch data before the route renders, in parallel for nested routes, which removes fetch-on-render waterfalls and centralizes loading and error handling.

**Q89. How do you protect routes?**
A guard layout route that renders `<Navigate to="/login" replace />` or `<Outlet />` (or a loader that throws `redirect`), while the API enforces authorization on every request.

**Q90. Link vs a plain anchor for internal navigation?**
`<Link>` navigates on the client without a reload, keeping app state; a plain `<a href>` triggers a full page load.

**Q91. How do you read and update query-string state?**
`const [searchParams, setSearchParams] = useSearchParams()`, then `searchParams.get('page')` and `setSearchParams(...)`. Parse and validate values: they are strings controlled by the user.

## React 19 and Server Components (see [09-react-19-and-server-components.md](09-react-19-and-server-components.md))

**Q92. Name the headline features of React 19.**
Actions (async transitions), `useActionState`, `useOptimistic`, `useFormStatus`, `use`, `ref` as a prop, `<Context>` as a provider, document metadata tags, resource preloading, and stable Server Components and Server Functions for frameworks.

**Q93. What does useActionState return?**
`[state, formAction, isPending]`. The action receives the previous state and the form data and returns the next state.

**Q94. What does useOptimistic do?**
It shows a temporary value while an Action runs and switches back to the real value automatically when the Action finishes or fails.

**Q95. Why does useFormStatus always report pending: false in my form component?**
It reads the nearest parent `<form>`, and the component that renders the form is not inside it. Call it from a child such as a `SubmitButton`.

**Q96. How is use() different from other hooks?**
It reads a promise (suspending until it resolves) or a context, and it may be called conditionally or in loops.

**Q97. Server Components vs Client Components?**
Server Components run only on the server, can be async, access data directly and ship no JavaScript. Client Components (`'use client'`) are server-rendered and then hydrated, and can use state, effects and event handlers.

**Q98. Does 'use client' mean the component renders only in the browser?**
No. It marks a client boundary for bundling; Client Components still render to HTML on the server for the first load, then hydrate.

**Q99. What is the main security concern with 'use server'?**
Server Functions are public endpoints callable with any arguments, so each one must validate input, authenticate and authorize.

**Q100. What causes hydration errors?**
The first client render differing from the server HTML: dates and random values, locale formatting, `window` or `localStorage` checks during render, invalid HTML nesting, or extensions modifying the DOM.

**Q101. CSR vs SSR vs SSG in one line each?**
CSR renders in the browser after the JavaScript loads; SSR renders HTML on the server per request and hydrates; SSG renders HTML at build time and hydrates.

## Testing (see [10-testing.md](10-testing.md))

**Q102. How do you test React components?**
With React Testing Library and Vitest or Jest: render with real children and providers, interact through user-event, query by role, label and text, assert on visible output, and mock the network with MSW. A few end-to-end tests cover critical journeys.

**Q103. getBy vs queryBy vs findBy?**
`getBy` throws when the element is missing, `queryBy` returns `null` (for asserting absence), `findBy` returns a promise and waits for async UI.

**Q104. Why prefer user-event over fireEvent?**
It simulates complete interactions (focus, pointer, keyboard and input events) and respects disabled elements, so it catches more real bugs. Its methods are async: `await` them.

**Q105. How do you mock APIs in component tests?**
With MSW handlers such as `http.get(url, () => HttpResponse.json(data))`, so components run their real fetching code; override per test with `server.use`.

**Q106. How do you set up TanStack Query in tests?**
A new `QueryClient` per test with `retry: false`, wrapped in `QueryClientProvider`, and `findBy*` queries for the loaded data.

**Q107. What does the "not wrapped in act(...)" warning mean?**
A state update happened after the test stopped waiting. Await the final UI with `findBy*` or `waitFor`, or control the async source with MSW or fake timers.

**Q108. Are snapshot tests useful?**
For small, stable serialized output, yes; for components they are brittle and assert no behavior, so prefer explicit assertions.

## Live-coding exercises

**Q109. Live coding: build a counter with a step input.**
Requirements: buttons to decrement, increment and reset; a number input for the step; reject empty, zero, negative or fractional steps.

```tsx
import { useState } from 'react';

export function StepCounter() {
  const [count, setCount] = useState(0);
  const [stepText, setStepText] = useState('1'); // raw text, so the user can clear the field while typing

  const step = Number(stepText);
  const stepValid = stepText.trim() !== '' && Number.isInteger(step) && step > 0;

  return (
    <div>
      <p aria-live="polite">Count: {count}</p>
      <label>
        Step{' '}
        <input
          type="number"
          min={1}
          value={stepText}
          onChange={(e) => setStepText(e.target.value)}
          aria-invalid={!stepValid}
        />
      </label>
      <button aria-label="Decrease" onClick={() => setCount((c) => c - step)} disabled={!stepValid}>-</button>
      <button aria-label="Increase" onClick={() => setCount((c) => c + step)} disabled={!stepValid}>+</button>
      <button onClick={() => setCount(0)}>Reset</button>
      {!stepValid && <p role="alert">Step must be a positive whole number</p>}
    </div>
  );
}
```

Notes: functional updates keep rapid clicks correct; the input stays a string in state (number inputs report strings, and an empty field must be allowed while typing); validity is derived during render instead of stored.

**Q110. Live coding: build a todo list with add, toggle, delete and filter.**
Requirements: add a todo from a form, toggle done, delete, filter by all, active and done, and show how many are left.

```tsx
import { useState, type FormEvent } from 'react';

type Todo = { id: string; title: string; done: boolean };
type Filter = 'all' | 'active' | 'done';

export function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [title, setTitle] = useState('');
  const [filter, setFilter] = useState<Filter>('all');

  function addTodo(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const trimmed = title.trim();
    if (!trimmed) return;
    setTodos((ts) => [...ts, { id: crypto.randomUUID(), title: trimmed, done: false }]); // id created in a handler, not in render
    setTitle('');
  }

  const toggle = (id: string) =>
    setTodos((ts) => ts.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));
  const remove = (id: string) => setTodos((ts) => ts.filter((t) => t.id !== id));

  const matches = (t: Todo) => filter === 'all' || (filter === 'done' ? t.done : !t.done);
  const visible = todos.filter(matches); // derived, never stored
  const remaining = todos.filter((t) => !t.done).length;

  return (
    <section>
      <form onSubmit={addTodo}>
        <label htmlFor="new-todo">New todo</label>
        <input id="new-todo" value={title} onChange={(e) => setTitle(e.target.value)} />
        <button type="submit">Add</button>
      </form>

      <div role="group" aria-label="Filter todos">
        {(['all', 'active', 'done'] as const).map((f) => (
          <button key={f} type="button" aria-pressed={filter === f} onClick={() => setFilter(f)}>
            {f}
          </button>
        ))}
      </div>

      <ul>
        {visible.map((t) => (
          <li key={t.id}>
            <label>
              <input type="checkbox" checked={t.done} onChange={() => toggle(t.id)} />
              <span style={{ textDecoration: t.done ? 'line-through' : 'none' }}>{t.title}</span>
            </label>
            <button type="button" onClick={() => remove(t.id)} aria-label={`Delete ${t.title}`}>
              Delete
            </button>
          </li>
        ))}
      </ul>
      <p>{remaining} left</p>
    </section>
  );
}
```

Notes: stable ids as keys (never the index, because items are deleted and filtered); immutable updates with `map` and `filter`; the filtered list and the counter are derived values. `crypto.randomUUID()` needs a secure context (HTTPS or localhost); a counter ref works too.

**Q111. Live coding: build a debounced user search that fetches from an API and aborts stale requests.**
Requirements: search after the user stops typing for 300 ms, only for 2+ characters, show loading and error states, never show results for an older query.

```tsx
import { useEffect, useState } from 'react';

type User = { id: number; name: string };

function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

export function UserSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query.trim(), 300);
  const [results, setResults] = useState<User[]>([]);
  const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle');

  useEffect(() => {
    if (debouncedQuery.length < 2) return; // nothing to fetch
    const controller = new AbortController();
    setStatus('loading');

    fetch(`/api/users?q=${encodeURIComponent(debouncedQuery)}`, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<User[]>;
      })
      .then((data) => {
        setResults(data);
        setStatus('idle');
      })
      .catch(() => {
        if (!controller.signal.aborted) setStatus('error'); // aborts come from newer queries: ignore them
      });

    return () => controller.abort(); // a newer query cancels this request
  }, [debouncedQuery]);

  const visible = debouncedQuery.length < 2 ? [] : results;

  return (
    <div role="search">
      <label htmlFor="user-search">Search users</label>
      <input id="user-search" type="search" value={query} onChange={(e) => setQuery(e.target.value)} />
      {status === 'loading' && <p aria-live="polite">Searching...</p>}
      {status === 'error' && <p role="alert">Search failed, try again</p>}
      <ul>
        {visible.map((u) => (
          <li key={u.id}>{u.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

Notes: the input stays controlled and instant; debouncing cuts requests; aborting in the cleanup prevents out-of-order responses; `encodeURIComponent` protects the query string. With TanStack Query: `useQuery({ queryKey: ['users', debouncedQuery], queryFn: ({ signal }) => ..., enabled: debouncedQuery.length >= 2 })`.

**Q112. Live coding: build an infinite scroll list.**
Requirements: load 20 items per page from a cursor-based API, load the next page when the user nears the bottom, avoid duplicate requests, handle errors with a retry button and stop when there are no more pages.

```tsx
import { useCallback, useEffect, useRef, useState } from 'react';

type Item = { id: number; title: string };
type Page = { items: Item[]; nextCursor: number | null };

async function fetchPage(cursor: number, signal: AbortSignal): Promise<Page> {
  const res = await fetch(`/api/items?cursor=${cursor}&limit=20`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<Page>;
}

export function InfiniteList() {
  const [items, setItems] = useState<Item[]>([]);
  const [cursor, setCursor] = useState<number | null>(0); // null means no more pages
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const loadingRef = useRef(false); // synchronous guard: the observer can fire again before state updates
  const controllerRef = useRef<AbortController | null>(null);
  const sentinelRef = useRef<HTMLDivElement>(null);

  const loadMore = useCallback(async () => {
    if (loadingRef.current || cursor === null) return;
    loadingRef.current = true;
    setLoading(true);
    setError(null);
    const controller = new AbortController();
    controllerRef.current = controller;
    try {
      const page = await fetchPage(cursor, controller.signal);
      setItems((prev) => [...prev, ...page.items]);
      setCursor(page.nextCursor);
    } catch (err) {
      if (!controller.signal.aborted) setError(err instanceof Error ? err.message : 'Failed to load');
    } finally {
      loadingRef.current = false;
      setLoading(false);
    }
  }, [cursor]);

  useEffect(() => () => controllerRef.current?.abort(), []); // abort an in-flight request on unmount

  useEffect(() => {
    const el = sentinelRef.current;
    if (!el || cursor === null) return;
    // Re-created whenever the cursor changes; a new observer reports the current intersection,
    // so a still-visible sentinel triggers the next page without any scrolling.
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0]?.isIntersecting) void loadMore();
      },
      { rootMargin: '200px' }, // start loading before the user reaches the very bottom
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, [loadMore, cursor]);

  return (
    <div>
      <ul>
        {items.map((item) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
      {loading && <p aria-live="polite">Loading...</p>}
      {error && (
        <p role="alert">
          {error} <button onClick={() => void loadMore()}>Retry</button>
        </p>
      )}
      {cursor === null ? <p>No more items</p> : <div ref={sentinelRef} aria-hidden="true" />}
    </div>
  );
}
```

Notes: a ref guards against duplicate requests because state updates are asynchronous; errors do not auto-retry (no infinite loop), the user retries; the observer is disconnected in the cleanup. In production, `useInfiniteQuery` replaces most of this state, and very long lists should also be virtualized.

**Q113. Live coding: build a modal that renders through a portal and closes on Escape.**
Requirements: render into `document.body`, close on Escape, backdrop click and a Close button, move focus into the dialog and restore it on close, lock background scroll.

```tsx
import { useEffect, useId, useRef, useState, type ReactNode } from 'react';
import { createPortal } from 'react-dom';

type ModalProps = { open: boolean; onClose: () => void; title: string; children: ReactNode };

export function Modal({ open, onClose, title, children }: ModalProps) {
  const dialogRef = useRef<HTMLDivElement>(null);
  const titleId = useId();

  // Escape closes the modal
  useEffect(() => {
    if (!open) return;
    function onKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose();
    }
    document.addEventListener('keydown', onKeyDown);
    return () => document.removeEventListener('keydown', onKeyDown);
  }, [open, onClose]);

  // Focus and scroll handling depend only on `open`, so a new onClose identity never steals focus
  useEffect(() => {
    if (!open) return;
    const previouslyFocused = document.activeElement as HTMLElement | null;
    const previousOverflow = document.body.style.overflow;
    document.body.style.overflow = 'hidden';
    dialogRef.current?.focus();
    return () => {
      document.body.style.overflow = previousOverflow;
      previouslyFocused?.focus(); // give focus back to the button that opened the modal
    };
  }, [open]);

  if (!open) return null;

  return createPortal(
    <div className="modal-backdrop" onMouseDown={onClose}>
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        tabIndex={-1}
        className="modal"
        onMouseDown={(e) => e.stopPropagation()} // presses inside the dialog must not reach the backdrop
      >
        <h2 id={titleId}>{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body,
  );
}

export function DeleteBillDemo() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>Delete bill</button>
      <Modal open={open} onClose={() => setOpen(false)} title="Delete this bill?">
        <p>This cannot be undone.</p>
      </Modal>
    </>
  );
}
```

Notes: the portal escapes parent `overflow` and `z-index`; React events still bubble through the React tree, hence `stopPropagation` on the dialog; listeners are removed in cleanups. Mention what is missing for production: a focus trap (Tab can leave the dialog), which the native `<dialog>` with `showModal()`, the `inert` attribute on the app root, or Radix Dialog provide.

**Q114. Live coding: build a validated sign-up form without libraries.**
Requirements: name, email, password (8+ characters with a digit) and confirmation; show errors after a field is blurred or on submit; disable the button while submitting; show a server error.

```tsx
import { useId, useState, type ChangeEvent, type FocusEvent, type FormEvent } from 'react';

type Values = { name: string; email: string; password: string; confirm: string };
type Field = keyof Values;
type Errors = Partial<Record<Field, string>>;

const initialValues: Values = { name: '', email: '', password: '', confirm: '' };

function validate(v: Values): Errors {
  const e: Errors = {};
  if (v.name.trim().length < 2) e.name = 'Name must be at least 2 characters';
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v.email)) e.email = 'Enter a valid email';
  if (v.password.length < 8) e.password = 'Password must be at least 8 characters';
  else if (!/\d/.test(v.password)) e.password = 'Password must contain a digit';
  if (v.confirm !== v.password) e.confirm = 'Passwords do not match';
  return e;
}

export function SignupForm({ onSignup }: { onSignup: (v: Values) => Promise<void> }) {
  const idPrefix = useId();
  const [values, setValues] = useState<Values>(initialValues);
  const [touched, setTouched] = useState<Partial<Record<Field, boolean>>>({});
  const [submitting, setSubmitting] = useState(false);
  const [serverError, setServerError] = useState<string | null>(null);

  const errors = validate(values); // derived on every render, never stored

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    const { name, value } = e.target;
    setValues((v) => ({ ...v, [name]: value }));
  }

  function handleBlur(e: FocusEvent<HTMLInputElement>) {
    const { name } = e.target;
    setTouched((t) => ({ ...t, [name]: true }));
  }

  async function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setTouched({ name: true, email: true, password: true, confirm: true }); // reveal every error
    if (Object.keys(errors).length > 0 || submitting) return;
    setSubmitting(true);
    setServerError(null);
    try {
      await onSignup(values);
      setValues(initialValues);
      setTouched({});
    } catch (err) {
      setServerError(err instanceof Error ? err.message : 'Sign-up failed');
    } finally {
      setSubmitting(false);
    }
  }

  // A render helper (a plain function call), not a component defined inside a component,
  // so inputs keep their identity and focus between renders
  function renderField(name: Field, label: string, type = 'text') {
    const id = `${idPrefix}-${name}`;
    const error = touched[name] ? errors[name] : undefined;
    return (
      <div>
        <label htmlFor={id}>{label}</label>
        <input
          id={id}
          name={name}
          type={type}
          value={values[name]}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={error ? true : undefined}
          aria-describedby={error ? `${id}-error` : undefined}
        />
        {error && <p id={`${id}-error`} role="alert">{error}</p>}
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      {renderField('name', 'Name')}
      {renderField('email', 'Email', 'email')}
      {renderField('password', 'Password', 'password')}
      {renderField('confirm', 'Confirm password', 'password')}
      {serverError && <p role="alert">{serverError}</p>}
      <button type="submit" disabled={submitting}>
        {submitting ? 'Creating account...' : 'Sign up'}
      </button>
    </form>
  );
}
```

Notes: errors are derived, `touched` decides when to show them, and accessibility attributes link each error to its input. In a real project you would use React Hook Form with a Zod schema (see [forms](05-forms-and-validation.md)) and map server field errors onto inputs.

**Q115. Live coding: build a countdown timer with correct cleanup.**
Requirements: start, pause and reset; display `mm:ss`; call `onDone` once at zero; no leaked intervals; no drift when the tab is in the background.

```tsx
import { useEffect, useRef, useState } from 'react';

function format(ms: number) {
  const total = Math.ceil(ms / 1000);
  const m = String(Math.floor(total / 60)).padStart(2, '0');
  const s = String(total % 60).padStart(2, '0');
  return `${m}:${s}`;
}

export function Countdown({ seconds, onDone }: { seconds: number; onDone?: () => void }) {
  const [remainingMs, setRemainingMs] = useState(seconds * 1000);
  const [deadline, setDeadline] = useState<number | null>(null); // null means paused or finished
  const onDoneRef = useRef(onDone);

  useEffect(() => {
    onDoneRef.current = onDone; // latest callback without restarting the interval
  }, [onDone]);

  useEffect(() => {
    if (deadline === null) return;
    const id = setInterval(() => {
      const left = Math.max(0, deadline - Date.now()); // derived from the clock: no drift, survives throttling
      setRemainingMs(left);
      if (left === 0) {
        clearInterval(id); // stop immediately so onDone cannot fire twice
        setDeadline(null);
        onDoneRef.current?.();
      }
    }, 250);
    return () => clearInterval(id); // runs on pause, reset, finish and unmount
  }, [deadline]);

  function start() {
    if (deadline === null && remainingMs > 0) setDeadline(Date.now() + remainingMs);
  }

  function pause() {
    if (deadline !== null) setRemainingMs(Math.max(0, deadline - Date.now()));
    setDeadline(null);
  }

  function reset() {
    setDeadline(null);
    setRemainingMs(seconds * 1000);
  }

  return (
    <div>
      <p role="timer">{format(remainingMs)}</p>
      {deadline !== null ? (
        <button onClick={pause}>Pause</button>
      ) : (
        <button onClick={start} disabled={remainingMs === 0}>Start</button>
      )}
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

Notes: the interval lives in an effect keyed on `deadline`, so pausing, resetting, finishing and unmounting all clear it; time is computed from a deadline instead of decrementing a counter, because `setInterval` drifts and is throttled in background tabs; `onDone` sits in a ref so a new callback identity does not restart the timer. To restart when the `seconds` prop changes, render it with `key={seconds}`.

**Q116. Live coding: build an accordion as compound components.**
Requirements: `<Accordion>` with `<Accordion.Item>`, `<Accordion.Trigger>` and `<Accordion.Panel>`; single-open by default with a `multiple` option; accessible markup.

```tsx
import { createContext, useContext, useId, useMemo, useState, type ReactNode } from 'react';

type AccordionContextValue = { openItems: Set<string>; toggle: (value: string) => void };
type ItemContextValue = { value: string; isOpen: boolean; buttonId: string; panelId: string };

const AccordionContext = createContext<AccordionContextValue | null>(null);
const ItemContext = createContext<ItemContextValue | null>(null);

function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Accordion parts must be rendered inside <Accordion>');
  return ctx;
}

function useItem() {
  const ctx = useContext(ItemContext);
  if (!ctx) throw new Error('Accordion.Trigger and Accordion.Panel must be inside <Accordion.Item>');
  return ctx;
}

function Accordion({ multiple = false, children }: { multiple?: boolean; children: ReactNode }) {
  const [openItems, setOpenItems] = useState<Set<string>>(() => new Set());

  const value = useMemo<AccordionContextValue>(
    () => ({
      openItems,
      toggle: (item) =>
        setOpenItems((prev) => {
          const next = new Set(multiple ? prev : []); // single mode starts from an empty set
          if (prev.has(item)) next.delete(item);
          else next.add(item);
          return next; // always a new Set, so React sees the change
        }),
    }),
    [openItems, multiple],
  );

  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ value, children }: { value: string; children: ReactNode }) {
  const { openItems } = useAccordion();
  const id = useId();
  const item = useMemo(
    () => ({ value, isOpen: openItems.has(value), buttonId: `${id}-button`, panelId: `${id}-panel` }),
    [value, openItems, id],
  );
  return <ItemContext.Provider value={item}>{children}</ItemContext.Provider>;
}

function AccordionTrigger({ children }: { children: ReactNode }) {
  const { toggle } = useAccordion();
  const { value, isOpen, buttonId, panelId } = useItem();
  return (
    <h3>
      <button id={buttonId} aria-expanded={isOpen} aria-controls={panelId} onClick={() => toggle(value)}>
        {children}
      </button>
    </h3>
  );
}

function AccordionPanel({ children }: { children: ReactNode }) {
  const { isOpen, buttonId, panelId } = useItem();
  return (
    <div id={panelId} role="region" aria-labelledby={buttonId} hidden={!isOpen}>
      {children}
    </div>
  );
}

Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Panel = AccordionPanel;
export { Accordion };

export function FaqDemo() {
  return (
    <Accordion>
      <Accordion.Item value="password">
        <Accordion.Trigger>How do I reset my password?</Accordion.Trigger>
        <Accordion.Panel>Use the "Forgot password" link on the login page.</Accordion.Panel>
      </Accordion.Item>
      <Accordion.Item value="invoices">
        <Accordion.Trigger>Where can I download invoices?</Accordion.Trigger>
        <Accordion.Panel>Under Billing, in your account settings.</Accordion.Panel>
      </Accordion.Item>
    </Accordion>
  );
}
```

Notes: two contexts keep each part simple (accordion-level state and per-item ids); hooks throw a clear error when parts are misused; a new `Set` is created on every toggle because mutating the old one would not trigger a render; `hidden` keeps panel content mounted (state preserved), while conditional rendering would unmount it. The same structure gives you Tabs (see [patterns](07-patterns-and-architecture.md)).
