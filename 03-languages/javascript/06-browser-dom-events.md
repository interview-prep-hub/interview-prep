# Browser, DOM and Events

> **Tier:** T1 (know cold) · **Related:** [Async and the event loop](04-async-event-loop-promises.md), [Implement from scratch](07-implement-from-scratch.md), [How the web works](../../04-web-fundamentals/01-how-the-web-works.md), [HTML and CSS](../../04-web-fundamentals/02-html-css.md), [Authentication and authorization](../../04-web-fundamentals/06-authentication-authorization.md), [Web security (OWASP)](../../04-web-fundamentals/07-web-security-owasp.md), [Caching and performance](../../04-web-fundamentals/08-caching-and-performance.md)
> **On your resume:** web UIs in React and React on Rails (React components inside ERB views) at Devntech, the React front end of the PureLogics LMS, and the Web Development labs you assisted at PUCIT.

## Quick revision
- The DOM is a tree of objects built from HTML. `querySelector`/`querySelectorAll` (static NodeList), `getElementById`; `getElementsByClassName` returns a live HTMLCollection.
- `textContent` inserts text (safe, fast); `innerHTML` parses HTML (XSS risk with user input); `innerText` is layout-aware (forces layout).
- `addEventListener(type, fn, { capture, once, passive, signal })`. Removing needs the same function reference and the same `capture` flag.
- Phases: capturing (window down to the target), target, bubbling (back up). Listeners default to bubbling.
- `event.target` is where the event started; `event.currentTarget` is the element whose listener is running.
- `stopPropagation()` stops travel to other elements; `stopImmediatePropagation()` also skips other listeners on the same element; `preventDefault()` cancels the default action (submit, navigation) without stopping propagation.
- Delegation: one listener on a parent plus `event.target.closest(selector)`. Fewer listeners, and it covers elements added later.
- Debounce: run once after events stop for N ms (search, autosave). Throttle: run at most once per N ms (scroll, mousemove).
- Cookies (about 4 KB, sent with requests, can be HttpOnly), localStorage (about 5 MB, persistent, synchronous), sessionStorage (per tab), IndexedDB (large, async, structured).
- Tokens in localStorage are readable by any injected script. HttpOnly + Secure + SameSite cookies keep tokens away from JavaScript but need CSRF protection.
- `fetch` rejects only on network errors, CORS blocks and aborts; 404 and 500 resolve, so check `res.ok`. `res.json()` throws on non-JSON bodies.
- Same origin = same scheme, host and port. Servers opt in to CORS with `Access-Control-Allow-Origin`. PUT/PATCH/DELETE, JSON bodies and `Authorization` headers trigger an OPTIONS preflight.
- CORS is enforced only by browsers. Fix it on the server.
- Rendering: DOM + CSSOM, render tree, layout (reflow), paint, composite. Animate `transform`/`opacity`; batch reads before writes.
- `defer`: parallel download, runs in order after parsing. `async`: runs as soon as downloaded, any order. Module scripts are deferred by default.
- Web Workers run JavaScript off the main thread (no DOM, `postMessage`); Service Workers proxy the network (offline, push).
- `IntersectionObserver` reports visibility changes: lazy loading, infinite scroll. Better than scroll listeners.
- `history.pushState` changes the URL without reloading; `popstate` fires on back/forward; servers must return the app shell for deep links.
- Accessibility: semantic elements, labels, keyboard support, visible focus, alt text, ARIA only when HTML cannot express it.

## 1. DOM selection and manipulation

```js
const form = document.getElementById('signup-form');
const firstItem = document.querySelector('.todo-list li');   // first match or null
const items = document.querySelectorAll('.todo-list li');   // static NodeList
items.forEach((li) => li.classList.add('ready'));           // NodeList has forEach

const li = document.createElement('li');
li.textContent = 'Buy milk';            // treated as text: safe
li.dataset.id = '42';                   // <li data-id="42">
li.classList.toggle('done', false);
document.querySelector('.todo-list').append(li);
li.remove();

// Many insertions: build in a fragment, attach once
const fragment = document.createDocumentFragment();
for (const name of ['a', 'b', 'c']) {
  const item = document.createElement('li');
  item.textContent = name;
  fragment.append(item);
}
document.querySelector('.todo-list').append(fragment);
```

