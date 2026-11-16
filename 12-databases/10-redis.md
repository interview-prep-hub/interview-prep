# Redis

> **Tier:** T2 (working knowledge) · **Related:** [Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [Rails caching, performance and security](../05-ruby-on-rails/14-caching-performance-security.md), [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Realtime and WebSockets](../04-web-fundamentals/09-realtime-websockets.md)
> **On your resume:** Redis is in your skills list and backs Sidekiq, which runs background jobs in Colochousing's Rails app at Devntech (CrystalCommerce also relies on background jobs for the PriceCharting sync).

## Quick revision
- Redis is an **in-memory data structure server**: keys map to strings, hashes, lists, sets, sorted sets, streams and more. Sub-millisecond operations.
- Fast because data lives in RAM, commands run on a **single thread** (no locks, each command atomic), structures are compact, and the protocol is simple. Optional I/O threads help with network work, not command execution.
- `EXPIRE` / `SET ... EX` set a time to live; `TTL` returns seconds left (-1 no expiry, -2 no key). A plain `SET` on an existing key clears its TTL.
- `maxmemory-policy`: `noeviction` (default; writes fail when full), `allkeys-lru` / `allkeys-lfu` for caches, `volatile-*` only evict keys with a TTL. **Sidekiq needs `noeviction`.**
- Persistence: **RDB** snapshots (compact, can lose minutes) and **AOF** append log (`everysec` loses about 1 s); both, or none for a pure cache.
- Use cases: cache-aside, sessions, rate limiting (INCR + EXPIRE, or sorted sets), job queues (Sidekiq, BullMQ), pub/sub, distributed locks (`SET key token NX PX ms`), leaderboards (sorted sets), counters, unique counts (HyperLogLog).
- Cache invalidation: TTL, delete after commit, versioned keys (`cache_key_with_version`), namespace version bumps. Stampedes: locking, `race_condition_ttl`, early refresh, TTL jitter.
- Pub/sub is fire-and-forget (offline subscribers miss messages); **Streams** keep history with consumer groups and acknowledgements.
- Never run `KEYS *` in production (blocks the single thread); use `SCAN`.
- Redis vs Memcached: Redis has data types, persistence, replication and scripting; Memcached is a multi-threaded string cache.
- 2024 license change led to the **Valkey** fork (BSD, Linux Foundation); licensing changed again later, so check current terms.

## 1. What Redis is and why it is fast

Redis keeps the whole dataset in memory and serves commands over TCP. Each command executes on one main thread, one at a time, so a command like `INCR` is atomic without any locking, and there is no lock contention overhead. Persistence (snapshots and log rewrites) happens in forked child processes.

The flip side: one slow command (`KEYS *`, `SMEMBERS` on a set with millions of members, a heavy Lua script) blocks every other client. Throughput comes from many tiny commands, pipelining (sending several commands without waiting for each reply), and scaling out with replicas or Redis Cluster.

Grouping commands: `MULTI` ... `EXEC` runs a queue of commands atomically (no rollback if one fails at runtime), `WATCH` adds optimistic concurrency, and Lua scripts (`EVAL`) run atomically on the server.

## 2. Data types and commands

```bash
# Strings: values, counters, flags, serialized JSON
SET user:42:name "Ali"
SET otp:42 "839201" EX 300 NX         # only if absent, expires in 5 minutes
GET otp:42
INCR page:views:2025-06-15            # atomic counter
MGET user:42:name user:43:name

# Hashes: field-value maps (an object without serialization)
HSET user:42 name "Ali" city "Lahore" plan "pro"
HGET user:42 city
HINCRBY user:42 logins 1

# Lists: ordered, push/pop at both ends (queues, recent items)
LPUSH recent:42 "listing:9"
LTRIM recent:42 0 49                  # keep only the 50 newest
BRPOP queue:emails 5                  # block up to 5 seconds for an item

# Sets: unique members, set algebra
SADD house:7:members 42 43 44
SISMEMBER house:7:members 42
SINTER house:7:members event:9:attendees

# Sorted sets: members ordered by a score (rankings, schedules, sliding windows)
ZADD leaderboard 120 "user:42" 95 "user:43"
ZINCRBY leaderboard 10 "user:43"
ZRANGE leaderboard 0 9 REV WITHSCORES # top 10 (Redis 6.2+; ZREVRANGE on older servers)
ZREVRANK leaderboard "user:42"        # 0-based position from the top

# Streams: append-only log with consumer groups
XGROUP CREATE orders:events billing $ MKSTREAM
XADD orders:events * type paid order_id 1001
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS orders:events >
XACK orders:events billing 1718000000000-0
```

| Type | Typical use |
|---|---|
| String | Cache entries, counters, feature flags, OTPs, locks |
| Hash | Objects with fields (session data, per-user settings), memory-efficient small maps |
| List | Simple queues, capped "recent activity" lists |
| Set | Unique members: tags, who is online, deduplication |
| Sorted set | Leaderboards, delayed jobs by timestamp, sliding-window rate limits |
| Stream | Durable event logs, work distribution with acknowledgements |
| Bitmap (`SETBIT`, `BITCOUNT`) | Daily active users by user id, compact flags |
| HyperLogLog (`PFADD`, `PFCOUNT`) | Approximate unique counts (about 0.81% error) in at most 12 KB per key |

## 3. Expiry: EXPIRE and TTL

```bash
EXPIRE session:abc 1800      # seconds; PEXPIRE for milliseconds, EXPIREAT for a Unix time
TTL session:abc              # remaining seconds; -1 = no expiry, -2 = key does not exist
PERSIST session:abc          # remove the expiry
SET session:abc "new data" KEEPTTL     # overwrite the value but keep the expiry (Redis 6.0+)
```

Expired keys are removed lazily (when accessed) and by a background sampler, so memory is reclaimed gradually. Gotcha: `SET` on an existing key removes its TTL unless you pass a new `EX` or `KEEPTTL` (Redis 6.0+). Add random jitter to cache TTLs so thousands of keys written together do not expire together.

## 4. Eviction policies

When memory reaches `maxmemory`, `maxmemory-policy` decides what happens:

| Policy | Behavior | Fits |
|---|---|---|
| `noeviction` (default) | Writes fail with an out-of-memory error | Queues, Sidekiq, sessions: data you must not lose |
| `allkeys-lru` | Evict the least recently used key | General caches |
| `allkeys-lfu` | Evict the least frequently used key | Caches with stable hot sets |
| `volatile-lru` / `volatile-lfu` | Same, but only among keys with a TTL | Mixed cache and persistent data (better: separate instances) |
| `allkeys-random` / `volatile-random` | Random eviction | Rarely |
| `volatile-ttl` | Evict keys closest to expiring | Rarely |

LRU and LFU are approximated by sampling. Because a cache wants eviction and a job queue must never evict, production setups usually run **separate Redis instances** (or databases on separate servers) for cache and Sidekiq.

## 5. Persistence: RDB vs AOF

| | RDB snapshots | AOF (append-only file) |
|---|---|---|
| How | A forked child writes a point-in-time dump on a schedule (`BGSAVE`) | Every write is appended to a log; replayed on restart; periodically rewritten to compact it |
| Data loss on crash | Everything since the last snapshot (often minutes) | About 1 second with `appendfsync everysec`; none with `always` (slow) |
| Restart time | Fast (load one file) | Slower (replay), improved by an RDB preamble |
| Size | Compact | Larger |

Pure caches can disable persistence; queues and sessions should use AOF (often together with RDB). Forking for snapshots or rewrites uses copy-on-write memory, so leave headroom.

## 6. Use cases

### 6.1 Cache-aside, key naming, invalidation and stampedes

Cache-aside (lazy loading): read the cache; on a miss, load from the database, store with a TTL, return.

```ruby
def product_card(id)
  Rails.cache.fetch("v1/product_card/#{id}", expires_in: 10.minutes, race_condition_ttl: 10.seconds) do
    Product.find(id).slice(:id, :name, :price_cents)        # runs only on a miss
  end
end
```

```ts
import Redis from "ioredis";

type ProductCard = { id: string; name: string; priceCents: number };
declare function loadProductCard(id: string): Promise<ProductCard>;  // your database query

const redis = new Redis(process.env.REDIS_URL!);

export async function getProductCard(id: string): Promise<ProductCard> {
  const key = `app:v1:product_card:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached) as ProductCard;

  const card = await loadProductCard(id);
  const ttl = 600 + Math.floor(Math.random() * 60);               // jitter
  await redis.set(key, JSON.stringify(card), "EX", ttl);
  return card;
}
```

**Key naming**: colon-separated namespaces (`app:v1:product_card:42`), a tenant prefix in multi-tenant apps (`org:7:...`), and a version segment you can bump to invalidate a whole family of keys at once.

**Invalidation strategies**:
- **TTL only**: simplest; staleness bounded by the TTL.
- **Delete on write**: delete the key when the source changes, **after the transaction commits** (`after_commit`), otherwise a concurrent reader can re-cache the old value.
- **Write-through**: update the cache together with the database.
- **Key-based expiration**: put the record's version in the key (`cache_key_with_version` uses `updated_at`), so a change produces a new key and old entries simply age out; with `touch: true` on associations, this powers Rails Russian-doll caching.
- **Namespace bump**: increment `v1` to `v2` to drop everything.

**Stampede** (thundering herd): a hot key expires and hundreds of requests recompute it at once, hammering the database. Mitigations: one recomputer holds a short lock (`SET lock NX PX`) while others wait or serve stale data; Rails `race_condition_ttl` serves the stale value briefly while one process refreshes; refresh early (before expiry) for hot keys; add TTL jitter; pre-warm caches after deploys.

### 6.2 Sessions
Store `session:<random id>` with the session data and a TTL, refreshed on each request (sliding expiration). Server-side sessions can be revoked instantly by deleting the key, unlike stateless JWTs. In Rails, `config.session_store :cache_store` keeps sessions in `Rails.cache` (for example Redis); in Express, `express-session` with a Redis store does the same. See [authentication](../04-web-fundamentals/06-authentication-authorization.md).

### 6.3 Rate limiting
**Fixed window** with `INCR` + `EXPIRE`:
```ruby
def allow_request?(redis, user_id, limit: 100, window: 60)
  key = "rate:#{user_id}:#{Time.now.to_i / window}"   # one key per user per window
  count, _ttl_set = redis.multi do |tx|
    tx.incr(key)
    tx.expire(key, window)
  end
  count <= limit
