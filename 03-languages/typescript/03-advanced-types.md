# Advanced Types

> **Tier:** T1 (know cold) · **Related:** [TypeScript fundamentals](01-fundamentals.md), [Generics and utility types](02-generics-utility-types.md), [TypeScript in practice](04-ts-in-practice.md), [Multi-tenancy](../../13-architecture-and-practices/03-multi-tenancy.md), [Hono](../../09-nodejs-backend/04-hono.md), [Multi-tenant platform migration](../../17-projects-deep-dive/04-multi-tenant-platform-migration.md)
> **On your resume:** discriminated unions for screen and request states in React and React Native (Expo) code, typed request context and org-scoped access control on the multi-tenant TypeScript platform (oRPC, TanStack Start, Drizzle) at Devntech.

## Quick revision
- A type guard `function isX(v): v is X` narrows in the caller's `if`. An assertion function `asserts v is X` (or `asserts condition`) narrows everything after the call and throws otherwise.
- Guards are trusted, not verified: a wrong guard is as dangerous as a wrong `as`.
- Discriminated union = union of object types sharing a literal property (`status`, `type`, `kind`). `switch` on it narrows each branch; `const x: never = value` in `default` makes missing cases a compile error.
- Model states as unions (`idle | loading | success | error`) instead of loose booleans and optional fields: illegal states become unrepresentable.
- Conditional types: `T extends U ? X : Y`. They distribute over unions when `T` is a naked type parameter; wrap in a tuple (`[T] extends [U]`) to stop that.
- `infer` captures part of a type inside a conditional type: element types, promise results, function arguments.
- Mapped types: `{ [K in keyof T]: ... }`, with `readonly`/`?` modifiers (`-readonly`, `-?` remove them) and key remapping with `as` (rename keys or filter them out with `never`).
- Template literal types build string unions: `` `${Entity}.${Action}` ``, `` `on${Capitalize<K>}` ``; with `infer` they can parse strings such as route params.
- Recursive types: `JsonValue`, `DeepPartial<T>`, tree nodes.
- `: Type` checks and widens; `satisfies Type` checks and keeps the precise inferred type; `as const` narrows to readonly literals without checking; `as const satisfies Type` does both.
- Interfaces merge; `declare global` and `declare module 'pkg'` augment existing types (add `user` to a request, variables to `window`, env vars to `import.meta.env`).
- Branded types (`string & { [brand]: 'UserId' }`) give nominal safety: a `UserId` cannot be passed where an `OrgId` is expected.
- Index signatures claim every key exists. `noUncheckedIndexedAccess` adds `| undefined` to index reads (objects and arrays).
- Prefer overloads for a few fixed call shapes; use conditional types for systematic type-level mappings (they need casts inside the implementation).

## 1. User-defined type guards and assertion functions

```ts
type Admin = { role: 'admin'; permissions: string[] };
type Guest = { role: 'guest' };
type Person = Admin | Guest;

function isAdmin(person: Person): person is Admin {
  return person.role === 'admin';
}

function showPermissions(person: Person) {
  if (isAdmin(person)) return person.permissions.join(', '); // narrowed to Admin
  return 'none';
}

// A guard for unknown data: a small hand-written validator
type Product = { id: string; priceCents: number };

function isProduct(value: unknown): value is Product {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'string' &&
    'priceCents' in value &&
    typeof value.priceCents === 'number'
  );
}

// Guards work as filter callbacks
const maybeProducts: (Product | null)[] = [{ id: 'p1', priceCents: 100 }, null];
const products = maybeProducts.filter((p): p is Product => p !== null); // Product[]
```

TypeScript 5.5 infers simple predicates like `(p) => p !== null` on its own, so `filter` narrows without the explicit `p is Product` in recent versions.

**Assertion functions** narrow everything after the call and throw when the condition fails:

```ts
function assert(condition: unknown, message: string): asserts condition {
  if (!condition) throw new Error(message);
}

function assertIsDefined<T>(value: T, name: string): asserts value is NonNullable<T> {
  if (value === null || value === undefined) throw new Error(`${name} is required`);
}

function connect(apiUrl: string | undefined) {
  assertIsDefined(apiUrl, 'API_URL');
  return new URL(apiUrl); // apiUrl is string from here on
}
```

