# REST API Design

> **Tier:** T1 (know cold) · **Related:** [How the web works](01-how-the-web-works.md), [GraphQL](05-graphql.md), [Authentication and authorization](06-authentication-authorization.md), [Web security and OWASP](07-web-security-owasp.md), [Rails API development](../05-ruby-on-rails/10-api-development.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md)
> **On your resume:** REST APIs and background jobs at Devntech; the Colochousing Rails REST API serving the Coloc mobile app; Rails sync APIs for the multi-tenant platform migration; consuming the PriceCharting API for CrystalCommerce; DRF APIs at PureLogics and Next.js APIs at RemoteDev.

## Quick revision
- REST is an architectural style: resources identified by URLs, manipulated through representations (usually JSON) with standard HTTP methods, stateless requests, cacheable responses and a uniform interface.
- Nouns, not verbs; plural collections; nest one level at most: `GET /houses/12/bills`, `POST /houses/12/bills`, `PATCH /bills/981`.
- CRUD mapping: GET list and show (200), POST create (201 + `Location`), PUT/PATCH update (200 or 204), DELETE (204), async work (202).
- Status codes honestly: 400 malformed, 401 unauthenticated, 403 forbidden, 404 missing or hidden, 409 conflict, 422 validation, 429 rate limited, 5xx server faults. Never 200 with an error body.
- One error shape everywhere; RFC 9457 problem details (`application/problem+json` with `type`, `title`, `status`, `detail`, `instance` plus extensions) is a standard choice.
- JSON: consistent casing, ISO 8601 UTC timestamps, money as integer minor units plus currency, IDs as strings when they may exceed 2^53, a `data`/`meta` envelope for room to grow.
- **Offset pagination** (`?page=3&per_page=25`): page numbers and totals, but slow deep pages and duplicates or gaps when rows change. **Cursor/keyset** (`?after=<opaque>`): fast and stable with an index, but next/previous only.
- Filtering, sorting and sparse fields via query params (`?status=open&sort=-due_on&fields=id,title`); allowlist every field name.
- Versioning: URI (`/api/v1`) is the most common and visible; header versioning keeps URLs clean. Additive changes need no new version.
- Mobile apps cannot be updated instantly: stay backward compatible, send the app version on every request, enforce a minimum supported version with a forced-update screen.
- **Idempotency keys** make POST retry-safe: the client sends `Idempotency-Key`, and the server stores the first result and replays it for duplicates.
- **Optimistic concurrency:** `ETag` on GET, `If-Match` on PUT/PATCH, 412 on mismatch (or 409 from a `lock_version` check).
- Rate limit with 429 and `Retry-After`; clients back off exponentially with jitter.
- Long-running work: 202 Accepted plus a status resource the client polls, or a webhook or push when it finishes.
- **Webhooks:** HMAC-signed payloads with timestamps, retries with backoff, at-least-once delivery; consumers verify, dedupe by event ID, answer 2xx fast and process in a job.
- HATEOAS = responses carry links to related resources and allowed actions; rarely done fully, pagination links are the common subset.
- OpenAPI describes the API in YAML or JSON for docs, generated clients and contract tests.
- REST for resource APIs, integrations and HTTP caching; GraphQL for client-shaped reads across many entities; RPC (gRPC, tRPC, oRPC) for internal, action-oriented, type-safe calls.
- Security: HTTPS, authenticate every request, authorize every object (BOLA tops the OWASP API Security Top 10), validate input, cap page and payload sizes, rate limit.

## 1. What REST actually means

REST (Representational State Transfer) comes from Roy Fielding's 2000 dissertation. It is a set of constraints, not a protocol:

| Constraint | Meaning | In practice |
|---|---|---|
| Client-server | UI and data storage evolve independently | web app, mobile app and integrations share one API |
| Stateless | each request carries everything needed; no server-side conversation state | auth on every request; any server instance can handle any request |
| Cacheable | responses declare whether they can be cached | `Cache-Control`, `ETag`, 304 |
| Uniform interface | resources with URLs, standard methods, self-descriptive messages, hypermedia | `GET /bills/981`, `Content-Type`, status codes |
| Layered system | a client cannot tell whether it talks to the origin or an intermediary | CDNs, load balancers, API gateways |
| Code on demand (optional) | the server may send executable code | JavaScript; rarely discussed |

The Richardson maturity model is a handy way to describe real APIs: level 0 sends everything to one endpoint (RPC over HTTP), level 1 introduces resources, level 2 adds proper methods and status codes, level 3 adds hypermedia. Most "REST APIs", including typical Rails and DRF APIs, are level 2: JSON over HTTP following REST conventions. Saying so shows you know the difference.

