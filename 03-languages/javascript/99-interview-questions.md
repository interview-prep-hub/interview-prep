# JavaScript: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. Every "what's the output" answer was checked in Node; browser-only items are marked.

## Types, coercion and equality (see [01-basics-types-coercion.md](01-basics-types-coercion.md))

**Q1. What are the primitive types in JavaScript?**
string, number, bigint, boolean, undefined, null and symbol. Everything else (arrays, functions, dates, maps) is an object. Primitives are immutable and compared by value.

**Q2. What's the output?**
```js
console.log(typeof null, typeof [], typeof function () {}, typeof NaN);
```
`object object function number`. `typeof null` is a historical bug; arrays are objects (use `Array.isArray`); NaN is a number value.

**Q3. What is the difference between `null` and `undefined`?**
`undefined` means "not assigned" (uninitialized variables, missing properties and arguments); `null` is an intentional "empty" you assign. `null == undefined` is true, `===` is false, and only `undefined` triggers default parameters.

**Q4. When would you ever use `==`?**
Only for `value == null`, which matches both `null` and `undefined`. Everywhere else use `===` so no hidden conversion happens.

**Q5. What's the output?**
```js
console.log([] + [], [] + {}, '5' - 2, '5' + 2);
```
`''`, `'[object Object]'`, `3`, `'52'`. Arrays become strings through ToPrimitive; `-` is numeric only; `+` concatenates when either side is a string.

**Q6. What's the output?**
```js
console.log(0.1 + 0.2 === 0.3, 0.1 + 0.2);
```
`false 0.30000000000000004`. IEEE 754 doubles cannot represent 0.1 or 0.2 exactly. Compare with a tolerance and keep money in integer minor units.

**Q7. What's the output?**
```js
console.log(null == 0, null >= 0, null == undefined, NaN === NaN);
```
`false true true false`. `null` loosely equals only `undefined`, but relational operators convert it to 0; NaN equals nothing, not even itself.

