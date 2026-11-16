# Implement From Scratch: Classic JavaScript Utilities

> **Tier:** T1 (know cold) · **Related:** [Scope and closures](02-scope-closures-hoisting.md), [this, prototypes and classes](03-this-prototypes-classes.md), [Async and promises](04-async-event-loop-promises.md), [Browser, DOM and events](06-browser-dom-events.md), [Big O and problem solving](../../02-dsa/00-big-o-and-problem-solving.md)
> **On your resume:** machine-coding rounds for JS/TS full-stack and React Native roles. These utilities mirror real work: debounced search in React and React Native screens, retries and concurrency limits for third-party API syncs such as PriceCharting, and caches in TypeScript services.

## Quick revision
- Every snippet below is self-contained: save it as a file, run `node file.js`, and it prints `<name> ok` when all assertions pass.
- Interview flow: clarify the API and edge cases, write the simplest correct version, test it out loud, then add options (leading/trailing, cancel, limits).
- **debounce:** restart a timer on every call; run after `wait` ms of silence. `leading` runs on the first call of a burst.
- **throttle:** run at most once per `wait` ms; keep the latest arguments for one trailing call.
- **memoize:** `Map` keyed by the argument (or a resolver); check with `has()` so `undefined` results are cached; bound it in long-lived processes.
- **curry:** collect arguments until `fn.length` is reached. **compose** runs right to left (`reduceRight`), **pipe** left to right (`reduce`).
- **deepClone:** recurse with a `WeakMap` from original to copy (handles cycles and shared references); special-case Date, RegExp, Map, Set.
- **deepEqual:** SameValueZero for primitives, same prototype, same keys, recurse, guard cycles.
- **flatten:** recursion with a depth counter, or an explicit stack to avoid deep recursion.
- **Promise.all:** count what remains, store results by index, reject on the first error, resolve `[]` for empty input. **allSettled** = all over mapped outcomes. **race** = first settlement wins.
- **promisify:** return a function that wraps the call in `new Promise` and appends an `(err, value)` callback.
- **MyPromise:** state, value and a handler queue; `then` returns a new promise; handlers run as microtasks; thenables are adopted; settle once.
- **call/apply:** attach the function to the context under a `Symbol` key and call it as a method. **bind:** return a wrapper with a fixed `this` that constructs when used with `new`.
- **map/filter/reduce:** loop with `i in this` to skip holes; reduce without an initial value starts at the first element and throws on an empty array.
- **EventEmitter:** `Map` of event to listener array; `once` wraps the listener; copy the array before emitting.
- **retry:** loop with try/catch, delay = `base * factor ** attempt`, capped, with jitter; retry only retryable errors.
- **Promise pool:** N workers pull the next index from a shared counter; results are stored by index.
- **LRU cache:** a `Map` remembers insertion order; on `get`, delete and re-insert; evict `map.keys().next().value`.

## 1. Debounce (with leading option)

Run `fn` only after calls stop for `wait` ms (search box, autosave). With `leading: true`, the first call of a burst runs immediately; `trailing` controls the call at the end of the burst.

```js
function debounce(fn, wait, { leading = false, trailing = true } = {}) {
  let timerId = null;
  let pendingArgs = null;
  let pendingThis = null;

  function debounced(...args) {
    const startsBurst = timerId === null;
    pendingArgs = args;
    pendingThis = this;
    clearTimeout(timerId);
    timerId = setTimeout(() => {
      timerId = null;
      if (trailing && pendingArgs) fn.apply(pendingThis, pendingArgs);
      pendingArgs = pendingThis = null;
    }, wait);
    if (leading && startsBurst) {
      fn.apply(this, args);
      pendingArgs = pendingThis = null; // handled; a trailing call needs another call in the burst
    }
  }

  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = pendingArgs = pendingThis = null;
  };
  return debounced;
}

// Tests
const trailingCalls = [];
const save = debounce((value) => trailingCalls.push(value), 40);
save(1); save(2); save(3);
console.assert(trailingCalls.length === 0, 'trailing: nothing runs during the burst');

const leadingCalls = [];
const click = debounce((value) => leadingCalls.push(value), 40, { leading: true, trailing: false });
click('a'); click('b');
console.assert(leadingCalls.join() === 'a', 'leading: the first call runs immediately');

const bothCalls = [];
const both = debounce((value) => bothCalls.push(value), 40, { leading: true });
both('x'); both('y'); both('z');

const cancelled = [];
const cancelMe = debounce(() => cancelled.push(1), 20);
cancelMe();
cancelMe.cancel();

const counter = { count: 0, inc: debounce(function () { this.count++; }, 20) };
counter.inc();

setTimeout(() => {
  console.assert(trailingCalls.join() === '3', 'trailing: only the last call runs');
  console.assert(leadingCalls.join() === 'a', 'leading only: no trailing call');
  console.assert(bothCalls.join() === 'x,z', 'leading + trailing: first and last call');
  console.assert(cancelled.length === 0, 'cancel drops the pending call');
  console.assert(counter.count === 1, 'this is preserved');
  click('c');
  console.assert(leadingCalls.join() === 'a,c', 'a new burst runs the leading call again');
  console.log('debounce ok');
}, 150);
```

**Complexity:** O(1) per call, one timer at a time.
**Edge cases:** preserve `this` and the latest arguments (hence `function` and `apply`, not an arrow call); `cancel` for component unmounts; in React, create the debounced function once (`useMemo`/`useRef`) or every render gets a fresh timer.

## 2. Throttle

Run `fn` at most once per `wait` ms (scroll, mousemove, drag). This version runs on the leading edge and schedules one trailing call with the latest arguments, so the final position is never lost.

