# How the Web Works: HTTP, Cookies, CORS and Caching

> **Tier:** T1 (know cold) · **Related:** [Networking essentials](../01-cs-fundamentals/06-networking.md), [REST API design](04-rest-api-design.md), [Authentication and authorization](06-authentication-authorization.md), [Web security and OWASP](07-web-security-owasp.md), [Caching and performance](08-caching-and-performance.md), [Browser DOM and events](../03-languages/javascript/06-browser-dom-events.md)
> **On your resume:** REST APIs at Devntech (including the Colochousing Rails API that serves the Coloc mobile app), Next.js APIs at RemoteDev, DRF APIs at PureLogics, and the Web Development labs you assisted at PUCIT.

## Quick revision
- HTTP is a stateless request/response protocol. A request has a method, a target (path and query), headers and an optional body; a response has a status code, headers and an optional body.
- **Safe** methods do not change server state (GET, HEAD, OPTIONS). **Idempotent** methods leave the same server state whether sent once or ten times (the safe ones plus PUT and DELETE). POST is neither; PATCH is not guaranteed to be idempotent.
- **PUT** replaces the whole resource at a known URL; **PATCH** applies a partial change.
- Status classes: 2xx success, 3xx redirect, 4xx client error, 5xx server error.
- **401** = not authenticated (missing, invalid or expired credentials). **403** = authenticated but not allowed. Return 404 instead when even admitting the resource exists would leak information.
- Redirects: 301/308 permanent, 302/307 temporary; 307 and 308 keep the method and body; 303 means "now GET this other URL" (Post/Redirect/Get).
- **304 Not Modified** answers a conditional request: the cached copy is still valid, no body is sent.
- 400 malformed request, 409 conflict with the current state, 422 well-formed but fails validation, 429 rate limited (send `Retry-After`).
- 502 = the proxy got a bad response or none from the app, 503 = temporarily unavailable (overload, maintenance), 504 = the app did not answer within the proxy timeout.
- Cookies add state: the server sends `Set-Cookie`, the browser returns `Cookie` on matching requests. A session is server-side state found through an ID in a cookie, or the whole session signed and encrypted inside the cookie.
- Cookie attributes: `Secure` (HTTPS only), `HttpOnly` (invisible to JavaScript), `SameSite` (`Strict`, `Lax`, `None`; `None` requires `Secure`), `Domain`, `Path`, `Expires`/`Max-Age`.
- **Origin** = scheme + host + port. **Site** = scheme + registrable domain, so `app.example.com` and `api.example.com` are same-site but cross-origin.
- The same-origin policy stops scripts from *reading* cross-origin responses; it does not stop requests from being *sent*.
- **CORS** is the server opting in with `Access-Control-Allow-*` headers. A JSON body, an `Authorization` header or a PUT/PATCH/DELETE triggers an `OPTIONS` **preflight** first.
- Cookies on a cross-origin `fetch` need `credentials: "include"`, `Access-Control-Allow-Credentials: true` and an exact origin (never `*`).
- Only browsers enforce CORS. curl, Postman, servers and native mobile apps ignore it, and it is not authentication.
- `Cache-Control`: `max-age` (fresh for N seconds), `no-cache` (store but revalidate before every use), `no-store` (never store), `private` vs `public`, `s-maxage` (shared caches), `immutable`, `stale-while-revalidate`.
- Validators: `ETag` + `If-None-Match`, `Last-Modified` + `If-Modified-Since`; a match returns 304 with no body.
- Content negotiation: `Accept`, `Accept-Language` and `Accept-Encoding` go in; `Content-Type`, `Content-Language`, `Content-Encoding` (gzip, br) and `Vary` come out.
- HTTP/2 multiplexes many requests over one TCP connection; HTTP/3 runs over QUIC on UDP. Methods, status codes and headers mean the same in every version.
- Debug with the DevTools Network tab (status, headers, timing, "Copy as cURL") and `curl -v`.

## 1. Requests and responses

HTTP is a client-server protocol. The client (browser, mobile app, curl, another server) sends a request, the server returns exactly one response, and neither side has to remember anything about earlier exchanges. In HTTP/1.1 the messages are plain text:

```text
POST /api/v1/houses/12/bills?notify=true HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiJ9.signature

{"title":"Internet","amount_cents":4500}
```

```text
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Location: /api/v1/bills/981
Cache-Control: no-store
X-Request-Id: 7c1e9d04-5b1f-4d0a-9a49-2f1b3f6e8c21

{"id":981,"title":"Internet","amount_cents":4500,"house_id":12}
```

The first line is the **request line** (method, target, protocol version) or the **status line** (version, code, reason phrase). Headers follow, one per line, then a blank line, then the optional body. Clients and frameworks add `Content-Length` (or stream with `Transfer-Encoding: chunked`) for you. HTTP/2 and HTTP/3 send the same information as binary frames, but the meaning is identical.

Anatomy of a URL such as `https://api.example.com:443/v1/houses/12/bills?status=open&page=2#summary`:

| Part | Example | Notes |
|---|---|---|
| Scheme | `https` | `http` defaults to port 80, `https` to 443 |
| Host | `api.example.com` | resolved through DNS and sent in the `Host` header |
| Port | `443` | usually implied by the scheme |
| Path | `/v1/houses/12/bills` | identifies the resource |
| Query | `status=open&page=2` | filters and pagination; percent-encoded; lands in logs and history, so never put secrets here |
| Fragment | `summary` | stays in the browser; never sent to the server |

