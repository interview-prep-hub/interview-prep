# Ruby OOP, Modules and Mixins

> **Tier:** T1 (know cold) · **Related:** [OOP principles](../../01-cs-fundamentals/02-oop.md), [SOLID](../../01-cs-fundamentals/03-solid-and-design-principles.md), [Metaprogramming](05-metaprogramming.md), [Advanced internals](06-advanced-internals.md), [Rails architecture patterns](../../05-ruby-on-rails/12-architecture-patterns.md), [C++ OOP](../c-cpp/03-cpp-oop.md)
> **On your resume:** Models, concerns, service objects and class hierarchies in the legacy Rails monoliths you maintain at Devntech; OOP was also one of the labs you assisted as a PUCIT TA.

## Quick revision
- `Klass.new` allocates an object and calls `initialize` (always private). `attr_reader`, `attr_writer` and `attr_accessor` generate getters and setters.
- Inside a method, call a setter as `self.qty = 1`; a bare `qty = 1` creates a local variable.
- Instance methods live in the class. Class methods are singleton methods of the class object (`def self.x` or `class << self`).
- `@@var` is one slot shared by a class and all its subclasses. A class-level `@var` (class instance variable) belongs to one class object only. Rails `class_attribute` gives inheritable, overridable per-class values.
- `private`: no explicit receiver except `self.` (Ruby 2.7+). `protected`: callable on other instances of the same class family (comparisons). `send` bypasses visibility; `public_send` respects it.
- Single inheritance only. Bare `super` forwards the same arguments and block; `super()` passes no arguments; `super(a)` passes exactly `a`.
- Modules are namespaces (`Billing::Invoice`) and mixins. They cannot be instantiated or subclassed.
- `include` puts a module after the class in `ancestors` (instance methods); `extend` adds module methods to one object (class methods when the object is a class); `prepend` puts the module before the class so it wraps the class's methods.
- Lookup: singleton class, prepended modules, the class, included modules (last included first), then up the superclass chain to `Object`, `Kernel`, `BasicObject`, then `method_missing`, then `NoMethodError`.
- `Comparable` needs `<=>`; `Enumerable` needs `each`.
- Duck typing: depend on what an object responds to, not on its class.
- `to_s` is for humans (`puts`, interpolation); `inspect` is for developers (`p`, console, logs).
- Value objects override `==`, `eql?` and `hash` together. `Struct` (mutable) and `Data` (Ruby 3.2, immutable) generate them.
- `ActiveSupport::Concern`: `included do ... end` runs inside the including class; `class_methods do ... end` defines class methods.

## 1. Classes, objects and attributes

```ruby
class Listing
  attr_reader :sku, :price_cents       # getters only
  attr_accessor :quantity              # getter and setter

  def initialize(sku:, price_cents:, quantity: 0)
    @sku = sku
    @price_cents = price_cents
    @quantity = quantity
  end

  def in_stock? = quantity.positive?

  def restock(amount)
    self.quantity += amount            # self. is required to call the setter
  end

  def reset
    quantity = 0                       # BUG: creates a local variable; the attribute is unchanged
    self.quantity = 0                  # calls the quantity= writer
  end
end

listing = Listing.new(sku: "ZELDA-01", price_cents: 4_999)
listing.restock(3)
listing.quantity          # => 3
listing.in_stock?         # => true
listing.price_cents = 1   # NoMethodError: there is no writer
```

`new` allocates the object and calls `initialize`, which is private automatically. `attr_accessor :quantity` generates a `quantity` reader and a `quantity=` writer; instance variables are created on first assignment and are reachable from outside only through methods. **The setter trap** in `reset` happens because Ruby decides at parse time that `name = value` is a local variable assignment.

| Where the code runs | `self` is |
|---|---|
| Inside an instance method | the instance |
| Inside the class body | the class itself |
| Inside `def self.x` or `class << self` | the class |
| Top level of a script | `main`, an instance of `Object` |

## 2. Instance methods vs class methods

