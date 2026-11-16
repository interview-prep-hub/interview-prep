# Ruby Advanced Topics and Internals

> **Tier:** T1 (know cold) · **Related:** [OOP, modules and mixins](03-oop-modules-mixins.md), [Metaprogramming](05-metaprogramming.md), [Ruby basics](01-basics.md), [Operating systems](../../01-cs-fundamentals/05-operating-systems.md), [Background jobs with Sidekiq](../../05-ruby-on-rails/09-background-jobs-sidekiq.md), [Caching, performance and security](../../05-ruby-on-rails/14-caching-performance-security.md)
> **On your resume:** Sidekiq background jobs, Puma-served Rails apps and third-party API error handling at Devntech (CrystalCommerce, Colochousing): the internals that explain thread-safety bugs, memory growth, flaky retries and slow requests.

## Quick revision
- Object model: `Class` inherits from `Module`, which inherits from `Object` (which mixes in `Kernel`), which inherits from `BasicObject`. Every class is an instance of `Class`; class methods live in singleton classes that inherit from the superclass's singleton class.
- CRuby's GC is mark-and-sweep, generational (minor and major collections), incremental, with optional compaction. Fewer allocations means less GC time.
- Frozen string literals cut allocations. Ruby 3.4 "chills" unannotated literals: mutation still works but can warn, a step toward frozen-by-default.
- `@x ||= compute` recomputes whenever the cached value is `nil` or `false`; use `defined?(@x)` to memoize falsy results.
- Bare `rescue` catches `StandardError`. Order clauses from specific to general; `else` runs on success, `ensure` always runs, `retry` restarts `begin`, bare `raise` re-raises. Never `rescue Exception`.
- Custom errors inherit from `StandardError`, usually through one base class per library. Raising inside `rescue` records the original as `cause`.
- `catch`/`throw` is non-exception control flow; Rails callbacks halt with `throw(:abort)`.
- The GVL lets one thread run Ruby code at a time per process (per Ractor). It is released during blocking I/O, so threads help I/O-bound work, not CPU-bound work. Ruby threads are real OS threads.
- Production concurrency: Puma runs worker processes times threads; Sidekiq runs job threads in a separate process. Mutable class-level state is shared across those threads.
- Fibers are cooperative and lightweight; the fiber scheduler (3.0) enables async I/O. Ractors (3.0) are experimental parallelism with isolated objects.
- Pattern matching (`case/in`): array, hash and find patterns, guards, pin `^`, alternatives `|`, binding `=>`. Hash patterns match symbol keys only. Custom classes implement `deconstruct` and `deconstruct_keys`.
- Endless methods (`def area = w * h`, 3.0) and hash or keyword shorthand (`{ x:, y: }`, 3.1).
- YJIT is CRuby's JIT (3.1, production-ready 3.2); Rails 7.2 enables it by default on supported Ruby versions.
- Bundler: the `Gemfile` declares ranges, `Gemfile.lock` pins exact versions. `~> 7.1` means `>= 7.1, < 8.0`; `~> 7.1.3` means `>= 7.1.3, < 7.2`.
- `<=>` returns -1, 0, 1 or `nil` (not comparable) and powers `sort`, `min`, `max` and `Comparable`.

## 1. The object model

```ruby
class Foo; end

Foo.class                       # => Class
Foo.superclass                  # => Object
Class.superclass                # => Module
Module.superclass               # => Object
Object.superclass               # => BasicObject
BasicObject.superclass          # => nil
Class.class                     # => Class
Foo.singleton_class             # => #<Class:Foo>
Foo.singleton_class.superclass  # => #<Class:Object>
```

```text
BasicObject          minimal: ==, equal?, !, instance_eval, __send__, method_missing
  +-- Object         mixes in Kernel (puts, require, raise, loop, ...)
        +-- Module   namespaces and mixins: include, define_method, const_get
              +-- Class   adds new, allocate, superclass
```