**Q8. List the falsy values.**
`false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is truthy, including `'0'`, `'false'`, `[]` and `{}`.

**Q9. What's the output?**
```js
console.log([] == ![], [] == [], '' == 0);
```
`true false true`. `![]` is false, so `[] == false` becomes `0 == 0`; two different arrays are different references; `''` converts to 0.

**Q10. How do you reliably check for NaN?**
`Number.isNaN(x)` or `Object.is(x, NaN)`. Global `isNaN` converts first, so `isNaN('abc')` is true and `isNaN('')` is false.

**Q11. What's the output?**
```js
console.log(typeof typeof 1, !!'false', !!'', +true, +'');
```
`string true false 1 0`. `typeof` always returns a string; any non-empty string is truthy; unary plus converts to a number.

**Q12. What's the output?**
```js
console.log(0 || 'a', 0 ?? 'a', '' || 'b', null ?? 'c');
```
`a 0 b c`. `||` falls back on any falsy value; `??` only on `null` and `undefined`.

**Q13. Shallow copy vs deep copy: how do you deep copy?**
Spread, `Object.assign` and `slice` copy only the top level. Use `structuredClone` for data (handles Date, Map, Set and cycles; throws on functions; drops prototypes) or a custom clone; `JSON.parse(JSON.stringify())` only suits plain JSON.

**Q14. What's the output?**
```js
const a = { n: 1, inner: { m: 1 } };
const b = { ...a };
b.n = 2;
b.inner.m = 2;
console.log(a.n, a.inner.m);
```
`1 2`. The spread copied `n` but `inner` is the same object in both.

**Q15. What's the output?**
```js
function update(obj) {
  obj.name = 'B';
  obj = { name: 'C' };
}
const person = { name: 'A' };
update(person);
console.log(person.name);
```
`B`. Arguments are passed by value, and an object's value is a reference: mutation is visible to the caller, reassigning the parameter is not.

**Q16. What is `Number.MAX_SAFE_INTEGER`, and what do you use beyond it?**
2^53 - 1 (9007199254740991). Above it integers lose precision, so use `BigInt` (`123n`) or keep large IDs as strings in APIs.

## Scope, closures and hoisting (see [02-scope-closures-hoisting.md](02-scope-closures-hoisting.md))

**Q17. What is a closure? Give a real use.**
A function that keeps access to the variables of the scope where it was created, even after that scope returned. Uses: private state, counters, memoization caches, debounce timers, React hooks.

**Q18. What's the output?**
```js
console.log(a);
var a = 5;
console.log(b);
let b = 10;
```
`undefined`, then `ReferenceError: Cannot access 'b' before initialization`. `var` is hoisted as `undefined`; `let` is in the temporal dead zone.

**Q19. What is the temporal dead zone?**
The time between entering a scope and the line that initializes a `let`, `const` or `class` binding; any access there (even `typeof`) throws a ReferenceError.

**Q20. What's the output, and how do you fix it?**
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
`3 3 3`. All callbacks share one function-scoped `i`, read after the loop ends. Use `let` (a binding per iteration), an IIFE, or pass `i` as setTimeout's third argument.

**Q21. What's the output?**
```js
foo();
bar();
function foo() { console.log('foo'); }
var bar = function () { console.log('bar'); };
```
`foo`, then `TypeError: bar is not a function`. Declarations are hoisted with their body; for the expression only `var bar` (undefined) is hoisted.

**Q22. What's the output?**
```js
var x = 1;
function f() {
  console.log(x);
  var x = 2;
}
f();
```
`undefined`. The inner `var x` is hoisted to the top of `f` and shadows the outer `x`.

**Q23. What's the output?**
```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}
const c1 = makeCounter();
const c2 = makeCounter();
c1();
c1();
console.log(c1(), c2());
```
`3 1`. Each call to `makeCounter` creates a separate `count`.

**Q24. What's the output?**
```js
for (var i = 0; i < 3; i++) {}
console.log(i);
for (let j = 0; j < 3; j++) {}
console.log(typeof j);
```
`3`, then `undefined`. `var` leaks out of the loop; `let` is block scoped (and `typeof` on an undeclared name returns `'undefined'`).

**Q25. What's the output?**
```js
let message = 'hello';
const say = () => console.log(message);
message = 'bye';
say();
```
`bye`. Closures capture variables, not snapshots of their values.

**Q26. What's the output?**
```js
console.log(typeof a);
var a = 1;
function a() {}
console.log(typeof a);
```
`function`, then `number`. The function declaration initializes `a` during creation; the assignment runs during execution.

**Q27. What is lexical scope?**
Scope decided by where code is written. A function resolves free variables in the scope where it was defined, walking outward through the scope chain, no matter where it is called.

**Q28. What is an IIFE, and why was it used?**
`(function () { ... })()`: a function invoked immediately to create a private scope, used before ES modules to avoid polluting the global scope (the module pattern). Today it survives mainly as an async IIFE in CommonJS.

**Q29. What happens when you assign to an undeclared variable?**
Sloppy mode silently creates a global; strict mode (ES modules, classes, `'use strict'`) throws a ReferenceError.

**Q30. What is a stale closure in React?**
A callback or effect that captured state from an old render, such as a `setInterval` in an effect with `[]` deps always reading the initial count. Fix with functional updates, correct dependencies or a ref.

**Q31. How can closures leak memory?**
They keep their whole environment reachable: forgotten listeners, intervals or subscriptions can retain big objects and detached DOM nodes. Remove listeners, clear timers, and bound caches.

**Q32. What is the call stack, and what causes "Maximum call stack size exceeded"?**
The LIFO stack of execution contexts for active function calls. Recursion without a reachable base case keeps pushing frames until the engine throws a RangeError.

## this, prototypes and classes (see [03-this-prototypes-classes.md](03-this-prototypes-classes.md))

**Q33. What are the rules for `this`?**
`new` -> the new object; `call`/`apply`/`bind` -> the given object; `obj.method()` -> `obj`; plain call -> `undefined` in strict mode, `globalThis` in sloppy mode. Arrow functions use the enclosing scope's `this`.

**Q34. What's the output?**
```js
'use strict';
const user = {
  name: 'Sara',
  greet() { return this?.name; },
};
const greet = user.greet;
console.log(user.greet(), greet());
```
`Sara undefined`. The extracted function is called without an object, so `this` is `undefined` in strict mode.

**Q35. What's the output (in an ES module)?**
```js
const obj = {
  value: 42,
  regular() { return this.value; },
  arrow: () => this?.value,
  nested() { return [1].map(() => this.value)[0]; },
};
console.log(obj.regular(), obj.arrow(), obj.nested());
```
`42 undefined 42`. The arrow property captured the module's top-level `this` (`undefined`); the arrow inside `nested` captured `nested`'s `this`, which is `obj`.

**Q36. Explain `call`, `apply` and `bind`.**
`call(thisArg, a, b)` and `apply(thisArg, [a, b])` invoke immediately; `bind(thisArg, a)` returns a new function with `this` and leading arguments fixed.

**Q37. What's the output?**
```js
function show() { return this.name; }
const a = show.bind({ name: 'A' });
const b = a.bind({ name: 'B' });
console.log(b(), a.call({ name: 'C' }));
```
`A A`. A bound function's `this` cannot be changed by another `bind` or by `call`; only `new` overrides it.

**Q38. What does `new` do?**
Creates an empty object, links its prototype to `Constructor.prototype`, runs the constructor with `this` set to the object, and returns it unless the constructor returns another object.

**Q39. What's the output?**
```js
function Car() {
  this.brand = 'Honda';
  return { brand: 'Toyota' };
}
function Bike() {
  this.brand = 'Yamaha';
  return 'Suzuki';
}
console.log(new Car().brand, new Bike().brand);
```
`Toyota Yamaha`. Returning an object replaces the new instance; returning a primitive is ignored.

**Q40. What is prototypal inheritance?**
Objects link to other objects through a hidden `[[Prototype]]`; property lookups walk this chain up to `Object.prototype` and then `null`, so objects share methods without copying them.

**Q41. What is the difference between `__proto__` and `prototype`?**
`obj.__proto__` (properly `Object.getPrototypeOf`) is the object `obj` inherits from; `Fn.prototype` is the object that instances created by `new Fn()` will inherit from.

**Q42. What's the output?**
```js
function Team() {}
Team.prototype.members = [];
const t1 = new Team();
const t2 = new Team();
t1.members.push('Ali');
console.log(t2.members.length);
```
`1`. The array lives on the prototype and is shared by every instance; initialize per-instance state in the constructor.

**Q43. What's the output?**
```js
const base = { greeting: 'hi' };
const child = Object.create(base);
child.greeting = 'hello';
delete child.greeting;
console.log(child.greeting);
```
`hi`. Assignment created an own property that shadowed the prototype's; deleting it reveals the inherited value again.

**Q44. Are classes just syntactic sugar?**
Mostly (methods live on the prototype), but classes must be called with `new`, have strict bodies, are in the TDZ before their declaration, have non-enumerable methods, and support fields, `#private` members and `super`.