```ruby
class Listing
  def self.from_api(payload)           # class method, used as a factory
    new(sku: payload.fetch("sku"), price_cents: payload.fetch("price_cents"))
  end

  class << self                        # opens the singleton class of Listing
    def table_label = "Listings"
    private
    def normalize(raw) = raw.strip.upcase   # a private class method
  end
end

Listing.from_api({ "sku" => "A1", "price_cents" => 100 })
Listing.table_label       # => "Listings"
Listing.normalize(" a1 ") # NoMethodError (private method called)
```

Class methods suit factories, finders and configuration (`Listing.from_api`, `User.find_by`, `has_many`). Instance methods suit behavior that depends on one object's state.

## 3. Class variables vs class instance variables

```ruby
class Channel
  @@registry = []            # class variable: ONE slot shared with every subclass
  @timeout = 5               # class instance variable: belongs to the Channel object only

  class << self
    attr_accessor :timeout   # accessor for the class instance variable
  end

  def self.register(name) = @@registry << name
  def self.registry = @@registry
end

class Marketplace < Channel
  register "marketplace"
end

Channel.registry          # => ["marketplace"]: shared through the hierarchy
Channel.timeout           # => 5
Marketplace.timeout       # => nil: class instance variables are not inherited
Marketplace.timeout = 30
Channel.timeout           # => 5: each class object keeps its own value
```

Subclasses can silently overwrite a class variable:

```ruby
class Base
  @@setting = "base"
  def self.setting = @@setting
end
class Child < Base
  @@setting = "child"     # reassigns the SAME variable Base uses
end
Base.setting # => "child"
```

| | `@@var` | `@var` at class level | `class_attribute` (Rails) |
|---|---|---|---|
| Owner | the whole hierarchy (one slot) | one class object | the class, overridable per subclass |
| Seen by subclasses | yes, the same variable | no, a subclass reads `nil` | yes, and a subclass can override without touching the parent |
| Reading before assignment | `NameError` | `nil` | its default |
| Typical use | rare; usually avoided | per-class config, registries, caches | framework-style configuration |

ActiveSupport's `cattr_accessor`/`mattr_accessor` wrap `@@` variables, so they share one value across the hierarchy. All class-level state is shared by every thread in a Puma or Sidekiq process, so treat it as read-only after boot (see [Advanced internals](06-advanced-internals.md)).

## 4. Visibility: public, private and protected

```ruby
class Account
  attr_reader :id

  def initialize(id, balance_cents)
    @id = id
    @balance_cents = balance_cents
  end

  def richer_than?(other)
    balance_cents > other.balance_cents  # protected reader called on another Account
  end

  def to_s
    "#{label}: #{balance_cents} cents"   # private method, implicit receiver
  end

  protected
  attr_reader :balance_cents

  private
  def label = "Account ##{id}"
end

a = Account.new(1, 500)
b = Account.new(2, 300)
a.richer_than?(b)      # => true
a.to_s                 # => "Account #1: 500 cents"
a.balance_cents        # NoMethodError (protected method called)
a.label                # NoMethodError (private method called)
a.send(:label)         # => "Account #1": send ignores visibility
a.public_send(:label)  # NoMethodError
```

| | Implicit receiver (`label`) | `self.label` | `other.label` inside the class | From outside |
|---|---|---|---|---|
| public | yes | yes | yes | yes |
| protected | yes | yes | yes, when `self` is the same kind of object | no |
| private | yes | yes (Ruby 2.7+; setters were always allowed) | no | no |

Ruby's `private` is per object, not per class as in Java or C++: an `Account` cannot call another `Account`'s private method. That is exactly what `protected` exists for. Ways to declare visibility: `private` alone (applies to the following `def`s), `private :name`, `private def name ... end` (works because `def` returns a symbol), `private_class_method :build` and `private_constant :SECRET`. The bare `private` keyword does **not** affect `def self.x` methods, a common mistake; use `private_class_method` or `private` inside `class << self`.

## 5. Inheritance and super