| | Type guard (`v is T`) | Assertion function (`asserts v is T`) |
|---|---|---|
| Returns | boolean | nothing (throws on failure) |
| Narrowing | inside the `if` or ternary that uses it | for the rest of the scope after the call |
| Use | branching logic, `filter` | preconditions, invariants, tests |

Both are trusted blindly: if `isProduct` forgot to check `priceCents`, the compiler would still believe it. For anything bigger than a few fields, use a schema library (Zod) and let it produce the type.

## 2. Discriminated unions and exhaustive checks

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function renderSearch(state: RequestState<string[]>): string {
  switch (state.status) {
    case 'idle':
      return 'Start typing to search';
    case 'loading':
      return 'Loading...';
    case 'success':
      return state.data.join(', ');      // data exists only in this branch
    case 'error':
      return `Failed: ${state.error}`;
    default: {
      const unreachable: never = state;  // compile error if a status is not handled
      return unreachable;
    }
  }
}
```

A reusable helper does the same job and also throws if bad data sneaks in at runtime:

```ts
function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
}

type PaymentEvent =
  | { type: 'payment.succeeded'; paymentId: string; amountCents: number }
  | { type: 'payment.failed'; reason: string }
  | { type: 'refund.created'; refundId: string; amountCents: number };

function describeEvent(event: PaymentEvent): string {
  switch (event.type) {
    case 'payment.succeeded':
      return `Paid ${event.amountCents} cents (${event.paymentId})`;
    case 'payment.failed':
      return `Payment failed: ${event.reason}`;
    case 'refund.created':
      return `Refund ${event.refundId}`;
    default:
      return assertNever(event); // add a new event type and this line stops compiling
  }
}
```

**Make illegal states unrepresentable.** Compare with the loose version:

```ts
type LooseState = { loading: boolean; data?: string[]; error?: string };
// Allows nonsense: { loading: true, data: [...], error: 'x' } all at once,
// and every consumer must guess which combination is valid.
```

The union version encodes the rules in the type, so the compiler enforces them everywhere. The same pattern models order statuses, payment flows, websocket messages, reducer actions and API results (`{ ok: true; data } | { ok: false; error }`).

## 3. Conditional types, distribution and infer

```ts
type IsString<T> = T extends string ? 'yes' : 'no';
type A = IsString<'hello'>;          // 'yes'
type B = IsString<number>;           // 'no'
type C = IsString<string | number>;  // 'yes' | 'no': distributes over each member

type IsStringStrict<T> = [T] extends [string] ? 'yes' : 'no';
type D = IsStringStrict<string | number>; // 'no': the tuple wrapper prevents distribution

// infer: capture part of a type
type ElementType<T> = T extends readonly (infer U)[] ? U : never;
type E = ElementType<string[]>;                         // string
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type F = UnwrapPromise<Promise<number>>;                // number
type FirstArg<T> = T extends (first: infer A, ...rest: any[]) => any ? A : never;
type G = FirstArg<(id: string, force: boolean) => void>; // string
```

- **Distribution** happens only when the checked type is a naked type parameter (`T extends ...`). That is how `Exclude<'a' | 'b', 'a'>` removes members one by one.
- **`never` distributes to `never`**: `IsString<never>` is `never`, not `'no'`, because there are no members to map. Use the tuple wrapper if that surprises you.
- **`infer`** can only appear in the `extends` clause of a conditional type; it is how `ReturnType`, `Parameters` and `Awaited` are written.

## 4. Mapped types, modifiers and key remapping

```ts
type Mutable<T> = { -readonly [K in keyof T]: T[K] };    // strip readonly
type AllOptional<T> = { [K in keyof T]+?: T[K] };       // + is the default, like Partial

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]; // rename keys
};
type Profile = { name: string; age: number };
type ProfileGetters = Getters<Profile>; // { getName: () => string; getAge: () => number }

type StringKeysOnly<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]; // filter keys out with never
};
type OnlyStrings = StringKeysOnly<{ id: string; count: number; title: string }>;
// { id: string; title: string }

type FormErrors<T> = { [K in keyof T]?: string };
type SignupErrors = FormErrors<{ email: string; password: string }>;
// { email?: string; password?: string }
```

Mapped types iterate over a union of keys (`K in keyof T`). Modifiers add or remove `readonly` and `?`; the `as` clause (TypeScript 4.1) renames keys or drops them by mapping to `never`. `FormErrors<T>` is the kind of type form libraries derive from your form values.

## 5. Template literal types

```ts
type Entity = 'order' | 'payment';
type Action = 'created' | 'updated';
type EventName = `${Entity}.${Action}`;
// 'order.created' | 'order.updated' | 'payment.created' | 'payment.updated'