## 2. Methods: safe, idempotent, and PUT vs PATCH

| Method | Typical use | Safe | Idempotent | Body |
|---|---|---|---|---|
| GET | read a resource or a collection | yes | yes | no |
| HEAD | like GET, headers only (existence, size, freshness) | yes | yes | no |
| OPTIONS | ask what is allowed; CORS preflight | yes | yes | rarely |
| POST | create a child resource or run an action | no | no | yes |
| PUT | replace (or create) the resource at this exact URL | no | yes | yes |
| PATCH | change part of a resource | no | not guaranteed | yes |
| DELETE | remove the resource | no | yes | rarely |

**Safe** means read-only from the client's point of view. Crawlers, link prefetchers and caches send safe requests freely, which is why a GET must never delete, log out or change anything: prefetchers have historically "clicked" destructive GET links in admin pages.

**Idempotent** is about the effect on server state, not the response. `DELETE /bills/981` returns 204 the first time and 404 the second, but the end state (the bill is gone) is the same. Browsers, proxies, HTTP client libraries and job retries may resend idempotent requests after a network failure; they must not blindly resend a POST, or you charge a card twice. Making POST retry-safe needs an idempotency key ([REST API design](04-rest-api-design.md)).

### PUT vs PATCH

```text
Current:   {"id": 7, "title": "Zelda", "price_cents": 4999, "condition": "used"}

PUT /listings/7     body {"title": "Zelda", "price_cents": 3999}
Result:    {"id": 7, "title": "Zelda", "price_cents": 3999}            condition is gone or reset

PATCH /listings/7   body {"price_cents": 3999}
Result:    {"id": 7, "title": "Zelda", "price_cents": 3999, "condition": "used"}
```

- **PUT** sends the complete new representation. Sending it twice gives the same result, so it is idempotent.
- **PATCH** sends a description of changes. With JSON Merge Patch (RFC 7396, `application/merge-patch+json`) you send the fields to change and `null` removes a field; setting absolute values like this is idempotent in practice. With JSON Patch (RFC 6902, `application/json-patch+json`) you send operations such as `{"op": "add", "path": "/tags/-", "value": "retro"}`, which appends again on every retry, so PATCH as a method cannot promise idempotency.
- In practice Rails routes both PUT and PATCH to `update` and treats them as partial updates (Rails forms send PATCH), and most JSON APIs do the same. In an interview, state the semantics first, then say what your framework does.

**POST vs PUT for creation:** POST to a collection when the server assigns the ID (`POST /listings` returns 201 with `Location: /listings/8`). Use PUT when the client already knows the target URL (`PUT /users/42/avatar`, `PUT /me/settings`, or a client-generated UUID), so repeating it is harmless.

## 3. Status codes

| Code | Name | When you send or see it |
|---|---|---|
| 200 | OK | successful GET, or an update that returns the resource |
| 201 | Created | POST created something; include `Location` and usually the new resource |
| 202 | Accepted | work queued for later, such as an export (see [REST API design](04-rest-api-design.md)) |
| 204 | No Content | success with no body: DELETE, some PUT/PATCH |
| 301 / 308 | Moved Permanently / Permanent Redirect | the URL changed for good (domain move, HTTP to HTTPS) |
| 302 / 307 | Found / Temporary Redirect | temporary redirect |
| 303 | See Other | after a POST: "go GET this page" |
| 304 | Not Modified | a conditional GET matched; reuse the cached copy |
| 400 | Bad Request | malformed JSON, missing required parameter, wrong type |
| 401 | Unauthorized | not authenticated: credentials missing, invalid or expired |
| 403 | Forbidden | authenticated, but not allowed to do this |
| 404 | Not Found | no such resource, or you are hiding that it exists |
| 405 | Method Not Allowed | e.g. DELETE on a read-only resource; send an `Allow` header |
| 409 | Conflict | duplicate natural key, stale version, invalid state transition |
| 412 | Precondition Failed | `If-Match` did not match (optimistic locking) |
| 415 | Unsupported Media Type | the server does not accept this body format |
| 422 | Unprocessable Content | valid syntax, failed validation (price negative, email taken) |
| 429 | Too Many Requests | rate limited; send `Retry-After` |
| 500 | Internal Server Error | unhandled exception in the app |
| 502 | Bad Gateway | the proxy could not get a valid response from the upstream app |
| 503 | Service Unavailable | overloaded, in maintenance or no healthy instances; may send `Retry-After` |
| 504 | Gateway Timeout | the upstream app took longer than the proxy's timeout |

### Redirects

| Code | Permanent | Method when followed | Use it for |
|---|---|---|---|
| 301 | yes | browsers turn POST into GET | permanent moves of pages; browsers cache it hard |
| 302 | no | browsers turn POST into GET | the classic temporary redirect (Rails `redirect_to` default) |
| 303 | no | always GET | Post/Redirect/Get after a form submission |
| 307 | no | method and body kept | temporary redirect of API calls |
| 308 | yes | method and body kept | permanent move of an API endpoint |

301 and 302 only rewrite POST to GET. A DELETE or PATCH sent with `fetch` and answered with 302 is re-sent as DELETE or PATCH to the new URL. That is why Rails apps using Turbo answer a destroy action with `redirect_to bills_path, status: :see_other` (303). Be careful with 301: browsers cache it, so a wrong permanent redirect can stick on users' machines long after you fix the server.

