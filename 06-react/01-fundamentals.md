# React Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [State management](03-state-management.md), [Performance](06-performance.md), [Patterns](07-patterns-and-architecture.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [DOM and events](../03-languages/javascript/06-browser-dom-events.md), [React on Rails](../05-ruby-on-rails/16-react-on-rails.md)
> **On your resume:** React and TypeScript in your daily work at Devntech (React on Rails in the Colochousing Rails 7 app, the new TanStack Start platform), the DRF + React LMS at PureLogics, and the React Native (Expo) resident app, which runs on the same component model.

## Quick revision
- React is a library for building UIs from components. The UI is a function of state (`UI = f(state)`): you describe what the screen looks like for the current data, and React updates the DOM to match.
- JSX is syntax sugar. `<Button kind="primary">Save</Button>` compiles to `jsx(Button, { kind: "primary", children: "Save" })` (automatic runtime, React 17+) or `React.createElement(...)` (classic runtime). The result is a plain object called a React element.
- A component is a function that takes props and returns JSX. Its name must start with a capital letter. Class components are legacy, but error boundaries still need them.
- Props are read-only inputs from the parent; `children` is whatever sits between the tags. Defaults come from destructuring (`{ size = "md" }`).
- State is data a component owns that changes over time. Calling the setter schedules a re-render; inside the current render the value is a fixed snapshot.
- Render cycle: trigger (initial render or state update) -> render (React calls your components and diffs the result) -> commit (React applies the minimal DOM changes) -> browser paints -> effects run.
- Rendering must be pure: the same props and state give the same JSX, with no side effects during render.
- Reconciliation: a different element type at the same position destroys and rebuilds that subtree; the same type keeps the DOM node or component instance (and its state) and updates props; keys match list children.
- Keys must be stable and unique among siblings, ideally database ids. Index keys attach state to the wrong items when the list is inserted into, filtered or reordered. Changing a key remounts the component and resets its state.
- Conditional rendering: ternary, `&&` (beware: `0 && <X />` renders `0`), or an early `return null`.
- Events: camelCase props taking functions (`onClick={handleClick}`, not `onClick={handleClick()}`). React passes a `SyntheticEvent` and, since React 17, listens at the root container instead of `document`.
- One-way data flow: data goes down through props, changes go up through callback props. Shared state lives in the closest common parent ("lifting state up").
- Composition over inheritance: build with `children` and slot props, never component class hierarchies.
- `StrictMode` (development only) renders components twice, runs effects setup-cleanup-setup on mount and warns about deprecated APIs, to expose impure renders and missing cleanup.
- React DevTools: the Components tab shows props, state, hooks and the owner tree; the Profiler records commits and why each component rendered.
- New projects: Vite for a single-page app, or a framework (Next.js, React Router framework mode, TanStack Start, Expo for mobile). Create React App is deprecated.

## 1. What React is and why it exists

Before React, interactive pages were built imperatively: find a DOM node, mutate it, and remember every other node that depends on the same data. jQuery made that shorter, not simpler. As screens grow, the DOM itself becomes the place where state hides, and bugs come from forgetting one of the updates.

```ts
// Imperative: you keep every piece of UI in sync by hand
let count = 0;
const label = document.querySelector<HTMLSpanElement>('#count')!;
const warning = document.querySelector<HTMLParagraphElement>('#warning')!;

document.querySelector('#inc')!.addEventListener('click', () => {
  count++;
  label.textContent = String(count);
  warning.hidden = count <= 10; // forget this line and the UI lies
});
```

React flips the model. State lives in JavaScript, and a component describes the whole UI for the current state. When state changes, React calls the component again and works out the DOM operations itself.

