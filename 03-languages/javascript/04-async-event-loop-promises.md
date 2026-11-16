# Async JavaScript: Event Loop, Promises and async/await

> **Tier:** T1 (know cold) · **Related:** [Scope and closures](02-scope-closures-hoisting.md), [Browser, DOM and events](06-browser-dom-events.md), [Implement from scratch](07-implement-from-scratch.md), [Node.js fundamentals](../../09-nodejs-backend/01-nodejs-fundamentals.md), [Data fetching with TanStack Query](../../06-react/04-data-fetching-tanstack-query.md)
> **On your resume:** every API call in your React, React Native (Expo) and Next.js work is async: screen data loading, push notification handlers, payment flows, and the Next.js + MongoDB APIs at RemoteDev.

## Quick revision
- Your code runs on one thread with one call stack. The host (browser APIs or libuv in Node) does the waiting for timers, network and I/O, then queues callbacks.
- Event loop: run one task to completion, drain **all** microtasks, (browser) render if it is time, repeat.
- Microtasks: promise reactions (`then`/`catch`/`finally`), `await` continuations, `queueMicrotask`, `MutationObserver`. Tasks: `setTimeout`, `setInterval`, I/O, UI events, `MessageChannel`, `setImmediate` (Node).
- Order: sync code, then microtasks, then the next task. `Promise.resolve().then(a)` beats `setTimeout(b, 0)`.
- `setTimeout(fn, 0)` means "no sooner than 0 ms, after the current task and its microtasks"; browsers clamp nested timers to about 4 ms.
- `requestAnimationFrame` runs right before the next paint (paused in background tabs): use it for visual updates.
- Node: `process.nextTick` runs before promise microtasks; `setImmediate` runs in the check phase after I/O.
- A promise is pending, then fulfilled or rejected exactly once. The executor runs synchronously; reactions always run later as microtasks.
- `then` returns a new promise: return a value and the next step gets it; throw and the next step rejects; return a promise and the chain waits.
- A rejection skips fulfillment handlers until the nearest `catch`. `finally` gets no argument and passes the result through.
- `Promise.all` fails fast; `allSettled` never rejects; `race` takes the first to settle; `any` takes the first success (`AggregateError` if all fail).
- `async` functions always return a promise; `await` pauses only that function, never the thread.
- Back-to-back awaits of independent work are sequential. Start them together with `Promise.all`.
- `forEach` does not wait for async callbacks: use `for...of` (sequential) or `map` + `Promise.all` (parallel).
- Unhandled rejections crash Node by default (since Node 15) and fire `unhandledrejection` in browsers. Never leave a promise floating.
- `AbortController` cancels `fetch` (rejects with an `AbortError`); abort in effect cleanups to avoid race conditions.

## 1. The runtime: stack, heap, host APIs and queues

- **Call stack:** one frame per running function. One stack means one thing at a time.
- **Heap:** where objects live.
- **Host APIs:** timers, networking and DOM events in browsers; libuv with the OS and a thread pool in Node. They wait off the main thread.
- **Task queues:** callbacks that are ready (a timer fired, a click happened, a response arrived).
- **Microtask queue:** promise reactions and similar high-priority callbacks, emptied completely after every task.
- **Rendering (browsers):** style, layout and paint, about every 16.7 ms on a 60 Hz screen, and only between tasks.

## 2. The event loop step by step

1. Take the oldest runnable task and run it until the call stack is empty. Running your main script is the first task.
2. Run every microtask in the queue. Microtasks queued during this step run too, before moving on.
3. If the browser decides it is time to render: run `requestAnimationFrame` callbacks, recalculate styles, lay out, paint.
4. Go back to step 1, idling until a task arrives.

What follows from this:
- A long synchronous job (parsing a huge JSON payload, a heavy loop) blocks clicks, timers and painting: the page freezes.
- A microtask that keeps scheduling microtasks starves tasks and rendering just as badly.
- Promise callbacks always run before the next timer, even a 0 ms one.
- Browsers clamp timers nested more than five levels deep to at least 4 ms and throttle timers in background tabs.

