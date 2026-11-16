# Architecture and Practices: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## MVC and layered architecture (see [MVC and layered architecture](01-mvc-and-layered-architecture.md))

**Q1. What is MVC, and how does a Rails request flow through it?**
The model holds data and domain rules, the view handles presentation, and the controller handles the request and coordinates the other two. In Rails, Rack middleware runs first, the router maps the request to a controller action, the action loads or changes models (Active Record talks to the database), and then it renders a view (ERB or JSON) or redirects.

**Q2. "Fat model, skinny controller": is it still good advice?**
Thin controllers, yes. But pushing everything into models produces 2,000-line God objects held together by callbacks. Keep models for persistence-level rules (validations, associations, scopes, small domain methods) and move workflows into service objects, form objects, query objects and jobs.

**Q3. Where does business logic go in a Rails app?**
Invariants of a single model: the model. Multi-step workflows across models or external APIs: service objects (`Orders::Checkout.call`). Forms that do not map to one table: form objects built on Active Model. Complex reusable queries: scopes or query objects. Presentation logic: helpers, presenters or view components. Slow side effects: background jobs.

**Q4. What problems do Active Record callbacks cause?**
Hidden control flow (saving a user sends emails and charges cards), order dependencies, side effects in tests and seeds, callbacks silently skipped by `update_column` and bulk operations, and external calls made inside transactions. Keep callbacks for normalizing the record's own data, trigger side effects explicitly from services, and enqueue jobs after commit.

**Q5. What is a layered architecture?**
Code split into layers with one-way dependencies: presentation (controllers, views, API handlers), application (use cases and services), domain (entities and business rules) and infrastructure (database, HTTP clients, queues). Upper layers call lower ones, and the domain does not depend on the web framework.

**Q6. MVC vs MVP vs MVVM?**
All three separate UI from logic. In MVC, the controller handles input and chooses a view. In MVP, a presenter drives a passive view through an interface (common in older Android apps). In MVVM, the view binds to a view model that exposes state and commands (WPF, Knockout); React with hooks resembles it, with a custom hook acting as the view model.

**Q7. Monolith vs microservices?**
A monolith is one deployable unit: simple development, real transactions, easy refactoring and debugging; it struggles when many teams collide or parts need very different scaling. Microservices give independent deploys and scaling per service, but bring network failures, distributed data, eventual consistency, and far more observability and operations work. Most teams should start with a well-structured monolith.

**Q8. What is a modular monolith?**
One deployable application split into modules with explicit boundaries and public interfaces (Rails engines, packwerk packages, feature folders in TypeScript), so code stays decoupled without network calls between parts. It captures most of the benefits of microservices at a fraction of the cost and makes a later extraction easier.

**Q9. When would you extract a service from a monolith?**
When a part has a clearly separate domain and data, a different scaling or reliability profile (image processing, a high-volume webhook receiver), its own team, or a genuinely different technology need. Not because the codebase feels messy: fix modularity inside the monolith first.

## Clean architecture and dependency injection (see [Clean architecture](02-clean-architecture.md))

**Q10. What is the dependency rule in clean architecture?**
Source code dependencies point inward: entities (enterprise rules) at the center, then use cases, then interface adapters (controllers, presenters, repositories), then frameworks and drivers (web, database, UI). Inner layers define interfaces that outer layers implement, so business rules never import Rails, Express or the ORM.

**Q11. What is hexagonal architecture (ports and adapters)?**
The application core declares ports (interfaces) for what it offers and what it needs: driving ports for incoming calls (HTTP, CLI, jobs) and driven ports for outgoing ones (database, payment gateway, email). Adapters implement those ports for specific technologies, so you can swap Postgres for an in-memory fake in tests or one payment provider for another.

**Q12. Clean vs hexagonal vs onion architecture?**
Variations of one idea: keep the domain at the center, independent of frameworks, with dependencies pointing inward. Hexagonal emphasizes ports and adapters at the boundary; onion and clean emphasize concentric layers. In an interview, explain the shared principle rather than the differences between diagrams.

**Q13. What is dependency injection, and what forms does it take?**
Giving an object its collaborators instead of letting it construct them. Constructor injection is the most common and explicit; method (parameter) and setter injection also exist. Containers (NestJS, ASP.NET Core, Spring) wire object graphs automatically; in Ruby and TypeScript a default argument is often enough: `def initialize(client: PriceClient.new)`.