```tsx
import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <span>{count}</span>
      {count > 10 && <p>That is a lot of clicks</p>}
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

Why teams pick React:
- **Declarative:** you describe what the UI should be, React handles how to get there.
- **Components:** small reusable pieces with clear inputs (props) and private state.
- **Portability:** the same mental model renders to the DOM (`react-dom`), to native views (React Native) and on the server.
- **It is a library, not a framework.** React covers rendering and state. Routing, data fetching and forms come from libraries (React Router, TanStack Query, React Hook Form) or from a framework (Next.js, TanStack Start).

## 2. JSX and what it compiles to

JSX lets you write markup inside TypeScript. The compiler (TypeScript, esbuild, Babel or SWC) turns it into function calls.

```tsx
// What you write
const link = <a href="/bills" className="link">Bills</a>;

// What the compiler emits (automatic runtime, React 17+), roughly:
// import { jsx as _jsx } from "react/jsx-runtime";
// const link = _jsx("a", { href: "/bills", className: "link", children: "Bills" });

// Classic runtime (the reason old files needed `import React`):
// const link = React.createElement("a", { href: "/bills", className: "link" }, "Bills");

// The runtime value is a plain, immutable object (a React element), roughly:
// { type: "a", key: null, props: { href: "/bills", className: "link", children: "Bills" } }
```

Rules that follow from "JSX is just function calls":
- A lowercase tag becomes a string type (`"div"`), a capitalized tag becomes a variable reference (`Button`). That is why components must be capitalized.
- A function returns one value, so a component returns a single root. A fragment (`<>...</>`) groups siblings without an extra DOM node.
- `{}` embeds any JavaScript expression (not statements): `{user.name}`, `{items.map(...)}`, `{isAdmin ? <Admin /> : null}`.
- Attribute names follow DOM properties: `className`, `htmlFor`, `tabIndex`, `onClick`; `style` takes an object (`style={{ marginTop: 8 }}`).
- JSX escapes strings before inserting them, which blocks most XSS. `dangerouslySetInnerHTML={{ __html: html }}` bypasses that, so sanitize first (see [web security](../04-web-fundamentals/07-web-security-owasp.md)).
- Creating elements is cheap: they are descriptions, not DOM nodes.

## 3. Components and props

```tsx
import type { ComponentProps, ReactNode } from 'react';

type ButtonProps = {
  variant?: 'primary' | 'ghost' | 'danger';
  children: ReactNode;
} & Omit<ComponentProps<'button'>, 'className'>; // also accept every native <button> prop

export function Button({ variant = 'primary', children, ...rest }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} {...rest}>
      {children}
    </button>
  );
}

// <Button variant="danger" onClick={deleteBill} disabled={isDeleting}>Delete</Button>
```

- **Props are read-only.** Never write `props.user.name = 'x'`. If a child needs to change data, the parent passes a callback (`onRename`) and owns the update.
- **`children`** is a normal prop: strings, elements, arrays or `null`. It is how components nest.
- **Defaults** come from destructuring. `defaultProps` on function components was deprecated and then removed in React 19 (classes still support it).
- **Typing:** a plain function with a typed props object is the common style. `React.FC` also works; since the React 18 types it no longer adds an implicit `children` prop. See [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md) for `ComponentProps`, generic components and event types.

### Class components (legacy)

You will meet them in older code, including older React on Rails bundles.

```tsx
import { Component } from 'react';

type Props = { label: string };
type State = { now: Date };

export class Clock extends Component<Props, State> {
  state: State = { now: new Date() };
  private timer?: ReturnType<typeof setInterval>;

  componentDidMount() {
    this.timer = setInterval(() => this.setState({ now: new Date() }), 1000);
  }

  componentWillUnmount() {
    clearInterval(this.timer);
  }

  render() {
    return <p>{this.props.label}: {this.state.now.toLocaleTimeString()}</p>;
  }
}
```

| Class API | Function component equivalent |
|---|---|
| `this.state` / `this.setState` (shallow merge) | `useState` / `useReducer` (the setter replaces the value) |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate(prevProps)` | `useEffect` with dependencies |
| `componentWillUnmount` | cleanup function returned from `useEffect` |
| `shouldComponentUpdate` / `PureComponent` | `React.memo` |
| `getDerivedStateFromError` / `componentDidCatch` | no hook yet, so error boundaries stay classes |