```js
function throttle(fn, wait) {
  let lastRun = 0;
  let timerId = null;
  let pendingArgs = null;
  let pendingThis = null;

  function run(context, args) {
    lastRun = Date.now();
    fn.apply(context, args);
  }

  function throttled(...args) {
    const remaining = wait - (Date.now() - lastRun);
    if (remaining <= 0) {
      clearTimeout(timerId);              // a late timer is replaced by this fresher call
      timerId = pendingArgs = pendingThis = null;
      run(this, args);                    // leading edge
    } else {
      pendingArgs = args;                 // remember only the latest call
      pendingThis = this;
      if (timerId === null) {
        timerId = setTimeout(() => {
          timerId = null;
          run(pendingThis, pendingArgs);  // trailing edge
          pendingArgs = pendingThis = null;
        }, remaining);
      }
    }
  }

  throttled.cancel = () => {
    clearTimeout(timerId);
    timerId = pendingArgs = pendingThis = null;
    lastRun = 0;
  };
  return throttled;
}

// Tests
const calls = [];
const onScroll = throttle((position) => calls.push(position), 50);
for (let position = 1; position <= 5; position++) onScroll(position); // one synchronous burst
console.assert(calls.join() === '1', 'the leading call runs immediately, the rest wait');

setTimeout(() => {
  console.assert(calls.join() === '1,5', 'one trailing call with the latest arguments');
  setTimeout(() => {
    onScroll(6);
    console.assert(calls.join() === '1,5,6', 'after a quiet period the next call runs at once');
    const cancelled = [];
    const limited = throttle(() => cancelled.push(1), 30);
    limited();
    limited();
    limited.cancel();
    setTimeout(() => {
      console.assert(cancelled.length === 1, 'cancel drops the scheduled trailing call');
      console.log('throttle ok');
    }, 60);
  }, 100);
}, 100);
```

**Complexity:** O(1) per call.
**Edge cases:** timers can fire late, so compute `remaining` from timestamps instead of assuming exact delays; drop or keep the trailing call depending on the use case; `requestAnimationFrame` is a natural throttle for visual updates.

## 3. Memoize

Cache results by argument so repeated calls with the same input skip the work. Works only for pure functions.

```js
function memoize(fn, resolver = (...args) => (args.length === 1 ? args[0] : JSON.stringify(args))) {
  const cache = new Map();
  function memoized(...args) {
    const key = resolver(...args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  }
  memoized.cache = cache; // exposed so callers can inspect or clear it
  return memoized;
}

// Tests
let computed = 0;
const square = memoize((n) => { computed++; return n * n; });
console.assert(square(4) === 16 && square(4) === 16 && computed === 1, 'the second call is a cache hit');
square(5);
console.assert(computed === 2, 'a new argument is computed');

const add = memoize((a, b) => a + b);
console.assert(add(1, 2) === 3 && add(1, 2) === 3 && add.cache.size === 1, 'multi-argument key');

let lookups = 0;
const findUser = memoize(() => { lookups++; return undefined; });
findUser(1);
findUser(1);
console.assert(lookups === 1, 'undefined results are cached because has() is used');

const fib = memoize((n) => (n < 2 ? n : fib(n - 1) + fib(n - 2)));
console.assert(fib(50) === 12586269025, 'memoized recursion is linear');

const shout = memoize((user) => user.name.toUpperCase(), (user) => user.id);
console.assert(shout({ id: 1, name: 'sara' }) === 'SARA' && shout({ id: 1, name: 'x' }) === 'SARA', 'custom resolver');
console.log('memoize ok');
```

**Complexity:** O(1) average lookup plus the cost of building the key (`JSON.stringify` is O(size of the arguments)); memory grows with every distinct input.
**Edge cases:** object arguments are keyed by identity with the single-argument default (a mutated object returns a stale result); `JSON.stringify` loses `undefined` and functions; promises are cached too (good for deduping requests, bad if a rejection stays cached); unbounded growth in servers, so add an LRU or TTL.

## 4. Curry

Turn `f(a, b, c)` into a function that accepts arguments in any grouping: `f(a)(b)(c)`, `f(a, b)(c)`.

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args);   // enough arguments: call
    return (...more) => curried.apply(this, [...args, ...more]);  // otherwise keep collecting
  };
}

// Variadic "sum(1)(2)(3)()" style: an empty call ends the chain
function sum(a) {
  return (b) => (b === undefined ? a : sum(a + b));
}

// Tests
const volume = curry((length, width, height) => length * width * height);
console.assert(volume(2)(3)(4) === 24, 'one argument at a time');
console.assert(volume(2, 3)(4) === 24 && volume(2)(3, 4) === 24 && volume(2, 3, 4) === 24, 'any grouping');
const double = curry((factor, n) => factor * n)(2);
console.assert([1, 2, 3].map((n) => double(n)).join() === '2,4,6', 'partial application');
console.assert(sum(1)(2)(3)() === 6, 'variadic curry');
console.log('curry ok');
```

**Complexity:** O(n) total for n arguments (each partial call copies the collected arguments).
**Edge cases:** `fn.length` ignores default and rest parameters, so curry those explicitly with an arity argument; passing `map` callbacks directly (`arr.map(curried)`) supplies extra arguments (index, array).

## 5. Compose and pipe

Combine small functions into a pipeline. `pipe(f, g)(x)` is `g(f(x))`; `compose(f, g)(x)` is `f(g(x))`.

```js
const compose = (...fns) => (input) => fns.reduceRight((acc, fn) => fn(acc), input);
const pipe = (...fns) => (input) => fns.reduce((acc, fn) => fn(acc), input);
const pipeAsync = (...fns) => (input) =>
  fns.reduce((acc, fn) => acc.then(fn), Promise.resolve(input));

// Tests
const trim = (s) => s.trim();
const lower = (s) => s.toLowerCase();
const dasherize = (s) => s.replace(/\s+/g, '-');
console.assert(pipe(trim, lower, dasherize)('  Hello World  ') === 'hello-world', 'pipe: left to right');
console.assert(compose(dasherize, lower, trim)('  Hello World  ') === 'hello-world', 'compose: right to left');
console.assert(pipe()(5) === 5 && compose()(5) === 5, 'no functions returns the input');