| Mechanism | Queue | Runs | Use it for |
|---|---|---|---|
| `setTimeout(fn, 0)`, `setInterval` | task | a later loop turn | deferring work, yielding to the browser |
| `Promise.then`, `await` continuation | microtask | right after the current task or microtask | async control flow |
| `queueMicrotask(fn)` | microtask | same as promise reactions | a tiny follow-up without creating a promise |
| `requestAnimationFrame(fn)` | animation frame callbacks | just before the next paint | animations, one layout read/write per frame |
| `process.nextTick(fn)` (Node) | next-tick queue | after the current operation, before promise microtasks | library internals, emitting after a constructor returns |
| `setImmediate(fn)` (Node) | check phase | after the I/O poll phase | running right after pending I/O callbacks |

One Node detail worth a sentence: in the main module, the order of `setTimeout(fn, 0)` and `setImmediate` is not guaranteed, but inside an I/O callback `setImmediate` always runs first. The libuv phases are in [Node.js fundamentals](../../09-nodejs-backend/01-nodejs-fundamentals.md).

## 3. Callbacks and callback hell

Before promises, async APIs took callbacks. Node uses the error-first convention:

```js
const fs = require('node:fs');

fs.readFile('config.json', 'utf8', (err, text) => {
  if (err) return console.error('read failed', err);
  console.log(JSON.parse(text));
});
```

Dependent steps nest into the "pyramid of doom":

```js
getUser(id, (err, user) => {
  if (err) return handle(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handle(err);
    getInvoice(orders[0].id, (err, invoice) => {
      if (err) return handle(err);
      sendEmail(user.email, invoice, (err) => {
        if (err) return handle(err);
        console.log('done');
      });
    });
  });
});
```

The problems go beyond indentation: every level must check errors, `try/catch` cannot catch errors thrown inside later callbacks, and you suffer **inversion of control** (you trust a library to call you exactly once, not twice or never). Promises fix this with an object that settles once and composes:

```js
async function sendLatestInvoice(id) {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  const invoice = await getInvoice(orders[0].id);
  await sendEmail(user.email, invoice);
}
```

To convert callback APIs, use `util.promisify` in Node, the promise versions of core modules (`node:fs/promises`), or wrap them yourself (see [promisify](07-implement-from-scratch.md)).

## 4. Promises

A promise represents a future result. It starts **pending** and settles exactly once, as **fulfilled** (with a value) or **rejected** (with a reason).

```js
const loadUser = (id) =>
  new Promise((resolve, reject) => {
    // The executor runs synchronously, right now
    if (!id) return reject(new Error('id required'));
    setTimeout(() => resolve({ id, name: 'Sara' }), 100);
  });

loadUser(1)
  .then((user) => user.name)              // return a value: the next then receives it
  .then((name) => {
    if (name === 'Sara') throw new Error('demo failure'); // throw: the next promise rejects
    return name;
  })
  .catch((err) => {
    console.error(err.message);           // 'demo failure'
    return 'fallback';                    // recover: the chain continues fulfilled
  })
  .finally(() => console.log('done'));    // no argument; the previous result passes through
```

Chaining rules:
- `then(onFulfilled, onRejected)` returns a **new** promise, resolved with whatever the handler returns.
- Returning a promise (or any thenable) makes the next step wait for it and adopt its outcome.
- Throwing, or returning a rejected promise, rejects the next promise.
- A rejection skips fulfillment handlers until it meets a `catch` (or a `then` with a second argument).
- `catch(fn)` is `then(undefined, fn)`. `finally(fn)` runs either way and keeps the original outcome unless it throws or returns a rejected promise.
- Calling `resolve` or `reject` after settling is ignored.
- `.then(ok, fail)` differs from `.then(ok).catch(fail)`: only the second catches errors thrown inside `ok`.

Common mistakes:

```js
// 1. Forgetting to return: the chain does not wait, and the next step gets undefined
fetchUser()
  .then((user) => { fetchOrders(user.id); })
  .then((orders) => console.log(orders)); // undefined

// 2. Nesting thens instead of returning the inner promise (callback hell again)

// 3. Wrapping something that already is a promise (the explicit-construction anti-pattern)
const bad = () => new Promise((resolve, reject) => fetchUser().then(resolve).catch(reject));
const good = () => fetchUser();
```

## 5. Promise combinators

