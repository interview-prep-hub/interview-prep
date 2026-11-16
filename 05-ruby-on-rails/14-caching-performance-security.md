# Caching, Performance and Security in Rails

> **Tier:** T1 (know cold) · **Related:** [Caching and performance concepts](../04-web-fundamentals/08-caching-and-performance.md), [Web security (OWASP)](../04-web-fundamentals/07-web-security-owasp.md), [Querying and N+1](05-activerecord-querying-n-plus-1.md), [Views and forms](07-views-erb-forms-assets.md), [Authentication and authorization](11-authentication-authorization.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md), [Redis](../12-databases/10-redis.md), [Observability and debugging](../14-devops-and-tools/05-observability-and-debugging.md)
> **On your resume:** performance and security work in legacy Rails monoliths at Devntech, CrystalCommerce's catalog at scale (700+ stores, 8M+ customers, PriceCharting market prices), and the Colochousing Rails 7 web app and API.

## Quick revision
- Caching layers: HTTP (ETags, `Cache-Control`), fragment (`cache` in views), low-level (`Rails.cache.fetch`), and the per-request SQL query cache. Page and action caching left core in Rails 4.
- Fragment keys combine the template digest and the record's `cache_key`; the record's `cache_version` (its `updated_at`) is stored with the entry, so a changed record or template simply misses.
- **Russian doll caching:** nested fragments plus `belongs_to ..., touch: true`, so a child change refreshes every parent key while unchanged siblings are reused. `render partial:, collection:, cached: true` fetches a collection in one `read_multi`.
- `Rails.cache.fetch(key, expires_in:, race_condition_ttl:) { ... }`: cache plain data (not ActiveRecord objects), scope keys by tenant and user, and give every key an expiry or a version.
- Stores: memory (per process), Memcached, Redis, Solid Cache (Rails 8 default, database-backed and disk-sized), null (tests). Never share an evicting cache Redis with Sidekiq.
- HTTP caching: `fresh_when(@record)` / `stale?(@record)` send ETag and Last-Modified and answer 304 when the client's copy is current; `expires_in 1.hour, public: true` for shared caches.
- Performance method: measure (logs, rack-mini-profiler, APM, pg_stat_statements), fix the biggest contributor, measure again.
- Usual fixes: N+1 (`includes`), missing indexes, pagination, `find_each` and `pluck` for memory, background jobs for slow work, caching, `load_async` for independent queries (needs an async executor configured).
- **CSRF:** per-session token in forms and `csrf_meta_tags`, verified on non-GET requests (`protect_from_forgery`, on by default in `ActionController::Base`). Token-header APIs skip it; cookie-authenticated endpoints never should.
- **XSS:** ERB escapes by default; `raw`, `html_safe` and `<%==` only mark strings safe; `sanitize` cleans with an allowlist; validate URL schemes before `link_to`.
- **SQL injection:** hash conditions and placeholders, allowlists for ORDER BY, never `Arel.sql(params[...])`.
- **Mass assignment:** strong parameters (`require.permit`, Rails 8 `params.expect`); never permit roles or tenant IDs. **IDOR:** scope lookups through the user or a policy.
- Headers: `force_ssl` (HTTPS redirect, HSTS, Secure cookies), default `X-Frame-Options` and `X-Content-Type-Options`, a Content Security Policy (start in report-only mode).
- Secrets in encrypted credentials or ENV; `filter_parameters` hides values from logs and `inspect`.
- Cookies: `Secure`, `HttpOnly`, `SameSite=Lax` (Rails default since 6.1). `reset_session` on sign-in.
- CI: Brakeman (static analysis, in the Rails 7.2+ default CI) and bundler-audit (vulnerable gems).

## 1. The caching layers

| Layer | What is stored | Rails API |
|---|---|---|
| HTTP (browser, CDN, proxy) | Whole responses | `fresh_when`, `stale?`, `expires_in`, `http_cache_forever` |
| Page and action caching | Whole pages or actions | Extracted to gems in Rails 4; rare today |
| Fragment | Rendered pieces of views | `cache` helper, `cached: true` collections |
| Low-level | Any value: API responses, computed stats | `Rails.cache.fetch`, `read`, `write`, `delete`, `fetch_multi` |
| SQL query cache | Identical SELECTs within one request | Automatic in controllers; cleared by any write |

