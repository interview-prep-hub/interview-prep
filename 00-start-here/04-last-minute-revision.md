# Last-Minute Revision

> The night-before sheet: the highest-yield facts from every track in one place. Each line links to the file that explains it. If a line does not make sense to you, read that file's `Quick revision` instead of this page.

## Your story (60 seconds)

- **Intro skeleton:** present (full-stack engineer at Devntech: Rails, React, React Native, TypeScript for international clients), past (Django LMS at PureLogics, Next.js + MongoDB APIs at RemoteDev, Rails training at Visnext, TA for core CS labs), future (why this role). See [tell me about yourself](../18-behavioral-and-hr/01-tell-me-about-yourself.md).
- **Scale numbers are platform context, not personal impact:** always follow them with "my part was...". See [resume defense](../18-behavioral-and-hr/06-resume-line-by-line-defense.md).
- **STAR:** Situation (1-2 sentences), Task, Action (most of the time, "I" not "we"), Result (measurable or concrete). See [STAR stories](../18-behavioral-and-hr/02-star-stories-bank.md).

## OOP and design ([OOP](../01-cs-fundamentals/02-oop.md), [SOLID](../01-cs-fundamentals/03-solid-and-design-principles.md), [C++ OOP](../03-languages/c-cpp/03-cpp-oop.md))

- **Encapsulation** hides state behind methods that protect invariants; **abstraction** exposes what, hides how.
- **Runtime polymorphism** in C++ needs `virtual` and a pointer or reference; dispatch goes through the vtable.
- **Base classes need virtual destructors**, or deleting a derived object through a base pointer is undefined behavior.
- **Rule of 3/5/0:** if you write a destructor, copy constructor or copy assignment, you probably need all of them (plus the two move operations).
- **Composition over inheritance:** has-a is more flexible than is-a.
- **SOLID:** Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion.

## DSA in C++ ([framework](../02-dsa/00-big-o-and-problem-solving.md), [STL](../03-languages/c-cpp/04-stl-for-dsa.md))

| Signal in the problem | Try |
|---|---|
| Pairs or sums in a sorted array | Two pointers |
| Longest/shortest substring or subarray with a condition | Sliding window |
| "Seen before?", counting, grouping | Hash map or set |
| Sorted input, or "minimum value that works" | Binary search (on the answer) |
| Top K, running median, merging sorted lists | Heap |
| Next greater element, matching brackets | Monotonic stack, stack |
| Grids, shortest path in unweighted graph | BFS |
| All combinations or permutations | Backtracking |
| Optimal value with overlapping subproblems | Dynamic programming |
| Dependencies and ordering | Topological sort |

- `unordered_map` lookup: average O(1), worst O(n). `map`/`set`: O(log n), ordered (red-black tree).
- `priority_queue` is a max-heap by default; min-heap: `priority_queue<int, vector<int>, greater<int>>`.
- Sorting: O(n log n). Binary search: O(log n). BFS/DFS: O(V + E).

## Ruby ([Ruby 99](../03-languages/ruby/99-interview-questions.md))

- Only `nil` and `false` are falsy. Symbols are immutable and unique; strings are mutable objects.
- **Proc vs lambda:** lambdas check arity and `return` returns from the lambda; procs are lenient and `return` returns from the enclosing method.
- **include** adds instance methods, **extend** adds singleton (class-level) methods, **prepend** inserts the module before the class in the lookup chain.
- `==` value equality, `eql?` value and type (used by Hash), `equal?` same object, `===` case equality.
- `@x ||= value` recomputes if the value is `nil` or `false`.
- `method_missing` must be paired with `respond_to_missing?`.

## Rails ([Rails 99](../05-ruby-on-rails/99-interview-questions.md))

- **Request lifecycle:** Puma, Rack middleware, router, controller (filters, action), model, view, response.
- **Eager loading:** `preload` uses separate queries, `eager_load` uses one LEFT OUTER JOIN, `includes` picks one of them, `joins` is an INNER JOIN that loads nothing extra. N+1 = one query per row; detect with logs, Bullet or `strict_loading`.
- **Callbacks on create:** before_validation, after_validation, before_save, before_create, after_create, after_save, after_commit. Enqueue jobs in **after_commit**, never inside an open transaction.
- **Validations vs constraints:** validations give friendly errors; only database constraints (unique index, NOT NULL, foreign keys) are race-proof.
- **Strong parameters** stop mass assignment; Rails 8 adds `params.expect`.
- **Locking:** optimistic (`lock_version`, retry on conflict) for rare conflicts; pessimistic (`with_lock`, `SELECT ... FOR UPDATE`) for hot rows like stock counts.
- **Sidekiq:** jobs as JSON in Redis, threads in a separate process, 25 retries with exponential backoff over about three weeks, at-least-once delivery, so every job must be idempotent. Pass IDs, not objects.
- **Hotwire:** Turbo Drive (no full reloads), Frames (replace one part of the page), Streams (append/replace/remove over HTTP or WebSockets), Stimulus (small JS controllers).
- **Rails 8:** Solid Queue, Solid Cache, Solid Cable, Kamal 2, Propshaft by default, built-in authentication generator.

