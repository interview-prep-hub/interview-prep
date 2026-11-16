# Caching and Performance

> **Tier:** T2 (working knowledge) · **Related:** [How the web works](01-how-the-web-works.md), [Redis](../12-databases/10-redis.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md), [N+1 queries in ActiveRecord](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Rails caching, performance and security](../05-ruby-on-rails/14-caching-performance-security.md), [React performance](../06-react/06-performance.md), [Observability and debugging](../14-devops-and-tools/05-observability-and-debugging.md)
> **On your resume:** CrystalCommerce's legacy Rails platform serving 700+ stores and 8M+ customers (including importing PriceCharting market prices into the catalog), the Colochousing Rails 7 app and API, and the Coloc mobile app at Devntech.

## Quick revision
- Cache layers from the user inward: browser (HTTP cache, service worker, client data caches such as TanStack Query), CDN edge, reverse proxy, application cache (Redis, Memcached, in-process), database (buffer cache, materialized views).
- Cache public, rarely changing data as close to the user as possible; cache per-user data only in the browser or in carefully keyed application caches.
- HTTP caching: `Cache-Control` sets freshness, `ETag`/`Last-Modified` enable 304 revalidation. Fingerprinted assets get a year plus `immutable`; HTML gets `no-cache`.
- CDNs cache at edge locations, keyed by URL plus chosen headers, cookies and query params; `s-maxage` controls them; purge by URL or tag, or avoid purges with versioned URLs.
- **Cache-aside:** read the cache, on a miss load from the DB and fill it. **Read-through:** the cache layer loads on a miss. **Write-through:** write cache and DB together. **Write-behind:** write the cache, flush to the DB later.
- Invalidation: TTLs, delete-on-write after commit, versioned keys (Rails `cache_version` from `updated_at`), change events. Decide how stale is acceptable per piece of data.
- **Cache stampede:** a hot key expires and many requests recompute it at once. Fix with a lock (single flight), serve-stale-while-refreshing (`race_condition_ttl`), jittered TTLs, early refresh.
- **Core Web Vitals** (75th percentile of real page loads): **LCP** at most 2.5 s (loading), **INP** at most 200 ms (responsiveness; replaced FID in 2024), **CLS** at most 0.1 (visual stability).
- LCP: fast server response, prioritized hero image (never lazy-loaded), smaller images, no render-blocking resources.
- INP: less JavaScript, short tasks, cheap event handlers, yielding to the main thread, transitions, virtualized lists.
- CLS: dimensions on images and embeds, reserved space for late content, metric-matched font fallbacks, transform-based animation.
- Ship less: route-level code splitting, `React.lazy`, dynamic `import()`, tree shaking, `loading="lazy"` below the fold, fewer third-party scripts.
- Images: AVIF or WebP, `srcset` plus `sizes`, explicit width and height. Fonts: WOFF2, `font-display: swap`, preload, subset.
- Brotli for text where supported (precompressed for static files), gzip as fallback; never recompress images.
- Backend: fix N+1, add the right indexes, paginate, select only needed columns, move slow work to background jobs, size connection pools, cache expensive reads, put timeouts on external calls.
- Measure first: Lighthouse and PageSpeed Insights, WebPageTest, DevTools Performance, real-user monitoring with `web-vitals`, APM traces, `EXPLAIN ANALYZE`, `pg_stat_statements`. Use percentiles, not averages.

## 1. Caching layers: what to cache where

| Layer | Examples | Good to cache | Avoid |
|---|---|---|---|
| Browser | HTTP cache, service worker, TanStack Query or Apollo caches | static assets; the user's own data for short periods | other users' data; secrets in plain storage |
| CDN / edge | CloudFront, Cloudflare, Fastly | assets, images, public pages and public API responses | personalized responses (unless keyed very deliberately) |
| Reverse proxy | Nginx, Varnish | static files, very short "microcaching" of public pages | per-user responses |
| Application | Redis, Memcached, `Rails.cache`, an in-process LRU | expensive computations, third-party API responses, rendered fragments, rate-limit counters | data that must be exactly current (balances, stock at checkout) unless invalidated precisely |
| Database | buffer cache, materialized views | hot rows and indexes (automatic), precomputed aggregates | (managed by the database) |

An in-process memory cache is the fastest of all but lives per process: each server holds its own copy, it disappears on deploy and it competes with the app for memory. Redis or Memcached gives every server one shared view.

## 2. HTTP caching recap