General concepts (cache-aside, TTLs, CDNs) are in [caching and performance](../04-web-fundamentals/08-caching-and-performance.md). In development, `bin/rails dev:cache` toggles caching on and off.

## 2. Fragment caching and cache keys

```erb
<% cache listing do %>
  <%= render "listings/card", listing: listing %>
<% end %>
```

The key looks like `views/listings/index:<template digest>/listings/42`, and the entry stores the version `20250301101530123456` (from `updated_at`). Since Rails 5.2 keys are **recyclable**: the key stays the same and a new version overwrites the old entry instead of filling the cache with dead keys.

- The template digest covers the template and the partials it renders, so editing markup expires its fragments automatically after deploy.
- Add every input to the key: `cache [listing, I18n.locale, current_user.admin?]`.
- Never cache user-specific content (names, "your price", CSRF tokens in forms) under a shared key.

## 3. Russian doll and collection caching

```erb
<%# app/views/houses/show.html.erb %>
<% cache @house do %>
  <h1><%= @house.name %></h1>
  <%= render partial: "rooms/room", collection: @house.rooms, cached: true %>   <%# one read_multi for all rooms %>
<% end %>

<%# app/views/rooms/_room.html.erb %>
<h2><%= room.name %></h2>
<% room.residents.each do |resident| %>
  <% cache resident do %>
    <%= render "residents/badge", resident: resident %>
  <% end %>
<% end %>
```

```ruby
class Resident < ApplicationRecord
  belongs_to :room, touch: true       # resident change -> room.updated_at bumps
end

class Room < ApplicationRecord
  belongs_to :house, touch: true      # room change -> house.updated_at bumps
end
```

When one resident changes, the touch chain changes the room's and house's versions, so the outer fragment re-renders; inside it, every other room and resident fragment is still valid and comes straight from the cache. That is the point of the "dolls": a small change re-renders a small part.

Costs and traps:
- Every child write now issues extra UPDATEs up the chain; wrap imports in `ActiveRecord::Base.no_touching { ... }` and refresh caches afterwards.
- `update_all`, `delete_all` and `insert_all` do not touch parents, so fragments stay stale after bulk writes.
- A fragment that reads an association missing from its key (`cache product` while showing `product.store.name`) shows stale data when only the store changes; add `store` to the key or touch.

## 4. Low-level caching with Rails.cache

```ruby
def market_price_cents(product)
  Rails.cache.fetch(["market_price", product.external_id], expires_in: 12.hours, race_condition_ttl: 30.seconds) do
    PriceClient.new.fetch(product.external_id).loose_price_cents     # runs only on a miss
  end
end

Rails.cache.fetch([current_organization, "dashboard_stats", Date.current], expires_in: 10.minutes) do
  DashboardStats.new(current_organization).to_h                     # plain hash, not ActiveRecord objects
end

Rails.cache.delete(["market_price", product.external_id])
Rails.cache.fetch_multi(*keys) { |key| compute(key) }               # batch reads, computes only the misses
Rails.cache.increment("price_api_calls/#{Date.current}")            # counters
```

- Cache plain data (IDs, hashes, strings). Cached ActiveRecord objects go stale, are large when marshalled, and can break after a deploy changes the class.
- `fetch` caches `nil` too; pass `skip_nil: true` when "nothing found" should be retried.
- Scope keys by tenant, user, locale and version: a key like `"dashboard_stats"` shared by every organization leaks data between customers.
- Expire everything you cannot invalidate precisely (third-party prices, exchange rates) with `expires_in`.

## 5. Cache stores

| Store | Lives in | Notes |
|---|---|---|
| `:memory_store` | Process RAM | Not shared across Puma workers or servers; the development default when caching is on |
| `:file_store` | Local disk | Single server only |
| `:mem_cache_store` | Memcached (dalli gem) | Shared, fast, LRU eviction, limited by RAM |
| `:redis_cache_store` | Redis | Shared and fast; set `maxmemory-policy allkeys-lru`; keep it apart from Sidekiq's Redis, which must never evict |
| `:solid_cache_store` | Your SQL database (Rails 8 default) | Disk-backed, so caches can be far larger and live longer at low cost; reads are a bit slower than RAM stores; trims oldest entries by size or age |
| `:null_store` | Nowhere | The test default: caching code runs, nothing is kept |

