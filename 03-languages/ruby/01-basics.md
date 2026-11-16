# Ruby Basics

> **Tier:** T1 (know cold) · **Related:** [Collections and Enumerable](02-collections-enumerable.md), [OOP, modules and mixins](03-oop-modules-mixins.md), [Blocks, procs and lambdas](04-blocks-procs-lambdas.md), [Advanced internals](06-advanced-internals.md), [Rails fundamentals](../../05-ruby-on-rails/01-rails-fundamentals.md), [Programming fundamentals](../../01-cs-fundamentals/01-programming-fundamentals.md)
> **On your resume:** Ruby is your daily language at Devntech (legacy Rails monoliths, REST APIs and background jobs for CrystalCommerce and Colochousing) and was the core of the Visnext Rails training.

## Quick revision
- Everything is an object, including `1`, `nil` and classes. Operators are methods: `1 + 2` is `1.+(2)`.
- Dynamic (types are checked at runtime) and strong (`"1" + 1` raises `TypeError`; no silent coercion).
- Only `nil` and `false` are falsy. `0`, `""`, `[]` and `{}` are all truthy.
- Variable kinds: `local`, `@instance`, `@@class`, `$global`, `CONSTANT`. `def`, `class` and `module` are scope gates; blocks are not.
- Integer division floors: `7 / 2 == 3` and `-7 / 2 == -4`. Use `fdiv` or a Float operand for decimals.
- Floats are binary approximations: `0.1 + 0.2 != 0.3`. Money goes in `BigDecimal` or integer cents, never Float.
- Strings are mutable. `<<` mutates in place, `+` builds a new string. `# frozen_string_literal: true` freezes literals.
- Symbols are immutable, interned names (`:status`); strings are text data. `{ "a" => 1 }[:a]` is `nil`.
- `case/when` uses `===`: classes match instances, ranges check inclusion, regexes match strings, lambdas get called.
- `&.` protects only against `nil` and covers only the next call; repeat it at each level of a chain.
- Methods return their last expression. `?` methods answer yes/no; `!` methods are the "dangerous" twin (mutate or raise).
- Keyword arguments are separate from positional arguments since Ruby 3.0; pass a hash as keywords with `**opts`.
- `puts`/`print` use `to_s` and return `nil`; `p`/`pp` use `inspect` and return their argument.
- `==` same value, `eql?` same value and type (Hash keys), `equal?` same object, `===` case match.
- `dup` drops frozen state and singleton methods; `clone` keeps both. Both copies are shallow, and so is `freeze`.
- `require` loads once from `$LOAD_PATH`, `require_relative` resolves from the current file, `load` re-runs every time. Rails app code is autoloaded by Zeitwerk.

Return values in comments use the Ruby 3.4 `inspect` format (`{name: "Ali"}`); Ruby 3.3 and older print `{:name=>"Ali"}`.

## 1. Everything is an object

```ruby
1.class                   # => Integer
nil.class                 # => NilClass
true.class                # => TrueClass
Integer.class             # => Class (classes are objects too)
1.+(2)                    # => 3, the same call as 1 + 2
5.times.map { |i| i * 2 } # => [0, 2, 4, 6, 8]
"hi".respond_to?(:upcase) # => true
```

Why it matters: you can call methods on anything (`nil.to_a`, or `3.days.ago` in Rails), and you can define operators such as `==`, `<=>` and `+` on your own classes. There are no primitives with special rules like Java's `int`.

## 2. Dynamic and strong typing

Dynamic means variables have no declared type; values do, and type errors surface at runtime. Strong means Ruby rarely converts types for you.

```ruby
x = 42
x = "now a string"   # fine: the variable has no type

"Total: " + 5        # TypeError (no implicit conversion of Integer into String)
"Total: " + 5.to_s   # => "Total: 5"
"Total: #{5}"        # => "Total: 5" (interpolation calls to_s)
5 + "5"              # TypeError (String can't be coerced into Integer)
```

JavaScript is dynamic and weak (`"5" + 5` is `"55"`). Ruby has optional static typing through RBS signature files (shipped with Ruby 3.0) and Sorbet, but most Rails codebases rely on tests instead.

## 3. Variables, constants and scope