- `querySelectorAll` returns a snapshot; `getElementsByClassName` returns a live collection that changes under you inside loops that add or remove elements.
- `el.closest('.card')` walks up to the nearest matching ancestor (or the element itself); the core of delegation.
- Attributes vs properties: `input.getAttribute('value')` is the initial HTML value, `input.value` is what the user typed.

### textContent vs innerHTML vs innerText

| | Parses HTML? | Cost | Notes |
|---|---|---|---|
| `textContent` | no | cheap | includes hidden text; the default for user data |
| `innerText` | no | forces layout | respects CSS, so hidden text is excluded |
| `innerHTML` | yes | parses markup | XSS risk with untrusted input |

```js
const comment = '<img src=x onerror="alert(document.cookie)">';
el.innerHTML = comment;    // XSS: the onerror handler runs
el.textContent = comment;  // safe: shows the literal text
```

If you must render user HTML (rich text), sanitize it with a library such as DOMPurify. Frameworks escape by default: React escapes interpolated strings (`dangerouslySetInnerHTML` is the explicit escape hatch), and Rails ERB escapes `<%= %>` output (`raw` and `html_safe` bypass it). More in [web security](../../04-web-fundamentals/07-web-security-owasp.md).

## 2. addEventListener and its options

```js
const button = document.querySelector('#save');

function handleSave(event) {
  console.log('clicked', event.currentTarget.id);
}

button.addEventListener('click', handleSave);
button.removeEventListener('click', handleSave);   // must be the same function reference

button.addEventListener('click', handleSave, { once: true }); // removed after the first call

// passive: you promise not to call preventDefault, so scrolling never waits for your code
window.addEventListener('touchmove', () => { /* track the finger */ }, { passive: true });

// signal: remove several listeners at once (inline arrows included)
const controller = new AbortController();
window.addEventListener('resize', () => console.log('resize'), { signal: controller.signal });
window.addEventListener('scroll', () => console.log('scroll'), { signal: controller.signal });
controller.abort(); // both listeners are gone
```

- `removeEventListener('click', () => {})` never works: a new arrow is a different function.
- `el.onclick = fn` allows one handler and silently replaces the previous one; `addEventListener` allows many.
- Custom events: `el.dispatchEvent(new CustomEvent('cart:updated', { detail: { count: 3 }, bubbles: true }))`.

## 3. Capturing, target and bubbling

When you click a button inside a card, the event travels in three phases:
1. **Capturing:** from `window` down through `document`, `html`, `body` and the ancestors. Listeners added with `{ capture: true }` run here.
2. **Target:** listeners on the target itself run.
3. **Bubbling:** back up from the target to `window`. Normal listeners run here.

```html
<div id="card">
  <button id="buy">Buy</button>
</div>
```

```js
const card = document.querySelector('#card');
const buy = document.querySelector('#buy');

card.addEventListener('click', () => console.log('card capture'), { capture: true });
card.addEventListener('click', (event) => {
  console.log('card bubble', event.target.id, event.currentTarget.id); // 'buy' 'card'
});
buy.addEventListener('click', () => console.log('button'));

// Clicking the button logs: card capture, button, card bubble buy card
```

In the card's listener, `event.target` is still the button (where the click started) and `event.currentTarget` is the card. Not every event bubbles: `focus`, `blur`, `mouseenter` and `mouseleave` do not, so delegate with `focusin`/`focusout` or `mouseover`/`mouseout` instead.

## 4. stopPropagation vs preventDefault

| Method | Stops propagation to other elements | Skips other listeners on this element | Cancels the default action |
|---|---|---|---|
| `stopPropagation()` | yes | no | no |
| `stopImmediatePropagation()` | yes | yes | no |
| `preventDefault()` | no | no | yes (form submit, link navigation, checkbox toggle, context menu) |

```js
form.addEventListener('submit', async (event) => {
  event.preventDefault();                          // no full-page form submission
  const data = Object.fromEntries(new FormData(form));
  await fetch('/api/signup', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
});
```