type CssLength = `${number}px` | `${number}rem`;
const gap: CssLength = '16px';
// const wrong: CssLength = '16pt';  // error

type Handlers<T extends string> = { [K in T as `on${Capitalize<K>}`]: () => void };
type ButtonHandlers = Handlers<'click' | 'focus'>; // { onClick: () => void; onFocus: () => void }
```

### Going deeper: parsing strings at the type level

```ts
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | RouteParams<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
      ? Param
      : never;

type MemberParams = RouteParams<'/orgs/:orgId/members/:memberId'>; // 'orgId' | 'memberId'
```

Typed routers use this idea so that a path string determines which params a link or loader must provide. The intrinsic helpers `Uppercase`, `Lowercase`, `Capitalize` and `Uncapitalize` transform string literal types.

## 6. Recursive types

```ts
type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue };

const payload: JsonValue = { user: { id: 1, tags: ['a'], active: true, manager: null } };
// const notJson: JsonValue = { createdAt: new Date() }; // error: a Date is not a JSON value

type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

type AppConfig = { db: { host: string; port: number }; cache: { ttlSeconds: number } };
const overrides: DeepPartial<AppConfig> = { db: { port: 5433 } }; // nested keys optional

type CategoryNode = { id: string; name: string; children: CategoryNode[] }; // trees
```

`DeepPartial` here also maps over functions, Dates and Maps (they are objects); production versions special-case them. Very deep or very wide recursive types slow the compiler down, and TypeScript stops with "Type instantiation is excessively deep" when recursion gets out of hand.

## 7. satisfies vs annotation vs as const

```ts
type Route = { path: string; auth: boolean };

// 1. Annotation: checked, but the exact keys are forgotten
const routesA: Record<string, Route> = {
  home: { path: '/', auth: false },
  billing: { path: '/billing', auth: true },
};
routesA.typo;            // no error: the type says any string key exists

// 2. satisfies (TypeScript 4.9): checked, and the inferred type is kept
const routesB = {
  home: { path: '/', auth: false },
  billing: { path: '/billing', auth: true },
} satisfies Record<string, Route>;
// routesB.typo;         // error: Property 'typo' does not exist
routesB.billing.path;    // string
routesB.billing.auth;    // true: literals survive because the contextual type (boolean) is a literal union

// 3. as const: readonly literal types, but nothing is checked
const routesC = { home: { path: '/', auth: false } } as const; // routesC.home.path is '/'

// 4. Both: literal types and a check
const routesD = {
  home: { path: '/', auth: false },
  billing: { path: '/billing', auth: true },
} as const satisfies Record<string, Route>;
type RouteName = keyof typeof routesD; // 'home' | 'billing'
```

| Form | Checked against the type? | Resulting type | Use |
|---|---|---|---|
| `const x: T = ...` | yes | `T` (specific info lost) | function boundaries, public APIs |
| `const x = ... satisfies T` | yes, including excess properties | the precise inferred type | config objects, route maps, theme tokens, lookup tables |
| `const x = ... as const` | no | readonly literal types | constant lists, tuples, enum-like objects |
| `const x = ... as T` | no (assertion) | `T` | last resort |
| `as const satisfies T` | yes | readonly literal types | constant configs that must also match a contract |

## 8. Declaration merging and module augmentation

Interfaces with the same name merge, and you can reopen types that live in another module or in the global scope. This is how you extend a framework's request or context type.

```ts
// src/types/express.d.ts: add `user` to every Express request
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; orgId: string; role: 'owner' | 'member' };
    }
  }
}

export {}; // makes this file a module, which `declare global` requires
```

```ts
// vite-env.d.ts: type your environment variables in a Vite app
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

Some frameworks prefer generics over augmentation. In Hono, the context variables are a type parameter of the app:

```ts
import { Hono } from 'hono';

type AppEnv = { Variables: { user: { id: string; orgId: string } } };
const app = new Hono<AppEnv>();

app.use('*', async (c, next) => {
  c.set('user', { id: 'u1', orgId: 'o1' }); // a wrong shape does not compile
  await next();
});

app.get('/me', (c) => c.json(c.get('user'))); // typed as { id: string; orgId: string }
```