## 4. State and the render cycle

State is a component's memory. Each render sees a snapshot: calling the setter does not change the variable in the running code, it asks React for a new render with a new value.

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function addThree() {
    setCount(count + 1); // count is 0 in this render
    setCount(count + 1); // still 0
    setCount(count + 1); // still 0, so the next render shows 1
    // setCount((c) => c + 1) three times shows 3: each updater receives the latest queued value
  }

  return <button onClick={addThree}>{count}</button>;
}
```

Every update goes through the same steps:

```text
trigger   initial createRoot(...).render(<App />), a state setter, or a context change
render    React calls the component (and its children), gets a new element tree and diffs it
          against the previous one. Must be pure; React may pause it or throw it away
commit    React applies only the needed DOM mutations, attaches refs, runs layout effects
paint     the browser draws the screen
effects   useEffect callbacks run
```

```tsx
// main.tsx: the entry point of a Vite + React app
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { App } from './App';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Consequences worth saying out loud:
- A render is not a DOM update. If the new tree matches the old one, the commit touches nothing.
- When a component re-renders, its children re-render by default, whether or not their props changed (details in [performance](06-performance.md)).
- React 18 batches setter calls from the same event (even after `await` or inside `setTimeout`) into one render. See [hooks](02-hooks.md).
- Do not copy props into state (`useState(props.value)`): the argument is only the initial value, so later prop changes are ignored. Derive from props during render instead.

## 5. The virtual DOM and reconciliation

The "virtual DOM" is the tree of React elements your components return, plus React's internal bookkeeping (the Fiber tree, one node per component instance). On every update React compares the new tree with the previous one (reconciliation) and turns the differences into the smallest set of DOM operations.

Why bother: touching the real DOM triggers style, layout and paint work, while comparing plain objects is cheap. The real win is that it makes the declarative model practical: you re-describe everything, React applies only the diff. It is not magic speed; Svelte and Solid skip the virtual DOM and use compile-time or fine-grained reactivity instead.

A general tree diff is O(n^3). React uses heuristics that make it O(n):
1. **Different element types** at the same position (`<div>` to `<section>`, or `<LoginForm>` to `<SignupForm>`): React unmounts the old subtree (state is lost, effects are cleaned up) and mounts the new one.
2. **Same DOM element type:** React keeps the DOM node and patches only the changed attributes.
3. **Same component type:** React keeps the instance and its state, and re-renders it with the new props.
4. **Lists:** children are matched by `key`, not by position.

State is attached to a position in the tree, which explains a classic surprise:

```tsx
// Both branches render <TextField> at the same position, so React reuses the instance
// and the text typed into "Email" survives the switch to "Username".
{isLogin ? <TextField label="Email" /> : <TextField label="Username" />}

// Different keys mean different identities: switching now resets the field.
{isLogin ? <TextField key="email" label="Email" /> : <TextField key="username" label="Username" />}
```

```tsx
// Bug: Row is a brand-new function (a new component type) on every render of Parent,
// so React unmounts and remounts it: the note input loses focus and its text on each keystroke.
function Parent() {
  const [query, setQuery] = useState('');
  function Row() {
    return <input placeholder="note" />;
  }
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Row />
    </>
  );
}
// Fix: define Row at module level and pass it props.
```

### Going deeper: Fiber and concurrent rendering

Fiber (React 16) rewrote the reconciler so rendering is split into small units of work, one per component, that can be paused, prioritized and resumed. Concurrent rendering (React 18) builds on it: React can prepare a low-priority render in the background, interrupt it when the user types, and throw it away if it is outdated. That is why rendering must be pure, and it is what powers `useTransition`, `useDeferredValue` and streaming SSR.

## 6. Keys and lists

```tsx
type Todo = { id: string; title: string; done: boolean };

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} /> // the key goes on the element returned by map
      ))}
    </ul>
  );
}
```

