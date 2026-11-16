# Node.js Backend: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. Output puzzles assume a CommonJS file unless stated otherwise.

## Runtime and event loop (see [01-nodejs-fundamentals.md](01-nodejs-fundamentals.md))

**Q1. What is Node.js in one sentence?**
A JavaScript runtime built on V8 and libuv that runs JavaScript outside the browser with non-blocking, event-driven I/O.

**Q2. What does V8 do, and what does libuv do?**
V8 parses, JIT-compiles and runs JavaScript and manages memory. libuv provides the event loop, asynchronous network I/O through the OS, timers and a thread pool for blocking work.

**Q3. Is Node single-threaded?**
Your JavaScript runs on one main thread. The runtime itself uses more threads: libuv's pool (4 by default) and V8's background threads for garbage collection and compilation.

**Q4. How does one thread serve thousands of concurrent requests?**
Requests mostly wait on I/O. Node registers the I/O with the OS or the thread pool and keeps running other JavaScript; callbacks run when results arrive, so waiting never occupies the thread.

**Q5. Which operations use the libuv thread pool?**
File system calls, `dns.lookup`, CPU-heavy crypto (`pbkdf2`, `scrypt`, async `randomBytes`) and `zlib`. Network sockets do not; they use epoll, kqueue or IOCP.

**Q6. How do you change the thread pool size, and why would you?**
Set `UV_THREADPOOL_SIZE` before the process starts. Raise it when heavy file or crypto work queues up behind four threads.

**Q7. List the event loop phases in order.**
Timers, pending callbacks, idle/prepare (internal), poll, check, close callbacks, and then the cycle repeats.

**Q8. What runs in the poll phase?**
Most I/O callbacks (socket data, file reads). When nothing else is scheduled, the loop waits here for new I/O or the next timer.

**Q9. When do `process.nextTick` callbacks and promise callbacks run?**
After the current operation completes: Node drains the whole nextTick queue, then the whole microtask queue, after every callback and after the main script.

**Q10. `setImmediate` vs `setTimeout(fn, 0)`?**
`setImmediate` runs in the check phase right after poll; `setTimeout(fn, 0)` runs in the timers phase after at least 1 ms. Their order is random in the main module, but inside an I/O callback `setImmediate` always runs first.

**Q11. What is the danger of recursive `process.nextTick`?**
The nextTick queue must drain before the loop continues, so endless recursion starves I/O and timers.

**Q12. What's the output?**
```ts
console.log('start');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('then'));
console.log('end');
```
`start`, `end`, `then`, `timeout`.

**Q13. What's the output?**
```ts
process.nextTick(() => console.log('tick 1'));
Promise.resolve().then(() => {
  console.log('promise 1');
  process.nextTick(() => console.log('tick 2'));
});
process.nextTick(() => console.log('tick 3'));
```
`tick 1`, `tick 3`, `promise 1`, `tick 2`. All ticks first; a tick queued inside a microtask waits until the microtask queue is empty.

**Q14. What's the output?**
```ts
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('tick'));
Promise.resolve().then(() => console.log('promise'));
(async () => {
  console.log('async start');
  await undefined;
  console.log('async end');
})();
console.log('sync');
```
`async start`, `sync`, `tick`, `promise`, `async end`, `immediate`. The async function runs synchronously until `await`; its continuation is queued after the earlier `then`.

**Q15. What's the output?**
```ts
import { readFile } from 'node:fs';

readFile('package.json', () => {
  process.nextTick(() => console.log('tick'));
  setImmediate(() => {
    console.log('immediate');
    process.nextTick(() => console.log('tick in immediate'));
  });
  setTimeout(() => console.log('timeout'), 0);
});
```
`tick`, `immediate`, `tick in immediate`, `timeout`. Inside an I/O callback the check phase comes before timers, and queues drain after each callback.

**Q16. What's the output?**
```ts
setTimeout(() => console.log('timeout 1'), 0);
setTimeout(() => {
  console.log('timeout 2');
  Promise.resolve().then(() => console.log('promise in timeout 2'));
}, 0);
setTimeout(() => console.log('timeout 3'), 0);
```
`timeout 1`, `timeout 2`, `promise in timeout 2`, `timeout 3`. Since Node 11 microtasks drain between individual timer callbacks.

**Q17. What's the output?**
```ts
const p = new Promise((resolve) => {
  console.log('executor');
  resolve('done');
  console.log('after resolve');
});
p.then((v) => console.log(v));
console.log('sync');
```
`executor`, `after resolve`, `sync`, `done`. The executor is synchronous and `resolve` does not stop it.

