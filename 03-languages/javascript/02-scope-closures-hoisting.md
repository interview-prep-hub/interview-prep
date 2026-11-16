# Scope, Closures and Hoisting

> **Tier:** T1 (know cold) · **Related:** [Types and coercion](01-basics-types-coercion.md), [this, prototypes and classes](03-this-prototypes-classes.md), [Async and the event loop](04-async-event-loop-promises.md), [Implement from scratch](07-implement-from-scratch.md), [React hooks](../../06-react/02-hooks.md)
> **On your resume:** closures and scope sit under every React hook and event handler you write at Devntech (React, React on Rails, React Native with Expo) and under the legacy JavaScript that lives inside Rails monoliths.

## Quick revision
- Every function call creates an execution context: an environment record (its variables), a link to the outer environment, and a `this` binding. Contexts live on the call stack (LIFO).
- Two phases: creation (bindings set up: `var` = `undefined`, function declarations fully created, `let`/`const`/`class` uninitialized), then execution (code runs line by line).
- Scope is lexical: decided by where code is written, not where it is called. Lookups walk the scope chain outward to global; not found means ReferenceError.
- Scopes: global, module, function, block (`{}` with `let`/`const`/`class`). `var` ignores blocks.
- Hoisting: `var` -> `undefined`; function declarations -> the whole function; `let`/`const`/`class` -> exist but sit in the TDZ until their line runs.
- TDZ access throws `ReferenceError: Cannot access 'x' before initialization`, even with `typeof`.
- Function expression in a `var`: calling it early gives `TypeError: x is not a function`; in `let`/`const`: ReferenceError.
- A closure is a function plus the environment it was created in. It keeps variables alive (live references, not snapshots) after the outer function returns.
- Uses: private state, counters, factories, memoization, `once`, debounce/throttle timers, event handlers, every React hook.
- `for (var i ...)` + `setTimeout` logs the final value three times (one shared `i`); `let` creates a fresh binding per iteration.
- IIFE `(function () { ... })()` creates a private scope; the module pattern returns a public API. ES modules replaced both.
- Stale closure in React: an effect or callback captured old state. Fix with correct dependencies, functional updates (`setX((x) => x + 1)`) or a ref.
- Closures leak memory when they keep big objects, DOM nodes, timers or listeners reachable. Clean up in effect cleanup functions.
- GC is reachability-based mark-and-sweep (generational in V8). Unreachable objects are freed, cycles included.

## 1. Execution contexts and the call stack

```js
const rate = 0.1;

function tax(amount) {
  return amount * rate;
}

function total(price) {
  const t = tax(price);   // pushes tax() on top of total()
  return price + t;
}

console.log(total(100)); // 110
```

```text
1. [global]
2. [global, total(100)]
3. [global, total(100), tax(100)]   tax returns 10 and is popped
4. [global, total(100)]             total returns 110 and is popped
5. [global]                         console.log(110) runs
```

Each execution context holds:
- **Environment record:** parameters, local variables and inner function declarations (plus `arguments` in regular functions).
- **Outer reference:** a link to the environment where the function was *defined*. Following these links gives the scope chain.
- **`this` binding:** decided by how the function is called (see [this, prototypes and classes](03-this-prototypes-classes.md)).

| Phase | What happens |
|---|---|
| Creation | bindings for parameters, `var` (set to `undefined`), function declarations (fully created), `let`/`const`/`class` (uninitialized, in the TDZ); `this` and the outer reference are set |
| Execution | code runs top to bottom; assignments happen; `let`/`const` become initialized when their line runs |

Recursion without a base case keeps pushing frames until `RangeError: Maximum call stack size exceeded`. An error's stack trace is a snapshot of the call stack: read it from the top to find where it was thrown.

## 2. Lexical scope and the scope chain

```js
const label = 'global';

function makeLogger() {
  const label = 'logger';
  return function log() {
    console.log(label);   // resolved where log was written
  };
}

function run(fn) {
  const label = 'run';
  fn();                   // where it is called does not matter
}

run(makeLogger());        // 'logger'
```

Lookup goes current scope, then outer scope, up to global; the first match wins, so inner declarations shadow outer ones. If nothing matches, reading throws a ReferenceError. *Assigning* to an undeclared name in sloppy mode silently creates a global; strict mode (always on in ES modules and class bodies) throws instead:

```js
'use strict';
function save() {
  counter = 1; // ReferenceError in strict mode; sloppy mode creates an accidental global
}
```

