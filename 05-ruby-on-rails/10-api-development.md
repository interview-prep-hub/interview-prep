# API Development in Rails

> **Tier:** T1 (know cold) · **Related:** [Routing and controllers](02-routing-and-controllers.md), [Authentication and authorization](11-authentication-authorization.md), [Testing with RSpec](13-testing-rspec.md), [Caching, performance and security](14-caching-performance-security.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md)
> **On your resume:** REST APIs in legacy Rails monoliths at Devntech, the Colochousing Rails 7 REST API consumed by the Coloc resident app (React Native, Expo), and the Rails sync APIs built for the multi-tenant platform migration.

## Quick revision
- `rails new app --api` sets `config.api_only = true`: controllers inherit `ActionController::API`, and there are no views, assets, cookies, session or flash by default. Inside a full app, JSON endpoints live under an `Api::V1::BaseController < ActionController::API`.
- `ActionController::API` leaves out CSRF protection, cookies, flash, layouts and `respond_to`; include modules back only when needed (for example `ActionController::HttpAuthentication::Token::ControllerMethods` for bearer tokens).
- Status codes: 200 read/update, 201 create (+ `Location`), 202 queued, 204 no body, 400 malformed, 401 not authenticated, 403 not allowed, 404 missing or hidden, 409 conflict, 422 validation, 429 rate limited.
- One error envelope for every failure (`{ "error": { "code": "validation_failed", "message": "...", "details": {...} } }`), produced by `rescue_from` handlers in the base controller.
- Serializers: `as_json` (tiny APIs), Jbuilder (templates), ActiveModel::Serializers (legacy favorite), Blueprinter (simple and fast), jsonapi-serializer (JSON:API format). Any of them can hide N+1 queries: preload in the controller.
- Offset pagination is simple but slow on deep pages and shifts when rows change; cursor (keyset) pagination (`WHERE (created_at, id) < (?, ?)`) is stable and index-friendly. Always cap the page size.
- Filter with hash conditions and placeholders; sort through an allowlist; escape LIKE patterns with `sanitize_sql_like`.
- Version with `/api/v1` namespaces. Additive changes are safe; removals, renames and type changes need a new version. Old mobile builds live for months: send `X-App-Version`, enforce a minimum supported version, offer a forced-update flag.
- Mobile auth: `Authorization: Bearer <token>`, short-lived access tokens plus rotating refresh tokens, stored as digests on the server and in SecureStore on the device.
- CORS (rack-cors) only matters to browsers and is not authentication; native apps and servers ignore it.
- Rate limiting: rack-attack (middleware, rules by IP or path) or `rate_limit to: 10, within: 1.minute` in controllers (Rails 7.2+, stored in the cache store).
- Idempotency keys make POST retries safe: key + user + request fingerprint + saved response under a unique index; repeats replay the saved response.
- Store UTC, render ISO 8601 (`"2025-03-30T08:00:00.000Z"`), compute with `Time.current` and `Time.use_zone(user.time_zone)`; send plain dates as `"2025-04-01"`.
- Document with OpenAPI (rswag generates it from specs); test with request specs that assert status, body shape, auth failures and tenant isolation.
- Sync APIs for another system: cursor on `(updated_at, id)`, tombstones for deletes, idempotent upserts on the consumer, a scoped service token or HMAC signature.

## 1. API-only apps vs API controllers inside a full app

| | API-only app (`rails new shop_api --api`) | API inside a full app |
|---|---|---|
| Base class | `ApplicationController < ActionController::API` | `Api::V1::BaseController < ActionController::API`, next to HTML controllers on `ActionController::Base` |
| Middleware | No cookies, session, flash or `Rack::MethodOverride` | Full stack; API controllers just do not use the session |
| Typical clients | SPA, mobile app, other services | Server-rendered web app plus a mobile app on the same models (the Colochousing shape) |

Most APIs you meet in Rails jobs live inside a monolith: the same models serve ERB pages and JSON for a mobile app. Inheriting the API controllers from `ActionController::API` still pays off: no CSRF checks (token clients send no cookies, so there is nothing to forge), no accidental HTML layouts, less work per request. If an API controller must inherit from `ApplicationController`, call `skip_forgery_protection` only when authentication is header-based; a cookie-authenticated JSON endpoint still needs CSRF protection.

What `ActionController::API` leaves out and you may add back: `ActionController::MimeResponds` (`respond_to`), `ActionController::Cookies`, `ActionController::HttpAuthentication::Token::ControllerMethods` (`authenticate_with_http_token`). An action that renders nothing returns `204 No Content`.

