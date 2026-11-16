# ActiveRecord Basics: Migrations, CRUD, Validations and Callbacks

> **Tier:** T1 (know cold) · **Related:** [Associations](04-activerecord-associations.md), [Querying and N+1](05-activerecord-querying-n-plus-1.md), [ActiveRecord advanced](06-activerecord-advanced.md), [SQL from zero](../12-databases/02-sql-from-zero.md), [ORMs compared](../12-databases/12-orms-compared.md), [Background jobs](09-background-jobs-sidekiq.md)
> **On your resume:** ActiveRecord and PostgreSQL in the CrystalCommerce catalog and order work, the Colochousing Rails 7 app, legacy Rails monoliths at Devntech, and the Visnext ActiveRecord training.

## Quick revision
- **Active Record pattern:** one object wraps one row and carries both data and persistence (`product.save`). Attributes come from the table's columns at runtime.
- Conventions: class `OrderItem` -> table `order_items`, primary key `id`, foreign key `order_id`, `created_at` / `updated_at` managed automatically.
- Migrations are versioned schema changes. `change` is auto-reversible for most operations; use `up`/`down` or `reversible` for raw SQL and `change_column`.
- `schema.rb` (or `structure.sql`) is the source of truth for building a fresh database (`db:schema:load`); the migration history is not.
- Keep data backfills out of schema migrations: they lock tables, break when models change, and rarely roll back.
- Non-bang (`save`, `update`, `create`) returns false or an invalid object; bang (`save!`, `update!`, `create!`) raises `ActiveRecord::RecordInvalid`. Use bang in jobs, scripts and transactions.
- `find` raises `RecordNotFound`; `find_by` returns nil; `where` returns a lazy, possibly empty relation; `first` orders by primary key, `take` does not order.
- `destroy` loads the record and runs callbacks and `dependent:`; `delete` / `delete_all` run one SQL DELETE with no callbacks.
- `update` validates and runs callbacks; `update_attribute` skips validations; `update_column(s)` and `update_all` skip validations, callbacks and `updated_at`.
- Validations: `presence`, `length`, `numericality`, `inclusion`, `format`, `uniqueness`; custom `validate :method` or an `EachValidator`; `if:`, `unless:`, `on:`.
- `uniqueness` validation has a race condition: two requests can both pass the check. Back it with a unique index and rescue `ActiveRecord::RecordNotUnique`.
- Save callback order: before_validation, after_validation, before_save, around_save, before_create/update, around_create/update, after_create/update, after_save, then after_commit or after_rollback.
- Halt a callback chain with `throw :abort` (returning false stopped working in Rails 5).
- Enqueue jobs and send emails in `after_commit` (`after_create_commit`), never `after_save`: inside the transaction the row is not committed yet, and it may roll back.
- Dirty tracking: `price_changed?` and `will_save_change_to_price?` before saving; `saved_change_to_price?` and `saved_changes` after.
- Enums map names to stored values: `enum :status, { pending: 0, paid: 1 }` gives `paid?`, `paid!` and `Order.paid`. Always use an explicit hash so values never shift.
- `normalizes :email, with: ->(e) { e.strip.downcase }` (Rails 7.1) cleans values on assignment and in `find_by` / `where`.
- `touch` bumps `updated_at` without validations; `belongs_to :store, touch: true` bumps the parent for cache invalidation.

## 1. ORM and the Active Record pattern

An **ORM** (object-relational mapper) lets you work with rows as objects: tables map to classes, rows to instances, columns to attributes, and SQL is generated for you. Rails implements the **Active Record pattern**: each model object knows how to load, save and delete itself.

```ruby
class Product < ApplicationRecord   # ApplicationRecord < ActiveRecord::Base
end

Product.column_names            # => ["id", "name", "price", "store_id", "created_at", ...]
product = Product.new(name: "Mug", price: 9.5)
product.save                    # INSERT INTO "products" ...
product.update(price: 12)       # UPDATE "products" SET "price" = 12 ... WHERE "id" = 1
```

There are no attribute declarations: Rails reads the table schema at boot and defines accessors. The alternative, **Data Mapper**, keeps domain objects ignorant of the database and uses separate mapper or repository classes.

