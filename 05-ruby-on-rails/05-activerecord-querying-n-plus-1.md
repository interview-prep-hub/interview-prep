# ActiveRecord Querying and N+1

> **Tier:** T1 (know cold) · **Related:** [ActiveRecord basics](03-activerecord-basics.md), [Associations](04-activerecord-associations.md), [ActiveRecord advanced](06-activerecord-advanced.md), [SQL from zero](../12-databases/02-sql-from-zero.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md), [Caching, performance and security](14-caching-performance-security.md)
> **On your resume:** PostgreSQL-backed catalog and order queries in CrystalCommerce (700+ stores, 8M+ customers), the Colochousing Rails 7 app and API, and performance work in legacy Rails monoliths at Devntech.

## Quick revision
- Relations are **lazy**: `where`, `order`, `joins` only build SQL. The query runs when you iterate, call `to_a`, `first`, `count`, `pluck`, or inspect it in the console.
- `where` accepts a hash (`store_id: 3`), an array of values (`IN`), a range (`BETWEEN`), `nil` (`IS NULL`), or a SQL string with `?` / `:name` placeholders. Never interpolate user input.
- `where.not`, `or` (structurally compatible relations), `where.missing(:assoc)` (6.1, LEFT JOIN ... IS NULL), `where.associated(:assoc)` (7.0, INNER JOIN ... IS NOT NULL).
- `pluck(:id, :name)` returns arrays without building models; `select(:id, :name)` returns models with only those attributes; `ids` plucks primary keys.
- `exists?` and `any?` run `SELECT 1 ... LIMIT 1`; `present?` / `blank?` load **every** row. `count` always runs COUNT; `length` loads all rows; `size` uses the loaded records, a counter cache, or COUNT.
- `find_each` / `in_batches` iterate large tables in primary-key batches (default 1000) instead of loading everything into memory.
- `find_or_create_by` = SELECT then INSERT (racy); `create_or_find_by` = INSERT then SELECT on unique violation (needs a unique index).
- Scopes always return a relation (a nil result becomes `all`), so they always chain; class methods are better for multi-step logic but must return a relation themselves.
- **N+1:** one query for the parents plus one per parent for an association. Spot it in logs (the same SELECT repeated), Bullet, or `strict_loading`; fix it with `includes` / `preload` / `eager_load`.
- `joins` = INNER JOIN for filtering, loads nothing. `preload` = separate `WHERE id IN (...)` queries. `eager_load` = one LEFT OUTER JOIN query. `includes` = preload by default, switches to eager_load when you reference the association's table.
- Nested: `Order.includes(:customer, order_items: { listing: :product })`.
- `association.count` inside a loop still queries even after preloading; use `size`, or a counter cache.
- Safe raw SQL: placeholders, `sanitize_sql_like`, `find_by_sql([sql, binds])`; whitelist column names for ORDER BY.
- Debug with `relation.to_sql` and `relation.explain`, then add the right index.

## 1. The query interface

```ruby
# where: hash, array, range, nil
Listing.where(store_id: 3)                          # store_id = 3
Listing.where(store_id: [3, 4, 5])                  # store_id IN (3, 4, 5)
Listing.where(price_cents: 1_000..5_000)            # BETWEEN 1000 AND 5000
Listing.where(created_at: 1.week.ago..)             # endless range: created_at >= ...
Listing.where(archived_at: nil)                     # archived_at IS NULL
Listing.where("stock < ?", 5)                       # string with a placeholder
Listing.where("price_cents BETWEEN :min AND :max", min: 100, max: 900)

Listing.where.not(condition: :graded)               # condition != 2
Listing.where(active: true).or(Listing.where(featured: true))   # (active) OR (featured)
Product.where.missing(:listings)                    # products with no listings (6.1)
Product.where.associated(:listings)                 # products with at least one listing (7.0)

# ordering, paging, shaping
Listing.order(price_cents: :desc, id: :asc).limit(20).offset(40)
Listing.select(:id, :price_cents)                   # partial models; other attributes raise MissingAttributeError
Listing.pluck(:id, :price_cents)                    # => [[1, 999], [2, 1250]] (no models)
Listing.where(store_id: 3).ids                      # => [1, 2, 3]
Listing.where(store_id: 3).pick(:price_cents)       # first value only
Product.joins(:listings).distinct                   # SELECT DISTINCT products.*

# grouping and aggregates
Order.group(:status).count                          # => { "pending" => 4, "paid" => 10 }
Order.where(store_id: 3).sum(:total_cents)
Listing.group(:store_id).average(:price_cents)
Order.group(:customer_id).having("COUNT(*) > ?", 5).count

# joins and merging scopes across models
Listing.joins(:product).where(products: { platform: "N64" })   # INNER JOIN, filter on products
Store.left_joins(:listings).where(listings: { id: nil })       # LEFT OUTER JOIN
Store.joins(:listings).merge(Listing.active)                   # reuse Listing's scope in a join
```

