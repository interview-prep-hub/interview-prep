# TypeScript in Practice: tsconfig, React, Node and Zod

> **Tier:** T1 (know cold) · **Related:** [TypeScript fundamentals](01-fundamentals.md), [Generics and utility types](02-generics-utility-types.md), [Advanced types](03-advanced-types.md), [React hooks](../../06-react/02-hooks.md), [Forms and validation](../../06-react/05-forms-and-validation.md), [oRPC and type-safe APIs](../../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [TanStack Start](../../07-nextjs-and-tanstack/05-tanstack-start.md), [Drizzle ORM](../../12-databases/13-drizzle-orm.md)
> **On your resume:** the tsconfig, React and React Native (Expo) typing, validation and end-to-end types of the oRPC + TanStack Start + Drizzle platform at Devntech, plus the Next.js (TypeScript) API routes you built at RemoteDev.

## Quick revision
- `strict: true` for all new code; add `noUncheckedIndexedAccess` when you can.
- `target` = syntax level of the emitted JS; `lib` = which built-in APIs exist (ES2022, DOM); `module` = output module format; `moduleResolution`: `bundler` for apps built by a bundler, `nodenext` for code Node runs directly and for published packages.
- Under `nodenext`, relative imports in ES modules need the runtime extension (`./app.js`), even in `.ts` files.
- `paths` aliases (`@/*`) only teach the type checker; the bundler or runtime must resolve them too.
- `isolatedModules`: every file must be transpilable on its own. `verbatimModuleSyntax` (5.0): imports are emitted exactly as written, so type-only imports must say `import type`.
- `skipLibCheck` skips checking `.d.ts` files; `esModuleInterop` makes `import express from 'express'` work for CommonJS packages; `noEmit` when a bundler does the build.
- `.d.ts` files describe shapes without code; `@types/*` packages come from DefinitelyTyped; shim untyped modules with `declare module 'x'`.
- React: props as a `type`, `children: ReactNode`, `ChangeEvent<HTMLInputElement>`, `useState<User | null>(null)`, `useRef<HTMLInputElement>(null)`, reducer actions as a discriminated union, context created with `| null` plus a hook that throws, `ComponentProps<'button'>` to extend native props.
- Server: validate env vars at startup into a typed `env`; request bodies are `unknown` until validated.
- Zod: one schema = runtime validator + static type (`z.infer`). `parse` throws a `ZodError`; `safeParse` returns `{ success: true, data } | { success: false, error }`.
- Validate at every trust boundary: request bodies, query params, env vars, webhooks, third-party responses, local storage, forms.
- End-to-end types: oRPC and tRPC infer the client from the server router type, with no codegen; Drizzle tables give `$inferSelect` and `$inferInsert`.
- `import type` keeps server code out of client bundles.
- typescript-eslint adds type-aware rules: `no-floating-promises`, `no-misused-promises`, `no-explicit-any`, `consistent-type-imports`.
- Migrate gradually: `allowJs`, JSDoc with `// @ts-check`, convert leaf modules first, tighten flags one at a time, prefer `@ts-expect-error` to `@ts-ignore`.
- Read long errors from the bottom: the last "is not assignable" line is usually the real mismatch.

## 1. tsconfig essentials

A typical config for a bundled React app (Vite style):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "noEmit": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

A Node API compiled by `tsc` and run with `node dist/server.js`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

| Option | What it does | Notes |
|---|---|---|
| `strict` | enables the strict family (`strictNullChecks`, `noImplicitAny`, ...) | always on for new code |
| `noUncheckedIndexedAccess` | adds `undefined` to index reads | not part of `strict`; catches real bugs |
| `exactOptionalPropertyTypes` | `a?: string` no longer accepts an explicit `undefined` | stricter, sometimes noisy |
| `target` | JS syntax level of the output | lower targets downlevel classes, async and so on |
| `lib` | built-in API types available | include `DOM` for browsers, leave it out for pure Node |
| `module` | module format of the output | `ESNext` with bundlers, `NodeNext` for Node |
| `moduleResolution` | how imports are located | `bundler` or `nodenext` (older `node`/`node10` is legacy) |
| `paths` | import aliases | the bundler (Vite alias, Next.js, Metro) must agree |
| `isolatedModules` | reports code a single-file transpiler cannot handle | required by Vite, esbuild, SWC and Babel setups |
| `verbatimModuleSyntax` | keeps imports as written; forces `import type` for types | replaces older `importsNotUsedAsValues` options |
| `esModuleInterop` | interop helpers for default-importing CommonJS | implies `allowSyntheticDefaultImports` |
| `skipLibCheck` | skips type-checking `.d.ts` files | faster builds, fewer third-party type conflicts |
| `noEmit` | type-check only | when a bundler or framework emits the JS |
| `jsx` | how JSX is compiled | `react-jsx` (automatic runtime); `preserve` when a framework handles it, as in Next.js |
| `allowJs` / `checkJs` | include and type-check `.js` files | gradual migration |
| `resolveJsonModule` | import `.json` with types | config files, fixtures |

### moduleResolution: bundler vs nodenext

| | `bundler` | `nodenext` |
|---|---|---|
| For | apps compiled by a bundler (Vite, Next.js, webpack, Metro) | code run directly by Node, and published packages |
| Relative imports | extensionless is fine (`./utils`) | ES module files need the runtime extension (`./utils.js`) |
| package.json `exports` | respected | respected, including `import`/`require` conditions |
| Module format | decided by the bundler | decided by package.json `"type"` and `.mts`/`.cts` extensions |

```ts
// src/server.ts in a "type": "module" package with moduleResolution nodenext
import { createApp } from './app.js'; // .js on purpose: it names the emitted file
```

## 2. Declaration files and @types

A `.d.ts` file contains only types: it describes the shape of JavaScript that exists elsewhere. Libraries either ship their own (`"types"` in package.json) or get them from DefinitelyTyped as `@types/<name>` dev dependencies, which TypeScript picks up automatically.

```ts
// src/types/shims.d.ts
declare module 'legacy-analytics' {
  export function track(event: string, props?: Record<string, unknown>): void;
}

declare module '*.svg' {
  const url: string;
  export default url;
}
```

- "Could not find a declaration file for module 'x'" (TS7016): install `@types/x` if it exists, otherwise write a shim like the one above. The one-line shim `declare module 'x';` types everything as `any`, a temporary fix at best.
- Library authors generate `.d.ts` files with `declaration: true`.
- Global augmentation (adding to `Window`, typing `import.meta.env`) is covered in [advanced types](03-advanced-types.md).

## 3. TypeScript with React

```tsx
import { createContext, useContext, useRef, useState } from 'react';
import type { ChangeEvent, ComponentProps, FormEvent, ReactNode } from 'react';

// Props: a type, defaults for optional props, children as ReactNode
type CardProps = {
  title: string;
  variant?: 'plain' | 'highlighted';
  children: ReactNode;
};

export function Card({ title, variant = 'plain', children }: CardProps) {
  return (
    <section className={variant}>
      <h2>{title}</h2>
      {children}
    </section>
  );
}

// Everything a native <button> accepts, plus one extra prop
type ButtonProps = ComponentProps<'button'> & { loading?: boolean };

export function Button({ loading = false, disabled, children, ...rest }: ButtonProps) {
  return (
    <button {...rest} disabled={disabled || loading}>
      {loading ? 'Saving...' : children}
    </button>
  );
}

type User = { id: string; name: string };

export function SearchForm({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState('');               // inferred: string
  const [selected, setSelected] = useState<User | null>(null); // starts as null: annotate
  const inputRef = useRef<HTMLInputElement>(null);      // DOM ref: null until mounted
  const timerRef = useRef<number | null>(null);         // mutable value (a timer id)

  const handleChange = (event: ChangeEvent<HTMLInputElement>) => setQuery(event.target.value);

  const handleSubmit = (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    onSearch(query);
    inputRef.current?.focus();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} value={query} onChange={handleChange} />
      <button type="submit">Search</button>
    </form>
  );
}

// Typed context with a hook that guarantees a value
type AuthContextValue = { user: User | null; signOut: () => Promise<void> };
const AuthContext = createContext<AuthContextValue | null>(null);

export function useAuth(): AuthContextValue {
  const value = useContext(AuthContext);
  if (!value) throw new Error('useAuth must be used inside <AuthProvider>');
  return value; // callers never deal with null
}
```

- `ReactNode` covers anything renderable (elements, strings, numbers, arrays, `null`), so it is the right type for `children`. `ReactElement` (or `JSX.Element`) means exactly one element.
- Inline handlers (`onChange={(e) => ...}`) infer the event type; annotate only when the handler is defined separately.
- `useState('')` infers `string`; annotate when the initial value does not show the full type (`null`, `[]`).
- React 19's types make `useRef` require an argument; `useRef<HTMLInputElement>(null)` works across versions.
- `React.FC` is optional; plain functions with typed props are the common style today.

Reducers pair naturally with discriminated unions:

```tsx
type CartItem = { id: string; priceCents: number; qty: number };
type CartAction =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; id: string }
  | { type: 'clear' };

function cartReducer(state: CartItem[], action: CartAction): CartItem[] {
  switch (action.type) {
    case 'add':
      return [...state, action.item];
    case 'remove':
      return state.filter((item) => item.id !== action.id);
    case 'clear':
      return [];
  }
}
// Inside a component: const [items, dispatch] = useReducer(cartReducer, []);
// dispatch({ type: 'remove' }) does not compile: 'id' is required for 'remove'.
```

React Native uses the same patterns with its own prop types (`TextInputProps`, `PressableProps`, `ViewStyle` from `react-native`), and `StyleSheet.create` infers style types. Generic components (lists, selects) are covered in [generics](02-generics-utility-types.md); hooks typing in depth in [React hooks](../../06-react/02-hooks.md).

## 4. TypeScript on the server

**Typed, validated environment variables:** parse once at startup, export a typed object, and never read `process.env` elsewhere.

```ts
// src/env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
});

export const env = EnvSchema.parse(process.env); // fails at boot, not at the first request
// env.PORT is a number; env.DATABASE_URL is a string. No `process.env.X!` scattered around.
```

**Request input is `unknown` until validated.** A Next.js App Router route handler:

```ts
// app/api/bookings/route.ts
import { z } from 'zod';

const CreateBookingInput = z.object({
  roomId: z.string().min(1),
  startDate: z.coerce.date(),
  nights: z.number().int().min(1).max(30),
});
type CreateBookingInput = z.infer<typeof CreateBookingInput>; // same name for schema and type

declare function createBooking(input: CreateBookingInput): Promise<{ id: string }>;

export async function POST(request: Request) {
  const result = CreateBookingInput.safeParse(await request.json());
  if (!result.success) {
    return Response.json({ issues: result.error.issues }, { status: 422 });
  }
  const booking = await createBooking(result.data); // fully typed, already validated
  return Response.json(booking, { status: 201 });
}
```

- In `catch (err)`, `err` is `unknown` under `strict`: `const message = err instanceof Error ? err.message : String(err);`.
- Framework types help: Express's `Request<Params, ResBody, ReqBody, Query>` generics, Hono's validator middleware (`@hono/zod-validator`) and oRPC's `.input(schema)` all connect a schema to typed handler input. See [Next.js backend APIs](../../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md) and [Hono](../../09-nodejs-backend/04-hono.md).

## 5. Zod at the boundaries

```ts
import { z } from 'zod';

export const SignupSchema = z
  .object({
    email: z.string().email(),
    password: z.string().min(8, 'At least 8 characters'),
    confirmPassword: z.string(),
    role: z.enum(['resident', 'manager']),
    phone: z.string().optional(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  });

export type SignupInput = z.infer<typeof SignupSchema>;
// { email: string; password: string; confirmPassword: string;
//   role: 'resident' | 'manager'; phone?: string | undefined }

// parse: typed data, or a thrown ZodError
const valid = SignupSchema.parse({
  email: 'sara@example.com',
  password: 'correct-horse',
  confirmPassword: 'correct-horse',
  role: 'resident',
});

// safeParse: never throws; returns a discriminated union
const result = SignupSchema.safeParse({ email: 'not-an-email' });
if (!result.success) {
  console.log(result.error.issues.map((issue) => `${issue.path.join('.')}: ${issue.message}`));
} else {
  console.log(result.data.email); // typed
}
```

| | `parse` | `safeParse` |
|---|---|---|
| Success | returns the data | `{ success: true, data }` |
| Failure | throws `ZodError` | `{ success: false, error }` |
| Use when | failure is exceptional (env at boot) or a framework catches it (oRPC input) | you report field errors (forms, 422 responses) |

Async refinements need `parseAsync` / `safeParseAsync`. With transforms, the input and output types differ: `z.input<typeof S>` vs `z.output<typeof S>` (`z.infer` is the output). Recent Zod versions also add top-level format helpers such as `z.email()`; check the docs for the version you use.

**Forms:** the same schema validates on the client through react-hook-form's resolver and on the server in the endpoint, so the rules live in one place.

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

export function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<SignupInput>({ resolver: zodResolver(SignupSchema) });

  return (
    <form onSubmit={handleSubmit((data) => console.log(data.email))}>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}
      <button type="submit">Sign up</button>
    </form>
  );
}
```

More on forms in [forms and validation](../../06-react/05-forms-and-validation.md).

**Why runtime validation is still needed with TypeScript:** types are erased at compile time, so nothing checks data that arrives while the program runs. HTTP bodies, query strings, env vars, webhook payloads, third-party API responses, `localStorage`, deep-link params in a mobile app and JSON columns are all `unknown` in reality. A cast (`as User`) only silences the compiler; a schema checks the data and produces the type from the same definition.

## 6. End-to-end type safety

The goal: one source of truth, with types flowing from the database through the API to the UI, so renaming a column breaks compilation in every place that depends on it.

**Drizzle** infers row types from the table definition:

```ts
import { integer, pgTable, text, timestamp } from 'drizzle-orm/pg-core';