## 2. Resource naming

| Prefer | Avoid | Why |
|---|---|---|
| `GET /bills` | `GET /getBills` | the method is already the verb |
| `POST /houses/12/bills` | `POST /createBillForHouse?id=12` | nouns and hierarchy |
| `GET /bills/981` | `GET /bill?id=981` | plural collections, IDs in the path |
| `GET /rental-applications` | `GET /rentalApplications` | lowercase, hyphenated paths |
| `GET /houses/12/bills`, then `GET /bills/981` | `GET /orgs/3/houses/12/bills/981/payments/7` | nest one level; deeper resources get their own top-level route |
| `GET /bills?status=open` | `GET /open-bills` | filters are query parameters |
| `GET /me`, `PATCH /me/settings` | `GET /users/current-user-settings` | singular resources for "the current one" |

Not everything is CRUD. Options for actions, from most to least RESTful:
- **A state change** when it is a simple field update without side effects: `PATCH /applications/42` with `{"status": "approved"}`.
- **A sub-resource noun** when the action has its own data and side effects: `POST /applications/42/approval`, `POST /payments/7/refunds`.
- **A scoped action** when nothing else fits: `POST /bills/981/reminders` or `POST /bills/981/remind` (in Rails, a `member` route). Consistency across the API matters more than purity.

## 3. Operations, methods and status codes

| Operation | Request | Success | Common failures |
|---|---|---|---|
| List | `GET /houses/12/bills?limit=25` | 200 with data and pagination meta | 401, 403, 404 (house hidden) |
| Show | `GET /bills/981` | 200, or 304 if conditional | 404 |
| Create | `POST /houses/12/bills` | 201 with `Location` and the resource | 400, 409, 422 |
| Replace | `PUT /bills/981` | 200 with the resource, or 204 | 400, 404, 412, 422 |
| Update | `PATCH /bills/981` | 200 with the resource, or 204 | 400, 404, 409, 412, 422 |
| Delete | `DELETE /bills/981` | 204 | 404, 409 (cannot delete a paid bill) |
| Start async work | `POST /exports` | 202 with `Location: /exports/55` | 400, 429 |

Method semantics (safe, idempotent, PUT vs PATCH) and the meaning of each status code are covered in [How the web works](01-how-the-web-works.md).

## 4. JSON conventions

```json
{
  "data": [
    {
      "id": "981",
      "title": "Internet",
      "amount_cents": 4500,
      "currency": "EUR",
      "status": "open",
      "due_on": "2026-11-01",
      "created_at": "2026-10-16T09:30:00Z"
    }
  ],
  "meta": { "next_cursor": "eyJpZCI6OTgxfQ", "has_more": true }
}
```

- **Casing:** pick one. Rails and DRF APIs often use snake_case and JavaScript clients prefer camelCase; transform at a single boundary (serializer or API client), never mix.
- **Time:** ISO 8601 in UTC with a zone (`2026-10-16T09:30:00Z`), dates without time as `2026-11-01`. Never send ambiguous local times: Belgium and Pakistan are three or four hours apart depending on daylight saving.
- **Money:** integer minor units plus currency (`"amount_cents": 4500, "currency": "EUR"`) or a decimal string (`"45.00"`), never floats.
- **IDs:** strings when they can exceed 2^53 (JavaScript numbers lose precision beyond that) or when they are UUIDs.
- **Enums** as documented lowercase strings (`"status": "overdue"`), and tell clients to tolerate unknown values.
- **Nulls:** decide between omitting a key and sending `null`, and be consistent; TypeScript clients model them differently (`field?: T` vs `field: T | null`).
- **Envelope:** `{ "data": ..., "meta": ... }` leaves room for pagination and metadata without breaking clients; a bare array cannot be extended.
- **Related data:** embed small related objects or offer `?include=payer`; avoid deep object graphs (that is GraphQL's strength). Keep one serializer per resource so its shape is identical everywhere (jbuilder, Alba or Blueprinter in Rails, DRF serializers, zod schemas in TypeScript).

## 5. Errors

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "2 fields are invalid.",
  "instance": "/houses/12/bills",
  "request_id": "7c1e9d04-5b1f-4d0a-9a49-2f1b3f6e8c21",
  "errors": [
    { "field": "amount_cents", "code": "greater_than", "message": "must be greater than 0" },
    { "field": "due_on", "code": "blank", "message": "can't be blank" }
  ]
}
```

This follows RFC 9457 (problem details, served as `application/problem+json`), which replaced RFC 7807. The rules matter more than the exact format:
- The same shape for every error, including 401, 404, 429 and 500, so clients need one parser.
- A machine-readable `code` per error for client logic and translations, plus a human `message`.
- Field-level errors, so web forms and mobile screens can highlight the right input.
- A request ID that matches your logs; never stack traces, SQL or internal hostnames.

```ruby
class Api::BaseController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound do
    render_problem(404, "Not found")
  end

  rescue_from ActionController::ParameterMissing do |e|
    render_problem(400, "Bad request", detail: e.message)
  end

  private

  def render_problem(status, title, detail: nil, errors: nil)
    body = { type: "about:blank", title: title, status: status, detail: detail,
             errors: errors, request_id: request.request_id }.compact
    render json: body, status: status, content_type: "application/problem+json"
  end