Lexical scope is static. `this` is the one thing decided at call time, which is why it feels dynamic.

## 3. Global, module, function and block scope

| Scope | Created by | Contains |
|---|---|---|
| Global | the page or process | top-level `var` and function declarations of classic scripts (also become `window` properties) and top-level `let`/`const`/`class` (global, but not on `window`) |
| Module | each ES module file | all top-level declarations, private unless exported |
| Function | each function call | parameters, `var`, inner functions, function-level `let`/`const` |
| Block | `{}`, `if`, `for`, `switch`, `try/catch` | `let`, `const`, `class`, and the `catch (err)` parameter |

```js
function demo() {
  if (true) {
    var fnScoped = 'visible in the whole function';
    let blockScoped = 'only inside this block';
  }
  console.log(fnScoped);        // works
  // console.log(blockScoped);  // ReferenceError
}
```

Node's CommonJS wraps every file in a function `(exports, require, module, __filename, __dirname)`, so top-level variables are file-scoped, not global.

## 4. Hoisting rules

Hoisting is not code physically moving. It is the creation phase registering declarations before any line runs.

| Declaration | Hoisted | Value before its line | Reading or calling early |
|---|---|---|---|
| `var x = 1` | yes | `undefined` | reads `undefined` |
| `function f() {}` | yes, with body | the function | works |
| `let` / `const` | yes | uninitialized (TDZ) | ReferenceError |
| `class C {}` | yes | uninitialized (TDZ) | ReferenceError |
| `var f = function () {}` | the variable only | `undefined` | `TypeError: f is not a function` |
| `const f = () => {}` | the variable only | uninitialized (TDZ) | ReferenceError |
| `import x from './x.js'` | yes, linked before the module body runs | live binding | works |

```js
console.log(a);   // undefined
var a = 1;

greet();          // 'hi': function declarations are fully hoisted
function greet() { console.log('hi'); }

// sayBye();      // TypeError: sayBye is not a function (it is still undefined)
var sayBye = function () {};

// console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;
```

### The temporal dead zone (TDZ)
The TDZ runs from the start of a scope to the line that initializes a `let`, `const` or `class`. The binding already exists (so it shadows outer variables) but cannot be read or written:

```js
let x = 'outer';
function show() {
  // console.log(x);  // ReferenceError: the inner x shadows the outer one and is in its TDZ
  let x = 'inner';
  console.log(x);     // 'inner'
}

// Default parameters are evaluated left to right, so they have a TDZ too
function f(a = b, b = 1) {}
// f();               // ReferenceError: Cannot access 'b' before initialization
```

Why it exists: "used before declared" becomes a loud error instead of a silent `undefined`, and a `const` never observably changes value.

### A function and a var with the same name
During creation the function declaration initializes the name; then execution runs the assignment:

```js
console.log(typeof foo); // 'function'
var foo = 1;
function foo() {}
console.log(typeof foo); // 'number'
```

Avoid function declarations inside blocks in sloppy mode: their hoisting differs between strict and sloppy mode. Use `const fn = () => {}` inside blocks.

## 5. Closures

A **closure** is a function bundled with references to the variables of the scope where it was created. Because the inner function keeps that environment reachable, the variables survive after the outer function returns. Closures capture **variables (bindings), not values**: if the variable changes, the closure sees the new value.

```js
function makeCounter() {
  let count = 0;                       // private: reachable only through the returned functions
  return {
    increment: () => ++count,
    decrement: () => --count,
    value: () => count,
  };
}

const c1 = makeCounter();
const c2 = makeCounter();              // separate call, separate environment
c1.increment();
c1.increment();
c2.increment();
console.log(c1.value(), c2.value());   // 2 1
```

### Practical uses

```js
// 1. Factory / partial application
const makeMultiplier = (factor) => (n) => n * factor;
const double = makeMultiplier(2);
double(21); // 42

// 2. once: run an expensive initialization a single time
function once(fn) {
  let called = false;
  let result;
  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}
const initSdk = once(() => { console.log('SDK init'); return 'ready'; });
initSdk();
initSdk(); // 'SDK init' was logged only once

// 3. Memoization: the cache lives in the closure
function memoize(fn) {
  const cache = new Map();
  return (n) => {
    if (cache.has(n)) return cache.get(n);
    const value = fn(n);
    cache.set(n, value);
    return value;
  };
}
const fastSquare = memoize((n) => n * n);
fastSquare(9); // computed
fastSquare(9); // cache hit

// 4. Remembering configuration
function createApiClient(baseUrl, token) {
  return (path) =>
    fetch(`${baseUrl}${path}`, { headers: { Authorization: `Bearer ${token}` } });
}
```

