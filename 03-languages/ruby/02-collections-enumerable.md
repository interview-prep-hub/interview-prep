# Ruby Collections and Enumerable

> **Tier:** T1 (know cold) · **Related:** [Ruby basics](01-basics.md), [Blocks, procs and lambdas](04-blocks-procs-lambdas.md), [OOP, modules and mixins](03-oop-modules-mixins.md), [ActiveRecord querying and N+1](../../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Arrays, strings and hashing](../../02-dsa/01-arrays-strings-hashing.md)
> **On your resume:** Daily data shaping in Rails at Devntech, for example background jobs that sync listings, prices, stock and orders with PriceCharting and import market prices into the CrystalCommerce catalog.

## Quick revision
- Arrays are ordered and zero-indexed; negative indexes count from the end. `arr[1, 2]` means (start, length); `arr[1..2]` is a range.
- `push`/`<<` and `pop` work on the end; `unshift` and `shift` work on the front.
- `Array.new(3, [])` shares ONE inner array; write `Array.new(3) { [] }`.
- `hash[:missing]` returns `nil` (or the default); `hash.fetch(:missing)` raises `KeyError`, surfacing bad data at the source.
- `Hash.new([])` hands out one shared default and never stores keys on `<<`; use `Hash.new { |h, k| h[k] = [] }`. `Hash.new(0)` is fine for counters.
- Hashes keep insertion order. `select`/`reject` on a Hash return a Hash; `map` returns an Array (add `.to_h`).
- `map` transforms, `select`/`reject` filter, `filter_map` does both, `find` returns the first match, `reduce` folds, `each_with_object` builds a collection.
- `group_by` gives a Hash of arrays, `partition` two arrays, `tally` counts, Rails `index_by` a Hash keyed by an attribute.
- `each_slice(n)` makes batches, `each_cons(n)` sliding windows, `zip` pairs, `flat_map` maps then flattens one level.
- Edge cases: `[].all?` is `true`, `[].sum` is `0`, `[].max` and `[].reduce(:+)` are `nil`.
- Include `Enumerable` and define `each` to get dozens of methods; return `enum_for(:each)` when no block is given.
- Enumerators allow external iteration (`next`) and infinite sequences; `.lazy` pushes one element at a time through the chain and stops early.
- Bang methods mutate in place and many return `nil` when nothing changed, so never chain them.
- `Set` gives O(1) membership checks and needs no `require` since Ruby 3.2.
- On ActiveRecord relations, filter, count and sum in SQL (`where`, `count`, `sum`, `pluck`) rather than loading rows into Ruby.

Return values in comments use the Ruby 3.4 `inspect` format (`{paid: 2}`); older versions print `{:paid=>2}`.

## 1. Arrays

### Creating and reading

```ruby
nums = [10, 20, 30, 40, 50]
nums[0]              # => 10
nums[-1]             # => 50
nums[1, 2]           # => [20, 30]      (start, length)
nums[1..3]           # => [20, 30, 40]  (range)
nums[1...-1]         # => [20, 30, 40]
nums.first(2)        # => [10, 20]
nums.last(2)         # => [40, 50]
nums[99]             # => nil
nums.fetch(99)       # IndexError
nums.fetch(99, 0)    # => 0
nums.values_at(0, 2) # => [10, 30]
nums.take(2)         # => [10, 20]
nums.drop(3)         # => [40, 50]

Array.new(3) { |i| i * i }  # => [0, 1, 4]
Array(nil)                  # => []   Kernel#Array normalizes "one or many" input
Array([1, 2])               # => [1, 2]
Array("a")                  # => ["a"]
```

### Adding and removing

```ruby
stack = [1, 2]
stack.push(3, 4)     # => [1, 2, 3, 4]   (<< appends a single item)
stack.pop            # => 4
queue = [1, 2, 3]
queue.shift          # => 1, queue is now [2, 3]
queue.unshift(0)     # => [0, 2, 3]
queue.insert(1, :x)  # => [0, :x, 2, 3]
queue.delete(:x)     # => :x (removes every equal element)
queue.delete_at(0)   # => 0
```