end
```

## 6. Pagination

**Offset pagination:**

```text
GET /houses/12/bills?page=3&per_page=25

{ "data": [ ... ], "meta": { "page": 3, "per_page": 25, "total_count": 412, "total_pages": 17 } }
```

The database runs `ORDER BY created_at DESC, id DESC LIMIT 25 OFFSET 50`, reading and discarding the skipped rows, so page 8,000 is slow. If a row is inserted while someone pages, everything shifts: they see a duplicate on the next page (or miss a row after a delete). `COUNT(*)` for totals is also expensive on big tables.

**Cursor (keyset) pagination:**

```text
GET /houses/12/bills?limit=25&after=eyJjcmVhdGVkX2F0IjoiMjAyNi0xMC0xMFQwODowMDowMFoiLCJpZCI6OTgxfQ

{ "data": [ ... ], "meta": { "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0xMC0wOVQxNzozMDowMFoiLCJpZCI6OTU1fQ", "has_more": true } }
```

The cursor is an opaque, base64-encoded copy of the last row's sort key. The server decodes it and seeks directly to the next rows:

```sql
SELECT * FROM bills
WHERE house_id = $1
  AND (created_at, id) < ($2, $3)      -- rows after the last one seen, in descending order
ORDER BY created_at DESC, id DESC
LIMIT 26;                             -- one extra row tells you whether there is a next page
```

With an index on `(house_id, created_at DESC, id DESC)` every page costs the same. The `id` tie-breaker makes the order total, so rows with identical timestamps are never skipped. Keeping the cursor opaque lets you change its contents later.

| | Offset | Cursor / keyset |
|---|---|---|
| Jump to page N | yes | no: next and previous only |
| Deep page performance | degrades (reads and discards rows) | constant with the right index |
| Stable while rows change | no: duplicates and gaps | yes |
| Total count | easy, but slow on huge tables | usually omitted |
| Sorting by any column | easy | each sort order needs its own cursor and index |
| Best for | admin tables with page numbers, small datasets | feeds, infinite scroll, mobile lists, syncs and exports |

Always cap the page size (for example `per_page` at most 100). Some APIs also send `Link: <https://api.example.com/bills?after=abc>; rel="next"` headers, as GitHub does. In Rails, pagy and kaminari handle offset pagination (recent pagy versions also offer keyset); keyset is easy to hand-write. More on the SQL side in [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md).

## 7. Filtering, sorting and sparse fieldsets

```text
GET /listings?status=active&price_min=1000&price_max=5000&q=zelda&sort=-updated_at,title&fields=id,title,price_cents&include=store
```

- Filters are query params; ranges use `_min`/`_max` (or `price[gte]=1000`); free-text search uses `q`.
- `sort=-updated_at,title` (a minus means descending, the JSON:API convention).
- Sparse fieldsets (`fields=id,title`) slim payloads for mobile lists; `include=store` embeds related data, which means preloading on the server to avoid N+1 queries.
- Allowlist every field name. Column names cannot be bound as SQL parameters, so `order(params[:sort])` is an injection risk.

```ruby
SORTABLE = { "created_at" => :created_at, "due_on" => :due_on, "amount" => :amount_cents }.freeze

def sorted(scope)
  raw = params[:sort].to_s
  direction = raw.start_with?("-") ? :desc : :asc
  column = SORTABLE[raw.delete_prefix("-")] || :created_at
  scope.order(column => direction, id: direction)   # id as a tie-breaker keeps the order stable
end
```

Index the filter and sort combinations you actually support, and reject unknown parameters with a 400 rather than silently ignoring them when they would change results.

## 8. Versioning and evolving an API

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URI path | `/api/v1/bills` | obvious, easy to route, cache and try in a browser | URLs change per version; tempting to version too often |
| Header | `Accept: application/vnd.example.v2+json` or a dated `API-Version` header | clean URLs, fine-grained | invisible in browsers and logs unless you log it; harder to test |
| Query param | `/bills?version=2` | easy | mixes with filters and cache keys |