`joins` on a `has_many` repeats the parent once per matching child, so add `distinct` (or use `where(id: subquery)`) when you want each parent once. `or` raises an `ArgumentError` when the two relations differ structurally (for example one has a `joins` the other lacks).

Offset paging gets slower as the offset grows because the database still walks the skipped rows. For deep pages or infinite scroll, use keyset (cursor) paging: `where("id > ?", last_seen_id).order(:id).limit(20)`.

## 2. Lazy relations

Each query method returns a new `ActiveRecord::Relation`, so you can build a query in steps and nothing hits the database until the records are needed:

```ruby
listings = Listing.where(store_id: 3)                 # no SQL yet
listings = listings.where(active: true) if params[:active]
listings = listings.order(:price_cents)               # still no SQL
listings.each { |l| puts l.price_cents }              # SQL runs here, once
listings.each { |l| puts l.id }                       # no SQL: the relation is loaded and cached
listings.reload                                       # forces a fresh query
```

The console looks eager only because it calls `inspect` on the result. This laziness is what makes scopes composable and what lets controllers build a relation that the view iterates. Rails 7.0 also added `load_async`, which starts a query on a background thread so independent queries in one action can run in parallel.

## 3. Existence checks and counting

| Call | SQL on an unloaded relation | Notes |
|---|---|---|
| `relation.exists?` | `SELECT 1 AS one ... LIMIT 1` | Cheapest existence check |
| `relation.any?` / `empty?` | `SELECT 1 AS one ... LIMIT 1` | Checks memory if already loaded |
| `relation.present?` / `blank?` | `SELECT * ...` (loads every row) | Fine only if you are about to use the records anyway |
| `relation.count` | `SELECT COUNT(*) ...` | Always queries, even when loaded |
| `relation.length` | `SELECT * ...`, then counts in Ruby | Loads everything |
| `relation.size` | COUNT if not loaded; in-memory count if loaded; counter cache for associations | Usually the right default |

The classic slow line in a legacy view is `if @store.orders.present?` followed by nothing that uses the orders: it loads thousands of rows just to check for one. Use `exists?` or `any?`.

## 4. Batches, and find-or-create

```ruby
Listing.find_each(batch_size: 1000) do |listing|   # SELECT ... WHERE id > last_id ORDER BY id LIMIT 1000
  listing.recalculate_market_price!
end

Listing.where(store_id: 3).in_batches(of: 5000) do |batch|  # batch is a relation
  batch.update_all(needs_sync: true)               # one UPDATE per batch
end

Listing.find_in_batches(batch_size: 500) do |listings|      # yields arrays of records
  ExternalChannel.bulk_push(listings)
end
```

`Listing.all.each` loads the whole table into memory at once, which can kill a job's memory on a large catalog. `find_each` walks the table in primary-key order with a constant memory footprint; it ignores any custom `order` you added (it orders by primary key), and because it uses keyset paging (`id > last_id`) instead of offsets, it stays fast deep into the table. `start:` and `finish:` bound the ID range (handy for splitting work across jobs), and `order: :desc` (6.1) walks backwards.

```ruby
User.find_or_create_by(email: email)       # SELECT; if none, INSERT
User.find_or_create_by!(email: email) do |user|
  user.name = name                         # block runs only when creating
end
User.find_or_initialize_by(email: email)   # SELECT; if none, new (unsaved)

User.create_or_find_by(email: email)       # INSERT; on unique violation, SELECT (6.0)
```

`find_or_create_by` has a race: two workers can both see no row and both insert, so you get a duplicate or (with a unique index) a `RecordNotUnique` error. `create_or_find_by` flips the order and relies on a unique constraint, so the database decides; its costs are a savepoint per call, a burned ID on conflict, and it only works when the unique index exists. For bulk syncs, `upsert_all ... unique_by:` is usually better (see [ActiveRecord advanced](06-activerecord-advanced.md)).