**Q18. Why can nextTick-vs-promise puzzles give different answers in an ES module?**
ESM top-level code is evaluated inside a promise job, so V8 drains the promise callbacks it queued before Node processes nextTick callbacks. In `.mjs`, a top-level `then` can print before a top-level `nextTick`.

**Q19. What blocks the event loop?**
Long synchronous work: `*Sync` APIs in handlers, huge `JSON.parse`/`stringify`, CPU loops, catastrophic regexes, endless microtask recursion.

**Q20. How do you detect event-loop blocking in production?**
Track event-loop delay (`monitorEventLoopDelay` from `node:perf_hooks`) as a metric with alerts, then profile with `--cpu-prof` or `--inspect` to find the hot function.

**Q21. Three ways to handle CPU-heavy work in a Node API?**
Chunk it and yield with `setImmediate`, move it to a worker thread pool, or enqueue it as a background job processed by separate workers.

**Q22. Does making a function `async` make CPU work non-blocking?**
No. The body still runs on the main thread; async only helps while awaiting I/O.

**Q23. Concurrency vs parallelism in Node?**
Concurrency: many requests in progress, interleaved on one thread. Parallelism: JavaScript running at the same instant on several cores, which needs worker threads or more processes.

**Q24. Where is Node a poor fit?**
CPU-bound services (video encoding, ML inference, heavy computation) unless that work is offloaded.

## Modules, packages and configuration (see [01-nodejs-fundamentals.md](01-nodejs-fundamentals.md))

**Q25. CommonJS vs ES modules in three differences?**
CommonJS uses `require`/`module.exports` and loads synchronously at call time; ESM uses static `import`/`export`, loads asynchronously and supports top-level `await`; ESM needs file extensions in relative imports and has no `__dirname`.

**Q26. What decides whether a `.js` file is ESM?**
The nearest package.json `"type"` field (`"module"` or the default `"commonjs"`); `.mjs` is always ESM and `.cjs` always CommonJS.

**Q27. Can CommonJS code load an ES module?**
Always with `await import()`. Recent Node versions (20.19+, 22.12+) also allow `require()` of ES modules without top-level await; otherwise `ERR_REQUIRE_ASYNC_MODULE`.

**Q28. How do you get `__dirname` in ESM?**
`import.meta.dirname` (Node 20.11+), or `path.dirname(fileURLToPath(import.meta.url))`.

**Q29. What does the `exports` field in package.json do?**
Declares public entry points with conditions (`types`, `import`, `require`, `default`) and blocks imports of unlisted subpaths.

**Q30. `^1.2.3` vs `~1.2.3` vs `^0.2.3`?**
`<2.0.0`; `<1.3.0`; `<0.3.0` (in 0.x the caret stays within the minor version).

**Q31. `npm install` vs `npm ci`?**
`npm install` resolves ranges and may update the lockfile; `npm ci` deletes `node_modules`, installs exactly what the lockfile says and fails if it disagrees with package.json. Use `npm ci` in CI and Docker.

**Q32. What is a peer dependency?**
A package the host application must provide, such as `react` for a component library, so only one copy exists in the app.

**Q33. Why do env-var bugs like `if (process.env.FLAG)` happen?**
Environment values are strings, so `"false"` is truthy. Parse and validate config once with Zod at startup.

**Q34. How do you run TypeScript in Node?**
Compile with `tsc` (or a bundler) for production, `tsx` for development; recent Node versions strip erasable types natively, without type checking, so CI still runs `tsc --noEmit`.

## Core modules, streams and errors (see [02-core-modules-streams-errors.md](02-core-modules-streams-errors.md))

**Q35. Why prefer `node:fs/promises` over sync fs calls?**
Sync calls block the event loop for every request; the promise API runs on the thread pool and keeps the server responsive.

**Q36. `path.join` vs `path.resolve`?**
`join` concatenates and normalizes segments; `resolve` returns an absolute path based on the working directory.

**Q37. How do you prevent path traversal with user-supplied file names?**
Resolve the path against an allowed root and reject it unless the result starts with that root plus a separator.

**Q38. What is a Buffer?**
A fixed-size chunk of raw bytes (a `Uint8Array` subclass) for binary data, files and sockets; encodings include utf8, base64, base64url and hex.

**Q39. Name the four stream types.**
Readable, Writable, Duplex and Transform.

**Q40. What is backpressure?**
A slow consumer signalling a fast producer to pause: `write()` returns `false` when the buffer is full and the producer waits for `'drain'`.

