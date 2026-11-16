# Observability and Debugging

> **Tier:** T2 (working knowledge) · **Related:** [CI/CD and deployment](04-ci-cd-and-deployment.md), [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md), [Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md)
> **On your resume:** the PriceCharting sync jobs on CrystalCommerce, the Colochousing Rails app with Sidekiq, the Coloc mobile app live on both stores, and REST APIs at Devntech.

## Quick revision
- **Monitoring** tells you *that* something is wrong (known failure modes, dashboards, alerts); **observability** lets you ask *why*, including questions you did not plan for, from the data the system emits.
- **Logs** = timestamped events with context (what happened). **Metrics** = numeric time series, cheap to store and aggregate (how much, how often). **Traces** = one request's path across components as timed spans (where the time went).
- Structured logs are JSON with consistent fields (level, message, request_id, user or org ID, duration), one line per event, written to stdout.
- Correlate with a request ID (`X-Request-Id`) on every log line, passed into background jobs and outbound calls; trace IDs (W3C `traceparent`) link services.
- Never log secrets, tokens, passwords, payment data or unnecessary personal data; filter parameters and scrub error reports.
- Levels: debug (local detail), info (normal business events), warn (unexpected but handled), error (an operation failed), fatal (the process cannot continue).
- **Error tracking** (Sentry): grouped exceptions with stack traces, breadcrumbs, user and release context, and alerts; source maps make minified JS and React Native stack traces readable.
- **APM** (New Relic, Datadog, Scout, Skylight, AppSignal): throughput, latency percentiles, error rate, slow transactions broken into database, external and rendering time, N+1 detection.
- Golden signals: latency, traffic, errors, saturation. RED for services (rate, errors, duration); USE for resources (utilization, saturation, errors).
- **SLI** = a measurement, **SLO** = its target (99.9% over 30 days), **SLA** = a contractual promise with penalties; the **error budget** at 99.9% is about 43 minutes per 30 days.
- Alert on symptoms users feel (error rate, latency, SLO burn, queue latency, stale syncs); every alert actionable, owned and documented in a runbook. Noise causes **alert fatigue**.
- **OpenTelemetry:** vendor-neutral SDKs, auto-instrumentation and the OTLP protocol for traces, metrics and logs, exportable to any backend.
- Production debugging: scope the impact, check what changed, **mitigate first** (roll back, disable the flag), gather evidence, test one hypothesis at a time, fix with a regression test, follow up with a postmortem.
- Rails console in production: read first, `--sandbox` for experiments (outside side effects still happen), `find_each` batches, transactions, reviewed scripts for real data fixes.
- Debuggers: the `debug` gem (`debugger`, `binding.break`), `binding.irb`, `node --inspect`, browser DevTools, React DevTools, React Native DevTools (the default debugger from RN 0.76).
- Background jobs: watch queue latency (age of the oldest job), retries, dead jobs and duration; alert when a sync's last success gets too old.
- **Blameless postmortems:** timeline, impact, root and contributing causes, owned action items; fix systems, not people.

## 1. Logs vs metrics vs traces

| | Logs | Metrics | Traces |
|---|---|---|---|
| What | discrete events with context | numeric measurements aggregated over time | the timed path of one request through components (spans) |
| Answers | what exactly happened to this request or user? | how much, how often, is it getting worse? | where did the time go, and which hop failed? |
| Example | `{"level":"error","msg":"price sync failed","store_id":42,"status":429}` | p95 latency of `GET /bills`, Sidekiq queue latency, error rate | `POST /payments` 900 ms: 40 ms auth, 120 ms database, 700 ms payment provider |
| Cost | grows with traffic and verbosity | cheap and compact | moderate; usually sampled |
| Tools | CloudWatch Logs, Datadog Logs, Loki, the ELK stack, Papertrail | Prometheus with Grafana, Datadog, CloudWatch | Jaeger, Grafana Tempo, Honeycomb, Datadog APM, New Relic, Sentry |

