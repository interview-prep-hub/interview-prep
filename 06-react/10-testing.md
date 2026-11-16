# Testing React Applications

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [Data fetching](04-data-fetching-tanstack-query.md), [Forms](05-forms-and-validation.md), [Routing](08-routing.md), [Testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md), [RSpec](../05-ruby-on-rails/13-testing-rspec.md), [RN testing](../08-react-native-expo/09-native-modules-permissions-testing.md)
> **On your resume:** Testing is part of shipping features at Devntech across the React on Rails code, the TanStack Start platform and the Coloc resident app (React Native projects usually test with Jest and the `jest-expo` preset), and it matters most for flows such as rent payments and split bills.

## Quick revision
- Static checks (TypeScript, ESLint) catch a whole class of bugs for free; then unit tests for pure logic, integration tests for components with a mocked network (the most value per test), and a few end-to-end tests for critical flows.
- Vitest is the natural runner for Vite projects (Jest-compatible API, `vi` instead of `jest`); Jest remains common in older projects and in React Native.
- React Testing Library (RTL): render, interact and assert the way a user would; avoid implementation details such as state, internal methods and CSS classes.
- Query priority: `getByRole` (with `name`) > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByDisplayValue` > `getByAltText`/`getByTitle` > `getByTestId`.
- `getBy*` throws when nothing matches, `queryBy*` returns `null` (assert absence), `findBy*` returns a promise and retries until it matches (async UI).
- `const user = userEvent.setup()` then `await user.click(...)` / `await user.type(...)`: realistic event sequences, preferred over `fireEvent`.
- `renderHook` tests custom hooks; wrap manual state updates in `act`.
- Mock modules with `vi.mock`; mock the network with MSW (`http.get(url, () => HttpResponse.json(data))`), so components run their real fetching code.
- TanStack Query in tests: a new `QueryClient` per test with `retry: false`, wrapped in `QueryClientProvider`.
- Snapshots are cheap but brittle; prefer explicit assertions.
- End-to-end: Playwright (multi-browser, auto-waiting, traces) or Cypress; for React Native, Maestro or Detox.
- An "update was not wrapped in act(...)" warning means an async update landed after your assertions: await `findBy*` or `waitFor`.

## 1. What to test: the pyramid and the trophy

| Layer | What it covers | Tools | Volume |
|---|---|---|---|
| Static | types, lint rules, obvious mistakes | TypeScript, ESLint | everywhere |
| Unit | pure functions, reducers, hooks, small components | Vitest or Jest | many, fast |
| Integration | a component tree with real children, routing and a mocked network | RTL + MSW | the bulk of UI tests |
| End-to-end | real browser, real backend (or a seeded test one), critical journeys | Playwright, Cypress | few, slow, high confidence |

The classic pyramid puts unit tests at the base. For frontends, many teams follow the "testing trophy" idea: integration tests give the best confidence per minute, because they exercise components the way users do without the cost of a full browser and backend. Test behavior ("clicking Pay marks the bill as paid"), not implementation ("setPaid was called with true").

## 2. Tooling: Jest vs Vitest

| | Vitest | Jest |
|---|---|---|
| Best fit | Vite projects (reuses the Vite config and transforms) | existing Jest codebases, React Native (`jest-expo`) |
| API | Jest-compatible (`describe`, `it`, `expect`), mocks via `vi` | `jest.fn`, `jest.mock` |
| ES modules and TypeScript | native through Vite | needs a transform (Babel, ts-jest or SWC) |
| Speed and watch mode | very fast, smart re-runs | good, slower on large TypeScript codebases |
| DOM environment | `jsdom` or `happy-dom` | `jest-environment-jsdom` |

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/dom @testing-library/user-event @testing-library/jest-dom msw
```

```ts
// vite.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',               // a simulated DOM in Node
    globals: true,                      // describe/it/expect without imports; also enables RTL auto-cleanup
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'; // toBeInTheDocument, toBeDisabled, toHaveValue, ...
import { afterAll, afterEach, beforeAll } from 'vitest';
import { server } from './server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' })); // unknown requests fail loudly
afterEach(() => server.resetHandlers());                         // drop per-test overrides
afterAll(() => server.close());
```

## 3. React Testing Library: philosophy and queries