Debounce and throttle keep their timer id in a closure (see [implement from scratch](07-implement-from-scratch.md)). Every React hook relies on closures: each render's handlers close over that render's props and state.

## 6. The classic loop puzzle

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3
```

**Why:** `var` creates a single `i` for the whole function. All three callbacks close over that one binding, and they run only after the loop has finished (timers are tasks that run once the call stack is empty), when `i` is 3.

**Fixes:**

```js
// 1. let: a fresh binding per iteration (cleanest)
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 0 1 2

// 2. IIFE: capture the current value in a new function scope (the pre-ES6 fix)
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}

// 3. Pass the value as setTimeout's extra argument
for (var i = 0; i < 3; i++) setTimeout((j) => console.log(j), 0, i);
```

A variation interviewers like: with a delay of `i * 1000`, the output is `3`, `3`, `3`, one second apart. The delay is computed during the loop; the value is read later.

## 7. IIFE and the module pattern

Before ES modules, all scripts shared one global scope. An IIFE (Immediately Invoked Function Expression) gave each file a private scope, and the module pattern returned only a public API:

```js
const cart = (function () {
  const items = [];                  // private

  function totalCents() {
    return items.reduce((sum, item) => sum + item.priceCents, 0);
  }

  return {                           // public API (revealing module pattern)
    add(item) { items.push(item); },
    totalCents,
  };
})();

cart.add({ name: 'Booster pack', priceCents: 450 });
cart.totalCents(); // 450
cart.items;        // undefined: truly private
```

You still meet IIFEs in Rails apps that use the Sprockets asset pipeline (files are concatenated into one bundle, so each file wraps itself), in old jQuery plugins and in bundler output. Today each ES module already has its own scope. One modern use remains: an async IIFE in CommonJS, which has no top-level `await`: `(async () => { await main(); })();`.

## 8. Stale closures in React hooks

Every render creates new functions that close over that render's state. A function created during an old render keeps seeing old values:

```jsx
import { useEffect, useRef, useState } from 'react';