pipeAsync(async (n) => n + 1, (n) => n * 2)(3).then((result) => {
  console.assert(result === 8, 'async pipe awaits each step');
  console.log('compose/pipe ok');
});
```

**Complexity:** O(k) function calls for k functions.
**Edge cases:** an empty list returns the input; only the first function in the chain may take several arguments in some variants; async steps need the promise-based version.

## 6. Deep clone (with cycles)

Copy nested data so the clone shares nothing with the original. A `WeakMap` from original to copy handles cycles and keeps shared references shared.

```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value; // primitives and functions
  if (seen.has(value)) return seen.get(value);                     // cycle or shared reference

  if (value instanceof Date) return new Date(value.getTime());
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);
  if (value instanceof Map) {
    const copy = new Map();
    seen.set(value, copy);
    value.forEach((v, k) => copy.set(deepClone(k, seen), deepClone(v, seen)));
    return copy;
  }
  if (value instanceof Set) {
    const copy = new Set();
    seen.set(value, copy);
    value.forEach((v) => copy.add(deepClone(v, seen)));
    return copy;
  }

  const copy = Array.isArray(value) ? new Array(value.length) : Object.create(Object.getPrototypeOf(value));
  seen.set(value, copy); // register before recursing so cycles resolve to this copy
  for (const key of [...Object.keys(value), ...Object.getOwnPropertySymbols(value)]) {
    copy[key] = deepClone(value[key], seen);
  }
  return copy;
}

// Tests
const secret = Symbol('secret');
const original = {
  name: 'Sara',
  createdAt: new Date('2025-01-01T00:00:00Z'),
  tags: ['a', { deep: true }],
  roles: new Set(['admin']),
  meta: new Map([['k', { v: 1 }]]),
  pattern: /ab+c/gi,
  [secret]: 42,
};
original.self = original;           // cycle
original.tagsAgain = original.tags; // shared reference

const copy = deepClone(original);
console.assert(copy !== original && copy.name === 'Sara', 'new top-level object');
console.assert(copy.self === copy, 'the cycle points at the copy');
console.assert(copy.tags !== original.tags && copy.tags[1] !== original.tags[1] && copy.tags[1].deep, 'nested copies');
console.assert(copy.tagsAgain === copy.tags, 'shared references stay shared');
console.assert(copy.createdAt !== original.createdAt && copy.createdAt.getTime() === original.createdAt.getTime(), 'Date');
console.assert(copy.roles.has('admin') && copy.roles !== original.roles, 'Set');
console.assert(copy.meta.get('k').v === 1 && copy.meta.get('k') !== original.meta.get('k'), 'Map values cloned');
console.assert(copy.pattern.source === 'ab+c' && copy.pattern.flags === 'gi', 'RegExp');
console.assert(copy[secret] === 42, 'symbol keys');

class Point {
  constructor(x) { this.x = x; }
  double() { return this.x * 2; }
}
const point = deepClone(new Point(2));
console.assert(point instanceof Point && point.double() === 4, 'prototype preserved');
console.log('deepClone ok');
```

**Complexity:** O(n) time and space for n reachable values.
**Edge cases:** cycles and shared references (the `WeakMap`); getters run and become plain values; non-enumerable properties, property flags and `#private` fields are not copied; constructors are not re-run; functions are shared, not cloned. In real code prefer `structuredClone` when you do not need prototypes.

## 7. Deep equal

Compare structure and values instead of references.

```js
function deepEqual(a, b, seen = new WeakMap()) {
  if (a === b || (Number.isNaN(a) && Number.isNaN(b))) return true; // SameValueZero
  if (typeof a !== 'object' || typeof b !== 'object' || a === null || b === null) return false;
  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) return false;
  if (seen.get(a) === b) return true; // already comparing this exact pair (cycle)
  seen.set(a, b);

  if (a instanceof Date) return a.getTime() === b.getTime();
  if (a instanceof RegExp) return a.toString() === b.toString();
  if (a instanceof Map) {
    return a.size === b.size && [...a].every(([k, v]) => b.has(k) && deepEqual(v, b.get(k), seen));
  }
  if (a instanceof Set) return a.size === b.size && [...a].every((v) => b.has(v)); // membership only

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every((key) => Object.hasOwn(b, key) && deepEqual(a[key], b[key], seen));
}

// Tests
console.assert(deepEqual({ a: [1, { b: 2 }] }, { a: [1, { b: 2 }] }), 'nested equal');
console.assert(!deepEqual({ a: 1 }, { a: 1, b: undefined }), 'an extra key, even undefined, differs');
console.assert(deepEqual(NaN, NaN) && deepEqual(0, -0), 'SameValueZero for primitives');
console.assert(!deepEqual([1, 2], { 0: 1, 1: 2 }), 'array vs object: different prototypes');
console.assert(deepEqual(new Date(5), new Date(5)) && !deepEqual(new Date(5), new Date(6)), 'Date');
console.assert(deepEqual(new Map([['k', [1]]]), new Map([['k', [1]]])), 'Map values compared deeply');
console.assert(!deepEqual(new Set([1, 2]), new Set([1, 3])), 'Set membership');
console.assert(!deepEqual(() => 1, () => 1), 'functions compare by reference');
const x = { name: 'x' };
x.self = x;
const y = { name: 'x' };
y.self = y;
console.assert(deepEqual(x, y), 'cycles terminate');
console.log('deepEqual ok');
```

**Complexity:** O(n) over all compared values.
**Edge cases:** NaN, `0` vs `-0` (decide; this version treats them as equal), missing vs `undefined` keys, arrays vs array-like objects, Dates, cycles; Set elements are compared by membership, not deeply; symbol keys are ignored here.

## 8. Flatten (recursive and iterative)

Flatten nested arrays to a given depth, like `Array.prototype.flat`. A bonus helper flattens nested objects into dot paths (a common follow-up).

```js
function flattenRecursive(arr, depth = Infinity) {
  const result = [];
  for (const item of arr) {
    if (Array.isArray(item) && depth > 0) result.push(...flattenRecursive(item, depth - 1));
    else result.push(item);
  }
  return result;
}

function flattenIterative(arr) {
  const stack = [...arr];
  const result = [];
  while (stack.length > 0) {
    const item = stack.pop();
    if (Array.isArray(item)) stack.push(...item); // unpack and revisit
    else result.push(item);
  }
  return result.reverse(); // popping from the end collected items in reverse order
}

function flattenObject(obj, prefix = '', out = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    if (value && typeof value === 'object' && !Array.isArray(value)) flattenObject(value, path, out);
    else out[path] = value;
  }
  return out;
}

// Tests
const nested = [1, [2, [3, [4, [5]]]], 6];
console.assert(JSON.stringify(flattenRecursive(nested)) === '[1,2,3,4,5,6]', 'full flatten');
console.assert(JSON.stringify(flattenRecursive(nested, 1)) === '[1,2,[3,[4,[5]]],6]', 'depth 1');
console.assert(JSON.stringify(flattenIterative(nested)) === JSON.stringify(nested.flat(Infinity)), 'iterative');
console.assert(flattenIterative([]).length === 0 && flattenRecursive([[]]).length === 0, 'empty input');
const flatObject = flattenObject({ a: 1, b: { c: 2, d: { e: 3 } } });
console.assert(JSON.stringify(flatObject) === '{"a":1,"b.c":2,"b.d.e":3}', 'object to dot paths');
console.log('flatten ok');
```