**Q14. What does clean architecture cost?**
More files, interfaces and mapping code (entity to model to DTO), slower feature work, and indirection that confuses newcomers. It pays off for complex, long-lived domains with several delivery channels; for CRUD-heavy apps, pragmatic layering (service objects plus Active Record) is usually enough.

**Q15. How do you apply clean architecture ideas pragmatically in Rails or TypeScript?**
Keep controllers and route handlers thin; put use cases in plain service objects or functions that receive their dependencies (API clients, repositories, a clock); wrap third-party APIs in adapters you own; keep framework types out of domain code; and test use cases with fakes. You do not need every layer to get most of the benefit.

**Q16. Entities vs value objects vs aggregates?**
An entity has an identity that survives changes (an `Order` with id 42). A value object is defined only by its values and is immutable (`Money(1000, "EUR")`, an address). An aggregate is a cluster of objects changed together through one root that enforces the invariants (an order and its line items); a transaction should not span several aggregates.

**Q17. What is a bounded context?**
A boundary within which a model and its terms have one consistent meaning. "Product" in the catalog (descriptions, images) is not the same thing as "product" in inventory (stock per location) or billing; separate contexts keep each model simple and make natural module or service boundaries.

## Multi-tenancy (see [Multi-tenancy](03-multi-tenancy.md))

**Q18. What is multi-tenancy?**
One application serving many customers (tenants, such as organizations or stores), each seeing only its own data and configuration. The core challenges are isolation (no cross-tenant leaks), per-tenant customization, fair resource use, and operating many tenants from one codebase.

**Q19. Compare the three main multi-tenancy data models.**
Shared database and shared schema, with a `tenant_id` on every table: cheapest, one migration run, easy cross-tenant reporting, but isolation depends on every query being scoped. Schema per tenant (Postgres schemas): stronger isolation and per-tenant restores, but migrations run once per tenant and thousands of schemas strain tooling. Database per tenant: the strongest isolation, per-tenant scaling and data residency, but the highest cost and operational load. Many SaaS products use a shared schema and move a few very large customers to dedicated databases.

**Q20. How do you resolve the current tenant for a request?**
From the subdomain (`acme.app.com`), a custom-domain lookup, a path prefix (`/orgs/acme/...`), a header or a claim in the auth token, or the organization the user selected (stored in the session). Resolve it once in middleware, verify the user belongs to it, and keep it in request-scoped context: Rails `CurrentAttributes`, `AsyncLocalStorage` in Node, or the context object of an RPC framework such as oRPC.

**Q21. How do you enforce tenant isolation in a shared schema?**
Scope every query through the tenant (`current_org.projects`), centralize the scoping (the acts_as_tenant gem, a base repository, or procedure middleware that injects `org_id`), add Postgres Row Level Security as a safety net, include `tenant_id` in unique constraints and indexes, and write tests that try to read and modify another tenant's records.

**Q22. Why are Rails default scopes risky for tenancy?**
They are implicit, easily bypassed with `unscoped`, behave surprisingly with joins and `new`, and raw SQL ignores them entirely. A tenancy gem or explicit scoping through the tenant association is clearer, and Row Level Security catches what slips through.

**Q23. How does Postgres Row Level Security help?**
You enable RLS on a table and create a policy such as `USING (org_id = current_setting('app.current_org')::uuid)`; the database then filters every query, including raw SQL, for roles subject to the policy. Set the variable per transaction with `SET LOCAL`, which matters with poolers like PgBouncer in transaction mode. Superusers and roles with `BYPASSRLS` always bypass policies, and table owners do too unless you add `FORCE ROW LEVEL SECURITY`, so connect as a dedicated non-owner role.

**Q24. What changes in unique constraints and indexes in a multi-tenant schema?**
Uniqueness is usually per tenant: `UNIQUE (org_id, slug)` instead of `UNIQUE (slug)`. Composite indexes should usually start with `org_id`, for example `(org_id, created_at)`, because nearly every query filters by tenant.

**Q25. What besides web requests must carry the tenant context?**
Background jobs (pass the tenant id and set the context at the start of the job), cache keys, file storage paths, search indexes, WebSocket channels, emails (branding and URLs), log and metric tags, and rate limits. A job that forgets its tenant context is a classic cross-tenant bug.

**Q26. What is the noisy neighbor problem?**
One large tenant consumes shared resources (database CPU, job queues, third-party API quotas) and slows everyone else down. Mitigate with per-tenant rate limits and quotas, fair queueing or per-tenant queues, query timeouts, and dedicated resources for the biggest tenants.