| Method | Fulfills when | Rejects when | Result | Empty input |
|---|---|---|---|---|
| `Promise.all` | all fulfill | first rejection (fail fast) | values in input order | fulfills with `[]` |
| `Promise.allSettled` | all settle | never | `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }` per input | fulfills with `[]` |
| `Promise.race` | first to settle fulfills | first to settle rejects | that first outcome | pending forever |
| `Promise.any` | first fulfills | all reject | first value, or an `AggregateError` holding every reason | rejects with `AggregateError` |

```js
// Need every result
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);

// Independent batch work: report each outcome
const results = await Promise.allSettled(listingIds.map(syncListing));
const failed = results.filter((r) => r.status === 'rejected');

// Timeout
const timeout = (ms) =>
  new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms));
const prices = await Promise.race([fetchPrices(), timeout(5000)]);

// Redundancy: first success wins
const config = await Promise.any([fetchFromMirrorA(), fetchFromMirrorB()]);
```

None of them cancel the remaining operations: the requests keep running. Pair them with `AbortController` when cancellation matters. Non-promise values in the input count as already fulfilled.

## 6. async/await

`async` makes a function return a promise. `await` suspends that function until the promise settles and resumes it in a microtask; everything else keeps running.

```js
async function getProfile(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();   // `return await` inside try so a rejection is caught below
  } catch (err) {
    console.error('profile failed', err);
    throw err;                 // rethrow so the caller can react
  } finally {
    console.log('request finished');
  }
}
```

- Returning fulfills the function's promise; throwing (even before the first `await`) rejects it.
- `await` on a non-promise wraps the value in a resolved promise, so it still yields one microtask.
- `return await` matters only inside `try/catch/finally`; elsewhere it behaves like `return promise`.

### Sequential vs parallel

```js
// Sequential: about 2 s if each call takes 1 s
const user = await getUser(id);
const settings = await getSettings(id);

// Parallel: about 1 s, both requests start immediately
const [user2, settings2] = await Promise.all([getUser(id), getSettings(id)]);
```

Await sequentially only when a step needs the previous result (`getOrders(user.id)`).

### await in loops

```js
// BUG: forEach ignores the promises; 'done' logs before any upload finishes
files.forEach(async (file) => {
  await upload(file);
});
console.log('done');

// Sequential: one at a time (ordering, rate limits)
for (const file of files) {
  await upload(file);
}

// Parallel: all at once
await Promise.all(files.map((file) => upload(file)));

// Parallel with a limit (say 3 at a time): use a promise pool (see the implementations file)
```

`for await...of` consumes async iterables, such as paginated APIs or streams:

```js
async function* fetchAllPages(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const page = await res.json();
    yield page.items;
    next = page.nextUrl; // null on the last page
  }
}

for await (const items of fetchAllPages('/api/listings?page=1')) {
  console.log('got', items.length);
}
```

**Top-level await** (ES2022) works only in ES modules, not CommonJS. It pauses evaluation of that module and every module importing it, so keep it to startup work such as loading configuration.

## 7. Error handling and unhandled rejections

- End chains with `.catch()`; in async functions, wrap the awaits you can actually handle in `try/catch`.
- A rejected promise with no handler is an **unhandled rejection**. Browsers log it and fire `unhandledrejection` on `window`; Node (since version 15) crashes the process by default, so one forgotten `catch` in an API route can take the server down.
- "Floating" promises (called without `await` or `catch`) are the usual cause; typescript-eslint's `no-floating-promises` rule finds them.
- Errors in async React event handlers are not caught by error boundaries; catch them and set error state.

```js
// Last-resort logging only, never a substitute for real handling
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled rejection:', event.reason);
});
// Node equivalent: process.on('unhandledRejection', (reason) => { ... })
```

## 8. Cancellation with AbortController

```js
const controller = new AbortController();

fetch('/api/search?q=pika', { signal: controller.signal })
  .then((res) => res.json())
  .then((data) => console.log(data))
  .catch((err) => {
    if (err.name === 'AbortError') return; // expected: we cancelled it
    throw err;
  });

controller.abort(); // fetch rejects with a DOMException named 'AbortError'

// Built-in timeout signal (modern browsers and recent Node versions)
fetch('/api/prices', { signal: AbortSignal.timeout(5000) }); // rejects with 'TimeoutError'
```

**Race conditions in effects:** when the user types quickly, an older request can resolve after a newer one and overwrite fresh results. Abort the previous request in the cleanup:

```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setResults)
    .catch((err) => {
      if (err.name !== 'AbortError') setError(err);
    });
  return () => controller.abort(); // runs before the next effect and on unmount
}, [query]);
```

TanStack Query does this for you by passing a `signal` to your query function (see [data fetching with TanStack Query](../../06-react/04-data-fetching-tanstack-query.md)). A signal can also remove event listeners: `addEventListener('scroll', fn, { signal })`.

## 9. Output-ordering exercises

Method: (1) run all synchronous code, noting what gets queued and where; (2) drain microtasks in order, appending new ones to the back; (3) run the next task and drain microtasks again.

### Exercise 1: the basics
```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
console.log('D');
```
**Output:** A, D, C, B. A and D are synchronous. C is a microtask, so it runs as soon as the script task ends. B is a task and waits for a later loop turn.

### Exercise 2: the executor is synchronous
```js
console.log(1);
new Promise((resolve) => {
  console.log(2);
  resolve();
  console.log(3); // resolve does not stop the executor
}).then(() => console.log(4));
console.log(5);
```
**Output:** 1, 2, 3, 5, 4. The executor runs immediately (2 and 3); `resolve()` only queues the reaction, which runs after the remaining sync code (5).

### Exercise 3: await splits a function
```js
async function task() {
  console.log('task start');
  await null;
  console.log('task end');
}
console.log('script start');
task();
console.log('script end');
```
**Output:** script start, task start, script end, task end. Code before the first `await` runs synchronously; the rest is a continuation queued as a microtask.

### Exercise 4: microtasks drain after every task
```js
setTimeout(() => {
  console.log('t1');
  Promise.resolve().then(() => console.log('p1'));
}, 0);
setTimeout(() => console.log('t2'), 0);
```
**Output:** t1, p1, t2. After the first timer task, the microtask queue is drained (p1) before the second timer runs. (Node versions before 11 printed t1, t2, p1.)

### Exercise 5: chains interleave
```js
Promise.resolve()
  .then(() => console.log('a1'))
  .then(() => console.log('a2'));
Promise.resolve()
  .then(() => console.log('b1'))
  .then(() => console.log('b2'));
```
**Output:** a1, b1, a2, b2. Only a1 and b1 are queued at first. Running a1 fulfills its `then` promise, which queues a2 behind b1. Each chain advances one step per microtask.

### Exercise 6: everything together
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
queueMicrotask(() => console.log('3'));
Promise.resolve().then(() => {
  console.log('4');
  setTimeout(() => console.log('5'), 0);
});
(async () => {
  console.log('6');
  await undefined;
  console.log('7');
})();
console.log('8');
```
**Output:** 1, 6, 8, 3, 4, 7, 2, 5.
- Sync: 1; timer 2 queued; microtasks 3 and 4 queued; the async IIFE logs 6 and queues its continuation (7); 8.
- Microtasks in order: 3, then 4 (which queues timer 5 behind timer 2), then 7.
- Tasks: 2, then 5.

### Exercise 7: async functions calling async functions
```js
async function a1() {
  console.log('a1 start');
  await a2();
  console.log('a1 end');
}
async function a2() {
  console.log('a2');
}
console.log('script start');
setTimeout(() => console.log('setTimeout'), 0);
a1();
new Promise((resolve) => {
  console.log('promise1');
  resolve();
}).then(() => console.log('promise2'));
console.log('script end');
```
**Output:** script start, a1 start, a2, promise1, script end, a1 end, promise2, setTimeout.
- `a1()` logs `a1 start` and calls `a2()`, which logs `a2` synchronously and returns an already fulfilled promise, so `a1 end` is queued as a microtask.
- The executor logs `promise1`; `promise2` is queued behind `a1 end`.
- After `script end`: microtasks `a1 end` and `promise2`, then the timer task.

### Exercise 8: error propagation
```js
Promise.reject(new Error('boom'))
  .then(() => console.log('then 1'))
  .catch((err) => {
    console.log('catch', err.message);
    return 'recovered';
  })
  .then((value) => console.log('then 2', value))
  .finally(() => console.log('finally'));