## 5. Scopes vs class methods

```ruby
class Listing < ApplicationRecord
  scope :active, -> { where(active: true) }
  scope :in_stock, -> { where("stock > 0") }
  scope :cheaper_than, ->(cents) { where("price_cents < ?", cents) if cents.present? }

  def self.search(term)
    return all if term.blank?                        # must return a relation, not nil
    joins(:product).where("products.title ILIKE ?", "%#{sanitize_sql_like(term)}%")
  end
end

Listing.active.in_stock.cheaper_than(params[:max]).search(params[:q]).order(:price_cents)
store.listings.active                                # works on associations too
```

| | `scope` | Class method |
|---|---|---|
| Returns | Always a relation: if the body returns nil or false, you get `all` | Whatever you return; returning nil breaks the chain with `NoMethodError` |
| Best for | One-line, composable filters | Multi-step logic, early returns, several branches |
| Chaining on relations and associations | Yes | Yes (relations delegate to class methods) |
| Readability | Declarative list at the top of the model | Normal Ruby |

Both are fine; the interview point is the nil behavior. Keep scopes free of side effects and avoid `default_scope` (see [ActiveRecord advanced](06-activerecord-advanced.md)).

## 6. The N+1 problem

N+1 happens when you load N parent records with one query and then trigger one extra query per parent to fetch an association:

```ruby
# controller
@listings = current_store.listings.order(:created_at).limit(50)
```

```erb
<%# view %>
<% @listings.each do |listing| %>
  <%= listing.product.title %>    <%# one query per listing %>
<% end %>
```

```text
Listing Load (0.8ms)  SELECT "listings".* FROM "listings" WHERE "listings"."store_id" = $1 ORDER BY "listings"."created_at" ASC LIMIT $2
Product Load (0.3ms)  SELECT "products".* FROM "products" WHERE "products"."id" = $1 LIMIT $2
Product Load (0.2ms)  SELECT "products".* FROM "products" WHERE "products"."id" = $1 LIMIT $2
Product Load (0.2ms)  SELECT "products".* FROM "products" WHERE "products"."id" = $1 LIMIT $2
... 47 more
```

51 queries instead of 2. Each query is fast, but round trips add up, and the page gets slower as data grows. The fix is to load the association up front:

```ruby
@listings = current_store.listings.includes(:product).order(:created_at).limit(50)
```

```text
Listing Load (0.8ms)  SELECT "listings".* FROM "listings" WHERE "listings"."store_id" = $1 ORDER BY ... LIMIT $2
Product Load (0.5ms)  SELECT "products".* FROM "products" WHERE "products"."id" IN ($1, $2, $3, ...)
```

**Where N+1s hide:** views and partials, JSON serializers and `as_json(include: ...)`, mailers that loop over records, background jobs that iterate parents, and counts: `store.listings.count` in a loop runs one COUNT per store, even if the listings were preloaded.

**How to detect:**
- **Logs:** the same SELECT repeated with different IDs. In development, recent versions also print the source line that triggered each query (verbose query logs).
- **Bullet gem:** watches queries in development and test and warns (or raises) about N+1s and unused eager loading. `Bullet.enable = true`, `Bullet.raise = true` in the test environment.
- **`strict_loading` (6.1):** makes lazy loading raise `ActiveRecord::StrictLoadingViolationError`, so a missing `includes` fails loudly:

```ruby
Listing.strict_loading.limit(50).each { |l| l.product }   # raises on l.product
class Order < ApplicationRecord
  has_many :order_items, strict_loading: true             # per association
  self.strict_loading_by_default = true                   # per model
end
# config.active_record.strict_loading_by_default = true (app-wide)
# config.active_record.action_on_strict_loading_violation = :log (log instead of raise)
```

- **APM tools and rack-mini-profiler** in staging and production show query counts per request.

## 7. includes vs preload vs eager_load vs joins

| Method | SQL | Loads the association? | Filter or sort by association columns? | Use when |
|---|---|---|---|---|
| `joins(:product)` | `INNER JOIN products` | No: accessing `listing.product` still queries | Yes | You only need to filter, sort or count by the association |
| `left_joins(:product)` | `LEFT OUTER JOIN` | No | Yes | Same, but keep parents without children |
| `preload(:product)` | Separate query: `WHERE id IN (...)` | Yes | No (referencing `products` in `where` fails) | You display the association; works with polymorphic associations |
| `eager_load(:product)` | One query with `LEFT OUTER JOIN` and aliased columns | Yes | Yes | You need to filter or sort by it and also display it |
| `includes(:product)` | `preload` by default; switches to `eager_load` when the association's table is referenced | Yes | Yes, once it switches | The default choice; let Rails decide |