end
```
Simple and cheap, but allows bursts of up to twice the limit around window boundaries.

**Sliding window log** with a sorted set (score = timestamp):
```ts
export async function allowRequest(userId: string, limit = 100, windowMs = 60_000) {
  const key = `rate:sliding:${userId}`;
  const now = Date.now();
  const results = await redis
    .multi()
    .zremrangebyscore(key, 0, now - windowMs)       // drop requests older than the window
    .zadd(key, now, `${now}-${Math.random()}`)       // record this request (unique member)
    .zcard(key)                                      // requests inside the window
    .pexpire(key, windowMs)
    .exec();
  const count = Number(results?.[2]?.[1] ?? 0);      // ioredis returns [error, reply] pairs
  return count <= limit;
}
```
Accurate but stores one entry per request; at high volume use a sliding-window counter or a token bucket in a Lua script. In Rails, `rate_limit` in controllers (7.2+) and the Rack::Attack gem both store counters in the cache store, which can be Redis.

### 6.4 Queues, and how Sidekiq uses Redis
A basic queue is `LPUSH` + `BRPOP`; a more reliable one moves items atomically to a processing list (`LMOVE`/`BLMOVE`) and removes them after success.

Sidekiq:
- `perform_async` serializes the job (class, JSON arguments, job id, queue, retry settings) and pushes it onto a Redis **list** such as `queue:default`.
- Sidekiq server processes run worker **threads** that block on `BRPOP` across their queues and execute jobs.
- `perform_in` / `perform_at` jobs wait in the `schedule` **sorted set** (score = run time); failures go to the `retry` sorted set with exponential backoff, then to the `dead` set. A poller moves due jobs back into queues.
- Delivery is effectively **at least once** (retries rerun jobs), so jobs must be idempotent; on a hard crash the open-source basic fetch can lose in-flight jobs, which Sidekiq Pro's reliable fetch addresses.
- Pass ids and simple JSON values as arguments, not objects; the record may have changed by the time the job runs.
- Run Sidekiq's Redis with `noeviction` and persistence enabled.

In Node, BullMQ is the common Redis-backed queue. Rails 8's Solid Queue is a database-backed alternative that removes Redis from the stack. See [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md).

### 6.5 Pub/sub and streams
```bash
SUBSCRIBE house:7:chat           # this connection now only receives messages
PUBLISH house:7:chat "hello"     # returns how many subscribers received it
```
Pub/sub delivers to currently connected subscribers only: no storage, no acknowledgement, at most once. That is fine for fan-out of realtime events: Action Cable's Redis adapter uses it so a broadcast from any server process reaches WebSocket clients connected to every process. When consumers must not miss events, use **Streams** with consumer groups (`XREADGROUP` + `XACK`) or a real broker.

### 6.6 Distributed locks
```ruby
RELEASE_SCRIPT = <<~LUA
  if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  end
  return 0
