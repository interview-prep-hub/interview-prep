# React Hooks

> **Tier:** T1 (know cold) · **Related:** [Fundamentals](01-fundamentals.md), [State management](03-state-management.md), [Data fetching](04-data-fetching-tanstack-query.md), [Performance](06-performance.md), [React 19](09-react-19-and-server-components.md), [Closures](../03-languages/javascript/02-scope-closures-hoisting.md)
> **On your resume:** Hooks run through every React and React Native screen you write at Devntech (the Colochousing web app, the Coloc resident app, the TanStack Start platform) and through the PureLogics LMS frontend.

## Quick revision
- Hooks (React 16.8) give function components state, effects and context; custom hooks reuse stateful logic as plain functions.
- Rules: call hooks only at the top level of a component or custom hook (no conditions, loops, nested functions or calls after an early return). React matches hook state by call order. React 19's `use()` is the one hook that may be called conditionally.
- `useState`: lazy initializer `useState(() => expensive())`; functional update `setX((prev) => next)` when the next value depends on the previous; an identical value (`Object.is`) bails out. Update objects and arrays immutably.
- React 18 batches every update (handlers, promises, timeouts) into one render; `flushSync` opts out.
- `useEffect(setup, deps)` synchronizes with an external system. It runs after commit, usually after paint; the cleanup runs before the next run and on unmount.
- Dependencies: omitted = every render, `[]` = mount only, `[a, b]` = when `a` or `b` changes. List every reactive value the effect reads. Objects and functions created during render change identity every time.
- Fetching in an effect needs an `ignore` flag or `AbortController` against race conditions.
- StrictMode (development) runs effects setup, cleanup, setup. Fix the cleanup, do not hide it.
- You might not need an effect: derive values during render, handle user actions in event handlers, reset state with `key`.
- `useLayoutEffect` runs before paint, for measuring layout; it blocks painting.
- `useRef` holds a mutable `{ current }` that survives renders without causing them: DOM nodes, timer ids, latest values.
- `useMemo` caches a value, `useCallback` caches a function: useful for referential equality (memoized children, effect dependencies) and expensive work, and never free.
- `useContext` reads the nearest provider; consumers re-render whenever its value changes.
- `useReducer` moves update logic into a pure reducer; good for related state and state machines; `dispatch` is stable.
- `useId` gives unique, SSR-stable ids for accessibility attributes, never for keys.
- `useTransition` marks an update as non-urgent and gives `isPending`; `useDeferredValue` renders with a lagging copy of a value.
- `useSyncExternalStore` subscribes safely to external stores; `useImperativeHandle` customizes what a ref exposes.
- Custom hooks start with `use` and share logic, not state. React 19 hooks (`use`, `useActionState`, `useOptimistic`, `useFormStatus`) are in [React 19 and Server Components](09-react-19-and-server-components.md).

## 1. Why hooks exist and the rules of hooks

Before hooks, stateful logic lived in class components. Reuse meant higher-order components or render props that nested wrappers several levels deep, and one concern was split across lifecycle methods (subscribe in `componentDidMount`, re-subscribe in `componentDidUpdate`, unsubscribe in `componentWillUnmount`). Hooks keep each concern together and make reuse a function call.

**How React tracks hooks.** For each component instance React keeps an ordered list of hook slots. On every render the first hook call reads slot 1, the second slot 2, and so on. Hooks have positions, not names, so skipping a hook on one render makes every later hook read the wrong slot.

```tsx
function Profile({ userId }: { userId?: string }) {
  if (!userId) return null;                // early return BEFORE a hook...
  const [tab, setTab] = useState('posts'); // ...so this call is skipped on some renders (rules-of-hooks error)
  // Fix: call every hook first, then branch or return early
}
```

1. Call hooks only at the top level of a function component or custom hook: never inside conditions, loops, nested functions, `try/catch`, or after an early return.
2. Call hooks only from React function components or custom hooks, never from regular functions or classes.

`eslint-plugin-react-hooks` enforces both (`rules-of-hooks`) and checks dependency arrays (`exhaustive-deps`). Keep it on and treat its warnings as bugs.

## 2. useState in depth

**Lazy initialization.** The argument is evaluated on every render even though React uses it once.

