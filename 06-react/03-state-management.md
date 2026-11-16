# State Management in React

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [Data fetching](04-data-fetching-tanstack-query.md), [Forms](05-forms-and-validation.md), [Performance](06-performance.md), [Routing](08-routing.md), [RN state and storage](../08-react-native-expo/04-state-storage-auth-offline.md)
> **On your resume:** Deciding where state lives is part of every feature you build at Devntech (the Colochousing web app, the Coloc resident app, the multi-tenant TanStack Start platform with org-scoped access) and of the role-based dashboards in the PureLogics LMS.

## Quick revision
- Classify state before picking a tool: local UI, shared client, server cache, URL, form.
- Most "global state" problems are server-cache problems: fetched data belongs in TanStack Query (or RTK Query), not in a hand-synced store.
- Keep state as close as possible to where it is used; lift it only as far as the closest common parent.
- Prop drilling means passing props through components that only forward them. Fix it in order: composition (pass elements as children or slots), context, a store.
- Context is dependency injection, not a state manager: every consumer re-renders when the value changes, and there are no selectors.
- Context hygiene: a custom hook that throws outside the provider, `useMemo` on the value, separate contexts for state and dispatch or for unrelated concerns, low-frequency data only.
- Redux Toolkit: `configureStore`, `createSlice` (Immer lets reducers "mutate" a draft), `createAsyncThunk` (pending, fulfilled, rejected), `createSelector` for memoized derived data, typed hooks.
- `useSelector` re-renders when the selected value changes by reference: select the smallest piece and memoize derived arrays.
- Redux pays off for large shared client state with many writers, strict conventions across a big team, middleware needs, or an existing Redux codebase.
- Zustand: `create()` returns a hook, no provider, subscribe through selectors, `useShallow` for multi-field picks, `persist` middleware for storage, `getState()` outside React.
- Jotai: atoms (`atom(0)`), derived atoms, `useAtom`; components subscribe to exactly the atoms they read.
- URL state (filters, tabs, page, search term) makes views shareable and back-button friendly: `useSearchParams`.
- Form state stays in the form (React Hook Form), not in a global store.

## 1. Kinds of state

| Kind | Examples | Where it should live | Typical tool |
|---|---|---|---|
| Local UI | open or closed, hovered row, input draft | the component | `useState`, `useReducer` |
| Shared client | auth user, theme, cart, current organization, wizard steps | nearest common parent or a store | lifting, Context, Zustand, Redux |
| Server cache | bills, residents, courses, products | a cache keyed by request | TanStack Query, RTK Query, router loaders |
| URL | filters, sort, page, selected tab, search term | the address bar | `useSearchParams`, typed router search params |
| Form | field values, touched, errors, submitting | the form | React Hook Form, local state |

Server state differs from client state: the backend owns it, it can change without you knowing, it goes stale, and it needs loading, error, refetch and invalidation logic. Copying fetched data into Redux and syncing it by hand is the most common state-management mistake.

## 2. Lifting state and prop drilling

Start local. When two components need the same state, lift it to their closest common parent (see [fundamentals](01-fundamentals.md)). When that parent is far up, intermediate components start forwarding props they never use.

```tsx
// Drilling: Layout and Header only forward `user`
function App({ user }: { user: User }) {
  return <Layout user={user} />;
}
function Layout({ user }: { user: User }) {
  return <Header user={user} />;
}
function Header({ user }: { user: User }) {
  return <UserMenu user={user} />;
}
```

```tsx
// Composition fix: build the element where the data is and pass it down as a slot
function App({ user }: { user: User }) {
  return <Layout header={<Header menu={<UserMenu user={user} />} />} />;
}
function Layout({ header }: { header: ReactNode }) {
  return <div className="layout">{header}</div>;
}
function Header({ menu }: { menu: ReactNode }) {
  return <header>{menu}</header>;
}
```

Composition fixes many drilling cases with zero new concepts. Reach for context or a store when many distant components need the same data.

## 3. Context API

Context makes a value available to a whole subtree without passing props. A production-quality setup:

```tsx
import { createContext, useCallback, useContext, useMemo, useState, type ReactNode } from 'react';

type AuthContextValue = {
  user: User | null;
  login: (user: User) => void;
  logout: () => void;
};

const AuthContext = createContext<AuthContextValue | null>(null); // null means "no provider"

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const login = useCallback((u: User) => setUser(u), []);
  const logout = useCallback(() => setUser(null), []);

  // Without useMemo, every render of AuthProvider creates a new object
  // and re-renders every consumer, even when user did not change
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used inside <AuthProvider>');
  return ctx; // typed as AuthContextValue, so consumers need no null checks
}
```

**Re-render pitfalls:**
- Every consumer re-renders whenever the value changes by `Object.is`, and `memo` does not stop it.
- A consumer that only needs `logout` still re-renders when `user` changes: context has no selectors.
- A new object literal as `value` re-renders all consumers on every provider render.

**Fixes:** memoize the value; split contexts by concern (auth, theme, feature flags) and by update frequency; separate state from dispatch so components that only dispatch never re-render:

```tsx
const CartStateContext = createContext<CartItem[] | null>(null);
const CartDispatchContext = createContext<Dispatch<CartAction> | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const [items, dispatch] = useReducer(cartReducer, []);
  return (
    <CartDispatchContext.Provider value={dispatch}> {/* dispatch is stable */}
      <CartStateContext.Provider value={items}>{children}</CartStateContext.Provider>
    </CartDispatchContext.Provider>
  );
}
// An AddToCartButton reads only CartDispatchContext, so cart changes never re-render it
```

**Good fits:** theme, locale, the authenticated user, feature flags, the current organization or tenant, dependency injection (an API client, a QueryClient). **Poor fits:** values that change many times per second (pointer position, keystrokes, timers) or large collections read by many components; use a store with selectors there. React 19 also accepts `<AuthContext value={value}>` as the provider and `use(AuthContext)` to read it (see [React 19](09-react-19-and-server-components.md)).

## 4. Redux Toolkit

Redux keeps app state in one store. Components dispatch actions describing what happened, pure reducers compute the next state, and components read slices of it with selectors. Redux Toolkit (RTK) is the official way to write Redux today: less boilerplate, Immer built in, good defaults.

```ts
// features/todos/todosSlice.ts
import { createAsyncThunk, createSlice, type PayloadAction } from '@reduxjs/toolkit';

export type Todo = { id: string; title: string; done: boolean };
type TodosState = { items: Todo[]; status: 'idle' | 'loading' | 'failed' };

export const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const res = await fetch('/api/todos');
  if (!res.ok) throw new Error('Failed to load todos');
  return (await res.json()) as Todo[];
}); // dispatches todos/fetch/pending, then todos/fetch/fulfilled or todos/fetch/rejected

const initialState: TodosState = { items: [], status: 'idle' };

const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    todoAdded(state, action: PayloadAction<Todo>) {
      state.items.push(action.payload); // Immer turns this "mutation" into an immutable update
    },
    todoToggled(state, action: PayloadAction<string>) {
      const todo = state.items.find((t) => t.id === action.payload);
      if (todo) todo.done = !todo.done;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = 'idle';
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state) => {
        state.status = 'failed';
      });
  },
});

export const { todoAdded, todoToggled } = todosSlice.actions;
export default todosSlice.reducer;
```

```ts
// app/store.ts
import { configureStore } from '@reduxjs/toolkit';
import { useDispatch, useSelector } from 'react-redux';
import todosReducer from '../features/todos/todosSlice';

export const store = configureStore({ reducer: { todos: todosReducer } }); // thunk + devtools by default

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
export const useAppDispatch = useDispatch.withTypes<AppDispatch>(); // react-redux 9.1+
export const useAppSelector = useSelector.withTypes<RootState>();
```

```tsx
// main.tsx wraps the app: <Provider store={store}><App /></Provider>
function TodoList() {
  const todos = useAppSelector((s) => s.todos.items);
  const dispatch = useAppDispatch();

  useEffect(() => {
    dispatch(fetchTodos());
  }, [dispatch]);

  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id} onClick={() => dispatch(todoToggled(t.id))}>{t.title}</li>
      ))}
    </ul>
  );
}
```