### 401 vs 403 (vs 404)

- **401 "Who are you?"** Credentials are missing, wrong or expired. The client should log in or refresh its token. The spec says a 401 should carry a `WWW-Authenticate` header naming the scheme (for example `Bearer`). Despite the name "Unauthorized", it is about authentication.
- **403 "I know who you are, and the answer is no."** A resident opening another house's bills, a member calling an admin-only endpoint. Logging in again does not help.
- **404 to hide existence.** GitHub answers 404 for private repositories you cannot see, so outsiders cannot even confirm they exist. In multi-tenant apps, another organization's records should usually look like they do not exist.
- A mobile client should react differently: on 401, try one token refresh and then send the user to the login screen; on 403, show "no access" and never loop on refresh.

### 400 vs 422 vs 409

- **400:** the server cannot parse or understand the request: invalid JSON, a missing required parameter (Rails raises `ActionController::ParameterMissing`, rendered as 400), a string where a number belongs.
- **422:** the request parsed fine but breaks business rules, i.e. validation errors. RFC 9110 renamed it from "Unprocessable Entity" to "Unprocessable Content"; Rails scaffolds and many APIs use it for model validation failures.
- **409:** the request conflicts with the resource's current state: approving an application that was already cancelled, a stale version number, booking a room that was just taken. Teams differ on uniqueness errors (409 or 422); pick one and be consistent.

### 502 vs 503 vs 504

In a typical deployment a load balancer or Nginx sits in front of Puma or Node.
- **502:** the proxy reached for the app and got a refused or reset connection or a garbage response: the app crashed, is still booting, or listens on a different port. Nginx logs `connect() failed (111: Connection refused) while connecting to upstream`.
- **503:** the service says "not now": maintenance mode, no healthy targets behind the load balancer, overload or platform limits.
- **504:** the app accepted the connection but did not answer within the proxy timeout: a slow query, a hanging third-party call, a report generated inside the request. Move slow work to a background job and set timeouts on outbound calls.

## 4. Headers you should know

| Header | Direction | Purpose |
|---|---|---|
| `Host` | request | which site to serve on a shared IP; required in HTTP/1.1 |
| `Content-Type` | both | media type of the body: `application/json`, `multipart/form-data`, `text/html; charset=utf-8` |
| `Accept` | request | media types the client wants back |
| `Authorization` | request | credentials: `Bearer <token>` or `Basic <base64 of user:password>` |
| `Cookie` / `Set-Cookie` | request / response | return stored cookies / ask the browser to store one |
| `Cache-Control` | both | caching rules (section 7) |
| `ETag` / `If-None-Match` | response / request | version tag of a representation / "send it only if it changed" |
| `Last-Modified` / `If-Modified-Since` | response / request | date-based validation |
| `If-Match` | request | "update only if the version is still this one" (optimistic locking) |
| `Location` | response | URL of the created resource (201) or the redirect target (3xx) |
| `Origin` | request | scheme, host and port of the page making a cross-origin or non-GET request; used by CORS and CSRF checks |
| `Referer` | request | the page that linked here (misspelled in the spec); limited by `Referrer-Policy` |
| `User-Agent` | request | client software and version |
| `Vary` | response | request headers that change the response, such as `Accept-Encoding` or `Origin` |
| `Retry-After` | response | seconds (or a date) to wait, with 429 and 503 |
| `X-Request-Id` | both | correlation ID to follow one request through proxy, app and job logs |

## 5. Statelessness, cookies and sessions

HTTP itself remembers nothing between requests, so "logged in" has to be carried by every request. Cookies are the browser's built-in mechanism:

```text
POST /session                          (email and password in the body)

HTTP/1.1 303 See Other
Location: /dashboard
Set-Cookie: _app_session=Zm9vYmFyYmF6; Path=/; Max-Age=1209600; Secure; HttpOnly; SameSite=Lax

GET /dashboard
Cookie: _app_session=Zm9vYmFyYmF6
```

The browser stores the cookie and attaches it automatically to every later request that matches its domain and path. "Automatically" is the key word: it is what makes cookies convenient, and it is also what makes CSRF possible ([Web security](07-web-security-owasp.md)).

### Server-side sessions vs cookie sessions

| | Server-side session | Cookie session (signed and encrypted) |
|---|---|---|
| The cookie holds | a random session ID | the session data itself |
| Data lives in | Redis, the database or memcached | the browser |
| Size limit | none in practice | about 4 KB per cookie |
| Revoking one session | delete the server record | hard: valid until it expires unless you also keep a version or denylist on the server |
| Scaling | every app server needs the shared store | nothing to share |
| Examples | Django's default (database sessions), Rails with a Redis or database store | Rails' default `CookieStore` |

Tokens in an `Authorization` header (often JWTs) are the third option, common for mobile apps and third-party clients. The full comparison of sessions vs JWTs lives in [Authentication and authorization](06-authentication-authorization.md).

### Cookie attributes

| Attribute | Example | Effect |
|---|---|---|
| `Domain` | `Domain=example.com` | sent to example.com and every subdomain; omit it for a host-only cookie, which is safer |
| `Path` | `Path=/` | sent only for URLs under this path |
| `Expires` / `Max-Age` | `Max-Age=1209600` | a persistent cookie with a lifetime (Max-Age wins if both are set); with neither it is a session cookie that dies with the browser session |
| `Secure` | `Secure` | sent only over HTTPS |
| `HttpOnly` | `HttpOnly` | hidden from `document.cookie`, so injected scripts cannot read and exfiltrate it (they can still send requests while the page is open) |
| `SameSite` | `SameSite=Lax` | whether the cookie travels on cross-site requests (below) |