They work together: a metric alert fires, dashboards narrow it to one endpoint, a trace shows the slow span, and logs carrying the same request or trace ID show the exact error. Error trackers and profilers are useful extra signals.

## 2. Structured logging

Plain-text logs work until you need to answer "how many 500s did org 7 get on `/api/bills` after the last deploy?". JSON logs with consistent fields can be filtered, counted, graphed and alerted on. Useful fields: timestamp, level, message, service, environment and release, request or trace ID, user and org IDs (IDs, not emails), route, status, duration, job class and job ID, external service name and status.

```ruby
# config/environments/production.rb (lograge gem: one JSON line per request)
config.log_tags = [:request_id]
config.lograge.enabled = true
config.lograge.formatter = Lograge::Formatters::Json.new
config.lograge.custom_options = ->(event) { { request_id: event.payload[:request_id], user_id: event.payload[:user_id] } }

# app/controllers/application_controller.rb
def append_info_to_payload(payload)
  super
  payload[:request_id] = request.request_id
  payload[:user_id] = current_user&.id
end
```

```ts
import { Hono } from "hono";
import pino, { type Logger } from "pino";
import { randomUUID } from "node:crypto";

const logger = pino({ level: process.env.LOG_LEVEL ?? "info", redact: ["password", "token", "*.authorization"] });
const app = new Hono<{ Variables: { log: Logger } }>();

app.use(async (c, next) => {
  const requestId = c.req.header("x-request-id") ?? randomUUID();
  const log = logger.child({ requestId }); // every line logged for this request carries the ID
  c.set("log", log);
  c.header("x-request-id", requestId);
  const start = performance.now();
  await next();
  log.info({ method: c.req.method, path: c.req.path, status: c.res.status, durationMs: Math.round(performance.now() - start) }, "request");
});
```

- Pass the request ID into background jobs (as an argument or through job middleware) and onto outbound API calls, so one ID follows the work end to end.
- Log decisions and outcomes ("sync finished: 1,240 updated, 3 failed"), not every loop iteration, and pick levels deliberately so production can run at `info`.
- Rails 8.1 added a structured event reporter (`Rails.event`); check the release notes if your app is on it.

## 3. Never log secrets or personal data

- Never log passwords, tokens, API keys, session cookies, `Authorization` headers, card data or one-time codes. Avoid personal data (emails, phone numbers, addresses, rental application details) unless genuinely needed; log IDs instead.
- Rails: `config.filter_parameters` (the generated initializer already covers names such as password, token and secret; add your own) filters request logs and ActiveRecord `inspect` output.
- Error trackers: keep Sentry's `send_default_pii` off, scrub request bodies and headers, and set the user context to an ID.
- Logs are personal data under GDPR, with retention limits and access control ([Web security and OWASP](../04-web-fundamentals/07-web-security-owasp.md)). Do not dump full third-party responses that may contain credentials or customer data.

## 4. Error tracking with Sentry

An error tracker captures exceptions with stack traces, request data, user and tags; groups them into issues; shows first and last seen, frequency, affected users and the release that introduced each issue; and alerts the team.

```ruby
# Gemfile: gem "sentry-ruby", gem "sentry-rails", gem "sentry-sidekiq"
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]
  config.environment = Rails.env
  config.release = ENV["GIT_SHA"]                               # ties every event to a deploy
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]
  config.traces_sample_rate = 0.1                               # performance tracing for 10% of requests
  config.send_default_pii = false
end

# reporting a handled error with context
Sentry.with_scope do |scope|
  scope.set_tags(integration: "pricecharting")
  scope.set_context("sync", { store_id: store.id, page: page })
  Sentry.capture_exception(error)
end
```

```tsx
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN, // Vite-style env; other bundlers expose env differently
  release: import.meta.env.VITE_GIT_SHA,
  integrations: [Sentry.browserTracingIntegration()],
  tracesSampleRate: 0.1,
});

export function App() {
  return (
    <Sentry.ErrorBoundary fallback={<p>Something went wrong.</p>}>
      <AppRoutes />
    </Sentry.ErrorBoundary>
  );
}
```