The browser cache follows `Cache-Control` (`max-age`, `no-cache`, `no-store`, `private`, `public`, `immutable`) and revalidates with `ETag`/`If-None-Match` to get cheap 304s; [How the web works](01-how-the-web-works.md) covers the directives and policies in detail. For layered caching, remember three things: `s-maxage` gives shared caches (CDNs) a different lifetime from browsers; `stale-while-revalidate` and `stale-if-error` let caches serve a slightly old copy while refreshing or while the origin is failing; and every header listed in `Vary` multiplies the cache entries.

## 3. CDNs: edge caching, cache keys and purging

- **How it works:** DNS sends users to a nearby edge location. On a miss, the edge fetches from your origin (optionally through an "origin shield" layer that collapses misses), stores the response for its TTL, and serves later requests at low latency.
- **Cache key:** usually scheme, host and path, plus whichever query parameters, headers and cookies you configure. Include too much (every tracking parameter, every cookie) and the hit ratio collapses; include too little (forget the language header) and users get the wrong variant.
- **Personalized content:** responses with `Set-Cookie` or for authenticated users should not be shared; most CDNs skip them by default, but verify, because a cached logged-in page served to strangers is a data leak.
- **Invalidation:** fingerprinted asset URLs (`application-4f9a1c.js`) need no purge, since every deploy references new URLs. Otherwise purge by URL or path, or by tag (surrogate keys: purge every page tagged `product-123` when its price changes). Purges take time to propagate and can cost money.
- **Extras:** TLS termination near users, HTTP/3, DDoS absorption, image resizing, edge functions. In Rails, `config.asset_host = "https://cdn.example.com"` serves assets from a CDN domain.
- The key metric is the **hit ratio**, together with origin load.

## 4. Caching patterns

| Pattern | Read path | Write path | Pros | Cons |
|---|---|---|---|---|
| Cache-aside (lazy loading) | app checks the cache; on a miss reads the DB and fills the cache | app writes the DB, then deletes the key | simple; caches only what is requested; a cache outage degrades to DB reads | first request is slow; stale until TTL or invalidation; stampede risk |
| Read-through | app asks the cache layer, which loads from the DB on a miss | same as cache-aside | loading logic in one place | needs a library or store that supports loaders |
| Write-through | read from the cache | write cache and DB together, synchronously | cached keys always fresh after writes | slower writes; caches data that may never be read |
| Write-behind (write-back) | read from the cache | write the cache; flush to the DB asynchronously in batches | very fast writes, absorbs bursts (view counters) | data loss if the cache dies before flushing; complex consistency |

```ruby
# Cache-aside in one call: the block runs only on a miss
def market_price(product_id)
  Rails.cache.fetch(["market_price", product_id], expires_in: 1.hour, race_condition_ttl: 10.seconds) do
    MarketPriceClient.new.fetch(product_id) # slow third-party call (illustrative client)
  end
end

# Key-based invalidation: the record's cache_version (from updated_at) is part of the lookup,
# so any update makes the old entry a miss and nothing has to be deleted explicitly
Rails.cache.fetch([listing, "price_badge"]) { PriceBadge.new(listing).to_h }
```

```ts
import Redis from "ioredis";

const redis = new Redis(process.env.REDIS_URL!);

export async function getHouseSummary(houseId: string): Promise<HouseSummary> {
  const key = `house-summary:v1:${houseId}`;          // bump v1 when the cached shape changes
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit) as HouseSummary;

  const summary = await loadHouseSummary(houseId);     // the expensive database work
  const ttl = 300 + Math.floor(Math.random() * 60);    // 5 minutes plus jitter against synchronized expiry
  await redis.set(key, JSON.stringify(summary), "EX", ttl);
  return summary;
}

export async function onHouseChanged(houseId: string) {
  await redis.del(`house-summary:v1:${houseId}`);      // call after the database transaction commits
}
```

## 5. TTLs, invalidation and consistency

Caching trades freshness for speed, so the first question is always "how stale may this be?" Market prices refreshed hourly are fine; a rent balance right after a payment is not.