| Kind | Syntax | Visible in | Reading before assignment |
|---|---|---|---|
| Local | `total` | current method, block body or file | `NameError` |
| Instance | `@total` | one object | `nil` |
| Class | `@@count` | the class and all subclasses (one shared slot) | `NameError` |
| Global | `$stdout` | the whole process | `nil` |
| Constant | `MAX_RETRIES` | lexical scope, then ancestors | `NameError` |

```ruby
MAX_RETRIES = 3
MAX_RETRIES = 5        # works, but prints "warning: already initialized constant"

x = 10
def show_x
  x                    # NameError: def opens a brand new local scope
end

[1, 2].each { |n| puts n + x }  # blocks are closures: they see outer locals
```

**Scope gates.** `def`, `class` and `module` start a fresh local scope and change `self`. Blocks keep access to the surrounding locals, which is why metaprogramming uses `define_method` and `Class.new` with blocks to "flatten" scope (see [Metaprogramming](05-metaprogramming.md)).

A parse-time surprise: Ruby decides whether a name is a local variable while parsing, not while running.

```ruby
if false
  ghost = 1
end
ghost # => nil, not NameError: the parser saw an assignment, so ghost is a local
```

**Constants** start with an uppercase letter, and class and module names are constants too. Assigning a constant inside a method is a `SyntaxError` ("dynamic constant assignment"). Freeze constant collections: `ROLES = %w[admin member].freeze`.

**Globals** are best avoided. You will meet built-ins such as `$stdout`, `$PROGRAM_NAME` and `$!` (the exception being handled).

## 4. Numbers: Integer, Float and BigDecimal

```ruby
2**100            # => 1267650600228229401496703205376 (Integers never overflow)
1_000_000         # underscores for readability
7 / 2             # => 3   integer division
-7 / 2            # => -4  floors toward negative infinity (C++ and Java give -3)
7 % 3             # => 1
-7 % 3            # => 2   the result takes the sign of the divisor
7.fdiv(2)         # => 3.5
7 / 2.0           # => 3.5
10.divmod(3)      # => [3, 1]
3.7.to_i          # => 3   truncates
3.14159.round(2)  # => 3.14
2.5.round         # => 3   half away from zero
2.5.round(half: :even) # => 2 (banker's rounding)
"42abc".to_i      # => 42  lenient
"abc".to_i        # => 0   silently
Integer("abc")    # ArgumentError: strict parsing, better for input
Integer("abc", exception: false) # => nil
```

```ruby
0.1 + 0.2                               # => 0.30000000000000004
0.1 + 0.2 == 0.3                        # => false
(0.1 + 0.2 - 0.3).abs < Float::EPSILON  # => true
10.0 / 0                                # => Infinity
10 / 0                                  # ZeroDivisionError
```

**Money.** Floats store binary fractions, so `0.1` cannot be represented exactly and errors accumulate across thousands of order lines. Use `BigDecimal` (exact decimal arithmetic) or store integer cents and format at the edges.

```ruby
require "bigdecimal"
require "bigdecimal/util"     # adds String#to_d

price = BigDecimal("19.99")   # build from a String, never from a Float
(price * 3).to_s("F")         # => "59.97"
BigDecimal("0.1") + BigDecimal("0.2") == BigDecimal("0.3") # => true
"19.99".to_d                  # => 0.1999e2
```

In Rails, `t.decimal :price, precision: 10, scale: 2` columns come back as `BigDecimal`. In recent Ruby versions `bigdecimal` ships as a bundled gem rather than a default gem, so a plain Bundler project may need it in the Gemfile; check the release notes for your version.

## 5. Strings

```ruby
name = "Ali"
"Hello, #{name}\n"       # double quotes: interpolation and escapes
'Hello, #{name}\n'       # single quotes: literal text, no interpolation
%w[red green blue]       # => ["red", "green", "blue"]
%i[draft published]      # => [:draft, :published]

sql = <<~SQL             # squiggly heredoc strips the common indentation
  SELECT id, total
  FROM orders
  WHERE status = 'paid'
SQL
```

**Mutability.** Variables hold references, so two names can point at one string.