**React Native and Expo:** `@sentry/react-native` with its Expo config plugin captures JavaScript errors and native crashes; wrap the root component with `Sentry.wrap`, set the release and distribution to the app version and build number, and upload source maps (and native debug symbols) during EAS Build, plus source maps for each EAS Update bundle. Follow Sentry's Expo guide, because the setup details change between SDK versions.

- **Breadcrumbs** are the automatic trail before an error (navigation, clicks, HTTP requests, console output, SQL in Rails); add your own for domain steps.
- **Releases** tag events with a version or commit SHA, so you can see which deploy introduced or re-opened an issue and compare error rates across deploys.
- **Source maps** translate minified production JavaScript back to your original files and lines; upload them from CI rather than serving them publicly.
- Capture all errors but sample traces and replays to control cost; add context such as user ID, org ID, app version and platform.

## 5. APM

APM agents auto-instrument requests, database queries, cache calls, external HTTP calls and background jobs. You get throughput, latency percentiles (p50, p95, p99), error rate and Apdex per endpoint and job; transaction traces that split one slow request into SQL, rendering and external time; N+1 and slow-query detection; deploy markers; and host metrics. New Relic and Datadog cover many languages, Scout APM and Skylight are simple Rails-focused options, and AppSignal and Sentry Performance sit in between.

Watch percentiles, not averages: a 200 ms average can hide that one request in twenty takes four seconds, and those slow requests often belong to your biggest customers (the ones with the most data).

## 6. Dashboards, alerting, SLIs and SLOs

- **Signals:** the four golden signals are latency, traffic, errors and saturation (how full a resource is: CPU, memory, connection pool, queue depth). RED covers request-driven services; USE covers resources.
- **A service dashboard:** request rate, error rate, p95 latency per endpoint, Puma or Node saturation, database time and connections, Sidekiq queue latency and failures, and deploy markers.
- **SLI** (indicator): a measured ratio, such as the share of requests that succeed in under 500 ms. **SLO** (objective): the target, such as 99.9% over 30 days. **SLA** (agreement): an external, contractual promise with penalties, set looser than the SLO. **Error budget:** the failure the SLO allows (0.1% of 30 days is about 43 minutes); burning it quickly means slowing releases and investing in reliability.
- **Alerting:** alert on symptoms users feel (error rate, latency, SLO burn rate, queue latency, stale syncs) rather than every possible cause (80% CPU may be fine). Each alert needs an owner, a runbook and the right channel: a page for urgent problems, a ticket or chat message for the rest.
- **Alert fatigue:** noisy, flapping or unactionable alerts train people to ignore every alert. Tune thresholds and durations, deduplicate, delete alerts nobody acts on, and review alerting after each incident.

## 7. Distributed tracing with OpenTelemetry

```text
POST /api/payments                           trace_id=4bf92f35...   total 912 ms
  auth middleware                            12 ms
  SELECT ... FROM memberships                8 ms
  PaymentService.charge                      870 ms
    HTTP POST api.payment-provider.example   842 ms   (the slow span)
  INSERT INTO payments                       6 ms
```

A **trace** is a tree of **spans** (name, start time, duration, attributes, status, parent). Context travels between services in the W3C `traceparent` header and can be stored in job payloads, so a Sidekiq job's spans join the trace of the request that enqueued it. **OpenTelemetry** (a CNCF project) standardizes the APIs, SDKs, auto-instrumentation and the OTLP protocol; its Collector receives, samples, scrubs and exports data to Jaeger, Grafana Tempo, Honeycomb, Datadog or New Relic, so you can change vendors without re-instrumenting.

```ts
// instrumentation.ts: load before the app, e.g. node --require ./dist/instrumentation.js dist/server.js
// (ESM apps need the loader setup described in the OpenTelemetry docs)
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

const sdk = new NodeSDK({
  serviceName: "api",
  traceExporter: new OTLPTraceExporter(),            // endpoint read from OTEL_EXPORTER_OTLP_ENDPOINT
  instrumentations: [getNodeAutoInstrumentations()], // http, pg, redis and common frameworks
});
sdk.start();
```