## 2. RESTful JSON: status codes, error envelope and rescue_from

Resource design, verbs and method idempotency are in [REST API design](../04-web-fundamentals/04-rest-api-design.md). The Rails-specific part is mapping exceptions to statuses in one place.

| Status | Symbol | When |
|---|---|---|
| 200 | `:ok` | Read or update that returns a body |
| 201 | `:created` | Resource created; add `location: api_v1_listing_url(listing)` |
| 202 | `:accepted` | Work queued (import, export); return a status URL |
| 204 | `:no_content` | Success without a body (`head :no_content` after DELETE) |
| 400 | `:bad_request` | Malformed JSON, missing required param (`ParameterMissing`) |
| 401 | `:unauthorized` | Missing, invalid or expired token |
| 403 | `:forbidden` | Authenticated but not allowed |
| 404 | `:not_found` | Missing, or belongs to another user or tenant (do not reveal it exists) |
| 409 | `:conflict` | Duplicate request in progress, stale `lock_version`, invalid state transition |
| 422 | `:unprocessable_entity` | Validation failed; newer Rack also names it `:unprocessable_content` |
| 429 | `:too_many_requests` | Rate limited; add `Retry-After` |

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods

      before_action :authenticate!

      rescue_from ActiveRecord::RecordNotFound do
        render_error :not_found, "not_found", "Resource not found"
      end
      rescue_from ActionController::ParameterMissing do |e|
        render_error :bad_request, "bad_request", e.message
      end
      rescue_from ActiveRecord::RecordInvalid do |e|
        render_error :unprocessable_entity, "validation_failed", "Validation failed",
                     details: e.record.errors.to_hash(true)   # { email: ["Email has already been taken"] }
      end
      rescue_from Pundit::NotAuthorizedError do                  # only if the app uses Pundit
        render_error :forbidden, "forbidden", "You are not allowed to do that"
      end

      private

      attr_reader :current_user

      def authenticate!
        @current_user = authenticate_with_http_token { |token, _options| ApiToken.user_for(token) }
        render_error :unauthorized, "unauthorized", "Missing or invalid token" unless @current_user
      end

      def render_error(status, code, message, details: nil)
        error = { code: code, message: message }
        error[:details] = details if details
        render json: { error: error }, status: status
      end
    end
  end
end
```

- `code` is a stable string the mobile app can switch on; `message` is for humans and may change or be translated.
- A before_action that renders (`authenticate!`) halts the chain, so the action never runs.
- Actions stay short: `listing = current_user.listings.create!(listing_params)` then `render json: ..., status: :created`; a failure becomes a 422 through the handler.
- Handlers are matched bottom-up, so declare broad ones first. A catch-all `rescue_from StandardError` must still report to your error tracker, or bugs disappear.

## 3. Serializers: shaping the JSON

Never `render json: @user` straight from a model: `to_json` serializes every column by default, so digests, tokens and internal flags leak unless something filters them, and every future column leaks automatically. Choose fields explicitly.

```ruby
# as_json options: fine for one small endpoint
render json: listing.as_json(only: %i[id title price_cents], include: { store: { only: %i[id name] } })

# Jbuilder: app/views/api/v1/listings/index.json.jbuilder
json.data @listings, partial: "api/v1/listings/listing", as: :listing
# app/views/api/v1/listings/_listing.json.jbuilder
json.extract! listing, :id, :title, :price_cents
json.store_name listing.store.name               # one query per listing unless the controller preloaded :store

# Blueprinter: one class, several views
class ListingBlueprint < Blueprinter::Base
  identifier :id
  fields :title, :price_cents
  view :with_store do
    association :store, blueprint: StoreBlueprint
  end
end
render json: ListingBlueprint.render(@listings, view: :with_store)

# jsonapi-serializer: { data: [{ id, type, attributes, relationships }], included: [...] }
class ListingSerializer
  include JSONAPI::Serializer
  attributes :title, :price_cents
  belongs_to :store
