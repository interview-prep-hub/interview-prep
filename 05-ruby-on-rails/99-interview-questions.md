# Ruby on Rails: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## Rails fundamentals and the request lifecycle (see [Rails fundamentals](01-rails-fundamentals.md))

**Q1. What is Rails, and what does "convention over configuration" mean?**
A full-stack MVC web framework in Ruby. Following naming conventions wires things up without configuration: `Order` maps to the `orders` table, `belongs_to :store` uses `store_id`, `OrdersController#index` renders `orders/index.html.erb`.

**Q2. What happens when a request hits a Rails app?**
Puma parses it into a Rack env and calls `Rails.application`; it passes through the middleware stack (cookies, session, flash), the router picks `controller#action`, before_actions run, the action talks to models and renders a view inside a layout (or redirects), and the response travels back out through the middleware.

**Q3. What is Rack, and what is middleware?**
Rack is the interface between Ruby web servers and apps: an object with `call(env)` returning `[status, headers, body]`. Middleware wraps the app to add behavior to every request (logging, sessions, CORS, throttling); `bin/rails middleware` lists the stack.

**Q4. How does MVC map onto a Rails app?**
Models (ActiveRecord classes) hold data and domain rules, views (ERB, Jbuilder) present it, controllers handle HTTP: they read params, call models and choose a response.

**Q5. What does Zeitwerk do, and what happens when a file name does not match its constant?**
It autoloads constants from file paths (`app/services/billing/refund.rb` defines `Billing::Refund`) and reloads them in development. A mismatch raises `Zeitwerk::NameError`; production fails at boot because it eager loads. `bin/rails zeitwerk:check` catches it.

**Q6. Why does production eager load all code?**
Thread safety (no autoloading while threads serve requests), memory shared between forked Puma workers, and missing-constant errors at boot instead of in a user's request.

**Q7. Puma threads vs workers?**
Threads share one process and suit IO-bound work (the GVL lets one run Ruby at a time); workers are forked processes that use more cores and more RAM. Each thread can hold a DB connection, so the pool must be at least the thread count.

**Q8. Where do secrets go in a Rails app?**
Encrypted credentials (`bin/rails credentials:edit`, optionally per environment) with the key in `config/master.key` or `RAILS_MASTER_KEY`, or environment variables. Never in git as plain text.

**Q9. Gemfile vs Gemfile.lock, and what does `~> 7.1.0` mean?**
The Gemfile lists dependencies and constraints; the lock file pins exact resolved versions for every environment. `~> 7.1.0` allows 7.1.x only; `~> 7.1` allows anything below 8.0.

**Q10. What changes with `rails new app --api`?**
`config.api_only = true`, controllers inherit `ActionController::API`, the middleware stack drops cookies, session and flash, and generators skip views, helpers and assets.

## Routing and controllers (see [Routing and controllers](02-routing-and-controllers.md))

**Q11. Which routes does `resources :orders` generate?**
index (`GET /orders`), new, create (`POST /orders`), show (`GET /orders/:id`), edit, update (`PATCH/PUT /orders/:id`) and destroy (`DELETE /orders/:id`), with helpers like `orders_path` and `edit_order_path(order)`.

**Q12. Member vs collection routes?**
A member route acts on one record and includes the ID (`PATCH /orders/:id/cancel`); a collection route acts on the set (`GET /orders/search`).

**Q13. `namespace` vs `scope` in routes?**
`namespace :admin` changes the URL prefix, the controller module (`Admin::`) and the helper prefix; `scope "/admin"` changes only the URL; `scope module: :admin` changes only the controller module.

**Q14. What are strong parameters?**
An allowlist for mass assignment: `params.require(:order).permit(:note, :address_id)`; unfiltered params raise `ForbiddenAttributesError` and unpermitted keys are dropped. Rails 8's `params.expect(order: [:note])` also checks the shape and returns 400 for tampered input.

**Q15. `render` vs `redirect_to`?**
`render` builds the response from a template in the same request (instance variables kept); `redirect_to` sends a 3xx with `Location`, and the browser makes a new GET. Re-render forms with 422 on failure; redirect after a successful write.

**Q16. How does a before_action stop a request?**
If it renders, redirects or calls `head`, the chain halts and the action never runs; that is how `authenticate_user!` works.

**Q17. `flash` vs `flash.now`?**
`flash` survives one redirect and shows on the next request; `flash.now` shows only in the current render. Mixing them shows a message twice or never.