oRPC follows the same idea with a typed initial context declared on its `os` builder (for example `os.$context<{ user?: User }>()`), and TanStack Router has `createRootRouteWithContext` for router context; check their docs for the exact current API. See [Hono](../../09-nodejs-backend/04-hono.md) and [oRPC and type-safe APIs](../../09-nodejs-backend/05-orpc-and-typesafe-apis.md).

| Technique | What it does | Example |
|---|---|---|
| Declaration merging | same-name interfaces in one scope combine | two `interface Settings` blocks |
| Global augmentation | `declare global { ... }` inside a module adds to global types | `Window`, the `Express` namespace |
| Module augmentation | `declare module 'pkg' { ... }` adds to a package's exported interfaces | a UI library's theme interface |

## 9. Branded types

Structural typing means every `string` is interchangeable, so a user ID can be passed where an org ID is expected and the compiler stays silent. In multi-tenant code that is exactly the bug that leaks data across organizations. A brand adds a phantom property that exists only in the type system:

```ts
declare const brand: unique symbol;
type Brand<T, Name extends string> = T & { readonly [brand]: Name };

type UserId = Brand<string, 'UserId'>;
type OrgId = Brand<string, 'OrgId'>;

const toUserId = (id: string) => id as UserId; // the only places a cast is allowed
const toOrgId = (id: string) => id as OrgId;

function getMembership(userId: UserId, orgId: OrgId) {
  return { userId, orgId };
}

const userId = toUserId('u_123');
const orgId = toOrgId('org_456');
getMembership(userId, orgId);       // ok
// getMembership(orgId, userId);    // error: 'OrgId' is not assignable to 'UserId'
// getMembership('u_123', orgId);   // error: a plain string is not a UserId
```

- Create branded values only at trusted boundaries: after validation, from the session, or from the database layer. Zod has `.brand<'UserId'>()` so a parsed value comes out already branded, and Drizzle columns can be typed with `.$type<OrgId>()`.
- Brands are erased at runtime; they cost nothing and only constrain the code you write.
- They pair naturally with org-scoped access control: functions that touch tenant data can demand an `OrgId` taken from the authenticated context rather than any string from a request. See [multi-tenancy](../../13-architecture-and-practices/03-multi-tenancy.md).

## 10. Index signatures and noUncheckedIndexedAccess

```ts
type PriceBook = { [sku: string]: number };   // same as Record<string, number>
const prices: PriceBook = { 'sku-1': 1200 };

const missing = prices['sku-404'];            // type: number; runtime value: undefined
// missing.toFixed(2) compiles but throws at runtime

const items = ['a', 'b'];
const third = items[2];                       // type: string; runtime value: undefined
```

With `"noUncheckedIndexedAccess": true`, both reads become `number | undefined` and `string | undefined`, so the compiler makes you check before use. It is noisy at first (every `arr[i]` needs handling) but catches real bugs in lookups and parsing code; `for...of`, `map` and destructuring with defaults keep the noise down.

Related facts:
- Template literal index signatures (TypeScript 4.4): `{ [key: `data-${string}`]: string }`.
- `Object.keys(obj)` returns `string[]`, not `(keyof T)[]`, because structural typing allows objects to carry extra keys at runtime. Cast deliberately (`Object.keys(obj) as (keyof T)[]`) only when you control the object.
- Use `Map` when keys are dynamic and `get` returning `T | undefined` is what you want.

## 11. Overloads vs conditional types

```ts
// Overloads: explicit signatures for a few input shapes; no casts needed
function parse(input: string): number;
function parse(input: string[]): number[];
function parse(input: string | string[]): number | number[] {
  return Array.isArray(input) ? input.map(Number) : Number(input);
}
const one = parse('1');          // number
const many = parse(['1', '2']);  // number[]

// Conditional return type: one generic signature, but the body needs an assertion
type Parsed<T> = T extends string[] ? number[] : number;

function parseGeneric<T extends string | string[]>(input: T): Parsed<T> {
  return (Array.isArray(input) ? input.map(Number) : Number(input)) as Parsed<T>;
}
const manyAgain = parseGeneric(['1', '2']); // number[]
```

