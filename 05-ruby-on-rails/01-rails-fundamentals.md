# Rails Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Routing and controllers](02-routing-and-controllers.md), [ActiveRecord basics](03-activerecord-basics.md), [How the web works](../04-web-fundamentals/01-how-the-web-works.md), [MVC and layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md), [Ruby metaprogramming](../03-languages/ruby/05-metaprogramming.md), [Legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md)
> **On your resume:** Ruby on Rails across Devntech client work (legacy Rails monoliths, CrystalCommerce, the Colochousing Rails 7 app) and the Visnext Rails training (MVC, ActiveRecord, Hotwire).

## Quick revision
- Rails is a full-stack MVC framework in Ruby. Core ideas: **convention over configuration** (follow naming rules, skip the config), **DRY** (one home for each piece of knowledge) and **omakase** (a curated default stack you can swap out).
- Request path: client -> proxy -> **Puma** -> **Rack** -> **middleware stack** -> **router** -> **controller#action** -> **model** (SQL) -> **view** (ERB inside a layout) -> response travels back out through the middleware.
- **Rack** is the contract between Ruby web servers and apps: an object with `call(env)` that returns `[status, headers, body]`. Rails itself is a Rack app (`Rails.application`).
- **Middleware** wraps the app: `initialize(app)` plus a `call(env)` that calls `@app.call(env)`. List it with `bin/rails middleware`, add with `config.middleware.use` or `insert_before`.
- Boot order: `config/boot.rb` (Bundler, bootsnap) -> `config/application.rb` (frameworks, `Bundler.require`, settings) -> `config/environment.rb` (`Rails.application.initialize!`) -> `config/environments/<env>.rb` -> `config/initializers/*.rb` (alphabetical) -> `config.ru` (`run Rails.application`).
- Environments: development (code reloading, lazy loading, detailed errors), test, production (eager loading, caching, SSL, quieter logs). Check with `Rails.env.production?`.
- Credentials: `bin/rails credentials:edit` edits the encrypted `config/credentials.yml.enc`; the key is `config/master.key` (gitignored) or the `RAILS_MASTER_KEY` env var.
- **Zeitwerk** maps file paths to constants: `app/services/billing/charge_card.rb` must define `Billing::ChargeCard`. Never `require` app code yourself. `bin/rails zeitwerk:check` validates names.
- Development loads code lazily and reloads it; production **eager loads** everything at boot (`config.eager_load = true`) for thread safety, shared memory across forked workers and fail-fast boots.
- **Puma threads** share one process (good for IO-bound requests; the GVL lets only one thread run Ruby at a time). **Workers** are forked processes that use more CPU cores. Concurrency = workers x threads. DB pool >= threads per process.
- Generators: `bin/rails g model|controller|migration|scaffold ...`, undo with `bin/rails destroy`.
- Console: `reload!`, `bin/rails c --sandbox` (rolls back on exit), `app.get "/path"`, `helper.number_to_currency(9.5)`. `bin/rails runner` runs Ruby with the app loaded; custom tasks live in `lib/tasks/*.rake` and depend on `:environment`.
- Gemfile = what you want (constraints like `~> 7.1`); Gemfile.lock = exact versions resolved. Commit both for apps and run commands with `bundle exec` (binstubs like `bin/rails` do this for you).
- ActiveSupport: `blank?` / `present?` / `presence`, `try` (nil-safe) vs Ruby's `&.`, `3.days.ago`, and `Time.current` (app time zone) instead of `Time.now`.
- Railtie = the hook a gem or framework uses to plug into boot; Engine = a Railtie with its own `app/`, routes and migrations (Devise is one); `Rails::Application` is itself an Engine.
- `rails new app --api`: controllers inherit `ActionController::API`, slimmer middleware (no cookies, session or flash by default), no views or asset pipeline.

## 1. What Rails is

Rails is an opinionated, full-stack web framework written in Ruby. David Heinemeier Hansson extracted it from Basecamp and open-sourced it in 2004. "Full stack" means one install gives you everything a typical web product needs, as a set of gems that work together:

| Component | Job |
|---|---|
| Active Record | ORM: models, migrations, validations, associations, queries |
| Action Pack (Action Dispatch + Action Controller) | Routing, middleware, controllers, params, sessions, cookies |
| Action View | Templates (ERB), layouts, partials, helpers, forms |
| Active Job | Background job API on top of Sidekiq, Solid Queue and others |
| Action Mailer, Action Cable, Active Storage | Email, WebSockets, file uploads |
| Active Support | Core extensions (`3.days.ago`, `blank?`), concerns, caching, instrumentation |

