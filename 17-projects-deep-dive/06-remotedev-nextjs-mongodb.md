# RemoteDev Internship: Backend APIs with Next.js and MongoDB

> **Tier:** T1 (know cold) · **Related:** [Next.js backend APIs and MongoDB](../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md), [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [MongoDB fundamentals](../12-databases/08-mongodb-fundamentals.md), [MongoDB schema design and aggregation](../12-databases/09-mongodb-schema-design-aggregation-mongoose.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [LMS internship (same months)](05-lms-coursera-clone.md)
> **On your resume:** RemoteDev, Full-Stack Intern (Jul 2025 to Sep 2025, remote): built and integrated backend APIs with Next.js (TypeScript) and MongoDB.

## Quick revision

| Item | What to remember |
|---|---|
| Where | RemoteDev, Full-Stack Intern, remote, Jul to Sep 2025 |
| What | Built and integrated backend APIs |
| Stack | Next.js with TypeScript, MongoDB |
| Overlap | The same months as your onsite PureLogics internship: prepare the time-management answer |
| Two verbs | "Built" (you wrote API code) and "integrated" (you connected APIs to something: know exactly what) |

- **Route handlers (App Router):** `app/api/<name>/route.ts` exports functions named after HTTP methods (`GET`, `POST`, ...) that take a Web `Request` and return a `Response` (often `NextResponse.json(...)`). The Pages Router equivalent is `pages/api/*.ts` with `(req, res)`.
- **Next.js 15 changes:** GET route handlers are no longer cached by default, and dynamic `params` arrive as a Promise you must `await`.
- **One MongoClient per process:** cache the connection promise on `globalThis`, so hot reloads in development and warm serverless invocations in production reuse it instead of opening new connections.
- **Validate at the boundary:** parse every body and query with a schema (Zod `safeParse`) and return 400 with details; never pass raw JSON into a Mongo filter (operator injection such as `{ "$ne": null }`).
- **Schema design:** embed what is read together and bounded; reference what is shared or grows without limit; documents max out at 16 MB; index the fields you filter and sort on.
- **IDs:** validate an ID string before converting it to an `ObjectId`, and return `_id` as a string `id`.
- **Transactions:** multi-document transactions need a replica set (MongoDB 4.0+) or a sharded cluster (4.2+); design documents so most writes touch one document.
- **Route handlers vs a separate API server:** one deploy and shared types vs independent scaling, long-running work, websockets and many clients.
- **Remote habits:** written daily updates, small pull requests with clear descriptions, async questions that include what you tried, agreed overlap hours.

> **Fill in:** what the product or client project was (as far as you can share), team size and location, App Router or Pages Router, native driver or Mongoose, the auth approach, where it was deployed, and what "integrated" means in your case (your own frontend, a client's frontend, third-party services).

## The 30-second pitch

> "At RemoteDev, a remote internship, I built and integrated backend APIs with Next.js and TypeScript on MongoDB. I wrote API endpoints as Next.js route handlers backed by MongoDB collections and integrated them with [the frontend or services you connected]. [One sentence on what working remotely, in the same months as your onsite PureLogics internship, taught you.]"

> **Fill in:** replace both brackets with what really happened, and add one concrete endpoint or feature you are proud of. If you used server actions or the Pages Router instead of route handlers, change the wording.

## Domain primer

> **Fill in:** what RemoteDev's product or client project does, who uses it, and which part of it your APIs served. If it is under NDA, prepare a one-line generic description ("a project-management tool for small teams") that is still true.

### Next.js as a backend

| Server-side option | What it is for |
|---|---|
| Route handlers (`app/api/**/route.ts`) | HTTP endpoints any client can call: your React pages, a mobile app, webhooks |
| Server Actions (Server Functions) | Mutations called from your own React components, often forms; stable since Next.js 14 |
| Server Components | Read data directly while rendering, with no API endpoint in between |
| Middleware (renamed `proxy` in Next.js 16; check the docs) | Runs before matching routes: redirects, headers, coarse auth checks |

### Serverless constraints
If the app runs on a serverless platform, each request may hit a fresh or a reused instance:

| Constraint | Consequence for your API code |
|---|---|
| Instances start and stop on demand | Cold starts; keep module-level setup light |
| Many instances in parallel | Each opens its own database connections; reuse one client per instance and size pools small |
| Execution time limits | Long jobs belong in a queue or a background worker, not in a request |
| No shared memory between instances | Sessions, rate limits and caches go to a database or a store such as Redis |

### MongoDB vocabulary

| Term | Meaning |
|---|---|
| Document, collection | A JSON-like (BSON) record; a group of documents, like a table without a fixed schema |
| `ObjectId` | The default `_id`: 12 bytes that include a creation timestamp |
| Embedding vs referencing | Nesting related data inside a document vs storing its ID |
| Index | A sorted structure that makes a filter or sort fast; compound indexes follow field order |
| Aggregation pipeline | Stages (`$match`, `$group`, `$lookup`, `$sort`, ...) that transform documents |
| Replica set | A primary plus secondaries; required for transactions and change streams |

## Typical architecture

> A typical setup for a Next.js and MongoDB API, to adapt to what you actually built.

```text
Browser (React pages of the same Next.js app, or another client)
   | fetch("/api/...") or Server Actions
   v
Next.js app (TypeScript)
   app/api/**/route.ts  -> parse input (Zod) -> auth check -> data layer -> JSON
   lib/mongodb.ts       -> one cached MongoClient per process
   lib/projects.ts      -> queries, mapping _id -> id, index assumptions
   |
   v
MongoDB (a managed cluster such as Atlas, or a self-hosted replica set)
```

> **Fill in:** the real folder structure, whether there was a data layer or handlers queried Mongo directly, and how the frontend consumed the APIs.

## Bullet-by-bullet

### "Built and integrated backend APIs with Next.js (TypeScript) and MongoDB"

| Part | What they ask | How to answer |
|---|---|---|
| "Built" | Which endpoints, what they did, your design decisions | Two or three endpoints with method, path, validation, query and response |
| "Integrated" | Integrated with what, and what broke | The consumer (frontend, client app, third-party API) and one integration problem you solved (Part G below) |
| Next.js (TypeScript) | Route handlers vs alternatives, caching, typing requests and responses | Parts A and F below |
| MongoDB | Schema, indexes, connection reuse, transactions | Parts B and C below |

> **Fill in:** two or three endpoints you wrote, each as method + path + one sentence, and one integration problem you solved.

> **Fill in:** how your code was reviewed (pull requests, who reviewed) and how often you shipped.

## Design deep-dive

> Typical designs to adapt. Where your real project differs, describe what you actually built and why.

### Part A: a route handler done properly

```ts
// app/api/projects/route.ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { getDb } from "@/lib/mongodb";
import { requireUser } from "@/lib/auth"; // illustrative: verifies the session, returns the user or null

const CreateProject = z.object({
  name: z.string().trim().min(1).max(120),
  description: z.string().max(2000).optional(),
});
const ListQuery = z.object({ limit: z.coerce.number().int().min(1).max(100).default(20) });

export async function GET(request: Request) {
  const user = await requireUser(request);
  if (!user) return NextResponse.json({ error: "unauthorized" }, { status: 401 });

  const query = ListQuery.safeParse(Object.fromEntries(new URL(request.url).searchParams));
  if (!query.success) return NextResponse.json({ error: query.error.issues }, { status: 400 });

  const db = await getDb();
  const projects = await db
    .collection("projects")
    .find({ ownerId: user.id })                 // always scope by the caller
    .sort({ createdAt: -1 })
    .limit(query.data.limit)
    .toArray();

  return NextResponse.json(projects.map(({ _id, ...rest }) => ({ id: _id.toHexString(), ...rest })));
}

export async function POST(request: Request) {
  const user = await requireUser(request);
  if (!user) return NextResponse.json({ error: "unauthorized" }, { status: 401 });

  const body = await request.json().catch(() => null); // invalid JSON becomes a 400 below
  const parsed = CreateProject.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: parsed.error.issues }, { status: 400 });

  const db = await getDb();
  const result = await db.collection("projects").insertOne({ ...parsed.data, ownerId: user.id, createdAt: new Date() });
  return NextResponse.json({ id: result.insertedId.toHexString() }, { status: 201 });
}
```

- Order inside every handler: authenticate, validate, authorize (scope), act, respond with a consistent shape and status code.
- Dynamic segments in Next.js 15+: `export async function GET(req: Request, { params }: { params: Promise<{ id: string }> })`, then `const { id } = await params`.
- Status codes: 201 for creation, 400 for invalid input, 401 when not logged in, 403 when logged in but not allowed, 404 for missing (or not yours), 409 for duplicates.

### Part B: reusing the Mongo connection

```ts
// lib/mongodb.ts
import { MongoClient, type Db } from "mongodb";

const uri = process.env.MONGODB_URI;
if (!uri) throw new Error("MONGODB_URI is not set");

// Cache the connecting promise on globalThis: module state is re-created on every hot reload
// in development, while globalThis survives; in production it lives as long as the instance.
const globalForMongo = globalThis as unknown as { mongoClientPromise?: Promise<MongoClient> };

const clientPromise = globalForMongo.mongoClientPromise ?? new MongoClient(uri).connect();
globalForMongo.mongoClientPromise = clientPromise;

export async function getDb(): Promise<Db> {
  return (await clientPromise).db(); // database name taken from the connection string
}
```

Why it matters: the driver keeps a connection pool per client. Creating a client per request opens new connections every time, which is slow (TCP, TLS, auth) and can exhaust the cluster's connection limit when many serverless instances run at once. One client per instance, created lazily and reused, fixes both. With Mongoose, the same idea applies: cache the `mongoose.connect` promise.

> **Fill in:** whether you used the native driver or Mongoose, and whether you ran into connection or cold-start issues.

### Part C: schema design behind the API

| Embed when | Reference when |
|---|---|
| Data is read together with its parent (an order's line items) | Data is shared by many parents (a user referenced by many projects) |
| The embedded list stays small and bounded | The list grows without limit (all activity of a user) |
| You want single-document atomic updates | The child has its own lifecycle, permissions or queries |
| Fewer round trips matter most | Smaller documents and independent updates matter more |

```ts
// Run once at deploy time (a script), not on every request.
const db = await getDb();
await db.collection("projects").createIndex({ ownerId: 1, createdAt: -1 }); // matches find + sort above
await db.collection("users").createIndex({ email: 1 }, { unique: true });   // uniqueness enforced by the DB
```

- Compound index order follows equality filters first, then sort fields, then range filters.
- Pagination: for feeds, prefer range-based pagination on `(createdAt, _id)` over large `skip` values, which scan and discard documents.

> **Fill in:** the main collections you worked with and one embed-or-reference decision you made.

### Part D: validation and security
- **Operator injection:** JSON bodies can contain objects, so a field you expect to be a string can arrive as an operator.

```ts
// Dangerous: if body.token is { "$ne": null }, this matches the first user with any reset token.
await db.collection("users").findOne({ resetToken: body.token });

// Safe: validate the shape first, so only a string can reach the filter.
const ResetInput = z.object({ token: z.string().min(32).max(128) });
```

- **IDs:** check the format (`/^[a-f\d]{24}$/i`) before `new ObjectId(id)`, and answer 404 for malformed IDs instead of a 500.
- **Mass assignment:** never `$set` the whole request body; pick allowed fields (the Zod schema is your allowlist).
- **Errors:** log details on the server, return generic messages to clients.
- **Rate limiting:** on login and other sensitive endpoints, using a shared store when there are many instances.

### Part E: authentication

| Option | Notes |
|---|---|
| Auth.js (NextAuth) | Sessions or JWTs, OAuth providers, adapters for databases including MongoDB |
| A hosted auth service | Fast to integrate; vendor dependency and cost |
| Custom JWT in an HttpOnly cookie | Full control; you own expiry, rotation and revocation |

Check auth inside each handler (or a shared helper every handler calls), not only in middleware: a 2025 Next.js vulnerability (CVE-2025-29927) let crafted requests skip middleware on unpatched versions, and handlers that verify the session themselves were not exposed by it.

> **Fill in:** how authentication worked in the project and where it was checked.

### Part F: route handlers vs a separate API server

| Next.js route handlers | Separate API (Express, Hono, NestJS, ...) |
|---|---|
| One repo and one deploy with the frontend | Deployed and scaled independently |
| Shared TypeScript types with the UI for free | Shared types need a package or a contract |
| Fits request/response CRUD | Better for long-running jobs, websockets, heavy background work |
| Platform limits (timeouts, cold starts) apply when serverless | Long-lived process, connection pools behave predictably |
| Ideal when the Next.js app is the main client | Better when many clients (mobile, partners) share the API |

### Part G: integrating the APIs with a frontend
"Integrated" usually means the API and its consumer had to agree on a contract. Typical pieces (keep the ones that match your work):

```ts
// lib/contracts.ts: one schema, used by the handler to validate and by the client to type and check.
import { z } from "zod";

export const Project = z.object({ id: z.string(), name: z.string(), description: z.string().optional() });
export type Project = z.infer<typeof Project>;

export async function fetchProjects(limit = 20): Promise<Project[]> {
  const res = await fetch(`/api/projects?limit=${limit}`);   // same origin: the session cookie is sent
  if (!res.ok) throw new Error(`GET /api/projects failed with ${res.status}`);
  return z.array(Project).parse(await res.json());           // fail loudly if the contract drifts
}
```

```ts
// Only when another origin calls the API: answer the CORS preflight in the route file.
export function OPTIONS() {
  return new Response(null, {
    status: 204,
    headers: {
      "Access-Control-Allow-Origin": process.env.FRONTEND_ORIGIN ?? "",
      "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    },
  });
}
```

- Agree on one error shape (`{ error: ... }`) and status codes, so the UI can show validation messages next to fields.
- Keep secrets server-side: only variables prefixed `NEXT_PUBLIC_` reach the browser bundle, so database URIs and API keys must never use that prefix.
- Typical integration bugs: CORS failures (missing preflight handling or a wrong origin), dates arriving as strings and never parsed, `_id` vs `id` mismatches, and stale data because a client cache was not invalidated after a mutation.

> **Fill in:** what consumed your APIs, whether types were shared, and one integration bug you fixed.

### Part H: working remotely
Habits interviewers like to hear, as a checklist to compare with what you really did:
- A short written update each day: done, next, blocked.
- Questions that are easy to answer async: context, what you tried, what you expected, a specific ask.
- Small pull requests with a description, screenshots or sample requests, and testing notes.
- Agreed overlap hours with the team and a quick response time inside them.
- Decisions written down (in the ticket or pull request), so nobody depends on a call nobody recorded.

> **Fill in:** the team's location and time zone, the tools you used (chat, tracker, code hosting), the meeting rhythm, and one remote-specific challenge you solved.

## Likely interview questions

**Q1. What did you build at RemoteDev?**
Product in one line (or a generic description if confidential), then two or three endpoints with what each does, how inputs are validated and how data is stored, then what you integrated them with.
> **Fill in:** the endpoints and the integration.

**Q2. Why build APIs in Next.js instead of a separate Express or NestJS server?**
One deploy, shared types and less infrastructure, good for CRUD served mainly to the app's own frontend. A separate server wins for long-running work, websockets, independent scaling or many clients. Say which situation RemoteDev was in.

**Q3. How do you reuse the MongoDB connection in a serverless or hot-reloading environment?**
Create one `MongoClient` per process and cache its connection promise on `globalThis`; every handler awaits the same promise. Otherwise each request or reload opens new connections and the cluster runs out of them.

**Q4. How did you design your collections? When do you embed vs reference?**
Embed bounded data that is read with the parent and updated with it; reference shared or unbounded data. Then index the fields you filter and sort on, and mention the 16 MB document limit.
> **Fill in:** one real decision.

**Q5. How did you validate input?**
A schema per endpoint (Zod `safeParse` for bodies and query strings), 400 responses with issue details, and the schema doubling as the allowlist of writable fields.
> **Fill in:** the validation approach you really used.

**Q6. How did you handle authentication and authorization?**
Describe the real mechanism; then: verify the session in every handler, scope every query by the user (or their team), return 404 for resources that are not theirs.

**Q7. How do you prevent NoSQL injection?**
Validate types before building filters, never spread request bodies into filters or updates, and avoid `$where` and server-side JavaScript evaluation.

**Q8. How did you paginate?**
`limit` plus `skip` for small admin lists; range-based (cursor) pagination on an indexed sort key such as `(createdAt, _id)` for feeds and large collections.

**Q9. How do transactions work in MongoDB?**
Multi-document ACID transactions exist since 4.0 on replica sets and 4.2 on sharded clusters, through a session (`session.withTransaction(...)` in the Node driver). They cost performance, so model data so most writes touch one document atomically.

**Q10. What changed for route handlers in Next.js 15?**
GET handlers are dynamic (not cached) by default, and `params` (plus `cookies()` and `headers()`) became async. Newer releases changed more (for example Next.js 16 renamed middleware to proxy), so check the docs for the version in use.

**Q11. How did you collaborate as a remote intern?**
Daily written updates, async questions with context, small reviewed pull requests, agreed overlap hours, decisions written down.
> **Fill in:** the real tools and rhythm.

**Q12. How did you manage this internship alongside the PureLogics one?**
Be factual and calm: one was onsite with fixed hours, the other remote and more flexible; you planned your weeks, protected the hours each one needed, communicated availability clearly, and delivered on both. Never exaggerate the hours. See [common HR questions](../18-behavioral-and-hr/03-common-hr-questions.md).
> **Fill in:** how you actually split your time (hours per week or a typical day), whether both teams knew, and one example of handling a clash.

**Q13. How did you test the APIs?**
Manual requests with an HTTP client during development, then automated tests that call handlers with a test database (or an in-memory MongoDB) and check status codes, validation errors and scoping.
> **Fill in:** what testing really existed.

**Q14. MongoDB or PostgreSQL for this kind of project?**
MongoDB suits flexible, document-shaped data read as a whole and fast iteration; PostgreSQL suits relational data, strong constraints, joins and reporting. Many CRUD apps fit either; the team's experience and the data's shape decide.

**Q15. What would you do differently now?**
Pick real improvements, for example stricter shared types between API and UI, an index review, automated tests, or a clearer data layer.
> **Fill in:** two genuine improvements.

## Fill-in checklist
- [ ] What the product or client project was and what your APIs served.
- [ ] Team, time zone, tools, review process.
- [ ] App Router or Pages Router; route handlers, server actions or both.
- [ ] Native driver or Mongoose; how the connection was managed.
- [ ] Collections, indexes and one schema decision.
- [ ] Validation, auth and authorization approach.
- [ ] What "integrated" means for you, and one integration problem.
- [ ] Testing and deployment.
- [ ] How you split your time with PureLogics, and how you would describe it in one calm sentence.