end
render json: ListingSerializer.new(@listings, include: [:store]).serializable_hash
```

| Option | Style | Strengths | Watch out for |
|---|---|---|---|
| `as_json` | Options (or an override) on the model | No dependency | Couples the model to one API shape; easy to over-expose |
| Jbuilder | View templates | In the default Gemfile of full apps; feels like views | Slower on big payloads; queries hide in partials |
| ActiveModel::Serializers | `class ListingSerializer < ActiveModel::Serializer`, found implicitly by `render json:` | Everywhere in older apps | Maintenance has been slow for years; implicit lookup surprises |
| Blueprinter | Declarative class with views | Simple, fast, one class serves several use cases | Plain JSON only |
| jsonapi-serializer | Class per model | Fast; JSON:API compound documents | A verbose format all clients must accept |

**N+1 inside serializers.** A serializer runs per record, so `association :store` or `listing.store.name` queries once per listing. Fix it where the records are loaded:

```ruby
def index
  listings = current_user.listings.includes(:store, photos_attachments: :blob).order(id: :desc).limit(50)
  render json: ListingBlueprint.render(listings, view: :with_store)
end
```

Catch regressions with `strict_loading` or the Bullet gem; see [N+1 queries](05-activerecord-querying-n-plus-1.md).

## 4. Pagination: offset vs cursor

```ruby
# Offset pagination with kaminari
listings = current_user.listings.order(created_at: :desc, id: :desc).page(params[:page]).per(25)
render json: { data: ListingBlueprint.render_as_hash(listings),
               meta: { page: listings.current_page, total_pages: listings.total_pages } }
```

Pagy does the same with less memory and is popular in newer apps; its setup and option names changed across major versions, so follow the README for the version in your Gemfile.

| | Offset (`LIMIT 25 OFFSET 5000`) | Cursor / keyset (`WHERE (created_at, id) < (?, ?) LIMIT 25`) |
|---|---|---|
| Jump to page N | Yes | No, next and previous only |
| Deep pages | Slow: skipped rows are still read | Fast: an index seek at any depth |
| Rows added or removed while paging | Items repeat or get skipped | Stable |
| Total count | Usually shown, costs a `COUNT(*)` | Usually omitted |
| Best for | Admin tables, small lists | Feeds, infinite scroll in mobile apps, sync APIs |

```ruby
class Api::V1::ListingsController < Api::V1::BaseController
  PER_PAGE = 50

  def index
    scope = current_user.listings.order(created_at: :desc, id: :desc)
    if (cursor = decode_cursor(params[:cursor]))
      scope = scope.where("(listings.created_at, listings.id) < (?, ?)", cursor[:created_at], cursor[:id])
    end
    rows = scope.limit(PER_PAGE + 1).to_a              # one extra row tells you whether a next page exists
    page = rows.first(PER_PAGE)
    render json: { data: ListingBlueprint.render_as_hash(page),
                   meta: { next_cursor: (encode_cursor(page.last) if rows.size > PER_PAGE) } }
  end

  private

  def encode_cursor(listing)
    Base64.urlsafe_encode64({ t: listing.created_at.utc.iso8601(6), id: listing.id }.to_json)  # keep microseconds
  end

  def decode_cursor(raw)
    return if raw.blank?
    data = JSON.parse(Base64.urlsafe_decode64(raw))
    { created_at: Time.iso8601(data.fetch("t")), id: Integer(data.fetch("id")) }
  rescue ArgumentError, KeyError, TypeError, JSON::ParserError
    raise ActionController::BadRequest, "invalid cursor"
  end
end
```

- The `id` tiebreaker makes the order total, because many rows share a timestamp. Back the query with a matching index such as `[:user_id, :created_at, :id]`.
- Postgres timestamps keep microseconds; a cursor rounded to seconds skips or repeats rows.
- Cap client-chosen sizes: `params.fetch(:limit, 25).to_i.clamp(1, 100)`.

## 5. Filtering and sorting safely

```ruby
SORTS = {
  "newest"     => { created_at: :desc },
  "price_asc"  => { price_cents: :asc },
  "price_desc" => { price_cents: :desc }
}.freeze

def index
  scope = current_user.listings
  scope = scope.where(condition: params[:condition]) if params[:condition].present?     # bound as a parameter
  scope = scope.where(price_cents: params[:min_price].to_i..) if params[:min_price].present?
  scope = scope.where("listings.title ILIKE ?", "%#{Listing.sanitize_sql_like(params[:q])}%") if params[:q].present?
  scope = scope.order(SORTS.fetch(params[:sort], SORTS["newest"])).order(:id)          # allowlist + tiebreaker
  render json: ListingBlueprint.render(scope.limit(50))