LUA

def with_redis_lock(redis, key, ttl_ms: 30_000)
  token = SecureRandom.hex(16)
  return false unless redis.set(key, token, nx: true, px: ttl_ms)   # someone else holds it
  begin
    yield
  ensure
    redis.eval(RELEASE_SCRIPT, keys: [key], argv: [token])          # only delete our own lock
  end
  true
end
```
`SET ... NX PX` acquires atomically with an expiry (so a crashed holder cannot block forever); the token prevents deleting a lock that expired and was taken by someone else. Caveats: if the work outlives the TTL (a GC pause, a slow API), a second process gets in. The **Redlock** algorithm (acquire on a majority of independent Redis nodes) is debated because it relies on timing assumptions and offers no fencing tokens. Rule: Redis locks are fine to avoid duplicate work (efficiency); for correctness (money, stock), rely on database constraints, row locks or advisory locks ([06](06-transactions-and-concurrency.md)).

### 6.7 Leaderboards
A sorted set per period (`leaderboard:2025-06`) with `ZINCRBY` on each scoring event, `ZRANGE ... REV WITHSCORES` for the top N and `ZREVRANK` for a user's position, all O(log n). Set an `EXPIRE` on old periods. Ties are ordered by member name.

## 7. Redis in Rails and Node

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV["REDIS_CACHE_URL"], expires_in: 1.hour }

# config/initializers/sidekiq.rb
Sidekiq.configure_server { |config| config.redis = { url: ENV["REDIS_URL"] } }
Sidekiq.configure_client { |config| config.redis = { url: ENV["REDIS_URL"] } }
```

