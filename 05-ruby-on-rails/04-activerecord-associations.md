# ActiveRecord Associations

> **Tier:** T1 (know cold) · **Related:** [ActiveRecord basics](03-activerecord-basics.md), [Querying and N+1](05-activerecord-querying-n-plus-1.md), [ActiveRecord advanced](06-activerecord-advanced.md), [DBMS fundamentals](../12-databases/01-dbms-fundamentals.md), [E-commerce domain](../13-architecture-and-practices/07-ecommerce-domain.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md)
> **On your resume:** data modeling in the CrystalCommerce multi-store catalog (stores, products, listings, orders, PriceCharting channel) and the Colochousing co-living domain (houses, residents, events, tasks, split bills, rent payments).

## Quick revision
- `belongs_to` goes on the model whose table holds the foreign key; `has_one` / `has_many` go on the other side.
- `belongs_to` is **required by default since Rails 5** (error "Store must exist"); use `optional: true` and a nullable column when the parent is optional.
- `has_many :through` uses a real join model (extra columns, validations, callbacks); `has_and_belongs_to_many` uses a bare join table. Default to `:through`.
- `has_one :through` reaches one record via another association (`Resident has_one :house, through: :room`).
- `dependent:` `:destroy` (callbacks, one query per child), `:delete_all` (one SQL DELETE), `:nullify` (FK set to NULL), `:restrict_with_error` (adds an error, destroy returns false), `:restrict_with_exception` (raises), `:destroy_async` (background job, 6.1).
- `class_name:` and `foreign_key:` break naming conventions; `inverse_of:` links both sides in memory (needed when Rails cannot infer it, for example with a custom `foreign_key`).
- `counter_cache: true` keeps `parent.children_count` updated by callbacks, so `size` needs no query; bulk SQL bypasses it, and `reset_counters` repairs it.
- `touch: true` bumps the parent's `updated_at`; `autosave: true` saves changed, already-loaded children with the parent.
- **Polymorphic** (`belongs_to :commentable, polymorphic: true`): one child, many parent types, no FK constraint. **STI**: subclasses share one table with a `type` column. **delegated_type** (6.1): shared parent table plus one table per type.
- Self-referential: `belongs_to :manager, class_name: "Employee"`; followers through a `Follow` join model with two `belongs_to ..., class_name: "User"` and `source:`.
- Collection methods: `build` (memory), `create` (saves), `<<` (saves when the owner is persisted), `ids`, `ids=` (replaces the set immediately).
- `accepts_nested_attributes_for :items, allow_destroy: true` accepts `items_attributes` (with `id` and `_destroy`) and saves parent and children together.
- Scoped association: `has_many :active_listings, -> { where(active: true) }, class_name: "Listing"`. A `limit` inside it does not work per parent when preloading.

## 1. Association basics: the foreign key decides

```ruby
class Store < ApplicationRecord
  has_many :products, dependent: :destroy
  has_one  :setting, class_name: "StoreSetting", dependent: :destroy  # store_settings.store_id
end

class Product < ApplicationRecord
  belongs_to :store                    # products.store_id; required by default
  belongs_to :category, optional: true # category_id may be NULL
end
```

| Macro | Foreign key lives on | Example |
|---|---|---|
| `belongs_to` | This model's table | `Product belongs_to :store` (`products.store_id`) |
| `has_one` | The other table | `Store has_one :setting` (`store_settings.store_id`) |
| `has_many` | The other table | `Store has_many :products` |
| `has_many :through` | A join model | `Product has_many :tags, through: :taggings` |
| `has_one :through` | Reached through another association | `Resident has_one :house, through: :room` |
| `has_and_belongs_to_many` | A join table with no model | `products_tags (product_id, tag_id)` |

Since Rails 5, `belongs_to` adds a presence validation for the parent object. Apps upgraded from Rails 4 sometimes still have `belongs_to_required_by_default = false` in their framework defaults, so check before assuming. Back the rule with `null: false` and a foreign key in the migration (`t.references :store, null: false, foreign_key: true`).

## 2. has_many :through vs has_and_belongs_to_many