The philosophy matters in interviews because it explains why Rails code looks the way it does:

| Principle | Meaning | Example |
|---|---|---|
| Convention over configuration | Names imply behavior, so you write config only when you break a convention | `Product` maps to the `products` table; `ProductsController#show` renders `app/views/products/show.html.erb` with no code |
| DRY (Don't Repeat Yourself) | Every piece of knowledge has one authoritative place | Validations live in the model, not in every controller; repeated markup goes into partials and helpers |
| Omakase | Like a chef's tasting menu, Rails picks a default stack for you | New apps get Minitest, Hotwire, import maps, Solid Queue (8.0); you can swap in RSpec, esbuild, Sidekiq (see [Rails 7 and 8](18-rails-7-8-whats-new.md)) |

Strengths: very fast to build CRUD-heavy products, a mature gem ecosystem, a single "majestic monolith" that a small team can own end to end. Costs: the "magic" (callbacks, autoloading, metaprogramming) hides control flow, so large legacy monoliths need discipline; raw per-request speed is lower than compiled stacks, though the database is usually the real bottleneck.

## 2. MVC and the full request lifecycle

**MVC** splits responsibilities: the **Model** holds data and business rules (Active Record classes), the **View** turns data into HTML or JSON (ERB, Jbuilder), and the **Controller** handles the HTTP request, talks to models and picks a response. Here is what happens for `GET /products/42`:

```text
Browser: GET /products/42
  -> Reverse proxy / load balancer (Nginx, Thruster, cloud LB): TLS, buffering, static files
  -> Puma: accepts the connection, parses HTTP, builds the Rack env hash, hands it to a thread
  -> Rails.application.call(env)
  -> Middleware (outer to inner): host check, static files, request id, logger,
     exception pages, cookies, session, flash, CSP, ETag ...
  -> Router (config/routes.rb): GET /products/:id => ProductsController#show, params[:id] = "42"
  -> Controller: before_actions, then #show
       -> Model: Product.find(42) -> connection from the pool -> SQL -> PostgreSQL -> Product object
       -> View: products/show.html.erb rendered inside layouts/application.html.erb
  <- [200, headers, body] travels back OUT through the middleware (ETag, Set-Cookie, log line)
  <- Puma writes the HTTP response to the socket
```

Step by step:
1. **Puma** builds the `env` hash (`REQUEST_METHOD`, `PATH_INFO`, `QUERY_STRING`, `HTTP_*` headers, `rack.input` for the body) and calls the Rack app on one of its threads.
2. **Middleware** layers each get a chance to inspect or change the request and response, or to short-circuit: `ActionDispatch::Static` can return a file from `public/` before the router ever runs, and `HostAuthorization` rejects unknown hosts.
3. The **router** matches verb + path against `config/routes.rb` top to bottom (first match wins), extracts path params and dispatches to `ProductsController.action(:show)`, which is itself a Rack endpoint.
4. Rails creates a **new controller instance per request** (so instance variables never leak between requests), runs callbacks, then the action. The action either renders explicitly, redirects, or falls through to the implicit render of `products/show`.
5. The **model** builds SQL, checks out a DB connection from the pool, and maps rows to objects.
6. The **view** gets the controller's instance variables, renders inside the layout (`yield`), and uses partials and helpers.
7. The response goes back out through every middleware in reverse order, and the log records it.

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :products, only: [:index, :show]
end

# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id]) # RecordNotFound becomes a 404 in production
  end                                    # no render call: Rails renders products/show
end

# app/models/product.rb
class Product < ApplicationRecord
  validates :name, presence: true
end
```

```erb
<%# app/views/products/show.html.erb %>
<h1><%= @product.name %></h1>
<p><%= number_to_currency(@product.price) %></p>
```

The development log shows the whole lifecycle, and reading it is your first debugging tool (format varies slightly by version):

```text
Started GET "/products/42" for 127.0.0.1
Processing by ProductsController#show as HTML
  Parameters: {"id"=>"42"}
  Product Load (0.4ms)  SELECT "products".* FROM "products" WHERE "products"."id" = $1 LIMIT $2
  Rendered products/show.html.erb within layouts/application (Duration: 6.1ms)
Completed 200 OK in 18ms (Views: 9.8ms | ActiveRecord: 0.4ms)
```

## 3. Rack: the interface under Rails

Before Rack, every Ruby framework needed custom adapters for every server. Rack defines one tiny interface: **an app is any object that responds to `call(env)` and returns a three-element array `[status, headers, body]`**, where `body` responds to `each` (an array of strings works). Puma, Unicorn and other servers speak Rack, and Rails, Sinatra and Hanami are Rack apps, so any server can run any framework.

```ruby
# config.ru: a complete Rack app (run it with `bundle exec puma`, which loads config.ru)
class HelloApp
  def call(env)
    request = Rack::Request.new(env)
    name = request.params["name"] || "world"
    [200, { "content-type" => "text/plain" }, ["Hello, #{name}!"]]
  end
end

run HelloApp.new
```

Header names are lowercase because Rack 3 requires it (Rack 2 accepted either). A lambda also works as an app: `run ->(env) { [200, {}, ["ok"]] }`.

**Middleware** is a Rack app that wraps another Rack app. It receives the next app in `initialize`, and in `call` it can act before and after passing the request inward:

```ruby
# lib/middleware/request_timer.rb
class RequestTimer
  def initialize(app)
    @app = app
  end

  def call(env)
    started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)              # everything inside runs here
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started
    headers["x-elapsed-ms"] = (elapsed * 1000).round(1).to_s
    [status, headers, body]
  end