**Q18. What is wrong with this action?**
```ruby
def update
  redirect_to root_path, alert: "Locked" if @order.locked?
  @order.update!(order_params)
  redirect_to @order
end
```
`redirect_to` does not stop execution: locked orders are still updated, and the second `redirect_to` raises `AbstractController::DoubleRenderError`. Use `return redirect_to(root_path, alert: "Locked") if @order.locked?`.

**Q19. Why does Turbo need 422 on failed forms and 303 after deletes?**
Turbo only renders a form response with a 4xx/5xx status, so a failed save must return 422 or the page looks frozen; 303 See Other guarantees the follow-up request after a DELETE or PATCH is a GET.

## ActiveRecord basics (see [ActiveRecord basics](03-activerecord-basics.md))

**Q20. `save` vs `save!`?**
`save` returns false when validation fails; `save!` raises `ActiveRecord::RecordInvalid`. Use bang methods in jobs, scripts and transactions so failures are not silently ignored.

**Q21. `find` vs `find_by` vs `where`?**
`find(id)` raises `RecordNotFound`; `find_by(attrs)` returns the first match or nil; `where` returns a lazy relation that may be empty.

**Q22. `destroy` vs `delete`?**
`destroy` loads the record and runs callbacks and `dependent:` options; `delete` and `delete_all` issue one SQL DELETE with no callbacks.

**Q23. `update` vs `update_column` vs `update_all`?**
`update` validates and runs callbacks; `update_column(s)` writes directly, skipping validations, callbacks and `updated_at`; `update_all` is one SQL UPDATE across a relation with no callbacks.

**Q24. Why can a uniqueness validation still allow duplicates?**
Two requests can both run the SELECT check before either inserts. Back it with a unique index and rescue `ActiveRecord::RecordNotUnique`.

**Q25. In what order do save callbacks run, and how do you halt them?**
before_validation, after_validation, before_save, around_save, before_create/update, after_create/update, after_save, then after_commit or after_rollback. Halt with `throw :abort`.

**Q26. What is wrong with this model?**
```ruby
class Order < ApplicationRecord
  after_save { OrderConfirmationJob.perform_later(id) }
end
```
`after_save` runs inside the transaction, so the job can run before COMMIT (record not found) or after a rollback (for an order that never existed), and it fires on every update. Use `after_create_commit` or enqueue explicitly from a service after the transaction.

**Q27. How should you declare an enum?**
With explicit values, `enum :status, { pending: 0, paid: 1, shipped: 2 }`, so reordering the list never changes stored integers. You get `paid?`, `paid!` and the `Order.paid` scope.

**Q28. What is `schema.rb` for?**
It is the current schema snapshot used to create fresh databases (`db:schema:load`); migrations are the change history. Commit it, and use `structure.sql` when you rely on database features `schema.rb` cannot express.

## Associations (see [Associations](04-activerecord-associations.md))

**Q29. Which side gets `belongs_to`?**
The model whose table holds the foreign key. Since Rails 5 `belongs_to` is required by default; add `optional: true` when the parent may be missing.

**Q30. `has_many :through` or `has_and_belongs_to_many`?**
`:through` uses a real join model that can have columns, validations and callbacks; HABTM uses a bare join table. Default to `:through`.

**Q31. What do the `dependent:` options do?**
`:destroy` runs callbacks per child, `:delete_all` deletes in one query, `:nullify` clears the foreign key, `:restrict_with_error` / `:restrict_with_exception` block deletion, and `:destroy_async` destroys children in a job.

**Q32. Polymorphic associations vs STI?**
Polymorphic: one child table belongs to several parent types via `*_type` and `*_id` (no real foreign key). STI: subclasses share one table with a `type` column. Use STI only when subclasses share most columns.

**Q33. What do `counter_cache` and `touch` do?**
`counter_cache: true` keeps `parent.children_count` updated so `size` needs no query; `touch: true` bumps the parent's `updated_at` when a child changes, which drives cache invalidation. Bulk SQL bypasses both.

**Q34. What does `accepts_nested_attributes_for` do?**
Lets a parent accept `children_attributes` (with `id` and `_destroy`) to create, update and delete children in one save, typically with `fields_for` in the form.

## Querying and N+1 (see [Querying and N+1](05-activerecord-querying-n-plus-1.md))