`push` with `pop` gives you a stack; `push` with `shift` gives you a queue. For real job queues across processes, use Redis or the database (Sidekiq, Solid Queue), not an in-memory array.

### Reshaping

```ruby
[1, [2, [3, [4]]]].flatten      # => [1, 2, 3, 4]
[1, [2, [3, [4]]]].flatten(1)   # => [1, 2, [3, [4]]]
[1, nil, 2, nil].compact        # => [1, 2]
[1, 2, 2, 3, 1].uniq            # => [1, 2, 3]
%w[Ali ali Sara].uniq(&:downcase)               # => ["Ali", "Sara"]
[3, 1, 2].sort                                  # => [1, 2, 3]
[3, 1, 2].sort { |a, b| b <=> a }               # => [3, 2, 1]
%w[pear fig apple].sort_by { |w| [w.length, w] } # => ["fig", "pear", "apple"]
[3, 1, 2].max(2)                                # => [3, 2]
[[1, 2], [3, 4]].transpose                      # => [[1, 3], [2, 4]]
```

`sort_by` with an array key sorts by several fields: arrays compare element by element with `<=>`.

### Nested data and combinations

```ruby
data = { user: { addresses: [{ city: "Lahore" }] } }
data.dig(:user, :addresses, 0, :city)   # => "Lahore"
data.dig(:user, :phone, :mobile)        # => nil instead of NoMethodError on nil

[1, 2].product(["a", "b"])       # => [[1, "a"], [1, "b"], [2, "a"], [2, "b"]]
[1, 2, 3].combination(2).to_a    # => [[1, 2], [1, 3], [2, 3]]
[1, 2, 3].permutation(2).count   # => 6
[1, 2] | [2, 3]                  # => [1, 2, 3]  union
[1, 2] & [2, 3]                  # => [2]        intersection
[1, 2, 2, 3] - [2]               # => [1, 3]     difference
```

`dig` is ideal for third-party JSON where any level may be missing. `product` generates variant combinations (size by color), and `combination(2)` generates unique pairs.

## 2. Hashes

### Reading safely: `[]` vs `fetch`

```ruby
store = { name: "Game Haven", city: "Lahore", rating: nil }
store[:name]                 # => "Game Haven"
store[:zip]                  # => nil  (missing key)
store[:rating]               # => nil  (present key, nil value): same result!
store.key?(:rating)          # => true
store.fetch(:zip)            # KeyError (key not found: :zip)
store.fetch(:zip, "54000")   # => "54000"
store.fetch(:zip) { |k| "no #{k}" } # => "no zip"
store.dig(:owner, :email)    # => nil
```

`[]` returns `nil` for both "missing" and "present but nil", and that `nil` usually explodes far away (a `NoMethodError` in a view or job). `fetch` fails at the source with a clear `KeyError`. Use it for required config (`ENV.fetch("API_KEY")`) and required fields in API payloads.

### Defaults and the shared-default trap

```ruby
counts = Hash.new(0)               # immutable default: safe
%w[a b a].each { |w| counts[w] += 1 }
counts                             # => {"a" => 2, "b" => 1}

groups = Hash.new([])              # ONE array shared by every missing key
groups[:paid] << 1                 # mutates the shared default; stores nothing
groups[:pending] << 2
groups                             # => {}
groups[:anything]                  # => [1, 2]

groups = Hash.new { |hash, key| hash[key] = [] }  # block runs once per missing key
groups[:paid] << 1
groups[:pending] << 2
groups                             # => {paid: [1], pending: [2]}
```

Why: `Hash.new(obj)` returns the same `obj` for every missing key and never assigns it. `counts[w] += 1` works because it expands to `counts[w] = counts[w] + 1`, an assignment of a new Integer. The block form creates and stores a fresh array per key. Often `group_by` or `each_with_object` is clearer than either.

### Transforming