Stripe-style dated versions pinned per account are a header variant. Whatever you choose, most changes should not need a new version:
- **Non-breaking:** new endpoints, new optional request fields, new response fields, and new enum values only if clients tolerate unknown values (many do not).
- **Breaking:** removing or renaming fields, changing types or formats, changing meaning or defaults (sorting, pagination), making an optional field required, changing the error format or auth scheme.
- **Deprecating:** announce, send `Deprecation` and `Sunset` (RFC 8594) headers with a date, track usage per version in logs, contact the remaining consumers, then remove.

### Supporting old mobile app versions
- Store releases roll out over days and many users never update, so several app versions call the API at once, for months.
- Send the app version and platform with every request (`X-App-Version: 2.4.1`, `X-Platform: ios`) and log them; then you know when nobody reads an old field anymore.
- Prefer additive changes: add the new field, keep the old one until old app versions fade out, and run old and new serializers side by side when you must.
- Enforce a **minimum supported version**: a config endpoint (`GET /app_config` returning `min_supported_version` and `latest_version`) or a dedicated error code drives a blocking "please update" screen, plus a softer prompt for recommended updates. Some teams reuse 426 Upgrade Required for this, though it was designed for protocol upgrades.
- JavaScript-only fixes can ship through EAS Update without a store release, but only to builds with a matching `runtimeVersion`; native changes need a new store build ([EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)).

## 9. Idempotency keys for POST

A resident pays rent from the app on a weak connection. The server charges the card, but the response never arrives, so the app retries. Without protection, the resident pays twice.

1. The client generates a UUID per *logical operation* (not per HTTP attempt), sends it as `Idempotency-Key`, and reuses it on every retry.
2. The server atomically claims `(user, key)`, using a unique index or Redis `SET key value NX EX 86400`.
3. A duplicate for a completed key gets the stored response (same status and body) without redoing the work.
4. A duplicate while the first attempt is still running gets 409; the same key with a different body gets 422.
5. Keys expire after a retention window (Stripe keeps them for about 24 hours). The IETF has a draft standardizing the header; Stripe popularized it.

```ruby
class Api::V1::PaymentsController < Api::BaseController
  def create
    key = request.headers["Idempotency-Key"]
    return render_problem(400, "Idempotency-Key header is required") if key.blank?

    fingerprint = Digest::SHA256.hexdigest(request.raw_post)
    begin
      claim = IdempotencyKey.create!(user: current_user, key: key, request_hash: fingerprint) # unique index on (user_id, key)
    rescue ActiveRecord::RecordNotUnique
      existing = IdempotencyKey.find_by!(user: current_user, key: key)
      return render_problem(422, "Idempotency-Key reused with a different request") if existing.request_hash != fingerprint
      return render_problem(409, "The original request is still in progress") if existing.response_status.nil?
      return render(json: existing.response_body, status: existing.response_status) # replay
    end

    payment = Payments::Charge.call(user: current_user, params: payment_params) # your service object
    claim.update!(response_status: 201, response_body: payment.as_json)
    render json: payment, status: :created
  end
end
```

If processing fails with a retryable error, delete or mark the claim so the client can retry with the same key. For payments, pass the key on to the payment provider too; providers such as Stripe accept idempotency keys themselves.

## 10. Optimistic concurrency with ETag and If-Match

Two admins edit the same house at once; without a check, the last save silently overwrites the first (a lost update).

```text
GET /houses/12
HTTP/1.1 200 OK
ETag: "v7"

PATCH /houses/12
If-Match: "v7"
Content-Type: application/json

{"monthly_rent_cents": 65000}

HTTP/1.1 412 Precondition Failed        someone saved v8 first: refetch, merge, retry
```

The server compares `If-Match` with the current version and answers 412 on a mismatch, or 428 Precondition Required if it insists on the header. Rails' optimistic locking (a `lock_version` column) raises `ActiveRecord::StaleObjectError` on stale updates, which you map to 409 or 412; clients send the version back in the body or as the ETag. For short critical sections where conflicts are likely, pessimistic locking (`SELECT ... FOR UPDATE`, Rails `lock!`) is the alternative ([Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md)).

## 11. Rate limiting

