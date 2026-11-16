# JavaScript Basics: Types, Coercion and Equality

> **Tier:** T1 (know cold) · **Related:** [Scope, closures and hoisting](02-scope-closures-hoisting.md), [ES6+ features](05-es6-plus.md), [TypeScript fundamentals](../typescript/01-fundamentals.md), [JavaScript rapid-fire questions](99-interview-questions.md)
> **On your resume:** JavaScript and TypeScript behind your React, React Native (Expo) and React on Rails work at Devntech, the Next.js APIs at RemoteDev, and the Web Development labs you assisted at PUCIT.

## Quick revision
- ECMAScript is the specification (TC39 ships one edition per year; ES2015 = ES6). JavaScript implements it. Engines: V8 (Chrome, Node, Deno), SpiderMonkey (Firefox), JavaScriptCore (Safari, Bun), Hermes (React Native).
- The engine runs the language; the host adds APIs: DOM, `fetch`, `localStorage` in browsers; `fs`, `process`, `Buffer` in Node. `globalThis` works everywhere.
- Default to `const`, use `let` when you reassign, avoid `var` (function-scoped, hoisted as `undefined`, becomes a `window` property at script top level).
- 7 primitives: string, number, bigint, boolean, undefined, null, symbol. Everything else is an object (arrays, functions, dates, maps).
- `typeof null === 'object'` (historic bug), `typeof function () {} === 'function'`, `typeof [] === 'object'` (use `Array.isArray`), `typeof NaN === 'number'`.
- Numbers are IEEE 754 doubles: `0.1 + 0.2 === 0.30000000000000004`. Keep money in integer minor units (cents, paisa) or decimal strings.
- Integers are exact up to `Number.MAX_SAFE_INTEGER` (2^53 - 1). Beyond that use `BigInt` (`123n`), which cannot be mixed with `number` in arithmetic.
- `NaN !== NaN`. Use `Number.isNaN(x)` (no coercion), not global `isNaN(x)` (coerces first).
- Falsy values (8): `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is truthy, including `'0'`, `'false'`, `[]`, `{}`.
- `+` concatenates if either side is a string after ToPrimitive; `-`, `*`, `/` always convert to numbers.
- `===` compares without coercion, `==` coerces, `Object.is` is `===` except `Object.is(NaN, NaN)` is true and `Object.is(0, -0)` is false.
- `null == undefined` is true and neither loosely equals anything else, so `x == null` checks both at once.
- Primitives are copied by value; objects are shared by reference. Arguments are always passed by value, but an object's value is a reference ("call by sharing").
- Spread and `Object.assign` are shallow. `structuredClone` deep-copies (cycles, Map, Set, Date) but throws on functions and drops prototypes.
- `a ?? b` falls back only on `null`/`undefined`; `a || b` falls back on any falsy value (bug with `0` and `''`).
- `?.` short-circuits to `undefined`. Logical assignment: `a ||= b`, `a &&= b`, `a ??= b`.

## 1. ECMAScript, engines and hosts

Interviewers ask "what is V8?" or "how is Node different from the browser?" to check that you know where the language ends and the platform begins.

- **ECMAScript** is the standard written by the TC39 committee. Proposals move through stages 0 to 4; stage 4 features ship in the next yearly edition.
- **Engines** implement the standard. V8 parses source into an AST, compiles it to bytecode for its interpreter (Ignition), and hot functions get compiled to machine code by its optimizing compiler (TurboFan). If an optimized function suddenly sees a different object shape, it is deoptimized, which is one reason consistent object shapes are faster.
- **Hosts** embed an engine and add APIs plus an event loop:

| | Browser | Node.js |
|---|---|---|
| Global object | `window` (also `globalThis`) | `global` (also `globalThis`) |
| Extra APIs | DOM, `fetch`, `localStorage`, `history`, Web Workers | `fs`, `path`, `process`, `Buffer`, `http`, `child_process`; global `fetch` from Node 18 |
| Modules | ES modules (`<script type="module">`) | CommonJS (`require`) and ES modules |
| Event loop | HTML spec: tasks, microtasks, rendering | libuv phases (timers, poll, check) plus `process.nextTick` |
| Security | Sandboxed, same-origin policy | Full OS access |

React Native runs your JavaScript on Hermes with no DOM: components map to native views, so `document` does not exist, but `fetch`, timers and promises do.

## 2. var, let and const

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Hoisted, initialized to `undefined` | Hoisted, uninitialized (TDZ) | Hoisted, uninitialized (TDZ) |
| Redeclare in same scope | Allowed | SyntaxError | SyntaxError |
| Reassign | Yes | Yes | No (TypeError) |
| Top-level script adds a `window` property | Yes | No | No |

```js
const user = { name: 'Sara' };
user.name = 'Ali';   // fine: const blocks reassignment, not mutation
// user = {};        // TypeError: Assignment to constant variable.