```tsx
useState<Todo[]>(JSON.parse(localStorage.getItem('todos') ?? '[]'));       // parses on every render
useState<Todo[]>(() => JSON.parse(localStorage.getItem('todos') ?? '[]')); // lazy: runs on the first render only
```

**Functional updates.** `setCount((c) => c + 1)` receives the latest queued value instead of the render's snapshot. Use it whenever the next state depends on the previous one, and always inside intervals, subscriptions and async callbacks that may hold a stale closure.

**Automatic batching.** With `createRoot` (React 18+), updates in the same tick are batched into one render everywhere: event handlers, promises, `setTimeout`, native listeners. Before React 18 only updates inside React event handlers were batched, so each setter after an `await` caused its own render. `flushSync(() => setItems(next))` from `react-dom` forces a synchronous render (rarely needed, for example to scroll to an item right after adding it).

```tsx
async function handleSave() {
  setSaving(true);
  const saved = await api.saveBill(form);
  setSaving(false); // React 18+: these two updates after the await
  setBill(saved);   // produce one re-render
}
```

**Immutable updates.** React detects changes with `Object.is`. `todos.push(x); setTodos(todos)` passes the same reference, so React may bail out and memoized children never see the change.

```tsx
setUser((u) => ({ ...u, address: { ...u.address, city: 'Lahore' } }));       // copy every level you change
setTodos((ts) => [...ts, newTodo]);                                           // add
setTodos((ts) => ts.filter((t) => t.id !== id));                              // remove
setTodos((ts) => ts.map((t) => (t.id === id ? { ...t, done: !t.done } : t))); // update one item
setTodos((ts) => [...ts].sort(byDueDate));                                    // sort a copy, never ts.sort()
```

Avoid `push`, `pop`, `splice`, `sort`, `reverse` and direct assignment on state; use spread, `map`, `filter`, `slice` or `toSorted` (ES2023). For deeply nested state, flatten the shape or use Immer (`produce`, or `useImmer` from `use-immer`). Setting a value identical to the current one skips re-rendering the children.

## 3. useEffect: synchronizing with external systems

An effect keeps your component in sync with something outside React: a network connection, a browser API, a timer, a third-party widget. Think "keep X in sync with these values", not "run on mount".

```tsx
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const connection = createConnection(roomId); // the external system
    connection.connect();
    return () => connection.disconnect(); // runs before re-running for a new roomId, and on unmount
  }, [roomId]);

  return <h1>Room {roomId}</h1>;
}
```

**Timing.** Effects run after React commits the DOM changes, usually after the browser has painted, so they never block the first frame. To measure or adjust the DOM before the user sees it, use `useLayoutEffect`.

| Dependencies | When the effect runs | Typical use |
|---|---|---|
| omitted | after every render | rarely correct |
| `[]` | after the first mount; cleanup on unmount | global listeners, one-time setup |
| `[a, b]` | after mount and whenever `a` or `b` changes (`Object.is`) | sync with props or state |

The array must contain every reactive value the effect reads: props, state, values derived from them, functions declared in the component. If a dependency makes the effect run too often, change the code (move the function inside the effect, use a functional update, depend on a primitive) rather than silencing the linter.

**Cleanup order.** When dependencies change, React runs the previous cleanup with the old values, then the new setup with the new values; on unmount it runs the last cleanup. Cleanups must undo exactly what setup did: `removeEventListener` with the same function reference, `clearInterval` with the same id, `abort()` on the same controller.

## 4. Common useEffect bugs

**Missing dependency (stale closure).**

```tsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000); // count is frozen at its first-render value
  return () => clearInterval(id);
}, []); // the counter goes 0 -> 1 and stays there
// Fix: setCount((c) => c + 1), which needs no dependency on count
```

**Objects, arrays and functions as dependencies.**

```tsx
const filters = { houseId, status };               // a new object every render
useEffect(() => { fetchBills(filters).then(setBills); }, [filters]); // so this runs after every render
// Fix: depend on [houseId, status] and build the object inside the effect
```

**Infinite loops.** An effect that sets state it also depends on (`setItems(items.filter(...))` with `[items]`) or an effect without a dependency array that sets state renders forever. Usually the value should be derived during render: `const activeItems = items.filter((i) => i.active);`.

**Race conditions when fetching.**