- **Why:** protect against abuse, brute force and runaway clients, share capacity fairly and control costs.
- **Key** by API key, user or IP. IP alone punishes everyone behind a shared NAT, such as an office or a mobile carrier.
- **Algorithms:** fixed window counters (simple, but bursts at window edges), sliding windows (smoother) and token buckets (bursts up to the bucket size, then a steady refill rate). Counters usually live in Redis with atomic `INCR` and `EXPIRE` ([Redis](../12-databases/10-redis.md)).
- **Response:** 429 with `Retry-After`. Many APIs advertise limits with `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; an IETF draft standardizes `RateLimit` headers.
- **Tools:** Rails 7.2+ `rate_limit to: 10, within: 1.minute`, Rack::Attack, `express-rate-limit` or Hono middleware, API gateways and CDNs, Nginx `limit_req`.
- **As a client** (including background jobs calling third parties, like a PriceCharting sync): honor `Retry-After`, back off exponentially with jitter, cap retries and lower concurrency.

## 12. Bulk endpoints

A mobile app syncing 200 task updates should not send 200 requests. A bulk endpoint takes a capped list (say 100 items) and returns per-item results in the same order:

```json
{
  "results": [
    { "index": 0, "status": 200, "data": { "id": "t_1", "done": true } },
    { "index": 1, "status": 422, "errors": [{ "field": "due_on", "code": "invalid" }] }
  ]
}
```

- Decide between **atomic** (one transaction, all or nothing, simplest for clients) and **partial success** (per-item status; the overall response is 200, or 207 Multi-Status, a WebDAV code some APIs borrow).
- Large imports should accept the payload or file and process it asynchronously (202).
- Avoid N+1 inside bulk handlers: load every referenced record in one query. Rails `insert_all`/`upsert_all` are fast but skip validations and callbacks, so validate first.

## 13. Long-running work: 202 and a status resource

```text
POST /exports
{"type": "bills", "house_id": 12, "format": "csv"}

HTTP/1.1 202 Accepted
Location: /exports/55
{"id": "55", "status": "queued"}

GET /exports/55    {"id": "55", "status": "processing", "progress": 40}
GET /exports/55    {"id": "55", "status": "completed", "download_url": "https://files.example.com/exports/55.csv?signature=..."}
```

- The POST enqueues a background job (Sidekiq) that updates the status record: `queued`, `processing`, `completed` or `failed` with an error message.
- The client polls with backoff, or the server notifies it when done (webhook, push notification, realtime event).
- Large results are delivered through short-lived signed URLs (for example S3 presigned URLs).
- Make the POST idempotent so a retry does not start two exports. Job design details: [Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md).

## 14. Webhooks

**As a provider:**
- Send an event envelope such as `{"id": "evt_123", "type": "payment.succeeded", "created_at": "...", "data": {...}}`, or a "thin" event with IDs only that consumers refetch.
- Sign each delivery: HMAC-SHA256 over `timestamp + "." + raw body` with a per-endpoint secret, sent in headers such as `X-Webhook-Timestamp` and `X-Webhook-Signature`. Consumers can then prove authenticity and reject replays of old deliveries.
- Deliver from a background job with a short timeout; any 2xx counts as success; retry everything else with exponential backoff over hours or days; disable endpoints that keep failing and notify their owners; let customers see and replay deliveries.
- Promise at-least-once delivery and no ordering, and document both.

**As a consumer:**

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { Hono } from "hono";

const app = new Hono();

app.post("/webhooks/payments", async (c) => {
  const raw = await c.req.text(); // verify the exact bytes received, not re-serialized JSON
  const timestamp = c.req.header("X-Webhook-Timestamp") ?? "";
  const signature = c.req.header("X-Webhook-Signature") ?? "";

  const expected = createHmac("sha256", process.env.WEBHOOK_SECRET!).update(`${timestamp}.${raw}`).digest("hex");
  const fresh = Math.abs(Date.now() / 1000 - Number(timestamp)) < 300;
  const valid = signature.length === expected.length && timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
  if (!fresh || !valid) return c.text("invalid signature", 401);

  const event = JSON.parse(raw) as { id: string; type: string };
  await enqueueWebhookJob(event); // the job dedupes on event.id (unique index) and does the real work
  return c.body(null, 200);       // answer fast so the provider does not time out and retry
});
```

- Verify the signature over the raw body with a constant-time comparison and check the timestamp tolerance.
- Respond 2xx quickly and process in a job; slow handlers cause timeouts, which cause retries, which cause duplicates.
- Dedupe by event ID, and handle out-of-order events by comparing timestamps or versions, or by refetching the resource from the provider's API. A Ruby version of the verification is in [Web security and OWASP](07-web-security-owasp.md).

## 15. HATEOAS, briefly

```json
{
  "id": "42",
  "status": "pending",
  "_links": {
    "self": { "href": "/applications/42" },
    "approve": { "href": "/applications/42/approval", "method": "POST" },
    "house": { "href": "/houses/12" }
  }
}
```

