# Web Fundamentals: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## HTTP (see [How the web works](01-how-the-web-works.md))

**Q1. What is HTTP, and what does "stateless" mean?**
An application-layer request/response protocol: the client sends a method, a path, headers and an optional body; the server answers with a status code, headers and a body. Stateless means each request carries everything the server needs; continuity such as logins and carts is layered on top with cookies, sessions or tokens.

**Q2. What are the main HTTP methods?**
GET reads, HEAD returns only headers, POST creates or triggers processing, PUT replaces a resource, PATCH partially updates it, DELETE removes it, and OPTIONS asks what is allowed (browsers use it for CORS preflights).

**Q3. Safe vs idempotent methods?**
Safe methods do not change state (GET, HEAD, OPTIONS). Idempotent methods have the same effect whether sent once or many times (the safe methods plus PUT and DELETE). POST is neither, and PATCH is not guaranteed idempotent. It matters for retries: clients, proxies and job systems can safely repeat idempotent requests.

**Q4. PUT vs PATCH?**
PUT replaces the whole resource with the request body (fields you leave out are cleared or reset) and is idempotent. PATCH applies a partial change: JSON Merge Patch sends only the changed fields, JSON Patch sends a list of operations, and an operation like "append to this array" makes PATCH non-idempotent. Rails routes both to `update` and many APIs treat PUT like PATCH, so document what yours does.

**Q5. What are the status code classes, and which codes should you know?**
1xx informational, 2xx success, 3xx redirection, 4xx client error, 5xx server error. Know 200, 201 (with `Location`), 202, 204, 301, 302, 304, 400, 401, 403, 404, 409, 422, 429, 500, 502, 503 and 504.

**Q6. 301 vs 302 vs 307 vs 308?**
301 and 308 are permanent (browsers and search engines update the URL, and ranking signals move with it); 302 and 307 are temporary. 307 and 308 guarantee the method and body are kept, while with 301 and 302 browsers may turn a POST into a GET. Use 301 or 308 for URLs that moved in a migration, and 303 See Other to redirect after a form POST.

**Q7. 401 vs 403?**
401 Unauthorized actually means unauthenticated: no valid credentials, so log in or refresh the token (the response should include `WWW-Authenticate`). 403 Forbidden means the server knows who you are and you are not allowed. Some APIs return 404 instead of 403 so they do not reveal that a resource exists, such as another tenant's record.

**Q8. 400 vs 422?**
400 Bad Request: the request is malformed (invalid JSON, wrong types). 422 Unprocessable Content: well-formed but failing business validation (email already taken, end date before start date). Rails returns 422 for failed form saves, which Turbo needs in order to render the errors. Pick one convention and apply it consistently.

**Q9. 500 vs 502 vs 503 vs 504?**
500: the application itself failed (an unhandled exception). 502 Bad Gateway: the proxy got an invalid response or none (Puma crashed or is not listening). 503 Service Unavailable: temporarily down or overloaded, ideally with `Retry-After`. 504 Gateway Timeout: the upstream took longer than the proxy's timeout (a slow query or external call).

**Q10. 409 vs 412 vs 429?**
409 Conflict: the request conflicts with the current state (a duplicate, or an edit based on an old version). 412 Precondition Failed: an `If-Match` ETag no longer matches, which is optimistic concurrency over HTTP. 429 Too Many Requests: a rate limit was hit; send `Retry-After`.

**Q11. Which headers come up most often?**
Request: `Host`, `Authorization`, `Content-Type`, `Accept`, `Cookie`, `User-Agent`, `Origin`, `If-None-Match`. Response: `Content-Type`, `Cache-Control`, `ETag`, `Set-Cookie`, `Location`, `Retry-After`, `Strict-Transport-Security`, `Content-Security-Policy`.

**Q12. What is content negotiation?**
The client states preferences with `Accept` (and `Accept-Language`, `Accept-Encoding`); the server picks a representation and labels it with `Content-Type`. Rails `respond_to` blocks build on it. When one URL can return several variants, send `Vary` (for example `Vary: Accept-Encoding`) so caches store them separately.

## Cookies, sessions and CORS (see [How the web works](01-how-the-web-works.md) and [Authentication and authorization](06-authentication-authorization.md))

**Q13. What is a cookie, and what do its attributes do?**
A small name-value pair the server sets with `Set-Cookie` and the browser sends back automatically on matching requests (about 4 KB each). `HttpOnly` hides it from JavaScript, `Secure` sends it only over HTTPS, `SameSite` controls cross-site sending, `Domain` and `Path` scope it, and `Expires` or `Max-Age` make it persistent (otherwise it disappears when the browser session ends).

**Q14. SameSite Strict vs Lax vs None?**
Strict: never sent on cross-site requests, not even when the user follows a link (they arrive looking logged out). Lax: sent on top-level GET navigations, but not on cross-site POSTs, iframes or fetches. None: always sent, and it requires `Secure` (needed for third-party embeds). Chromium-based browsers treat a missing attribute as Lax, but set it explicitly rather than relying on defaults.