```tsx
useEffect(() => {
  let ignore = false;
  fetch(`/api/users/${userId}`)
    .then((res) => res.json())
    .then((data: User) => {
      if (!ignore) setUser(data); // drop responses for a userId we no longer show
    });
  return () => {
    ignore = true;
  };
}, [userId]);
```

Without the flag, switching from user 1 to user 2 can show user 1 if that response arrives last. `AbortController` also cancels the request itself: see [data fetching](04-data-fetching-tanstack-query.md).

**StrictMode double invocation.** In development StrictMode runs setup, cleanup, setup on mount. Two requests in development are expected; a visible bug means the cleanup is missing. "Run only once" ref hacks hide exactly the bug the check exposes.

## 5. You might not need an effect

Effects are an escape hatch for external systems. Without one involved, there is usually a simpler tool:

| Situation | Instead of an effect |
|---|---|
| Value computed from props or state | compute it during render (`useMemo` if expensive) |
| Reset all state when a prop changes | a `key` on the component |
| Respond to a user action (submit, buy, click) | do it in the event handler |
| Tell the parent about a change | call its callback in the same handler |
| Fetch data | TanStack Query or router loaders |
| Run once when the app starts | module-level code or the entry point |

```tsx
// Bad: an effect for derived state (an extra render and a chance to drift out of sync)
const [fullName, setFullName] = useState('');
useEffect(() => setFullName(`${first} ${last}`), [first, last]);
```

```tsx
// Good: compute it during render
const fullName = `${first} ${last}`;
```

The same applies to event logic: an effect that POSTs when a `submitted` flag flips also fires after remounts; the request belongs in `handleSubmit`.

## 6. useLayoutEffect

`useLayoutEffect` runs synchronously after React updates the DOM and before the browser paints. Use it to measure layout and correct it before the first frame is visible: tooltips, popovers, auto-sizing.

```tsx
function Tooltip({ anchor, children }: { anchor: DOMRect; children: ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [top, setTop] = useState(0);

  useLayoutEffect(() => {
    const { height } = ref.current!.getBoundingClientRect();
    setTop(anchor.top - height - 8); // re-renders before paint, so there is no visible jump
  }, [anchor]);

  return (
    <div ref={ref} style={{ position: 'fixed', top, left: anchor.left }}>
      {children}
    </div>
  );
}
```

It blocks painting, so keep the work small and default to `useEffect`. It does not run during server rendering.

## 7. useRef

`useRef(initialValue)` returns the same `{ current }` object for the component's lifetime. Changing `ref.current` does not trigger a render. For DOM access, pass it as `ref={inputRef}` and React sets `current` to the node after mount (as in the tooltip above); for mutable values, read and write `current` in handlers and effects:

```tsx
function Ticker() {
  const [ticks, setTicks] = useState(0);
  const idRef = useRef<ReturnType<typeof setInterval> | null>(null); // persists, never rendered
  const start = () => {
    if (idRef.current === null) idRef.current = setInterval(() => setTicks((t) => t + 1), 1000);
  };
  const stop = () => {
    if (idRef.current !== null) clearInterval(idRef.current);
    idRef.current = null;
  };
  useEffect(() => () => {
    if (idRef.current !== null) clearInterval(idRef.current); // also clear on unmount
  }, []);
  return <p>{ticks} <button onClick={start}>Start</button> <button onClick={stop}>Stop</button></p>;
}
```

| | `useState` | `useRef` |
|---|---|---|
| Changing it re-renders | yes | no |
| Value during render | a snapshot | mutable, always the latest |
| Use for | anything the UI shows | DOM nodes, timer ids, latest callbacks, flags |

**Previous value.** The classic `usePrevious` stores the value in a ref inside an effect and returns `ref.current` during render, which therefore still holds the previous value. It reads a ref during render, which the React docs discourage (and React Compiler lint rules flag). The docs-friendly version keeps it in state: `const [prev, setPrev] = useState(status); if (status !== prev) { setPrev(status); /* react to the change */ }`.

Rules: do not read or write `ref.current` during render (except lazy initialization); if the UI depends on a value, it is state. Callback refs (`ref={(node) => {...}}`) run when the node attaches, and in React 19 they may return a cleanup function. With the React 19 types `useRef` requires an argument: write `useRef<T>(null)`.

## 8. useMemo and useCallback