**Q45. What's the output?**
```js
const p = new Person();
class Person {}
```
`ReferenceError: Cannot access 'Person' before initialization`. Class declarations are hoisted into the TDZ, unlike function declarations.

**Q46. How do private class fields differ from TypeScript `private`?**
`#field` is enforced by the engine at runtime and invisible outside the class body; TypeScript `private` is erased at compile time, so the property is still readable in JavaScript.

**Q47. How does `instanceof` work, and when does it fail?**
It checks whether `Constructor.prototype` appears in the object's prototype chain. It fails across iframes or realms and with duplicate library copies; use `Array.isArray` or duck typing.

**Q48. What's the output (strict mode)?**
```js
'use strict';
const config = Object.freeze({ db: { host: 'localhost' } });
config.db.host = 'prod';
console.log(config.db.host);
config.port = 5432;
```
`prod`, then `TypeError: Cannot add property port, object is not extensible`. `freeze` is shallow; adding to a frozen object throws in strict mode.

**Q49. Why must a subclass call `super()` before using `this`?**
The parent constructor creates and initializes the object in a derived class; before `super()` returns there is no `this`, so accessing it throws a ReferenceError.

**Q50. How do you reuse behavior without multiple inheritance?**
Composition (pass helpers in, use functions or hooks) or mixins: `Object.assign(Class.prototype, mixin)` or subclass factories `(Base) => class extends Base {}`.

## Async, event loop and promises (see [04-async-event-loop-promises.md](04-async-event-loop-promises.md))

**Q51. Explain the event loop in two sentences.**
JavaScript runs one task at a time on a single call stack while the host handles waiting. When the stack empties, the loop drains all microtasks, lets the browser render if needed, then runs the next task.