end
```

- Values go through hash conditions or `?` placeholders. Column names and directions cannot be bound, so map them through an allowlist. Rails 6+ raises on `order` arguments that do not look like columns, but `order(params[:sort])` still lets a client sort by any column, including sensitive ones.
- `sanitize_sql_like` escapes `%` and `_`, so searching for `100%` matches literally.
- Ransack 4 requires explicit `ransackable_attributes` allowlists, a common surprise when upgrading legacy apps that exposed every column.
- When filters multiply, move them into a query object (see [architecture patterns](12-architecture-patterns.md)).

## 6. Versioning, old app builds and forced updates

A web frontend deploys together with the backend; a mobile app does not. Once a build is in the stores, some users keep it for months, and store review delays every fix. Every API change must keep working for every build still installed.

| Change | Safe for old builds? |
|---|---|
| Add an optional response field, an endpoint or an optional param | Yes (clients must ignore unknown fields) |
| Remove or rename a field, change a type (`"12.50"` to `1250`) or enum values | No |
| Make a param required, change error codes the app switches on | No |
| Change meaning with the same shape (price now includes VAT) | No |

```ruby
# config/routes.rb
namespace :api, defaults: { format: :json } do
  namespace :v1 do
    resources :listings, only: %i[index show create]
  end
  namespace :v2 do
    resources :listings, only: %i[index show]     # Api::V2::ListingsController < Api::V1::ListingsController
  end                                             # overriding only the serializer view
end
```

URL versioning is the usual Rails choice: visible in logs, easy to route and test. Header versioning (`Accept: application/vnd.example.v2+json` checked by a routing constraint object) keeps URLs clean but is harder to try by hand and easier to get wrong in caches.

**Minimum supported version and forced updates.** Let the app send its version and keep the policy on the server, so you can change it without a release:

```ruby
before_action :enforce_minimum_app_version

def enforce_minimum_app_version
  raw = request.headers["X-App-Version"]                   # header name is your own convention
  return if raw.blank? || !Gem::Version.correct?(raw)      # web clients and very old builds: choose a policy
  return if Gem::Version.new(raw) >= Gem::Version.new(ENV.fetch("MIN_APP_VERSION", "0"))
  render json: { error: { code: "app_update_required", message: "Please update the app." } },
         status: :upgrade_required                         # 426, or any status the app knows to handle
end
```

- A public `GET /api/v1/app_config` returning `min_supported_version` and `latest_version` lets the app show a soft "update available" banner or a blocking update screen at launch.
- `Gem::Version` orders `"2.10.0"` after `"2.9.1"`; string comparison does not.
- Over-the-air updates (EAS Update) only replace the JavaScript and assets of builds with a compatible runtime version; native changes need a store release (see [EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)).
- Before removing v1, log which app versions still call it and wait until that traffic is negligible.

## 7. Token authentication for mobile clients

Mobile apps do not use the cookie session; they send a token with every request (`Authorization: Bearer 2m9fQx...`). A minimal opaque-token model:

```ruby
class ApiToken < ApplicationRecord
  belongs_to :user

  def self.issue_for(user, ttl: 30.days)
    raw = SecureRandom.urlsafe_base64(32)
    create!(user: user, token_digest: Digest::SHA256.hexdigest(raw), expires_at: ttl.from_now)
    raw                                              # returned to the client once, never stored
  end

  def self.user_for(raw)
    where("expires_at > ?", Time.current).find_by(token_digest: Digest::SHA256.hexdigest(raw))&.user
  end
end
```

- Only digests are stored, so a leaked database does not leak usable tokens; deleting rows revokes devices ("sign out everywhere").
- `authenticate_with_http_token` accepts both the `Bearer` and `Token` schemes.
- The app keeps tokens in the OS keychain (Expo SecureStore), never in AsyncStorage.

Access vs refresh tokens, JWT with devise-jwt, Doorkeeper and revocation are covered in [authentication and authorization](11-authentication-authorization.md).

## 8. CORS with rack-cors

Browsers stop JavaScript on `https://app.example.com` from reading responses from `https://api.example.com` unless the API opts in with CORS headers. Requests with JSON bodies, an `Authorization` header or verbs like PATCH trigger an `OPTIONS` preflight first.

```ruby
# Gemfile: gem "rack-cors"; config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com", %r{\Ahttps://[a-z0-9-]+\.example\.com\z}
    resource "/api/*",
      headers: :any,
      methods: %i[get post put patch delete options head],
      expose: %w[Link Retry-After],       # response headers JavaScript may read
      max_age: 600                         # browsers cache the preflight for 10 minutes
  end
end
```