Ruby uses the `opentelemetry-sdk` and `opentelemetry-instrumentation-all` gems (`OpenTelemetry::SDK.configure { |c| c.service_name = "web"; c.use_all }`). Sampling keeps costs down: keep a fixed share of traces up front, or keep the interesting ones (errors, slow requests) with tail sampling in the Collector.

## 8. Debugging a production issue, step by step

1. **Scope the impact:** what is broken, for whom (everyone, one org, one platform or app version), since when, and how badly. Declare an incident and communicate if users are seriously affected.
2. **Ask what changed:** recent deploys, configuration or environment changes, migrations, feature flags, dependency or SDK upgrades, a new mobile release, traffic spikes, scheduled jobs, third-party status pages.
3. **Mitigate first** if users are hurting: roll back, disable the flag, scale up, pause a runaway queue, fail over. Root cause can wait until the bleeding stops.
4. **Gather evidence:** the error tracker issue (first seen, release, stack trace, breadcrumbs), logs filtered by request, user or org, dashboards (errors, latency, saturation, queue latency), traces of slow requests, and database health (slow queries, locks, connections).
5. **Reproduce** locally or on staging with similar data and the same version, ideally as a failing test.
6. **Form hypotheses and test them one at a time:** compare a working case with a failing one, and bisect between the last good and first bad release.
7. **Fix,** add the regression test, deploy, and verify with the same signals that showed the problem.
8. **Follow up:** a blameless postmortem, the alert or dashboard that would have caught it sooner, and an updated runbook.

## 9. Tools

**Rails console in production, safely:**
- Investigate read-only first. `bin/rails console --sandbox` rolls back database changes on exit, but emails, enqueued jobs, API calls and Redis writes still happen, and a long sandbox session can hold locks.
- Check counts before acting, process large tables with `find_each` (never `Model.all.each`), and wrap changes in a transaction.
- For real data fixes, prefer a reviewed script, rake task or data migration over live typing; pair with someone and record what you ran.

```ruby
scope = Bill.where(house_id: 12, status: "overdue").where(due_on: ...1.year.ago)
scope.count                                                      # look first: is this the number you expect?
ActiveRecord::Base.transaction do
  scope.find_each { |bill| bill.update!(status: "written_off") } # validations and callbacks still run
end
```

**Debuggers:**
- **Ruby:** the `debug` gem (bundled with Ruby 3.1+ and in Rails 7+ Gemfiles). Put `debugger` or `binding.break` in the code, trigger the request, then step with `next`, `step` and `continue` and inspect values; `binding.irb` opens an IRB session at that line. Never commit breakpoints.
- **Node:** `node --inspect dist/server.js` (or `--inspect-brk` to pause on the first line), then attach Chrome DevTools through `chrome://inspect` or the VS Code debugger; a `debugger` statement acts as a breakpoint.

**Front end:**
- **Browser DevTools:** Console, Network (status, payloads, timing, Copy as cURL), Sources (breakpoints, including on fetch requests and DOM changes), Performance, and Application (storage, cookies, service workers).
- **React DevTools:** the component tree with props, state and hooks, plus a Profiler showing which components rendered and why ([React performance](../06-react/06-performance.md)).
- **React Native:** React Native DevTools is the default debugger from RN 0.76 (press `j` in the Expo CLI to open it), with console, breakpoints, components and profiler; LogBox shows errors and warnings in the app; native modules need a development build; release-only crashes rely on Sentry with source maps and native symbols, plus device logs (the Xcode console, `adb logcat`).

## 10. Performance debugging