```tsx
function TodoScreen({ initialTodos }: { initialTodos: Todo[] }) {
  const [todos, setTodos] = useState(initialTodos);
  const [filter, setFilter] = useState<Filter>('all');
  const [draft, setDraft] = useState(''); // every keystroke re-renders TodoScreen

  // Recomputed only when todos or filter change, not on every keystroke
  const visible = useMemo(() => todos.filter((t) => matchesFilter(t, filter)), [todos, filter]);

  // Same function identity on every render, so the memoized rows can skip re-rendering
  const toggle = useCallback((id: string) => {
    setTodos((ts) => ts.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));
  }, []);

  return (
    <>
      <input value={draft} onChange={(e) => setDraft(e.target.value)} />
      <FilterTabs value={filter} onChange={setFilter} />
      <ul>{visible.map((t) => <TodoRow key={t.id} todo={t} onToggle={toggle} />)}</ul>
    </>
  );
}

const TodoRow = memo(function TodoRow({ todo, onToggle }: { todo: Todo; onToggle: (id: string) => void }) {
  return <li onClick={() => onToggle(todo.id)}>{todo.title}</li>;
});
```

`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`. They exist because `{} !== {}` and `() => {} !== () => {}`: values created during render get a new identity each time.

They help when (1) the value goes to a `memo` component or a context provider, (2) it is a dependency of another hook, or (3) the calculation is measurably expensive (time it with `console.time`; about 1 ms or more per render is worth caching). They do not help for cheap work, for props to non-memoized children, or when a dependency changes every render anyway. They cost memory, a comparison on every render, readability, and stale-closure bugs when the dependency list is wrong. The React Compiler can insert this memoization automatically (see [performance](06-performance.md)).

## 9. useContext

```tsx
type Theme = 'light' | 'dark';
const ThemeContext = createContext<Theme>('light'); // default applies only when no provider is above

function ThemedButton() {
  const theme = useContext(ThemeContext); // nearest provider above this component
  return <button className={`btn-${theme}`}>OK</button>;
}

// <ThemeContext.Provider value="dark"><ThemedButton /></ThemeContext.Provider>
// React 19 also accepts <ThemeContext value="dark">
```

Every consumer re-renders when the provider's value changes (by `Object.is`), even inside `memo` components, so a provider that passes a new object literal on each render re-renders all consumers every time. Production patterns (a custom hook with a null check, a memoized value, split contexts) are in [state management](03-state-management.md).

## 10. useReducer

Prefer `useReducer` when several values change together, when transitions are complex enough to be a small state machine, or when you want the update logic in one pure function you can unit test. `dispatch` is stable, so it is cheap to pass down.

```tsx
type State = { status: 'idle' | 'loading' | 'success' | 'error'; data: User[]; error: string | null };
type Action =
  | { type: 'fetch' }
  | { type: 'resolved'; data: User[] }
  | { type: 'rejected'; error: string };

function usersReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'fetch':
      return { ...state, status: 'loading', error: null };
    case 'resolved':
      return { status: 'success', data: action.data, error: null };
    case 'rejected':
      return { ...state, status: 'error', error: action.error };
  }
}

// In the component:
const [state, dispatch] = useReducer(usersReducer, { status: 'idle', data: [], error: null });
// dispatch({ type: 'fetch' }); later dispatch({ type: 'resolved', data }) or ({ type: 'rejected', error })
```

| | `useState` | `useReducer` |
|---|---|---|
| Best for | independent, simple values | related values, state machines |
| Update logic lives in | each event handler | one pure, testable reducer |
| Passing updates down | several setters | one stable `dispatch` |

## 11. useId

```tsx
function EmailField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>We never share your email.</p>
    </>
  );
}
```

The id is unique per component instance and identical on server and client, so it cannot cause a hydration mismatch the way a module counter or `Math.random()` would. Call it once and add suffixes for related elements. Never use it for list keys.

## 12. useTransition and useDeferredValue

Both lower the priority of a render so urgent updates (typing, clicking) stay responsive; React can interrupt the low-priority render when new input arrives.

```tsx
function TabSwitcher() {
  const [tab, setTab] = useState<'about' | 'posts'>('about');
  const [isPending, startTransition] = useTransition();
  const select = (next: 'about' | 'posts') => startTransition(() => setTab(next)); // non-urgent

  return (
    <>
      <TabButtons onSelect={select} />
      {isPending && <Spinner />}
      {tab === 'posts' ? <SlowPostsList /> : <About />}
    </>
  );
}

function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // lags behind while the user types
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <div style={{ opacity: query !== deferredQuery ? 0.5 : 1 }}>
        <SlowList items={items} query={deferredQuery} /> {/* wrapped in memo so React can skip it */}
      </div>
    </>
  );
}
```