- An object stores its instance variables and a pointer to its class; methods live in classes and modules.
- A class is itself an object (an instance of `Class`), so it can hold instance variables (class instance variables) and singleton methods (class methods).
- Class methods are inherited because singleton classes mirror the class hierarchy:

```ruby
class Parent
  def self.build = new
end
class Child < Parent; end

Child.singleton_class.superclass == Parent.singleton_class # => true
Child.build.class                                         # => Child
```

`BasicObject` is the base for clean proxies: with almost no methods defined, nearly every call reaches `method_missing`. Ruby's `Delegator` (the parent of `SimpleDelegator`) inherits from it.

## 2. Garbage collection and allocation awareness

How CRuby collects garbage:
- **Mark and sweep:** starting from roots (globals, constants, thread stacks), mark every reachable object, then sweep (free) the unmarked ones.
- **Generational:** most objects die young. New objects start young and survivors are promoted to old. Minor GCs scan only young objects (fast); major GCs scan everything. Write barriers track old-to-young references so minor GCs stay correct.
- **Incremental:** major marking is split into small steps interleaved with your program, shortening pauses.
- **Compaction:** `GC.compact` (2.7) moves objects together to reduce fragmentation and improve copy-on-write sharing after `fork`; `GC.auto_compact = true` (3.0) enables it automatically.
- Objects live in slots inside heap pages; large strings and arrays also hold `malloc`ed buffers outside the slot.

```ruby
before = GC.stat(:total_allocated_objects)
1_000.times { |i| "sku-#{i}" }
GC.stat(:total_allocated_objects) - before   # at least 1,000 new objects
GC.stat(:minor_gc_count)                     # how many minor collections so far
GC.stat(:major_gc_count)
ObjectSpace.count_objects[:T_STRING]         # live String slots
```

| Allocation-heavy | Leaner |
|---|---|
| `str += "x"` in a loop (a new String every time) | `str << "x"` on a mutable buffer, or `join` |
| `hash.merge(k => v)` inside `reduce` | `each_with_object({})`, assigning into the memo hash |
| `map { }.flatten` | `flat_map` |
| `Order.all.each` | `Order.find_each` |
| `orders.map(&:id)` on a relation | `orders.pluck(:id)` |
| big intermediate arrays in long chains | `lazy`, or push the work into SQL |
| unannotated string literals in hot code | `# frozen_string_literal: true` |

Memory that grows in a Rails process is often fragmentation rather than a true leak; common mitigations are jemalloc and `MALLOC_ARENA_MAX=2` with glibc. Real leaks usually come from unbounded caches in constants or class variables, closures retaining objects, and subscriber lists that only grow. The `memory_profiler` and `derailed_benchmarks` gems help find allocation hot spots.

## 3. Frozen string literals and chilled strings

```ruby
# frozen_string_literal: true
a = "price"
b = "price"
a.frozen?         # => true
a.equal?(b)       # => true: identical frozen literals are deduplicated
a << "s"          # FrozenError
copy = +"price"   # an unfrozen copy you can mutate
```

Ruby 3.4 made literals in files **without** the magic comment "chilled". `frozen?` still returns `false` and mutation still works, but it emits a deprecation warning when deprecation warnings are enabled (`-W:deprecated` or `Warning[:deprecated] = true`). It is the migration path toward frozen literals by default; check the release notes of your Ruby version for the current status. Enable deprecation warnings in the test suite to find mutations early, and use `+"..."` or `String.new` where you really need a mutable string. An explicit `# frozen_string_literal: false` keeps the old behavior.

## 4. Memoization with `||=` and the falsy trap

```ruby
def current_store
  @current_store ||= Store.find(params[:store_id])   # @current_store || (@current_store = ...)
end
```

`||=` assigns whenever the current value is falsy, so `nil` and `false` are never cached:

```ruby
def new_checkout?
  @new_checkout ||= FeatureFlag.enabled?(:new_checkout)  # false: queries on every call
end

def new_checkout?
  return @new_checkout if defined?(@new_checkout)         # caches false and nil too
  @new_checkout = FeatureFlag.enabled?(:new_checkout)
end

def market_price(sku)                                     # memoizing per argument
  @market_prices ||= {}
  return @market_prices[sku] if @market_prices.key?(sku)  # handles nil prices
  @market_prices[sku] = PriceApi.fetch(sku)
end
```