### SameSite in practice

| Request to your site that starts on `https://evil.example` | Strict | Lax | None (with Secure) |
|---|---|---|---|
| User clicks a normal link (top-level GET navigation) | not sent | sent | sent |
| A form on that site POSTs to your site | not sent | not sent | sent |
| Background `fetch`, `<img>` or `<iframe>` request | not sent | not sent | sent |
| Any request that starts on your own site | sent | sent | sent |

- **Strict** is safest, but a user who clicks a link to your app from an email or Slack arrives logged out on that first page.
- **Lax** is the sensible default for session cookies: links from other sites still work, while cross-site form posts and background requests arrive without the cookie, which blocks most CSRF. Chromium-based browsers treat a cookie without the attribute as Lax, but set it explicitly rather than relying on defaults.
- **None; Secure** is for cookies that must work in a cross-site context: your app inside another site's iframe, an embeddable widget, a front end on a different site calling your API with cookies. Pair it with CSRF tokens. Safari and Firefox already block most third-party cookies by default and Chrome's plans have changed several times, so avoid designs that depend on them.

### Going deeper: cookie prefixes
A cookie named `__Host-session` is only accepted if it has `Secure`, `Path=/` and no `Domain`, so a compromised or careless subdomain cannot overwrite it. `__Secure-` only requires `Secure`. Both are cheap hardening for session cookies.

## 6. Same-origin policy and CORS

An **origin** is scheme + host + port. Compared with `https://app.example.com`:

| URL | Same origin? | Why |
|---|---|---|
| `https://app.example.com/bills?page=2` | yes | only path and query differ |
| `http://app.example.com` | no | scheme differs |
| `https://app.example.com:8443` | no | port differs |
| `https://api.example.com` | no (but same-site) | host differs |

The **same-origin policy (SOP)** lets a page embed cross-origin resources (images, scripts, stylesheets, iframes) and submit forms anywhere, but blocks JavaScript from *reading* cross-origin responses, cross-origin iframe DOMs and pixels of cross-origin images drawn on a canvas. Without it, any site you visit could call `fetch("https://yourbank.example/accounts")` with your cookies and read the result.

**CORS (Cross-Origin Resource Sharing)** is how a server relaxes the SOP for chosen origins. The browser adds `Origin: https://app.example.com` to the request, and only hands the response to JavaScript if the server answers with a matching `Access-Control-Allow-Origin`.

### Simple vs preflighted requests

A request is "simple" and goes straight out only if all of these hold:
- the method is GET, HEAD or POST;
- it only sets CORS-safelisted headers (`Accept`, `Accept-Language`, `Content-Language`, `Content-Type`);
- `Content-Type`, if set, is `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`.

Everything else is **preflighted**: the browser first asks permission with an `OPTIONS` request. In practice every JSON API call is preflighted, because `Content-Type: application/json` and `Authorization` are not safelisted.

```text
OPTIONS /api/v1/bills/981 HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: authorization, content-type

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Max-Age: 600
Vary: Origin
```

Then the real PATCH is sent, and its response must also carry `Access-Control-Allow-Origin`. Two consequences interviewers like:
- A **simple** request reaches your server and runs even when CORS fails; only the read is blocked. CORS therefore does not protect you from side effects of cross-site form posts. That is CSRF's job.
- A **failed preflight** stops the real request from being sent at all.

| Header | Sent on | Meaning |
|---|---|---|
| `Access-Control-Allow-Origin` | every CORS response | the allowed origin, or `*` for public resources without credentials |
| `Access-Control-Allow-Methods` | preflight response | methods allowed for the real request |
| `Access-Control-Allow-Headers` | preflight response | request headers allowed, e.g. `authorization`, `content-type`, `idempotency-key` |
| `Access-Control-Allow-Credentials` | both | `true` allows cookies or HTTP auth and lets JS read the credentialed response |
| `Access-Control-Max-Age` | preflight response | seconds the browser may cache the preflight result (browsers cap it; Chromium at two hours) |
| `Access-Control-Expose-Headers` | actual response | extra response headers JS may read, e.g. `Link`, `ETag`, `X-Total-Count` |

### Credentials
Cross-origin requests carry no cookies unless the client opts in with `fetch(url, { credentials: "include" })` (axios: `withCredentials: true`). The server must then answer with `Access-Control-Allow-Credentials: true` and the exact origin (a `*` is rejected when credentials are involved), plus `Vary: Origin` so caches do not hand one origin's response to another. The cookie must also be allowed to travel: same-site, or `SameSite=None; Secure`.

### Configuring CORS

```ruby
# Gemfile: gem "rack-cors"
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com", "http://localhost:5173"
    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options],
      expose: ["Link", "ETag"],
      credentials: true,
      max_age: 600
  end
end
```

```ts
import { Hono } from "hono";
import { cors } from "hono/cors";

const app = new Hono();
app.use(
  "/api/*",
  cors({
    origin: ["https://app.example.com", "http://localhost:5173"],
    allowMethods: ["GET", "POST", "PATCH", "DELETE"],
    allowHeaders: ["Authorization", "Content-Type"],
    exposeHeaders: ["Link", "ETag"],
    credentials: true,
    maxAge: 600,
  }),
);
```

