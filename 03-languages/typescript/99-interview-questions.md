# TypeScript: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. "Will this compile?" items assume `strict: true` unless the question says otherwise.

## Fundamentals (see [01-fundamentals.md](01-fundamentals.md))

**Q1. What is TypeScript, and what happens to its types at runtime?**
A statically typed superset of JavaScript. The compiler checks types and then erases them, so the running program is plain JavaScript with no type information (except constructs that emit code, such as enums).

**Q2. `interface` vs `type`: what are the real differences?**
Only `type` can name unions, tuples, primitives and mapped or conditional types. Only `interface` supports declaration merging, and `extends` reports conflicting members as errors while an intersection silently produces `never` members.

**Q3. `any` vs `unknown`?**
Both accept any value. `any` also allows any operation without checks and spreads through your code; `unknown` allows nothing until you narrow it, so it is the safe choice for external data.

**Q4. Will this compile?**
```ts
let value: unknown = 'hello';
const length = value.length;
```
No: `'value' is of type 'unknown'`. Narrow first: `typeof value === 'string' ? value.length : 0`.

**Q5. When do you see `never`?**
As the return type of functions that always throw or never end, in impossible branches after exhaustive narrowing, in exhaustive `switch` checks, and as the result of impossible intersections like `string & number`.

**Q6. `void` vs `undefined` as a return type?**
`void` means "callers should ignore the result"; `undefined` means the function returns the value `undefined`. A function type returning `void` accepts functions that return something, which is why array callbacks are so flexible.

**Q7. Will this compile?**
```ts
type User = { name: string };
const u: User = { name: 'Sara', age: 25 };
```
No: excess property check. `age` does not exist in type `User`, and fresh object literals cannot carry unknown keys.

**Q8. Will this compile?**
```ts
type User = { name: string };
const data = { name: 'Sara', age: 25 };
const u: User = data;
```
Yes. `data` is not a fresh literal, and structurally it has everything `User` needs; extra properties are allowed.

**Q9. What is structural typing?**
Type compatibility is based on shape, not on names or declarations: any value with the required members is assignable, even an instance of an unrelated class.

**Q10. Will this compile?**
```ts
let method = 'GET';
function send(m: 'GET' | 'POST') {}
send(method);
```
No: `let` widens `method` to `string`, which is not assignable to `'GET' | 'POST'`. Use `const`, an annotation, or `as const`.

**Q11. What does `as const` do?**
It infers the narrowest types: literals instead of `string`/`number`, readonly properties, and readonly tuples instead of arrays. It is a compile-time feature with no runtime effect.

**Q12. Enums vs unions of string literals?**
Enums emit a runtime object and string enums reject plain literals; unions cost nothing at runtime and accept literals directly. Use an `as const` object when you also need the values at runtime.

**Q13. Will this compile?**
```ts
enum Status { Active = 'active', Inactive = 'inactive' }
const s: Status = 'active';
```
No: a string literal is not assignable to a string enum type. Use `Status.Active`.

**Q14. What is a tuple, and how is it different from an array?**
A fixed-length array with a type per position, such as `[string, number]`, with optional (`?`), rest (`...T[]`) and readonly variants. A plain array is variable length with one element type.

**Q15. Will this compile?**
```ts
const nums: readonly number[] = [1, 2, 3];
nums.push(4);
```
No: `Property 'push' does not exist on type 'readonly number[]'`.

**Q16. `name?: string` vs `name: string | undefined`?**
The first allows the key to be missing; the second requires the key but allows `undefined`. With `exactOptionalPropertyTypes`, the first also rejects an explicit `undefined`.

**Q17. Will this compile?**
```ts
function len(s?: string) {
  return s.length;
}
```
No: `'s' is possibly 'undefined'`. Use `s?.length ?? 0` or narrow with `if (s)`.

**Q18. What does the non-null assertion `!` do, and why is it risky?**
It removes `null` and `undefined` from a type without any runtime check. If you are wrong, the code crashes later with a "cannot read properties of undefined" error that the compiler promised could not happen.