```ruby
# has_many :through: the join is a real model
class Tagging < ApplicationRecord
  belongs_to :product
  belongs_to :tag
  validates :tag_id, uniqueness: { scope: :product_id }
end

class Product < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
end

class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :products, through: :taggings
end

# HABTM: no model, just a join table
class Product < ApplicationRecord
  has_and_belongs_to_many :tags
end
# migration: create_join_table :products, :tags do |t|
#              t.index [:product_id, :tag_id], unique: true
#            end
```

| | `has_many :through` | `has_and_belongs_to_many` |
|---|---|---|
| Join model | Yes (`Tagging`) | No, only a table (`products_tags`) |
| Extra columns (position, added_by, created_at) | Yes | No practical way |
| Validations and callbacks on the join | Yes | No |
| Query the join directly | `Tagging.where(...)` | Awkward |
| Setup | Three classes | One line per side |

Prefer `has_many :through`. Requirements almost always grow ("when was this tag added, and by whom?"), and converting a HABTM later needs a migration. HABTM is fine for a truly dumb link that will never carry data.

## 3. has_one :through and nested through

```ruby
class House < ApplicationRecord
  has_many :rooms
  has_many :residents, through: :rooms     # house.residents: one JOIN through rooms
end

class Room < ApplicationRecord
  belongs_to :house
  has_many :residents
end

class Resident < ApplicationRecord
  belongs_to :room, optional: true
  has_one :house, through: :room           # resident.house without storing house_id
end
```

Through associations are read-mostly: you can read `resident.house` freely, but you change it by changing the `room`. If you query by house constantly, a denormalized `house_id` on residents can be worth its sync cost.

## 4. Association options

| Option | Effect |
|---|---|
| `dependent: :destroy` | Load each child and call `destroy` (callbacks and nested dependents run; one query per child) |
| `dependent: :delete_all` | One SQL DELETE for all children, no callbacks (`has_many`; `has_one` uses `:delete`) |
| `dependent: :nullify` | Set the children's FK to NULL; children survive |
| `dependent: :restrict_with_error` | Refuse to destroy the parent if children exist; adds an error and `destroy` returns false |
| `dependent: :restrict_with_exception` | Same, but raises `ActiveRecord::DeleteRestrictionError` |
| `dependent: :destroy_async` | Enqueue a job that destroys the children later (Rails 6.1) |
| `class_name:` / `foreign_key:` | Association name or FK column that does not follow the convention |
| `inverse_of:` | Declares the other side, so both point to the same object in memory |
| `counter_cache:` | Maintain a count column on the parent |
| `touch:` | Update the parent's `updated_at` on save or destroy |
| `autosave:` | `true` saves loaded children whenever the parent saves; `false` never saves them |

```ruby
class Order < ApplicationRecord
  belongs_to :customer, class_name: "User", foreign_key: :buyer_id, inverse_of: :purchases
  has_many :order_items, dependent: :destroy, inverse_of: :order
end

class User < ApplicationRecord
  has_many :purchases, class_name: "Order", foreign_key: :buyer_id, inverse_of: :customer
end
```

**inverse_of:** Rails detects simple inverses itself but gives up with options such as a custom `foreign_key` or a scope. With the inverse declared, `order.order_items.first.order` returns the already-loaded `order` instead of querying again, and `order.order_items.build` sets `item.order` to the unsaved parent, so the child's `belongs_to` presence validation passes when both are saved together.

**counter_cache:**

```ruby
class Product < ApplicationRecord
  belongs_to :store, counter_cache: true   # needs stores.products_count
end

# migration
add_column :stores, :products_count, :integer, null: false, default: 0
# backfill once (from a task, not the migration)
Store.find_each { |store| Store.reset_counters(store.id, :products) }

store.products.size     # reads products_count, no query
store.products.count    # always runs SELECT COUNT(*)
```

Rails updates the column with an atomic `UPDATE ... SET products_count = products_count + 1` from create and destroy callbacks (and when the FK changes). It is **not** updated by `delete_all`, `insert_all`, `update_all` or raw SQL, and a very busy parent row becomes a lock hotspot. Use `counter_cache: :listings_count` for a custom column name.