```ruby
prices = { "A1" => 100, "B2" => 250, "C3" => 75 }   # cents by SKU

prices.transform_values { |cents| cents * 2 }  # => {"A1" => 200, "B2" => 500, "C3" => 150}
prices.transform_keys(&:downcase)              # => {"a1" => 100, "b2" => 250, "c3" => 75}
prices.select { |_sku, cents| cents > 90 }     # => {"A1" => 100, "B2" => 250}  Hash in, Hash out
prices.reject { |_sku, cents| cents > 90 }     # => {"C3" => 75}
prices.map { |sku, cents| "#{sku}:#{cents}" }  # => ["A1:100", "B2:250", "C3:75"]  an Array!
prices.to_h { |sku, cents| [sku.to_sym, cents] } # => {A1: 100, B2: 250, C3: 75}
prices.min_by { |_sku, cents| cents }          # => ["C3", 75]
prices.sort_by { |_sku, cents| -cents }.to_h   # => {"B2" => 250, "A1" => 100, "C3" => 75}
prices.sum { |_sku, cents| cents }             # => 425
prices.count { |_sku, cents| cents < 200 }     # => 2
prices.filter_map { |sku, cents| sku if cents > 90 } # => ["A1", "B2"]
prices.slice("A1", "C3")                       # => {"A1" => 100, "C3" => 75}
prices.except("A1")                            # => {"B2" => 250, "C3" => 75}  (core since Ruby 3.0)
prices.merge({ "A1" => 120, "D4" => 10 }) { |_sku, current, incoming| [current, incoming].max }
# => {"A1" => 120, "B2" => 250, "C3" => 75, "D4" => 10}
{ a: 1, b: nil }.compact                       # => {a: 1}
```

The `merge` block resolves conflicts, which is handy when combining stock counts or prices from two sources.

### Symbol keys vs string keys

JSON has no symbols, so anything that crosses a JSON boundary comes back with string keys: `JSON.parse` output, API responses and plain Sidekiq job arguments, which are stored as JSON (so pass simple types with string keys; Sidekiq can also be configured to raise on arguments that are not native JSON types). ActiveJob uses its own serializer and preserves symbol keys. Normalize at the boundary with `JSON.parse(body, symbolize_names: true)`, or ActiveSupport's `symbolize_keys`, `deep_symbolize_keys`, `stringify_keys` and `with_indifferent_access`.

## 3. The Enumerable tour

Every example below uses this dataset:

```ruby
Order = Struct.new(:id, :store, :total, :status)
orders = [
  Order.new(1, "Lahore Games", 120, :paid),
  Order.new(2, "Karachi Cards", 80, :pending),
  Order.new(3, "Lahore Games", 45, :paid),
  Order.new(4, "Pindi Toys", 200, :refunded)
]
```

### Transform, filter, find

```ruby
orders.map(&:total)                                # => [120, 80, 45, 200]
orders.select { |o| o.status == :paid }.map(&:id)  # => [1, 3]   (alias: filter)
orders.reject { |o| o.status == :paid }.map(&:id)  # => [2, 4]
orders.filter_map { |o| o.id if o.total > 100 }    # => [1, 4]   select and map in one pass
orders.find { |o| o.store == "Pindi Toys" }&.id    # => 4        first match or nil (alias: detect)
orders.find_index { |o| o.total > 100 }            # => 0

stores = [{ name: "A", tags: %w[retro cards] }, { name: "B", tags: %w[cards toys] }]
stores.map { |s| s[:tags] }              # => [["retro", "cards"], ["cards", "toys"]]
stores.flat_map { |s| s[:tags] }.uniq    # => ["retro", "cards", "toys"]
```

### Aggregate

```ruby
orders.sum(&:total)                          # => 445
orders.reduce(0) { |acc, o| acc + o.total }  # => 445   (alias: inject)
[1, 2, 3].reduce(:*)                         # => 6
orders.count { |o| o.status == :paid }       # => 2
orders.min_by(&:total).id                    # => 3
orders.max_by(&:total).id                    # => 4
orders.max_by(2, &:total).map(&:id)          # => [4, 1]   top two
orders.minmax_by(&:total).map(&:id)          # => [3, 4]
orders.sort_by { |o| -o.total }.map(&:id)    # => [4, 1, 2, 3]
```

### Build new collections

