# Generics and Utility Types

> **Tier:** T1 (know cold) · **Related:** [TypeScript fundamentals](01-fundamentals.md), [Advanced types](03-advanced-types.md), [TypeScript in practice](04-ts-in-practice.md), [React patterns and architecture](../../06-react/07-patterns-and-architecture.md), [oRPC and type-safe APIs](../../09-nodejs-backend/05-orpc-and-typesafe-apis.md)
> **On your resume:** generic hooks, API clients and list components in your React and React Native (Expo) work at Devntech; typed API responses in the Next.js APIs at RemoteDev; and the inferred types that flow through the oRPC + Drizzle platform.

## Quick revision
- Generics are type parameters: they let one function, type or class work with many types while keeping the relationship between input and output. `any` throws that relationship away.
- Type arguments are usually inferred (`first(['a'])` gives `T = string`); pass them explicitly when inference has nothing to go on (`useState<User | null>(null)`).
- Constraints limit what `T` can be: `<T extends { length: number }>`, `<K extends keyof T>`.
- Defaults: `<T = unknown>`, like default function parameters.
- `keyof T` is the union of `T`'s keys; `typeof value` (in a type position) gives a value's type; `T['key']` is an indexed access type; `(typeof arr)[number]` is an element type.
- `getProperty<T, K extends keyof T>(obj: T, key: K): T[K]` is the classic example of a constraint plus indexed access.
- Utility types are mostly mapped and conditional types from the standard library: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited`, `InstanceType`.
- `Pick`/`Omit` work on object keys; `Extract`/`Exclude` work on union members.
- `Omit<T, K>` does not check that `K` exists in `T` (a typo silently omits nothing).
- `Awaited<ReturnType<typeof fn>>` is the resolved type of an async function.
- Useful generic patterns: `ApiResult<T>` (a discriminated success/failure), `Paginated<T>`, a typed fetch helper that validates with a schema, and generic React components such as `List<T>`.
- In `.tsx` files, write generic arrows as `<T,>(x: T) => ...` so the parser does not read `<T>` as JSX.
- A type parameter used only once is a smell: use the concrete type or `unknown` instead.

## 1. Why generics

```ts
// With any: the type information is lost
function firstAny(items: any[]): any {
  return items[0];
}
const lost = firstAny(['x']); // any: no autocomplete, no errors

// With a generic: the element type flows from input to output
function first<T>(items: T[]): T | undefined {
  return items[0];
}
const s = first(['x', 'y']);         // string | undefined
const n = first([1, 2, 3]);          // number | undefined
const explicit = first<string>([]);  // explicit type argument when inference has no data
```

A generic says "whatever type comes in, this related type comes out". The compiler fills in `T` at each call site, so you get full checking without writing one function per type.

## 2. Generic functions, interfaces, aliases and classes

```ts
// Generic arrow function (in .tsx files write <T,> to avoid JSX ambiguity)
const identity = <T>(value: T): T => value;

// Generic interface and type aliases
interface Box<T> {
  value: T;
}
type Nullable<T> = T | null;
type Paginated<T> = {
  items: T[];
  nextCursor: string | null;
  total: number;
};
const page: Paginated<{ id: string }> = { items: [{ id: 'a' }], nextCursor: null, total: 1 };

// Generic class
class TypedCache<K, V> {
  private store = new Map<K, V>();

  set(key: K, value: V): this {
    this.store.set(key, value);
    return this;
  }

  get(key: K): V | undefined {
    return this.store.get(key);
  }
}

const prices = new TypedCache<string, number>();
prices.set('sku-1', 1200);
// prices.set('sku-2', '1200'); // error: 'string' is not assignable to 'number'
```

Static members cannot use a class's type parameters (statics belong to the class itself, not to one instantiation).

## 3. Constraints and default type parameters

```ts
// T must have a numeric length (strings, arrays, anything with .length)
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest([1, 2, 3], [1]);   // [1, 2, 3]
longest('abc', 'de');      // 'abc'
// longest(10, 20);        // error: 'number' does not have a 'length' property

// Defaults work like default parameters
interface ApiResponse<TData = unknown, TError = { message: string }> {
  data?: TData;
  error?: TError;
}
const anything: ApiResponse = {};                     // TData = unknown
const names: ApiResponse<string[]> = { data: ['a'] }; // TError still defaults
```

Constraints let you use members of `T` inside the function; defaults reduce noise at use sites (TanStack Query's hooks rely on many defaulted type parameters).

## 4. keyof, typeof and indexed access

```ts
type User = {
  id: string;
  name: string;
  address: { city: string; zip: string };
};