| | Active Record | Data Mapper / query builder |
|---|---|---|
| Persistence lives in | The model (`product.save`) | A separate repository, session or query builder |
| Strength | Very little code for CRUD; models, validations and queries in one place | Clean separation, domain objects easy to test without a DB |
| Weakness | Models grow fat; easy to hide queries and side effects | More boilerplate |
| Examples | Rails ActiveRecord, Laravel Eloquent | Hibernate (JPA), Doctrine; query builders such as Drizzle |

See [ORMs compared](../12-databases/12-orms-compared.md) for Django ORM, Drizzle and Entity Framework side by side.

## 2. Naming conventions

| Thing | Convention | Example |
|---|---|---|
| Model class | Singular, CamelCase | `OrderItem` |
| Table | Plural, snake_case | `order_items` |
| Primary key | `id` (bigint) | `order_items.id` |
| Foreign key | `<singular association>_id` | `order_id` |
| HABTM join table | Both plurals, alphabetical | `products_tags` |
| Timestamps | Auto-managed | `created_at`, `updated_at` |
| Optimistic locking | `lock_version` integer | see [advanced](06-activerecord-advanced.md) |
| STI / polymorphic | `type`; `<name>_type` + `<name>_id` | `commentable_type`, `commentable_id` |
| Counter cache | `<plural>_count` | `stores.products_count` |

Legacy schemas break conventions; override explicitly: `self.table_name = "tbl_products"`, `self.primary_key = "product_code"`.

## 3. Migrations

A migration is a Ruby class that changes the schema. Its file name starts with a UTC timestamp (`20260110101500_create_products.rb`), which is the **version** recorded in the `schema_migrations` table once it has run.

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]   # [7.1]: behave like 7.1 migrations forever
  def change
    create_table :products do |t|
      t.string  :name, null: false
      t.text    :description
      t.decimal :price, precision: 10, scale: 2, null: false, default: 0  # money: never float
      t.integer :stock, null: false, default: 0
      t.boolean :active, null: false, default: true
      t.references :store, null: false, foreign_key: true  # store_id bigint + index + FK
      t.timestamps                                          # created_at, updated_at (NOT NULL)
    end
    add_index :products, [:store_id, :name], unique: true
  end
end

class AddSkuToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :sku, :string
    add_index :products, :sku
    add_reference :products, :category, foreign_key: true  # nullable FK column + index
    change_column_default :products, :description, from: nil, to: ""  # from/to makes it reversible
    change_column_null :products, :description, false, ""  # 4th arg backfills NULLs before NOT NULL
  end
end
```

Common column types: `string`, `text`, `integer`, `bigint`, `decimal`, `float`, `boolean`, `date`, `datetime`, `time`, `json` / `jsonb` (Postgres), `uuid`, `binary`, `references`. The `[7.1]` suffix freezes migration behavior, so old migrations keep working after an upgrade.

**`change` vs `up`/`down`:** in `change`, Rails records each operation and knows its inverse (`create_table` -> `drop_table`, `add_column` -> `remove_column`). Some operations cannot be inverted: `change_column` (the old type is unknown), `execute` with raw SQL, and `remove_column` without a type. For those, write both directions or use `reversible`:

```ruby
class ChangeStockToBigint < ActiveRecord::Migration[7.1]
  def up
    change_column :products, :stock, :bigint
  end

  def down
    change_column :products, :stock, :integer
  end
end

class AddSearchIndexSql < ActiveRecord::Migration[7.1]
  def change
    reversible do |dir|
      dir.up   { execute "CREATE INDEX products_name_lower_idx ON products (lower(name))" }
      dir.down { execute "DROP INDEX products_name_lower_idx" }
    end
  end