**Q35. When does an ActiveRecord query actually run?**
Relations are lazy: `where`, `order` and `joins` build SQL, which runs when you iterate, call `to_a`, `first`, `count`, `pluck` or inspect it.

**Q36. What is an N+1 query, and how do you detect it?**
One query loads N parents, then one more query per parent loads an association. Spot repeated SELECTs in the log, or use Bullet, `strict_loading` or an APM.

**Q37. `includes` vs `preload` vs `eager_load` vs `joins`?**
`preload` runs a separate `WHERE id IN (...)` query per association; `eager_load` uses one LEFT OUTER JOIN; `includes` picks preload unless you reference the association's table; `joins` is an INNER JOIN for filtering that loads nothing.

**Q38. What is wrong with this view?**
```erb
<% @orders.each do |order| %>
  <%= order.customer.name %> (<%= order.items.count %> items)
<% end %>
```
Two N+1s: one query per order for the customer and one COUNT per order. Load `@orders = Order.includes(:customer, :items)` and use `order.items.size`, or add a counter cache.

**Q39. `pluck` vs `select` vs `map`?**
`pluck(:id, :email)` returns arrays without building models; `select(:id, :email)` returns models with only those attributes; `map(&:email)` loads full models first.

**Q40. `exists?` vs `present?`, and `count` vs `size` vs `length`?**
`exists?` runs `SELECT 1 ... LIMIT 1`; `present?` loads every row. `count` always queries, `length` loads all rows, `size` uses loaded records or a counter cache, else COUNT.

**Q41. How do you iterate over a million rows?**
`find_each` or `in_batches` (batches of 1,000 by primary key) instead of `all.each`, which loads everything into memory.

**Q42. What is wrong with `Listing.where("title LIKE '%#{params[:q]}%'")`?**
SQL injection: user input is interpolated into SQL. Use `where("title LIKE ?", "%#{Listing.sanitize_sql_like(params[:q])}%")`.

**Q43. Scopes vs class methods?**
Both build queries, but a scope always returns a relation (a nil result becomes `all`), so it always chains; class methods suit multi-step logic but must return a relation themselves.

**Q44. How do you investigate a slow query?**
`relation.to_sql` to see the SQL, `explain` (or `EXPLAIN ANALYZE` in psql) to find sequential scans, then add the right index (composite column order matters) or rewrite the query.

## Transactions, locking and the database (see [ActiveRecord advanced](06-activerecord-advanced.md))

**Q45. How do transactions work in Rails?**
`ActiveRecord::Base.transaction do ... end` commits if the block finishes and rolls back if it raises (re-raising the error); `raise ActiveRecord::Rollback` rolls back silently. Use bang methods inside, and keep HTTP calls out.

**Q46. Optimistic vs pessimistic locking?**
Optimistic: a `lock_version` column makes stale updates raise `StaleObjectError`; best when conflicts are rare. Pessimistic: `lock` / `with_lock` issue `SELECT ... FOR UPDATE` so other writers wait; best for short hot spots like stock or balances.

**Q47. How do you prevent overselling stock?**
An atomic conditional update, `Listing.where(id: id).where("stock >= ?", qty).update_all(["stock = stock - ?", qty])`, checking that one row changed; or `with_lock` plus a re-check; plus a `CHECK (stock >= 0)` constraint as a safety net.

**Q48. What do `insert_all` and `upsert_all` trade away?**
They write thousands of rows in one statement but skip validations and callbacks; `upsert_all(..., unique_by:)` needs a unique index to resolve conflicts.

**Q49. Validations or database constraints?**
Both: validations give friendly errors; constraints (`null: false`, unique indexes, foreign keys, check constraints) guarantee integrity against races, bulk writes and raw SQL.

**Q50. How do you size the database connection pool?**
At least the number of threads per process (Puma threads, Sidekiq concurrency), with the total across all processes below the database's `max_connections`.

**Q51. Why avoid `default_scope`?**
It silently applies to every query, including associations and `create`, and is easy to forget and awkward to remove (`unscoped` drops everything). Prefer explicit scopes.

## Views, forms and assets (see [Views, ERB and forms](07-views-erb-forms-assets.md))

**Q52. What is the difference between `<% %>` and `<%= %>`?**
`<% %>` runs Ruby without output; `<%= %>` outputs the HTML-escaped result. `<%# %>` is a comment.