Use `stopPropagation` sparingly: it breaks delegated handlers and analytics higher up (a dropdown that stops propagation can break someone else's "click outside to close"). `return false` from an `addEventListener` handler does nothing; it only cancels the default in `on*` property handlers, and in jQuery it does both.

## 5. Event delegation

One listener on a stable parent instead of one per item:

```js
const list = document.querySelector('#todo-list');

list.addEventListener('click', (event) => {
  const deleteButton = event.target.closest('[data-action="delete"]');
  if (!deleteButton || !list.contains(deleteButton)) return; // the click was elsewhere
  deleteButton.closest('li').remove();
});
// One listener handles 1,000 items, including items added later.
```

Benefits: less memory, no re-binding when items change, simpler cleanup. Caveats: bubbling events only; `event.target` may be a nested child (an icon inside the button), hence `closest`; check containment in nested structures. React uses the same idea: it attaches its listeners to the root container and dispatches to your components. Legacy Rails pages often use jQuery's delegated form, `$(document).on('click', '.btn', handler)`, so handlers survive Turbo or Turbolinks page swaps.

## 6. Debounce vs throttle

| | Debounce | Throttle |
|---|---|---|
| Rule | run once after events stop for `wait` ms | run at most once every `wait` ms while events continue |
| During continuous typing | nothing until a pause, then one call | regular calls throughout |
| Use cases | search-as-you-type, autosave, validation, resize end | scroll position, infinite-scroll checks, mousemove, drag, rate-limiting a button |

```js
function debounce(fn, wait) {
  let timerId;
  return function (...args) {
    clearTimeout(timerId);                                // each call restarts the timer
    timerId = setTimeout(() => fn.apply(this, args), wait);
  };
}

const searchInput = document.querySelector('#search');
searchInput.addEventListener(
  'input',
  debounce((event) => console.log('search for', event.target.value), 300),
);
```

Versions with `leading`, `cancel` and a trailing-call throttle, with tests, are in [implement from scratch](07-implement-from-scratch.md). In React, create the debounced function once (`useMemo` or `useRef`); recreating it every render resets the timer and it may never fire.

## 7. Browser storage and token security

| | Cookies | localStorage | sessionStorage | IndexedDB |
|---|---|---|---|---|
| Capacity | about 4 KB per cookie | about 5 MB per origin (varies by browser) | about 5 MB per origin | large, quota-based |
| Sent with HTTP requests | yes, automatically (matching domain and path) | no | no | no |
| Readable by JavaScript | yes, unless `HttpOnly` | yes | yes | yes |
| Lifetime | session, or `Expires`/`Max-Age` | until cleared | until the tab closes | until cleared |
| Scope | domain and path | origin | origin and tab | origin |
| API | `document.cookie` string or server `Set-Cookie` | synchronous string key/value | synchronous string key/value | asynchronous, structured data, indexes, transactions |
| Typical use | sessions, auth, server-read preferences | UI preferences, small caches | per-tab wizard state | offline data, large caches, files |

**Where should tokens live?**
- **localStorage:** any script on your page can read it, so one XSS bug leaks every token. Simple, but the weakest option.
- **`HttpOnly; Secure; SameSite=Lax` (or `Strict`) cookie:** invisible to JavaScript, sent only over HTTPS, and SameSite blocks most cross-site sends. Because the browser attaches it automatically, protect state-changing requests from CSRF (SameSite plus CSRF tokens, which Rails includes by default).
- **Common SPA pattern:** short-lived access token in memory, refresh token in an HttpOnly cookie.
- **React Native:** there is no localStorage; keep tokens in the OS keychain or keystore via `expo-secure-store`, not plain AsyncStorage (see [state, storage, auth and offline](../../08-react-native-expo/04-state-storage-auth-offline.md)).

The full discussion is in [authentication and authorization](../../04-web-fundamentals/06-authentication-authorization.md).

```js
localStorage.setItem('theme', 'dark');                         // strings only
localStorage.setItem('filters', JSON.stringify({ city: 'Lahore' }));
const filters = JSON.parse(localStorage.getItem('filters') ?? '{}');
// A 'storage' event fires in OTHER tabs of the same origin when localStorage changes.
```

## 8. fetch and error handling

`fetch` returns a promise that **resolves with a Response for every HTTP status**, 404 and 500 included. It rejects only for network failures, CORS blocks, aborts and malformed requests.

```js
async function request(url, { body, headers, ...options } = {}) {
  const res = await fetch(url, {
    ...options,
    headers: { 'Content-Type': 'application/json', ...headers },
    body: body === undefined ? undefined : JSON.stringify(body),
  });

  const isJson = res.headers.get('content-type')?.includes('application/json');
  const data = isJson ? await res.json() : await res.text(); // a body can be read only once

  if (!res.ok) {
    const error = new Error(data?.message ?? `HTTP ${res.status}`);
    error.status = res.status;  // 401: refresh or log out; 422: show field errors
    error.data = data;
    throw error;
  }
  return data;
}

async function createUser() {
  try {
    return await request('/api/users', { method: 'POST', body: { name: 'Sara' } });
  } catch (err) {
    if (err.status === 422) console.warn('validation errors', err.data);
    else if (err instanceof TypeError) console.warn('network failure or CORS block');
    else throw err;
  }
}
```

- `res.json()` rejects if the body is not valid JSON, for example an HTML error page from a proxy or an empty 204 response; check status and content type first.
- Cookies: same-origin requests send them by default (`credentials: 'same-origin'`); cross-origin requests need `credentials: 'include'` plus server CORS support.
- `fetch` has no default timeout; use `AbortSignal.timeout(ms)` (see [async](04-async-event-loop-promises.md)).
- axios differs: it rejects on non-2xx by default, parses JSON for you and has interceptors (handy for attaching and refreshing tokens).

## 9. CORS and preflight from the browser side

The **same-origin policy** stops a page from reading responses from another origin (scheme + host + port). `http://localhost:5173` calling `http://localhost:3000` is cross-origin because the ports differ.

**CORS** lets the server opt in with response headers:
- `Access-Control-Allow-Origin: https://app.example.com` (or `*` for public APIs without credentials).
- `Access-Control-Allow-Credentials: true` when cookies are involved; the origin then cannot be `*`.
- `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` in preflight responses; `Access-Control-Max-Age` caches the preflight.

**Preflight:** for non-simple requests the browser first sends `OPTIONS` with `Access-Control-Request-Method` and `Access-Control-Request-Headers`, and sends the real request only if the answer allows it. Triggers: methods other than GET, HEAD and POST; a `Content-Type` other than `text/plain`, `multipart/form-data` or `application/x-www-form-urlencoded` (so JSON triggers it); custom headers such as `Authorization`.

Debugging facts:
- Browsers enforce CORS; curl, Postman and server-to-server calls ignore it. "Works in Postman, fails in the browser" is almost always CORS.
- Your code only sees `TypeError: Failed to fetch`; the real reason is in the console and in the Network tab (inspect the OPTIONS request).
- A simple request still reaches the server (only the response is hidden), so CORS is not server-side protection.
- Fix it on the server: `rack-cors` in Rails, the `cors` middleware in Express, `hono/cors` in Hono, or a same-origin dev proxy (Vite's `server.proxy`). Never "fix" it by disabling browser security.

## 10. Critical rendering path, reflow and repaint

1. Parse HTML into the **DOM** and CSS into the **CSSOM**.
2. Combine them into the **render tree** (visible nodes with computed styles).
3. **Layout (reflow):** compute every box's size and position.
4. **Paint:** fill in pixels (text, colors, borders, shadows).
5. **Composite:** assemble layers, often on the GPU.

CSS is render-blocking and classic scripts are parser-blocking, which is why CSS goes in the `<head>` and scripts get `defer`.

| Change | Triggers | Examples |
|---|---|---|
| Geometry | layout, paint, composite | `width`, `height`, `margin`, `top`, adding or removing nodes, font changes |
| Visual only | paint, composite | `color`, `background`, `visibility`, `box-shadow` |
| Compositor only | composite | `transform`, `opacity` (on their own layer) |

**Layout thrashing:** reading a layout property (`offsetHeight`, `getBoundingClientRect()`, `scrollTop`) right after a style write forces a synchronous layout; in a loop, that is one layout per iteration.

```js
const boxes = [...document.querySelectorAll('.box')];

// BAD: write, read, write, read... one forced layout per iteration
for (const box of boxes) {
  box.style.width = `${box.offsetWidth + 10}px`;
}

// GOOD: all reads first, then all writes in the next frame
const widths = boxes.map((box) => box.offsetWidth);
requestAnimationFrame(() => {
  boxes.forEach((box, i) => {
    box.style.width = `${widths[i] + 10}px`;
  });
});
```

## 11. Script loading: async vs defer

| | Download | Blocks parsing? | Executes | Order |
|---|---|---|---|---|
| `<script src>` | blocking | yes | immediately when reached | document order |
| `<script async src>` | parallel | only while executing | as soon as downloaded | whichever arrives first |
| `<script defer src>` | parallel | no | after parsing, before `DOMContentLoaded` | document order |
| `<script type="module" src>` | parallel | no | deferred by default | document order |

Use `defer` (or modules) for app code that needs the DOM or depends on other scripts, and `async` for independent scripts such as analytics. `DOMContentLoaded` fires when the HTML is parsed and deferred scripts have run; `load` fires after images, stylesheets and other subresources finish too.

## 12. Web Workers and Service Workers

```js
// main.js
const worker = new Worker(new URL('./parse-worker.js', import.meta.url), { type: 'module' });
worker.postMessage(csvText);                       // the data is copied (structured clone)
worker.onmessage = (event) => console.log('rows:', event.data.length);

// parse-worker.js
self.onmessage = (event) => {
  const rows = event.data.split('\n').map((line) => line.split(','));
  self.postMessage(rows);                          // workers have no DOM access
};
```

| | Web Worker | Service Worker |
|---|---|---|
| Purpose | CPU-heavy JavaScript off the main thread | a proxy between the page and the network |
| Lifetime | tied to the page that created it | independent of pages, event-driven |
| Typical use | parsing, image processing, heavy calculations | offline caching, background sync, push notifications (PWAs) |
| DOM access | no | no |

## 13. IntersectionObserver

Reports when elements enter or leave the viewport (or another container) without scroll listeners:

```js
const sentinel = document.querySelector('#load-more-sentinel');

const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      if (entry.isIntersecting) console.log('load the next page'); // infinite scroll trigger
    }
  },
  { rootMargin: '200px' },   // fire 200px before the sentinel becomes visible
);
observer.observe(sentinel);
// observer.disconnect() when done, for example when the component unmounts
```

Uses: infinite scroll, lazy-loading images and components, "seen" analytics, pausing off-screen videos. For images, the native `loading="lazy"` attribute is often enough.

## 14. History API and SPA routing

```js
function renderRoute(path) {
  document.querySelector('#app').textContent = `Current page: ${path}`;
}

// Navigate without a full page load
history.pushState({ page: 'orders' }, '', '/orders?status=paid');
renderRoute(location.pathname);

// Back and forward fire popstate (pushState itself does not)
window.addEventListener('popstate', () => renderRoute(location.pathname));

// replaceState rewrites the current entry, for example to sync filters into the URL
history.replaceState(null, '', '/orders?status=pending');
```

SPA routers (React Router, TanStack Router) intercept link clicks, call `pushState`, render the matching route and listen for `popstate`. Two requirements: the server must return `index.html` for every client route, or refreshing `/orders/42` gives a 404; and you must manage scroll position and focus yourself. Hash routing (`/#/orders`) avoids the server configuration at the cost of uglier URLs. See [React routing](../../06-react/08-routing.md) and [TanStack Router](../../07-nextjs-and-tanstack/04-tanstack-router.md).

## 15. Accessibility basics

- **Semantic HTML first:** `<button>` for actions, `<a href>` for navigation, `<nav>`, `<main>`, headings in order. A `<div onclick>` is not focusable, has no role and ignores Enter and Space.
- **Labels:** every input needs a `<label for>` (or `aria-label`); a placeholder is not a label.
- **Images:** meaningful `alt` text; `alt=""` for decorative images.
- **Keyboard:** everything works with Tab, Enter, Space and Escape; visible focus styles; move focus into modals and back out when they close.
- **ARIA only when HTML cannot express it:** `aria-expanded` on a disclosure button, `aria-live="polite"` for toasts, `role="dialog"` with `aria-modal`. Wrong ARIA is worse than none.
- **Color:** sufficient contrast, and never color alone to convey errors.
- **Testing:** a keyboard-only pass, Lighthouse or axe audits, a screen reader (VoiceOver, NVDA). React Native has its own props such as `accessibilityLabel` and `accessibilityRole`.

## Common pitfalls
- `innerHTML` with user data (XSS); use `textContent` or sanitize.
- `removeEventListener` with a new function reference, so the listener stays.
- One listener per item instead of delegation, or never removing listeners when a component unmounts.
- Calling `preventDefault` in a passive listener (ignored with a warning).
- Overusing `stopPropagation` and breaking parent handlers.
- Assuming `fetch` rejects on 404/500; calling `res.json()` on an HTML error page or a 204.
- JWTs in localStorage without thinking about XSS; cookies without CSRF protection.
- Trying to fix CORS in the frontend instead of the server.
- Layout thrashing in loops; animating `top`/`left`/`width` instead of `transform`.
- Blocking `<script>` tags in the `<head>` without `defer`.
- SPA deep links returning 404 because the server has no fallback.
- Clickable `div`s, missing labels and invisible focus.

## How to talk about this in an interview
- Frameworks hide the DOM, so interviewers check that you understand what happens underneath: explain bubbling and delegation, then connect them to how React attaches listeners at the root.
- Connect to real code: React components mounted inside Rails ERB views with React on Rails, forms in the PureLogics LMS front end, and the browser side of Next.js apps.
  > **Fill in:** a CORS, cookie or token-storage decision you dealt with (Rails API plus a separate front end, the Next.js APIs at RemoteDev, the LMS with DRF and React).
  > **Fill in:** a UI performance or accessibility fix you made (debounced search, long list, keyboard support for a modal).
- For token storage, present the trade-off rather than a rule: localStorage is simple but exposed to XSS, HttpOnly cookies need CSRF protection, mobile apps use secure storage.

## Interview questions

**Q1. Explain event bubbling and capturing.**
An event travels from `window` down to the target (capturing), fires on the target, then travels back up (bubbling). Listeners run during bubbling by default; `{ capture: true }` runs them during capturing. Bubbling is what makes delegation possible.
*Follow-up:* Which events do not bubble? `focus`, `blur`, `mouseenter`, `mouseleave`; use `focusin`/`focusout` for delegation.

**Q2. What is event delegation, and why use it?**
One listener on a common ancestor that uses `event.target.closest(selector)` to find which child was acted on. It saves memory, covers elements added later automatically and simplifies cleanup.
*Follow-up:* What can go wrong? `event.target` may be a nested child (use `closest`), and non-bubbling events cannot be delegated.

**Q3. What is the difference between `stopPropagation` and `preventDefault`?**
`stopPropagation` stops the event from reaching listeners on other elements; `preventDefault` cancels the browser's default action (form submit, navigation) while propagation continues. `stopImmediatePropagation` also skips the remaining listeners on the current element.
*Follow-up:* Does `return false` work with `addEventListener`? No, only in `on*` property handlers and jQuery.

**Q4. What is the difference between `event.target` and `event.currentTarget`?**
`target` is where the event originated (possibly a deeply nested child); `currentTarget` is the element whose listener is running. In a delegated listener on a list, `target` might be an icon while `currentTarget` is the list.
*Follow-up:* What is `this` in a regular-function listener? The same element as `currentTarget`.

**Q5. `textContent` vs `innerHTML`: why does it matter for security?**
`innerHTML` parses the string as HTML, so input like `<img onerror=...>` runs script (XSS); `textContent` inserts plain text and is faster. If you need user HTML, sanitize it with a library such as DOMPurify.
*Follow-up:* How do React and Rails protect you? Both escape interpolated output by default; `dangerouslySetInnerHTML`, `raw` and `html_safe` opt out.

**Q6. Debounce vs throttle: when do you use each?**
Debounce waits until events stop for N ms, then runs once (search-as-you-type, autosave, resize end). Throttle runs at most once every N ms during continuous events (scroll, mousemove, drag). Both cap how often expensive work runs.
*Follow-up:* How do you implement debounce? Clear and restart a timer kept in a closure on every call.

**Q7. Compare cookies, localStorage and sessionStorage. Where would you store a JWT?**
Cookies are small, sent automatically with requests and can be HttpOnly; localStorage persists per origin with a synchronous API; sessionStorage is per tab and cleared on close. Prefer an HttpOnly, Secure, SameSite cookie with CSRF protection, or an in-memory access token plus an HttpOnly refresh cookie; localStorage is exposed to any XSS.
*Follow-up:* And in React Native? `expo-secure-store` (Keychain/Keystore), not plain AsyncStorage.

**Q8. Does `fetch` reject on a 404 or 500? How do you handle errors properly?**
No: it resolves with a Response for every status and rejects only on network errors, CORS blocks and aborts. Check `res.ok`, parse the body carefully (`res.json()` fails on HTML or empty bodies), and throw an error with the status so the UI can react.
*Follow-up:* How is axios different? It rejects on non-2xx responses by default.

**Q9. What is CORS, and what triggers a preflight request?**
Browsers block reading cross-origin responses unless the server allows it with `Access-Control-Allow-*` headers. A preflight `OPTIONS` request precedes non-simple requests: methods other than GET/HEAD/POST, JSON or other non-simple content types, and custom headers like `Authorization`.
*Follow-up:* Why does the same request work in Postman? Only browsers enforce CORS.

**Q10. What is the difference between `async` and `defer` on script tags?**
Both download without blocking the parser. `async` runs the script as soon as it arrives, in any order, possibly mid-parse; `defer` runs scripts after parsing, in document order, before `DOMContentLoaded`. Use `defer` for app code and `async` for independent scripts like analytics.
*Follow-up:* What about module scripts? Deferred by default.

**Q11. Reflow vs repaint: how do you avoid layout thrashing?**
Reflow (layout) recomputes sizes and positions after geometry changes; repaint redraws pixels after visual-only changes; `transform` and `opacity` can skip both and only composite. Batch DOM reads before writes, use `requestAnimationFrame`, and animate `transform`/`opacity`.
*Follow-up:* Which reads force layout? `offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `clientWidth` and similar.

**Q12. What are Web Workers, and how do they differ from Service Workers?**
Web Workers run JavaScript on a background thread for CPU-heavy work, talk through `postMessage` and cannot touch the DOM. Service Workers act as a network proxy for the origin, enabling offline caching, background sync and push, and live independently of any page.
*Follow-up:* How is data passed to a worker? Copied with the structured clone algorithm, or transferred for ArrayBuffers.

**Q13. How would you implement infinite scroll or lazy loading?**
Observe a sentinel at the end of the list with `IntersectionObserver` (a `rootMargin` prefetches early) and load the next page when it intersects. Guard against duplicate requests while one is in flight and disconnect the observer on unmount. For images, `loading="lazy"` often suffices.
*Follow-up:* Why not a scroll listener? It fires constantly on the main thread and needs throttling plus layout reads.

**Q14. How does client-side routing work?**
The router intercepts link clicks, updates the URL with `history.pushState` (no reload), renders the matching component, and listens to `popstate` for back and forward. The server must return `index.html` for every client route so deep links and refreshes work.
*Follow-up:* What is hash routing? Routes after `#`, which never reach the server, so no fallback is needed.

**Q15. What makes a page accessible? Why is `<button>` better than a clickable `<div>`?**
Semantic HTML, labels, alt text, keyboard support, visible focus, enough contrast and ARIA only where needed. A `<button>` is focusable, announced as a button and activated by Enter and Space for free; a `<div>` needs `tabindex`, a role and key handlers to imitate it.
*Follow-up:* How do you test? Keyboard-only navigation, Lighthouse or axe, and a screen reader.

**Q16. Walk me through the critical rendering path.**
HTML becomes the DOM, CSS becomes the CSSOM, both combine into the render tree, layout computes geometry, paint fills pixels and compositing assembles layers. CSS blocks rendering and synchronous scripts block parsing, so inline critical CSS, defer scripts and keep the DOM lean.
*Follow-up:* Which metric measures when the main content appears? Largest Contentful Paint (LCP).

**Q17. What do the `passive`, `once` and `signal` listener options do?**
`passive: true` promises no `preventDefault`, so scrolling never waits for your handler (important for touch and wheel). `once: true` removes the listener after its first call. `signal` ties the listener to an `AbortController`, so `abort()` removes it, handy for removing many listeners at once.
*Follow-up:* What happens if you call `preventDefault` in a passive listener? It is ignored and the browser logs a warning.

**Q18. What is the difference between `DOMContentLoaded` and `load`?**
`DOMContentLoaded` fires when the HTML is parsed and deferred scripts have run; `load` fires later, after images, stylesheets, iframes and other subresources finish. Initialize UI on `DOMContentLoaded` (or use `defer`) so the page becomes interactive sooner.
*Follow-up:* What is the equivalent in Rails apps using Turbo? `turbo:load`, which fires on the first load and after every Turbo visit.

## Further reading
- [MDN: EventTarget.addEventListener()](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
