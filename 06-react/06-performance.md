# React Performance

> **Tier:** T1 (know cold) · **Related:** [Fundamentals](01-fundamentals.md), [Hooks](02-hooks.md), [State management](03-state-management.md), [Patterns](07-patterns-and-architecture.md), [Web caching and performance](../04-web-fundamentals/08-caching-and-performance.md), [RN performance](../08-react-native-expo/07-performance-and-animations.md)
> **On your resume:** Performance matters as data grows: the Colochousing platform has 16K+ users and 24K+ rental applications, the Coloc resident app renders lists of events, tasks and bills, and React on Rails bundles ship inside a Rails app where every kilobyte loads with the page.

## Quick revision
- Measure first with the React DevTools Profiler (and confirm with a production build), then fix the slowest thing.
- Render = calling components and diffing; commit = applying DOM changes. Most re-renders are cheap; expensive subtrees re-rendering often are the problem.
- A component re-renders when its state changes, its parent re-renders, or a context it reads changes. "Its props changed" is not a trigger by itself; props only decide for memoized components.
- `React.memo(Component)` skips a re-render when every prop is `Object.is`-equal to last time (shallow comparison).
- `memo` is defeated by new objects, arrays, functions or JSX passed as props on every render; stabilize them or restructure.
- Fix structure before memoizing: move state down into the component that uses it; lift content up by passing expensive subtrees as `children`.
- Context consumers re-render on every value change: memoize and split providers.
- Never define components inside components or use random keys: both remount subtrees on every render.
- Code split routes and heavy widgets with `React.lazy` + `Suspense`; load heavy libraries with dynamic `import()`.
- Virtualize long lists (TanStack Virtual, react-window); React Native lists (`FlatList`, FlashList) virtualize already.
- Keep input responsive with `useDeferredValue`/`useTransition`; debounce network requests.
- Tree shaking needs ES module imports and side-effect-free packages; inspect bundles with a visualizer.
- The React Compiler (stable since late 2025) auto-memoizes components and hooks that follow the Rules of React.
- Real-user metrics are the Core Web Vitals: LCP, INP, CLS.

## 1. Render vs commit, and what is actually slow

In the render phase React calls your components and diffs the new tree against the old one; in the commit phase it applies the changed DOM nodes. A re-render that produces identical output commits nothing (see [fundamentals](01-fundamentals.md)).

So re-renders are usually cheap: a function runs and returns objects. They become a problem when:
- an expensive component (big list, chart, rich editor) re-renders on every keystroke, scroll event or timer tick;
- thousands of DOM nodes are created or updated (long lists, large tables);
- rendering does heavy work (sorting or filtering thousands of items, formatting in a loop);
- the JavaScript bundle is so large that the page is slow before React even starts.

Find what is slow, then choose the fix. Unmeasured memoization adds complexity without guaranteed gains.

## 2. What actually triggers a re-render

```tsx
function Dashboard() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Clicked {count}</button>
      <ExpensiveChart /> {/* no props, yet it re-renders on every click */}
    </>
  );
}
```

1. **Its own state changes:** `useState`, `useReducer`, or a store subscription (Redux `useSelector`, a Zustand selector) returning a new value.
2. **Its parent re-renders.** By default React re-renders every child of a re-rendering component, whether or not their props changed.
3. **A context it reads changes value.**

The myth is "a component re-renders when its props change". A component re-renders because its parent re-rendered; new props are a side effect of that. Props become the deciding factor only for components wrapped in `memo`, where React compares them and may skip the render. Related facts:
- Setting state to an identical value (`Object.is`) bails out.
- Changing a `key` remounts (unmount plus mount), which costs more than a re-render.
- A component defined inside another component is a new type every render, so it remounts every time.

## 3. Measuring with the Profiler