```ruby
orders.each_with_object(Hash.new(0)) { |o, totals| totals[o.store] += o.total }
# => {"Lahore Games" => 165, "Karachi Cards" => 80, "Pindi Toys" => 200}

orders.group_by(&:status).transform_values { |list| list.map(&:id) }
# => {paid: [1, 3], pending: [2], refunded: [4]}

paid, others = orders.partition { |o| o.status == :paid }
paid.map(&:id)     # => [1, 3]
others.map(&:id)   # => [2, 4]

orders.map(&:status).tally
# => {paid: 2, pending: 1, refunded: 1}

orders.to_h { |o| [o.id, o.total] }
# => {1 => 120, 2 => 80, 3 => 45, 4 => 200}
```

**`reduce` vs `each_with_object`.** With `reduce`, the block's return value becomes the next accumulator. With `each_with_object`, the same memo object is passed every time and the block's return value is ignored, which suits building hashes and arrays. Note the opposite argument order: `reduce { |acc, x| }` but `each_with_object(memo) { |x, memo| }`.

```ruby
orders.reduce({}) { |h, o| h[o.id] = o.total }
# NoMethodError on the second element: the assignment returned 120,
# so the accumulator became an Integer. Return h, or use each_with_object.
```

### Batches, windows and pairs

```ruby
orders.each_slice(2).map { |batch| batch.map(&:id) }  # => [[1, 2], [3, 4]]
orders.each_cons(2).map { |a, b| b.total - a.total }  # => [-40, -35, 155]
orders.map(&:id).zip(orders.map(&:total))             # => [[1, 120], [2, 80], [3, 45], [4, 200]]
%w[a b c].each_with_index.map { |s, i| "#{i}:#{s}" }  # => ["0:a", "1:b", "2:c"]
%w[a b c].map.with_index(1) { |s, i| "#{i}. #{s}" }    # => ["1. a", "2. b", "3. c"]
```

Real uses: `each_slice(100)` to send SKUs to a rate-limited API in batches, `each_cons(2)` to compute price changes between consecutive snapshots.

### Predicates

```ruby
orders.any? { |o| o.status == :refunded }  # => true
orders.all? { |o| o.total > 50 }           # => false (order 3 is 45)
orders.none? { |o| o.total > 500 }         # => true
orders.one? { |o| o.status == :pending }   # => true
[1, "a", :b].any?(String)                  # => true (a pattern argument is matched with ===)
[nil, false].any?                          # => false (no block: checks truthiness)
[].all?                                    # => true  (vacuously true)
```

### Chaining

```ruby
orders
  .select { |o| o.status == :paid }
  .group_by(&:store)
  .transform_values { |list| list.sum(&:total) }
# => {"Lahore Games" => 165}
```

Each step allocates an intermediate array. That is fine for hundreds or thousands of items; for huge or infinite sources use `lazy` (section 6), and for database rows push the work into SQL (section 8).

| Goal | Method | Returns |
|---|---|---|
| Transform each element | `map` | Array |
| Keep or drop | `select` (`filter`), `reject` | Array (a Hash for a Hash) |
| Transform and drop `nil`/`false` | `filter_map` | Array |
| First match | `find` (`detect`) | element or `nil` |
| Fold into one value | `reduce` (`inject`), `sum` | value |
| Build a collection | `each_with_object` | the memo object |
| Bucket by key | `group_by` | Hash of arrays |
| Split in two | `partition` | `[matching, rest]` |
| Count occurrences | `tally`, `count` | Hash, Integer |
| Extremes | `min_by`, `max_by`, `minmax_by` | element(s) |
| Batches or windows | `each_slice`, `each_cons` | Enumerator (or `nil` with a block) |
| Pair up | `zip` | Array of arrays |
| Map then flatten one level | `flat_map` | Array |

## 4. Set

```ruby
require "set"                  # only needed before Ruby 3.2
seen = Set.new
seen << "A1"
seen.add?("A1")                # => nil (already present); returns the set when it adds
seen.include?("A1")            # => true, O(1) like a Hash lookup
(Set[1, 2] | Set[2, 3]).to_a   # => [1, 2, 3]
(Set[1, 2] & Set[2, 3]).to_a   # => [2]
Set[1, 2].subset?(Set[1, 2, 3]) # => true
```

Use a Set when you check membership repeatedly, such as de-duplicating SKUs during an import. `Array#include?` is O(n), so calling it inside a loop makes the whole thing O(n^2).

