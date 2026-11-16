# TypeScript Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Generics and utility types](02-generics-utility-types.md), [Advanced types](03-advanced-types.md), [TypeScript in practice](04-ts-in-practice.md), [JavaScript types and coercion](../javascript/01-basics-types-coercion.md), [React fundamentals](../../06-react/01-fundamentals.md)
> **On your resume:** TypeScript is part of your daily work at Devntech (React, React Native with Expo, and the oRPC + TanStack Start + Drizzle platform) and of the Next.js APIs you built at RemoteDev.

## Quick revision
- TypeScript = JavaScript + a static type system. The compiler checks types and then erases them: no types exist at runtime.
- Benefits: errors before running, autocomplete and safe refactors, types as documentation. It does not validate runtime data; use Zod (or similar) at boundaries.
- `tsc` parses, type-checks and emits JavaScript, and by default it emits even when there are type errors. Modern setups let a bundler strip types (esbuild, SWC, Babel) and run `tsc --noEmit` separately.
- Let inference handle locals; annotate parameters, exported APIs and values that start empty (`[]`, `null`).
- `let x = 'a'` is `string`; `const x = 'a'` is the literal type `'a'` (widening).
- Arrays: `string[]` or `Array<string>`. Tuples: `[string, number]`, with optional (`?`), rest (`...T[]`) and `readonly` variants.
- Object types support optional (`?`) and `readonly` properties. Typing is structural: the shape matters, not the name. Excess property checks only apply to fresh object literals.
- `interface` vs `type`: both describe objects; only `type` names unions, tuples, primitives and mapped or conditional types; only `interface` supports declaration merging.
- Union `A | B` = one of; intersection `A & B` = all of. Only members common to every union member are usable until you narrow.
- Prefer unions of string literals or `as const` objects over enums; enums emit runtime code and have quirks.
- `any` switches checking off; `unknown` is the safe top type (narrow before use); `never` has no values (impossible branches, exhaustive checks); `void` means "the return value is not used".
- `as` and `!` are promises you make to the compiler, not checks.
- Functions: `(a: number) => string` types, optional and default parameters, rest parameters, overloads for a few distinct call shapes.
- Narrowing: `typeof`, `instanceof`, `in`, equality, truthiness, discriminant properties, `Array.isArray`, type guards.
- `strict: true` turns on `strictNullChecks`, `noImplicitAny` and the rest of the strict family; use it for all new code.

## 1. Why TypeScript, and what it does not do

```ts
type User = { id: number; name: string; email?: string };

function greet(user: User) {
  return `Hi ${user.name.toUpperCase()}`;
}

greet({ id: 1, name: 'Sara' });
// greet({ id: 1 });               // error: Property 'name' is missing
// greet({ id: 1, nme: 'Sara' });  // error: Object literal may only specify known properties
// user.email.toLowerCase();       // error inside a function: 'user.email' is possibly 'undefined'
```

What you get:
- **Errors before runtime:** typos, wrong arguments, missing null checks, forgotten cases in a union.
- **Tooling:** autocomplete, go to definition, rename across the codebase, inline docs.
- **Safer refactors:** change a type and the compiler lists every place that must change. In a large React Native or full-stack TypeScript codebase this is the main win.
- **Gradual adoption:** most JavaScript is already valid TypeScript, so you can migrate file by file.

What you do not get:
- **No runtime checks.** Types are erased. `const user = (await res.json()) as User` compiles even if the API returns something else entirely.
- **No speedup.** The output is plain JavaScript.
- **A few features emit code:** enums, namespaces, parameter properties and decorators. Everything else disappears.

## 2. The compiler pipeline

`tsc` parses your files, type-checks them and emits `.js` (plus optional `.d.ts` declaration files and source maps). By default it emits **even when there are type errors**; `noEmitOnError` changes that.

In most modern projects, a fast tool strips types file by file without checking them (Vite and esbuild, SWC, Babel in Expo/Metro), so type-checking is a separate step:

```bash
npx tsc --noEmit        # type-check only (editor, pre-commit hook, CI)
npx tsc --noEmit --watch
```