### Debugging a CORS error
1. Read the console message: it names the missing or mismatched header.
2. Find the `OPTIONS` row in the Network tab. A 401, 404 or 500 on the preflight fails it; authentication middleware that rejects `OPTIONS` is a classic cause.
3. Compare `Access-Control-Allow-Origin` with the page origin exactly: scheme, host and port, no trailing slash.
4. With cookies: `credentials: "include"` on the client, `Allow-Credentials: true` and an explicit origin on the server, and a cookie that is allowed cross-site.
5. Error responses often skip the CORS middleware, so a 500 shows up as a "CORS error". Check the server logs before touching CORS config.
6. Fix it on the server, or serve the API from the same origin (a dev-server proxy or framework rewrites). `mode: "no-cors"` only gives you an opaque response you cannot read.

The Coloc React Native app calls the API without any CORS at all, because CORS is a browser rule; only browser front ends (including Expo web builds) need it.

## 7. HTTP caching

### Freshness and validation
While a cached response is **fresh** (younger than its `max-age`), the browser reuses it without contacting the server; DevTools shows "(disk cache)" or "(memory cache)". Once it is **stale**, the browser **revalidates** with a conditional request, and the server answers 304 (reuse it, no body) or 200 with a new body.

| Directive | Meaning |
|---|---|
| `max-age=600` | fresh for 600 seconds in any cache |
| `s-maxage=3600` | overrides `max-age` in shared caches (CDN, proxy) |
| `public` | shared caches may store it, even for a request that had `Authorization` |
| `private` | only the user's browser may store it |
| `no-cache` | may be stored, but must be revalidated before every use |
| `no-store` | must not be stored anywhere |
| `must-revalidate` | once stale, never serve it without revalidating, even offline |
| `immutable` | will not change while fresh; skip revalidation on reload |
| `stale-while-revalidate=30` | after expiry, serve the stale copy for up to 30 seconds while refreshing in the background |

`no-cache` does not mean "do not cache"; `no-store` does. If you send no `Cache-Control` at all, browsers may cache heuristically based on `Last-Modified`, so set it explicitly on API responses.

### ETag, Last-Modified and 304

```text
GET /api/v1/houses/12 HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: private, no-cache
ETag: "a1b2c3"
Content-Type: application/json

{"id":12,"name":"House 12","rooms":9}

GET /api/v1/houses/12 HTTP/1.1
If-None-Match: "a1b2c3"

HTTP/1.1 304 Not Modified
ETag: "a1b2c3"
```

- An **ETag** is an opaque version identifier: strong (`"a1b2c3"`) or weak (`W/"a1b2c3"`, meaning "equivalent", not byte-identical). Rails adds ETags to GET responses automatically from the body, and Express sends weak ETags by default.
- **Last-Modified** is date-based with one-second precision; when both are present, the ETag check wins.
- A body-hash ETag saves bandwidth, not server work: the app still built the response. An ETag computed from a version (`updated_at`, a lock version) lets the app skip the work entirely, which is what Rails `fresh_when` and `stale?` do. See [Rails caching](../05-ruby-on-rails/14-caching-performance-security.md).
- The same validators protect writes: `If-Match` on PUT/PATCH makes the server answer 412 if someone else changed the resource first.

### Policies that work

| Response | `Cache-Control` | Why |
|---|---|---|
| Fingerprinted assets (`application-4f9a1c.js`) | `public, max-age=31536000, immutable` | the URL changes whenever the content changes |
| HTML pages | `no-cache` (with an ETag) | always check, so users pick up new asset URLs after a deploy |
| Public API data (catalog, prices) | `public, max-age=60, s-maxage=300, stale-while-revalidate=30` | short browser life, longer CDN life |
| Per-user API data | `private, no-cache` | never in shared caches, always revalidated |
| Sensitive data (payments, tokens, personal data) | `no-store` | never written to disk |

`Vary` tells caches to store one copy per value of the listed request headers. `Vary: Accept-Encoding` is normal; `Vary: Cookie` or `Vary: User-Agent` destroys the hit rate. CDNs, cache keys and application caches are covered in [Caching and performance](08-caching-and-performance.md).

## 8. Content negotiation and compression

```text
GET /api/v1/houses HTTP/1.1
Accept: application/json
Accept-Language: fr-BE, fr;q=0.9, en;q=0.8
Accept-Encoding: gzip, br

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Language: fr
Content-Encoding: br
Vary: Accept-Encoding, Accept-Language
```

- The client lists what it can handle, weighted by `q` values (default 1); the server picks one and says what it chose.
- 406 Not Acceptable means the server cannot produce any acceptable type (many APIs ignore `Accept` and always return JSON); 415 Unsupported Media Type means it rejects the request body's type.
- Rails picks a format from the URL extension or `Accept` and renders through `respond_to { |format| format.html; format.json }`. Turbo asks for stream responses with `Accept: text/vnd.turbo-stream.html`.

**Compression:**
- gzip is universal and fast. Brotli (`br`) compresses text better, and browsers only advertise it over HTTPS; it shines for static assets precompressed at build time. Some recent browsers and CDNs also support zstd.
- Compress at the edge (CDN or Nginx), or with `Rack::Deflater` or Node `compression` middleware when nothing in front does it.
- Compress text (HTML, CSS, JS, JSON, SVG). Skip JPEG, PNG, WebP, video and zip files: they are already compressed, so you only burn CPU.
- Always send `Vary: Accept-Encoding` with compressed responses.
- **Going deeper:** compressing responses that mix secrets with attacker-controlled input over TLS enabled the BREACH attack. Rails masks its CSRF token differently on every request partly for this reason.