Where the memo lives matters. On a per-request object (controller, service, job instance) it dies with the object. On a class or in a constant it is shared by every thread and lives for the whole process: stale data, unbounded growth and races. Use `Rails.cache` with an expiry for cross-request caching and `ActiveSupport::CurrentAttributes` for per-request values.

## 5. Exceptions

```text
Exception
  NoMemoryError
  ScriptError          LoadError, NotImplementedError, SyntaxError
  SecurityError
  SignalException      Interrupt (Ctrl-C)
  SystemExit           raised by exit
  SystemStackError     "stack level too deep"
  StandardError        what a bare rescue catches
    ArgumentError
    EncodingError
    IOError            EOFError
    IndexError         KeyError, StopIteration
    LocalJumpError
    NameError          NoMethodError
    RangeError         FloatDomainError
    RuntimeError       FrozenError; raise "message" creates a RuntimeError
    SystemCallError    Errno::ECONNREFUSED, Errno::ENOENT, ...
    ThreadError
    TypeError
    ZeroDivisionError
```

```ruby
def charge(order)
  attempts = 0
  begin
    attempts += 1
    gateway.charge!(order.total_cents, idempotency_key: order.payment_key)
  rescue Gateway::CardDeclined => e     # most specific first: the first match wins
    order.mark_declined!(e.message)
  rescue Gateway::Timeout
    retry if attempts < 3               # re-runs the begin block
    raise                               # bare raise re-raises the current exception
  rescue Gateway::Error => e            # parent class of the two above
    Rails.logger.error("gateway error: #{e.class}: #{e.message}")
    raise
  else
    order.mark_paid!                    # runs only when nothing was raised
  ensure
    order.release_lock!                 # runs whenever control leaves, success or failure
  end
end
```

(`Gateway` stands for any payment client.) Method bodies and `do ... end` blocks (Ruby 2.5+) accept `rescue` without `begin`:

```ruby
def parse(json)
  JSON.parse(json)
rescue JSON::ParserError
  {}
end

items.each do |item|
  import(item)
rescue ImportError => e
  errors << [item, e.message]
end
```

**Custom errors.**

```ruby
module PriceSync
  class Error < StandardError; end          # one base class per library or domain

  class RateLimited < Error
    attr_reader :retry_after

    def initialize(retry_after:)
      @retry_after = retry_after
      super("rate limited, retry after #{retry_after}s")
    end
  end

  class BadResponse < Error; end
end

begin
  JSON.parse(body)
rescue JSON::ParserError
  raise PriceSync::BadResponse, "invalid JSON from price API"  # the original becomes #cause
end
```

Callers can `rescue PriceSync::Error` to handle the whole family or `PriceSync::RateLimited` to read `retry_after` and reschedule.

**Why never `rescue Exception`:** it also catches `Interrupt` and `SignalException` (Ctrl-C, SIGTERM during a deploy), `SystemExit` (from `exit`), `NoMemoryError` and `ScriptError` (a syntax error in a required file). Processes then ignore shutdown requests, so Puma and Sidekiq cannot stop gracefully, and genuine bugs vanish. Sidekiq goes further and signals job threads to stop with an exception that inherits from `Interrupt`, precisely so ordinary `rescue => e` blocks do not swallow it. Rescue `StandardError` or something narrower. Also avoid the `value = risky rescue nil` modifier, which hides every `StandardError`, and never `return` from an `ensure` block: it silently discards the exception in flight.

**catch and throw** jump out of deeply nested code without using exceptions:

```ruby
first_negative = catch(:found) do
  [[3, 5], [7, -2], [-9, 1]].each do |row|
    row.each { |cell| throw :found, cell if cell.negative? }
  end
  nil                                   # the value when nothing is thrown
end
first_negative # => -2
```