```yaml
# config/cable.yml (Action Cable)
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") %>
  channel_prefix: myapp_production
```

```ts
// node-redis
import { createClient } from "redis";

const client = createClient({ url: process.env.REDIS_URL });
client.on("error", (err) => console.error("Redis error", err));
await client.connect();
await client.set("otp:42", "839201", { EX: 300, NX: true });
const otp = await client.get("otp:42");
```
ioredis (`new Redis(url)`, positional options like `"EX", 60`) is the other popular Node client and the one BullMQ uses.

## 8. Redis vs Memcached, and Valkey

| | Redis | Memcached |
|---|---|---|
| Values | Strings, hashes, lists, sets, sorted sets, streams, more | Opaque strings only |
| Persistence | RDB and AOF | None |
| Replication and failover | Replicas, Sentinel, Redis Cluster | None built in (clients shard keys) |
| Execution | Single-threaded commands (optional I/O threads) | Multi-threaded |
| Extras | Pub/sub, transactions, Lua, TTL per key, eviction policies | TTL and LRU eviction |
| Max value | 512 MB per string | 1 MB per item by default |
| Pick it for | Caching plus queues, locks, counters, rankings | A simple, very large, horizontally sharded cache |

**Valkey**: in 2024 Redis moved from the BSD license to source-available licenses; the Linux Foundation-backed Valkey fork continued from Redis 7.2 under BSD, and major cloud providers offer it. It speaks the same protocol, so clients (and Sidekiq, which works with Redis-compatible stores) generally work unchanged. Redis licensing changed again afterwards, so check the current terms before choosing.