- It is inserted first so preflights are answered before authentication rejects them.
- Only browsers enforce CORS. The React Native app, curl and other servers ignore it, so it never replaces authentication.
- A wildcard origin is acceptable for a public token-authenticated API, but never with `credentials: true` (cookies).
- A "CORS error" in the console often hides a real failure: a proxy error or crash returned without CORS headers. Check the network tab and server logs for the real status.

## 9. Rate limiting: rack-attack and `rate_limit`

```ruby
# config/initializers/rack_attack.rb (rack-attack gem; uses Rails.cache unless Rack::Attack.cache.store is set)
class Rack::Attack
  throttle("api/ip", limit: 300, period: 5.minutes) do |req|
    req.ip if req.path.start_with?("/api/")
  end

  throttle("logins/ip", limit: 10, period: 1.minute) do |req|
    req.ip if req.path == "/api/v1/sessions" && req.post?
  end

  safelist("localhost") { |req| ["127.0.0.1", "::1"].include?(req.ip) }
end
```

```ruby
# Rails 7.2+: built into controllers
class Api::V1::SessionsController < Api::V1::BaseController
  skip_before_action :authenticate!
  rate_limit to: 10, within: 3.minutes, only: :create,
             by: -> { params[:email].to_s.downcase },               # default: request.remote_ip
             with: -> { render json: { error: { code: "rate_limited" } }, status: :too_many_requests }
end
```

| | rack-attack | Controller `rate_limit` |
|---|---|---|
| Runs | Rack middleware, before routing | Before action inside the controller |
| Sees | Path, IP, headers, form params (not parsed JSON bodies) | Parsed params, `current_user`, anything the controller has |
| Extras | Safelists, blocklists, Fail2Ban-style bans | Minimal by design |
| Availability | Any Rails version | Rails 7.2+ |

- Both need a cache store shared by all processes (Redis, Memcached, Solid Cache). With `:memory_store` each Puma worker counts separately; with `:null_store` nothing is limited.
- Behind a load balancer, make sure `request.remote_ip` is the client's address (trusted proxies configured), or every user shares one bucket.
- Return 429 with `Retry-After`, and make the mobile app back off instead of retrying in a loop.

## 10. Idempotency keys for POST requests

GET, PUT and DELETE are idempotent by definition; POST is not. On mobile networks a request can succeed while the response is lost, and the app retries: without protection a resident pays rent twice. The fix popularized by payment APIs (and drafted as the `Idempotency-Key` HTTP header) is a client-generated key per logical operation, reused on every retry.

```ruby
# idempotency_keys: user_id, key, request_fingerprint, response_status, response_body (jsonb), timestamps
# add_index :idempotency_keys, [:user_id, :key], unique: true
def create
  key = request.headers["Idempotency-Key"]
  return render_error(:bad_request, "idempotency_key_missing", "Send an Idempotency-Key header") if key.blank?

  fingerprint = Digest::SHA256.hexdigest(request.raw_post)
  idem = current_user.idempotency_keys.create_or_find_by!(key: key) { |r| r.request_fingerprint = fingerprint }

  if idem.request_fingerprint != fingerprint
    render_error(:unprocessable_entity, "idempotency_key_reused", "Key already used with a different body")
  elsif idem.response_status                                   # finished earlier: replay the saved response
    render json: idem.response_body, status: idem.response_status
  elsif !idem.previously_new_record?                           # another request with this key is running
    render_error(:conflict, "request_in_progress", "Retry in a moment")
  else
    result = RentPayments::Pay.call(user: current_user, params: payment_params)
    idem.update!(response_status: result.status, response_body: result.body)
    render json: result.body, status: result.status
  end
end
```

- The unique index makes it safe under concurrency: only one request can insert the key, and `create_or_find_by!` turns the loser into a lookup.
- The fingerprint rejects a reused key with a different body instead of replaying the wrong response.
- Expire keys after a day with a cleanup job, and let a key stuck "in progress" after a crash be retried after a timeout.
- Pass an idempotency key on to the payment provider as well. Database-level double-payment protection is in [ActiveRecord advanced](06-activerecord-advanced.md).

## 11. Timestamps and time zones

1. **Store UTC.** Rails writes `datetime` columns in UTC whatever `config.time_zone` says.
2. **Use the app zone, not the server's.** `Time.current` and `Date.current` respect `config.time_zone`; `Time.now` and `Date.today` use the OS zone.
3. **Render ISO 8601 with an offset.** `as_json` produces `"2025-03-30T08:00:00.000Z"`, which every client parses.
4. **Convert to the user's zone at the edges:** views, emails and push notification text, using a zone stored on the user or the house.
5. **Dates are not times.** A rent due date is a `date` column sent as `"2025-04-01"`; turned into midnight UTC it shows as March 31 in zones west of UTC.

