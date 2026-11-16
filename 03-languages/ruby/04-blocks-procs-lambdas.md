# Ruby Blocks, Procs and Lambdas

> **Tier:** T1 (know cold) · **Related:** [Ruby basics](01-basics.md), [Collections and Enumerable](02-collections-enumerable.md), [Metaprogramming](05-metaprogramming.md), [Routing and controllers](../../05-ruby-on-rails/02-routing-and-controllers.md), [JavaScript closures](../javascript/02-scope-closures-hoisting.md)
> **On your resume:** Blocks run through all the Rails code you write at Devntech: transactions, callbacks, scopes, `find_each` batches in background jobs, RSpec, and retry or timing wrappers around third-party APIs such as PriceCharting.

## Quick revision
- A block is code passed to a method with `{ }` or `do ... end`. It is not an object until you capture it.
- `yield` runs the block, `block_given?` checks for one, and a `&block` parameter captures it as a Proc.
- `{ }` binds to the nearest method call and `do ... end` to the outermost one, so `puts arr.map do ... end` hands the block to `puts`.
- Procs (and blocks) are lenient about arguments, and `return` inside them returns from the enclosing method. Lambdas check arity, and `return` exits only the lambda.
- `->(x) { }` and `lambda { }` make lambdas; `proc { }` and `Proc.new { }` make procs.
- Closures capture variables, not values: changes are visible in both directions.
- `&obj` in a call passes `obj.to_proc` as the block: `&:upcase` (Symbol#to_proc), `&method(:puts)`, `&some_lambda`.
- `next value` returns a value from a block; `break value` exits the method that yielded.
- `curry` partially applies a lambda; `>>` and `<<` compose callables (Ruby 2.6).
- Short blocks can use `_1` (Ruby 2.7) or `it` (Ruby 3.4) instead of a named parameter.
- `then` returns the block's result; `tap` yields the object and returns the object.
- Wrapper methods (retry, timing, tenant switching) combine `yield` with `ensure` for guaranteed cleanup.
- Rails examples: `transaction`, `respond_to`, `form_with`, `around_action`, `scope :x, -> { }`, `Rails.cache.fetch { }`, `find_each`.

## 1. Blocks, yield and block_given?

```ruby
def with_banner
  puts "start"
  result = yield              # run the caller's block and keep its value
  puts "end"
  result
end

with_banner { 21 * 2 }        # prints start and end, returns 42

def each_pair_of(items)
  return to_enum(:each_pair_of, items) unless block_given?  # no block: hand back an Enumerator
  items.each_cons(2) { |a, b| yield a, b }                  # yield can pass arguments
end

each_pair_of([1, 2, 3]) { |a, b| puts "#{a}->#{b}" }  # prints 1->2 and 2->3
each_pair_of([1, 2, 3]).to_a                          # => [[1, 2], [2, 3]]
```

Every method can receive one block, even though it does not appear in the signature. Calling `yield` when no block was given raises `LocalJumpError` (no block given). Returning an Enumerator when there is no block keeps your method chainable, exactly like the built-in iterators.

### Braces vs do/end

```ruby
puts [1, 2, 3].map { |x| x * 2 }     # prints 2, 4, 6: braces bind to map
puts [1, 2, 3].map do |x| x * 2 end  # prints #<Enumerator: [1, 2, 3]:map>
                                     # do/end binds to puts, so map got no block
```

The convention is braces for one-line blocks used for their value and `do ... end` for multi-line blocks run for side effects. Precedence bites in DSLs that omit parentheses: in RSpec, `expect { checkout }.to change { Order.count }.by(1)` uses braces so the block attaches to `change` rather than `to`.

## 2. Capturing a block with &block

```ruby
class EventBus
  def initialize
    @handlers = Hash.new { |hash, key| hash[key] = [] }
  end

  def on(event, &handler)            # capture the block to call it later
    @handlers[event] << handler
  end

  def emit(event, payload)
    @handlers[event].each { |handler| handler.call(payload) }
  end
end

bus = EventBus.new
bus.on(:order_paid) { |order| puts "email receipt for #{order[:id]}" }
bus.emit(:order_paid, { id: 42 })    # prints: email receipt for 42
```

`&` works in both directions. In a parameter list it turns the block into a Proc; in a method call it turns a Proc (or anything with `to_proc`) back into a block.

```ruby
shout = proc { |s| s.upcase }
%w[a b].map(&shout)                  # => ["A", "B"]

def forward(&block) = [1, 2].map(&block)  # pass the caller's block along
forward { |n| n * 10 }               # => [10, 20]
```

Prefer `yield` when you only need to call the block; capture it with `&block` when you must store it, pass it to another method, or call it later. Capturing historically allocated a Proc on every call, and recent versions optimize the pass-through case.

## 3. Procs vs lambdas

```ruby
square  = ->(x) { x * x }        # lambda, "stabby" syntax
square2 = lambda { |x| x * x }   # lambda
loose   = proc { |x| x }         # proc
loose2  = Proc.new { |x| x }     # proc

square.call(3)   # => 9
square.(3)       # => 9
square[3]        # => 9
square.lambda?   # => true
loose.lambda?    # => false
```

**Arity.**

```ruby
pr = proc { |a, b| "#{a.inspect}, #{b.inspect}" }
pr.call(1)         # => "1, nil"   missing arguments become nil
pr.call(1, 2, 3)   # => "1, 2"     extra arguments are dropped
pr.call([1, 2])    # => "1, 2"     a single array is auto-splatted

la = ->(a, b) { "#{a}, #{b}" }
la.call(1)         # ArgumentError (wrong number of arguments (given 1, expected 2))
```

**Return.**

```ruby
def run_proc
  pr = proc { return 10 }   # return leaves run_proc itself
  pr.call
  20
end

def run_lambda
  la = -> { return 10 }     # return leaves only the lambda
  la.call
  20
end

run_proc    # => 10
run_lambda  # => 20

def make_proc = proc { return 1 }
make_proc.call  # LocalJumpError: the method it would return from has already returned
```

| | Proc (`proc {}`, `Proc.new {}`, and blocks) | Lambda (`-> {}`, `lambda {}`) |
|---|---|---|
| Arity | lenient: missing args are `nil`, extras dropped, one array auto-splats | strict: a wrong count raises `ArgumentError` |
| `return` | returns from the method where the proc was defined | returns from the lambda only |
| `break` | exits the method that yielded (`LocalJumpError` if orphaned) | behaves like `return` |
| `next` | ends this call of the block, with an optional value | same |
| `lambda?` | `false` | `true` |
| Mental model | a code snippet inlined into the surrounding method | an anonymous method |
| Typical use | blocks and DSL bodies | callbacks, strategies, scopes, anything stored and called later |

## 4. Closures and bindings

```ruby
counter = 0
increment = -> { counter += 1 }  # captures the variable itself, not a copy of its value
increment.call
increment.call
counter           # => 2

def make_counter
  count = 0
  -> { count += 1 }               # count survives after make_counter returns
end
c1 = make_counter
c1.call           # => 1
c1.call           # => 2
make_counter.call # => 1: a fresh closure with its own count
```

Block parameters are new for every iteration, so the classic JavaScript `var`-in-a-loop bug does not happen with `each`. It does happen with `for`, which does not create a scope:

```ruby
procs = [1, 2, 3].map { |i| -> { i } }
procs.map(&:call)    # => [1, 2, 3]

procs = []
for i in 1..3
  procs << -> { i }  # every lambda shares the single variable i
end
procs.map(&:call)    # => [3, 3, 3]
```

A `Binding` is a captured execution context (local variables plus `self`):

```ruby
require "erb"

def template_context
  user = "Ali"
  binding
end

b = template_context
b.local_variable_get(:user)            # => "Ali"
ERB.new("Hi <%= user %>").result(b)    # => "Hi Ali"
```

`binding.irb` opens a console at the current line, and the `debug` gem's `debugger` (or `binding.break`) gives you a full breakpoint. Closures keep everything they capture alive, so a lambda stored in a long-lived registry can retain large objects and leak memory.

## 5. Method objects

```ruby
m = 10.method(:+)                  # a Method bound to the receiver 10
m.call(5)                          # => 15
m.owner                            # => Integer

[1, 2, 3].each(&method(:puts))     # passes Kernel#puts as the block

def tax(cents) = cents * 17 / 100
[100, 200].map(&method(:tax))      # => [17, 34]

um = String.instance_method(:upcase)   # UnboundMethod: no receiver yet
um.bind("hi").call                     # => "HI"
"hi".method(:upcase).source_location   # => nil (implemented in C)
```

`method(:name).source_location` returns `[file, line]` for Ruby-defined methods, and `parameters` describes the signature:

```ruby
def create(name, role = :member, *tags, email:, **opts, &blk); end
method(:create).parameters
# => [[:req, :name], [:opt, :role], [:rest, :tags], [:keyreq, :email], [:keyrest, :opts], [:block, :blk]]
```

## 6. Symbol#to_proc

```ruby
%w[a b].map(&:upcase)          # same as map { |s| s.upcase }
orders.sum(&:total)            # same as sum { |o| o.total }
:upcase.to_proc.call("hi")     # => "HI"
```

`&` calls `to_proc` on the symbol, which returns a proc that sends that method name to its first argument. It cannot pass arguments (`map(&:round(2))` is a syntax error), so use a block, `_1` or `it` when you need them.

## 7. Currying and composition

```ruby
add = ->(a, b) { a + b }
add5 = add.curry[5]              # fix the first argument
add5[3]                          # => 8

apply_discount = ->(percent, cents) { cents - cents * percent / 100 }
ten_off = apply_discount.curry[10]
[1_000, 2_500].map(&ten_off)     # => [900, 2250]

strip  = ->(s) { s.strip }
upcase = ->(s) { s.upcase }
normalize = strip >> upcase      # strip first, then upcase
normalize.call("  sku-1 ")       # => "SKU-1"
(upcase << strip).call(" a ")    # => "A": << composes right to left
```

Currying is rare in Rails code, but interviewers use it to check that you understand closures and partial application.

## 8. Numbered parameters and `it`

```ruby
[1, 2, 3].map { _1 * 2 }           # => [2, 4, 6]   Ruby 2.7+
{ a: 1 }.map { "#{_1}=#{_2}" }     # => ["a=1"]
[1, 2, 3].map { it * 2 }           # => [2, 4, 6]   Ruby 3.4+
```

They only work in blocks without an explicit parameter list. Use them for short, obvious blocks and name the parameter once the block grows beyond one line.

## 9. then and tap

```ruby
# then (alias yield_self): pass the value in, get the block's result out
"1,299".delete(",").then { |s| Integer(s) }.then { |n| n * 2 }  # => 2598

# tap: yield the object, return the object unchanged
user = User.new.tap do |u|
  u.name = "Ali"
  u.role = :admin
end

[3, 1, 2].sort.tap { |arr| Rails.logger.debug(arr.inspect) }.first  # peek mid-chain
```

## 10. Writing your own iterators and wrappers

### A custom iterator

```ruby
class Batch
  def initialize(items, size:)
    @items = items
    @size = size
  end

  def each_batch
    return enum_for(:each_batch) unless block_given?
    @items.each_slice(@size).with_index(1) { |slice, number| yield slice, number }
  end
end

Batch.new((1..5).to_a, size: 2).each_batch { |slice, n| puts "batch #{n}: #{slice}" }
# batch 1: [1, 2]
# batch 2: [3, 4]
# batch 3: [5]
```

### Retry with backoff

```ruby
require "net/http"

def with_retries(attempts: 3, on: [Net::OpenTimeout, Net::ReadTimeout], base_delay: 0.5)
  tries = 0                                # must live outside begin: retry re-runs begin
  begin
    tries += 1
    yield tries
  rescue *on
    raise if tries >= attempts             # give up and re-raise the last error
    sleep(base_delay * 2**(tries - 1))     # exponential backoff: 0.5s, 1s, 2s
    retry
  end
end

price = with_retries { |attempt| PriceApi.fetch("ZELDA-01") }  # PriceApi is a placeholder client
```

Retry only idempotent operations (a GET, an upsert keyed by external id), never "charge the card" without an idempotency key. Inside background jobs, keep in-process retries small and let the job system handle the rest: Sidekiq retries failed jobs with exponential backoff.

### Timing wrapper

```ruby
def measure(label)
  started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  yield                                    # the method returns the block's value
ensure
  elapsed_ms = (Process.clock_gettime(Process::CLOCK_MONOTONIC) - started) * 1000
  Rails.logger.info("#{label} took #{elapsed_ms.round(1)}ms")
end

listings = measure("listing-sync") { SyncListings.call(store) }
```

Use the monotonic clock for durations; `Time.now` can jump when the system clock is adjusted.

### Context wrappers (the "sandwich" pattern)

```ruby
def with_tenant(organization)
  previous = Current.organization          # Current < ActiveSupport::CurrentAttributes
  Current.organization = organization
  yield
ensure
  Current.organization = previous          # restored even when the block raises
end

with_tenant(org) { ImportListings.call(org) }
```

The same shape appears in `File.open(path) { |f| ... }` (closes the file), `Mutex#synchronize`, `I18n.with_locale(:fr) { }`, `Time.use_zone("Europe/Brussels") { }` and `ActiveRecord::Base.transaction { }`.

### Configuration blocks

```ruby
class PriceClient
  Config = Struct.new(:base_url, :timeout, :api_key)

  def self.config
    @config ||= Config.new("https://api.example.com", 5, nil)
  end

  def self.configure
    yield config
  end
end

PriceClient.configure do |c|
  c.timeout = 10
  c.api_key = ENV.fetch("PRICE_API_KEY")
end
```

This is the shape of `Sidekiq.configure_server do |config| ... end`. `Rails.application.configure do ... end` instead evaluates the block with a different `self`, so you write `config.x` without a block parameter (see [Metaprogramming](05-metaprogramming.md)).

## 11. How Rails uses blocks

```ruby
# Commit if the block finishes, roll back and re-raise if it raises
ActiveRecord::Base.transaction do
  order.update!(status: :paid)
  order.line_items.each { |item| item.listing.decrement!(:quantity, item.quantity) }
end

# respond_to yields a format collector
def show
  @order = Order.find(params[:id])
  respond_to do |format|
    format.html
    format.json { render json: @order }
  end
end

# around_action: the action itself arrives as the block
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    I18n.with_locale(params[:locale] || I18n.default_locale, &action)
  end
end

# Scopes take lambdas so the body runs at call time
scope :recent, -> { where(created_at: 1.week.ago..) }

# The block runs only on a cache miss
Rails.cache.fetch("market-price/#{sku}", expires_in: 1.hour) { PriceApi.fetch(sku) }

# Yields records in batches of 1,000 instead of loading the table
Listing.where(store: store).find_each { |listing| SyncListingJob.perform_later(listing.id) }
```

```erb
<%= form_with model: @listing do |form| %>
  <%= form.text_field :title %>
  <%= form.number_field :quantity %>
  <%= form.submit %>
<% end %>
```

Why lambdas for scopes: if the body were evaluated once while the class loads, `1.week.ago` would be frozen at boot time and the query would silently go stale. Modern Rails raises an error unless the scope body is callable. `ActiveRecord::Rollback` raised inside `transaction` rolls back without propagating; any other exception rolls back and re-raises.

## Common pitfalls
- `do ... end` attaching to the outer method (`puts`, `expect(...).to`) instead of the inner call.
- `return` inside a block or proc leaving the whole enclosing method, sometimes skipping important code after the loop.
- Expecting `break` to stop only the block: it returns from the method that yielded.
- A `for` loop where every closure shares one variable.
- A retry helper that resets its counter inside `begin` and loops forever, or retries a non-idempotent call.
- A context wrapper without `ensure`, leaking a tenant, locale or time zone into the next request served by the same thread.
- Storing long-lived closures in class-level registries and retaining large objects.
- `_1` or `it` in long blocks where a named parameter would document intent.

## How to talk about this in an interview
- Blocks are a good vehicle for reliability stories: a retry-with-backoff wrapper around a flaky third-party API, a timing wrapper around a slow sync, a tenant-switching wrapper with `ensure` in a multi-tenant app.
- When you explain proc vs lambda, lead with the two differences (arity and `return`), then give the "inlined snippet vs anonymous method" mental model.
- > **Fill in:** whether your PriceCharting integration retried failed calls in-process or relied on Sidekiq retries, and how duplicate work was prevented.
- > **Fill in:** a feature where a `transaction` block protected multi-step writes in the Colochousing API (for example around payments or rental applications), or where an `around_action` set context per request.

## Interview questions

**Q1. What is a block, and how is it different from a Proc?**
A block is a chunk of code attached to a method call; it is syntax, not an object, and a method can receive only one. A Proc is an object that wraps a block so it can be stored, passed around and called later. `&block` converts a block into a Proc, and `&proc_obj` converts back.
*Follow-up:* How many blocks can a method take? One, but it can accept any number of procs or lambdas as ordinary arguments.

**Q2. How do `yield` and `block_given?` work?**
`yield args` calls the block passed to the current method and evaluates to the block's return value. `block_given?` returns true if a block was passed, so methods can behave differently (built-in iterators return an Enumerator). Yielding without a block raises `LocalJumpError`.
*Follow-up:* What does `each` return without a block? An Enumerator, which is why `each.with_index` works.

**Q3. What are the differences between a proc and a lambda?**
Arity: lambdas raise `ArgumentError` on the wrong number of arguments, procs fill missing ones with `nil`, drop extras and auto-splat a single array. Return: `return` in a lambda exits the lambda, while `return` in a proc exits the method where the proc was defined. A lambda behaves like an anonymous method, a proc like a snippet inlined into its surroundings.
*Follow-up:* Which does `->(x) { }` create? A lambda; `proc { }` and `Proc.new { }` create procs.

**Q4. What happens when you `return` inside a block passed to `each`?**
The block behaves like a proc, so `return` exits the method containing the `each` call, not just the iteration. Use `next` to skip to the next element and `break` to stop the loop. Calling a stored proc whose defining method has already returned raises `LocalJumpError`.
*Follow-up:* What does `break` return? It makes the yielding method (here `each`) return the given value.

**Q5. What is a closure in Ruby?**
A block, proc or lambda that captures the local variables in scope where it was created and keeps them alive. It captures variables, not snapshots, so changes made later are visible and assignments inside the closure affect the outer variable. A counter factory is the classic example.
*Follow-up:* Why does a `for` loop behave differently from `each` with closures? `for` reuses one variable for every iteration, while each block call gets a fresh parameter.

**Q6. What is the difference between `{ }` and `do ... end`?**
Both create blocks, but braces bind to the nearest method call and `do ... end` binds to the outermost method in the expression. `puts list.map do |x| x * 2 end` passes the block to `puts`, so `map` returns an Enumerator. Convention: braces for one-liners used for their value, `do ... end` for multi-line side effects.
*Follow-up:* Where does this matter in tests? RSpec matchers such as `change { Order.count }` need braces to bind correctly.

**Q7. What does `map(&:upcase)` do under the hood?**
`&` calls `to_proc` on the symbol `:upcase`, producing a proc that calls `upcase` on its argument, and passes that proc as the block. It is shorthand for `map { |s| s.upcase }`. It cannot pass arguments to the method.
*Follow-up:* What does `&method(:puts)` do? Passes a Method object (converted with `to_proc`) as the block, so each element is printed.

**Q8. When do you capture a block with `&block` instead of using `yield`?**
When the block must outlive the call (stored as a callback), be passed to another method, or be called conditionally later. `yield` is simpler and slightly faster for "call it now" cases.
*Follow-up:* How do you forward a block without naming it? `def wrap(&) = inner(&)` with the anonymous block parameter in recent Ruby versions, or `&block` in older ones.

**Q9. Why do Rails scopes require a lambda?**
So the query is built at call time. A non-callable body would be evaluated once when the class loads, freezing values like `1.week.ago` at boot and returning stale results. Modern Rails raises an error if the scope body is not callable.
*Follow-up:* Can a scope take arguments? Yes, the lambda can take parameters: `scope :priced_above, ->(cents) { where("price_cents > ?", cents) }`.

**Q10. How does `around_action` use blocks?**
The callback method receives the controller action as a block (or calls `yield`), so it can run code before and after the action, and even skip it. A common example wraps the action in `I18n.with_locale(...)`, `Time.use_zone(...)` or a tenant context so the setting is restored afterwards.
*Follow-up:* What if the callback never yields? The action and the remaining callbacks are skipped, so the callback itself must render or redirect.

**Q11. What is the difference between `then` and `tap`?**
`then` yields the receiver and returns the block's result, so it pipelines transformations. `tap` yields the receiver and returns the receiver unchanged, which suits configuring an object or peeking into a chain for debugging.
*Follow-up:* What is `then`'s older name? `yield_self`.

**Q12. Implement `map` without using `map`.**
```ruby
def my_map(items)
  return to_enum(:my_map, items) unless block_given?
  result = []
  items.each { |item| result << yield(item) }
  result
end
my_map([1, 2]) { |n| n * 3 } # => [3, 6]
```
It iterates with `each`, collects each block result, and returns an Enumerator when no block is given, mirroring the built-in behavior.
*Follow-up:* How would you implement `select`? Push the item when `yield(item)` is truthy.

**Q13. Write a method that retries a block on network errors. What do you need to watch for?**
Wrap `yield` in `begin/rescue` with a counter defined outside the `begin`, use `retry` with exponential backoff, and re-raise once attempts run out (see section 10). Rescue only specific transient errors, retry only idempotent operations, cap total wait time, and log each attempt.
*Follow-up:* In a Sidekiq job, would you still do this? Keep in-process retries small and rely on Sidekiq's own retries with backoff for the rest.

**Q14. Compare `next`, `break` and `return` inside a block.**
`next` ends the current block call (optionally providing its value, as in `map { next 0 if ... }`). `break` stops the iteration and makes the yielding method return, optionally with a value. `return` exits the method that contains the block, or the lambda itself if it is a lambda.
*Follow-up:* What does `[1, 2, 3].each { |n| break n * 10 if n == 2 }` return? `20`.

**Q15. What is currying?**
Turning a function of several arguments into a chain of functions that take them one at a time, which allows partial application. In Ruby, `->(a, b) { a + b }.curry[5]` returns a lambda waiting for `b`. It relies on closures to remember the arguments supplied so far.
*Follow-up:* How do you compose two lambdas? `f >> g` runs `f` then `g`; `f << g` runs `g` then `f`.

**Q16. What are `_1` and `it`?**
Implicit block parameters: `_1`, `_2` and so on (Ruby 2.7) refer to positional block parameters, and `it` (Ruby 3.4) refers to the single parameter. They only work in blocks without an explicit parameter list. Keep them for short, obvious blocks.
*Follow-up:* What do you use on an older Ruby? Named parameters, or `_1` from Ruby 2.7 onward.

## Further reading
- [Proc (Ruby docs)](https://docs.ruby-lang.org/en/master/Proc.html)
- [Ruby syntax: calling methods and blocks](https://docs.ruby-lang.org/en/master/syntax/calling_methods_rdoc.html)
- [Action Controller overview: callbacks](https://guides.rubyonrails.org/action_controller_overview.html)