- **Slow requests:** start from APM or logs (which endpoint, at what percentile), then open a trace to see whether time goes to the database, external calls or rendering.
- **Slow queries:** `EXPLAIN ANALYZE` for one query (missing index, sequential scan on a big table, bad join), `pg_stat_statements` for the most expensive queries overall, and slow query logs. Fix with indexes, narrower selects, pagination or caching ([Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md)).
- **N+1 queries:** many near-identical queries per request in logs or APM; catch them with Bullet or `strict_loading` in development and fix with `includes` or `preload` ([N+1 queries](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md)).
- **Memory growth in Ruby:** processes swell until they are killed, usually from loading too many records at once, big in-memory caches or fragmentation (jemalloc helps); measure with `memory_profiler` or `derailed_benchmarks`. Restarting bloated workers is a band-aid, not a fix.
- **Memory leaks in Node:** take heap snapshots in Chrome DevTools a few minutes apart and compare which objects keep growing: unbounded maps or caches, listeners and timers never removed, closures holding large data. In React, look for effects without cleanup and subscriptions never unsubscribed.
- **CPU:** flame graphs from stackprof or rbspy in Ruby, `node --cpu-prof` or the DevTools profiler in Node. Front-end performance tools are covered in [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md).

## 11. Monitoring background jobs and third-party syncs

**Queue latency**, the age of the oldest waiting job, is the most important job metric: when it climbs, notifications arrive late and data goes stale even though nothing has "failed". Also watch throughput, job duration (p95), failure rate, the retry and dead sets, and worker memory and restarts. The Sidekiq Web UI (mounted behind admin authentication) shows queues, retries and dead jobs and lets you retry or delete them.

```ruby
require "sidekiq/api"

Sidekiq::Queue.new("default").latency  # seconds the oldest job has been waiting
Sidekiq::Queue.new("default").size     # jobs waiting
Sidekiq::RetrySet.new.size             # failed jobs scheduled to retry
Sidekiq::DeadSet.new.size              # jobs that exhausted their retries
```

Export these numbers to your metrics system and alert on thresholds (latency above a few minutes, a growing dead set), and use a `sidekiq_retries_exhausted` hook or Sentry's Sidekiq integration to report jobs that die.

**Third-party syncs**, such as a PriceCharting integration:
- Log every external call with the integration name, endpoint, status, latency and the provider's request ID, and tag errors by integration.
- Track freshness: store `last_successful_sync_at` per store or account and alert when it is older than expected. Silent failures (a sync that stops being scheduled, or "succeeds" with zero records) only show up this way.
- Classify errors: 4xx means bad data or revoked credentials (do not retry forever; surface it to a person), 429 means back off using `Retry-After`, 5xx and timeouts mean retry with backoff.
- Count records processed, created, updated and failed per run; a sudden drop is a signal.
- Run reconciliation jobs that compare both systems and repair drift, and use a circuit breaker so a failing provider is not hammered ([Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md)).

## 12. Incident response and blameless postmortems

- For bigger incidents, assign an **incident commander** (coordinates and decides), a **communicator** (status page, stakeholders, client updates) and responders; small teams combine roles, but someone must coordinate.
- Use severity levels, one shared channel and a running timeline of actions and findings. Move through detect, triage, mitigate, resolve and follow up, and communicate early and regularly, even if the update is "still investigating".

```text
Title: <what broke> (<date>)
Summary: two sentences on what happened
Impact: who was affected, for how long, and what they saw
Timeline (UTC): detection, key actions, mitigation, resolution
Root cause and contributing factors: technical and process ("five whys")
What went well, what went badly, where we got lucky
Action items: concrete, owned and dated (an alert on queue latency, a runbook, a retry policy fix)
```

**Blameless** means asking how the system and process allowed the failure (a missing alert, an unsafe deploy step, an unclear runbook) rather than who made the mistake, so people report problems early and honestly. The output is a list of action items that make the failure less likely or less harmful next time.

## Common pitfalls
- Unstructured logs without request IDs, so one user's failure cannot be followed through the system.
- Logging secrets, tokens or personal data, or shipping them to error trackers.
- No `release` on error events and no uploaded source maps, so stack traces are unreadable and regressions cannot be tied to deploys.
- Averages instead of percentiles on dashboards.
- Alerts on causes rather than symptoms, alerts without runbooks, and alert floods nobody reads.
- Root-causing for an hour while users suffer, instead of rolling back first.
- Destructive console sessions in production without a transaction, a count check or a second pair of eyes.
- Watching only job failures, not queue latency or sync freshness, so silent stalls go unnoticed.
- Postmortems that name a culprit and produce no action items.