**Complexity:** O(n) time for n total elements; recursion uses O(depth) stack, the iterative version O(n) heap.
**Edge cases:** very deep nesting can overflow the call stack (use the iterative version); spreading huge arrays into `push` can hit argument limits; `for...of` turns holes into `undefined`, while native `flat` skips holes.

## 9. Promise.all, Promise.allSettled and Promise.race (plus any)

```js
function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);
    items.forEach((item, index) => {
      Promise.resolve(item).then((value) => {
        results[index] = value; // input order, not completion order
        remaining -= 1;
        if (remaining === 0) resolve(results);
      }, reject);               // the first rejection wins; later ones are ignored
    });
  });
}

function promiseAllSettled(iterable) {
  return promiseAll(
    Array.from(iterable, (item) =>
      Promise.resolve(item).then(
        (value) => ({ status: 'fulfilled', value }),
        (reason) => ({ status: 'rejected', reason }),
      ),
    ),
  );
}

function promiseRace(iterable) {
  return new Promise((resolve, reject) => {
    for (const item of iterable) Promise.resolve(item).then(resolve, reject);
  });
}

function promiseAny(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const errors = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return reject(new AggregateError([], 'All promises were rejected'));
    items.forEach((item, index) => {
      Promise.resolve(item).then(resolve, (reason) => {
        errors[index] = reason;
        remaining -= 1;
        if (remaining === 0) reject(new AggregateError(errors, 'All promises were rejected'));
      });
    });
  });
}

// Tests
const delay = (ms, value, fail = false) =>
  new Promise((resolve, reject) => setTimeout(() => (fail ? reject(value) : resolve(value)), ms));

(async () => {
  const all = await promiseAll([delay(30, 'a'), 'b', delay(10, 'c')]);
  console.assert(all.join() === 'a,b,c', 'all keeps input order and accepts plain values');
  console.assert((await promiseAll([])).length === 0, 'empty input resolves to []');
  const allError = await promiseAll([delay(20, 'x'), delay(5, 'boom', true)]).catch((e) => e);
  console.assert(allError === 'boom', 'all rejects with the first rejection');

  const settled = await promiseAllSettled([delay(5, 1), delay(5, 'bad', true)]);
  console.assert(settled[0].status === 'fulfilled' && settled[0].value === 1, 'fulfilled outcome');
  console.assert(settled[1].status === 'rejected' && settled[1].reason === 'bad', 'rejected outcome');

  console.assert((await promiseRace([delay(30, 'slow'), delay(5, 'fast')])) === 'fast', 'race: first wins');
  const raceError = await promiseRace([delay(30, 'slow'), delay(5, 'fail', true)]).catch((e) => e);
  console.assert(raceError === 'fail', 'race can reject first');

  console.assert((await promiseAny([delay(5, 'e1', true), delay(15, 'ok')])) === 'ok', 'any skips rejections');
  const anyError = await promiseAny([delay(5, 'e1', true), delay(5, 'e2', true)]).catch((e) => e);
  console.assert(anyError instanceof AggregateError && anyError.errors.join() === 'e1,e2', 'AggregateError');
  console.log('promise combinators ok');
})();
```

**Complexity:** O(n) setup; total time is the slowest input for `all`/`allSettled` and the fastest for `race`/`any`.
**Edge cases:** empty input (`all` resolves `[]`, `race` stays pending forever, `any` rejects); non-promise values (wrap with `Promise.resolve`); results by index, not push order; a promise can settle once, so extra resolve/reject calls are harmless; none of these cancel the losers.

## 10. Promisify

Convert a Node-style function whose last argument is an `(err, value)` callback into one that returns a promise.

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn.call(this, ...args, (err, value) => (err ? reject(err) : resolve(value)));
    });
  };
}

// Tests
function readConfig(name, callback) {
  setTimeout(() => {
    if (name === 'missing') callback(new Error('not found'));
    else callback(null, { name, debug: true });
  }, 5);
}
const readConfigAsync = promisify(readConfig);

const wallet = {
  balance: 0,
  deposit(amount, callback) {
    this.balance += amount;
    callback(null, this.balance);
  },
};
wallet.depositAsync = promisify(wallet.deposit);

(async () => {
  const config = await readConfigAsync('app');
  console.assert(config.name === 'app' && config.debug === true, 'resolves with the value');
  const error = await readConfigAsync('missing').catch((e) => e);
  console.assert(error instanceof Error && error.message === 'not found', 'rejects with the error');
  console.assert((await wallet.depositAsync(5)) === 5, 'this is preserved');
  const thrown = await promisify(() => { throw new Error('sync throw'); })().catch((e) => e.message);
  console.assert(thrown === 'sync throw', 'synchronous throws become rejections');
  console.log('promisify ok');
})();
```

**Complexity:** O(1) overhead per call.
**Edge cases:** callbacks with several result values (only the first is kept); callbacks called twice (the promise ignores the second); functions where the callback is not last; Node already ships `util.promisify` and `fs/promises`.

## 11. A simplified MyPromise

The core rules: settle once; `then` returns a new promise; handlers run asynchronously (microtasks), even if the promise is already settled; a handler's return value resolves the next promise; returned thenables are adopted; non-function handlers pass values through.

```js
class MyPromise {
  #state = 'pending';
  #value = undefined;
  #handlers = [];

