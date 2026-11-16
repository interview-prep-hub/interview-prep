# Production-Ready Node.js APIs

> **Tier:** T1 (know cold) · **Related:** [Express](03-express.md), [Hono](04-hono.md), [oRPC](05-orpc-and-typesafe-apis.md), [Core modules, streams and errors](02-core-modules-streams-errors.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [Web security](../04-web-fundamentals/07-web-security-owasp.md), [Redis](../12-databases/10-redis.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Docker](../14-devops-and-tools/03-docker.md), [Observability](../14-devops-and-tools/05-observability-and-debugging.md)
> **On your resume:** the TypeScript backend of the Devntech multi-tenant platform (org-scoped access control, Rails sync APIs), Docker and Redis in your skills, and the Sidekiq background-job experience from CrystalCommerce and Colochousing, which maps directly onto Node job queues.

## Quick revision
- Structure by feature; keep HTTP, business logic and data access in separate layers; create pools and clients once in a composition root and close them on shutdown.
- Validate configuration at boot with Zod and fail fast; secrets come from the platform, never from Git.
- Sessions: server-side state, a cookie holding a random id, instant revocation. JWTs: stateless signed tokens, hard to revoke, so keep them short-lived and pair them with rotating refresh tokens.
- Browser auth cookies: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict`), narrow `Path`; add CSRF protection for cookie-authenticated mutations.
- Refresh tokens are random, stored hashed, rotated on every use; reuse of an old one revokes the whole family.
- Prefer a maintained auth library (Better Auth; Auth.js for Next.js, maintained by the Better Auth team since 2025) over hand-rolled token logic.
- Authorization: roles map to permissions checked on the server, and every query is scoped to the caller's org. IDOR means trusting a client-supplied id without checking access.
- Errors: one envelope (or RFC 9457 problem details) with a stable machine code, message, details and request id; correct status codes; no stack traces.
- Logs: structured JSON via pino to stdout, a request id on every line (pino-http plus `AsyncLocalStorage`), secrets redacted.
- Metrics: RED (rate, errors, duration) per route plus event-loop delay and pool saturation; traces with OpenTelemetry across HTTP, DB and queues.
- Rate limiting: per IP for anonymous routes, per user, org or API key otherwise; Redis-backed with several instances; `429` with `Retry-After`.
- One DB pool per process; instances x pool size must stay under the database's connection limit; use a pooler for serverless.
- Background jobs: BullMQ (Redis) or pg-boss (Postgres); idempotent handlers, retries with backoff, enqueue after commit (outbox for guarantees).
- Redis caching: cache-aside with TTLs, org-prefixed keys, invalidation on writes, stampede protection.
- Uploads: presigned S3 URLs so bytes go straight from the client to S3; validate type and size; private buckets.
- Testing: unit (Vitest), integration against a real database (Testcontainers), API tests (supertest or `app.request`), a few end-to-end flows.
- OpenAPI docs generated from code (oRPC generator, Zod-based tools) so they never drift.
- Deploy stateless Docker containers: multi-stage build, non-root user, liveness and readiness probes, graceful shutdown, horizontal scaling.
- Behind a load balancer, set Node's `keepAliveTimeout` above the balancer's idle timeout to avoid random 502s.
- Serverless trades ops work for cold starts, execution limits and database connection pressure.

## 1. Project structure

```text
apps/api/src/
  server.ts              process entry: listen, register shutdown hooks
  app.ts                 builds the HTTP app (Hono or Express): middleware, routers, error mapping
  config/env.ts          Zod-validated config: the only file that reads process.env
  lib/                   logger, errors, db pool, redis, queue clients
  middleware/            request id, auth, org context, rate limit
  modules/<feature>/     routes or oRPC procedures, service, repository, schemas, tests
  jobs/                  job handlers and the worker entry point (deployed as its own process)
packages/                monorepo-shared code: db schema, API contract, UI
```

- Run two process types from one codebase: `api` (HTTP) and `worker` (jobs). They scale and restart independently.
- Create the DB pool, Redis client and queues once at startup and pass them down (context, constructor arguments). That makes tests and shutdown simple.
- Keep processes stateless: sessions, rate-limit counters, caches and files live in Postgres, Redis or S3.

## 2. Configuration and secrets

Parse `process.env` once with Zod and export a typed `env` (full example in [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md)). Then:

| Kind | Examples | Where it lives |
|---|---|---|
| Plain config | `PORT`, `LOG_LEVEL`, feature flags | environment variables with safe defaults |
| Secrets | `DATABASE_URL`, `JWT_SECRET`, partner API keys | the platform's secret store, injected as env vars; different per environment |
| Public client config | API base URL | build-time variables with a public prefix (never secrets) |

Rotate secrets, never log the environment, and keep `.env` out of Git (commit `.env.example`).

## 3. Authentication

| | Server-side sessions | JWT access tokens |
|---|---|---|
| What the client holds | a cookie with a random session id | a signed token with claims |
| Per request | look up the session (cache it) | verify the signature, no lookup |
| Revocation | delete the session: immediate | wait for expiry or keep a denylist (state again) |
| Best for | browser apps talking to their own backend | mobile clients, service-to-service, several APIs |
| Main risk | session store availability | long-lived tokens you cannot revoke; tokens in `localStorage` exposed to XSS |

Access and refresh tokens:

```text
login         -> access JWT (5-15 min) + refresh token (random, 7-30 days, stored hashed with its family id)
API call      -> Authorization: Bearer <access>   (or an HttpOnly cookie for browsers)
access expired-> POST /auth/refresh -> new access token + NEW refresh token; the old refresh token is revoked
old refresh token used again -> likely theft -> revoke the whole family, force a new login
logout        -> revoke the refresh token, clear cookies
```

```ts
res.cookie('refresh_token', refreshToken, {
  httpOnly: true,                       // JavaScript cannot read it, so XSS cannot steal it
  secure: true,                         // HTTPS only
  sameSite: 'lax',                      // not sent on cross-site POSTs, iframes or fetches
  path: '/auth/refresh',                // only sent where it is needed
  maxAge: 30 * 24 * 60 * 60 * 1000,     // Express takes milliseconds
});
```

| Flag | Effect |
|---|---|
| `HttpOnly` | invisible to `document.cookie` |
| `Secure` | sent over HTTPS only |
| `SameSite=Lax` | sent on top-level navigation, not on cross-site subrequests (the browser default) |
| `SameSite=Strict` | never sent cross-site, even when following a link |
| `SameSite=None; Secure` | required when frontend and API are on different sites; add CSRF defenses |
| `Path`, `Domain` | scope; omit `Domain` to keep the cookie host-only |
| `__Host-` name prefix | forces `Secure`, `Path=/` and no `Domain`, so subdomains cannot overwrite it |

Cookies are sent automatically, so cookie-authenticated mutations need CSRF protection: `SameSite`, plus a CSRF token or a required custom header. Bearer tokens are not sent automatically, so they are not exposed to CSRF (XSS remains a risk for both).

Use a maintained library rather than writing token rotation yourself. Better Auth is a framework-agnostic TypeScript library that stores sessions in your database, with plugins for organizations, two-factor auth and social login, and integrations for Hono, Express, Next.js and TanStack Start. Auth.js (formerly NextAuth) has been maintained by the Better Auth team since September 2025. Check each library's docs for current APIs. Hash passwords with argon2id or bcrypt, rate-limit logins, and return the same message for unknown emails and wrong passwords.

## 4. Authorization: roles, org scoping and IDOR

```ts
type Role = 'owner' | 'admin' | 'member';

const permissions = {
  owner: ['listing:read', 'listing:write', 'listing:delete', 'member:manage', 'billing:manage'],
  admin: ['listing:read', 'listing:write', 'listing:delete', 'member:manage'],
  member: ['listing:read', 'listing:write'],
} as const satisfies Record<Role, readonly string[]>;

export type Permission = (typeof permissions)[Role][number];
export const can = (role: Role, p: Permission) => (permissions[role] as readonly Permission[]).includes(p);

// Hono middleware; the org (with the caller's role in it) was set by an earlier org-context middleware
export const requirePermission = (p: Permission) =>
  createMiddleware<{ Variables: { org: { id: string; role: Role } } }>(async (c, next) => {
    if (!can(c.get('org').role, p)) throw new HTTPException(403, { message: 'Forbidden' });
    await next();
  });
```

**IDOR (insecure direct object reference)** is the most common multi-tenant bug: the route checks that the user is logged in, but not that the record belongs to them.

```ts
// Vulnerable: any logged-in user can read any org's invoice by changing the id
app.get('/invoices/:id', requireUser, async (c) => {
  const [invoice] = await db.select().from(invoices).where(eq(invoices.id, c.req.param('id')));
  return invoice ? c.json(invoice) : c.json({ error: { code: 'NOT_FOUND' } }, 404);
});

// Fixed: the query itself is scoped to the caller's org
app.get('/invoices/:id', requireOrg, async (c) => {
  const [invoice] = await db.select().from(invoices)
    .where(and(eq(invoices.id, c.req.param('id')), eq(invoices.orgId, c.get('org').id)));
  return invoice ? c.json(invoice) : c.json({ error: { code: 'NOT_FOUND' } }, 404);
});
```

- Derive the org from the verified session and membership, never from the body, query or a header alone.
- Return 404 for other tenants' records (do not confirm they exist); return 403 when the caller may know the record exists but lacks permission.
- UUIDs make ids hard to guess but are not authorization.
- Defense in depth: scoped repositories, Postgres row-level security as a second layer, audit logs for sensitive actions, and tests with two orgs. See [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md).

## 5. Consistent error responses

```json
{
  "error": {
    "code": "VALIDATION",
    "message": "Request body is invalid",
    "details": [{ "path": ["priceCents"], "message": "Expected a positive integer" }],
    "requestId": "8f14e45f-ceea-467f-a0e6-5c3a1b2d9e10"
  }
}
```

RFC 9457 (which replaced RFC 7807) standardizes the same idea as `application/problem+json` with `type`, `title`, `status`, `detail` and `instance` fields. Either works; consistency matters more.

| Situation | Status |
|---|---|
| Malformed JSON | 400 |
| Validation failed | 400 or 422 (pick one, everywhere) |
| Not authenticated | 401 |
| Authenticated but not allowed | 403 |
| Missing, or belongs to another tenant | 404 |
| Unique conflict, stale version | 409 |
| Payload too large | 413 |
| Rate limited | 429 with `Retry-After` |
| Unexpected bug | 500 with a generic message |
| Upstream failure or timeout | 502 or 504 |
| Overloaded or in maintenance | 503 |

Clients switch on `code`, not on `message`. Map errors in one place (error middleware, `onError`, oRPC interceptors), log the full error with the same request id, and never return stack traces or SQL.

## 6. Structured logging and request ids

```ts
// src/lib/logger.ts
import pino from 'pino';
import { env } from '../config/env';

export const logger = pino({
  level: env.LOG_LEVEL,                                      // 'info' in production, 'debug' locally
  base: { service: 'platform-api', env: env.NODE_ENV },
  redact: { paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'], censor: '[redacted]' },
  ...(env.NODE_ENV === 'development' ? { transport: { target: 'pino-pretty' } } : {}), // JSON lines in production
});
```

```ts
// src/middleware/request-logging.ts (Express; Hono has a requestId() middleware and works the same way)
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';
import type { RequestHandler } from 'express';
import pinoHttp from 'pino-http';
import { logger } from '../lib/logger';

export const requestStore = new AsyncLocalStorage<{ requestId: string }>();

export const httpLogger = pinoHttp({
  logger,
  genReqId: (req, res) => {
    const id = (req.headers['x-request-id'] as string | undefined) ?? randomUUID();
    res.setHeader('x-request-id', id);             // clients and support can quote it
    return id;
  },
});

export const withRequestContext: RequestHandler = (req, _res, next) => {
  requestStore.run({ requestId: String(req.id) }, next); // available anywhere in this request's async chain
};
// handlers: req.log.info({ listingId }, 'listing created')   services: requestStore.getStore()?.requestId
```

- Log one line per request (method, route, status, duration, user id, org id, request id) and errors with stacks.
- Never log passwords, tokens, full card numbers or more personal data than needed.
- Write to stdout and let the platform ship logs; `console.log` gives you no levels, structure or redaction.
- Forward the request id to downstream services and into job payloads so one id follows the whole flow.

## 7. Metrics and tracing

- **Metrics** (aggregated numbers, cheap to keep): RED per route (request rate, error rate, duration p50/p95/p99), plus saturation signals: event-loop delay, heap usage, DB pool `totalCount`/`idleCount`/`waitingCount`, queue depth and failed jobs. Expose them for Prometheus (prom-client) or export through OpenTelemetry.
- **Traces** (one request across services): spans for the HTTP handler, each SQL query, Redis calls and outbound HTTP, linked by the W3C `traceparent` header.
- **Logs** answer "what happened in this request"; metrics answer "is something wrong"; traces answer "where is the time going".

```ts
// src/instrumentation.ts: loaded before the app, e.g. node --import ./dist/instrumentation.js dist/server.js
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { NodeSDK } from '@opentelemetry/sdk-node';

new NodeSDK({
  serviceName: 'platform-api',
  traceExporter: new OTLPTraceExporter(),            // endpoint from OTEL_EXPORTER_OTLP_ENDPOINT
  instrumentations: [getNodeAutoInstrumentations()], // HTTP, pg, Redis and more, automatically
}).start();
```

ESM apps need an extra loader hook for auto-instrumentation; check the OpenTelemetry JavaScript docs.

## 8. Rate limiting

| Algorithm | How it works | Trade-off |
|---|---|---|
| Fixed window | one counter per key per window (`INCR` plus `EXPIRE`) | simple; allows double bursts at window edges |
| Sliding window | weights the previous window, or stores timestamps in a sorted set | smoother; more work per request |
| Token bucket | tokens refill at a steady rate up to a capacity | allows controlled bursts with a fair average |

```ts
import type { Redis } from 'ioredis';

export async function hitRateLimit(redis: Redis, key: string, limit: number, windowSec: number) {
  const bucket = `rl:${key}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const results = await redis.multi().incr(bucket).expire(bucket, windowSec).exec(); // one round trip, atomic
  const count = Number(results?.[0]?.[1] ?? 0);
  return { allowed: count <= limit, remaining: Math.max(0, limit - count), retryAfterSec: windowSec };
}
```

- Keys: IP for anonymous and login routes; user, org or API key for authenticated traffic; tighter limits on expensive endpoints.
- Shared Redis state is required once you run more than one instance. Libraries such as express-rate-limit with a Redis store or rate-limiter-flexible implement these algorithms.
- Layer it: the edge (CDN, WAF, API gateway) absorbs floods; the app enforces business quotas (per org plan).

## 9. CORS, security headers and limits

- CORS: an explicit origin allowlist, `credentials: true` only when cookies cross origins, cached preflights (`maxAge`), never reflect arbitrary origins.
- Headers: `helmet()` or Hono's `secureHeaders()` for HSTS, `X-Content-Type-Options: nosniff`, frame protection and CSP on HTML responses.
- HTTPS everywhere (TLS usually terminates at the load balancer); body size limits; server and outbound timeouts; a least-privilege database user; dependency scanning in CI. Details in [web security](../04-web-fundamentals/07-web-security-owasp.md).

## 10. Database access and pooling

```ts
// src/lib/db.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { env } from '../config/env';
import { logger } from './logger';
import * as schema from '../db/schema';

export const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max: 10,                          // per process: instances x max must stay below the DB limit
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,   // fail fast instead of queueing forever when the pool is exhausted
  statement_timeout: 10_000,        // the server cancels runaway queries (ms)
});
pool.on('error', (err) => logger.error({ err }, 'idle pg client error')); // unhandled, this would crash the process

export const db = drizzle({ client: pool, schema });
// shutdown: await pool.end()
```

- Size the pool by measurement: start around 10 per process and watch `waitingCount` and query latency. PostgreSQL's default `max_connections` is 100, shared by API instances, workers, migrations and admin tools.
- `pool.query()` returns clients automatically; with `pool.connect()`, always `client.release()` in `finally`.
- Keep transactions short and never call third-party APIs inside one:

```ts
await db.transaction(async (tx) => {
  const [order] = await tx.insert(orders).values({ orgId, totalCents }).returning();
  await tx.insert(outbox).values({ type: 'order.created', payload: { orderId: order!.id } }); // same commit
});
```

- Serverless creates many short-lived instances, each with its own pool. Put a pooler in front (PgBouncer in transaction mode, provider poolers, RDS Proxy) or use HTTP-based drivers. Transaction pooling breaks session features (`SET`, advisory locks, `LISTEN`), and some drivers need prepared statements disabled.
- Run migrations as a separate deploy step, and make them backward compatible (expand, migrate data, switch code, contract) so old and new code can run side by side. See [Drizzle](../12-databases/13-drizzle-orm.md) and [transactions](../12-databases/06-transactions-and-concurrency.md).

## 11. Background jobs

Move work out of the request when it is slow (emails, PDFs, exports), talks to flaky third parties (sync with another system, like the PriceCharting jobs you know from Rails), must be retried, or runs on a schedule.

```ts
// src/lib/queues.ts (BullMQ)
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null }); // BullMQ workers require this

export const emailQueue = new Queue('emails', { connection });

// producer, after the database transaction has committed
await emailQueue.add('welcome', { userId: user.id }, {
  jobId: `welcome-${user.id}`,                  // dedupe: adding the same id again is ignored (no ':' allowed)
  attempts: 5,
  backoff: { type: 'exponential', delay: 2_000 },
  removeOnComplete: 1_000,                      // keep only the most recent 1,000 completed jobs
});

// src/jobs/worker.ts: a separate process
const worker = new Worker('emails', async (job) => {
  if (job.name === 'welcome') await sendWelcomeEmail(job.data.userId); // must be safe to run twice
}, { connection, concurrency: 10 });
// on SIGTERM: await worker.close() lets the current jobs finish
```

**pg-boss** stores jobs in Postgres tables and claims them with `SELECT ... FOR UPDATE SKIP LOCKED`: no Redis to run, and a job can be enqueued in the same transaction as the data it depends on.

| | BullMQ | pg-boss | Sidekiq (Rails) |
|---|---|---|---|
| Store | Redis | Postgres | Redis |
| Enqueue atomically with your data | no: use an outbox | yes | no: enqueue after commit |
| Retries with backoff | yes | yes | yes |
| Cron-style schedules | repeatable jobs (job schedulers in recent versions) | yes | Enterprise, or sidekiq-cron |
| Throughput | very high | enough for most apps | very high |

Rules that apply to all of them: delivery is at least once, so handlers must be idempotent (check state, use unique constraints, dedupe keys); pass ids, not whole objects; set timeouts; monitor failed jobs and queue depth; run workers as a separate deployment; enqueue only after the transaction commits, or write to an **outbox** table in the same transaction and let a relay publish it. More in [background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md) and [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md).

## 12. Caching with Redis

```ts
export async function cached<T>(key: string, ttlSec: number, load: () => Promise<T>): Promise<T> {
  const hit = await redis.get(key);
  if (hit !== null) return JSON.parse(hit) as T;           // note: Dates come back as strings
  const value = await load();
  await redis.set(key, JSON.stringify(value), 'EX', ttlSec); // ioredis argument style
  return value;
}

// org-prefixed key: tenants never share entries
const stats = await cached(`v1:org:${orgId}:marketplace:stats`, 60, () => loadMarketplaceStats(orgId));
// after a write that changes the stats:
await redis.del(`v1:org:${orgId}:marketplace:stats`);
```

- Always set a TTL (add jitter so keys do not expire together); invalidate on writes; version key prefixes so a deploy can change the format.
- Stampede: when a hot key expires, hundreds of requests hit the database at once. Use a short lock (`SET key value NX PX 5000`) so one request recomputes, or serve stale data while refreshing.
- Cache what is expensive and read-heavy; skip per-user secrets in shared keys; measure the hit rate.
- HTTP caching (`Cache-Control`, `ETag`, a CDN for public GETs) often beats application caching. See [caching and performance](../04-web-fundamentals/08-caching-and-performance.md) and [Redis](../12-databases/10-redis.md).

## 13. File uploads with presigned S3 URLs

```text
1. client -> API: "I want to upload photo.jpg, image/jpeg, 2 MB"
2. API validates type, size and permissions, picks a key, returns a short-lived presigned PUT URL
3. client -> S3: PUT the file directly (the API never handles the bytes)
4. client -> API: "upload done, key = ..."; the API verifies the object exists (HeadObject) and saves the key
5. later: serve through a CDN or short-lived presigned GET URLs; the bucket stays private
```

```ts
import { randomUUID } from 'node:crypto';
import { PutObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: env.AWS_REGION });
const ALLOWED = new Set(['image/jpeg', 'image/png', 'image/webp']);

export async function createUploadUrl(orgId: string, contentType: string) {
  if (!ALLOWED.has(contentType)) throw new AppError('Unsupported file type', 422, 'VALIDATION');
  const key = `orgs/${orgId}/listings/${randomUUID()}`;           // never use the client's file name as the key
  const url = await getSignedUrl(
    s3,
    new PutObjectCommand({ Bucket: env.UPLOADS_BUCKET, Key: key, ContentType: contentType }),
    { expiresIn: 300 },                                            // seconds
  );
  return { url, key };                                             // the client must PUT with the same Content-Type
}
```

A presigned PUT cannot cap the file size; a presigned POST policy (`content-length-range`) can. The bucket needs a CORS rule for browser uploads, and a lifecycle rule cleans up abandoned uploads. Post-processing (thumbnails, virus scans) belongs in a background job.

## 14. Testing strategy

| Level | What | Tools |
|---|---|---|
| Unit | pure logic: services with fakes, validators, permission maps | Vitest |
| Integration | repositories and procedures against a real Postgres | Vitest + Testcontainers or a CI database service |
| API | status codes, auth, validation, error shapes through the HTTP layer | supertest (Express), `app.request()` (Hono) |
| Contract | the OpenAPI spec or shared contract still matches | generated clients, schema diff in CI |
| End to end | a few critical user flows through the real UI | Playwright |

```ts
// test/global-setup.ts (Vitest globalSetup): a throwaway Postgres per test run
import { execSync } from 'node:child_process';
import { PostgreSqlContainer } from '@testcontainers/postgresql';

export default async function setup() {
  const container = await new PostgreSqlContainer('postgres:17-alpine').start(); // match production's major version
  process.env.DATABASE_URL = container.getConnectionUri();                      // or Vitest's provide/inject
  execSync('npx drizzle-kit migrate', { stdio: 'inherit', env: process.env });  // apply the real migrations
  return async () => { await container.stop(); };
}
```

- Each test creates its own data with factories and cleans up (truncate between files, or roll back a transaction per test).
- Always include the two-org isolation tests for every tenant-scoped endpoint.
- Mock only what you do not own: outbound HTTP (msw, nock or undici's `MockAgent`), email, payment providers.
- More in [testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md).

## 15. API documentation with OpenAPI

Generate the spec from the same schemas that validate requests, so documentation cannot drift: oRPC's `OpenAPIGenerator` ([oRPC](05-orpc-and-typesafe-apis.md)), Hono's Zod OpenAPI integration, or Zod-to-OpenAPI libraries for Express. Serve it with Swagger UI or Scalar, lint it in CI, generate clients for other languages (a Ruby client for a Rails consumer, for example), and mark deprecations in the spec before removing anything. Design-first (write the spec, then implement) suits APIs with external consumers agreed in advance.

## 16. Deployment

```text
# Dockerfile: multi-stage build
FROM node:24-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/package.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

| Probe | Question it answers | Should check | On failure |
|---|---|---|---|
| Liveness `/health/live` | is the process stuck? | nothing external, just respond | restart the container |
| Readiness `/health/ready` | can it take traffic now? | DB ping, not shutting down | remove from the load balancer |
| Startup | has it finished booting? | migrations done, warm-up | delay the other probes |

Putting the database check in liveness restarts every instance during a brief database blip, which makes outages worse.

```ts
const server = app.listen(env.PORT);
server.keepAliveTimeout = 65_000; // longer than the load balancer's idle timeout (60 s by default on AWS ALB)
server.headersTimeout = 66_000;   // keep it above keepAliveTimeout
```

- **Horizontal scaling:** stateless instances behind a load balancer; no sticky sessions needed when state lives in Redis or Postgres; scheduled jobs must run once (a queue scheduler, not a `setInterval` in every instance).
- **Zero-downtime deploys:** rolling updates gated by readiness, graceful shutdown on SIGTERM ([core modules](02-core-modules-streams-errors.md)), backward-compatible migrations, feature flags for risky changes. See [Docker](../14-devops-and-tools/03-docker.md) and [CI/CD](../14-devops-and-tools/04-ci-cd-and-deployment.md).

| | Containers (ECS, Kubernetes, Fly, Render) | Serverless functions (Lambda, Vercel) |
|---|---|---|
| Scaling | replicas and autoscaling rules you configure | automatic per request, down to zero |
| Cold starts | none after boot | yes, worse with large bundles |
| DB connections | stable pools | many instances: needs a pooler or HTTP driver |
| Long work, websockets | fine | execution time limits; background work needs platform support |
| In-memory state | per instance | none guaranteed between invocations |
| Cost | pay for uptime | pay per use: cheap when idle, pricey at steady high load |
| Operations | more | less |

## Common pitfalls
- Long-lived JWTs with no revocation story, or tokens stored in `localStorage`.
- Checking authentication but not ownership, the classic IDOR, and taking `orgId` from the request body.
- Different error shapes per endpoint, and stack traces in production responses.
- `console.log` everywhere: no levels, no request ids, secrets in logs.
- In-memory rate limits, sessions or caches with several instances.
- Pools sized without arithmetic, exhausting the database during a scale-out; no `connectionTimeoutMillis`.
- Enqueuing jobs before the transaction commits, so workers read rows that do not exist yet.
- Non-idempotent job handlers that double-charge or double-email on retry.
- Caches without TTLs or invalidation; cache keys without the org id.
- Proxying large uploads through the API instead of using presigned URLs.
- Mocking the ORM in every test, so SQL bugs reach production.
- Liveness probes that check the database; no graceful shutdown; `keepAliveTimeout` shorter than the load balancer's idle timeout.

## How to talk about this in an interview
- Org-scoped access control on the Devntech platform is your strongest production-API story: explain how the org was derived from the session, how membership and roles were checked, how queries were scoped, and how isolation was tested.
  > **Fill in:** what you implemented for org-scoped access (middleware, scoped repositories, role checks) and one edge case you handled (org switching, invited users, admins acting across orgs).
- Rails sync APIs bring service-to-service concerns: authentication between systems, idempotency keys, retries with backoff, and backward-compatible contracts.
  > **Fill in:** how sync calls were authenticated and retried, and what happened when one side was down.
- Background jobs: map your Sidekiq experience (PriceCharting listing, price, stock and order syncs at CrystalCommerce; Sidekiq in the Colochousing Rails app) onto BullMQ or pg-boss: same at-least-once semantics, same idempotency rules.
- Keep claims proportional: if you did not own deployment or observability, say what you observed and how you would set it up. Project context: [platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md).

## Interview questions

**Q1. How do you structure a production Node API?**
Feature modules with routes or procedures, services, repositories and schemas; shared lib code (logger, errors, db, redis); Zod-validated config; one composition root that creates clients at startup; separate API and worker processes from the same codebase; stateless instances. It keeps responsibilities testable and makes shutdown and scaling straightforward.
*Follow-up:* Why separate worker processes? Jobs scale and fail independently and cannot slow down HTTP requests.

**Q2. Sessions vs JWT: which would you choose?**
For a browser app with its own backend, server-side sessions in an HttpOnly cookie: simple and instantly revocable. For mobile clients, several services or third parties, short-lived JWT access tokens plus rotating refresh tokens stored server-side. Either way, revocation and storage location matter more than the token format.
*Follow-up:* How do you revoke a JWT? Keep it short-lived, revoke the refresh token, or check a denylist (which reintroduces state).

**Q3. How do access and refresh tokens work, and where do you store them?**
A short-lived access token authorizes API calls; a long-lived random refresh token, stored hashed on the server, gets new access tokens and is rotated on each use, with reuse detection revoking the family. In browsers, keep tokens in HttpOnly Secure cookies (refresh token scoped to the refresh path); in mobile apps, in secure storage (Keychain or Keystore via SecureStore).
*Follow-up:* Why rotate refresh tokens? A stolen token works at most once before reuse detection catches it.

**Q4. Which cookie flags do you set, and why?**
`HttpOnly` against XSS theft, `Secure` for HTTPS only, `SameSite=Lax` or `Strict` against CSRF, a narrow `Path`, no `Domain` unless subdomains need it, a sensible `Max-Age`, and the `__Host-` prefix where possible. Cross-site setups need `SameSite=None; Secure` plus CSRF tokens.
*Follow-up:* Does `SameSite=Lax` fully stop CSRF? It blocks cross-site POSTs, but not same-site attacks or state-changing GET routes, so keep GETs safe.

**Q5. What is IDOR, and how do you prevent it in a multi-tenant API?**
Insecure direct object reference: an endpoint loads a record by a client-supplied id without checking the caller may access it. Prevent it by scoping every query to the org derived from the verified session (`WHERE id = $1 AND org_id = $2`), centralizing that in repositories, returning 404 for other tenants' ids, optionally adding row-level security, and testing with two orgs.
*Follow-up:* Do UUIDs fix IDOR? No: they make guessing harder, but ids leak through URLs, logs and emails.

**Q6. How do you implement RBAC?**
Assign each membership a role, map roles to permissions in code, and check permissions on the server in middleware or services (`can(role, 'listing:delete')`). Combine it with resource checks (ownership, org scope), because a role alone does not say which records a user may touch.
*Follow-up:* When does RBAC stop being enough? When rules depend on attributes (owner of this listing, plan tier), which leads to ABAC or policy objects.

**Q7. What does a good error response look like?**
A consistent envelope (or RFC 9457 problem details) with a stable machine-readable code, a human message, optional field-level details and the request id, sent with the correct status. Unexpected errors return a generic 500 while the full error is logged with the same request id.
*Follow-up:* Why include the request id? Support and developers can find the exact log lines and trace.

**Q8. How do you approach logging in production?**
Structured JSON logs with pino to stdout, one line per request with method, route, status, duration and ids, a request id propagated through `AsyncLocalStorage`, child loggers for context, redaction of secrets and personal data, and sensible levels. The platform ships and indexes the logs.
*Follow-up:* Why not `console.log`? No levels, no structure for querying, and no redaction.

**Q9. What are metrics, logs and traces each good for?**
Metrics are cheap aggregates for dashboards and alerts (RED, event-loop delay, pool saturation). Logs are detailed events for investigating a specific request. Traces show one request's path and timing across services, database calls and queues. OpenTelemetry can produce all three with shared context.
*Follow-up:* First metric to alert on for an API? Error rate and p95 latency per route.

**Q10. How do you rate limit across multiple instances?**
Keep counters in Redis (fixed window with `INCR` and `EXPIRE`, sliding window, or token bucket) keyed by IP, user, org or API key; return 429 with `Retry-After`. Use a library with a Redis store, and let the edge (CDN, WAF) absorb volumetric attacks. Behind a proxy, configure trusted proxy headers so the IP is right.
*Follow-up:* Weakness of fixed windows? Bursts at window boundaries can reach twice the limit.

**Q11. How do you size a database connection pool, and what changes in serverless?**
Total connections are instances times pool size plus workers and tools, and must stay under the database limit with headroom; start around 10 per process and tune from pool wait metrics. Serverless multiplies instances, so use a pooler (PgBouncer in transaction mode, RDS Proxy, provider poolers) or an HTTP driver, with a tiny pool per function instance.
*Follow-up:* What breaks with transaction pooling? Session state: `SET`, advisory locks, `LISTEN/NOTIFY`, sometimes prepared statements.

**Q12. When do you use a job queue, and how do you make jobs safe to retry?**
For slow, flaky, retryable or scheduled work (emails, exports, third-party syncs). Queues deliver at least once, so handlers must be idempotent: dedupe keys or job ids, unique constraints, state checks before side effects. Enqueue after commit or use an outbox, pass ids not objects, use exponential backoff and alert on failures.
*Follow-up:* BullMQ or pg-boss? BullMQ for high throughput when Redis is already there; pg-boss to avoid Redis and enqueue atomically with data.

**Q13. How do you cache with Redis and keep it consistent?**
Cache-aside: read the cache, on a miss load from the database and store with a TTL; delete or update keys when the underlying data changes; prefix keys with the org and a version. Add jitter to TTLs, a lock or stale-while-revalidate against stampedes, and accept that TTLs bound how stale data can get.
*Follow-up:* What not to cache? Data that must be strongly consistent, like balances right after payment.

**Q14. How do you handle large file uploads?**
The API issues a presigned S3 URL after validating type, size and permissions; the client uploads directly to S3; the client confirms and the API verifies the object and stores its key. Buckets stay private; downloads use a CDN or presigned GET URLs; processing happens in background jobs.
*Follow-up:* How do you enforce a size limit? A presigned POST policy with `content-length-range`, plus a check after upload.

**Q15. What is your testing strategy for an API?**
Many fast unit tests for logic, integration tests against a real Postgres (Testcontainers) for repositories and procedures, API tests through the HTTP layer for status codes, auth and validation, and a few end-to-end flows. Mock only external services, and include tenant isolation tests for every scoped endpoint.
*Follow-up:* Why not mock the database? The bugs live in SQL, constraints and transactions, which mocks never exercise.

**Q16. What is the difference between liveness and readiness probes?**
Liveness asks whether the process is alive (restart it if not) and should not check dependencies. Readiness asks whether it can serve traffic now (DB reachable, not shutting down) and failing it only removes the instance from the load balancer. During shutdown, readiness fails first so traffic drains.
*Follow-up:* What goes wrong with a DB check in liveness? A short database outage restarts every instance at once.

**Q17. Would you deploy a Node API to containers or serverless?**
Containers for steady traffic, websockets, long jobs and stable DB pools; serverless for spiky or low traffic and minimal ops, accepting cold starts, time limits and connection management. Many teams run the API in containers and use serverless for webhooks or scheduled tasks.
*Follow-up:* Biggest serverless surprise for Node APIs? Database connection exhaustion from many concurrent instances.

## Further reading
- [Node.js: Security best practices](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [Express: Production best practices, performance](https://expressjs.com/en/advanced/best-practice-performance.html)
- [Hono: Best practices](https://hono.dev/docs/guides/best-practices)
- [Docker: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Redis: Developer documentation](https://redis.io/docs/latest/develop/)