**Q27. How do you handle per-tenant configuration and features?**
Store settings and feature flags per tenant (a settings table or a validated JSONB column with defaults), read them through the tenant context, and never branch on tenant names in code (`if org.slug == "acme"`). Plan for per-tenant branding, locale, time zone and billing plan from the start.

**Q28. How do you test tenant isolation?**
Create two tenants in tests and assert that every endpoint and query running as tenant A cannot read or change tenant B's records (expect 404 or 403), including list endpoints, search, exports and background jobs. Database-level RLS or a query guard adds defense in depth.

## Platform migrations (see [Platform migration strategies](04-platform-migration-strategies.md))

**Q29. Big-bang rewrite vs incremental migration?**
A big-bang rewrite freezes features for months and then switches everything at once: high risk, late feedback, and the old system's undocumented behavior gets lost. An incremental migration moves one route, feature or tenant at a time while both systems run: value arrives early and rollback stays possible, at the cost of running two systems and keeping data in sync.

**Q30. What is the strangler fig pattern?**
Put a routing layer (a proxy, the load balancer, or the old app itself) in front of the legacy system, then move functionality piece by piece into the new system and route those paths to it, until the old system serves nothing and can be retired. It is named after a vine that grows around a host tree and eventually replaces it.

**Q31. What is an anti-corruption layer?**
A translation layer between the new system and the legacy one, so legacy concepts, naming and quirks do not leak into the new domain model. Sync APIs that map old records to new entities are a typical example.

**Q32. How do you keep data in sync between the old and new systems?**
First pick one source of truth per entity for each stage of the migration. Then run an initial backfill (batched, resumable, idempotent upserts keyed by the legacy id), an incremental sync (an `updated_at` watermark, change data capture, or events from the old system), an id mapping table (legacy id to new id), and reconciliation jobs that compare counts and checksums and report drift.

**Q33. What is the dual-write problem?**
Writing the same change to two systems (or to a database and a message broker) from application code: if the second write fails or the process crashes in between, the systems diverge, and no transaction spans both. Avoid it with one source of truth plus asynchronous propagation (a transactional outbox or change data capture) and idempotent consumers, and reconcile regularly.

**Q34. How do you migrate users without forcing everyone to log in again?**
Share session or token verification between the systems (a shared cookie domain and signing key, or one identity provider whose tokens both trust), or migrate on first login by verifying against the old password hash and rehashing in the new system. Move password hashes as they are; never decrypt or reset them in bulk.

**Q35. How do you protect SEO during a web platform migration?**
Keep URLs where possible, and otherwise map every old URL to its new one with a single 301 (no redirect chains). Preserve titles, meta descriptions, canonical tags, structured data and internal links, serve server-rendered HTML for pages that must be crawled, return real 404 and 410 codes rather than soft 404s, update the sitemap, and watch Search Console coverage and rankings after each cutover.

**Q36. How do you roll out a migrated page safely?**
Behind a feature flag or routing rule: internal users first, then a percentage of traffic or a few tenants (a canary), comparing error rates, latency and business metrics against the old version, with a one-switch rollback. For read-only endpoints, shadow traffic (sending copies of real requests to the new system and comparing responses) catches differences before any user sees them.

**Q37. What is the expand-and-contract (parallel change) pattern?**
A way to change a schema or API without breaking running code: expand (add the new column or field next to the old one), migrate (write both, backfill, switch reads), then contract (remove the old one) once nothing uses it. Every step is deployable and reversible on its own.

**Q38. How do you know a migration is finished?**
Agreed exit criteria: every route and job moved, traffic to the old system at zero for a set period, clean data reconciliation, monitoring and alerts in place on the new side, updated runbooks, and finally decommissioning with an archived copy of the data. Without explicit criteria, two systems tend to run forever.

## Third-party API integration (see [Third-party API integration](05-third-party-api-integration.md))

**Q39. How do you structure code that talks to a third-party API?**
Wrap it in a client class or module you own (an adapter) that handles authentication, base URL, timeouts, retries, error mapping, logging and rate limiting, and expose domain methods such as `fetch_prices(ids)` to the rest of the app. The vendor stays isolated, tests stub one class, and switching providers becomes a local change.

**Q40. How do you handle an API's rate limits?**
Read the documented limits and rate-limit response headers, throttle on your side (a token bucket shared across workers, often in Redis, or a queue with limited concurrency), back off on 429 using `Retry-After`, batch requests where the API allows it, and cache data that rarely changes. Spread scheduled syncs out instead of firing everything at midnight.