## 9. HTTP/1.1, HTTP/2 and HTTP/3 in one minute

- **HTTP/1.1:** text, one request at a time per connection, so browsers open several connections per host.
- **HTTP/2:** binary frames, many concurrent streams over one TCP connection, header compression. Old tricks such as domain sharding and sprite sheets matter less. One lost TCP packet still stalls every stream.
- **HTTP/3:** HTTP over QUIC on UDP, with TLS 1.3 built in, faster connection setup and no head-of-line blocking between streams; it helps most on lossy mobile networks.
- Usually the CDN or load balancer speaks HTTP/2 or HTTP/3 to browsers while Puma or Node speaks HTTP/1.1 behind it. Details: [Networking essentials](../01-cs-fundamentals/06-networking.md).

## 10. What happens when you type a URL (short version)

1. **Parse and check caches:** the browser splits the URL, applies HSTS (forces HTTPS), and may answer from its HTTP cache or a service worker.
2. **DNS:** the hostname resolves to an IP address, often a CDN edge (browser cache, OS cache, recursive resolver, then root, TLD and authoritative servers).
3. **Connect:** TCP handshake plus TLS handshake, or one QUIC handshake for HTTP/3. Existing connections are reused when possible.
4. **Request:** `GET /path` with `Host`, `Cookie`, `Accept`, `User-Agent` and friends.
5. **Edge and proxy:** a CDN may answer from cache; otherwise a load balancer and reverse proxy (TLS usually ends here) forward to an app server.
6. **Application:** middleware (logging, sessions, authentication), routing, controller or handler, database and cache queries, rendering HTML or JSON.
7. **Response:** status, headers (`Content-Type`, `Cache-Control`, `Set-Cookie`) and a usually compressed body.
8. **Render:** HTML is parsed into the DOM and CSS into the CSSOM; together they form the render tree, then layout, paint and compositing. CSS blocks rendering and plain `<script>` tags block parsing (hence `defer` and `async`). Images, CSS and JS are fetched (repeating steps 2 to 7, often over the same connection), JavaScript runs, and React mounts or hydrates.

Give this skeleton in about a minute, then offer to go deep on the part the interviewer cares about. DNS, TCP and TLS internals are in [Networking essentials](../01-cs-fundamentals/06-networking.md); the rendering pipeline, reflow and repaint are in [Browser DOM and events](../03-languages/javascript/06-browser-dom-events.md).

## 11. Tools: the DevTools Network tab and curl

**DevTools Network tab:**
- Filter by Fetch/XHR or Doc; tick **Preserve log** to keep requests across redirects and reloads, and **Disable cache** (only active while DevTools is open).
- Columns show status, type, initiator (the code that fired the request), size ("(disk cache)" means no network trip) and time.
- Click a request for **Headers** (general, response, request), **Payload** (query and body), **Preview/Response**, **Cookies** and **Timing**: queueing, DNS, initial connection, SSL, waiting for server response (time to first byte) and content download. A long wait points at the backend; a long download points at payload size.
- Right-click, **Copy as cURL**, to replay the exact request outside the browser. Throttling (Slow 4G, Offline) reproduces mobile conditions.
- CORS failures appear in the Console and as failed rows; the preflight shows up as its own `OPTIONS` request.

**curl:**

```bash
curl -i https://api.example.com/v1/houses                   # status line, response headers and body
curl -I https://example.com                                  # HEAD: headers only
curl -v https://api.example.com/up                           # verbose: DNS, TLS, request and response headers
curl -L http://example.com                                   # follow redirects
curl -X POST https://api.example.com/v1/bills \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Internet","amount_cents":4500}'
curl -i https://api.example.com/v1/houses/12 -H 'If-None-Match: "a1b2c3"'   # expect 304
curl -i -X OPTIONS https://api.example.com/v1/bills \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PATCH"                 # simulate a CORS preflight
curl -s -o /dev/null -w "%{http_code} ttfb=%{time_starttransfer}s total=%{time_total}s\n" https://example.com
curl -c cookies.txt -b cookies.txt https://app.example.com/ # save and resend cookies
```

On Windows, `curl` in Windows PowerShell 5.1 is an alias for `Invoke-WebRequest`; call `curl.exe` (bundled with Windows 10 and later) or use Git Bash or WSL. PowerShell 7 removed the alias.

## Common pitfalls
- Reading `no-cache` as "never cache" and using it for sensitive data instead of `no-store`.
- Changing state on GET (logout links, "delete" links): prefetchers, crawlers and CSRF all exploit it.
- Returning 200 with `{"error": "..."}`: clients, monitoring and caches all trust the status code.
- Mixing up 401 and 403, so a mobile app keeps refreshing tokens for what is really a permission problem.
- `Access-Control-Allow-Origin: *` with credentials (browsers reject it), or reflecting any `Origin` back with credentials (a security hole).
- Authentication middleware that rejects `OPTIONS` preflights with 401.
- Chasing a "CORS error" that is really a 500 returned without CORS headers.
- A 301 to the wrong place, cached in browsers long after the fix.
- Session cookies without `Secure`, `HttpOnly` and `SameSite`, or with a `Domain` when a host-only cookie would do.
- Long-cached HTML that still references old asset fingerprints after a deploy.
- Secrets or personal data in query strings: they end up in server logs, browser history and `Referer` headers.