Rails uses the same mechanism for callbacks: `throw(:abort)` inside a `before_save` halts the chain and makes `save` return `false` (since Rails 5, returning `false` no longer halts).

## 6. Concurrency: GVL, threads, fibers and Ractors

**The GVL (Global VM Lock).** CRuby lets only one thread execute Ruby code at a time (per Ractor). A thread releases the lock while it waits on blocking I/O (database queries, HTTP calls, file reads, `sleep`), and many C extensions release it during long operations. So threads give real concurrency for I/O-bound work, which is most of a web request, but no speedup for CPU-bound Ruby code. Ruby threads are native OS threads; JRuby and TruffleRuby have no GVL at all.

```ruby
require "net/http"

urls = %w[https://example.com/a https://example.com/b https://example.com/c]
threads = urls.map do |url|
  Thread.new { Net::HTTP.get_response(URI(url)).code }   # waiting on the network releases the GVL
end
threads.map(&:value)   # joins each thread and returns its block's value
```

For many items, bound the concurrency with a pool (for example `Concurrent::FixedThreadPool.new(5)` from `concurrent-ruby`, already a Rails dependency) instead of one thread per item.

**Thread safety.**

```ruby
class Counter
  attr_reader :count

  def initialize
    @count = 0
    @lock = Mutex.new
  end

  def increment
    @lock.synchronize { @count += 1 }   # read, add, write must not interleave
  end
end
```

`@count += 1` is a read-modify-write sequence; a thread switch between the read and the write loses updates, GVL or not. Thread-safe building blocks: `Mutex`, `Queue`, and from `concurrent-ruby`, `Concurrent::Map` and `Concurrent::AtomicFixnum`.

The most common real bug is class-level state:

```ruby
class TenantContext
  class << self
    attr_accessor :current   # BUG: one value shared by every thread in the process
  end
end
```

Two Puma threads serving different organizations overwrite each other's value, leaking data across tenants. `ActiveSupport::CurrentAttributes` isolates values per request (per thread or fiber) and resets them automatically:

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :organization, :user
end

# in a controller before_action
Current.organization = current_user.organization
```

**Fibers** are cooperative coroutines: they switch only at explicit points, are cheap to create, and run on a single thread. External enumerators (`next`) are built on them.

```ruby
fiber = Fiber.new do
  puts "step 1"
  Fiber.yield              # pause and give control back
  puts "step 2"
end
fiber.resume               # prints step 1
fiber.resume               # prints step 2
```

Ruby 3.0 added a fiber scheduler interface, so libraries such as the `async` gem (and the Falcon server) can turn blocking I/O into non-blocking I/O and serve many connections from one thread.

**Ractors** (Ruby 3.0, experimental) are actor-style units with their own GVL, so they run Ruby code in parallel. They share only immutable ("shareable") objects and communicate by passing messages. Most gems are not Ractor-safe and the API has changed in recent versions, so they are rare in Rails apps; check the release notes before relying on them.

| | Puma (web) | Sidekiq (jobs) |
|---|---|---|
| Model | cluster mode: forked worker processes, each with a thread pool | a process with a pool of job threads (the `concurrency` setting); run more processes to scale |
| Parallel Ruby execution | across processes, each with its own GVL | across processes; threads overlap on I/O |
| Key settings | `workers`, `threads min, max`, `preload_app!` | `concurrency`, queues and weights |
| Classic pitfalls | connections opened before `fork` must be reopened; DB pool smaller than the thread count | DB pool smaller than concurrency; non-idempotent jobs (delivery is at-least-once); shared class-level state |

```ruby
# config/puma.rb
workers Integer(ENV.fetch("WEB_CONCURRENCY", 2))     # processes
threads_count = Integer(ENV.fetch("RAILS_MAX_THREADS", 3))
threads threads_count, threads_count                 # min and max threads per process
preload_app!                                         # load the app before forking (copy-on-write)
```

```yaml
# config/database.yml: give each process at least one connection per thread
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

The OS-level background (processes, threads, context switches) is in [Operating systems](../../01-cs-fundamentals/05-operating-systems.md).