```ruby
config.time_zone = "UTC"                          # config/application.rb

Time.use_zone("Europe/Brussels") do               # typically an around_action with the user's zone
  starts_at = Time.zone.parse("2025-03-30 10:00") # 10:00 in Brussels, on the day summer time starts
  starts_at.utc.iso8601                           # => "2025-03-30T08:00:00Z"
end

event.starts_at.in_time_zone(current_user.time_zone)   # for display
Time.zone.parse(params[:starts_at])               # honors an offset in the string, else assumes Time.zone
```

Ask clients to send ISO 8601 with `Z` or an offset, never wall-clock times without a zone. For "remind the resident at 9:00", store the zone and schedule in it so daylight-saving changes do not shift the reminder.

## 12. Documentation and request specs

OpenAPI is the standard contract format. rswag writes it from specs, so the docs are tested and cannot drift silently:

```ruby
# spec/requests/api/v1/listings_spec.rb (rswag DSL)
require "swagger_helper"

RSpec.describe "Listings API", type: :request do
  path "/api/v1/listings/{id}" do
    get "Show a listing" do
      produces "application/json"
      parameter name: :id, in: :path, type: :integer

      response "404", "listing not found" do
        let(:id) { 0 }
        run_test!                                     # sends the request and checks the status
      end
    end
  end
end
```

`bundle exec rake rswag:specs:swaggerize` writes the YAML file and rswag-ui serves it as interactive docs. Plain request specs cover behavior:

```ruby
RSpec.describe "POST /api/v1/listings", type: :request do
  let(:user) { create(:user) }
  let(:headers) { { "Authorization" => "Bearer #{ApiToken.issue_for(user)}" } }

  it "creates a listing" do
    post "/api/v1/listings", params: { listing: { title: "Zelda", price_cents: 4999 } }, headers: headers, as: :json
    expect(response).to have_http_status(:created)
    expect(response.parsed_body.dig("data", "title")).to eq("Zelda")
  end

  it "returns the error envelope when validation fails" do
    post "/api/v1/listings", params: { listing: { title: "" } }, headers: headers, as: :json
    expect(response).to have_http_status(:unprocessable_entity)
    expect(response.parsed_body.dig("error", "code")).to eq("validation_failed")
  end
end
```

`as: :json` sends a JSON body with the right content type; add specs for a missing token (401) and for another user's record (404). More in [testing with RSpec](13-testing-rspec.md).

## 13. Sync APIs consumed by another system

During a platform migration, the new system often pulls data from the Rails app until the switch-over. The client is a machine that wants everything changed since its last run and must never miss a change.

```ruby
# GET /api/sync/v1/residents?since=<cursor>&limit=500
class Api::Sync::V1::ResidentsController < Api::Sync::BaseController   # authenticates a ServiceClient token
  def index
    limit = params.fetch(:limit, 500).to_i.clamp(1, 1000)
    scope = Resident.where(organization: current_client.organization)  # soft-deleted rows included on purpose
                    .where("updated_at < ?", 30.seconds.ago)            # settle window, see below
                    .order(:updated_at, :id)
    if (cursor = decode_cursor(params[:since]))                         # same encoding idea as section 4
      scope = scope.where("(updated_at, id) > (?, ?)", cursor[:updated_at], cursor[:id])
    end
    rows = scope.limit(limit + 1).to_a
    page = rows.first(limit)
    render json: { data: ResidentBlueprint.render_as_hash(page, view: :sync),   # id, fields, updated_at, discarded_at
                   next_cursor: page.last ? encode_cursor(page.last) : params[:since],
                   has_more: rows.size > limit }
  end
end
```

- **Cursor on `(updated_at, id)`.** `>` on the timestamp alone skips rows sharing the last timestamp; `>=` re-sends them forever. Index `[:organization_id, :updated_at, :id]`.
- **Every write must bump `updated_at`.** `update_all`, `update_column(s)` and raw SQL leave it alone, so those changes never reach the consumer unless you set `updated_at: Time.current` yourself.
- **Deletes need tombstones.** A hard-deleted row just vanishes from the feed; soft deletes (`discarded_at`) or a `deleted_records` table tell the consumer to delete its copy.
- **The consumer upserts by a stable ID** and saves the cursor only after the page is stored, so re-reading a page is harmless.
- **Service-to-service auth:** a `ServiceClient` record per consumer with a hashed, rotatable token scoped to one organization and to read-only sync endpoints, or HMAC-signed requests (`OpenSSL::HMAC.hexdigest("SHA256", secret, "#{timestamp}.#{request.raw_post}")` compared with `ActiveSupport::SecurityUtils.secure_compare`, rejecting old timestamps to block replays), plus IP allowlisting or private networking.
- **Change-feed alternative:** append a row to a `change_events` table in the same transaction as each write (the outbox pattern) and let the consumer read by event ID; it captures deletes and every intermediate state at the cost of extra writes.