if (true) {
  var a = 1;   // function (or global) scoped: leaks out of the block
  let b = 2;   // block scoped
}
console.log(a); // 1
// console.log(b); // ReferenceError: b is not defined
```

To make the object itself immutable, use `Object.freeze` (shallow, see [this, prototypes and classes](03-this-prototypes-classes.md)). Hoisting and the TDZ are covered in [scope, closures and hoisting](02-scope-closures-hoisting.md).

## 3. Primitives, objects and typeof

Primitives are immutable values without methods of their own. When you write `'abc'.toUpperCase()`, the engine wraps the string in a temporary `String` object (auto-boxing), calls the method and throws the wrapper away. Never create wrappers yourself: `new Boolean(false)` is an object, so it is truthy.

| Expression | Result | Note |
|---|---|---|
| `typeof 'hi'` | `'string'` | |
| `typeof 42`, `typeof NaN` | `'number'` | NaN is a number value |
| `typeof 42n` | `'bigint'` | |
| `typeof undefined` | `'undefined'` | |
| `typeof null` | `'object'` | bug from the first implementation, kept for compatibility |
| `typeof Symbol()` | `'symbol'` | |
| `typeof {}`, `typeof []`, `typeof new Date()` | `'object'` | use `Array.isArray`, `instanceof Date` |
| `typeof function () {}`, `typeof class {}` | `'function'` | functions are callable objects |
| `typeof notDeclared` | `'undefined'` | no ReferenceError (unless it is a `let`/`const` in its TDZ) |

For a precise check, `Object.prototype.toString.call(value)` returns `'[object Array]'`, `'[object Null]'`, `'[object Date]'` and so on.

### null vs undefined

| | `undefined` | `null` |
|---|---|---|
| Meaning | "no value assigned yet" (set by the engine) | "intentionally empty" (set by you) |
| Appears in | uninitialized variables, missing arguments and properties, functions without `return` | APIs that return "nothing found" (`querySelector`, many DB drivers) |
| `typeof` | `'undefined'` | `'object'` |
| Arithmetic | `undefined + 1` is `NaN` | `null + 1` is `1` |
| Default parameters | trigger the default | do not trigger the default |
| `JSON.stringify` | key is dropped | kept as `null` |

`null == undefined` is true, `null === undefined` is false. Rails renders `nil` as `null`, while a key the API never sent is `undefined` on the JavaScript side, so your types should say which one the API really returns.

## 4. Numbers: IEEE 754, NaN and BigInt

Every `number` is a 64-bit IEEE 754 double (1 sign bit, 11 exponent bits, 52 fraction bits). Fractions like 0.1 have no exact binary representation, so tiny rounding errors appear.

```js
0.1 + 0.2;           // 0.30000000000000004
0.1 + 0.2 === 0.3;   // false

// Compare floats with a tolerance. EPSILON is the gap above 1, so scale it for larger values.
const almostEqual = (a, b) =>
  Math.abs(a - b) < Number.EPSILON * Math.max(1, Math.abs(a), Math.abs(b));
almostEqual(0.1 + 0.2, 0.3); // true

// Money: integer minor units, format only for display
const totalCents = 1999 + 2501;   // 4500
(totalCents / 100).toFixed(2);    // '45.00' (toFixed returns a string)
```

Integers are exact only up to 2^53 - 1:

```js
Number.MAX_SAFE_INTEGER;        // 9007199254740991
2 ** 53 === 2 ** 53 + 1;        // true: precision lost
Number.isSafeInteger(2 ** 53);  // false