**When `includes` switches to one JOIN query:**

```ruby
Listing.includes(:product).where(products: { platform: "N64" })          # hash condition on the table
Listing.includes(:product).where("products.title ILIKE ?", "%mario%").references(:products)  # string needs references
Listing.includes(:product).order("products.title").references(:products)
```

**Combine them:** filter with `joins`, load with `preload` or `includes`:

```ruby
Order.joins(:customer).where(users: { country: "PK" }).preload(:order_items)
```

**Gotcha with eager_load filters:** `Store.includes(:listings).where(listings: { active: true })` returns only stores that have active listings, and each `store.listings` then contains **only** the active ones. If you need all listings for stores that have at least one active listing, filter with a subquery or `joins(...).distinct` and preload separately.

**Why not always `eager_load`?** A single JOIN across two `has_many` associations multiplies rows (10 orders x 10 items x 5 payments = 500 rows per store), and Rails must de-duplicate them in Ruby. Separate `preload` queries are often faster for collections.

## 8. Nested includes and counting without N+1

```ruby
Order.includes(:customer, order_items: { listing: :product })   # hash = nested
Store.includes(listings: [:product, :channel_listings])         # array = several at one level
House.includes(rooms: :residents).find(params[:id])
```

Counting per parent without N COUNT queries:

```ruby
# Option 1: counter cache column (stores.listings_count), then store.listings.size
# Option 2: preload and count in memory
stores = Store.includes(:listings)
stores.each { |s| s.listings.size }        # size uses the loaded records; count would query again

# Option 3: one grouped query
counts = Listing.where(store_id: store_ids).group(:store_id).count   # => { 3 => 120, 4 => 87 }
```

## 9. Calculations and raw SQL done safely

```ruby
Order.where(store_id: 3).count
Order.where(status: :paid).sum(:total_cents)
Listing.average(:price_cents)
Listing.maximum(:updated_at)
Order.group(:store_id).sum(:total_cents)                 # => { 3 => 125_000, 4 => 98_000 }
Order.group("DATE(created_at)").count                     # string grouping expression
```

**SQL injection** happens when user input becomes part of the SQL text:

```ruby
# DANGEROUS: interpolation
Product.where("title = '#{params[:q]}'")
# params[:q] = "x' OR '1'='1"  =>  WHERE title = 'x' OR '1'='1'  (every row)

# SAFE: the value is sent separately or quoted by Rails
Product.where(title: params[:q])                                   # hash: always safe
Product.where("title = ?", params[:q])                             # positional placeholder
Product.where("price_cents < :max", max: params[:max].to_i)        # named placeholder
Product.where("title ILIKE ?", "%#{Product.sanitize_sql_like(params[:q])}%")  # escape % and _
Product.find_by_sql(["SELECT * FROM products WHERE store_id = ? LIMIT 10", store.id])
sql = Product.sanitize_sql_array(["UPDATE products SET stock = 0 WHERE id = ?", id])
```

Column names and sort directions cannot be placeholders, so whitelist them:

```ruby
SORTABLE = %w[title price_cents created_at].freeze
column    = SORTABLE.include?(params[:sort]) ? params[:sort] : "created_at"
direction = params[:dir] == "asc" ? :asc : :desc
Product.order(column => direction)
```

Recent Rails versions refuse obviously raw SQL in `order` and `pluck` unless you wrap it in `Arel.sql(...)`, which is a hint to double-check that no user input is inside. For complex reports, `ActiveRecord::Base.connection.select_all(sql)` returns rows as hashes without building models.

### Going deeper: Arel

**Arel** is the SQL AST library underneath Active Record. It lets you build conditions without strings, but it is mostly an internal API, so keep usage small:

```ruby
products = Product.arel_table
Product.where(products[:price_cents].gt(1000).and(products[:title].matches("%zelda%")))
```

## 10. to_sql and explain

```ruby
Listing.joins(:product).where(products: { platform: "N64" }).to_sql
# => SELECT "listings".* FROM "listings" INNER JOIN "products" ON "products"."id" = "listings"."product_id"
#    WHERE "products"."platform" = 'N64'

Listing.where(store_id: 3, active: true).order(:price_cents).explain
# Prints the database plan: look for "Seq Scan" on big tables (missing index)
# versus "Index Scan" / "Index Only Scan"
```