### Going deeper: why the settle window

`updated_at` is set when a transaction writes the row, not when it commits. If transaction A writes at 10:00:00.1 but commits at 10:00:05, while B writes and commits at 10:00:02, a sync at 10:00:03 returns B and moves the cursor past 10:00:02. A's row becomes visible later with an older timestamp, behind the cursor, and is never synced. Excluding rows newer than your longest write transaction, or re-reading an overlap window each run, closes the gap. Auto-increment IDs used as cursors have the same problem.

## Common pitfalls
- `render json: @model` without a serializer, leaking `password_digest`, tokens or internal columns.
- Returning 200 with `{ success: false }` for errors, or 500 for validation failures, so clients cannot branch on status.
- N+1 queries inside serializers or Jbuilder partials; nobody notices until a list endpoint gets slow.
- Unbounded `limit` params and deep `OFFSET` pages on big tables.
- `order(params[:sort])` and interpolated `where` strings.
- Breaking response changes shipped without a version while old app builds are still installed.
- Treating CORS as security, or testing rate limits against `:memory_store` or `:null_store`.
- Retrying POSTs from the app without an idempotency key (duplicate payments and bills).
- `Time.now`, `Date.today` and date-only values converted to midnight UTC.
- Sync cursors on `updated_at` alone, bulk updates that skip `updated_at`, and hard deletes the consumer never sees.

## How to talk about this in an interview
- Colochousing is your strongest API story: one Rails 7 app serves ERB and React on Rails pages and a REST API for the Coloc resident app, so versioning, token auth, error envelopes and old app builds are real concerns, not theory.
- For the migration, describe the sync API design (cursors, tombstones, idempotent upserts, service auth) and org-scoped access, and be precise about which parts you built.
- When asked an open design question ("design an endpoint for split bills"), walk through resource and URL, auth, validation and status codes, serializer and preloading, pagination, idempotency and tests.
- If you used a library only lightly (rswag, rack-attack), say so and explain the trade-off you would weigh.

> **Fill in:** how the Coloc app authenticates against the Rails API (token type, lifetime, refresh), whether the API is versioned, and how old app builds are handled.

> **Fill in:** one endpoint you built or changed for the resident app (events, tasks, split bills or rent payments): request shape, serializer, status codes and tests.

> **Fill in:** what the Rails sync APIs exposed to the new TypeScript platform, how the consumer paged through changes, and how the calls were authenticated.

## Interview questions

**Q1. When would you build an API-only Rails app, and when would you add API controllers to an existing app?**
API-only (`--api`) when Rails serves only machine clients (an SPA, mobile apps, other services): a slimmer middleware stack and no view or asset layer. Add `Api::V1` controllers to a full app when the same domain already powers server-rendered pages, which is the usual monolith case.
*Follow-up:* Why use `ActionController::API` inside a full app? No CSRF for token clients, no layouts, less per-request work.

**Q2. What is the difference between 401 and 403, and what do you return for another user's record?**
401 means you are not authenticated (missing or invalid credentials); 403 means you are authenticated but not allowed. For a record that belongs to another user or tenant, return 404 so the API does not confirm it exists.
*Follow-up:* And 422 vs 400? 400 for malformed requests or missing required params; 422 for well-formed input that fails validation.

**Q3. How do you make every API error look the same?**
A base controller with `rescue_from` handlers mapping `RecordNotFound` to 404, `ParameterMissing` to 400, `RecordInvalid` to 422 and authorization errors to 403, all rendering one envelope with a stable `code`, a human `message` and optional `details`.
*Follow-up:* Why a machine-readable code? Clients branch on it; messages change and get translated.

**Q4. Which serializer would you pick for a new Rails API, and why?**
For plain JSON, Blueprinter (explicit, fast, several views per model); jsonapi-serializer if clients want JSON:API; Jbuilder if the team prefers templates. In a legacy app, follow what exists (often ActiveModel::Serializers) rather than mixing styles.
*Follow-up:* Why not `as_json` everywhere? It couples models to the API shape and makes over-exposure easy.