const big = 9007199254740993n;  // BigInt literal
big + 1n;                       // 9007199254740994n
7n / 2n;                        // 3n (integer division truncates)
// big + 1;                     // TypeError: Cannot mix BigInt and other types
// JSON.stringify({ id: big }); // TypeError: JSON cannot serialize a BigInt
```

Use BigInt for values that can pass 2^53: 64-bit database IDs, counters, some external IDs. This is also why node-postgres returns PostgreSQL `bigint` (int8) columns as strings by default, and why many APIs send large IDs as strings.

**NaN** comes from failed conversions or invalid math (`0 / 0`, `Number('abc')`). It is the only value not equal to itself.

```js
NaN === NaN;          // false
Number.isNaN(NaN);    // true
Number.isNaN('abc');  // false: no coercion, 'abc' is a string, not NaN
isNaN('abc');         // true: global isNaN converts to a number first
isNaN('');            // false: '' converts to 0 (a trap for form validation)
Object.is(NaN, NaN);  // true
[NaN].includes(NaN);  // true (SameValueZero)
[NaN].indexOf(NaN);   // -1   (strict equality)
```

More number facts that come up:
- `-0` exists: `0 === -0` is true, `Object.is(0, -0)` is false, `1 / -0` is `-Infinity`.
- `Number('')` is `0`, `Number('12px')` is `NaN`, but `parseInt('12px', 10)` is `12`. Always pass the radix to `parseInt`.
- `(1.005).toFixed(2)` is `'1.00'`, because 1.005 is stored as 1.00499999... Use integer math, and `Intl.NumberFormat` for display.
- Numeric separators help readability: `1_000_000` (ES2021).

## 5. Strings and template literals

Strings are immutable sequences of UTF-16 code units; `toUpperCase`, `slice` and `replace` return new strings.

```js
const name = 'Sara';
const greeting = `Hi ${name}, you have ${3 + 2} new messages`; // interpolation
const twoLines = `Line 1
Line 2`;                         // real newline inside the string

'hello'.at(-1);                  // 'o' (ES2022)
'a-b-c'.replaceAll('-', '+');    // 'a+b+c' (ES2021)
'5'.padStart(3, '0');            // '005'
'abc'.slice(-2);                 // 'bc'

