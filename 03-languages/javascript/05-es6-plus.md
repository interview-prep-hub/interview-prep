# ES6+ Features (ES2015 and Later)

> **Tier:** T1 (know cold) · **Related:** [Types and coercion](01-basics-types-coercion.md), [this, prototypes and classes](03-this-prototypes-classes.md), [Async and promises](04-async-event-loop-promises.md), [Implement from scratch](07-implement-from-scratch.md), [Node.js fundamentals](../../09-nodejs-backend/01-nodejs-fundamentals.md)
> **On your resume:** modern syntax is what you write daily in React, React Native (Expo) and TypeScript platform code at Devntech; modules and bundling matter for the Next.js work at RemoteDev.

## Quick revision
- ES2015 (ES6) was the big release: `let`/`const`, arrows, classes, template literals, destructuring, default/rest/spread, modules, promises, Map/Set/WeakMap, Symbol, iterators, generators, Proxy/Reflect. Yearly editions followed.
- Arrow functions: no own `this`, `arguments`, `super` or `new.target`; not constructible; no `prototype`. Return an object literal with parentheses: `() => ({ a: 1 })`.
- Destructuring defaults apply only for `undefined` (not `null`); destructuring `null`/`undefined` throws a TypeError.
- Spread expands (`[...a]`, `{ ...obj }`, `fn(...args)`); rest collects (`(...args)`, `const { a, ...rest } = obj`). Both are shallow.
- Default parameters are evaluated on every call and can use earlier parameters.
- ESM: static `import`/`export`, always strict, live read-only bindings, tree-shakable, top-level await. CommonJS: `require`/`module.exports`, synchronous, dynamic, exports a value snapshot.
- `import()` returns a promise: code splitting, lazy routes, `React.lazy`.
- `Map`: any key type, insertion order, `size`, fast add/delete. `Set`: unique values, O(1) `has`. Objects for fixed-shape records and JSON.
- `WeakMap`/`WeakSet`: object keys held weakly, not iterable, no `size`. Per-object metadata, caches and cycle detection without leaks.
- Symbols are unique keys skipped by `Object.keys`, `for...in` and `JSON.stringify`. Well-known symbols (`Symbol.iterator`, `Symbol.asyncIterator`, `Symbol.toPrimitive`) customize the language.
- Iterable = has `[Symbol.iterator]()` returning an iterator whose `next()` gives `{ value, done }`. Arrays, strings, Map, Set, NodeList are iterable; plain objects are not.
- Generators (`function*`, `yield`) are lazy, pausable iterators: sequences, pagination, infinite streams.
- Copying array methods (ES2023): `toSorted`, `toReversed`, `toSpliced`, `with`. Also `at(-1)` (ES2022), `findLast` (ES2023), `flat`/`flatMap` (ES2019).
- `sort()` mutates and compares strings by default: `[10, 1, 2].sort()` gives `[1, 10, 2]`.
- `Object.entries` + `Object.fromEntries` transform objects; `Object.groupBy` (ES2024) returns a null-prototype object.
- `Proxy` intercepts operations with traps; `Reflect` performs the default operation. Vue reactivity, Immer and MobX are built on proxies.

## 1. Arrow functions