**Q19. When is a type assertion (`as`) allowed?**
When the source and target types sufficiently overlap (one is assignable to the other). Otherwise the compiler rejects it, and only a double assertion through `unknown` forces it, which is a red flag.

**Q20. Will this compile?**
```ts
const n = 'hello' as number;
```
No: the conversion "may be a mistake" because `string` and `number` do not overlap. `'hello' as unknown as number` would compile and lie.

**Q21. Union vs intersection?**
`A | B` is a value of either type, so only shared members are usable until narrowing. `A & B` must satisfy both; for object types it combines their properties.

**Q22. Will this compile?**
```ts
type A = { kind: 'a'; a: number };
type B = { kind: 'b'; b: string };
function f(x: A | B) {
  return x.a;
}
```
No: `Property 'a' does not exist on type 'B'`. Narrow first: `if (x.kind === 'a') return x.a;`.

**Q23. How do function overloads work?**
Several call signatures followed by one implementation whose signature callers cannot see. Use them when the return type depends on the argument types; otherwise a union parameter is simpler.

**Q24. Will this compile?**
```ts
const handler: () => void = () => 42;
```
Yes. A `void` return in a function type means the result is ignored, so a function returning a number is acceptable.

**Q25. Name the ways to narrow a union.**
`typeof`, `instanceof`, `in`, equality and truthiness checks, discriminant property checks (`switch (x.kind)`), `Array.isArray`, user-defined type guards and assertion functions.

**Q26. What does `strict: true` enable?**
`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict` and `useUnknownInCatchVariables`.

## Generics and utility types (see [02-generics-utility-types.md](02-generics-utility-types.md))

**Q27. What are generics?**
Type parameters that make functions, types and classes reusable while preserving relationships between inputs and outputs, such as `first<T>(items: T[]): T | undefined`.

**Q28. What is a generic constraint?**
`extends` on a type parameter limits what it can be and unlocks its members: `<T extends { length: number }>` allows reading `.length`; `<K extends keyof T>` restricts keys.

**Q29. Will this compile?**
```ts
function getLength<T>(value: T) {
  return value.length;
}
```
No: `Property 'length' does not exist on type 'T'`. Add a constraint: `<T extends { length: number }>`.

**Q30. What do `keyof` and indexed access types give you?**
`keyof T` is the union of `T`'s keys; `T['key']` is the type of that property; `T[keyof T]` is the union of all property types; `(typeof arr)[number]` is an array's element type.

**Q31. Will this compile?**
```ts
function getProp<T, K extends keyof T>(obj: T, key: K) {
  return obj[key];
}
getProp({ id: 1 }, 'name');
```
No: `K` must be a key of `{ id: number }`, so `'name'` is rejected (`'"name"'` is not assignable to `'"id"'`).

**Q32. What do `Partial`, `Required` and `Readonly` do?**
They map over every property to make it optional, required or readonly: `{ [P in keyof T]?: T[P] }`, `{ [P in keyof T]-?: T[P] }`, `{ readonly [P in keyof T]: T[P] }`.

**Q33. `Pick` vs `Omit`?**
`Pick<T, K>` keeps the listed keys; `Omit<T, K>` drops them. `Pick` requires the keys to exist; `Omit` does not check them.

**Q34. Will this compile?**
```ts
type User = { id: string; name: string };
type NoEmail = Omit<User, 'email'>;
```
Yes: `Omit` accepts any key (its `K extends keyof any`), so this typo-like key silently omits nothing. A `StrictOmit<T, K extends keyof T>` helper catches it.

**Q35. What is `Record<K, V>` good for?**
Objects whose keys come from a known union (exhaustive label maps like `Record<OrderStatus, string>`) or dictionaries (`Record<string, number>`).

**Q36. Will this compile?**
```ts
type Status = 'draft' | 'published';
const labels: Record<Status, string> = { draft: 'Draft' };
```
No: `Property 'published' is missing`. That is the point: adding a status forces you to add its label.

**Q37. `Exclude` vs `Extract`?**
They filter union members: `Exclude<U, X>` removes members assignable to `X`; `Extract<U, X>` keeps only them. Both rely on distributive conditional types.