// Tagged template: the function receives the literal parts and the values separately
function highlight(strings, ...values) {
  // strings: ['Total: ', ' items'], values: [5]
  return strings.reduce(
    (out, str, i) => out + str + (i < values.length ? `[${values[i]}]` : ''),
    '',
  );
}
highlight`Total: ${5} items`;   // 'Total: [5] items'
```

- `.length` counts UTF-16 code units, so characters outside the Basic Multilingual Plane (most emoji) count as 2. `[...str].length` counts code points.
- Tagged templates are how libraries build safe SQL: Drizzle's `sql` tag turns interpolated values into bound parameters instead of concatenating them into the query string.

## 6. Type coercion

Coercion converts a value to another type. **Explicit**: `Number(x)`, `String(x)`, `Boolean(x)`. **Implicit**: operators do it for you (`'5' * 2`). The spec defines abstract operations; you need four of them.

### ToPrimitive (object to primitive)
1. If the object has a `[Symbol.toPrimitive]` method, call it with a hint: `'number'`, `'string'` or `'default'`.
2. Otherwise, for hint `'string'` try `toString()` then `valueOf()`; for `'number'` or `'default'` try `valueOf()` then `toString()`. The first primitive result wins.

Plain objects and arrays return themselves from `valueOf`, so `toString` decides: `[]` becomes `''`, `[1, 2]` becomes `'1,2'`, `{}` becomes `'[object Object]'`. `Date` treats `'default'` as `'string'`, which is why `new Date() + 1` concatenates.

```js
const price = {
  amount: 50,
  [Symbol.toPrimitive](hint) {
    return hint === 'string' ? `Rs ${this.amount}` : this.amount;
  },
};
price * 2;    // 100     (hint 'number')
`${price}`;   // 'Rs 50' (hint 'string')
price + 1;    // 51      (hint 'default')
```

### ToNumber, ToString and ToBoolean

| Value | ToNumber | ToString | ToBoolean |
|---|---|---|---|
| `undefined` | `NaN` | `'undefined'` | false |
| `null` | `0` | `'null'` | false |
| `true` / `false` | `1` / `0` | `'true'` / `'false'` | itself |
| `''` | `0` | `''` | false |
| `' 42 '` | `42` (whitespace trimmed) | itself | true |
| `'42px'` | `NaN` | itself | true |
| `'0'` | `0` | itself | **true** |
| `[]` | `0` (via `''`) | `''` | **true** |
| `[7]` / `[1, 2]` | `7` / `NaN` | `'7'` / `'1,2'` | true |
| `{}` | `NaN` | `'[object Object]'` | true |
| `Symbol('x')` | TypeError | TypeError when implicit (`String(sym)` works) | true |

**Falsy list:** `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN` (plus the legacy browser oddity `document.all`). Everything else is truthy.

### Operator rules
- **`+`**: ToPrimitive both sides. If either result is a string, concatenate; otherwise convert to numbers and add.
- **`-`, `*`, `/`, `%`, `**`, unary `+`**: always ToNumber.
- **`<`, `>`, `<=`, `>=`**: ToPrimitive; two strings compare lexicographically (`'10' < '9'` is true), anything else compares as numbers.
- **`!`, `if`, `&&`, `||`, ternary**: ToBoolean. `&&` and `||` return one of the operands, not a boolean: `'' || 'default'` is `'default'`.

### Classic puzzles

| Expression | Result | Why |
|---|---|---|
| `[] + []` | `''` | both become `''` |
| `[] + {}` | `'[object Object]'` | `'' + '[object Object]'` |
| `{} + []` at the start of a console line | `0` | `{}` parses as an empty block, leaving `+[]` |
| `'5' - 2` | `3` | `-` is numeric only |
| `'5' + 2` | `'52'` | one side is a string |
| `'5' * '2'` | `10` | numeric |
| `true + 1` | `2` | `true` becomes 1 |
| `null + 1` / `undefined + 1` | `1` / `NaN` | `null` is 0, `undefined` is NaN |
| `'b' + 'a' + +'a' + 'a'` | `'baNaNa'` | `+'a'` is `NaN` |
| `[1, 2] + [3]` | `'1,23'` | `'1,2' + '3'` |
| `null == 0` | `false` | `null` loosely equals only `undefined` |
| `null >= 0` | `true` | relational operators convert `null` to 0 |
| `NaN === NaN` | `false` | NaN equals nothing |
| `[] == ![]` | `true` | `![]` is false; `[] == false` becomes `'' == 0`, then `0 == 0` |
| `'' == 0`, `'0' == false` | `true`, `true` | both sides end up as numbers |
| `1 < 2 < 3` / `3 > 2 > 1` | `true` / `false` | `true < 3` is 1 < 3; `true > 1` is 1 > 1 |
| `typeof typeof 1` | `'string'` | `typeof` always returns a string |

## 7. == vs === vs Object.is

**`===` (strict equality):** same type and same value, no conversion. Objects are equal only when they are the same reference. Two exceptions: `NaN !== NaN` and `+0 === -0`.

**`==` (loose equality)** follows the IsLooselyEqual algorithm:
1. Same type: behave like `===`.
2. `null` and `undefined` equal each other and nothing else.
3. Number vs string: convert the string to a number.
4. Boolean vs anything: convert the boolean to a number first. So `'1' == true` is true, but `'true' == true` is false (`NaN == 1`).
5. Object vs primitive: ToPrimitive the object, then compare again.
6. BigInt vs number or string: compare the mathematical values (`1n == 1` is true).

**`Object.is`** is `===` with both exceptions fixed. React uses it to decide whether state changed, which is why mutating an array and calling `setItems(items)` does not re-render: same reference, so "nothing changed". **SameValueZero** (used by `includes`, `Map` and `Set` keys) is `Object.is` except `+0` and `-0` are equal.

Rule of thumb: always use `===`, except `value == null` as a deliberate shorthand for "null or undefined" (many style guides allow exactly that one case).

## 8. Value vs reference, shallow vs deep copy

```js
let a = 10;
let b = a;      // copies the value
b++;            // a is still 10

const p1 = { name: 'Sara', tags: ['admin'] };
const p2 = p1;  // copies the reference: same object
p2.name = 'Ali';
console.log(p1.name); // 'Ali'

