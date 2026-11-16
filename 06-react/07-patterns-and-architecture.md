# React Patterns and Architecture

> **Tier:** T1 (know cold) · **Related:** [Fundamentals](01-fundamentals.md), [Hooks](02-hooks.md), [State management](03-state-management.md), [Performance](06-performance.md), [Routing](08-routing.md), [React 19](09-react-19-and-server-components.md), [Design patterns](../01-cs-fundamentals/04-design-patterns.md), [Tailwind CSS](../04-web-fundamentals/03-tailwind-css.md)
> **On your resume:** Structuring components and folders is daily work across Devntech's codebases (React on Rails inside the Colochousing Rails 7 app, the TanStack Start platform migration, the Coloc resident app) and shaped the role-based dashboards of the PureLogics LMS.

## Quick revision
- Composition first: `children` and slot props (`header`, `actions`, `sidebar`) build flexible components without an explosion of boolean props.
- Compound components share implicit state through context: `<Tabs><Tabs.List /><Tabs.Panel /></Tabs>`.
- Render props pass a function that returns UI. Hooks replaced most uses, but libraries still use them (React Hook Form's `Controller`, virtualized lists).
- Higher-order components (HOCs) wrap a component to inject props. Hooks replaced most of them because HOCs cause wrapper hell, prop name collisions and weak typing.
- Custom hooks are the main unit of reuse for stateful logic.
- Container/presentational splits data and side effects from pure UI; today the "container" is often just a custom hook.
- Component APIs can be controlled (`value` + `onChange`), uncontrolled (`defaultValue`), or support both like native inputs.
- Error boundaries catch errors thrown while rendering their children. They are still class components (or `react-error-boundary`) and do not catch errors in event handlers, async code, SSR or the boundary itself.
- Portals (`createPortal`) render into another DOM node (modals, tooltips) while keeping React context and React event bubbling.
- React 19 passes `ref` as a normal prop; `forwardRef` is legacy. `useImperativeHandle` exposes a custom handle instead of the DOM node.
- Feature-based folders (`features/bills`) beat type-based folders (`components/`, `hooks/`) as apps grow.
- Design systems: shadcn/ui copies Radix-plus-Tailwind components into your repo; MUI is an installed library with its own theming; headless libraries give behavior and accessibility without styles.
- Avoid prop drilling in this order: composition, context, a store.

## 1. Composition with children and slots

```tsx
import type { ReactNode } from 'react';

type PageLayoutProps = { title: string; actions?: ReactNode; sidebar?: ReactNode; children: ReactNode };

export function PageLayout({ title, actions, sidebar, children }: PageLayoutProps) {
  return (
    <div className="page">
      <header className="page-header">
        <h1>{title}</h1>
        <div className="page-actions">{actions}</div>
      </header>
      <div className="page-body">
        {sidebar && <aside>{sidebar}</aside>}
        <main>{children}</main>
      </div>
    </div>
  );
}

// <PageLayout title="Bills" actions={<Button>New bill</Button>} sidebar={<BillFilters />}>
//   <BillTable />
// </PageLayout>
```

The anti-pattern is "prop explosion": `showIcon`, `iconPosition`, `titleSize`, `hasFooter`, `footerText`, one new flag per use case until the component is an unreadable switchboard. Slots let each caller pass exactly the UI it needs, and the component stays small.

## 2. Compound components

Compound components are a set of components that work together and share state implicitly, like `<select>` and `<option>`. The parent owns the state and exposes it through context; the children read it.

```tsx
import { createContext, useContext, useId, useMemo, useState, type ReactNode } from 'react';

type TabsContextValue = { active: string; setActive: (value: string) => void; baseId: string };
const TabsContext = createContext<TabsContextValue | null>(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.* components must be rendered inside <Tabs>');
  return ctx;
}

function Tabs({ defaultValue, children }: { defaultValue: string; children: ReactNode }) {
  const [active, setActive] = useState(defaultValue);
  const baseId = useId();
  const value = useMemo(() => ({ active, setActive, baseId }), [active, baseId]);
  return <TabsContext.Provider value={value}>{children}</TabsContext.Provider>;
}

function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ value, children }: { value: string; children: ReactNode }) {
  const { active, setActive, baseId } = useTabs();
  const selected = active === value;
  return (
    <button
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={selected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={selected ? 0 : -1}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }: { value: string; children: ReactNode }) {
  const { active, baseId } = useTabs();
  if (active !== value) return null;
  return (
    <div role="tabpanel" id={`${baseId}-panel-${value}`} aria-labelledby={`${baseId}-tab-${value}`}>
      {children}
    </div>
  );
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
export { Tabs };
```

```tsx
<Tabs defaultValue="events">
  <Tabs.List>
    <Tabs.Tab value="events">Events</Tabs.Tab>
    <Tabs.Tab value="tasks">Tasks</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="events"><EventList /></Tabs.Panel>
  <Tabs.Panel value="tasks"><TaskList /></Tabs.Panel>
</Tabs>
```

Why it is nice: the caller controls structure and markup (put the list below the panels, add icons, wrap tabs in tooltips) while the state logic stays inside. A production version adds arrow-key navigation between tabs, which headless libraries such as Radix provide. The question bank has an accordion built the same way.

## 3. Render props

A render prop is a function prop that the component calls to decide what to render, passing its internal state.

```tsx
type Position = { x: number; y: number };

function MouseTracker({ children }: { children: (pos: Position) => ReactNode }) {
  const [pos, setPos] = useState<Position>({ x: 0, y: 0 });
  return <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>{children(pos)}</div>;
}

// <MouseTracker>{({ x, y }) => <p>Pointer at {x}, {y}</p>}</MouseTracker>
// Hook version of the same logic: const { x, y } = useMousePosition();
```

Hooks replaced render props for sharing logic, but the pattern is still right when a component must let the caller render part of its output with data only the component has: React Hook Form's `Controller` (`render={({ field }) => ...}`), virtualized lists rendering rows, autocomplete libraries rendering options.

## 4. Higher-order components

A HOC is a function that takes a component and returns a new component with extra behavior.

```tsx
import type { ComponentType } from 'react';
import { Navigate } from 'react-router';

function withAuth<P extends object>(Component: ComponentType<P>) {
  return function WithAuth(props: P) {
    const { user } = useAuth();
    if (!user) return <Navigate to="/login" replace />;
    return <Component {...props} />;
  };
}

export const BillingPage = withAuth(BillingPageContent);
```

Why hooks replaced most HOCs:
- **Wrapper hell:** several HOCs nest the tree (`withAuth(withTheme(withRouter(Page)))`) and clutter DevTools.
- **Prop collisions:** two HOCs injecting a prop with the same name silently overwrite each other.
- **Indirection and typing:** it is unclear where a prop comes from, and typing injected props is painful.
- **Static composition:** HOCs are applied at definition time; hooks can use props and state directly.

You still meet HOCs in legacy code (`connect()` from react-redux, `withRouter` from React Router v5, `withStyles`), and `memo` and `lazy` are HOC-shaped functions. Modern equivalents: `useAuth()` inside the component, or an auth guard layout route ([routing](08-routing.md)).

## 5. Custom hooks as the main reuse unit

Custom hooks package stateful logic so components stay declarative ([hooks](02-hooks.md)).

```tsx
export function useHouseBills(houseId: string) {
  const query = useQuery(billQueries.list({ houseId }));
  const unpaidTotalCents = useMemo(
    () => (query.data ?? []).filter((b) => !b.paid).reduce((sum, b) => sum + b.amountCents, 0),
    [query.data],
  );
  return { ...query, unpaidTotalCents };
}

// function BillsSummary({ houseId }: { houseId: string }) {
//   const { isPending, unpaidTotalCents } = useHouseBills(houseId);
//   ...
// }
```

Benefits: components read like a description of the UI, the hook is testable on its own with `renderHook`, and changing the data layer (REST to oRPC, for example) touches one file. Keep hooks focused: a "god hook" that returns 20 values is a component in disguise.

## 6. Container and presentational components

```tsx
// Presentational: props in, UI out. Easy to test, easy to show in Storybook.
function BillListView({ bills, onPay }: { bills: Bill[]; onPay: (id: string) => void }) {
  return (
    <ul>
      {bills.map((b) => (
        <li key={b.id}>
          {b.title} <button onClick={() => onPay(b.id)} disabled={b.paid}>Pay</button>
        </li>
      ))}
    </ul>
  );
}

// Container: data and side effects, now usually thin thanks to custom hooks
function BillListContainer({ houseId }: { houseId: string }) {
  const { data = [], isPending } = useHouseBills(houseId);
  const payBill = usePayBill();
  if (isPending) return <Spinner />;
  return <BillListView bills={data} onPay={(id) => payBill.mutate(id)} />;
}
```

The pattern was popularized before hooks; its author later said hooks make the strict split less necessary. The useful core remains: keep reusable UI components free of data fetching and global state, and keep data logic in hooks or route-level components.

## 7. Controlled vs uncontrolled component APIs

Reusable inputs, toggles, tabs and accordions should behave like native inputs: controlled when the parent passes `value`, uncontrolled otherwise.

```tsx
type ToggleProps = { checked?: boolean; defaultChecked?: boolean; onChange?: (checked: boolean) => void };

export function Toggle({ checked, defaultChecked = false, onChange }: ToggleProps) {
  const [internal, setInternal] = useState(defaultChecked);
  const isControlled = checked !== undefined; // decided by whether the parent passes `checked`
  const value = isControlled ? checked : internal;

  function toggle() {
    if (!isControlled) setInternal(!value);
    onChange?.(!value); // controlled parents update their own state here
  }

  return (
    <button role="switch" aria-checked={value} onClick={toggle}>
      {value ? 'On' : 'Off'}
    </button>
  );
}

// Uncontrolled: <Toggle defaultChecked onChange={savePreference} />
// Controlled:   <Toggle checked={notifyByEmail} onChange={setNotifyByEmail} />
```

Rules: never switch between modes during a component's life (React warns for native inputs for the same reason), and name props after the native conventions (`value`/`defaultValue`/`onChange`, `open`/`defaultOpen`/`onOpenChange`).

## 8. Error boundaries

An error boundary catches errors thrown while rendering, in lifecycle methods and in constructors of its descendants, and shows a fallback instead of unmounting the whole app. There is no hook for this yet, so boundaries are classes:

```tsx
import { Component, type ErrorInfo, type ReactNode } from 'react';

type Props = { fallback: ReactNode; children: ReactNode };
type State = { hasError: boolean };

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true }; // render the fallback on the next render
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    logError(error, info.componentStack); // your logger or error tracker
  }

  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}
```

**What they do not catch:**
- errors in event handlers (they happen outside rendering; use `try/catch` and set error state);
- async code (`setTimeout`, promise callbacks) unless you route the error back into rendering;
- server-side rendering errors;
- errors thrown by the boundary itself (a parent boundary catches those).

In practice most teams use the `react-error-boundary` package:

```tsx
import { ErrorBoundary, useErrorBoundary, type FallbackProps } from 'react-error-boundary';

function Fallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <p>Something went wrong: {error instanceof Error ? error.message : 'Unknown error'}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

// resetKeys: the boundary resets automatically when houseId changes
<ErrorBoundary FallbackComponent={Fallback} resetKeys={[houseId]}>
  <HouseDashboard houseId={houseId} />
</ErrorBoundary>;

function ExportButton() {
  const { showBoundary } = useErrorBoundary();
  return <button onClick={() => exportCsv().catch(showBoundary)}>Export</button>; // async error -> nearest boundary
}
```

Placement: one boundary at the root as a last resort, one per route, and around risky widgets (third-party charts, maps, embeds) so one failure does not blank the page. Next.js `error.tsx` and React Router's `errorElement` are route-level boundaries. React 19 adds `onCaughtError` and `onUncaughtError` options on `createRoot` for central logging.

## 9. Portals for modals

```tsx
import { createPortal } from 'react-dom';

function Modal({ title, onClose, children }: { title: string; onClose: () => void; children: ReactNode }) {
  return createPortal(
    <div className="backdrop" onClick={onClose}>
      <div role="dialog" aria-modal="true" aria-label={title} onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body, // rendered here, outside the parent's DOM subtree
  );
}
```

- **Why:** a modal inside a card with `overflow: hidden` or a low `z-index` stacking context gets clipped or hidden. A portal moves the DOM node to `document.body` while the component stays where it is in the React tree.
- **Events bubble through the React tree, not the DOM tree:** a click inside the portal triggers `onClick` handlers on React ancestors of `Modal`, even though the DOM node lives in `body`. Context also works normally.
- **An accessible modal needs more:** a label (`aria-labelledby` or `aria-label`), focus moved into the dialog and restored to the trigger on close, Escape to close, focus kept inside while open, and background scroll locked. The native `<dialog>` element with `showModal()` gives you Escape, the top layer and inert background for free; Radix Dialog or Headless UI handle the rest. A complete implementation is a live-coding exercise in the [question bank](99-interview-questions.md).

## 10. Refs: forwardRef, ref as a prop, useImperativeHandle

```tsx
import { forwardRef, type ComponentProps, type ComponentPropsWithoutRef } from 'react';

// React 18 and earlier: function components need forwardRef to receive a ref
export const InputLegacy = forwardRef<HTMLInputElement, ComponentPropsWithoutRef<'input'>>(
  function InputLegacy(props, ref) {
    return <input ref={ref} className="input" {...props} />;
  },
);

// React 19: ref is a regular prop for function components
export function Input({ ref, ...props }: ComponentProps<'input'>) {
  return <input ref={ref} className="input" {...props} />;
}
```

- `forwardRef` still works in React 19 but is expected to be deprecated; new components can take `ref` like any prop.
- `useImperativeHandle(ref, () => ({ focus, scrollToBottom }))` exposes a small custom API instead of the raw DOM node, for imperative needs such as focus, scrolling or media control (example in [hooks](02-hooks.md)). Prefer props and state for everything else.
- React 19 ref callbacks can return a cleanup function, which is handy for observers attached to a node.

## 11. Feature-based folder structure

```text
src/
  app/                      # app shell: providers, router, global styles
    providers.tsx
    router.tsx
  routes/                   # thin route components: read params, compose features
  features/
    bills/
      api/                  # queryOptions, mutation hooks, API types
      components/           # BillList, BillForm, BillRow
      hooks/                # useHouseBills, useToggleBillPaid
      utils/                # splitAmount.ts with splitAmount.test.ts next to it
      index.ts              # public API: the only file other features import
    events/
    auth/
  shared/
    ui/                     # Button, Modal, TextField: the design system
    lib/                    # fetchJson, formatMoney, date helpers
    hooks/                  # useDebounce, useMediaQuery
```

- Group by feature (domain), not by file type. When you change bills, everything you touch is in one folder.
- Dependencies point one way: `app` and `routes` use `features`, features use `shared`, features talk to each other only through `index.ts`. Lint rules can enforce boundaries in big codebases.
- Route files stay thin; colocate tests, stories and styles with the code they cover.
- Keep `shared/` for genuinely generic code; a giant `utils/` folder is a smell. Formal methods such as Feature-Sliced Design formalize the same idea.
- With React on Rails, components often live under the Rails app's JavaScript directory and are registered for use from views; the same feature grouping applies ([React on Rails](../05-ruby-on-rails/16-react-on-rails.md)). With file-based routers (TanStack Router, Next.js, Expo Router), the `routes/` or `app/` folder is dictated by the router, and features live beside it.

## 12. Component libraries and design systems

| | shadcn/ui | MUI | Headless (Radix UI, Headless UI, React Aria) |
|---|---|---|---|
| How you get it | a CLI copies component source into your repo (`npx shadcn@latest add dialog`) | npm dependency | npm dependency |
| Styling | Tailwind classes you edit freely | its theme system (`createTheme`, `ThemeProvider`, `sx` prop) | you style everything |
| Default look | neutral, fully yours | Material Design | none |
| Behavior and accessibility | from the Radix primitives underneath | built in | built in (their main value) |
| Trade-off | you own updates and maintenance | heavier, harder to make it look non-Material | most styling work |

A design system is more than a component library: design tokens (colors, spacing, typography, radii) as CSS variables or a Tailwind theme, components that implement them, documentation (often Storybook) and usage guidelines. In React Native, NativeWind brings Tailwind-style classes to native components ([Tailwind CSS](../04-web-fundamentals/03-tailwind-css.md)).

### Going deeper: the asChild (slot) pattern

Radix UI components, and therefore shadcn/ui, accept an `asChild` prop: instead of rendering their own element, they merge their props, event handlers and ref onto their single child. `<Dialog.Trigger asChild><Button variant="ghost">Open</Button></Dialog.Trigger>` keeps your `Button` styling while the trigger supplies the behavior and ARIA attributes. It solves the same problem as a polymorphic `as` prop (`<Text as="label">`) with simpler typing.

## 13. Avoiding prop drilling

1. **Composition:** build the element where the data is and pass it down as `children` or a slot, so intermediate components never see the prop.
2. **Context:** for values many distant components need and that rarely change (user, theme, current organization).
3. **A store:** Zustand, Redux Toolkit or Jotai for frequently changing shared client state with selectors.
4. Server data never needs drilling: any component can call the same `useQuery` and hit the cache.

Details and code in [state management](03-state-management.md).

## Common pitfalls
- Boolean-prop explosions instead of slots and composition.
- Compound components whose children crash outside the parent: always throw a clear error from the context hook.
- New HOCs for logic a custom hook would express more simply.
- Error boundaries expected to catch event handler or promise errors.
- One root error boundary only, so any widget failure blanks the whole app.
- Modals without focus management, Escape handling or labels.
- Switching a reusable component between controlled and uncontrolled during its life.
- `forwardRef` boilerplate in new React 19 code, or forgetting it when still on React 18.
- Type-based folders (`components/`, `hooks/`, `utils/`) that become dumping grounds with hundreds of files.
- Forking a component library's components without a plan for updates.

## How to talk about this in an interview
- Show you pick patterns for reasons: composition for flexibility, compound components for related widgets, custom hooks for logic reuse, error boundaries per route and risky widget, portals for overlays.
- For architecture questions, describe a feature-based structure, one-way dependencies, a thin route layer, a shared UI kit, and the data layer (query options per feature).

> **Fill in:** how the codebases you work on at Devntech are organized (React on Rails components inside the Rails app, the TanStack Start platform, the Expo app), and one structural improvement you made or proposed.

> **Fill in:** a reusable component you built (a modal, a form field, tabs, a list row) and how you designed its props.

- At PureLogics, role-based dashboards are a good composition story: shared layout and widgets, with role-specific panels plugged into slots.
- If you mostly used a library such as shadcn/ui or MUI, say how you customized it rather than claiming you built a design system from scratch.

## Interview questions

**Q1. What are compound components, and how would you build Tabs?**
A group of components that share implicit state, like `<select>` and `<option>`. `Tabs` holds the active value in state and provides it through context; `Tabs.List`, `Tabs.Tab` and `Tabs.Panel` read it with a hook that throws if used outside `Tabs`. Callers control structure and markup while the logic stays inside.
*Follow-up:* What makes it accessible? `tablist`, `tab` and `tabpanel` roles, `aria-selected`, `aria-controls`, and arrow-key navigation.

**Q2. What are render props, and are they still useful?**
A prop that is a function returning UI, called by the component with its internal state: `<MouseTracker>{(pos) => ...}</MouseTracker>`. Hooks replaced them for sharing logic, but they remain useful when the caller must render part of a component's output with data only the component has, such as React Hook Form's `Controller` or row renderers in virtualized lists.
*Follow-up:* Hook equivalent of `MouseTracker`? A `useMousePosition()` hook returning `{ x, y }`.

**Q3. What are higher-order components, and why did hooks replace most of them?**
A HOC is a function that takes a component and returns an enhanced component (`withAuth(Page)`). They cause wrapper hell, prop name collisions, unclear prop origins and awkward typing. Hooks share the same logic as a function call inside the component, with direct access to props and state.
*Follow-up:* Where do you still see HOCs? Legacy `connect()` from react-redux, `withRouter` in old React Router, and HOC-shaped utilities like `memo`.

**Q4. Why are custom hooks the main reuse unit in modern React?**
They package stateful logic (data fetching, subscriptions, form logic, derived values) as plain functions that compose without changing the component tree. Each caller gets its own state, they are testable with `renderHook`, and they keep components declarative. Changing an implementation detail, such as the data layer, touches one file.
*Follow-up:* What makes a bad custom hook? One that does too much and returns a dozen loosely related values.

**Q5. What is the container/presentational pattern, and is it still relevant?**
Containers handle data and side effects; presentational components receive props and render UI. Hooks made the strict split less necessary because data logic now lives in custom hooks, but the core idea still pays off: reusable UI components without fetching or global state are easy to test, document and reuse.
*Follow-up:* Where does data fetching go then? In custom hooks used by route-level or feature components.

**Q6. How do you design a component that works both controlled and uncontrolled?**
Accept `value` and `defaultValue` (or `checked` and `defaultChecked`) plus `onChange`. Keep internal state initialized from the default, decide `isControlled = value !== undefined`, render from the prop when controlled and from internal state otherwise, and always call `onChange`. Never switch modes during the component's life.
*Follow-up:* Why support both? Simple uses need no state in the parent, while forms and synced UIs need full control.

**Q7. What are error boundaries, and what do they not catch?**
Class components with `static getDerivedStateFromError` (to show a fallback) and `componentDidCatch` (to log) that catch errors thrown while rendering their descendants. They do not catch errors in event handlers, async callbacks such as timers and promises, server rendering, or the boundary itself. Most teams use `react-error-boundary` for a ready-made component with reset support.
*Follow-up:* Why are they still classes? There is no hook equivalent of those two lifecycle methods.

**Q8. How do you handle errors from event handlers and async code?**
Wrap the handler's work in `try/catch` and show the error through state (an inline message or a toast). If the error should take down the section like a render error, pass it to the nearest boundary with `useErrorBoundary().showBoundary(error)`. Data libraries expose `error` state and can opt into boundaries (`throwOnError` in TanStack Query).
*Follow-up:* Can you rethrow inside a state updater? Yes, `setState(() => { throw error; })` turns it into a render error, which is essentially what `showBoundary` does for you.

**Q9. What are portals, and how do events behave inside them?**
`createPortal(children, domNode)` renders children into a different DOM node, typically `document.body`, to escape `overflow: hidden` and stacking contexts for modals, tooltips and toasts. The component stays in the same place in the React tree, so context works and events bubble to React ancestors, not to DOM ancestors.
*Follow-up:* Gotcha? A click inside a portal can trigger an `onClick` on a React parent; stop propagation where needed.

**Q10. How would you build an accessible modal?**
Render it through a portal with `role="dialog"`, `aria-modal="true"` and a label. Move focus into it on open and back to the trigger on close, close on Escape and on backdrop click, keep Tab focus inside, and lock background scroll. Or use the native `<dialog>` element with `showModal()`, or Radix Dialog, which implement most of this.
*Follow-up:* Why restore focus? Keyboard and screen reader users would otherwise be dropped at the top of the page.

**Q11. How do you pass a ref to a child component, and what changed in React 19?**
Before React 19, a function component had to be wrapped in `forwardRef((props, ref) => ...)` to receive a ref. In React 19, `ref` is a regular prop for function components, so you destructure it and pass it on; `forwardRef` still works but is on the way out. Ref callbacks can also return a cleanup function in React 19.
*Follow-up:* Does a class component need any of this? No, a ref to a class component points at its instance.

**Q12. When would you use useImperativeHandle?**
When a parent needs to trigger an imperative action on a child (focus an input, scroll a list to the bottom, play a video) and you want to expose a small API instead of the whole DOM node or internals. Most interactions should still flow through props and state.
*Follow-up:* What does the parent's ref contain then? Only the object returned by the `useImperativeHandle` factory.

**Q13. How would you structure a large React app?**
Feature-based folders (`features/bills` with its components, hooks, API layer and tests), a `shared/` folder for the design system and generic helpers, a thin route layer, and app-level providers in one place. Dependencies point one way (routes to features to shared), features expose a public `index.ts`, server state lives in query hooks per feature, and lint rules plus TypeScript enforce boundaries.
*Follow-up:* What about cross-feature needs? Promote the shared piece into `shared/` or communicate through server state or URL state.

**Q14. What makes a good reusable component API?**
Native-like naming (`value`, `defaultValue`, `onChange`, `disabled`), composition through `children` and slots instead of many boolean flags, support for native props via `ComponentProps<'button'>`, ref forwarding, both controlled and uncontrolled modes where it makes sense, accessible defaults, and strict TypeScript types. It should do one thing well.
*Follow-up:* How do you avoid breaking consumers? Additive changes, deprecations with warnings, and tests or stories for every variant.

**Q15. shadcn/ui vs MUI: how do you choose?**
shadcn/ui copies accessible Radix-based components styled with Tailwind into your repo, so you own and freely edit them, which suits a custom design and Tailwind stack. MUI is an installed, complete library with theming and many complex components (data grids, date pickers), which suits fast delivery when a Material-like look is acceptable. Headless libraries sit in between: behavior without styles.
*Follow-up:* Downside of shadcn/ui? Updates are manual because the code is yours.

**Q16. How do you avoid prop drilling?**
First composition: build elements where the data lives and pass them down as `children` or slots. Then context for rarely changing values needed far away, and a store with selectors for frequently changing shared state. Server data needs no drilling at all, because any component can read the query cache.
*Follow-up:* When is drilling fine? For two or three explicit levels; it is the most readable option.

## Further reading
- [Passing data deeply with context](https://react.dev/learn/passing-data-deeply-with-context)
- [Error boundaries in the Component reference](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [createPortal reference](https://react.dev/reference/react-dom/createPortal)