```ruby
a = "sku"          # file without the frozen_string_literal comment
b = a              # b points to the SAME object
b << "-42"         # << appends in place
a                  # => "sku-42"
c = a + "!"        # + allocates a new string; a is untouched
a.equal?(c)        # => false
```

**Frozen string literals.** The magic comment must be the first line (after an optional shebang).

```ruby
# frozen_string_literal: true
label = "draft"
label << "!"       # FrozenError (can't modify frozen String)
label += "!"       # fine: += rebinds label to a new string
buffer = +""       # unary plus returns a mutable copy (String.new works too)
buffer << "row 1\n"
```

Frozen literals save allocations and prevent accidental shared mutation. Ruby 3.4 made literals in files without the comment "chilled": mutating them still works but can emit a deprecation warning, a step toward frozen literals by default (details in [Advanced internals](06-advanced-internals.md)).

| Task | Code | Result |
|---|---|---|
| Case | `"ruby dev".upcase`, `"ruby dev".capitalize` | `"RUBY DEV"`, `"Ruby dev"` |
| Trim | `"  hi  ".strip` | `"hi"` |
| Split and join | `"a,b,,c".split(",")`, `%w[a b].join("-")` | `["a", "b", "", "c"]`, `"a-b"` |
| Search | `"hello".include?("ell")`, `"hello".start_with?("he")` | `true`, `true` |
| Replace | `"hello".sub("l", "L")`, `"hello".gsub("l", "L")` | `"heLlo"`, `"heLLo"` |
| Slice | `"hello"[0]`, `"hello"[1..3]`, `"hello"[-3..]` | `"h"`, `"ell"`, `"llo"` |
| Size | `"résumé".length`, `"résumé".bytesize` | `6`, `8` (UTF-8 bytes) |
| Format | `format("%.2f", 3.14159)`, `"%05d" % 42` | `"3.14"`, `"00042"` |
| Regex | `"a1b2".scan(/\d/)`, `"x9".match?(/\d/)` | `["1", "2"]`, `true` |
| Convert | `"3.5".to_f`, `"name".to_sym` | `3.5`, `:name` |

`split` drops trailing empty fields: `"a,b,,".split(",")` is `["a", "b"]`. Pass `-1` as the limit (`split(",", -1)`) to keep them, which matters when importing CSV-like data. ActiveSupport adds `blank?`, `present?`, `squish`, `truncate`, `titleize`, `parameterize`, `pluralize`, `camelize` and `underscore`.

## 6. Symbols vs strings

```ruby
:status.object_id == :status.object_id   # => true: one object per name
"status".object_id == "status".object_id # => false: a new String each time (no magic comment)
:status.frozen?                          # => true
:status.to_s                             # => "status"
"status".to_sym                          # => :status
```

| | Symbol | String |
|---|---|---|
| Mutability | Immutable | Mutable unless frozen |
| Identity | One object per name, process-wide | A new object per literal evaluation, unless frozen literals are deduplicated |
| Use for | Names: hash keys, method names, enum values, options | Data: user input, text, JSON values |
| Comparison | Identity check, cheap | Byte by byte |
| Garbage collection | Symbols created at runtime (`to_sym`) are collected since Ruby 2.2 | Normal |

The classic bug is mixing key types:

```ruby
payload = { "name" => "Ali" }
payload[:name]                                       # => nil, the key is a String
JSON.parse('{"name":"Ali"}')                         # => {"name" => "Ali"}
JSON.parse('{"name":"Ali"}', symbolize_names: true)  # => {name: "Ali"}
```

Rails `params` (`ActionController::Parameters`) and `HashWithIndifferentAccess` accept both, which hides the difference until the data reaches a plain Hash, for example inside a background job.

## 7. nil, true, false and truthiness

Only `nil` and `false` are falsy. Everything else, including `0`, `""` and `[]`, is truthy.

```ruby
puts "runs" if 0         # prints
puts "runs" if ""        # prints
puts "runs" unless nil   # prints
nil.to_a                 # => []
nil.to_s                 # => ""
nil.to_i                 # => 0
!!"anything"             # => true (double negation converts to a boolean)
```