function rename(person) {
  person.name = 'Zara';    // mutation: visible to the caller
  person = { name: 'X' };  // rebinding: only changes the local parameter
}
rename(p1);
console.log(p1.name); // 'Zara'
```

JavaScript always **passes by value**; for objects, the value being copied is a reference. This is called call by sharing: mutation is visible to the caller, reassignment is not.

A **shallow copy** duplicates only the top level, so nested objects stay shared:

```js
const original = { id: 1, address: { city: 'Lahore' } };
const shallow = { ...original };          // or Object.assign({}, original)
shallow.address.city = 'Karachi';
console.log(original.address.city);       // 'Karachi': nested object is shared

const deep = structuredClone(original);   // browsers and Node 17+
deep.address.city = 'Multan';
console.log(original.address.city);       // still 'Karachi'
```

| Method | Depth | Handles | Breaks on |
|---|---|---|---|
| `{ ...obj }`, `Object.assign`, `[...arr]`, `arr.slice()` | shallow | anything (copies references) | nested data stays shared |
| `JSON.parse(JSON.stringify(x))` | deep | plain JSON data | drops `undefined`, functions and symbols; Dates become strings; `NaN` becomes `null`; Map/Set become `{}`; throws on cycles and BigInt |
| `structuredClone(x)` | deep | Date, RegExp, Map, Set, ArrayBuffer, typed arrays, cycles | throws `DataCloneError` on functions and DOM nodes; class instances become plain objects; getters are invoked and stored as plain values |

A custom deep clone with cycle handling is in [implement from scratch](07-implement-from-scratch.md). In React state you rarely need deep copies: copy only the path you change (`{ ...state, user: { ...state.user, name } }`) so untouched branches keep their identity and memoized children skip re-rendering.

## 9. Modern operators: ??, ?. and logical assignment

```js
const settings = { retries: 0, title: '' };
settings.retries || 3;          // 3  (bug: 0 is falsy)
settings.retries ?? 3;          // 0  (only null/undefined fall back)
settings.title ?? 'Untitled';   // ''

const user = { profile: null };
user.profile?.city;             // undefined instead of a TypeError
user.orders?.[0];               // optional element access
user.onSave?.();                // call only if the function exists