function Ticker() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);  // BUG: count is always 0 here (captured on the first render)
    }, 1000);
    return () => clearInterval(id);
  }, []);                   // empty deps: the effect never sees a newer count

  return <p>{count}</p>;    // stuck at 1
}
```

Fixes, in order of preference:
1. **Functional update:** `setCount((c) => c + 1)`. React passes the latest value, so no dependency is needed.
2. **Correct dependencies:** `[count]` recreates the interval on every change (works, but restarts the timer).
3. **A ref holding the latest value**, for long-lived callbacks that must read current state:

```jsx
const latestCount = useRef(count);
useEffect(() => {
  latestCount.current = count;
}, [count]);
// a long-lived listener reads latestCount.current
```

The `react-hooks/exhaustive-deps` lint rule catches most stale closures; treat its warnings as bugs, not noise. React Native has the same trap, for example a notification listener registered once in an effect that reads state captured at registration time. More in [React hooks](../../06-react/02-hooks.md).

## 9. Memory leaks and garbage collection

### How garbage collection works
JavaScript frees memory based on **reachability**. Roots are the global object, the current call stack and active handles (timers, listeners). Anything reachable from a root survives; everything else is garbage.
- **Mark-and-sweep:** mark everything reachable from the roots, then sweep the rest. Unreachable cycles (`a.ref = b; b.ref = a`) are collected, unlike with plain reference counting.
- **Generational (V8):** most objects die young, so new objects go to a small young generation that is scavenged often; survivors are promoted to an old generation that is collected less often.
- Collection can pause your code briefly; modern engines do much of it incrementally and concurrently.

### How closures leak

```js
function attachHandler(button) {
  const hugeReport = new Array(1_000_000).fill('row');
  button.addEventListener('click', () => {
    console.log(hugeReport.length);   // the listener keeps hugeReport alive
  });
}
// If the listener is never removed and the button stays referenced (even detached
// from the DOM), hugeReport can never be collected.
```

Common sources:
- Listeners and subscriptions that are never removed.
- `setInterval` never cleared, with a callback that closes over components or big data.
- Unbounded module-level caches (memoizing by user id forever). Use an LRU, a TTL or a `WeakMap` keyed by objects.
- Detached DOM nodes still referenced from JavaScript.
- In Node, request data pushed into module-level arrays, so it outlives every request.

```jsx
// inside a component
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize); // cleanup prevents the leak
}, []);
```

### Debugging a leak
In the Chrome DevTools Memory tab, take a heap snapshot, repeat the suspicious action a few times, take another snapshot and compare. Look for growing object counts and "Detached" DOM trees, then follow the retainers path to the closure holding them. In Node, run with `--inspect` and use the same tools, or watch `process.memoryUsage().heapUsed` over time.

### Going deeper
- V8 can share one context object among all closures created in the same scope. If one closure uses a big variable, the variable may stay alive while *any* sibling closure is alive. Keep long-lived closures small and drop big references when done.
- `WeakRef` and `FinalizationRegistry` (ES2021) exist for advanced caches, but GC timing is never guaranteed, so never put program logic in finalizers.

## Common pitfalls
- `var` in loops with async callbacks (the `3 3 3` bug).
- Calling a function expression before its line (`TypeError: x is not a function`).
- Reading a `let`/`const` before its declaration, especially when an inner variable shadows an outer one with the same name.
- Accidental globals from assigning to undeclared names in sloppy-mode scripts.
- Stale closures in `useEffect`, intervals and native listeners; silencing `exhaustive-deps` instead of fixing the cause.
- Missing effect cleanup for listeners, intervals and subscriptions.
- Believing closures copy values. They hold live references to variables.
- Module-level caches that grow forever in a long-running Node process.

## How to talk about this in an interview
- Define, then demonstrate: "A closure is a function that remembers the variables from where it was created; `makeCounter` returns functions sharing a private `count`." Then give a real use: debounce timers, memoization, React hooks.
- Hooks are your strongest link: every handler in the React and React Native code you write at Devntech is a closure over one render. Explaining a stale-closure bug and its functional-update fix sounds like real experience because it is.
  > **Fill in:** a stale closure or missing-cleanup bug you met in the Coloc resident app or a React screen (interval, notification listener, subscription) and how you fixed it.
- For legacy code, mention that asset-pipeline JavaScript in Rails apps often relies on IIFEs and globals, and that understanding scope helps when moving it into modules.
  > **Fill in:** did you touch legacy JavaScript in a Rails monolith (ERB views, asset pipeline files)? What did it look like?
- For hoisting, use the precise model (the creation phase registers declarations; `let`/`const` wait in the TDZ) rather than "code moves to the top".

## Interview questions

**Q1. What is a closure? Give a practical example.**
A function together with references to the variables of the scope where it was defined; those variables stay alive after the outer function returns. `makeCounter` returning `increment` and `value` over a private `count` is the classic example; real uses are debounce timers, memoization caches, `once` and React hooks.
*Follow-up:* Does a closure capture the value or the variable? The variable, so later changes are visible.

**Q2. What is hoisting, and what exactly gets hoisted?**
During the creation phase, declarations are registered before any code runs. `var` bindings start as `undefined`, function declarations are created with their body, and `let`/`const`/`class` bindings exist but stay uninitialized (TDZ) until their line runs. Assignments and function expressions are not hoisted.
*Follow-up:* Is code physically moved? No, it is a model for the two-phase setup.

**Q3. What is the temporal dead zone?**
The span from the start of a scope until a `let`, `const` or `class` declaration is evaluated. Touching the binding there, even with `typeof`, throws `ReferenceError: Cannot access 'x' before initialization`. It turns early use into a loud error and keeps `const` meaningful.
*Follow-up:* Can default parameters hit it? Yes: in `function f(a = b, b = 1)`, `a`'s default reads `b` before it is initialized.

**Q4. What does `for (var i = 0; i < 3; i++) setTimeout(() => console.log(i))` print, and how do you fix it?**
`3 3 3`: one function-scoped `i` is shared by every callback, and the callbacks run after the loop. Fix with `let` (a fresh binding per iteration), an IIFE that captures `i` as a parameter, or by passing `i` as setTimeout's third argument.
*Follow-up:* Why does `let` work? The spec creates a new binding for each iteration and copies the current value into it.

**Q5. What is lexical scope, and what is the scope chain?**
Scope is decided by where code is written. Each environment links to its outer environment, forming a chain that lookups walk outward until global. A function called from elsewhere still resolves names where it was defined.
*Follow-up:* What is not lexical? `this` in regular functions, which depends on the call site.

**Q6. Function declaration vs function expression: what is the hoisting difference?**
A declaration (`function f() {}`) is hoisted with its body, so it works before its line. An expression (`const f = function () {}` or an arrow) only hoists the variable: with `var` an early call is a TypeError (calling `undefined`), with `let`/`const` a ReferenceError.
*Follow-up:* Where is a named function expression's name visible? Only inside that function, which is handy for recursion.

**Q7. What is an execution context? Describe the creation and execution phases.**
The environment code runs in: an environment record of variables, a reference to the outer environment, and a `this` binding. Creation sets up bindings (`var` = undefined, functions created, `let`/`const` in the TDZ) and `this`; execution runs the code and performs assignments. Each call pushes a new context on the call stack.
*Follow-up:* What causes "Maximum call stack size exceeded"? Recursion that never reaches a base case.

**Q8. How do you create private state in JavaScript?**
With closures (a factory or module pattern that returns functions using local variables) or with class private fields (`#count`). Both are enforced by the language; an underscore prefix is only a convention.
*Follow-up:* Which is lighter for many instances? Private fields, because methods live once on the prototype; closure-based methods are recreated per instance.