| | Regular function | Arrow function |
|---|---|---|
| `this` | set by the call | inherited from the enclosing scope |
| `arguments` | own `arguments` object | none (sees the enclosing function's) |
| `new` | works as a constructor | TypeError: not a constructor |
| `prototype` | yes | no |
| Generator form | `function*` | not possible |
| Best for | methods, constructors, generators | callbacks, short transforms, keeping `this` |

```js
const double = (n) => n * 2;                    // implicit return
const toUser = (row) => ({ id: row.id });       // parentheses to return an object literal
const sum = (...nums) => nums.reduce((total, n) => total + n, 0); // rest instead of arguments

class Poller {
  constructor() { this.attempts = 0; }
  start() {
    setInterval(() => { this.attempts++; }, 1000); // the arrow keeps the instance as this
  }
}
```

`() => { a: 1 }` returns `undefined`: the braces are parsed as a block containing a label. Avoid arrows as object methods, prototype methods, and anywhere you need `arguments` or a constructor (see [this, prototypes and classes](03-this-prototypes-classes.md)).

## 2. Destructuring

```js
// Objects: nested, renamed, defaulted, rest
const response = { data: { user: { id: 7, name: 'Sara' } }, status: 200 };
const {
  status,
  data: { user: { name: userName, role = 'member' } },
} = response;
// status 200, userName 'Sara', role 'member'

const { id, ...withoutId } = { id: 1, title: 'Desk', price: 500 }; // withoutId: { title, price }

// Arrays: skip, default, rest, swap
const [first, , third = 'none', ...others] = [1, 2, undefined, 4, 5]; // 1, 'none', [4, 5]
let a = 1;
let b = 2;
[a, b] = [b, a];

// Parameter destructuring; `= {}` lets callers omit the argument
function listOrders({ page = 1, limit = 20, status } = {}) {
  return { page, limit, status };
}
listOrders();            // { page: 1, limit: 20, status: undefined }
listOrders({ page: 3 }); // { page: 3, limit: 20, status: undefined }
```

- Defaults apply only for `undefined`: `const { x = 10 } = { x: null }` gives `null`. This matters with APIs (Rails `nil`) that send `null`.
- Destructuring `null` or `undefined` throws `TypeError: Cannot destructure property 'z' of 'null'`.
- `const [count, setCount] = useState(0)` is array destructuring of a tuple; that is why you can name the pair anything.

## 3. Spread and rest

Same `...` syntax, opposite jobs: spread expands an iterable or object; rest collects what is left.

```js
const base = { theme: 'light', lang: 'en' };
const prefs = { ...base, lang: 'ur' };            // later keys win: { theme: 'light', lang: 'ur' }
const merged = [...[1, 2], ...new Set([2, 3])];   // [1, 2, 2, 3]: any iterable works
const chars = [...'abc'];                         // ['a', 'b', 'c']
Math.max(...[4, 9, 2]);                           // 9

function log(level, ...messages) {                // rest parameter: a real array
  console.log(`[${level}]`, ...messages);
}
```

- Both are shallow: nested objects stay shared (see [shallow vs deep copy](01-basics-types-coercion.md)).
- Object spread copies own enumerable properties only; the prototype is not copied and getters are invoked.
- `Math.max(...hugeArray)` can exceed the engine's argument limit; use `reduce` for very large arrays. `Math.max(...[])` is `-Infinity`.

## 4. Default parameters

```js
function addItem(item, list = []) {   // evaluated on every call, so no shared default (unlike Python)
  list.push(item);
  return list;
}
addItem(1); // [1]
addItem(2); // [2]

function createRange(start, end = start + 10) { // can use earlier parameters
  return [start, end];
}
createRange(5); // [5, 15]
```

Defaults trigger only for `undefined`, not `null`. Parameters are evaluated left to right, so a default cannot use a later parameter (TDZ, see [hoisting](02-scope-closures-hoisting.md)).

## 5. Enhanced object literals and computed keys

```js
const id = 7;
const field = 'status';

const listing = {
  id,                                  // shorthand for id: id
  [field]: 'active',                   // computed key
  [`${field}ChangedAt`]: Date.now(),
  describe() {                         // method shorthand (supports super)
    return `Listing ${this.id}`;
  },
};

// Common form-state pattern: update a dynamic key immutably
const handleChange = (form, event) => ({ ...form, [event.target.name]: event.target.value });
```

## 6. Modules: ES modules vs CommonJS

```js
// pricing.js (ES module)
export const TAX_RATE = 0.18;                    // named export
export function withTax(cents) {
  return Math.round(cents * (1 + TAX_RATE));
}
export default function formatPrice(cents) {    // default export (at most one)
  return `Rs ${(cents / 100).toFixed(2)}`;
}

// app.js
import formatPrice, { withTax, TAX_RATE as RATE } from './pricing.js';
import * as pricing from './pricing.js';        // namespace object
export { withTax } from './pricing.js';          // re-export (barrel files)

// Dynamic import: returns a promise and creates a separate chunk (code splitting)
async function showChart() {
  const { renderChart } = await import('./chart.js');
  renderChart();
}
```

**Live bindings:** an ESM import is a live, read-only view of the exporter's variable. CommonJS copies the value into `module.exports`.

```js
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
increment();
console.log(count);   // 1: live binding
// count = 5;         // TypeError: Assignment to constant variable.
```

```js
// counter.cjs
let count = 0;
module.exports = { count, increment() { count++; } };

// main.cjs
const counter = require('./counter.cjs');
counter.increment();
console.log(counter.count); // 0: the number was copied when the object was created
```

| | ES modules | CommonJS |
|---|---|---|
| Syntax | `import` / `export` | `require()` / `module.exports` |
| Loading | static structure, linked before running, async loading | synchronous, at runtime, anywhere |
| Bindings | live and read-only | copied values (references for objects) |
| Strict mode | always | opt-in |
| Top-level await | yes | no |
| Top-level `this` | `undefined` | `module.exports` |
| File path helpers | `import.meta.url` (plus `import.meta.dirname` in recent Node) | `__dirname`, `__filename` |
| Tree shaking | yes (static analysis) | limited |
| In Node | `.mjs`, or `"type": "module"` in package.json | `.cjs`, or the default without `"type": "module"` |

ESM can import CommonJS (the `module.exports` object becomes the default import). CommonJS historically needed `await import()` to load ESM; recent Node versions also allow `require()` of ES modules, so check the release notes for your version. Typical errors from mixing formats: "require is not defined in ES module scope" and "Cannot use import statement outside a module".

**Named vs default exports:** named exports keep one name across the codebase, autocomplete well and make renames safer; default exports let each importer choose a name. Many teams prefer named exports except where a framework requires a default (Next.js pages and layouts, Expo Router screens, `React.lazy` targets).

## 7. Map and Set vs objects and arrays

```js
const stock = new Map();
stock.set('sku-1', 10).set(42, 'numeric key').set({ id: 1 }, 'object key');
stock.get('sku-1');   // 10
stock.has(42);        // true
stock.size;           // 3
for (const [key, value] of stock) console.log(key, value); // insertion order

const unique = [...new Set([3, 1, 3, 2, 1])]; // [3, 1, 2]: dedupe, first occurrences kept
```

| | Object | Map |
|---|---|---|
| Key types | strings and symbols (numbers become strings) | any value, including objects |
| Order | insertion order, except integer-like keys come first in ascending order | insertion order |
| Size | `Object.keys(obj).length` | `map.size` |
| Inherited keys | yes (`toString`, `__proto__` traps) unless `Object.create(null)` | none |
| Iteration | via `Object.entries(obj)` | directly iterable |
| Frequent add/delete | slower | optimized for it |
| JSON | native | convert with `Object.fromEntries(map)` |

Use objects for records with known fields and data you serialize; use `Map` for dictionaries with dynamic or non-string keys and heavy add/remove. Use a `Set` instead of `array.includes` inside loops: `has` is O(1) versus O(n), turning an O(n^2) filter into O(n).

## 8. WeakMap and WeakSet

Keys must be objects (recent editions also accept non-registered symbols) and are held weakly: when nothing else references a key, its entry can be garbage collected. Because GC timing is unpredictable, these collections are not iterable and have no `size`.

```js
// Memoize by object identity: entries disappear together with the object
const summaryCache = new WeakMap();
function orderTotal(order) {
  if (!summaryCache.has(order)) {
    summaryCache.set(order, order.items.reduce((sum, item) => sum + item.priceCents, 0));
  }
  return summaryCache.get(order);
}

// WeakSet: remember objects already visited (cycle detection) without keeping them alive
const visited = new WeakSet();
```

Use cases: metadata for objects you do not own (DOM nodes, third-party objects), caches keyed by objects, and cycle detection in deep clone (see [implement from scratch](07-implement-from-scratch.md)). `new WeakMap().set('key', 1)` throws `TypeError: Invalid value used as weak map key`.

## 9. Symbol

```js
const id = Symbol('id');                        // the description is only for debugging
const user = { name: 'Sara', [id]: 123 };
Object.keys(user);                              // ['name']: symbol keys are skipped
JSON.stringify(user);                           // '{"name":"Sara"}'
Object.getOwnPropertySymbols(user);             // [Symbol(id)]
Symbol('a') === Symbol('a');                    // false: every symbol is unique
Symbol.for('app.id') === Symbol.for('app.id');  // true: global registry
```

| Well-known symbol | Customizes |
|---|---|
| `Symbol.iterator` | `for...of`, spread, array destructuring |
| `Symbol.asyncIterator` | `for await...of` |
| `Symbol.toPrimitive` | conversion to primitives (see [coercion](01-basics-types-coercion.md)) |
| `Symbol.hasInstance` | `instanceof` |
| `Symbol.toStringTag` | `Object.prototype.toString` output |

## 10. Iterables and iterators

An **iterable** has a `[Symbol.iterator]()` method that returns an **iterator**: an object whose `next()` returns `{ value, done }`. Consumers include `for...of`, spread, array destructuring, `Array.from`, `new Map(iterable)` and `Promise.all`.

```js
const range = {
  from: 1,
  to: 4,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next: () =>
        current <= last ? { value: current++, done: false } : { value: undefined, done: true },
    };
  },
};

[...range];                       // [1, 2, 3, 4]
Array.from(range, (n) => n * 2);  // [2, 4, 6, 8]
Array.from({ length: 3 }, (_, i) => i); // [0, 1, 2]: array-likes work with Array.from too
```

Plain objects are not iterable (`for (const x of {})` throws `TypeError: {} is not iterable`); iterate `Object.entries(obj)` instead. `for...in` is a different loop: it walks enumerable string keys, including inherited ones, so avoid it for arrays.

## 11. Generators

A generator function (`function*`) returns an iterator. Each `yield` pauses the function and hands out a value; `next()` resumes it. Values are computed lazily, so infinite sequences are fine.

```js
function* idGenerator() {
  let id = 1;
  while (true) {
    const reset = yield id++;   // next(value) sends a value back in
    if (reset) id = 1;
  }
}
const ids = idGenerator();
ids.next().value;      // 1
ids.next().value;      // 2
ids.next(true).value;  // 1 (reset)

function* take(iterable, n) {   // lazy pipeline helper
  let i = 0;
  for (const value of iterable) {
    if (i++ >= n) return;
    yield value;
  }
}
[...take(idGenerator(), 3)];    // [1, 2, 3]: only three values are ever computed
```

- Generator objects are iterators and iterables, so `for...of` and spread work on them.
- `gen.return(value)` finishes a generator early (a `for...of` that `break`s calls it for you); `gen.throw(err)` raises an error at the paused `yield`.
- Async generators (`async function*`) pair with `for await...of` for pagination and streams (see [async](04-async-event-loop-promises.md)).
- Before async/await, libraries such as co and redux-saga used generators to write async flows.

## 12. Modern array and object methods

| Method | Edition | Mutates? | Note |
|---|---|---|---|
| `flat(depth = 1)`, `flatMap(fn)` | ES2019 | no | `flat(Infinity)` flattens fully; `flatMap` = map, then flat one level |
| `at(index)` | ES2022 | no | negative index from the end; also on strings |
| `findLast(fn)`, `findLastIndex(fn)` | ES2023 | no | search from the end |
| `toSorted`, `toReversed`, `toSpliced`, `with(i, v)` | ES2023 | no | copying versions of `sort`, `reverse`, `splice` and index assignment |
| `sort`, `reverse`, `splice` | original methods | **yes** | change the array in place and return it |
| `Object.entries` / `Object.fromEntries` | ES2017 / ES2019 | no | object to `[key, value]` pairs and back |
| `Object.groupBy(items, fn)` / `Map.groupBy` | ES2024 | no | null-prototype object / Map of arrays |

```js
const prices = [500, 120, 999];
const ascending = prices.toSorted((a, b) => a - b); // [120, 500, 999]; prices unchanged
prices.at(-1);                                       // 999
[10, 1, 2].sort();                                   // [1, 10, 2]: string comparison
[10, 1, 2].sort((a, b) => a - b);                    // [1, 2, 10]

// Transform an object: keep only non-zero totals
const totals = { a: 100, b: 0, c: 250 };
const nonZero = Object.fromEntries(Object.entries(totals).filter(([, cents]) => cents > 0));
// { a: 100, c: 250 }

const orders = [
  { id: 1, status: 'paid' },
  { id: 2, status: 'pending' },
  { id: 3, status: 'paid' },
];
const byStatus = Object.groupBy(orders, (order) => order.status);
// { paid: [order 1, order 3], pending: [order 2] }
```

`Object.groupBy` is available in current browsers and recent Node versions; check your targets (older browsers and some React Native engine versions lag) or fall back to `reduce`. The copying methods matter in React: `state.items.sort()` mutates state, `toSorted()` returns a new array.

## 13. Proxy and Reflect basics

A `Proxy` wraps an object and intercepts operations through **traps** (`get`, `set`, `has`, `deleteProperty`, `apply`, `construct` and more). `Reflect` exposes the default behavior of each trap, so you add logic and then forward.

```js
function withValidation(target) {
  return new Proxy(target, {
    set(obj, prop, value, receiver) {
      if (prop === 'age' && (!Number.isInteger(value) || value < 0)) {
        throw new TypeError('age must be a non-negative integer');
      }
      return Reflect.set(obj, prop, value, receiver); // default behavior
    },
    get(obj, prop, receiver) {
      if (!(prop in obj)) console.warn(`Unknown property: ${String(prop)}`);
      return Reflect.get(obj, prop, receiver);
    },
  });
}

const user = withValidation({ name: 'Sara', age: 25 });
user.age = 30;     // ok
// user.age = -1;  // TypeError
user.email;        // warns, returns undefined
```

Real uses: Vue 3 reactivity tracks reads and writes through proxies; Immer hands you a draft proxy you "mutate" and produces an immutable copy; MobX observables; validation, logging and test mocks. Costs: slower property access, harder debugging, and `proxy !== target`. You rarely write proxies in app code, but you should recognize them.

## Common pitfalls
- `() => { a: 1 }` returning `undefined`; wrap object literals in parentheses.
- Arrow functions as methods that need `this`.
- Expecting destructuring defaults to replace `null` sent by an API.
- Destructuring a value that may be undefined (`const { data } = response`) without a default.
- Treating spread as a deep copy.
- Inconsistent default exports (the same module imported under different names) and barrel files that re-export everything, slowing builds and hurting tree shaking.
- Objects as dictionaries with user-controlled keys (`__proto__`, `constructor`); use a `Map`.
- `sort()`, `reverse()` or `splice()` on React state; use the copying versions or copy first.
- `for...in` over arrays (string keys, inherited properties); use `for...of`.
- Assuming the newest methods exist in every runtime you ship to.

## How to talk about this in an interview
- Most of these features are invisible habits: destructured props, spread state updates, `import`/`export`, a `Map` for lookups. Name them when describing code ("I group bills by member with a Map, then spread them into new state").
- ESM vs CommonJS is a practical question: Next.js and Vite projects use ES modules, older Node code and many config files use CommonJS, and mixing them produces "require is not defined" or "Cannot use import statement outside a module". Explain the fixes (`"type": "module"`, file extensions, dynamic `import()`).
  > **Fill in:** a module-format or build error you fixed (Next.js at RemoteDev, Expo, the TypeScript platform) and what the fix was.
- For generators and Proxy, it is fine to say you rarely write them directly, then show where they appear (async iterators for pagination, Immer and Vue for proxies).
  > **Fill in:** a place you used `Map`/`Set`, grouping, or async iteration in real code.

## Interview questions

**Q1. How are arrow functions different from regular functions, beyond `this`?**
They have no own `arguments`, `super` or `new.target`, cannot be called with `new`, have no `prototype`, cannot be generators, and the expression form returns implicitly. Use rest parameters instead of `arguments`.
*Follow-up:* How do you return an object literal from an arrow? Wrap it in parentheses: `() => ({ id: 1 })`.

**Q2. How do destructuring defaults work, and when do they apply?**
`const { page = 1 } = query` uses the default only when the value is `undefined`, not `null`. You can rename (`{ data: user }`), nest and collect the rest (`...others`). Parameter destructuring with `= {}` lets callers omit the argument.
*Follow-up:* What happens when you destructure `null`? A TypeError.

**Q3. What is the difference between spread and rest?**
Same `...` syntax: spread expands an iterable or object into elements or properties (`[...a]`, `{ ...obj }`, `fn(...args)`); rest gathers the remaining items into an array or object (`function f(...args)`, `const { a, ...rest } = obj`). Rest must be last.
*Follow-up:* Is `{ ...obj }` a deep copy? No, shallow.

**Q4. Compare ES modules and CommonJS.**
ESM uses static `import`/`export`, is always strict, provides live read-only bindings, supports top-level await and enables tree shaking. CommonJS uses `require`/`module.exports`, loads synchronously at runtime and exports a value snapshot. Node picks the format from the file extension and the package.json `"type"` field.
*Follow-up:* Can CommonJS load ESM? With dynamic `import()`; recent Node versions also support `require()` of ES modules.

**Q5. Named vs default exports: which do you prefer?**
Named exports keep one name everywhere, autocomplete better, make renames safer and allow several exports per file; defaults let importers pick any name. Prefer named exports, except where a framework requires defaults (Next.js pages and layouts, Expo Router screens, `React.lazy`).
*Follow-up:* Can a module have both? Yes: one default plus any number of named exports.

**Q6. What are live bindings?**
An ES module import is a live, read-only reference to the exporter's variable: if the exporter updates it, importers see the new value, but importers cannot assign to it. CommonJS copies values into `module.exports`.
*Follow-up:* What error does assigning to an import give? `TypeError: Assignment to constant variable.`

**Q7. What is dynamic `import()`, and when do you use it?**
An expression that loads a module at runtime and returns a promise for its namespace object. Use it for code splitting (charts, editors, admin screens), lazy routes with `React.lazy`, conditional loading, and loading ESM from CommonJS.
*Follow-up:* How do bundlers treat it? As a split point that produces a separate chunk.

**Q8. When do you use a `Map` instead of an object, or a `Set` instead of an array?**
`Map` for dynamic or non-string keys, guaranteed insertion order, frequent add/delete and an easy `size`; objects for fixed-shape records and JSON. `Set` for uniqueness and O(1) membership checks instead of O(n) `includes`.
*Follow-up:* How do you dedupe an array? `[...new Set(arr)]`.

**Q9. What is a WeakMap for, and why can't you iterate it?**
Attaching data to objects without keeping them alive: per-object metadata, identity-based caches, cycle detection. Entries vanish when their key object is collected, and since GC timing is unpredictable, iteration and `size` would be nondeterministic, so they are not offered.
*Follow-up:* Can keys be strings? No; objects (or non-registered symbols in recent editions).

**Q10. What is a Symbol? Name some well-known symbols.**
A unique primitive, often used as a collision-free property key; symbol keys are skipped by `Object.keys`, `for...in` and `JSON.stringify`. `Symbol.for` uses a global registry. Well-known symbols include `Symbol.iterator`, `Symbol.asyncIterator`, `Symbol.toPrimitive` and `Symbol.hasInstance`.
*Follow-up:* Are symbol properties private? No; `Object.getOwnPropertySymbols` lists them.

**Q11. Iterable vs iterator: how do you make an object iterable?**
An iterable has a `[Symbol.iterator]()` method that returns an iterator, an object whose `next()` returns `{ value, done }`. Implement `[Symbol.iterator]` (often as a generator method, `*[Symbol.iterator]() { ... }`) and the object works with `for...of` and spread.
*Follow-up:* Why does `for (const x of {})` throw? Plain objects are not iterable; use `Object.entries`.

**Q12. What are generators, and where are they useful?**
`function*` functions that return an iterator; `yield` pauses and `next()` resumes, so values are produced lazily. Uses: infinite or large sequences, custom iterables, lazy pipelines, and async generators for paginated APIs and streams.
*Follow-up:* How do you send a value into a generator? `gen.next(value)` becomes the result of the paused `yield`.

**Q13. Which array methods mutate, and what are the non-mutating alternatives?**
`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill` and `copyWithin` mutate. ES2023 added `toSorted`, `toReversed`, `toSpliced` and `with`; `map`, `filter`, `slice`, `concat` and spread never mutated.
*Follow-up:* Why does it matter in React? A mutated array keeps its reference, so React may skip the re-render.

**Q14. Why does `[10, 1, 2].sort()` return `[1, 10, 2]`?**
Without a comparator, `sort` converts elements to strings and compares UTF-16 code units, so `'10'` sorts before `'2'`. Pass `(a, b) => a - b` for numbers or `a.localeCompare(b)` for text.
*Follow-up:* Is `sort` stable? Yes, required since ES2019.

**Q15. How do you filter or transform an object's keys?**
Convert to entries, use array methods, convert back: `Object.fromEntries(Object.entries(obj).filter(([key]) => allowed.includes(key)))`. The same pattern maps values or renames keys.
*Follow-up:* How do you group an array of objects by a field? `Object.groupBy(items, fn)` (ES2024) or `reduce` into an object or Map.

**Q16. What is a Proxy, and where is it used?**
A wrapper that intercepts operations through traps like `get`, `set` and `has`, usually forwarding to `Reflect` for the default behavior. Vue 3 reactivity, Immer drafts, MobX, validation layers and test mocks use proxies.
*Follow-up:* Downsides? Slower access, harder debugging, and the proxy is a different object from its target.

**Q17. `map` vs `forEach` vs `reduce`: when do you use each?**
`map` transforms each element into a new array of the same length; `forEach` performs side effects and returns `undefined`; `reduce` folds the array into a single value (a sum, an object, a Map). Prefer `map`/`filter` for readability, `reduce` for one aggregate.
*Follow-up:* Can you `break` out of `forEach`? No; use `for...of`, `some` or `find`.

**Q18. What is tree shaking, and why do ES modules enable it?**
Bundlers remove exports that nothing imports. That requires static `import`/`export`, so the bundler knows at build time what is used; dynamic `require` is hard to analyze. Top-level side effects can still block removal.
*Follow-up:* How does a package declare it is safe to shake? `"sideEffects": false` in package.json.

## Further reading
- [MDN: JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
- [MDN: Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)
- [MDN: Keyed collections](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Keyed_collections)