**Q38. How do you get the resolved type of an async function?**
`Awaited<ReturnType<typeof fn>>`. `ReturnType` alone gives `Promise<...>`, and `Awaited` unwraps it recursively.

**Q39. Implement `Pick`.**
`type MyPick<T, K extends keyof T> = { [P in K]: T[P] };`

**Q40. Implement `ReturnType`.**
`type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never;`

**Q41. What are default type parameters?**
Fallback types for type parameters, like `interface ApiResponse<T = unknown>`, so callers can omit them in the common case.

**Q42. How do you write a generic arrow function in a `.tsx` file?**
`const identity = <T,>(value: T) => value;` (or `<T extends unknown>`). Plain `<T>` would be parsed as a JSX tag.

**Q43. With `noUncheckedIndexedAccess` on, will this compile?**
```ts
const scores = [10, 20, 30];
const best = scores[0];
console.log(best.toFixed(1));
```
No: `best` is `number | undefined`, so `toFixed` needs a check first (`best?.toFixed(1)` or an `if`).

**Q44. When should you not use a generic?**
When the type parameter appears only once (it relates nothing; use `unknown` or the concrete type), when every caller uses the same type, or when the signature becomes harder to understand than two simple functions.

**Q45. `NonNullable` vs `Required`?**
`NonNullable<T>` removes `null` and `undefined` from a type (`T & {}`); `Required<T>` removes the optional modifier from an object's properties.

## Advanced types (see [03-advanced-types.md](03-advanced-types.md))

**Q46. What is a user-defined type guard?**
A function returning a type predicate, `(x: unknown): x is User`, so that `if (isUser(x))` narrows `x`. The compiler trusts the predicate without checking the body.

**Q47. Will this compile?**
```ts
function isString(x: unknown): boolean {
  return typeof x === 'string';
}
function shout(x: unknown) {
  if (isString(x)) return x.toUpperCase();
  return '';
}
```
No: the declared return type is plain `boolean`, so nothing narrows and `x` stays `unknown`. Declare `x is string` (TypeScript 5.5 can infer the predicate, but only when the return type is not annotated).

**Q48. What does an assertion function do?**
`function assertIsDefined<T>(v: T): asserts v is NonNullable<T>` throws when the check fails and narrows `v` for the rest of the scope after the call.

**Q49. What is a discriminated union?**
A union of object types sharing a literal property (`status`, `type`, `kind`); checking that property narrows to one member, which makes state machines and event handling type-safe.

**Q50. Will this compile?**
```ts
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'square'; side: number }
  | { kind: 'triangle'; base: number };
function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2;
    case 'square': return s.side ** 2;
    default: {
      const unreachable: never = s;
      return unreachable;
    }
  }
}
```
No: in `default`, `s` is still the triangle member, which is not assignable to `never`. That error is the exhaustive check working: handle `'triangle'`.

**Q51. What is a distributive conditional type?**
A conditional type whose checked type is a naked type parameter: given a union, it is applied to each member separately and the results are unioned. `[T] extends [U]` turns distribution off.

**Q52. What is `Exclude<'a' | 'b' | 'c', 'a' | 'c'>`?**
`'b'`. Each member is tested separately; `'a'` and `'c'` map to `never` and disappear.

**Q53. What does `infer` do?**
Declares a type variable inside a conditional type's `extends` clause that captures the matching part: `T extends Promise<infer U> ? U : T`.

**Q54. What is key remapping in mapped types?**
An `as` clause that renames or filters keys: `` { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] } ``, or `as never` to drop a key.

**Q55. Give a practical use of template literal types.**
Event names (`` `${Entity}.${Action}` ``), handler props (`` `on${Capitalize<E>}` ``), CSS units (`` `${number}px` ``) and extracting route params from a path string.

**Q56. `satisfies` vs a type annotation vs `as const`?**
An annotation checks and widens to the declared type; `satisfies` checks (including excess properties) but keeps the precise inferred type; `as const` narrows to readonly literals without checking. `as const satisfies T` does both.