**Q41. Which failures should you retry, and how?**
Retry transient failures: timeouts, connection errors, 429, 502, 503, 504, and often 500. Do not retry 400, 401, 403, 404 or 422, which will fail again. Use exponential backoff with jitter and a maximum number of attempts, and retry only operations that are idempotent or protected by idempotency keys.

**Q42. Why do timeouts matter so much?**
Without one, a hung API holds a Puma thread or a job worker indefinitely; soon every thread is stuck and one slow dependency becomes your outage. Set connect and read timeouts on every HTTP client (a few seconds for user-facing calls), and move slow integrations into background jobs.

**Q43. What is a circuit breaker?**
A wrapper that counts failures to a dependency. After a threshold it "opens" and fails fast for a cool-down period instead of waiting on timeouts, then lets a trial request through ("half-open") and closes again if it succeeds. It protects your threads and gives the provider time to recover.

**Q44. What is idempotency, and why does it matter for integrations?**
An operation is idempotent if performing it twice has the same effect as performing it once. Retries, at-least-once job delivery and webhook redeliveries all create duplicates, so every side effect must tolerate repetition: upsert by external id instead of inserting, record processed event ids, send idempotency keys with create requests, and let unique constraints be the final guard.

**Q45. Webhooks vs polling?**
Webhooks push events as they happen (low latency, little wasted traffic) but can arrive late, out of order, duplicated or not at all, and they need a public endpoint. Polling is simple and fully under your control but slower and more expensive. Robust integrations use webhooks for speed plus a periodic reconciliation poll to catch anything missed.

**Q46. How do you verify a webhook?**
Compute an HMAC (usually SHA-256) of the raw request body with the shared signing secret and compare it with the signature header using a constant-time comparison; reject events whose signed timestamp is too old (for example, more than five minutes) to stop replays; and always use the raw bytes, never re-serialized JSON. Provider SDKs wrap this, such as `Stripe::Webhook.construct_event` in Ruby.

**Q47. How should a webhook endpoint process events?**
Verify the signature, store the event (id, type, raw payload) with a unique constraint on the event id, respond 2xx quickly, and do the real work in a background job. Handle out-of-order delivery by comparing timestamps or versions, or by re-fetching the current state of the resource from the API instead of trusting the payload.

**Q48. How do you test code that calls external APIs?**
Unit-test your client against stubbed HTTP (WebMock or VCR cassettes in Ruby, MSW or nock in Node), test business logic against a fake client, keep a few smoke or contract tests against the provider's sandbox, and cover the failure paths: timeouts, 429, 500 and malformed payloads.

**Q49. How do you map external data into your models?**
Store the external id with a unique index such as `(channel, external_id)`, translate fields and enums inside the adapter, validate and normalize at the boundary (currencies, time zones, decimals), keep the raw payload for debugging when it is cheap, and never let the external schema leak into your domain.

**Q50. How do you keep a price or inventory sync with an external channel reliable?**
Make sync jobs idempotent upserts keyed by external id, process changes incrementally (everything changed since the last successful watermark) with a periodic full reconciliation, respect rate limits with throttled batches, record failures per item instead of failing the whole batch, alert on drift, and make ownership explicit: which system is the source of truth for price, stock and order status.

## Background processing patterns (see [Background processing patterns](06-background-processing-patterns.md))

**Q51. Why move work into background jobs?**
To keep requests fast (emails, PDFs, API calls, imports), to retry failures automatically, to absorb traffic spikes through a queue, and to run scheduled work. The request records the intent and returns; workers do the rest.

**Q52. At-most-once vs at-least-once vs exactly-once delivery?**
At-most-once: a message may be lost but is never processed twice. At-least-once: never lost, but possibly processed more than once (Sidekiq, standard SQS queues, most brokers). Exactly-once delivery cannot be guaranteed end to end over unreliable networks; you get exactly-once effects by combining at-least-once delivery with idempotent processing.

**Q53. How do you make a job idempotent?**
Pass ids, reload the current state and check it before acting (skip if the order is already `paid`), use upserts and unique constraints, record processed message ids, send idempotency keys to external APIs, and make multi-step jobs resumable by recording their progress.

**Q54. What goes wrong if you enqueue a job inside a database transaction?**
The job can run before the transaction commits and find no record (`RecordNotFound`) or stale data; if the transaction rolls back, the job runs for something that never existed. Enqueue after commit: use `after_commit` callbacks, or the enqueue-after-commit support in recent Active Job versions (check the default for your Rails version).