type UserKey = keyof User;              // 'id' | 'name' | 'address'
type City = User['address']['city'];    // string
type IdOrName = User['id' | 'name'];    // string

const defaultSettings = { theme: 'light', pageSize: 20 };
type Settings = typeof defaultSettings;            // { theme: string; pageSize: number }
type SettingKey = keyof typeof defaultSettings;    // 'theme' | 'pageSize'

const statuses = ['draft', 'published', 'archived'] as const;
type Status = (typeof statuses)[number];           // 'draft' | 'published' | 'archived'
```

`typeof` in a type position reads the type of a value, which lets one runtime constant be the single source of truth for both values and types (the `statuses` array can feed a dropdown and a type).

## 5. getProperty and friends

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: 'u1', name: 'Sara', address: { city: 'Lahore', zip: '54000' } };
const address = getProperty(user, 'address'); // { city: string; zip: string }
const userName = getProperty(user, 'name');   // string
// getProperty(user, 'email');                // error: '"email"' is not assignable to 'keyof User'

function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}
const allNames = pluck([user], 'name');        // string[]
```

How it works: `K extends keyof T` limits `key` to real keys of `T`; at the call site `K` is inferred as the literal `'address'`, so the return type `T[K]` resolves to `User['address']`. The relationship between the two parameters is exactly what `any` or `string` would lose.

## 6. Built-in utility types and how they are built

Most utility types are a line or two of mapped or conditional types (conditional types and `infer` are explained in [advanced types](03-advanced-types.md)). Seeing the implementations makes them easy to remember:

```ts
type MyPartial<T> = { [P in keyof T]?: T[P] };
type MyRequired<T> = { [P in keyof T]-?: T[P] };           // -? removes optionality
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyRecord<K extends keyof any, V> = { [P in K]: V };    // keyof any = string | number | symbol
type MyExclude<T, U> = T extends U ? never : T;             // distributes over unions
type MyExtract<T, U> = T extends U ? T : never;
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;
type MyNonNullable<T> = T & {};                             // removes null and undefined
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
type MyParameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type MyAwaited<T> = T extends PromiseLike<infer U> ? MyAwaited<U> : T; // simplified version
type MyInstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

| Utility | What it does | Typical real use |
|---|---|---|
| `Partial<T>` | every property optional | PATCH payloads, edit forms, merging defaults |
| `Required<T>` | every property required | config after defaults are applied |
| `Readonly<T>` | every property readonly | props, frozen config, state you must not mutate |
| `Pick<T, K>` | keep the listed keys | a public user DTO, a list-item view model |
| `Omit<T, K>` | drop the listed keys | create input without `id`/`createdAt`, a user without `passwordHash` |
| `Record<K, V>` | object with keys `K` and values `V` | `Record<OrderStatus, string>` label maps, lookup tables |
| `Exclude<U, X>` | remove union members assignable to `X` | statuses without `'archived'` |
| `Extract<U, X>` | keep union members assignable to `X` | the events of a union that carry a payload |
| `NonNullable<T>` | remove `null` and `undefined` | values after a guard or filter |
| `ReturnType<F>` | the return type of a function type | types of hooks, selectors and server functions |
| `Parameters<F>` | the parameter types as a tuple | wrappers that forward arguments (logging, retry) |
| `Awaited<T>` | unwraps promises recursively | `Awaited<ReturnType<typeof fetchUser>>` |
| `InstanceType<C>` | the instance type of a class constructor | factories, dependency injection containers |

```ts
type Product = { id: string; title: string; priceCents: number; createdAt: Date };

type CreateProductInput = Omit<Product, 'id' | 'createdAt'>;  // what the client sends
type UpdateProductInput = Partial<CreateProductInput>;        // PATCH: any subset
type ProductCard = Pick<Product, 'id' | 'title'>;             // what a list item needs

type OrderStatus = 'pending' | 'paid' | 'shipped' | 'cancelled';
const statusLabel: Record<OrderStatus, string> = {             // must cover every status
  pending: 'Pending',
  paid: 'Paid',
  shipped: 'Shipped',
  cancelled: 'Cancelled',
};
type OpenStatus = Exclude<OrderStatus, 'shipped' | 'cancelled'>; // 'pending' | 'paid'