**Database cascades vs `dependent:`:** `add_foreign_key :products, :stores, on_delete: :cascade` lets the database delete children in one step, faster and safe against bulk deletes, but no Ruby callbacks run. `dependent: :destroy` runs callbacks but is slow for thousands of children. Choose one deliberately per relationship, and for huge child sets prefer `:delete_all`, `:destroy_async` or a background job.

## 5. Polymorphic associations vs STI vs delegated_type

**Polymorphic:** one child model belongs to several parent types.

```ruby
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true   # commentable_type + commentable_id
end

class Event < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

class Task < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# migration: t.references :commentable, polymorphic: true, null: false  (type, id and an index)
```

**Single Table Inheritance (STI):** a class hierarchy stored in one table, with the class name in `type`.

```ruby
class Payment < ApplicationRecord          # payments table has a string "type" column
  def receipt_label = "Payment"
end

class CardPayment < Payment
  def receipt_label = "Card ending #{card_last4}"
end

class CashPayment < Payment; end

CardPayment.create!(amount_cents: 5000, card_last4: "4242")  # type = "CardPayment"
Payment.all          # returns CardPayment and CashPayment instances
CardPayment.all      # WHERE type = 'CardPayment'
```

**delegated_type (6.1):** a shared parent table for common columns, plus one table per type for type-specific columns. A house activity feed is a good fit:

```ruby
class FeedItem < ApplicationRecord          # house_id, author_id, created_at, feedable_type, feedable_id
  belongs_to :house
  delegated_type :feedable, types: %w[Event Task Bill], dependent: :destroy
end

module Feedable
  extend ActiveSupport::Concern
  included do
    has_one :feed_item, as: :feedable, touch: true
  end
end

class Event < ApplicationRecord
  include Feedable                          # events table keeps starts_at, location, ...
end

house.feed_items.order(created_at: :desc).limit(20)  # one table to paginate
feed_item.event?                            # type predicate
FeedItem.events                             # scope for one type
```

| | Polymorphic `belongs_to` | STI | `delegated_type` |
|---|---|---|---|
| Solves | One child, many parent types | Subclasses with shared columns, different behavior | Hierarchy where types have their own columns |
| Tables | Child table with `*_type` + `*_id` | One table for all subclasses | Parent table + one table per type |
| Foreign key constraint | Impossible on the polymorphic pair | Normal | Normal inside each type table; not on the delegated pair |
| Sparse NULL columns | n/a | Grows with each subclass's columns | None |
| Query across types | No JOIN across types; `eager_load` fails, `preload` runs one query per type | Trivial (one table) | Easy on the parent table |
| Typical example | Comments on events and tasks | Card vs cash payments | A feed of events, tasks and bills |

Watch-outs: polymorphic and STI both store class names as strings, so renaming a class needs a data migration. STI reserves the `type` column name (set `self.inheritance_column` if a legacy table uses `type` for something else).

## 6. Self-referential associations

A model can point at its own table. Manager and direct reports:

```ruby
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true, inverse_of: :reports
  has_many :reports, class_name: "Employee", foreign_key: :manager_id,
                     inverse_of: :manager, dependent: :nullify
end
```

Followers (many-to-many between users) need a join model with two foreign keys to the same table:

```ruby
# migration
create_table :follows do |t|
  t.references :follower, null: false, foreign_key: { to_table: :users }
  t.references :followed, null: false, foreign_key: { to_table: :users }
  t.timestamps
end
add_index :follows, [:follower_id, :followed_id], unique: true

class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
end

class User < ApplicationRecord
  has_many :active_follows,  class_name: "Follow", foreign_key: :follower_id, dependent: :destroy
  has_many :passive_follows, class_name: "Follow", foreign_key: :followed_id, dependent: :destroy
  has_many :following, through: :active_follows,  source: :followed
  has_many :followers, through: :passive_follows, source: :follower
end
```

`source:` tells the through association which `belongs_to` on the join model to follow, because the association name (`following`) does not match it.

## 7. Methods that associations add

