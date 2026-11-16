# Machine-Coding Rounds: Building React Features Under Time Pressure

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [State management](03-state-management.md), [Data fetching](04-data-fetching-tanstack-query.md), [Forms and validation](05-forms-and-validation.md), [Performance](06-performance.md), [Patterns and architecture](07-patterns-and-architecture.md), [Testing](10-testing.md), [React question bank](99-interview-questions.md), [HTML, CSS and accessibility](../04-web-fundamentals/02-html-css.md)
> **On your resume:** React, TypeScript and React Native (Expo) features for Devntech's international clients, React dashboards in the PureLogics LMS, Next.js and TypeScript APIs at RemoteDev, and Rails CRUD since the Visnext training.

## Quick revision
- A machine-coding round is a 60-120 minute build of a small working feature, live or as a short take-home. Grading order: working core flow, then component and state design, edge cases, accessibility, code quality, tests and communication.
- First 5-10 minutes: restate the task, ask about scope (libraries? real or mock API? styling? tests?), write down what is in and out of scope, and sketch the component tree and state shape.
- Build the happy path end to end with mock data first; then edge cases (empty, loading, error, rapid input, double submit); then polish. A complete simple app beats a half-built clever one.
- State: keep it minimal and derive the rest during render (filtered rows, counts, validity); use `useReducer` when several values change together; key lists by stable ids, never indexes; render recursive data with recursive components over flat, normalized state.
- Effects: debounce input, abort stale requests and clear timers and listeners in the cleanup (StrictMode's extra development run exposes missing cleanups); cache results by key and show loading, error and empty states.
- Accessibility: native elements first (`button`, `label`, `fieldset`, radio inputs), then the WAI-ARIA pattern for custom widgets (combobox, carousel), keyboard support and visible focus.
- Persistence: lazy initial state reads `localStorage` once inside `try/catch`, an effect writes on change, and the key carries a version.
- Narrate decisions and trade-offs, not keystrokes. When time runs out, say what you would do next.

## 1. What the round is and how it is judged

The round mimics the job: turn a loose request into working UI under time pressure while explaining yourself. It may be a live 60-90 minute session (CodeSandbox, StackBlitz or your editor), a 2-3 hour take-home, or pairing where the interviewer adds requirements halfway ("now persist it", "now make it keyboard accessible").

| Criterion | What earns points | What loses points |
| --- | --- | --- |
| Working core flow | The main journey works end to end, even if plain | A polished half that never works |
| Component and state design | Small components, one owner per piece of state, derived values | Duplicated state that drifts, one 400-line component |
| Edge cases | Empty, loading, error, invalid input and races handled | Crashes on empty arrays, double submits, stale responses |
| Accessibility | Labels, semantic elements, keyboard support, focus management | Clickable `div`s, unlabeled inputs, keyboard traps |
| Code quality | Clear names, real types, no dead code | `any` everywhere, copy-pasted blocks, magic numbers |
| Tests | One or two meaningful tests, or a clear test plan | No tests and no mention of them |
| Communication | Assumptions stated, trade-offs explained, hints used | Silence, then surprises at the end |

## 2. Strategy: a 90-minute plan

| Phase | Time | Output |
| --- | --- | --- |
| Clarify | 5-10 min | Requirements, out-of-scope list, assumptions as a comment at the top of the file |
| Design | 5-10 min | Component tree, state shape, data flow, mock API shape |
| Happy path | 30-40 min | The core flow working with mock data, layout only |
| Edge cases | 15 min | Loading, error and empty states, validation, races, keyboard |
| Polish | 10 min | Accessibility pass, naming, dead code removed, minimal styles |
| Tests and wrap-up | 10 min | One or two tests, a summary of trade-offs and next steps |

- Scale the boxes to the round, and ask before assuming: "Can I use TanStack Query, or should I write the fetching myself?" Many rounds forbid libraries to test fundamentals; using one when allowed is fine if you can explain what it does for you. Start from a template (Vite's `react-ts`), mock the API with a function that resolves after a delay (so loading states are real and errors easy to simulate), and checkpoint after each working milestone.
- Talk while you code: "I'll derive the filtered list so it can never go stale", "no virtualization for 50 rows; I'd add it past a few thousand". Stuck for more than a few minutes? Stub it, move on, come back.

## 3. Problem: autocomplete with debounce, abort, cache and keyboard navigation

- **Requirements:** [Q111 in the React bank](99-interview-questions.md) builds a basic debounced search; this version adds a result cache, keyboard navigation and the ARIA combobox pattern. Suggestions appear 300 ms after typing stops (2+ characters); stale requests are aborted; repeated queries are served from the cache; ArrowDown and ArrowUp move the highlight, Enter selects, Escape closes; mouse selection works; status changes are announced.
- **Components and state:** one component. State: `text` (instant), `query` (debounced, normalized), `results` (the cache, keyed by query), `failed`, `open` and `active` (the highlighted index). Visible options, loading, errors and the announcement are all derived.

```tsx
import { useEffect, useId, useState, type KeyboardEvent } from 'react';

type Option = { id: string; label: string };

export function Autocomplete({ onSelect }: { onSelect: (option: Option) => void }) {
  const [text, setText] = useState('');
  const [query, setQuery] = useState(''); // debounced, normalized copy of text
  const [results, setResults] = useState<Partial<Record<string, Option[]>>>({}); // cache: query -> options
  const [failed, setFailed] = useState<string | null>(null); // the query whose request failed
  const [open, setOpen] = useState(false);
  const [active, setActive] = useState(-1); // highlighted option, -1 for none
  const id = useId();

  useEffect(() => {
    const t = setTimeout(() => setQuery(text.trim().toLowerCase()), 300);
    return () => clearTimeout(t); // every keystroke restarts the wait
  }, [text]);

  const cached = results[query];
  useEffect(() => {
    if (query.length < 2 || cached) return; // too short, or a cache hit: no request
    const controller = new AbortController();
    setFailed(null);
    fetch(`/api/products/suggest?q=${encodeURIComponent(query)}`, { signal: controller.signal })
      .then((res) => (res.ok ? (res.json() as Promise<Option[]>) : Promise.reject(new Error(`HTTP ${res.status}`))))
      .then((options) => setResults((r) => ({ ...r, [query]: options })))
      .catch(() => { if (!controller.signal.aborted) setFailed(query); }); // aborts are expected, not errors
    return () => controller.abort(); // a newer query or unmount cancels this request
  }, [query, cached]);

  const ready = query.length >= 2;
  const visible = ready && cached ? cached : [];
  const expanded = open && visible.length > 0;
  const message = !open || !ready ? '' : cached ? `Results: ${cached.length}` : failed === query ? 'Search failed' : 'Loading...';
  const close = () => { setOpen(false); setActive(-1); };
  const choose = (option: Option) => { setText(option.label); close(); onSelect(option); };

  function onKeyDown(e: KeyboardEvent<HTMLInputElement>) {
    const n = visible.length;
    if ((e.key === 'ArrowDown' || e.key === 'ArrowUp') && n > 0) {
      e.preventDefault(); // keep the caret where it is
      const down = e.key === 'ArrowDown';
      setOpen(true);
      setActive((i) => (down ? (i + 1) % n : i <= 0 ? n - 1 : i - 1)); // wrap around
    } else if (e.key === 'Enter' && expanded && active >= 0) {
      e.preventDefault(); // do not submit a surrounding form
      choose(visible[active]);
    } else if (e.key === 'Escape') close();
  }

  return (
    <div className="autocomplete">
      <label htmlFor={`${id}-input`}>Search products</label>
      <input id={`${id}-input`} role="combobox" aria-autocomplete="list" autoComplete="off"
        aria-expanded={expanded} aria-controls={`${id}-list`}
        aria-activedescendant={expanded && active >= 0 ? `${id}-opt-${active}` : undefined}
        value={text} onChange={(e) => { setText(e.target.value); setOpen(true); setActive(-1); }}
        onKeyDown={onKeyDown} onBlur={close} />
      <ul id={`${id}-list`} role="listbox" aria-label="Suggestions" hidden={!expanded}>
        {visible.map((option, i) => (
          <li key={option.id} id={`${id}-opt-${i}`} role="option" aria-selected={i === active}
            onMouseDown={(e) => e.preventDefault()} // keep focus in the input, so blur does not close the list first
            onClick={() => choose(option)}>
            {option.label}
          </li>
        ))}
      </ul>
      <p aria-live="polite">{message}</p>
    </div>
  );
}
```

Notes: focus never leaves the input; `aria-activedescendant` tells screen readers which option is highlighted, and CSS styles `[aria-selected="true"]`. The cache is state rather than a ref because render reads it; cap it (say, the last 50 queries) in long sessions. Choosing an option sets the text, so one lookup runs for the chosen label; skip it by remembering the choice if asked. With TanStack Query, `useQuery({ queryKey: ['suggest', query], queryFn: ({ signal }) => fetchSuggest(query, signal), enabled: query.length >= 2, staleTime: 60_000 })` replaces the fetch effect, the cache and the abort logic.

## 4. Problem: kanban board with localStorage persistence

- **Requirements:** columns To do, In progress and Done; add a card to To do; move cards between columns by drag and drop and by keyboard; delete cards; the board survives a reload, and corrupted storage must not crash the app.
- **Components and state:** one `cards` array where each card stores its `column`, managed by a reducer with `add`, `move` and `remove` actions; per-column lists are derived. Storage is read once through lazy initialization and written by an effect.

```tsx
import { useEffect, useReducer, useState, type FormEvent } from 'react';

const COLUMNS = [{ id: 'todo', title: 'To do' }, { id: 'doing', title: 'In progress' }, { id: 'done', title: 'Done' }] as const;
type ColumnId = (typeof COLUMNS)[number]['id'];
type Card = { id: string; title: string; column: ColumnId };
type Action = { type: 'add'; card: Card } | { type: 'move'; id: string; to: ColumnId } | { type: 'remove'; id: string };
const STORAGE_KEY = 'kanban:v1'; // bump the version when the stored shape changes

function reducer(cards: Card[], action: Action): Card[] {
  if (action.type === 'add') return [...cards, action.card];
  if (action.type === 'remove') return cards.filter((c) => c.id !== action.id);
  const card = cards.find((c) => c.id === action.id); // TypeScript has narrowed action to 'move'
  if (!card || card.column === action.to) return cards; // nothing to do: same reference, no re-render
  return [...cards.filter((c) => c.id !== action.id), { ...card, column: action.to }]; // lands at the bottom
}

function loadCards(): Card[] {
  try {
    const parsed: unknown = JSON.parse(localStorage.getItem(STORAGE_KEY) ?? '[]');
    return Array.isArray(parsed) ? (parsed as Card[]) : [];
  } catch {
    return []; // corrupted JSON or blocked storage: start empty instead of crashing
  }
}

export function KanbanBoard() {
  const [cards, dispatch] = useReducer(reducer, undefined, loadCards); // lazy init: storage is read once
  const [title, setTitle] = useState('');
  useEffect(() => { localStorage.setItem(STORAGE_KEY, JSON.stringify(cards)); }, [cards]);

  function addCard(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    if (!title.trim()) return;
    dispatch({ type: 'add', card: { id: crypto.randomUUID(), title: title.trim(), column: 'todo' } }); // id made here keeps the reducer pure
    setTitle('');
  }

  return (
    <div>
      <form onSubmit={addCard}>
        <label htmlFor="new-card">New card</label>
        <input id="new-card" value={title} onChange={(e) => setTitle(e.target.value)} />
        <button type="submit">Add</button>
      </form>
      <div className="board">
        {COLUMNS.map((col) => {
          const columnCards = cards.filter((c) => c.column === col.id);
          return (
            <section key={col.id} aria-label={col.title}
              onDragOver={(e) => e.preventDefault()} // required, or the browser refuses the drop
              onDrop={(e) => dispatch({ type: 'move', id: e.dataTransfer.getData('text/plain'), to: col.id })}>
              <h2>{col.title} ({columnCards.length})</h2>
              <ul>
                {columnCards.map((card) => (
                  <li key={card.id} draggable onDragStart={(e) => e.dataTransfer.setData('text/plain', card.id)}>
                    {card.title}
                    <select aria-label={`Move ${card.title}`} value={card.column}
                      onChange={(e) => dispatch({ type: 'move', id: card.id, to: e.target.value as ColumnId })}>
                      {COLUMNS.map((c) => <option key={c.id} value={c.id}>{c.title}</option>)}
                    </select>
                    <button onClick={() => dispatch({ type: 'remove', id: card.id })} aria-label={`Delete ${card.title}`}>Delete</button>
                  </li>
                ))}
              </ul>
            </section>
          );
        })}
      </div>
    </div>
  );
}
```

Notes: the `select` is the keyboard and screen reader path, because native drag and drop is mouse-only and fails on touch screens (dnd-kit adds keyboard and touch dragging plus reordering in production). Unknown ids are ignored, so a drop from outside the board is harmless. Under SSR (Next.js) there is no `localStorage` on the server: load it in an effect or render the board client-only. Listen to the `storage` event to sync tabs. `crypto.randomUUID()` needs a secure context (HTTPS or localhost).

## 5. Problem: accessible star rating

- **Requirements:** a reusable 1-5 rating input; click to rate and hover to preview; full keyboard support; screen readers announce the group label and "3 stars"; controlled by its parent.
- **Components and state:** a controlled `StarRating` (`value` and `onChange` from the parent) with one piece of local UI state, `hover`. Native radio inputs provide the semantics and arrow-key behavior; the stars are decoration.

```tsx
import { useId, useState } from 'react';

type StarRatingProps = { value: number; onChange: (value: number) => void; max?: number; label?: string };

export function StarRating({ value, onChange, max = 5, label = 'Rating' }: StarRatingProps) {
  const [hover, setHover] = useState<number | null>(null);
  const name = useId(); // one radio group per instance
  const shown = hover ?? value; // preview while hovering
  return (
    <fieldset className="star-rating" onMouseLeave={() => setHover(null)}>
      <legend>{label}</legend>
      {Array.from({ length: max }, (_, i) => i + 1).map((star) => (
        <label key={star} onMouseEnter={() => setHover(star)}>
          <input type="radio" className="visually-hidden" name={name} value={star}
            checked={value === star} onChange={() => onChange(star)} />
          <span aria-hidden="true" className={star <= shown ? 'star filled' : 'star'}>★</span>
          <span className="visually-hidden">{star === 1 ? '1 star' : `${star} stars`}</span>
        </label>
      ))}
    </fieldset>
  );
}
```

```css
.visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip-path: inset(50%); white-space: nowrap; }
.star { font-size: 1.5rem; color: #6b7280; cursor: pointer; } .star.filled { color: #b45309; }
.star-rating input:focus-visible + .star { outline: 2px solid #2563eb; outline-offset: 2px; }
```

Notes: radios give you Tab into the group, arrow keys to change the value and correct announcements for free; a `div` version needs `role="radiogroup"`, `role="radio"`, `aria-checked`, a roving `tabIndex` and key handlers to match. `display: none` would remove the inputs from the keyboard and accessibility tree, hence the clip pattern. For read-only display, render `<span role="img" aria-label="4 out of 5 stars">` instead.

## 6. Problem: data table with sorting, filtering and pagination

- **Requirements:** columns name, email, role and joined date; click a header to sort ascending, then descending; a text filter on name and email; 10 rows per page with Previous and Next; filtering or re-sorting returns to page 1; an empty state; screen readers know the sort order.
- **Components and state:** `UsersTable` receives `rows` and owns only `filter`, `sort` and `page`. The filtered and sorted list, the page count and the visible rows are derived, and the page is clamped during render instead of fixed up in an effect.

```tsx
import { useMemo, useState } from 'react';

type User = { id: number; name: string; email: string; role: 'admin' | 'member'; joined: string }; // joined: ISO date
type SortKey = Exclude<keyof User, 'id'>;
const COLUMNS: { key: SortKey; label: string }[] = [
  { key: 'name', label: 'Name' }, { key: 'email', label: 'Email' }, { key: 'role', label: 'Role' }, { key: 'joined', label: 'Joined' },
];
const PAGE_SIZE = 10;

export function UsersTable({ rows }: { rows: User[] }) {
  const [filter, setFilter] = useState('');
  const [sort, setSort] = useState<{ key: SortKey; dir: 'asc' | 'desc' }>({ key: 'name', dir: 'asc' });
  const [page, setPage] = useState(1);
  const sorted = useMemo(() => {
    const q = filter.trim().toLowerCase();
    const matches = q ? rows.filter((r) => r.name.toLowerCase().includes(q) || r.email.toLowerCase().includes(q)) : rows;
    const sign = sort.dir === 'asc' ? 1 : -1;
    return [...matches].sort((a, b) => a[sort.key].localeCompare(b[sort.key]) * sign); // copy: never sort props in place
  }, [rows, filter, sort]);
  const pageCount = Math.max(1, Math.ceil(sorted.length / PAGE_SIZE));
  const current = Math.min(page, pageCount); // clamp if the data shrinks
  const pageRows = sorted.slice((current - 1) * PAGE_SIZE, current * PAGE_SIZE);

  function toggleSort(key: SortKey) {
    setSort((s) => (s.key === key ? { key, dir: s.dir === 'asc' ? 'desc' : 'asc' } : { key, dir: 'asc' }));
    setPage(1);
  }

  return (
    <div>
      <label>
        Filter <input type="search" value={filter} onChange={(e) => { setFilter(e.target.value); setPage(1); }} />
      </label>
      <table>
        <caption>Users: {sorted.length} found, page {current} of {pageCount}</caption>
        <thead>
          <tr>
            {COLUMNS.map((col) => {
              const dir = sort.key === col.key ? sort.dir : null;
              return (
                <th key={col.key} aria-sort={dir ? (dir === 'asc' ? 'ascending' : 'descending') : undefined}>
                  <button onClick={() => toggleSort(col.key)}>
                    {col.label} <span aria-hidden="true">{dir === 'asc' ? '▲' : dir === 'desc' ? '▼' : ''}</span>
                  </button>
                </th>
              );
            })}
          </tr>
        </thead>
        <tbody>
          {pageRows.map((u) => (
            <tr key={u.id}>{COLUMNS.map((col) => <td key={col.key}>{u[col.key]}</td>)}</tr>
          ))}
          {pageRows.length === 0 && <tr><td colSpan={COLUMNS.length}>No users match your filter.</td></tr>}
        </tbody>
      </table>
      <button onClick={() => setPage(current - 1)} disabled={current === 1}>Previous</button>
      <button onClick={() => setPage(current + 1)} disabled={current === pageCount}>Next</button>
    </div>
  );
}
```

Notes: sortable headers are real `button`s inside `th`, and `aria-sort` sits only on the sorted column. ISO dates sort correctly as strings; numbers need a numeric comparator. `useMemo` earns its place because filtering and sorting thousands of rows on every keystroke costs real time (React Compiler, where enabled, often makes it unnecessary). For large or server-side data, send `q`, `sort` and `page` (or a cursor) to the API, keep them in the URL, debounce the filter and use `placeholderData: keepPreviousData` in TanStack Query.

## 7. Problem: multi-step form wizard with per-step validation

- **Requirements:** steps Account (email, password), Profile (name, optional city) and Review; Next validates only the current step and blocks on errors; Back keeps the data; focus lands on the first invalid field or the new step's heading; submit shows pending and failure states. ([Q114](99-interview-questions.md) covers single-form validation; here the focus is step logic.)
- **Components and state:** one `data` object for all steps (so Back loses nothing), a `step` index, the current step's `errors` and a submit `status`. Steps and rules are configuration: a step lists its fields, and each field has an optional rule, so adding a step means adding an object.

```tsx
import { useEffect, useId, useRef, useState, type FormEvent } from 'react';

type Data = { email: string; password: string; name: string; city: string };
type Field = keyof Data;
type Errors = Partial<Record<Field, string>>;
const STEPS: { title: string; fields: Field[] }[] = [{ title: 'Account', fields: ['email', 'password'] }, { title: 'Profile', fields: ['name', 'city'] }];
const RULES: Partial<Record<Field, (value: string) => string | null>> = {
  email: (v) => (/^\S+@\S+\.\S+$/.test(v) ? null : 'Enter a valid email'),
  password: (v) => (v.length >= 8 ? null : 'Use at least 8 characters'),
  name: (v) => (v.trim() ? null : 'Name is required'), // city has no rule: optional
};
const LABELS: Record<Field, string> = { email: 'Email', password: 'Password', name: 'Full name', city: 'City' };

export function SignupWizard({ onSubmit }: { onSubmit: (data: Data) => Promise<void> }) {
  const [data, setData] = useState<Data>({ email: '', password: '', name: '', city: '' });
  const [step, setStep] = useState(0); // STEPS.length means the review step
  const [errors, setErrors] = useState<Errors>({});
  const [status, setStatus] = useState<'editing' | 'submitting' | 'failed' | 'done'>('editing');
  const headingRef = useRef<HTMLHeadingElement>(null);
  const navigated = useRef(false); // no focus jump on the first render
  const uid = useId();
  const current = step < STEPS.length ? STEPS[step] : null; // null on the review step

  useEffect(() => {
    if (navigated.current) headingRef.current?.focus(); // move focus to the new step's heading
  }, [step]);
  const goTo = (next: number) => { navigated.current = true; setErrors({}); setStep(next); };

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    if (current) {
      const stepErrors: Errors = {};
      for (const f of current.fields) {
        const message = RULES[f]?.(data[f]); // only this step's fields are checked
        if (message) stepErrors[f] = message;
      }
      setErrors(stepErrors);
      const firstInvalid = current.fields.find((f) => stepErrors[f]);
      if (firstInvalid) document.getElementById(`${uid}-${firstInvalid}`)?.focus();
      else goTo(step + 1);
      return;
    }
    setStatus('submitting');
    onSubmit(data).then(() => setStatus('done'), () => setStatus('failed'));
  }
  if (status === 'done') return <p role="status">Account created.</p>;
  return (
    <form onSubmit={handleSubmit} noValidate>
      <h2 ref={headingRef} tabIndex={-1}>Step {step + 1} of {STEPS.length + 1}: {current ? current.title : 'Review'}</h2>
      {current
        ? current.fields.map((field) => (
            <div key={field}>
              <label htmlFor={`${uid}-${field}`}>{LABELS[field]}{RULES[field] ? '' : ' (optional)'}</label>
              <input id={`${uid}-${field}`} type={field === 'password' ? 'password' : field === 'email' ? 'email' : 'text'}
                value={data[field]} onChange={(e) => setData({ ...data, [field]: e.target.value })}
                aria-invalid={errors[field] ? true : undefined} aria-describedby={errors[field] ? `${uid}-${field}-error` : undefined} />
              {errors[field] && <p id={`${uid}-${field}-error`}>{errors[field]}</p>}
            </div>
          ))
        : (['email', 'name', 'city'] as const).map((f) => <p key={f}>{LABELS[f]}: {data[f] || 'Not given'}</p>)}
      {status === 'failed' && <p role="alert">Could not create the account. Try again.</p>}
      <button type="button" onClick={() => goTo(step - 1)} disabled={step === 0 || status === 'submitting'}>Back</button>
      <button type="submit" disabled={status === 'submitting'}>
        {current ? 'Next' : status === 'submitting' ? 'Creating account...' : 'Create account'}
      </button>
    </form>
  );
}
```

Notes: one `form` with a submit button means Enter advances the wizard, and `noValidate` stops browser bubbles from competing with your messages. Cross-field rules (password confirmation) need a step-level validator next to the field rules. The server validates everything again, because the client can be bypassed. In production, a Zod schema per step with React Hook Form's `trigger(fields)` gives the same flow; keep the step in the URL if a refresh must not lose the user's place.

## 8. Problem: image carousel with autoplay and cleanup

- **Requirements:** one slide at a time; Previous and Next wrap around; dots jump to a slide; autoplay every 5 seconds that pauses on hover and focus; an explicit stop and start control; no autoplay for users who prefer reduced motion; no leaked timers.
- **Components and state:** `index`, `playing` (the user's choice) and `paused` (temporary). One `setTimeout` effect keyed on the current slide restarts the countdown after every change, manual or automatic, and its cleanup covers pausing, stopping and unmounting.

```tsx
import { useEffect, useState } from 'react';

type Slide = { src: string; alt: string };
const reducedMotion = () => typeof window !== 'undefined' && window.matchMedia('(prefers-reduced-motion: reduce)').matches;

export function Carousel({ slides, intervalMs = 5000 }: { slides: Slide[]; intervalMs?: number }) {
  const [index, setIndex] = useState(0);
  const [playing, setPlaying] = useState(() => !reducedMotion()); // the user's explicit choice
  const [paused, setPaused] = useState(false); // temporary: pointer or keyboard focus inside
  const count = slides.length;
  const current = count ? index % count : 0; // stays valid if the slides prop shrinks

  useEffect(() => {
    if (!playing || paused || count < 2) return;
    const t = setTimeout(() => setIndex((i) => (i + 1) % count), intervalMs);
    return () => clearTimeout(t); // runs on every slide change, pause and unmount
  }, [playing, paused, count, intervalMs, current]); // current: manual navigation restarts the countdown

  if (count === 0) return null;
  const go = (i: number) => setIndex((i + count) % count);
  return (
    <section aria-roledescription="carousel" aria-label="Featured houses"
      onMouseEnter={() => setPaused(true)} onMouseLeave={() => setPaused(false)} onFocus={() => setPaused(true)}
      onBlur={(e) => { if (!e.currentTarget.contains(e.relatedTarget)) setPaused(false); }} // focus left the carousel
    >
      <button onClick={() => setPlaying((p) => !p)}>{playing ? 'Stop slideshow' : 'Start slideshow'}</button>
      <div aria-live={playing ? 'off' : 'polite'}>
        {slides.map((slide, i) => (
          <div key={slide.src} role="group" aria-roledescription="slide" aria-label={`${i + 1} of ${count}`} hidden={i !== current}>
            <img src={slide.src} alt={slide.alt} loading={i === 0 ? 'eager' : 'lazy'} />
          </div>
        ))}
      </div>
      <button onClick={() => go(current - 1)} aria-label="Previous slide">&lsaquo;</button>
      <button onClick={() => go(current + 1)} aria-label="Next slide">&rsaquo;</button>
      {slides.map((slide, i) => (
        <button key={slide.src} onClick={() => go(i)} aria-label={`Show slide ${i + 1}`} aria-current={i === current} />
      ))}
    </section>
  );
}
```

Notes: a timeout per slide is simpler than an interval, because there is never a stale interval to reset after manual navigation. WCAG requires a way to pause content that moves automatically for more than five seconds, hence the explicit control; the live region stays silent during autoplay so screen readers are not interrupted every five seconds. In SSR frameworks, read the media query in an effect to avoid a hydration mismatch. Swipe needs pointer events or a library such as Embla.

## 9. Problem: nested comments with recursive components

- **Requirements:** render a thread of any depth; reply to any comment; collapse and expand replies with a count; new replies appear in place.
- **Components and state:** flat state (`{ id, parentId }` records) plus a memoized `parentId -> children` map. `CommentList` renders the children of one parent and each `CommentItem` renders a `CommentList` for its own replies; the recursion stops when a comment has none. Collapse and reply-box state stay local to each item.

```tsx
import { useMemo, useState, type FormEvent } from 'react';

type Comment = { id: string; parentId: string | null; author: string; text: string };
type ListProps = { parentId: string | null; childrenOf: Map<string | null, Comment[]>; onReply: (parentId: string, text: string) => void };

export function CommentThread({ initial }: { initial: Comment[] }) {
  const [comments, setComments] = useState(initial); // flat: each reply points to its parent
  const childrenOf = useMemo(() => {
    const map = new Map<string | null, Comment[]>();
    for (const c of comments) {
      const siblings = map.get(c.parentId);
      if (siblings) siblings.push(c);
      else map.set(c.parentId, [c]);
    }
    return map;
  }, [comments]);
  const reply = (parentId: string | null, text: string) =>
    setComments((cs) => [...cs, { id: crypto.randomUUID(), parentId, author: 'You', text }]);

  return (
    <section aria-label="Comments">
      <ReplyForm label="Add a comment" onSubmit={(text) => reply(null, text)} />
      <CommentList parentId={null} childrenOf={childrenOf} onReply={reply} />
    </section>
  );
}

function CommentList({ parentId, childrenOf, onReply }: ListProps) {
  const items = childrenOf.get(parentId) ?? [];
  if (items.length === 0) return null; // base case: no replies, stop recursing
  return <ul>{items.map((c) => <CommentItem key={c.id} comment={c} childrenOf={childrenOf} onReply={onReply} />)}</ul>;
}

function CommentItem({ comment, childrenOf, onReply }: Omit<ListProps, 'parentId'> & { comment: Comment }) {
  const [collapsed, setCollapsed] = useState(false);
  const [replying, setReplying] = useState(false);
  const replyCount = childrenOf.get(comment.id)?.length ?? 0;
  return (
    <li>
      <p><strong>{comment.author}</strong> {comment.text}</p>
      <button onClick={() => setReplying((r) => !r)}>Reply</button>
      {replyCount > 0 && (
        <button aria-expanded={!collapsed} onClick={() => setCollapsed((c) => !c)}>
          {collapsed ? `Show replies (${replyCount})` : 'Hide replies'}
        </button>
      )}
      {replying && <ReplyForm label={`Reply to ${comment.author}`} onSubmit={(text) => { onReply(comment.id, text); setReplying(false); }} />}
      {!collapsed && <CommentList parentId={comment.id} childrenOf={childrenOf} onReply={onReply} />}
    </li>
  );
}

function ReplyForm({ label, onSubmit }: { label: string; onSubmit: (text: string) => void }) {
  function submit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const text = String(new FormData(e.currentTarget).get('text') ?? '').trim();
    if (text) { onSubmit(text); e.currentTarget.reset(); } // uncontrolled input: the form owns its value
  }
  return (
    <form onSubmit={submit}>
      <label>{label} <input name="text" /></label>
      <button type="submit">Post</button>
    </form>
  );
}
```

Notes: with nested state, a deep reply means immutably copying every ancestor; flat state makes it one append, and nested JSON from an API can be normalized once on load. Pass a `depth` prop to cap indentation or show "continue thread". A folder tree is the same shape: a node renders its children behind a disclosure button with `aria-expanded`, and the full WAI-ARIA tree pattern (`role="tree"`, arrow-key navigation) comes in only if asked.

## 10. Full-stack timed CRUD

Some rounds ask for a small full-stack feature in 60-90 minutes (projects with tasks: list, create, edit, delete). Speed comes from generators and a fixed checklist.

**Rails checklist**
- `bin/rails g scaffold Task title:string done:boolean project:references`, then `bin/rails db:migrate`: model, migration, routes, controller, views and test stubs at once. Walk the interviewer through the generated controller.
- Validations and constraints: `validates :title, presence: true, length: { maximum: 120 }` plus `null: false` in the migration; `belongs_to :project` and `has_many :tasks, dependent: :destroy`.
- Strong parameters (`params.expect(task: [:title, :done, :project_id])` on Rails 8.0+, `params.require(:task).permit(...)` before) and owner-scoped lookups (`current_user.tasks.find(params[:id])`) when there is authentication.
- Turbo: scaffold forms submit through Turbo Drive, and failed saves re-render with a 422 status (the scaffold does this) so errors appear; for inline create or edit, use a Turbo Frame or respond with `turbo_stream.prepend`. Add `includes(:project)` on the index and pagination if lists can grow.
- Request specs for the happy path and a validation failure, for example `post tasks_path, params: { task: { title: "" } }` followed by `expect(response).to have_http_status(422)`.

**Next.js or Express checklist**
- Schema first: `const TaskInput = z.object({ title: z.string().trim().min(1).max(120), done: z.boolean().default(false) })`, with the type from `z.infer<typeof TaskInput>`.
- Routes: in the Next.js App Router, `app/api/tasks/route.ts` exports `GET` and `POST`, and `app/api/tasks/[id]/route.ts` exports `GET`, `PATCH` and `DELETE` (in Next.js 15+, `params` is a promise you `await`). In Express: `express.json()`, a router with the same five routes and a four-argument error middleware.
- Validate every write with `TaskInput.safeParse(body)` and return 422 (or 400) with `error.issues` on failure; 201 on create, 404 for unknown ids, 204 on delete. Use the template's ORM (Prisma, Drizzle, Mongoose), or an in-memory array behind a small repository module.
- UI: a list page (a Server Component, or `useQuery` from TanStack Query), a detail page and one form shared by create and edit that shows server field errors, with loading, empty and error states; after a mutation, invalidate the query or call `revalidatePath` or `router.refresh()`. In Next.js, a Server Function (`"use server"`) as the form `action` with `useActionState` is the alternative. Finish with one API test (supertest for Express, or calling the route handler directly) and one component test, or say which you would write.

## Common pitfalls
- Coding before clarifying, then building unrequested features while the core flow is still broken.
- Spending the first 20 minutes on tooling or styling.
- Storing derived values (filtered rows, counts, `isValid`) in state and forgetting to update them.
- Effects without cleanup: intervals, listeners and fetches that outlive the component or race each other.
- Index keys on lists that reorder or delete items; mutating props or state in place (`rows.sort()`); mouse-only widgets with no keyboard path, labels or visible focus.
- Coding in silence, then running out of time without saying what is left.

## How to talk about this in an interview
- Ask the recruiter about the format in advance (live or take-home, length, plain React or a framework, libraries allowed) and practice that exact format with a timer.
- Tie the problems to real work: the Coloc resident app's events, tasks and split bills screens and the PureLogics role-based dashboards are lists, forms and tables underneath. Claim only widgets you actually built, and if you normally lean on libraries (TanStack Query, React Hook Form, dnd-kit), say so and show you know what they replace.
- Your PUCIT teaching assistant work is real practice at explaining code while writing it; mention it when asked how you communicate while pairing.

> **Fill in:** one component from your Devntech or PureLogics work that resembles these problems (a search box, a filterable table, a multi-step form), and one edge case you handled in it.

## Interview questions

**Q1. How do you start a machine-coding round?**
Restate the problem, ask scoping questions (libraries, API, styling, tests), write down the requirements and what is out of scope, then spend a few minutes on the component tree and state shape, and confirm the plan before coding.
*Follow-up:* What if the requirements are vague on purpose? State your assumption out loud, write it as a comment, and pick the simplest reasonable behavior.

**Q2. How do you decide where state lives?**
List the state, delete anything derivable, then put each piece in the lowest component that owns all of its readers, lifting it only when a sibling needs it. In a 90-minute round, local state and props are almost always enough.
*Follow-up:* When do you switch to `useReducer`? When several values change together or transitions are named actions, like the kanban board's add, move and remove.

**Q3. Debounce or throttle: which for a typeahead, and which for scroll?**
Debounce waits until events stop for N ms and fires once: right for typeahead requests. Throttle fires at most once every N ms during a stream: right for scroll or resize handlers that must keep updating.
*Follow-up:* Does debouncing fix race conditions? No: a slow earlier response can still land after a later one, so abort stale requests or ignore old responses.

**Q4. How do you prevent stale responses in a search box?**
Abort the previous request in the effect cleanup with an `AbortController` (the next query or an unmount triggers it) and ignore abort errors; alternatives are an `ignore` flag set in the cleanup or a request id checked before setting state.
*Follow-up:* How does TanStack Query handle it? Each query key has its own cache entry, and the `signal` passed to `queryFn` is aborted when the query is no longer needed.

**Q5. How do you make a custom autocomplete keyboard accessible?**
Follow the WAI-ARIA combobox pattern: the input has `role="combobox"`, `aria-expanded`, `aria-controls` and `aria-activedescendant`; options have `role="option"` and `aria-selected` inside a `role="listbox"`; arrows move the highlight, Enter selects, Escape closes, and focus stays in the input.
*Follow-up:* `aria-activedescendant` vs roving `tabIndex`? The first keeps DOM focus on one element (ideal for inputs); the second moves real focus between items (toolbars, tab lists).

**Q6. How do you persist state to localStorage safely?**
Read once with lazy initialization inside `try/catch` and validate the shape, because storage can be full, disabled or corrupted; write in an effect when the state changes; version the key; and guard against SSR, where `window` does not exist.
*Follow-up:* How do you keep two tabs in sync? Listen for the `storage` event, which fires in the other tabs when one tab writes.

**Q7. Client-side or server-side sorting, filtering and pagination?**
Client-side is fine for a few hundred or thousand rows that are already loaded: instant and simple. Beyond that, or when data changes often, send the sort, filter and page or cursor to the API and keep them in the URL so views are shareable.
*Follow-up:* Why reset to page 1 when the filter changes? The current page may not exist in the new results, and users expect the first matches.

**Q8. How do you stop users from skipping steps in a wizard?**
Next validates only the current step and refuses to advance on errors, the step indicator never links ahead of the furthest valid step, and the server validates everything again on submit. Keeping all data in one object means Back never loses input.
*Follow-up:* Where does the step live if a refresh must not reset it? In the URL (`?step=2`), with the draft in `sessionStorage`.

**Q9. Why does an autoplay carousel need careful cleanup and a pause control?**
Without cleanup, timers keep firing after unmount or stack up so the slideshow speeds up. WCAG requires a way to pause content that moves automatically for more than five seconds, and users who prefer reduced motion should not get autoplay at all.
*Follow-up:* How do you catch a missing cleanup early? StrictMode runs effects twice in development, so a leaked timer shows up as double speed.

**Q10. Nested or flat state for a comment tree?**
Flat, normalized state (each comment has a `parentId`) turns adding, editing and deleting into simple array operations and avoids deep immutable copies; a memoized `parentId -> children` map drives the recursive rendering.
*Follow-up:* What stops the recursion? The base case: a `CommentList` whose parent has no children returns `null`.

**Q11. How would you test one of these components?**
React Testing Library with `user-event`: render it, interact as a user (type into the combobox, press ArrowDown and Enter) and assert on what the user sees through accessible queries such as `getByRole('option', { name: /lamp/i })`, with the network mocked by MSW.
*Follow-up:* How do you test the debounce? Use fake timers and advance them, or `findBy` queries that wait for the result to appear.

**Q12. You have 15 minutes left and two features unfinished. What do you do?**
Finish the one that completes the core flow, remove half-done code that breaks the app, and spend the last minutes explaining what is missing, how you would build it and how you would test it.
*Follow-up:* Is it bad to leave TODOs? Not if they are explicit and you mention them; a working subset with a clear plan beats two broken features.