**Q41. Why use `pipeline` instead of `pipe`?**
`pipeline` forwards errors, destroys all streams on failure and returns a promise; `pipe` leaks resources when a stage fails.

**Q42. How do you send a 2 GB file over HTTP without loading it into memory?**
`await pipeline(createReadStream(path), res)`, which streams chunks with backpressure.

**Q43. Is `EventEmitter.emit` synchronous?**
Yes. Listeners run in registration order before `emit` returns, and an unhandled `'error'` event throws.

**Q44. What does `MaxListenersExceededWarning` usually indicate?**
A listener leak: code adds a listener per request or per call without removing it.

**Q45. How do you verify a webhook signature?**
HMAC-SHA256 over the raw body with the shared secret, compared to the header with `timingSafeEqual` after a length check, plus a timestamp window and dedupe by event id.

**Q46. Why must signature checks use the raw body?**
The provider signed the exact bytes; parsing and re-serializing JSON changes whitespace and key order, so the HMAC no longer matches.

**Q47. How should passwords be hashed?**
With a slow salted KDF: argon2id, bcrypt or the built-in scrypt. Never with fast hashes like SHA-256.

**Q48. `spawn` vs `exec`?**
`spawn` streams output and uses no shell by default; `exec` buffers output and runs through a shell, so user input in the command string allows injection.

**Q49. worker_threads vs cluster?**
Worker threads run CPU-heavy JavaScript in the same process with message passing; cluster forks processes that share a server port to use every core.

**Q50. Why doesn't try/catch catch an error thrown in a callback?**
The callback runs later on a new call stack, after the try block has finished.

**Q51. What happens to an unhandled promise rejection in Node 15+?**
It is raised as an uncaught exception and crashes the process, unless an `unhandledRejection` handler is registered.

**Q52. What should an `uncaughtException` handler do?**
Log the error with context and exit with a non-zero code; the process state may be corrupt, so the orchestrator should start a fresh one.