end
```

```ruby
# config/application.rb
require_relative "../lib/middleware/request_timer"

module MyApp
  class Application < Rails::Application
    config.middleware.use RequestTimer                        # append (innermost)
    # config.middleware.insert_before Rack::Runtime, RequestTimer
    # config.middleware.delete Rack::Runtime
  end
end
```

The stack is built once at boot, so middleware does not reload in development; keep it outside reloadable autoload paths (if your app uses `config.autoload_lib`, add `middleware` to its `ignore:` list) and require it explicitly. The generated `config/initializers/cors.rb` registers the `rack-cors` middleware the same way, from an initializer.

`bin/rails middleware` prints the stack for your app (trimmed; the exact list depends on version and config):

```text
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use Rack::Runtime
use Rack::MethodOverride
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use ActionDispatch::DebugExceptions
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
use ActionDispatch::Flash
use ActionDispatch::ContentSecurityPolicy::Middleware
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
run MyApp::Application.routes
```

| Middleware | What it does |
|---|---|
| `ActionDispatch::HostAuthorization` | Rejects requests whose `Host` header is not allowed (DNS rebinding protection) |
| `ActionDispatch::Static` | Serves files from `public/` without hitting the router |
| `Rack::MethodOverride` | Turns a POST with `_method=patch` into PATCH, because HTML forms only send GET and POST |
| `ActionDispatch::RequestId` | Sets `X-Request-Id`, which you can tag in logs to trace one request |
| `ActionDispatch::RemoteIp` | Works out the real client IP behind proxies |
| `ActionDispatch::ShowExceptions` / `DebugExceptions` | Turn exceptions into error pages (detailed in development, `public/500.html` in production) |
| `ActiveRecord::Migration::CheckPending` | In development, raises if you have pending migrations |
| `Cookies`, `Session::CookieStore`, `Flash` | Cookie jar, session (stored in a signed, encrypted cookie by default), flash messages |
| `Rack::ETag`, `Rack::ConditionalGet` | Add an ETag and answer `304 Not Modified` when the client copy is fresh |
| Gems: `Rack::Attack`, `Rack::Cors` | Throttling and blocking abusive clients; CORS headers for browser API calls |

**Middleware vs `before_action`:** middleware runs for every request (including mounted apps and static files) and knows nothing about controllers or `current_user`; use it for cross-cutting HTTP concerns (timing, CORS, throttling, request IDs). A `before_action` runs inside a controller with access to params, session and models; use it for authentication and loading records.

## 4. Directory structure

```text
myapp/
  app/
    assets/          stylesheets and images (Propshaft or Sprockets)
    channels/        Action Cable channels
    controllers/     ApplicationController, concerns/
    helpers/         view helpers
    javascript/      Stimulus controllers and entry points
    jobs/            Active Job classes
    mailers/         Action Mailer classes
    models/          Active Record models, concerns/
    views/           templates, layouts/, partials
  bin/               rails, setup, dev (binstubs that load Bundler first)
  config/
    application.rb   settings shared by every environment
    environments/    development.rb, test.rb, production.rb
    initializers/    Ruby files run once at boot, in alphabetical order
    locales/         I18n YAML files
    credentials.yml.enc, master.key (gitignored)
    database.yml, routes.rb, puma.rb, storage.yml
  db/                migrate/, schema.rb (or structure.sql), seeds.rb
  lib/               code that is not specific to this app; tasks/ for rake tasks
  log/ tmp/ storage/ logs, caches and pids, local Active Storage files
  public/            static files served as-is (404.html, robots.txt)
  test/ or spec/     Minitest or RSpec
  config.ru          Rack entry point
  Gemfile, Gemfile.lock, Rakefile, Dockerfile (generated since 7.1)