```ruby
class Notifier
  def initialize(recipient)
    @recipient = recipient
  end

  def deliver(message, urgent: false)
    "to #{@recipient}: #{message}#{' (urgent)' if urgent}"
  end
end

class SmsNotifier < Notifier
  def initialize(recipient, sender_id)
    super(recipient)               # passes exactly this argument
    @sender_id = sender_id
  end

  def deliver(message, urgent: false)
    "[#{@sender_id}] " + super     # bare super: same message and urgent: values, same block
  end
end

SmsNotifier.new("+92300", "SHOP").deliver("Rent due", urgent: true)
# => "[SHOP] to +92300: Rent due (urgent)"
```

| Form | Passes |
|---|---|
| `super` | the current method's arguments (their current values) |
| `super()` | no arguments |
| `super(a, b)` | exactly `a, b` |

All three forms pass the current block along implicitly. A classic bug is a subclass `initialize` with extra parameters calling bare `super`: it forwards both arguments to a parent that takes one and raises `ArgumentError`. For type checks, `sms.is_a?(Notifier)` (alias `kind_of?`) is true for the class or any ancestor, `sms.instance_of?(Notifier)` only for the exact class, and `SmsNotifier < Notifier` is true for a subclass.

Ruby allows only one superclass; behavior shared across unrelated classes goes into modules. Rails Single Table Inheritance (`class AdminUser < User` stored in `users` with a `type` column) is covered in [ActiveRecord advanced](../../05-ruby-on-rails/06-activerecord-advanced.md).

## 6. Modules: namespaces and mixins

```ruby
module Billing
  TAX_BASIS_POINTS = 1_700          # constants are namespaced too
  class Invoice; end
  class Error < StandardError; end
end

Billing::Invoice.new
Billing::TAX_BASIS_POINTS           # => 1700

module Discountable                 # a mixin relies on a contract with its host
  def discounted_cents(percent)
    price_cents - (price_cents * percent / 100)   # the host must define price_cents
  end
end

class Listing
  include Discountable
end
Listing.new(sku: "A1", price_cents: 5_000).discounted_cents(10) # => 4500
```

Modules have no instances (`Discountable.new` raises `NoMethodError`) and no subclasses. Use a class when you need objects with state; use a module to share behavior across classes, to group related constants and classes, or to hold stateless helpers (`module_function` makes `Slug.call(text)` callable directly on the module).

## 7. include vs extend vs prepend

```ruby
module Auditable
  def save
    puts "audit: before save"
    super                         # continue to the class's own save
  end
end

module Greeting
  def hello = "hello from Greeting"
end

module ClassHelpers
  def table_name = name.downcase + "s"
end

class Record
  def save
    puts "Record#save"
  end
end

class Invoice < Record
  prepend Auditable               # goes BEFORE Invoice in the lookup chain
  include Greeting                # goes right AFTER Invoice
  extend ClassHelpers             # its methods become Invoice's class methods

  def save
    puts "Invoice#save"
    super
  end
end

Invoice.ancestors.take(4)  # => [Auditable, Invoice, Greeting, Record]
Invoice.new.save
# audit: before save
# Invoice#save
# Record#save
Invoice.new.hello          # => "hello from Greeting"
Invoice.table_name         # => "invoices"
```

| | `include M` | `extend M` | `prepend M` |
|---|---|---|---|
| Where `M` goes | after the class in `ancestors` | into the receiver's singleton class | before the class in `ancestors` |
| `M`'s methods become | instance methods | singleton methods of that object (class methods for a class) | instance methods that run first |
| `super` inside `M` reaches | the superclass's method | the object's original method, if any | the class's own method |
| Typical use | shared behavior (`Comparable`, concerns) | class-level DSL helpers, decorating one object | wrapping: logging, caching, instrumentation, patching a gem method |
| Hook called on `M` | `included(base)` | `extended(base)` | `prepended(base)` |