ActiveSupport adds `blank?` (true for `nil`, `false`, `""`, whitespace-only strings, `[]` and `{}`), `present?` (the opposite) and `presence` (the object or `nil`): `params[:q].presence || "all"`.

A real production bug: form and query params are strings, so `params[:active]` may be `"false"`, which is truthy. Cast it:

```ruby
ActiveModel::Type::Boolean.new.cast("false") # => false
ActiveModel::Type::Boolean.new.cast("1")     # => true
```

## 8. Ranges

```ruby
(1..5).to_a            # => [1, 2, 3, 4, 5]  inclusive
(1...5).to_a           # => [1, 2, 3, 4]     exclusive end
("a".."e").to_a        # => ["a", "b", "c", "d", "e"]
(1..10).step(3).to_a   # => [1, 4, 7, 10]
(1..).first(3)         # => [1, 2, 3]        endless range (2.6); beginless (..5) came in 2.7
(1.0..2.0).cover?(1.5) # => true

case 30
when 0...18 then "minor"
when (18..) then "adult"
end                    # => "adult"
```

In Rails, `Order.where(created_at: 1.week.ago..)` produces `created_at >= ...` in SQL.

## 9. Conditionals: if, unless, case, ternary, modifiers, `&.`

```ruby
status = if order.paid? then "paid" else "due" end  # if is an expression
label  = order.paid? ? "paid" : "due"              # ternary
return unless user                                 # guard clause with a modifier
puts "big order" if order.total > 1_000
```

**`case` uses `===`**, called on the `when` value with the subject as the argument:

```ruby
def classify(value)
  case value
  when nil                then "missing"
  when Integer, Float     then "number"            # Class#=== means is_a?
  when /\A[A-Z]{3}-\d+\z/ then "SKU"               # Regexp#=== matches strings
  when String             then "text"
  when ->(v) { v.respond_to?(:each) } then "collection" # Proc#=== calls it
  else "unknown"
  end
end

classify(42)        # => "number"
classify("ABC-123") # => "SKU"
classify("hello")   # => "text"
classify([1, 2])    # => "collection"
```

| `when` value | `===` means | Example |
|---|---|---|
| Class or module | `value.is_a?(klass)` | `Integer === 3` is true |
| Range | the value falls inside (`cover?`) | `(1..5) === 3` is true |
| Regexp | the pattern matches | `/\d/ === "a1"` is true |
| Proc or lambda | call it with the value | `->(n) { n.odd? } === 3` is true |
| Anything else | `==` | `"a" === "a"` is true |

`===` is not symmetric: `Integer === 3` is true but `3 === Integer` is false. For destructuring matches, Ruby 3 also has `case/in` pattern matching (see [Advanced internals](06-advanced-internals.md)).

**`and`/`or` have very low precedence**, lower than `=`:

```ruby
saved = true and false
saved   # => true, parsed as (saved = true) and false
saved = true && false
saved   # => false
```

Use `&&` and `||` for logic. Most style guides ban `and`/`or`, or allow them only for control flow such as `user.save or raise "failed"`.

**Safe navigation** (`&.`, Ruby 2.3) returns `nil` instead of calling a method on `nil`:

```ruby
user&.profile&.city   # nil if user or profile is nil
nil&.length           # => nil
false&.to_s           # => "false": only nil is skipped
user&.profile.city    # raises NoMethodError when profile is nil:
                      # &. covers only the call right after it
```

JavaScript's `?.` short-circuits the rest of the chain; Ruby's does not. ActiveSupport's `try(:method)` also returns `nil` when the receiver does not respond to the method, while `&.` raises `NoMethodError` in that case.

## 10. Iterators over loops

Idiomatic Ruby iterates with methods that take blocks; `while` and `until` exist but are rarer.

```ruby
3.times { |i| puts i }              # 0, 1, 2
1.upto(3) { |i| puts i }            # 1, 2, 3
10.step(1, -3) { |i| print i, " " } # 10 7 4 1
%w[a b c].each_with_index { |s, i| puts "#{i}: #{s}" }

attempts = 0
loop do                             # runs until break (also stops on StopIteration)
  attempts += 1
  break if attempts == 3
end

until queue.empty?                  # until is while-not
  process(queue.shift)
end
```