| | `useTransition` | `useDeferredValue` |
|---|---|---|
| Wraps | a state update you own | a value (often a prop you do not control) |
| Gives you | `isPending` and `startTransition` | the lagging value; compare it to detect staleness |
| Typical use | tab switches, navigation, filters | expensive child rendering driven by input |

Neither replaces debouncing network requests: they schedule rendering, not fetching. Never put a controlled input's own value update inside a transition, or typing feels stuck. In React 19 `startTransition` also accepts async functions (Actions) and `useDeferredValue` takes an optional initial value.

## 13. useSyncExternalStore and useImperativeHandle

`useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` reads from a store outside React without tearing (components seeing different values during one concurrent render). Redux and Zustand use it internally.

```tsx
function subscribe(onChange: () => void) {
  window.addEventListener('online', onChange);
  window.addEventListener('offline', onChange);
  return () => {
    window.removeEventListener('online', onChange);
    window.removeEventListener('offline', onChange);
  };
}

export function useOnlineStatus() {
  return useSyncExternalStore(subscribe, () => navigator.onLine, () => true); // 3rd: server snapshot
}
```

`getSnapshot` must return the same value while nothing changed (a cached object, not a fresh one each call), or React re-renders forever.

`useImperativeHandle` customizes what a parent receives through a ref, exposing a few methods instead of the whole DOM node:

```tsx
export type VideoHandle = { play: () => void; pause: () => void };

export function VideoPlayer({ src, ref }: { src: string; ref?: Ref<VideoHandle> }) { // React 19: ref is a prop
  const videoRef = useRef<HTMLVideoElement>(null);
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
  }), []);
  return <video ref={videoRef} src={src} />;
}
```

Use it sparingly, for focus, scroll or media control. In React 18 the component must be wrapped in `forwardRef` to receive the ref.

## 14. Custom hooks

A custom hook is a function whose name starts with `use` and that calls other hooks. It shares logic, not state: two components calling `useLocalStorage('cart', [])` each get their own state. Return a tuple for one value and its setter, an object for more.

```tsx
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // a newer value cancels the pending timer
  }, [value, delay]);
  return debounced;
}

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = window.localStorage.getItem(key);
      return stored !== null ? (JSON.parse(stored) as T) : initialValue;
    } catch {
      return initialValue; // corrupted JSON, storage blocked, or no window (server)
    }
  });
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // quota exceeded or private mode: keep working in memory
    }
  }, [key, value]);
  return [value, setValue] as const;
}
```

In an SSR app, reading `localStorage` during the first render makes the client HTML differ from the server's; read it after mount (or via `useSyncExternalStore`) to avoid a hydration mismatch.

```tsx
export function useOnClickOutside<T extends HTMLElement>(
  ref: RefObject<T | null>,
  handler: (event: MouseEvent | TouchEvent) => void,
) {
  const handlerRef = useRef(handler);
  useEffect(() => {
    handlerRef.current = handler; // call the latest handler without re-subscribing
  });

  useEffect(() => {
    function listener(event: MouseEvent | TouchEvent) {
      const el = ref.current;
      if (!el || el.contains(event.target as Node)) return; // click inside: ignore
      handlerRef.current(event);
    }
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref]);
}

type FetchState<T> = { status: 'loading' } | { status: 'success'; data: T } | { status: 'error'; error: Error };

export function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({ status: 'loading' });
  useEffect(() => {
    const controller = new AbortController();
    setState({ status: 'loading' });
    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then((data) => setState({ status: 'success', data }))
      .catch((error: unknown) => {
        if (controller.signal.aborted) return; // url changed or unmounted
        setState({ status: 'error', error: error instanceof Error ? error : new Error(String(error)) });
      });
    return () => controller.abort();
  }, [url]);
  return state;
}
```

`useFetch` shows you understand effects; in production a cache such as TanStack Query adds deduplication, retries and invalidation ([data fetching](04-data-fetching-tanstack-query.md)).