**React DevTools Profiler:**
1. Open the Profiler tab, click record, perform the slow interaction, stop.
2. The commit bar chart shows one bar per commit; tall bars are slow commits.
3. The flame graph shows the tree for one commit: width is render time, grey means it did not render. The ranked chart sorts components by render time.
4. Enable "Record why each component rendered while profiling" to see "props changed", "state changed", "context changed" or "parent rendered".
5. In the Components tab settings, "Highlight updates when components render" flashes re-rendering components live.

**Measure realistically.** Development builds are much slower (extra checks, StrictMode double renders). Confirm findings in a production build; React ships a profiling build of `react-dom` for this, and enabling it depends on your bundler or framework. Use the browser's Performance panel, with CPU throttling, for long tasks and layout work. React 19.2 adds React-specific tracks to Chrome's performance panel (check the release notes).

```tsx
import { Profiler, type ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (id, phase, actualDuration, baseDuration) => {
  // actualDuration: time spent in this commit; baseDuration: estimated cost with no memoization
  console.log(`${id} ${phase}: ${actualDuration.toFixed(1)} ms (base ${baseDuration.toFixed(1)} ms)`);
};

<Profiler id="BillList" onRender={onRender}>
  <BillList bills={bills} />
</Profiler>;
```

For a quick check of one calculation, wrap it in `console.time('filter')` and `console.timeEnd('filter')`.

## 4. React.memo and shallow comparison

```tsx
import { memo, useCallback, useState } from 'react';

type RowProps = { bill: Bill; onToggle: (id: string) => void };

const BillRow = memo(function BillRow({ bill, onToggle }: RowProps) {
  return (
    <li>
      <label>
        <input type="checkbox" checked={bill.paid} onChange={() => onToggle(bill.id)} />
        {bill.title}
      </label>
    </li>
  );
});

function BillList({ initialBills }: { initialBills: Bill[] }) {
  const [bills, setBills] = useState(initialBills);
  const [note, setNote] = useState(''); // typing here must not re-render 500 rows

  const toggle = useCallback((id: string) => {
    setBills((bs) => bs.map((b) => (b.id === id ? { ...b, paid: !b.paid } : b))); // only one row object changes
  }, []);

  return (
    <>
      <input value={note} onChange={(e) => setNote(e.target.value)} />
      <ul>{bills.map((b) => <BillRow key={b.id} bill={b} onToggle={toggle} />)}</ul>
    </>
  );
}
```

The comparison runs `Object.is(prev[key], next[key])` for each prop; if all are equal, React reuses the previous result. Immutable updates make this effective: toggling one bill creates one new object, so only that row re-renders.

What silently defeats `memo`:
- inline objects and arrays: `style={{ padding: 8 }}`, `columns={['a', 'b']}`;
- inline functions: `onToggle={(id) => toggle(id)}`;
- JSX passed as props or `children` (a new element object every render);
- derived arrays recreated by the parent each render (`bills.filter(...)` without `useMemo`).

A custom comparator, `memo(Component, (prev, next) => prev.bill.id === next.bill.id && prev.bill.paid === next.bill.paid)`, returns `true` to skip rendering. Use it rarely: forgetting a prop causes stale UI. Reserve `memo` for components that render often with the same props and are expensive (list rows, charts, large forms), not as a default wrapper.

## 5. When useMemo and useCallback actually help

Only when something consumes the stable identity or the work is expensive:
- a function or object passed to a `memo` child, or used as a context value;
- a value in the dependency array of `useEffect` or another `useMemo`;
- an expensive calculation (sorting thousands of rows, building chart data) that should not repeat on unrelated renders.

```tsx
const sortedApplications = useMemo(
  () => [...applications].sort((a, b) => b.submittedAt.localeCompare(a.submittedAt)), // copy, then sort
  [applications],
);
```

They do not help with cheap work, with props for non-memoized children, or when their dependencies change every render anyway. More in [hooks](02-hooks.md).

## 6. Moving state down and lifting content up

Two structural fixes that need no memoization.

**Move state down:** keep state in the smallest component that uses it.