**Q53. `html_safe` vs `sanitize`?**
`html_safe` (and `raw`) only marks a string as safe so it is not escaped; `sanitize` actually removes disallowed tags and attributes. Never mark user input as safe.

**Q54. How do partials and collection rendering work?**
Partials are templates starting with `_`; `render "card", listing: @listing` passes locals; `render @listings` renders `_listing.html.erb` for each record. Prefer locals over instance variables inside partials.

**Q55. How does `form_with model: @listing` know where to submit?**
A new record posts to `listings_path` (create); a persisted one sends PATCH (via a hidden `_method`) to `listing_path(@listing)` (update). Field names become `listing[title]`.

**Q56. How did Rails' asset handling evolve?**
Sprockets (3.1 to 6), Webpacker (5.1 to 6, now retired; Shakapacker continues it), Rails 7 import maps by default or jsbundling/cssbundling, Rails 8 Propshaft by default.

## Hotwire (see [Hotwire](08-hotwire-turbo-stimulus.md))

**Q57. What is Hotwire?**
HTML over the wire: the server renders HTML and Turbo swaps it into the page (Drive for navigation, Frames for regions, Streams for targeted updates), with Stimulus adding small JavaScript behaviors. Default since Rails 7.0.

**Q58. Turbo Frames vs Turbo Streams?**
A frame scopes navigation to one region, replacing the matching `turbo_frame_tag` from the response; streams apply actions (append, replace, remove, update) to any elements by ID, from a response or over WebSockets.

**Q59. What problem does Stimulus solve?**
It attaches small JavaScript controllers to server-rendered HTML through data attributes (`data-controller`, `data-action`, targets, values) and reconnects automatically when Turbo swaps content.

**Q60. How do Turbo Stream broadcasts work?**
A model declares `broadcasts_to :house` (or calls `broadcast_*_later_to`), a page subscribes with `turbo_stream_from @house`; on change, the server renders a partial into a stream and pushes it over Action Cable.

**Q61. Hotwire or React?**
Hotwire for CRUD-heavy apps, forms and dashboards built by a Rails team; React for rich client state, offline needs or a separate frontend team. They can coexist with React islands inside Turbo pages.

## Background jobs (see [Background jobs](09-background-jobs-sidekiq.md))

**Q62. Why use background jobs?**
To keep requests fast and isolate slow or unreliable work (emails, third-party APIs, imports, PDFs), with retries, scheduling and fan-out.

**Q63. How does Sidekiq work?**
The client pushes a JSON job onto a Redis list; a separate Sidekiq process runs many threads that pop jobs and call `perform`. Retries, scheduled and dead jobs live in Redis sorted sets.

**Q64. Why must jobs be idempotent?**
Delivery is at least once: retries, crashes and duplicate enqueues can run a job twice. Re-check state, use upserts and unique constraints, and send idempotency keys to external APIs.

**Q65. What is wrong with this job?**
```ruby
class SyncListingJob
  include Sidekiq::Job
  def perform(listing)
    listing.update!(price_cents: PriceClient.fetch(listing.sku))
  end
end
SyncListingJob.perform_async(@listing)
```
Sidekiq arguments must be plain JSON types, so passing a record fails (or arrives as a useless string). Pass `@listing.id`, load the record inside `perform`, and handle it being deleted.

**Q66. Active Job or `Sidekiq::Job`?**
Active Job is backend-neutral with GlobalID arguments and integrations like `deliver_later`; `Sidekiq::Job` is leaner, needs JSON-native arguments and exposes Sidekiq options and paid features directly.

**Q67. What happens when a Sidekiq job keeps failing?**
It is retried with exponential backoff (25 times over about three weeks by default), then moves to the Dead set. Tune with `sidekiq_options retry:` and alert from `sidekiq_retries_exhausted`.

**Q68. Solid Queue or Sidekiq?**
Sidekiq (Redis) is faster with a mature ecosystem and paid features; Solid Queue (Rails 8 default) stores jobs in SQL with `FOR UPDATE SKIP LOCKED` and needs no Redis. Choose by volume, infrastructure and features.

## API development (see [API development](10-api-development.md))

**Q69. Why use `ActionController::API` for JSON endpoints?**
It drops CSRF protection, cookies, flash and layouts that token-authenticated clients do not need, and it cannot accidentally render HTML. Include modules back (token helpers, `respond_to`) when needed.