## Common pitfalls
- Calling a hook conditionally or after an early return.
- Missing dependencies, producing stale closures (intervals that never advance, handlers reading old props).
- Objects, arrays or inline functions as effect dependencies, re-running the effect after every render.
- Setting state in an effect from a value the same effect depends on: an infinite loop.
- Fetching in an effect without an `ignore` flag or abort, so a slow old response overwrites the new one.
- Storing derived data in state and syncing it with an effect.
- Wrapping everything in `useMemo`/`useCallback` without a memoized consumer or a measured cost.
- Reading or writing `ref.current` during render, or keeping UI data in a ref so the screen never updates.
- Expecting two components that call the same custom hook to share state.

## How to talk about this in an interview
- Explain effects as synchronization: "an effect keeps the component in sync with an external system; the dependency array lists what it reads and the cleanup undoes the setup." That framing answers most follow-ups.
- Show judgment: derived values and event handlers first, effects only for subscriptions, timers and imperative APIs, and a data library for fetching.
- Use settings from your work: subscribing to push notification events in an Expo app (the Coloc resident app has push notifications) is a textbook effect-with-cleanup case, and React Native uses exactly the same hooks.

> **Fill in:** a custom hook you wrote or used at Devntech (for example around bills, events or tasks in the resident app) and what logic it hid from the screens.

> **Fill in:** an effect bug you debugged (stale closure, double subscription, race), how you found it, and the fix.

- On `useMemo`/`useCallback`, say you add them for referential equality where a memoized child or an effect depends on the value, after profiling, and that the React Compiler is making manual memoization less necessary.

## Interview questions

**Q1. What are the rules of hooks, and why do they exist?**
Call hooks only at the top level of function components or custom hooks (never in conditions, loops, nested functions or after an early return) and never from plain functions or classes. React stores hook state in an ordered list per component and matches each call by position, so the call order must be identical on every render. The ESLint plugin enforces this.
*Follow-up:* Any exception? React 19's `use()` may be called conditionally.

**Q2. Explain the useEffect dependency array.**
No array: the effect runs after every render. `[]`: once after mount, cleanup on unmount. `[a, b]`: after mount and whenever `a` or `b` changed by `Object.is`. The array must include every reactive value the effect reads; it describes what the effect depends on, not when you would like it to run.
*Follow-up:* What if a function is a dependency? Move it inside the effect, or wrap it in `useCallback` if it is shared.