## How to talk about this in an interview
- Anchor on the Colochousing REST API: one Rails 7 backend serves the web front end (React on Rails and ERB) and the Coloc mobile app. Natural talking points: why the mobile app never hits CORS, how 401 vs 403 should drive app behavior (refresh then log out, vs "no access"), which status codes validation errors return, and which caching headers API responses carry.
- On CrystalCommerce you were the HTTP *client*: background jobs calling the PriceCharting API need timeouts, retries only where safe, and handling of 429 and 5xx responses.
- The PUCIT Web Development labs are a good sign that you can explain request/response, forms and status codes simply; say so when asked to explain a basic concept.
- Show debugging habits: the Network tab, "Copy as cURL", replaying with curl, reading response headers and matching `X-Request-Id` in the logs.
- Use precise words: safe vs idempotent, origin vs site, fresh vs stale, revalidation.

> **Fill in:** how the Colochousing web front end and the Coloc app authenticate against the Rails API (session cookie, token, or both).

> **Fill in:** a bug you diagnosed from the Network tab or response headers (a CORS error, a wrong status code, a stale cache, a redirect loop) and what the fix was.

## Interview questions

**Q1. What happens when you type a URL into the browser and press Enter?**
The browser parses the URL, applies HSTS and checks its caches; DNS resolves the host; a TCP and TLS handshake (or QUIC) opens a connection; the browser sends an HTTP request with headers and cookies; a CDN, load balancer and reverse proxy route it to an app server, which runs middleware, routing, the controller, database queries and rendering; the response comes back with status, headers and body; the browser builds the DOM and CSSOM, lays out, paints and composites, fetches subresources and runs JavaScript. Then offer to go deep on one step.
*Follow-up:* Where do you look if the page is slow? The Timing tab first: DNS and connection (network), waiting for server response (backend), content download (payload size), then the Performance panel for rendering and JavaScript.

**Q2. What is the difference between safe and idempotent methods? Classify the common ones.**
Safe methods do not change server state (GET, HEAD, OPTIONS). Idempotent methods leave the same state no matter how many times the identical request is repeated: the safe ones plus PUT and DELETE. POST is neither, and PATCH is not guaranteed. Idempotency is about state, not responses: a second DELETE may return 404 while the state stays the same.
*Follow-up:* Why does it matter? Clients, proxies and job systems retry idempotent requests automatically; making POST retry-safe needs an idempotency key.

**Q3. PUT vs PATCH?**
PUT sends the full replacement representation for a known URL; omitted fields are reset, and repeating it is harmless, so it is idempotent. PATCH sends only the changes, as a merge patch or a list of JSON Patch operations. Rails maps both to `update` and treats them as partial updates.
*Follow-up:* Is PATCH ever idempotent? When it sets absolute values (merge patch) yes in practice; when it appends or increments, no.

**Q4. When would you create a resource with POST, and when with PUT?**
POST to the collection when the server assigns the ID, answering 201 with a `Location` header. PUT when the client already knows the URL, such as a client-generated UUID or a singleton like `/me/settings`; because PUT is idempotent, retries are safe.
*Follow-up:* What does a PUT that replaced an existing resource return? 200 with the body or 204 without; 201 only if it created something new.

**Q5. 401 vs 403?**
401 means the server does not know who you are: credentials are missing, invalid or expired, and logging in or refreshing a token can fix it. 403 means you are identified but not allowed, and re-authenticating will not help. Many APIs return 404 instead of 403 when even confirming the resource exists would leak information.
*Follow-up:* How should a mobile app react to each? 401: try one refresh, then go to login. 403: show an access message and stop.

**Q6. Explain 301, 302, 303, 307 and 308.**
301 and 308 are permanent, 302 and 307 temporary. Browsers turn a POST into a GET when following 301 or 302, while 307 and 308 keep the method and body. 303 always means "GET this other URL" and is the right answer after a form POST (Post/Redirect/Get).
*Follow-up:* Why do Rails apps with Turbo redirect destroy actions with 303? A 302 after a DELETE sent by `fetch` would be followed with another DELETE.

**Q7. Which status would you return for a malformed body, a validation failure and a double booking?**
Malformed JSON or a missing required parameter: 400. Parsed fine but fails validation (negative price): 422. Conflicts with current state (room already booked, stale version): 409. Whatever you choose for uniqueness errors, use it consistently and pair it with a structured error body.
*Follow-up:* What is wrong with 200 plus an error message? Clients, retries, monitoring and caches all read the status code, so failures look like successes.

**Q8. Users see 502s and 504s. What is the difference, and where do you look?**
502: the proxy could not get a valid response from the app (crashed, booting, wrong port, connection refused). 504: the app accepted the connection but did not respond within the proxy timeout (slow query, hanging external API). Check proxy logs, app logs, recent deploys and slow-request traces.
*Follow-up:* And 503? The service declares itself unavailable: maintenance, overload or no healthy instances.

**Q9. HTTP is stateless. How does a site keep you logged in?**
After login the server sets a cookie, and the browser sends it with every matching request. The cookie holds either a session ID pointing to server-side data (Redis or database) or the session itself, signed and encrypted (Rails `CookieStore`). APIs for mobile apps often use bearer tokens in the `Authorization` header instead.
*Follow-up:* Why do native apps usually use tokens? Browser cookie protections such as SameSite do not apply there, and explicit tokens in secure storage are simpler to manage across API domains.