Key rules:
- Unique among siblings (not globally) and stable across renders. Use ids from your data.
- Never generate keys during render (`Math.random()`, `crypto.randomUUID()`): every render gets new keys, so every item remounts and loses its state.
- `key` is not passed to the component as a prop. Pass the id separately if the child needs it.
- Use `<Fragment key={id}>` when one list item renders several siblings.

Why index keys break:

```tsx
// Each row holds DOM state: the uncontrolled note input
{todos.map((todo, index) => (
  <li key={index}>
    {todo.title} <input placeholder="note" />
  </li>
))}
// Delete the first todo. Before: keys 0, 1, 2. After: keys 0, 1.
// React concludes that key 2 was removed and that keys 0 and 1 got new titles.
// Titles re-render correctly from props, but the typed notes stay with the positions,
// so each note now sits next to the wrong todo.
```

Index keys are acceptable only for static lists that never reorder, filter or insert and whose items hold no state. Keys also give you a deliberate reset: `<ProfileForm key={userId} userId={userId} />` throws away the old form state when the user changes.

## 7. Conditional rendering

```tsx
function Inbox({ messages, loading }: { messages: Message[]; loading: boolean }) {
  if (loading) return <Spinner />; // early return

  return (
    <section>
      {messages.length > 0 && <Badge count={messages.length} />}
      {messages.length === 0 ? <EmptyState /> : <MessageList messages={messages} />}
    </section>
  );
}
```

- `false`, `null`, `undefined` and `true` render nothing. Numbers do render, so `{messages.length && <Badge />}` shows a stray `0` on the web and crashes React Native ("Text strings must be rendered within a `<Text>` component"). Compare explicitly.
- Prefer early returns and small extracted components over nested ternaries.
- Hiding with CSS keeps the DOM and state; conditional rendering unmounts and resets state. Choose deliberately.

## 8. Events

```tsx
import { useState, type ChangeEvent, type FormEvent } from 'react';

function RenameForm({ onRename }: { onRename: (name: string) => void }) {
  const [name, setName] = useState('');

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    setName(e.target.value);
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault(); // stop the browser's full-page form submission
    onRename(name.trim());
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={handleChange} />
      <button type="submit">Save</button>
    </form>
  );
}
```

- Pass a function, do not call it: `onClick={handleClick}` or `onClick={() => remove(id)}`. `onClick={remove(id)}` runs during render.
- Convention: props are named `onSomething`, handlers `handleSomething`.
- `SyntheticEvent` normalizes browser differences and exposes `preventDefault()`, `stopPropagation()`, `target`, `currentTarget` and `nativeEvent`.
- Since React 17, listeners live on the root container rather than `document` (this matters when React shares a page with other scripts, as in a Rails app), and event pooling is gone, so reading `e.target` after an `await` is safe.
- React's `onChange` on inputs fires on every keystroke (like the native `input` event), not on blur like the native `change` event.
- Events bubble through the React tree, including through portals. Use `onClickCapture` for the capture phase.

## 9. One-way data flow and lifting state up

Data flows down, changes flow up through callbacks. When two siblings need the same data, move the state to their closest common parent and pass it down.

```tsx
type Product = { id: string; name: string };

function ProductsPage({ products }: { products: Product[] }) {
  const [query, setQuery] = useState(''); // lifted: both children depend on it
  const visible = products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase())); // derived

  return (
    <>
      <SearchBar query={query} onQueryChange={setQuery} />
      <ProductList products={visible} />
    </>
  );
}

function SearchBar({ query, onQueryChange }: { query: string; onQueryChange: (q: string) => void }) {
  return <input value={query} onChange={(e) => onQueryChange(e.target.value)} placeholder="Search" />;
}
```

- Keep a single source of truth. `visible` is computed during render instead of living in a second state variable that can drift.
- Lifting too high causes prop drilling and re-renders more of the tree than needed. Composition, context and stores are the next steps: see [state management](03-state-management.md).

## 10. Composition over inheritance