- `next build` runs the type checker as part of the build; Expo's Metro bundler only strips types, so run `tsc` yourself.
- Because those tools see one file at a time, some cross-file features are restricted (`isolatedModules`, see [TypeScript in practice](04-ts-in-practice.md)).
- Recent Node versions can run `.ts` files directly by stripping types (erasable syntax only); check the Node release notes for your version. A Go-based native compiler was previewed in 2025 as the basis of TypeScript 7; treat its details as "check the announcements".

## 3. Annotations vs inference

```ts
let count = 0;                 // inferred: number
const status = 'active';       // inferred: 'active' (a const can never change)
let mode = 'dark';             // inferred: string (widened, because let can be reassigned)

type Listing = { id: string; priceCents: number };
const cart: Listing[] = [];             // annotate empty arrays
let selected: Listing | null = null;    // annotate values that start as null

function total(items: Listing[]): number {             // annotate parameters
  return items.reduce((sum, item) => sum + item.priceCents, 0); // callback types come from context
}
```

Rules of thumb:
- **Annotate:** function parameters, return types of exported functions (a stable contract and clearer errors), empty or null-initialized values, and public object shapes.
- **Let inference work:** local variables with obvious initializers and callback parameters (contextual typing gives `item` its type from `items`).

## 4. Primitives, arrays and tuples

```ts
const id: number = 42;                 // integers and floats alike
const big: bigint = 9007199254740993n;
const title: string = 'Charizard';
const active: boolean = true;
const nothing: null = null;
const missing: undefined = undefined;
const key: symbol = Symbol('key');

const ids: number[] = [1, 2, 3];
const names: Array<string> = ['a', 'b'];
const fixedIds: readonly number[] = [1, 2];
// fixedIds.push(3);  // error: Property 'push' does not exist on type 'readonly number[]'

type Point = [x: number, y: number];             // labeled tuple elements (labels are documentation)
type HttpResult = [status: number, body?: string]; // optional element
type Command = [name: string, ...args: string[]];  // rest element
const cmd: Command = ['git', 'commit', '-m', 'message'];
const pair = [1, 'one'] as const;                 // readonly [1, 'one']

function minMax(values: number[]): [min: number, max: number] {
  return [Math.min(...values), Math.max(...values)];
}
const [min, max] = minMax([3, 9, 1]);
```

Use lowercase `string`, `number`, `boolean`, never the wrapper types `String`, `Number`, `Boolean`. Tuples are what `useState` returns (`[value, setValue]`), which is why you can destructure them with any names.

## 5. Object types

```ts
type Product = {
  readonly id: string;     // cannot be reassigned through this type
  title: string;
  priceCents: number;
  description?: string;    // optional: may be missing (reads as string | undefined)
  tags: string[];
};

const product: Product = { id: 'p1', title: 'Booster box', priceCents: 12000, tags: [] };
// product.id = 'p2';      // error: Cannot assign to 'id' because it is a read-only property
product.description?.trim(); // must handle undefined
```

`readonly` is shallow and compile-time only (`product.tags.push('x')` still works). For dictionaries use an index signature or `Record<string, number>` (see [advanced types](03-advanced-types.md)).

## 6. interface vs type

```ts
interface Member {
  id: string;
  name: string;
}
interface Admin extends Member {        // extension with `extends`
  permissions: string[];
}

type Account = { id: string; name: string };
type OrgOwner = Account & { orgId: string };  // composition with an intersection

type Status = 'active' | 'suspended';        // only a type alias can name a union
type Pair = [string, number];                // or a tuple
type Formatter = (value: number) => string;  // function types read best as aliases

// Declaration merging: interfaces with the same name combine
interface Settings { theme: string }
interface Settings { locale: string }
const settings: Settings = { theme: 'dark', locale: 'en' };
```

| Feature | `interface` | `type` alias |
|---|---|---|
| Object shapes | yes | yes |
| Unions, tuples, primitives | no | yes |
| Function types | via call signatures | yes, directly |
| Mapped and conditional types | no | yes |
| Declaration merging | yes (reopen and add members) | no (duplicate identifier error) |
| Extending | `extends`; incompatible members are an error | `&`; incompatible members silently become `never` |
| `implements` in a class | yes | yes, if it is an object type (not a union) |
| Error messages | shown by name | sometimes expanded inline |