The workflow for a slow page: find the slow query in the logs or APM, reproduce it with `to_sql`, run `explain` (recent versions accept options such as `explain(:analyze)`; otherwise run `EXPLAIN ANALYZE` in `psql`), then add or fix an index that matches the `WHERE` and `ORDER BY` columns. Index design is covered in [indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md).

## Common pitfalls
- `relation.present?` or `.blank?` just to check existence: loads every row. Use `exists?` / `any?`.
- `.count` on preloaded associations inside loops: one COUNT per parent. Use `size` or a counter cache.
- `Model.all.each` over a big table in a job: memory blowup. Use `find_each` or `in_batches`.
- Adding `.order(...)` before `find_each` and expecting it to be respected (it is ignored in favor of primary key order).
- `joins` on a `has_many` without `distinct`, producing duplicate parents and wrong counts.
- `includes` plus a filter on the association, then being surprised that the loaded collection is filtered too.
- Interpolating params into `where`, `order` or `find_by_sql` strings.
- `where.not(status: "archived")` also excludes rows where `status` is NULL (SQL three-valued logic); add `.or(where(status: nil))` if NULLs should be kept.
- Over-eager loading: `includes` of associations the page never uses (Bullet reports these too).
- Class-method "scopes" that return nil and break chaining.

## How to talk about this in an interview
- N+1 is the most common Rails performance bug in legacy monoliths, so have one concrete story: the page, the repeated query in the log, the `includes` fix and the before/after query count.
- CrystalCommerce works at real scale (700+ stores, 8M+ customers), which makes `find_each`, batched `update_all`, grouped counts and indexes natural talking points for catalog imports and PriceCharting syncs.
- For the Colochousing API, N+1s often hide in JSON responses (bills with shares and residents, events with attendees); mention preloading in the controller before serializing.
- If you have not personally measured an improvement, describe the method rather than inventing numbers: "I would confirm with the log's query count and `explain`."

> **Fill in:** an N+1 or slow query you found at Devntech: where it was, how you spotted it, and the query count before and after the fix.

> **Fill in:** how the PriceCharting catalog import iterated over listings (batches, batch size, upserts) if you know.

## Interview questions

**Q1. What is the N+1 query problem, and how do you detect it?**
Loading N records with one query, then running one more query per record for an association: 1 + N queries. You see it in the log as the same SELECT repeated with different IDs. Bullet flags it automatically, `strict_loading` makes lazy loads raise, and APM tools show high query counts per request.
*Follow-up:* Where does it hide besides views? Serializers, mailers, jobs and per-row `count` calls.

**Q2. How do you fix an N+1?**
Load the association up front: `Listing.includes(:product)`, nested as `Order.includes(order_items: { listing: :product })`, so Rails runs one extra query per association instead of one per record. For counts, use a counter cache, a grouped count or preloading plus `size`.
*Follow-up:* Can you over-fix it? Yes: eager loading associations you never use wastes queries and memory.

**Q3. What is the difference between `includes`, `joins`, `preload` and `eager_load`?**
`joins` adds an INNER JOIN for filtering or sorting but loads nothing. `preload` loads associations with separate `IN` queries. `eager_load` loads them in one LEFT OUTER JOIN query, so you can also filter on them. `includes` uses preload by default and switches to eager_load when the query references the association's table.
*Follow-up:* Which one works for polymorphic associations? `preload` (and `includes` in preload mode); `eager_load` cannot join an unknown table.

**Q4. When does `includes` switch to a JOIN?**
When conditions or ordering reference the included table: a hash condition like `where(products: { platform: "N64" })`, or a string condition or order combined with `.references(:products)`.
*Follow-up:* Side effect? The loaded association contains only the rows that matched the condition.

**Q5. Why can `joins` return duplicate records?**
An INNER JOIN on a `has_many` yields one row per matching child, so a store with three matching listings appears three times. Add `distinct`, or filter with a subquery: `Store.where(id: Listing.active.select(:store_id))`.
*Follow-up:* Does `distinct` affect `count`? `relation.distinct.count` counts distinct rows; check the SQL with `to_sql`.

