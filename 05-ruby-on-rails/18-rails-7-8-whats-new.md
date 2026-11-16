# Rails 7 and 8: What's New

> **Tier:** T1 (know cold) · **Related:** [Rails fundamentals](01-rails-fundamentals.md), [Hotwire](08-hotwire-turbo-stimulus.md), [Background jobs](09-background-jobs-sidekiq.md), [Authentication and authorization](11-authentication-authorization.md), [Caching, performance and security](14-caching-performance-security.md), [Legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md), [Docker](../14-devops-and-tools/03-docker.md), [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md)
> **On your resume:** the Colochousing Rails 7 web app and API, legacy monoliths on older Rails versions at Devntech, and Hotwire from the Visnext training.

## Quick revision

| Version | Released | Headlines |
|---|---|---|
| 6.0 | Aug 2019 | Zeitwerk, multiple databases, Action Text, Action Mailbox, parallel tests, `insert_all`/`upsert_all` |
| 6.1 | Dec 2020 | Horizontal sharding, `strict_loading`, `delegated_type`, `destroy_async`, signed IDs, `where.missing` |
| 7.0 | Dec 2021 | Hotwire and import maps by default, Webpacker retired, Active Record encryption, `load_async`, Zeitwerk only |
| 7.1 | Oct 2023 | Default Dockerfile, `normalizes`, `generates_token_for`, `authenticate_by`, composite primary keys, `/up` health check |
| 7.2 | Aug 2024 | `allow_browser`, default CI with RuboCop and Brakeman, dev containers, YJIT on by default, controller `rate_limit` |
| 8.0 | Nov 2024 | Solid Queue, Solid Cache, Solid Cable, Kamal 2 with Thruster, Propshaft default, authentication generator, `params.expect` |
| 8.1 | Oct 2025 | Active Job continuations, structured events (`Rails.event`) |

- The Rails 7 theme was **"no build step required"**: HTML over the wire (Turbo, Stimulus) and import maps instead of webpack and Node.
- The Rails 8 theme was **"no PaaS required"**: the Solid trio puts jobs, cache and WebSocket pub/sub in the database, and Kamal 2 deploys containers to any server.
- Solid Queue (`FOR UPDATE SKIP LOCKED` polling) replaces Redis-backed job queues for many apps; Sidekiq is still faster at high volume and offers Pro/Enterprise features.
- Kamal 2 deploys Docker images over SSH with kamal-proxy for zero-downtime switches and automatic TLS; Thruster sits in front of Puma (HTTP/2, compression, asset caching, X-Sendfile).
- New defaults apply to new apps. Upgrading an existing app does not force Solid Queue, Propshaft or Kamal on you.
- Anything after 8.1: say "in recent versions" and check the release notes.

## 1. Why interviewers ask about versions

"What's new in Rails?" tests whether you follow the ecosystem, understand why defaults changed, and can judge what to adopt in an existing app. A good answer names a few changes per release, explains the motivation (simpler stack, security by default, cheaper deployment) and connects to work you have done. Many companies run older versions, so knowing 6.x matters as much as knowing 8.x.

## 2. Rails 6.x: the context legacy apps still run on

**6.0 (Aug 2019)**
- **Zeitwerk** autoloader: file paths map to constants (see [Rails fundamentals](01-rails-fundamentals.md)).
- **Multiple databases:** `connects_to database: { writing: :primary, reading: :primary_replica }` and `connected_to(role: :reading) { ... }`.
- **Action Text** (rich text with Trix) and **Action Mailbox** (inbound email routed to mailboxes).
- Parallel testing, `insert_all` / `upsert_all`, per-environment credentials, host authorization (`config.hosts`), Webpacker as the default JavaScript setup.

**6.1 (Dec 2020)**
- Horizontal **sharding** (`connected_to(role: :writing, shard: :eu) { ... }`), `strict_loading` to catch N+1s, `delegated_type`, `dependent: :destroy_async`.
- **Signed IDs** (`record.signed_id(expires_in: 15.minutes)`, `Model.find_signed`), `where.missing`.
- Active Storage proxy mode and public services; `SameSite=Lax` cookies and non-remote `form_with` by default.

*Why it matters in interviews:* legacy apps you join are often on 6.x or earlier, and the path to 7.x starts with Zeitwerk and these defaults.

## 3. Rails 7.0 (Dec 2021)