## 5. Including Enumerable in your own class

```ruby
class Catalog
  include Enumerable

  def initialize(products)
    @products = products
  end

  def each(&block)
    return enum_for(:each) unless block_given?  # lets callers chain: catalog.each.with_index
    @products.each(&block)
    self
  end
end

Product = Struct.new(:sku, :price)
catalog = Catalog.new([Product.new("A1", 300), Product.new("B2", 150)])
catalog.map(&:sku)                                 # => ["A1", "B2"]
catalog.min_by(&:price).sku                        # => "B2"
catalog.select { |p| p.price > 200 }.map(&:sku)    # => ["A1"]
catalog.each.with_index(1).map { |p, i| "#{i}. #{p.sku}" } # => ["1. A1", "2. B2"]
```

Every Enumerable method is written in terms of `each`, so supplying that one primitive unlocks `map`, `select`, `sort_by`, `each_slice`, `lazy` and the rest. Methods such as `sort`, `min` and `max` without a block also need the elements to implement `<=>` (include `Comparable` in the element class; see [OOP, modules and mixins](03-oop-modules-mixins.md)).

## 6. Enumerators and lazy evaluation

```ruby
e = [10, 20].each        # no block: returns an Enumerator
e.next                   # => 10
e.next                   # => 20
e.next                   # StopIteration (loop do ... end rescues it for you)

fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a               # hand a value to whoever is consuming
    a, b = b, a + b
  end
end
fib.take(8)                        # => [0, 1, 1, 2, 3, 5, 8, 13]
fib.lazy.select(&:even?).first(4)  # => [0, 2, 8, 34]
```

A normal chain builds a complete intermediate array at every step. `lazy` sends each element through the whole chain one at a time and stops when `first(n)` has enough, so it works on infinite sequences and on files too large to load.

```ruby
(1..Float::INFINITY).lazy.map { |x| x * 2 }.select { |x| (x % 3).zero? }.first(3)
# => [6, 12, 18]

File.foreach("prices.csv")                     # reads one line at a time
    .lazy
    .map { |line| line.chomp.split(",") }
    .select { |_sku, price| price.to_f > 100 }
    .first(10)
```

Lazy is not automatically faster: per-element overhead makes it slower on small arrays. Reach for it when you need early termination or bounded memory, and call `.force` or `.to_a` to evaluate the whole chain.

## 7. Bang methods and mutation pitfalls

Most collection methods return a new object; their `!` twins change the receiver in place. Many of those return `nil` when nothing changed, so their return value is not safe to chain.

```ruby
tags = ["sale", "new", "sale"]
tags.uniq            # => ["sale", "new"], tags unchanged
tags.uniq!           # => ["sale", "new"], tags is now changed
tags.uniq!           # => nil: nothing left to remove
tags.map!(&:upcase)  # => ["SALE", "NEW"] (map! always returns the array)
tags.uniq!.sort      # NoMethodError on nil, but only when there were no duplicates
"x".strip!           # => nil, so "x".strip!.upcase raises
```

Shared references mean a mutation is visible through every name and every container holding the object:

```ruby
defaults = { tags: [] }
copy = defaults.dup
copy[:tags] << "sale"
defaults[:tags]               # => ["sale"]: dup is shallow, the inner array is shared

grid = Array.new(2, [])       # the same inner array in both slots
grid[0] << :x
grid                          # => [[:x], [:x]]
grid = Array.new(2) { [] }    # the block runs per slot: independent arrays

def add_tag(listing_tags, tag)
  listing_tags << tag         # mutates the caller's array
end
```

Mutating a collection while iterating over it skips elements, because the index keeps moving while elements shift under it:

```ruby
nums = [1, 2, 3, 4]
nums.each { |n| nums.delete(n) }
nums                          # => [2, 4]
nums = [1, 2, 3, 4]
nums.reject!(&:odd?)          # => [2, 4], the intended in-place removal
nums.delete_if { |n| n > 3 }  # => [2]
```