end
```

```bash
bin/rails g migration AddSkuToProducts sku:string:index
bin/rails db:migrate                  # run pending migrations, regenerate schema.rb
bin/rails db:rollback STEP=2          # undo the last two
bin/rails db:migrate:status           # up/down per version
bin/rails db:schema:load              # build an empty DB straight from schema.rb (CI, new laptops)
bin/rails db:prepare                  # create + load schema, or migrate, as needed
bin/rails db:seed
```

**schema.rb vs structure.sql:** `db/schema.rb` is a database-agnostic Ruby dump regenerated after each migration. It cannot express everything (triggers, views, some functions and custom types), so apps that rely on those switch to `config.active_record.schema_format = :sql`, which writes `db/structure.sql` with the database's own dump tool. Commit whichever you use: it, not hundreds of old migrations, is how new environments get the schema. Replaying ancient migrations is slow and often broken because they reference models that changed long ago.

**Why data changes inside migrations are risky:**
- They often use model classes, and the migration breaks months later when the model gains a validation, renames a column or disappears.
- A large `UPDATE` inside the migration's transaction can lock a hot table for minutes and block the deploy.
- Rollbacks rarely restore data, and a partial failure leaves a mess (MySQL cannot roll back DDL at all).

Safer: ship the schema change, then run a separate idempotent backfill task or job in batches (`in_batches.update_all`). If a migration really must touch data, define a minimal model class inside the migration itself (`class MigrationProduct < ActiveRecord::Base` with `self.table_name = "products"`) so later model changes cannot break it, and use set-based SQL. Zero-downtime techniques (concurrent indexes, strong_migrations) are in [ActiveRecord advanced](06-activerecord-advanced.md).

## 4. CRUD and bang vs non-bang

```ruby
# Create
product = Product.new(name: "Mug")   # in memory only; product.new_record? => true
product.save                          # true or false (runs validations and callbacks)
product.save!                         # true, or raises ActiveRecord::RecordInvalid
Product.create(name: "")              # returns the (invalid, unsaved) object, NOT false
Product.create!(name: "Mug")          # raises if invalid

# Read
Product.find(42)                      # raises ActiveRecord::RecordNotFound (404 in controllers)
Product.find_by(sku: "MUG-1")         # first match or nil
Product.find_by!(sku: "MUG-1")        # raises RecordNotFound
Product.where(active: true)           # lazy Relation, never nil, maybe empty
Product.first                         # ORDER BY id ASC LIMIT 1
Product.take                          # LIMIT 1, no ORDER BY (any row)

# Update
product.update(price: 12)                   # assign + save: true or false
product.update!(price: 12)                  # raises on failure
product.update_attribute(:price, 12)        # skips validations, runs callbacks, saves
product.update_column(:price, 12)           # direct UPDATE: no validations, callbacks or updated_at
product.update_columns(price: 12, stock: 3)
Product.where(store_id: 3).update_all(active: false)  # one UPDATE for many rows

# Delete
product.destroy          # callbacks + dependent: associations; returns the frozen object or false
product.destroy!         # raises ActiveRecord::RecordNotDestroyed if a callback halts
product.delete           # one DELETE, no callbacks, no dependent cleanup
Product.where(store_id: 3).destroy_all   # loads each row and destroys it (slow, runs callbacks)
Product.where(store_id: 3).delete_all    # one DELETE statement
```

| Method | Validations | Callbacks | Sets `updated_at` | SQL |
|---|---|---|---|---|
| `save`, `update` | Yes | Yes | Yes | One statement per record |
| `update_attribute` | No | Yes | Yes | One |
| `update_column(s)` | No | No | No | One |
| `update_all` | No | No | No (set it yourself) | One for all rows |
| `destroy` / `destroy_all` | n/a | Yes | n/a | SELECT, then DELETE per record plus dependents |
| `delete` / `delete_all` | n/a | No | n/a | One |
| `insert_all` / `upsert_all` | No | No | n/a | One bulk statement |

**Which to use:** non-bang methods where failure is normal and you branch on it (`if @product.save` in a controller). Bang methods where failure is unexpected: jobs, rake tasks, seeds, service objects and transactions. A classic bug is calling `save` inside a `transaction` block: it returns false, nothing raises, and the transaction commits the other half of the work.

## 5. Validations

Validations run on `save`, `create`, `update` and `valid?`, and fill `errors` instead of hitting the database with bad data.

```ruby
class Product < ApplicationRecord
  validates :name, presence: true, length: { maximum: 120 }
  validates :sku, presence: true, uniqueness: { scope: :store_id, case_sensitive: false }
  validates :price, numericality: { greater_than_or_equal_to: 0 }
  validates :condition, inclusion: { in: %w[new used graded] }
  validates :barcode, format: { with: /\A\d{12,13}\z/ }, allow_blank: true  # \A \z, not ^ $
  validate :sale_price_below_price                                          # custom method

  private

  def sale_price_below_price
    return if sale_price.blank? || price.blank?
    errors.add(:sale_price, "must be lower than the price") if sale_price >= price
  end
