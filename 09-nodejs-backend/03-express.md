# Express

> **Tier:** T1 (know cold) · **Related:** [Node.js fundamentals](01-nodejs-fundamentals.md), [Core modules, streams and errors](02-core-modules-streams-errors.md), [Hono](04-hono.md), [Production APIs](06-production-apis.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [Web security](../04-web-fundamentals/07-web-security-owasp.md), [Layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md)
> **On your resume:** Node.js backend skills (Hono, oRPC). Express is the reference framework interviewers use for Node questions; its middleware model maps directly onto Hono and onto the Rails controller and Rack middleware concepts you use at Devntech.

## Quick revision
- Express is a minimal framework over Node's `http` module: routing, middleware, and helpers on `req` and `res`.
- Middleware is `(req, res, next) => void`. It either ends the response or calls `next()`. Middleware runs in registration order, so order is behavior.
- `next(err)` jumps to error-handling middleware: functions with **four** parameters `(err, req, res, next)`, registered after the routes.
- Express 5 (stable since late 2024, Node 18+) forwards rejected promises from async handlers to error middleware. Express 4 needs a wrapper or `express-async-errors`.
- Express 5 also changed path syntax (named wildcards `/*splat`, optional parts in braces), leaves `req.body` `undefined` without a parser, and makes `req.query` a read-only getter using the "simple" parser.
- Body parsing is opt-in: `express.json({ limit })`, `express.urlencoded()`, `express.raw()` for webhooks, multer for `multipart/form-data`.
- `express.Router()` groups routes and middleware; mount it with `app.use('/api/listings', router)`.
- Register specific paths before parameterized ones: `/listings/mine` before `/listings/:id`.
- Validate `req.body`, `req.query` and `req.params` with Zod at the edge; TypeScript types do not exist at runtime.
- Layers: routes (URL to handler), controllers (HTTP in and out), services (business rules), repositories (data access).
- JWT middleware: read `Authorization: Bearer`, `jwt.verify` with a pinned algorithm, validate the claims, set `req.user`; 401 when unauthenticated, 403 when the role is wrong.
- Security baseline: `helmet()`, a `cors()` allowlist, `express-rate-limit` (Redis store across instances), body size limits, `app.set('trust proxy', 1)` behind a load balancer, no stack traces in responses.
- `express.static` serves files; use a CDN or reverse proxy in front of it in production.
- Test with supertest against the app object (no `listen`). Split `app.ts` (build the app) from `server.ts` (listen and shut down).
- "Cannot set headers after they are sent" means something responded twice, usually a missing `return` after `res.json()`.
- Alternatives: Fastify (fast, JSON Schema validation, plugins), Hono (Web Standards, multi-runtime, typed RPC), NestJS (modules, dependency injection, decorators).

## 1. The smallest useful app

```ts
// src/app.ts: builds the app but never listens, so tests can import it
import express from 'express';

export function createApp() {
  const app = express();
  app.use(express.json({ limit: '100kb' }));          // parse JSON bodies into req.body

  app.get('/health', (_req, res) => {
    res.json({ ok: true });
  });

  app.get('/listings/:id', async (req, res) => {
    const listing = await findListing(req.params.id);  // Express 5: a rejection here reaches the error handler
    if (!listing) {
      res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Listing not found' } });
      return;                                          // always return after responding
    }
    res.json({ data: listing });
  });

  return app;
}
declare function findListing(id: string): Promise<{ id: string; title: string } | null>;
```

```ts
// src/server.ts
import { createApp } from './app';
import { env } from './config/env';

const server = createApp().listen(env.PORT, () => console.log(`API listening on ${env.PORT}`));
// then register SIGTERM handling (see the graceful shutdown section in core modules)
```

## 2. Routing

```ts
// src/modules/listings/listings.routes.ts
import { Router } from 'express';

export const listingsRouter = Router();

listingsRouter.use(requireAuth);                                    // router-level: every route below
listingsRouter.get('/', validateQuery(ListQuery), listingsController.list);
listingsRouter.post('/', requireRole('owner', 'admin'), validateBody(CreateListing), listingsController.create);
listingsRouter.get('/mine', listingsController.mine);               // before '/:id', or 'mine' matches as an id
listingsRouter.get('/:id', listingsController.show);
listingsRouter.patch('/:id', validateBody(UpdateListing), listingsController.update);
listingsRouter.delete('/:id', requireRole('owner', 'admin'), listingsController.remove);

// src/app.ts
// app.use('/api/listings', listingsRouter);
```

- Path params arrive as strings in `req.params`. Query values can be strings, arrays or (with the extended parser) objects, so validate them.
- `Router({ mergeParams: true })` lets a nested router read parent params, for `/orgs/:orgId/listings`.
- `app.route('/listings').get(list).post(create)` chains handlers for one path.

| Express 4 path | Express 5 path |
|---|---|
| `/files/*` | `/files/*splat` (`req.params.splat` is an array of segments) |
| `/*` including the root | `/{*splat}` |
| `/:file.:ext?` | `/:file{.:ext}` |
| regex-like strings such as `/users?` | an array of paths: `['/user', '/users']` |

## 3. req and res essentials

| `req` | Meaning | `res` | Meaning |
|---|---|---|---|
| `req.params` | path params (strings) | `res.status(201).json(obj)` | status plus JSON body |
| `req.query` | query string values | `res.send(body)` | string, Buffer or object |
| `req.body` | parsed body (needs a parser) | `res.sendStatus(204)` | status with a default body |
| `req.get('authorization')` | header, case-insensitive | `res.set('Cache-Control', 'no-store')` | set a header |
| `req.ip` | client IP (needs `trust proxy` behind a proxy) | `res.cookie(name, value, options)` | set a cookie |
| `req.method`, `req.path`, `req.originalUrl` | request line | `res.redirect(302, '/login')` | redirect (status first in Express 5) |
| `req.cookies` | parsed cookies (needs `cookie-parser`) | `res.locals` | per-request data for later middleware |

## 4. The middleware model

A request travels through a stack of functions. Each one can read or modify `req` and `res`, end the response, or call `next()` to pass control on. A middleware that does neither leaves the request hanging until a timeout.

```text
request -> helmet -> cors -> express.json -> requestContext -> router: requireAuth -> validate -> handler -> response
                                                   any step: next(err) -> errorHandler -> response
```

```ts
// src/middleware/request-context.ts: request id plus timing
import { randomUUID } from 'node:crypto';
import type { RequestHandler } from 'express';

export const requestContext: RequestHandler = (req, res, next) => {
  const id = req.get('x-request-id') ?? randomUUID();
  res.locals.requestId = id;
  res.set('x-request-id', id);
  const start = process.hrtime.bigint();
  res.on('finish', () => {                          // fires after the response has been handed to the OS
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    console.log(JSON.stringify({ id, method: req.method, path: req.originalUrl, status: res.statusCode, ms }));
  });
  next();
};
```

Where middleware can live:
- **Application-level:** `app.use(mw)` for every request, `app.use('/api', mw)` for a path prefix.
- **Router-level:** `router.use(mw)`.
- **Route-level:** `router.post('/', mw1, mw2, handler)`.
- **Error-handling:** four parameters, registered last.
- **Built-in:** `express.json`, `express.urlencoded`, `express.raw`, `express.text`, `express.static`.
- **Third-party:** `helmet`, `cors`, `morgan` (request logs), `cookie-parser`, `compression`, `multer`, `express-rate-limit`.

A typical order:

```ts
const app = express();
app.set('trust proxy', 1);                          // one proxy or load balancer in front: correct req.ip
app.use(helmet());
app.use(cors({ origin: env.CORS_ORIGINS, credentials: true }));
app.post('/webhooks/payments', express.raw({ type: 'application/json' }), paymentsWebhook); // before json()
app.use(express.json({ limit: '100kb' }));
app.use(requestContext);
app.use(morgan('combined'));                        // or pino-http for structured JSON logs
app.use('/api/auth', authRateLimiter);
app.use('/api/listings', listingsRouter);
app.use(notFoundHandler);                           // nothing matched
app.use(errorHandler);                              // always last
```

Classic order bugs: auth registered after the router (routes unprotected), `express.json()` after the routes (`req.body` undefined), the 404 handler before the routes (everything 404s), the error handler first (never reached), and a global `express.json()` in front of a webhook route (raw body gone).

## 5. Error handling

```ts
// src/middleware/errors.ts
import type { ErrorRequestHandler, RequestHandler } from 'express';
import { ZodError } from 'zod';
import { AppError } from '../lib/errors';
import { logger } from '../lib/logger';

export const notFoundHandler: RequestHandler = (req, res) => {
  res.status(404).json({ error: { code: 'ROUTE_NOT_FOUND', message: `No route for ${req.method} ${req.path}` } });
};

export const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) return next(err);            // failed mid-response: let Express close the connection
  const requestId = res.locals.requestId as string | undefined;

  if (err instanceof ZodError) {
    res.status(422).json({ error: { code: 'VALIDATION', issues: err.issues, requestId } });
  } else if (err instanceof AppError) {
    res.status(err.statusCode).json({ error: { code: err.code, message: err.message, requestId } });
  } else if (err?.type === 'entity.parse.failed') { // malformed JSON from express.json()
    res.status(400).json({ error: { code: 'INVALID_JSON', requestId } });
  } else {
    logger.error({ err, requestId, path: req.originalUrl }, 'unhandled error');
    res.status(500).json({ error: { code: 'INTERNAL', message: 'Something went wrong', requestId } });
  }
};
```

Express recognizes error middleware by its parameter count (`fn.length === 4`). Drop the unused `next` to please a linter and it silently becomes normal middleware; name it `_next` instead. `AppError` is the custom error class from [core modules](02-core-modules-streams-errors.md).

**Async errors, Express 4 vs 5.** Synchronous throws inside handlers were always caught. Rejected promises were not:

```ts
// Express 4: this rejection is never seen by Express; the request hangs and Node 15+ crashes on it
app.get('/listings', async (_req, res) => {
  res.json(await listingsRepo.listAll());
});

// Express 4 fix: forward rejections to next()
const asyncHandler = (fn: RequestHandler): RequestHandler => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};
app.get('/listings', asyncHandler(async (_req, res) => {
  res.json(await listingsRepo.listAll());
}));
```

Express 5 calls `next(err)` for a rejected handler promise itself. Errors thrown later inside callbacks or timers (not part of the returned promise) are still uncaught in both versions.

## 6. Validation with Zod

```ts
// src/middleware/validate.ts
import type { RequestHandler } from 'express';
import type { ZodType } from 'zod';

export const validateBody = (schema: ZodType): RequestHandler => (req, _res, next) => {
  const result = schema.safeParse(req.body);
  if (!result.success) return next(result.error);   // ZodError becomes a 422 in errorHandler
  req.body = result.data;                           // coerced, defaults applied, unknown keys stripped
  next();
};

export const validateQuery = (schema: ZodType): RequestHandler => (req, res, next) => {
  const result = schema.safeParse(req.query);
  if (!result.success) return next(result.error);
  res.locals.query = result.data;                   // Express 5: req.query is a read-only getter
  next();
};
```

```ts
// src/modules/listings/listings.schemas.ts
import { z } from 'zod';

const Category = z.enum(['room', 'studio', 'service']);

export const CreateListing = z.object({
  title: z.string().trim().min(3).max(120),
  priceCents: z.number().int().positive(),
  category: Category,
});
export type CreateListingInput = z.infer<typeof CreateListing>;

export const ListQuery = z.object({
  page: z.coerce.number().int().min(1).default(1),  // query values arrive as strings
  category: Category.optional(),
});
```

- Why validate when you have TypeScript: types are erased at runtime, and clients can send anything.
- `z.object` strips unknown keys by default, which blocks mass assignment (`{ "role": "admin" }` smuggled into a signup body).
- Middleware validation leaves `req.body` typed as `any`. Many teams parse inside the controller instead (`const input = CreateListing.parse(req.body)`), so the inferred type flows on and a thrown `ZodError` still reaches the error handler.

## 7. Layering: routes, controllers, services, repositories

| Layer | Knows about | Does | Must not |
|---|---|---|---|
| Route | URLs, middleware | connects a path and its middleware to a controller | contain logic |
| Controller | `req`, `res` | reads validated input, calls a service, picks status codes | run SQL or business rules |
| Service | the domain | business rules, authorization rules, transactions, side effects | touch `req` or `res` |
| Repository | the database | queries (Drizzle or SQL), tenant scoping | know about HTTP |

```ts
// listings.repository.ts (imports omitted)
export const listingsRepo = {
  findById: async (orgId: string, id: string) => {
    const [row] = await db.select().from(listings)
      .where(and(eq(listings.orgId, orgId), eq(listings.id, id)));
    return row ?? null;                       // another org's listing looks exactly like a missing one
  },
  create: async (orgId: string, input: CreateListingInput) => {
    const [row] = await db.insert(listings).values({ ...input, orgId }).returning();
    return row!;
  },
};

// listings.service.ts
export const listingsService = {
  async get(user: AuthUser, id: string) {
    const listing = await listingsRepo.findById(user.orgId, id);
    if (!listing) throw new NotFoundError('Listing');
    return listing;
  },
  async create(user: AuthUser, input: CreateListingInput) {
    // business rules live here: plan limits, duplicate checks, enqueueing jobs, audit logs
    return listingsRepo.create(user.orgId, input);
  },
};

// listings.controller.ts
export const listingsController = {
  async show(req: Request, res: Response) {
    res.json({ data: await listingsService.get(req.user!, req.params.id) });
  },
  async create(req: Request, res: Response) {
    const listing = await listingsService.create(req.user!, req.body as CreateListingInput);
    res.status(201).location(`/api/listings/${listing.id}`).json({ data: listing });
  },
};
```

```text
src/
  app.ts  server.ts
  config/env.ts                      Zod-validated environment
  lib/                               logger, errors, db client
  middleware/                        auth, validate, errors, request-context
  modules/listings/                  routes, controller, service, repository, schemas, tests
```

Feature folders (`modules/listings`) scale better than global `controllers/` and `services/` folders. This is the same separation as Rails service objects and query objects; see [layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md) and [clean architecture](../13-architecture-and-practices/02-clean-architecture.md).

## 8. JWT authentication middleware

```ts
// src/middleware/auth.ts
import type { RequestHandler } from 'express';
import jwt from 'jsonwebtoken';
import { z } from 'zod';
import { env } from '../config/env';
import { AppError } from '../lib/errors';

type Role = 'owner' | 'admin' | 'member';
const Claims = z.object({ sub: z.string(), orgId: z.string(), role: z.enum(['owner', 'admin', 'member']) });

export const requireAuth: RequestHandler = (req, _res, next) => {
  const header = req.get('authorization');
  const token = header?.startsWith('Bearer ') ? header.slice(7) : undefined;
  if (!token) return next(new AppError('Missing token', 401, 'UNAUTHENTICATED'));
  try {
    const payload = jwt.verify(token, env.JWT_SECRET, { algorithms: ['HS256'] }); // pin the algorithm
    const claims = Claims.parse(payload);           // the token body is input too
    req.user = { id: claims.sub, orgId: claims.orgId, role: claims.role };
    next();
  } catch {
    next(new AppError('Invalid or expired token', 401, 'UNAUTHENTICATED'));
  }
};

export const requireRole = (...roles: Role[]): RequestHandler => (req, _res, next) => {
  if (!req.user || !roles.includes(req.user.role)) return next(new AppError('Forbidden', 403, 'FORBIDDEN'));
  next();
};
```

```ts
// src/types/express.d.ts: teach TypeScript about req.user
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; orgId: string; role: 'owner' | 'admin' | 'member' };
    }
  }
}
export {};

// Issuing a short-lived access token at login:
// jwt.sign({ orgId, role }, env.JWT_SECRET, { subject: user.id, expiresIn: '15m', algorithm: 'HS256' })
```

- A JWT is signed, not encrypted: anyone can read the payload, so keep it small and non-sensitive.
- Browsers: prefer an httpOnly cookie over `localStorage` (then add CSRF protection). Mobile apps: a Bearer header with the token in secure storage.
- `jose` is an alternative library that also runs on edge runtimes. Refresh tokens, sessions vs JWT and revocation are in [production APIs](06-production-apis.md).

## 9. File uploads with multer

```ts
import { randomUUID } from 'node:crypto';
import multer from 'multer';

const upload = multer({
  storage: multer.memoryStorage(),                  // req.file.buffer: only for small files
  limits: { fileSize: 5 * 1024 * 1024, files: 1 },  // larger files fail with a MulterError (LIMIT_FILE_SIZE)
  fileFilter: (_req, file, cb) => {
    cb(null, ['image/jpeg', 'image/png', 'image/webp'].includes(file.mimetype)); // false skips the file
  },
});

router.post('/me/avatar', requireAuth, upload.single('avatar'), async (req, res) => {
  if (!req.file) {
    res.status(422).json({ error: { code: 'INVALID_FILE' } });
    return;
  }
  const key = `avatars/${req.user!.id}/${randomUUID()}`;
  await storage.put(key, req.file.buffer, req.file.mimetype);   // your S3 wrapper
  res.status(201).json({ data: { key } });
});
```

- `file.mimetype` comes from the client. For anything security-sensitive, check the file's magic bytes too.
- Disk storage avoids holding files in memory but needs cleanup and does not work across instances.
- For large files, skip the API entirely: hand the client a presigned S3 URL ([production APIs](06-production-apis.md)).
- Map `MulterError` with code `LIMIT_FILE_SIZE` to `413 Payload Too Large` in the error handler.

## 10. Static files

```ts
import path from 'node:path';

app.use('/assets', express.static(path.join(import.meta.dirname, '..', 'public'), { maxAge: '7d', immutable: true }));
// SPA fallback, registered after the API routes (Express 5 syntax):
app.get('/{*splat}', (_req, res) => res.sendFile(path.join(import.meta.dirname, '..', 'public', 'index.html')));
```

Express 5 ignores dotfiles by default (`/.well-known` needs `dotfiles: 'allow'`). In production a CDN or Nginx serves static assets far more efficiently than Node.

## 11. Security checklist

```ts
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';
import helmet from 'helmet';

app.use(helmet());                                 // CSP, HSTS, nosniff, frame protection; removes X-Powered-By
app.use(cors({ origin: ['https://app.example.com'], credentials: true })); // allowlist, never '*' with cookies
const authRateLimiter = rateLimit({ windowMs: 15 * 60 * 1000, limit: 20, standardHeaders: 'draft-7', legacyHeaders: false });
app.use(express.json({ limit: '100kb' }));
```

| Threat | Mitigation |
|---|---|
| Missing security headers | `helmet()` |
| Brute force and credential stuffing | rate limits on auth routes, with a Redis store when several instances run (older versions name `limit` as `max`) |
| Wrong client IP behind a proxy | `app.set('trust proxy', 1)` with the real number of proxies, so attackers cannot spoof `X-Forwarded-For` |
| Mass assignment, SQL injection | Zod schemas (strip unknown keys), parameterized queries or an ORM, never spread `req.body` into queries |
| NoSQL operator injection | validate types: `{ "email": { "$ne": null } }` must fail validation |
| Huge payloads | body and upload limits, server timeouts |
| Information leaks | generic 500 messages, no stack traces, `NODE_ENV=production` |
| CORS mistakes | explicit allowlist; credentials only when needed |
| CSRF with cookie auth | `SameSite` cookies plus CSRF tokens for state-changing requests |
| Vulnerable dependencies | `npm audit`, Renovate or Dependabot |

More in [web security](../04-web-fundamentals/07-web-security-owasp.md).

## 12. Testing with supertest and Vitest

```ts
// src/modules/listings/listings.test.ts
import request from 'supertest';
import { afterEach, describe, expect, it, vi } from 'vitest';
import { createApp } from '../../app';
import { makeListing, signTestToken } from '../../test/helpers';
import { listingsService } from './listings.service';

describe('listings API', () => {
  const app = createApp();
  afterEach(() => vi.restoreAllMocks());

  it('returns 401 without a token', async () => {
    const res = await request(app).get('/api/listings/l1');
    expect(res.status).toBe(401);
    expect(res.body.error.code).toBe('UNAUTHENTICATED');
  });

  it('returns 422 for an invalid body', async () => {
    const res = await request(app)
      .post('/api/listings')
      .set('Authorization', `Bearer ${signTestToken({ role: 'admin' })}`)
      .send({ title: 'x' });
    expect(res.status).toBe(422);
  });

  it('returns the listing', async () => {
    vi.spyOn(listingsService, 'get').mockResolvedValue(makeListing({ id: 'l1', title: 'Room' }));
    const res = await request(app).get('/api/listings/l1').set('Authorization', `Bearer ${signTestToken({ role: 'member' })}`);
    expect(res.status).toBe(200);
    expect(res.body.data.title).toBe('Room');
  });
});
```

- supertest binds the app to an ephemeral port itself, so `createApp()` without `listen` is all it needs.
- Unit-test services without HTTP; test routes for status codes, validation and auth; run integration tests against a real test database (Testcontainers) instead of mocking the ORM. Strategy details: [production APIs](06-production-apis.md) and [testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md).

## 13. Express vs Fastify vs Hono vs NestJS

| | Express | Fastify | Hono | NestJS |
|---|---|---|---|---|
| Style | minimal, unopinionated | plugins, schema-first | minimal, Web Standards | opinionated: modules, DI, decorators |
| Request/response | Node `req`/`res` with helpers | wrapped Node objects | Web `Request`/`Response` | Express or Fastify underneath |
| Runtimes | Node | Node | Node, Bun, Deno, Workers, Lambda, Vercel | Node |
| Validation | bring your own (Zod) | JSON Schema built in (Ajv), type providers | validator middleware (Zod) | pipes with class-validator (or Zod) |
| Async errors | Express 5 forwards rejections | native | native | native, exception filters |
| Performance | slowest of the four in typical benchmarks | fast | fast, tiny | depends on the adapter |
| TypeScript | via `@types/express` | good | first-class, typed RPC client | first-class |
| Pick when | existing codebases, huge middleware ecosystem | high-throughput Node APIs | edge or serverless, multi-runtime, typed clients | large teams that want enforced structure |

## Common pitfalls
- No `return` after `res.json()`, so execution continues and responds again ("Cannot set headers after they are sent").
- Middleware that neither responds nor calls `next()`, leaving requests hanging.
- Error middleware with three parameters, or registered before the routes.
- Async handlers in Express 4 without a wrapper.
- Upgrading to Express 5 without rewriting `*` wildcards or code that assigns to `req.query`.
- Trusting `req.query` types instead of validating and coercing.
- `cors({ origin: '*', credentials: true })`, or reflecting any `Origin` with credentials.
- Missing `trust proxy` behind a load balancer, so the rate limiter sees one IP for every user.
- In-memory rate limits or sessions while running several instances.
- `multer.memoryStorage()` for large files, and trusting the client's mimetype.
- Business logic and SQL in controllers or route files, which makes code hard to test and reuse.
- Sending internal error messages or stack traces to clients.

## How to talk about this in an interview
- Be honest about where your hands-on Node work is: your resume lists Node.js with Hono and oRPC, plus years of Rails. Express is the shared vocabulary, so show you understand its model and map it: Express middleware is like Rack middleware plus Rails `before_action`, error middleware is like `rescue_from`, `Router` is like `namespace` in `routes.rb`, and services and repositories are the same idea as Rails service objects.
  > **Fill in:** any Express project you built (university labs, side projects, internship work). If there is none, say so plainly and explain how the same concepts appear in Hono.
- Show maturity with specifics: the middleware order you use, why error middleware has four parameters, what changed in Express 5, how you validate and where authorization lives.
- When comparing frameworks, connect to [Hono](04-hono.md): same mental model, but Web Standard `Request`/`Response`, onion-style middleware and typed RPC.

## Interview questions

**Q1. What is middleware in Express, and how does `next` work?**
A function `(req, res, next)` in the request pipeline. It can read or change `req` and `res`, end the response, or call `next()` to hand control to the next matching middleware. `next(err)` skips ahead to error-handling middleware.
*Follow-up:* What if a middleware does neither? The request hangs until a client or server timeout.

**Q2. Why does middleware order matter? Give an example bug.**
Express runs middleware in registration order. Registering `requireAuth` after a router leaves those routes unprotected; putting `express.json()` after the routes leaves `req.body` undefined; a 404 handler before the routes answers everything with 404.
*Follow-up:* Where does the error handler go? After all routes and other middleware.

**Q3. How does Express recognize error-handling middleware? Write a basic one.**
By arity: exactly four parameters `(err, req, res, next)`. A minimal one checks `res.headersSent` (and delegates to `next(err)` if so), maps known error classes to status codes, logs unknown errors and returns a generic 500 JSON body.
*Follow-up:* Why might a linter break it? Removing the unused `next` changes the arity, so Express treats it as normal middleware. Name it `_next`.

**Q4. How are async errors handled in Express 4 vs Express 5?**
Express 4 only catches synchronous throws; a rejected promise from an async handler goes unhandled (hanging request, and a crash on Node 15+), so you wrap handlers to call `.catch(next)` or use `express-async-errors`. Express 5 forwards rejected promises to `next(err)` automatically.
*Follow-up:* Is everything caught in Express 5? No: errors thrown later inside callbacks or timers are outside the returned promise.

**Q5. What is `express.Router` for, and where can middleware be attached?**
A mini-app that groups routes and middleware for one feature, mounted under a prefix with `app.use('/api/listings', router)`. Middleware can be application-level (`app.use`), router-level (`router.use`), route-level (extra handlers before the final one), or error-handling.
*Follow-up:* How does a nested router read `:orgId` from its parent path? `Router({ mergeParams: true })`.

**Q6. How do you validate request input?**
Define Zod schemas for body, query and params and validate in middleware or at the top of the controller. Invalid input becomes a 422 with the issues; valid input is replaced by the parsed value (coerced numbers, defaults, unknown keys stripped). TypeScript alone cannot do this because types are erased at runtime.
*Follow-up:* Why coerce query params? They always arrive as strings.

**Q7. How would you structure an Express API, and why separate controllers, services and repositories?**
Feature folders with routes, controller, service, repository and schemas. Controllers translate HTTP; services hold business rules and transactions; repositories own queries and tenant scoping. Each layer can be tested and changed alone, and services can be reused by jobs or other transports.
*Follow-up:* Where does authorization go? Coarse role checks in middleware; resource-level and tenant checks in the service and repository.

**Q8. Write a JWT auth middleware. How do you type `req.user`?**
Read the Bearer token, `jwt.verify` it with the secret and a pinned algorithm list, validate the claims with Zod, set `req.user` and call `next()`; on failure, `next` a 401 error. Type it by augmenting `Express.Request` in a `.d.ts` file with `declare global`.
*Follow-up:* Why pin `algorithms`? To block algorithm-confusion attacks such as `alg: none` or swapping asymmetric for symmetric keys.

**Q9. How do you handle file uploads in Express?**
Multer parses `multipart/form-data`; set `limits` and a `fileFilter`, use memory storage only for small files, and stream or hand off to object storage. For large files, give the client a presigned S3 URL so the bytes never pass through the API.
*Follow-up:* Can you trust `file.mimetype`? No: it is client-supplied; check magic bytes for sensitive cases.

**Q10. Which security middleware would you add, and why?**
`helmet` for security headers, a strict `cors` allowlist, `express-rate-limit` on auth and expensive routes (Redis-backed with multiple instances), body size limits, Zod validation, and a generic error handler that never leaks stack traces. Behind a proxy, set `trust proxy` correctly.
*Follow-up:* Does helmet stop XSS by itself? No: it adds defenses such as CSP; output encoding and input validation still matter.

**Q11. How do you configure CORS for a frontend on another domain that uses cookies?**
`cors({ origin: ['https://app.example.com'], credentials: true })`: an explicit origin (not `*`) and `Access-Control-Allow-Credentials: true`; the frontend sends `credentials: 'include'`. Cookies also need `SameSite=None; Secure` when the sites differ, plus CSRF protection.
*Follow-up:* Does CORS protect your API from curl? No: CORS is enforced by browsers only.

**Q12. What does `app.set('trust proxy', 1)` do, and why does it matter for rate limiting?**
It tells Express to trust one proxy hop, so `req.ip`, `req.protocol` and `req.secure` come from `X-Forwarded-*` headers set by your load balancer. Without it, every request appears to come from the load balancer and one shared limit applies to all users; trusting too much lets clients spoof their IP.
*Follow-up:* Why not `trust proxy: true`? It trusts any `X-Forwarded-For` value, including forged ones.

**Q13. What causes "Cannot set headers after they are sent to the client"?**
Responding twice: calling `res.json()` and then continuing to another `res.send()`, calling `next()` after responding, or an error handler trying to respond after a streamed response started. Return right after responding, and check `res.headersSent` in error handlers.
*Follow-up:* How do you find it quickly? The stack trace points at the second write; look for the missing `return` above it.

**Q14. How do you test an Express API?**
Export a `createApp()` without `listen`, then use supertest to make real HTTP calls in Vitest or Jest, asserting status codes, bodies and headers. Unit-test services directly; use a real test database for integration tests and mock only external HTTP services.
*Follow-up:* Why split `app.ts` and `server.ts`? Tests import the app without binding ports or running shutdown hooks.

**Q15. What changed in Express 5?**
Rejected promises reach error middleware; wildcards must be named (`/*splat`) and optional parts use braces; `req.body` is `undefined` without a parser; `req.query` is a read-only getter with the simple parser by default; `res.redirect(status, url)` argument order; removed APIs such as `app.del` and `req.param()`; dotfiles ignored by `express.static`; Node 18+.
*Follow-up:* Biggest migration risk? Route patterns and code that mutates `req.query`.

**Q16. Express vs Fastify vs Hono vs NestJS: how would you choose?**
Express for existing codebases and its ecosystem; Fastify for high-throughput Node APIs with schema validation; Hono for Web-Standard code that runs on Workers, Bun, Deno or Node with a typed RPC client; NestJS when a large team wants enforced structure and dependency injection. For a new TypeScript API on an edge or serverless platform, Hono (possibly with oRPC) is a strong default.
*Follow-up:* What do they all share? Routing, middleware or hooks, and the need for validation, auth, error mapping and tests.

## Further reading
- [Express: Using middleware](https://expressjs.com/en/guide/using-middleware.html)
- [Express: Error handling](https://expressjs.com/en/guide/error-handling.html)
- [Express: Migrating to Express 5](https://expressjs.com/en/guide/migrating-5.html)
- [Express: Production best practices, security](https://expressjs.com/en/advanced/best-practice-security.html)