`next` skips to the next iteration, `break` leaves the loop, and both can carry a value:

```ruby
[1, 2, 3, 4].each do |n|
  next if n.even?
  break if n > 2
  puts n                                         # prints only 1
end

[5, 8, 12].each { |n| break n if n > 6 }         # => 8
[1, 2, 3].map { |n| next 0 if n == 2; n * 10 }   # => [10, 0, 30]
```

`for` exists but does not create a scope, so its variable leaks: after `for i in 1..3; end`, `i` is still `3`. That also changes closure behavior (see [Blocks, procs and lambdas](04-blocks-procs-lambdas.md)).

## 11. Methods

```ruby
def log(message, level = :info, *tags, user: nil, **context, &formatter)
  line = "[#{level}] #{message}"
  line += " tags=#{tags.join(',')}" if tags.any?
  line += " user=#{user}" if user
  line += " #{context}" unless context.empty?
  formatter ? formatter.call(line) : line   # the last expression is the return value
end

log("saved")                                # => "[info] saved"
log("sync", :warn, "api", "retry", user: 7) # => "[warn] sync tags=api,retry user=7"
log("paid", order_id: 42)                   # => "[info] paid {order_id: 42}"
log("x") { |l| l.upcase }                   # => "[INFO] X"
```

| Kind | Syntax | Receives |
|---|---|---|
| Required | `message` | a positional argument |
| Optional | `level = :info` | a positional argument with a default, evaluated on every call |
| Splat | `*tags` | extra positionals as an Array |
| Required keyword | `email:` | must be passed by name |
| Optional keyword | `user: nil` | a keyword with a default |
| Double splat | `**context` | extra keywords as a Hash |
| Block | `&formatter` | the block as a Proc (see [Blocks](04-blocks-procs-lambdas.md)) |