## 9. Production concerns
- Always set `maxmemory` and a policy that matches the data; separate cache and queue instances.
- Never `KEYS *` (O(n), blocks everyone): iterate with `SCAN`. Watch for big keys (`redis-cli --bigkeys`) and slow commands (`SLOWLOG GET 10`).
- Size connection pools: web threads plus Sidekiq concurrency plus Action Cable all hold connections.
- Managed Redis usually requires TLS (`rediss://` URLs) and has connection limits.
- Treat Redis as a cache that can be lost unless you deliberately configured persistence; never keep the only copy of business data there.

### Going deeper: replication, Sentinel and Cluster
Replicas copy the primary asynchronously (so a failover can lose the last writes). **Sentinel** monitors a primary and its replicas and promotes a replica when the primary fails, telling clients the new address. **Redis Cluster** shards keys across nodes using 16,384 hash slots; commands touching several keys must hit the same slot, which you force with hash tags such as `{user:42}:cart` and `{user:42}:wishlist` (only the part in braces is hashed).

## Common pitfalls
- Running cache and Sidekiq on one instance with an eviction policy: under memory pressure, queued jobs are silently evicted.
- Deleting cache keys inside a transaction (before commit), letting readers re-cache stale data.
- Caching without TTLs or a size limit until memory runs out.
- `KEYS` or other O(n) commands on large keysets in production.
- Expecting pub/sub to deliver to subscribers that were offline.
- Using a Redis lock as the only guard for payments or stock.
- Non-idempotent Sidekiq jobs (sending an email twice on retry), or passing whole objects as job arguments.
- Assuming `SET` keeps an existing TTL.
- Storing large JSON blobs under one key and rewriting them on every small change; use hashes or smaller keys.

## How to talk about this in an interview
- Your direct Redis exposure is through Sidekiq on Colochousing's Rails app: say what Sidekiq stores in Redis (queues as lists, scheduled and retry sets as sorted sets) and why jobs must be idempotent.
- Then show breadth: caching with `Rails.cache` on Redis, rate limiting, locks for "one sync per store at a time", Action Cable fan-out.
- Be honest about operations: if hosting and tuning were handled by others, say so and explain what you would check (eviction policy, persistence, memory).

> **Fill in:** where Redis runs for the Devntech projects (managed service or self-hosted) and whether it is used for anything besides Sidekiq (cache store, Action Cable, rate limiting).

> **Fill in:** one Sidekiq job you wrote or changed, and how you made it safe to retry.

## Interview questions

**Q1. What is Redis, and why is it so fast?**
An in-memory key-value store whose values are data structures (strings, hashes, lists, sets, sorted sets, streams). It is fast because everything is in RAM, commands execute on a single thread without locking, data structures are compact and specialized, and the protocol is lightweight. Persistence runs in forked background processes.

**Q2. Redis is single-threaded. Why is that not a bottleneck, and what does it imply?**
Most commands take microseconds, so one thread handles a very high request rate, and network I/O can be offloaded to I/O threads in recent versions. The implications: every command is atomic, but one slow command blocks all clients, so avoid O(n) commands on large keys and heavy scripts. Scale with replicas or Cluster.

**Q3. Which Redis data type would you use for a leaderboard, unique visitors, a job queue, a user session and tags?**
Leaderboard: sorted set. Exact unique visitors: set (or HyperLogLog for approximate counts in tiny memory). Job queue: list (or stream for acknowledgements). Session: hash or string with a TTL. Tags: set, with `SINTER` for "items with both tags".