Hypermedia as the Engine of Application State means clients discover URLs and allowed actions from links in responses instead of hardcoding them; here the `approve` link only appears while the application is pending. Few APIs implement it fully because clients are written against documentation anyway; pagination links and `Location` headers are the parts nearly everyone uses.

## 16. Documenting with OpenAPI

```yaml
openapi: 3.1.0
info:
  title: Bills API
  version: 1.0.0
paths:
  /houses/{houseId}/bills:
    get:
      summary: List bills for a house
      parameters:
        - { name: houseId, in: path, required: true, schema: { type: string } }
        - { name: after, in: query, schema: { type: string } }
      responses:
        "200":
          description: A page of bills
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { type: array, items: { $ref: "#/components/schemas/Bill" } }
        "404":
          description: House not found or not visible to the caller
components:
  schemas:
    Bill:
      type: object
      required: [id, title, amount_cents]
      properties:
        id: { type: string }
        title: { type: string }
        amount_cents: { type: integer }
```

An OpenAPI document powers interactive docs (Swagger UI, Redoc), generated clients and types (openapi-generator, openapi-typescript), mock servers, request validation and contract tests. **Design-first** means writing and reviewing the spec before code; **code-first** generates it from code: rswag from RSpec request specs in Rails, drf-spectacular for DRF, and zod-based tooling such as `@hono/zod-openapi` or oRPC's OpenAPI support in TypeScript. Generating from tested code is the easiest way to keep docs honest.

## 17. REST vs GraphQL vs RPC

| | REST | GraphQL | RPC (gRPC, tRPC, oRPC) |
|---|---|---|---|
| Model | resources and HTTP methods | a typed graph; the client picks fields | procedures (functions) |
| Endpoints | many URLs | one (`/graphql`) | one per procedure |
| Over- and under-fetching | common: fixed shapes, extra requests | solved by client-written queries | shapes per procedure |
| HTTP caching | natural (GET, `Cache-Control`, CDNs) | hard (POST to one URL); persisted queries help | usually none |
| Types | through OpenAPI | schema built in | built in (protobuf, or TypeScript inference) |
| Errors | HTTP status codes | `errors` array, usually with HTTP 200 | framework error types |
| Best for | public APIs, CRUD, web plus mobile clients, integrations, webhooks | many clients needing different shapes, aggregating services | internal services, full-stack TypeScript monorepos, fast service-to-service calls (gRPC) |

The multi-tenant TypeScript platform in your Colochousing work uses oRPC, which gives end-to-end types without code generation, while the Rails side exposes REST sync APIs: a realistic mix. See [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md) and [GraphQL](05-graphql.md).

## 18. API security basics

- HTTPS only, with HSTS.
- Authenticate every request (session, token or API key) and keep tokens short-lived ([Authentication and authorization](06-authentication-authorization.md)).
- Authorize every object: scope queries to the current user or organization (`current_org.bills.find(id)`). Broken Object Level Authorization is number one in the OWASP API Security Top 10 (2023).
- Validate and allowlist input (strong parameters, zod, DRF serializers) so nobody can set `role` or `org_id` (mass assignment).
- Return explicit serializer fields so password digests and internal notes never leak.
- Limit everything: page size, body size, bulk item count, timeouts, rate limits.
- Keep secrets and tokens out of URLs; restrict CORS to browser origins you own; log with request IDs but never tokens or personal data.
- Retire old versions properly: forgotten, unpatched endpoints are their own OWASP API risk (improper inventory management). Full list: [Web security and OWASP](07-web-security-owasp.md).

## Common pitfalls
- Verbs in URLs, inconsistent naming and casing across endpoints.
- 200 for errors, 500 for validation errors, or a different error shape per endpoint.
- Unbounded list endpoints; offset pagination for infinite-scroll feeds.
- `order(params[:sort])` and other unvalidated field names.
- Breaking changes without a version, or removing fields that old mobile app versions still read.
- POST retries without idempotency keys (duplicate payments and bookings).
- Lost updates because nobody checks versions.
- Floats for money and local times without a zone.
- Slow work inside the request (timeouts and 504s) instead of 202 plus a job.
- Webhook consumers that skip signature checks or do heavy work before responding.
- Missing object-level authorization checks (IDOR).

## How to talk about this in an interview
- The Colochousing REST API is your anchor: one Rails 7 API serves the web front end and the Coloc app used by 400+ residents. Talk about backward compatibility with installed app versions, status codes and error formats the app relies on, and pagination for lists such as events, tasks and bills.
- Devntech background jobs connect directly to section 13 (202 plus a job) and to idempotent processing.
- CrystalCommerce makes you an API *consumer*: syncing listings, prices, stock and orders with PriceCharting means pagination, rate limits, retries with backoff and idempotent upserts on your side.
- The multi-tenant migration adds Rails sync APIs between the legacy app and the new oRPC/TanStack Start platform: a natural place to discuss idempotency, versioning and REST vs RPC.
- When you describe a design, name the trade-off out loud (offset vs cursor, URI vs header versioning) instead of presenting one option as the only answer.