end
```

A reusable rule becomes an `EachValidator`; the class name `UrlValidator` enables the `url:` key:

```ruby
# app/validators/url_validator.rb
class UrlValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    uri = URI.parse(value.to_s)
    valid = uri.is_a?(URI::HTTP) && uri.host.present?   # URI::HTTPS is a subclass of URI::HTTP
    record.errors.add(attribute, options[:message] || "is not a valid URL") unless valid
  rescue URI::InvalidURIError
    record.errors.add(attribute, options[:message] || "is not a valid URL")
  end
end

class Store < ApplicationRecord
  validates :website, url: true, allow_blank: true
end
```

Conditional validations and contexts:

```ruby
validates :tracking_number, presence: true, if: :shipped?
validates :card_last4, presence: true, unless: -> { payment_method == "cash" }
validates :password, length: { minimum: 8 }, on: :create
validate  :stock_available, on: :checkout   # runs only for order.valid?(:checkout) / save(context: :checkout)
```

The `errors` object:

```ruby
product = Product.new
product.valid?                       # => false (runs validations, clears old errors first)
product.errors.full_messages         # => ["Name can't be blank", "Sku can't be blank", ...]
product.errors[:name]                # => ["can't be blank"]
product.errors.added?(:name, :blank) # => true
product.errors.add(:base, "Store is closed for edits")  # error not tied to one attribute
```

Validations are skipped by `save(validate: false)`, `update_attribute`, `update_column(s)`, `update_all`, `insert_all` / `upsert_all`, `delete` and `touch`, which is one reason database constraints matter.

**The uniqueness race:** `uniqueness` runs a `SELECT` before the `INSERT`. Two concurrent requests (a double-clicked button, two Sidekiq threads importing the same SKU) can both see "no duplicate" and both insert. Only the database can enforce uniqueness atomically:

```ruby
add_index :products, [:store_id, :sku], unique: true    # migration

def create
  @product = current_store.products.new(product_params)
  @product.save ? redirect_to(@product) : render(:new, status: :unprocessable_entity)
rescue ActiveRecord::RecordNotUnique
  @product.errors.add(:sku, "has already been taken")
  render :new, status: :unprocessable_entity
end
```

Keep the validation for friendly error messages in the normal case and the index for correctness. More on constraints vs validations in [ActiveRecord advanced](06-activerecord-advanced.md).

## 6. Callbacks

Callbacks are hooks around a record's lifecycle. The full order when saving:

| Step | New record (`create`) | Existing record (`update`) |
|---|---|---|
| 1 | `before_validation` | `before_validation` |
| 2 | validations, then `after_validation` | validations, then `after_validation` |
| 3 | `before_save`, `around_save` (before yield) | same |
| 4 | `before_create`, `around_create` (before yield) | `before_update`, `around_update` |
| 5 | **INSERT** | **UPDATE** |
| 6 | `after_create` | `after_update` |
| 7 | `after_save` | `after_save` |
| 8 | transaction commits: `after_commit` / `after_create_commit` | `after_commit` / `after_update_commit` |
| or | transaction rolls back: `after_rollback` | `after_rollback` |

`destroy` runs `before_destroy`, `around_destroy`, the DELETE, `after_destroy`, then the commit or rollback callbacks. `after_initialize` and `after_find` fire when objects are built or loaded, so keep them cheap.

```ruby
class Order < ApplicationRecord
  before_validation :normalize_email
  before_save :compute_total
  before_destroy :ensure_not_shipped
  after_create_commit :enqueue_confirmation

  private

  def normalize_email
    self.email = email.to_s.strip.downcase
  end

  def compute_total
    self.total = order_items.sum { |item| item.unit_price * item.quantity }
  end

  def ensure_not_shipped
    throw :abort if shipped?          # halts: destroy returns false, destroy! raises
  end

  def enqueue_confirmation
    OrderConfirmationJob.perform_later(id)
  end
