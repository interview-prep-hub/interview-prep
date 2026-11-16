# Next.js Backend: Route Handlers, Server Actions and MongoDB

> **Tier:** T1 (know cold) · **Related:** [Next.js fundamentals](01-nextjs-fundamentals.md), [Data fetching and caching](02-nextjs-data-fetching-and-caching.md), [MongoDB schema design and Mongoose](../12-databases/09-mongodb-schema-design-aggregation-mongoose.md), [MongoDB fundamentals](../12-databases/08-mongodb-fundamentals.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [Production APIs](../09-nodejs-backend/06-production-apis.md)
> **On your resume:** RemoteDev (remote Full-Stack Intern, Jul to Sep 2025): you built and integrated backend APIs with Next.js (TypeScript) and MongoDB. This file is the technical depth behind that line.

## Quick revision
- Route Handlers live in `app/**/route.ts` and export `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD` or `OPTIONS`, using Web `Request`/`Response` plus the `NextRequest`/`NextResponse` helpers (introduced in 13.2).
- Since Next.js 15 the context's `params` is a promise: `{ params }: { params: Promise<{ id: string }> }`. Recent versions also generate a global `RouteContext<'/api/listings/[id]'>` type.
- Read input with `await request.json()`, `request.nextUrl.searchParams`, `request.headers.get()`, `request.cookies.get()` or `await cookies()`; use `await request.text()` for raw webhook bodies.
- Respond with `Response.json(data, { status })` or `NextResponse.json(...)`, `new Response(null, { status: 204 })`, or a `ReadableStream` for streaming.
- GET handlers are dynamic (uncached) by default since Next.js 15.
- Server Actions for mutations from your own UI; Route Handlers for public or mobile APIs, webhooks, non-POST methods and streaming; a separate backend when other clients, teams or long-running work need it.
- Middleware (`middleware.ts`, renamed `proxy.ts` in Next.js 16) runs before routes for redirects, rewrites, headers and optimistic auth checks. Keep it light and never make it the only auth check.
- Auth: a session in an HttpOnly cookie (Auth.js, Better Auth, or a signed token via `jose`), verified by a helper that pages, Route Handlers and Server Actions all call.
- MongoDB: cache the Mongoose connection promise on `globalThis` so hot reloads and warm serverless invocations reuse one pool; keep `maxPoolSize` small; reset the cache when connecting fails.
- Models: `models.Listing || model('Listing', schema)` avoids `OverwriteModelError` when modules re-run.
- `.lean()` returns plain objects (faster, less memory, no document methods); map `_id` to string ids in DTOs.
- Indexes follow query shapes: `{ orgId: 1, status: 1, _id: -1 }`; a compound index's prefix also serves `orgId`-only queries.
- Validate bodies and query strings with Zod at the boundary; Mongoose validation is a second line of defense.
- Pagination: `skip/limit` gives page numbers but slows on deep pages; a cursor on `_id` (or a sort key plus `_id`) is fast and stable.
- Errors: invalid ObjectId (`CastError`) -> 400 or 404, `ValidationError` -> 422, duplicate key `11000` -> 409, unknown -> 500 with a generic message.
- Status codes: 200 read and update, 201 plus `Location` on create, 204 on delete, 400, 401, 403, 404 (also for other orgs' ids), 409, 422.
- The Edge runtime has Web APIs only, so TCP drivers such as Mongoose do not run there; the Node.js runtime is the default, and recent docs mark the Edge option for routes as deprecated.
- Secrets stay in server-only env vars (never `NEXT_PUBLIC_`), and database modules import `server-only`.
- Test handlers by calling the exported function with a `Request`; use `mongodb-memory-server` or Testcontainers for database tests.
- Serverless limits: maximum duration, cold starts, connection counts, request body size (about 4.5 MB on Vercel functions), no persistent disk.

## 1. Route Handlers

```ts
// app/api/health/route.ts
export async function GET() {
  return Response.json({ ok: true, time: new Date().toISOString() });
}
```

| Input | How to read it |
|---|---|
| Path params | second argument: `const { id } = await params` |
| Query string | `request.nextUrl.searchParams.get('page')` (on `NextRequest`) or `new URL(request.url).searchParams` |
| JSON body | `await request.json()` (throws on malformed JSON) |
| Form data | `await request.formData()` |
| Raw body | `await request.text()` or `await request.arrayBuffer()` (webhook signatures) |
| Headers | `request.headers.get('authorization')` or `await headers()` |
| Cookies | `request.cookies.get('session')?.value` or `await cookies()` (setting cookies works here and in Server Actions) |

- A `route.ts` cannot sit in the same segment as a `page.tsx`; API routes usually live under `app/api/`.
- If `OPTIONS` is not exported, Next.js answers it automatically with an `Allow` header.
- For browsers on other origins, return CORS headers (or set them for many routes in `next.config` headers or the proxy), and allowlist specific origins rather than `*` when credentials are involved.
- Route Handlers use the same segment config as pages (`runtime`, `maxDuration`, and in the previous caching model `dynamic` and `revalidate`).

A webhook endpoint, which needs the exact raw bytes for its signature check (see [webhook verification](../09-nodejs-backend/02-core-modules-streams-errors.md)):

```ts
// app/api/webhooks/payments/route.ts
export async function POST(request: Request) {
  const raw = await request.text();
  if (!isValidSignature(Buffer.from(raw), request.headers.get('x-signature-256') ?? undefined, process.env.WEBHOOK_SECRET ?? '')) {
    return new Response('Invalid signature', { status: 401 });
  }
  const event = JSON.parse(raw) as { id: string; type: string };
  await recordEventOnce(event);          // idempotent: providers retry deliveries
  return Response.json({ received: true });
}
```

Streaming (server-sent events, LLM tokens, progress updates):

```ts
// app/api/progress/route.ts
export async function GET() {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      for (const step of ['validating', 'importing', 'done']) {
        controller.enqueue(encoder.encode(`data: ${step}\n\n`));   // one SSE message
        await new Promise((resolve) => setTimeout(resolve, 500));
      }
      controller.close();
    },
  });
  return new Response(stream, { headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache' } });
}
```

## 2. Server Actions vs Route Handlers vs a separate backend

| Need | Server Actions | Route Handlers | Separate backend (Express, Hono, Rails, an oRPC service) |
|---|---|---|---|
| Mutations from your own Next.js forms and buttons | best fit | works | works, with an extra hop |
| Reads for your own pages | no: use Server Components | possible, but Server Components can read directly | through fetch or TanStack Query |
| Mobile apps, partners, public API | no: unstable ids, POST only | yes | yes |
| Webhooks (payments, GitHub) | no | yes (raw body) | yes |
| GET/PUT/DELETE semantics, custom status codes, streaming | no | yes | yes |
| Long-running jobs, queues, websockets | no | limited on serverless | best fit (workers, persistent processes) |
| Several frontends or languages share it | no | partly | best fit |
| Independent scaling and deploys | tied to the Next.js app | tied to the Next.js app | independent |

A typical small product keeps everything in Next.js: Server Components for reads, Server Actions for form mutations, Route Handlers for webhooks and the mobile app. A larger platform puts the API in its own service and lets Next.js (or TanStack Start) be one client among several.

## 3. Middleware, renamed proxy in Next.js 16

Middleware runs before a request reaches a route, which suits cross-cutting, cheap decisions: redirects, rewrites (A/B tests, multi-tenant hostnames), security headers, locale detection and optimistic auth checks.

```ts
// middleware.ts (Next.js 15). In Next.js 16: proxy.ts exporting a function named proxy.
import { NextResponse, type NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  if (!request.cookies.has('session')) {                  // optimistic: the cookie exists, it is not verified
    const url = new URL('/login', request.url);
    url.searchParams.set('next', request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }
  return NextResponse.next();
}

export const config = { matcher: ['/dashboard/:path*', '/settings/:path*'] };
```

- Runtime: middleware originally ran only on the Edge runtime (Web APIs only); Node.js runtime support arrived during the 15.x releases, and Next.js 16's `proxy.ts` runs on Node.js. Check the docs for your version.
- Keep it fast: it runs for every matched request. Avoid database queries there; decode or check cookies and leave real authorization to the data layer.
- Never rely on it alone. In 2025, CVE-2025-29927 let attackers skip middleware in self-hosted Next.js apps by sending a crafted internal header; apps that also verified sessions in pages, handlers and actions stayed protected.

## 4. Authentication patterns

A signed session cookie with `jose` (the pattern from the Next.js authentication guide):

```ts
// lib/session.ts
import 'server-only';
import { jwtVerify, SignJWT } from 'jose';
import { cookies } from 'next/headers';
import { z } from 'zod';

const key = new TextEncoder().encode(process.env.SESSION_SECRET);
const SessionSchema = z.object({ userId: z.string(), orgId: z.string(), role: z.enum(['admin', 'member']) });
export type Session = z.infer<typeof SessionSchema>;

export async function createSession(session: Session) {       // call from a Server Action or Route Handler
  const token = await new SignJWT(session).setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt().setExpirationTime('7d').sign(key);
  (await cookies()).set('session', token, {
    httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'lax', path: '/',
    maxAge: 60 * 60 * 24 * 7,                                   // seconds (unlike Express, which uses ms)
  });
}

export async function getSession(): Promise<Session | null> {
  const token = (await cookies()).get('session')?.value;
  if (!token) return null;
  try {
    const { payload } = await jwtVerify(token, key, { algorithms: ['HS256'] });
    return SessionSchema.parse(payload);
  } catch {
    return null;                                                // expired, tampered or malformed
  }
}
```

Libraries save you from writing OAuth flows and session rotation:

```ts
// auth.ts (Auth.js v5 style; check the docs for your version)
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';

export const { handlers, auth, signIn, signOut } = NextAuth({ providers: [GitHub] });
// app/api/auth/[...nextauth]/route.ts:  import { handlers } from '@/auth'; export const { GET, POST } = handlers;
// anywhere on the server:               const session = await auth();
```

Auth.js (formerly NextAuth) has been maintained by the Better Auth team since September 2025, and Better Auth is their recommendation for new projects; both store users and sessions in your database (MongoDB adapters exist).

| Where | Check | On failure |
|---|---|---|
| Middleware or proxy | optimistic: is there a session cookie? | redirect to `/login` |
| Layout or page | `await getSession()` (through the DAL) | `redirect('/login')` |
| Route Handler | `await requireApiUser()` | 401 JSON |
| Server Action | `await requireUser()` | throw, or return an error state |
| Data access function | session plus org or ownership filter in the query | not found, or throw |

## 5. MongoDB with Mongoose in Next.js

**Connection caching.** In development, Next.js re-evaluates modules on every edit, so a module-level connection would be recreated (and leaked) constantly; `globalThis` survives those reloads. In serverless production, each warm instance keeps its module scope between invocations, so one cached connection serves every request on that instance. The full discussion is in [Mongoose in serverless](../12-databases/09-mongodb-schema-design-aggregation-mongoose.md).

```ts
// lib/mongodb.ts
import 'server-only';
import mongoose from 'mongoose';
import { env } from '@/env';                      // Zod-validated environment

type MongooseCache = { conn: typeof mongoose | null; promise: Promise<typeof mongoose> | null };
const globalForMongoose = globalThis as typeof globalThis & { mongooseCache?: MongooseCache };
const cache = (globalForMongoose.mongooseCache ??= { conn: null, promise: null });

export async function connectToDatabase() {
  if (cache.conn) return cache.conn;
  cache.promise ??= mongoose.connect(env.MONGODB_URI, {
    bufferCommands: false,                         // fail fast instead of queueing queries while disconnected
    maxPoolSize: 10,                               // per instance: many serverless instances multiply this
  });
  try {
    cache.conn = await cache.promise;              // concurrent first requests share one in-flight promise
  } catch (err) {
    cache.promise = null;                          // allow the next request to retry
    throw err;
  }
  return cache.conn;
}
```

**Models, indexes and DTOs:**

```ts
// models/listing.ts
import 'server-only';
import { model, models, Schema, type InferSchemaType } from 'mongoose';

const listingSchema = new Schema(
  {
    orgId: { type: Schema.Types.ObjectId, required: true },
    ownerId: { type: Schema.Types.ObjectId, required: true },
    title: { type: String, required: true, trim: true, maxlength: 120 },
    priceCents: { type: Number, required: true, min: 0 },
    status: { type: String, enum: ['draft', 'published'], default: 'draft' },
  },
  { timestamps: true },                                    // createdAt and updatedAt
);
listingSchema.index({ orgId: 1, status: 1, _id: -1 });     // equality fields first, then the sort key

export type ListingDoc = InferSchemaType<typeof listingSchema>;
const createListingModel = () => model('Listing', listingSchema);
export const Listing = (models.Listing as ReturnType<typeof createListingModel> | undefined) ?? createListingModel();

export const toListingDto = (d: { _id: unknown; title: string; priceCents: number; status: string; createdAt: Date }) => ({
  id: String(d._id), title: d.title, priceCents: d.priceCents, status: d.status, createdAt: d.createdAt.toISOString(),
});
```

- Without the `models.Listing ??` guard, a hot reload calls `model('Listing', ...)` twice and Mongoose throws `OverwriteModelError`.
- `.lean()` skips hydrating full Mongoose documents, so reads are faster and use less memory, but there are no getters, virtuals or `save()`. Use it for reads you serialize to JSON.
- Map documents to DTOs: you control exactly which fields leave the server, and `ObjectId`s and dates become strings.
- Zod validates the request shape at the API boundary (clear 422 errors, unknown keys stripped); the Mongoose schema protects the data from every code path, including scripts and jobs.

## 6. A complete CRUD API for one resource

```ts
// lib/validation/listing.ts
import { z } from 'zod';

const fields = {
  title: z.string().trim().min(3).max(120),
  priceCents: z.number().int().min(0),
  status: z.enum(['draft', 'published']),
};
export const ListingCreate = z.object({ ...fields, status: fields.status.default('draft') });
export const ListingUpdate = z.object(fields).partial()            // built from the default-free fields on purpose
  .refine((v) => Object.keys(v).length > 0, { message: 'Send at least one field' });
export const ListQuery = z.object({
  status: fields.status.optional(),
  cursor: z.string().regex(/^[a-f\d]{24}$/i).optional(),
  limit: z.coerce.number().int().min(1).max(50).default(20),
});
export const isObjectIdHex = (id: string) => /^[a-f\d]{24}$/i.test(id);
```

```ts
// lib/api-errors.ts
import mongoose from 'mongoose';
import { NextResponse } from 'next/server';
import { ZodError } from 'zod';
import { getSession } from './session';

export class ApiError extends Error {
  readonly status: number;
  readonly code: string;
  constructor(status: number, code: string) {
    super(code);
    this.status = status;
    this.code = code;
  }
}

const fail = (status: number, code: string, extra: Record<string, unknown> = {}) =>
  NextResponse.json({ error: { code, ...extra } }, { status });

export const notFound = () => fail(404, 'NOT_FOUND');

export async function requireApiUser() {
  const session = await getSession();
  if (!session) throw new ApiError(401, 'UNAUTHENTICATED');
  return session;
}

export function toErrorResponse(err: unknown) {
  if (err instanceof ApiError) return fail(err.status, err.code);
  if (err instanceof ZodError) return fail(422, 'VALIDATION', { issues: err.issues });
  if (err instanceof SyntaxError) return fail(400, 'INVALID_JSON');            // from request.json()
  if (err instanceof mongoose.Error.ValidationError) return fail(422, 'VALIDATION', { message: err.message });
  if (err instanceof mongoose.Error.CastError) return fail(400, 'INVALID_ID');
  if (typeof err === 'object' && err !== null && 'code' in err && err.code === 11000) return fail(409, 'CONFLICT');
  console.error(err);                                                           // structured logger in production
  return fail(500, 'INTERNAL');
}
```

```ts
// app/api/listings/route.ts: GET (list) and POST (create)
import type { NextRequest } from 'next/server';
import { NextResponse } from 'next/server';
import { requireApiUser, toErrorResponse } from '@/lib/api-errors';
import { connectToDatabase } from '@/lib/mongodb';
import { ListingCreate, ListQuery } from '@/lib/validation/listing';
import { Listing, toListingDto } from '@/models/listing';

export async function GET(request: NextRequest) {
  try {
    const user = await requireApiUser();
    const q = ListQuery.parse(Object.fromEntries(request.nextUrl.searchParams));
    await connectToDatabase();
    const docs = await Listing.find({
      orgId: user.orgId,                                     // tenant scope from the session, never the query string
      ...(q.status ? { status: q.status } : {}),
      ...(q.cursor ? { _id: { $lt: q.cursor } } : {}),       // Mongoose casts the hex string to an ObjectId
    }).sort({ _id: -1 }).limit(q.limit + 1).lean();          // one extra row tells us whether there is a next page
    const items = docs.slice(0, q.limit).map(toListingDto);
    return NextResponse.json({ data: items, nextCursor: docs.length > q.limit ? items.at(-1)?.id ?? null : null });
  } catch (err) {
    return toErrorResponse(err);
  }
}

export async function POST(request: NextRequest) {
  try {
    const user = await requireApiUser();
    const input = ListingCreate.parse(await request.json());
    await connectToDatabase();
    const doc = await Listing.create({ ...input, orgId: user.orgId, ownerId: user.userId });
    const body = toListingDto(doc.toObject());
    return NextResponse.json({ data: body }, { status: 201, headers: { Location: `/api/listings/${body.id}` } });
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/listings/[id]/route.ts: GET (one), PATCH (update), DELETE
import type { NextRequest } from 'next/server';
import { NextResponse } from 'next/server';
import { ApiError, notFound, requireApiUser, toErrorResponse } from '@/lib/api-errors';
import { connectToDatabase } from '@/lib/mongodb';
import { isObjectIdHex, ListingUpdate } from '@/lib/validation/listing';
import { Listing, toListingDto } from '@/models/listing';

type Ctx = { params: Promise<{ id: string }> };

export async function GET(_request: NextRequest, { params }: Ctx) {
  try {
    const user = await requireApiUser();
    const { id } = await params;
    if (!isObjectIdHex(id)) return notFound();
    await connectToDatabase();
    const doc = await Listing.findOne({ _id: id, orgId: user.orgId }).lean();  // org-scoped: no IDOR
    return doc ? NextResponse.json({ data: toListingDto(doc) }) : notFound();
  } catch (err) {
    return toErrorResponse(err);
  }
}

export async function PATCH(request: NextRequest, { params }: Ctx) {
  try {
    const user = await requireApiUser();
    const { id } = await params;
    if (!isObjectIdHex(id)) return notFound();
    const patch = ListingUpdate.parse(await request.json());
    await connectToDatabase();
    const doc = await Listing.findOneAndUpdate(
      { _id: id, orgId: user.orgId },
      { $set: patch },
      { new: true, runValidators: true },      // return the updated document; run schema validators on the update
    ).lean();
    return doc ? NextResponse.json({ data: toListingDto(doc) }) : notFound();
  } catch (err) {
    return toErrorResponse(err);
  }
}

export async function DELETE(_request: NextRequest, { params }: Ctx) {
  try {
    const user = await requireApiUser();
    if (user.role !== 'admin') throw new ApiError(403, 'FORBIDDEN');
    const { id } = await params;
    if (!isObjectIdHex(id)) return notFound();
    await connectToDatabase();
    const { deletedCount } = await Listing.deleteOne({ _id: id, orgId: user.orgId });
    return deletedCount ? new Response(null, { status: 204 }) : notFound();    // 204 must not have a body
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

| Request | Success | Errors |
|---|---|---|
| `GET /api/listings?status=&cursor=&limit=` | 200 with `data` and `nextCursor` | 401, 422 (bad query) |
| `POST /api/listings` | 201 with `Location` | 400 (malformed JSON), 401, 409 (duplicate), 422 (validation) |
| `GET /api/listings/:id` | 200 | 401, 404 (missing, invalid id, or another org's listing) |
| `PATCH /api/listings/:id` | 200 with the updated listing | 400, 401, 404, 422 |
| `DELETE /api/listings/:id` | 204 | 401, 403 (not an admin), 404 |

## 7. Pagination

| | Offset (`skip`/`limit`) | Cursor (keyset) |
|---|---|---|
| Query | `.skip((page - 1) * limit).limit(limit)` | `{ _id: { $lt: lastId } }` with `.sort({ _id: -1 })` |
| Page numbers, total count | easy (`countDocuments`) | no random access; "load more" style |
| Deep pages | slow: the server walks and discards skipped documents | fast with an index |
| Inserts while paging | duplicates or skipped items | stable |

```ts
const filter = { orgId: user.orgId, status: 'published' as const };
const [items, total] = await Promise.all([
  Listing.find(filter).sort({ createdAt: -1 }).skip((page - 1) * limit).limit(limit).lean(),
  Listing.countDocuments(filter),
]);
```

`_id` works as a cursor because ObjectIds start with a timestamp, so they sort roughly by creation time. To sort by another field (price), the cursor must combine that field and `_id` as a tiebreaker.

## 8. Edge vs Node.js runtime

| | Node.js runtime (default) | Edge runtime |
|---|---|---|
| APIs | full Node.js: `fs`, `net`, `crypto`, `Buffer`, any npm package | Web APIs (`fetch`, Web Crypto, streams) and a subset of Node APIs |
| Database drivers | TCP drivers: Mongoose, `pg` | HTTP-based drivers only |
| Cold start | slower | very fast |
| Limits | longer execution, larger bundles | small bundles, short CPU time |
| Location | usually one region near your database | possibly far from your database |
| Status | recommended default | recent docs mark the route-level `runtime = 'edge'` option as deprecated |

`export const runtime = 'edge'` in a route that imports Mongoose fails, because Mongoose needs TCP sockets.

## 9. Environment variables and secrets

- `MONGODB_URI`, `SESSION_SECRET`, API keys: server-only variables, set per environment in the hosting platform; locally in `.env.local` (not committed).
- Never prefix secrets with `NEXT_PUBLIC_`: those are inlined into the browser bundle at build time.
- `import 'server-only'` in database, session and model modules turns an accidental client import into a build error.
- Validate env with Zod at startup so a missing `MONGODB_URI` fails the deploy, not the first request.

## 10. Testing Route Handlers

```ts
// app/api/listings/route.test.ts (Vitest)
import { NextRequest } from 'next/server';
import { describe, expect, it, vi } from 'vitest';

vi.mock('@/lib/api-errors', async (importOriginal) => ({
  ...(await importOriginal<typeof import('@/lib/api-errors')>()),
  requireApiUser: vi.fn().mockResolvedValue({ userId: 'u1', orgId: '665f1f77bcf86cd799439011', role: 'admin' }),
}));
vi.mock('@/lib/mongodb', () => ({ connectToDatabase: vi.fn() }));

import { POST } from './route';

describe('POST /api/listings', () => {
  it('returns 422 for an invalid body', async () => {
    const request = new NextRequest('http://localhost/api/listings', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ title: 'x' }),
    });
    const response = await POST(request);
    expect(response.status).toBe(422);
    expect((await response.json()).error.code).toBe('VALIDATION');
  });
});
```

- Handlers are plain async functions taking a `Request`, so you can call them directly without a server.
- For real database behavior, start an in-memory MongoDB with `mongodb-memory-server` (`await MongoMemoryServer.create()`, then `getUri()`) or a container, and test org isolation with two orgs.
- Add a few end-to-end tests (Playwright) for flows that cross pages, actions and APIs.

## 11. Serverless limits

| Limit | Effect | Mitigation |
|---|---|---|
| Maximum duration (plan-dependent; `export const maxDuration` on supporting platforms) | long requests are killed | queue the work, stream progress, split it up |
| Cold starts | slow first request on a new instance | small bundles, lazy imports, cached connections |
| Connection counts | many instances times `maxPoolSize` can exhaust MongoDB Atlas limits | small pools, connection caching, the right cluster tier |
| Request body size (about 4.5 MB on Vercel functions) | uploads fail | presigned uploads straight to object storage |
| No persistent disk | written files disappear | object storage |
| Work after the response | may be cut off | `after()` from `next/server` (Next.js 15.1+) for short tasks; a queue for real jobs |
| WebSockets | not supported by standard functions | a separate realtime service |

## Common pitfalls
- Calling `mongoose.connect` inside every handler with no cache, or caching a rejected promise forever.
- Defining models without the `models.X ??` guard and hitting `OverwriteModelError` on hot reload.
- Querying by `_id` alone in a multi-tenant API (IDOR); filter by `orgId` from the session.
- Trusting `orgId`, `ownerId` or `role` from the request body.
- Returning raw Mongoose documents: internal fields leak and `ObjectId`s serialize unexpectedly; map to DTOs.
- A malformed id causing a 500 (`CastError`) instead of a 404 or 400.
- `NextResponse.json(...)` with status 204 (a 204 must not have a body); use `new Response(null, { status: 204 })`.
- Deriving a PATCH schema with `.partial()` from one that has `.default()` values: Zod 4 applies defaults inside optional fields, so missing fields get overwritten.
- `skip`/`limit` on huge collections; missing indexes for the list query.
- Mongoose code in an Edge route or in middleware.
- Treating middleware as the authorization layer.
- Secrets in `NEXT_PUBLIC_` variables.

## How to talk about this in an interview
- This is your RemoteDev story. Describe the shape: Route Handlers (or Server Actions) validated input with Zod, checked the session, called Mongoose through a cached connection, and returned DTOs with correct status codes; the frontend consumed them.
  > **Fill in:** which resources or features you built APIs for at RemoteDev, App Router or Pages Router API routes, and how authentication worked.
  > **Fill in:** what "integrated" meant in your role: connecting the frontend to the APIs, a third-party service, or both; and one bug you fixed (connection leaks, validation gaps, slow queries).
- Show you know why the serverless details exist: hot reload and warm instances explain the `globalThis` cache and the model guard.
- Keep claims at internship scale and honest, then show growth: at Devntech you saw the same API concerns (validation, tenant scoping, typed contracts) on a larger TypeScript platform with oRPC and Drizzle. Deep dive: [RemoteDev project](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md).

## Interview questions

**Q1. What are Route Handlers, and how do they differ from Pages Router API routes?**
Files named `route.ts` in `app/` that export functions per HTTP method and use Web `Request`/`Response` (with `NextRequest`/`NextResponse` helpers). Pages Router API routes (`pages/api`) export one `(req, res)` handler using Node-style objects and a body parser. Route Handlers support streaming and the same segment config as pages.
*Follow-up:* Are GET handlers cached? Not by default since Next.js 15.

**Q2. How do you read params, query, body and cookies in a Route Handler in Next.js 15+?**
`const { id } = await params` from the second argument, `request.nextUrl.searchParams.get('page')`, `await request.json()` (or `text()` for raw bodies) and `request.cookies.get('session')` or `await cookies()`.
*Follow-up:* What does `request.json()` do with malformed JSON? It throws a `SyntaxError`, which you map to a 400.

**Q3. When do you use Server Actions, Route Handlers or a separate backend?**
Server Actions for mutations triggered by your own Next.js UI; Route Handlers for webhooks, mobile or partner clients, non-POST methods and streaming; a separate backend when several clients or teams share the API, or you need long-running jobs, websockets and independent scaling.
*Follow-up:* Can a mobile app call a Server Action? Not reliably: ids change between builds and only POST is supported.

**Q4. How do you reuse a MongoDB connection in Next.js and serverless?**
Cache the `mongoose.connect` promise on `globalThis`, reuse it on later calls, reset it when connecting fails, and keep `maxPoolSize` small. Hot reload re-runs modules in development, and warm serverless instances keep globals between invocations, so one pool serves many requests.
*Follow-up:* Why cache the promise, not just the connection? Concurrent first requests then share one connection attempt instead of opening several.

**Q5. Why `mongoose.models.Listing || mongoose.model('Listing', schema)`?**
Modules re-execute on hot reload (and can be loaded more than once), and compiling the same model name twice throws `OverwriteModelError`. The guard reuses the compiled model if it exists.
*Follow-up:* How do you keep its TypeScript type? Cast `models.Listing` to the type returned by `model(...)`.

**Q6. What does `.lean()` do, and when should you avoid it?**
It returns plain JavaScript objects instead of Mongoose documents: faster and lighter, ideal for read-only responses. Avoid it when you need document behavior: `save()`, getters and setters, virtuals or middleware tied to documents.
*Follow-up:* Does `lean()` skip schema validation on reads? Reads do not validate anyway; validation applies to writes.

**Q7. How do you paginate in MongoDB, and which approach do you prefer?**
Offset pagination with `skip`/`limit` plus `countDocuments` gives page numbers but slows on deep pages and shifts when data changes. Cursor pagination filters past the last seen `_id` (or sort key plus `_id`) with an index: fast and stable. Use cursors for feeds and large lists, offsets for small admin tables.
*Follow-up:* How do you know there is a next page without counting? Fetch `limit + 1` rows.

**Q8. Walk me through a CRUD API for one resource with status codes.**
`GET /api/listings` lists with filters and a cursor (200); `POST` validates and creates (201 with `Location`; 422 or 409 on errors); `GET /:id` returns one (200 or 404); `PATCH /:id` partially updates (200, 404, 422); `DELETE /:id` removes (204, 403, 404). Every handler authenticates, scopes by org and maps errors centrally.
*Follow-up:* PUT vs PATCH? PUT replaces the whole resource; PATCH changes only the fields sent.

**Q9. How do you prevent IDOR in these routes?**
Include the caller's org (or owner) in every query, `findOne({ _id: id, orgId: session.orgId })`, with the org taken from the verified session, and return 404 when nothing matches so other tenants' ids are indistinguishable from missing ones.
*Follow-up:* Is checking the role enough? No: roles say what kind of action is allowed, not which records.

**Q10. Why validate with Zod if Mongoose already validates?**
Zod checks the request contract at the boundary: types, coercion, unknown keys stripped, and clear 422 responses before any database work. Mongoose validation protects stored data from every code path. They complement each other; Zod also gives TypeScript types for handler code.
*Follow-up:* What is a mass assignment bug? Spreading the raw body into an update, letting clients set fields like `role` or `orgId`.

**Q11. What is middleware (proxy in Next.js 16), and what should it not do?**
Code that runs before matched requests to redirect, rewrite, set headers or do optimistic auth checks. It should not run heavy logic or database queries, and it must not be the only authorization layer; verify sessions again in pages, Route Handlers and Server Actions.
*Follow-up:* Why the rename to proxy? To make the network-boundary role explicit; `proxy.ts` runs on the Node.js runtime.

**Q12. How do you protect pages, Route Handlers and Server Actions?**
One server-only session helper (`getSession`, `requireUser`) used everywhere: pages redirect to login, Route Handlers return 401, actions throw or return errors, and data functions scope queries by org or owner. Middleware can add an early redirect for user experience.
*Follow-up:* Where should the session live? An HttpOnly, Secure, SameSite cookie, signed or pointing to a server-side session.

**Q13. Edge vs Node.js runtime: which do you pick for a MongoDB API?**
Node.js: Mongoose needs TCP sockets and Node APIs that the Edge runtime does not provide. Edge suits light logic close to users (redirects, headers) with HTTP-based data access, and recent Next.js docs mark the route-level Edge option as deprecated.
*Follow-up:* What breaks if you set `runtime = 'edge'` on that route? The Mongoose import or connection fails.

**Q14. How do you test Route Handlers?**
Import the exported `GET`/`POST` functions and call them with a `NextRequest`, mocking auth and the database layer for unit tests; for integration tests, run against `mongodb-memory-server` or a container. Assert status codes, response bodies and tenant isolation.
*Follow-up:* What do you not mock? The query logic you want to trust, so integration tests use a real database.

**Q15. Which serverless limits affect Next.js APIs?**
Maximum execution time, cold starts, connection counts across many instances, request body size limits, no persistent disk, and work cut off after the response. Mitigate with queues, presigned uploads, cached connections, small pools and `after()` for short follow-up tasks.
*Follow-up:* Where do websockets go? A separate realtime service.

**Q16. How do you handle MongoDB errors in an API?**
Map them centrally: `CastError` (invalid ObjectId) to 400 or 404, `ValidationError` to 422 with field messages, duplicate key error code 11000 to 409, Zod errors to 422, malformed JSON to 400, and anything else to a generic 500 that is logged with details.
*Follow-up:* How do you avoid the `CastError` entirely? Validate ids (24 hex characters) before querying.

**Q17. How do you handle webhooks in Next.js?**
A Route Handler reads the raw body with `await request.text()`, verifies the HMAC signature in constant time, rejects stale timestamps, dedupes by event id, does minimal work (store or enqueue) and returns 2xx quickly.
*Follow-up:* Why not a Server Action? Webhooks are plain HTTP calls from another service, with their own headers and raw bodies.

## Further reading
- [Next.js: Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)
- [Next.js: Authentication guide](https://nextjs.org/docs/app/guides/authentication)
- [Next.js: Proxy (formerly middleware)](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)
- [MongoDB: Indexes](https://www.mongodb.com/docs/manual/indexes/)
- [Mongoose documentation](https://mongoosejs.com/docs/)