```ruby
# config/environments/production.rb (pick one)
config.cache_store = :solid_cache_store
config.cache_store = :redis_cache_store, { url: ENV.fetch("REDIS_CACHE_URL"), expires_in: 1.day }
config.cache_store = :mem_cache_store, ENV.fetch("MEMCACHED_SERVERS")
```

## 6. Invalidation strategies and stampedes

| Strategy | How | Trade-off |
|---|---|---|
| Key-based expiry | Put a version (`updated_at`, a counter) in the key; old entries age out | No delete calls to forget; needs eviction or TTLs |
| Explicit delete | `after_commit { Rails.cache.delete(key) }` | Precise, but every write path must remember it (bulk SQL, console) |
| Time-based (TTL) | `expires_in: 10.minutes` | Simple; data can be stale for up to the TTL |
| Namespace bump | Add `"v2"` to keys when the cached shape changes | Instantly abandons old entries after a deploy |

A **stampede** (dogpile) happens when a hot key expires and hundreds of requests recompute it at once, hammering the database or a rate-limited API. `race_condition_ttl` lets the first request recompute while others briefly serve the old value; for very hot keys, refresh them from a scheduled job before they expire.

## 7. HTTP caching: ETags and Cache-Control

```ruby
class ListingsController < ApplicationController
  etag { current_user&.id }                  # personalized pages: vary the ETag per user

  def show
    @listing = Listing.find(params[:id])
    fresh_when @listing                      # ETag + Last-Modified; 304 if the client's copy is current
  end

  def index
    @listings = current_store.listings.order(:id)
    render json: ListingBlueprint.render(@listings) if stale?(@listings)   # relation: count + max(updated_at)
  end
end

class StoreLogosController < ApplicationController
  def show
    expires_in 1.hour, public: true          # Cache-Control: max-age=3600, public
    send_file logo_path, type: "image/png"
  end
end
```

The first response carries `ETag: W/"..."`. The browser (or a mobile HTTP client that supports it) sends `If-None-Match` next time; Rails recomputes the ETag cheaply from record keys and, if it matches, returns `304 Not Modified` with no body, skipping rendering and the download. It does not skip the queries needed to compute the ETag. Rails' default header is `Cache-Control: max-age=0, private, must-revalidate`; mark responses `public` only when they contain nothing user-specific, because CDNs and proxies will share them.

## 8. Finding slow endpoints

1. **Logs:** `Completed 200 OK in 840ms (Views: 610.2ms | ActiveRecord: 190.4ms | Allocations: 250114)` shows whether time goes to rendering, the database or elsewhere (often external HTTP).
2. **rack-mini-profiler:** a speed badge on every page in development showing SQL count and duration per request, with flamegraphs (stackprof) and memory profiles (memory_profiler).
3. **APM** (New Relic, Datadog, Scout, Skylight, AppSignal): p95 per endpoint, traces showing slow spans, N+1 detection in production.
4. **Database:** pg_stat_statements for the queries with the most total time, `EXPLAIN ANALYZE` for one query, and query log tags (`config.active_record.query_log_tags_enabled = true`, Rails 7.0+) that append controller, action or job names to SQL as comments.

Prioritize by p95 multiplied by traffic, change one thing, and measure again. Observability tooling is covered in [observability and debugging](../14-devops-and-tools/05-observability-and-debugging.md).

## 9. Common fixes

- **N+1 queries:** `includes`, `preload`, counter caches, `strict_loading` (see [querying and N+1](05-activerecord-querying-n-plus-1.md)).
- **Indexes:** on foreign keys and on the columns of frequent filters and sorts, in the right composite order (see [indexing](../12-databases/05-indexing-and-query-optimization.md)).
- **Less data:** paginate, `select` only needed columns, `pluck` instead of instantiating models, `exists?` instead of `present?`.
- **Memory bloat:** `Model.all.each` loads every row; use `find_each` or `in_batches`. Build large CSVs and reports in a job that writes a file, not in the request. Watch Puma worker memory over time; jemalloc or `MALLOC_ARENA_MAX=2` reduce fragmentation; memory_profiler and derailed_benchmarks find allocation hotspots.
- **Offload slow work** (emails, PDFs, imports, third-party calls) to background jobs (see [background jobs](09-background-jobs-sidekiq.md)).
- **Parallel queries with `load_async`** (Rails 7.0) when an action runs several independent slow queries:

```ruby
def dashboard
  @recent_orders = current_store.orders.order(created_at: :desc).limit(10).load_async  # starts in a background thread
  @low_stock     = current_store.listings.where("stock < 5").load_async
  @revenue       = current_store.orders.paid.async_sum(:total_cents)                   # Rails 7.1+: a promise, use .value
end
```

It needs `config.active_record.async_query_executor` (for example `:global_thread_pool`); without an executor the queries simply run in the foreground. Each concurrent query holds its own connection, so size the pool accordingly. YJIT, enabled by default from Rails 7.2 on supported Rubies, is a cheap CPU win.

## 10. CSRF protection

Cross-site request forgery makes a logged-in user's browser send a state-changing request from another site; the browser attaches your session cookie automatically. Rails' defense (details in [forms and CSRF](07-views-erb-forms-assets.md)):

- A per-session authenticity token in every `form_with` and in `csrf_meta_tags` for JavaScript (sent as `X-CSRF-Token`).
- `protect_from_forgery`, enabled by default in `ActionController::Base`, verifies it on every non-GET request and raises `ActionController::InvalidAuthenticityToken` when it is missing or wrong.
- GET requests are not checked, so they must never change data.
- `SameSite=Lax` cookies stop most cross-site POSTs from carrying the session at all.

APIs: `ActionController::API` has no CSRF protection because clients authenticate with a token header that another site cannot make the browser send. If a JSON endpoint authenticates with the session cookie (an SPA on the same domain), keep CSRF protection and send the token header.

## 11. XSS

```erb
<%= review.body %>                                   <%# escaped: <script> becomes &lt;script&gt; %>
<%= raw review.body %>                               <%# XSS if the body contains markup %>
<%= review.body.html_safe %>                         <%# same: html_safe marks a string, it does not clean it %>
<%= sanitize review.body, tags: %w[p br strong em a], attributes: %w[href] %>   <%# allowlist cleaning %>
<%= link_to "Website", store.website_url %>          <%# a javascript: URL runs on click: validate the scheme %>
```

- Validate user URLs on save: `URI.parse(url).scheme.in?(%w[http https])` (rescuing `URI::InvalidURIError`).
- JSON embedded in pages (React on Rails props, `data-` attributes): Rails escapes `<`, `>` and `&` inside JSON by default, so a `</script>` in the data cannot break out; keep that default on.
- A Content Security Policy is the second layer that limits damage when escaping fails.

## 12. SQL injection, mass assignment and IDOR

```ruby
Listing.where("title = '#{params[:q]}'")                 # injectable: q = "x' OR '1'='1"
Listing.where("title = ?", params[:q])                   # safe: bound parameter
Listing.where(title: params[:q])                         # safe: hash condition
Listing.order(Arel.sql(params[:sort]))                   # injectable: Arel.sql switches off Rails' check
Listing.order(SORTS.fetch(params[:sort], { id: :desc })) # safe: allowlist
```

- **Mass assignment:** strong parameters (`params.require(:listing).permit(:title, :price_cents)`, or `params.expect(listing: [:title, :price_cents])` in Rails 8) decide which attributes users may set. Never permit `:role`, `:admin`, `:organization_id` or prices the server should compute, and never call `permit!` on user input. See [routing and controllers](02-routing-and-controllers.md).
- **IDOR:** `Order.find(params[:id])` exposes every order; load through `current_user.orders` or a policy scope (see [authorization](11-authentication-authorization.md)).

## 13. HTTPS, security headers and Content Security Policy

- `config.force_ssl = true` redirects HTTP to HTTPS, sends `Strict-Transport-Security`, and marks cookies `Secure`. Behind a proxy that terminates TLS without forwarding headers, `config.assume_ssl = true` makes Rails treat requests as HTTPS.
- Default headers include `X-Frame-Options: SAMEORIGIN` (clickjacking), `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`; change them with `config.action_dispatch.default_headers`.
- `config.hosts` (host authorization) blocks requests for unexpected hostnames (DNS rebinding).

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self
    policy.style_src   :self
    policy.img_src     :self, :https, :data
    policy.connect_src :self, "wss://app.example.com"     # Action Cable
    policy.object_src  :none
    policy.frame_ancestors :none
  end
  config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
  config.content_security_policy_nonce_directives = %w[script-src]
  config.content_security_policy_report_only = true      # observe violations first, then enforce