**Q55. What is the transactional outbox pattern?**
Instead of writing to the database and publishing to a broker separately (the dual-write problem), you insert an outbox row describing the event in the same transaction as the business change. A relay process polls the outbox (or reads the database log through change data capture), publishes each event and marks it sent. No event is lost, delivery is at-least-once, and consumers deduplicate by event id.

**Q56. What is a dead letter queue?**
The place where messages go after exhausting their retries, so a poison message stops burning resources and a human can inspect, fix and replay it. Sidekiq's equivalent is the Dead set, reached after the default 25 retries (roughly three weeks of exponential backoff); SQS and RabbitMQ use a configured dead letter queue. Alert when it grows.

**Q57. What should job arguments look like?**
Small, JSON-serializable primitives: ids and simple values, not objects. Sidekiq requires plain JSON types; Active Job can serialize records through GlobalID, but the record may be gone by the time the job runs. Reload fresh data inside the job, and keep arguments backward compatible across deploys, because jobs enqueued by old code are still waiting in the queue.

**Q58. How do you prioritize and isolate jobs?**
Separate queues by urgency and workload (critical, default, low, imports), give queues weights or dedicated processes, cap concurrency for jobs that hit rate-limited APIs or heavy database work, and keep long-running jobs away from latency-sensitive ones.

**Q59. How do you run recurring jobs?**
With a scheduler: sidekiq-cron or Sidekiq Enterprise periodic jobs, Solid Queue recurring tasks in Rails 8, or system cron or the platform's scheduler triggering a task. Make them idempotent and guard against overlap (a lock, or skipping when the previous run is still going), because deploys and retries cause double runs.

**Q60. How do you process a million records in the background?**
Split the work: a coordinator job enqueues batches by id range (`find_each` or `in_batches`), each batch job is idempotent and finishes in seconds to minutes, progress is recorded, and failures retry per batch. Throttle so the database and external APIs survive, and run it off-peak.

**Q61. What happens to running jobs during a deploy?**
Workers receive SIGTERM, stop fetching new jobs, give running jobs a grace period, and push unfinished ones back to the queue, where they run again later, possibly on the new code. So jobs must be safe to re-run, and their arguments must be understood by both old and new code.

**Q62. What is the saga pattern?**
A way to run a business process across services or steps without a distributed transaction: each step commits locally, and if a later step fails, compensating actions undo the earlier ones (refund the payment, release the stock). Orchestrated sagas have a coordinator; choreographed sagas react to each other's events.

**Q63. Job queue vs message broker vs event log?**
Job queues (Sidekiq, Solid Queue) run units of work with retries for your own workers. Message brokers (RabbitMQ, SQS) route messages between producers and consumers with acknowledgements. Event logs (Kafka) keep an ordered, replayable stream that many consumer groups read at their own pace. Ordering, replay and fan-out needs decide which one fits.

**Q64. Which metrics tell you a job system is healthy?**
Queue latency (the age of the oldest waiting job, the best single signal), queue size, throughput, failure and retry rates, growth of the dead set, job duration percentiles and worker saturation. Alert on latency for critical queues, not only on size.

## E-commerce domain (see [E-commerce domain](07-ecommerce-domain.md))

**Q65. What are the core entities of an e-commerce system?**
Products and variants (SKUs with options such as size or condition), prices, inventory per location or channel, customers, carts, orders and order line items, payments, shipments, returns and refunds, plus categories and promotions.

**Q66. Why snapshot prices on order line items?**
Product prices change, but an order must keep what the customer actually paid. Store the unit price, quantity, discounts, tax and currency on each line item at checkout instead of joining to the current product price.

**Q67. How do you prevent overselling?**
Make the stock decrement atomic and conditional, `UPDATE variants SET stock = stock - 1 WHERE id = $1 AND stock >= 1`, and check that exactly one row changed. Alternatives: lock the row inside the checkout transaction (`SELECT ... FOR UPDATE`, Rails `with_lock`), or optimistic locking (`lock_version`) with a retry. Add a `CHECK (stock >= 0)` constraint as the last guard, and for high-demand items reserve stock for a limited time during checkout.

**Q68. Optimistic vs pessimistic locking for inventory?**
Pessimistic locking (`FOR UPDATE`) blocks other writers until commit: safe and simple, but hot items create lock contention. Optimistic locking (`lock_version`) lets everyone read and makes the later writer fail with a conflict and retry: great when conflicts are rare, poor during a flash sale. The atomic conditional update is often the simplest of all.