## How to talk about this in an interview
- For "how do you debug a production issue?", walk through the method in section 8 and anchor it in a real example: scope, what changed, mitigate, evidence, hypothesis, fix, follow-up.
- The PriceCharting integration on CrystalCommerce is a natural monitoring story: queue latency, error classification (4xx vs 429 vs 5xx), freshness alerts and reconciliation. Be clear about what existed and what you would add.
- Colochousing spans a Rails app with Sidekiq and the Coloc mobile app, so error tracking with releases and source maps across Rails, React and React Native, and push notification failures, are concrete topics.
- Show habits: request IDs in logs, release tags on errors, percentiles, rolling back before debugging, blameless follow-ups.

> **Fill in:** the monitoring stack on your Devntech projects (Sentry, an APM, log platform, Sidekiq dashboards) and what you checked first when something broke.

> **Fill in:** one production bug you debugged end to end: symptom, how you found the cause (logs, traces, console, bisect), the fix, and what you added afterwards (test, alert, runbook).

## Interview questions

**Q1. Logs vs metrics vs traces?**
Logs are detailed event records that tell you exactly what happened to a request or user. Metrics are cheap numeric time series for trends, dashboards and alerts (rates, latencies, queue depth). Traces follow one request across components as timed spans, showing where time went and what failed. Metrics tell you something is wrong, traces where, and logs why.
*Follow-up:* How do you connect them? Shared request and trace IDs in log lines, spans and error events.

**Q2. How do you debug a production issue? Walk me through it.**
Scope the impact (who, what, since when), check what changed (deploys, config, flags, third parties), and mitigate first if users are hurting (roll back, disable the flag). Then gather evidence from the error tracker, logs by request ID, dashboards and traces, reproduce with a failing test, test hypotheses one at a time, fix and verify with the same signals, and follow up with a postmortem and better alerts.
*Follow-up:* Why mitigate before finding the root cause? Every minute of investigation is a minute of user impact, and a rollback is usually fast and safe.

**Q3. What is structured logging, and why use request or correlation IDs?**
Logging events as machine-readable records (usually JSON) with consistent fields, so logs can be filtered, aggregated and alerted on instead of grepped. A request ID generated at the edge and attached to every log line, job and outbound call lets you reconstruct everything that happened for one request across processes.
*Follow-up:* How does Rails help? `config.log_tags = [:request_id]` and the `X-Request-Id` header that Rails sets on every request, plus lograge for one JSON line per request.

**Q4. What should you never log?**
Passwords, tokens, API keys, session cookies, `Authorization` headers, card data and one-time codes, and personal data beyond what you truly need (log IDs instead of emails or addresses). Use parameter filtering, logger redaction and error-tracker scrubbing, because logs are copied widely and are themselves personal data under GDPR.
*Follow-up:* Where else does sensitive data leak? Error reports, APM traces with SQL parameters, and third-party responses dumped into logs.

**Q5. How would you set up error tracking across Rails, React and React Native? What are releases and source maps for?**
Install the Sentry SDK in each (sentry-rails plus sentry-sidekiq, `@sentry/react` with an error boundary, `@sentry/react-native` with the Expo config plugin), set environment and release on every event, attach user and org IDs, and upload source maps (and native symbols for mobile) in CI and EAS builds. Releases show which deploy introduced or regressed an issue; source maps turn minified stack traces back into real file and line numbers.
*Follow-up:* What are breadcrumbs? The automatic trail of navigation, clicks, requests and logs before an error.

**Q6. SLI vs SLO vs SLA? What is an error budget?**
An SLI is a measurement (the share of requests that succeed in under 500 ms), an SLO is the internal target for it (99.9% over 30 days), and an SLA is an external contract with penalties, set looser than the SLO. The error budget is the failure the SLO allows (about 43 minutes per 30 days at 99.9%); when it is burning fast, the team slows feature releases and works on reliability.
*Follow-up:* What do you alert on? The burn rate of the error budget, not every individual error.