Prefer overloads for two or three fixed call shapes: they read clearly and give targeted errors. Use conditional types when the output type is a systematic function of the input type (library-style utilities), accepting that TypeScript cannot check the implementation against them, hence the cast.

## Common pitfalls
- Writing a type guard that does not check what it claims; the compiler trusts it completely.
- Forgetting the `never` check, so a new union member silently falls through.
- Using booleans plus optional fields for state instead of a discriminated union.
- Unexpected distribution in conditional types (a union input producing a union output), or surprise `never` results.
- `satisfies` confused with `as`: `satisfies` checks, `as` does not.
- Augmenting a global type in a file that is not a module (missing `export {}`), or a `.d.ts` file not included by `tsconfig`.
- Creating branded values with casts all over the codebase instead of at one boundary.
- Reading from index signatures as if every key existed; trusting `arr[i]` without bounds checks.
- Type-level cleverness in application code that teammates cannot maintain.

## How to talk about this in an interview
- Discriminated unions are the most practical advanced feature: describe request states in a React or React Native screen, payment or bill statuses in the Coloc resident app, or API result types on the TypeScript platform.
  > **Fill in:** a real union you modeled (screen state, event type, order or payment status) and the bug the exhaustive check prevented.
- For the multi-tenant platform migration, connect branded IDs and typed context to org-scoped access control: the type system can make "forgot to scope by org" harder to write.
  > **Fill in:** how org scoping was enforced on the platform (middleware putting the org in context, query helpers that require an org ID, Postgres policies) and what your part was.
- When asked about `satisfies`, give the one-line contrast with an annotation: same check, but the precise inferred type survives.

## Interview questions

**Q1. What is a user-defined type guard?**
A function returning a type predicate, `function isAdmin(p: Person): p is Admin`, whose boolean result narrows the argument in the caller's branch. It is useful for reusable checks and as a `filter` callback. The compiler trusts the predicate without verifying the body.
*Follow-up:* What changed in TypeScript 5.5? Simple predicates such as `(x) => x !== null` are inferred automatically.

**Q2. How do assertion functions differ from type guards?**
An assertion function (`asserts value is T` or `asserts condition`) returns nothing and throws when the check fails; after the call, the value stays narrowed for the rest of the scope. Guards return a boolean and narrow only inside the branch that tests them.
*Follow-up:* Any declaration restriction? Every name in the call target needs an explicit type, so `const assertX = (v: unknown): asserts v is X => {...}` only works if the `assertX` variable itself is annotated; a function declaration avoids the issue.

**Q3. What is a discriminated union, and how do you make a `switch` exhaustive?**
A union of object types that share a literal property such as `status` or `type`; checking that property narrows to one member. In the `default` branch, assign the value to `never` (or call `assertNever(value)`): if a member is unhandled, the value is not `never` and compilation fails.
*Follow-up:* Why also throw at runtime? Data from outside the type system (an older API, bad JSON) can still contain unexpected values.

**Q4. How do you make illegal states unrepresentable?**
Model each valid state as its own union member with only the fields that make sense there (`{ status: 'success'; data }`, `{ status: 'error'; error }`) instead of booleans and optional fields that can combine into nonsense. Consumers must narrow before reading `data` or `error`.
*Follow-up:* Where else does this help? Reducer actions, payment flows, websocket message types.

**Q5. What are conditional types, and when do they distribute?**
Type-level ternaries, `T extends U ? X : Y`. When `T` is a naked type parameter and receives a union, the condition is applied to each member and the results are unioned. Wrapping both sides in a tuple, `[T] extends [U]`, disables distribution.
*Follow-up:* What is `IsString<never>` for a distributive `IsString`? `never`, because there are no members to map.

**Q6. What does `infer` do? Give an example.**
It declares a type variable inside the `extends` clause of a conditional type and captures whatever matches that position: `T extends Promise<infer U> ? U : T` unwraps a promise; `T extends (...args: infer P) => any ? P : never` extracts parameters.
*Follow-up:* Which built-ins use it? `ReturnType`, `Parameters`, `InstanceType`, `Awaited`.

**Q7. Explain mapped type modifiers and key remapping.**
`{ [K in keyof T]: T[K] }` rebuilds a type key by key; adding `readonly` or `?` (or removing them with `-readonly` and `-?`) changes modifiers, as in `Readonly`, `Partial` and `Required`. An `as` clause renames keys (`` `get${Capitalize<K>}` ``) or removes them by mapping to `never`.
*Follow-up:* How would you keep only function-valued keys? `[K in keyof T as T[K] extends Function ? K : never]: T[K]`.