With several includes, the last one included is checked first: `include A` then `include B` gives `[Widget, B, A, ...]`. Including a module that is already in the chain does nothing. `prepend` replaced the old `alias_method_chain` technique (deprecated in Rails 5) for wrapping existing methods.

## 8. How Ruby finds a method

```text
invoice.save
  1. invoice's singleton class         methods defined on this one object
  2. modules prepended to Invoice      Auditable
  3. Invoice itself
  4. modules included in Invoice       Greeting (last included is checked first)
  5. superclass Record, with its own prepends and includes, and so on upward
  6. Object, then Kernel, then BasicObject
  7. not found: repeat the search for method_missing
  8. no method_missing override: raise NoMethodError
```

Tools for finding out what actually runs:

```ruby
Invoice.ancestors                                  # the whole chain
Invoice.instance_method(:save).owner               # => Auditable (first match)
Invoice.new.method(:save).super_method.owner       # => Invoice
Invoice.new.method(:save).source_location          # => [file path, line number]
```

In a legacy monolith, `source_location` is the fastest way to discover which gem, concern or monkey patch really defines a method.

## 9. Comparable and Enumerable mixins

```ruby
class Money
  include Comparable
  attr_reader :cents, :currency

  def initialize(cents, currency = "PKR")
    @cents = cents
    @currency = currency
  end

  def <=>(other)
    return nil unless other.is_a?(Money) && other.currency == currency
    cents <=> other.cents
  end
end

a = Money.new(500)
b = Money.new(300)
a > b                              # => true
[a, b].max.cents                   # => 500 (max uses <=>)
b.between?(Money.new(100), a)      # => true
Money.new(900).clamp(b, a).cents   # => 500
a == Money.new(500)                # => true: Comparable derives == from <=>
a > Money.new(1, "USD")            # ArgumentError (comparison of Money with Money failed)
```

Returning `nil` from `<=>` signals "not comparable", which makes the operators raise instead of giving a wrong answer. `Enumerable` works the same way with `each` as the single required method (see [Collections and Enumerable](02-collections-enumerable.md)).

## 10. Singleton classes and `class << self`

Every object can have methods of its own. They live in its singleton class (also called the eigenclass or metaclass), a hidden class placed first in that object's lookup chain.

```ruby
config = Object.new
def config.region
  "eu-west"
end
config.singleton_methods          # => [:region]
Object.new.respond_to?(:region)   # => false

class Report
  def self.build = new            # a singleton method of the Report object
  class << self                   # self is now Report's singleton class
    attr_accessor :default_format # a class-level attribute
  end
end
Report.singleton_methods.sort     # => [:build, :default_format, :default_format=]
Report.singleton_class            # => #<Class:Report>
```

So "class methods" are not special: they are singleton methods of a class object. `class << self` is handy for grouping many class methods, defining class-level `attr_accessor`s and making class methods private. `obj.extend(SomeModule)` also works through the singleton class, adding the module's methods to that one object only.

## 11. Duck typing and respond_to?

```ruby
require "stringio"

class CsvExporter
  def export(rows, io)            # io can be a File, StringIO, socket or Rack body
    rows.each { |row| io.write(row.join(",") + "\n") }
  end
end
buffer = StringIO.new
CsvExporter.new.export([["A1", 100]], buffer)
buffer.string                     # => "A1,100\n"

class SalesChannel                # an "abstract" base class documents the contract
  def publish(_listing) = raise(NotImplementedError, "#{self.class} must implement publish")
end
```

The exporter needs "something with `write`", not a specific class. Use `respond_to?` when a method must accept several shapes, and prefer moving behavior into each class over `case obj when CardPayment ... when WalletPayment` type switches.

Ruby has no `interface` keyword; the contract lives in documentation, tests and base classes like the one above. Note that `NotImplementedError` inherits from `ScriptError`, not `StandardError`, so a bare `rescue` does not catch it. Implicit conversion methods (`to_str`, `to_ary`, `to_proc`, `to_hash`) let your objects stand in for built-ins; for example `&obj` calls `obj.to_proc`.

## 12. to_s and inspect