**Q3. When does the cleanup function run? Give examples.**
Before the effect re-runs (with the previous render's values) and when the component unmounts. Typical cleanups remove listeners, clear intervals, unsubscribe from sockets or stores, abort fetches and disconnect observers. In development StrictMode adds one extra setup-cleanup cycle on mount.
*Follow-up:* Does an effect without a dependency array get cleaned up? Yes, before every re-run.

**Q4. What is a stale closure in hooks, and how do you fix it?**
A callback created in an earlier render keeps reading that render's props and state. Classic case: `setInterval(() => setCount(count + 1))` in an effect with `[]`, so `count` stays 0. Fix it with a functional update, by adding the value to the dependencies, or by keeping the latest value in a ref when re-subscribing is expensive.
*Follow-up:* Why do closures capture old values? Each render is a separate function call with its own constants.

**Q5. Why might an effect run in an infinite loop?**
It sets state that is also its dependency (directly or via a new object or array each time), or it has no dependency array and sets state. Each set causes a render, which re-runs the effect. Usually the value should be derived during render, or the dependency should be a primitive.
*Follow-up:* How does it show up? A frozen tab or "Maximum update depth exceeded", and endless commits in the Profiler.

**Q6. Why does my effect run twice in development?**
StrictMode mounts, unmounts and remounts each component once in development, so effects run setup, cleanup, setup. It proves your cleanup works: if the second run causes a visible bug, the cleanup is missing. Production runs it once.
*Follow-up:* My API is called twice; is that bad? In development it is expected; abort or ignore the first request, or use a library that deduplicates.

**Q7. How do you avoid race conditions when fetching in useEffect?**
Return a cleanup that marks the request as stale: an `ignore` flag checked before `setState`, or an `AbortController` whose `signal` goes to `fetch` and whose `abort()` runs in the cleanup. A response for an old `userId` then never overwrites the current one. Also check `res.ok`, because `fetch` only rejects on network failure.
*Follow-up:* Why not use a library? Usually you should: TanStack Query passes an abort signal and keys responses by query key.

**Q8. useEffect vs useLayoutEffect?**
Both run after React updates the DOM. `useEffect` usually runs after paint and does not block it, so it is the default. `useLayoutEffect` runs synchronously before paint, so you can measure layout and re-render without flicker, at the cost of blocking the frame; it does not run on the server.
*Follow-up:* Example? Positioning a tooltip from the anchor's `getBoundingClientRect()`.

**Q9. useRef vs useState: when do you use each?**
State for anything that appears on screen, since updating it re-renders. A ref for values that must persist across renders without re-rendering: DOM nodes, interval ids, the latest callback, a flag that blocks double submission. Refs are mutable and should not be read or written during render.
*Follow-up:* Why does changing a ref not update the screen? React does not track refs, so nothing schedules a render.

**Q10. What is the difference between useMemo, useCallback and React.memo?**
`useMemo` caches a computed value, `useCallback` caches a function identity (it is `useMemo` returning the function), and `React.memo` makes a component skip re-rendering when its props are shallow-equal. They work together: `memo` on a child is useless if the parent passes new objects or functions on every render.
*Follow-up:* Memoize everything? No: it costs memory, comparisons and readability, so measure first.

**Q11. When would you choose useReducer over useState?**
When several pieces of state change together (status, data and error of a request), when transitions are complex enough to be a state machine, or when you want update logic in one pure, testable function. `dispatch` is stable, so passing it down does not break memoization.
*Follow-up:* Is useReducer Redux? Same reducer idea, but local to one component, with no store, middleware or devtools.

**Q12. What problem do functional state updates solve?**
`setX(x + 1)` uses the current render's snapshot, so several calls in one handler, or a call from a stale callback, lose updates. `setX((prev) => prev + 1)` receives the latest queued value, so each update builds on the previous one.
*Follow-up:* Calling `setCount((c) => c + 1)` three times from 0 gives? 3.

**Q13. What is automatic batching in React 18?**
React groups state updates from the same tick into one render, now everywhere: React handlers, promises, `setTimeout`, native listeners. Before React 18 only updates inside React handlers were batched, so code after an `await` rendered once per setter. It requires `createRoot`; `flushSync` opts out.
*Follow-up:* When would you opt out? When you must read the updated DOM immediately, such as scrolling to a just-added item.

**Q14. What is a custom hook? Write useDebounce.**
A function starting with `use` that calls other hooks to package reusable stateful logic; each call has its own state. `useDebounce(value, delay)` keeps a debounced copy in state, sets a timeout in an effect to update it, and clears the timeout in the cleanup so each new value restarts the timer (code in section 14).
*Follow-up:* Why the `use` prefix? So the linter and React tooling apply the rules of hooks to it.

**Q15. useTransition vs useDeferredValue? Do they replace debouncing?**
`useTransition` wraps a state update you control and gives `isPending`; `useDeferredValue` gives a lagging copy of a value you may not control. Both mark rendering as low-priority and interruptible so input stays responsive. They do not reduce network calls, so debounce requests separately.
*Follow-up:* Why wrap the slow child in `memo`? So React can skip it during the urgent render that still uses the old deferred value.

**Q16. What is useId for, and why not Math.random()?**
It generates an id per component instance that is identical on server and client, for linking labels and `aria-*` attributes. `Math.random()` or a global counter differ between server and client (hydration mismatch) and change between renders. It is not for list keys.
*Follow-up:* Several ids in one component? Call `useId` once and add suffixes.

**Q17. Why can't you pass an async function directly to useEffect?**
An async function returns a promise, but React expects the effect to return nothing or a cleanup function. Define the async function inside the effect, call it, and use a flag or `AbortController` in the cleanup to ignore late results.
*Follow-up:* Can a cleanup be async? It can start async work, but React does not wait for it.

**Q18. When do you not need an effect?**
When a value can be derived from props or state (compute it during render or with `useMemo`), when code responds to a specific user action (event handler), when state should reset on a prop change (`key`), and usually for data fetching (a library or loader). Effects are for synchronizing with systems outside React.
*Follow-up:* What does an unnecessary effect cost? An extra render, possible flicker, and state that can drift.

## Further reading
- [Built-in hooks reference](https://react.dev/reference/react/hooks)
- [You might not need an effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Reusing logic with custom hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