> **Fill in:** how the Colochousing API is versioned and how the Coloc app handles old versions (minimum version check, forced update screen, EAS Update for JS-only fixes).

> **Fill in:** one endpoint you designed or changed end to end: route, payload, status codes, pagination, and how you tested it.

> **Fill in:** how the Rails sync APIs for the platform migration handled retries and duplicates.

## Interview questions

**Q1. What is REST, and what are its constraints?**
An architectural style for networked systems: resources identified by URLs, manipulated through representations using standard methods. Constraints: client-server, stateless, cacheable, uniform interface, layered system and optional code on demand. Most real APIs are level 2 on the Richardson scale: resources, methods and status codes, without hypermedia.
*Follow-up:* Why does statelessness help? Any server instance can handle any request, which makes horizontal scaling and load balancing simple.

**Q2. Design endpoints for houses, rooms and rental applications, including approving an application.**
`GET/POST /houses`, `GET/PATCH/DELETE /houses/:id`, `GET/POST /houses/:id/rooms`, `GET /rooms/:id`, `POST /rooms/:id/applications`, `GET /applications/:id`, and approval as `POST /applications/:id/approval` (it has side effects: notifications, a contract), returning 201 or 200 with the updated application, or 409 if it is no longer pending.
*Follow-up:* Why not `/houses/:id/rooms/:id/applications/:id`? Deep nesting makes URLs brittle; once a resource has a unique ID, give it a top-level route.

**Q3. Which status codes do you return for create, update, delete, validation errors and missing resources?**
Create: 201 with `Location`. Update: 200 with the resource or 204. Delete: 204. Validation failure: 422 with field errors. Malformed request: 400. Missing, or not visible to this user: 404. Not allowed: 403. State conflict: 409.
*Follow-up:* When 202? When the work is queued and finishes later, with a status resource to poll.

**Q4. How do you design error responses?**
One consistent shape for every error, such as RFC 9457 problem details: `type`, `title`, `status`, `detail`, plus extensions like field-level `errors` with machine-readable codes and a `request_id`. Correct status codes, no stack traces or internals.
*Follow-up:* Why machine-readable codes? Clients branch and translate on codes; messages change.

**Q5. Offset vs cursor pagination: how does keyset pagination work?**
Offset uses `LIMIT/OFFSET`: page numbers and totals, but deep pages get slow and inserts cause duplicates or gaps. Keyset remembers the last row's sort key (encoded as an opaque cursor) and queries `WHERE (created_at, id) < (last values) ORDER BY created_at DESC, id DESC LIMIT n+1`, which an index serves at constant cost and which stays stable as data changes.
*Follow-up:* Why the `id` in the key? It breaks ties between equal timestamps so no row is skipped or repeated.

**Q6. How do you implement filtering and sorting safely?**
Query parameters (`?status=open&sort=-due_on`), mapped through allowlists to known columns and directions, with stable tie-breakers, capped page sizes and indexes for supported combinations. Never interpolate a field name from params into SQL.
*Follow-up:* Why can't you bind the sort column as a parameter? SQL parameters bind values, not identifiers, so identifiers must be allowlisted.

**Q7. How do you version an API, and what counts as a breaking change?**
URI versioning (`/api/v1`) is the most common and visible; header versioning keeps URLs clean but is less visible. Additive changes (new endpoints, new optional fields) need no new version; removing or renaming fields, changing types, meanings or defaults, or adding required fields are breaking.
*Follow-up:* How do you retire v1? Announce, send `Deprecation` and `Sunset` headers, measure usage, contact remaining consumers, then remove.

**Q8. How do you evolve an API when old mobile app versions stay installed?**
Keep changes additive and backward compatible, send `X-App-Version` and platform headers and log them, keep old fields until usage disappears, and enforce a minimum supported version that triggers a forced-update screen. JS-only fixes can go out through EAS Update to compatible builds; native changes need a store release.
*Follow-up:* What if you must break something now? Ship the new behavior under a new version or field, raise the minimum version, and keep the old path alive until the forced update rolls out.

