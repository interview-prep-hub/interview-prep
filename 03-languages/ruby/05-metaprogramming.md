# Ruby Metaprogramming

> **Tier:** T1 (know cold) · **Related:** [OOP, modules and mixins](03-oop-modules-mixins.md), [Blocks, procs and lambdas](04-blocks-procs-lambdas.md), [Advanced internals](06-advanced-internals.md), [ActiveRecord associations](../../05-ruby-on-rails/04-activerecord-associations.md), [Legacy codebases and upgrades](../../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [Web security](../../04-web-fundamentals/07-web-security-owasp.md)
> **On your resume:** Reading and debugging the macros, concerns and patches inside the legacy Rails monoliths you work on at Devntech; Rails itself (`has_many`, `validates`, `enum`, `before_action`) is metaprogramming you use every day.

## Quick revision
- Metaprogramming is code that defines or changes code at runtime: generating methods, reopening classes, intercepting unknown calls.
- Classes are open, so you can reopen `String` and add methods (monkey patching). The change is global; refinements (`refine` plus `using`) limit it to a file.
- `send` calls any method, private ones included; `public_send` respects visibility. Never `send` user input without an allow-list.
- `define_method(name) { |args| ... }` creates real methods whose bodies are closures.
- `method_missing` catches unknown calls ("ghost methods"). Call `super` for names you do not handle and define `respond_to_missing?` alongside it.
- `instance_variable_get`/`instance_variable_set` reach into objects from outside; keep them for tooling and tests.
- `class_eval`: `self` is the class and `def` creates instance methods. `instance_eval`: `self` is the receiver and `def` creates singleton methods. `instance_exec` is `instance_eval` with arguments.
- Hooks: `included`, `extended`, `prepended`, `inherited`, `method_added`. Call `super` inside them.
- `Object.const_get("Billing::Invoice")` and Rails `constantize`/`safe_constantize` resolve names at runtime; allow-list before using input.
- Rails macros (`has_many`, `validates`, `before_action`, `enum`) are ordinary class methods that run while the class body executes, defining methods and registering callbacks.
- Costs: harder to read and grep, invisible to tooling, slower `method_missing` dispatch, global side effects. Write the plain version first.

## 1. Why Ruby makes this easy

A class body is executable code, classes are objects you can modify at runtime, and calling a method means sending a message that can be intercepted. So a "declaration" such as `has_many :orders` is simply a method call that runs while the class is being defined. Rails reads those calls (plus your database schema) and generates methods for you.

The introspection tools you will use constantly:

```ruby
Listing.instance_methods(false)        # methods defined directly in Listing
listing.public_methods.grep(/price/)   # search methods by name
listing.respond_to?(:price_cents)      # can it handle this message?
Listing.method_defined?(:archive!)     # does an instance method exist?
listing.instance_variables             # names of its instance variables
Listing.ancestors                      # the lookup chain
listing.method(:save).owner            # which module or class wins
listing.method(:save).source_location  # [file, line] of the definition
```

## 2. Open classes, monkey patching and refinements

```ruby
class String
  def shout = upcase + "!"
end
"hi".shout   # => "HI!" for every string in the process
```

ActiveSupport's core extensions (`3.days.ago`, `"".blank?`, `"user".pluralize`) are sanctioned monkey patches. Problems with ad-hoc ones:

- **Global reach:** every gem and file sees the change.
- **Collisions:** two libraries define `String#to_slug` differently and the last one loaded silently wins.
- **Upgrades:** Ruby or Rails later adds a method with the same name (`Hash#except` and `Enumerable#sum` existed in ActiveSupport before Ruby added them), and behavior shifts underneath you.
- **Discoverability:** a method no longer behaves like its documentation says, and nobody knows why.

If you must patch, keep patches in one obvious place (for example `config/initializers/` or `lib/core_ext/`), wrap with `prepend` and `super` instead of redefining, and pin the behavior with tests so upgrades fail loudly:

```ruby
module PriceApiLogging                # PriceApi::Client stands for any gem's client class
  def fetch(sku)
    Rails.logger.info("price fetch #{sku}")
    super                             # the gem's original implementation
  end
end
PriceApi::Client.prepend(PriceApiLogging)
```

**Refinements** scope a patch lexically:

```ruby
module SkuFormatting
  refine String do
    def to_sku = upcase.gsub(/[^A-Z0-9]+/, "-")
  end
end

using SkuFormatting               # active from here to the end of this file
"zelda botw".to_sku               # => "ZELDA-BOTW"
```

Files that do not call `using` never see the method, which is the point. The trade-offs: reflection such as `respond_to?` does not see refined methods, some dynamic dispatch paths ignore them, and few Rails codebases use them. They are still the best answer to "how would you patch a core class safely?"

## 3. send vs public_send

```ruby
class Order
  def total = 100

  private

  def recalculate! = :recalculated
end

order = Order.new
order.send(:total)                # => 100
order.send(:recalculate!)         # => :recalculated (send ignores visibility)
order.public_send(:recalculate!)  # NoMethodError (private method called)
order.public_send("total")        # strings work too
```

Dynamic dispatch is useful for generic exporters, sorting by a configurable column or calling a method named in configuration. It is dangerous with input: `record.send(params[:field])` lets a caller invoke `destroy`, `delete` or worse. Always allow-list:

```ruby
SORTABLE = %w[price_cents created_at title].freeze

def sort_value(record, field)
  raise ArgumentError, "unsupported field: #{field}" unless SORTABLE.include?(field)
  record.public_send(field)
end
```

`__send__` exists for objects that define their own `send` method (sockets, mailers).

## 4. define_method

```ruby
class Order
  STATUSES = %i[pending paid shipped cancelled].freeze
  attr_reader :status

  def initialize(status)
    @status = status
  end

  STATUSES.each do |name|
    define_method("#{name}?") { status == name }        # pending?, paid?, ...
    define_method("mark_#{name}!") { @status = name }   # mark_paid!, ...
  end
end

order = Order.new(:pending)
order.paid?       # => false
order.mark_paid!
order.paid?       # => true
```

This is roughly what Rails `enum` generates (`paid?`, `paid!` and a `paid` scope). Newer Rails versions write it as `enum :status, { pending: 0, paid: 1 }`; older code uses `enum status: { ... }`.

Why `define_method` instead of `def`: the method name is computed at runtime, and the block is a closure, so it can use `name` from the loop. A `def` is a scope gate and cannot see that local. One gotcha: methods created by `define_method` cannot use bare (implicit-argument) `super`; pass the arguments explicitly (`super(arg)`). Use `define_singleton_method` to add a method to one object.

## 5. method_missing and respond_to_missing?

```ruby
class Settings
  def initialize(values)
    @values = values
  end

  def method_missing(name, *args)
    key = name.to_s
    if key.end_with?("?") && @values.key?(key.chomp("?").to_sym)
      !!@values[key.chomp("?").to_sym]            # settings.sync_enabled?
    elsif @values.key?(name) && args.empty?
      @values[name]                               # settings.timeout
    else
      super                                       # unknown name: normal NoMethodError
    end
  end

  def respond_to_missing?(name, include_private = false)
    @values.key?(name.to_s.chomp("?").to_sym) || super
  end
end

settings = Settings.new(timeout: 5, sync_enabled: false)
settings.timeout                # => 5
settings.sync_enabled?          # => false
settings.respond_to?(:timeout)  # => true, thanks to respond_to_missing?
settings.method(:timeout).call  # => 5, also thanks to respond_to_missing?
settings.colour                 # NoMethodError, because method_missing called super
```

**Why both methods:** `method_missing` makes the calls work; `respond_to_missing?` keeps introspection truthful, so `respond_to?`, `method(:name)`, duck-typing checks and delegation libraries all agree. Forgetting `super` in `method_missing` turns every typo into a silent `nil`.

**Costs:** Ruby walks the entire ancestor chain, fails, then walks it again looking for `method_missing`, so ghost methods are slower than real ones. They are also invisible to `instance_methods`, grep and IDEs. When the names are known up front, generate real methods once with `define_method` instead.

Where you meet it in the wild: ActiveRecord's dynamic finders (`find_by_email`), `OpenStruct`, proxy and delegator objects (`SimpleDelegator`), test doubles and older API client gems. Rails generates real attribute methods from the schema rather than resolving each attribute call through `method_missing`.

## 6. instance_variable_get and instance_variable_set

```ruby
class Listing
  attr_reader :sku

  def initialize(sku, price_cents)
    @sku = sku
    @price_cents = price_cents
  end
end

listing = Listing.new("A1", 100)
listing.instance_variables                 # => [:@sku, :@price_cents]
listing.instance_variable_get(:@price_cents) # => 100
listing.instance_variable_set(:@sku, "B2")
listing.sku                                # => "B2"
listing.instance_variable_defined?(:@foo)  # => false
```

These bypass encapsulation and any invariants your setters enforce, so in application code add a proper method instead. They are legitimate in tooling: serializers, debuggers, test setup and frameworks. Rails itself uses this idea to copy a controller's instance variables (`@order`) into the view.

## 7. class_eval, instance_eval and instance_exec

```ruby
class Product; end

Product.class_eval do              # self is Product; def creates instance methods
  def label = "product"
end
Product.new.label                  # => "product"

Product.instance_eval do           # self is the Product object; def creates class methods
  def table = "products"
end
Product.table                      # => "products"

multiplier = ->(factor) { @base * factor }
holder = Object.new
holder.instance_variable_set(:@base, 10)
holder.instance_exec(3, &multiplier)   # => 30: self is holder, and 3 is passed in
```

| | `self` inside the block | `def` inside defines | Typical use |
|---|---|---|---|
| `Klass.class_eval { }` | the class | instance methods | adding methods to a class held in a variable, from a hook |
| `obj.instance_eval { }` | `obj` | singleton methods of `obj` (class methods when `obj` is a class) | DSLs whose blocks call methods without a receiver |
| `obj.instance_exec(*args) { }` | `obj` | same as `instance_eval` | DSL blocks that also need arguments |

Rails evaluates zero-argument condition lambdas such as `validates :sku, presence: true, if: -> { published? }` against the record in this way.

A tiny DSL in the style of `routes.draw`:

```ruby
class RouteSet
  attr_reader :routes

  def initialize
    @routes = []
  end

  def get(path, to:) = routes << [:get, path, to]
  def post(path, to:) = routes << [:post, path, to]

  def self.draw(&block)
    set = new
    set.instance_eval(&block)      # inside the block, self is the RouteSet
    set
  end
end

routes = RouteSet.draw do
  get "/listings", to: "listings#index"
  post "/listings", to: "listings#create"
end
routes.routes.size   # => 2
```

The trade-off: inside an `instance_eval` block, `self` changes, so the caller's instance variables and helper methods are not reachable. That surprises people, which is why some APIs yield an object instead (`configure do |config| ... end`). Rails also generates methods with string `class_eval` plus `__FILE__` and `__LINE__` for speed and accurate backtraces; never build such strings from user input.

## 8. Hooks

```ruby
module Trackable
  def self.included(base)            # runs when a class includes Trackable
    super
    base.extend(ClassMethods)
  end

  module ClassMethods
    def tracked_fields
      @tracked_fields ||= []
    end

    def track(*fields) = tracked_fields.concat(fields)
  end
end

class Listing
  include Trackable
  track :price_cents, :quantity
end
Listing.tracked_fields   # => [:price_cents, :quantity]
```

A subclass registry with `inherited`:

```ruby
class SalesChannel
  def self.registry
    @registry ||= []
  end

  def self.inherited(subclass)
    super
    SalesChannel.registry << subclass   # explicit receiver: @registry lives on SalesChannel
  end
end

class StorefrontChannel < SalesChannel; end
class MarketplaceChannel < SalesChannel; end

SalesChannel.registry   # => [StorefrontChannel, MarketplaceChannel]
```

Rails gotcha: in development, Zeitwerk loads classes lazily, so a subclass is not registered until something references it. Registries built with `inherited` (and STI `descendants`) look incomplete in development while production, which eager loads, looks fine. Use an explicit list or eager load that namespace.

| Hook | Defined as | Fires when |
|---|---|---|
| `included(base)` | module method | the module is included |
| `extended(base)` | module method | the module extends an object |
| `prepended(base)` | module method | the module is prepended |
| `inherited(subclass)` | class method | a subclass is defined |
| `method_added(name)` | class method | an instance method is defined in the class |
| `const_missing(name)` | class method | an unknown constant is referenced (the classic Rails autoloader relied on it; Zeitwerk uses `Module#autoload`) |

Always call `super` in hooks so other libraries hooking the same event still work.

## 9. Constants at runtime: const_get and constantize

```ruby
Object.const_get("Billing::Invoice")     # => Billing::Invoice
Billing.const_get(:Invoice)              # => Billing::Invoice
Object.const_defined?("Billing::Refund") # => false
"billing/invoice".camelize               # => "Billing::Invoice"  (ActiveSupport)
"Billing::Invoice".constantize           # => Billing::Invoice, NameError if missing
"Nope".safe_constantize                  # => nil
```

Choosing a strategy class by name is common, for example picking a sales channel integration. Prefer an explicit map to constantizing input:

```ruby
CHANNELS = {
  "storefront"  => StorefrontChannel,
  "marketplace" => MarketplaceChannel
}.freeze

def channel_for(name)
  CHANNELS.fetch(name) { raise ArgumentError, "unknown channel: #{name}" }.new
end
```

`"#{params[:type].camelize}Channel".constantize` lets a request choose which class to instantiate, a known source of vulnerabilities, and it hides the list of valid classes from grep.

## 10. How Rails macros work

`has_many :orders` runs when Ruby evaluates the class body, with `self` set to the model class. It is a class method that:

1. builds a reflection object describing the association (`Store.reflect_on_association(:orders)`),
2. defines methods such as `orders`, `orders=` and `order_ids` in a generated module included into the model, so you can override them and call `super`,
3. registers callbacks for options such as `dependent: :destroy`.

`validates :email, presence: true` adds a validator to a class-level list that `valid?` runs. `before_action :authenticate_user!` appends to a callback chain stored at class level, which subclasses inherit and can extend or skip (`skip_before_action`).

### A mini `has_many` from scratch

```ruby
module MiniRecord
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def all_records
      @all_records ||= []                 # per-class in-memory "table"
    end

    def create(**attrs)
      new(**attrs).tap { |record| all_records << record }
    end

    def where(**conditions)
      all_records.select do |record|
        conditions.all? { |key, value| record.public_send(key) == value }
      end
    end

    # The macro: runs once, while the class body is evaluated
    def has_many(name, class_name:, foreign_key:)
      define_method(name) do
        Object.const_get(class_name).where(foreign_key => id)
      end
    end
  end
end

class Store
  include MiniRecord
  attr_reader :id, :name

  def initialize(id:, name:)
    @id = id
    @name = name
  end

  has_many :products, class_name: "Product", foreign_key: :store_id
end

class Product
  include MiniRecord
  attr_reader :store_id, :title

  def initialize(store_id:, title:)
    @store_id = store_id
    @title = title
  end
end

store = Store.create(id: 1, name: "Game Haven")
Product.create(store_id: 1, title: "Zelda")
Product.create(store_id: 2, title: "Mario")
store.products.map(&:title)   # => ["Zelda"]
```

The class name is resolved when `products` is called, not when `has_many` runs, which is why `Product` can be defined after `Store`. Real Rails does the same lazily through reflections.

### An attribute DSL from scratch

```ruby
module Attributes
  def self.included(base)
    base.extend(ClassMethods)
  end

  def self.cast(raw, type)
    case type
    when :integer then Integer(raw)
    when :boolean then [true, "true", "1"].include?(raw)
    else raw.to_s
    end
  end

  module ClassMethods
    def attribute(name, type = :string, default: nil)
      define_method(name) do
        value = instance_variable_get("@#{name}")
        value.nil? ? default : value
      end

      define_method("#{name}=") do |raw|
        instance_variable_set("@#{name}", Attributes.cast(raw, type))
      end
    end
  end

  def initialize(**attrs)
    attrs.each { |key, value| public_send("#{key}=", value) }
  end
end

class ListingForm
  include Attributes
  attribute :title
  attribute :quantity, :integer, default: 0
  attribute :published, :boolean, default: false
end

form = ListingForm.new(title: "Zelda", quantity: "3")
form.quantity    # => 3 (cast from "3")
form.published   # => false (default)
form.published = "1"
form.published   # => true
```

This is a toy version of `ActiveModel::Attributes` (`attribute :quantity, :integer, default: 0`) and the attributes API on Rails models.

## 11. Costs and guidelines

| Cost | Why it hurts | Mitigation |
|---|---|---|
| Readability | readers cannot find `def paid?` anywhere | keep macros small and obvious from the call site; comment what they generate |
| Grep-ability | `define_method("#{name}?")` defeats text search | write explicit methods when there are only a few |
| Tooling | jump-to-definition, linters and type checkers miss generated methods | signatures (RBS, Sorbet), `source_location` |
| Performance | `method_missing` walks the ancestor chain twice; defining methods at runtime invalidates method caches | define methods once at load time; turn hot ghost methods into real ones |
| Security | `send`, `constantize` and string `eval` on input create arbitrary code paths | allow-lists, `public_send`, never eval input |
| Global effects | monkey patches change behavior everywhere | refinements, `prepend` with `super`, tests that pin behavior |

Rule of thumb: write the boring version first and metaprogram only when the duplication is real and the generated API reads naturally at the call site, like `has_many`.

## Common pitfalls
- `method_missing` without `super` (typos return `nil`) or without `respond_to_missing?` (introspection lies).
- `send` or `constantize` on request parameters.
- Bare `super` inside a `define_method` body, which raises; pass arguments explicitly.
- Forgetting that `self` changes inside `instance_eval`, then wondering why `@ivar` from the caller is `nil`.
- Hooks (`included`, `inherited`) that skip `super`.
- Subclass registries that are incomplete in development because of lazy loading.
- Core-class monkey patches scattered across random files.
- Defining methods dynamically per request or per object in hot paths, which hurts method caching and memory.

## How to talk about this in an interview
- Legacy monoliths accumulate macros, concerns and patches over the years. Your practical skill is navigating them: `method(:x).owner`, `source_location`, `ancestors`, and grepping for `define_method`, `class_eval` and `method_missing`.
- Show judgment as much as knowledge: explain how `has_many` works, then say you prefer explicit code unless metaprogramming removes real repetition.
- > **Fill in:** a time you traced surprising behavior to a monkey patch, gem override, concern or generated method in the CrystalCommerce or Colochousing codebase, and how you found it.
- > **Fill in:** any macro-style helper or dynamic method generation you wrote or reviewed at Devntech, and why it was (or was not) worth the indirection.

## Interview questions

**Q1. What is metaprogramming? Give a Rails example.**
Code that writes or modifies code at runtime: defining methods, reopening classes or handling unknown calls. `has_many :orders` is a class method that runs while the model body is evaluated and defines `orders`, `orders=` and friends; `enum` generates predicate methods, bang methods and scopes.
*Follow-up:* Why is Ruby good at it? Class bodies are executable, classes are open objects, and method calls are messages.

**Q2. What is monkey patching and what are its risks?**
Reopening an existing class, often a core or gem class, to add or change methods. The change is global, can collide with other libraries or future Ruby and Rails versions, and makes behavior differ from documentation. Mitigate with refinements, `prepend` plus `super` instead of redefinition, a single patches folder and tests.
*Follow-up:* Which monkey patches does every Rails app rely on? ActiveSupport core extensions such as `blank?`, `3.days.ago` and `pluralize`.

**Q3. What are refinements?**
A way to scope a monkey patch: you `refine` a class inside a module and activate it with `using`, which applies only from that point to the end of the file (or class body). Code elsewhere never sees the change. Reflection such as `respond_to?` ignores refinements.
*Follow-up:* Why are they rare in Rails apps? Most patches are needed app-wide, and the lexical scoping surprises people.

**Q4. What is the difference between `send` and `public_send`?**
`send` invokes any method, including private and protected ones; `public_send` raises `NoMethodError` for non-public methods. Prefer `public_send` for dynamic dispatch, and never pass request parameters to either without an allow-list.
*Follow-up:* What is `__send__` for? Objects that define their own `send` method.

**Q5. When do you use `define_method` instead of `def`?**
When method names are computed at runtime (one predicate per status) or when the body needs a local variable from the surrounding scope, since `define_method`'s block is a closure and `def` is a scope gate. The result is a real method, visible to `respond_to?` and `instance_methods`.
*Follow-up:* Any limitation? Bare `super` is not allowed inside a `define_method` body; pass arguments explicitly.

**Q6. Explain `method_missing` and why `respond_to_missing?` matters.**
When lookup fails, Ruby calls `method_missing(name, *args, &block)`, which lets an object handle dynamic names such as configuration keys. `respond_to_missing?` tells `respond_to?` and `method(:name)` about those names so introspection, duck typing and delegation stay consistent. Always call `super` for names you do not handle.
*Follow-up:* What is the performance cost? Each call walks the ancestor chain twice before your code runs.

**Q7. What happens if `method_missing` does not call `super`?**
Unknown method names no longer raise `NoMethodError`: they return whatever your method returns, usually `nil`. Typos then fail silently far from the cause.
*Follow-up:* What should the fallback look like? `else super` at the end of the branching.

**Q8. What is the difference between `class_eval` and `instance_eval`?**
`class_eval` evaluates the block with `self` as the class, and `def` inside it creates instance methods. `instance_eval` evaluates with `self` as the receiver, and `def` creates singleton methods (class methods when the receiver is a class). DSLs use `instance_eval` so blocks can call the builder's methods without a receiver.
*Follow-up:* What do you lose inside `instance_eval`? Access to the caller's instance variables and private helpers, because `self` changed.

**Q9. When would you use `instance_exec`?**
When you need `instance_eval` behavior and also want to pass arguments to the block, for example evaluating a stored lambda against a record. Rails uses this style to evaluate condition lambdas such as `if: -> { published? }` in the record's context.
*Follow-up:* What does `instance_exec(3, &lambda)` require? The lambda's arity must match the arguments passed.

**Q10. How does `has_many` work under the hood?**
It is a class method executed during class definition. It records a reflection, defines association methods (`orders`, `orders=`, `order_ids`) in a generated module included into the model so they can be overridden with `super`, and registers callbacks for options like `dependent: :destroy`. Target classes are resolved lazily when the association is used.
*Follow-up:* Why define the methods in a module rather than directly on the class? So your own method with the same name can call `super`.

**Q11. Which hooks does Ruby provide, and how have you seen them used?**
`included`, `extended` and `prepended` on modules (adding class methods, as `ActiveSupport::Concern` does), `inherited` on classes (subclass registries, framework bookkeeping), `method_added` (validation of definitions), and `const_missing` (the classic Rails autoloader). Call `super` in each.
*Follow-up:* Why can an `inherited` registry be incomplete in Rails development? Subclasses are autoloaded lazily, so unreferenced ones have not been loaded.

**Q12. How does Rails `enum` generate its methods?**
It loops over the declared values and uses `define_method` for predicates (`paid?`) and bang setters (`paid!`), defines scopes (`Order.paid`), and stores the mapping for casting between names and integers. It is the same technique as the `define_method` loop in section 4.
*Follow-up:* Why store enums as integers? Compact, indexable columns; the trade-off is that reordering values breaks existing data.

**Q13. Why is `params[:type].constantize` dangerous?**
It lets the client choose which class your code loads and instantiates, which can reach classes with side effects in their constructors or class methods. It also hides the valid options. Use an explicit Hash from allowed names to classes, or `safe_constantize` plus an allow-list.
*Follow-up:* What is the difference between `constantize` and `safe_constantize`? The first raises `NameError` for unknown names, the second returns `nil`.

**Q14. How do you find where a method comes from in a big codebase?**
`obj.method(:name).owner` gives the class or module that defines the version that runs, `source_location` gives file and line, and `Klass.ancestors` shows the lookup order. `super_method` walks to the next definition up the chain.
*Follow-up:* What if `source_location` is `nil`? The method is implemented in C (core Ruby or a native extension).

**Q15. What are the downsides of metaprogramming, and when do you avoid it?**
Generated code is hard to read, grep and navigate, tooling misses it, `method_missing` is slow, and global patches create spooky action at a distance. Avoid it when a few explicit methods would do, when only one call site benefits, or in performance-critical paths. Use it when it removes real repetition behind a clear, declarative API.
*Follow-up:* How do you make generated methods discoverable? Comment what the macro generates, add tests, and consider type signatures.

## Further reading
- [Module#define_method and friends (Ruby docs)](https://docs.ruby-lang.org/en/master/Module.html)
- [Refinements (Ruby docs)](https://docs.ruby-lang.org/en/master/syntax/refinements_rdoc.html)
- [Autoloading and reloading constants (Rails guides)](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