end
```

**after_save vs after_commit:** everything up to `after_save` runs **inside** the database transaction. If you enqueue a Sidekiq job in `after_save`, a fast worker can pick it up before the COMMIT and fail with `RecordNotFound` (or read old data), and if the transaction later rolls back, the job or email refers to a row that never existed. `after_commit` runs only after the data is durable, so use `after_create_commit`, `after_update_commit`, `after_destroy_commit` or `after_save_commit` for jobs, emails, webhooks, cache busting and broadcasts. One gotcha: registering the same method with both `after_create_commit` and `after_update_commit` keeps only the last one; use `after_commit :method, on: [:create, :update]` or `after_save_commit`.

**What skips callbacks:** `update_column(s)`, `update_all`, `delete`, `delete_all`, `insert_all`, `upsert_all`, and `touch` (which runs only `after_touch` and the commit/rollback callbacks).

**Callback pitfalls:**
- Hidden side effects: an email in a callback also fires from seeds, imports, tests and console fixes.
- Callbacks that call `save` on the same record cause loops or double writes; callbacks on other models make one `save` trigger a cascade.
- Order dependencies between callbacks are hard to see and easy to break.
- Bulk methods skip them, so "the callback always runs" is false.

Rule of thumb: callbacks are fine for logic about the record's own data (normalizing, defaults, derived columns). Cross-model workflows (checkout, payment, notifications) read better as an explicit service object called from the controller or job (see [architecture patterns](12-architecture-patterns.md)).

## 7. Dirty tracking

Active Record remembers original values until the next save:

```ruby
product = Product.find(1)            # price 12
product.price = 15
product.changed?                     # => true
product.price_changed?               # => true
product.price_was                    # => 12
product.changes                      # => { "price" => [12, 15] }
product.will_save_change_to_price?   # => true (use inside before_* callbacks)

product.save
product.saved_change_to_price?       # => true (use inside after_save / after_commit)
product.saved_changes                # => { "price" => [12, 15], "updated_at" => [..., ...] }
product.price_before_last_save       # => 12
```

Typical use: `after_update_commit :notify_price_drop, if: :saved_change_to_price?`, which syncs a listing to an external channel only when the price really changed.

## 8. Enums

```ruby
class Order < ApplicationRecord
  enum :status, { pending: 0, paid: 1, shipped: 2, cancelled: 3 }, default: :pending
  enum :payment_state, { pending: 0, captured: 1 }, prefix: :payment  # payment_pending?
end

order.status             # => "pending" (a string, not the integer)
order.paid?              # predicate
order.shipped!           # update!(status: :shipped)
Order.paid               # scope: WHERE status = 1
Order.not_cancelled      # negative scope
Order.statuses           # => { "pending" => 0, "paid" => 1, ... }
Order.where(status: [:paid, :shipped])
```

The `enum :status, {...}` form is the Rails 7 syntax; older apps write `enum status: {...}`. Without `prefix:` (or `suffix: true`), two enums sharing a value name like `pending` raise a conflict error at class load.

Pitfalls:
- **Never use the array form and reorder it.** `enum :status, [:pending, :paid]` stores positions 0 and 1; inserting a value in the middle silently changes the meaning of every stored row. Use an explicit hash (or a string column with string values, which reads better in SQL).
- Assigning an unknown value raises `ArgumentError`, which becomes a 500 if it comes from a form. Validate input first; recent versions also accept `validate: true` on the enum.
- Raw SQL and other services reading the table see integers, so document the mapping (or use strings or a Postgres enum type).

## 9. Attribute API, normalizes, timestamps and touch

The **attribute API** declares typed attributes, with or without a column:

```ruby
class Product < ApplicationRecord
  attribute :featured, :boolean, default: false   # virtual: "1" from a checkbox becomes true
  attribute :currency, :string, default: "USD"    # overrides the column default in Ruby
end
```

**`normalizes`** (Rails 7.1) cleans input on assignment and also in finders, replacing hand-written `before_validation` callbacks:

```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
  normalizes :phone, with: ->(phone) { phone.delete("^0-9") }   # keep digits only
end