**Q70. Which status codes for create, invalid input, no token, forbidden and another tenant's record?**
201 (with `Location`), 422, 401, 403, and 404 so the API does not reveal that the record exists.

**Q71. How do you return consistent API errors?**
`rescue_from` handlers in a base controller map `RecordNotFound` to 404, `ParameterMissing` to 400, `RecordInvalid` to 422 and authorization errors to 403, each rendering one envelope with a stable `code`.

**Q72. How can serializers cause N+1 queries?**
They run per record, so each association access queries. Preload in the controller with `includes` and catch regressions with `strict_loading` or Bullet.

**Q73. Offset or cursor pagination?**
Offset supports page numbers but slows on deep pages and shifts when rows change; cursor pagination (`WHERE (created_at, id) < (?, ?)`) is stable and index-friendly, ideal for feeds and sync.

**Q74. Why version an API that a mobile app uses?**
Installed app builds stay in use for months and cannot be forced to update instantly, so breaking changes go to `/api/v2` while v1 keeps working; add a minimum-version check for forced updates.

**Q75. What is CORS, and does it secure your API?**
A browser mechanism (rack-cors in Rails) that lets an API allow cross-origin JavaScript to read responses. It is not security: native apps and servers ignore it.

**Q76. How do you rate limit a Rails API?**
rack-attack throttles in middleware by IP, path or token; Rails 7.2+ adds `rate_limit to:, within:` in controllers. Both need a shared cache store and should return 429 with `Retry-After`.

**Q77. What is an idempotency key?**
A client-generated key sent with a POST and reused on retries; the server stores it under a unique index with the response and replays the response for repeats, so a lost response never causes a double payment.

## Authentication and authorization (see [Authentication and authorization](11-authentication-authorization.md))

**Q78. Authentication vs authorization?**
Authentication establishes who you are (401 or sign-in redirect on failure); authorization decides what you may do with a resource (403, or 404 to hide it).

**Q79. How does `has_secure_password` work?**
With bcrypt and a `password_digest` column it adds `password` and `password_confirmation` attributes, validations and `authenticate`; Rails 7.1's `authenticate_by` adds a timing-safe lookup.

**Q80. How does Devise work, and what are its main modules?**
An engine on Warden, a Rack middleware that keeps the user in the session; `authenticate_user!` triggers Warden and its failure app. Modules: database_authenticatable, registerable, recoverable, rememberable, validatable, confirmable, lockable, trackable, timeoutable, omniauthable.

**Q81. What does the Rails 8 authentication generator leave out?**
Sign-up, email confirmation, lockout, OAuth, 2FA, API tokens and authorization. It gives database-backed sessions, sign-in, sign-out and password reset.

**Q82. How do you secure an API used by a mobile app?**
HTTPS, short-lived access tokens plus rotating refresh tokens stored as digests per device, revocation on sign-out and password change, tokens kept in SecureStore on the phone, rate limits on auth endpoints, and authorization on every endpoint.

**Q83. How do Pundit policies and scopes work?**
A policy class per model receives `user` and `record` and answers predicates like `update?`; `authorize @record` raises `Pundit::NotAuthorizedError` when false; `policy_scope(Model)` returns only the records the user may see via `Scope#resolve`.

**Q84. Pundit or CanCanCan?**
Pundit: small explicit policy classes, scales with contextual rules. CanCanCan: one central `Ability` DSL with automatic resource loading, quick for compact rule sets but tends to grow huge.

**Q85. What is wrong with this customer-facing action?**
```ruby
def show
  @order = Order.find(params[:id])
end
```
An IDOR: any signed-in customer can read any order by changing the ID. Use `current_user.orders.find(params[:id])` or `policy_scope(Order).find(params[:id])`, and test cross-user access.

**Q86. How do you model roles when users belong to several organizations?**
Put the role on a `memberships` join table (user, organization, role enum, unique index), resolve the current organization through the user's memberships, scope every query by it, and check roles in policies.

## Architecture patterns (see [Architecture patterns](12-architecture-patterns.md))

**Q87. What is a service object?**
A class for one use case (`RentalApplications::Approve.call(...)`) with one public method, a result object for expected failures, database writes in a transaction, and jobs or emails enqueued after commit.

**Q88. Concerns, service objects or fat models?**
Model methods for logic about one record's data; a concern for a cohesive trait shared by several models; a service for a workflow coordinating several models or side effects.

