# Legacy Rails Codebases and Upgrades

> **Tier:** T1 (know cold) · **Related:** [Rails fundamentals](01-rails-fundamentals.md), [ActiveRecord advanced](06-activerecord-advanced.md), [Architecture patterns](12-architecture-patterns.md), [Testing with RSpec](13-testing-rspec.md), [Rails 7 and 8: what's new](18-rails-7-8-whats-new.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [Git](../14-devops-and-tools/01-git.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md)
> **On your resume:** working across legacy Rails monoliths at Devntech, enhancing CrystalCommerce's legacy e-commerce platform (700+ stores, 8M+ customers), the Colochousing Rails 7 app, and the multi-tenant platform migration.

## Quick revision
- Reading a monolith: README and `bin/setup`, Gemfile and `bin/rails about`, `config/routes.rb`, `db/schema.rb`, models (associations, callbacks, concerns), then follow one request end to end through the log.
- `git log --follow`, `git log -S "method"` (pickaxe) and `git blame -w -C` explain why code exists; tests are executable documentation.
- Safe changes: characterization tests first, small PRs that separate refactors from behavior changes, feature flags (Flipper), the strangler fig for replacing whole areas, and a rollback plan.
- Callback spaghetti: inventory callbacks, pin behavior with tests, keep data-integrity callbacks, move side effects (emails, jobs, API calls) into explicit services, use `after_commit` for anything that must stay.
- Upgrade one minor version at a time: green tests, latest patch, fix deprecations (`config.active_support.deprecation = :raise` in test), unblock gems, bump Rails, `bin/rails app:update`, deploy, then flip new defaults.
- `config.load_defaults 7.1` sets framework defaults per version; after upgrading, keep the old value and enable settings one by one from `new_framework_defaults_7_1.rb`, then bump it.
- Dual boot (next_rails `Gemfile.next`, Shopify's bootboot) runs the suite on the current and next Rails in CI from one branch.
- Ruby upgrades are separate projects: 3.0 separated keyword arguments; 3.4 moved csv, base64 and bigdecimal to bundled gems that need Gemfile entries.
- Zeitwerk: classic autoloader removed in Rails 7.0; `bin/rails zeitwerk:check`, file names must match constants, inflector for acronyms, no `require_dependency`.
- Support: Rails publishes time-based support windows at rubyonrails.org/maintenance; a series past its security date gets no patches, so Rails 7.x apps should be planning 8.x.
- Zero-downtime migrations: expand (add), migrate (backfill in batches), switch reads, contract (`ignored_columns`, then drop). Concurrent indexes, `NOT VALID` constraints, `validate: false` foreign keys, lock timeouts; strong_migrations flags unsafe steps.
- Never `rename_column` or `remove_column` on a live table in one step: old processes still use the old column.
- Data migrations belong in maintenance_tasks, data-migrate or one-off jobs: idempotent, batched, throttled, resumable.
- Monkey patches: find them in initializers and `lib/`, test them, prefer `prepend` + `super`, guard them with version checks; replace abandoned gems with Rails built-ins where possible.

## 1. Reading an unfamiliar monolith

| Step | Look at | What it tells you |
|---|---|---|
| 1 | README, `bin/setup`, `Procfile.dev`, `docker-compose.yml` | How to run it locally, which processes exist (web, Sidekiq, webpack) |
| 2 | `Gemfile`, `Gemfile.lock`, `.ruby-version` | Versions and architecture: Devise, Pundit, Sidekiq, AASM, React on Rails |
| 3 | `config/routes.rb` | The app's surface: every feature starts at a route |
| 4 | `db/schema.rb` (or `structure.sql`) | The real data model: tables, indexes, foreign keys |
| 5 | `app/models` | Associations, callbacks, concerns and scopes, where behavior hides |
| 6 | `config/initializers`, `lib/`, `app/services`, `app/jobs` | Integrations, monkey patches, background work |
| 7 | `spec/` or `test/` | Intended behavior and how to exercise it |
| 8 | Git history | Why things are the way they are |

```bash
bin/rails about                                   # Ruby, Rails, database adapter, environment
bin/rails routes -g rental_application            # routes for one feature
git log --oneline --follow app/models/order.rb    # a file's history, across renames
git log -S "recalculate_totals" --oneline         # commits that added or removed that string
git blame -w -C app/models/order.rb               # line authors, ignoring whitespace and moved code
grep -rn "after_commit\|after_save" app/models    # where side effects live
bin/rails runner 'puts Order._save_callbacks.map(&:filter)'   # callbacks actually registered on save
```

Read the PR or ticket linked from a commit message before "fixing" odd code; a strange condition is often a fix for a production incident. Keep notes of what you learn (a glossary of domain terms, a diagram of key models): onboarding docs are a cheap way to add value in your first weeks.

## 2. Following one request end to end

```text
Started PATCH "/rental_applications/42/approve" for 203.0.113.5 at 2025-03-01 10:15:30 +0100
Processing by RentalApplicationsController#approve as TURBO_STREAM
  Parameters: {"id"=>"42"}
  RentalApplication Load (0.4ms)  SELECT "rental_applications".* FROM "rental_applications" WHERE ...
  TRANSACTION (0.2ms)  BEGIN
  Lease Create (0.9ms)  INSERT INTO "leases" ...
Enqueued ActionMailer::MailDeliveryJob (Job ID: ...) to Sidekiq(default)
Completed 302 Found in 118ms (ActiveRecord: 21.3ms | Allocations: 18234)
```

1. **Route:** `bin/rails routes -g approve` confirms the controller and action.
2. **Controller:** read the inherited before_actions too (ApplicationController, concerns), then the action.
3. **Models:** validations, callbacks and concerns triggered by each save; the log's SQL shows what actually ran.
4. **Side effects:** enqueued jobs, mailers, broadcasts and HTTP calls.
5. **Response:** template, partials or serializer.

Put a `debugger` (the debug gem ships with Rails 7) or `binding.irb` in the action to inspect state, and use `bin/rails console --sandbox` to experiment without keeping changes.

## 3. Making safe changes

- **Characterization tests:** pin current behavior around the code you will touch, right or wrong, then change it (see [testing](13-testing-rspec.md)).
- **Small PRs:** one behavior per PR; refactoring PRs separate from behavior changes so reviewers can verify "no behavior change".
- **Feature flags** decouple deploying from releasing:

```ruby
if Flipper.enabled?(:new_pricing, current_store)          # actor-based flag (Flipper gem)
  Pricing::MarketBased.call(listing)
else
  LegacyPricing.price_for(listing)
end
# Roll out gradually: Flipper.enable_actor(:new_pricing, pilot_store), then
# Flipper.enable_percentage_of_actors(:new_pricing, 25); turning it off is the rollback.
```

- **Strangler fig:** put a seam in front of the old behavior (a route, a service interface), build the new implementation beside it, move traffic over gradually, and delete the old code once nothing calls it. The same idea scales up to moving pages or whole domains to a new platform (see [platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md)).
- **Compare before switching:** add logs or metrics around the area first, or run old and new code side by side and compare results while still returning the old one (GitHub's Scientist gem does this).
- **Rollback plan:** a flag to turn off, a revertable PR, migrations that old code tolerates.

## 4. Untangling callback and concern spaghetti

Symptoms: saving a model sends emails, enqueues jobs, calls APIs and updates other models; code uses `update_column` or `skip_callback` to dodge side effects; imports and console fixes trigger surprises; the order of callbacks matters and nobody knows why.

```ruby
# Before: saving an order recalculates, emails and syncs as side effects
class Order < ApplicationRecord
  after_save :recalculate_totals
  after_save :send_confirmation, if: :saved_change_to_status?
  after_save :sync_to_marketplace
end

# After: data integrity stays on the model, side effects become an explicit use case
class Order < ApplicationRecord
  before_save :recalculate_totals                       # derived data on the record itself
end

class Orders::Confirm
  def self.call(order)
    order.update!(status: :confirmed)
    OrderMailer.with(order: order).confirmation.deliver_later
    MarketplaceSyncJob.perform_later(order.id)
  end
end
```

Steps: list every callback and concern on the model; write request or service specs at the entry points that depend on them; classify each callback as integrity (normalization, defaults, derived columns, `touch`) or side effect; move side effects into explicit services called from controllers and jobs, one at a time; switch any side effect that must stay a callback to `after_commit`; delete the `skip_callback` and `update_column` workarounds that are no longer needed. A concern included by one model is usually better inlined or turned into a plain object (see [architecture patterns](12-architecture-patterns.md)).

## 5. Upgrading Rails one minor version at a time

```bash
# 1. Start green: tests pass on the latest patch of your current minor (e.g. the newest 7.0.x)
bundle update rails --conservative
# 2. Fix deprecation warnings; they list what breaks in the next version.
#    config/environments/test.rb: config.active_support.deprecation = :raise
# 3. Unblock gems: bundle outdated; upgrade or replace gems that pin the old Rails
# 4. Bump one minor version in the Gemfile: gem "rails", "~> 7.1.0"
bundle update rails
bin/rails app:update          # interactive diff of config files; writes new_framework_defaults_7_1.rb
# 5. Run the suite, fix failures, deploy to staging, watch error tracking, deploy to production
# 6. Enable the new framework defaults one by one, then bump config.load_defaults
```

Why one minor at a time: each upgrade guide and deprecation cycle assumes the previous version (deprecated in one release, removed in the next), and small jumps make it obvious which change broke what. Read the official upgrade guide and release notes for each step, and expect gem upgrades to be most of the work.

## 6. load_defaults and the new framework defaults

```ruby
# config/application.rb
config.load_defaults 7.0        # leave the old value right after upgrading the gem to 7.1

# config/initializers/new_framework_defaults_7_1.rb (generated by app:update, everything commented out)
# Rails.application.config.active_support.cache_format_version = 7.1
```

`load_defaults` switches on the behavior new apps of that version get (cookie and cache formats, CSRF and redirect protections, Active Record behaviors). Upgrading the gem does not flip them, so you can enable each setting in the generated file, run tests, deploy and watch, and only when all are on set `config.load_defaults 7.1` and delete the file. Some settings need two deploys because old and new servers coexist during a rolling deploy: for format changes (cache entries, cookies, message serialization), first deploy code that can read both formats, and switch writing only when no server or rollback still needs the old one. The generated file's comments say which settings carry that warning.

## 7. Dual booting

```ruby
# Gemfile (`next_rails --init` creates Gemfile.next and adds a next? helper; its exact code varies by version)
def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem "rails", "~> 8.0.0"
else
  gem "rails", "~> 7.2.0"
end
```

```bash
next bundle install          # resolves Gemfile.next.lock (same as BUNDLE_GEMFILE=Gemfile.next bundle install)
next bundle exec rspec       # the same suite on the next Rails version
```

CI runs both lockfiles; code that must differ checks `NextRails.next?`; production switches when the next build is green and stable, then the old branch of the Gemfile is deleted. Shopify's bootboot achieves the same with a second lockfile selected by an environment variable. Dual booting lets a long upgrade progress in small PRs on the main branch instead of a months-long upgrade branch.

## 8. Gems and Ruby versions

- **Gems first:** a gem that requires `rails < 7.1` blocks `bundle update`. Check its changelog, upgrade it, replace it, or fork it. RailsBump-style compatibility checks and `bundle outdated` show the work.
- **Ruby separately from Rails:** each Rails release supports a Ruby range (for example Rails 7.2 requires Ruby 3.1+ and Rails 8.0 requires 3.2+), so you sometimes upgrade Ruby first. Change one variable at a time.
- **Ruby 2.7 to 3.0:** keyword arguments are separated from positional hashes, the biggest break for old code; 2.7's warnings point at the call sites.
- **Ruby 3.4:** csv, base64, bigdecimal and a few others became bundled gems (add them to the Gemfile if you use them); mutating string literals emits "chilled string" warnings; Prism became the default parser. Ruby's December 2025 release shipped as 4.0; read its release notes before planning that jump.
- Update `.ruby-version`, the Dockerfile, CI images and the production runtime together, and enable YJIT once on a supported Ruby.

## 9. Moving from the classic autoloader to Zeitwerk

The classic autoloader (based on `const_missing`) was replaced by Zeitwerk in Rails 6.0 and removed in 7.0, so every legacy app passing 6.1 must switch (basics in [Rails fundamentals](01-rails-fundamentals.md)).

```ruby
# Rails 6.x: config.autoloader = :zeitwerk, then run bin/rails zeitwerk:check
# config/initializers/zeitwerk.rb: teach the inflector your acronyms
Rails.autoloaders.main.inflector.inflect("html_parser" => "HTMLParser", "csv_export" => "CSVExport")
```

- Every file must define the constant its path implies: `app/services/billing/pdf_invoice.rb` defines `Billing::PdfInvoice` (or `PDFInvoice` with an inflection).
- Remove `require_dependency` and `require` calls for app code.
- Initializers must not keep references to reloadable classes; wrap such code in `Rails.application.config.to_prepare { ... }`.
- `lib/` is not autoloaded unless configured (Rails 7.1 added `config.autoload_lib(ignore: %w[assets tasks])`).

## 10. Support status and planning the move to 8.x

Rails documents which release series get bug and security fixes on its maintenance policy page, with exact end dates. The published policy uses fixed windows: bug fixes for one year and security fixes for two years after the first release of each minor series. Rails 7.2 shipped in August 2024, so by that rule the whole 7.x line loses security support once two years have passed, and even 8.0 (November 2024) has a short window; a Rails 7.x app should be planning its move to a supported 8.x series. Check the page before quoting dates.

Why it matters: unsupported versions get no patches for new vulnerabilities (a compliance problem for e-commerce and payments), gems drop support, and each skipped year makes the eventual upgrade bigger. Upgrading to 8.x does not force the new defaults on an existing app: Sidekiq, Redis, Sprockets or Shakapacker and Capistrano keep working; Solid Queue, Propshaft and Kamal are defaults for new apps that you can adopt later (see [Rails 7 and 8](18-rails-7-8-whats-new.md)).

## 11. Zero-downtime migrations

During a deploy, old code keeps running against the new schema for a while, and Active Record caches each table's columns when a process boots. So a migration must be compatible with both the old and the new code, and must not hold locks that block traffic.

| Phase | Deploy | Action |
|---|---|---|
| Expand | 1 | Add the new column or table (nullable); deploy code that writes to old and new |
| Migrate | 2 | Backfill existing rows in batches, outside the schema migration |
| Switch | 3 | Read from the new column; keep dual writes until verified |
| Contract | 4 | Stop writing the old column, add it to `ignored_columns`, deploy; drop it in a later deploy |

```ruby
class Listing < ApplicationRecord
  self.ignored_columns += ["legacy_sku"]       # deploy this first; remove the column in a later migration
end

class AddEmailNotNullToResidents < ActiveRecord::Migration[7.1]
  def change                                   # NOT NULL on a big table without a long lock
    add_check_constraint :residents, "email IS NOT NULL", name: "residents_email_null", validate: false
  end
end
# Next migration: validate_check_constraint :residents, name: "residents_email_null"
# Then (Postgres 12+ reuses the validated constraint, no full scan): change_column_null :residents, :email, false
```

Recipes:
- **Add a column with a default:** Postgres 11+ adds a constant default without rewriting the table; volatile defaults (`gen_random_uuid()`, `clock_timestamp()`) still rewrite. On older setups add it nullable, backfill, then set the default.
- **Indexes:** `algorithm: :concurrently` with `disable_ddl_transaction!` (see [ActiveRecord advanced](06-activerecord-advanced.md)).
- **Foreign keys:** `add_foreign_key :orders, :stores, validate: false`, then `validate_foreign_key :orders, :stores` in a separate migration.
- **Rename a column or change its type:** expand/contract with a new column, never `rename_column` on a live table.
- **Remove a column:** `ignored_columns` and deploy first, then `remove_column`.
- **Lock timeouts:** a migration waiting for a lock makes every query queue behind it; set `lock_timeout` and `statement_timeout` for migrations (strong_migrations can configure them) so a blocked ALTER fails fast and can be retried.
- The **strong_migrations** gem raises on unsafe operations during `db:migrate` and prints the safe recipe, which teaches the whole team these rules.

## 12. Data migrations vs schema migrations

| | Schema migration | Data migration |
|---|---|---|
| Changes | Structure: tables, columns, indexes, constraints | Rows: backfills, fixes, splitting or merging values |
| Duration | Seconds | Minutes to hours on big tables |
| Runs | On every deploy and in every environment; rebuildable from `schema.rb` | Once per environment, often only in production |
| Must be | Fast, reversible, compatible with old code | Idempotent, batched, throttled, resumable |

Putting backfills in schema migrations holds long transactions and locks, blocks the deploy, fails halfway with no resume, and breaks `db:migrate` from scratch when the models it used change later. Use one of:

```ruby
# maintenance_tasks (Shopify): bin/rails generate maintenance_tasks:task backfill_listing_currency
module Maintenance
  class BackfillListingCurrencyTask < MaintenanceTasks::Task
    def collection
      Listing.where(currency: nil)                       # iterated with a cursor that survives restarts
    end

    def process(listing)
      listing.update!(currency: listing.store.default_currency)
    end
  end
end
```

maintenance_tasks runs tasks as Active Job jobs with a web UI showing progress, and they can be paused, resumed and throttled. The data-migrate gem instead keeps data migrations in `db/data` with their own version table (`bin/rails data:migrate`). For simple cases, a batched job works:

```ruby
Listing.where(currency: nil).in_batches(of: 1_000) do |batch|
  batch.update_all(currency: "USD", updated_at: Time.current)   # set updated_at so sync consumers notice
  sleep 0.1                                                      # spare the primary and the replicas
end
```

## 13. Performance archaeology

Legacy apps accumulate slowness quietly. Collect evidence before optimizing: APM endpoints by total time, pg_stat_statements' top queries, Sidekiq queue latency, memory graphs per worker. Typical findings:
- N+1 queries in views and serializers that grew with the data (Bullet or `strict_loading` in development; see [querying and N+1](05-activerecord-querying-n-plus-1.md)).
- Missing indexes on foreign keys and on columns used by frequent filters and sorts (`EXPLAIN ANALYZE` shows sequential scans).
- `default_scope` with ordering or joins applied to every query; callbacks doing heavy work on every save.
- Jobs and rake tasks iterating `Model.all` instead of `find_each`, and exports built in memory.
- Expensive pages that could be cached (see [caching and performance](14-caching-performance-security.md)).

Fix one thing, measure again, and write down the before and after numbers.

## 14. Old gems and monkey patches

```bash
grep -rn "class_eval\|prepend\|alias_method\|module_eval" config lib app   # where patches usually live
```

```ruby
# config/initializers/patches/price_client_timeout.rb
if Gem::Version.new(PriceClient::VERSION) >= Gem::Version.new("2.4")
  raise "Remove this patch: timeouts are configurable upstream since price_client 2.4"
end

module PriceClientTimeoutPatch
  def request(path, **options)
    super(path, **options.merge(timeout: 10))     # prepend + super keeps the original reachable
  end
end
PriceClient::Connection.prepend(PriceClientTimeoutPatch)
```

- Give every patch a test for the behavior it changes and a guard that fails loudly when the gem upgrades.
- Prefer `Module#prepend` with `super` over redefining methods; Rails' old `alias_method_chain` helper was deprecated in 5.0 and removed in 5.1. Refinements limit a patch to files that opt in (see [Ruby metaprogramming](../03-languages/ruby/05-metaprogramming.md)).
- **Unmaintained gems:** check the last release, open issues and Rails support. Replace with Rails built-ins where possible (Paperclip to Active Storage, attr_encrypted to `encrypts`, protected_attributes to strong parameters), upgrade, or fork and pin (`gem "foo", github: "your-org/foo", branch: "rails-8"`) while you plan a replacement.

## Common pitfalls
- "Big bang" rewrites or upgrade branches that live for months and never merge.
- Skipping minor versions, or upgrading Ruby, Rails and many gems in one PR.
- Ignoring deprecation warnings until the release that removes the feature.
- Flipping all new framework defaults at once, including format changes that break mixed-version rolling deploys.
- `rename_column` / `remove_column` on live tables; indexes built without `concurrently`; NOT NULL added with a full-table lock.
- Backfills inside schema migrations; data migrations that are not idempotent or resumable.
- Refactoring code you do not understand without characterization tests.
- Deleting "dead" code without checking jobs, rake tasks, webhooks and external callers.
- Monkey patches with no test and no reminder to remove them.

## How to talk about this in an interview
- Legacy monoliths are a strength on your resume: you have worked in CrystalCommerce's long-lived e-commerce codebase and other client monoliths. Describe your method for getting productive (routes, schema, models, logs, git history, tests) and one concrete change you made safely.
- For upgrades, give the process even if you have not led one: one minor at a time, deprecations, gems, `app:update`, `load_defaults` step by step, dual boot for long upgrades. Say honestly whether you participated in an upgrade or are describing the approach.
- Zero-downtime migrations connect to real production concerns at CrystalCommerce's scale: concurrent indexes, expand/contract, batched backfills.
- The multi-tenant platform migration is a strangler-fig story at the system level: sync APIs, page-by-page migration, keeping the old system running.

> **Fill in:** the Ruby and Rails versions of the monoliths you worked on, and any upgrade or deprecation work you did.

> **Fill in:** one legacy area you changed (callbacks, a slow page, an old gem): how you understood it, how you tested it and what you changed.

> **Fill in:** a migration on a large table you wrote or reviewed, and how it was deployed safely.

## Interview questions

**Q1. You join a team with a ten-year-old Rails monolith. How do you get productive?**
Get it running locally (`bin/setup`), read the Gemfile for versions and key gems, skim `routes.rb` and `schema.rb` for the surface and data model, then follow one real request through the log, controller, models and side effects. Use git history and tests to learn why code is the way it is, and start with small, well-tested changes.
*Follow-up:* What do you produce early? Notes, a domain glossary or onboarding docs, which help the next person too.

**Q2. How do you trace a bug through an unfamiliar Rails app?**
Reproduce it, find the request in the logs (controller, params, SQL, jobs), confirm the route, read inherited filters and the action, check model callbacks and concerns, and inspect state with `debugger` or a sandboxed console. `git log -S` and `git blame` show when the behavior changed.
*Follow-up:* And if it only happens in production? Use logs, error tracker context and APM traces, and reproduce with production-like data.

**Q3. How do you change legacy code that has no tests?**
Write characterization tests around the entry points first (request or service specs pinning current behavior), then make small refactors or changes, each in its own PR, verified by the tests and code review, ideally behind a feature flag.
*Follow-up:* What if the current behavior looks wrong? Pin it, raise it, and fix it in a separate, deliberate change.

**Q4. What is the strangler fig pattern?**
Replace a legacy system incrementally: put a routing seam in front of the old behavior, build new functionality beside it, move traffic piece by piece, and remove old code once unused. It avoids risky big-bang rewrites and keeps the system shippable throughout.
*Follow-up:* Where does it apply in Rails? Replacing a legacy service or page behind a route or interface, or migrating pages to a new platform.

**Q5. How do feature flags help with legacy changes?**
They separate deploy from release: new code ships dark, is enabled for staff or a percentage of users, and can be turned off instantly as a rollback. Flipper supports actors, groups and percentages. Remove flags once fully rolled out, or they become new legacy.
*Follow-up:* Risk? Many long-lived flags multiply code paths and test cases.

**Q6. How do you untangle callback hell?**
Inventory the callbacks, pin behavior with tests at entry points, keep integrity callbacks on the model, move side effects into explicit services called from controllers and jobs, convert remaining side-effect callbacks to `after_commit`, and remove `skip_callback` workarounds, one step at a time.
*Follow-up:* Why are side-effect callbacks risky? They fire in imports, consoles and tests, and `after_save` runs inside the transaction.

**Q7. How do you upgrade a Rails app?**
One minor version at a time: get tests green on the latest patch, fix deprecation warnings, upgrade blocking gems, bump Rails, run `bin/rails app:update`, fix the suite, deploy through staging, then enable the new framework defaults one by one and bump `load_defaults`. Dual boot helps for long upgrades.
*Follow-up:* Why not jump straight from 6.1 to 8.0? Deprecations and upgrade guides assume each previous version; big jumps hide which change broke what.

**Q8. What are `load_defaults` and the new framework defaults file?**
`config.load_defaults <version>` enables the framework behaviors new apps of that version get. After a gem upgrade you keep the old value and turn on settings individually from the generated `new_framework_defaults_X_Y.rb`, deploying in between; when all are on, you bump `load_defaults` and delete the file.
*Follow-up:* Which settings need extra care? Format changes such as cache and cookie serialization during rolling deploys.

**Q9. What is dual booting?**
Running the app on the current and the next Rails version from the same branch with two lockfiles (next_rails' `Gemfile.next` or bootboot), so CI tests both and fixes land in small PRs on main instead of a long-lived upgrade branch.
*Follow-up:* How does code branch on the version? `NextRails.next?`, removed after the switch.

**Q10. How do you migrate from the classic autoloader to Zeitwerk?**
On Rails 6.x enable `config.autoloader = :zeitwerk`, run `bin/rails zeitwerk:check`, rename files and constants so paths match, add inflections for acronyms, remove `require_dependency`, and move references to reloadable classes in initializers into `to_prepare`. It is mandatory before Rails 7.0, which removed the classic autoloader.
*Follow-up:* What does `app/services/billing/pdf_invoice.rb` define? `Billing::PdfInvoice`.

**Q11. Why does the Rails maintenance policy matter, and what would you tell a team on Rails 7.x?**
Series past their security window get no patches for new vulnerabilities, gems drop support, and upgrades grow harder with time. Rails publishes exact dates on its maintenance page; with the 7.x series at or past the end of its security support, I would plan the move to 8.x now, one minor version at a time.
*Follow-up:* Does upgrading to 8.0 force Solid Queue or Kamal? No, those are defaults for new apps.

**Q12. What is a zero-downtime migration, and what is expand/contract?**
A schema change that never breaks running code or blocks traffic during a deploy. Expand/contract splits a breaking change into compatible steps: add the new structure, dual-write and backfill, switch reads, then stop writing and drop the old structure in a later deploy.
*Follow-up:* Why do old processes break on dropped columns? Active Record caches column lists at boot and still references them.

**Q13. How do you safely rename or remove a column?**
Rename: add a new column, write to both, backfill in batches, switch reads, stop writing the old one, ignore it, deploy, then drop it. Remove: add the column to `ignored_columns`, deploy, then drop it in a later migration.
*Follow-up:* Why not `rename_column`? Old code still running during the deploy queries the old name and errors.

**Q14. How do you add an index or a NOT NULL constraint to a large table?**
Index: `add_index ..., algorithm: :concurrently` with `disable_ddl_transaction!`. NOT NULL: add a check constraint with `validate: false`, validate it in a separate migration, then `change_column_null` (Postgres 12+ uses the validated constraint) and drop the check. Set lock and statement timeouts so blocked DDL fails fast.
*Follow-up:* Foreign keys? `add_foreign_key ... validate: false`, then `validate_foreign_key`.

**Q15. What is the difference between data migrations and schema migrations?**
Schema migrations change structure and must be fast, reversible and run everywhere; data migrations change rows, can take a long time and must be idempotent, batched and resumable. Run data changes with maintenance_tasks, data-migrate or one-off jobs, not inside schema migrations.
*Follow-up:* Why not use models inside old migrations? The model may change later, breaking `db:migrate` from scratch.

**Q16. How do you deal with monkey patches and unmaintained gems?**
Find patches (initializers, `lib/`, `class_eval`, `prepend`), add tests for their behavior and version guards that fail when the gem changes, and prefer `prepend` with `super`. For abandoned gems, replace them with Rails built-ins or maintained alternatives, or fork and pin as a stopgap.
*Follow-up:* Example replacement? Paperclip to Active Storage, attr_encrypted to `encrypts`.

**Q17. How do you find performance problems in a legacy app?**
Start from evidence: APM endpoints by total time, pg_stat_statements, slow logs, queue latency and memory graphs. Common culprits are N+1 queries, missing indexes, heavy callbacks, unbatched jobs and uncached expensive pages; fix one at a time and measure before and after.
*Follow-up:* Which tool in development? rack-mini-profiler and Bullet or `strict_loading`.

## Further reading
- [Upgrading Ruby on Rails](https://guides.rubyonrails.org/upgrading_ruby_on_rails.html)
- [Rails maintenance policy](https://rubyonrails.org/maintenance)
- [Autoloading and Reloading Constants](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
- [Active Record Migrations](https://guides.rubyonrails.org/active_record_migrations.html)