Rules of thumb: prefer non-bang methods and reassign (`tags = tags.uniq`) unless you are deliberately mutating a large local collection for performance; never chain on a bang method's return value; `dup` (or `deep_dup`) before mutating anything you received from a caller; and freeze constants (`STATUSES = %i[pending paid].freeze`) so accidental mutation fails loudly.

## 8. Collections meet ActiveRecord

Relations look like arrays, but they build SQL lazily. Calling plain Enumerable methods on them loads every matching row into Ruby objects first.

| Ruby side (loads records) | SQL side (the database does the work) |
|---|---|
| `Order.all.select(&:paid?)` | `Order.where(status: :paid)` |
| `orders.map(&:id)` | `orders.pluck(:id)` or `orders.ids` |
| `orders.sum(&:total)` | `orders.sum(:total)` |
| `orders.group_by(&:store_id).transform_values(&:count)` | `orders.group(:store_id).count` |
| `Order.all.each { ... }` | `Order.find_each(batch_size: 1000) { ... }` |

`count` always runs `SELECT COUNT(*)`, `length` loads the records and counts the array, and `size` uses the loaded array if there is one, otherwise it runs `COUNT`. `find_each` walks the table in primary-key batches so memory stays flat. More in [ActiveRecord querying and N+1](../../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md).

## 9. ActiveSupport extras

```ruby
users_by_id = users.index_by(&:id)             # => {1 => user1, 2 => user2}
flags = %i[email sms push].index_with(false)   # => {email: false, sms: false, push: false}
orders.many?                                   # => true when there is more than one element
(1..7).to_a.in_groups_of(3)                    # => [[1, 2, 3], [4, 5, 6], [7, nil, nil]]
(1..7).to_a.in_groups_of(3, false)             # => [[1, 2, 3], [4, 5, 6], [7]]
%w[Ali Sara Omar].to_sentence                  # => "Ali, Sara, and Omar"
[1, 2, 3].excluding(2)                         # => [1, 3]
[{ sku: "A1" }, { sku: "B2" }].pluck(:sku)     # => ["A1", "B2"]
["", nil, "x", []].compact_blank               # => ["x"]
Array.wrap(nil)                                # => []
Array.wrap({ a: 1 })                           # => [{a: 1}]  (Array({ a: 1 }) gives [[:a, 1]])
```

`index_by` removes per-item lookups inside loops: load once, then do O(1) hash lookups.

```ruby
market = MarketPrice.where(sku: listings.map(&:sku)).index_by(&:sku)  # one query
listings.each { |l| l.suggested_price = market[l.sku]&.amount }       # no query per listing
```

### Going deeper: float sums

`Array#sum` uses compensated (Kahan-Babuska) summation for floats, so `([0.1] * 10).sum` is `1.0` while `([0.1] * 10).reduce(:+)` is `0.9999999999999999`. It is still Float arithmetic, so money belongs in `BigDecimal` or integer cents.

## Common pitfalls
- `Hash.new([])` and `Array.new(n, [])` share one mutable object across keys or slots.
- Reading required data with `[]` and getting a silent `nil` instead of a `KeyError`.
- Expecting `map` on a Hash to return a Hash; it returns an Array.
- Forgetting that the `reduce` block must return the accumulator.
- Using `each` where you meant `map`: `ids = orders.each { |o| o.id }` returns `orders` itself.
- Chaining bang methods: `tags.uniq!.sort` blows up when there were no duplicates.
- Mutating an array while iterating it: `nums.each { |n| nums.delete(n) }` leaves `[2, 4]` from `[1, 2, 3, 4]`. Use `reject`, `reject!` or `delete_if`.
- `Array#include?` inside a loop (quadratic). Use a `Set` or `index_by`.
- Ruby-side `select`, `map` and `sum` on large relations instead of SQL.
- Symbol keys turning into strings after a JSON round trip (API payloads, plain Sidekiq arguments).
- Surprising empty-collection results: `[].all?` is `true` and `[].max` is `nil`.