**Q8. Where are template literal types useful?**
Building string unions from parts (event names like `'order.created'`, CSS units, `on${Capitalize<Event>}` handler props) and parsing strings at the type level with `infer`, such as extracting `:params` from a route path.
*Follow-up:* What are the intrinsic string types? `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.

**Q9. What does `satisfies` do, and how is it different from an annotation and `as const`?**
`satisfies T` checks that a value matches `T` (including excess properties) but keeps the value's own inferred type, so exact keys and literal details survive. An annotation checks and then widens to `T`; `as const` narrows to readonly literals without checking; `as const satisfies T` gives both.
*Follow-up:* Typical use? Config and route maps where you want typo checks and autocomplete on the actual keys.

**Q10. How do you add a `user` property to Express's `Request` type?**
Augment the global `Express.Request` interface in a `.d.ts` or module file: `declare global { namespace Express { interface Request { user?: AuthUser } } }`, with `export {}` if the file has no other imports or exports, and make sure `tsconfig` includes it. In Hono, pass `Variables` as a generic to `new Hono<...>()` instead.
*Follow-up:* Why is `user` optional? Only authenticated routes have it; middleware that guarantees it can narrow or assert.

**Q11. What is the difference between declaration merging, global augmentation and module augmentation?**
Declaration merging combines same-name interfaces (or namespaces) in one scope. Global augmentation (`declare global`) adds members to global types from inside a module. Module augmentation (`declare module 'pkg'`) adds members to interfaces exported by a package. All rely on interfaces being open.
*Follow-up:* Can you augment a type alias? No, only interfaces (and namespaces) merge.

**Q12. What are branded types, and why do they matter in multi-tenant code?**
A brand intersects a primitive with a phantom property (`string & { [brand]: 'OrgId' }`) so values are no longer interchangeable with plain strings or other brands. Passing a `UserId` where an `OrgId` is required, or an unscoped string from a request, becomes a compile error, which helps prevent cross-tenant mistakes.
*Follow-up:* Where do branded values come from? Validated boundaries only: a Zod schema with `.brand()`, the session, or the database layer.

**Q13. What does `noUncheckedIndexedAccess` change?**
Reads through index signatures and array indexes get `| undefined` added (`prices[sku]` is `number | undefined`, `items[0]` is `string | undefined`), forcing a check before use. Property access on declared keys is unaffected.
*Follow-up:* How do you keep the noise down? Use `for...of`, array methods, destructuring defaults and `Map`.

**Q14. Why does `Object.keys(obj)` return `string[]` instead of `(keyof T)[]`?**
Because of structural typing: a value typed as `T` may carry extra properties at runtime, so claiming the keys are exactly `keyof T` would be unsound. Cast only when you know the object has no extra keys.
*Follow-up:* Safer alternative for iteration? Iterate a known `as const` key list, or use `Object.entries` with a type guard.

**Q15. How do you type JSON, and what is the catch with `DeepPartial`?**
`type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue }` (a recursive type) rejects Dates, functions and `undefined`. A naive `DeepPartial` recurses into everything that `extends object`, including functions, Dates and Maps, so real implementations special-case them.
*Follow-up:* Compiler limit? Deep recursion can hit "Type instantiation is excessively deep and possibly infinite".

**Q16. Overloads vs conditional types: how do you choose?**
Overloads suit a few fixed input shapes, read clearly and need no casts. Conditional return types suit systematic type-level mappings in generic utilities, but the implementation must assert its return value because TypeScript cannot verify it against the conditional type.
*Follow-up:* Is the overload implementation signature visible to callers? No, only the overload signatures are.

**Q17. Why might `type R = IsString<string | number>` be `'yes' | 'no'` when you expected `'no'`?**
The conditional type distributes over the union, evaluating `string` and `number` separately. If you want to test the union as a whole, write `[T] extends [string] ? 'yes' : 'no'`.
*Follow-up:* Which built-ins depend on distribution? `Exclude`, `Extract` and `NonNullable`'s older definition.

## Further reading
- [TypeScript Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript Handbook: Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript Handbook: Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