async function fetchProduct(id: string) {
  return { id, title: 'Booster box', priceCents: 12000, createdAt: new Date() };
}
type FetchedProduct = Awaited<ReturnType<typeof fetchProduct>>; // the object, not a Promise
type FetchArgs = Parameters<typeof fetchProduct>;               // [id: string]

class ApiClient {
  constructor(public baseUrl: string) {}
}
type Client = InstanceType<typeof ApiClient>;                   // ApiClient
```

Adding a new status to `OrderStatus` makes the `statusLabel` object fail to compile until you add its label, which is exactly the kind of safety you want in a status-heavy app (orders, bookings, rent payments).

### The Omit typo trap

```ts
type NoError = Omit<Product, 'titel'>;            // compiles: Omit accepts any key, omits nothing
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
// type Caught = StrictOmit<Product, 'titel'>;    // error: '"titel"' does not satisfy 'keyof Product'
```

`Pick` constrains its keys to `keyof T`, but `Omit` deliberately does not (so it works with unions of keys that may not all exist). A `StrictOmit` helper catches typos.

### Going deeper
- Other built-ins: `ConstructorParameters`, `ThisParameterType`, `OmitThisParameter`, and the intrinsic string types `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.
- `NoInfer<T>` (TypeScript 5.4) stops a parameter from influencing inference: in `function pick<T extends string>(options: T[], fallback: NoInfer<T>)`, calling `pick(['sm', 'md'], 'xl')` is an error instead of silently widening `T`.
- `const` type parameters (TypeScript 5.0): `function routes<const T extends readonly string[]>(paths: T)` infers `readonly ['a', 'b']` instead of `string[]` without the caller writing `as const`.

## 7. Practical generic patterns

### A generic API result wrapper

```ts
type ApiSuccess<T> = { ok: true; data: T };
type ApiFailure = { ok: false; error: { code: string; message: string } };
type ApiResult<T> = ApiSuccess<T> | ApiFailure;

function unwrap<T>(result: ApiResult<T>): T {
  if (result.ok) return result.data;    // narrowed to ApiSuccess<T>
  throw new Error(`${result.error.code}: ${result.error.message}`);
}

const listResult: ApiResult<Paginated<Product>> = {
  ok: true,
  data: { items: [], nextCursor: null, total: 0 },
};
const products = unwrap(listResult).items;  // Product[]
```

The `ok` literal is a discriminant, so checking it narrows the union (see [advanced types](03-advanced-types.md)). The same shape works for server actions, oRPC procedure results or a mobile API client.

### A typed fetch helper

```ts
import { z } from 'zod';

class HttpError extends Error {
  constructor(public status: number, public body: string) {
    super(`HTTP ${status}`);
  }
}

// Version 1: the caller picks T. Convenient, but the cast is a promise, not a check.
async function getJson<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init);
  if (!res.ok) throw new HttpError(res.status, await res.text());
  return (await res.json()) as T;
}

// Version 2: a schema validates at runtime and supplies the type.
async function getValidated<T>(url: string, schema: z.ZodType<T>): Promise<T> {
  const res = await fetch(url);
  if (!res.ok) throw new HttpError(res.status, await res.text());
  return schema.parse(await res.json()); // throws a ZodError if the shape is wrong
}

const ProductSchema = z.object({ id: z.string(), title: z.string(), priceCents: z.number() });

async function loadProduct() {
  const product = await getValidated('/api/products/1', ProductSchema); // inferred from the schema
  return product.title;
}
```

Version 1 is what many codebases have; version 2 is what you should argue for at trust boundaries (third-party APIs, webhooks, anything you do not control). More in [TypeScript in practice](04-ts-in-practice.md).

### A generic React list component

```tsx
import type { ReactNode } from 'react';

type ListProps<T> = {
  items: T[];
  getKey: (item: T) => string | number;
  renderItem: (item: T) => ReactNode;
  emptyText?: string;
};

export function List<T>({ items, getKey, renderItem, emptyText = 'Nothing here yet' }: ListProps<T>) {
  if (items.length === 0) return <p>{emptyText}</p>;
  return (
    <ul>
      {items.map((item) => (
        <li key={getKey(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

type Booking = { id: string; guest: string };

export function BookingList({ bookings }: { bookings: Booking[] }) {
  // T is inferred as Booking, so `b` is fully typed in both callbacks
  return <List items={bookings} getKey={(b) => b.id} renderItem={(b) => <strong>{b.guest}</strong>} />;
}
```