**Q52. Microtasks vs macrotasks?**
Microtasks: promise reactions, `await` continuations, `queueMicrotask`. Macrotasks: `setTimeout`, `setInterval`, I/O, UI events. All microtasks run after each task, before the next one.

**Q53. What's the output?**
```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
```
`1 4 3 2`. Sync code first, then the microtask, then the timer task.

**Q54. What's the output?**
```js
const p = new Promise((resolve) => {
  console.log('A');
  resolve('B');
  console.log('C');
});
p.then(console.log);
console.log('D');
```
`A C D B`. The executor runs synchronously and `resolve` does not stop it; the `then` callback is a microtask.

**Q55. What's the output?**
```js
async function getValue() {
  return 42;
}
const result = getValue();
console.log(result instanceof Promise);
result.then((v) => console.log(v));
console.log('end');
```
`true end 42`. An async function always returns a promise.

**Q56. What's the output?**
```js
async function first() {
  console.log(1);
  await second();
  console.log(2);
}
async function second() {
  console.log(3);
}
first();
Promise.resolve().then(() => console.log(4));
console.log(5);
```
`1 3 5 2 4`. Code before the first `await` runs synchronously (including `second()`); the continuation of `first` was queued before the `then` callback.

**Q57. What's the output?**
```js
setTimeout(() => console.log('timeout 1'), 0);
setTimeout(() => {
  console.log('timeout 2');
  Promise.resolve().then(() => console.log('promise in timeout 2'));
}, 0);
setTimeout(() => console.log('timeout 3'), 0);
```
`timeout 1`, `timeout 2`, `promise in timeout 2`, `timeout 3`. The microtask queue drains after each task.

**Q58. What's the output?**
```js
Promise.reject(new Error('fail'))
  .catch((e) => e.message)
  .then((msg) => console.log('then:', msg))
  .catch(() => console.log('catch 2'));
```
`then: fail`. The first `catch` recovers by returning a value, so the chain continues fulfilled and the second `catch` never runs.

**Q59. What's the output?**
```js
Promise.resolve(1)
  .then((x) => { throw new Error(`oops ${x}`); })
  .then(() => console.log('skipped'))
  .catch((e) => console.log(e.message))
  .finally(() => console.log('done'));
```
`oops 1`, then `done`. A throw rejects the next promise, skipping fulfillment handlers until `catch`.

**Q60. What's the output?**
```js
try {
  setTimeout(() => { throw new Error('late'); }, 0);
} catch (e) {
  console.log('caught', e.message);
}
console.log('after');
```
`after`, then an uncaught `Error: late` (it crashes a Node script). The callback runs later, after the `try` block has finished, so the `catch` never sees it.

**Q61. What's the output?**
```js
const results = [];
[1, 2, 3].forEach(async (id) => {
  await null;
  results.push(id);
});
console.log(results.length);
```
`0`. `forEach` ignores the promises returned by async callbacks; the pushes happen later as microtasks.

**Q62. What's the output?**
```js
Promise.all([Promise.resolve(1), Promise.reject(new Error('x')), Promise.resolve(3)])
  .then((values) => console.log('values', values))
  .catch((err) => console.log('error', err.message));
```
`error x`. `Promise.all` rejects as soon as any input rejects.

**Q63. What's the output (about how many ms)?**
```js
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
async function run() {
  const start = Date.now();
  await wait(100);
  await wait(100);
  await Promise.all([wait(100), wait(100)]);
  console.log(Math.round((Date.now() - start) / 100) * 100);
}
run();
```
`300`. Two sequential waits (200 ms) plus two parallel ones (100 ms).

**Q64. What's the output?**
```js
Promise.resolve()
  .then(() => {
    console.log(0);
    return Promise.resolve(4);
  })
  .then((x) => console.log(x));
Promise.resolve()
  .then(() => console.log(1))
  .then(() => console.log(2))
  .then(() => console.log(3))
  .then(() => console.log(5))
  .then(() => console.log(6));
```
`0 1 2 3 4 5 6`. Returning a promise from a `then` handler costs two extra microtask ticks (one job to call its `then`, one to resolve), so `4` lands after `3`. A senior-level detail.