Components reuse each other by nesting, not by `extends`. Slot props keep layouts flexible without a pile of boolean flags.

```tsx
import type { ReactNode } from 'react';

type CardProps = { title: ReactNode; actions?: ReactNode; children: ReactNode };

function Card({ title, actions, children }: CardProps) {
  return (
    <div className="card">
      <header className="card-header">
        <h3>{title}</h3>
        {actions}
      </header>
      <div className="card-body">{children}</div>
    </div>
  );
}

// Specialization is just wrapping
function RentCard({ amount }: { amount: number }) {
  return (
    <Card title="Rent" actions={<Button>Pay now</Button>}>
      Due on the 1st: {amount} EUR
    </Card>
  );
}
```

More patterns (compound components, render props, HOCs) live in [patterns and architecture](07-patterns-and-architecture.md).

## 11. StrictMode, DevTools and starting a project

**StrictMode** wraps part or all of the tree and adds development-only checks:
- calls component bodies and state initializers twice, to catch impure renders;
- mounts, unmounts and remounts every component once, so each effect runs setup, cleanup, setup (your effect "runs twice" on purpose);
- in React 19, also re-runs ref callbacks once to check ref cleanup;
- warns about deprecated APIs.

Production builds do none of this. If double invocation breaks something, the real bug is a missing cleanup or an impure render: fix that instead of removing StrictMode.

**React DevTools** (browser extension; React Native tooling ships it too):
- Components tab: select a component, inspect and edit props, state and hooks, see which component rendered it, jump to the source.
- Settings: "Highlight updates when components render" makes re-renders visible.
- Profiler tab: record an interaction, then read the flame graph commit by commit, with an option to record why each component rendered (see [performance](06-performance.md)).

**Starting a project today:**