The same idea powers typed wrappers around `FlatList` in React Native and table components on the web. For arrow-function components in `.tsx`, write `const List = <T,>(props: ListProps<T>) => ...`.

## 8. When not to use generics

- If `T` appears only once, there is no relationship to preserve: `function log<T>(value: T): void` is just `function log(value: unknown): void`.
- If every call uses the same type, use that type.
- If the signature needs several type parameters, conditional types and casts inside, consider overloads or simpler, separate functions.
- Generic code is harder to read; favor clarity in application code and save heavy generics for shared utilities.

## Common pitfalls
- Using `any` where a generic (or `unknown`) would keep type information.
- `getJson<User>(url)` treated as validation; it is only a cast.
- Forgetting that `Omit` does not check its keys.
- Confusing `Pick`/`Omit` (object keys) with `Extract`/`Exclude` (union members).
- `ReturnType<typeof asyncFn>` giving `Promise<...>` when you wanted the resolved value; wrap with `Awaited`.
- Writing `<T>(x: T) => x` in a `.tsx` file (parsed as JSX); use `<T,>`.
- Over-constraining (`T extends object` when you only need one property) or under-constraining (accessing `value.id` on an unconstrained `T`).
- Explicit type arguments everywhere when inference would do; they add noise and can hide inference problems.

## How to talk about this in an interview
- Anchor generics to code you write: a typed API client or hook, a generic list or select component in React or React Native, and types inferred from Drizzle tables and oRPC procedures on the platform migration at Devntech.
  > **Fill in:** a generic helper or component you wrote or used (an API wrapper, a `useFetch<T>`-style hook, a typed `FlatList` wrapper) and what it saved.
- Show you know the utility types by naming a use: "`Omit<User, 'passwordHash'>` for the public user shape, `Partial` for PATCH bodies, `Record<Status, string>` for labels".
  > **Fill in:** utility types that appear in the codebases you work on, for example derived input types for create and update forms.
- If asked to implement `Partial` or `Pick`, write the mapped type and explain `keyof` and `in`; that is usually the whole question.

## Interview questions

**Q1. What are generics, and why not just use `any`?**
Type parameters that let one function, type or class work with many types while preserving relationships, such as "the return type is the element type of the input array". `any` accepts everything but returns `any`, so autocomplete and checks are lost downstream.
*Follow-up:* What is the safe alternative when you truly do not know the type? `unknown`, then narrow.

**Q2. What is a generic constraint? Give an example.**
A bound on what a type parameter can be, written with `extends`: `<T extends { length: number }>` lets the function read `.length`; `<K extends keyof T>` restricts a key to real keys of `T`. Without a constraint, `T` could be anything, so no members are available.
*Follow-up:* Can a constraint reference another type parameter? Yes, `K extends keyof T` does exactly that.

**Q3. Explain `getProperty<T, K extends keyof T>(obj: T, key: K): T[K]`.**
`K` is limited to `T`'s keys, so passing a non-existent key is an error. At the call site `K` is inferred as a literal key such as `'name'`, so the return type `T[K]` becomes the exact type of that property instead of a union of all property types.
*Follow-up:* What would `key: keyof T` (without a second type parameter) return? `T[keyof T]`, the union of all property types.

**Q4. What do `keyof`, `typeof` and indexed access types do?**
`keyof T` produces a union of `T`'s keys; `typeof value` in a type position gives the type of a runtime value; `T['prop']` looks up a property type, and `(typeof arr)[number]` gives an array's element type. Together they derive types from existing values instead of duplicating them.
*Follow-up:* How do you get a union from a constant array? Declare it `as const`, then `(typeof arr)[number]`.

**Q5. Give real uses for `Partial`, `Pick`, `Omit` and `Record`.**
`Partial<Product>` for a PATCH body or edit form, `Pick<User, 'id' | 'name'>` for a list item or public DTO, `Omit<User, 'passwordHash'>` for API responses or create inputs without generated fields, and `Record<OrderStatus, string>` for a label map that must cover every status.
*Follow-up:* Why is `Record<Status, string>` better than `{ [key: string]: string }` there? It forces every status key to exist and rejects unknown keys.

**Q6. Implement `Partial`, `Pick` and `Omit`.**
`type Partial<T> = { [P in keyof T]?: T[P] }`; `type Pick<T, K extends keyof T> = { [P in K]: T[P] }`; `type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>`. They are mapped types iterating over keys, with `?` added for `Partial`.
*Follow-up:* How do you make every property required? `-?` in the mapped type: `{ [P in keyof T]-?: T[P] }`.