**Q65. What do `f()` and `g()` resolve to?**
```js
async function f() {
  try {
    return Promise.reject(new Error('no await'));
  } catch {
    return 'caught';
  }
}
async function g() {
  try {
    return await Promise.reject(new Error('awaited'));
  } catch {
    return 'caught';
  }
}
```
`f()` rejects with `no await`; `g()` resolves to `'caught'`. Only `return await` lets the local `try/catch` see the rejection.

**Q66. What's the output in Node (a CommonJS script)?**
```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
```
`nextTick`, `promise`, then `timeout` and `immediate` (their relative order is not guaranteed in the main module). In an ES module the promise callback can run before `nextTick`, because module code itself runs inside a promise job.

**Q67. Compare `Promise.all`, `allSettled`, `race` and `any`.**
`all`: every value or the first rejection. `allSettled`: every outcome, never rejects. `race`: the first to settle. `any`: the first fulfillment, or an `AggregateError` if all reject.

**Q68. Is `await` blocking?**
It suspends only the current async function and resumes it in a microtask; the thread keeps running other code.

**Q69. How do you run independent requests in parallel?**
Start them together: `const [a, b] = await Promise.all([getA(), getB()])`. Sequential `await`s add their durations together.

**Q70. What happens with an unhandled rejection?**
Browsers log it and fire `unhandledrejection`; Node crashes by default since version 15. Always `await` or attach `catch`.

**Q71. How do you cancel a fetch?**
Pass `signal` from an `AbortController` and call `abort()`; the promise rejects with an `AbortError`. `AbortSignal.timeout(ms)` gives a timeout signal.

**Q72. How can you block the event loop, and how do you fix it?**
Long synchronous work (huge loops or `JSON.parse`, sync file and crypto APIs). Chunk the work, move it to a Web Worker or worker thread, or push it to a background job.

## ES6+ features (see [05-es6-plus.md](05-es6-plus.md))

**Q73. Name four differences between arrow and regular functions.**
No own `this`, no `arguments`, cannot be used with `new` (no `prototype`), cannot be generators. The expression form returns implicitly.

**Q74. What's the output?**
```js
const make1 = () => { name: 'A' };
const make2 = () => ({ name: 'A' });
console.log(make1(), make2());
```
`undefined { name: 'A' }`. In `make1` the braces are a function body containing a label; wrap object literals in parentheses.

**Q75. What's the output?**
```js
const { a = 1, b = 2, c = 3 } = { a: undefined, b: null };
console.log(a, b, c);
```
`1 null 3`. Destructuring defaults apply only to `undefined`.

**Q76. Spread vs rest?**
Spread expands an iterable or object (`[...arr]`, `{ ...obj }`, `fn(...args)`); rest collects the remaining items (`(...args)`, `const { a, ...rest } = obj`).

**Q77. ES modules vs CommonJS?**
ESM: static `import`/`export`, strict, live read-only bindings, tree-shakable, top-level await. CommonJS: `require`/`module.exports`, synchronous, dynamic, value snapshot.

**Q78. What are live bindings?**
An imported ESM binding always reflects the exporter's current value and cannot be assigned by the importer; CommonJS copies values at export time.

**Q79. What's the output?**
```js
const obj = {};
const keyA = { id: 1 };
const keyB = { id: 2 };
obj[keyA] = 'A';
obj[keyB] = 'B';
console.log(obj[keyA], Object.keys(obj));
const map = new Map([[keyA, 'A'], [keyB, 'B']]);
console.log(map.get(keyA));
```
`B [ '[object Object]' ]`, then `A`. Object keys are converted to strings, so both keys became `'[object Object]'`; a `Map` keys by identity.

**Q80. What's the output?**
```js
console.log([...new Set([1, '1', 1, NaN, NaN])]);
```
`[ 1, '1', NaN ]`. Sets use SameValueZero: `1` and `'1'` differ, and `NaN` equals `NaN`.

**Q81. What is a WeakMap good for?**
Attaching data to objects without keeping them alive: per-object metadata, identity-based caches, cycle detection. Keys must be objects; it is not iterable and has no `size`.

**Q82. What is a Symbol used for?**
Unique, collision-free property keys (skipped by `Object.keys` and `JSON.stringify`) and well-known hooks such as `Symbol.iterator` and `Symbol.toPrimitive`.