```tsx
// Before: every keystroke re-renders SlowChart, because the state lives in Dashboard
function Dashboard() {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <SlowChart />
    </>
  );
}

// After: the state lives in SearchBox, so typing re-renders only SearchBox
function DashboardFixed() {
  return (
    <>
      <SearchBox />
      <SlowChart />
    </>
  );
}
```

**Lift content up:** when a wrapper must own fast-changing state, let the parent create the expensive subtree and pass it as `children`.

```tsx
function ScrollProgress({ children }: { children: ReactNode }) {
  const [progress, setProgress] = useState(0);
  useEffect(() => {
    const onScroll = () => {
      const max = document.documentElement.scrollHeight - window.innerHeight;
      setProgress(max > 0 ? window.scrollY / max : 0);
    };
    window.addEventListener('scroll', onScroll, { passive: true });
    return () => window.removeEventListener('scroll', onScroll);
  }, []);

  return (
    <>
      <progress value={progress} />
      {children} {/* created by the parent: the same element as last render, so React skips it */}
    </>
  );
}

// <ScrollProgress><SlowArticle /></ScrollProgress>
```

`<SlowArticle />` is created when the parent of `ScrollProgress` renders. When only `ScrollProgress`'s own state changes, `children` is the same element object as before, so React does not re-render that subtree.

### Going deeper: how React decides to skip a child

During a re-render React compares each child element with the one from the previous render. If it is the very same element object (`===`), its props cannot have changed, and if the component also has no pending state or context update, React reuses the previous result without calling it. `children` passed from above and JSX cached with `useMemo` qualify; a freshly written `<Child />` does not, because every render creates a new element object. `memo` covers that case by comparing props shallowly instead of by object identity.

## 7. Code splitting with React.lazy and Suspense

```tsx
import { lazy, Suspense } from 'react';

const ReportsPage = lazy(() => import('./pages/ReportsPage')); // needs a default export
const ChartPanel = lazy(() => import('./ChartPanel').then((m) => ({ default: m.ChartPanel }))); // named export

export function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <ReportsPage />
    </Suspense>
  );
}
```

- Split by route first (the biggest win), then heavy widgets that appear only after an interaction: charts, maps, rich text editors, PDF viewers.
- Declare `lazy` components at module level; calling `lazy` inside a component creates a new type each render, so the subtree remounts and loses state.
- Preload likely next chunks: keep the import function (`const loadReports = () => import('./pages/ReportsPage')`), pass it to `lazy`, and call it on hover or focus of the link.
- Load heavy libraries inside the handler that needs them: `const { jsPDF } = await import('jspdf');` when the user clicks "Download PDF".
- A chunk can fail to load after a deploy removed old files; wrap lazy areas in an error boundary that offers a reload.
- Routers have built-in support: React Router's `lazy` route property ([routing](08-routing.md)), `next/dynamic` in Next.js, automatic route splitting in TanStack Router.

## 8. List virtualization