**Q7. What is the difference between `Exclude` and `Omit`?**
`Exclude<Union, Members>` removes members from a union type (`Exclude<'a' | 'b', 'a'>` is `'b'`); `Omit<Object, Keys>` removes properties from an object type. `Omit` is built from `Pick` and `Exclude` over `keyof T`.
*Follow-up:* What is the counterpart of `Exclude`? `Extract`, which keeps only the matching members.

**Q8. How are `ReturnType` and `Parameters` implemented?**
With conditional types and `infer`: `T extends (...args: any) => infer R ? R : any` captures the return type; `T extends (...args: infer P) => any ? P : never` captures the parameter tuple.
*Follow-up:* Where do you use them? Deriving types from functions you do not own, or keeping wrapper signatures in sync with the wrapped function.

**Q9. What does `Awaited` do? How do you get the resolved type of an async function?**
`Awaited<T>` recursively unwraps promise-like types to their resolved value type. For an async function, use `Awaited<ReturnType<typeof fn>>`, because `ReturnType` alone gives `Promise<...>`.
*Follow-up:* Why recursive? A promise can resolve to another promise, and `await` unwraps all levels.

**Q10. What are default type parameters useful for?**
They let callers omit type arguments that usually have a sensible value, such as `ApiResponse<TData = unknown, TError = ApiError>`. Libraries use them heavily so simple calls stay short while advanced callers can override them.
*Follow-up:* Can a default refer to an earlier type parameter? Yes, for example `<T, U = T[]>`.

**Q11. How do you write a generic React component, and what is special about generic arrows in TSX?**
Declare the props type with a parameter (`ListProps<T>`) and the component as `function List<T>(props: ListProps<T>)`; `T` is inferred from the props at each use. In `.tsx`, an arrow must be written `<T,>(props: ListProps<T>) => ...` (or `<T extends unknown>`) so `<T>` is not parsed as a JSX tag.
*Follow-up:* What is inferred in `<List items={bookings} ... />`? `T = Booking`, so callbacks receive typed items.

**Q12. Why is `return (await res.json()) as T` unsafe in a fetch helper? What is better?**
The cast promises the compiler a shape that nobody checked; if the API changes or returns an error body, the code fails somewhere far away. Better: accept a schema (Zod) and return `schema.parse(json)`, so the data is validated and the type is derived from the schema.
*Follow-up:* When is the cast acceptable? Internal endpoints with shared, end-to-end inferred types (oRPC or tRPC), where the contract is enforced elsewhere.

**Q13. `Record<string, T>` vs an index signature vs a `Map`?**
`Record<string, T>` and `{ [key: string]: T }` are the same type: a plain object whose reads claim to return `T` for any key (unsafe unless `noUncheckedIndexedAccess` is on). A `Map<string, T>` is a runtime structure whose `get` returns `T | undefined` and supports any key type.
*Follow-up:* Which serializes to JSON directly? The plain object.

**Q14. What is the difference between `Required` and `NonNullable`?**
`Required<T>` removes the optional modifier from every property of an object type; `NonNullable<T>` removes `null` and `undefined` from a type (often a union). They solve different problems and are often combined.
*Follow-up:* How is `NonNullable` defined today? `T & {}`.

**Q15. When should you not use generics?**
When a type parameter appears only once (it relates nothing, so use the concrete type or `unknown`), when every caller uses the same type, or when the generic signature becomes harder to understand than two simple functions.
*Follow-up:* What is the "use type parameters twice" rule? A type parameter should link at least two positions, such as a parameter and the return type.

**Q16. What are `InstanceType` and `ConstructorParameters` for?**
They extract the instance type and the constructor argument tuple from a class constructor type, such as `InstanceType<typeof ApiClient>`. They are handy in factories and dependency injection helpers that receive classes as values.
*Follow-up:* Why `typeof ApiClient` and not `ApiClient`? `ApiClient` as a type already means the instance; `typeof ApiClient` is the constructor.

**Q17. What do `const` type parameters and `NoInfer` do?**
A `const` type parameter (5.0) infers the narrowest literal type, as if the caller wrote `as const`. `NoInfer<T>` (5.4) marks a parameter so it is checked against `T` but not used to infer it, preventing accidental widening from a default or fallback argument.
*Follow-up:* Give a `NoInfer` example. `createStore(initial: T, fallback: NoInfer<T>)`: the fallback must match the state type inferred from `initial`.

## Further reading
- [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TypeScript: Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