- **Hotwire by default:** Turbo (Drive, Frames, Streams) and Stimulus replace Rails UJS and Turbolinks (see [Hotwire](08-hotwire-turbo-stimulus.md)).
- **Import maps by default:** the browser loads ES modules directly, so a new app needs no Node or bundler. Need JSX or TypeScript? `rails new -j esbuild --css tailwind` uses jsbundling-rails and cssbundling-rails. **Webpacker was retired** (Shakapacker continues it for React on Rails).
- **Active Record encryption** for sensitive columns.
- **`load_async`** runs independent queries in parallel; `where.associated`, `in_order_of`, `excluding` and `invert_where` joined the query API.
- **Zeitwerk only:** the classic autoloader was removed.
- **`Rails.error`** reporter, query log tags, open-redirect protection (`raise_on_open_redirects`), libvips as the default Active Storage variant processor. Requires Ruby 2.7+.

```ruby
class Resident < ApplicationRecord
  encrypts :iban                          # encrypted at rest, decrypted transparently (keys via bin/rails db:encryption:init)
  encrypts :email, deterministic: true    # same input, same ciphertext, so where(email: ...) still works
end

Order.where.associated(:payments)         # orders with at least one payment
Task.in_order_of(:status, %w[todo doing done])
Rails.error.handle(Timeout::Error) { PriceClient.new.fetch(id) }   # reports to subscribers, returns nil on error
```

*Why it matters in interviews:* it explains why new Rails apps look different from legacy ones (no webpack, Turbo instead of jQuery UJS) and gives you encryption and `load_async` as concrete features.

### Going deeper: moving a legacy app from Rails UJS to Turbo

| Rails UJS and Turbolinks (6.x and earlier) | Turbo (7.0+) |
|---|---|
| `link_to "Delete", item, method: :delete, data: { confirm: "Sure?" }` | `button_to "Delete", item, method: :delete, form: { data: { turbo_confirm: "Sure?" } }`, or a link with `data: { turbo_method: :delete, turbo_confirm: "Sure?" }` |
| `remote: true` forms answered with `.js.erb` templates | Turbo Drive submits every form; answer with a redirect, a 422 render or Turbo Streams |
| `turbolinks:load` listeners | `turbo:load`, or better, Stimulus `connect()` |
| `ajax:success` / `ajax:error` handlers | `turbo:submit-end` events or Stimulus actions |

The most common breakage after the switch: failed forms returning 200 (Turbo ignores them; return 422) and deletes that are plain GET links (see [routing and controllers](02-routing-and-controllers.md)).

## 4. Rails 7.1 (Oct 2023)

- **Default Dockerfile** (plus `.dockerignore` and `bin/docker-entrypoint`) tuned for production, and a **health check** route at `/up` (`rails/health#show`) for load balancers and deploy tools.
- **Authentication building blocks:** `normalizes`, `generates_token_for`, `authenticate_by`, and `password_challenge` for has_secure_password.
- **Composite primary keys** for tables keyed by several columns.
- **Async calculations:** `async_count`, `async_sum`, `async_pluck` and friends return promises.
- **`ActiveJob.perform_all_later`** enqueues many jobs in one call (one round trip with supporting adapters).
- **Strict locals** declare a partial's arguments; `config.autoload_lib`; `Rails.env.local?`; Bun support; the Trilogy MySQL adapter.

```ruby
class User < ApplicationRecord
  has_secure_password
  normalizes :email, with: ->(email) { email.strip.downcase }       # applied on assignment and in queries
  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)                                          # token dies once the password changes
  end
end

User.authenticate_by(email: params[:email], password: params[:password])   # timing-safe
ActiveJob.perform_all_later(residents.map { |r| RentReminderJob.new(r.id) })
```

```erb
<%# app/views/listings/_card.html.erb: rendering without listing: raises instead of failing silently %>
<%# locals: (listing:, show_price: true) -%>
```

*Why it matters in interviews:* 7.1 is where Rails started shipping deployment (Dockerfile, `/up`) and the pieces later used by the 8.0 authentication generator.

## 5. Rails 7.2 (Aug 2024)

- **`allow_browser versions: :modern`** in new apps' `ApplicationController`: browsers without modern features get a 406 page.
- **Default CI and tooling:** a GitHub Actions workflow, RuboCop with the rubocop-rails-omakase style, `bin/brakeman` security scanning, and optional dev containers.
- **YJIT on by default** when running on Ruby 3.3+, and fewer default Puma threads (3 instead of 5) for better latency.
- **Controller `rate_limit`** backed by the cache store; generated PWA manifest and service worker templates.
- **Transaction-aware job enqueueing:** jobs enqueued inside a transaction can wait for the commit; the default for this changed again in 8.x, so check your version.
- Requires Ruby 3.1+.