User.create!(email: "  Ali@Example.COM ").email   # => "ali@example.com"
User.find_by(email: "ALI@example.com ")            # normalized before querying
User.normalize_value_for(:email, " A@B.com ")      # => "a@b.com"
```

Normalizers are not applied to `nil` unless you pass `apply_to_nil: true`.

**Timestamps:** `t.timestamps` adds `created_at` and `updated_at`, which Active Record sets on create and update (not on `update_column` or `update_all`). **`touch`** updates `updated_at` (plus any column you name, such as `product.touch(:last_synced_at)`) without validations. `belongs_to :store, touch: true` touches the parent whenever the child is saved or destroyed, which keeps cache keys fresh for nested fragment caching (see [caching](14-caching-performance-security.md)).

## Common pitfalls
- Storing money in `float` columns (rounding errors). Use `decimal` with precision and scale, or integer cents.
- Forgetting `null: false` and defaults, so `nil` sneaks in through bulk methods and console fixes that skip validations.
- Treating `Product.create(...)` as returning false on failure; it returns the unsaved object. Check `persisted?` or use `create!`.
- Calling non-bang `save` inside a transaction or job, so failures pass silently.
- Relying on `validates uniqueness` without a unique index.
- Using `^` and `$` in format regexes (they match per line, so `"ok\n<script>"` passes). Use `\A` and `\z`.
- Enqueuing jobs or sending emails in `after_save` instead of `after_commit`.
- Using `update_column` or `update_all` and then wondering why `updated_at`, callbacks or caches did not change.
- Editing an old migration that has already run in production instead of writing a new one.
- Reordering an array-style enum, which silently remaps stored data.

## How to talk about this in an interview
- Legacy Rails monoliths at Devntech are full of callbacks, enums and old migrations. A strong answer shows that you read `schema.rb` and the model's callbacks before changing anything, and that you know which methods skip them.
- CrystalCommerce: the PriceCharting import writes a lot of catalog data. Talk about where validations help (single edits from store owners) and where bulk writes skip them (imports), so constraints and normalization must hold either way.
- Colochousing: statuses for rental applications, bills and rent payments are natural enum examples; jobs that notify residents belong in `after_commit` or in explicit service code.
- When unsure about a detail, reason from the transaction boundary: "inside the transaction" vs "after commit" answers most callback questions.

> **Fill in:** one model you changed at Devntech: which validations, callbacks or enums it had, and what you learned from reading them.

> **Fill in:** a migration you wrote (new column, index, backfill) and how it was deployed.

## Interview questions

**Q1. What is the Active Record pattern, and how does it differ from Data Mapper?**
In Active Record, each object wraps a row and handles its own persistence (`product.save`), with attributes inferred from the table. Data Mapper keeps domain objects free of persistence and uses separate mapper or repository classes. Active Record is faster to build with; Data Mapper separates concerns more cleanly.
*Follow-up:* Main downside in big apps? Fat models with hidden queries and side effects.

**Q2. What is the difference between `change` and `up`/`down` in a migration?**
`change` lists operations Rails can invert automatically for rollback. Irreversible ones (`change_column`, raw `execute`, `remove_column` without a type) need explicit `up` and `down` methods or a `reversible` block; otherwise rollback raises `ActiveRecord::IrreversibleMigration`.
*Follow-up:* What does `Migration[7.1]` mean? The migration keeps 7.1 behavior even after you upgrade Rails.

**Q3. Why load `schema.rb` instead of running every migration for a new database?**
The schema file is the current, authoritative snapshot; replaying years of migrations is slow and often fails because old migrations reference models that changed. Switch to `structure.sql` when you use database features the Ruby DSL cannot express, such as triggers or views.
*Follow-up:* Should you edit a migration that already ran in production? No: write a new migration.

**Q4. Why are data changes in migrations risky, and what do you do instead?**
They can lock large tables inside the migration transaction, break later when model code changes, and are hard to roll back. Ship the schema change first, then backfill with an idempotent task or job in batches; if data must change in a migration, use a minimal model defined inside it and set-based SQL.
*Follow-up:* How do you backfill a million rows? `in_batches(of: 1000).update_all(...)` from a job or task, with throttling and progress logging.

**Q5. What is the difference between `save` and `save!`, or `create` and `create!`?**
Non-bang versions return false (`save`) or the unsaved object (`create`) when validations fail; bang versions raise `ActiveRecord::RecordInvalid`. Use non-bang when you branch on the result in controllers, bang in jobs, scripts and transactions where failure should stop everything.
*Follow-up:* What does a halted `before_save` do to `save!`? It raises `ActiveRecord::RecordNotSaved`.

**Q6. `find` vs `find_by` vs `where`?**
`find(id)` raises `RecordNotFound` when missing (a 404 in controllers); `find_by(attrs)` returns the first match or nil; `where` returns a lazy relation that may be empty and is never nil.
*Follow-up:* `first` vs `take`? `first` orders by primary key; `take` adds no ORDER BY and returns any row.

**Q7. What is the difference between `delete` and `destroy`, and between `update_column` and `update`?**
`destroy` loads the record, runs callbacks and handles `dependent:` associations; `delete` issues one SQL DELETE with no callbacks or cleanup. `update` assigns, validates, runs callbacks and sets `updated_at`; `update_column` writes straight to the database with none of that.
*Follow-up:* When would you choose `delete_all` or `update_all`? Bulk maintenance on many rows where callbacks are not needed and speed matters.

**Q8. How do you write a custom validation?**
For one model, `validate :method_name` with `errors.add(:attribute, "message")` inside. For reuse across models, a class inheriting `ActiveModel::EachValidator` with `validate_each(record, attribute, value)`, used as `validates :website, url: true`.
*Follow-up:* How do you validate only in one situation? `on: :create`, `if:`/`unless:`, or a custom context like `valid?(:checkout)`.

**Q9. Does `validates :sku, uniqueness: true` guarantee unique SKUs?**
No. It checks with a SELECT before inserting, so two concurrent requests can both pass and both insert. Add a unique index (`add_index ... unique: true`) and rescue `ActiveRecord::RecordNotUnique`; keep the validation for friendly messages.
*Follow-up:* Case-insensitive uniqueness? A unique index on `lower(sku)`, or a normalized column via `normalizes`.

**Q10. Walk through the callback order when you save a new record.**
`before_validation`, validations, `after_validation`, `before_save`, `around_save`, `before_create`, `around_create`, INSERT, `after_create`, `after_save`; then `after_commit` once the transaction commits, or `after_rollback` if it rolls back. Updates use the `update` variants.
*Follow-up:* How do you stop the chain? `throw :abort` in a before callback.

**Q11. What is the difference between `after_save` and `after_commit`, and where do you enqueue jobs?**
`after_save` runs inside the transaction before COMMIT; `after_commit` runs once the data is committed. Enqueue jobs in `after_commit` (`after_create_commit`), or a worker may run before the row is visible, or the transaction may roll back after the job was queued.
*Follow-up:* Gotcha with `after_create_commit` and `after_update_commit` on the same method? Only the last registration wins; use `after_commit on: [:create, :update]`.

**Q12. What are the downsides of callbacks?**
Hidden side effects that fire from every code path (seeds, imports, console), ordering surprises, cascades across models, slower saves, and inconsistency because bulk methods skip them. Keep callbacks for a record's own data and move workflows into service objects.
*Follow-up:* Which methods skip callbacks? `update_column(s)`, `update_all`, `delete(_all)`, `insert_all`, `upsert_all`, and `touch` (except `after_touch` and commit callbacks).

**Q13. How do you know inside a callback whether an attribute changed?**
Dirty tracking: before saving use `price_changed?`, `will_save_change_to_price?` and `price_was`; after saving use `saved_change_to_price?`, `saved_changes` and `price_before_last_save`.
*Follow-up:* Why does `price_changed?` return false in `after_commit`? The save already happened; use `saved_change_to_price?`.

**Q14. How do enums work, and what are their pitfalls?**
`enum :status, { pending: 0, paid: 1 }` maps names to stored values and adds predicates (`paid?`), bang setters (`paid!`) and scopes (`Order.paid`, `Order.not_paid`). Pitfalls: reordering an array-style enum remaps existing data, unknown values raise `ArgumentError`, and name clashes need `prefix:` or `suffix:`.
*Follow-up:* Integer or string storage? Strings are readable in SQL and safe to reorder; integers are compact.

**Q15. What does `normalizes` do, and what did people use before Rails 7.1?**
It declares a transformation applied on assignment and to finder arguments, so `find_by(email: " A@B.com")` matches the stored lowercase value. Before 7.1 you used a `before_validation` callback, which did not affect queries.
*Follow-up:* Is it applied to nil? Not unless you pass `apply_to_nil: true`.

**Q16. What is `touch`, and why use `belongs_to ..., touch: true`?**
`touch` updates `updated_at` (and optionally another timestamp column) without validations. `touch: true` on `belongs_to` touches the parent whenever the child changes, which invalidates cache keys built from the parent's `updated_at`.
*Follow-up:* Cost? An extra UPDATE per save, and possible lock contention on a busy parent row.

## Further reading
- [Active Record Basics](https://guides.rubyonrails.org/active_record_basics.html) and [Active Record Migrations](https://guides.rubyonrails.org/active_record_migrations.html)
- [Active Record Validations](https://guides.rubyonrails.org/active_record_validations.html) and [Active Record Callbacks](https://guides.rubyonrails.org/active_record_callbacks.html)