### Going deeper: thread-local vs fiber-local

`Thread.current[:key]` is actually fiber-local, while `Thread.current.thread_variable_get(:key)` is truly thread-local. Under a fiber-based server, values stored with `Thread.current[]` do not leak between fibers, but code that assumes "one request per thread" can break. `CurrentAttributes` hides this choice behind Rails' isolation level setting.

## 7. Pattern matching

```ruby
response = { status: 200, body: { items: [{ sku: "A1", price: 10 }] } }

case response
in { status: 200, body: { items: [] } }
  puts "no items"
in { status: 200, body: { items: [{ sku: String => sku }, *] } }
  puts "first sku #{sku}"                 # this branch matches and binds sku
in { status: 429 | 503 => code }
  puts "retry later (#{code})"
in { status: 400..599 => code }
  puts "failed with #{code}"
end
# prints: first sku A1
```

- **Hash patterns** match a subset of keys (extra keys are fine) and only work with **symbol keys**, so parse JSON with `symbolize_names: true` first. `**nil` means "no other keys allowed".
- **Array patterns** match the exact length unless you add `*rest`.
- **Find pattern** (Ruby 3.0): `in [*, { role: "admin" } => admin, *]` finds the first matching element anywhere.
- `=> name` binds, `|` gives alternatives, `if` adds a guard, and `^var` pins an existing value instead of rebinding: `in { status: ^expected }`.
- No matching branch and no `else` raises `NoMatchingPatternError`.

One-line forms:

```ruby
config = { db: { user: "app", port: 5432 } }
config in { db: { port: Integer } }   # => true (a boolean test)
config => { db: { user: } }           # binds user = "app"; raises if it does not match
user                                  # => "app"
```

Custom classes join in by defining `deconstruct` (array patterns) and `deconstruct_keys` (hash patterns). `Struct` and `Data` provide both.

```ruby
class Money
  attr_reader :cents, :currency

  def initialize(cents, currency)
    @cents = cents
    @currency = currency
  end

  def deconstruct_keys(_keys)
    { cents: cents, currency: currency }
  end
end

case Money.new(500, "PKR")
in { currency: "PKR", cents: } if cents < 1_000
  puts "small PKR amount: #{cents}"       # prints small PKR amount: 500
end

Point = Data.define(:x, :y)
case Point.new(x: 0, y: 5)
in { x: 0, y: } then puts "on the y-axis at #{y}"   # prints on the y-axis at 5
in Point(x:, y:) then puts "at #{x}, #{y}"
end
```

Pattern matching shines when handling third-party API responses and webhook payloads with several shapes.

## 8. Modern syntax: endless methods and shorthand

```ruby
def area(width, height) = width * height     # endless method (3.0)
def full_name = "#{first_name} #{last_name}"

name = "Ali"
role = :admin
user = { name:, role: }       # => {name: "Ali", role: :admin}  (3.1 shorthand)
invite(name:, role:)          # the same shorthand for keyword arguments
```

Endless methods cannot be setters (`def price=(v) = ...` is a syntax error) and should stay one simple expression. Always use parentheses when calling a method with keyword shorthand; without them, a call at the end of a line can absorb the next line as an argument.

## 9. YJIT

- A JIT compiler built into CRuby (written in Rust) that compiles hot methods to machine code at runtime. Added in 3.1, production-ready in 3.2.
- Enable it with `ruby --yjit` or `RUBY_YJIT_ENABLE=1`; Rails 7.2 turns it on by default on supported Ruby versions. Check with `RubyVM::YJIT.enabled?`.
- It speeds up Ruby-heavy work (view rendering, serialization, business logic) at the cost of extra memory. Time spent waiting on the database or APIs does not change.
- Measure your own app's latency and memory before and after rather than trusting headline numbers.

## 10. Bundler, gems and versioning