- **TTL only:** simplest; staleness is bounded by the TTL. Add jitter so keys created together do not expire together.
- **Delete on write:** after the DB transaction commits (`after_commit`), delete the key and let the next read refill it. Deleting is safer than writing the new value, which can race with a slower request writing an older value. Inside the transaction is too early: a reader may refill the cache with the old row before the commit.
- **Versioned keys:** put a version in the key (`updated_at`, a counter, a schema version); old entries simply age out under an LRU eviction policy. Rails Russian-doll caching nests fragments and uses `touch: true` so a child's update changes its parent's version too.
- **Event-driven invalidation:** publish change events (pub/sub, a message bus) so other services or processes drop their copies.
- **Read-your-writes:** after a user edits their profile, they must see the edit; invalidate or bypass the cache for their next read.
- **Do not cache** what must be exactly current at decision time (stock during checkout, permissions after revocation) unless you re-check the source of truth when it matters.

## 6. Cache stampedes and other failure modes

- **Stampede (dogpile, thundering herd):** a hot key expires and hundreds of concurrent requests miss and recompute it, hammering the database. Fixes: a short lock so only one request recomputes (`SET lock:key 1 NX EX 10` in Redis) while others wait briefly or get the stale value; serve-stale-while-refreshing (Rails `race_condition_ttl` keeps serving the old value for a few seconds while one process recomputes); refreshing hot keys early in the background; jittered TTLs.
- **Penetration:** requests for keys that never exist always miss and hit the database (a bug or an attack). Cache negative results briefly, or check a Bloom filter first.
- **Avalanche:** many keys expire together or the cache restarts empty, and the database takes the full load. Jitter TTLs, warm critical keys, and make sure the database can survive a cold cache.
- **The cache as a dependency:** give cache calls short timeouts and treat errors as misses, so a Redis outage slows the app instead of taking it down. Configure memory limits and an eviction policy such as `allkeys-lru` for pure caches ([Redis](../12-databases/10-redis.md)).

## 7. Core Web Vitals and how to improve each

| Metric | Measures | Good / poor | Common causes | Fixes |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | when the largest image or text block in the viewport renders | at most 2.5 s / over 4 s | slow server response, render-blocking CSS and JS, a huge or late-discovered hero image, client-only rendering | CDN and caching for HTML, server rendering, preload the hero image with `fetchpriority="high"`, never lazy-load it, compress and size images, defer non-critical JS |
| **INP** (Interaction to Next Paint) | time from a user interaction to the next painted frame, close to the worst interaction of the visit | at most 200 ms / over 500 ms | long JavaScript tasks, heavy event handlers, large re-renders, hydration, third-party scripts | ship less JS, split long tasks and yield to the main thread, debounce, `useTransition` and `useDeferredValue`, virtualize long lists, move work to web workers, trim third parties |
| **CLS** (Cumulative Layout Shift) | how much visible content moves unexpectedly | at most 0.1 / over 0.25 | images and iframes without dimensions, late banners, web fonts swapping, content injected above existing content | `width`/`height` or `aspect-ratio`, reserved space and skeletons, inserting content below or on interaction, metric-matched font fallbacks, animating `transform` |

- Google assesses each metric at the **75th percentile** of real page loads (field data from the Chrome UX Report or your own real-user monitoring), separately for mobile and desktop.
- **INP replaced FID** as a Core Web Vital in March 2024: FID only measured the delay before the first input was handled, while INP covers every interaction's full latency.
- Time to First Byte (TTFB) and First Contentful Paint are supporting metrics; a poor TTFB is a backend, caching or CDN problem.
- Core Web Vitals feed into search ranking and correlate with conversion, which is how to sell the work to a client.

## 8. Loading less: code splitting, lazy loading, third parties

```tsx
import { lazy, Suspense } from "react";

const BillsChart = lazy(() => import("./BillsChart")); // separate chunk, fetched on first render

export function Dashboard() {
  return (
    // the fallback reserves the chart's height, so loading it causes no layout shift
    <Suspense fallback={<div className="h-64 animate-pulse rounded bg-zinc-100" />}>
      <BillsChart />
    </Suspense>
  );
}
```

- **Code splitting:** routers in Next.js, TanStack Router and React Router split per route; use `React.lazy` for heavy components (charts, rich-text editors, maps) and dynamic `import()` for rarely used libraries.
- **Tree shaking:** ES modules let bundlers drop unused exports; import specific functions (`import { debounce } from "lodash-es"`) and inspect bundles with an analyzer.
- **Lazy loading:** `loading="lazy"` for below-the-fold images and iframes, IntersectionObserver for heavier components.
- **Resource hints:** `preconnect` to critical third-party origins, `preload` for the hero image or a critical font, `prefetch` for likely next pages, `defer` on scripts.
- **Third-party scripts** (analytics, chat widgets, tag managers) are often the biggest INP and LCP cost: audit them, load them async or after interaction, and remove unused ones.
- **Server rendering** (Rails views, Hotwire, SSR or static generation) puts content in the first HTML response; hydration then has its own cost. React specifics: [React performance](../06-react/06-performance.md).