end
```

CSP tells the browser which sources may load scripts, styles, images and connections, so injected inline scripts do not run. Inline scripts you own get a nonce (`javascript_tag nonce: true`, plus `csp_meta_tag` in the layout for Turbo). Roll out in report-only mode, fix violations (third-party widgets, inline handlers), then enforce.

## 14. Secrets, credentials and filter_parameters

```bash
bin/rails credentials:edit --environment production   # config/credentials/production.yml.enc + production.key
```

```ruby
Rails.application.credentials.dig(:price_api, :token)
ENV.fetch("PRICE_API_TOKEN")                           # ENV style: fails at boot when missing

# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [:passw, :secret, :token, :_key, :crypt, :salt, :otp, :ssn, :iban]
```

- Keys never go into git (`master.key`, per-environment keys); production reads `RAILS_MASTER_KEY` or real ENV values.
- `filter_parameters` matches partially (`:passw` covers `password_confirmation`), replaces values with `[FILTERED]` in logs, and also hides those attributes in Active Record's `inspect` (Rails 6+). Apply the same scrubbing in your error tracker and APM, including the `Authorization` header.
- A leaked secret must be rotated, not just deleted from the repo; changing `secret_key_base` invalidates sessions and signed cookies.

## 15. Cookies and sessions

| Flag | Protects against | In Rails |
|---|---|---|
| `Secure` | Cookie sent over plain HTTP | Set by `force_ssl`; `secure: true` per cookie |
| `HttpOnly` | JavaScript reading it (token theft via XSS) | The session cookie is HttpOnly by default; `httponly: true` per cookie |
| `SameSite` | Cookie attached to cross-site requests (CSRF) | `Lax` by default since Rails 6.1; `same_site: :strict` per cookie |

The default session store is an encrypted, signed cookie. Call `reset_session` on sign-in (fixation) and sign-out, set an idle limit with `config.session_store :cookie_store, expire_after: 2.weeks`, and remember that a cookie session cannot be revoked server-side unless you track sessions in the database, as the Rails 8 authentication generator does.

## 16. Tooling and other common holes

- **Brakeman:** static analysis for SQL injection, XSS, unsafe redirects, mass assignment, command injection and more; Rails 7.2+ generates `bin/brakeman` and runs it in the default CI. Record accepted false positives in `config/brakeman.ignore`.
- **bundler-audit:** `bundle exec bundle-audit check --update` compares `Gemfile.lock` with the Ruby advisory database; pair it with Dependabot or Renovate, and `bin/importmap audit` or `npm audit` for JavaScript.
- **Open redirects:** `redirect_to params[:return_to]`; Rails 7.0+ raises `UnsafeRedirectError` for other hosts unless you pass `allow_other_host: true`, so only allow relative paths.
- **Command injection:** `system("convert #{params[:file]}")`; use the array form `system("convert", path, out_path)`.
- **Uploads:** check content type and size server-side (see [Active Storage](15-actioncable-mailer-activestorage.md)); **brute force:** rate limiting (see [API development](10-api-development.md)).

## Common pitfalls
- Cache keys that ignore the tenant, user or locale, serving one customer's data to another.
- Caching ActiveRecord objects, or fragments containing CSRF tokens and user-specific text.
- Russian doll caches without `touch: true`, or bulk writes that skip touches.
- A Redis instance shared by cache (evicting) and Sidekiq (must not evict).
- Optimizing without measuring, or measuring only in development with tiny data.
- `load_async` without an executor configured, or without enough pool connections.
- `html_safe` / `raw` on user content; `link_to` with unvalidated URLs.
- Skipping CSRF on cookie-authenticated endpoints because "it is JSON".
- `Arel.sql(params[:sort])`, interpolated `where` strings, `permit!`.
- Secrets in git, in logs, or in VCR cassettes.

## How to talk about this in an interview
- For CrystalCommerce, caching third-party market prices with expiry and scoping keys per store, plus N+1 and index fixes at catalog scale, are natural examples; describe the method and claim only what you actually did.
- For Colochousing, mention ETags or caching on API endpoints the resident app polls, and security basics of a Rails app serving both web and mobile clients: CSRF for the web, tokens for the app, scoped queries everywhere.
- Frame security as layers: escaping plus CSP, strong params plus policies, CSRF tokens plus SameSite, Brakeman plus bundler-audit in CI.
- If you have not configured CSP or a CDN yourself, say so and explain how you would roll it out safely.

> **Fill in:** one performance problem you diagnosed at Devntech: the symptom, the tool that showed the cause (logs, mini-profiler, APM) and the fix.

> **Fill in:** which cache store and caching layers the Colochousing and CrystalCommerce apps use, if any.

> **Fill in:** any security issue you found or fixed (unscoped query, unsafe `html_safe`, missing authorization) and how you verified the fix.

## Interview questions

**Q1. What caching options does Rails give you?**
HTTP caching (ETags and Last-Modified via `fresh_when`/`stale?`, `Cache-Control` via `expires_in`), fragment caching with the `cache` view helper and cached collections, low-level caching with `Rails.cache.fetch`, and the automatic per-request SQL query cache. Page and action caching exist only as gems now.
*Follow-up:* Which do you reach for first? Fix queries first, then fragment caching for expensive views and low-level caching for expensive computations or API calls.

**Q2. How does a fragment cache know when it is stale?**
The key includes the template digest and the record's `cache_key`, and the stored entry carries the record's `cache_version` (`updated_at`). When the record changes, the version no longer matches and the fragment is rebuilt; when the template changes, the digest changes. No manual expiry is needed.
*Follow-up:* What are recyclable cache keys? Since Rails 5.2 the version lives in the entry instead of the key, so updates overwrite one key rather than creating new ones.

**Q3. What is Russian doll caching?**
Nested fragment caches (house, then rooms, then residents) with `touch: true` on the child associations, so updating a child bumps every parent's `updated_at`. The outer fragment re-renders, but unchanged inner fragments are reused, so only the changed part is rendered again.
*Follow-up:* What breaks it? Bulk writes (`update_all`) that skip touches, and fragments that read data missing from their key.

**Q4. How would you cache an expensive third-party API call?**
`Rails.cache.fetch` with a key containing the external ID (and tenant if relevant), an `expires_in` that matches how fresh the data must be, and `race_condition_ttl` to avoid stampedes; cache the parsed plain data, handle errors without caching them, and warm hot keys from a job if needed.
*Follow-up:* Why not cache the raw response object? It may not serialize cleanly and ties the cache to a client library's classes.

**Q5. Redis, Memcached or Solid Cache?**
Memcached and Redis are shared RAM stores with very fast reads and LRU eviction; Redis also brings persistence options and other data structures. Solid Cache (Rails 8 default) stores entries in a database on disk, allowing much larger, longer-lived caches without another service, at slightly slower reads. Pick by existing infrastructure, cache size and latency needs.
*Follow-up:* Can the cache share Sidekiq's Redis? Not if it evicts keys; Sidekiq needs a non-evicting instance.

**Q6. How do you invalidate caches?**
Prefer key-based expiry (versions in keys, old entries age out); use explicit deletes in `after_commit` only when keys cannot carry versions; add TTLs to anything you cannot track; bump a namespace when the cached shape changes.
*Follow-up:* What is a cache stampede? Many requests recomputing the same expired hot key at once; mitigate with `race_condition_ttl` or background refresh.

**Q7. What is wrong with `Rails.cache.fetch("dashboard_stats") { DashboardStats.new(current_organization).to_h }`?**
The key ignores the organization, so the first tenant's stats are cached and served to every other tenant: a data leak. It also has no expiry. Use `[current_organization, "dashboard_stats"]` with `expires_in`.
*Follow-up:* What else belongs in keys? User (for personalized data), locale and a version for the cached shape.

**Q8. How does HTTP caching with ETags work in Rails?**
`fresh_when(@record)` or `stale?(@record)` computes an ETag (and Last-Modified) from the record; if the request's `If-None-Match` matches, Rails returns 304 with no body, otherwise it renders and sends the new ETag. It saves rendering and bandwidth, not the queries needed to load the record.
*Follow-up:* When is `public: true` dangerous? On personalized responses, because shared caches would serve them to other users.

**Q9. An endpoint is slow in production. How do you investigate?**
Check APM traces or logs to see whether time goes to SQL, views or external calls; reproduce with production-like data and rack-mini-profiler; look for N+1 patterns, missing indexes (`EXPLAIN ANALYZE`, pg_stat_statements), large payloads and slow HTTP calls; fix the biggest cost and measure again.
*Follow-up:* What do query log tags add? Controller and action names in SQL comments, so database tools point back to the code.

**Q10. What causes memory bloat in Rails, and how do you fix it?**
Loading huge result sets (`all.each`, `to_a`), instantiating models where `pluck` would do, building big strings or arrays for exports, and allocation-heavy code; fix with `find_each`, `pluck`, streaming or background exports, and profiling with memory_profiler or derailed_benchmarks. jemalloc or `MALLOC_ARENA_MAX` reduce fragmentation.
*Follow-up:* How do you spot it? Puma worker memory growing over time in your metrics.

**Q11. What does `load_async` do?**
It starts a relation's query on a background thread pool so the controller can continue (Rails 7.0); when the view uses the records it waits only if the query has not finished. It helps actions with several independent slow queries, needs an async executor configured, and uses an extra connection per concurrent query.
*Follow-up:* What did 7.1 add? Async calculations such as `async_count`, `async_sum` and `async_pluck`, which return promises.

**Q12. How does Rails protect against CSRF, and when can you skip it?**
`ActionController::Base` verifies a per-session authenticity token (from forms or the `X-CSRF-Token` header) on every non-GET request and raises when it is missing or wrong, and SameSite cookies add defense. Skip it only for endpoints authenticated by tokens in headers, which is why `ActionController::API` leaves it out; never for cookie-authenticated endpoints.
*Follow-up:* Why must GET be safe? It is not CSRF-checked and can be triggered by an image tag.

**Q13. How does Rails prevent XSS, and how can you still introduce it?**
ERB escapes all output by default. You reintroduce XSS with `raw`, `html_safe`, `<%==` on user content, helpers that concatenate HTML strings, `javascript:` URLs in `link_to`, or unescaped JSON in script tags. Use `sanitize` with an allowlist when HTML is needed, validate URL schemes, and add a CSP.
*Follow-up:* What does `html_safe` actually do? Marks the string as safe so it skips escaping; it never removes anything.

**Q14. How do you prevent SQL injection with ActiveRecord?**
Hash conditions or bound placeholders for values (`where("title = ?", q)`), `sanitize_sql_like` for LIKE patterns, and allowlists for column names and sort directions, which cannot be bound. Never interpolate params into SQL strings or wrap them in `Arel.sql`.
*Follow-up:* Is `find_by_sql` safe? With the array form and placeholders, yes.

**Q15. What is mass assignment, and how do strong parameters help?**
Assigning many attributes from user input at once (`User.new(params[:user])`); without filtering, attackers add fields like `admin=true` or `organization_id`. Strong parameters raise on unfiltered params and let only permitted keys through; Rails 8's `params.expect` also checks the shape and returns 400 for tampered structures.
*Follow-up:* Is permitting `:organization_id` fine if the form has a select? Only if the server verifies membership; otherwise derive it server-side.

**Q16. What is a Content Security Policy, and how do you roll one out in Rails?**
A response header telling the browser which sources may provide scripts, styles, images, frames and connections, blocking injected inline scripts. Configure it in `content_security_policy.rb`, use nonces for your inline scripts, start in report-only mode, fix violations, then enforce.
*Follow-up:* What usually breaks? Inline event handlers, inline scripts without nonces and third-party widgets.

**Q17. How do you keep secrets safe in a Rails app?**
Encrypted credentials (per environment) or ENV variables, keys outside git, `filter_parameters` for logs and `inspect`, scrubbing in error trackers, Brakeman and bundler-audit in CI, cookies with Secure, HttpOnly and SameSite, and rotation after any leak.
*Follow-up:* What happens if `secret_key_base` changes? Existing sessions, signed and encrypted cookies and signed IDs stop verifying.

## Further reading
- [Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html)
- [Securing Rails Applications](https://guides.rubyonrails.org/security.html)
- [Active Record Query Interface: async queries](https://guides.rubyonrails.org/active_record_querying.html)