**Q6. What is the difference between `count`, `size` and `length`?**
`count` always runs `SELECT COUNT(*)`; `length` loads all records and counts them in Ruby; `size` counts in memory if loaded, uses a counter cache for associations if present, and otherwise runs COUNT.
*Follow-up:* Which one inside a loop over preloaded parents? `size`.

**Q7. What SQL do `exists?`, `any?` and `present?` run?**
`exists?` and `any?` (on an unloaded relation) run `SELECT 1 AS one ... LIMIT 1`. `present?` and `blank?` load all matching records, because they work on the loaded array.
*Follow-up:* When is `present?` fine? When you are about to iterate the same records anyway, since they are then cached.

**Q8. When does an ActiveRecord relation actually run its query?**
Only when the data is needed: iteration (`each`, `map`), `to_a`, `load`, `first`, `last`, calculations like `count` and `pluck`, or when the console inspects it. Until then, methods like `where` and `order` just build a new relation, which is why queries compose.
*Follow-up:* What does `load_async` do? Starts the query on a background thread so independent queries can overlap (Rails 7.0).

**Q9. What is the difference between a scope and a class method?**
Both return relations and chain, but a scope always returns a relation: if its body returns nil or false, Rails returns `all`. A class method returns whatever you return, so a nil breaks the chain. Use scopes for simple filters and class methods for multi-step logic that explicitly returns `all` or a relation.
*Follow-up:* Do scopes work on associations? Yes: `store.listings.active`, and so do class methods.

**Q10. What does `find_each` do, and why use it?**
It loads records in batches (1000 by default) ordered by primary key, yielding them one at a time, so memory stays flat even across millions of rows. `Model.all.each` would load everything at once. It uses keyset paging, so it stays fast on deep batches.
*Follow-up:* What happens to a custom `order`? It is ignored (primary key order is required); use `in_batches` for set-based updates per batch.

**Q11. What is the difference between `find_or_create_by` and `create_or_find_by`?**
`find_or_create_by` selects first and inserts if nothing is found, so two concurrent callers can both insert. `create_or_find_by` inserts first and, when a unique constraint rejects it, selects the existing row; it needs a unique index and costs a savepoint and a burned ID on conflicts.
*Follow-up:* What about bulk syncs? `upsert_all` with `unique_by:` handles thousands of rows in one statement.

**Q12. What is the difference between `pluck`, `select` and `map`?**
`pluck(:id, :name)` returns raw values without instantiating models (fast, low memory). `select(:id, :name)` returns models with only those attributes loaded. `map(&:name)` loads full models first and then extracts in Ruby.
*Follow-up:* When is `map` acceptable? When the records are already loaded for other reasons.

**Q13. How do you prevent SQL injection in ActiveRecord?**
Never interpolate input into SQL strings. Use hash conditions, `?` or named placeholders, `sanitize_sql_like` for LIKE patterns, and bound arrays in `find_by_sql`. Whitelist anything that cannot be a placeholder, such as column names and sort directions.
*Follow-up:* Show an attack. `where("title = '#{q}'")` with `q = "x' OR '1'='1"` returns every row.

**Q14. How would you find stores that have no listings?**
`Store.where.missing(:listings)` (6.1), which runs a LEFT OUTER JOIN with `listings.id IS NULL`; or `Store.left_joins(:listings).where(listings: { id: nil })`; or `Store.where.not(id: Listing.select(:store_id))`.
*Follow-up:* And stores with at least one? `Store.where.associated(:listings)` (7.0) or `joins(:listings).distinct`.

**Q15. How do you investigate a slow ActiveRecord query?**
Find it in logs or APM, print it with `to_sql`, run `explain` (or `EXPLAIN ANALYZE` in `psql`) and look for sequential scans on big tables or huge row estimates, then add an index matching the filter and sort columns, or rewrite the query (fewer columns with `select`, avoid N+1, avoid big offsets).
*Follow-up:* Why might an index not be used? Low selectivity, a function applied to the column, or a mismatched composite column order.

**Q16. What is `strict_loading`?**
A Rails 6.1 feature that raises `StrictLoadingViolationError` (or logs) when an association is lazily loaded. You enable it per query (`.strict_loading`), per record, per association, per model or app-wide, to catch missing `includes` during development and tests.
*Follow-up:* Why not enable it app-wide on day one in a legacy app? Every existing lazy load would start raising; roll it out gradually, starting with logging.

## Further reading
- [Active Record Query Interface](https://guides.rubyonrails.org/active_record_querying.html)
- [ActiveRecord::QueryMethods API](https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html)
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