  constructor(executor) {
    let called = false; // the resolve/reject pair works only once
    const resolve = (value) => { if (!called) { called = true; this.#resolve(value); } };
    const reject = (reason) => { if (!called) { called = true; this.#settle('rejected', reason); } };
    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err);
    }
  }

  #resolve(value) {
    if (value === this) return this.#settle('rejected', new TypeError('A promise cannot resolve to itself'));
    if (value !== null && (typeof value === 'object' || typeof value === 'function')) {
      let then;
      try {
        then = value.then;
      } catch (err) {
        return this.#settle('rejected', err);
      }
      if (typeof then === 'function') {
        // Adopt the thenable's state (the spec defers this call to a job; kept synchronous for brevity)
        let called = false;
        try {
          then.call(
            value,
            (v) => { if (!called) { called = true; this.#resolve(v); } },
            (r) => { if (!called) { called = true; this.#settle('rejected', r); } },
          );
        } catch (err) {
          if (!called) { called = true; this.#settle('rejected', err); }
        }
        return;
      }
    }
    this.#settle('fulfilled', value);
  }

  #settle(state, value) {
    if (this.#state !== 'pending') return;
    this.#state = state;
    this.#value = value;
    this.#handlers.forEach((handler) => this.#runHandler(handler));
    this.#handlers = [];
  }

  #runHandler({ onFulfilled, onRejected, resolve, reject }) {
    queueMicrotask(() => {
      const callback = this.#state === 'fulfilled' ? onFulfilled : onRejected;
      if (typeof callback !== 'function') {
        return this.#state === 'fulfilled' ? resolve(this.#value) : reject(this.#value); // pass through
      }
      try {
        resolve(callback(this.#value));
      } catch (err) {
        reject(err);
      }
    });
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handler = { onFulfilled, onRejected, resolve, reject };
      if (this.#state === 'pending') this.#handlers.push(handler);
      else this.#runHandler(handler);
    });
  }

  catch(onRejected) {
    return this.then(undefined, onRejected);
  }

  finally(onFinally) {
    return this.then(
      (value) => MyPromise.resolve(onFinally()).then(() => value),
      (reason) => MyPromise.resolve(onFinally()).then(() => { throw reason; }),
    );
  }

  static resolve(value) {
    return value instanceof MyPromise ? value : new MyPromise((resolve) => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }
}

// Tests
const order = [];
new MyPromise((resolve) => {
  order.push('executor');
  resolve(1);
})
  .then((v) => { order.push(`then ${v}`); return v + 1; })
  .then((v) => { order.push(`then ${v}`); throw new Error('boom'); })
  .then(() => order.push('skipped'))
  .catch((err) => {
    order.push(`caught ${err.message}`);
    return new MyPromise((resolve) => setTimeout(() => resolve('async value'), 10)); // adopted
  })
  .finally(() => order.push('finally'))
  .then((v) => order.push(v));
order.push('sync end');

const settleOnce = new MyPromise((resolve, reject) => {
  resolve('first');
  resolve('second');
  reject(new Error('ignored'));
});
const thenable = { then(onFulfilled) { onFulfilled('from thenable'); } };

setTimeout(async () => {
  const expected = 'executor | sync end | then 1 | then 2 | caught boom | finally | async value';
  console.assert(order.join(' | ') === expected, `order was: ${order.join(' | ')}`);
  console.assert((await settleOnce) === 'first', 'settles only once');
  console.assert((await MyPromise.resolve(thenable)) === 'from thenable', 'adopts thenables');
  console.assert((await MyPromise.reject(new Error('nope')).catch((e) => e.message)) === 'nope', 'catch');
  const selfRef = new MyPromise((resolve) => setTimeout(() => resolve(selfRef), 0));
  console.assert((await selfRef.catch((e) => e)) instanceof TypeError, 'self-resolution is rejected');
  console.log('MyPromise ok');
}, 60);
```

**Complexity:** O(1) per `then`; each settlement runs its queued handlers once.
**Edge cases:** executor throws (reject); resolve called twice (ignored); resolving with itself (TypeError); thenables that call back twice or throw; handlers registered after settlement still run asynchronously. Real promises also track unhandled rejections and implement the full Promises/A+ resolution procedure.

## 12. call, apply and bind polyfills

`call`/`apply` invoke a function with a chosen `this`; the trick is to temporarily make the function a method of that object. `bind` returns a wrapper that fixes `this` and leading arguments, and still works with `new`.

```js
Function.prototype.myCall = function (thisArg, ...args) {
  if (typeof this !== 'function') throw new TypeError('myCall must be called on a function');
  // Sloppy-mode semantics: null/undefined become globalThis, primitives are boxed
  const context = thisArg === null || thisArg === undefined ? globalThis : Object(thisArg);
  const key = Symbol('fn'); // unique key: cannot clash with existing properties
  context[key] = this;      // calling context[key]() makes `this` the context
  try {
    return context[key](...args);
  } finally {
    delete context[key];
  }
};

Function.prototype.myApply = function (thisArg, argsArray = []) {
  return this.myCall(thisArg, ...argsArray);
};

Function.prototype.myBind = function (thisArg, ...boundArgs) {
  const targetFn = this;
  return function bound(...args) {
    if (new.target) return new targetFn(...boundArgs, ...args); // `new` ignores the bound this
    return targetFn.myCall(thisArg, ...boundArgs, ...args);
  };
};

// Tests
function introduce(greeting, punctuation) {
  return `${greeting}, I am ${this.name}${punctuation}`;
}
const sara = { name: 'Sara' };

console.assert(introduce.myCall(sara, 'Hi', '!') === 'Hi, I am Sara!', 'call');
console.assert(introduce.myApply(sara, ['Hello', '.']) === 'Hello, I am Sara.', 'apply');
console.assert(Object.getOwnPropertySymbols(sara).length === 0, 'the temporary key is removed');

const greetSara = introduce.myBind(sara, 'Hey');
console.assert(greetSara('?') === 'Hey, I am Sara?', 'bind with partial application');
console.assert(greetSara.myCall({ name: 'Other' }, '!') === 'Hey, I am Sara!', 'a bound this cannot be overridden');

function Person(name) { this.name = name; }
const BoundPerson = Person.myBind({ name: 'ignored' });
const ali = new BoundPerson('Ali');
console.assert(ali.name === 'Ali' && ali instanceof Person, 'new wins over bind');

function getThis() { return this; }
console.assert(typeof getThis.myCall(5) === 'object', 'primitive thisArg is boxed');
console.log('call/apply/bind ok');
```

**Complexity:** O(n) in the number of arguments.
**Edge cases:** `null`/`undefined` thisArg; primitives; frozen objects (cannot attach the temporary key, a real limitation of this trick); `new` on a bound function; the native bound function has no `prototype` property, unlike this wrapper.

## 13. Array.prototype.map, filter and reduce polyfills

```js
Array.prototype.myMap = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(`${String(callback)} is not a function`);
  const result = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) result[i] = callback.call(thisArg, this[i], i, this); // skip holes
  }
  return result;
};

Array.prototype.myFilter = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(`${String(callback)} is not a function`);
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) result.push(this[i]);
  }
  return result;
};

Array.prototype.myReduce = function (callback, ...initial) {
  if (typeof callback !== 'function') throw new TypeError(`${String(callback)} is not a function`);
  let i = 0;
  let acc;
  if (initial.length > 0) {
    acc = initial[0];                                  // explicit initial value (even undefined)
  } else {
    while (i < this.length && !(i in this)) i++;       // first existing element
    if (i >= this.length) throw new TypeError('Reduce of empty array with no initial value');
    acc = this[i++];
  }
  for (; i < this.length; i++) {
    if (i in this) acc = callback(acc, this[i], i, this);
  }
  return acc;
};

// Tests
const nums = [1, 2, 3, 4];
console.assert(nums.myMap((n) => n * 2).join() === '2,4,6,8', 'map');
console.assert(nums.myFilter((n) => n % 2 === 0).join() === '2,4', 'filter');
console.assert(nums.myReduce((total, n) => total + n) === 10, 'reduce without initial value');
console.assert(nums.myReduce((total, n) => total + n, 10) === 20, 'reduce with initial value');
console.assert([[1], [2, 3]].myReduce((flat, arr) => flat.concat(arr), []).join() === '1,2,3', 'reduce to array');

const sparse = [1, , 3]; // eslint-disable-line no-sparse-arrays
const mapped = sparse.myMap((n) => n * 10);
console.assert(mapped.length === 3 && !(1 in mapped) && mapped[2] === 30, 'holes are preserved by map');
console.assert(sparse.myReduce((a, b) => a + b) === 4, 'holes are skipped by reduce');

const context = { factor: 3 };
console.assert(nums.myMap(function (n) { return n * this.factor; }, context)[0] === 3, 'thisArg');

let threw = false;
try {
  [].myReduce((a, b) => a + b);
} catch (e) {
  threw = e instanceof TypeError;
}
console.assert(threw && [].myReduce((a, b) => a + b, 0) === 0, 'empty array rules');
console.log('map/filter/reduce ok');
```

**Complexity:** O(n) time; `map` and `filter` use O(n) extra space.
**Edge cases:** holes in sparse arrays; `thisArg`; reduce with no initial value on an empty array (TypeError) versus an explicit `undefined` initial value; never patch built-in prototypes in production code (do it only in exercises like this).

## 14. EventEmitter (on, off, once, emit)

```js
class EventEmitter {
  #events = new Map(); // event name -> array of listeners