```ts
interface Base { id: string }
// interface Broken extends Base { id: number }  // error: incorrectly extends interface 'Base'
type Weird = { id: string } & { id: number };     // no error here, but Weird['id'] is never
```

Pick one convention and stay consistent. A common one: `interface` for object shapes that others extend or augment (library types, framework augmentation), `type` for unions, tuples, function types and compositions. Many teams simply use `type` everywhere and reach for `interface` only when they need merging.

## 7. Unions and intersections

```ts
type Id = string | number;

function formatId(id: Id) {
  // id.toUpperCase();   // error: Property 'toUpperCase' does not exist on type 'number'
  return typeof id === 'string' ? id.toUpperCase() : id.toFixed(0); // narrowed per branch
}

type Timestamps = { createdAt: Date; updatedAt: Date };
type Post = { id: string; title: string } & Timestamps; // needs all four fields
```

A union value can be any member, so only shared members are usable until you narrow. An intersection must satisfy every member; `string & number` is `never` because no value is both.

## 8. Literal types

```ts
type Method = 'GET' | 'POST' | 'PUT' | 'DELETE';

function request(url: string, method: Method) {
  return fetch(url, { method });
}

request('/api/orders', 'GET');
// request('/api/orders', 'get');      // error: '"get"' is not assignable to type 'Method'

const options = { method: 'GET' };      // property widened to string
// request('/api/orders', options.method); // error: 'string' is not assignable to 'Method'
const fixed = { method: 'GET' } as const; // method: 'GET', readonly
request('/api/orders', fixed.method);     // ok
```

Object properties widen because they are mutable. Fix it with `as const`, an explicit annotation (`const options: { method: Method } = ...`) or `satisfies` (see [advanced types](03-advanced-types.md)).

## 9. Enums vs unions of literals vs `as const` objects

```ts
// 1. Enum: emits a runtime object
enum Role {
  Admin = 'admin',
  Member = 'member',
}
function canDelete(role: Role) {
  return role === Role.Admin;
}
canDelete(Role.Admin);
// canDelete('admin');        // error: a string literal is not assignable to the enum type

// 2. Union of string literals: zero runtime cost
type RoleName = 'admin' | 'member';
function canEdit(role: RoleName) {
  return role === 'admin';
}
canEdit('admin');

// 3. `as const` object: runtime values plus a derived type
const ROLES = { Admin: 'admin', Member: 'member' } as const;
type RoleValue = (typeof ROLES)[keyof typeof ROLES]; // 'admin' | 'member'
const roleOptions: RoleValue[] = Object.values(ROLES); // for dropdowns and validation
canEdit(ROLES.Admin);
```

| | `enum` | Union of literals | `as const` object |
|---|---|---|---|
| Runtime code | yes (an object) | none | a plain object you wrote |
| List all values at runtime | yes (numeric enums also contain reverse-mapping keys) | no | `Object.values` |
| Accepts a plain `'admin'` literal | no (string enums) | yes | yes |
| Works with type-stripping tools | needs a real transform | yes | yes |
| Typical use | legacy code, some style guides | most props, statuses and API fields | values needed at runtime (selects, Zod enums) |

Numeric enums create a reverse mapping (`Role[0]`) and are loosely compatible with `number`. `const enum` inlines values but breaks with per-file transpilers (`isolatedModules`). Recent TypeScript versions add an `erasableSyntaxOnly` option that rejects enums entirely, aimed at Node's type stripping. For these reasons most modern codebases prefer unions or `as const` objects.

## 10. any vs unknown vs never vs void

```ts
const raw = '{"x": 1}';

const loose: any = JSON.parse(raw);
// loose.foo.bar();            // compiles, then crashes at runtime: checking is off

const safe: unknown = JSON.parse(raw);
// safe.x;                     // error: 'safe' is of type 'unknown'
if (typeof safe === 'object' && safe !== null && 'x' in safe) {
  console.log(safe.x);         // narrowed, so access is allowed
}

function fail(message: string): never {
  throw new Error(message);    // never returns normally
}

function logEvent(name: string): void {
  console.log(name);           // nothing useful is returned
}
```