// Logical assignment (ES2021): assigns only when needed
const options = { timeout: null, name: '  Sara ' };
options.timeout ??= 5000;       // assigned because it was null
options.retries ||= 3;          // assigned because it was undefined (falsy)
options.name &&= options.name.trim(); // updated only because it was truthy
```

- `??` cannot be mixed with `||` or `&&` without parentheses: `a || b ?? c` is a SyntaxError.
- Optional chaining cannot be an assignment target: `user?.name = 'x'` is a SyntaxError.
- Do not sprinkle `?.` everywhere. If a value must exist, a loud error points at the real bug faster than a silent `undefined` three components later.

## Common pitfalls
- Using `||` for defaults when `0`, `''` or `false` are valid (page 0, a zero discount, an empty search). Use `??`.
- Storing money as floats or comparing floats with `===`. Split bills and rent payments are exactly where `0.1 + 0.2` bugs appear.
- `typeof x === 'object'` without excluding `null` (and arrays).
- Parsing 64-bit IDs from an API as numbers and silently losing precision. Keep them as strings.
- `if (count)` when `count` can legitimately be `0`.
- Mutating nested data in a spread "copy" and accidentally changing the original (a classic React state bug).
- `JSON.parse(JSON.stringify(x))` turning Dates into strings and dropping `undefined` fields without any error.
- Global `isNaN` on form input: `isNaN('')` and `isNaN(' ')` are false because they convert to 0.
- Default `sort()` on numbers is lexicographic: `[10, 9, 1].sort()` gives `[1, 10, 9]`. Pass `(a, b) => a - b`.
- Writing `==` puzzles into real code. Use `===` and explicit conversion (`Number(input)`).

## How to talk about this in an interview
- Lead with the rule, then a tiny example: "`==` coerces using a defined algorithm, for example `'' == 0` is true because both become numbers, so I use `===` everywhere except the deliberate `== null` check."
- Tie precision to real work: money in the Coloc resident app (split bills, rent payments) and prices in the CrystalCommerce catalog are textbook places for floating point bugs. Describe what the code actually did, not what it ideally should do.
  > **Fill in:** how were amounts represented in the apps you worked on (integer cents, decimal strings from Rails, a money library)?
- For `null` vs `undefined`, mention the Rails boundary: `nil` arrives as `null`, a missing key arrives as `undefined`, and the TypeScript types should match what the API really sends.
  > **Fill in:** one bug you have seen (or caused) from coercion, a falsy check or a shallow copy, and how you found it.
- If you blank on a puzzle, reason out loud with the abstract operations (ToPrimitive, then ToNumber or ToString). Interviewers care more about the method than a memorized answer.

## Interview questions

**Q1. What are the data types in JavaScript?**
Seven primitives (string, number, bigint, boolean, undefined, null, symbol) plus objects. Primitives are immutable and compared by value; objects (arrays, functions, dates, maps) are mutable and compared by reference. `typeof null === 'object'` is a historical bug.
*Follow-up:* How do you check for an array? `Array.isArray(x)`; `typeof` says `'object'` and `instanceof Array` fails across iframes.

**Q2. What is the difference between `null` and `undefined`?**
`undefined` means a value was never assigned (uninitialized variables, missing properties and arguments, no `return`); `null` is an explicit "empty" that you assign. `typeof` gives `'undefined'` vs `'object'`, `null` becomes 0 in arithmetic while `undefined` becomes NaN, and only `undefined` triggers default parameters.
*Follow-up:* How do you check for both? `value == null`, or `value ?? fallback` for defaults.

**Q3. Explain `==` vs `===`. Where does `Object.is` fit?**
`===` compares type and value with no conversion. `==` converts first: null equals undefined, strings and booleans become numbers, objects go through ToPrimitive. `Object.is` is like `===` but treats `NaN` as equal to itself and distinguishes `0` from `-0`; React uses it to detect state changes.
*Follow-up:* Any valid use of `==`? `x == null` to catch both null and undefined.

**Q4. Why is `0.1 + 0.2 !== 0.3`, and how do you handle money?**
Numbers are IEEE 754 binary doubles, and 0.1 and 0.2 have no exact binary representation, so the sum is 0.30000000000000004. Compare with a tolerance based on `Number.EPSILON`, and keep money in integer minor units (or decimal strings from the backend), formatting only for display.
*Follow-up:* Why is `(1.005).toFixed(2)` `'1.00'`? 1.005 is stored as 1.00499999..., so it rounds down.

**Q5. How do you check whether a value is NaN?**
`Number.isNaN(x)` or `Object.is(x, NaN)`. `x === NaN` is always false. Global `isNaN` converts first, so `isNaN('abc')` is true and `isNaN('')` is false, which makes it unreliable for input validation.
*Follow-up:* Why does `[NaN].includes(NaN)` work but `indexOf` does not? `includes` uses SameValueZero, `indexOf` uses strict equality.

**Q6. Compare `var`, `let` and `const`.**
`var` is function-scoped, hoisted with the value `undefined`, can be redeclared and becomes a global object property at script top level. `let` and `const` are block-scoped, hoisted but in the TDZ until their declaration runs, and cannot be redeclared. `const` cannot be reassigned, but the object it points to can still be mutated.
*Follow-up:* Which do you default to? `const`, then `let` when reassignment is needed; never `var` in new code.

**Q7. List the falsy values. Are `[]` and `'0'` truthy?**
`false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is truthy, including `[]`, `{}`, `'0'` and `'false'`.
*Follow-up:* How do you check for an empty array? `arr.length === 0`, never `if (arr)`.

**Q8. What do `'5' + 2`, `'5' - 2`, `[] + []` and `[] + {}` return?**
`'52'` (one side is a string, so `+` concatenates), `3` (`-` is numeric only), `''` (both arrays become empty strings) and `'[object Object]'` (`'' + '[object Object]'`).
*Follow-up:* And `{} + []` typed in a console? `0`, because the leading `{}` is parsed as an empty block and `+[]` is 0.

**Q9. Why is `[] == ![]` true?**
`![]` is evaluated first: arrays are truthy, so it is `false`. Then `[] == false`: the boolean becomes 0, the array becomes `''` through ToPrimitive, `''` becomes 0, and `0 == 0` is true.
*Follow-up:* What is the lesson? Avoid `==` with mixed types; it hides several conversions.