```ruby
class ApplicationController < ActionController::Base
  allow_browser versions: :modern
end

class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create   # 429 by default when exceeded
end
```

*Why it matters in interviews:* it shows Rails moving security and quality checks into the defaults (Brakeman, RuboCop, rate limiting) rather than leaving them to each team.

## 6. Rails 8.0 (Nov 2024)

- **Solid Queue, Solid Cache and Solid Cable** become the defaults for Active Job, the cache store and Action Cable, each backed by a database (section 9).
- **Kamal 2 and Thruster** are configured in new apps for deploying to your own servers (section 10).
- **Propshaft** replaces Sprockets as the default asset pipeline (section 11).
- **Authentication generator:** `bin/rails generate authentication` (sessions and password resets; no sign-up).
- **`params.expect`** for safer strong parameters.
- SQLite tuned to be production-ready for small apps; `bin/jobs` to run Solid Queue, or the Puma plugin to run it inside the web server. Requires Ruby 3.2+.

```yaml
# config/database.yml (Rails 8 production, Postgres flavor): one database per role
production:
  primary:
    <<: *default
    database: myapp_production
  cache:
    <<: *default
    database: myapp_production_cache
    migrations_paths: db/cache_migrate
  queue:
    <<: *default
    database: myapp_production_queue
    migrations_paths: db/queue_migrate
  cable:
    <<: *default
    database: myapp_production_cable
    migrations_paths: db/cable_migrate
```

*Why it matters in interviews:* 8.0 is the release people ask about most: it changes how a new Rails app is hosted and which services it needs.

## 7. Rails 8.1 (Oct 2025)

- **Active Job continuations** let a long job be split into steps with a cursor, so after a deploy restarts the worker it resumes where it stopped instead of starting over. The API shape from the release looks like this (check the guide for details):

```ruby
class ImportCatalogJob < ApplicationJob
  include ActiveJob::Continuable

  def perform(import_id)
    import = Import.find(import_id)

    step :fetch_rows do |step|
      import.rows.find_each(start: step.cursor) do |row|
        row.process!
        step.advance! from: row.id                  # saved progress: resume after this row
      end
    end

    step :finalize do
      import.finalize!
    end
  end
end
```

- **Structured event reporting:** `Rails.event.notify("checkout.completed", order_id: order.id)` emits machine-readable events to subscribers, aimed at observability pipelines.
- The release notes list more, such as a local CI runner, Markdown rendering and deprecated associations; read them before relying on specifics.

*Why it matters in interviews:* continuations solve a real Sidekiq-era pain (long imports killed by deploys), which connects to catalog syncs like PriceCharting.

## 8. After 8.1

Rails keeps shipping minor releases. For anything newer than 8.1, say "in recent versions" and check the official blog and release notes rather than guessing; interviewers respect "I have not used that release yet, but here is how I would evaluate it".

## 9. The Solid trio and "no Redis by default"

| | Solid Queue | Solid Cache | Solid Cable |
|---|---|---|---|
| Replaces | Redis-backed queues (Sidekiq, Resque) | Redis or Memcached cache | Redis pub/sub for Action Cable |
| Storage | Job tables in a queue database | Cache entries in a cache database, on disk | Message table in a cable database |
| How | Workers poll with `FOR UPDATE SKIP LOCKED`; dispatchers handle scheduled jobs; recurring tasks in `config/recurring.yml` | Large, disk-sized cache with age and size limits | Servers poll for new messages and trim old ones |
| Trade-off | More database load; lower peak throughput than Sidekiq | Slightly slower reads than RAM, but far bigger and cheaper | Polling latency; database load on busy apps |

Why the defaults changed: one fewer service to provision, monitor, secure and back up; modern SSDs make database-backed queues and caches fast enough for most apps; it fits the "deploy anywhere without a PaaS" goal. It is not a verdict against Redis: Sidekiq with Redis remains a great choice for high job volume, existing infrastructure or Pro/Enterprise features (batches, unique jobs, rate limiting). The Sidekiq vs Solid Queue comparison is in [background jobs](09-background-jobs-sidekiq.md).

Solid Queue extras worth naming: concurrency limits per job (`limits_concurrency to: 1, key: ->(store_id) { store_id }`), queue pausing, recurring tasks, and Mission Control Jobs as a web UI.