**Q83. What's the output?**
```js
function* gen() {
  const x = yield 1;
  console.log('received', x);
  yield x * 2;
}
const it = gen();
console.log(it.next().value);
console.log(it.next(5).value);
console.log(it.next().done);
```
`1`, `received 5`, `10`, `true`. The value passed to `next(5)` becomes the result of the paused `yield`.

**Q84. Which array methods mutate?**
`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin`. Copying alternatives: `toSorted`, `toReversed`, `toSpliced`, `with` (ES2023), plus `map`, `filter`, `slice`, spread.

**Q85. What's the output?**
```js
console.log([10, 1, 3].sort());
```
`[ 1, 10, 3 ]`. The default comparator compares strings; use `(a, b) => a - b` for numbers.

**Q86. What's the output?**
```js
const user = { profile: null, getName: undefined };
console.log(user.profile?.name, user.getName?.(), user.orders?.[0]?.id);
```
`undefined undefined undefined`. Optional chaining short-circuits on `null`/`undefined` for property access, calls and indexes.

**Q87. Are default parameters evaluated once or on every call?**
On every call, so `function add(item, list = [])` gets a fresh array each time (unlike Python's mutable default trap).

**Q88. What is tree shaking, and why does it need ES modules?**
Bundlers drop unused exports; static `import`/`export` lets them know at build time what is used, while dynamic `require` cannot be analyzed reliably.

## Browser, DOM and events (see [06-browser-dom-events.md](06-browser-dom-events.md))

**Q89. Explain bubbling and capturing.**
Events travel from `window` down to the target (capturing), fire on the target, then travel back up (bubbling). Listeners run in the bubbling phase unless registered with `{ capture: true }`.

**Q90. What's the output when the user clicks the button? (browser)**
```js
// <div id="outer"><button id="inner">Click</button></div>
const outer = document.querySelector('#outer');
const inner = document.querySelector('#inner');
outer.addEventListener('click', () => console.log('outer bubble'));
outer.addEventListener('click', () => console.log('outer capture'), { capture: true });
inner.addEventListener('click', () => console.log('inner'));
```
`outer capture`, `inner`, `outer bubble`. Capturing listeners on ancestors run first, then the target, then bubbling.

**Q91. What is event delegation?**
One listener on a parent that uses `event.target.closest(selector)` to handle events from many children, including children added later.

**Q92. `preventDefault` vs `stopPropagation`?**
`preventDefault` cancels the browser's default action (submit, navigation); `stopPropagation` stops the event from reaching other elements. They are independent.

**Q93. `event.target` vs `event.currentTarget`?**
`target` is where the event started; `currentTarget` is the element whose listener is running.

**Q94. Debounce vs throttle?**
Debounce runs once after events stop for N ms (search input); throttle runs at most once every N ms during continuous events (scroll).

**Q95. Where should a web app store auth tokens?**
Prefer an HttpOnly, Secure, SameSite cookie (with CSRF protection) or an in-memory access token with an HttpOnly refresh cookie; localStorage is readable by any XSS payload.

**Q96. Does `fetch` reject on a 404?**
No. It rejects only on network errors, CORS blocks and aborts; check `response.ok` and throw yourself.

**Q97. What triggers a CORS preflight?**
Methods other than GET/HEAD/POST, non-simple content types such as `application/json`, and custom headers such as `Authorization`. The browser sends `OPTIONS` first.

**Q98. `async` vs `defer` scripts?**
Both download in parallel; `async` runs as soon as it arrives (any order), `defer` runs after parsing in document order. Module scripts are deferred by default.

**Q99. Reflow vs repaint?**
Reflow recomputes layout after geometry changes; repaint redraws pixels after visual changes. Batch DOM reads before writes and animate `transform`/`opacity`.

**Q100. Why is `innerHTML` with user input dangerous?**
It parses HTML, so injected markup like `<img onerror=...>` runs script (XSS). Use `textContent` or sanitize with a library such as DOMPurify.

**Q101. What are Web Workers for?**
Running CPU-heavy JavaScript on a background thread without freezing the UI; they communicate with `postMessage` and cannot touch the DOM.

## Implementations and coding (see [07-implement-from-scratch.md](07-implement-from-scratch.md))

**Q102. Describe a debounce implementation in one breath.**
Return a function that clears a timer stored in a closure and starts a new `setTimeout`, calling `fn.apply(this, args)` when the timer fires.

**Q103. What are the key points of a `Promise.all` implementation?**
Wrap each input with `Promise.resolve`, store results by index, count the remaining ones, reject on the first error, and resolve `[]` immediately for empty input.

**Q104. How do you handle cycles in a deep clone?**
Keep a `WeakMap` from original to copy, register each copy before recursing, and return the stored copy when an original appears again.

**Q105. How does a `bind` polyfill handle `new`?**
Inside the returned function, check `new.target`; if set, construct the target with `new targetFn(...boundArgs, ...args)` and ignore the bound `this`.

**Q106. How do you implement an O(1) LRU cache in JavaScript?**
Use a `Map`: on `get`, delete and re-insert the key; on `put`, insert and evict `map.keys().next().value` when over capacity.

**Q107. How do you run 100 async tasks with at most 5 at a time?**
Pass task functions (not promises) to N workers that each loop, claiming the next index from a shared counter and awaiting it; collect results by index and `Promise.all` the workers.

**Q108. Implement `once` in words.**
Return a wrapper that keeps `called` and `result` in a closure: the first call runs the function and stores the result; later calls return the stored result.

**Q109. How do you flatten an arbitrarily nested array without recursion?**
Use a stack: pop an item; if it is an array, push its elements; otherwise collect it. Reverse the collected items at the end (or use `arr.flat(Infinity)` in real code).

**Q110. How does `curry` know when to call the original function?**
It compares the number of collected arguments with `fn.length`; once enough are collected it calls `fn`, otherwise it returns a function that collects more.

## Errors, tooling and miscellaneous

**Q111. What's the output?**
```js
function test() {
  try {
    return 'try';
  } finally {
    console.log('finally');
  }
}
console.log(test());
```
`finally`, then `try`. `finally` runs before the function actually returns (and a `return` inside `finally` would override `'try'`).

**Q112. What's the output?**
```js
console.log(JSON.stringify({ a: undefined, b: () => 1, c: NaN, d: new Date(0), e: [undefined] }));
```
`{"c":null,"d":"1970-01-01T00:00:00.000Z","e":[null]}`. `undefined` and functions are dropped from objects but become `null` in arrays; NaN becomes `null`; Dates call `toJSON`.

**Q113. What's the output?**
```js
const d = new Date(2025, 0, 31);
d.setMonth(1);
console.log(d.getMonth(), d.getDate());
```
`2 3`. Months are zero-based, and "February 31" overflows to March 3. Date arithmetic is full of traps; use a date library or `Intl` for formatting, and store timestamps in UTC.

**Q114. Name the common built-in error types.**
`TypeError` (wrong type, calling a non-function, reading a property of undefined), `ReferenceError` (unknown name, TDZ), `SyntaxError` (invalid code, bad `JSON.parse` input), `RangeError` (invalid length, stack overflow). Subclass `Error` for your own and set `name`.

**Q115. What does strict mode change?**
Assigning to undeclared variables throws, `this` is `undefined` in plain calls, writes to read-only properties throw, duplicate parameter names and `with` are forbidden. ES modules and classes are always strict.

**Q116. What is prototype pollution?**
An attack where user input like `{"__proto__": {"isAdmin": true}}` is merged into an object and modifies `Object.prototype`, affecting every object. Validate input, block `__proto__`/`constructor` keys in merges, and use `Object.create(null)` or `Map` for dictionaries.

**Q117. What do Babel and a bundler do, and what is a polyfill?**
Babel (or SWC, esbuild) transpiles newer syntax into older syntax; bundlers (Vite, webpack, Metro in React Native) combine modules into optimized bundles. A polyfill adds missing runtime APIs (such as `Promise.allSettled`), which syntax transforms cannot do.

**Q118. What do `^1.2.3` and `~1.2.3` mean in package.json, and why commit the lockfile?**
`^` allows minor and patch updates (`<2.0.0`), `~` allows patch updates only (`<1.3.0`). The lockfile pins exact versions so every machine and CI installs the same dependency tree.

**Q119. `slice` vs `splice`?**
`slice(start, end)` returns a copy of a portion without changing the array; `splice(start, deleteCount, ...items)` removes or inserts in place and returns the removed items.

**Q120. Why avoid `eval` and `new Function` with user input?**
They execute arbitrary code with your page's privileges (code injection, XSS), defeat engine optimizations and are blocked by strict Content Security Policies.