**Selectors.** `useSelector` runs after every dispatch and re-renders the component when the returned value changes by reference. A selector that filters creates a new array each time, so the component re-renders on every action. Memoize derived data with `createSelector` (re-exported from Reselect):

```ts
import { createSelector } from '@reduxjs/toolkit';

const selectTodos = (s: RootState) => s.todos.items;
export const selectOpenTodos = createSelector([selectTodos], (todos) => todos.filter((t) => !t.done));
```

**Immer rules.** In a slice reducer, either mutate the draft or return a brand-new value, never both. Never mutate store state outside reducers.

**RTK Query** is RTK's data-fetching and caching layer, the Redux-native alternative to TanStack Query:

```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Todo'],
  endpoints: (build) => ({
    getTodos: build.query<Todo[], void>({ query: () => 'todos', providesTags: ['Todo'] }),
    addTodo: build.mutation<Todo, { title: string }>({
      query: (body) => ({ url: 'todos', method: 'POST', body }),
      invalidatesTags: ['Todo'], // refetches getTodos after a successful add
    }),
  }),
});
export const { useGetTodosQuery, useAddTodoMutation } = api;
// Register api.reducer under api.reducerPath and add api.middleware in configureStore
```

**When Redux is worth it:** lots of client state shared by distant components and updated from many places; complex flows that benefit from explicit actions, middleware and time-travel debugging; large teams that want one enforced pattern; an existing Redux codebase. It is overkill when your "global state" is mostly server data plus a theme and a user.

## 5. Zustand

Zustand is a small store library: `create` returns a hook, there is no provider, and components subscribe with selectors so they re-render only when the selected value changes.

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type CartItem = { id: string; name: string; qty: number };
type CartState = {
  items: CartItem[];
  add: (item: CartItem) => void;
  remove: (id: string) => void;
  clear: () => void;
};

export const useCartStore = create<CartState>()(
  persist(
    (set) => ({
      items: [],
      add: (item) => set((s) => ({ items: [...s.items, item] })), // set merges at the top level
      remove: (id) => set((s) => ({ items: s.items.filter((i) => i.id !== id) })),
      clear: () => set({ items: [] }),
    }),
    { name: 'cart' }, // storage key; localStorage by default
  ),
);
```

```tsx
import { useShallow } from 'zustand/react/shallow';

function CartBadge() {
  const count = useCartStore((s) => s.items.length); // re-renders only when the count changes
  return <span>{count}</span>;
}

function ClearCartButton() {
  // Picking several fields returns a new object each time; useShallow compares it field by field
  const { itemCount, clear } = useCartStore(useShallow((s) => ({ itemCount: s.items.length, clear: s.clear })));
  return <button onClick={clear} disabled={itemCount === 0}>Clear cart</button>;
}

// Outside React (an API interceptor on 401, a logout helper):
useCartStore.getState().clear();
```

- Select the smallest value you need. Calling `useCartStore()` without a selector re-renders on every change, and in Zustand v5 a selector that returns a new object or array on every call can cause an infinite render loop: use `useShallow` or select primitives.
- `persist` options include `storage` (`createJSONStorage(() => sessionStorage)`, or AsyncStorage in React Native), `partialize` (persist only some fields) and `version` plus `migrate` for shape changes. Persisted state loads on the client, so SSR apps must handle the first render before rehydration.
- Other middleware: `devtools` (from `zustand/middleware`) and `immer` (from `zustand/middleware/immer`).

## 6. Jotai (briefly)

Jotai builds state bottom-up from atoms. Each component subscribes to exactly the atoms it reads, and derived atoms recompute automatically.

```tsx
import { atom, useAtom, useAtomValue } from 'jotai';