```ruby
# config/puma.rb (Rails 8): small deployments can run the Solid Queue supervisor inside Puma
plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"]
# Larger ones run `bin/jobs` as its own process (for example a separate Kamal "job" role)
```

## 10. Kamal 2 and Thruster

Kamal deploys Docker images to any servers you control (cloud VMs or bare metal) over SSH. Kamal 2 replaced Traefik with its own **kamal-proxy**, which switches traffic to the new container only after its health check passes (zero downtime) and provisions Let's Encrypt certificates automatically. Images travel through a container registry; Kamal 2.8 (shipped alongside Rails 8.1) can use a local registry by default, so a remote one such as Docker Hub or GHCR becomes optional for basic deploys.

```yaml
# config/deploy.yml (trimmed)
service: myapp
image: myorg/myapp
servers:
  web:
    - 192.0.2.10
  job:
    hosts:
      - 192.0.2.11
    cmd: bin/jobs
proxy:
  ssl: true
  host: app.example.com
registry:                            # remote registry (optional for basic deploys in recent Kamal)
  username: myorg
  password:
    - KAMAL_REGISTRY_PASSWORD        # read from .kamal/secrets
env:
  secret:
    - RAILS_MASTER_KEY
```

```bash
kamal setup        # first deploy: installs Docker on the hosts, boots accessories, deploys
kamal deploy       # build, push, pull on servers, health check, switch traffic
kamal rollback <version>
kamal app logs
```

**Accessories** run supporting containers such as a database or Redis on your hosts. Compared with Heroku, Render or Fly, Kamal gives you cheap servers and full control in exchange for owning operations (patching, backups, monitoring); compared with Capistrano, it ships immutable container images instead of updating code in place.

**Thruster** is a small proxy that runs inside the app container in front of Puma (the default Dockerfile starts `./bin/thrust ./bin/rails server`). It adds HTTP/2, gzip compression, caching of static assets and X-Sendfile acceleration, so a single container can serve production traffic without a separate Nginx.

## 11. Propshaft and the asset story

Propshaft is a deliberately small pipeline: it adds digests to file names and resolves load paths, and leaves transpiling and bundling to import maps, jsbundling or cssbundling. Sprockets still runs countless legacy apps, and React on Rails apps use Shakapacker. The full history (Sprockets, Webpacker, import maps, jsbundling, Propshaft) is in [views and assets](07-views-erb-forms-assets.md).

## 12. The authentication generator and `params.expect`

- `bin/rails generate authentication` (8.0) writes a User model with `has_secure_password`, database-backed sessions, a `Current` object, an `Authentication` concern, and password reset by email, as readable code in your app. It deliberately leaves out sign-up, confirmation, OAuth and roles (details in [authentication and authorization](11-authentication-authorization.md)).
- `params.expect(product: [:name, :price])` (8.0) combines `require` and `permit`, checks that the shape is right (a hash where a hash is expected, double brackets for arrays of hashes) and responds with 400 instead of 500 when a client sends tampered structures (details in [routing and controllers](02-routing-and-controllers.md)).

## 13. How to answer "What's new in Rails?"

**30-second version:**
"Rails 7 made the frontend simpler: Hotwire and import maps by default, so many apps need no Node build, plus encrypted attributes and `load_async`. 7.1 and 7.2 added production defaults like a Dockerfile, a health check, `normalizes` and `generates_token_for`, and CI with Brakeman and RuboCop. Rails 8 aims at deploying without a PaaS: Solid Queue, Solid Cache and Solid Cable move jobs, cache and pub/sub into the database, Kamal 2 with Thruster deploys containers to any server, Propshaft is the default asset pipeline, and there is a built-in authentication generator. 8.1 added resumable jobs with continuations."

**Two-minute version, organized by theme:**
1. *Frontend:* Hotwire and import maps (7.0), Turbo 8 morphing, Propshaft (8.0); React still fits through jsbundling or React on Rails.
2. *Data layer:* encryption, `load_async` (7.0), composite primary keys and async calculations (7.1), `normalizes`.
3. *Security by default:* `generates_token_for` and `authenticate_by` (7.1), `allow_browser`, `rate_limit` and Brakeman in CI (7.2), the authentication generator and `params.expect` (8.0).
4. *Operations:* Dockerfile and `/up` (7.1), YJIT by default (7.2), Solid trio, Kamal 2 and Thruster (8.0), job continuations (8.1).
5. *Your judgment:* "In an existing Rails 7 app with Sidekiq and Redis, I would upgrade to 8.x for support and features like `params.expect`, but keep Sidekiq unless there is a reason to change; for a new small app I would happily start with the Solid defaults."