```ruby
class Sku
  def initialize(code)
    @code = code
  end
  def to_s = @code                   # puts, interpolation, string contexts
  def inspect = "#<Sku #{@code}>"    # p, console, error messages
end
sku = Sku.new("ZELDA-01")
puts sku          # prints ZELDA-01
"SKU: #{sku}"     # => "SKU: ZELDA-01"
p sku             # prints #<Sku ZELDA-01>
```

Without overrides, `to_s` prints the class and address and `inspect` also lists instance variables. Override `inspect` to keep tokens and secrets out of logs; Rails models do this for sensitive columns through `filter_attributes`.

## 13. Value objects: ==, eql? and hash

```ruby
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def ==(other)
    other.is_a?(Point) && x == other.x && y == other.y
  end
  alias eql? ==

  def hash = [self.class, x, y].hash
end

Point.new(1, 2) == Point.new(1, 2)              # => true
{ Point.new(1, 2) => "home" }[Point.new(1, 2)]  # => "home"
[Point.new(1, 2), Point.new(1, 2)].uniq.size    # => 1
```

The contract: if `a.eql?(b)` then `a.hash == b.hash`. Override `==` alone and Hash lookups, `uniq` and Set membership silently treat equal objects as different.

## 14. Struct and Data

```ruby
Coordinates = Struct.new(:lat, :lng) do
  def to_s = "#{lat},#{lng}"
end

c = Coordinates.new(31.52, 74.35)
c.lat = 0                          # Struct generates setters: mutable
c == Coordinates.new(0, 74.35)     # => true: value equality is built in
c.to_a                             # => [0, 74.35]
Coordinates.new(1).lng             # => nil: missing arguments default to nil

Price = Data.define(:amount_cents, :currency) do   # Ruby 3.2+, immutable
  def to_s = "#{currency} #{amount_cents}"
end

original = Price.new(amount_cents: 4_999, currency: "PKR")  # keyword or positional
cheaper  = original.with(amount_cents: 3_999)                # copy with changes
original.amount_cents              # => 4999, unchanged
original.frozen?                   # => true
original.amount_cents = 1          # NoMethodError: no setters
Price.new(amount_cents: 1)         # ArgumentError (missing keyword: :currency)
original == Price.new(4_999, "PKR") # => true
```

| | Plain class | `Struct` | `Data` (3.2) |
|---|---|---|---|
| `==`, `eql?`, `hash`, `to_h`, `inspect` | write them yourself | generated | generated |
| Setters | whatever you define | generated (mutable) | none (immutable, frozen) |
| Missing constructor arguments | your choice | `nil` | `ArgumentError` |
| Pattern matching support | define `deconstruct`/`deconstruct_keys` | built in | built in |
| Best for | behavior-rich domain objects | quick mutable records, scripts | value objects, results, DTOs |

`OpenStruct` is slow and accepts any attribute name, so typos pass silently; prefer `Struct`, `Data` or a plain Hash.

## 15. ActiveSupport::Concern

```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  included do                         # evaluated inside the including class
    scope :archived, -> { where.not(archived_at: nil) }
    scope :active,   -> { where(archived_at: nil) }
  end

  class_methods do                    # become class methods of the includer
    def archive_all
      update_all(archived_at: Time.current)
    end
  end

  def archive!                        # plain instance methods
    update!(archived_at: Time.current)
  end

  def archived? = archived_at.present?
end

class Listing < ApplicationRecord
  include Archivable
end

Listing.active.count
Listing.where(store_id: 7).archive_all  # class methods also work on relations
```

Without Concern you would write the hook yourself: a `def self.included(base)` that calls `base.extend(ClassMethods)` for class methods and `base.class_eval { scope ... }` for class-level declarations. Concern also handles concerns that depend on other concerns, so each module's `included` block runs against the final model class. Good concerns capture genuinely shared behavior (archiving, slugs, tenant scoping such as a `belongs_to :organization` plus a `for_organization` scope). Bad concerns split one fat model into several files of model-specific code, which hides coupling instead of removing it.