**Q53. Operational vs programmer errors?**
Operational: expected failures in correct code (bad input, timeouts, a partner's 503), handled with 4xx responses or retries. Programmer: bugs (TypeErrors, broken invariants), fixed in code; crash and restart is acceptable.

**Q54. Steps of a graceful shutdown?**
On SIGTERM: fail readiness, `server.close()`, close idle keep-alive connections, wait for in-flight requests, close DB pools, Redis and queue workers, exit; force exit after a timeout.

**Q55. How do you put a timeout on `fetch`?**
`fetch(url, { signal: AbortSignal.timeout(5000) })`; combine with a request-scoped signal using `AbortSignal.any`.

**Q56. Does `fetch` reject on a 404 or 500?**
No. It rejects only on network failures and aborts; check `res.ok`.

## Express (see [03-express.md](03-express.md))

**Q57. What is Express middleware?**
A function `(req, res, next)` that can modify the request or response, end the cycle, or pass control with `next()`.

**Q58. How does Express identify error-handling middleware?**
By its four parameters: `(err, req, res, next)`.

**Q59. Where should the error handler be registered?**
After all routes and other middleware.

**Q60. How did Express 4 handle a rejected promise from an async handler?**
It did not: the request hung and Node logged an unhandled rejection (crashing Node 15+), so teams wrapped handlers with `.catch(next)` or used `express-async-errors`.

**Q61. What changed for async errors in Express 5?**
Rejected promises from handlers and middleware are forwarded to error middleware automatically.

**Q62. Name two Express 5 routing changes.**
Wildcards must be named (`/*splat`, or `/{*splat}` to include the root) and optional parts use braces (`/:file{.:ext}`).

**Q63. Why must `/users/me` be registered before `/users/:id`?**
Routes match in order, so `/users/:id` would capture `me` as an id.

**Q64. What causes "Cannot set headers after they are sent"?**
Responding twice, usually a missing `return` after `res.json()` or calling `next()` after sending.

**Q65. App-level vs router-level vs route-level middleware?**
`app.use()` applies to every request (or a prefix), `router.use()` to one router's routes, and extra handlers in a route definition to that route only.

**Q66. How do you validate request bodies in Express?**
Zod schemas in middleware or at the top of the controller; invalid input becomes a 422 with issues, valid input is replaced by the parsed value.

**Q67. Why separate controllers, services and repositories?**
Controllers translate HTTP, services hold business rules, repositories own queries; each is testable alone and services are reusable from jobs.

**Q68. What does `helmet()` add?**
Security headers such as CSP, HSTS, `X-Content-Type-Options: nosniff` and frame protection, and it removes `X-Powered-By`.

**Q69. What does `app.set('trust proxy', 1)` do?**
Trusts one proxy hop so `req.ip` and `req.protocol` come from `X-Forwarded-*` headers, which rate limiting and secure cookies depend on.

**Q70. How do you test an Express app without starting a server?**
Export `createApp()` without `listen` and call it with supertest (`request(app).get('/health')`) inside Vitest or Jest.

## Hono (see [04-hono.md](04-hono.md))

**Q71. What is Hono?**
A small TypeScript web framework built on Web Standard `Request`/`Response`, running on Workers, Deno, Bun, Node, Lambda and Vercel.

**Q72. How do you run a Hono app on Node?**
`serve({ fetch: app.fetch, port: 3000 })` from `@hono/node-server`.

**Q73. How does Hono middleware differ from Express middleware?**
It is `async (c, next)` and forms an onion: code after `await next()` runs on the way out and can read or change the response.

**Q74. How do you share typed data between middleware and handlers?**
Declare `Variables` in the app's generic or `createMiddleware<...>`, then `c.set('user', u)` and `c.get('user')`.

**Q75. How do you validate a JSON body in Hono?**
`zValidator('json', schema)` from `@hono/zod-validator`, then read `c.req.valid('json')`.

**Q76. How do you return errors in Hono?**
Throw `HTTPException(status, { message })` and map everything centrally in `app.onError`; handle unmatched routes with `app.notFound`.

**Q77. What is Hono RPC?**
Export the type of your chained routes and create a typed client with `hc<AppType>(url)`: paths, inputs and responses are inferred without codegen.

**Q78. The most common Hono RPC mistake?**
Registering routes as separate statements instead of chaining them, so the exported type contains no routes.

**Q79. How do you test Hono routes?**
`await app.request('/path', init, env)` returns a `Response` without starting a server.

**Q80. Where do Workers secrets come from in Hono?**
`c.env`, typed through the `Bindings` generic; on Node, read validated `process.env` or use `env(c)` from `hono/adapter`.

## oRPC and type-safe APIs (see [05-orpc-and-typesafe-apis.md](05-orpc-and-typesafe-apis.md))

**Q81. What does end-to-end type safety mean?**
The client's types are derived from the server (router type or shared contract), so API mismatches fail at compile time; runtime validation still protects the boundary.

**Q82. What is oRPC?**
A TypeScript RPC framework: procedures with input/output schemas and handlers, plain-object routers, typed middleware context, typed errors, a codegen-free typed client, OpenAPI generation and TanStack Query utilities.

**Q83. How is an oRPC procedure defined (v1 style)?**
`os.input(schema).output(schema).handler(async ({ input, context }) => result)`, often starting from a base with `$context<T>()` and middleware applied.

**Q84. Initial vs execution context?**
Initial context is passed by the caller (`handler.handle(req, { context })`) and typed with `$context<T>()`; execution (injected) context is added by middleware through `next({ context })`.

**Q85. How do you scope oRPC procedures to an org?**
Middleware verifies the session, loads the membership for the session's active org, throws `FORBIDDEN` if missing and injects `{ org }`; handlers filter every query by `context.org.id`.

**Q86. How do typed errors work in oRPC?**
Declare `.errors({ CONFLICT: { data: schema } })`, throw `errors.CONFLICT({ data })`, and on the client use `safe()` plus `isDefinedError()` to get the typed code and data.

**Q87. What happens if a handler returns data that fails its output schema?**
The client receives an internal server error instead of malformed or leaked data.

**Q88. oRPC vs tRPC in two sentences?**
Both infer types end to end without codegen. tRPC splits queries, mutations and subscriptions and relies on adapters for OpenAPI; oRPC has one procedure kind, first-class OpenAPI and contract-first support, typed error maps and native `Date`/`File` handling.

**Q89. How do non-TypeScript clients, such as a Rails app, call an oRPC backend?**
Through OpenAPI routes: add route metadata to procedures, serve them with `OpenAPIHandler` and publish a generated spec they can build clients from.

**Q90. How do you use oRPC with TanStack Query?**
`createTanstackQueryUtils(client)`, then `useQuery(orpc.x.list.queryOptions({ input }))`, `useMutation(orpc.x.create.mutationOptions())`, and invalidate with `orpc.x.key()`.

**Q91. What is contract-first development?**
Defining the API (names, input and output schemas) in a shared contract with `oc`, implementing it with `implement(contract)` so the compiler enforces it, and typing clients from the contract.

**Q92. How do you test an oRPC procedure?**
`call(procedure, input, { context })` with fake sessions, against a real test database for queries, asserting results and error codes.

## Production APIs (see [06-production-apis.md](06-production-apis.md))

**Q93. Sessions vs JWT in one line each?**
Sessions: a random id in a cookie pointing at server-side state, instantly revocable. JWT: a signed stateless token, verified without a lookup but hard to revoke, so keep it short-lived.

**Q94. Why rotate refresh tokens?**
A stolen refresh token works at most once; reuse of an already-rotated token reveals theft and revokes the whole family.

**Q95. Which cookie flags protect a session cookie?**
`HttpOnly`, `Secure`, `SameSite=Lax` or `Strict`, a narrow `Path`, and ideally the `__Host-` prefix.

**Q96. What is IDOR?**
Loading a record by a client-supplied id without checking that the caller may access it; prevented by scoping queries to the caller's org or ownership.

**Q97. 403 or 404 for another tenant's resource?**
404, so you do not confirm that the resource exists.

**Q98. What belongs in an error response body?**
A stable machine code, a human message, optional field details and the request id; never stack traces.

**Q99. Why structured logs with request ids?**
JSON logs are searchable and aggregatable; a request id ties together every line, trace and downstream call of one request.

**Q100. What are RED metrics?**
Rate, errors and duration per endpoint: the core signals for alerting on a request-driven service.

**Q101. How do you rate limit with several instances?**
Shared counters in Redis keyed by IP, user, org or API key, with 429 and `Retry-After`.

**Q102. How many DB connections can you open?**
Instances times pool size, plus workers and tools, must stay below the database limit (Postgres defaults to 100); use a pooler for serverless.

**Q103. Why must job handlers be idempotent?**
Queues deliver at least once, so retries and duplicates happen; handlers must produce the same result when run twice.

**Q104. What is the transactional outbox pattern?**
Write the event to an outbox table in the same transaction as the data change; a relay publishes it afterwards, so events are never lost or sent for rolled-back data.

**Q105. Why presigned URLs for uploads?**
Files go straight from the client to S3, so the API avoids large bodies, memory spikes and timeouts while still controlling who uploads what.

**Q106. Liveness vs readiness?**
Liveness checks the process is not stuck (failure restarts it); readiness checks it can serve traffic (failure removes it from the load balancer).

**Q107. Why set `keepAliveTimeout` above the load balancer's idle timeout?**
Otherwise Node closes idle connections the balancer still considers open, and the next request on them fails with a 502.

## Spot the bug

**Q108. What's wrong?**
```ts
app.get('/listings/:id', async (req, res) => {
  const listing = await db.select().from(listings).where(eq(listings.id, req.params.id));
  res.json(listing);
});
```
No tenant or ownership check: any authenticated user can read any listing (IDOR). Scope by `orgId` from the verified session, and return 404 when nothing matches.

**Q109. What's wrong?**
```ts
async function fail() { throw new Error('boom'); }
try {
  fail();
} catch {
  console.log('caught');
}
```
`fail()` returns a rejected promise; without `await` the try block cannot catch it, so the rejection is unhandled and crashes Node 15+. Use `await fail()` inside the try.

**Q110. What's wrong?**
```ts
app.use(express.json());
app.post('/webhooks/payments', express.raw({ type: 'application/json' }), verifyAndHandle);
```
The global JSON parser consumes the body first, so the HMAC is computed over re-serialized data and never matches. Register the raw parser on the webhook route before `express.json()`.

**Q111. What's wrong?**
```ts
export async function GET() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const { rows } = await pool.query('SELECT * FROM listings');
  return Response.json(rows);
}
```
A new pool per request exhausts database connections and never closes them. Create one pool per process at module scope (or a cached client in serverless) and reuse it.

**Q112. What's wrong?**
```ts
const user = jwt.decode(req.get('authorization')!.slice(7));
if (user) next();
```
`jwt.decode` does not verify the signature, so anyone can forge a token. Use `jwt.verify` with the secret and a pinned `algorithms` list, and validate the claims.

**Q113. What's wrong?**
```ts
app.use((err: Error, req: Request, res: Response) => {
  res.status(500).json({ error: err.message });
});
```
Only three parameters, so Express treats it as normal middleware and never calls it for errors; it also leaks internal messages. Declare `(err, req, res, _next)` and return a generic message for unknown errors.