> **Fill in:** which Rails version Colochousing and the other Devntech apps run, and which 7.x features you have actually used (Hotwire, encryption, `normalizes`, `load_async`).

## Common pitfalls
- Claiming Rails 8 "removed Redis" or "replaced Sidekiq": it changed defaults for new apps; nothing forces existing apps to switch.
- Mixing up versions: the authentication generator and `params.expect` are 8.0; `rate_limit` and `allow_browser` are 7.2; `normalizes` and `generates_token_for` are 7.1.
- Saying the 8.0 generator includes sign-up; it does not.
- Describing Webpacker as current; it is retired, and Shakapacker is its successor.
- Saying you have deployed with Kamal or run Solid Queue in production when you have only read about them.
- Stating details of releases after 8.1 without checking the release notes.

## How to talk about this in an interview
- Anchor on what you know first-hand: the Colochousing app is Rails 7 with React on Rails, ERB and Sidekiq, and you trained on Hotwire at Visnext. Then show awareness of 8.x and how you would evaluate it.
- Present adoption as judgment: Solid Queue for a new low-volume app, Sidekiq for high volume or Pro features; Kamal when the team wants control and low hosting cost, a PaaS when it wants zero ops.
- Tie upgrades to the maintenance policy and to your legacy-monolith experience (see [legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md)).

> **Fill in:** any Rails 8 feature you tried in a side project or tutorial (generator, Solid Queue, Kamal), so you can speak from experience.

## Interview questions

**Q1. What is new in Rails 7?**
7.0 made Hotwire (Turbo and Stimulus) and import maps the defaults, retired Webpacker, added Active Record encryption and `load_async`, and dropped the classic autoloader. 7.1 added a production Dockerfile, the `/up` health check, `normalizes`, `generates_token_for`, `authenticate_by` and composite primary keys; 7.2 added `allow_browser`, default CI with RuboCop and Brakeman, YJIT by default and controller `rate_limit`.
*Follow-up:* Why import maps? Modern browsers support ES modules and HTTP/2, so many apps can skip a Node build entirely.

**Q2. What is new in Rails 8?**
The Solid trio (Solid Queue, Solid Cache, Solid Cable) as database-backed defaults for jobs, caching and Action Cable; Kamal 2 with Thruster for deploying containers to any server; Propshaft as the default asset pipeline; the authentication generator; and `params.expect`. 8.1 added Active Job continuations and structured events.
*Follow-up:* What is the theme? Fewer moving parts and no PaaS required.

**Q3. What does "no Redis by default" mean, and would you remove Redis from an existing app?**
New Rails 8 apps run jobs, cache and Action Cable on the database, so they need no Redis server. For an existing app with Sidekiq and Redis working well, I would not switch just for the default; I would consider it only to simplify infrastructure when job volume is modest and no Sidekiq Pro features are needed.
*Follow-up:* What would you measure first? Job throughput and latency, and the extra load on the database.

**Q4. Solid Queue or Sidekiq?**
Sidekiq keeps jobs in Redis, is very fast, mature, and has paid features like batches and unique jobs; Solid Queue keeps jobs in SQL tables, polls with `FOR UPDATE SKIP LOCKED`, needs no extra service and works through Active Job, with concurrency limits and recurring tasks built in. Sidekiq for high volume or existing Redis; Solid Queue for new apps wanting fewer moving parts.
*Follow-up:* Can Solid Queue run native `Sidekiq::Job` classes? No, only Active Job jobs.

**Q5. What is Solid Cache, and when is it better than Redis?**
A cache store that keeps entries in a database table on disk. Reads are slightly slower than RAM, but the cache can be much larger and longer-lived at low cost, which raises hit rates; it also avoids running a cache server. Redis or Memcached still win when you need the lowest latency.
*Follow-up:* How does it evict? By age and total size limits configured in `config/cache.yml`.

**Q6. What is Solid Cable?**
The Rails 8 default Action Cable adapter: broadcasts are written to a database table that cable servers poll, with old messages trimmed after a retention period. It removes Redis at the cost of polling latency and database load.
*Follow-up:* When keep Redis? Busy realtime apps with many broadcasts per second.