Rendering 10,000 rows creates 10,000 DOM nodes, which is slow to mount, update and scroll. Virtualization renders only the rows in the viewport (plus a small overscan) and uses a spacer to keep the scrollbar correct.

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function ApplicationList({ rows }: { rows: Application[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // estimated row height in px
    overscan: 8,            // extra rows above and below the viewport
  });

  return (
    <div ref={parentRef} style={{ height: 480, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div
            key={item.key}
            style={{ position: 'absolute', top: 0, left: 0, width: '100%', height: item.size, transform: `translateY(${item.start}px)` }}
          >
            {rows[item.index].applicantName}
          </div>
        ))}
      </div>
    </div>
  );
}
```

- react-window is the other common choice; its recent major version changed the API, so check its docs before copying older examples.
- Trade-offs: browser find-in-page only sees rendered rows, variable heights need measuring, and screen readers need care. Pagination is often the simpler answer when users do not need to scroll through everything.
- React Native lists (`FlatList`, `SectionList`, Shopify's FlashList) virtualize by default; see [RN performance](../08-react-native-expo/07-performance-and-animations.md).

## 9. Responsiveness: transitions and deferred values

Filtering a long list on every keystroke blocks the main thread, so typing lags. Two React 18 tools help:
- `useDeferredValue(query)` keeps the input urgent while the expensive list re-renders with the slightly older value in the background (wrap the list in `memo`).
- `startTransition(() => setFilter(next))` marks an update as interruptible; `isPending` lets you dim the old content.

Debouncing is different: it delays work (often a request) until typing pauses. Throttling caps how often a handler runs (scroll, resize). Use debounce for requests, deferred values for expensive local rendering, throttle for high-frequency events. Code in [hooks](02-hooks.md).

## 10. Bundle size and tree shaking

- **Tree shaking** removes unused exports at build time (Rollup in Vite production builds, webpack, esbuild). It needs static ES module `import`/`export` and packages marked side-effect free (`"sideEffects": false` in their `package.json`). CommonJS packages usually cannot be shaken.
- **Import narrowly:** `import debounce from 'lodash/debounce'` or `lodash-es` instead of all of CommonJS `lodash`; per-icon imports from icon libraries; `Intl` or a small date library instead of a large one.
- **Analyze:** `rollup-plugin-visualizer` (or `npx vite-bundle-visualizer`) for Vite, `@next/bundle-analyzer` for Next.js, `source-map-explorer` for any build with source maps. Look for duplicated and surprisingly large dependencies.
- **Barrel files** (an `index.ts` that re-exports everything) can pull unrelated modules into a chunk and slow dev servers; import from the specific file in hot paths.
- **Assets:** compress and size images, lazy-load below-the-fold images (`loading="lazy"`), subset fonts. These often matter more for LCP than JavaScript tweaks.
- In a Rails app with React on Rails, check that each page's bundle contains only the components that page mounts.

## 11. React Compiler

The React Compiler is a build-time tool (a Babel plugin with integrations for Vite, Next.js and Expo) that analyzes components and hooks and inserts memoization automatically, at a finer grain than hand-written `useMemo`, `useCallback` and `memo`. It reached a stable 1.0 release in late 2025; check the docs for setup and supported React versions.

- It relies on the Rules of React: pure rendering, no mutation of props or state, the rules of hooks. Code it cannot prove safe is skipped rather than miscompiled, and lint rules (in recent versions of `eslint-plugin-react-hooks`) flag violations.
- A `"use no memo"` directive opts a component out while you investigate a problem.
- In new code it makes most manual memoization unnecessary; existing `useMemo`/`useCallback` keep working.
- It does not fix structural problems: rendering 10,000 rows, request waterfalls, huge bundles or slow effects still need the techniques above.

## 12. Other quick wins and Core Web Vitals

- Stable keys from data; never `key={Math.random()}`, which remounts every row on every render.
- Keep context values small and memoized; move fast-changing state out of app-wide providers.
- Throttle scroll, resize and pointer handlers, or use CSS and `IntersectionObserver` instead.
- Move CPU-heavy work (parsing big CSV files, image processing) to a Web Worker.
- Avoid layout thrashing in effects (read layout, write styles, read again, in a loop).
- Core Web Vitals measure what users feel: LCP (loading), INP (responsiveness, hurt by long tasks in handlers and renders) and CLS (layout shifts, often images without dimensions). See [caching and performance](../04-web-fundamentals/08-caching-and-performance.md).

## Common pitfalls
- Optimizing without profiling, or trusting absolute numbers from a development build.
- Wrapping everything in `memo`, `useMemo` and `useCallback` "just in case".
- `memo` children that receive inline objects, arrays, callbacks or JSX, so memoization never hits.
- One big context with fast-changing values at the root of the app.
- Defining components, or calling `lazy`, inside another component.
- Random or index keys in dynamic lists, causing remounts or wrong reuse.
- Rendering thousands of rows without virtualization or pagination.
- Heavy synchronous work in render or click handlers, which hurts INP.
- Importing a whole library for one function, or shipping admin-only code to every user.

## How to talk about this in an interview
- Describe a method, not a bag of tricks: "I reproduce the slowness, profile it with the React DevTools Profiler and the Performance panel, find the component or work that dominates, apply the smallest fix (move state down, memoize a row, virtualize, split a chunk), and measure again."
- Know the re-render triggers cold (state, parent, context) and correct the props myth politely; it is a favorite trap.
- Mention the React Compiler as the direction of travel, and that it does not replace good structure.

> **Fill in:** a slow screen you investigated or would investigate at Devntech (for example a long list of applications, events or bills): what you measured, what caused it, and what changed.

> **Fill in:** whether React on Rails pages in the Colochousing app load one shared bundle or per-page bundles, and what you did (or would do) to keep them small.

- For React Native, connect this to list performance (`keyExtractor`, `getItemLayout`, memoized rows); see [RN performance](../08-react-native-expo/07-performance-and-animations.md).

## Interview questions

**Q1. What causes a React component to re-render?**
Its own state changes (including a subscribed store value), its parent re-renders, or a context it reads changes. When a component re-renders, its children re-render by default. Setting state to an identical value bails out, and changing a key remounts instead of re-rendering.
*Follow-up:* Does every re-render touch the DOM? No, only the differences are committed.

**Q2. Does a component re-render when its props change?**
Not exactly. A child re-renders because its parent re-rendered, and new props arrive as part of that. Without `memo`, the child re-renders even with identical props. With `memo`, React compares props shallowly and skips the render when nothing changed; only then do props decide.
*Follow-up:* Can props change without the parent re-rendering? No, props only come from the parent's render.

**Q3. What is the difference between render and commit?**
Render calls your components and diffs the new element tree against the previous one; it is pure and can be interrupted or discarded. Commit applies the resulting DOM changes, attaches refs and runs layout effects; effects run after paint. An expensive render blocks the main thread even when the commit is tiny.
*Follow-up:* Which phase does `memo` save? The render of the skipped subtree, and any commit work it would have caused.

**Q4. How do you find performance problems in a React app?**
Reproduce the slow interaction, record it in the React DevTools Profiler, look for slow commits and components that render too often, and enable "record why each component rendered". Check the browser Performance panel for long tasks, layout work and network waterfalls, and confirm with a production build. Fix the biggest cost first and re-measure.
*Follow-up:* Why not trust development timings? Development builds add checks and double renders.

**Q5. What does React.memo do, and when does it fail?**
It makes React skip re-rendering a component when every prop is `Object.is`-equal to the previous render. It fails when the parent passes new objects, arrays, inline functions or JSX on every render, since those are never equal. Stabilize them with `useMemo`/`useCallback`, or pass primitives.
*Follow-up:* Does `memo` block context updates? No, a memoized component still re-renders when a context it reads changes.

**Q6. useMemo vs useCallback vs React.memo: when does each help?**
`React.memo` skips a component's render when its props are unchanged. `useCallback` and `useMemo` keep function and object identities stable so that `memo`, or an effect's dependency array, can see "unchanged"; `useMemo` also caches expensive calculations. Without a memoized consumer or a real cost, they only add overhead.
*Follow-up:* Can you avoid writing them by hand? The React Compiler inserts equivalent memoization at build time.

**Q7. What are "moving state down" and "lifting content up"?**
Moving state down keeps state in the smallest component that uses it, so updates re-render only that subtree. Lifting content up passes expensive subtrees as `children` from a parent; when the wrapper's own state changes, the `children` element is unchanged, so React skips it. Both fix re-render problems without memoization.
*Follow-up:* Why does `children` not re-render? It is the same element object created by the parent's previous render.

**Q8. How does Context affect performance?**
Every consumer re-renders when the provider's value changes, regardless of `memo`, and there are no selectors. A new object literal as the value re-renders all consumers on every provider render. Memoize the value, split contexts by concern and update frequency, and use a store with selectors for high-frequency state.
*Follow-up:* Good fits for context? Rarely changing values such as theme, locale or the current user.

**Q9. What is code splitting, and how do you implement it in React?**
Breaking the bundle into chunks that load on demand, so the first load ships less JavaScript. Use `React.lazy(() => import('./Page'))` inside a `<Suspense fallback>` boundary, split by route first and heavy widgets second, use dynamic `import()` for big libraries inside handlers, and preload likely next chunks on hover.
*Follow-up:* What can break after a deploy? Old chunk files disappear; an error boundary should offer a reload.

**Q10. How would you render a list of 10,000 items?**
First ask whether users need them all at once: pagination or infinite loading may be better. If they do, virtualize with TanStack Virtual or react-window so only visible rows (plus overscan) exist in the DOM, memoize the row component, use stable keys and keep rows cheap. In React Native, use `FlatList` or FlashList.
*Follow-up:* Downsides of virtualization? Find-in-page misses unrendered rows, and variable heights need measuring.

**Q11. How do you keep an input responsive while filtering a large list?**
Keep the input's own state urgent and render the expensive list from `useDeferredValue(query)` with the list wrapped in `memo`, or update the filter inside `startTransition`. Memoize the filtered result and virtualize if the list is long. If filtering happens on the server, debounce the request instead.
*Follow-up:* How do you show results catching up? Compare `query !== deferredQuery` and dim the list.

**Q12. What is tree shaking, and why might it not work?**
The bundler drops exports nobody imports. It needs static ES module syntax and side-effect-free modules; it fails with CommonJS packages, modules with side effects on import, dynamic access to namespace objects and some barrel-file setups. Verify with a bundle visualizer.
*Follow-up:* Example fix? Replace `import _ from 'lodash'` with `lodash-es` or per-function imports.

**Q13. What is the React Compiler, and does it make useMemo obsolete?**
A build-time compiler, stable since late 2025, that automatically memoizes components and hooks that follow the Rules of React. In new code it removes most manual `useMemo`/`useCallback`/`memo`, but existing ones keep working, a few cases still need manual control, and it does not fix structural problems such as huge lists or waterfalls.
*Follow-up:* What does it require from your code? Pure renders, no mutation of props or state, and the rules of hooks.

**Q14. Why is defining a component inside another component a performance bug?**
The inner function is recreated on every render, so React sees a new component type each time and unmounts and remounts the subtree, losing state and focus and rebuilding the DOM. Define components at module level and pass data through props.
*Follow-up:* Is calling a render helper the same? No: `renderRow()` just returns JSX; the problem is using a freshly created function as a component type (`<Row />`).

**Q15. Debounce vs throttle vs useDeferredValue?**
Debounce waits until events stop for a delay, then runs once (search requests). Throttle runs at most once per interval while events continue (scroll and resize handlers). `useDeferredValue` delays nothing; it renders an expensive part with a lagging value at lower priority and interrupts it when new input arrives.
*Follow-up:* Which one reduces network calls? Debounce; deferred values only change rendering priority.

**Q16. A page lags when you type in a form. Walk me through fixing it.**
Profile a keystroke and see what re-renders. Typical causes: form state high in the tree re-rendering the whole page (move it down, or use React Hook Form, which is uncontrolled), an expensive list or chart re-rendering per keystroke (memoize it and defer its input), a context updated on every keystroke (split it), or heavy validation per keystroke (validate on blur or debounce). Re-measure after each change.
*Follow-up:* How do you prove the fix? Compare Profiler commit durations before and after.

## Further reading
- [memo reference](https://react.dev/reference/react/memo)
- [lazy reference](https://react.dev/reference/react/lazy)
- [React Compiler](https://react.dev/learn/react-compiler)
- [TanStack Virtual](https://tanstack.com/virtual/latest)