```ruby
store.products                      # CollectionProxy: a lazy, chainable relation
store.products.build(name: "Mug")   # new Product with store_id set; not saved
store.products.create!(name: "Mug") # saves immediately (store must be persisted)
store.products << product           # sets store_id and saves at once if store is persisted
store.products.delete(product)      # unlinks: nullify by default, or per the dependent option
store.products.destroy(product)     # destroys the record
store.product_ids                   # => [1, 2, 3]
store.product_ids = [1, 2]          # replaces the whole set right away (writes immediately!)
store.products.exists?(sku: "X-1")
store.products.where(active: true).order(:name)  # chain like any relation

product.store                       # belongs_to reader (cached after the first load)
product.build_store(name: "New")    # singular builders: build_x / create_x / create_x!
```

`tag_ids=` with checkboxes is convenient (`permit(tag_ids: [])`), but remember it writes on assignment for persisted records, before your later `save` and outside its validations.

## 8. accepts_nested_attributes_for

Lets one form and one `save` create, update and delete children together, in one transaction:

```ruby
class Order < ApplicationRecord
  has_many :order_items, dependent: :destroy, inverse_of: :order
  accepts_nested_attributes_for :order_items, allow_destroy: true, reject_if: :all_blank
end

# The params it expects
# { order: { note: "Gift", order_items_attributes: [
#     { id: 5, quantity: 2 },          # update an existing child
#     { listing_id: 9, quantity: 1 },  # create a new one
#     { id: 6, _destroy: "1" }         # delete one (needs allow_destroy)
# ] } }

params.require(:order).permit(:note, order_items_attributes: [:id, :listing_id, :quantity, :_destroy])
```

In the view, `form.fields_for :order_items` renders the nested fields (see [forms](07-views-erb-forms-assets.md)). It is great for simple parent-child forms; for complex multi-model workflows, a form object (a plain class including `ActiveModel::Model`) is easier to validate and test. Other options: `limit:` (max children) and `update_only:` (for `has_one`).

## 9. Scoped associations

```ruby
class Store < ApplicationRecord
  has_many :listings
  has_many :active_listings, -> { where(active: true) }, class_name: "Listing"
  has_many :recent_orders, -> { order(created_at: :desc) }, class_name: "Order"
end

class Resident < ApplicationRecord
  has_one :latest_rent_payment, -> { order(period: :desc) }, class_name: "RentPayment"
end

store.active_listings.build(price_cents: 999)  # active: true is set automatically from the where hash
Store.includes(:active_listings)               # scoped associations can be preloaded too
```

Do not rely on `limit` inside an association scope when preloading: the limit applies to the whole preload query, not to each parent, so most parents come back with nothing.

## 10. Worked model: a co-living platform (illustrative)

This is a typical way to model a co-living product like the one behind your Colochousing work. It is an illustrative model, not the real schema.

```text
House 1--* Room 1--* Resident            (resident.room_id; nil after move-out)
House 1--* Event   (organizer: Resident)
House 1--* Task    (assignee: Resident, optional)
House 1--* Bill    (paid_by: Resident) 1--* BillShare *--1 Resident   (split bills)
Resident 1--* RentPayment                (one per resident per month)
```