  on(event, listener) {
    if (!this.#events.has(event)) this.#events.set(event, []);
    this.#events.get(event).push(listener);
    return this; // chainable
  }

  off(event, listener) {
    const listeners = this.#events.get(event);
    if (!listeners) return this;
    // Match the listener itself or the original behind a once() wrapper
    const index = listeners.findIndex((l) => l === listener || l.original === listener);
    if (index !== -1) listeners.splice(index, 1);
    if (listeners.length === 0) this.#events.delete(event);
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      this.off(event, wrapper);
      listener.apply(this, args);
    };
    wrapper.original = listener;
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.#events.get(event);
    if (!listeners) return false;
    [...listeners].forEach((listener) => listener.apply(this, args)); // copy: safe if listeners change
    return true;
  }

  listenerCount(event) {
    return this.#events.get(event)?.length ?? 0;
  }
}

// Tests
const emitter = new EventEmitter();
const received = [];
const onMessage = (text, from) => received.push(`${from}: ${text}`);

emitter.on('message', onMessage);
console.assert(emitter.emit('message', 'hi', 'Sara') === true, 'emit returns true with listeners');
console.assert(emitter.emit('unknown') === false, 'emit returns false without listeners');

let readyCount = 0;
emitter.once('ready', () => readyCount++);
emitter.emit('ready');
emitter.emit('ready');
console.assert(readyCount === 1, 'once fires a single time');

emitter.off('message', onMessage);
emitter.emit('message', 'ignored', 'Ali');
console.assert(received.length === 1 && received[0] === 'Sara: hi', 'off removes the listener');

const neverRuns = () => { throw new Error('should not run'); };
emitter.once('cancelled', neverRuns);
emitter.off('cancelled', neverRuns); // remove a once() listener by its original reference
console.assert(emitter.listenerCount('cancelled') === 0, 'once listener removed via off');

const calls = [];
const first = () => { calls.push('first'); emitter.off('tick', first); };
emitter.on('tick', first).on('tick', () => calls.push('second'));
emitter.emit('tick');
console.assert(calls.join() === 'first,second', 'a listener removing itself does not skip others');
console.log('EventEmitter ok');
```

**Complexity:** `on` O(1); `off` and `emit` O(k) for k listeners of that event.
**Edge cases:** removing a `once` listener before it fires; listeners added or removed during `emit` (iterate a copy); duplicate registrations; a throwing listener stops the loop (Node's emitter does the same, and throws on an `'error'` event with no listener); returning an unsubscribe function from `on` is a handy variant for React effects.

## 15. Retry with exponential backoff

Retry a failing async operation with growing delays: 100 ms, 200 ms, 400 ms, capped. Jitter (randomizing the delay) prevents many clients from retrying in lockstep and hammering a recovering server.

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function retry(fn, options = {}) {
  const {
    retries = 3,              // retries after the first attempt
    baseDelay = 100,
    factor = 2,
    maxDelay = 5000,
    jitter = true,
    shouldRetry = () => true, // for example: only network errors and 5xx/429 responses
  } = options;

  for (let attempt = 0; ; attempt++) {
    try {
      return await fn(attempt);
    } catch (error) {
      if (attempt >= retries || !shouldRetry(error)) throw error;
      const backoff = Math.min(maxDelay, baseDelay * factor ** attempt);
      await sleep(jitter ? Math.random() * backoff : backoff); // "full jitter"
    }
  }
}

// Tests
(async () => {
  let calls = 0;
  const flaky = async () => {
    calls++;
    if (calls < 3) throw new Error(`fail ${calls}`);
    return 'success';
  };
  const start = Date.now();
  const result = await retry(flaky, { retries: 5, baseDelay: 10, jitter: false });
  const elapsed = Date.now() - start;
  console.assert(result === 'success' && calls === 3, 'succeeds on the third attempt');
  console.assert(elapsed >= 25, `waited about 10 + 20 ms of backoff (took ${elapsed} ms)`);

  let attempts = 0;
  const failure = await retry(async () => { attempts++; throw new Error('always'); }, { retries: 2, baseDelay: 5 })
    .catch((e) => e);
  console.assert(failure.message === 'always' && attempts === 3, 'gives up after retries + 1 attempts');

  let clientErrors = 0;
  const badRequest = async () => {
    clientErrors++;
    throw Object.assign(new Error('bad request'), { status: 400 });
  };
  await retry(badRequest, { shouldRetry: (e) => e.status >= 500, baseDelay: 5 }).catch(() => {});
  console.assert(clientErrors === 1, '4xx errors are not retried');
  console.log('retry ok');
})();
```

**Complexity:** at most `retries + 1` attempts; worst-case waiting time is the sum of the capped delays.
**Edge cases:** retry only idempotent operations (never blindly retry a payment or an order creation without an idempotency key); do not retry 4xx except 429; honor a `Retry-After` header when the API sends one; add an overall timeout or `AbortSignal`. Sidekiq retries failed jobs with its own backoff, which is why jobs must be idempotent.

## 16. Promise pool (concurrency limit)

Run many async tasks with at most `limit` in flight, for example syncing 1,000 listings without hitting an API's rate limit. Tasks are functions, so nothing starts until a worker picks it up.

```js
async function promisePool(tasks, limit) {
  const results = new Array(tasks.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < tasks.length) {
      const index = nextIndex++;               // claimed synchronously: no race on one thread
      results[index] = await tasks[index]();
    }
  }

  const workerCount = Math.max(0, Math.min(limit, tasks.length));
  await Promise.all(Array.from({ length: workerCount }, () => worker()));
  return results;
}

// Tests
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
let active = 0;
let maxActive = 0;
const makeTask = (id, ms) => async () => {
  active++;
  maxActive = Math.max(maxActive, active);
  await wait(ms);
  active--;
  return id;
};

(async () => {
  const tasks = [makeTask(1, 30), makeTask(2, 10), makeTask(3, 20), makeTask(4, 5), makeTask(5, 15)];
  const results = await promisePool(tasks, 2);
  console.assert(results.join() === '1,2,3,4,5', 'results keep input order');
  console.assert(maxActive === 2, `never more than 2 tasks at once (saw ${maxActive})`);
  console.assert((await promisePool([], 3)).length === 0, 'empty task list');
  const error = await promisePool([async () => { throw new Error('task failed'); }], 2).catch((e) => e);
  console.assert(error.message === 'task failed', 'rejects when a task fails');
  console.log('promise pool ok');
})();
```

**Complexity:** O(n) bookkeeping; total time is roughly the sum of task durations divided by `limit`.
**Edge cases:** tasks must be functions (promises start running the moment they are created); `limit` larger than the task count; a failing task rejects the pool but other workers keep going (wrap tasks for allSettled-style results); libraries such as p-limit do the same in production.

## 17. LRU cache with Map

Least Recently Used cache with O(1) `get` and `put` (LeetCode 146). A `Map` iterates in insertion order, so the first key is always the least recently used one, provided every access re-inserts the key.

```js
class LRUCache {
  #capacity;
  #map = new Map(); // first entry = least recently used, last = most recently used

  constructor(capacity) {
    if (!Number.isInteger(capacity) || capacity <= 0) throw new RangeError('capacity must be a positive integer');
    this.#capacity = capacity;
  }

  get(key) {
    if (!this.#map.has(key)) return undefined;
    const value = this.#map.get(key);
    this.#map.delete(key); // move to the most recent position
    this.#map.set(key, value);
    return value;
  }

  put(key, value) {
    if (this.#map.has(key)) this.#map.delete(key);
    this.#map.set(key, value);
    if (this.#map.size > this.#capacity) {
      this.#map.delete(this.#map.keys().next().value); // evict the least recently used
    }
  }

  get size() {
    return this.#map.size;
  }
}

// Tests
const cache = new LRUCache(2);
cache.put('a', 1);
cache.put('b', 2);
console.assert(cache.get('a') === 1, 'a becomes the most recently used');
cache.put('c', 3); // evicts b
console.assert(cache.get('b') === undefined, 'b was evicted');
console.assert(cache.get('c') === 3 && cache.size === 2, 'capacity respected');
cache.put('a', 10); // update moves a to the most recent position
cache.put('d', 4);  // evicts c
console.assert(cache.get('c') === undefined && cache.get('a') === 10 && cache.get('d') === 4, 'update refreshes recency');
let threw = false;
try {
  new LRUCache(0);
} catch (e) {
  threw = e instanceof RangeError;
}
console.assert(threw, 'invalid capacity throws');
console.log('LRU cache ok');
```

**Complexity:** O(1) average for `get` and `put`; O(capacity) space.
**Edge cases:** updating an existing key must refresh its position; capacity 1; `get` of a missing key (LeetCode expects `-1`); without `Map` ordering, the classic solution is a hash map plus a doubly linked list.

## Common pitfalls
- Losing `this` or arguments in wrappers (debounce, throttle, memoize, once): use a `function` wrapper and `fn.apply(this, args)`.
- Recreating a debounced or throttled function on every React render, so the timer never accumulates calls.
- `Promise.all` versions that push results in completion order, never resolve for empty input, or forget plain values.
- Deep clone or deep equal without cycle handling (stack overflow on the first self-reference).
- Retrying non-idempotent operations, or retrying 4xx errors that will never succeed.
- Passing already-started promises to a pool instead of task functions, so the limit does nothing.
- Unbounded memo caches in a long-running Node process.
- Patching built-in prototypes outside of an exercise.

## How to talk about this in an interview
- Narrate the process: restate the API, list edge cases, write the simple version, run one example by hand, then improve. Mention complexity and edge cases before you are asked.
- Connect the patterns to real systems: retries with backoff and concurrency limits are what a third-party sync needs (such as the PriceCharting integration on CrystalCommerce, where background jobs sync listings, prices, stock and orders), even when the jobs themselves are Ruby; debounce is how search inputs in React and React Native screens avoid flooding the API.
  > **Fill in:** did the PriceCharting sync or other integrations use retries, rate limiting or batching? How were failures surfaced?
- If you forget a detail of the native API, say what you would check ("native `allSettled` returns `{ status, value }` or `{ status, reason }`") and keep going; reasoning beats recall.

## Interview questions

**Q1. Implement debounce and explain how it differs from throttle.**
Debounce keeps a timer in a closure; every call clears it and starts a new one, so `fn` runs only after `wait` ms without calls. Throttle instead runs at most once per `wait` ms during continuous calls, usually with a trailing call for the latest arguments. Debounce suits search inputs; throttle suits scroll handlers.
*Follow-up:* What does `leading: true` change? The first call of a burst runs immediately instead of waiting.

**Q2. Why do debounce and throttle use a `function` wrapper and `fn.apply(this, args)`?**
The wrapper is what callers invoke, often as a method or event handler, so its `this` and arguments must be forwarded to `fn`. An arrow wrapper would capture the outer `this`, and calling `fn()` directly would drop both.
*Follow-up:* Where does this matter in practice? Debouncing an object method or a DOM handler that reads `this`.

**Q3. How does your deep clone handle cycles, and why a WeakMap?**
Before recursing into an object, the copy is registered in a `WeakMap` keyed by the original; meeting the original again returns the existing copy, which ends cycles and keeps shared references shared. A `WeakMap` does not keep the originals alive after cloning finishes.
*Follow-up:* What does it not copy? Non-enumerable properties, property flags, private fields; functions are shared.

**Q4. When would you use `structuredClone` instead of your own deep clone or JSON?**
Almost always for plain data: it handles Date, Map, Set, typed arrays and cycles natively. Use a custom clone when you need prototypes or class instances preserved, or special handling; avoid JSON cloning except for plain JSON data because it drops `undefined` and turns Dates into strings.
*Follow-up:* What does `structuredClone` throw on? Functions and DOM nodes (`DataCloneError`).

**Q5. How does your `Promise.all` keep the order, and what about empty input or plain values?**
Results are written by index (`results[index] = value`) and a counter tracks how many are left, so completion order does not matter. Empty input resolves immediately with `[]`, and every item is wrapped in `Promise.resolve` so plain values count as fulfilled.
*Follow-up:* Does it stop the other operations on rejection? No, it only stops waiting for them.

**Q6. How can you implement `Promise.allSettled` on top of `Promise.all`?**
Map each input to `Promise.resolve(item).then(value => ({ status: 'fulfilled', value }), reason => ({ status: 'rejected', reason }))`; these never reject, so `Promise.all` over them resolves with every outcome.
*Follow-up:* And `race`? Attach `resolve` and `reject` of one outer promise to every input; the first settlement wins.

**Q7. What rules must a promise implementation follow?**
Settle exactly once; `then` returns a new promise; handlers run asynchronously even for settled promises; a handler's return value resolves the next promise and a throw rejects it; thenables are adopted; non-function handlers pass values through; resolving a promise with itself is a TypeError.
*Follow-up:* Why microtasks? So handlers run soon after the current code, consistently async, before the next task.

**Q8. How does `bind` behave with `new`, and how does your polyfill handle it?**
`new` overrides the bound `this`: a fresh object is created with the target's prototype, and pre-bound arguments are still prepended. The polyfill checks `new.target` and, if set, returns `new targetFn(...boundArgs, ...args)`.
*Follow-up:* How do `call` and `apply` set `this` without native support? Attach the function to the context under a unique Symbol key, call it as a method, then delete the key.

**Q9. How do you build an O(1) LRU cache? What if `Map` ordering was not available?**
With a `Map`, `get` deletes and re-inserts the key (making it most recent) and `put` evicts the first key once over capacity, all O(1) on average. Without ordered maps, combine a hash map (key to node) with a doubly linked list ordered by recency, moving nodes to the head on access and removing from the tail.
*Follow-up:* Where are LRU caches used? Memoization with bounded memory, HTTP and query caches, database buffer pools.

**Q10. How would you run 1,000 API calls with at most 5 in flight?**
Represent each call as a function, then start 5 workers that loop, each claiming the next index from a shared counter and awaiting that task, storing results by index. When all workers finish, return the results. Add retries per task and decide between fail-fast and collect-all.
*Follow-up:* Why must tasks be functions? A promise starts running when created, so passing promises would run all 1,000 at once.

**Q11. When should you retry, and why add jitter?**
Retry transient failures (network errors, timeouts, 5xx, 429) on idempotent operations, with exponential backoff and a cap. Jitter randomizes delays so many clients that failed together do not retry at the same instant and overload the recovering service.
*Follow-up:* What makes a non-idempotent request safe to retry? An idempotency key the server uses to deduplicate.

**Q12. How does memoize choose cache keys, and when is memoization a bad idea?**
By default the single argument itself (objects by identity) or a serialized form of all arguments; a custom resolver can pick a field such as an id. Avoid it for impure or time-dependent functions, cheap functions, and inputs with huge variety where the cache only grows.
*Follow-up:* How is this related to React's `useMemo`? Same idea, but `useMemo` caches only the latest result per component and compares dependencies with `Object.is`.

**Q13. How do you implement `once`, and how does an emitter's `once` relate to it?**
`once` returns a wrapper with a `called` flag and a cached result in a closure, invoking the original only the first time. An emitter's `once` registers a wrapper that removes itself before calling the listener, and keeps a reference to the original so `off(original)` still works.
*Follow-up:* Why copy the listener array before emitting? A listener removing itself mid-loop would otherwise make the loop skip the next listener.

**Q14. Why write an iterative flatten with a stack?**
Recursion depth equals nesting depth, so extremely deep arrays can overflow the call stack; an explicit stack moves that state to the heap. The trade-off is slightly trickier ordering (pop from the end, then reverse).
*Follow-up:* How would you support a depth limit iteratively? Push `[item, depth]` pairs onto the stack.

## Further reading
- [MDN: Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [MDN: Function.prototype.bind()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