**Q9. What is an IIFE and why was it used?**
A function expression invoked immediately, `(function () { ... })()`, giving code a private scope so nothing leaks into the shared global scope of browser scripts; the module pattern returns a public object from it. ES modules made it mostly unnecessary, but it survives in legacy code and as an async IIFE in CommonJS.
*Follow-up:* Why the wrapping parentheses? They make the parser read `function` as an expression, not a declaration.

**Q10. What is a stale closure in React, and how do you fix it?**
A callback or effect from an earlier render keeps reading that render's state, for example a `setInterval` created in an effect with `[]` dependencies that always sees `count = 0`. Fix with functional updates, correct dependency arrays, or a ref that holds the latest value.
*Follow-up:* What catches it automatically? The `react-hooks/exhaustive-deps` ESLint rule.

**Q11. How can closures cause memory leaks?**
A closure keeps its reachable environment alive, so a forgotten listener, interval or subscription can retain large data and detached DOM nodes indefinitely; unbounded module-level caches are the server-side version. Remove listeners, clear timers, and bound caches with an LRU, a TTL or a `WeakMap`.
*Follow-up:* How do you find a leak? Compare heap snapshots in DevTools and follow the retainers path.

**Q12. How does garbage collection work in JavaScript?**
Objects are reclaimed once they are unreachable from the roots (globals, the call stack, active handles). Engines use mark-and-sweep, so unreachable cycles are collected too. V8 is generational: a young generation scavenged often and an old generation collected less often.
*Follow-up:* Can you force a collection? Not from normal code; design for reachability instead.

**Q13. Block scope vs function scope: what changes in a `for` loop with `let`?**
`var` belongs to the enclosing function and ignores blocks; `let` and `const` belong to the nearest block. In `for (let ...)`, every iteration gets its own binding, so closures created in the loop see different values.
*Follow-up:* Is the `catch (err)` parameter block scoped? Yes, it exists only inside the catch block.

**Q14. What happens when you assign to an undeclared variable?**
In sloppy mode it creates a property on the global object (an accidental global); in strict mode it throws a ReferenceError. ES modules and class bodies are always strict.
*Follow-up:* How do you opt a classic script into strict mode? Put `'use strict';` at the top of the file or function.

**Q15. What is logged by `console.log(typeof foo); var foo = 1; function foo() {} console.log(typeof foo);`?**
`'function'`, then `'number'`. The function declaration initializes `foo` in the creation phase and the `var` does not overwrite it; then the assignment `foo = 1` runs during execution.
*Follow-up:* And with `let foo = 1`? SyntaxError: a name cannot be declared with `let` and as a function in the same scope.

**Q16. How would you implement `memoize` with a closure, and what are the risks?**
Return a wrapper that checks a `Map` kept in the closure before calling the original, keyed by the argument (or a serialized key for several arguments). Risks: unbounded growth, wrong keys for object arguments, and caching impure or time-dependent functions.
*Follow-up:* How do you bound the cache? LRU eviction or a TTL (the LRU cache in the implementations file).

**Q17. Module scope vs global scope: what is the difference?**
Top-level declarations in an ES module are private to that file and shared only through `export`. In classic scripts, top-level `var` and function declarations become `window` properties visible to every script. CommonJS gets file scope by wrapping each file in a function.
*Follow-up:* What is `this` at the top level of an ES module? `undefined`.

## Further reading
- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- [MDN: Memory management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management)