```ruby
class House < ApplicationRecord
  has_many :rooms, dependent: :restrict_with_error
  has_many :residents, through: :rooms
  has_many :events, dependent: :destroy
  has_many :tasks, dependent: :destroy
  has_many :bills, dependent: :restrict_with_error
end

class Room < ApplicationRecord
  belongs_to :house
  has_many :residents, dependent: :nullify
end

class Resident < ApplicationRecord
  belongs_to :room, optional: true
  has_one :house, through: :room
  has_many :organized_events, class_name: "Event", foreign_key: :organizer_id, inverse_of: :organizer
  has_many :assigned_tasks, class_name: "Task", foreign_key: :assignee_id,
                            inverse_of: :assignee, dependent: :nullify
  has_many :bill_shares, dependent: :restrict_with_error
  has_many :bills, through: :bill_shares
  has_many :rent_payments, dependent: :restrict_with_error   # never lose payment history
end

class Event < ApplicationRecord
  belongs_to :house
  belongs_to :organizer, class_name: "Resident", inverse_of: :organized_events
end

class Task < ApplicationRecord
  belongs_to :house
  belongs_to :assignee, class_name: "Resident", optional: true, inverse_of: :assigned_tasks
  enum :status, { open: 0, done: 1 }
end

class Bill < ApplicationRecord
  belongs_to :house
  belongs_to :paid_by, class_name: "Resident"
  has_many :bill_shares, dependent: :destroy, inverse_of: :bill
  has_many :residents, through: :bill_shares
  accepts_nested_attributes_for :bill_shares
  validate :shares_add_up

  private

  def shares_add_up
    return if bill_shares.sum(&:amount_cents) == amount_cents
    errors.add(:base, "Shares must add up to the bill total")
  end
end

class BillShare < ApplicationRecord
  belongs_to :bill
  belongs_to :resident
  validates :resident_id, uniqueness: { scope: :bill_id }   # plus a unique index
end

class RentPayment < ApplicationRecord
  belongs_to :resident
  enum :status, { pending: 0, paid: 1, failed: 2, refunded: 3 }
  validates :period, uniqueness: { scope: :resident_id }    # plus a unique index
end
```

Design points to say out loud: money in integer cents; split shares stored per resident so "who owes what" is a simple query; unique indexes on `(bill_id, resident_id)` and `(resident_id, period)` so retries and double taps cannot duplicate rows; `restrict_with_error` on financial records so deleting a resident cannot wipe payment history. See [payments and split bills](../08-react-native-expo/06-payments-and-split-bills.md) for the mobile side.

## 11. Worked model: a multi-store e-commerce catalog (illustrative)

A typical way to model a platform where many stores sell from a shared catalog and also publish listings to external sales channels, like the CrystalCommerce and PriceCharting work. Again illustrative, not the real schema.

```text
Product (shared catalog entry, e.g. one game title)
  1--* Listing *--1 Store          (a store's offer: condition, price, stock)
Listing 1--* ChannelListing        (the same offer published on an external channel)
Store 1--* Order 1--* OrderItem *--1 Listing   (price copied at checkout)
```

```ruby
class Store < ApplicationRecord
  has_many :listings, dependent: :destroy
  has_many :products, -> { distinct }, through: :listings
  has_many :orders, dependent: :restrict_with_exception
end

class Product < ApplicationRecord
  has_many :listings, dependent: :restrict_with_error
  has_many :stores, through: :listings
end

class Listing < ApplicationRecord
  belongs_to :store, counter_cache: true          # stores.listings_count
  belongs_to :product
  has_many :order_items, dependent: :restrict_with_error
  has_many :channel_listings, dependent: :destroy
  enum :condition, { brand_new: 0, used: 1, graded: 2 }  # a value named "new" would clash with Listing.new
  validates :price_cents, numericality: { greater_than_or_equal_to: 0 }
  validates :product_id, uniqueness: { scope: [:store_id, :condition] }
end

class ChannelListing < ApplicationRecord
  belongs_to :listing
  enum :sync_status, { pending: 0, synced: 1, failed: 2 }
  validates :channel, presence: true
  validates :external_id, uniqueness: { scope: :channel }, allow_nil: true
end

class Order < ApplicationRecord
  belongs_to :store
  belongs_to :customer, class_name: "User"
  has_many :order_items, dependent: :destroy, inverse_of: :order
  has_many :listings, through: :order_items
end

class OrderItem < ApplicationRecord
  belongs_to :order
  belongs_to :listing
  validates :quantity, numericality: { only_integer: true, greater_than: 0 }
  # unit_price_cents and title are copied from the listing at checkout
end
```

Design points: `OrderItem` snapshots the price because listing prices change (for example after a market-price import), and an old order must still show what the customer paid. `ChannelListing` keeps the external representation (external ID, sync status, last synced time) separate from the store's own listing, and the unique `(channel, external_id)` index makes sync jobs idempotent through upserts (see [background jobs](09-background-jobs-sidekiq.md) and [third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md)). Stock lives on `Listing` and must be decremented with locking (see [ActiveRecord advanced](06-activerecord-advanced.md)).