## How to talk about this in an interview
- Sync and import jobs are the natural stage for this topic: grouping payloads by SKU, batching requests to a rate-limited API, building lookup hashes so a job does not query once per row. Mention that you think about where the work happens (Ruby or SQL) and its complexity.
- In live coding, narrate your method choices ("`tally` for counts, `max_by(3)` for the top three, `each_with_object` because I'm building a hash"). Ruby interviewers listen for idiomatic vocabulary.
- > **Fill in:** how listing, stock or price data moved through your PriceCharting sync jobs (batch sizes, de-duplication, lookup hashes, what ran in SQL vs Ruby).
- > **Fill in:** a refactor where you replaced a manual loop or an N+1 lookup with `group_by`, `index_by`, `each_with_object` or a SQL query.

## Interview questions

**Q1. What is the difference between `each` and `map`?**
`each` runs the block for its side effects and returns the original receiver. `map` collects the block's return values into a new array. Using `each` where you need the results is a common bug because it silently returns the input.
*Follow-up:* What about `map!`? It replaces the elements in place and returns the same array.

**Q2. When would you use `select`, `filter_map` and `find`?**
`select` keeps every element whose block is truthy; `filter_map` maps and drops `nil`/`false` results in one pass (`users.filter_map { |u| u.email if u.active? }`); `find` returns only the first match and stops iterating. Use `find` for "is there one", `select` for "all of them".
*Follow-up:* What does `find` return when nothing matches? `nil`, so guard with `&.` or `fetch`-style handling.

**Q3. Explain the `Hash.new([])` trap and the fix.**
The default object is shared by every missing key and is never stored, so `h[:a] << 1` mutates that single default and `h` stays `{}` while `h[:b]` now returns `[1]`. Use the block form `Hash.new { |hash, key| hash[key] = [] }`, which creates and stores a new array per key. `Hash.new(0)` is safe because `+=` assigns a new Integer.
*Follow-up:* What is a clearer alternative? `group_by`, or `each_with_object(Hash.new { |h, k| h[k] = [] })`.

**Q4. `hash[:key]` vs `hash.fetch(:key)`: which do you prefer?**
`[]` returns `nil` (or the default) for a missing key, which hides the difference between missing and nil and fails later. `fetch` raises `KeyError` immediately, or uses a default value or block you pass. Prefer `fetch` for required data such as `ENV.fetch("API_KEY")` and mandatory payload fields.
*Follow-up:* How do you read nested optional keys? `dig(:a, :b, :c)`, which returns `nil` if any level is missing.

**Q5. `reduce` vs `each_with_object`?**
`reduce` threads the block's return value through as the accumulator, which is natural for numbers (`reduce(:+)`). `each_with_object` passes one mutable memo and ignores the block's return value, which is safer for building hashes and arrays. Forgetting to return the accumulator from `reduce` is a classic bug.
*Follow-up:* What does `[].reduce(:+)` return? `nil`; pass an initial value (`reduce(0, :+)`) or use `sum`.

**Q6. Compare `group_by`, `partition`, `tally` and `index_by`.**
`group_by` returns a Hash from key to an array of elements; `partition` splits into two arrays by a boolean; `tally` counts occurrences of each value; ActiveSupport's `index_by` returns a Hash from key to a single element (the last one wins on duplicates). Pick by the shape you need afterwards.
*Follow-up:* How do you count orders per status in SQL? `Order.group(:status).count`.

**Q7. What does `lazy` do and when is it worth it?**
It turns the chain into a pipeline that processes one element at a time and only as far as needed, so `first(n)` stops early and intermediate arrays are never built. It enables infinite sequences and streaming large files. On small arrays it is slower due to overhead.
*Follow-up:* How do you force evaluation? `to_a`, `force` or a terminal method such as `first(n)`.

**Q8. How do you make your own class work with `map`, `select` and friends?**
`include Enumerable` and define `each` that yields each element. Return `enum_for(:each)` when no block is given so callers can chain `each.with_index`. For `sort`, `min` and `max` without blocks, the elements must implement `<=>`.
*Follow-up:* What is the parallel for comparisons? `include Comparable` and define `<=>`.

**Q9. Why do many bang methods return `nil`, and why does it matter?**
Methods such as `uniq!`, `compact!`, `strip!` and `gsub!` return `nil` when they changed nothing, so callers can detect whether a change happened. Chaining them (`s.strip!.downcase`) therefore raises `NoMethodError` on inputs that were already clean, which makes the bug data-dependent.
*Follow-up:* Which bang methods always return self? `map!`, `sort!` and `reverse!`, among others; check the docs rather than guessing.