| Type | What can be assigned to it | What you can do with it | Typical use |
|---|---|---|---|
| `any` | anything | anything, unchecked (and it spreads) | escape hatch while migrating JavaScript |
| `unknown` | anything | nothing until you narrow | external data, `JSON.parse`, `catch (err)` |
| `never` | nothing (only `never`) | n/a | functions that always throw, impossible branches, exhaustive checks |
| `void` | `undefined` (and any function's result when it is a callback return type) | ignore it | functions without a meaningful return value |

`catch (err)` gives `err: unknown` under `strict` (`useUnknownInCatchVariables`), so narrow with `err instanceof Error` before reading `err.message`.

## 11. Type assertions and their risks

```ts
const input = document.getElementById('email') as HTMLInputElement; // "trust me, it is an input"
const priceEl = document.querySelector('.price')!;                  // "trust me, it is not null"

type Customer = { id: string; name: string };
const customer = JSON.parse('{"id": 1}') as Customer; // compiles; customer.name.trim() would crash

// const n = 'hello' as number;          // error: conversion may be a mistake
const n = 'hello' as unknown as number;  // double assertion forces it: a red flag in code review
```

Assertions never check anything at runtime; they only silence the compiler. Prefer narrowing (`if (el instanceof HTMLInputElement)`), handling `null` explicitly, or runtime validation for external data. Reserve `as` for cases where you genuinely know more than the compiler (and `as const` is a different, safe feature).

## 12. Function types, optional parameters and overloads

```ts
// Parameter types, a default and an optional parameter
function formatPrice(cents: number, currency = 'PKR', locale?: string): string {
  return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(cents / 100);
}

// Function type aliases and rest parameters
type PriceComparator = (a: Listing, b: Listing) => number;
const byPrice: PriceComparator = (a, b) => a.priceCents - b.priceCents;

function sum(...values: number[]): number {
  return values.reduce((total, v) => total + v, 0);
}

// Overloads: several call signatures, one implementation
function toArray(value: string): string[];
function toArray(value: number): number[];
function toArray(value: string | number): (string | number)[] {
  return [value];
}
const letters = toArray('x');  // string[]
const numbers = toArray(1);    // number[]
// The implementation signature itself is not callable from outside.
```

- Optional parameters must come after required ones; a default makes a parameter optional too.
- A callback type returning `void` accepts functions that return values: `[1, 2].forEach((n) => results.push(n))` is fine although `push` returns a number. The value is simply ignored.
- Prefer a union parameter over overloads when the return type does not depend on the input; overloads shine when it does.

## 13. Structural typing and excess property checks

```ts
type Point2D = { x: number; y: number };
function distance(p: Point2D) {
  return Math.hypot(p.x, p.y);
}

const point3D = { x: 3, y: 4, z: 5 };
distance(point3D);                 // ok: it has x and y; the extra z is fine
// distance({ x: 3, y: 4, z: 5 }); // error: excess property check on a fresh object literal

class Coordinates {
  constructor(public x: number, public y: number) {}
}
distance(new Coordinates(3, 4));   // ok: the shape matches; the class name is irrelevant
```

TypeScript compares shapes, not names (structural typing), so any value with the required members fits. Excess property checks are a deliberate exception for **fresh object literals** in assignments, arguments and returns: an extra key there is usually a typo (`{ nmae: 'x' }`). When you need nominal behavior (a `UserId` that is not just any string), use branded types (see [advanced types](03-advanced-types.md)).

## 14. Narrowing basics

```ts
function describe(value: string | number | Date | null | string[]) {
  if (value === null) return 'nothing';                      // equality
  if (typeof value === 'string') return value.toUpperCase(); // typeof
  if (typeof value === 'number') return value.toFixed(2);
  if (value instanceof Date) return value.toISOString();     // instanceof
  return value.join(', ');                                   // only string[] is left
}

type Card = { kind: 'card'; last4: string };
type Wallet = { kind: 'wallet'; provider: string };

function paymentLabel(method: Card | Wallet) {
  if ('last4' in method) return `Card ending ${method.last4}`; // the in operator
  return method.provider;
}
```

The compiler follows control flow (returns, `if`, `switch`, assignments) and narrows types in each branch. Be careful with truthiness narrowing: `if (count)` also removes a valid `0`. Discriminated unions (`switch (method.kind)`) and custom type guards are covered in [advanced types](03-advanced-types.md).

## 15. strictNullChecks

Without `strictNullChecks`, `null` and `undefined` are assignable to every type, so the compiler never warns about them. With it, they must be part of the type and handled:

```ts
const customers: Customer[] = [];

function findCustomer(id: string): Customer | undefined {
  return customers.find((c) => c.id === id); // find returns Customer | undefined
}

const found = findCustomer('1');
// found.name;                    // error: 'found' is possibly 'undefined'
const label = found?.name ?? 'Guest';
if (found) console.log(found.name); // narrowed
```

`strict: true` enables the whole family: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict` and `useUnknownInCatchVariables`. Details in [TypeScript in practice](04-ts-in-practice.md).

## Common pitfalls
- Treating types as runtime validation, for example casting API responses with `as`.
- `any` creeping in through `JSON.parse`, untyped libraries or loose configs, then spreading through the codebase.
- Silencing errors with `as` and `!` instead of fixing the cause.
- Using `Object`, `String` or `{}` as types; they accept almost anything. Use `object`, `string` or `Record<string, unknown>`.
- Forgetting that `readonly` and `private` exist only at compile time.
- Enums in projects that use per-file transpilers or Node type stripping.
- Confusing `{ a?: string }` (the key may be missing) with `{ a: string | undefined }` (the key must be present).
- Truthiness checks that drop valid `0` or `''`.
- Expecting excess property checks on variables; they only apply to fresh object literals.

## How to talk about this in an interview
- Frame TypeScript as a daily tool, not a buzzword: it types your React and React Native (Expo) components at Devntech, the oRPC + TanStack Start + Drizzle platform where types flow from the database to the client, and the Next.js API routes at RemoteDev.
  > **Fill in:** how strict the tsconfig was in each project (`strict`, `noUncheckedIndexedAccess`), and one bug TypeScript caught for you before it shipped.
- Be honest about the limits: "types disappear at runtime, so I validate external input with Zod and derive the types from the schemas". This answer signals maturity.
  > **Fill in:** a time an `as` cast or `any` hid a real bug, and how you fixed it.
- For "interface vs type", give the technical differences (merging, unions, extends vs intersection), then your team convention. Avoid claiming one is always better.

## Interview questions

**Q1. Why use TypeScript, and what does it not do for you?**
It adds static types that catch errors before runtime, power autocomplete and refactoring, and document contracts. It does not validate data at runtime, does not make code faster, and cannot protect you from wrong assertions or `any`.
*Follow-up:* How do you protect runtime boundaries? Validate with a schema library such as Zod and infer types from the schemas.

**Q2. What happens to TypeScript types at runtime?**
They are erased; the output is plain JavaScript. Only a few constructs emit code (enums, namespaces, parameter properties, decorators). That is why `typeof x === 'User'` is impossible and runtime checks need real JavaScript.
*Follow-up:* How can a library check types at runtime then? With schemas or type guards written in JavaScript.

**Q3. What is the difference between `interface` and `type`?**
Both describe object shapes. Only `type` can name unions, tuples, primitives and mapped or conditional types; only `interface` supports declaration merging and reports incompatible `extends` as errors (an intersection silently produces `never`). Choose one convention per codebase.
*Follow-up:* When do you need an interface? To augment third-party types through declaration merging.

**Q4. Explain `any`, `unknown`, `never` and `void`.**
`any` disables checking; `unknown` accepts anything but must be narrowed before use; `never` has no values and marks impossible code (throwing functions, exhaustive checks); `void` marks a return value that should be ignored.
*Follow-up:* What type should `JSON.parse` results get? `unknown`, then validate.

**Q5. Enums vs union types vs `as const` objects?**
Enums emit a runtime object and string enums reject plain literals; unions of literals cost nothing at runtime and are easy to use; `as const` objects give runtime values plus a derived union type. Most modern code prefers unions or `as const` because they are erasable and simpler.
*Follow-up:* What is wrong with `const enum`? Per-file transpilers cannot inline it across files.

**Q6. What is structural typing? What are excess property checks?**
Compatibility is decided by shape: any value with the required members fits, whatever its name or class. Excess property checks are a targeted exception: fresh object literals with unknown keys are rejected because that usually means a typo.
*Follow-up:* How do you get nominal-style types? Branded types such as `string & { __brand: 'UserId' }`.

**Q7. Annotation vs assertion: why are `as` and `!` risky?**
An annotation (`const x: User = ...`) asks the compiler to check the value; an assertion (`value as User`, `value!`) tells it to trust you without any runtime check. Wrong assertions compile and crash later.
*Follow-up:* When is `as` acceptable? When you truly know more than the compiler, such as a DOM element type you control; prefer narrowing or validation otherwise.

**Q8. When do you annotate types, and when do you rely on inference?**
Annotate parameters, exported function return types, empty or null-initialized values and public object shapes. Let inference handle locals with obvious initializers and callback parameters.
*Follow-up:* Why annotate return types of exported functions? A stable contract, clearer errors at the source, and faster checking in big projects.

**Q9. What does `strictNullChecks` do, and what else does `strict` turn on?**
It removes `null` and `undefined` from every type unless you include them, so the compiler forces you to handle them. `strict` also enables `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict` and `useUnknownInCatchVariables`.
*Follow-up:* How do you turn it on in a legacy project? Flag by flag, fixing errors module by module.

**Q10. Union vs intersection: what is `string & number`?**
A union `A | B` is a value of either type; an intersection `A & B` must satisfy both. `string & number` is `never` since no value is both; intersections are mostly used to combine object types.
*Follow-up:* Which members can you use on `A | B` without narrowing? Only those common to both.

**Q11. What are literal types, and why does `let` widen but `const` does not?**
A literal type is a single value such as `'GET'` or `42`. A `const` binding can never change, so it keeps the literal type; a `let` (or an object property) can be reassigned, so it widens to `string` or `number`. Use `as const` or annotations to keep literals.
*Follow-up:* What does `as const` do to an object? Makes every property readonly and literal-typed.

**Q12. What are tuples? Where do you see them in React?**
Fixed-length arrays with a type per position, such as `[number, string]`, with optional, rest and readonly variants. `useState` returns `[state, setState]`, and custom hooks often return tuples so callers can name the parts.
*Follow-up:* Why return `as const` from a custom hook? Otherwise TypeScript infers an array of a union instead of a tuple.

**Q13. How do function overloads work, and when should you use them?**
You write several signatures followed by one implementation whose signature is hidden from callers. Use them when the return type depends on the argument types; if it does not, a union parameter is simpler.
*Follow-up:* Alternative for type-dependent returns? A generic with a conditional return type, at the cost of assertions inside.

**Q14. What is the difference between `name?: string` and `name: string | undefined`?**
The optional form allows the key to be missing entirely; the second requires the key but allows the value `undefined`. With `exactOptionalPropertyTypes`, the optional form also forbids explicitly assigning `undefined`.
*Follow-up:* Which matches a form that may omit a field? The optional form.

**Q15. `readonly` vs `const` vs `Object.freeze`?**
`const` stops reassigning a variable; `readonly` stops writes through a type at compile time (shallow, erased at runtime); `Object.freeze` stops changes at runtime (shallow too). They complement each other.
*Follow-up:* How do you type a frozen config? `as const` gives deeply readonly literal types.

**Q16. How do you narrow a union type?**
With `typeof`, `instanceof`, the `in` operator, equality checks, truthiness, discriminant properties in a `switch`, `Array.isArray` and custom type guards (`value is T`). The compiler tracks narrowing through control flow.
*Follow-up:* What is the risk of truthiness narrowing? It also removes `0`, `''` and `false`.

**Q17. Why can a function that returns a number be assigned to a `() => void` type?**
A `void` return in a function type means "the caller ignores the result", so returning something is harmless. This lets you write `arr.forEach((x) => list.push(x))` even though `push` returns a number.
*Follow-up:* Does that apply to declared functions? No: a function declared with a `void` return type cannot return a value.

**Q18. Does `tsc` stop emitting JavaScript when there are type errors? How do modern build tools handle types?**
By default no, it still emits; `noEmitOnError` changes that. Bundlers such as Vite (esbuild), SWC and Metro (Babel) strip types without checking them, so you run `tsc --noEmit` in the editor, a pre-commit hook or CI.
*Follow-up:* Which framework type-checks during its build? `next build` does.

## Further reading
- [TypeScript Handbook: Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