**Q57. Will this compile?**
```ts
type Theme = Record<'primary' | 'secondary', string>;
const theme = { primary: '#000', secondary: '#fff', tertiary: '#ccc' } satisfies Theme;
```
No: `satisfies` performs excess property checks, so `tertiary` is rejected.

**Q58. Will this compile?**
```ts
const config = { mode: 'dark' } satisfies { mode: 'dark' | 'light' };
const m: 'dark' = config.mode;
```
Yes: `satisfies` supplies the literal union as context, so `config.mode` keeps the literal type `'dark'`. With an annotation (`const config: { mode: 'dark' | 'light' }`) it would be `'dark' | 'light'` and the assignment would fail.

**Q59. How do you add a property to a third-party type, such as `user` on Express's `Request`?**
Augment it: `declare global { namespace Express { interface Request { user?: AuthUser } } }` in a module file (with `export {}` if needed) included by tsconfig. Interfaces are open for merging; type aliases are not.

**Q60. What problem do branded types solve?**
They make structurally identical primitives incompatible (`UserId` vs `OrgId`, both strings), so mixing IDs is a compile error, which matters in multi-tenant code where an org ID must come from a trusted context.

**Q61. Will this compile?**
```ts
type UserId = string & { readonly __brand: 'UserId' };
function loadUser(id: UserId) {}
loadUser('u_123');
```
No: a plain `string` lacks the brand. Create branded values in one trusted place, such as a validating `toUserId()` function or a Zod schema with `.brand()`.

**Q62. What does `noUncheckedIndexedAccess` do?**
Adds `| undefined` to reads through index signatures and array indexes, forcing a check before using a value that might not exist.

**Q63. With `noUncheckedIndexedAccess` on, will this compile?**
```ts
const prices: Record<string, number> = { a: 1 };
const total = prices['a'] + 1;
```
No: `prices['a']` is `number | undefined`, so the addition is rejected until you handle `undefined`.

**Q64. Why does `Object.keys(obj)` return `string[]`?**
Structural typing allows objects to carry extra keys at runtime, so promising `(keyof T)[]` would be unsound. Cast only when you control the object.

**Q65. How do you type arbitrary JSON?**
With a recursive union: `type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue }`.

**Q66. Overloads vs conditional return types?**
Overloads for a few fixed call shapes (clear, no casts); conditional types for systematic type-level mappings in generic utilities (the implementation needs an assertion).

**Q67. Will this compile?**
```ts
interface Point { x: number }
interface Point { y: number }
const p: Point = { x: 1 };
```
No: the two declarations merge into `{ x: number; y: number }`, so `y` is missing.

**Q68. Will this compile?**
```ts
type Point = { x: number };
type Point = { y: number };
```
No: `Duplicate identifier 'Point'`. Type aliases cannot be reopened.

## TypeScript in practice (see [04-ts-in-practice.md](04-ts-in-practice.md))

**Q69. Which tsconfig options do you consider essential?**
`strict`, `noUncheckedIndexedAccess`, the right `module`/`moduleResolution` pair, `isolatedModules` or `verbatimModuleSyntax` with single-file transpilers, `skipLibCheck`, and `noEmit` when a bundler builds.

**Q70. `moduleResolution: bundler` vs `nodenext`?**
`bundler` for apps compiled by Vite, Next.js, webpack or Metro (extensionless imports allowed); `nodenext` for code Node runs directly and published packages (ES module files need `.js` extensions in relative imports).

**Q71. What does `isolatedModules` protect against?**
Code that a per-file transpiler (esbuild, SWC, Babel) cannot compile correctly without type information, such as re-exporting a type without `export type`.

**Q72. With `verbatimModuleSyntax` on, will this compile?**
```ts
import { User } from './types'; // User is only a type
export function greet(u: User) {
  return u.name;
}
```
No: a type must be imported with `import type { User }` (or `import { type User }`) when `verbatimModuleSyntax` is enabled.

**Q73. What do `skipLibCheck` and `esModuleInterop` do?**
`skipLibCheck` skips type-checking `.d.ts` files (faster, avoids dependency conflicts); `esModuleInterop` lets you default-import CommonJS modules such as `import express from 'express'`.