const countAtom = atom(0);
const doubledAtom = atom((get) => get(countAtom) * 2); // derived, read-only

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledAtom);
  return <button onClick={() => setCount((c) => c + 1)}>{count} (x2 = {doubled})</button>;
}
```

It suits many small independent pieces of state (editor settings, filter panels) without one big store. It works without a provider (a default store) and supports scoped `<Provider>`s; `jotai/utils` adds helpers such as `atomWithStorage`.

## 7. Comparison

| | Lifting / `useState` | Context | Redux Toolkit | Zustand | Jotai | TanStack Query |
|---|---|---|---|---|---|---|
| Best for | local and nearby state | low-frequency app-wide values | large, complex shared client state | small to medium shared client state | many independent atoms | server state |
| Re-render control | component tree | all consumers on any change | selectors | selectors (+ `useShallow`) | per atom | per query (+ `select`) |
| Boilerplate | none | low | medium (far less than classic Redux) | very low | very low | low |
| Provider needed | no | yes | yes | no | optional | yes (`QueryClientProvider`) |
| Devtools | React DevTools | React DevTools | excellent (time travel) | via middleware | community tools | dedicated devtools |
| Access outside React | no | no | `store.getState()` | `useStore.getState()` | store API | `queryClient` methods |

### Going deeper: why stores use useSyncExternalStore

With concurrent rendering React may pause a render halfway. If components read a mutable external store directly, the ones rendered before and after the pause can show different values from the same store ("tearing"). `useSyncExternalStore` lets React notice that the snapshot changed during the render and re-render synchronously, which is why modern React Redux, Zustand and similar libraries are built on it. Context does not have this problem because its value flows through React itself.

## 8. Decision guide

1. **Does it come from the server?** TanStack Query (or RTK Query in a Redux app, or framework loaders). Do not copy it into a store.
2. **Should it survive a reload or be shareable by link?** Put it in the URL.
3. **Is it form input?** Keep it in the form (React Hook Form or local state).
4. **Used by one component or a small subtree?** `useState` or `useReducer`, lifted only as far as needed.
5. **Rarely changing and app-wide (theme, locale, auth user, current org)?** Context with a custom hook.
6. **Frequently changing client state shared across distant components?** Zustand for most apps; Redux Toolkit for large apps, many writers, strict conventions or an existing Redux codebase; Jotai for many independent atoms.

## 9. Server state belongs in TanStack Query

The anti-pattern: an effect fetches bills and dispatches them into Redux, and every mutation must remember to patch or refetch the store. You end up re-implementing caching, deduplication, loading and error flags, retries, background refresh and invalidation by hand, usually with bugs.

TanStack Query treats fetched data as a cache keyed by query key. Components declare what they need (`useQuery({ queryKey: ['bills', houseId], queryFn })`), duplicate requests collapse into one, stale data refreshes in the background, and mutations invalidate exactly the affected keys. Once server state moves there, the remaining client state is usually small enough for `useState`, context or a tiny Zustand store. Details in [data fetching with TanStack Query](04-data-fetching-tanstack-query.md).

## 10. URL state with search params

If a user would expect a link or the back button to restore a view, the state belongs in the URL: filters, sorting, pagination, selected tab, search term, open item id.

```tsx
import { useSearchParams } from 'react-router';

