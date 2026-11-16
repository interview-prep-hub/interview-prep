# Background Jobs: Active Job and Sidekiq

> **Tier:** T1 (know cold) · **Related:** [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [ActiveRecord advanced](06-activerecord-advanced.md), [Redis](../12-databases/10-redis.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md), [Testing with RSpec](13-testing-rspec.md)
> **On your resume:** background jobs syncing listings, prices, stock and orders with PriceCharting and importing market prices for CrystalCommerce; Sidekiq in the Colochousing Rails 7 app; Sidekiq listed in your skills.

## Quick revision
- Jobs move slow or unreliable work (emails, third-party APIs, imports, PDFs, push notifications) out of the request, add retries, and let you schedule and fan out work.
- **Active Job** is Rails' common API: `perform_later`, `set(wait: 10.minutes)`, `queue_as`, `retry_on`, `discard_on`; records are serialized as **GlobalID** and reloaded when the job runs. Adapters: Sidekiq, Solid Queue (Rails 8 default), GoodJob, Delayed Job.
- **Sidekiq:** the client serializes the job to JSON and pushes it to a Redis list (`queue:default`); a separate server process runs many threads that pop jobs and call `perform`. Scheduled jobs, retries and dead jobs live in Redis sorted sets.
- `Sidekiq::Job` (`perform_async`, JSON-native args only) is leaner and exposes Sidekiq features directly; Active Job is portable across backends.
- Default retries: **25**, with exponential backoff spread over roughly **three weeks**, then the job moves to the **Dead** set. Tune with `sidekiq_options retry: 5`, `sidekiq_retry_in`, `sidekiq_retries_exhausted`.
- Delivery is **at least once**: retries, crashes and duplicate enqueues mean a job can run more than once, so every job must be **idempotent**.
- Idempotency toolkit: pass IDs, reload and re-check state, atomic claims, upserts keyed on external IDs, unique constraints, absolute values instead of increments, idempotency keys for external calls.
- Unique jobs, rate limiting and periodic jobs are Sidekiq Enterprise features; batches and `super_fetch` are Pro. Community gems: sidekiq-unique-jobs, sidekiq-cron.
- Open-source Sidekiq's basic fetch can **lose in-flight jobs on a hard crash**; a graceful shutdown waits (25 seconds by default) and then re-queues unfinished jobs; Pro's `super_fetch` recovers jobs after crashes.
- Enqueue after commit (`after_commit`), never inside an open transaction.
- DB pool per Sidekiq process >= its concurrency.
- Test with `Sidekiq::Testing.fake!` (`Job.jobs`, `drain`) or Active Job matchers (`have_enqueued_job`), and test `perform` directly, including running it twice.
- **Solid Queue** stores jobs in SQL and fetches with `FOR UPDATE SKIP LOCKED`: no Redis to run, lower peak throughput than Sidekiq.

## 1. Why background jobs

A web request should finish in milliseconds. Anything slow, flaky or not needed for the response goes to a job:

| Use case | Example |
|---|---|
| Slow side effects | Emails, push notifications, PDF invoices, image processing |
| Unreliable third-party calls | Syncing listings to a sales channel, charging a card, geocoding |
| Large batch work | Importing market prices for a big catalog, recalculating counters |
| Scheduled and recurring work | Nightly imports, rent reminders, cleanup |
| Fan-out | One trigger becomes one job per store or per batch |
| Load smoothing | A queue absorbs bursts instead of timing out web requests |

The trade-offs: eventual consistency (the result is not ready when the response returns), more moving parts (Redis or queue tables, worker processes, monitoring), and the need to design for retries and duplicates.

## 2. Active Job

```ruby
class SyncListingJob < ApplicationJob
  queue_as :catalog_sync

  retry_on Net::OpenTimeout, Net::ReadTimeout, wait: :polynomially_longer, attempts: 5
  retry_on ChannelClient::RateLimited, wait: 1.minute, attempts: 10
  discard_on ActiveJob::DeserializationError        # the record was deleted before the job ran

  def perform(listing)
    ChannelClient.new.push_listing(listing)
  end
end

SyncListingJob.perform_later(listing)                         # GlobalID: "gid://shop/Listing/42"
SyncListingJob.set(wait: 10.minutes).perform_later(listing)   # scheduled
SyncListingJob.set(queue: :critical).perform_later(listing)
SyncListingJob.perform_now(listing)                           # runs inline, in this process
```

- **GlobalID** serializes Active Record arguments as a reference and reloads them when the job runs, so the job sees fresh data. If the row is gone, the job raises `ActiveJob::DeserializationError`.
- `retry_on` re-enqueues with a delay for listed exceptions (`:polynomially_longer` was called `:exponentially_longer` before Rails 7.1). When its attempts run out, the exception bubbles up to the backend, and Sidekiq may then apply its own retries; decide which layer owns retries.
- Supported argument types include strings, numbers, booleans, nil, arrays, hashes, symbols, dates and times, durations and GlobalID objects.
- `ChannelClient` here and below is an illustrative wrapper around a third-party API.

| Adapter | Backend | Notes |
|---|---|---|
| Sidekiq | Redis | Threads in a separate process; very fast; Pro and Enterprise add features |
| Solid Queue | Your SQL database | Rails 8 default; `FOR UPDATE SKIP LOCKED`; recurring tasks built in |
| GoodJob | PostgreSQL | Advisory locks, LISTEN/NOTIFY, cron and dashboard built in |
| Delayed Job | SQL table `delayed_jobs` | Older, still common in legacy apps |
| `:async` / `:inline` | In process | Development and tests only; `:async` loses jobs on restart |

## 3. How Sidekiq works

```ruby
class SendRentReminderJob
  include Sidekiq::Job                              # Sidekiq::Worker in older code
  sidekiq_options queue: :notifications, retry: 5

  def perform(rent_payment_id)                      # JSON-native args only: pass IDs
    payment = RentPayment.find_by(id: rent_payment_id)
    return if payment.nil? || payment.paid?
    PushNotifier.rent_due(payment)                  # illustrative notifier
  end
end

SendRentReminderJob.perform_async(payment.id)
SendRentReminderJob.perform_in(2.hours, payment.id)
SendRentReminderJob.perform_at(Date.tomorrow.noon, payment.id)
```

1. **Client** (inside your web process): `perform_async` builds a JSON payload like `{"class":"SendRentReminderJob","args":[42],"queue":"notifications","jid":"...","retry":5}` and pushes it onto the Redis list `queue:notifications`. That is the whole enqueue: one fast Redis write.
2. **Server**: a separate process (`bundle exec sidekiq -C config/sidekiq.yml`) boots your Rails app and starts `concurrency` threads (recent versions default to 5). Each thread blocks on Redis waiting for work, pops a job, instantiates the class and calls `perform(*args)`.
3. **Queues and weights**: `queues: [[critical, 6], [default, 3], [low, 1]]` picks queues randomly in proportion to weight, so low-priority work still progresses; listing queues without weights checks them in strict order, which can starve the last ones.
4. **Sorted sets**: `perform_in` / `perform_at` put the job in the **schedule** set (score = run time); a failed job goes to the **retry** set with its next retry time; a poller moves due jobs back to their queues. Jobs that exhaust retries go to the **dead** set (kept for months, up to 10,000 by default), from which you can retry them manually.
5. **Web UI**: `mount Sidekiq::Web => "/sidekiq"` (behind admin authentication) shows queues, busy threads, retries, scheduled and dead jobs, with retry and delete buttons.

Because jobs are plain JSON in Redis, Redis must be persistent and configured with `maxmemory-policy noeviction`; an evicting Redis silently drops jobs.

| | `Sidekiq::Job` | Active Job on the Sidekiq adapter |
|---|---|---|
| API | `perform_async`, `perform_in`, `sidekiq_options` | `perform_later`, `set(wait:)`, `queue_as` |
| Arguments | JSON-native only (IDs, strings, numbers, arrays, string-keyed hashes) | GlobalID records and more types |
| Retries | `sidekiq_options retry:`, `sidekiq_retry_in` | `retry_on` / `discard_on`, plus Sidekiq's layer |
| Portability | Tied to Sidekiq | Swap backends without touching job code |
| Overhead and features | Leaner; Pro/Enterprise features apply directly | Small wrapper overhead; Rails conveniences (mailers' `deliver_later` use it) |

## 4. Retries

When `perform` raises, Sidekiq records the error, increments `retry_count` and schedules the job in the retry set. The default (`retry: true`) is **25 retries** with exponential backoff (roughly `retry_count ** 4` seconds plus jitter), which spreads attempts over about three weeks; after the last one, the job moves to the Dead set.

```ruby
class PushListingJob
  include Sidekiq::Job
  sidekiq_options queue: :pricecharting, retry: 10   # retry: 0 sends failures straight to Dead,
                                                     # retry: false drops them entirely
  sidekiq_retry_in do |count, exception|
    case exception
    when ChannelClient::RateLimited then exception.retry_after.to_i  # honor the API's Retry-After
    else (count**4) + 15 + rand(10)                                  # roughly the default curve
    end
  end

  sidekiq_retries_exhausted do |job, exception|
    listing_id = job["args"].first
    ChannelListing.where(listing_id: listing_id).update_all(sync_status: :failed)
    Rails.logger.error("push_listing_dead listing_id=#{listing_id} error=#{exception.message}")
  end

  def perform(listing_id)
    listing = Listing.find_by(id: listing_id)
    return unless listing                       # deleted since enqueue: nothing to do
    ChannelClient.new.push_listing(listing)
  end
end
```

- Retries are what make jobs resilient to timeouts and outages, and also the main reason a job runs more than once.
- Do not retry errors that can never succeed (validation failures, 404 for a deleted remote object): rescue them and return, or use `discard_on` in Active Job, so the retry set does not fill with noise.
- `sidekiq_retries_exhausted` (per job) and `config.death_handlers` (global) are where you mark records as failed and alert.

## 5. Idempotency under at-least-once delivery

Sidekiq guarantees a job runs **at least once**, not exactly once. Duplicates come from retries after partial success (the email was sent, then the job crashed before recording it), re-queued jobs after a shutdown, duplicate enqueues (a double-clicked button, two callbacks) and a client retrying a push it thought had failed. An **idempotent** job produces the same end state no matter how many times it runs.

| Technique | Example |
|---|---|
| Pass IDs, reload fresh state | `perform(listing_id)` then `Listing.find_by(id:)`, never a serialized object or price |
| Re-check state before acting | `return if payment.paid?` |
| Claim work atomically | `update_all(reminder_sent_at: Time.current)` on `WHERE reminder_sent_at IS NULL`, continue only if 1 row changed |
| Upsert keyed on an external ID | `upsert_all(rows, unique_by: %i[channel external_id])` |
| Unique constraints | Unique index on `external_order_id`, so an order imported twice fails or is skipped |
| Absolute values, not deltas | `update!(stock: remote_stock)` instead of `decrement!(:stock)` when syncing from a source of truth |
| Idempotency keys for external calls | Send the same key on retries so the payment provider charges once |

```ruby
class SendRentReminderJob
  include Sidekiq::Job
  sidekiq_options queue: :notifications, retry: 5

  def perform(rent_payment_id)
    payment = RentPayment.find_by(id: rent_payment_id)
    return if payment.nil? || payment.paid?                 # state changed since enqueue

    claimed = RentPayment.where(id: payment.id, reminder_sent_at: nil)
                         .update_all(reminder_sent_at: Time.current)
    return if claimed.zero?                                 # another run already sent it

    begin
      PushNotifier.rent_due(payment)
    rescue PushNotifier::TemporaryError
      RentPayment.where(id: payment.id).update_all(reminder_sent_at: nil)  # release the claim
      raise                                                 # let Sidekiq retry
    end
  end
end
```

The claim makes concurrent duplicates harmless; releasing it on a temporary failure lets the retry try again. For side effects that cannot be made idempotent, choose deliberately between "might happen twice" and "might not happen", and record what happened so a reconciliation job can repair it.

## 6. Unique jobs and locks

Uniqueness at enqueue time reduces wasted work (one sync per store at a time) but does not replace idempotency: locks expire and processes crash.

- **Sidekiq Enterprise:** `sidekiq_options unique_for: 10.minutes` for unique jobs, plus `Sidekiq::Limiter` rate limiters (concurrent, window, bucket) and leader election.
- **Community:** the sidekiq-unique-jobs gem (`sidekiq_options lock: :until_executed`, among other lock types); Solid Queue's `limits_concurrency` controls for Active Job.
- **Database:** an advisory lock (for example through the with_advisory_lock gem) or a row in a `sync_runs` table with a unique index on the store and status.

## 7. Recurring jobs

```yaml
# config/schedule.yml loaded by sidekiq-cron (community gem)
import_market_prices:
  cron: "0 3 * * *"              # every day at 03:00
  class: "ImportMarketPricesJob"
  queue: catalog_sync
```

```ruby
# Sidekiq Enterprise periodic jobs
Sidekiq.configure_server do |config|
  config.periodic do |mgr|
    mgr.register("0 3 * * *", "ImportMarketPricesJob")
  end
end
```

```yaml
# config/recurring.yml with Solid Queue (check its README for the keys in your version)
production:
  import_market_prices:
    class: ImportMarketPricesJob
    schedule: every day at 3am
```

Avoid plain system cron (the whenever gem) on several servers: each server fires the job, so you get duplicates. Scheduler tools above coordinate through Redis or the database so one instance enqueues. Keep the recurring job tiny: it should only enqueue the real work.

## 8. Crashes, shutdowns and the connection pool

- **Graceful shutdown** (deploys send `TERM`): Sidekiq stops fetching, waits for running jobs up to the timeout (25 seconds by default), then pushes unfinished jobs back to Redis. Those jobs restart from the beginning later, which is one more reason for idempotency and short jobs. Send `TSTP` first ("quiet") to stop fetching during long deploys.
- **Hard crash** (`KILL`, the out-of-memory killer, a lost machine): open-source Sidekiq's basic fetch has already removed in-flight jobs from Redis, so they are **lost**. Sidekiq Pro's `super_fetch` keeps in-progress jobs in a per-process list and recovers them after a crash. Without Pro, reconciliation jobs that re-enqueue stale work are your safety net.
- **Connection pool:** each Sidekiq thread may hold a database connection, so the pool in `database.yml` must be at least the process's concurrency, and all processes together must fit under the database connection limit (math in [ActiveRecord advanced](06-activerecord-advanced.md)). The same applies to Redis connections and HTTP clients shared across threads.
- **Thread safety:** job code runs on many threads at once; avoid mutable class-level state and memoized globals.

## 9. Designing a third-party catalog sync (PriceCharting-style)

Integrating a marketplace or price-data API as a sales channel is a classic jobs design question, and it maps directly onto your CrystalCommerce bullets (listings, prices, stock and orders synced by background jobs; market prices imported into the catalog). The sketch below is a typical design to adapt, not the real CrystalCommerce code.

```text
cron (nightly or every N minutes)
  -> CatalogSyncCoordinatorJob           enqueues one job per enabled store (fan-out)
       -> StoreChannelSyncJob(store_id)  pages through the API with a saved cursor,
                                         upserts by external ID, checkpoints after each page
local change (price, stock edit)
  -> after_commit, if relevant columns changed -> PushListingJob(listing_id)
ReconciliationJob (daily)                compares local vs remote, repairs drift, re-enqueues failures
```

```ruby
class CatalogSyncCoordinatorJob
  include Sidekiq::Job
  sidekiq_options queue: :catalog_sync, retry: 3

  def perform
    Store.where(channel_enabled: true).find_each do |store|
      StoreChannelSyncJob.perform_async(store.id)
    end
  end
end

class StoreChannelSyncJob
  include Sidekiq::Job
  sidekiq_options queue: :channel_api, retry: 10   # a dedicated queue served by few threads

  sidekiq_retry_in do |count, exception|
    exception.is_a?(ChannelClient::RateLimited) ? exception.retry_after.to_i : (count**4) + 15
  end

  def perform(store_id, cursor = nil)
    store = Store.find_by(id: store_id) or return
    state = SyncState.find_or_create_by!(store_id: store.id, channel: "pricecharting")
    page  = ChannelClient.new(store).changed_listings(since: cursor || state.cursor)  # raises RateLimited on 429

    rows = page.items.map { |item| item.to_row(store_id: store.id) }
    ChannelListing.upsert_all(rows, unique_by: %i[channel external_id]) if rows.any?
    state.update!(cursor: page.next_cursor, last_synced_at: Time.current)             # checkpoint

    self.class.perform_async(store.id, page.next_cursor) if page.next_cursor          # next page = new job
  end
end
```

Design points to say out loud:
- **Fan-out** per store (or per batch of IDs): one failing store does not block the others, work spreads across threads and processes, and each job stays short enough to survive deploys.
- **Rate limiting:** respect the provider's quota by routing API jobs to a dedicated queue served by a small, fixed number of threads, or with a rate limiter (Enterprise `Sidekiq::Limiter`, or a Redis token bucket). On HTTP 429, read `Retry-After` and reschedule instead of hammering the API; make the client's timeouts explicit.
- **Cursors and checkpoints:** saving the cursor after every page means a retry or a re-queued job resumes where it stopped instead of starting over.
- **Idempotent writes:** upserts keyed on `(channel, external_id)`, absolute prices and stock from the source of truth, unique indexes on external order IDs so an order imported twice is still one order.
- **Enqueue after commit:** local edits trigger pushes from `after_commit`, only when `saved_change_to_price_cents?` or stock changed, and bursts are collapsed (unique jobs or a "dirty" flag swept periodically).
- **Reconciliation:** jobs and webhooks get lost; a daily job compares counts or checksums, re-pushes stale listings (`last_synced_at` too old) and alerts on drift.
- **Observability:** structured logs with `store_id`, `jid`, page and counts; metrics for processed, failed, 429s and duration; alerts on a growing Dead set or stale `last_synced_at`; the Sidekiq Web UI for triage.
More patterns in [third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md) and [background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md).

## 10. Testing jobs

```ruby
# Sidekiq::Job classes
require "sidekiq/testing"
Sidekiq::Testing.fake!                                # default: jobs collect in an array

it "enqueues a sync per enabled store" do
  create_list(:store, 2, channel_enabled: true)
  CatalogSyncCoordinatorJob.new.perform
  expect(StoreChannelSyncJob.jobs.size).to eq(2)
  StoreChannelSyncJob.drain                           # run them (stub the HTTP client first)
end

# Active Job classes (test adapter)
it "enqueues the confirmation after checkout" do
  expect { post orders_path, params: valid_params }
    .to have_enqueued_job(OrderConfirmationJob).on_queue("default")
end

# Idempotency: running twice has one effect
it "sends the rent reminder once" do
  expect(PushNotifier).to receive(:rent_due).once
  2.times { SendRentReminderJob.new.perform(payment.id) }
end
```

Test the logic by calling `perform` directly, stub external HTTP (WebMock or VCR), and use `Sidekiq::Testing.inline!` or `perform_enqueued_jobs` only when you need end-to-end behavior. Minitest equivalents: `assert_enqueued_with(job: OrderConfirmationJob)` and `assert_enqueued_jobs 1`. More in [testing with RSpec](13-testing-rspec.md).

## 11. Solid Queue vs Sidekiq

| | Sidekiq | Solid Queue |
|---|---|---|
| Storage | Redis | Your SQL database (often a separate queue database) |
| Fetching | Blocking pops from Redis lists (or Pro `super_fetch`) | Polling with `SELECT ... FOR UPDATE SKIP LOCKED` |
| Throughput | Very high | Good for most apps; adds load to the database |
| Infrastructure | A Redis server to run and persist | Nothing beyond the database |
| Crash safety | Basic fetch can lose in-flight jobs; Pro fixes it | Claimed jobs live in tables and are released when a worker dies |
| Features | Mature ecosystem; Pro/Enterprise: batches, unique jobs, rate limits, periodic jobs | Recurring tasks, concurrency limits, queue pausing; Mission Control as a UI |
| API | `Sidekiq::Job` or Active Job | Active Job only |
| Status | Long-time community standard | Default in Rails 8 |

Pick Sidekiq for high-volume queues, an existing Redis setup or Pro/Enterprise features (the common case in established apps). Pick Solid Queue for new apps that want fewer moving parts and moderate job volume (more on the Solid trio in [Rails 7 and 8](18-rails-7-8-whats-new.md)). If Solid Queue's tables live in the same database as your data, enqueueing can even join the app's transaction; with the default separate queue database it cannot, so the after-commit rules still apply.

## Common pitfalls
- Enqueuing inside a transaction or in `after_save`: the job runs before COMMIT or after a rollback.
- Passing whole objects or stale values (`perform(listing.price)`) instead of IDs, so the job acts on outdated data.
- Non-idempotent jobs: increments, duplicate emails or double charges when a job is retried or re-queued.
- Long-running jobs that loop over thousands of records: killed at deploy, restarted from zero. Split work and checkpoint.
- Retrying errors that can never succeed, which floods the retry set and hides real failures.
- No rate limiting on third-party API jobs, so a fan-out triggers 429s or an account ban.
- An evicting or non-persistent Redis, which loses queued jobs.
- DB pool smaller than Sidekiq concurrency (`ConnectionTimeoutError` in jobs).
- Leaving the Sidekiq Web UI unauthenticated.
- Ignoring the Dead set; nobody notices sync failures until a customer does.

## How to talk about this in an interview
- CrystalCommerce is your flagship jobs story: PriceCharting as a sales channel with background jobs syncing listings, prices, stock and orders, plus importing market prices into the catalog for 700+ stores. Structure the answer as fan-out, rate limits, idempotent upserts, checkpoints, retries and reconciliation, and be explicit about which parts you built and which you are describing as the general pattern.
- Colochousing uses Sidekiq in a Rails 7 app with a resident mobile app on top, so notifications, reminders and syncs are natural examples of after-commit enqueueing and idempotent jobs.
- Use the vocabulary precisely: at-least-once, idempotent, retry set, dead set, backoff, fan-out, checkpoint.
- If asked about something you have not used (Sidekiq Pro, Solid Queue in production), say so and explain how it works and when you would pick it.

> **Fill in:** how the PriceCharting sync was triggered (schedule, webhook, user action) and how often it ran.

> **Fill in:** how the sync handled rate limits, failures and retries, and what you monitored (logs, Sidekiq UI, alerts).

> **Fill in:** one Sidekiq job you wrote or fixed in the Colochousing app and what made it safe to retry.

## Interview questions

**Q1. Why use background jobs? Give examples.**
To keep requests fast and isolate slow or unreliable work: emails, push notifications, third-party API calls, imports, PDF generation, image processing, and scheduled tasks. Jobs also add retries and let you fan out large work across workers.
*Follow-up:* What do you give up? Immediate consistency, plus extra infrastructure and failure modes to monitor.

**Q2. How does Sidekiq work?**
The client serializes the job class, arguments and options to JSON and pushes it onto a Redis list for its queue. A separate Sidekiq process boots the app and runs a pool of threads; each thread pops a job, instantiates the class and calls `perform`. Scheduled jobs, retries and dead jobs live in Redis sorted sets, and a poller moves due jobs back onto queues.
*Follow-up:* Why threads rather than processes? Jobs are mostly IO-bound, so threads give high concurrency with little memory; the GVL is released while waiting on IO.

**Q3. How do retries work in Sidekiq?**
If `perform` raises, the job goes to the retry set with an exponentially growing delay; the default is 25 retries over about three weeks, after which it moves to the Dead set. You can change the count (`sidekiq_options retry: 5`), the delay (`sidekiq_retry_in`) and add a `sidekiq_retries_exhausted` hook for alerts or cleanup.
*Follow-up:* `retry: 0` vs `retry: false`? `0` sends a failure straight to the Dead set; `false` discards it.

**Q4. What makes a job idempotent?**
Running it twice leaves the same state as running it once. Pass IDs and reload state, re-check conditions before acting, claim work atomically, use upserts and unique constraints keyed on stable IDs, write absolute values instead of deltas, and send idempotency keys to external services.
*Follow-up:* Why does it matter? Sidekiq delivers at least once: retries, re-queues at shutdown and duplicate enqueues all cause repeats.

**Q5. Why pass IDs instead of objects to jobs?**
Arguments are serialized to JSON in Redis, so objects cannot be stored faithfully, and any data captured at enqueue time may be stale when the job runs. An ID lets the job load the current record and handle the case where it was deleted.
*Follow-up:* How does Active Job handle records? GlobalID stores a reference and reloads the record when the job runs, raising `DeserializationError` if it is gone.

**Q6. Active Job or `Sidekiq::Job`?**
Active Job gives a backend-neutral API, GlobalID arguments and Rails integrations like `deliver_later`. `Sidekiq::Job` is leaner, requires JSON-native arguments and exposes Sidekiq features (options, Pro and Enterprise APIs) directly. Many apps use Active Job for mailers and simple jobs and native Sidekiq jobs for heavy pipelines.
*Follow-up:* A retry gotcha when mixing them? Active Job's `retry_on` retries first, then Sidekiq may retry again once the error escapes; configure one layer deliberately.

**Q7. What happens to running jobs when Sidekiq restarts or crashes?**
On a graceful shutdown, Sidekiq stops fetching, waits up to its timeout (25 seconds by default), then pushes unfinished jobs back to Redis to run again. On a hard crash, open-source Sidekiq loses in-flight jobs because they were already popped; Sidekiq Pro's `super_fetch` recovers them.
*Follow-up:* How do you protect against lost jobs without Pro? Short idempotent jobs plus reconciliation jobs that re-enqueue stale work.

**Q8. How do you prevent duplicate jobs?**
Sidekiq Enterprise unique jobs (`unique_for:`), the sidekiq-unique-jobs gem in the community version, Solid Queue concurrency limits, or a database lock or unique row per unit of work. Uniqueness only reduces duplicates; the job itself must still be idempotent.
*Follow-up:* Why is uniqueness not enough? Locks expire and processes crash, so a duplicate can still slip through.

**Q9. How do you schedule recurring jobs?**
sidekiq-cron (a YAML file with cron expressions), Sidekiq Enterprise periodic jobs, Solid Queue recurring tasks or GoodJob's cron. Avoid system cron on several servers, which fires once per server. The scheduled job should only enqueue the real work.
*Follow-up:* What does a cron expression `0 3 * * *` mean? Every day at 03:00.

**Q10. Design a job pipeline that syncs a store catalog with a third-party API like PriceCharting.**
A scheduled coordinator fans out one job per store; each job pages through the API with a saved cursor, upserts rows keyed on external IDs and checkpoints after each page; API jobs run on a rate-limited queue and back off on 429 using `Retry-After`; local edits enqueue pushes after commit; a reconciliation job repairs drift; logs, metrics and the Dead set are monitored.
*Follow-up:* How do you keep one slow store from delaying everyone? Fan-out per store or batch, with per-job time limits and a separate queue for heavy accounts.

**Q11. How do you handle a 429 Too Many Requests from an API inside a job?**
Raise a specific error carrying the `Retry-After` value and use `sidekiq_retry_in` (or reschedule with `perform_in`) to wait that long, and reduce pressure upstream with a limited-concurrency queue or a rate limiter. Never retry immediately in a tight loop.
*Follow-up:* What if 429s persist? Lower concurrency, spread schedules, and alert, because the quota does not fit the workload.

**Q12. How do you prioritize critical jobs?**
Use separate queues (`critical`, `default`, `low`) with weights so higher queues are checked more often without starving lower ones, or run a dedicated Sidekiq process for critical queues. Strict ordering (no weights) processes higher queues first but can starve the rest.
*Follow-up:* Why a dedicated process? Slow, bulky jobs cannot block urgent ones that have their own threads.

**Q13. How do you size the database pool for Sidekiq?**
Each Sidekiq thread may need a connection, so the pool per Sidekiq process must be at least its concurrency, and the total across web and worker processes must stay under the database's connection limit.
*Follow-up:* Symptom of getting it wrong? Jobs fail with `ActiveRecord::ConnectionTimeoutError`.

**Q14. How do you test background jobs?**
Unit-test `perform` directly with external HTTP stubbed; assert enqueueing with `Sidekiq::Testing.fake!` (`Job.jobs`) or Active Job matchers (`have_enqueued_job`); run queued jobs with `drain` or `perform_enqueued_jobs` when needed; and run a job twice to prove idempotency.
*Follow-up:* Why not `inline!` everywhere? It hides enqueue-timing bugs and slows the suite.

**Q15. Solid Queue or Sidekiq?**
Sidekiq stores jobs in Redis and is very fast with a mature ecosystem and paid features; Solid Queue (Rails 8 default) stores jobs in SQL tables using `FOR UPDATE SKIP LOCKED`, so there is no Redis to run, at the cost of database load and lower peak throughput. Choose by volume, existing infrastructure and needed features.
*Follow-up:* Which runs `Sidekiq::Job` classes? Only Sidekiq; Solid Queue works through Active Job.

**Q16. Why is enqueuing a job inside a database transaction risky?**
The job reaches Redis immediately, so it can run before the transaction commits (and not find the row) or after it rolls back (acting on data that never existed). Enqueue after commit, for example with `after_create_commit`. Details in [ActiveRecord advanced](06-activerecord-advanced.md).
*Follow-up:* What about Active Job's transaction-aware enqueueing? Newer Rails versions can defer enqueues to after commit; the default changed between versions, so check yours.

## Further reading
- [Active Job Basics](https://guides.rubyonrails.org/active_job_basics.html)
- [Active Job API](https://api.rubyonrails.org/classes/ActiveJob/Base.html) and the [Sidekiq wiki](https://github.com/sidekiq/sidekiq/wiki)
- [Redis documentation](https://redis.io/docs/)