**Q9. Is POST idempotent? How do idempotency keys work?**
No. The client sends a unique `Idempotency-Key` per logical operation and reuses it on retries; the server claims the key atomically, processes once, stores the status and body, and replays them for duplicates. In-flight duplicates get 409, and the same key with a different body gets 422.
*Follow-up:* Where do you store keys? A table with a unique index on (user, key) or Redis `SET NX` with an expiry, kept for around a day.

**Q10. How do you prevent lost updates when two users edit the same resource?**
Optimistic concurrency: return a version as an `ETag` (or `lock_version`), require `If-Match` on updates, and answer 412 (or 409) on a mismatch so the client refetches and merges. Use pessimistic row locks only for short, high-contention critical sections.
*Follow-up:* What does Rails give you? A `lock_version` column enables optimistic locking and raises `ActiveRecord::StaleObjectError` on stale writes.

**Q11. How would you rate limit an API, and what should clients do on 429?**
Count requests per API key, user or IP in Redis using a fixed window, sliding window or token bucket; return 429 with `Retry-After` and optionally limit headers. Clients honor `Retry-After`, back off exponentially with jitter and reduce concurrency.
*Follow-up:* Why jitter? So many clients that failed together do not retry at the same instant.

**Q12. An endpoint generates a report that takes two minutes. How do you design it?**
`POST /reports` validates, enqueues a background job and returns 202 with `Location: /reports/:id`; the job updates the status record; the client polls with backoff or receives a webhook or push; the result is a short-lived signed download URL. Make the POST idempotent.
*Follow-up:* Why not just raise the timeout? Proxies time out (504), threads stay busy, and retries start duplicate work.

**Q13. How do you design webhooks as a provider, and consume them safely?**
Provider: event envelopes with IDs, HMAC signatures over timestamp plus raw body, delivery from jobs with retries and backoff, at-least-once semantics, a delivery log with replay. Consumer: verify the signature with a constant-time comparison and a timestamp window, respond 2xx fast, process in a job, dedupe by event ID and tolerate out-of-order delivery.
*Follow-up:* Why is ordering not guaranteed? Retries and parallel delivery reorder events, so compare versions or refetch the resource.

**Q14. How would you design a bulk update endpoint?**
Accept a capped array, validate everything, load referenced records in one query, and choose atomic (one transaction) or partial success with per-item results in request order. Move very large batches to an async job with 202.
*Follow-up:* What do `insert_all` and `upsert_all` skip? Validations and callbacks.

**Q15. REST vs GraphQL vs RPC: when would you choose each?**
REST for public and integration APIs, CRUD resources, HTTP caching and webhooks. GraphQL when many clients need different shapes of deeply related data. RPC for internal or action-oriented calls: gRPC between services, tRPC or oRPC in TypeScript monorepos for end-to-end types.
*Follow-up:* Which did you use where? REST on the Rails side and oRPC in the new TypeScript platform during the Colochousing migration.

**Q16. What is HATEOAS, and do you use it?**
Hypermedia as the Engine of Application State: responses include links to related resources and to the actions allowed in the current state, so clients follow links instead of hardcoding URLs. Few APIs implement it fully; pagination links and `Location` headers are the common practical subset.
*Follow-up:* What is its benefit? The server can change URLs and expose state-dependent actions without breaking clients.

**Q17. What is OpenAPI, and how do you keep docs in sync with code?**
A machine-readable description of endpoints, parameters, schemas and responses that drives docs, generated clients and types, mocks and contract tests. Keep it in sync by generating it from tested code (rswag, drf-spectacular, zod-based OpenAPI tooling) or by validating requests and responses against the spec in CI.
*Follow-up:* Design-first or code-first? Design-first for public APIs with many consumers, code-first for internal APIs that move fast.

**Q18. How do you secure a REST API?**
HTTPS, authentication on every request, object-level authorization (scope every query to the current user or org), input validation and allowlisting, explicit response serializers, limits on page size, payload size and rate, secrets out of URLs and logs, tight CORS, and monitoring. The OWASP API Security Top 10 is a good checklist.
*Follow-up:* What is BOLA? Broken Object Level Authorization: changing an ID in the URL returns someone else's data because only authentication was checked.

**Q19. How do you represent money and timestamps in JSON?**
Money as integer minor units plus an ISO 4217 currency code (`4500` and `"EUR"`) or a decimal string, never a float. Timestamps as ISO 8601 in UTC (`2026-10-16T09:30:00Z`), plain dates as `YYYY-MM-DD`; convert to local time in the client.
*Follow-up:* Why not floats? Binary floating point cannot represent most decimal fractions exactly, so sums drift by fractions of a cent.

## Further reading
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) and [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OpenAPI Initiative](https://www.openapis.org/)
- [Rails guide: Using Rails for API-only applications](https://guides.rubyonrails.org/api_app.html)