**Q15. How do server-side sessions work?**
On login the server stores session data (such as the user id) in a store (Redis, the database, memory) under a random session id and sends only that id in an `HttpOnly` cookie; every request looks it up. Rails' default cookie store works differently: it keeps the session data itself in an encrypted, signed cookie, so there is no server storage, but the 4 KB limit applies and you cannot revoke one session on the server.

**Q16. What is session fixation, and how do you prevent it?**
An attacker plants a session id they know in the victim's browser, waits for the victim to log in, then reuses it. Issue a new session id on login and privilege changes (Rails `reset_session`; authentication libraries usually do it, but verify) and keep the cookie `HttpOnly`, `Secure` and `SameSite`.

**Q17. What is the same-origin policy?**
Browsers let a page read responses only from its own origin: scheme plus host plus port. `http` vs `https`, `api.example.com` vs `app.example.com`, or a different port are all different origins. It stops a malicious site from reading your bank's pages through your logged-in browser.

**Q18. What is CORS, and how does a preflight work?**
CORS lets a server relax the same-origin policy for chosen origins through response headers. For non-simple requests (methods other than GET, HEAD and POST, custom headers such as `Authorization`, or `Content-Type: application/json`) the browser first sends an `OPTIONS` request with `Origin`, `Access-Control-Request-Method` and `Access-Control-Request-Headers`. Only if the server answers with a matching `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` does the real request go out; `Access-Control-Max-Age` lets the browser cache that answer.

**Q19. Why doesn't CORS protect your API?**
Browsers enforce it, and only for reading responses. curl, Postman, servers and scripts ignore it, and even in a browser a simple POST still reaches your server (only the response is hidden from the page). Protect APIs with authentication, authorization and CSRF defenses; CORS only decides which web origins may read responses in a browser.

**Q20. How do you send cookies on a cross-origin request?**
The client opts in (`fetch(url, { credentials: 'include' })`, or `withCredentials: true` in axios), and the server answers with `Access-Control-Allow-Credentials: true` plus an explicit origin in `Access-Control-Allow-Origin`, never `*`. If the two sites are cross-site, the cookie also needs `SameSite=None; Secure`.