```

Any new folder under `app/` (for example `app/services`, `app/queries`, `app/policies`) is autoloaded automatically after a server restart, which is why legacy apps grow folders like these.

## 5. Boot process and config.ru

What happens when you run `bin/rails server` (or when Puma loads `config.ru` in production):

```ruby
# config/boot.rb
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)
require "bundler/setup"   # put the locked gem versions on the load path
require "bootsnap/setup"  # cache expensive boot work (load path scans, compiled Ruby)

# config/application.rb (trimmed)
require_relative "boot"
require "rails/all"                # or require only the frameworks you use
Bundler.require(*Rails.groups)     # require gems in the default group + current env group

module MyApp
  class Application < Rails::Application
    config.load_defaults 7.1       # opt into the framework defaults of that version
    config.time_zone = "Karachi"
  end
end

# config/environment.rb
require_relative "application"
Rails.application.initialize!      # run every initializer

# config.ru
require_relative "config/environment"
run Rails.application
Rails.application.load_server      # present in newer generated apps
```

`initialize!` runs the framework Railtie initializers (Active Record connection handling, Action Mailer settings and so on), then loads `config/environments/<RAILS_ENV>.rb`, then every file in `config/initializers/` sorted by name (prefix with numbers if order matters), then `after_initialize` hooks, and finally eager loads the app when `eager_load` is on. `config.load_defaults` matters during upgrades: you bump it only after working through the new defaults one by one (see [legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md)).

## 6. Environments and configuration

`RAILS_ENV` (or `RACK_ENV`) picks the environment; it defaults to development. Settings in `config/application.rb` apply everywhere and `config/environments/*.rb` override them.

```ruby
# config/environments/development.rb (trimmed)
Rails.application.configure do
  config.enable_reloading = true             # older apps: config.cache_classes = false
  config.eager_load = false
  config.consider_all_requests_local = true  # detailed error pages with a console
end

# config/environments/production.rb (trimmed)
Rails.application.configure do
  config.enable_reloading = false
  config.eager_load = true
  config.consider_all_requests_local = false # generic public/500.html
  config.force_ssl = true                    # redirect to HTTPS, secure cookies, HSTS
  config.log_level = :info
  config.active_job.queue_adapter = :sidekiq
end
```

App-specific settings have three common homes:

```ruby
# 1. config.x namespace
config.x.catalog_sync.batch_size = 500
Rails.configuration.x.catalog_sync.batch_size # => 500

# 2. A YAML file with one block per environment (config/catalog_sync.yml)
config.catalog_sync = config_for(:catalog_sync)
Rails.configuration.catalog_sync[:batch_size]

# 3. Environment variables: fetch fails fast at boot if the value is missing
ENV.fetch("REDIS_URL")
```

**Initializers** configure gems once at boot. Changing one needs a server restart because `reload!` does not re-run them.

```ruby
# config/initializers/sidekiq.rb
redis_config = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
Sidekiq.configure_server { |config| config.redis = redis_config }
Sidekiq.configure_client { |config| config.redis = redis_config }
```

## 7. Credentials

Credentials keep secrets (API tokens, `secret_key_base`) in an encrypted file you can commit.

```bash
EDITOR="code --wait" bin/rails credentials:edit           # config/credentials.yml.enc + config/master.key
bin/rails credentials:edit --environment production       # config/credentials/production.yml.enc + production.key
bin/rails credentials:show
```

```yaml
# Decrypted content you see in the editor
secret_key_base: 3b1f0c...
pricecharting:
  api_token: abc123
```

```ruby
Rails.application.credentials.dig(:pricecharting, :api_token)
Rails.application.credentials.secret_key_base
```

Rules: commit the `.enc` file, never the key; in production pass the key as `RAILS_MASTER_KEY` from the platform's secret store; if the key is lost, the secrets are gone and must be re-created. `secret_key_base` signs and encrypts cookies and sessions, so rotating it logs everyone out. Many teams use plain ENV vars for infrastructure (`DATABASE_URL`, `REDIS_URL`) and credentials for third-party keys; pick one convention per app and document it.

## 8. Zeitwerk autoloading

Rails 6 introduced the Zeitwerk loader and Rails 7 removed the old classic autoloader. You never `require` your own app files: Zeitwerk knows which constant each file defines from its path.

- Every directory directly under `app/` (`models`, `controllers`, `services`, `models/concerns`, ...) is a **root**, so `app/services/foo.rb` defines `Foo`, not `Services::Foo`.
- File names are snake_case, constants are the camelized name: `app/models/order_item.rb` -> `OrderItem`.
- Subdirectories are namespaces: `app/services/pricecharting/sync_listings.rb` -> `Pricecharting::SyncListings`. The module is created implicitly if there is no `pricecharting.rb`.
- Acronyms: `api_client.rb` -> `ApiClient`. If you want `APIClient`, add `inflect.acronym "API"` in `config/initializers/inflections.rb` (this affects every `api` in the app, so decide early).
- One constant per file. A second top-level class hidden in another file is never found.

```text
Zeitwerk::NameError: expected file app/services/price_charting/client.rb
to define constant PriceCharting::Client, but didn't
```

Fix by renaming the file or the class, and run `bin/rails zeitwerk:check` in CI to catch these before production eager loading does.

| | Development | Production |
|---|---|---|
| Loading | Lazy: a constant loads when first referenced | Eager: all autoload paths load at boot |
| Reloading | Yes: changed files reload on the next request, or `reload!` in the console | No |
| Why | Fast boot and an edit-refresh loop | Thread safety (no loading in the middle of concurrent requests), copy-on-write memory with forked workers, and missing-constant errors fail the deploy instead of a user's request |
| Setting | `config.eager_load = false` | `config.eager_load = true` |

`lib/` is not autoloaded by default; Rails 7.1 added `config.autoload_lib(ignore: %w[assets tasks])` to opt in. Avoid referencing reloadable classes (models) from initializers; wrap such code in `Rails.application.config.to_prepare { ... }` so it re-runs after each reload.

## 9. Puma: threads vs workers

Puma is the default app server. It runs **threads** inside one process and, in cluster mode, several forked **worker** processes.

```ruby
# config/puma.rb (simplified)
threads_count = ENV.fetch("RAILS_MAX_THREADS", 3).to_i
threads threads_count, threads_count          # min and max threads per process
workers ENV.fetch("WEB_CONCURRENCY", 2).to_i  # forked processes (cluster mode)
preload_app!                                  # load the app once, then fork (copy-on-write)
port ENV.fetch("PORT", 3000)
```

| | Threads | Workers (processes) |
|---|---|---|
| Memory | Shared and cheap | Each worker has its own heap; `preload_app!` shares unchanged pages |
| CPU parallelism | The GVL lets one thread run Ruby at a time; others proceed while waiting on IO | True parallelism across cores |
| Best for | IO-bound requests (DB queries, HTTP calls, Redis) | Using every core, CPU-heavy work |
| Risk | Thread-safety bugs in shared mutable state | More RAM |

Capacity = workers x threads per server (2 x 3 = 6 requests in flight). Each thread can hold a database connection, so `pool` in `database.yml` must be at least the threads per process, and total connections (servers x workers x threads, plus every Sidekiq process's concurrency) must stay under Postgres `max_connections`. See [connection pools](06-activerecord-advanced.md) for the full math.

```yaml
# config/database.yml
production:
  adapter: postgresql
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

Because threads share memory, never keep per-request data in class variables, class-level instance variables or globals. Use `ActiveSupport::CurrentAttributes` (a `Current` model) instead, which Rails resets for every request.

## 10. Generators, console, runner and tasks

```bash
bin/rails new shop -d postgresql                     # add --api, -j esbuild, --css tailwind ...
bin/rails g model Product name:string "price:decimal{10,2}" store:references
bin/rails g migration AddSkuToProducts sku:string:index
bin/rails g controller Products index show
bin/rails g scaffold Post title:string body:text     # migration, model, controller, views, routes, tests
bin/rails destroy model Product                      # undo a generator
```

Quote `"price:decimal{10,2}"` or the shell expands the braces. Scaffolds are great for learning the conventions and for quick admin screens; production controllers are usually hand-written.

```ruby
# bin/rails console --sandbox   (every change is rolled back when you exit)
reload!                                   # load code changes; objects already in variables keep old code
app.get "/products/42"                    # send a fake request through the full stack
app.response.status                       # => 200
app.products_path                         # => "/products"
helper.number_to_currency(1234.5)         # => "$1,234.50"
Product.where(store_id: 3).to_sql         # see the SQL without running it
Product.instance_method(:total).source_location # where is this method defined?
```

```ruby
# lib/tasks/catalog.rake
namespace :catalog do
  desc "Reset cached product counts for every store"
  task recount: :environment do         # :environment boots the app first
    Store.find_each { |store| Store.reset_counters(store.id, :products) }
  end
end
```

Run it with `bin/rails catalog:recount` and list tasks with `bin/rails -T`. `bin/rails runner script/backfill.rb` (or an inline string) runs a script with the app loaded. For production data fixes, prefer a reviewed, idempotent task or job over pasting code into a console. To pause code while debugging, call `debugger` (the `debug` gem) or `binding.irb`.

## 11. Bundler: Gemfile and Gemfile.lock

```ruby
# Gemfile
source "https://rubygems.org"

gem "rails", "~> 7.1.0"         # >= 7.1.0 and < 7.2 (pessimistic operator)
gem "pg", "~> 1.5"              # >= 1.5 and < 2.0
gem "sidekiq"
gem "bootsnap", require: false  # installed but not auto-required by Bundler.require

group :development, :test do
  gem "rspec-rails"
  gem "debug"
end
```

| Gemfile | Gemfile.lock |
|---|---|
| Written by you: gem names, sources, version constraints, groups | Written by Bundler: the exact version of every gem, including transitive dependencies |
| Loose (`~> 7.1`) | Exact (`rails (7.1.3)`) |
| Changes when you add or remove gems | Changes on `bundle install` with new gems, or `bundle update` |

`bundle install` installs what the lock file says (resolving only new entries); `bundle update sidekiq` upgrades one gem within its constraints (add `--conservative` to leave shared dependencies alone); `bundle exec` runs a command with the locked versions. Commit `Gemfile.lock` so every laptop, CI run and server uses identical versions, and never run a bare `bundle update` on a legacy monolith: it upgrades everything at once.

## 12. ActiveSupport core extensions

```ruby
nil.blank?          # => true
"   ".blank?        # => true  (whitespace only)
[].blank?           # => true
0.blank?            # => false (numbers are never blank)
"hi".present?       # => true  (present? is !blank?)

params[:sort].presence || "newest"   # presence returns the value, or nil when blank

user.try(:name)     # nil if user is nil, and also nil if user does not respond to :name
user&.name          # plain Ruby: nil if user is nil, NoMethodError if the method is missing

3.days.ago
2.hours.from_now
Date.current.beginning_of_month
Time.current        # in config.time_zone; Time.now uses the server's zone
"order_item".camelize          # => "OrderItem"
"person".pluralize             # => "people"
%w[a b c].to_sentence          # => "a, b, and c"
{ "a" => 1 }.with_indifferent_access[:a] # => 1
```

Prefer `&.` when you expect the method to exist, because `try` silently hides typos. In tests, `travel_to` and `freeze_time` (from `ActiveSupport::Testing::TimeHelpers`) control the clock. See [Ruby basics](../03-languages/ruby/01-basics.md) for the plain-Ruby side.

## 13. Engines and Railties

A **Railtie** is how a framework or gem plugs into the Rails boot: it can add initializers, config options, rake tasks and generators. Active Record, Action Mailer and gems such as `sidekiq` or `rspec-rails` ship one. An **Engine** is a Railtie that also brings its own `app/` folder, routes and migrations: a mini application packaged as a gem. Devise, Active Storage and Action Text are engines. A mountable engine lives in its own namespace and is attached to a path with `mount Blog::Engine, at: "/blog"` (generate one with `bin/rails plugin new blog --mountable`). The hierarchy is `Rails::Application < Rails::Engine < Rails::Railtie`.

### Going deeper: engines as boundaries in a monolith
Large monoliths sometimes move a domain (billing, catalog sync) into an in-repo engine under `engines/` or `components/`, with its own models, routes and tests. The usual rule is that the engine may depend on core code but not the other way around, which makes dependencies visible and is a cheaper step than extracting a separate service.

## 14. Full app vs API-only

`rails new backend --api` sets `config.api_only = true`, makes `ApplicationController` inherit from `ActionController::API` (no view helpers, cookies, flash or CSRF protection by default), trims the middleware stack (no cookies, session or flash; add one back with `config.middleware.use ActionDispatch::Cookies`) and makes generators skip views and assets. Choose it for a pure JSON backend for an SPA or a mobile app. Many real products are full apps with a JSON namespace instead, for example `Api::V1::BaseController < ActionController::API` serving a React Native app next to the HTML controllers. See [API development](10-api-development.md).

## Common pitfalls
- File and constant names that disagree: the error may only appear when that constant is first used in development, but it breaks production boot. Run `bin/rails zeitwerk:check` in CI.
- Calling `require` on app files, which bypasses reloading and can load a file twice.
- Using `Time.now` or `Date.today` instead of `Time.current` / `Date.current`, which causes off-by-five-hours bugs between a UTC server and Pakistan time, especially around midnight.
- Storing per-request state in class variables or globals: fine with one thread in development, a data leak between users under Puma threads.
- Committing `config/master.key`, or keeping secrets in plain YAML.
- A DB pool smaller than Puma threads or Sidekiq concurrency, giving `ActiveRecord::ConnectionTimeoutError` under load.
- Editing an initializer and expecting `reload!` to apply it (restart the server).
- Putting expensive work in middleware, which runs for every request including static assets and health checks.

## How to talk about this in an interview
- Frame fundamentals as "how I find my way around an unfamiliar Rails codebase", which is exactly the Devntech legacy monolith work: read `config/routes.rb` (or `bin/rails routes -g`), follow one request through the log, scan the Gemfile and `config/initializers` to learn what the app depends on, and explore data with `bin/rails c --sandbox`.
- Present the Visnext training (MVC, ActiveRecord, Hotwire) as your structured foundation and the client work since November 2025 as where you applied it. Be honest about the timeline; depth of reasoning matters more than years.
- For Colochousing (Rails 7, React on Rails, ERB, Sidekiq), talk about config only where you touched it, for example a Sidekiq initializer, credentials or environment-specific settings.
- Good phrasing: "When I pick up a ticket in a legacy area, I start from the request lifecycle: route, controller, model, view, and I read the development log while I click through the feature."

> **Fill in:** which Ruby and Rails versions the CrystalCommerce and Colochousing apps run, and whether secrets live in credentials or ENV vars.

> **Fill in:** one bug you traced through the request lifecycle (log, route, controller, model) and what the cause turned out to be.

## Interview questions

**Q1. Explain the Rails request lifecycle.**
Puma (often behind Nginx or a load balancer) parses the HTTP request into a Rack env hash and calls `Rails.application`. The request passes through the middleware stack (host check, static files, logging, cookies, session, flash), then the router matches verb and path to `controller#action` and builds params. Rails creates a new controller instance, runs before_actions and the action, which queries models through a pooled DB connection and renders a view inside a layout (or redirects). The `[status, headers, body]` result travels back out through the middleware and Puma writes it to the socket.
*Follow-up:* Where does a 404 for `Product.find(999)` come from? `find` raises `ActiveRecord::RecordNotFound`, which the exception-handling middleware maps to a 404 response in production.

**Q2. What is Rack? Can you name some middleware?**
Rack is the interface between Ruby web servers and frameworks: an app is any object with `call(env)` returning `[status, headers, body]`, and middleware is an app that wraps another app. Rails is a Rack app with a middleware stack such as `ActionDispatch::Static`, `Rack::MethodOverride`, `ActionDispatch::RequestId`, `ActionDispatch::Cookies`, `ActionDispatch::Session::CookieStore`, `ActionDispatch::Flash` and `Rack::ETag`; `bin/rails middleware` lists them.
*Follow-up:* How does a form send PATCH when browsers only send GET and POST? Rails adds a hidden `_method=patch` field and `Rack::MethodOverride` rewrites the request method.

**Q3. What does convention over configuration mean in Rails?**
Names imply wiring, so you only configure exceptions: `Order` maps to the `orders` table, `belongs_to :store` uses `store_id`, `OrdersController#index` renders `orders/index.html.erb`, and `app/services/billing/refund.rb` defines `Billing::Refund`.
*Follow-up:* How do you break a convention? Explicit options: `self.table_name = "legacy_orders"`, `class_name:` and `foreign_key:` on associations, `render :other_template`.

**Q4. How do you write a custom middleware, and when would you use one instead of a before_action?**
A class with `initialize(app)` and `call(env)` that calls `@app.call(env)` and returns the triple; register it with `config.middleware.use` or `insert_before`. Use middleware for cross-cutting HTTP concerns that must run for every request (timing, CORS, throttling with Rack::Attack, request IDs); use a before_action when you need controller context such as params or `current_user`.
*Follow-up:* Does order matter? Yes: a throttle should run before expensive layers, and anything reading the session must sit after the session middleware.

**Q5. What happens when a Rails app boots?**
`config/boot.rb` sets up Bundler and bootsnap; `config/application.rb` requires the frameworks, runs `Bundler.require` and defines settings; `config/environment.rb` calls `Rails.application.initialize!`, which runs Railtie initializers, the environment file, `config/initializers/*` alphabetically and `after_initialize` hooks, then eager loads in production. `config.ru` runs `Rails.application` for the Rack server.
*Follow-up:* Why is my initializer change ignored? Initializers run once at boot; restart the server.

**Q6. How does autoloading work, and what happens when a file name does not match its class?**
Zeitwerk maps each path under the autoload roots to a constant (`app/models/order_item.rb` -> `OrderItem`, subfolders become namespaces) and loads it on first reference in development. A mismatch raises `Zeitwerk::NameError` when the constant is used, and production boot fails because it eager loads everything.
*Follow-up:* Which constant does `app/services/billing/invoice_pdf.rb` define? `Billing::InvoicePdf`.

**Q7. Why does production eager load the code?**
Loading everything at boot avoids autoloading while several threads serve requests, lets forked Puma workers share memory pages, and surfaces missing constants at deploy time instead of in a user's request.
*Follow-up:* How do you catch eager-load errors earlier? `bin/rails zeitwerk:check` in CI, or eager load in the CI test run (new apps set `config.eager_load = ENV["CI"].present?` in `test.rb`).

**Q8. Puma threads vs workers: what is the difference, and how do they affect the database pool?**
Threads share one process and suit IO-bound requests; the GVL means only one runs Ruby at a time. Workers are forked processes that use multiple cores at the cost of RAM. Each thread may hold a DB connection, so the pool must be at least threads per process, and total connections across servers, workers and Sidekiq must fit under the database limit.
*Follow-up:* What happens if the pool is too small? Threads wait and then raise `ActiveRecord::ConnectionTimeoutError`.

**Q9. How do you manage secrets in a Rails app?**
Encrypted credentials (`bin/rails credentials:edit`, optionally per environment) with the key in `config/master.key` locally and `RAILS_MASTER_KEY` in production, or ENV variables injected by the platform. Never commit keys or plain-text secrets.
*Follow-up:* What is `secret_key_base` for? Signing and encrypting cookies, sessions and other signed messages; changing it invalidates them.

**Q10. What is the difference between Gemfile and Gemfile.lock, and what does `~>` mean?**
The Gemfile lists dependencies with constraints; the lock file records the exact resolved versions of everything, so all environments match. `~> 7.1.0` allows 7.1.x only, `~> 7.1` allows anything below 8.0.
*Follow-up:* `bundle install` vs `bundle update`? Install respects the lock file; update re-resolves (ideally one named gem at a time).

**Q11. What is the difference between `blank?`, `present?` and `presence`? And `try` vs `&.`?**
`blank?` is true for nil, false, empty strings and collections, and whitespace-only strings; `present?` is its negation; `presence` returns the object if present, otherwise nil, which makes `params[:q].presence || "all"` possible. `try` returns nil for a nil receiver and also when the method is missing; `&.` only guards against nil.
*Follow-up:* Is `0.blank?` true? No, numbers are never blank.

**Q12. Why prefer `Time.current` over `Time.now`?**
`Time.current` uses the app's configured `config.time_zone`, while `Time.now` uses the server's system zone, so mixing them causes off-by-hours bugs. Rails stores times in UTC and converts on read.
*Follow-up:* And for dates? `Date.current` instead of `Date.today`.

**Q13. How do the development and production environments differ?**
Development reloads code, loads lazily and shows detailed error pages; production eager loads, caches, forces SSL, logs less and shows generic error pages. Settings live in `config/environments/*.rb` and override `config/application.rb`.
*Follow-up:* How do you add a custom setting per environment? `config.x.*` or `config_for(:file)` with a YAML block per environment.

**Q14. What are engines and Railties?**
A Railtie lets a gem or framework hook into boot (initializers, config, rake tasks). An engine is a Railtie with its own app code, routes and migrations, mountable inside a host app; Devise and Active Storage are engines, and `Rails::Application` itself inherits from `Rails::Engine`.
*Follow-up:* Why use an in-repo engine? To give a large monolith's domain a clear boundary without extracting a separate service.

**Q15. When would you build an app with `--api`, and what changes?**
For a JSON-only backend serving an SPA or mobile app. Controllers inherit `ActionController::API`, the middleware drops cookies, session and flash, and generators skip views and assets.
*Follow-up:* Can a full app also serve an API? Yes: a namespaced `Api::V1` set of controllers that inherit `ActionController::API`.

**Q16. How do you inspect or fix data safely in a production console?**
Start read-only where possible, use `--sandbox` to experiment, wrap writes in a transaction, test the exact code on staging first, and for anything repeatable write a reviewed, idempotent rake task or job instead of ad hoc console code.
*Follow-up:* Why not `--sandbox` for real fixes? Everything rolls back on exit, and a long open transaction can hold locks on production tables.

## Further reading
- [Rails on Rack](https://guides.rubyonrails.org/rails_on_rack.html) and [The Rails Initialization Process](https://guides.rubyonrails.org/initialization.html)
- [Autoloading and Reloading Constants](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
- [Configuring Rails Applications](https://guides.rubyonrails.org/configuring.html) and [The Rails Command Line](https://guides.rubyonrails.org/command_line.html)