**Q10. When would you use a `Set` instead of an Array?**
When you need uniqueness or frequent membership checks: `Set#include?` is a hash lookup (O(1) on average) while `Array#include?` scans (O(n)). De-duplicating SKUs during an import with an array inside a loop is quadratic.
*Follow-up:* Do you need `require "set"`? Not since Ruby 3.2.

**Q11. In Rails, what is the difference between `count`, `size` and `length` on a relation?**
`count` always runs `SELECT COUNT(*)`. `length` loads every record and counts the array. `size` uses the loaded records if the relation is already loaded, otherwise it runs `COUNT`. Picking the wrong one either loads a whole table or adds a redundant query.
*Follow-up:* And `pluck` vs `map`? `pluck(:id)` selects one column in SQL; `map(&:id)` instantiates full models first.

**Q12. `flat_map` vs `map.flatten`?**
`flat_map` maps and flattens exactly one level in one pass without an intermediate nested array. `map { }.flatten` flattens all levels, which can wrongly flatten nested data you meant to keep. Use `flat_map` when each element maps to a list.
*Follow-up:* How do you flatten exactly one level of an existing array? `flatten(1)`.

**Q13. You need to send 10,000 SKUs to an API that accepts 100 per request. How?**
`skus.each_slice(100) { |batch| client.fetch_prices(batch) }`, typically one background job per batch so failures retry independently. Add rate limiting and idempotent upserts because jobs can run more than once.
*Follow-up:* How do you batch database reads? `find_each` or `in_batches` instead of `all.each`.

**Q14. Return the three most frequent words in a string.**
```ruby
text.downcase.scan(/\w+/).tally.sort_by { |word, count| [-count, word] }.first(3)
```
`tally` counts, the sort key orders by count descending with alphabetical tie-breaking, and `first(3)` takes the top three as `[word, count]` pairs. `max_by(3) { |_w, c| c }` also works but leaves ties in arbitrary order.
*Follow-up:* Complexity? O(n) to count plus O(k log k) to sort the k distinct words.

**Q15. How do you find duplicates in an array?**
`arr.tally.select { |_value, count| count > 1 }.keys`, or `arr.group_by(&:itself).select { |_k, v| v.size > 1 }.keys`. For a quick boolean, `arr.uniq.size != arr.size`.
*Follow-up:* How would you find duplicate emails in the database? `User.group(:email).having("COUNT(*) > 1").pluck(:email)`.

**Q16. How do you sort by several keys, one descending?**
Use an array sort key: `users.sort_by { |u| [-u.score, u.name] }` sorts by score descending, then name ascending. Negation only works for numbers; for descending strings use `sort { |a, b| [b.name, a.id] <=> [a.name, b.id] }`.
*Follow-up:* Is `sort_by` stable? Not guaranteed; add a unique tie-breaker such as the id when order matters.

**Q17. What does `map` on a Hash return, and how do you get a Hash back?**
An Array, because `map` comes from Enumerable and collects block results. Use `to_h { |k, v| [k, v] }`, `transform_values`, `transform_keys`, or `map { }.to_h`. `select`, `reject`, `slice`, `except` and `compact` already return hashes, while `filter_map` returns an Array.
*Follow-up:* How do you sort a Hash by value? `hash.sort_by { |_k, v| v }.to_h`.

**Q18. What is an Enumerator and why does `each` without a block return one?**
An Enumerator is an object wrapping an iteration that you can drive later: chain it (`each.with_index`), pull values one by one with `next`, or make it lazy. Returning one when no block is given keeps methods chainable and allows external iteration.
*Follow-up:* How do you create an infinite sequence? `Enumerator.new { |y| loop { y << value } }` consumed with `take`, `first` or `lazy`.

## Further reading
- [Enumerable (Ruby docs)](https://docs.ruby-lang.org/en/master/Enumerable.html)
- [Hash (Ruby docs)](https://docs.ruby-lang.org/en/master/Hash.html)
- [Active Support core extensions](https://guides.rubyonrails.org/active_support_core_extensions.html)