```
**Output:** catch boom, then 2 recovered, finally. The rejection skips `then 1` (no rejection handler) and lands in `catch`; returning a value recovers the chain, so `then 2` receives `'recovered'`; `finally` runs last with no argument.

## Common pitfalls
- Assuming `setTimeout(fn, 0)` runs immediately or before promise callbacks.
- `await` inside `forEach`, expecting it to wait.
- Sequential awaits for independent requests, doubling load time.
- Starting promises early and awaiting them one at a time: if a later one rejects while you are still awaiting an earlier one, it has no handler yet and Node can treat it as unhandled. Prefer `Promise.all`.
- Forgetting `return` inside a `then`, so the chain stops waiting.
- Floating promises with no `catch`; in Node 15+ an unhandled rejection kills the process.
- Believing `Promise.all` cancels the other requests when one fails.
- Expecting `fetch` to reject on HTTP 404/500; it does not, so check `res.ok` (see [browser, DOM and events](06-browser-dom-events.md)).
- Heavy synchronous work on the main thread; in React Native it blocks the JS thread, so taps and JS-driven animations stall.
- Stale responses overwriting fresh ones; abort or ignore outdated requests.

## How to talk about this in an interview
- Tell the loop as a short story: "one call stack; the host does the waiting; when the stack empties, the loop drains all microtasks, then takes the next task, and the browser renders between tasks." Prove it with exercise 1.
- Tie it to your work: data loading in React and React Native screens, push notification handlers in the Coloc resident app, Next.js API routes at RemoteDev.
  > **Fill in:** a real async bug you handled (search race condition, unhandled rejection, a slow sequential loop you parallelized) and how you found it.
  > **Fill in:** how the Coloc resident app or the TypeScript platform fetches data (TanStack Query, oRPC client calls, plain fetch) and how errors reach the UI.
- For "promises vs async/await", say they are one mechanism: async/await is syntax over promises that reads like synchronous code and works with `try/catch`, and you still use `Promise.all` for parallel work.

## Interview questions

**Q1. Explain the event loop.**
JavaScript runs on one call stack. Async operations are handed to the host (browser APIs or libuv), which queues callbacks when they are ready. The loop runs one task to completion, drains every microtask, lets the browser render if needed, and repeats.
*Follow-up:* Why does a long loop freeze the page? Rendering and input handling only happen between tasks.

**Q2. What is the difference between microtasks and macrotasks? Give examples.**
Macrotasks (tasks) are `setTimeout`, `setInterval`, I/O, UI events and `MessageChannel`; one runs per loop turn. Microtasks are promise reactions, `await` continuations, `queueMicrotask` and `MutationObserver`; the whole queue drains after every task, so they always beat the next timer.
*Follow-up:* Can microtasks starve the page? Yes, a microtask that keeps queueing more blocks rendering and tasks.

**Q3. Does `setTimeout(fn, 0)` run immediately?**
No. It queues a task that runs after the current script, all pending microtasks and any earlier tasks; 0 is a minimum. Browsers clamp deeply nested timers to about 4 ms and throttle background tabs.
*Follow-up:* How do you let the browser paint between chunks of work? Yield with `setTimeout` or `requestAnimationFrame`, not with a promise.

**Q4. What are the states of a promise, and what does `then` return?**
Pending, then fulfilled or rejected, once only. `then` always returns a new promise: resolved with the handler's return value, rejected if the handler throws, or following a returned promise. That is what makes chaining work.
*Follow-up:* Does the executor run synchronously? Yes; only the reactions are asynchronous.

**Q5. Compare `Promise.all`, `allSettled`, `race` and `any`.**
`all` fulfills with every value in order and rejects on the first failure; `allSettled` waits for all and never rejects; `race` settles like whichever settles first; `any` fulfills with the first success and rejects with an `AggregateError` only if all fail. None cancel the remaining work.
*Follow-up:* Which one for syncing 50 independent listings? `allSettled`, then retry or report the failures.

**Q6. Promises vs async/await: what is the difference? Does `await` block?**
async/await is syntax over promises: an async function returns a promise, and `await` suspends only that function, resuming it in a microtask while the thread runs other code. It reads like synchronous code and uses `try/catch`, but parallelism still needs `Promise.all`.
*Follow-up:* What happens if an async function throws before any `await`? It returns a rejected promise rather than throwing synchronously.

**Q7. How do you run two independent requests in parallel with async/await?**
Start both, then await them together: `const [a, b] = await Promise.all([getA(), getB()])`. Two `await`s in a row are sequential, so the total time becomes the sum instead of the maximum.
*Follow-up:* What if one may fail and you still want the other? `Promise.allSettled`.

**Q8. Why does `await` inside `forEach` not work as expected?**
`forEach` calls the async callback and ignores the promise it returns, so every iteration starts at once and the code after the loop runs before any finishes. Use `for...of` for sequential work or `map` with `Promise.all` for parallel work.
*Follow-up:* How do you limit concurrency to 5? A promise pool (see the implementations file) or a small library such as p-limit.

**Q9. How does error handling work in promise chains and async functions?**
A thrown error or rejection skips fulfillment handlers until the nearest `catch`; returning a value from `catch` recovers the chain. In async functions, a rejected `await` throws at that line, so `try/catch/finally` works; rethrow when the caller should know.
*Follow-up:* `.then(ok, fail)` vs `.then(ok).catch(fail)`? Only the second catches errors thrown inside `ok`.

**Q10. What happens with an unhandled promise rejection?**
Browsers log an error and fire `unhandledrejection` on `window`. Node, since version 15, crashes the process by default. Always await or attach a `catch`, and keep global handlers for logging only.
*Follow-up:* Which lint rule helps? typescript-eslint's `no-floating-promises`.

**Q11. How do you cancel a fetch, and how do you avoid race conditions in a React effect?**
Pass the `signal` of an `AbortController` and call `abort()`; fetch rejects with an `AbortError`, which you ignore. In an effect, create the controller inside it and abort in the cleanup, so a stale request can never overwrite newer results.
*Follow-up:* How do you add a timeout? `AbortSignal.timeout(ms)` or `Promise.race` with a timer.

**Q12. What is callback hell, and how do you get out of it?**
Deeply nested callbacks for dependent async steps, with error checks at every level and inversion of control (trusting a library to call you exactly once). Flatten it with promises, async/await, named functions and `util.promisify` for Node-style APIs.
*Follow-up:* What is an error-first callback? Node's `(err, result)` convention.

**Q13. `requestAnimationFrame` vs `setTimeout` for animations?**
rAF runs just before the next paint, in step with the display, and pauses in background tabs. `setTimeout` is not aligned with frames, so it can drop or double frames and waste work. Use rAF for visual updates and animate `transform` and `opacity`.
*Follow-up:* Is rAF a microtask? No, it runs in the rendering step of the loop.

**Q14. In Node, how do `process.nextTick`, `setImmediate` and `setTimeout(0)` differ?**
`nextTick` callbacks run right after the current operation, even before promise microtasks. `setImmediate` runs in the check phase after I/O polling. `setTimeout(0)` runs in the timers phase; its order against `setImmediate` is random in the main module, but `setImmediate` wins inside I/O callbacks.
*Follow-up:* Danger of `nextTick`? Recursive `nextTick` calls starve I/O.

**Q15. How can you block the event loop, and how do you avoid it?**
Long synchronous work: heavy loops, huge `JSON.parse` calls, synchronous file or crypto APIs in Node, catastrophic regexes. Split work into chunks that yield, move CPU work to Web Workers or Node worker threads, paginate, or hand it to a background job.
*Follow-up:* And in React Native? Heavy JS-thread work freezes touch handling and JS-driven animations; batch it or move it off the JS thread.

**Q16. What is `for await...of`?**
A loop over async iterables that awaits each next value. Typical uses are paginated API clients written as async generators, and Node streams.
*Follow-up:* Does it run iterations in parallel? No, strictly one after another.

**Q17. When does `return await` matter?**
Inside `try/catch/finally`: `return await p` lets the local `catch` handle a rejection and makes `finally` run after `p` settles. Outside a `try`, `return p` and `return await p` behave the same apart from stack traces.
*Follow-up:* Why did linters once flag it? ESLint's old `no-return-await` rule did; it was deprecated because `return await` costs nothing and improves stack traces.

**Q18. How do you convert a callback-based function into a promise-returning one?**
Wrap it: return `new Promise((resolve, reject) => fn(...args, (err, value) => (err ? reject(err) : resolve(value))))`. In Node, `util.promisify` does this, and most core modules ship promise versions (`node:fs/promises`).
*Follow-up:* What if the callback fires twice? The promise ignores the second call because it settles only once.

## Further reading
- [MDN: Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN: Using microtasks with queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [Node.js: The event loop, timers and process.nextTick()](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