**Q10. What do HttpOnly, Secure and SameSite do?**
`HttpOnly` hides the cookie from JavaScript, so an XSS payload cannot read and send it away. `Secure` sends it over HTTPS only. `SameSite` decides whether it travels on cross-site requests: `Strict` never, `Lax` only on top-level navigations such as link clicks, `None` always (and requires `Secure`).
*Follow-up:* Does HttpOnly stop XSS? No. It stops cookie theft, but the injected script can still make requests as the user while the page is open.

**Q11. Which SameSite value would you choose for a session cookie?**
`Lax` for most apps: links from other sites keep users logged in, while cross-site form posts and background requests carry no cookie, which blocks most CSRF. `Strict` for very sensitive apps that accept the "logged out after clicking an external link" effect. `None; Secure` only for genuine cross-site use (embeds, widgets), together with CSRF tokens.
*Follow-up:* Is `app.example.com` calling `api.example.com` cross-site? No, it is same-site (same registrable domain) but cross-origin, so SameSite does not block it while CORS still applies.

**Q12. What is the same-origin policy, and how does CORS relax it?**
The SOP lets pages embed cross-origin resources and submit forms anywhere, but stops scripts from reading cross-origin responses. CORS lets a server opt in: the browser sends `Origin`, and the server returns `Access-Control-Allow-Origin` (and related headers) for origins it trusts.
*Follow-up:* Does CORS secure your API? No. It protects users' browsers from other sites; curl and mobile apps ignore it, so the API still needs authentication and authorization.

**Q13. What triggers a CORS preflight, and what happens during it?**
Anything that is not a simple request: methods other than GET, HEAD and POST, non-safelisted headers such as `Authorization`, or a content type such as `application/json`. The browser sends `OPTIONS` with `Origin`, `Access-Control-Request-Method` and `Access-Control-Request-Headers`; the server answers with the allowed origin, methods, headers and an optional `Max-Age`; only then is the real request sent.
*Follow-up:* How do you reduce preflight overhead? `Access-Control-Max-Age` caches the answer, or serve the API from the same origin.

**Q14. A request works in Postman but fails in the browser with a CORS error. How do you fix it?**
Postman is not a browser, so it never enforces CORS. Inspect the preflight in the Network tab: it must succeed and return a matching origin, methods and headers; make sure auth middleware lets `OPTIONS` through; with cookies, use `credentials: "include"`, `Allow-Credentials: true` and an explicit origin. Check whether the real response was a 500 without CORS headers. Fix it on the server or proxy the API through the same origin.
*Follow-up:* Why not `mode: "no-cors"`? It returns an opaque response your code cannot read.

**Q15. How does HTTP caching work, and what is the difference between no-cache and no-store?**
A response is reused without a network trip while fresh (`max-age`, or `s-maxage` for shared caches); once stale it is revalidated with `If-None-Match` or `If-Modified-Since`, and the server replies 304 or a new 200. `no-cache` allows storage but forces revalidation before each use; `no-store` forbids storage entirely, which is what sensitive data needs.
*Follow-up:* How do you cache JS bundles safely? Fingerprinted file names with `public, max-age=31536000, immutable`, and `no-cache` on the HTML that references them.

**Q16. What is an ETag, and how does a 304 save work?**
An ETag is a version identifier for a representation. The client stores it and sends `If-None-Match` next time; if the version still matches, the server returns 304 with no body and the client reuses its copy. If the ETag comes from a record version rather than hashing the rendered body, the server can skip rendering too.
*Follow-up:* ETag vs Last-Modified? ETags are exact versions; `Last-Modified` is a date with one-second precision. Send both if you like; the ETag wins.

**Q17. How do content negotiation and compression work?**
The client sends `Accept`, `Accept-Language` and `Accept-Encoding` (with `q` weights); the server chooses and reports `Content-Type`, `Content-Language` and `Content-Encoding`, adding `Vary` so caches keep separate copies. Compression (gzip or Brotli) is normally done by the CDN or reverse proxy, for text responses only.
*Follow-up:* Why not compress images? JPEG, PNG and WebP are already compressed, so you waste CPU for no gain.

**Q18. HTTP/1.1 vs HTTP/2 vs HTTP/3, briefly?**
HTTP/1.1 is text with one request at a time per connection. HTTP/2 multiplexes binary streams over one TCP connection with header compression, but one lost packet stalls all streams. HTTP/3 runs over QUIC on UDP, removing that blocking and speeding up connection setup. Semantics are identical in all three.
*Follow-up:* Does your Rails app need to speak HTTP/2? Usually not; the CDN or load balancer does, and Puma speaks HTTP/1.1 behind it.

**Q19. How do you debug a failing API call from a web front end?**
Open the Network tab, find the request and read the status, request and response headers, payload and body; check the Console for CORS messages; confirm the token or cookie is actually sent; "Copy as cURL" and replay it to separate browser issues from server issues; then search the server logs by request ID and reproduce with curl.
*Follow-up:* What if it only fails in production? Compare environment-specific things: origins in the CORS config, cookie domain and `Secure`, HTTPS redirects, proxy timeouts and environment variables.

## Further reading
- [MDN: HTTP overview and reference](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) and [MDN: Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) and [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