RTL renders components into a real (simulated) DOM and gives you queries that find elements the way users and assistive technology do: by role, label and visible text. If a test cannot find a button by its accessible name, a screen reader user probably cannot either. Tests written this way survive refactors, because they do not know about state variables, hooks or class names.

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

test('increments the count', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

| Priority | Query | Use for |
|---|---|---|
| 1 | `getByRole('button', { name: /save/i })` | almost everything: buttons, links, headings, textboxes, checkboxes, dialogs |
| 2 | `getByLabelText(/email/i)` | form fields |
| 3 | `getByPlaceholderText` | only when there is no label |
| 4 | `getByText` | non-interactive text: paragraphs, list items |
| 5 | `getByDisplayValue` | inputs with a current value |
| 6 | `getByAltText`, `getByTitle` | images, titled elements |
| 7 | `getByTestId` | last resort, when nothing user-visible identifies the element |

| Variant | No match | Many matches | Async | Use for |
|---|---|---|---|---|
| `getBy*` | throws | throws | no | elements that should be there now |
| `queryBy*` | returns `null` | throws | no | asserting absence: `expect(screen.queryByRole('alert')).not.toBeInTheDocument()` |
| `findBy*` | rejects after timeout | rejects | yes (retries, about 1 s by default) | elements that appear after async work |
| `*AllBy*` | same rules | returns an array | per variant | lists |

`within(row).getByRole('button', { name: /pay/i })` scopes a query to part of the screen. Avoid `container.querySelector`, asserting on state or props, and wrapping everything in `act` yourself: `render` and user-event already handle that.

## 4. user-event vs fireEvent

`fireEvent.click(button)` dispatches one DOM event. `user.click(button)` simulates what a real user does: pointer events, focus, `mousedown`, `mouseup`, `click`; `user.type(input, 'abc')` fires keydown, keypress, input and keyup for each character and respects disabled and read-only fields. That realism catches bugs `fireEvent` misses.

```tsx
const user = userEvent.setup(); // call before render
await user.type(screen.getByLabelText(/search/i), 'lahore');
await user.keyboard('{Enter}');
await user.selectOptions(screen.getByLabelText(/status/i), 'paid');
await user.upload(screen.getByLabelText(/receipt/i), new File(['x'], 'receipt.png', { type: 'image/png' }));
```

All user-event v14 methods are async: always `await` them.

## 5. Example: testing a form

```tsx
// LoginForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email('Enter a valid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});
type LoginValues = z.infer<typeof loginSchema>;

export function LoginForm({ onSubmit }: { onSubmit: (values: LoginValues) => void | Promise<void> }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<LoginValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  return (
    // handleSubmit calls its callback with (values, event); forward only the values
    <form onSubmit={handleSubmit((values) => onSubmit(values))} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" {...register('email')} />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <label htmlFor="password">Password</label>
      <input id="password" type="password" {...register('password')} />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <button type="submit" disabled={isSubmitting}>Sign in</button>
    </form>
  );
}
```

```tsx
// LoginForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('shows validation errors and does not submit invalid data', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(await screen.findByText(/enter a valid email/i)).toBeInTheDocument(); // validation is async
    expect(screen.getByText(/at least 8 characters/i)).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('submits the typed values', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'ada@example.com');
    await user.type(screen.getByLabelText(/password/i), 'correct-horse');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() =>
      expect(onSubmit).toHaveBeenCalledWith({ email: 'ada@example.com', password: 'correct-horse' }),
    );
  });
});
```

The test never mentions React Hook Form or Zod; you could swap the form library and it would still pass. If `LoginForm` passed `onSubmit` straight to `handleSubmit`, the mock would also receive the event object and `toHaveBeenCalledWith` with one argument would fail, a small example of why component contracts matter.

## 6. Mocking the network with MSW

Mock Service Worker intercepts requests at the network level (a service worker in the browser, request interception in Node). Your components keep their real `fetch` or axios code, query keys and error handling, and the same handlers can power local development and Storybook.

```ts
// src/lib/api.ts: an absolute base URL works in the browser, in Node tests and in React Native
export const API_URL = import.meta.env.VITE_API_URL ?? 'http://localhost:3000/api';

export async function fetchJson<T>(path: string, signal?: AbortSignal): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}
```

```ts
// src/test/server.ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { API_URL } from '../lib/api';

export const handlers = [
  http.get(`${API_URL}/bills`, () =>
    HttpResponse.json([
      { id: '1', title: 'Internet', amountCents: 3000, paid: false },
      { id: '2', title: 'Electricity', amountCents: 5400, paid: true },
    ]),
  ),
];

export const server = setupServer(...handlers);
```

Per-test overrides use `server.use(...)` and are reset after each test by the setup file. MSW v2 uses `http` and `HttpResponse`; older examples with `rest` and `res(ctx.json())` are v1.

## 7. Example: testing a component that fetches with TanStack Query

```tsx
// BillList.tsx
import { useQuery } from '@tanstack/react-query';
import { fetchJson } from '../lib/api';

export function BillList() {
  const { data, isPending, isError } = useQuery({
    queryKey: ['bills'],
    queryFn: ({ signal }) => fetchJson<Bill[]>('/bills', signal),
  });

  if (isPending) return <p>Loading bills...</p>;
  if (isError) return <p role="alert">Could not load bills</p>;
  return (
    <ul>
      {data.map((b) => (
        <li key={b.id}>{b.title}{b.paid ? ' (paid)' : ''}</li>
      ))}
    </ul>
  );
}
```

```tsx
// BillList.test.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render, screen } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import type { ReactElement } from 'react';
import { API_URL } from '../lib/api';
import { server } from '../test/server';
import { BillList } from './BillList';

function renderWithClient(ui: ReactElement) {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false } }, // fail fast instead of retrying 3 times with backoff
  });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

test('renders bills from the API', async () => {
  renderWithClient(<BillList />);
  expect(screen.getByText(/loading bills/i)).toBeInTheDocument();
  expect(await screen.findByText('Internet')).toBeInTheDocument();
  expect(screen.getByText(/electricity \(paid\)/i)).toBeInTheDocument();
});

test('shows an error when the API fails', async () => {
  server.use(http.get(`${API_URL}/bills`, () => new HttpResponse(null, { status: 500 })));
  renderWithClient(<BillList />);
  expect(await screen.findByRole('alert')).toHaveTextContent(/could not load bills/i);
});
```

Why a new client per test: a shared `QueryClient` carries cached data from one test into the next, which makes tests order-dependent. Why `retry: false`: the default three retries with exponential backoff would make error tests slow or time out.

## 8. Mocking modules, timers and browser APIs

```ts
import { beforeEach, expect, test, vi } from 'vitest';
import { getCurrentUser } from '../api/auth';

vi.mock('../api/auth', () => ({ getCurrentUser: vi.fn() })); // hoisted above the imports

beforeEach(() => {
  vi.mocked(getCurrentUser).mockReset();
});

test('shows a login link to anonymous users', async () => {
  vi.mocked(getCurrentUser).mockResolvedValue(null);
  render(<Header />);
  expect(await screen.findByRole('link', { name: /log in/i })).toBeInTheDocument();
});
```

- **Mock at the boundaries:** network (MSW), time, randomness, browser APIs, native modules. Do not mock your own child components unless they are genuinely heavy (a map, a chart).
- **Spies:** `vi.spyOn(window, 'confirm').mockReturnValue(true)`, restored with `vi.restoreAllMocks()`.
- **Missing browser APIs:** jsdom lacks `IntersectionObserver`, `ResizeObserver` and `matchMedia`; stub them with `vi.stubGlobal('IntersectionObserver', MockObserver)` or a small polyfill in the setup file.
- **Timers:** `vi.useFakeTimers()`, then `vi.advanceTimersByTime(ms)`; with user-event, create it as `userEvent.setup({ advanceTimers: vi.advanceTimersByTime })` so its internal delays use the fake clock. Restore with `vi.useRealTimers()`.
- **Jest equivalents:** `jest.mock`, `jest.fn`, `jest.spyOn`, `jest.useFakeTimers`.

## 9. Testing custom hooks with renderHook

```tsx
import { act, renderHook } from '@testing-library/react';
import { afterEach, expect, test, vi } from 'vitest';
import { useDebounce } from './useDebounce';

afterEach(() => vi.useRealTimers());

test('useDebounce updates only after the delay', () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ value }) => useDebounce(value, 300), {
    initialProps: { value: 'a' },
  });

  rerender({ value: 'ab' });
  expect(result.current).toBe('a'); // not yet

  act(() => {
    vi.advanceTimersByTime(300); // the timer callback sets state, so wrap it in act
  });
  expect(result.current).toBe('ab');
});
```

`renderHook` lives in `@testing-library/react` (the separate hooks package is deprecated). For hooks that need providers, pass a `wrapper`: `renderHook(() => useHouseBills('h1'), { wrapper })`. Prefer testing hooks through a component when the hook is only used in one place.

## 10. Providers: router, context and a custom render

```tsx
// src/test/render.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render, type RenderOptions } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import type { ReactElement, ReactNode } from 'react';
import { MemoryRouter } from 'react-router';

export function renderWithProviders(ui: ReactElement, { route = '/', ...options }: RenderOptions & { route?: string } = {}) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={client}>
        <MemoryRouter initialEntries={[route]}>{children}</MemoryRouter>
      </QueryClientProvider>
    );
  }
  return { user: userEvent.setup(), ...render(ui, { wrapper: Wrapper, ...options }) };
}

// const { user } = renderWithProviders(<BillsPage />, { route: '/houses/42/bills' });
```

For routes with loaders and actions, build a memory data router: `const router = createMemoryRouter(routes, { initialEntries: ['/houses/42'] }); render(<RouterProvider router={router} />);`. Add auth, theme or tenant providers to the wrapper the same way, with test values.

## 11. Snapshot tests: pros and cons

| Pros | Cons |
|---|---|
| one line to write | large snapshots nobody reads in review |
| catch unexpected markup changes | break on harmless changes (class names, wrappers) |
| fine for serializers and stable output | tempt people to run `-u` without looking |
| | do not assert behavior or accessibility |

Use them sparingly: small inline snapshots (`toMatchInlineSnapshot`) for stable output such as formatted values or error objects, and explicit assertions (`toHaveTextContent`, `toBeDisabled`, `toHaveBeenCalledWith`) for UI behavior.

## 12. End-to-end tests with Playwright or Cypress

```ts
// e2e/signup.spec.ts
import { expect, test } from '@playwright/test';

test('a visitor can sign up', async ({ page }) => {
  await page.goto('/signup');
  await page.getByLabel('Email').fill('ada@example.com');
  await page.getByLabel('Password', { exact: true }).fill('correct-horse-battery');
  await page.getByRole('button', { name: 'Create account' }).click();
  await expect(page.getByRole('heading', { name: 'Welcome' })).toBeVisible(); // auto-waits
});
```

| | Playwright | Cypress |
|---|---|---|
| Browsers | Chromium, Firefox, WebKit | Chromium-family, Firefox (WebKit experimental) |
| Model | runs outside the browser, async/await API | runs inside the browser, chained commands |
| Strengths | auto-waiting locators, parallelism, trace viewer, codegen, multiple tabs and origins | interactive time-travel debugging, approachable API, `cy.intercept` |

Keep end-to-end suites small: the money paths (sign-up, login, checkout or payment, the core create-and-edit flow). Run them against a seeded test backend in CI. For a Rails app with React on Rails, Rails system tests (Capybara) play the same role ([RSpec](../05-ruby-on-rails/13-testing-rspec.md)); for React Native, Maestro or Detox ([RN testing](../08-react-native-expo/09-native-modules-permissions-testing.md)).

## 13. Debugging tests and fixing flakiness

- `screen.debug()` prints the current DOM; `screen.logTestingPlaygroundURL()` suggests the best query for an element.
- "Unable to find an accessible element with the role..." errors list the roles that do exist: often the label is missing or the name differs.
- **act warnings** ("An update to X inside a test was not wrapped in act(...)") mean state changed after the test stopped waiting, usually a fetch or timer resolving late. Await the final UI with `findBy*` or `waitFor`, or mock the work away.
- **Flaky tests** usually come from unawaited async work, real timers, shared state between tests (a module-level `QueryClient`, `localStorage`, MSW overrides not reset) or reliance on test order.
- In `waitFor`, put only assertions; side effects inside it run many times.

## Common pitfalls
- Testing implementation details: state values, hook calls, class names, component instances.
- Using `getBy*` to assert absence (it throws) instead of `queryBy*`.
- Forgetting to `await` user-event calls or `findBy*` queries.
- Sharing a `QueryClient` or other caches between tests; leaving retries on.
- Mocking `fetch` by hand in every test instead of using MSW handlers.
- Relative URLs in code under test: Node's `fetch` needs absolute URLs, so configure an API base URL.
- Snapshot-only tests for interactive components.
- Huge end-to-end suites that are slow and flaky, or none at all for the payment flow.
- `getByTestId` everywhere, which hides missing labels and roles.
- Not restoring fake timers, spies and stubbed globals after each test.

## How to talk about this in an interview
- Lead with the philosophy: "I test behavior the way a user experiences it: render, interact with user-event, assert on what is visible, mock the network with MSW. Most of my UI tests are integration tests; pure logic gets unit tests; a few end-to-end tests cover the money paths."
- Mention the practical details that show experience: fresh `QueryClient` with retries off, `findBy` for async UI, `queryBy` for absence, act warnings and how to fix them.

> **Fill in:** what tests exist in the projects you work on at Devntech (RSpec and system specs on the Rails side, Jest or Vitest on the TypeScript side, any end-to-end suite) and which ones you wrote or extended.

> **Fill in:** a bug that a test caught (or that a test would have caught) in a flow such as rent payments, split bills or sign-up.

- If your frontend testing experience is limited, say so honestly, then describe exactly how you would test a form and a data-fetching component (the two examples above). Concrete reasoning beats vague claims. See [testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md) for the broader vocabulary (mocks vs stubs, coverage, TDD).

## Interview questions

**Q1. How do you test React components? What is your approach?**
With React Testing Library and Vitest or Jest: render the component with its real children and providers, interact through user-event, and assert on what users see, querying by role and label. The network is mocked with MSW so the real data layer runs. Pure logic gets unit tests, and a few Playwright or Cypress tests cover critical journeys end to end.
*Follow-up:* What do you avoid? Asserting on state, hook calls or class names, which break on refactors.

**Q2. What is the guiding principle of React Testing Library?**
Tests should resemble how the software is used: find elements the way users and assistive technologies do (role, label, text), interact like a user, and check visible results rather than internals. Tests then survive refactors and nudge you toward accessible markup.
*Follow-up:* Why is `getByTestId` the last resort? Users cannot see test ids, so it proves nothing about accessibility.

**Q3. What is the difference between getBy, queryBy and findBy?**
`getBy*` returns the element or throws immediately if there is none (or more than one). `queryBy*` returns `null` instead of throwing, which makes it the tool for asserting that something is absent. `findBy*` returns a promise that retries until the element appears or a timeout passes, for UI that shows up after async work. Each has an `All` variant for lists.
*Follow-up:* How do you wait for something to disappear? `waitForElementToBeRemoved` or `waitFor` with a `queryBy*` assertion.

**Q4. Which query should you prefer, and why?**
`getByRole` with an accessible name, such as `getByRole('button', { name: /save/i })`, then `getByLabelText` for form fields and `getByText` for plain content. These mirror how users find things, so a failing query often reveals an accessibility problem (an unlabeled input, a div used as a button).
*Follow-up:* What if an element has no role or text? Fix the markup first; use a test id only when that is not reasonable.

**Q5. user-event vs fireEvent?**
`fireEvent` dispatches a single synthetic DOM event. `user-event` simulates complete interactions (focus, pointer and keyboard events, input events per character) and respects disabled elements, so it catches more real bugs. Since v14 its methods are async: create a session with `userEvent.setup()` and `await` every call.
*Follow-up:* When is `fireEvent` still useful? For events user-event does not model, such as a raw `scroll` event.

**Q6. How do you test asynchronous UI, such as data loading?**
Mock the network with MSW, render, optionally assert the loading state, then `await screen.findBy...` for the final content, or `await waitFor(() => expect(...))` for non-element assertions. Test the error path too by overriding the handler with a 500 response in that test.
*Follow-up:* What if the test passes but logs act warnings? Some update happened after the assertions; wait for the final state.

**Q7. How do you mock API calls, and why MSW?**
MSW intercepts requests at the network level with handlers such as `http.get(url, () => HttpResponse.json(data))`, set up once in a test setup file, with `server.use` for per-test overrides. Components keep their real fetch code, query keys and error handling, so tests catch bugs in that layer, and the same handlers work in Storybook and local development. Mocking `fetch` or the API module is quicker but skips that code.
*Follow-up:* How do you catch unexpected requests? `server.listen({ onUnhandledRequest: 'error' })`.

**Q8. How do you test components that use TanStack Query?**
Wrap them in a `QueryClientProvider` with a new `QueryClient` per test, with `retry: false` so error cases fail fast, and mock responses with MSW. Assert with `findBy*` because data arrives asynchronously. A shared helper such as `renderWithClient` keeps this to one line per test.
*Follow-up:* Why not reuse one client? Cached data leaks between tests and makes them order-dependent.

**Q9. How do you test a custom hook?**
With `renderHook` from `@testing-library/react`: read `result.current`, change inputs with `rerender(newProps)`, and wrap manual updates (calling a returned function, advancing fake timers) in `act`. Provide context through the `wrapper` option. If the hook is used by a single component, testing through that component is often more meaningful.
*Follow-up:* Why is `act` needed? It flushes React's state updates and effects before you assert.

**Q10. What causes "not wrapped in act(...)" warnings, and how do you fix them?**
A component updated state after the test stopped waiting, typically when a fetch, timer or promise resolved after the last assertion or after the test ended. Fix it by awaiting the final UI (`findBy*`, `waitFor`), by awaiting user-event calls, or by controlling the async source (MSW, fake timers). Do not just sprinkle `act` around.
*Follow-up:* Does RTL wrap anything in act for you? Yes: `render`, user-event interactions and the async utilities.

**Q11. Are snapshot tests a good idea?**
Sometimes: they are quick to write and catch unexpected changes in stable output. For components they tend to be large, brittle and blindly updated, and they say nothing about behavior or accessibility. I use small inline snapshots for serialized values and explicit assertions for UI.
*Follow-up:* What makes a snapshot useful? Being small enough to review in a pull request.

**Q12. Jest vs Vitest?**
Vitest integrates with Vite (same config, transforms and plugins), handles ES modules and TypeScript natively, is very fast in watch mode and keeps a Jest-compatible API, so it is the default for Vite projects. Jest is mature, ubiquitous in older codebases and the standard in React Native. Testing Library works the same with both.
*Follow-up:* What changes when migrating? Mostly `jest.*` becomes `vi.*`, plus config and setup files.

**Q13. Unit vs integration vs end-to-end: how do you split the effort?**
Static typing and linting everywhere; unit tests for pure logic (formatters, reducers, splitting a bill); integration tests for most UI, rendering real component trees with a mocked network; a handful of end-to-end tests for critical journeys like sign-up and payment. Integration tests give the best confidence per cost on the frontend.
*Follow-up:* Why few end-to-end tests? They are slow, need environments and data, and flake more.

**Q14. How do you test components that use React Router or Context?**
Render them inside the providers they need. For routing, wrap in `MemoryRouter` with `initialEntries`, or build a `createMemoryRouter` with the real route objects when loaders and actions are involved. For context, wrap in the real provider with test values. A custom `renderWithProviders` helper keeps tests short.
*Follow-up:* How do you assert a navigation happened? Render the destination route in the memory router and assert its content.

**Q15. How do you test timers and debounced behavior?**
Switch to fake timers (`vi.useFakeTimers()`), trigger the behavior, assert that nothing happened yet, advance time with `vi.advanceTimersByTime(ms)` inside `act`, and assert again. With user-event, pass `advanceTimers: vi.advanceTimersByTime` to `userEvent.setup`. Restore real timers after each test.
*Follow-up:* Why not just wait 300 ms in the test? Real waiting makes suites slow and flaky.

**Q16. What would you test in a payment or sign-up flow?**
Integration tests: validation messages, the disabled button while submitting, server errors mapped to fields, success navigation, and no double submission on fast clicks. Unit tests for amount calculations and formatting. An end-to-end test for the happy path against a test backend or a payment provider's test mode, plus one failure path. The server needs its own tests for idempotency and authorization.
*Follow-up:* How do you avoid real charges in tests? Use the provider's test mode and test keys, or mock the provider at the network boundary.

## Further reading
- [Testing Library: queries](https://testing-library.com/docs/queries/about)
- [Vitest guide](https://vitest.dev/guide/)
- [MSW docs](https://mswjs.io/docs/)
- [Playwright docs](https://playwright.dev/docs/intro)