## 9. Images and fonts

```html
<img src="/images/house-800.webp"
     srcset="/images/house-400.webp 400w, /images/house-800.webp 800w, /images/house-1600.webp 1600w"
     sizes="(min-width: 1024px) 50vw, 100vw"
     width="1600" height="900" fetchpriority="high"
     alt="Shared living room with a large window">

<picture>
  <source srcset="/images/room.avif" type="image/avif">
  <source srcset="/images/room.webp" type="image/webp">
  <img src="/images/room.jpg" width="800" height="600" loading="lazy" alt="Bedroom with a desk">
</picture>
```

- **Formats:** AVIF and WebP are far smaller than JPEG and PNG for photos and support transparency; SVG for icons and logos.
- `srcset` with `sizes` lets the browser pick the smallest adequate file for the layout width and pixel density; `width` and `height` reserve space and prevent layout shift.
- The hero (LCP) image gets `fetchpriority="high"` and is never lazy-loaded; everything below the fold gets `loading="lazy"`.
- Resize and convert at upload time or through an image CDN: `next/image`, Active Storage variants (libvips), and `expo-image` with caching on mobile.

```css
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-var.woff2") format("woff2");
  font-weight: 100 900;
  font-display: swap; /* show fallback text immediately, swap when the font arrives */
}
```

Serve WOFF2 only, self-host or `preconnect` to the font provider, preload the one or two critical files (`<link rel="preload" as="font" type="font/woff2" crossorigin>`), subset to the characters you need, prefer one variable font over many weights, and tune the fallback with `size-adjust` so the swap barely shifts layout. `font-display: optional` avoids the swap entirely on slow connections.

## 10. Compression: gzip vs Brotli

| | gzip | Brotli (`br`) |
|---|---|---|
| Browser support | universal | all modern browsers, over HTTPS only |
| Output on text | good | usually noticeably smaller |
| Speed | fast at every level | high levels are slow: precompress static assets at build time, use moderate levels for dynamic responses |
| Typical setup | dynamic compression in Nginx or the app | precompressed `.br` files plus CDN compression |

Compress HTML, CSS, JavaScript, JSON and SVG; JSON API responses often shrink several times over. Skip formats that are already compressed (JPEG, PNG, WebP, AVIF, WOFF2, video, zip). Minification is separate and complementary, and compressed responses need `Vary: Accept-Encoding`.

## 11. Backend performance checklist

- **N+1 queries:** preload associations (`includes`, `preload`, `eager_load`), catch regressions with Bullet or `strict_loading`, and batch in GraphQL with DataLoader ([N+1 queries](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md)).
- **Indexes:** on foreign keys and on the WHERE plus ORDER BY combinations you actually run, verified with `EXPLAIN ANALYZE` ([Indexing](../12-databases/05-indexing-and-query-optimization.md)).
- **Fetch less:** paginate, `select` or `pluck` only needed columns, avoid loading large blobs, process in batches with `find_each`, use counter caches for counts.
- **Leave the request:** emails, PDFs, imports and third-party calls belong in background jobs ([Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md)).
- **Connection pooling:** opening database connections is expensive, so each process keeps a pool. Size it to the threads that need connections (Puma threads, Sidekiq concurrency) and keep the total across processes below the database's `max_connections`; PgBouncer multiplexes many client connections onto fewer server connections (transaction pooling has caveats with prepared statements and session features). Symptom of an undersized pool: `ActiveRecord::ConnectionTimeoutError`.
- **External APIs:** timeouts, retries with backoff, response caching, batching, parallel calls for independent requests, circuit breakers.
- **Runtime:** Puma workers and threads, YJIT (Rails 7.2+ enables it by default on supported Rubies), several Node processes or instances, watching memory growth.
- **Profile, do not guess:** rack-mini-profiler (per-request SQL and timings), stackprof or rbspy flame graphs, memory_profiler, Node `--cpu-prof` or clinic.js, py-spy, and APM traces in production.
- **Load test with realistic data:** k6 or Locust against production-sized datasets. A query that is instant on a thousand rows can crawl on millions, which is exactly the gap between a dev database and a platform with 8M+ customers.

## 12. Measuring