**Q10. Shallow vs deep copy: how do you deep copy an object?**
A shallow copy (spread, `Object.assign`, `slice`) duplicates only the top level, so nested objects are shared. For a deep copy use `structuredClone` (handles Date, Map, Set and cycles, throws on functions, drops prototypes) or a custom or library clone. `JSON.parse(JSON.stringify())` works only for plain JSON data.
*Follow-up:* Do you need deep copies for React state? Usually not; copy only the changed path so untouched branches keep their identity.

**Q11. Is JavaScript pass-by-value or pass-by-reference?**
Always pass-by-value, but for objects the value is a reference (call by sharing). Mutating a parameter's properties affects the caller's object; reassigning the parameter does not.
*Follow-up:* How do you avoid accidental mutation in a function? Copy first (`{ ...obj }`) or freeze inputs in development.

**Q12. What is the difference between `??` and `||`?**
`a || b` returns `b` for any falsy `a` (`0`, `''`, `false`, `NaN`, `null`, `undefined`); `a ?? b` only for `null` or `undefined`. Use `??` when `0` or `''` are valid values, such as a page number, a quantity or a port.
*Follow-up:* Can you write `a || b ?? c`? No, mixing them without parentheses is a SyntaxError.

**Q13. What is BigInt and when would you use it?**
An arbitrary-precision integer type (`123n`, `BigInt('123')`) for values beyond 2^53 - 1, such as 64-bit IDs. It cannot be mixed with `number` in arithmetic, division truncates, `Math` methods do not accept it, and `JSON.stringify` throws on it, so APIs usually send big IDs as strings.
*Follow-up:* What is `Number.MAX_SAFE_INTEGER`? 9007199254740991 (2^53 - 1); above it, distinct integers can map to the same double.

**Q14. What does optional chaining return, and when should you avoid it?**
`a?.b` returns `undefined` instead of throwing when `a` is `null` or `undefined`, and short-circuits the rest of the chain; it also works for calls (`fn?.()`) and indexes (`arr?.[0]`). Avoid it on values that must exist, because it turns a clear crash into a silent `undefined` later.
*Follow-up:* What does `a?.b` give when `a` is `0`? `undefined`, but not because of `?.`: it only stops on null/undefined, and `(0).b` is simply `undefined` after auto-boxing.

**Q15. What is the difference between ECMAScript, JavaScript and V8? What does Node add?**
ECMAScript is the specification, JavaScript is the language that implements it, and V8 is Google's engine that runs it (Chrome, Node, Deno). Node embeds V8 and adds an event loop on libuv plus server APIs (`fs`, `http`, `process`, `Buffer`), but no DOM.
*Follow-up:* Which engine does React Native use? Hermes by default.

**Q16. Why does `typeof undeclaredVar` not throw, and when does `typeof` throw?**
`typeof` on an unresolvable name returns `'undefined'` by design, so feature detection like `typeof window !== 'undefined'` works in Node. It throws a ReferenceError for a `let`, `const` or `class` binding that exists but is still in its temporal dead zone.
*Follow-up:* What is a safer cross-platform global check? `typeof globalThis.window !== 'undefined'`.

**Q17. What does `['1', '2', '3'].map(parseInt)` return?**
`[1, NaN, NaN]`. `map` passes `(value, index)`, so the calls are `parseInt('1', 0)` (radix 0 means 10), `parseInt('2', 1)` (invalid radix) and `parseInt('3', 2)` ('3' is not a binary digit).
*Follow-up:* Fix it? `.map(Number)` or `.map((s) => parseInt(s, 10))`.

**Q18. How does JavaScript convert an object to a primitive?**
Through ToPrimitive: it calls `[Symbol.toPrimitive](hint)` if defined; otherwise it tries `valueOf` then `toString` (or the reverse for the `'string'` hint) and uses the first primitive result. Arrays and plain objects fall through to `toString`, giving `'1,2'` and `'[object Object]'`.
*Follow-up:* Why does `new Date() + 1` concatenate? Date treats the `'default'` hint as `'string'`.

## Further reading
- [MDN: JavaScript data types and data structures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures)
- [MDN: Equality comparisons and sameness](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness)