## Common pitfalls
- Writing `quantity = 0` inside a method and wondering why the attribute did not change.
- Expecting the bare `private` keyword to hide `def self.x` class methods.
- Using `@@` variables and having a subclass overwrite the parent's value.
- Expecting class instance variables to be inherited; subclasses read `nil`.
- Bare `super` in an `initialize` whose parameters differ from the parent's.
- Overriding `==` without `eql?` and `hash`, breaking Hash keys and `uniq`.
- `is_a?` type switches where polymorphism (or duck typing) would remove the conditional.
- Two included modules defining the same method: the last one included wins silently.
- Mutable class-level state (registries, caches) shared across threads and requests.

## How to talk about this in an interview
- Anchor answers in Rails structures you work with daily: models inheriting from `ApplicationRecord`, concerns for shared behavior, service objects for workflows, `prepend` for safely wrapping a gem's method, `ancestors` and `source_location` for finding out what really runs in a legacy codebase.
- As a former OOP lab TA you can explain these ideas cleanly and contrast them with C++ or Java (Ruby's per-object `private`, mixins instead of multiple inheritance, no interfaces).
- > **Fill in:** which language the PUCIT OOP labs used and one concept students struggled with that you can explain with a Ruby comparison.
- > **Fill in:** a concern, module or class hierarchy you added or changed at Devntech (for example in the Colochousing Rails app or the CrystalCommerce platform) and why you chose a module over inheritance or vice versa.

## Interview questions

**Q1. What is the difference between `include`, `extend` and `prepend`?**
`include` inserts the module after the class in the ancestor chain, so its methods become instance methods that the class can override. `extend` adds the module's methods to a single object's singleton class; extending a class gives class methods. `prepend` inserts the module before the class, so its methods run first and can call `super` to reach the class's own implementation, which makes it ideal for wrapping.
*Follow-up:* How do you see the difference? Compare `Klass.ancestors` after each; a prepended module appears before the class.

**Q2. How does Ruby find a method when you call it?**
It walks the receiver's singleton class, then prepended modules, the class, included modules (most recent first), and repeats up the superclass chain through `Object`, `Kernel` and `BasicObject`. If nothing matches, it searches the same chain for `method_missing`, and the default implementation raises `NoMethodError`.
*Follow-up:* How do you find where a method is defined? `obj.method(:name).owner` and `.source_location`.

**Q3. Private vs protected?**
Private methods cannot be called with an explicit receiver (except `self.` since Ruby 2.7), so only the object itself can call them. Protected methods can be called on another object when the caller is an instance of the same class family, which suits comparisons like `balance > other.balance`. Neither is callable from outside, though `send` bypasses both.
*Follow-up:* How is Ruby's private different from Java's? It is per object, not per class.

**Q4. What is a singleton class?**
A hidden class that every object can have, holding methods defined for that object alone (`def obj.greet`). It sits first in the object's method lookup. Class methods are singleton methods of the class object, which is why `class << self` opens the place where they live.
*Follow-up:* What does `extend` have to do with it? `obj.extend(M)` includes `M` into `obj`'s singleton class.

**Q5. Class variables vs class instance variables?**
`@@var` is a single slot shared by the class and every subclass, so a subclass assignment changes the parent's value. A class instance variable is a normal instance variable on the class object, so each class has its own and subclasses start with `nil`. Prefer class instance variables, or Rails `class_attribute` when subclasses should inherit and override.
*Follow-up:* Why care in a Rails app? Class-level state is shared across all threads in a Puma or Sidekiq process.

**Q6. Does Ruby support multiple inheritance?**
No. A class has exactly one superclass. Modules mixed in with `include` or `prepend` give you shared behavior from many sources, and the linear `ancestors` list resolves conflicts predictably (no diamond ambiguity).
*Follow-up:* What happens when two modules define the same method? The one later in lookup order loses; the last included module wins.

**Q7. What is the difference between `super`, `super()` and `super(a)`?**
Bare `super` forwards the current method's arguments with their current values. `super()` passes none and `super(a)` passes exactly what you list. All forms pass the block along implicitly.
*Follow-up:* What bug does bare `super` cause in `initialize`? When the subclass takes extra parameters, they are forwarded too and the parent raises `ArgumentError`.

**Q8. What is duck typing, and how does it replace interfaces?**
Code depends on behavior ("responds to `write`") rather than class membership, so any object with the right methods works: a File, a StringIO or a socket. The contract is enforced by tests and documentation, optionally with a base class whose methods raise `NotImplementedError`. Use `respond_to?` sparingly and prefer polymorphism over type checks.
*Follow-up:* What is the risk? Errors appear at runtime, so test coverage matters more than in statically typed languages.

**Q9. When would you use Struct, Data or a plain class?**
`Struct` for quick mutable records with generated equality and accessors. `Data` (Ruby 3.2) for immutable value objects: no setters, required arguments, and `with` for modified copies. A plain class when the object carries real behavior and invariants.
*Follow-up:* Why avoid `OpenStruct`? It is slow and accepts any attribute name, so typos go unnoticed.

**Q10. What does `ActiveSupport::Concern` give you over a plain module?**
An `included do ... end` block evaluated inside the including class (for scopes, callbacks and validations), a `class_methods do ... end` block that becomes class methods, and correct handling of concerns that depend on other concerns. Without it you write `self.included(base)` with `base.extend` and `base.class_eval` by hand.
*Follow-up:* When are concerns a smell? When they only split one model's code into files without real reuse.

**Q11. Why do you need `self.` when calling a setter inside a method?**
Because Ruby treats `name = value` as a local variable assignment at parse time. `self.name = value` makes it a method call to `name=`. Readers do not need `self.` because there is no ambiguity.
*Follow-up:* Is `self.` allowed for private setters? Yes, private setters were always callable with `self.`.

**Q12. How do you make objects work as Hash keys or with `uniq`?**
Override `eql?` (usually aliased to `==`) and `hash` consistently so equal objects produce equal hash values. `Struct` and `Data` do this automatically.
*Follow-up:* What happens if only `==` is overridden? Lookups and de-duplication treat equal objects as different keys.

**Q13. How does the `Comparable` mixin work?**
You define `<=>` returning -1, 0, 1, or `nil` when the objects cannot be compared, then `include Comparable`. You get `<`, `<=`, `==`, `>`, `>=`, `between?` and `clamp` for free, and `sort`, `min` and `max` work on collections of your objects.
*Follow-up:* What happens when `<=>` returns `nil`? The comparison operators raise `ArgumentError`.

**Q14. What is the difference between `to_s` and `inspect`?**
`to_s` is the human-readable form used by `puts` and string interpolation. `inspect` is the developer form used by `p`, the console and error messages, and by default shows the class and instance variables. Override both on value objects, and override `inspect` to avoid leaking secrets into logs.
*Follow-up:* Which one does `"#{obj}"` call? `to_s`.

**Q15. Module or class: how do you choose?**
Use a class when you need instances with their own state and a lifecycle. Use a module to share behavior among unrelated classes (mixins), to group stateless helpers (`module_function`), or as a namespace. Note `is_a?` is true for mixed-in modules too, while `instance_of?` matches only the exact class.
*Follow-up:* Can a module have state? It can hold constants and module-level instance variables, but it has no instances.

**Q16. How do you define a private class method?**
Use `private_class_method :name` after defining it, or define it under `private` inside `class << self`. Writing `private` in the class body and then `def self.name` does not make it private.
*Follow-up:* And a private constant? `private_constant :NAME`.

## Further reading
- [Ruby modules and classes (Ruby docs)](https://docs.ruby-lang.org/en/master/syntax/modules_and_classes_rdoc.html)
- [Module (Ruby docs)](https://docs.ruby-lang.org/en/master/Module.html)
- [ActiveSupport::Concern API](https://api.rubyonrails.org/classes/ActiveSupport/Concern.html)