Defaults are evaluated per call, so `def add(item, list = [])` gets a fresh array every time (unlike Python's shared mutable default).

**Keyword separation (Ruby 3.0).** A Hash is no longer silently converted into keywords:

```ruby
def notify(user, urgent: false); end
opts = { urgent: true }
notify(user, opts)    # ArgumentError (wrong number of arguments)
notify(user, **opts)  # correct
```

**Naming and returns.**

```ruby
def empty_cart? = items.empty?     # ? for predicates (endless method syntax, Ruby 3.0)

def min_max(nums)
  [nums.min, nums.max]             # "multiple return values" are just an Array
end
low, high = min_max([4, 9, 1])     # destructuring: low = 1, high = 9

name = "  ali  "
name.strip    # => "ali", name unchanged
name.strip!   # => "ali", name is now "ali"
name.strip!   # => nil because nothing changed, so never chain bang methods
```

`!` means "the dangerous version", not always "mutates": in Rails, `save` returns `false` on validation failure while `save!` raises `ActiveRecord::RecordInvalid`. Setters are methods named with `=`: `def price=(value)`.

## 12. Printing: puts, print, p and pp

| Method | Converts with | Newline | Returns | Use for |
|---|---|---|---|---|
| `puts` | `to_s` | yes; arrays print one element per line | `nil` | user-facing output |
| `print` | `to_s` | no | `nil` | progress output |
| `p` | `inspect` | yes | its argument | quick debugging |
| `pp` | pretty `inspect` | yes, wraps long structures | its argument | nested hashes and arrays |

```ruby
puts nil          # prints an empty line
p nil             # prints nil
puts [1, [2]]     # prints 1 and 2 on separate lines (flattened)
p [1, [2]]        # prints [1, [2]]
puts "42"         # prints 42
p "42"            # prints "42" (the quotes tell you it is a String)
total = p(2 + 3)  # prints 5 and assigns 5
```

In Rails code, prefer `Rails.logger.debug` or a `debugger` breakpoint (the `debug` gem ships with Ruby 3.1+) over leftover `p` calls.

## 13. Equality and identity

| Method | Question it answers | Override it? | Example |
|---|---|---|---|
| `==` | Same value? | Yes, the usual one | `1 == 1.0` is true |
| `eql?` | Same value and same type? Used by Hash keys | Yes, together with `hash` | `1.eql?(1.0)` is false |
| `equal?` | Same object? | Never | `"a".equal?("a")` is false |
| `===` | Does it match or belong? Used by `case` | Yes | `Integer === 1` is true |

```ruby
a = "hi"
b = "hi"
a == b           # => true
a.eql?(b)        # => true
a.equal?(b)      # => false: two different objects
h = { 1 => "int" }
h[1.0]           # => nil, because 1.eql?(1.0) is false
```

`object_id` is unique per live object. Small integers, `nil`, `true`, `false` and symbols are single shared objects, so their ids never change. ActiveRecord overrides `==` to compare class and primary key, so two separately loaded copies of the same row are `==` but not `equal?`.

## 14. Copying and freezing: dup, clone and freeze

```ruby
s = "draft".freeze
s.dup.frozen?                  # => false
s.clone.frozen?                # => true
s.clone(freeze: false).frozen? # => false

obj = Object.new
def obj.special; "only me"; end   # a singleton method on this one object
obj.clone.special              # => "only me"
obj.dup.special                # NoMethodError: dup drops singleton methods

order = { items: ["A1"] }
copy = order.dup
copy[:items] << "B2"
order[:items]                  # => ["A1", "B2"]: shallow copy, the inner array is shared

CONFIG = { hosts: ["db1"] }.freeze
CONFIG[:port] = 5432           # FrozenError
CONFIG[:hosts] << "db2"        # allowed: freeze is shallow too
```

For deep copies use ActiveSupport's `deep_dup`, or `Marshal.load(Marshal.dump(obj))` for plain data. In Rails, `record.dup` returns a new, unsaved copy with no id, which is how "duplicate this listing" features are usually built.

## 15. Loading code: require, require_relative and load

| | Looks in | Runs again when repeated? | Typical use |
|---|---|---|---|
| `require "json"` | `$LOAD_PATH` (stdlib and gems) | No, tracked in `$LOADED_FEATURES` | libraries |
| `require_relative "support/helpers"` | the directory of the current file | No | scripts, gem internals, spec helpers |
| `load "seeds.rb"` | the path as given (extension required) | Yes, re-executes each time | reloading a script or config |

`require` returns `true` the first time and `false` afterwards. In a Rails app you almost never `require` your own code: Zeitwerk (the default since Rails 6 and the only autoloader since Rails 7) maps `app/models/sales_channel.rb` to `SalesChannel`, so file names must match constant names.

## 16. Style conventions

- `snake_case` for methods, variables and file names; `CamelCase` for classes and modules; `SCREAMING_SNAKE_CASE` for constants.
- Two-space indentation. Predicates end in `?` without an `is_` prefix (`active?`, not `is_active?`).
- Guard clauses instead of nested `if`; `unless` only without an `else`; `&&`/`||` instead of `and`/`or`.
- `each`/`map` instead of `for`; implicit return instead of a trailing `return`.
- Parentheses are omitted for DSL-style calls (`validates :email, presence: true`) and kept for normal calls with arguments.
- Let a linter settle the rest. RuboCop defaults to single quotes, while Standard and the `rubocop-rails-omakase` config that Rails 7.2+ generates prefer double quotes; follow whatever the project uses.

## Common pitfalls
- Storing prices in `Float` and seeing totals drift by a cent. Use `BigDecimal` or integer cents.
- Expecting `0`, `""` or the string `"false"` from params to be falsy.
- Averages with integer division: `[1, 2].sum / 2` is `1`. Use `fdiv` or `to_f`.
- `"abc".to_i` silently returns `0`; validate input with `Integer(value)`.
- Mutating a string that came from a caller (`<<`, `gsub!`) and changing their data too.
- Chaining bang methods (`str.strip!.downcase`) that return `nil` when nothing changed.
- `x = true and false` leaving `x` as `true`.
- Looking up a string-keyed Hash (for example parsed JSON) with symbols.
- Assuming `a&.b.c` is safe when `b` can be `nil`.
- Assuming `dup` or `freeze` reaches nested objects.
- Treating constant reassignment as an error; it is only a warning.

## How to talk about this in an interview
- Ruby is your daily language at Devntech across legacy Rails monoliths, REST APIs and background jobs, and it was the focus of your Visnext training. Rails interviewers expect these basics to be instant: give the one-sentence definition, then add a production angle ("which is why prices should never be Floats").
- Connect fundamentals to your e-commerce and API work: money and rounding in a catalog that prices inventory from market data, string vs symbol keys when parsing third-party JSON, truthiness of query params in REST endpoints.
- > **Fill in:** how prices are stored in the CrystalCommerce codebase (decimal column, integer cents or a money gem) and where rounding happens.
- > **Fill in:** one Ruby gotcha you actually hit at Devntech (for example a string key vs symbol key mismatch in an API payload) and how you found it.
- > **Fill in:** the Ruby and Rails versions your current projects run, so you can speak to version-specific features honestly.

## Interview questions

**Q1. What does "everything is an object" mean in Ruby?**
Every value, including integers, `nil`, `true` and classes themselves, is an instance of some class and responds to methods. Operators are method calls (`a + b` is `a.+(b)`), so your classes can define `==`, `<=>` or `+`. There are no primitive types with separate rules.
*Follow-up:* Is a class an object? Yes, every class is an instance of `Class`, which is why it can have its own (singleton) methods.

**Q2. Is Ruby statically or dynamically typed? Strong or weak?**
Dynamic: variables have no declared types and errors appear at runtime. Strong: it does not coerce unrelated types implicitly, so `"1" + 1` raises `TypeError`. Explicit conversion (`to_s`, `Integer()`) or interpolation is required.
*Follow-up:* How do teams get type safety? Tests first; optionally RBS signatures or Sorbet.

**Q3. Symbols vs strings: what is the difference and when do you use each?**
A symbol is an immutable, interned name: every `:status` is the same object, so comparing symbols is an identity check. Strings are mutable text, and each literal normally creates a new object. Use symbols for identifiers (hash keys, method names, enum values, options) and strings for data (user input, JSON values). Mixing them as hash keys is a common bug.
*Follow-up:* Can symbols leak memory? Symbols created at runtime with `to_sym` have been garbage collected since Ruby 2.2; literal symbols live for the whole process.

**Q4. Which values are falsy in Ruby?**
Only `nil` and `false`. `0`, `""`, `[]` and `{}` are truthy, unlike JavaScript and Python. ActiveSupport's `blank?` covers the "empty" cases when you need them.
*Follow-up:* Why is `if params[:active]` buggy? Params are strings, and `"false"` is truthy; cast with `ActiveModel::Type::Boolean`.

**Q5. Explain `==`, `eql?`, `equal?` and `===`.**
`==` compares values and is what classes usually override. `eql?` is stricter (same value and type, `1.eql?(1.0)` is false) and is what Hash uses with `hash` for keys. `equal?` checks object identity and should never be overridden. `===` is case equality, used by `case/when`: `Integer === 1`, `(1..5) === 3`, `/a/ === "cat"`.
*Follow-up:* What must you override to use your object as a Hash key? `eql?` and `hash`, consistently.

**Q6. What does `-7 / 2` return in Ruby, and why might that surprise you?**
`-4`, because integer division floors toward negative infinity, and `-7 % 2` is `1`. C++, Java and JavaScript's `Math.trunc` round toward zero and give `-3`. For a decimal result use `-7.fdiv(2)` or a Float operand.
*Follow-up:* What is `7 % -3`? `-2`: the remainder takes the sign of the divisor.

**Q7. How would you represent money in Ruby?**
Never as Float, because binary floating point cannot represent most decimal fractions exactly (`0.1 + 0.2` is `0.30000000000000004`). Use `BigDecimal` built from strings or decimal database columns, or integer cents with formatting at the edges. Be explicit about the rounding mode where it matters (BigDecimal supports modes such as `BigDecimal::ROUND_HALF_EVEN`).
*Follow-up:* Why not `BigDecimal(0.1)`? A Float is already inexact, so Ruby requires an explicit precision; build from `"0.1"` instead.

**Q8. What is the difference between `puts`, `print`, `p` and `pp`?**
`puts` and `print` call `to_s`; `puts` adds a newline (one per array element) and both return `nil`. `p` calls `inspect`, adds a newline and returns its argument, which makes it handy inside expressions while debugging. `pp` pretty-prints nested structures and also returns its argument.
*Follow-up:* What does `x = puts "hi"` assign? `nil`.

**Q9. What do `?` and `!` at the end of a method name mean?**
They are naming conventions, not syntax. `?` marks a predicate that answers yes/no (`empty?`, `paid?`). `!` marks the more dangerous version of a method that also exists without it: it mutates the receiver (`sort!`, `strip!`) or raises instead of returning a status (`save!`, `create!`).
*Follow-up:* What does `"abc".strip!` return? `nil`, because nothing changed, so chaining it breaks.

**Q10. What is the difference between `dup` and `clone`?**
Both make shallow copies. `clone` also copies the frozen state and singleton methods (and accepts `freeze: false`); `dup` returns an unfrozen copy without singleton methods. Nested objects are shared in both cases, so use `deep_dup` for deep copies.
*Follow-up:* What does `ActiveRecord#dup` give you? A new, unsaved record with the same attributes and no id.

**Q11. What does `# frozen_string_literal: true` do and why do projects add it?**
It freezes every string literal in that file, so mutating one raises `FrozenError` and identical literals can share one object. It reduces allocations and prevents accidental shared mutation. Use `+""`, `String.new` or `dup` when you need a mutable buffer.
*Follow-up:* What changed in Ruby 3.4? Literals in files without the comment are "chilled": mutation still works but can emit a deprecation warning.

**Q12. What is a scope gate?**
`def`, `class` and `module` each start a new local variable scope and change `self`, so locals defined outside are invisible inside. Blocks are closures and keep access to outer locals. That is why `define_method` with a block can use a local variable that a `def` could not.
*Follow-up:* How do you share a value across a scope gate without a global? Use a constant, an instance variable on the right object, or a block-based method such as `define_method`.

**Q13. Explain `require`, `require_relative` and `load`.**
`require` searches `$LOAD_PATH` and loads a file once, tracking it in `$LOADED_FEATURES`. `require_relative` loads a path relative to the current file, also once. `load` executes the file every time it is called. Rails apps rely on Zeitwerk autoloading, so app code is loaded by constant name instead.
*Follow-up:* Why must Rails file names match class names? Zeitwerk derives the expected constant from the file path.

**Q14. What is the difference between `&.` and ActiveSupport's `try`?**
`a&.b` returns `nil` when `a` is `nil` and otherwise calls `b`, raising `NoMethodError` if `a` does not respond to it. `a.try(:b)` also returns `nil` when `a` does not respond to `b`, which can hide typos. `&.` covers only the next call, so a chain needs it at every nullable level.
*Follow-up:* What does `false&.to_s` return? `"false"`: only `nil` is skipped.

**Q15. How do keyword arguments work, and what changed in Ruby 3.0?**
Keywords are named parameters, required (`email:`) or optional (`role: :member`), with `**opts` collecting extras. Since Ruby 3.0 they are fully separated from positional arguments: passing a Hash where keywords are expected raises `ArgumentError`, so you write `method(**hash)`.
*Follow-up:* Why prefer keywords over an options hash? Typos fail loudly and required keywords document the API.

**Q16. Why does `x = true and false` leave `x` as `true`?**
`and` has lower precedence than `=`, so Ruby parses it as `(x = true) and false`. `&&` binds tighter than `=`, so `x = true && false` assigns `false`. That is why style guides restrict `and`/`or` to control flow, if they allow them at all.
*Follow-up:* Where do people use `or` intentionally? Patterns such as `record.save or raise "failed"`.

**Q17. How does `case` decide which `when` branch matches?**
It calls `when_value === subject` for each branch in order and takes the first truthy result. Classes check `is_a?`, ranges check inclusion, regexes match, lambdas are called, and other objects fall back to `==`. A `when` can list several values separated by commas.
*Follow-up:* When would you use `case/in` instead? When you need to destructure arrays or hashes and bind variables (pattern matching).

## Further reading
- [Ruby documentation](https://docs.ruby-lang.org/en/)
- [Ruby syntax: methods](https://docs.ruby-lang.org/en/master/syntax/methods_rdoc.html)
- [Active Support core extensions](https://guides.rubyonrails.org/active_support_core_extensions.html)