**Q69. How do you handle inventory across several sales channels?**
Keep one source of truth for stock, push changes to channels quickly (events or frequent syncs) and pull channel orders back promptly, because sync lag is exactly when overselling happens. Mitigate with safety buffers (list slightly fewer units than you have), reservations, and fast cancellation and customer notification when a race is lost anyway.

**Q70. How should you store money?**
As integers in the smallest currency unit (cents) or as exact decimals (`numeric(12,2)` in Postgres, `BigDecimal` in Ruby, `Decimal` in Python), always together with a currency code; never as floats. Define rounding explicitly (per line or per order, half-up or half-even) and allocate remainders when splitting amounts.

**Q71. How do you split an amount evenly without losing cents?**
Work in minor units and hand the remainder out one unit at a time: 1,000 cents split three ways is 334, 333 and 333. Libraries do this for you (the `money` gem's `allocate`, Dinero.js); dividing floats and rounding loses or invents cents.

**Q72. What does an order state machine look like?**
States such as `pending`, `paid`, `fulfilled` (or `shipped`), `delivered`, `cancelled` and `refunded`, with explicit allowed transitions: you cannot ship an unpaid order or cancel a delivered one. Enforce transitions in one place (a state machine library or model methods with guards), record the history, and make transitions idempotent so replayed webhooks are harmless.

**Q73. How do you make checkout safe against double submission?**
Disable the button on the client, but rely on the server: an idempotency key per checkout attempt (or a unique constraint on the cart id for orders), order creation inside a transaction, and provider idempotency keys on the payment request.

**Q74. How do you handle a flash sale with thousands of buyers for 100 items?**
Avoid lock contention on one hot row: decrement a counter atomically in Redis (`DECR`, rejecting the buyer when it goes below zero) or split the stock into buckets, queue the orders, hold short-lived reservations ahead of payment, rate limit (or use a waiting room), and reconcile the counter with the database afterwards.

**Q75. How would you let stores price items from market data?**
Import market prices on a schedule into your own table (with source, condition and timestamp), let each store define rules (a percentage of market price, rounding, floors and ceilings), compute prices in background jobs, log every change for audit, and add sanity checks against wild swings (ignore outliers, require manual review above a threshold).

## Payments and subscriptions (see [Payments and subscriptions](08-payments-and-subscriptions.md))

**Q76. Walk through a typical card payment flow with a provider like Stripe.**
The server creates a payment (a PaymentIntent with amount and currency, sent with an idempotency key); the client confirms it with the provider's SDK, which collects the card details and handles 3-D Secure; the provider sends a webhook (`payment_intent.succeeded`), and the server marks the order paid when it processes that webhook. The client-side success redirect is only for the user's benefit, never the source of truth.

**Q77. Why should card data never touch your servers?**
PCI DSS scope: handling raw card numbers brings heavy audits and controls. Hosted fields, hosted checkout pages and mobile SDKs send card data straight to the provider and give you back a token or payment method id, which keeps you in the lightest compliance category.

**Q78. Why are webhooks the source of truth for payments?**
Payments complete asynchronously (3-D Secure, bank transfers, retries), users close the tab before redirects finish, and statuses change later (refunds, disputes). The provider's signed events are the reliable record; your handler verifies them, processes them idempotently and updates local state.

**Q79. How do you model subscriptions?**
Plans and prices (amount, currency, interval), plus a subscription per customer with a status (`trialing`, `active`, `past_due`, `canceled`), the current period's start and end, and the provider's ids. Derive entitlements (what the user may access) from status and plan, and keep local state in sync from webhooks such as `customer.subscription.updated`, `invoice.paid` and `invoice.payment_failed`.

**Q80. What is dunning?**
The process after a renewal payment fails: retry the charge on a schedule, email the customer to update their card, keep access during a grace period, then downgrade or cancel. Drive each step from webhook events and log it.

**Q81. How do proration and plan changes work?**
An upgrade mid-cycle usually charges the price difference for the remaining period, immediately or on the next invoice; a downgrade often takes effect at the end of the period to avoid refunds. Providers compute proration for you; your job is to choose the policy and show the customer the amount before they confirm.

**Q82. Cancel immediately or at period end?**
At period end is the common default: the customer keeps what they paid for, and the cancellation can be undone before renewal (Stripe models it with `cancel_at_period_end`). Immediate cancellation may involve a prorated refund. Either way, update entitlements from the webhook, not from the button click.

**Q83. Refund vs chargeback?**
A refund is initiated by the merchant (full or partial) and returns money through the provider. A chargeback (dispute) is initiated by the cardholder's bank: the funds are pulled immediately with a fee, and the merchant can submit evidence. Track both against the order and adjust inventory, entitlements and accounting.

**Q84. How would you model split bills between roommates?**
Store each expense with the payer, the amount in minor units, the currency, and each participant's share (equal, exact or percentage, allocated without losing cents). Each person's balance is what they paid minus what they owe, derived from the records rather than stored; settle-up payments are just more entries. An append-only ledger, never balances updated in place, keeps the history auditable.

**Q85. What is double-entry bookkeeping, and why keep a ledger?**
Every money movement is recorded as balanced debit and credit entries across accounts, so the entries always sum to zero and balances are derived from immutable records. It gives you an audit trail, makes bugs visible as imbalances, and supports reconciliation against provider payout reports.

**Q86. In-app purchases vs regular payment providers on mobile?**
App stores generally require their own in-app purchase systems for digital goods and content used inside the app, while physical goods and real-world services (rent, deliveries) go through normal payment providers. These rules vary by region and change over time, so check the current App Store and Google Play policies; services like RevenueCat unify receipts and entitlements across both stores.

## Testing fundamentals (see [Testing fundamentals](09-testing-fundamentals.md))

**Q87. What is the testing pyramid?**
Many fast, isolated unit tests at the bottom, fewer integration tests in the middle (several components together, a real database), and a few slow end-to-end tests at the top (a real browser through the full stack). It balances confidence against speed and flakiness. The "testing trophy" variant puts the most weight on integration tests, which suits frontend code.

**Q88. Unit vs integration vs end-to-end tests?**
Unit tests check one function or class in isolation, run in milliseconds and pinpoint failures. Integration tests check several parts together (a request spec through the router, controller, models and database) and catch wiring bugs. End-to-end tests drive a real user flow (Capybara system tests, Playwright, Cypress, or Detox and Maestro on mobile): the most confidence, but the slowest and flakiest.

**Q89. Dummy vs stub vs spy vs mock vs fake?**
A dummy is passed but never used. A stub returns canned answers (`allow(client).to receive(:price).and_return(10)`). A spy records calls so you can assert on them afterwards. A mock is set up with expectations that fail the test if they are not met (`expect(mailer).to receive(:deliver)`). A fake is a working lightweight implementation (an in-memory repository, a fake payment gateway).

**Q90. What is wrong with over-mocking?**
The tests mirror the implementation: they pass while the real integration is broken and fail on harmless refactors. Mock at boundaries you do not control (HTTP, time, randomness, third-party services), use real objects and a real test database inside, and assert on outcomes rather than on which methods were called.

**Q91. What is TDD?**
Write a failing test (red), write the minimum code to pass it (green), clean up (refactor), and repeat. It pushes you toward small interfaces and leaves a safety net, and it works best for logic with clear inputs and outputs and for bug fixes (reproduce first). Be honest in interviews about how much you actually practice it.

**Q92. What makes a good unit test?**
It is fast, isolated and deterministic, checks one behavior, has a descriptive name, follows Arrange-Act-Assert, asserts on observable behavior, and does not depend on test order. When it fails, it should tell you what broke without a debugger.

**Q93. What causes flaky tests, and how do you fix them?**
Async timing (fixed sleeps, not waiting for the UI to update), shared state between tests, order dependence, real clocks and time zones, randomness, real network calls, and genuine race conditions in the app. Fix them with proper waiting (Capybara's waiting matchers, `findBy` queries in Testing Library), frozen time, seeded randomness, isolated data and a stubbed network. Quarantine and fix flaky tests quickly; never just retry forever.

**Q94. Is 100% code coverage a good goal?**
No. Coverage shows what ran, not what was verified, and a 100% target invites low-value tests of trivial code. Use coverage to find untested critical paths, prefer branch coverage for business logic, and watch the trend rather than a hard number. Mutation testing measures test quality more directly.

**Q95. Factories vs fixtures?**
Fixtures (YAML files in Rails) load one fixed dataset: very fast, but shared and implicit. Factories (FactoryBot) build exactly what each test needs: explicit and flexible, but slower, and associations can quietly create large object graphs. Use `build` or `build_stubbed` instead of `create` when the test does not need the database.

**Q96. How do you test a bug fix?**
Write a test that reproduces the bug and fails, then fix the code and watch it pass. The test documents the bug and prevents a regression; name it after the behavior, not the ticket number.

**Q97. What are contract tests?**
Tests that check a consumer and a provider agree on an API: the consumer records its expectations (Pact) or both sides validate against a shared schema (OpenAPI, a GraphQL schema, shared Zod contracts in a monorepo). They catch breaking API changes without running both systems end to end.

**Q98. How do you test time-dependent code?**
Inject a clock or freeze time (Rails `travel_to` and `freeze_time`; Jest or Vitest fake timers with `setSystemTime`), test boundaries such as month ends, daylight saving changes and leap years, and store times in UTC.

## Code quality and Agile (see [Code quality and Agile](10-code-quality-and-agile.md))

**Q99. Name some common code smells.**
Long methods, large classes, long parameter lists, duplicated logic, feature envy (a method more interested in another class's data), shotgun surgery (one change touches many files), primitive obsession (plain strings for money or status), deeply nested conditionals, and comments that explain confusing code instead of the code being made clear.

**Q100. How do you refactor legacy code safely?**
Add characterization tests that pin the current behavior first, refactor in small behavior-preserving steps, commit often, keep refactoring separate from feature changes in pull requests, use feature flags for risky swaps, and watch monitoring after the deploy. The Boy Scout rule (leave the code a little better than you found it) keeps it incremental.

**Q101. What is technical debt, and how do you argue for paying it down?**
Shortcuts and outdated designs that make future changes slower or riskier; some of it is deliberate and reasonable. Argue in business terms: incidents, delivery speed, onboarding time, security exposure. Pay it down incrementally as part of feature work, and keep a visible list with impact estimates.

**Q102. What do linters, formatters and static analysis give a team?**
Formatters (Prettier, RuboCop's layout cops) end style debates; linters (ESLint, RuboCop) catch bug-prone patterns; type checkers (TypeScript in strict mode) eliminate whole classes of errors; security scanners (Brakeman for Rails, dependency audits) catch known vulnerabilities. Run them in pre-commit hooks and in CI so reviews can focus on design.

**Q103. What makes a good code review?**
Reviewers check correctness, edge cases, security, tests, readability and fit with the existing design, on small pull requests. Comment on the code, not the person, explain why, separate blocking issues from suggestions ("nit:"), and approve when it is good enough rather than perfect. Authors keep pull requests small and focused, write a description with context and screenshots, and review their own diff first.

**Q104. What are the Agile values?**
Individuals and interactions over processes and tools, working software over comprehensive documentation, customer collaboration over contract negotiation, and responding to change over following a plan. The items on the right still have value; the items on the left matter more.

**Q105. Explain Scrum's roles, events and artifacts.**
Accountabilities: the Product Owner (owns the backlog and its priorities), the Scrum Master (coaches the process and removes impediments) and the Developers (build the increment). Events: the Sprint (a fixed length of one month or less, often two weeks), Sprint Planning, the Daily Scrum, the Sprint Review and the Sprint Retrospective. Artifacts: the Product Backlog, the Sprint Backlog and the Increment, which must meet the Definition of Done.

**Q106. Scrum vs Kanban?**
Scrum works in fixed sprints with planned goals and defined accountabilities. Kanban is continuous flow: visualize the work on a board, limit work in progress per column, and optimize lead and cycle time, with no sprints. Many teams mix them, such as sprints plus WIP limits.

**Q107. What makes a good user story?**
"As a [type of user], I want [goal], so that [benefit]", with clear acceptance criteria (often written as Given/When/Then), small enough to finish in a few days. INVEST: Independent, Negotiable, Valuable, Estimable, Small, Testable.

**Q108. How do story points and estimation work?**
Story points estimate relative size and uncertainty, not hours, often on a Fibonacci-like scale through planning poker. Velocity (points completed per sprint) helps one team forecast; it must not be used to compare teams or judge individuals. For work with big unknowns, time-box a spike first.

**Q109. Definition of Ready vs Definition of Done?**
Ready: a story is clear enough to start (acceptance criteria, designs, dependencies sorted). Done: the shared quality bar for finished work (code reviewed, tests passing, deployed to staging, documented, acceptance criteria met). A strict Definition of Done prevents "done except testing".

**Q110. How do you handle scope creep or an unrealistic deadline?**
Make the trade-off visible early: break the work down, estimate it, and offer options (cut scope to an MVP, deliver in phases, move the date) instead of silently working nights. Write down what was agreed, and raise risks as soon as you see them.