**Q89. What is a form object?**
A class with `ActiveModel::Model` and `ActiveModel::Attributes` that backs a form spanning several models or carrying use-case validations, and saves everything in one transaction.

**Q90. What are query objects and presenters for?**
A query object turns many filters into a chainable relation outside the controller; a presenter or decorator wraps a model with display logic like formatted prices.

**Q91. How do you model an order or application lifecycle?**
A status column with explicit allowed transitions, enforced by AASM (events, guards, callbacks) or an enum plus a transitions map, run inside a transaction with a row lock, with side effects after commit.

## Testing (see [Testing with RSpec](13-testing-rspec.md))

**Q92. `let` vs `let!`?**
`let` is lazy and memoized per example; `let!` runs before each example. Use `let!` for records that must exist without being referenced.

**Q93. Mocks vs stubs vs spies?**
A stub returns a canned value (`allow(...).to receive(...).and_return(...)`); a mock is an expectation that a call happens (`expect(...).to receive(...)`); a spy records calls so you assert afterwards (`have_received`). Prefer verified doubles (`instance_double`).

**Q94. Factories vs fixtures, and `build` vs `create` vs `build_stubbed`?**
Fixtures are shared YAML rows loaded once (fast, distant from tests); factories build per-test data with traits (explicit, slower if overused). `build` stays in memory, `create` saves, `build_stubbed` fakes a persisted record and forbids DB access.

**Q95. Request specs or controller specs?**
Request specs exercise routing, middleware, auth and rendering, and have been the recommendation since Rails 5 and RSpec 3.5; controller specs test an action in isolation and are discouraged.

**Q96. How do you test code that calls a third-party API?**
Block real HTTP with WebMock and stub responses (including 429s and timeouts), use VCR cassettes with secrets filtered for realistic payloads, and wrap the API in an adapter you can stub elsewhere.

**Q97. What is wrong with this spec?**
```ruby
let(:published) { create(:post, published: true) }
let(:draft) { create(:post, published: false) }

it "returns only published posts" do
  expect(Post.published).to eq([published])
end
```
`draft` is lazy and never referenced, so it is never created; the test passes even if the scope returns every post. Use `let!(:draft)` (or reference it) so the excluded record exists.

**Q98. Why are system specs flaky, and how do you fix them?**
Timing: assertions that do not wait, `sleep`, JavaScript and Turbo updates, data leaking between tests. Use Capybara's waiting matchers (`have_content`, `have_css`), keep them few, and isolate data.

**Q99. How do you test time-dependent code?**
`travel_to`, `freeze_time` or `travel` from ActiveSupport's time helpers, with `Time.current` and `Date.current` in the code, testing boundaries like midnight and month end.

## Caching, performance and security (see [Caching, performance and security](14-caching-performance-security.md))

**Q100. What is Russian doll caching?**
Nested fragment caches with `touch: true` on child associations: a child change bumps every parent's key, so outer fragments re-render while unchanged inner fragments are reused.

**Q101. What is wrong with this fragment?**
```erb
<% cache "sidebar" do %>
  Hello, <%= current_user.name %>! You have <%= current_user.notifications.unread.count %> unread notifications.
<% end %>
```
User-specific content is cached under one shared key, so every user sees the first user's name and count. Cache per user (`cache [current_user, "sidebar"]`) or keep personal bits outside the cached fragment.

**Q102. How does HTTP caching with ETags work in Rails?**
`fresh_when(@record)` / `stale?(@record)` send an ETag and Last-Modified; when the client's `If-None-Match` matches, Rails returns 304 with no body, skipping rendering and download.

**Q103. How does Rails protect against CSRF, and when is it skipped?**
A per-session authenticity token in forms and `csrf_meta_tags` is verified on every non-GET request by `ActionController::Base`; SameSite cookies add defense. Token-header APIs skip it; cookie-authenticated endpoints must not.

**Q104. How does Rails defend against XSS, SQL injection and mass assignment?**
ERB escapes output by default (`raw`/`html_safe` bypass it, `sanitize` cleans); hash conditions and placeholders bind values (allowlist column names); strong parameters restrict which attributes users can set.

**Q105. Which cookie flags and headers matter?**
`Secure`, `HttpOnly` and `SameSite` on cookies; `force_ssl` for HTTPS redirects and HSTS; default `X-Frame-Options` and `X-Content-Type-Options`; a Content Security Policy rolled out in report-only mode first.