**Q7. What is Kamal?**
A deployment tool from the Rails team that builds a Docker image, ships it through a registry (local by default in recent versions) and runs it on your own servers over SSH. Kamal 2's kamal-proxy switches traffic to new containers after health checks for zero downtime and handles TLS certificates; accessories run databases or Redis next to the app.
*Follow-up:* Kamal vs Heroku? Kamal is cheaper and gives full control, but you own patching, backups and monitoring.

**Q8. What is Thruster?**
A lightweight proxy bundled in Rails 8's Docker setup that runs in front of Puma inside the container, adding HTTP/2, compression, static asset caching and X-Sendfile, so you do not need a separate Nginx for most apps.
*Follow-up:* How is it started? The Dockerfile runs `./bin/thrust ./bin/rails server`.

**Q9. Propshaft vs Sprockets, and where do import maps fit?**
Sprockets compiles, concatenates and fingerprints assets with its own directives; Propshaft only fingerprints and serves files, leaving JavaScript and CSS processing to import maps (no build) or jsbundling and cssbundling (esbuild, Tailwind). Propshaft is the Rails 8 default.
*Follow-up:* Why did Webpacker go away? Rails moved to import maps and jsbundling; ShakaCode continues it as Shakapacker.

**Q10. What did Rails 7.1 add for authentication?**
`normalizes` (clean emails on assignment and in queries), `generates_token_for` (purpose-bound, expiring tokens that can be invalidated by data changes), `authenticate_by` (timing-safe login lookup) and `password_challenge` (require the current password to change it).
*Follow-up:* What uses them? The Rails 8 authentication generator.

**Q11. What do `allow_browser` and `rate_limit` do?**
`allow_browser versions: :modern` (7.2) rejects browsers that lack modern features with a 406 page. `rate_limit to: 10, within: 3.minutes` (7.2) limits requests per client in a controller, stored in the cache store, returning 429 by default.
*Follow-up:* What does `rate_limit` need in production? A cache store shared by all processes.

**Q12. What is `params.expect`?**
A Rails 8 strong-parameters method that requires and permits in one call and verifies the structure, so malformed input (a string instead of a hash, wrong array shapes) yields a 400 instead of a 500. Arrays of hashes use double brackets: `params.expect(items: [[:sku, :quantity]])`.
*Follow-up:* What did you use before? `params.require(:product).permit(:name, :price)`.

**Q13. What are Active Job continuations?**
A Rails 8.1 feature where a job includes `ActiveJob::Continuable` and splits work into steps with a saved cursor; when a worker shuts down mid-job (for example during a deploy), the job resumes from the last step and cursor instead of restarting.
*Follow-up:* What did people do before? Split work into many small jobs or checkpoint progress manually in the database.

**Q14. Which Rails 6 features do legacy apps still depend on?**
Zeitwerk autoloading, multiple databases and read replicas, `insert_all`/`upsert_all`, Action Text, per-environment credentials, and from 6.1 `strict_loading`, `delegated_type`, `destroy_async` and signed IDs.
*Follow-up:* Why does Zeitwerk matter for upgrades? The classic autoloader was removed in 7.0, so apps must switch first.

**Q15. What is Active Record encryption, and what is deterministic encryption?**
Built-in (7.0) attribute encryption at rest declared with `encrypts :iban`; values are encrypted before saving and decrypted on read. Non-deterministic encryption is the default and safest; `deterministic: true` produces the same ciphertext for the same value, so you can query by it, at some cost in secrecy.
*Follow-up:* Where do the keys live? In credentials, generated with `bin/rails db:encryption:init`.

**Q16. Why did Rails 7.1 add a Dockerfile and a `/up` endpoint?**
To make production deployment a default: the Dockerfile builds a slim, production-ready image, and `/up` gives load balancers and deploy tools (like Kamal's proxy) a health check that confirms the app booted.
*Follow-up:* What does `/up` return? 200 when the app boots without exceptions, 500 otherwise.

## Further reading
- [Rails 7.0 release notes](https://guides.rubyonrails.org/7_0_release_notes.html), [7.1](https://guides.rubyonrails.org/7_1_release_notes.html), [7.2](https://guides.rubyonrails.org/7_2_release_notes.html)
- [Rails 8.0 release notes](https://guides.rubyonrails.org/8_0_release_notes.html) and [8.1](https://guides.rubyonrails.org/8_1_release_notes.html)
- [Kamal documentation](https://kamal-deploy.org/)