function BillFilters() {
  const [searchParams, setSearchParams] = useSearchParams();
  const status = searchParams.get('status') ?? 'all';
  const page = Number(searchParams.get('page') ?? '1');

  function setStatus(next: string) {
    setSearchParams((prev) => {
      const params = new URLSearchParams(prev);
      params.set('status', next);
      params.set('page', '1'); // reset pagination when the filter changes
      return params;
    });
  }

  return (
    <>
      <select value={status} onChange={(e) => setStatus(e.target.value)}>
        <option value="all">All</option>
        <option value="unpaid">Unpaid</option>
        <option value="paid">Paid</option>
      </select>
      <span>Page {page}</span>
    </>
  );
}
```

- Search params are strings: parse and validate them (numbers, enums) and fall back to defaults. TanStack Router validates and types them with a schema ([TanStack Router](../07-nextjs-and-tanstack/04-tanstack-router.md)); Next.js exposes `useSearchParams` from `next/navigation`.
- For keystroke-level updates (a search box), replace the history entry (`setSearchParams(next, { replace: true })`) and debounce, so the back button is not flooded.
- Include the URL params in your query keys so cached results always match the view.

## Common pitfalls
- Copying server data into Redux, Zustand or `useState` and syncing it by hand.
- One giant context holding everything, so any change re-renders the whole app.
- Passing `value={{ user, logout }}` without `useMemo` from a provider that re-renders often.
- Using context for high-frequency updates (pointer position, typing).
- Selecting too much from a store (the whole store or slice), or returning new arrays from selectors without memoization.
- Mutating Redux state outside a reducer, or both mutating the draft and returning a value in an Immer reducer.
- Keeping filters and pagination in component state, so refreshes and shared links lose the view.
- Putting form fields in a global store.
- Persisting sensitive data (tokens, personal data) to `localStorage` via `persist` without considering XSS exposure.

## How to talk about this in an interview
- Open with the classification: "I first ask what kind of state it is: server cache, URL, form, or real client state. Server data goes into TanStack Query, view state into the URL, and only what is left goes into local state, context or a small store." The question tests judgment more than library trivia.
- Connect it to your platforms: a multi-tenant app has a "current organization" that many components need and that rarely changes, a classic context case, while the org-scoped data itself is server state keyed by organization.

> **Fill in:** how the TanStack Start platform (or the Colochousing React code) shares the current user or organization across components, and which tool holds it.

> **Fill in:** in the Coloc resident app, which state was global (session, selected house) and which stayed local to a screen, and whether anything was persisted on the device.

- PureLogics LMS: role-based dashboards depend on the logged-in user's role, a good example of auth state in context driving conditional rendering.
- If you have not used Redux in production, say so plainly and explain when you would choose it; interviewers value the reasoning more than the library.

## Interview questions

**Q1. What kinds of state exist in a React app, and where should each live?**
Local UI state in the component, shared client state lifted or in context or a store, server state in a cache such as TanStack Query, URL state in search params, and form state in React Hook Form or local state. Server state is special because the backend owns it and it goes stale, so it needs caching and invalidation rather than a hand-written store.
*Follow-up:* Most common mistake? Treating server data as client state and syncing it manually.

**Q2. What is prop drilling, and how do you avoid it?**
Passing props through intermediate components that only forward them. First try composition: build the element where the data lives and pass it down as `children` or a slot prop. If many distant components need the value, use context; if it changes often or is complex, use a store with selectors.
*Follow-up:* Is drilling always bad? No, two or three explicit levels are easy to follow and to refactor.

**Q3. Lifting state up vs Context vs Redux or Zustand: how do you decide?**
Lift when a few nearby components share state. Use context for rarely changing values needed across the tree (theme, user, locale, current org). Use a store when shared client state changes often or is updated from many places and components need fine-grained subscriptions: Zustand for simplicity, Redux Toolkit for large apps with strict conventions. Server data goes to TanStack Query in every case.
*Follow-up:* Can you combine them? Yes, a typical app uses several, each for a different kind of state.

**Q4. What are the performance pitfalls of the Context API, and how do you fix them?**
All consumers re-render whenever the value changes, there are no selectors, and `memo` does not block context updates. A new object literal as the value re-renders every consumer on each provider render. Fix it by memoizing the value, splitting contexts by concern and update frequency, separating state from dispatch, and moving high-frequency state into a store.
*Follow-up:* Why does `memo` not help? Context changes reach consumers directly, regardless of props.

**Q5. Is Context a state management tool?**
Not by itself: it transports a value through the tree (dependency injection). The state still lives in `useState` or `useReducer` inside the provider. Context plus a reducer works well for small shared state, but it lacks selectors, middleware and devtools.
*Follow-up:* What does a store library add? Subscriptions to slices, so only affected components re-render.

**Q6. Explain the Redux data flow. What does Redux Toolkit add?**
The UI dispatches an action, the store runs reducers that compute a new immutable state, and components whose selected values changed re-render. RTK adds `configureStore` (thunk and devtools by default), `createSlice` (actions and reducers together with Immer), `createAsyncThunk`, `createSelector` and RTK Query.
*Follow-up:* Why must reducers be pure? Predictable state, time-travel debugging and easy tests.

**Q7. How does Immer let you write mutating code in reducers?**
Immer gives the reducer a draft proxy. You "mutate" the draft, Immer records the changes and produces a new immutable object that shares unchanged parts with the old state. Either mutate the draft or return a new value, not both.
*Follow-up:* Is the result truly immutable? Immer freezes produced state by default, and RTK adds a development-only check that catches accidental mutations.

**Q8. What is createAsyncThunk, and what actions does it dispatch?**
It wraps an async function into a thunk action creator. When dispatched it emits `type/pending`, then `type/fulfilled` with the returned value or `type/rejected` with the error; slices handle them in `extraReducers`. It suits occasional workflows; for server data, RTK Query or TanStack Query save you from writing loading flags by hand.
*Follow-up:* How do you get the result in a component? `await dispatch(fetchTodos()).unwrap()`, which throws on rejection.

**Q9. Why use memoized selectors such as createSelector?**
`useSelector` re-renders the component whenever the returned value changes by reference. Selectors that filter or map create a new array every time, so the component re-renders on every dispatched action. `createSelector` recomputes only when its inputs change and otherwise returns the same reference.
*Follow-up:* What if you select a primitive? Primitives compare by value, so no memoization is needed.

**Q10. Redux Toolkit vs Zustand?**
Both are external stores with selector-based subscriptions. Zustand is tiny, needs no provider or action boilerplate, and is a great default for small to medium client state. Redux Toolkit adds explicit actions, middleware, excellent devtools and strong conventions that pay off in large apps and teams. Neither should hold server data.
*Follow-up:* What do they share under the hood? Modern versions of both subscribe through `useSyncExternalStore`.

**Q11. How does Zustand avoid unnecessary re-renders, and what is useShallow for?**
Each component passes a selector, and Zustand re-renders it only when the selected value changes by `Object.is`. Selecting several fields as an object creates a new object on every call, which would re-render each time (or loop, in v5); `useShallow` compares that object's fields shallowly instead.
*Follow-up:* How do you read state outside a component? `useStore.getState()`, and `setState` or store actions to write.

**Q12. Why shouldn't server data live in Redux or useState?**
You would rebuild caching, request deduplication, loading and error flags, retries, background refresh, pagination and invalidation by hand, and the copy goes stale silently. A server-state library keys data by request, refetches when stale and lets mutations invalidate exactly the affected keys.
*Follow-up:* And if the app already uses Redux? Use RTK Query for server data.

**Q13. When would you put state in the URL?**
When users should be able to refresh, bookmark, share or press back and get the same view: filters, sort order, pagination, tabs, search terms, the selected item. It also lets the server see the state for SSR. Parse and validate params, because they are untrusted strings.
*Follow-up:* What stays out of the URL? Secrets, large data and fast-changing values such as an unsubmitted draft.

**Q14. How would you persist client state across reloads?**
For small, non-sensitive state, `localStorage` through a custom hook or Zustand's `persist` middleware (AsyncStorage or secure storage in React Native). Version the stored shape and migrate it, persist only what you need (`partialize`), and handle the first render before rehydration in SSR apps. Tokens belong in httpOnly cookies on the web and secure storage on mobile.
*Follow-up:* Why not persist everything? Stale or corrupted data after deploys, and privacy risk on shared devices.

**Q15. What is Jotai's atomic model?**
State is split into atoms, small independent units created with `atom()`. Components read and write atoms with `useAtom`, derived atoms compute from other atoms, and only components that use a changed atom re-render. It is bottom-up, unlike Redux's single top-down store.
*Follow-up:* When does it shine? Many independent pieces of state with derived values between them.

**Q16. How would you share the logged-in user or current organization across the app?**
An `AuthProvider` near the root holds the user (loaded from a `/me` endpoint, ideally through TanStack Query), exposes it with a memoized value, and offers a `useAuth()` hook that throws outside the provider. For the current organization, a similar context or a URL segment (`/orgs/:orgId/...`) keeps links and refreshes org-scoped. The server still enforces authorization.
*Follow-up:* Why put the org in the URL? Links keep the right tenant, and query keys can include it.

## Further reading
- [Scaling up with reducer and context](https://react.dev/learn/scaling-up-with-reducer-and-context)
- [Redux Toolkit docs](https://redux-toolkit.js.org/)
- [Zustand docs](https://zustand.docs.pmnd.rs/)
- [Jotai docs](https://jotai.org/)