**Q21. The console shows a CORS error. How do you debug it?**
Open the Network tab and inspect the preflight `OPTIONS` response: a missing or wrong `Access-Control-Allow-Origin`, a header absent from `Access-Control-Allow-Headers`, a redirect or auth failure on the preflight itself, or `*` combined with credentials. Fix the server configuration (`rack-cors` in Rails, the `cors` package in Express, Hono's `cors()` middleware) or avoid cross-origin calls with a same-origin proxy that forwards `/api` to the backend.

## HTML, CSS and accessibility (see [HTML and CSS](02-html-css.md))

**Q22. What is semantic HTML, and why does it matter?**
Elements that describe meaning: `header`, `nav`, `main`, `article`, `section`, `aside`, `footer`, `button`, `label`, `table`. Screen readers expose them as landmarks and roles, keyboards work for free (a `button` is focusable and activates on Enter and Space), and search engines understand the page structure.

**Q23. Why use a `button` instead of a `div` with a click handler?**
A `div` is not focusable, has no role and ignores Enter and Space, so matching a `button` takes `role="button"`, `tabIndex={0}` and key handlers. Use `button` for actions and `a` with an `href` for navigation.

**Q24. What is the first rule of ARIA?**
Do not use ARIA when a native element already provides the semantics and behavior. ARIA only changes what assistive technology announces; it adds no keyboard behavior. Common legitimate uses: `aria-label` on icon-only buttons, `aria-expanded` on disclosure toggles, `aria-live` for status messages, `aria-invalid` and `aria-describedby` for field errors.

**Q25. How do you make a form accessible?**
Every input has a `label` (via `for`/`htmlFor` or by wrapping it), related controls sit in a `fieldset` with a `legend`, errors are linked with `aria-describedby` and announced, required fields are marked in text and not only by color, and focus moves to the first invalid field on submit.

**Q26. What does WCAG require for contrast and keyboard use?**
At level AA, normal text needs a contrast ratio of at least 4.5:1, and large text and UI components at least 3:1. Every function must work with a keyboard, with a visible focus indicator and no keyboard traps, and information must not rely on color alone. Test with the keyboard only, a screen reader, and tools such as Lighthouse or axe.

**Q27. Explain the CSS box model and `box-sizing`.**
Every element is content plus padding plus border plus margin. With the default `content-box`, `width` sets only the content, so padding and border make the box wider than you asked; `border-box` includes padding and border in `width`, which is why most resets apply `box-sizing: border-box` to every element.

**Q28. How does specificity work?**
Inline styles beat id selectors, which beat classes, attribute selectors and pseudo-classes, which beat element selectors and pseudo-elements. When specificity ties, the later rule wins, and `!important` overrides normal declarations. Keeping selectors flat (single classes, as utility CSS and BEM do) avoids specificity wars.

**Q29. `display: none` vs `visibility: hidden` vs `opacity: 0`?**
`display: none` removes the element from layout and from the accessibility tree. `visibility: hidden` keeps its space but hides it from sight and from screen readers. `opacity: 0` keeps it in layout, clickable and announced. For "visually hidden but read by screen readers", use the clip pattern (Tailwind's `sr-only`).

**Q30. What do the CSS position values do?**
`static` is normal flow. `relative` offsets the element from its normal spot and makes it the containing block for absolutely positioned children. `absolute` removes it from flow and positions it against the nearest positioned ancestor. `fixed` positions it against the viewport. `sticky` behaves normally until a scroll threshold, then sticks within its container.

**Q31. Flexbox vs Grid?**
Flexbox lays out items along one axis (a row or a column): navbars, toolbars, centering. Grid controls rows and columns together: page layouts, card galleries, dashboards. They combine well: Grid for the page, Flexbox inside components.

**Q32. How do you center an element?**
On the parent, `display: flex; justify-content: center; align-items: center;` or `display: grid; place-items: center;`. To center a block with a set width horizontally, `margin-inline: auto`.

**Q33. rem vs em vs px vs viewport units?**
`px` is a fixed CSS pixel. `rem` is relative to the root font size, so it respects the user's font settings; it is the usual choice for type and spacing. `em` is relative to the element's own font size and compounds when nested. `vh` and `vw` are viewport percentages; on mobile, `dvh` follows the visible height as browser toolbars appear and disappear.

**Q34. What is mobile-first responsive design?**
Write base styles for small screens, then add `min-width` media queries for larger ones (Tailwind's `md:` and `lg:` prefixes work this way). Include `<meta name="viewport" content="width=device-width, initial-scale=1">`, or phones render a zoomed-out desktop layout.

**Q35. Why does `z-index: 9999` sometimes not work?**
`z-index` only competes within one stacking context. Positioned elements with a `z-index`, `opacity` below 1, `transform`, `filter` and a few other properties create a new context, and a child can never rise above its parent's siblings. Fix the parent's stacking context, or render overlays such as modals in a portal at the end of `body`.

**Q36. Plain `script` vs `defer` vs `async`?**
A plain script blocks HTML parsing while it downloads and runs. `defer` downloads in parallel and runs after parsing, in document order (best for application code). `async` downloads in parallel and runs as soon as it arrives, in any order (fine for independent scripts like analytics). `type="module"` scripts are deferred by default.

## Tailwind CSS (see [Tailwind CSS](03-tailwind-css.md))

**Q37. What is utility-first CSS, and why do teams like Tailwind?**
You style with small single-purpose classes in the markup (`flex gap-4 p-6 text-sm`) instead of writing custom CSS. You skip naming things, styles live next to the component, a design scale keeps spacing and colors consistent, and the generated CSS contains only the classes you use. The costs are long class strings and a learning curve; reuse comes from components, not from new CSS classes.

**Q38. How does Tailwind decide which classes to generate, and why does `bg-${color}-500` fail?**
It scans your source files as plain text for complete class names and generates CSS only for those. A class assembled at runtime never appears in the source, so no CSS exists for it. Map values to complete names instead: `const tone = { danger: 'bg-red-500', ok: 'bg-green-500' }[status]`.

**Q39. What changed in Tailwind v4?**
v4 (January 2025) moved configuration into CSS: `@import "tailwindcss";` plus a `@theme` block of CSS variables instead of `tailwind.config.js`, with automatic content detection. Some utilities were renamed (`shadow-sm` became `shadow-xs`, `outline-none` became `outline-hidden`) and some defaults changed (ring width, border color), so follow the official upgrade guide.

**Q40. How do responsive and state variants work?**
A prefix applies a utility under a condition: `md:grid-cols-3` from the `md` breakpoint upward (mobile-first), plus `hover:`, `focus-visible:`, `disabled:`, `dark:`, and `group-hover:` or `peer-invalid:` for parent and sibling state. Arbitrary values such as `w-[37px]` cover one-offs.

**Q41. Should you use `@apply`?**
Sparingly. It brings back the naming and indirection that utilities remove. Prefer extracting a component (`<Button variant="primary">`) or a class map in code, and keep `@apply` for HTML you do not control, such as CMS or third-party markup.

**Q42. How do you handle conditional and conflicting classes?**
Use `clsx` (or `classnames`) for conditionals and `tailwind-merge` so a caller's `p-2` cleanly overrides a component's default `p-4`; shadcn/ui wraps both in a `cn()` helper. Without merging, the winner depends on the order of rules in the stylesheet, not on the order of classes in the string.

**Q43. How do you use Tailwind in React Native?**
NativeWind turns Tailwind classes into React Native styles, so `className="flex-1 p-4"` works on a `View`. NativeWind v4 targets Tailwind v3, so check compatibility before upgrading either one.

## REST API design (see [REST API design](04-rest-api-design.md))

**Q44. What makes an API RESTful?**
Resources identified by URLs (nouns such as `/orders/42`), manipulated with standard HTTP methods and their semantics, stateless requests, representations (usually JSON) with standard status codes, and cacheable responses where safe. Strict REST also includes hypermedia links (HATEOAS), which most JSON APIs skip.

**Q45. How do you name endpoints?**
Plural nouns, with nesting for ownership: `GET /orders`, `POST /orders`, `GET /orders/42`, `PATCH /orders/42`, `GET /orders/42/items`. Avoid verbs like `/getOrders`. For actions that do not fit CRUD, use a sub-resource (`POST /orders/42/cancellation`) or a clearly named action route (`POST /orders/42/cancel`), and keep nesting to one level.

**Q46. What should create, update and delete endpoints return?**
Create: 201 Created with the resource and a `Location` header (or 202 Accepted if processing is asynchronous). Update: 200 with the updated resource, or 204 No Content. Delete: 204, or 200 with a body. A delete of a missing resource returns 404, or 204 if you treat "it is gone" as the idempotent end state.

**Q47. How do you design error responses?**
One consistent JSON shape with a machine-readable code, a human message and field details, for example `{"error": {"code": "validation_failed", "message": "...", "details": {"email": ["is taken"]}}}`, sent with the right status code. RFC 9457 Problem Details (`application/problem+json`) is a standard option. Never leak stack traces or SQL.

**Q48. Offset vs cursor pagination?**
Offset pagination (`?page=3&per_page=20`, `LIMIT 20 OFFSET 40`) is simple and supports jumping to page N, but the database still walks every skipped row (deep pages get slow), and inserts or deletes shift the rows, so items repeat or vanish between pages. Cursor (keyset) pagination returns an opaque cursor encoding the last row's sort key and queries `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20`: constant speed with a matching index and stable under writes, but no random page jumps. Use cursors for feeds, infinite scroll, sync endpoints and large tables.

**Q49. How do you version an API?**
URL versioning (`/api/v1/...`) is explicit and easy to route, test and cache; header or media-type versioning keeps URLs clean but is harder to use. Prefer additive, backward-compatible changes so new versions are rare, announce deprecations early (the `Sunset` header can carry the retirement date), and remember that old mobile app versions keep calling you for months.

**Q50. Which API changes break clients?**
Removing or renaming fields, changing types or formats, making optional inputs required, changing status codes or error shapes, tightening validation, and changing default sorting or page sizes. Adding endpoints and optional fields is usually safe; adding enum values can break strict clients, so document that clients must tolerate unknown values.

**Q51. What is an idempotency key, and when do you need one?**
A unique, client-generated key sent with a non-idempotent request such as creating a payment (Stripe uses an `Idempotency-Key` header). The server stores the key with the result; a retry with the same key returns the stored result instead of charging twice. Store keys with a unique constraint and an expiry, and reject a reused key that arrives with a different payload.

**Q52. How do you support filtering, sorting and sparse fields?**
With query parameters: `?status=paid&created_after=2025-01-01`, `?sort=-created_at,name`, `?fields=id,total`. Allowlist the sortable and filterable fields (never pass them raw into SQL), index the common combinations, and cap `per_page`.

**Q53. How do you handle long-running operations?**
Return 202 Accepted with a status resource (`Location: /exports/123`) and do the work in a background job. The client polls the status resource or receives a webhook or realtime event when it finishes. Holding an HTTP request open for minutes ends in proxy timeouts (504).

**Q54. How should an API communicate rate limits?**
Return 429 with `Retry-After` when a client exceeds its limit, and expose the remaining quota in headers (commonly `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; an IETF draft standardizes `RateLimit` header fields). Limit per API key or user rather than only per IP, and document the limits.

**Q55. Do you need HATEOAS?**
Rarely. Most JSON APIs sit at level 2 of the Richardson maturity model (resources plus HTTP methods) and describe their links in OpenAPI documentation instead. Know what it is, then explain why typed clients and good docs usually win in practice.

**Q56. How do you document an API?**
With OpenAPI (formerly Swagger): endpoints, schemas, authentication and errors in one spec that drives documentation UIs, client generation and contract tests. In Rails, rswag can generate it from request specs; in TypeScript, Zod-based tooling or oRPC's OpenAPI support can derive it from the same schemas you validate with.

## GraphQL (see [GraphQL](05-graphql.md))

**Q57. REST vs GraphQL?**
REST exposes many resource endpoints with server-defined responses; it gets HTTP caching and simple tooling almost for free. GraphQL serves a typed schema, usually from one endpoint, and clients request exactly the fields they need across related objects in one round trip: no over-fetching or under-fetching, strong typing and introspection, and schema evolution without versions. Its costs are harder HTTP caching, N+1 risks, query cost control and field-level authorization. GraphQL suits several clients with varied data needs over a rich domain; REST suits simple, cache-heavy or public APIs.

**Q58. Query vs mutation vs subscription?**
Queries read data (sibling fields may resolve in parallel); mutations write data (top-level mutation fields run one after another); subscriptions push updates to the client over a long-lived connection, usually a WebSocket.

**Q59. What are resolvers?**
Functions that produce the value of a single field. Each receives the parent object, the field's arguments, a shared context (current user, loaders) and query info. The engine walks the query and calls a resolver per field, which is exactly why naive resolvers cause N+1 queries.

**Q60. How do you solve N+1 queries in GraphQL?**
Batch and cache per request with a data loader: each resolver calls `loader.load(userId)`, and the loader collects the ids requested during one tick and runs a single `WHERE id IN (...)` query. graphql-ruby ships `GraphQL::Dataloader` (graphql-batch is the older gem), and JavaScript servers use the `dataloader` package.

**Q61. Over-fetching vs under-fetching?**
Over-fetching: an endpoint returns more fields than the screen needs, wasting bandwidth, which hurts on mobile networks. Under-fetching: one endpoint is not enough, so the client chains requests (`/users/1`, then `/users/1/orders`). GraphQL addresses both by letting the client shape the response.

**Q62. Why is caching harder with GraphQL?**
Queries usually go out as POST requests to a single URL, so HTTP caches and CDNs cannot key on the URL. Clients instead keep a normalized cache by type and id (Apollo Client, urql, Relay), and servers use persisted queries (a hash sent with GET, which is cacheable) plus field-level caching.

**Q63. How does GraphQL report errors?**
Typically with HTTP 200 and a response holding `data` (possibly partial) plus an `errors` array with messages and paths. Many schemas model expected failures as data instead (union result types or `userErrors` fields) so clients handle them through types. Monitoring must read response bodies, not just status codes.

**Q64. How do you secure a GraphQL API?**
Authenticate at the endpoint and authorize in the business layer or per field (never trust the client to hide fields), limit query depth and complexity, paginate every list, add timeouts and cost-based rate limits, and for public APIs consider allowlisting persisted queries or disabling introspection.

**Q65. How do you evolve a GraphQL schema without versioning?**
Add fields freely, mark old ones with `@deprecated(reason: "...")`, watch field usage metrics, and remove a field only when clients stop requesting it. Never change a field's type or meaning in place; add a new field instead.

**Q66. How does pagination work in GraphQL?**
The common convention is Relay-style connections: `users(first: 20, after: $cursor) { edges { node { id } cursor } pageInfo { hasNextPage endCursor } }`. It is cursor pagination with a uniform shape that client libraries understand.

## Authentication and authorization (see [Authentication and authorization](06-authentication-authorization.md))

**Q67. Authentication vs authorization?**
Authentication proves who you are (a password, an OAuth login, a passkey); authorization decides what you may do (roles, ownership, tenant scope). Failures map to different responses: 401 for authentication, 403 (or 404) for authorization.

**Q68. How should passwords be stored?**
Hashed with a slow, salted, adaptive algorithm (bcrypt, scrypt or Argon2id); never encrypted and never hashed with a fast hash such as SHA-256. The salt defeats precomputed tables and the work factor slows brute force. Rails `has_secure_password` uses bcrypt; Django defaults to PBKDF2 and supports Argon2 and bcrypt.

**Q69. JWT vs server-side sessions?**
Sessions: an opaque id in a cookie, with state on the server. Revocation is easy and it fits browser apps naturally, but you need a shared store once you run several servers (Redis or the database). JWT: a signed token carrying claims that any server can verify without a lookup, which suits mobile clients and service-to-service calls; but it cannot be revoked before it expires without extra machinery, it is larger, and stored in `localStorage` it can be stolen by XSS. A common middle ground is short-lived access tokens (5 to 15 minutes) plus rotating refresh tokens tracked on the server; for a browser-only app, a session cookie is often the simplest secure choice.

**Q70. What is inside a JWT?**
Three base64url-encoded parts separated by dots: a header (`alg`, `typ`), a payload of claims (`sub`, `exp`, `iat`, `iss`, `aud`, plus custom ones like `role`) and a signature. It is signed, not encrypted: anyone can decode the payload, so never put secrets in it. HS256 signs and verifies with one shared secret; RS256 and ES256 sign with a private key and verify with a public key, so many services can verify tokens without being able to mint them.

**Q71. What are common JWT mistakes?**
Skipping signature verification, accepting `alg: none` or whatever algorithm the token claims, long lifetimes with no revocation path, not checking `aud` and `iss`, putting sensitive data in the payload, and storing tokens where injected scripts can read them. Pin the expected algorithm when verifying and keep access tokens short-lived.

**Q72. Where should a web app keep its tokens?**
Preferably in an `HttpOnly`, `Secure`, `SameSite` cookie, which JavaScript cannot read (you then need CSRF protection). `localStorage` is readable by every script on the page, so one XSS bug leaks the token. A common SPA pattern keeps the access token in memory and the refresh token in an `HttpOnly` cookie. On mobile, use the platform keystore (for example Expo SecureStore), not AsyncStorage.

**Q73. How do refresh tokens work, and what is rotation?**
The access token is short-lived; when it expires, the client sends the long-lived refresh token to get a new pair. With rotation each refresh token works only once: the server issues a new one and invalidates the old, and if an old token is ever presented again, that signals theft, so the whole token family is revoked.

**Q74. Explain OAuth 2.0 and the Authorization Code flow with PKCE.**
Roles: the resource owner (the user), the client (your app), the authorization server (for example Google) and the resource server (the API). The client generates a random `code_verifier` and sends its SHA-256 hash as `code_challenge` (with `code_challenge_method=S256`) and a random `state` in the authorization request; the user logs in and consents; the authorization server redirects back with a short-lived `code`; the client exchanges the code plus the original `code_verifier` for tokens. An attacker who intercepts the code cannot redeem it without the verifier, which is why PKCE is required for public clients such as SPAs and mobile apps (which cannot keep a client secret) and recommended for every client.

**Q75. Why was the implicit flow abandoned?**
It returned access tokens directly in the redirect URL, exposing them to browser history, referrer headers and injected scripts, with no way to bind them to the client. The Authorization Code flow with PKCE gives browser and mobile apps the same capability safely, and current OAuth security guidance tells you not to use implicit.

**Q76. What does the OAuth `state` parameter protect against?**
CSRF on the callback. The client generates a random value, remembers it (in the session or in memory) and checks that the callback returns the same value, so an attacker cannot make your browser complete a login with the attacker's own authorization code.

**Q77. OAuth 2.0 vs OpenID Connect?**
OAuth 2.0 is delegated authorization: an access token lets an app call an API on the user's behalf. OIDC adds authentication on top: requesting the `openid` scope returns an ID token (a JWT with claims such as `sub`, `email`, `aud` and `nonce`) that tells the client who logged in, plus a userinfo endpoint and discovery metadata at `/.well-known/openid-configuration`. "Sign in with Google" is OIDC.

**Q78. Which OAuth flow fits which client?**
Server-rendered web apps, SPAs and mobile apps: Authorization Code with PKCE. Machine-to-machine calls with no user: Client Credentials. TVs and CLIs without a convenient browser: the Device Authorization flow. Avoid the legacy password grant.

**Q79. RBAC vs ABAC?**
RBAC grants permissions through roles (admin, manager, member): simple to reason about and audit. ABAC decides from attributes of the user, the resource and the context (`user.org_id == record.org_id && record.status == "draft"`): flexible but harder to audit. Real apps mix them, with roles plus ownership and tenant checks, enforced in policy objects (Pundit, CanCanCan, DRF permission classes).

**Q80. Where should authorization checks live?**
On the server, in one consistent layer (policies, permission classes, procedure middleware) applied to every endpoint, and inside queries by scoping records (`current_org.projects.find(params[:id])`). Hiding a button in the UI is user experience, not security.

**Q81. What is MFA, and which second factors are strongest?**
Multi-factor authentication combines things you know, have and are. Authenticator apps (TOTP) beat SMS codes, which SIM swapping defeats; hardware security keys and passkeys (WebAuthn) are phishing-resistant because each credential is bound to the site's origin.

## Web security and OWASP (see [Web security and OWASP](07-web-security-owasp.md))

**Q82. What is the OWASP Top 10?**
A ranking of the most critical web application risk categories. The widely quoted 2021 edition: broken access control, cryptographic failures, injection, insecure design, security misconfiguration, vulnerable and outdated components, identification and authentication failures, software and data integrity failures, security logging and monitoring failures, and server-side request forgery. A 2025 update reorganizes several categories (it adds software supply chain failures, for example), so check the OWASP site for the current list before quoting exact rankings.

**Q83. XSS vs CSRF?**
XSS (cross-site scripting): an attacker gets their script to run inside your pages, so it can do anything the user can: read the page, call your APIs, steal tokens that are not `HttpOnly`. CSRF (cross-site request forgery): another site makes the victim's browser send a request to your site with their cookies attached, performing an action without ever reading the response. XSS defeats CSRF protection, because injected script can read the CSRF token. Prevent XSS with output escaping, sanitization and CSP; prevent CSRF with SameSite cookies and CSRF tokens.

**Q84. Stored vs reflected vs DOM-based XSS?**
Stored: the payload is saved (a comment, a profile field) and served to other users. Reflected: it comes from the request, such as a query parameter echoed into the page. DOM-based: client-side JavaScript writes untrusted data into the DOM (`element.innerHTML = location.hash`) without the server being involved.

**Q85. How do React and Rails protect against XSS, and how do developers break it?**
Both escape interpolated values by default (`{value}` in JSX, `<%= %>` in ERB). The escape hatches cause the bugs: `dangerouslySetInnerHTML`, `html_safe`, `raw` and `<%== %>` with user input, `href={userUrl}` that allows `javascript:` URLs, and HTML strings built by hand. Sanitize rich text against an allowlist (DOMPurify in the browser, Rails `sanitize`).

**Q86. How do CSRF tokens work in Rails?**
`protect_from_forgery` (enabled by default) embeds a per-session token in every form and in `csrf_meta_tags` for JavaScript, and non-GET requests must send it back as a parameter or an `X-CSRF-Token` header. A cross-site attacker cannot read the token, so forged requests fail. Never change state on GET, because GET requests are not checked.

**Q87. Is a JSON API that uses bearer tokens vulnerable to CSRF?**
Not to classic CSRF: browsers never attach an `Authorization` header automatically, so a forged request arrives unauthenticated. The risk returns as soon as authentication moves into cookies, so cookie-based APIs need SameSite plus CSRF tokens or `Origin` checks.

**Q88. What is SQL injection, and how do you prevent it?**
User input concatenated into SQL changes the query's meaning (`' OR 1=1 --`). Use parameterized queries everywhere: `where(email: params[:email])` or `where("email = ?", params[:email])` in Rails, placeholders in raw SQL, never string interpolation. ORMs still have raw escape hatches (`order` with strings, `find_by_sql`, `Arel.sql`, Drizzle's `sql.raw`), so allowlist column names for sorting.

**Q89. What is a Content Security Policy?**
A response header that tells the browser which sources may provide scripts, styles, images, frames and connections, for example `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-r4nd0m'; object-src 'none'; frame-ancestors 'none'`. A strict policy blocks injected inline scripts even when an XSS bug exists. Roll it out with `Content-Security-Policy-Report-Only` first; Rails has a `content_security_policy` configuration DSL with nonce support.

**Q90. What is clickjacking?**
An attacker loads your page in an invisible iframe and tricks users into clicking real buttons on it. Forbid framing with CSP `frame-ancestors 'none'` (or a list of allowed parents) or the older `X-Frame-Options: DENY`.

**Q91. What is IDOR?**
Insecure direct object reference: the app trusts an id from the request without checking ownership, so `/invoices/1002` shows another customer's invoice. Scope every lookup to the current user or tenant (`current_user.invoices.find(params[:id])`) and test cross-account access. It falls under broken access control, the top OWASP category.

**Q92. What is mass assignment?**
Passing request parameters straight into a model lets attackers set fields they should not (`role=admin`, `org_id`). Allowlist fields instead: Rails strong parameters (`params.require(:user).permit(:name)`, or `params.expect(user: [:name])` in Rails 8.0+), explicit DRF serializer `fields`, or a Zod object schema, which strips unknown keys by default.

**Q93. What is SSRF?**
Server-side request forgery: your server fetches a URL the user supplies (webhook targets, image imports, link previews), and the attacker points it at internal services or the cloud metadata endpoint (`169.254.169.254`) to steal credentials. Allowlist schemes and hosts, resolve the hostname and block private and link-local ranges, disable redirects, and run fetchers with minimal network access.

**Q94. Which security headers should every app send?**
`Strict-Transport-Security` (forces HTTPS on future visits), `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, a `Referrer-Policy` such as `strict-origin-when-cross-origin`, frame protection (`frame-ancestors` or `X-Frame-Options`) and a restrictive `Permissions-Policy`. Rails sends several of these by default; verify yours in the browser's developer tools.

**Q95. How do you handle secrets?**
Never commit them: use environment variables or encrypted credentials (Rails `credentials.yml.enc`, with the master key kept out of the repository), a secrets manager in production, separate keys per environment, least privilege and regular rotation. If a secret gets pushed, rotate it first; cleaning git history comes second, because clones and forks keep it.

**Q96. How do you protect a login endpoint from brute force?**
Rate limit by IP and by account (Rails 7.2+ controller `rate_limit`, rack-attack, or an API gateway), add increasing delays or temporary lockouts carefully (lockouts can be abused for denial of service), offer MFA, return the same generic error for unknown users and wrong passwords, and monitor for credential stuffing.

## Caching and performance (see [Caching and performance](08-caching-and-performance.md))

**Q97. What do the main `Cache-Control` directives mean?**
`max-age=N`: fresh for N seconds. `s-maxage`: the same for shared caches such as CDNs. `public` and `private`: whether shared caches may store it. `no-cache`: may be stored but must be revalidated before each use. `no-store`: never store it (sensitive data). `immutable`: never revalidate while fresh. `stale-while-revalidate=N`: serve a stale copy while refreshing in the background.

**Q98. How do ETags and 304 responses work?**
The server sends an `ETag` (a fingerprint of the representation) or a `Last-Modified` date; the browser revalidates with `If-None-Match` or `If-Modified-Since`, and if nothing changed the server replies `304 Not Modified` with an empty body. Rails `fresh_when(@product)` and `stale?` implement this. It saves bandwidth and rendering, not the round trip.

**Q99. What is a sensible caching strategy for a typical web app?**
Fingerprinted assets (`application-3f9a1c.js`) get `Cache-Control: public, max-age=31536000, immutable`, because every deploy changes their names. HTML gets `no-cache` or a short max-age so users pick up new asset references quickly. Personalized API responses get `private` or `no-store`. On the server, add fragment, query and object caching (Redis, Solid Cache) with clear invalidation rules.

**Q100. What cache layers sit between a user and the database?**
The browser cache, a service worker, the CDN edge, a reverse proxy cache, application caches (fragments and computed objects in Redis or Memcached), the ORM's per-request query cache, and the database's own buffer cache. Each needs its own key and invalidation strategy, including user, locale and tenant in keys where relevant.

**Q101. What are the Core Web Vitals and their thresholds?**
Google's user-centric metrics, judged at the 75th percentile of real page loads: LCP (Largest Contentful Paint, loading) is good at 2.5 s or less; INP (Interaction to Next Paint, responsiveness) is good at 200 ms or less; CLS (Cumulative Layout Shift, visual stability) is good at 0.1 or less. INP replaced FID in March 2024. Field data (CrUX, or the `web-vitals` library feeding your analytics) is what counts; Lighthouse lab runs help you diagnose.

**Q102. How do you improve LCP?**
Find the LCP element, usually a hero image or a headline. Cut server time (caching, a CDN, faster queries), never lazy-load the LCP image, preload it or give it `fetchpriority="high"`, serve modern formats (WebP, AVIF) at the right size with `srcset`, inline critical CSS and defer non-critical JavaScript, and avoid rendering above-the-fold content only on the client.

**Q103. How do you improve INP?**
Keep the main thread free: split long tasks (over 50 ms), ship less JavaScript (code splitting, dropping heavy dependencies), avoid expensive re-renders on input (memoization, `useTransition`, virtualized lists), debounce costly handlers, and move heavy computation into a Web Worker. Show visual feedback immediately, before slow work starts.

**Q104. How do you fix CLS?**
Reserve space: set `width` and `height` (or `aspect-ratio`) on images, videos and embeds, keep fixed slots for ads and banners, avoid inserting content above existing content unless the user asked for it, limit font swaps with `font-display` and similar fallback fonts, and animate `transform` instead of layout properties.

**Q105. Lab data vs field data?**
Lab data (Lighthouse, WebPageTest) comes from a controlled run: reproducible and great for debugging, but it cannot observe real interactions, so Lighthouse reports Total Blocking Time as a proxy for INP. Field data (real user monitoring, CrUX) comes from real users on real devices and networks, and it is what Core Web Vitals assessments use.

**Q106. Describe the critical rendering path.**
The browser parses HTML into the DOM and CSS into the CSSOM, combines them into the render tree, computes layout (geometry), paints pixels and composites layers. CSS blocks rendering and synchronous scripts block parsing, so inline critical CSS, defer scripts and keep the DOM lean.

**Q107. Reflow vs repaint?**
A reflow (layout) recalculates geometry after changes to size, position or content, and it can cascade through the page; a repaint redraws pixels after visual-only changes such as color. Animating `transform` and `opacity` can skip both and run on the compositor. Avoid layout thrashing: do not interleave DOM reads (`offsetHeight`) and writes in a loop.

**Q108. Which frontend techniques reduce load time?**
Code splitting and lazy loading (`import()`, `React.lazy`, route-based chunks), tree shaking, Brotli or gzip compression, optimized images with `loading="lazy"` below the fold, HTTP/2 or HTTP/3, a CDN, `preconnect` to critical third-party origins, and fewer, lighter third-party scripts, which are often the biggest cost.

## Realtime: WebSockets and SSE (see [Realtime and WebSockets](09-realtime-websockets.md))

**Q109. Short polling vs long polling vs SSE vs WebSockets?**
Short polling asks every N seconds: simple, wasteful and delayed. Long polling holds each request open until data arrives, then the client reconnects. Server-Sent Events stream events from server to client over one long-lived HTTP response. WebSockets give a full-duplex connection where either side sends messages at any time. Pick the simplest one that meets your latency and direction needs.

**Q110. How does a WebSocket connection start?**
With an HTTP/1.1 GET carrying `Upgrade: websocket`, `Connection: Upgrade` and a `Sec-WebSocket-Key`; the server answers `101 Switching Protocols` with `Sec-WebSocket-Accept`, and from then on the same TCP connection carries WebSocket frames (`wss://` runs over TLS). Ping and pong frames keep it alive.

**Q111. When do you choose SSE over WebSockets?**
When data flows only from server to client: notifications, live dashboards, progress updates, streamed LLM tokens. SSE is plain HTTP (existing auth, proxies and HTTP/2 just work), and `EventSource` reconnects automatically and sends `Last-Event-ID` so the server can resume. Choose WebSockets for chat, collaborative editing, games and other bidirectional, high-frequency traffic. Over HTTP/1.1, browsers allow only about six connections per host, which limits SSE across many tabs.

**Q112. How do you scale WebSockets across several servers?**
Each connection lives on one server process, so a message published on server A must also reach clients connected to server B: use a pub/sub backplane (Redis for Action Cable and the Socket.IO adapter, or a managed realtime service). Also plan for per-process connection limits, load balancer idle timeouts, sticky sessions where a protocol needs them (Socket.IO's long-polling fallback), and mass reconnections during deploys.

**Q113. How do you authenticate a WebSocket connection?**
During the handshake: browsers send cookies with the upgrade request (Action Cable identifies `current_user` from the session or a signed cookie in `ApplicationCable::Connection`), or the client sends a short-lived token in its first message. Avoid long-lived tokens in the query string, because URLs end up in logs. Check the `Origin` header, and authorize every channel or subscription separately.

**Q114. How do clients handle disconnects and missed messages?**
Reconnect with exponential backoff plus jitter (so thousands of clients do not reconnect at the same instant after a deploy), detect dead connections with heartbeats, and resync on reconnect: fetch the events missed since the last received id or sequence number, or refetch the current state. Make message handling idempotent, because duplicates happen.

**Q115. How do Rails apps do realtime?**
Action Cable provides WebSocket channels on the server and a JavaScript consumer on the client, and Turbo Streams can broadcast model changes (`broadcasts_to` in the model, `turbo_stream_from` in the view) that update pages without custom JavaScript. Rails 8.0 added Solid Cable, a database-backed adapter, as an alternative to Redis.