**Q7. What is alert fatigue, and how do you avoid it?**
When alerts are noisy, flapping or unactionable, people start ignoring them, including the important ones. Alert on user-facing symptoms with sensible thresholds and durations, make every alert actionable with an owner and a runbook, route by urgency, deduplicate, and delete or tune alerts that nobody acts on.
*Follow-up:* Is high CPU a good page? Usually not on its own; page on errors, latency or queue delays that users feel.

**Q8. What is distributed tracing, and what does OpenTelemetry give you?**
Tracing records each request as a tree of timed spans across services, databases, caches, external APIs and jobs, with context propagated in the `traceparent` header. OpenTelemetry provides vendor-neutral APIs, SDKs, auto-instrumentation, the OTLP protocol and a Collector, so you instrument once and export to any backend.
*Follow-up:* How do you control cost? Sample: keep a share of traces, or use tail sampling to keep errors and slow requests.

**Q9. How do you use the Rails console safely in production?**
Start read-only, check counts before changing anything, use `find_each` for large tables, wrap changes in a transaction, and remember that `--sandbox` rolls back the database but not emails, jobs, API calls or Redis writes. For real data fixes, prefer a reviewed script or data migration, pair with someone and record what ran.
*Follow-up:* Why avoid `update_all` for data fixes? It skips validations and callbacks; sometimes that is intended, so decide deliberately.

**Q10. How would you find a memory leak?**
Confirm the growth with metrics (memory per process over time, restarts or OOM kills), then narrow it down. In Node, compare heap snapshots taken minutes apart to find object types that keep growing (unbounded caches, listeners, timers, closures). In Ruby, look for code loading too many records, growing in-process caches and fragmentation, using `memory_profiler` or `derailed_benchmarks`; jemalloc reduces fragmentation.
*Follow-up:* Is a worker killer a fix? No, it is a band-aid that hides the cause.

**Q11. How do you monitor Sidekiq and background jobs?**
Track queue latency (age of the oldest job) per queue, queue sizes, throughput, job duration, failure rate and the retry and dead sets, from Sidekiq's API exported to your metrics system, with alerts on latency and dead-set growth. Report exceptions to Sentry with job context, and use the Sidekiq Web UI behind admin authentication for inspection and retries.
*Follow-up:* Why latency over queue size? A thousand fast jobs may clear in seconds, while ten slow ones can delay everything; latency measures what users feel.

**Q12. How would you detect that a third-party sync silently stopped working?**
Record a `last_successful_sync_at` per account and alert when it is older than the expected interval, and track records processed per run so a drop to zero raises a flag. Log every external call with status and latency, classify errors (4xx, 429, 5xx), alert on rising failure rates, and run reconciliation jobs that compare both systems.
*Follow-up:* Why are silent failures common? A job that is never enqueued, or that succeeds with empty results, produces no errors at all.

**Q13. What is a blameless postmortem?**
A written review after an incident covering impact, timeline, root and contributing causes, what went well and badly, and owned, dated action items. It focuses on the systems and processes that allowed the failure rather than on who made a mistake, so people share information openly and the same class of problem becomes less likely.
*Follow-up:* What makes action items good? They are specific, owned, dated and actually change the system (an alert, a guardrail, a runbook), not "be more careful".

**Q14. Why do you watch p95 and p99 instead of averages?**
Averages hide the slow tail: a 200 ms mean can coexist with one request in twenty taking seconds. Percentiles show what the slowest users experience, and those are often your heaviest users or worst network conditions. SLOs and latency alerts are therefore defined on percentiles.
*Follow-up:* Why not alert on p100? The maximum is dominated by one-off outliers; p99 balances sensitivity and noise.

## Further reading
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Sidekiq wiki: Monitoring](https://github.com/sidekiq/sidekiq/wiki/Monitoring)
- [Rails guide: Debugging Rails Applications](https://guides.rubyonrails.org/debugging_rails_applications.html)
- [Google SRE book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