**Q106. Which security tools belong in CI?**
Brakeman (static analysis, in the Rails 7.2+ default CI) and bundler-audit (vulnerable gems), plus dependency update bots and JavaScript audits.

## Action Cable, Action Mailer and Active Storage (see [Action Cable, Mailer and Storage](15-actioncable-mailer-activestorage.md))

**Q107. How does Action Cable work?**
Each browser opens one WebSocket connection (authenticated in `ApplicationCable::Connection`), subscribes to channels that stream from named topics, and server code broadcasts to those streams through a pub/sub adapter (Redis, PostgreSQL or Solid Cable) that reaches every process.

**Q108. `deliver_later` vs `deliver_now`?**
`deliver_later` enqueues a delivery job with serialized arguments (enqueue after commit); `deliver_now` sends synchronously, which slows the request and fails it on provider errors.

**Q109. How do Active Storage direct uploads work?**
The browser sends file metadata to Rails, gets a signed upload URL, PUTs the file straight to S3 or GCS, and submits only the blob's `signed_id`; app servers never stream the bytes. The bucket needs CORS.

**Q110. How do you serve private files with Active Storage?**
Not through the default blob URLs, which do not expire or authorize. Authorize in your own controller, then redirect to a short-lived service URL (`file.url(expires_in: 5.minutes)`) or stream the file.

## React on Rails (see [React on Rails](16-react-on-rails.md))

**Q111. How does React on Rails pass props and do server-side rendering?**
`react_component("Name", props:)` serializes a hash into the page as JSON next to a placeholder; the Shakapacker client bundle renders the registered component there. With `prerender: true`, Rails runs the server bundle in embedded V8 (ExecJS/mini_racer, or Pro's Node renderer) and the client hydrates the HTML.

**Q112. What causes a hydration mismatch?**
The first client render differs from the server HTML: dates and time zones, random IDs, `typeof window` branches, different data or invalid HTML nesting. Pass values as props, use `useId`, and render browser-only parts after mount.

**Q113. React on Rails, Inertia or a separate SPA?**
React on Rails embeds components in ERB pages (gradual adoption, optional SSR); Inertia replaces ERB views with React pages while keeping Rails routing; a separate SPA consumes a JSON API, suiting separate teams and several clients.

## Legacy codebases and upgrades (see [Legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md))

**Q114. How do you upgrade a Rails app?**
One minor version at a time: green tests on the latest patch, fix deprecations, upgrade blocking gems, bump Rails, run `bin/rails app:update`, deploy through staging, then enable new framework defaults one by one and bump `load_defaults`. Dual boot for long upgrades.

**Q115. How do you rename a column with zero downtime?**
Expand/contract: add the new column, write to both, backfill in batches, switch reads, stop writing the old one, add it to `ignored_columns`, deploy, then drop it. Never `rename_column` on a live table.

**Q116. What do `load_defaults` and `new_framework_defaults_X_Y.rb` do?**
`load_defaults` turns on the framework behaviors of a version; after an upgrade you keep the old value, enable the generated settings one at a time with deploys in between, then bump it and delete the file.

**Q117. Why keep data backfills out of schema migrations?**
They hold long locks and transactions, block deploys, cannot resume after failure, and break `db:migrate` later when models change. Use maintenance_tasks, data-migrate or batched jobs that are idempotent and throttled.

## Rails 7 and 8 (see [Rails 7 and 8: what's new](18-rails-7-8-whats-new.md))

**Q118. What is new in Rails 7 and Rails 8?**
7.0: Hotwire and import maps by default, encryption, `load_async`; 7.1: Dockerfile, `normalizes`, `generates_token_for`, composite keys; 7.2: `allow_browser`, default CI, `rate_limit`. 8.0: Solid Queue, Cache and Cable, Kamal 2 with Thruster, Propshaft, the authentication generator, `params.expect`; 8.1: job continuations and structured events.

**Q119. What is the "no Redis by default" story in Rails 8?**
New apps run jobs (Solid Queue), cache (Solid Cache) and Action Cable (Solid Cable) on the database, removing a service to run. Existing apps can keep Sidekiq and Redis; the change is a default, not a requirement.

**Q120. What are Kamal and Thruster?**
Kamal deploys Docker images to your own servers over SSH, with kamal-proxy switching traffic after health checks and handling TLS; Thruster is a small proxy in front of Puma inside the container adding HTTP/2, compression, asset caching and X-Sendfile.