## JavaScript ([JS 99](../03-languages/javascript/99-interview-questions.md))

- `var` is function-scoped and hoisted as `undefined`; `let`/`const` are block-scoped and sit in the temporal dead zone until declared.
- **Closure:** a function remembers the variables of the scope where it was created.
- **`this`:** `new` > explicit (`call`/`apply`/`bind`) > implicit (`obj.method()`) > default; arrow functions take `this` from the enclosing scope.
- **Event loop:** run the call stack, then **all** microtasks (promise callbacks, `queueMicrotask`), then one macrotask (timers, I/O), repeat.
- `Promise.all` fails fast; `allSettled` waits for all; `race` takes the first settled; `any` takes the first fulfilled.
- `==` coerces types, `===` does not. `typeof null === "object"`. `NaN !== NaN`.
- **Debounce** waits for a pause (search input); **throttle** limits the rate (scroll handler).

## TypeScript ([TS 99](../03-languages/typescript/99-interview-questions.md))

- `interface` can be merged and extended; `type` can express unions, tuples and mapped types.
- `unknown` must be narrowed before use; `any` turns checking off; `never` is the impossible type (exhaustive checks).
- Narrowing: `typeof`, `instanceof`, `in`, discriminated unions, user-defined guards (`x is T`).
- Utility types: `Partial`, `Required`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`.
- Types vanish at runtime: validate external data with Zod (`schema.parse`, `z.infer`).
- `satisfies` checks a value against a type without widening its inferred type.

## React ([React 99](../06-react/99-interview-questions.md))

- **Re-renders** happen on state change, parent re-render or context change; reconciliation diffs by element type and key.
- **Keys** must be stable and unique among siblings; index keys break when items are reordered or inserted.
- **useEffect** syncs with external systems; list every value it uses in the dependencies and clean up subscriptions and timers. StrictMode runs effects twice in development on purpose.
- **React.memo** skips re-rendering on equal props; **useMemo** caches a value; **useCallback** caches a function. Use them when measured, not everywhere.
- **Context** re-renders every consumer when its value changes: split contexts and memoize values.
- **TanStack Query:** `staleTime` (default 0) controls freshness, `gcTime` (default 5 minutes) controls how long unused data stays cached; invalidate after mutations.
- **Server Components** run on the server with no client JavaScript; `"use client"` marks the client boundary; `"use server"` marks Server Functions.

## Next.js and TanStack ([Next.js and TanStack 99](../07-nextjs-and-tanstack/99-interview-questions.md))

- CSR (browser renders), SSR (per request), SSG (at build), ISR (static and revalidated).
- App Router: Server Components by default; `layout`, `page`, `loading`, `error` files.
- Next.js 15: `fetch` is not cached by default; `params` and `cookies()` are async.
- Server Actions are public endpoints: validate input and check authorization in every one.
- TanStack Router: type-safe routes, validated search params, loaders. TanStack Start: SSR plus `createServerFn` server functions.

## React Native and Expo ([RN 99](../08-react-native-expo/99-interview-questions.md))

- RN renders real native views. New Architecture: JSI (direct JS-native calls), Fabric (renderer), TurboModules (lazy native modules); Hermes is the default engine.
- Expo Go cannot load custom native code: use a development build. Prebuild (CNG) generates the native projects; config plugins customize them.
- Long lists: `FlatList` (or FlashList) with stable `keyExtractor`, memoized rows and `getItemLayout`, never a big `ScrollView`.
- Push: permission, Expo push token, store it on the backend, send through the Expo Push API, check receipts, delete tokens on `DeviceNotRegistered`.
- OTA updates (EAS Update) change JavaScript and assets only; native changes need a new build. `runtimeVersion` keeps updates compatible with the installed binary.
- Tokens go in SecureStore (Keychain/Keystore), not AsyncStorage.

## Node ([Node 99](../09-nodejs-backend/99-interview-questions.md))

- One JavaScript thread plus libuv: non-blocking I/O, with a thread pool for fs, crypto and dns.
- Event loop phases: timers, pending callbacks, poll, check (`setImmediate`), close. `process.nextTick` runs before promise microtasks.
- Never block the loop with CPU work: use worker threads or a queue.
- Streams handle large data piece by piece; `pipeline` handles backpressure and errors.
- Graceful shutdown on SIGTERM: stop accepting connections, finish in-flight work, close pools, exit.
- oRPC: typed procedures (input schema, handler), middleware injects context such as the current org; the client infers types end to end.

## Python and Django ([Python 99](../03-languages/python/99-interview-questions.md), [Django 99](../10-python-django/99-interview-questions.md))

- Mutable default arguments are shared between calls: use `None` and create inside.
- `is` compares identity, `==` compares value. Lists are mutable, tuples are immutable (and hashable if their items are).
- Decorators wrap functions; generators `yield` lazily; context managers guarantee cleanup with `with`.
- The GIL lets one thread run Python bytecode at a time: threads for I/O, processes for CPU, asyncio for many concurrent I/O tasks.
- `select_related` = SQL JOIN for ForeignKey/OneToOne; `prefetch_related` = separate query for many-to-many and reverse relations.
- `F()` does atomic updates in the database; `Q()` builds OR/NOT conditions.
- DRF: serializers validate and convert, ViewSets plus routers build endpoints, permission classes authorize, filter `get_queryset` by user to prevent IDOR.

## Databases ([DB 99](../12-databases/99-interview-questions.md))

- **Joins:** INNER (matches only), LEFT (all left rows), FULL (all rows from both). WHERE filters rows before grouping, HAVING filters groups after.
- **Window functions:** `ROW_NUMBER` (unique), `RANK` (gaps after ties), `DENSE_RANK` (no gaps).
- **B-tree indexes** make lookups O(log n); composite indexes follow the leftmost prefix rule; functions on a column, leading wildcards and type mismatches prevent index use.
- **ACID:** atomicity, consistency, isolation, durability. Postgres defaults to Read Committed, MySQL InnoDB to Repeatable Read.
- **Anomalies:** dirty read, non-repeatable read, phantom read, lost update, write skew.
- **Normalization:** 1NF atomic values, 2NF no partial dependency on part of a key, 3NF no transitive dependency.
- **MongoDB:** embed what you read together and is bounded; reference what grows without limit or is shared.
- **Redis:** cache, sessions, rate limits, queues (Sidekiq), pub/sub, locks; set TTLs and plan invalidation.

## Web ([web 99](../04-web-fundamentals/99-interview-questions.md))

- GET, PUT and DELETE are idempotent; POST is not; PATCH is not guaranteed to be.
- **401** = not authenticated, **403** = authenticated but not allowed, **404** = not found, **409** = conflict, **422** = validation failed, **429** = rate limited.
- Cookie flags: `Secure`, `HttpOnly`, `SameSite`. CORS is enforced by browsers; preflight `OPTIONS` checks non-simple requests.
- Sessions are easy to revoke; JWTs are stateless but hard to revoke, so keep access tokens short and rotate refresh tokens.
- OAuth 2.0 for SPAs and mobile: authorization code flow with PKCE.
- **XSS** injects script into your page (defense: escaping, CSP); **CSRF** forges a request with the victim's cookies (defense: SameSite, CSRF tokens).
- Pagination: offset is simple but slow and unstable deep down; cursor (keyset) is fast and stable.

## Architecture ([architecture 99](../13-architecture-and-practices/99-interview-questions.md))

- **Multi-tenancy:** database per tenant (strongest isolation), schema per tenant, shared schema with `tenant_id` (cheapest, most common). Enforce scope centrally, include the tenant in unique indexes, carry it into jobs and cache keys, and test for cross-tenant leaks.
- **Strangler fig:** replace a legacy system piece by piece behind a router; avoid dual writes, prefer one source of truth plus sync.
- **Idempotency** makes retries safe: idempotency keys, upserts on external IDs, state checks.
- **Transactional outbox:** write the event in the same transaction as the data; a relay publishes it later.
- **Webhooks:** verify the HMAC signature, dedupe by event ID, respond fast, process async.
- **Overselling:** atomic decrements or row locks, reservations, and fast stock sync across channels.
- **Money:** integer minor units (cents), never floats; split remainders deterministically.

## DevOps ([DevOps 99](../14-devops-and-tools/99-interview-questions.md))

- **Merge** keeps history and adds a merge commit; **rebase** rewrites your commits on top (never rebase shared branches).
- **reset** moves the branch pointer (rewrites local history); **revert** adds a new commit that undoes one (safe on shared branches). `git reflog` recovers "lost" commits.
- **Image** = read-only template built from a Dockerfile; **container** = running instance. `ENTRYPOINT` is the executable, `CMD` its default arguments.
- **Zero-downtime deploys:** rolling or blue-green releases plus expand/contract migrations.
- **Debugging production:** check recent deploys, logs, metrics and error tracking; mitigate (roll back) first, then find the root cause.

## The morning of the interview

- [ ] Re-read this page and the `Quick revision` of the role's top 5 files
- [ ] Say your intro out loud twice
- [ ] Solve one easy DSA problem to warm up
- [ ] Re-read the job description and the company's product page; prepare two specific questions
- [ ] Check camera, mic, internet backup, quiet room (or route and timing for onsite)
- [ ] Water, notepad, a copy of your resume