## Common pitfalls
- Putting `belongs_to` on the wrong side: it always goes on the table that has the `_id` column.
- Adding `optional: true` to silence "must exist" errors when the real bug is a missing parent.
- `dependent: :destroy` on a parent with tens of thousands of children: one request runs thousands of queries and may time out.
- No `dependent:` option and no DB foreign key, leaving orphaned rows after deletes.
- Forgetting the unique index on join tables (`taggings`, `bill_shares`, `follows`), so duplicates appear under concurrency.
- Counter caches drifting after `delete_all`, imports or raw SQL; schedule `reset_counters` or avoid bulk paths.
- Using HABTM and later needing a timestamp or extra column on the join.
- Polymorphic associations where a real foreign key matters (financial data): the database cannot protect integrity.
- Assigning `*_ids=` on a persisted record and assuming nothing is written until `save`.

## How to talk about this in an interview
- These domains are your strongest modeling stories. Describe the shape of the data (stores, catalog products, listings, orders, external channel listings; houses, residents, bills and shares) and the reasons behind the choices: snapshots, unique indexes, restrict on financial records.
- Be explicit that you are describing how you would model it or how such systems are typically modeled, unless you are sure of the real schema.
- Legacy monoliths often contain polymorphic associations and STI hierarchies; mention that you check `*_type` values and `type` columns in the database before refactoring them.

> **Fill in:** two or three real associations you worked with in CrystalCommerce (for example how listings or orders relate to stores) and one you changed.

> **Fill in:** how split bills and rent payments were represented in the Colochousing Rails API that the resident app consumed.

## Interview questions

**Q1. How do you decide where `belongs_to` goes?**
On the model whose table stores the foreign key: `products.store_id` means `Product belongs_to :store`, and the other side gets `has_many` or `has_one`.
*Follow-up:* What if the FK column has a non-standard name? `belongs_to :customer, class_name: "User", foreign_key: :buyer_id`.

**Q2. Why does saving fail with "Store must exist"?**
Since Rails 5, `belongs_to` validates that the associated record is present. Either assign the parent or mark the association `optional: true` (and allow NULL in the column) when it is genuinely optional.
*Follow-up:* Why might an old app not show this error? It may still set `belongs_to_required_by_default = false` from its Rails 4 era defaults.

**Q3. What is the difference between `has_many :through` and `has_and_belongs_to_many`?**
`has_many :through` goes through a real join model, so the join can have extra columns, validations, callbacks and its own queries. HABTM uses a bare join table with only the two foreign keys. Prefer `:through`, because join data almost always grows.
*Follow-up:* What naming does a HABTM table need? Both table names in alphabetical order, such as `products_tags`.

**Q4. Explain the `dependent:` options.**
`:destroy` loads and destroys each child with callbacks; `:delete_all` deletes children in one SQL statement without callbacks; `:nullify` sets their FK to NULL; `:restrict_with_error` blocks the parent's destroy with an error; `:restrict_with_exception` raises; `:destroy_async` destroys children in a background job.
*Follow-up:* Which would you use for rent payments? A restrict option, because financial history should never be deleted as a side effect.

**Q5. When would you use a polymorphic association instead of STI?**
Polymorphic solves "one child, many parent types" (comments on events and tasks) with `*_type` and `*_id` columns. STI solves "subclasses of one concept sharing a table" (card and cash payments) with a `type` column. Polymorphic cannot have a foreign key constraint; STI tables grow sparse when subclasses need different columns.
*Follow-up:* Can you `eager_load` a polymorphic association? No, because there is no single table to join; `preload` works with one query per type.

**Q6. What is `delegated_type`, and when is it better than STI?**
Added in Rails 6.1: a parent table holds the shared columns plus a `*_type`/`*_id` pair, and each type has its own table for specific columns. Use it when types share some data and must be listed together (a feed, a timeline) but have very different attributes, which would make an STI table sparse.
*Follow-up:* What helpers does it add? Type predicates like `feed_item.event?`, accessors like `feed_item.event`, and scopes like `FeedItem.events`.

