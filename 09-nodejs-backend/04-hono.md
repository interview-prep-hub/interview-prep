# Hono

> **Tier:** T1 (know cold) · **Related:** [Express](03-express.md), [oRPC and type-safe APIs](05-orpc-and-typesafe-apis.md), [Production APIs](06-production-apis.md), [Node.js fundamentals](01-nodejs-fundamentals.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [Drizzle ORM](../12-databases/13-drizzle-orm.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)
> **On your resume:** Node.js (Hono, oRPC) in your skills list, next to the TypeScript backend work on the Devntech multi-tenant platform (Rails sync APIs, org-scoped access control).

## Quick revision
- Hono is a small, fast web framework built on Web Standards: a handler receives a context `c` wrapping a standard `Request` and returns a standard `Response`.
- One app runs on Cloudflare Workers, Deno, Bun, Node (`@hono/node-server`), AWS Lambda (`hono/aws-lambda`), Vercel and more. An app is just `app.fetch(request, env, ctx)`.
- Routing: `app.get('/listings/:id', h)`, optional `:type?`, regex `:id{[0-9]+}`, wildcards `*`; sub-apps with `app.route('/listings', listings)`; prefixes with `new Hono().basePath('/api')`.
- Request: `c.req.param('id')`, `c.req.query('page')`, `c.req.queries('tag')`, `c.req.header('authorization')`, `await c.req.json()`, `await c.req.parseBody()`, `c.req.raw`.
- Response: `c.json(data, 201)`, `c.text()`, `c.html()`, `c.body()`, `c.redirect()`, `c.status()`, `c.header()`, `c.notFound()`.
- Typed per-request data: `new Hono<{ Variables: { user: User } }>()`, then `c.set('user', u)` and `c.get('user')` (or `c.var.user`). Platform bindings (Workers KV, D1, secrets) arrive in `c.env`, typed through `Bindings`.
- Middleware is `async (c, next) => { before; await next(); after }`: an onion, so code after `await next()` runs on the way out and can read or change `c.res`.
- Built-in middleware: `logger`, `cors`, `secureHeaders`, `jwt` (the `alg` option is required in recent versions), `bearerAuth`, `basicAuth`, `etag`, `compress`, `bodyLimit`, `timeout`, `requestId`, `csrf`.
- Typed custom middleware: `createMiddleware<{ Variables: {...} }>(...)` from `hono/factory`.
- Validation: `zValidator(target, schema)` from `@hono/zod-validator` for `json`, `query`, `param`, `header`, `form` or `cookie`; then `c.req.valid('json')` is typed. Failures return 400 unless you pass a hook.
- Errors: `throw new HTTPException(401, { message })`; `app.onError((err, c) => ...)` catches everything thrown; `app.notFound((c) => ...)` handles unmatched routes.
- RPC mode: chain routes, `export type AppType = typeof routes`, and `hc<AppType>(baseUrl)` gives a client with typed paths, inputs and responses, no codegen.
- Testing: `await app.request('/path', init, env)` returns a `Response`; no server or port needed.
- Mount other fetch-style handlers (oRPC, Better Auth) by passing them `c.req.raw` and returning their `Response`.
- vs Express: Web Standards and multi-runtime, onion middleware, first-class TypeScript and RPC, tiny with fast cold starts; a smaller ecosystem, and Express middleware does not plug in.

## 1. What Hono is and why teams pick it

Express hands you Node's `req` and `res` objects and expects you to mutate `res`. Hono hands you the Web Platform's `Request` and expects you to return a `Response`, the same objects `fetch`, Service Workers and every edge runtime use. That one decision buys:
- **Portability:** the same code runs on Node, Bun, Deno, Workers and serverless platforms.
- **Size and speed:** no dependencies, fast routers, quick cold starts on serverless.
- **Types:** route params, validated inputs, context variables and even the client are inferred.

```ts
// src/index.ts: a Node server
import { serve } from '@hono/node-server';
import { Hono } from 'hono';

const app = new Hono();
app.get('/', (c) => c.text('Hello Hono'));
app.get('/health', (c) => c.json({ ok: true }));

serve({ fetch: app.fetch, port: 3000 }, (info) => console.log(`Listening on http://localhost:${info.port}`));
```

## 2. One app, many runtimes

| Runtime | Entry point |
|---|---|
| Cloudflare Workers | `export default app` |
| Bun | `export default app` (or `export default { port: 3000, fetch: app.fetch }`) |
| Deno | `Deno.serve(app.fetch)` |
| Node.js | `serve({ fetch: app.fetch, port })` from `@hono/node-server` |
| AWS Lambda | `export const handler = handle(app)` from `hono/aws-lambda` |
| Next.js / Vercel | `export const GET = handle(app)` from `hono/vercel` in a catch-all route handler |

```ts
// app/api/[[...route]]/route.ts: a Hono app inside Next.js
import { Hono } from 'hono';
import { handle } from 'hono/vercel';

const app = new Hono().basePath('/api');
app.get('/hello', (c) => c.json({ message: 'Hello from Hono' }));

export const GET = handle(app);
export const POST = handle(app);
```

What changes between runtimes is the edges, not the routes: how you read configuration (`c.env` vs `process.env`), static file helpers, and whether your libraries work there (a TCP Postgres driver is fine on Node and Bun; Workers need compatible drivers).

## 3. Routing

```ts
const app = new Hono();

app.get('/listings/:id', (c) => c.json({ id: c.req.param('id') }));      // typed: 'id' is known
app.get('/orders/:id{[0-9]+}', (c) => c.json({ id: Number(c.req.param('id')) })); // digits only
app.get('/animals/:type?', (c) => c.text(c.req.param('type') ?? 'any'));  // optional segment
app.get('/docs/*', (c) => c.text(`doc: ${c.req.path}`));                   // wildcard
app.on(['PUT', 'PATCH'], '/listings/:id', (c) => c.json({ updated: c.req.param('id') }));
app.all('/legacy/*', (c) => c.redirect('/v2', 301));
```

Split features into sub-apps and mount them:

```ts
// src/routes/listings.ts
export const listings = new Hono<AppEnv>()
  .get('/', (c) => c.json({ data: [] }))
  .get('/:id', (c) => c.json({ id: c.req.param('id') }));

// src/app.ts
export const app = new Hono<AppEnv>().basePath('/api');
app.route('/listings', listings); // GET /api/listings and GET /api/listings/:id
```

Handlers and middleware run in registration order and the first handler that returns a response wins, so register catch-all routes last.

## 4. The context object

| Request side | Returns |
|---|---|
| `c.req.param('id')` / `c.req.param()` | one path param / all of them |
| `c.req.query('page')` / `c.req.queries('tag')` | one query value / all values for a repeated key |
| `c.req.header('authorization')` | a header |
| `await c.req.json()`, `text()`, `arrayBuffer()`, `parseBody()` | the body as JSON, text, bytes, or form data |
| `c.req.valid('json')` | the validated, typed input (after a validator) |
| `c.req.raw`, `c.req.path`, `c.req.method`, `c.req.url` | the underlying `Request` and its parts |

Responses: `c.json(obj, 201)`, `c.text('ok')`, `c.html('<p>hi</p>')`, `c.body(stream)`, `c.redirect('/login', 302)`, `c.notFound()`; set status and headers ahead of time with `c.status(201)` and `c.header('Cache-Control', 'no-store')`.

Typed variables and bindings:

```ts
type SessionUser = { id: string; orgId: string; role: 'owner' | 'admin' | 'member' };
type AppEnv = {
  Bindings: { JWT_SECRET: string };        // c.env on Workers (secrets, KV, D1, queues)
  Variables: { user: SessionUser };        // c.set / c.get, per request
};

const app = new Hono<AppEnv>();
app.get('/me', (c) => {
  const user = c.get('user');              // SessionUser (also c.var.user)
  return c.json({ id: user.id, orgId: user.orgId });
});
```

The type promises `user` exists, so make sure a middleware sets it for these routes. On Node, `c.env` holds the raw Node request and response rather than your config: read validated `process.env` values, or use `env(c)` from `hono/adapter`, which picks the right source per runtime. On Workers, `c.executionCtx.waitUntil(promise)` lets work finish after the response is sent.

## 5. Middleware: the onion model

```ts
app.use(async (c, next) => {
  const start = Date.now();
  await next();                                            // the rest of the chain and the handler run here
  c.header('X-Response-Time', `${Date.now() - start}ms`);  // on the way out: the response exists now
});
```

Each middleware wraps everything registered after it, like Rails `around_action` or Rack middleware. Express middleware is a flat list, so "after" logic there needs `res.on('finish')`.

```ts
import { bearerAuth } from 'hono/bearer-auth';
import { bodyLimit } from 'hono/body-limit';
import { cors } from 'hono/cors';
import { jwt } from 'hono/jwt';
import { logger } from 'hono/logger';
import { secureHeaders } from 'hono/secure-headers';

app.use(logger());
app.use(secureHeaders());
app.use('/api/*', cors({ origin: ['https://app.example.com'], credentials: true }));
app.use('/api/*', bodyLimit({ maxSize: 100 * 1024 }));                      // bytes
app.use('/api/*', jwt({ secret: env.JWT_SECRET, alg: 'HS256' }));           // payload in c.get('jwtPayload')
app.use('/internal/*', bearerAuth({ token: env.INTERNAL_API_TOKEN }));      // service-to-service calls
```

For your own typed middleware, use `createMiddleware`, which carries its variable types into every handler that uses it:

```ts
import { createMiddleware } from 'hono/factory';
import { HTTPException } from 'hono/http-exception';

export const requireUser = createMiddleware<{ Variables: { user: SessionUser } }>(async (c, next) => {
  const session = await getSession(c.req.raw.headers);   // your auth library: cookie or Bearer token
  if (!session) throw new HTTPException(401, { message: 'Unauthorized' });
  c.set('user', session.user);
  await next();
});

export const requireRole = (...roles: SessionUser['role'][]) =>
  createMiddleware<{ Variables: { user: SessionUser } }>(async (c, next) => {
    if (!roles.includes(c.get('user').role)) throw new HTTPException(403, { message: 'Forbidden' });
    await next();
  });
```

Middleware can also end the chain by returning a response (`return c.json({ error: 'Unauthorized' }, 401)`). If a middleware neither returns a response nor awaits `next()`, Hono throws a "Context is not finalized" error.

## 6. Validation with @hono/zod-validator

```ts
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';

const CreateListing = z.object({ title: z.string().trim().min(3), priceCents: z.number().int().positive() });
const ListQuery = z.object({
  status: z.enum(['draft', 'published']).optional(),
  page: z.coerce.number().int().min(1).default(1),      // query values are strings: coerce
});

const listingsRoutes = new Hono<AppEnv>()
  .use(requireUser)
  .get('/', zValidator('query', ListQuery), async (c) => {
    const { status, page } = c.req.valid('query');       // typed; page is a number
    return c.json({ data: await listingsFor(c.var.user).list({ status, page }) }, 200);
  })
  .post(
    '/',
    zValidator('json', CreateListing, (result, c) => {   // optional hook: your own error shape
      if (!result.success) return c.json({ error: { code: 'VALIDATION', issues: result.error.issues } }, 422);
    }),
    async (c) => {
      const row = await listingsFor(c.var.user).create(c.req.valid('json'));
      return c.json({ data: row }, 201);
    },
  );
```

`listingsFor` is a tenant-scoped repository (see [Drizzle multi-tenant scoping](../12-databases/13-drizzle-orm.md)). Without a hook, invalid input gets a 400 with the Zod error; the hook keeps error responses consistent with the rest of your API. Recent versions of `@hono/zod-validator` support Zod 4; check the package notes if types misbehave after an upgrade.

## 7. Errors and 404s

```ts
import { HTTPException } from 'hono/http-exception';

app.notFound((c) => c.json({ error: { code: 'ROUTE_NOT_FOUND', message: `No route for ${c.req.path}` } }, 404));

app.onError((err, c) => {
  if (err instanceof HTTPException) {
    return c.json({ error: { code: 'HTTP_ERROR', message: err.message } }, err.status);
  }
  if (err instanceof NotFoundError) {                       // your own domain errors
    return c.json({ error: { code: 'NOT_FOUND', message: err.message } }, 404);
  }
  console.error(err);                                        // use a structured logger in production
  return c.json({ error: { code: 'INTERNAL', message: 'Internal Server Error' } }, 500);
});
```

- `HTTPException` carries a status and message (or a full custom `res`); `err.getResponse()` turns it into a `Response`.
- Without `onError`, an unexpected error becomes a plain 500. With it, every thrown error, including middleware errors, takes one path.

## 8. RPC mode: a typed client without codegen

```ts
// server: src/app.ts
const app = new Hono<AppEnv>().basePath('/api');
const routes = app.route('/listings', listingsRoutes);   // keep the returned value: it carries the types
export type AppType = typeof routes;
export default app;
```

```ts
// client: any TypeScript frontend in the same repo or monorepo
import { hc } from 'hono/client';
import type { AppType } from '../server/app';            // type-only import: no server code bundled

const client = hc<AppType>('https://api.example.com', {
  headers: () => ({ Authorization: `Bearer ${getAccessToken()}` }),
});

const res = await client.api.listings.$get({ query: { status: 'published' } });
if (res.ok) {
  const body = await res.json();                          // typed from the handler's c.json(...)
  console.log(body.data.length);
}
await client.api.listings.$post({ json: { title: 'Room in Brussels', priceCents: 65000 } });
```

- Types only flow through **chained** definitions (`new Hono().get(...).post(...)`) or the value returned by `app.route`. Separate `app.get(...)` statements leave `typeof app` without routes.
- Pass explicit status codes (`c.json(data, 200)`) so the client can narrow on `res.status`; `InferRequestType` and `InferResponseType` from `hono/client` extract types for forms and hooks.
- Very large apps can slow the editor; split into sub-apps and follow the docs' advice on compiling types.
- With TanStack Query, call the client inside `queryFn`, check `res.ok`, and return `res.json()`.
- Hono RPC keeps resource-style HTTP routes and types the client. oRPC is procedure-based, with typed error maps, contract-first support, OpenAPI generation and TanStack Query helpers ([oRPC](05-orpc-and-typesafe-apis.md)).

## 9. Testing with app.request()

```ts
import { describe, expect, it } from 'vitest';
import app from './app';

describe('listings routes', () => {
  it('responds to health checks', async () => {
    const res = await app.request('/api/health');
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ ok: true });
  });

  it('rejects an invalid listing', async () => {
    const res = await app.request('/api/listings', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${await testToken()}` },
      body: JSON.stringify({ title: 'x' }),
    });
    expect(res.status).toBe(422);
  });
});
```

- `app.request(path, init, env)` takes a third argument with mock bindings for Workers-style apps.
- `testClient(app)` from `hono/testing` gives the RPC client against the app in memory.
- To swap dependencies, build the app in a factory (`createApp({ db })`) or set them per request in middleware (`c.set('db', db)`).

## 10. Mounting other handlers inside Hono

Any library that speaks `Request`/`Response` plugs in. Mounting an oRPC router (oRPC v1 style; check the oRPC Hono adapter docs):

```ts
import { RPCHandler } from '@orpc/server/fetch';
import { router } from './orpc/router';

const rpcHandler = new RPCHandler(router);

app.use('/rpc/*', async (c, next) => {
  const { matched, response } = await rpcHandler.handle(c.req.raw, {
    prefix: '/rpc',
    context: { headers: c.req.raw.headers, db },            // oRPC initial context
  });
  if (matched) return c.newResponse(response.body, response);
  await next();                                             // not an RPC path: continue to other routes
});
```

- Do not read the body in earlier middleware for these paths: the inner handler needs it, or you get a "body already used" error.
- Auth libraries mount the same way, for example Better Auth: `app.on(['GET', 'POST'], '/api/auth/*', (c) => auth.handler(c.req.raw))` (check the library's Hono guide).
- `app.mount('/legacy', otherApp.fetch)` mounts apps built with other fetch-based frameworks.

## 11. Hono vs Express

| | Express | Hono |
|---|---|---|
| API | Node `req`/`res`; you mutate `res` | Web `Request`/`Response`; handlers return a `Response` |
| Runtimes | Node | Workers, Deno, Bun, Node, Lambda, Vercel |
| Middleware | `(req, res, next)`, a flat list | `async (c, next)`, an onion: code after `await next()` sees the response |
| Async errors | Express 5 forwards rejections; Express 4 needs wrappers | handled; one `app.onError` |
| Validation | add Zod middleware yourself | `@hono/zod-validator`, typed `c.req.valid()` |
| Request-scoped data | `req.user` via global type augmentation | typed `Variables`, `c.set`/`c.get` |
| Typed client | none (OpenAPI codegen) | `hc<AppType>()` |
| Body parsing | middleware (`express.json()`) | on demand (`await c.req.json()`) |
| Footprint | heavier; slower cold starts | tiny, zero dependencies |
| Ecosystem | huge (passport, multer, many more) | smaller; Express middleware does not work |

## Common pitfalls
- Defining routes as separate `app.get()` statements and then wondering why `hc<AppType>` has no routes.
- Middleware that forgets `await next()` (or a `return`), causing "Context is not finalized".
- Consuming the body in middleware before a mounted handler (oRPC, auth) that needs it.
- Expecting `c.env` to hold `process.env` values on Node.
- Using Node-only packages (native drivers, `fs`) in code deployed to Workers.
- Omitting `alg` in the `jwt` middleware on recent versions, or forgetting that JWT payloads are readable by anyone.
- Registering CORS after routes, or only on some paths, so preflight `OPTIONS` requests fail.
- Leaving the validator's default 400 when the rest of the API uses a 422 error envelope.
- Throwing plain `Error` for expected cases, which turns them into 500s; map domain errors in `onError`.
- Omitting explicit status codes in `c.json`, so RPC clients cannot narrow on `res.status`.

## How to talk about this in an interview
- Your skills list Node.js with Hono and oRPC, so be ready to explain why Hono fits a TypeScript platform: Web Standards, typed middleware context, Zod validation, tiny footprint, and it hosts an oRPC handler with a few lines.
  > **Fill in:** where you used Hono (an API server for the platform, a sync service for the Rails integration, a side project), and what you built with it: routes, auth middleware, validation, mounting oRPC.
- Map it to what you know: the onion middleware is Rails `around_action`; `HTTPException` plus `onError` is `rescue_from`; sub-apps with `app.route` are `namespace` blocks in `routes.rb`.
- Have the RPC decision ready: Hono RPC when you want plain HTTP routes with a typed client; oRPC when you want procedures, typed errors, OpenAPI output and TanStack Query helpers. Background: [oRPC and type-safe APIs](05-orpc-and-typesafe-apis.md) and the [platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md).

## Interview questions

**Q1. What is Hono, and why choose it over Express?**
A lightweight TypeScript web framework built on the Web `Request`/`Response` APIs, so one app runs on Node, Bun, Deno, Workers and serverless platforms. You get typed routes, validated inputs, typed context variables and an RPC client, with tiny bundles and fast cold starts. Express wins on ecosystem and familiarity.
*Follow-up:* When would you still pick Express? An existing Express codebase, or a dependency on Express-only middleware.

**Q2. What does "built on Web Standards" mean in practice?**
Handlers take a standard `Request` (via `c.req.raw`) and return a standard `Response`, using `Headers`, `URL`, Web Streams and Web Crypto rather than Node-specific objects. The code is portable across runtimes and interoperates with anything fetch-based, such as oRPC handlers or auth libraries.
*Follow-up:* What still differs per runtime? Configuration access, file system access and driver compatibility.

**Q3. How do you run the same Hono app on Node and on Cloudflare Workers?**
Keep routes in an `app` module. On Workers, `export default app`; on Node, `serve({ fetch: app.fetch, port })` from `@hono/node-server`. Read configuration through an adapter (`env(c)`) or a small config layer instead of touching `process.env` inside routes.
*Follow-up:* What breaks first when moving to Workers? Libraries that need Node APIs or raw TCP.

**Q4. Explain the context object `c`.**
The per-request object: `c.req` wraps the request (params, query, headers, body, validated data); `c.json`, `c.text`, `c.redirect` and friends build responses; `c.set`/`c.get` store typed variables between middleware and handlers; `c.env` holds platform bindings; `c.executionCtx` exposes `waitUntil` on Workers.
*Follow-up:* Why `c.req.raw`? To pass the original `Request` to libraries that expect it.

**Q5. How does Hono middleware differ from Express middleware?**
Hono middleware is `async (c, next)` and wraps everything after it (an onion): code after `await next()` runs on the way back and can read or modify `c.res`, which makes timing, headers and error wrapping natural. Express middleware is a flat chain; post-response logic needs `res.on('finish')`.
*Follow-up:* What happens if middleware never calls `next` or returns a response? Hono throws "Context is not finalized".

**Q6. How do you pass typed data, such as the current user, from middleware to handlers?**
Declare `Variables` in the app's generic (`new Hono<{ Variables: { user: User } }>()`) or in `createMiddleware<...>`; the middleware calls `c.set('user', user)` and handlers read `c.get('user')` or `c.var.user` with full types.
*Follow-up:* Express equivalent? Augmenting `Express.Request` globally, which types `req.user` on every route whether or not auth ran.

**Q7. How do you validate input in Hono?**
`zValidator('json', schema)` (or `query`, `param`, `header`, `form`, `cookie`) from `@hono/zod-validator`, placed before the handler; inside, `c.req.valid('json')` returns typed, parsed data. A hook argument customizes the failure response, for example a 422 with your error envelope.
*Follow-up:* Why `z.coerce.number()` for query params? Query values are always strings.

**Q8. How do you handle errors and 404s?**
Throw `HTTPException(status, { message })` for expected HTTP failures, and register `app.onError` to map `HTTPException`, domain errors and unknown errors into consistent JSON (logging the unknown ones). `app.notFound` handles unmatched routes.
*Follow-up:* Do errors thrown in middleware reach `onError`? Yes, everything thrown in the chain does.

**Q9. What is Hono RPC, and what are its gotchas?**
Export the type of your chained routes (`export type AppType = typeof routes`) and create a client with `hc<AppType>(url)`: paths, inputs (from validators) and responses (from `c.json`) are typed, with no code generation. Gotchas: routes must be chained or the types are lost, status codes should be explicit for narrowing, client and server must share the type (same repo or monorepo), and very large route trees slow the editor.
*Follow-up:* Is it REST? The routes are ordinary HTTP endpoints, so non-TypeScript clients can still call them.

**Q10. How do you organize a larger Hono app?**
One sub-app per feature (`new Hono()` with its routes and middleware), mounted with `app.route('/listings', listings)` under a `basePath('/api')`; shared middleware and the app type defined centrally; services and repositories outside the route files, as in any layered API; `createFactory` helps share typed handlers and middleware.
*Follow-up:* Where does tenant scoping live? In the repository layer, fed by the org from the verified session in middleware.

**Q11. How do you test Hono routes?**
Call `app.request(path, init, env)` in Vitest: it runs the full middleware chain in memory and returns a `Response` to assert on. `testClient` gives a typed RPC client for tests. Inject fakes through an app factory or middleware.
*Follow-up:* What about Workers bindings? Pass a mock `env` object as the third argument.

**Q12. How do you read configuration across runtimes?**
On Workers, bindings and secrets arrive in `c.env`, typed through `Bindings`. On Node, read a Zod-validated config module built from `process.env`. `env(c)` from `hono/adapter` reads the right source per runtime when you need one code path.
*Follow-up:* Why not read `process.env` in route handlers everywhere? It does not exist on Workers, and unvalidated reads hide missing config until runtime.

**Q13. How would you mount an oRPC handler or an auth library inside Hono?**
In middleware on a path prefix, pass `c.req.raw` to the handler (`rpcHandler.handle(c.req.raw, { prefix: '/rpc', context })`), return its response when it matched, and call `next()` otherwise. Auth libraries usually expose a `handler(request)` you route with `app.on(['GET', 'POST'], '/api/auth/*', ...)`.
*Follow-up:* Common bug? Earlier middleware consuming the body, so the inner handler fails with "body already used".

**Q14. Hono RPC vs oRPC vs tRPC?**
Hono RPC types a client over normal HTTP routes and stays lightweight. tRPC and oRPC are procedure-based RPC with end-to-end types; tRPC splits queries and mutations and relies on adapters for OpenAPI, while oRPC treats every procedure the same, adds typed error maps, contract-first development and first-class OpenAPI. oRPC can run inside Hono, so they combine well.
*Follow-up:* Which suits a public API? Plain routes with OpenAPI (Hono with an OpenAPI helper, or oRPC's generator).

**Q15. How do you handle CORS with credentials in Hono?**
`app.use('/api/*', cors({ origin: ['https://app.example.com'], credentials: true }))`, registered before the routes so preflight `OPTIONS` requests are answered. The origin must be explicit, not `*`, when cookies are involved, and the frontend must send `credentials: 'include'`.
*Follow-up:* Can `origin` be dynamic? Yes, a function can check the request origin against an allowlist.

## Further reading
- [Hono documentation](https://hono.dev/docs/)
- [Hono: Middleware guide](https://hono.dev/docs/guides/middleware)
- [Hono: RPC guide](https://hono.dev/docs/guides/rpc)
- [Hono: Testing guide](https://hono.dev/docs/guides/testing)
- [oRPC: Hono adapter](https://orpc.unnoq.com/docs/adapters/hono)