**Q5. How can serializers cause N+1 queries?**
They run per record, so each association access queries the database. Preload in the controller (`includes(:store)`), and use `strict_loading` or Bullet to catch new cases.
*Follow-up:* Attachments? `with_attached_photos` or `includes(photos_attachments: :blob)`.

**Q6. Offset or cursor pagination?**
Offset is simple and supports page numbers but gets slow on deep pages and duplicates or skips items when rows change. Cursor pagination filters past the last seen sort key with a tiebreaker (`(created_at, id) < (?, ?)`), uses an index at any depth and stays stable: the right choice for feeds, infinite scroll and sync.
*Follow-up:* Why the `id` in the cursor? Timestamps collide; the tiebreaker makes the order total.

**Q7. How do you let clients sort and search safely?**
Map sort params through an allowlist hash, bind values with hash conditions or placeholders, escape LIKE input with `sanitize_sql_like`, and cap page sizes.
*Follow-up:* Is `order(params[:sort])` injectable? Rails 6+ blocks non-column strings, but clients could still sort by any column; allowlist anyway.

**Q8. How do you version a Rails API, and why does it matter for mobile apps?**
Namespaced routes and controllers (`/api/v1`, `Api::V1::`), with additive changes inside a version and breaking changes in a new one; V2 controllers can inherit from V1 and override only what changed. Mobile builds stay installed for months and you cannot force everyone to update, so a breaking change can crash the app for real users.
*Follow-up:* URL or header versioning? URL is visible and simple; headers keep URLs clean but are harder to test and cache correctly.

**Q9. How do you force users of an old app build to update?**
The app sends its version in a header; the server compares it with `Gem::Version` against a configurable minimum and returns a dedicated error code, and a config endpoint exposes minimum and latest versions so the app can show a blocking or soft update prompt.
*Follow-up:* Why server-side? You can raise the minimum without shipping a release.

**Q10. What is CORS, and does it protect your API?**
A browser mechanism that lets a server allow cross-origin JavaScript to read its responses, configured in Rails with rack-cors. It does not protect the API: mobile apps, scripts and servers ignore it, so authentication and authorization remain the defense.
*Follow-up:* What is a preflight? An `OPTIONS` request browsers send before non-simple requests to check allowed origins, methods and headers.

**Q11. How do you rate limit a Rails API?**
rack-attack throttles in middleware by IP, path or token, with safelists and blocklists; Rails 7.2+ adds `rate_limit to:, within:, by:` in controllers. Both need a shared cache store, should return 429 with `Retry-After`, and must see the real client IP behind proxies.
*Follow-up:* What do you limit first? Login, password reset, signup and expensive endpoints.

**Q12. What is an idempotency key, and how would you implement one?**
A client-generated key sent with a non-idempotent request and reused on retries. Store it per user under a unique index with a request fingerprint and the response; a repeat replays the response, a different body with the same key is rejected, and a concurrent duplicate gets 409.
*Follow-up:* Why not rely on the client not retrying? Networks drop responses after the server already did the work.

**Q13. How do you handle time zones in an API used across countries?**
Store UTC, compute with `Time.current` in the app zone, render ISO 8601 with offsets, convert to the user's zone for display and scheduling, and keep date-only fields as dates.
*Follow-up:* A classic bug? `Date.today` on a UTC server near midnight returns the wrong day for Brussels users; use `Date.current` inside the user's zone.

**Q14. Design an API that lets another system sync data from your Rails app.**
Cursor on `(updated_at, id)` with a composite index and a settle window, pages with `has_more`, tombstones for deletes, stable IDs for consumer upserts, bulk updates that set `updated_at`, and a scoped service token or HMAC signature per consumer.
*Follow-up:* How do you avoid missing late-committing transactions? Exclude very recent rows or re-read an overlap window every run.

**Q15. How do you document and test an API?**
OpenAPI generated from rswag specs (or a hand-written spec validated in tests), and request specs for each endpoint covering success, validation errors, missing auth and cross-tenant access, asserting status and JSON shape.
*Follow-up:* Why request specs rather than controller specs? They exercise routing, middleware, auth and serialization, which is what clients actually hit.

## Further reading
- [Using Rails for API-only Applications](https://guides.rubyonrails.org/api_app.html)
- [Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html)
- [ActionController::RateLimiting](https://api.rubyonrails.org/classes/ActionController/RateLimiting/ClassMethods.html)