**Q7. What is `counter_cache`, and what are its caveats?**
`belongs_to :store, counter_cache: true` keeps `stores.products_count` updated on child create and destroy, so `store.products.size` needs no COUNT query. Bulk operations and raw SQL bypass it, so it can drift (fix with `reset_counters`), and the parent row becomes a write hotspot under heavy inserts.
*Follow-up:* `size` vs `count` here? `size` reads the cache; `count` always queries.

**Q8. What does `inverse_of` do?**
It tells Rails that two associations are opposite sides of the same relationship, so loaded objects are shared in memory (no extra query for `item.order`) and children built through an unsaved parent see that parent, which lets presence validations pass. Rails infers simple cases but not custom foreign keys or scoped associations.
*Follow-up:* Symptom of a missing inverse? Extra queries, or "must exist" errors when saving a new parent with nested children.

**Q9. How would you model users following users?**
A `Follow` join model with `follower_id` and `followed_id`, both referencing `users`, and a unique index on the pair. On `User`, two `has_many` associations to `Follow` with explicit foreign keys, and two `has_many :through` associations using `source:`.
*Follow-up:* How do you prevent self-follows? A validation plus a check constraint `follower_id <> followed_id`.

**Q10. What does `accepts_nested_attributes_for` do?**
It adds a `children_attributes=` writer so a parent's `save` can create, update (rows with `id`) and destroy (rows with `_destroy`, if `allow_destroy: true`) its children in one transaction, typically from a `fields_for` form. Permit the nested keys in strong params.
*Follow-up:* When would you avoid it? Complex multi-model workflows, where a form object is clearer.

**Q11. What is the difference between `build`, `create` and `<<` on a collection?**
`build` creates an unsaved child with the FK set; `create` saves it immediately; `<<` adds an existing or new object and saves it right away when the owner is persisted. `ids=` replaces the whole collection immediately.
*Follow-up:* What if the owner is a new record? `<<` and builds wait and are saved together with the owner.

**Q12. Why copy the price into `OrderItem` instead of reading it from the listing?**
Listing prices change (manual edits, market-price imports), but an order must keep what the customer actually paid, for receipts, refunds and accounting. The same applies to titles and tax rates.
*Follow-up:* What else would you snapshot? Shipping address, currency and discounts applied.

**Q13. How would you model split bills in a shared house?**
A `Bill` with a total, a payer and a house, plus a `BillShare` per resident with their amount and settled status, a unique index on `(bill_id, resident_id)`, and a validation that the shares add up to the total. Store money as integer cents.
*Follow-up:* How do you show "who owes whom"? Aggregate unsettled shares grouped by resident and payer.

**Q14. What does `autosave` do?**
With `autosave: true`, saving the parent also saves changes to already-loaded children and validates them; by default only new children are saved with a new parent. `accepts_nested_attributes_for` turns autosave on.
*Follow-up:* Risk? Saving a parent can unexpectedly write children you modified earlier.

**Q15. Why can `dependent: :destroy` be dangerous on big associations, and what are the alternatives?**
It loads every child and runs callbacks one by one, which can mean thousands of queries in one request. Alternatives: `:delete_all` when no callbacks are needed, `:destroy_async`, a database `on_delete: :cascade`, or a background job that deletes in batches.
*Follow-up:* Downside of the database cascade? Ruby callbacks and dependent cleanup do not run.

**Q16. What is a scoped association, and what is the gotcha with `limit`?**
An association with a lambda scope, such as `has_many :active_listings, -> { where(active: true) }, class_name: "Listing"`; building through it applies the where values. When preloading, a `limit` in the scope applies to the whole query, not per parent.
*Follow-up:* How do you get the latest payment per resident efficiently? A `has_one` with an order scope for single records, or a window-function query or denormalized column for lists.

## Further reading
- [Active Record Associations](https://guides.rubyonrails.org/association_basics.html)
- [ActiveRecord::Associations::ClassMethods API](https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)
- [ActiveRecord::DelegatedType API](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html)