**Q4. Explain cache-aside and your options for cache invalidation.**
Cache-aside: read from cache, on a miss load from the database and store with a TTL. Invalidation: TTL expiry, deleting keys after the database commit, write-through updates, versioned keys that change with `updated_at` (Rails `cache_key_with_version`), or bumping a namespace version. Choose by how stale the data may be.
*Follow-up:* Why delete after commit? Otherwise a concurrent request can read the old row and re-cache it right after you deleted the key.

**Q5. What is a cache stampede, and how do you prevent it?**
When a popular key expires, many requests miss simultaneously and all recompute it, overloading the database. Prevent it with a short lock so only one request recomputes, serving stale data during refresh (Rails `race_condition_ttl`), refreshing hot keys before expiry, and adding jitter to TTLs.

**Q6. What eviction policy would you use for a cache, and for Sidekiq?**
A cache: `allkeys-lru` or `allkeys-lfu` with `maxmemory` set, so old entries make room for new ones. Sidekiq: `noeviction`, because evicting a queue or retry set loses jobs; it is better for writes to fail loudly. Run them on separate instances.

**Q7. RDB or AOF?**
RDB takes periodic snapshots: compact, fast restarts, but you lose writes since the last snapshot. AOF logs every write: with `everysec` you lose at most about a second, at the cost of bigger files and slower restarts. Durable data (queues, sessions) uses AOF, often with RDB as well; a pure cache can skip persistence.

**Q8. How would you build a rate limiter with Redis?**
Fixed window: `INCR` a per-user, per-window key and `EXPIRE` it in the same `MULTI`; reject above the limit. It allows bursts at window edges. Sliding window: a sorted set of request timestamps; remove old entries, add the current one, count, all atomically. For high volume, a token bucket in a Lua script.

**Q9. How does Sidekiq use Redis?**
Jobs are JSON payloads pushed onto Redis lists (one per queue); Sidekiq processes run threads that pop jobs with blocking reads and execute them. Scheduled jobs, retries and dead jobs live in sorted sets keyed by time. Because retries can rerun a job, jobs must be idempotent, and Redis must not evict keys.

**Q10. Pub/sub or Streams?**
Pub/sub broadcasts to subscribers connected right now, with no storage or acknowledgement, so it suits realtime fan-out (Action Cable). Streams store an append-only log, support consumer groups, acknowledgements and replay, so they suit work that must not be lost.

**Q11. How do you implement a distributed lock with Redis, and what are the caveats?**
`SET lock:key <random token> NX PX <ttl>` acquires atomically with an expiry, and release uses a Lua script that deletes only if the token still matches. The lock can expire while the holder is still working (pauses, slow calls), and Redlock's safety depends on timing assumptions. Use Redis locks to avoid duplicate work; protect correctness with database constraints or locks.

**Q12. Redis vs Memcached?**
Memcached is a simple multi-threaded cache of opaque strings with no persistence or replication. Redis adds rich data types, persistence, replication and failover, pub/sub, streams and scripting, so it covers caching plus queues, locks and counters. For a Rails app, Redis (or a Rails 8 Solid alternative) is the usual choice.
*Follow-up:* What is Valkey? A BSD-licensed fork of Redis created in 2024 after Redis's license change, protocol-compatible and offered by major clouds.

**Q13. Why should you never run `KEYS *` in production?**
It scans every key in one blocking O(n) command, freezing the single command thread for all clients on large datasets. Use `SCAN` with a cursor (and `MATCH`), which returns small batches without blocking.

**Q14. What happens to a key's TTL when you overwrite it?**
A plain `SET` removes the existing TTL, so the key lives forever unless you pass `EX`/`PX` again or use `KEEPTTL` (Redis 6.0+). Commands that modify a value in place, such as `INCR` or `HSET`, keep the TTL.

## Further reading
- Redis data types: https://redis.io/docs/latest/develop/data-types/
- Key eviction: https://redis.io/docs/latest/develop/reference/eviction/
- Persistence: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