```ruby
# Gemfile
source "https://rubygems.org"

gem "rails", "~> 7.1.3"          # >= 7.1.3 and < 7.2
gem "pg", "~> 1.5"               # >= 1.5 and < 2.0
gem "sidekiq", ">= 7.0"          # open-ended: also allows future majors (risky)
gem "bootsnap", require: false   # installed, but not required automatically

group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end
```

```bash
bundle install           # resolve versions, install, write Gemfile.lock
bundle update sidekiq    # move one gem (and what it needs) within the constraints
bundle outdated          # list gems with newer releases
bundle exec rspec        # run with exactly the locked versions
bundle add faraday       # add a gem to the Gemfile and install it
```

- Semantic versioning is MAJOR.MINOR.PATCH: majors may break APIs, minors add features, patches fix bugs (in principle).
- The pessimistic operator `~>` lets only the last listed segment grow: `~> 2.3` allows any 2.x from 2.3 up, `~> 2.3.0` allows 2.3.x only.
- Commit `Gemfile.lock` for applications so every machine and deploy installs identical versions. A gem's own `.gemspec` declares version ranges for its dependencies instead.
- `bundle exec` makes a command use the locked versions rather than whatever is installed globally.
- Native-extension gems (`pg`, `nokogiri`) compile or ship platform-specific binaries, which is why lockfiles list platforms and why Docker or CI builds sometimes fail on them.
- `bundler-audit` (or GitHub's Dependabot) flags gems with known vulnerabilities. Rails upgrade mechanics are in [Legacy codebases and upgrades](../../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md).

## 11. The spaceship operator

```ruby
1 <=> 2             # => -1
2 <=> 2             # => 0
"b" <=> "a"         # => 1
[1, 2] <=> [1, 3]   # => -1 (element by element)
1 <=> "a"           # => nil: not comparable
[3, nil, 1].sort    # raises ArgumentError: nil cannot be compared with an Integer
[3, nil, 1].sort_by { |v| v || Float::INFINITY } # => [1, 3, nil]
```

`sort` calls `<=>` on pairs; `sort_by` computes each key once, then sorts by keys, which is faster when the key is expensive. Defining `<=>` and including `Comparable` gives your class all comparison operators (see [OOP, modules and mixins](03-oop-modules-mixins.md)).

## Common pitfalls
- `rescue Exception`, or a bare `rescue nil` modifier that hides real bugs.
- Rescuing a parent class before its subclasses, making the specific clause unreachable.
- `return` inside `ensure`, which discards the exception being raised.
- `@x ||= ...` for values that can legitimately be `nil` or `false`.
- Memoizing or caching in class variables or constants: shared across threads, never expires, grows forever.
- Assuming the GVL makes `@count += 1` or `hash[key] ||= value` atomic.
- A database pool smaller than Puma threads or Sidekiq concurrency, causing connection timeouts under load.
- Hash patterns with string keys (they must be symbols) and no `else`, causing `NoMatchingPatternError`.
- Open-ended version constraints (`>=`) that pull in breaking major versions on `bundle update`.
- Running commands without `bundle exec` and silently using different gem versions.

## How to talk about this in an interview
- Concurrency questions are where Rails developers separate themselves. Explain the GVL in one sentence, then connect it to how your apps run: Puma processes and threads for web requests, Sidekiq threads for jobs, and why class-level state and connection pool sizes matter.
- Error handling connects directly to integration work: transient vs permanent errors from a third-party API, custom error classes, retries with backoff and idempotency.
- > **Fill in:** how many Puma workers and threads and how much Sidekiq concurrency your Devntech projects run (or where those settings live), and whether you ever tuned them.
- > **Fill in:** how errors from the PriceCharting API were classified and retried in your sync jobs, and what happened to records that kept failing.
- > **Fill in:** whether your projects run with YJIT and the frozen string literal comment, and which Ruby version they use.

## Interview questions

**Q1. What is the GVL, and does Ruby have real threads?**
Yes, CRuby threads are real native OS threads, but the Global VM Lock lets only one of them execute Ruby code at a time per process (per Ractor). The lock is released during blocking I/O, so threads overlap while waiting on the database, Redis or HTTP, which is most of a web request. CPU-bound Ruby code does not get faster with threads; use processes (Puma workers, more Sidekiq processes) for that.
*Follow-up:* Do JRuby or TruffleRuby have a GVL? No, their threads run Ruby code in parallel.

**Q2. How do Puma workers and threads relate, and how do you size them?**
Workers are forked processes, each with its own GVL and memory; threads inside a worker share memory and overlap on I/O. Total concurrency is workers times threads. Common practice is roughly one worker per CPU core and a small thread count (recent Rails versions generate a default of 3; many older apps use 5), then adjusting by measuring latency and memory, with the database pool at least equal to the threads per process.
*Follow-up:* What does `preload_app!` do? Loads the app before forking so workers share memory pages through copy-on-write.

**Q3. How do you keep Rails code thread-safe?**
Avoid mutable class-level state and constants; keep request data in local variables, instance variables of per-request objects or `ActiveSupport::CurrentAttributes`. When shared state is unavoidable, protect it with a `Mutex` or use `concurrent-ruby` structures. Also make sure gems you rely on are thread-safe, and size connection pools to match threads.
*Follow-up:* Why is `@@cache[key] ||= value` unsafe? The check and the assignment can interleave between threads.

**Q4. Walk through `begin`, `rescue`, `else`, `ensure` and `retry`.**
`begin` holds the risky code. `rescue` clauses are tried top to bottom and the first matching class wins, so specific classes go first. `else` runs only when nothing was raised, `ensure` runs no matter how control leaves (success, exception or `return`), and `retry` inside a `rescue` restarts the `begin` block, so it needs a counter. A bare `raise` inside `rescue` re-raises the current exception.
*Follow-up:* What does a bare `rescue` catch? `StandardError` and its subclasses.

**Q5. Why should you never `rescue Exception`?**
It catches non-application errors: `Interrupt` and other signals, `SystemExit`, `NoMemoryError` and `ScriptError`. Your process can then ignore Ctrl-C or SIGTERM during deploys and hide syntax errors. Rescue `StandardError` or, better, the specific classes you can handle.
*Follow-up:* What happens to Sidekiq jobs during shutdown? Sidekiq interrupts them with an `Interrupt` subclass and pushes unfinished jobs back to Redis; rescuing `Exception` would break that.

**Q6. How do you design custom exceptions for a library or integration?**
Create one base class inheriting from `StandardError` (`PriceSync::Error`) and specific subclasses (`RateLimited`, `BadResponse`, `NotFound`) that carry useful data such as `retry_after`. Wrap low-level errors from HTTP or JSON libraries so callers depend only on your classes, and let Ruby keep the original as `cause`.
*Follow-up:* Why distinguish transient from permanent errors? Transient ones (timeouts, 429, 503) should be retried; permanent ones (404, validation) should not.

**Q7. What is the problem with `@x ||= compute`?**
`||=` only assigns when the current value is falsy, so a computed `nil` or `false` is never cached and the expensive call repeats on every access. Use `return @x if defined?(@x)` (or a Hash with `key?` for per-argument memoization).
*Follow-up:* Where should you not memoize? In class-level or constant storage, which is shared across threads and never expires.

**Q8. How does garbage collection work in CRuby?**
Mark and sweep from roots, made generational (minor GCs scan young objects, major GCs scan everything), incremental (marking split into steps to shorten pauses) and optionally compacting. Allocation rate drives GC frequency, so reducing temporary objects (frozen literals, `find_each`, `pluck`, `flat_map`) reduces GC time.
*Follow-up:* How do you inspect it? `GC.stat`, `GC.count`, `ObjectSpace.count_objects` and the `memory_profiler` gem.

**Q9. A Rails process's memory keeps growing. How do you investigate?**
First distinguish fragmentation from leaks: watch memory per worker over time, try jemalloc or `MALLOC_ARENA_MAX=2`, and see whether growth plateaus. For leaks, look for unbounded class-level caches, registries and closures, and profile allocations of suspect endpoints or jobs with `memory_profiler` or `derailed_benchmarks`. Batch large queries with `find_each`.
*Follow-up:* Why does loading a huge result set hurt even after the request ends? The heap grows to fit it and may not shrink back.

**Q10. What does `frozen_string_literal: true` do, and what changed in Ruby 3.4?**
It freezes every string literal in the file, so mutation raises `FrozenError` and identical literals are shared, saving allocations. Ruby 3.4 made unannotated literals "chilled": they still mutate, but mutation emits a deprecation warning when those warnings are enabled, preparing for frozen literals by default.
*Follow-up:* How do you get a mutable string in such a file? `+"text"`, `String.new("text")` or `"text".dup`.

**Q11. Show a practical use of pattern matching.**
Matching API responses by shape: `in { status: 200, body: { items: [first, *] } }` binds `first`, `in { status: 429 | 503 => code }` handles retryable errors, and a guard (`in { amount: Integer => a } if a > 0`) validates values. It replaces nested `dig` calls and conditionals with one readable `case`.
*Follow-up:* Why parse JSON with `symbolize_names: true` first? Hash patterns only match symbol keys.

**Q12. How do you make your own class work with pattern matching?**
Define `deconstruct` returning an Array for array patterns and `deconstruct_keys(keys)` returning a Hash for hash patterns. The `keys` argument lists the requested keys, so expensive objects can compute only what is needed. `Struct` and `Data` implement both automatically.
*Follow-up:* What does `in Point(x:, y:)` check? That the value is a `Point` (via `===`) and then matches its keys.

**Q13. Threads vs fibers vs Ractors?**
Threads are preemptively scheduled OS threads that share memory and overlap on I/O under the GVL. Fibers are cooperative coroutines on one thread that switch only when told to, made practical for async I/O by the fiber scheduler. Ractors are experimental isolated actors with their own GVL that run Ruby code in parallel but share only immutable objects.
*Follow-up:* Which do Rails apps use? Threads (Puma, Sidekiq) and processes; fibers through specific libraries; Ractors rarely.

**Q14. What is YJIT, and should you enable it?**
CRuby's built-in JIT compiler (added in 3.1, production-ready in 3.2), which compiles frequently executed methods to machine code. It usually reduces CPU time in Ruby-heavy code at the cost of more memory, and Rails 7.2 enables it by default on supported Rubies. Enable it, then measure latency and memory on your own workload.
*Follow-up:* How do you check it is on? `RubyVM::YJIT.enabled?`.

**Q15. What does `~>` mean in a Gemfile, and why commit `Gemfile.lock`?**
`~>` is the pessimistic constraint: only the last specified segment may grow, so `~> 7.1` allows 7.x from 7.1 up and `~> 7.1.3` allows 7.1.x from 7.1.3 up. The lockfile pins the exact versions resolved, so every developer, CI run and deploy uses the same dependency tree.
*Follow-up:* What does `bundle update` without arguments do? Re-resolves every gem to the newest versions allowed, which is risky in a big app.

**Q16. In Ruby's object model, what is the class of a class, and where do class methods live?**
Every class is an instance of `Class`, and `Class` is an instance of itself. Class methods live in the class's singleton class, which inherits from the superclass's singleton class, so class methods are inherited along the normal hierarchy.
*Follow-up:* What is at the top of the hierarchy? `BasicObject`, whose superclass is `nil`.

**Q17. When would you use `catch`/`throw` instead of exceptions?**
For non-error control flow such as escaping nested loops or callback chains once a result is found; it is cheaper and clearer than abusing exceptions. Rails halts callback chains with `throw(:abort)`.
*Follow-up:* What happens if nothing catches the symbol? Ruby raises `UncaughtThrowError`.

## Further reading
- [Exception class hierarchy (Ruby docs)](https://docs.ruby-lang.org/en/master/Exception.html)
- [Pattern matching (Ruby docs)](https://docs.ruby-lang.org/en/master/syntax/pattern_matching_rdoc.html)
- [Threading and code execution in Rails](https://guides.rubyonrails.org/threading_and_code_execution.html)