**Q74. How do you use a library that has no types?**
Install `@types/<name>` if DefinitelyTyped has it; otherwise write `declare module 'name' { ... }` describing what you use, or `declare module 'name';` as a temporary `any` shim.

**Q75. How do you type `children` in React?**
`children: ReactNode`, which accepts elements, strings, numbers, arrays, fragments and `null`. Use `ReactElement` only when exactly one element is required.

**Q76. Will this compile?**
```tsx
const [user, setUser] = useState(null);
setUser({ id: '1', name: 'Sara' });
```
No: the state type is inferred as `null`, so the object is rejected. Write `useState<User | null>(null)`.

**Q77. How do you type a change handler defined outside JSX?**
`const onChange = (e: ChangeEvent<HTMLInputElement>) => setValue(e.target.value);` with `ChangeEvent` imported from `react`. Inline handlers infer the event type automatically.

**Q78. How do you type a context so consumers never handle `null`?**
`createContext<Value | null>(null)` plus a custom hook that throws when the value is `null` and returns `Value`.

**Q79. How do you type `useRef` for a DOM element and for a mutable value?**
`useRef<HTMLInputElement>(null)` for DOM nodes (read with `ref.current?.`), and `useRef<number | null>(null)` for values you assign later, such as a timer id.

**Q80. Why do you need Zod if you already use TypeScript?**
Types are erased, so runtime data (request bodies, env vars, third-party responses, webhooks, storage) is unchecked. A schema validates it and produces the matching type, while `as` only silences the compiler.

**Q81. `parse` vs `safeParse`?**
`parse` returns data or throws a `ZodError`; `safeParse` returns `{ success: true, data }` or `{ success: false, error }` without throwing, which suits returning field errors.

**Q82. What does `z.infer<typeof Schema>` give you?**
The static output type of the schema, so the schema is the single source of truth for both validation and types.

**Q83. How do you make environment variables type-safe?**
Parse `process.env` with a Zod schema once at startup (coercing numbers, using enums and URL checks), fail fast on errors, and export the typed result as the only config access point.

**Q84. How do oRPC and tRPC give typed clients without code generation?**
The server router is a TypeScript value; its type is imported with `import type` on the client, and the client library maps that type into callable functions with inferred inputs and outputs.

**Q85. How do you get row types from a Drizzle table?**
`typeof table.$inferSelect` for rows returned by selects and `typeof table.$inferInsert` for inserts (columns with defaults become optional).

**Q86. Why use `import type`?**
The import is guaranteed to be erased, keeping server-only modules out of client bundles and avoiding side effects; it is required for types under `verbatimModuleSyntax`.

**Q87. What does typescript-eslint's `no-floating-promises` catch?**
Promises that are neither awaited nor given a rejection handler, which would otherwise produce unhandled rejections (fatal in recent Node versions).

**Q88. How would you migrate a JavaScript project to TypeScript?**
`allowJs`, convert leaf modules and API types first, JSDoc with `// @ts-check` for files that stay JavaScript, enable strict flags one at a time, replace `any` with `unknown` plus validation, and add lint rules so new code stays typed.

**Q89. `@ts-ignore` vs `@ts-expect-error`?**
Both suppress the next line's error, but `@ts-expect-error` fails once there is no error anymore, so stale suppressions get removed. Always include a reason.

**Q90. Will this compile?**
```ts
try {
  JSON.parse('{');
} catch (err) {
  console.log(err.message);
}
```
No under `strict`: `err` is `unknown` (`useUnknownInCatchVariables`). Use `err instanceof Error ? err.message : String(err)`.

**Q91. How do you read a long "is not assignable" error?**
Bottom-up: the last nested line names the actual mismatched property or type. Hover to inspect inferred types and split complex expressions into annotated variables.

**Q92. Does `tsc` emit JavaScript when there are type errors, and does TypeScript make code faster?**
It emits by default (unless `noEmitOnError` is set), and no: the output is ordinary JavaScript, so runtime speed is unchanged. The payoff is fewer bugs and safer refactoring.