export const listings = pgTable('listings', {
  id: text('id').primaryKey(),
  orgId: text('org_id').notNull(),
  title: text('title').notNull(),
  priceCents: integer('price_cents').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export type Listing = typeof listings.$inferSelect;    // what a select returns
export type NewListing = typeof listings.$inferInsert; // what an insert accepts (defaults optional)
```

**oRPC** (tRPC works the same way) defines procedures on the server; the client is typed from the router's type, with no code generation:

```ts
// server/router.ts
import { ORPCError, os } from '@orpc/server';
import { z } from 'zod';

type Context = { user: { id: string; orgId: string } | null };
const base = os.$context<Context>();

const getListing = base
  .input(z.object({ id: z.string() }))
  .handler(async ({ input, context }) => {
    if (!context.user) throw new ORPCError('UNAUTHORIZED');
    return { id: input.id, title: 'Sample listing', orgId: context.user.orgId };
  });

export const router = { listing: { get: getListing } };
export type AppRouter = typeof router;
```

```ts
// client/orpc.ts
import { createORPCClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { RouterClient } from '@orpc/server';
import type { AppRouter } from '../server/router'; // type-only: no server code in the bundle

const link = new RPCLink({ url: 'http://localhost:3000/rpc' });
export const orpc: RouterClient<AppRouter> = createORPCClient(link);

// await orpc.listing.get({ id: 'l1' }) returns { id: string; title: string; orgId: string }
// orpc.listing.get({ id: 1 }) does not compile: id must be a string
```

These snippets follow oRPC's documented builder style; package names and helpers can change between versions, so check the oRPC docs before relying on exact imports. TanStack Start's server functions (`createServerFn`) give a similar typed bridge inside one app. More in [oRPC and type-safe APIs](../../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [TanStack Start](../../07-nextjs-and-tanstack/05-tanstack-start.md) and [Drizzle ORM](../../12-databases/13-drizzle-orm.md).

**Type-only imports** make the intent explicit and guarantee the import disappears from the output:

```ts
import type { AppRouter } from '../server/router';
import { type Listing, formatListingTitle } from './listing';
```

Under `verbatimModuleSyntax`, importing a type without `type` is an error, which prevents accidentally bundling server modules (and their secrets or database drivers) into the client.

## 7. typescript-eslint

`tsc` checks types; ESLint with typescript-eslint catches risky patterns, and its type-aware rules use the type checker:

```js
// eslint.config.mjs
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
  },
);
```

| Rule | Catches |
|---|---|
| `no-floating-promises` | promises that are neither awaited nor handled (unhandled rejections) |
| `no-misused-promises` | async functions passed where a void callback is expected, promises in conditions |
| `no-explicit-any` | new `any` creeping in |
| `consistent-type-imports` | enforces `import type` for type-only imports |
| `no-unnecessary-type-assertion` | casts that do nothing |
| `switch-exhaustiveness-check` | switches over unions that miss a case |

Config helpers have changed across releases (newer setups may use ESLint's own `defineConfig`), so copy the current snippet from the typescript-eslint docs.

## 8. Migrating JavaScript to TypeScript gradually

1. Add a `tsconfig.json` with `allowJs: true` so `.js` and `.ts` files coexist; start with `strict: false` if the codebase is large, or strict for new files only.
2. Convert leaf modules first (utilities, API clients, constants), then shared types for API responses and models, then components.
3. For files that stay JavaScript for a while, add `// @ts-check` and JSDoc types to get checking without renaming:

```js
// @ts-check
/**
 * @param {number} cents
 * @param {string} [currency]
 * @returns {string}
 */
export function formatPrice(cents, currency = 'PKR') {
  return `${currency} ${(cents / 100).toFixed(2)}`;
}
```

4. Turn on strict flags one at a time (`noImplicitAny`, then `strictNullChecks`), fixing errors folder by folder with CI tracking progress.
5. Replace `any` with `unknown` plus validation; install `@types/*` or write shims for untyped packages.
6. Use `// @ts-expect-error <reason>` instead of `// @ts-ignore`: it fails once the underlying error is fixed, so suppressions do not rot.
7. Add typescript-eslint rules so new code does not reintroduce `any` or floating promises.

In a Rails monolith with a JavaScript front end (React on Rails), the same plan applies through the app's bundler configuration: TypeScript arrives file by file without a big-bang rewrite.

## 9. Reading common compiler errors

| Code | Message (shortened) | Usual fix |
|---|---|---|
| TS2322 | Type 'X' is not assignable to type 'Y' | fix the value or the declared type; read the nested lines |
| TS2345 | Argument of type 'X' is not assignable to parameter of type 'Y' | same, at a call site |
| TS2339 | Property 'x' does not exist on type 'Y' | typo, missing narrowing, or the type needs the property |
| TS2532 / TS18048 | Object is possibly 'undefined' / 'x' is possibly 'undefined' | narrow (`if`, `?.`, `??`) or fix the type |
| TS2531 / TS18047 | ... possibly 'null' | same for `null` |
| TS7006 | Parameter 'x' implicitly has an 'any' type | annotate the parameter |
| TS2307 | Cannot find module 'x' or its corresponding type declarations | path, alias config or missing package |
| TS7016 | Could not find a declaration file for module 'x' | `@types/x` or a shim |
| TS2554 | Expected 2 arguments, but got 1 | fix the call or make the parameter optional |
| TS2741 | Property 'x' is missing in type 'A' but required in type 'B' | add the property or make it optional |
| TS2353 | Object literal may only specify known properties | typo or an outdated type (excess property check) |
| TS2769 | No overload matches this call | read each overload's error; usually one argument is wrong |
| TS2564 | Property 'x' has no initializer and is not definitely assigned | initialize it, assign in the constructor, or rethink the design |

```text
Type '{ id: string; price: string; }' is not assignable to type 'Product'.
  Types of property 'price' are incompatible.
    Type 'string' is not assignable to type 'number'.
```

Read nested errors from the bottom: the last line is the root mismatch (`price` is a string, probably straight from a form or an API). Other tactics: hover over values in the editor to see the inferred types, break long expressions into annotated variables so the error moves to the right line, and use `satisfies` on object literals to get errors at the definition instead of at a distant use.

## Common pitfalls
- `paths` aliases that work in the editor but fail at runtime because the bundler or Node was not configured.
- Forgetting `.js` extensions under `nodenext`, or using `bundler` resolution for code Node runs directly.
- Relying on the bundler and never running `tsc --noEmit` in CI, so type errors ship.
- `skipLibCheck` hiding real conflicts in your own `.d.ts` files (it skips all declaration files).
- `useState(null)` without a type argument, leaving the state typed as `null` forever.
- Typing context as non-null with a fake default object instead of `| null` plus a guarding hook.
- Casting `await res.json()` or `request.json()` instead of validating.
- Reading `process.env` everywhere instead of one validated `env` module.
- Importing server modules as values into client code (bundling secrets or drivers); use `import type`.
- `@ts-ignore` sprinkled around during a migration and never removed.

## How to talk about this in an interview
- Walk through the full chain on the platform migration at Devntech: Drizzle schema types, oRPC procedures with Zod inputs and typed context, TanStack Start on the client, and how a schema change surfaces as compile errors in the UI.
  > **Fill in:** what your part of that chain was (Rails sync APIs, the marketplace page migration, org-scoped access control) and one concrete type-safety win.
- For the Expo app, mention typed navigation params, typed API clients and validated payloads; for RemoteDev, typed Next.js API routes over MongoDB.
  > **Fill in:** how request bodies were validated in the Next.js APIs at RemoteDev (Zod, manual checks, Mongoose schemas) and how errors were returned.
- Show judgment about strictness: which flags you enable, why `noUncheckedIndexedAccess` is worth the noise, and how you would migrate a loose codebase without stopping feature work.
  > **Fill in:** the tsconfig settings of a project you know well, and anything you would change.

## Interview questions

**Q1. Which tsconfig options do you always enable, and why?**
`strict` (null safety, no implicit `any`, safer function types), `noUncheckedIndexedAccess` (lookups may be undefined), `isolatedModules` or `verbatimModuleSyntax` when a single-file transpiler builds the code, `skipLibCheck` for speed, and `noEmit` when a bundler emits. Then the right `module`/`moduleResolution` pair for the runtime.
*Follow-up:* Why is `noUncheckedIndexedAccess` not in `strict`? It is noisy for existing code, so it is opt-in.

**Q2. What is the difference between `moduleResolution: bundler` and `nodenext`?**
`bundler` models tools like Vite, Next.js and Metro: extensionless relative imports and package `exports` are supported. `nodenext` models Node itself: the module format follows package.json `"type"`, and ES module files must import relative paths with their runtime extension (`./app.js`).
*Follow-up:* Which for a library published to npm? `nodenext`, so it works for consumers running plain Node.

**Q3. What do `isolatedModules` and `verbatimModuleSyntax` do?**
`isolatedModules` reports constructs a per-file transpiler cannot compile correctly, such as re-exporting a type without `export type`. `verbatimModuleSyntax` emits imports exactly as written: anything without `type` stays, so type-only imports must use `import type`, which makes the output predictable.
*Follow-up:* Why do Vite and SWC need this? They compile each file without type information, so they cannot tell whether an import is a type.

**Q4. What are `skipLibCheck` and `esModuleInterop` for?**
`skipLibCheck` skips type-checking every `.d.ts` file, speeding builds and avoiding conflicts between dependencies' types. `esModuleInterop` emits helpers so CommonJS modules can be default-imported (`import express from 'express'`) the way bundlers and Node allow.
*Follow-up:* What is the risk of `skipLibCheck`? Errors in your own declaration files are skipped too.

**Q5. What is a `.d.ts` file? How do you use an untyped library?**
A declaration file with types only, describing JavaScript implemented elsewhere. For an untyped package, install `@types/<name>` from DefinitelyTyped if available; otherwise write `declare module 'name' { ... }` with the functions you use (or a one-line shim as a stopgap).
*Follow-up:* How do you type non-code imports like SVGs? `declare module '*.svg' { const url: string; export default url; }`.

**Q6. How do you type component props and `children`?**
With a `type` (or interface) for props, defaults in destructuring for optional props, and `children: ReactNode`, which accepts anything renderable. `ReactElement`/`JSX.Element` is narrower: exactly one element.
*Follow-up:* How do you accept every native button prop? `ComponentProps<'button'> & { loading?: boolean }`.

**Q7. How do you type `useState` with an initial `null`, and `useRef` for DOM vs mutable values?**
`useState<User | null>(null)`, because the initial value alone would infer `null`. DOM refs: `useRef<HTMLInputElement>(null)`, then `ref.current?.focus()`. Mutable values: `useRef<number | null>(null)` for a timer id you assign later.
*Follow-up:* Why `?.` on `ref.current`? It is `null` until React attaches the element.

**Q8. How do you type a context so consumers never handle `null`?**
Create it as `createContext<Value | null>(null)` and expose a custom hook that reads it, throws a clear error if it is `null` (used outside the provider) and returns `Value`. Consumers call the hook and get a non-null type.
*Follow-up:* Why not a fake default object? It hides missing providers and lies about the shape.

**Q9. Why do you need runtime validation such as Zod if you use TypeScript?**
TypeScript types are erased, so nothing checks data that arrives at runtime: request bodies, query params, env vars, webhooks, third-party responses, storage. A schema validates the data and produces the static type from the same definition, while a cast only silences the compiler.
*Follow-up:* Where exactly do you validate? At trust boundaries; inside the app, trust the types.

**Q10. What is the difference between `parse` and `safeParse`, and what does `z.infer` do?**
`parse` returns the typed data or throws a `ZodError`; `safeParse` never throws and returns `{ success: true, data }` or `{ success: false, error }`, which suits returning field errors. `z.infer<typeof Schema>` extracts the static output type from a schema.
*Follow-up:* When do input and output types differ? With transforms, coercion or defaults; use `z.input` and `z.output`.

**Q11. How do you handle environment variables safely in a TypeScript server?**
Define a Zod schema for every variable (with `z.coerce.number()` for ports, enums for `NODE_ENV`, URL and length checks), parse `process.env` once at startup so the process fails fast, and export the typed result as the only way to read config.
*Follow-up:* What about client-side env vars? Only explicitly public ones are exposed (for example `VITE_` or `NEXT_PUBLIC_` prefixes), and they must never contain secrets.

**Q12. What is end-to-end type safety? How do oRPC and tRPC give typed clients without codegen?**
Types flow from the database schema through the API to the UI from one source of truth. The server router is a plain TypeScript object, its type is exported with `import type`, and the client library maps it into callable functions whose inputs and outputs are inferred, so a server change becomes a client compile error.
*Follow-up:* Does that replace validation? No: inputs are still validated at runtime by the procedure's schema.

**Q13. How do you derive TypeScript types from a Drizzle schema?**
`typeof table.$inferSelect` gives the row type returned by selects and `typeof table.$inferInsert` the insert type, where columns with defaults or nullability become optional. Query results are typed automatically from the columns you select.
*Follow-up:* How do you avoid duplicating these as Zod schemas? Generate schemas from tables with a Drizzle-Zod integration (check the current package for your version), or derive one from the other.

**Q14. Why use `import type`?**
It guarantees the import is erased, which keeps server-only modules, secrets and heavy dependencies out of client bundles and avoids runtime import side effects. Under `verbatimModuleSyntax` it is required for type-only imports.
*Follow-up:* Can you mix in one statement? Yes: `import { type Listing, formatListingTitle } from './listing'`.

**Q15. How would you migrate a JavaScript codebase to TypeScript?**
Enable `allowJs`, convert leaf modules and shared API types first, use JSDoc with `// @ts-check` for files that stay JavaScript, turn on strict flags one by one with CI tracking, replace `any` with `unknown` plus validation, and add typescript-eslint rules. Ship continuously instead of a big-bang rewrite.
*Follow-up:* What do you convert first in a React app? API clients and response types, because they protect every screen that uses them.

**Q16. `@ts-ignore` vs `@ts-expect-error`?**
Both suppress the error on the next line. `@ts-expect-error` itself errors when there is no longer an error to suppress, so stale suppressions get cleaned up; `@ts-ignore` silently stays forever. Always add a reason.
*Follow-up:* Better than both? Fixing the type, or narrowing with a real check.

**Q17. How do you read a long "is not assignable" error?**
From the bottom: TypeScript prints the outer mismatch first and the root cause last (for example "Type 'string' is not assignable to type 'number'" for one property). Hover to see inferred types, and split expressions into annotated variables or use `satisfies` so the error points at the right line.
*Follow-up:* What does TS2353 mean? An excess property in an object literal, usually a typo or an outdated type.

**Q18. What does `ComponentProps<'button'>` give you?**
The full prop type of a native `<button>` (onClick, disabled, type, aria attributes and so on), so a wrapper component can accept and spread all of them while adding its own props. `ComponentProps<typeof MyComponent>` does the same for a component.
*Follow-up:* What about refs? In React 19, `ref` is a regular prop for function components, so the wrapper can accept and forward it.

**Q19. What does typescript-eslint add beyond the compiler?**
Lint rules for TypeScript code, including type-aware ones that use the checker: `no-floating-promises` (unhandled promises), `no-misused-promises` (async callbacks where void is expected), `no-explicit-any`, `consistent-type-imports` and `switch-exhaustiveness-check`.
*Follow-up:* What do type-aware rules require? Parser access to your tsconfig (for example `projectService` in recent versions), which makes linting slower.

## Further reading
- [TypeScript: TSConfig reference](https://www.typescriptlang.org/tsconfig/)
- [TypeScript: Modules reference (module resolution)](https://www.typescriptlang.org/docs/handbook/modules/reference.html)
- [React: Using TypeScript](https://react.dev/learn/typescript)
- [Drizzle ORM documentation](https://orm.drizzle.team/docs/overview)
- [oRPC documentation](https://orpc.unnoq.com/)