```ts
import { onCLS, onINP, onLCP } from "web-vitals";

function report(metric: { name: string; value: number; id: string }) {
  navigator.sendBeacon("/analytics/vitals", JSON.stringify(metric)); // still delivered as the page unloads
}

onCLS(report);
onINP(report);
onLCP(report);
```

- **Lab vs field:** Lighthouse (in DevTools or Lighthouse CI) and WebPageTest (real devices and locations, filmstrips, waterfalls) give reproducible lab runs; PageSpeed Insights shows lab results plus Chrome UX Report field data; real-user monitoring (the `web-vitals` library or an APM's browser agent) shows what your actual users experience.
- **DevTools:** the Performance panel (long tasks, layout shifts, flame charts), Network (waterfall, sizes, cache hits), Coverage (unused JS and CSS).
- **Backend:** APM tools (New Relic, Datadog, Scout, Skylight, Sentry) show p95 latency per endpoint and slow traces broken into database, external and rendering time; the database side uses `EXPLAIN ANALYZE`, `pg_stat_statements` and slow query logs.
- **Method:** percentiles instead of averages, performance budgets checked in CI (bundle size, LCP), and one change at a time with a measurement before and after.

## Common pitfalls
- Optimizing before measuring, or measuring with averages that hide the slow tail.
- Caching without an invalidation plan, or deleting keys inside the transaction instead of after commit.
- Caching personalized responses at a shared layer (CDN or proxy).
- Identical TTLs on many keys, leading to synchronized expiry and stampedes.
- Treating Redis as infallible: no timeouts, no fallback, no eviction policy.
- Lazy-loading the hero image, or images without dimensions (LCP and CLS damage).
- Shipping one giant JavaScript bundle and every third-party script on every page.
- Fixing slow endpoints with caches while N+1 queries and missing indexes remain.
- Pool sizes that exceed the database's connection limit once every process is counted.
- Testing performance only on tiny development datasets.

## How to talk about this in an interview
- CrystalCommerce is your scale story: a legacy Rails platform serving 700+ stores and 8M+ customers, where N+1 queries, missing indexes, unbounded queries and slow catalog pages matter. Importing PriceCharting market prices into the catalog is a batch-processing and caching problem (background jobs, batched upserts, cached third-party responses).
- The Colochousing API serves a mobile app, so payload size, pagination and caching affect app speed on mobile networks; the Coloc app has its own list and rendering performance concerns ([React Native performance](../08-react-native-expo/07-performance-and-animations.md)).
- Describe a method rather than tricks: measure, find the biggest bottleneck, fix it, measure again. Quote numbers only if they are real.

> **Fill in:** a performance problem you fixed (N+1, a missing index, a slow page or job) with the before and after you measured, and the tool that revealed it.

> **Fill in:** how the PriceCharting price import processed large volumes (batch sizes, upserts, scheduling) and whether any caching was involved.

## Interview questions

**Q1. Where can you cache in a web application, and what goes where?**
Browser (HTTP cache and client data caches) for assets and the user's own data; CDN and reverse proxy for public assets, pages and API responses; application caches (Redis, Memcached, in-process) for expensive computations, third-party responses and fragments; the database caches hot pages itself and can precompute with materialized views. Public, stable data goes as close to the user as possible; personalized data stays in the browser or in keyed app caches.
*Follow-up:* Why not cache everything in process memory? Each server has its own copy, it disappears on deploy and invalidation across servers is hard.

**Q2. Cache-aside vs read-through vs write-through vs write-behind?**
Cache-aside: the app reads the cache, loads from the DB on a miss and fills it; writes go to the DB and delete the key. Read-through: the cache layer does the loading. Write-through: writes update cache and DB together, so cached keys stay fresh at the cost of write latency. Write-behind: writes hit the cache and flush to the DB later, which is fast but can lose data.
*Follow-up:* Which does `Rails.cache.fetch` give you? Cache-aside in a single call: the block runs only on a miss.

**Q3. How do you invalidate caches?**
Decide acceptable staleness per data type, then combine TTLs (with jitter), delete-on-write after the transaction commits, versioned keys that change when the data changes (Rails cache versions from `updated_at`, `touch: true` for nested fragments), and change events for other services.
*Follow-up:* Why delete instead of updating the cached value? Writing values can race with a slower request writing older data; a delete just forces a fresh read.

**Q4. What is a cache stampede, and how do you prevent it?**
A hot key expires and many concurrent requests miss and recompute it simultaneously, overloading the database. Use a short lock so only one request recomputes, serve the stale value meanwhile (Rails `race_condition_ttl`), refresh hot keys early in the background and add jitter to TTLs.
*Follow-up:* What is cache penetration? Repeated misses for keys that never exist; cache the negative result briefly.

**Q5. What are the Core Web Vitals, and how do you improve each?**
LCP (loading, good at 2.5 s or less): faster server response, a prioritized and properly sized hero image, no render-blocking resources. INP (responsiveness, 200 ms or less): less JavaScript, shorter tasks, cheaper handlers, transitions and virtualization. CLS (stability, 0.1 or less): dimensions and reserved space, careful font loading, transform animations. They are judged at the 75th percentile of real users.
*Follow-up:* What did INP replace, and why? FID, which only measured the first input's delay; INP covers every interaction's full latency.

**Q6. How does a CDN work, and how do you avoid stale assets after a deploy?**
Edge servers near users cache responses keyed by URL (plus configured headers and parameters) and fetch from the origin on a miss. Fingerprinted file names give every deploy new asset URLs with year-long immutable caching, while HTML is revalidated, so users get new asset references immediately without purges.
*Follow-up:* When do you purge? For content at stable URLs, by URL or by tag when the underlying data changes.

**Q7. A page loads slowly. How do you investigate and fix it?**
Measure with Lighthouse or PageSpeed Insights and the Network and Performance panels. A slow time to first byte points at the backend or caching; a late LCP at images or render-blocking resources; poor INP at long JavaScript tasks. Fix the biggest item first (cache or optimize the server response, prioritize and compress the hero image, split bundles, defer third parties), then re-measure, ideally against field data.
*Follow-up:* What if lab scores are good but users complain? Check field data by device and network; real users often run slow phones on mobile networks.

**Q8. An API endpoint is slow. How do you find and fix the bottleneck?**
Look at APM traces or logs to see where time goes (database, external calls, rendering). For the database: N+1 queries, missing indexes (`EXPLAIN ANALYZE`), unbounded result sets. Then move slow side work into background jobs, cache expensive reads, add timeouts and parallelism for external calls, and confirm with the same metrics.
*Follow-up:* Which tool shows per-request SQL in development? rack-mini-profiler, plus Bullet for N+1 detection.

**Q9. How do you serve responsive images?**
Generate several sizes in modern formats (AVIF, WebP), use `srcset` and `sizes` (or `<picture>` for format fallbacks), set `width` and `height` to reserve space, lazy-load below-the-fold images and give the hero image `fetchpriority="high"`. Resize at upload time or via an image CDN.
*Follow-up:* Why not lazy-load everything? Lazy-loading the LCP image delays the most important paint.

**Q10. gzip vs Brotli?**
Both compress text; Brotli usually produces smaller output and is supported by modern browsers over HTTPS, but its highest levels are slow, so precompress static assets at build time and use moderate levels for dynamic responses. gzip is the universal fallback. Neither should be applied to already-compressed images, fonts or video.
*Follow-up:* Which header tells caches about it? `Vary: Accept-Encoding`.

**Q11. What is connection pooling, and how do you size the pool?**
Opening database connections is expensive, so each process keeps a pool of open connections that threads borrow and return. Size the pool to the number of threads that use the database in that process (Puma threads, Sidekiq concurrency) and keep the sum across all processes under the database's connection limit, adding PgBouncer when there are many processes.
*Follow-up:* What happens when the pool is too small? Threads wait and eventually raise connection timeout errors.

**Q12. Lab data vs field data?**
Lab data (Lighthouse, WebPageTest) comes from controlled, repeatable runs, good for debugging and CI budgets. Field data (Chrome UX Report, real-user monitoring) reflects real devices, networks and behavior, and is what Core Web Vitals assessments use. Use lab data to diagnose and field data to judge.
*Follow-up:* Why can INP be fine in the lab? Lab runs rarely include real interactions, so INP mostly shows up in field data.

**Q13. What should you not cache?**
Data that must be exactly current when a decision is made (stock at checkout, balances, permissions just revoked) unless you re-check the source of truth at that moment; personalized responses at shared layers; secrets in browser storage; and anything cheap to compute that would only add invalidation risk.
*Follow-up:* How do you cache per-user data safely? Key it by user (and org), keep TTLs short and invalidate on writes.

## Further reading
- [web.dev: Web Vitals](https://web.dev/articles/vitals)
- [MDN: Web performance](https://developer.mozilla.org/en-US/docs/Web/Performance)
- [Rails guide: Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html)
- [Redis documentation](https://redis.io/docs/)