```bash
# Single-page app with Vite + React + TypeScript
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

- Create React App is deprecated; do not start new projects with it.
- When you need routing, data loading and SSR, the React docs point to frameworks: Next.js ([Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md)), React Router in framework mode, TanStack Start ([TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md)), or Expo for mobile ([Expo fundamentals](../08-react-native-expo/02-expo-fundamentals.md)).
- Inside an existing Rails app, React on Rails mounts components into ERB views and passes props from the server (see [React on Rails](../05-ruby-on-rails/16-react-on-rails.md)).

## Common pitfalls
- Calling the handler during render: `onClick={save()}` instead of `onClick={save}`.
- Mutating state or props (`todos.push(x); setTodos(todos)`): the reference is unchanged, so React may skip the update. Create new arrays and objects.
- `{count && <X />}` rendering a stray `0`.
- Index keys on dynamic lists, or random keys generated during render.
- Defining a component inside another component, which remounts it on every render.
- Storing derived values (`fullName`, `filteredItems`) in state and then fighting to keep them in sync.
- Expecting state to change immediately after `setState`, logging the old value and assuming the update failed.
- Copying a prop into state and expecting it to follow later prop changes.
- Panicking when StrictMode doubles logs or requests in development. It is intended; production does not do it.
- Writing `class` and `for` instead of `className` and `htmlFor`.

## How to talk about this in an interview
- Lead with the mental model in one sentence: "I describe the UI as a function of state; when state changes React re-renders and reconciles the minimal DOM changes." Then show depth with keys, render versus commit, and state preserved by position.
- Tie it to your work: you build features in React and TypeScript at Devntech, both inside the Colochousing Rails 7 app through React on Rails and on the new TanStack Start platform. Contrast the two settings: components embedded in server-rendered ERB pages versus a full React application on TanStack Start with its own router and a typed oRPC API.

> **Fill in:** one React screen you built at Devntech (for example during the marketplace page migration): what state it held, where its data came from, and one bug you fixed in it.

- At PureLogics the LMS used React on top of a DRF API. Role-based dashboards are a natural example of conditional rendering and composition.

> **Fill in:** how the LMS dashboards differed per role, and how the UI decided what each role could see.

- Mention transfer: the Coloc resident app is React Native, so components, props, state and hooks carry over unchanged; only the primitives differ (`View` and `Text` instead of `div` and `span`). See [React Native fundamentals](../08-react-native-expo/01-react-native-fundamentals.md).
- If your class-component experience is light, say so and show you can still read it by mapping lifecycle methods to hooks.

## Interview questions

**Q1. What is React, and why use it instead of updating the DOM yourself?**
React is a component library where the UI is a function of state. With manual DOM code (or jQuery) every dependent node is updated by hand, so state ends up scattered across the DOM and updates get missed. In React you change state, React re-renders and reconciles the minimal DOM changes. The result is predictable, reusable and testable UI code.
*Follow-up:* Library or framework? A library: routing, data loading and SSR come from other libraries or frameworks such as Next.js, React Router or TanStack Start.

**Q2. What does JSX compile to? Can you use React without JSX?**
JSX compiles to function calls: `jsx(type, props)` from `react/jsx-runtime` with the automatic runtime (React 17+), or `React.createElement(type, props, ...children)` with the classic runtime. Each call returns a plain element object with `type`, `props` and `key`. So yes, you can call `createElement` by hand; it is just verbose.
*Follow-up:* Why don't files need `import React` anymore? The automatic runtime imports `jsx` for you.

**Q3. What is the virtual DOM, and how does reconciliation work?**
It is the in-memory tree of React elements (with the Fiber tree behind it). On each update React diffs the new tree against the previous one and applies only the differences to the real DOM. Its O(n) heuristics: a different element type at a position replaces the whole subtree, the same type keeps the node or instance and updates props, and list children are matched by key.
*Follow-up:* Is the virtual DOM faster than the real DOM? Not inherently. It makes re-describing the whole UI cheap enough; some frameworks skip it entirely.

**Q4. Why do lists need keys, and why is the array index a bad key?**
Keys tell React which item is which between renders, so it keeps, moves or removes the right instance and its state. With index keys, an insert, delete or reorder shifts the indexes: state such as input values or focus stays with the position and ends up on the wrong item, and more rows re-render than necessary. Use stable ids from the data; the index is fine only for static lists without state.
*Follow-up:* What does `key={Math.random()}` do? New keys every render, so every item remounts and loses state.

**Q5. What is the difference between props and state?**
Props are inputs from the parent and are read-only for the child. State is private data the component owns and changes through a setter, which triggers a re-render. A value that is state in a parent is often a prop in the child. If a value can be computed from props or other state, it is neither: compute it during render.
*Follow-up:* Can a child change its props? No, it asks the parent to change them through a callback prop.

**Q6. Walk me through what happens when you call a state setter.**
React queues the update and schedules a render, batched with other updates from the same event. In the render phase it calls the component with the new state and diffs the output against the previous tree. In the commit phase it applies DOM changes and runs layout effects; the browser paints; then `useEffect` callbacks run. If the new value is identical by `Object.is`, React bails out.
*Follow-up:* Why might React render without committing? Concurrent rendering can interrupt or discard a render, which is why rendering must be pure.

**Q7. Why does `console.log(count)` right after `setCount(count + 1)` print the old value?**
Each render sees a snapshot of state. The setter does not mutate the local `count`; it requests a new render in which `count` has the new value. If you need the new value immediately, compute it first: `const next = count + 1; setCount(next); track(next);`.
*Follow-up:* How do you increment twice in one handler? Use functional updates: call `setCount((c) => c + 1)` twice.

**Q8. What is one-way data flow? How do two sibling components share data?**
Data moves down the tree through props, and children report changes upward through callbacks. Siblings share data by lifting the state to their closest common parent, which passes the value and a change callback to each. When that parent is far away, reach for composition, context or a store.
*Follow-up:* Downside of lifting too high? Prop drilling and re-rendering more of the tree than needed.

**Q9. What are synthetic events, and how do React events differ from native DOM events?**
React passes a `SyntheticEvent` that wraps the native event with a consistent cross-browser API. Handlers are camelCase props that take functions. React delegates listeners at the root container (since React 17; earlier versions used `document`), dropped event pooling in 17, and its `onChange` fires on every keystroke. Events bubble through the React tree, even across portals.
*Follow-up:* How do you reach the native event? `e.nativeEvent`.

**Q10. Why must component names start with a capital letter?**
JSX compiles lowercase tags to strings (`"div"`), which React treats as DOM elements, and capitalized tags to variable references. A lowercase `<button>` would never call your `button` function.
*Follow-up:* How do you render a component chosen at runtime? Put it in a capitalized variable first: `const Icon = icons[name]; return <Icon />;`.

**Q11. What does StrictMode do, and why does my effect run twice in development?**
In development only, StrictMode renders components twice, mounts-unmounts-remounts each component once (so effects run setup, cleanup, setup) and warns about deprecated APIs. It surfaces impure renders and missing cleanups before they become production bugs. The fix is a correct cleanup function, not removing StrictMode.
*Follow-up:* Does it slow down production? No, production builds skip all of it.

**Q12. Function components or class components?**
Function components with hooks are the standard: less code, logic grouped by concern instead of by lifecycle method, and reuse through custom hooks. Classes still work and are required for error boundaries. You should be able to read lifecycle code and translate it: `componentDidMount` plus `componentWillUnmount` become one effect with a cleanup.
*Follow-up:* Why do error boundaries still need a class? There is no hook equivalent of `getDerivedStateFromError` and `componentDidCatch`.

**Q13. What is wrong with `{items.length && <List items={items} />}`?**
When the array is empty, `items.length` is `0`, and React renders the number `0`. In React Native it crashes, because raw text must be inside `<Text>`. Write `items.length > 0 && ...` or use a ternary.
*Follow-up:* Which values render nothing? `null`, `undefined`, `true` and `false`.

**Q14. Why must rendering be pure?**
React may call a component several times, in a different order than you expect, and may discard a render entirely (StrictMode, concurrent rendering, Suspense). Side effects during render, such as mutating outer variables, fetching or writing to `localStorage`, would run unpredictably. Side effects belong in event handlers, or in effects when they synchronize with an external system.
*Follow-up:* Is mutating an array created in the same render impure? No, local mutation of values created during that render is fine.

**Q15. How do you reset a component's state when a prop changes?**
Give it a `key` tied to that prop: `<EditBillForm key={billId} billId={billId} />`. A new key is a new identity, so React unmounts the old instance and mounts a fresh one with initial state. That is simpler and more correct than an effect that resets state after the stale render.
*Follow-up:* What if only part of the state should reset? Store less state and derive the rest from props.

**Q16. What does composition over inheritance mean in React?**
Components reuse each other by nesting and by passing elements as props (`children`, `header`, `actions`), not by extending component classes. A `Card` takes `title` and `children`; a `DangerButton` wraps `Button` with fixed props. The React team has said it has not found use cases for component inheritance hierarchies.
*Follow-up:* What is a slot prop? A prop that receives JSX, such as `<Layout sidebar={<Nav />}>`.

**Q17. When do you need `<Fragment>` instead of `<>`?**
When it needs a key, for example when each list item renders a pair of siblings: `<Fragment key={row.id}><dt /><dd /></Fragment>`. The short syntax cannot take attributes.
*Follow-up:* Why use fragments at all? To return siblings without wrapper divs that break table, flex or grid layouts.

**Q18. How would you start a new React project today?**
For a single-page app, Vite with the `react-ts` template. When you need routing, data loading or SSR, a framework: Next.js, React Router framework mode or TanStack Start, and Expo for mobile. Create React App is deprecated. In an existing Rails monolith, React on Rails (or a Vite-based Rails integration) mounts components into views.
*Follow-up:* Why not Create React App? It is deprecated and unmaintained, and Vite is much faster.

## Further reading
- [React docs: Learn](https://react.dev/learn)
- [Preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state)
- [StrictMode reference](https://react.dev/reference/react/StrictMode)
