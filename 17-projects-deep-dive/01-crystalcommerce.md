# CrystalCommerce: Legacy Rails E-commerce and the PriceCharting Channel

> **Tier:** T1 (know cold) · **Related:** [Background jobs and Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [E-commerce domain](../13-architecture-and-practices/07-ecommerce-domain.md), [Legacy codebases and upgrades](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md), [System design from your projects](../15-system-design/03-case-studies-from-your-projects.md)
> **On your resume:** Project CrystalCommerce (Devntech client; Ruby on Rails, PostgreSQL, background jobs, PriceCharting API).

## Quick revision

| Item | What to remember |
|---|---|
| Where | A Devntech client project, from your role as an Associate Software Engineer at Devntech (Nov 2025 to present) |
| Stack on your resume | Ruby on Rails, PostgreSQL, background jobs, PriceCharting API |
| Platform scale | 700+ game stores and 8M+ customers. This is the size of the platform, not a count of users you brought in or a system you built alone |
| Bullet 1 (verbatim) | "Enhanced CrystalCommerce's legacy Rails e-commerce platform, serving 700+ game stores and 8M+ customers." |
| Bullet 2 (verbatim) | "Integrated PriceCharting as a sales channel: background jobs sync listings, prices, stock, and orders." |
| Bullet 3 (verbatim) | "Imported PriceCharting market prices into the catalog so stores can price inventory from market data." |

- **Say the numbers like this:** "I work on CrystalCommerce's platform, which serves 700+ game stores and 8M+ customers. My part was ..." Platform scale first, your scope second. Never "I built a system for 8 million customers."
- **PriceCharting in one line:** a price-guide service for video games and other collectibles that also supports selling. Describe only the parts of its API you actually used; never improvise endpoints, auth or limits.
- **Three data flows:** outbound (listings, prices and stock pushed to the channel), inbound (orders pulled from the channel), and a one-way import (market prices into the catalog).
- **The design in one breath:** a channel adapter, per-store jobs, idempotent upserts keyed by external IDs, rate limiting with backoff, reconciliation, alerting on partial failures.
- **Hardest correctness problem:** the same item selling on two channels (overselling). One source of truth for stock, atomic decrements, fast quantity pushes, and a recovery path when it still happens.
- **Idempotency keys:** external listing IDs and external order IDs, enforced by unique indexes, not by "check then insert" code.
- **Rate limits:** a dedicated queue with low concurrency, honoring `Retry-After` on HTTP 429, exponential backoff with jitter.
- **Legacy context:** read before you change, add characterization tests, keep diffs small, and put new code behind a boundary (the adapter) instead of spreading it through old models.

> **Fill in:** which background job library the codebase uses (Sidekiq, Delayed Job, Resque, GoodJob, Active Job on some adapter), plus the Rails and Ruby versions.

## The 30-second pitch

> "At Devntech I work on CrystalCommerce, a legacy Rails e-commerce platform that serves 700+ game stores and 8M+ customers. I integrated PriceCharting, a price guide for video games and collectibles that also supports selling, as a sales channel: background jobs keep listings, prices, stock and orders in sync between the stores and PriceCharting. I also imported PriceCharting's market prices into the catalog, so stores can price their inventory from market data."

> **Fill in:** one closing sentence that makes the pitch yours: the hardest problem you solved or the decision you are proudest of (for example how the sync avoided overselling, or how it handled rate limits). Use only what really happened.

> **Fill in:** did you build the integration alone, with a teammate, or as part of a larger team? Use "I" for your parts and "we" for the team's.

How to adapt it:
- **Rails role:** lead with the legacy monolith and the job design (idempotency, retries, rate limits).
- **General full-stack role:** add one line on the store-facing side (settings, sync status, pricing screens) if you built it.
- **System design round:** offer the sync as your case study and switch to the [design deep-dive](#design-deep-dive).

## Domain primer

### Game stores and what they sell
Game stores typically sell a mix of new products and used or one-off items: video games, consoles and accessories, trading cards, tabletop games and collectibles. That mix shapes the software:
- **Many low-quantity items.** A used cartridge or a single card in a specific condition often has a quantity of 1, so one sale can make a listing unavailable everywhere.
- **Condition drives price.** The same title can be worth very different amounts depending on condition (for video games, for example, loose vs complete in box vs new). Inventory, listings and market prices all need a condition dimension.
- **Prices move.** Collectible values change with demand, so stores reprice often, and a stale price either loses money or loses sales.

### Multichannel selling
A store can sell the same inventory on its own storefront and on external sales channels (marketplaces). Each channel holds its own copy of the listing (title, price, quantity) and produces its own orders. Because the stock is shared, the platform must keep every channel's copy up to date and pull every channel's orders back in. That is what "integrated as a sales channel" means.

### Price guides and market data
Price guides estimate what an item is worth by tracking real sales. A store can use a market price as a starting point: list at market, above market for rare items, or below it to move stock. Importing those prices into the catalog lets the platform suggest or set prices instead of a person looking each item up.

### PriceCharting
PriceCharting is a price-guide service focused on video games and other collectibles that also supports selling. In your project it plays both roles: a channel the stores sell through, and a source of market prices for the catalog. Its endpoints, authentication, rate limits and data formats are defined in its own API documentation. In interviews, describe what your integration used, and for anything else say you would check the docs.

> **Fill in:** which PriceCharting capabilities the integration used (product lookups, price data, listings or offers, orders), and how stores connected their accounts.

### Glossary

| Term | Meaning in this project |
|---|---|
| Listing | A store's offer of an item on a channel: price, quantity, condition |
| Channel | A place where a store sells: its storefront or an external marketplace |
| External ID | The ID the channel assigns to a listing, product or order; your idempotency key |
| Sync | Making both sides agree: push local changes out, pull remote changes in |
| Market price | A price guide's estimate of current value, usually per condition |
| Overselling | Selling more units than you have, typically when two channels sell the last unit |
| Reconciliation | A periodic comparison of both sides that finds and repairs drift |
| Idempotent | Safe to run twice: the second run changes nothing |

## Typical architecture

> This is a typical setup for a channel integration inside a Rails monolith, to adapt to what you actually saw. It is not a description of the real CrystalCommerce codebase.

```text
Store admin UI (Rails views)             Storefront (customers)
          |                                        |
          v                                        v
+------------------------- Rails monolith -------------------------+
|  Catalog, Inventory, Orders, Pricing (models + service objects)  |
|  SalesChannels::PriceChartingChannel (adapter: HTTP, auth,       |
|                                       paging, error mapping)     |
+------------------------------------------------------------------+
      | enqueue after commit                      ^ idempotent upserts
      v                                           |
   Job queue  ---->  worker processes  ---- HTTPS ---->  PriceCharting
      ^
   Scheduler: recurring order pulls, price imports, reconciliation
   PostgreSQL: listings, channel_listings, channel_orders,
               product_mappings, market_prices, pricing_rules, sync_runs
```

Typical tables an integration like this adds:

| Table | Key columns | Why |
|---|---|---|
| `channel_connections` | `store_id`, `channel`, encrypted credentials, `enabled`, `orders_synced_until` | Per-store settings and the order-sync checkpoint |
| `channel_listings` | `listing_id`, `channel`, `external_id` (unique), `last_pushed_quantity`, `synced_at`, `last_error` | Maps local listings to remote ones and shows sync health |
| `channel_orders` | `channel`, `external_id` (unique), `store_id`, `status`, `stock_applied`, `payload` (jsonb) | Dedupe, plus an audit trail of what the channel sent |
| `product_mappings` | `product_id`, `external_product_id`, `matched_by`, `confidence`, `confirmed_at` | Links catalog products to PriceCharting products |
| `market_prices` | `external_product_id`, `condition`, `price_cents`, `fetched_at` | Imported market data, with history if you keep old rows |
| `pricing_rules` | `store_id`, `condition`, `multiplier`, `min_price_cents`, `max_change`, `auto_apply` | How a store turns market prices into its own prices |
| `sync_runs` | `store_id`, `kind`, `started_at`, `finished_at`, `succeeded`, `failed` | Visibility and alerting |

Three pipelines run on top:
1. **Outbound:** a price or stock change commits, a job pushes it to the channel (event-driven), and a periodic sweep re-pushes anything stale.
2. **Inbound:** a recurring job per store pulls new or updated orders since a checkpoint, imports each one idempotently and decrements stock.
3. **Market prices:** a recurring import fetches prices for mapped products, stores them, and applies each store's pricing rules where the store opted in.

> **Fill in:** what the real flows looked like: what triggered each job, how often recurring jobs ran, and roughly how much data one run handled.

## Bullet-by-bullet

### Bullet 1: "Enhanced CrystalCommerce's legacy Rails e-commerce platform, serving 700+ game stores and 8M+ customers."

**What it signals:** you can work safely in a large, old codebase with real users, which describes most Rails jobs.

**What they will ask:** what "legacy" meant concretely; what you enhanced besides PriceCharting; how you avoided breaking stores; how you found your way around.

**How to answer:**
- Define legacy with specifics you saw (old Rails or gem versions, fat models, long callback chains, thin test coverage, unusual patterns). Typical examples are in [legacy codebases and upgrades](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md); claim only what was true.
- Describe your safety habits: tracing a code path end to end before changing it, characterization tests around the behavior you touched, small pull requests, feature flags, watching logs and errors after deploys.
- Attribute the scale correctly: "the platform serves 700+ stores"; your scope is the features you shipped.

> **Fill in:** two or three enhancements besides PriceCharting (if any), and one thing about the codebase that surprised you.

> **Fill in:** how changes reached production (code review, CI, staging, release cadence).

### Bullet 2: "Integrated PriceCharting as a sales channel: background jobs sync listings, prices, stock, and orders."

**What it signals:** third-party integration, background jobs and data consistency. Expect the longest follow-up chain of the interview here.

**What they will ask:** the flow end to end; why jobs; how jobs are made safe to retry; rate limits; overselling; failures and monitoring; testing.

**How to answer:** walk the three directions (listings and prices out, stock out, orders in), then the safety mechanisms (idempotent upserts, checkpoints, backoff, reconciliation). Use the [design deep-dive](#design-deep-dive) as your script and say clearly which parts you built and which already existed.

> **Fill in:** the jobs you wrote and what each was responsible for (for example "a job per store that pulls orders every N minutes"), what triggered pushes, and how 429s and timeouts were handled.

> **Fill in:** what store owners see: a connect or settings page, a sync status, error messages?

Study: [background jobs](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md).

### Bullet 3: "Imported PriceCharting market prices into the catalog so stores can price inventory from market data."

**What it signals:** batch data import and a feature with direct business value (pricing).

**What they will ask:** how products were matched to PriceCharting products; how often prices refresh; how a market price becomes a store price; how you stop one bad data point from repricing a whole store; how big the import is and how you kept it fast.

**How to answer:** matching first (identifiers before fuzzy names, human review for low confidence), then the import (batched upserts keyed by external product ID and condition), then pricing (per-store rules with guardrails, a preview before auto-apply). For performance: batches, `upsert_all`, `find_each`, no per-row callbacks, and indexes on the keys you upsert by.

> **Fill in:** how products were matched, how often the import ran, whether stores apply prices manually or automatically, and which guardrails existed.

Study: [ActiveRecord advanced](../05-ruby-on-rails/06-activerecord-advanced.md) for `upsert_all` and batching, [e-commerce domain](../13-architecture-and-practices/07-ecommerce-domain.md) for pricing and inventory.

## Design deep-dive

**Prompt:** "How would you design the PriceCharting sync?" Treat it as a system design question about your own project: clarify, sketch, then go deep on correctness. Everything below is a typical design to adapt. Where your real system differs, say so and explain why it was built that way.

### 1. Clarify first
- How many stores enable the channel, and how many listings does a store have? (Drives batch sizes and fan-out.)
- How fresh must each field be? Stock needs seconds to minutes (oversell risk); prices can lag longer.
- What does the API offer: bulk endpoints, change feeds or webhooks, documented rate limits, a sandbox?
- Who owns each field? Usually the platform owns listing data and stock, and the channel owns its own orders.

### 2. The adapter boundary
Keep everything PriceCharting-specific (HTTP, auth, paging, field mapping, error codes) in one class with a small interface. Jobs and models talk to the interface, so tests can use a fake, a second channel can reuse the jobs, and an API change touches one file.

```ruby
# Illustrative: these method names are your own wrapper, not PriceCharting's API.
module SalesChannels
  class TransientError < StandardError; end      # timeouts, 5xx: safe to retry

  class RateLimited < StandardError
    attr_reader :retry_after

    def initialize(retry_after = nil)
      @retry_after = retry_after
      super("rate limited")
    end
  end

  class PriceChartingChannel
    def initialize(connection)                   # per-store credentials and settings
      @connection = connection
    end

    def upsert_listing(listing); end             # create or update remotely, return the external ID
    def update_quantity(listing, quantity); end
    def orders_since(time); end                  # remote orders, page by page
  end
end
```

### 3. Per-store jobs, then outbound: listings, prices and stock
- **Fan out per store:** a scheduled coordinator job enqueues one job per store with the channel enabled. One store's bad data, expired credentials or sheer size then cannot block the others, each store uses its own connection settings, and every job stays short enough to survive deploys and restarts.
- **Upsert remote mappings by external ID:** when the channel returns an ID for a listing, store it in `channel_listings` with a unique index on `(channel, external_id)` and write with an upsert, so a retried push updates the same row instead of creating a duplicate mapping.
- Trigger pushes from `after_commit` (never inside the transaction, or the job can run before the data is visible) and only when synced fields changed, for example `saved_change_to_price_cents?`.
- Watch for writes that skip callbacks (`update_all`, `upsert_all`, raw SQL). Legacy code often has them and they silently skip the push, so enqueue explicitly next to those writes.
- Send absolute values ("quantity is 3"), not deltas ("minus 1"). Replaying an absolute value is harmless; replaying a delta corrupts stock.
- Coalesce bursts: a bulk repricing of thousands of listings should become batched pushes, not thousands of jobs fired at once.
- A periodic sweep re-pushes listings whose `synced_at` is older than their last change, which repairs anything a lost job missed.

### 4. Inbound: orders, deduped

```ruby
# In a migration: the unique index is the real dedupe guarantee; code alone races.
add_index :channel_orders, %i[channel external_id], unique: true

class ImportChannelOrder
  def self.call(store, remote)
    ChannelOrder.transaction do
      # create_or_find_by! is Rails 6+; on older apps, rescue ActiveRecord::RecordNotUnique and find instead.
      order = ChannelOrder.create_or_find_by!(channel: "pricecharting", external_id: remote.id) do |o|
        o.store = store
        o.payload = remote.to_h
      end
      order.lock!                                # SELECT ... FOR UPDATE: a concurrent duplicate waits here
      next order if order.stock_applied?         # a retry or an overlapping fetch: already done

      oversold = remote.line_items.reject { |item| Inventory.take(item.listing_id, item.quantity) }
      order.update!(stock_applied: true, status: oversold.empty? ? "imported" : "oversold")
      order
    end
  end
end

module Inventory
  # Check and decrement in one statement, so two concurrent sales cannot both take the last unit.
  def self.take(listing_id, quantity)
    Listing.where(id: listing_id).where("quantity >= ?", quantity)
           .update_all(["quantity = quantity - ?", quantity]) == 1
  end
end
```

- Whether the unique key is `(channel, external_id)` or `(store_id, channel, external_id)` depends on whether the channel's order IDs are globally unique or only unique per seller account.
- The pull job keeps a checkpoint per store (`orders_synced_until`). Record the time before fetching, fetch with a small overlap, and advance the checkpoint only after the batch succeeds. The overlap re-reads a few orders; the unique index makes that harmless.
- Cancellations and refunds arrive later: handle them as state changes on the same row, found by the same external ID.
- `update_all` skips callbacks, so enqueue the quantity pushes for the affected listings explicitly once the transaction commits.

### 5. Overselling: one unit, two channels
The race: the last unit sells on the storefront and on PriceCharting within the same minute, before either side hears about the other.

| Defense | How | Trade-off |
|---|---|---|
| Single source of truth | The platform's inventory is authoritative; channels only receive quantities | Every sale from every channel must flow back quickly |
| Atomic decrement | Conditional `UPDATE ... WHERE quantity >= n` (above) or a row lock | Stops double-selling inside your database, not on the remote side |
| Fast quantity pushes | Push the new quantity right after each sale, on a high-priority queue | Bounded by API latency and rate limits |
| Frequent order pulls | A short polling interval, or webhooks if the provider offers them | More API calls |
| Safety buffer | List `quantity - buffer` on the channel for low-stock or high-demand items | Fewer units visible on the channel |
| Recovery path | Flag oversold orders, alert the store, cancel or refund through the channel | Manual work and customer impact, but explicit |

State the honest limit: with two independent systems you can shrink the window, not close it, so a good design also detects and handles the oversell that slips through.

### 6. Rate limits and backoff

```ruby
# Sketched with Active Job so it stays library-agnostic; adapt it to the codebase's job library.
class PushListingJob < ApplicationJob
  queue_as :channel_api                                  # a queue served by few threads

  retry_on SalesChannels::TransientError, wait: :polynomially_longer, attempts: 8
  discard_on ActiveJob::DeserializationError             # the listing was deleted meanwhile

  rescue_from SalesChannels::RateLimited do |error|
    raise error if executions >= 10                      # give up loudly after many tries
    retry_job wait: (error.retry_after || 60).to_i.seconds
  end

  def perform(listing)
    connection = listing.store.channel_connection
    return unless connection&.enabled?

    external_id = SalesChannels::PriceChartingChannel.new(connection).upsert_listing(listing)
    ChannelListing.upsert(                                # idempotent mapping row keyed by the external ID
      { listing_id: listing.id, channel: "pricecharting", external_id: external_id, synced_at: Time.current },
      unique_by: %i[channel external_id]                  # needs that unique index; skips callbacks, so no push loop
    )
  end
end
```

- Limit concurrency per provider: a dedicated queue with a small, fixed number of threads, or a shared token bucket in Redis when several processes call the same API.
- On HTTP 429, honor `Retry-After`; otherwise use exponential backoff with jitter so retries do not arrive in waves.
- Set explicit connect and read timeouts on the HTTP client; a hung request ties up a worker.
- Prefer bulk endpoints when they exist: one call for 100 listings beats 100 calls.
- `:polynomially_longer` is the Rails 7.1+ name; older apps use `:exponentially_longer`.

### 7. Product matching
Market prices and listings are only as good as the link between a catalog product and a PriceCharting product.
1. Match on identifiers first: barcodes (UPC or EAN) or a stored PriceCharting product ID.
2. Fall back to normalized attributes (title, platform, region, edition) with fuzzy matching, for example PostgreSQL's `pg_trgm` similarity, and record a confidence score.
3. Auto-link only high-confidence matches; send the rest to a review queue for the store or an admin.
4. Store each mapping with how it was matched and who confirmed it, so a wrong match is fixed once.

A wrong match is worse than no match: it pushes a wrong price onto a real listing.

### 8. Reconciliation and partial-failure alerting
- Treat each item separately: one bad listing records an error on its own row (`last_error`) and the batch continues.
- Record every run (`sync_runs`: started, finished, succeeded, failed) and alert on patterns, not single errors: a failure rate above a threshold, no successful sync for a store in N hours, a growing failed or dead jobs set.
- Show actionable errors to store owners ("missing barcode", "price below the channel minimum") so they can fix their own data.
- Run a daily reconciliation that compares local and remote state (quantities, prices, whether listings exist) where the API allows it, repairs drift, and reports what it fixed.

### 9. From market prices to pricing rules

```ruby
# Illustrative per-store rule; the values live in a pricing_rules row per store and condition.
def suggested_price_cents(market_cents:, current_cents:, rule:)
  target = [(market_cents * rule.multiplier).round, rule.min_price_cents].max
  return target if current_cents.to_i.zero?

  change = (target - current_cents).abs.fdiv(current_cents)
  change > rule.max_change ? nil : target      # nil = too big a jump: queue it for human review
end
```

- Import prices in batches keyed by `(external_product_id, condition)` with `upsert_all`, and keep history if stores want trends.
- Rules are per store, often per condition: multiplier, rounding, floor, ceiling.
- Guardrails: skip stale prices, cap the change per run, require review for large jumps, and let stores preview before switching on auto-apply.
- Repricing triggers outbound pushes, so batch them (section 3) or one import can flood the channel API.

### 10. Rollout and testing
- **Rollout:** a per-store feature flag, a few pilot stores first, a dry-run mode that computes pushes without sending them, and a kill switch per store and globally.
- **Tests:** unit-test the adapter against recorded or hand-written HTTP fixtures (WebMock or VCR are the usual gems); use a fake adapter in job tests; run every job twice in a test to prove idempotency; test the stock decrement with concurrent callers; try the full flow against a sandbox or test account if the provider offers one. See [testing with RSpec](../05-ruby-on-rails/13-testing-rspec.md).

### Going deeper
- **Transactional outbox:** write a "push needed" row in the same transaction as the business change and let a worker drain that table. No lost enqueues, even if the queue is down at commit time.
- **Fairness:** one huge store should not starve the others; cap concurrency per store or round-robin between stores.
- **Circuit breaker:** after repeated provider failures, stop calling for a while and alert, instead of filling the retry queue.
- **Backfills:** a store's first sync is a bulk job with its own pacing, kept separate from incremental syncs.

## Likely interview questions

**Q1. Explain the PriceCharting integration.**
Two minutes, top-down: what PriceCharting is (a price guide for video games and collectibles that also supports selling), why stores wanted it (another place to sell, plus market prices), the three flows (listings, prices and stock out; orders in; market prices into the catalog), and how the jobs stay safe (idempotent upserts keyed by external IDs, checkpoints, backoff, reconciliation). End with your exact part and one hard problem.
> **Fill in:** your part of the integration and the hardest problem you hit.

**Q2. How did you prevent overselling when an item sells on both channels?**
Name the race, then the layers: the platform's inventory as the single source of truth, an atomic decrement when an order is imported, an immediate quantity push to the channel, and a recovery path (flag, alert, cancel or refund) for the oversell that still slips through. Admit the window can be shrunk but not eliminated.
> **Fill in:** what the real flow was: where stock was decremented, how quickly channel quantities refreshed, and what happened when an oversell occurred.

**Q3. How did you handle API rate limits?**
Concurrency control (a dedicated queue with few threads, or a shared limiter), honoring `Retry-After` on 429, exponential backoff with jitter for transient errors, bulk endpoints where available, and spreading scheduled work instead of starting every store at the same minute.
> **Fill in:** the provider's actual limits as you knew them, and what your jobs did when they hit one.

**Q4. Did you poll or use webhooks, and why?**
Polling is simple, works when the provider has no webhooks, and is easy to resume from a checkpoint, but it adds latency and API calls. Webhooks are near real time but can be lost, duplicated or reordered, so they still need signature checks, idempotent handlers and a reconciliation poll as a backstop. Many production integrations use both.
> **Fill in:** which one your sync used for orders and for price data.

**Q5. What was the source of truth for stock?**
The answer should be one system, usually the platform, because it sees sales from every channel. Channels receive absolute quantities; they never own them.
> **Fill in:** confirm the real owner, and whether any channel could change quantities on its own.

**Q6. How did you make the jobs safe to retry?**
Jobs can run more than once (retries, crashes, duplicate enqueues), so each one: passes IDs instead of objects, reloads state, upserts by external ID behind a unique index, writes absolute values, and checks "already done" flags under a lock. Then prove it with a test that runs the job twice.

**Q7. How do you match catalog products to PriceCharting products?**
Identifiers first (barcode or a stored external ID), fuzzy attribute matching second with a confidence score, human review for anything uncertain, and a stored mapping table so the decision is made once.
> **Fill in:** how matching actually worked in your import.

**Q8. How do market prices reach a store's listings?**
A recurring import upserts market prices per product and condition; a per-store rule (multiplier, floor, rounding) turns them into suggested prices; guardrails stop stale data and big jumps; changed prices flow out through the normal outbound sync.
> **Fill in:** whether stores accepted suggestions manually or prices applied automatically.

**Q9. What happens if PriceCharting is down for a day?**
Jobs fail with transient errors and back off; a circuit breaker or simple health flag stops hammering the API; checkpoints mean nothing is lost, and the next successful run resumes from them; stale-sync alerts tell the team; the oversell risk rises, so a store could pause channel listings for its last units. Reconciliation repairs drift afterwards.

**Q10. How did you test the integration?**
Adapter tests against HTTP fixtures, a fake adapter for job tests, idempotency tests (run twice, assert one order and one decrement), a concurrency test for the decrement, and a manual run in staging or against a test account.
> **Fill in:** which of these you really had, and what was tested manually.

**Q11. What made the legacy code hard, and how did you work with it safely?**
Be specific and fair: old patterns, implicit coupling through callbacks, missing tests, and code whose reason nobody remembers. Your method: read the path end to end, use `git log` and `git blame` to recover intent, add characterization tests, change in small steps, and isolate new code behind clear boundaries. Never insult the codebase or the people who built it.
> **Fill in:** one concrete example of something hard and how you handled it.

**Q12. How did you roll it out without risking 700+ stores?**
Feature flags per store, pilot stores first, dry-run mode, monitoring of sync runs and errors, and a kill switch.
> **Fill in:** how the rollout really happened.

**Q13. Why background jobs instead of calling the API in the request?**
Third-party calls are slow and unreliable, syncs touch many records, and users should not wait for them. Jobs add retries, scheduling, concurrency control and isolation from the provider's outages, at the cost of eventual consistency and more moving parts.

**Q14. How would the design change if the number of stores doubled?**
More fan-out and worker capacity, per-store fairness, bulk endpoints, smarter scheduling (only stores with changes), and a shared rate limiter so more workers do not mean more 429s. The database side: indexes on sync keys, batched upserts, and archiving old sync logs.

**Q15. What would you improve with more time?**
Pick real gaps you saw, such as an outbox for reliable enqueues, a sync dashboard for store owners, better reconciliation, or more tests around the legacy paths you touched.
> **Fill in:** two improvements you would genuinely make, and why.

**Q16. Tell me about a bug you hit in this project.**
Answer in STAR form (see [the story bank](../18-behavioral-and-hr/02-star-stories-bank.md)): the symptom, how you investigated (logs, job history, reproducing with real data shapes), the root cause, the fix, and what you added so it cannot silently return.
> **Fill in:** a real bug, its root cause, and the fix.

## Fill-in checklist
- [ ] Job library, Rails version and Ruby version of the codebase.
- [ ] Whether you are still on the project (use the right tense: "I work on" vs "I worked on").
- [ ] Your exact part of the PriceCharting integration vs the team's, and who you worked with.
- [ ] Which PriceCharting capabilities were used and how stores connected their accounts.
- [ ] What triggered each job and how often recurring jobs ran.
- [ ] Polling or webhooks for orders; how checkpoints worked.
- [ ] How rate limits, timeouts and retries were handled.
- [ ] The source of truth for stock and what happened on an oversell.
- [ ] How products were matched to PriceCharting products.
- [ ] How market prices became store prices, and any guardrails.
- [ ] How the integration was tested and rolled out.
- [ ] Other enhancements you made to the legacy platform.
- [ ] One real bug story and one thing you would improve.
- [ ] Where the 700+ stores and 8M+ customers figures come from, so you can answer "how do you know?"
