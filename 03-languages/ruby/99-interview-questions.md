# Ruby: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

Return values use the Ruby 3.4 `inspect` format (`{a: 1}`); Ruby 3.3 and older print `{:a=>1}`. Snippets assume no `frozen_string_literal` comment unless they show one.

## Basics (see [01-basics.md](01-basics.md))

**Q1. What kind of language is Ruby?**
A dynamic, strongly typed, object-oriented language. CRuby compiles source to YARV bytecode and interprets it, with an optional JIT (YJIT). Everything is an object and operators are methods.

**Q2. Which values are falsy in Ruby?**
Only `nil` and `false`. `0`, `""`, `[]` and `{}` are truthy.

**Q3. Symbols vs strings?**
Symbols are immutable and interned (one object per name), used as identifiers: hash keys, method names, enum values. Strings are mutable text data. `{ "a" => 1 }[:a]` is `nil` because the key types differ.

**Q4. `nil?` vs `empty?` vs `blank?`?**
`nil?` is true only for `nil`. `empty?` checks zero length on strings, arrays and hashes (`nil.empty?` raises `NoMethodError`). `blank?` (ActiveSupport) is true for `nil`, `false`, empty collections and whitespace-only strings.

**Q5. Single quotes vs double quotes?**
Double quotes support interpolation (`"#{x}"`) and escapes such as `\n`. Single quotes are literal apart from `\\` and `\'`.

**Q6. What does `# frozen_string_literal: true` do?**
Freezes every string literal in the file, so mutating one raises `FrozenError` and identical literals can share one object. Use `+""`, `dup` or `String.new` for a mutable string.

**Q7. Name Ruby's variable kinds.**
Local (`name`), instance (`@name`), class (`@@name`), global (`$name`) and constants (`NAME`). Uninitialized instance and global variables read as `nil`; locals, class variables and constants raise `NameError`.

**Q8. What is a scope gate?**
`def`, `class` and `module` open a new local scope, so outer locals are invisible inside. Blocks are closures and keep access to outer locals.

**Q9. What does `&.` do, and how does it differ from ActiveSupport's `try`?**
`a&.b` returns `nil` when `a` is `nil`, otherwise calls `b`. `try(:b)` also returns `nil` when `a` does not respond to `b`, which can hide typos. `&.` only covers the next call in a chain.

**Q10. Explain `==`, `eql?`, `equal?` and `===`.**
`==` is value equality, `eql?` is value plus type equality (used for Hash keys, so `1.eql?(1.0)` is false), `equal?` is object identity, and `===` is case equality used by `case/when`.

**Q11. `dup` vs `clone`?**
Both are shallow copies. `clone` keeps the frozen state and singleton methods; `dup` does not. Use `deep_dup` (ActiveSupport) for nested data.

**Q12. `puts` vs `print` vs `p` vs `pp`?**
`puts` and `print` use `to_s` and return `nil` (`puts` adds a newline). `p` and `pp` use `inspect` and return their argument, which makes them handy for debugging.

**Q13. `require` vs `require_relative` vs `load`?**
`require` loads from `$LOAD_PATH` once, `require_relative` loads relative to the current file once, `load` re-executes the file every call. Rails app code is autoloaded by Zeitwerk instead.

**Q14. What do `?` and `!` at the end of method names mean?**
Conventions: `?` methods answer yes or no; `!` marks the dangerous variant of a method that also exists without it, which mutates the receiver (`sort!`) or raises instead of returning false (`save!`).

**Q15. How did keyword arguments change in Ruby 3.0?**
They were fully separated from positional arguments: a Hash is no longer converted into keywords automatically, so you must write `call(**opts)`. Typos in keyword names raise `ArgumentError`.

**Q16. What does `case/when` use to compare values?**
`when_value === subject`: classes check `is_a?`, ranges check inclusion, regexes match, lambdas are called, and everything else falls back to `==`.

**Q17. How should you represent money?**
`BigDecimal` (built from strings or decimal columns) or integer cents, never `Float`, because binary floats cannot represent most decimal fractions exactly.

**Q18. `and`/`or` vs `&&`/`||`?**
Same logic, much lower precedence for `and`/`or` (below `=`). `x = true and false` assigns `true`. Use `&&`/`||` for expressions.

**Q19. `Integer("12abc")` vs `"12abc".to_i`?**
`Integer()` is strict and raises `ArgumentError` (or returns `nil` with `exception: false`). `to_i` is lenient: it returns `12` here and `0` for `"abc"`.

## Collections and Enumerable (see [02-collections-enumerable.md](02-collections-enumerable.md))

**Q20. `each` vs `map`?**
`each` iterates for side effects and returns the receiver. `map` returns a new array of the block's results.

**Q21. `select` vs `reject` vs `filter_map`?**
`select` keeps elements whose block is truthy, `reject` drops them, and `filter_map` maps and drops `nil`/`false` results in one pass.

**Q22. `reduce` vs `each_with_object`?**
`reduce` threads the block's return value as the accumulator (good for numbers). `each_with_object` passes the same memo object and ignores the block's return value (good for building hashes and arrays).

**Q23. `hash[:key]` vs `hash.fetch(:key)`?**
`[]` returns `nil` or the default for a missing key. `fetch` raises `KeyError` or uses the default value or block you pass, so missing required data fails loudly.

**Q24. What do `group_by`, `partition` and `tally` return?**
`group_by`: a Hash from key to an array of elements. `partition`: two arrays, `[matching, rest]`. `tally`: a Hash from value to count.

**Q25. How do you make your own class work with `map` and `select`?**
`include Enumerable` and define `each` that yields elements; return `enum_for(:each)` when no block is given.

**Q26. What does `lazy` do?**
Turns a chain into a pipeline that processes one element at a time and stops early, so `first(n)` works on infinite or huge sources without building intermediate arrays.

**Q27. When do you use a `Set` instead of an Array?**
For uniqueness and frequent membership checks: `Set#include?` is O(1) on average, `Array#include?` is O(n). Since Ruby 3.2 `Set` needs no `require`.

**Q28. `each_slice` vs `each_cons`?**
`each_slice(2)` makes non-overlapping chunks (`[1, 2], [3, 4]`); `each_cons(2)` makes sliding windows (`[1, 2], [2, 3], [3, 4]`).

**Q29. How do you sort by score descending, then name ascending?**
`users.sort_by { |u| [-u.score, u.name] }`. Arrays compare element by element.

**Q30. What does `dig` do?**
Safely reads nested Hash or Array data: `data.dig(:user, :address, :city)` returns `nil` if any level is missing instead of raising `NoMethodError`.

**Q31. In ActiveRecord, `count` vs `size` vs `length`?**
`count` always runs `SELECT COUNT(*)`, `length` loads all records and counts them, and `size` uses loaded records if present, otherwise runs `COUNT`.

## OOP, modules and mixins (see [03-oop-modules-mixins.md](03-oop-modules-mixins.md))

**Q32. Class vs module?**
A class can be instantiated and subclassed. A module cannot; it serves as a namespace or a mixin of shared behavior.

**Q33. `include` vs `extend` vs `prepend`?**
`include` adds module methods as instance methods, placed after the class in the lookup chain. `extend` adds them to one object's singleton class (class methods when extending a class). `prepend` places the module before the class so its methods wrap the class's methods via `super`.

**Q34. How does Ruby find a method?**
Singleton class, prepended modules, the class, included modules (last included first), then the superclass chain up to `Object`, `Kernel` and `BasicObject`. If nothing is found, it repeats the search for `method_missing`, which raises `NoMethodError` by default.

**Q35. Private vs protected?**
Private: callable only without an explicit receiver, or with `self.` (allowed since Ruby 2.7; private setters always allowed it). Protected: also callable on other instances of the same class family, which suits comparison methods. Ruby's private is per object, not per class.

**Q36. Class variables vs class instance variables?**
`@@var` is one slot shared by the class and all subclasses, so subclasses can overwrite it. A class instance variable (`@var` in the class body) belongs to that class object only and is not inherited.

**Q37. What is a singleton class?**
A hidden class attached to one object that holds methods defined only for that object. Class methods are singleton methods on the class object.

**Q38. Three ways to define a class method?**
`def self.name` in the class body, `class << self` with normal `def`s inside, or `extend` a module whose methods become class methods.

**Q39. Does Ruby support multiple inheritance?**
No. One superclass per class; modules provide shared behavior, and `ancestors` gives a single, predictable lookup order.

**Q40. `super` vs `super()`?**
Bare `super` forwards the current method's arguments; `super()` passes none. Both pass the block implicitly.

**Q41. What is duck typing?**
Relying on what an object can do (`respond_to?(:write)`) rather than its class, so any object with the right methods works. Tests enforce the contract instead of interfaces.

**Q42. `Struct` vs `Data`?**
Both generate accessors, equality and `to_h`. `Struct` is mutable and lets missing arguments default to `nil`; `Data` (Ruby 3.2) is immutable, requires every member and provides `with` for modified copies.

**Q43. What does `ActiveSupport::Concern` provide?**
An `included` block that runs inside the including class (scopes, callbacks, validations), a `class_methods` block for class methods, and dependency resolution between concerns.

**Q44. What must you override for objects to work as Hash keys?**
`eql?` and `hash`, consistently: equal objects must return equal hash values. Usually `alias eql? ==`.

**Q45. What does `Comparable` require, and what does it give you?**
A `<=>` method returning -1, 0, 1 or `nil`. In return you get `<`, `<=`, `==`, `>`, `>=`, `between?` and `clamp`.

## Blocks, procs and lambdas (see [04-blocks-procs-lambdas.md](04-blocks-procs-lambdas.md))

**Q46. Block vs proc vs lambda?**
A block is syntax attached to a method call, not an object. A proc is an object wrapping a block. A lambda is a stricter kind of proc that behaves like an anonymous method.

**Q47. What are the differences between a proc and a lambda?**
Lambdas check arity strictly and `return` exits only the lambda. Procs fill missing arguments with `nil`, drop extras, auto-splat a single array, and `return` exits the method where the proc was defined.

**Q48. What do `yield` and `block_given?` do?**
`yield` calls the block passed to the current method and returns its value; `block_given?` reports whether a block was passed. Yielding without a block raises `LocalJumpError`.

**Q49. `{ }` vs `do ... end`?**
Same block, different precedence: braces bind to the nearest method call, `do ... end` to the outermost one in the expression.

**Q50. What is a closure?**
A block, proc or lambda that captures the variables in scope where it was created and keeps them alive; it sees later changes to them and can modify them.

**Q51. What does `map(&:name)` do?**
`&` calls `Symbol#to_proc`, producing a proc that calls `name` on each element; it is shorthand for `map { |x| x.name }`.

**Q52. When do you capture a block with `&block` instead of using `yield`?**
When you need to store it, pass it to another method or call it later; otherwise `yield` is simpler.

**Q53. `then` vs `tap`?**
`then` yields the object and returns the block's result. `tap` yields the object and returns the object itself.

**Q54. `next` vs `break` inside a block?**
`next` ends the current block call, optionally providing its value. `break` stops the iteration and makes the yielding method return, optionally with a value.

**Q55. What is `it` in Ruby 3.4?**
An implicit name for the single parameter of a block without a parameter list: `users.map { it.email }`. Ruby 2.7 introduced the numbered form `_1`.

**Q56. Why do Rails scopes take lambdas?**
So the query body is evaluated on each call. A value like `1.week.ago` would otherwise be frozen at class-load time.

## Metaprogramming (see [05-metaprogramming.md](05-metaprogramming.md))

**Q57. What is monkey patching, and what are its risks?**
Reopening an existing class to add or change methods. It is global, can collide with other libraries or future Ruby versions, and makes behavior differ from documentation.

**Q58. What are refinements?**
Scoped monkey patches: `refine String do ... end` inside a module, activated with `using`, visible only from that point to the end of the file.

**Q59. `send` vs `public_send`?**
`send` calls any method, including private ones; `public_send` respects visibility. Never pass user input to either without an allow-list.

**Q60. Why must `method_missing` be paired with `respond_to_missing?`?**
So `respond_to?` and `method(:name)` report the dynamic methods truthfully. Also call `super` for unhandled names, or typos silently return `nil`.

**Q61. `define_method` vs `def`?**
`define_method` takes a computed name and a block that is a closure over surrounding locals; `def` is a scope gate with a fixed name. Bare `super` is not allowed inside `define_method` bodies.

**Q62. `class_eval` vs `instance_eval`?**
`class_eval` sets `self` to the class and `def` creates instance methods. `instance_eval` sets `self` to the receiver and `def` creates singleton methods. DSLs use `instance_eval`.

**Q63. How does `has_many` work?**
It is a class method run while the class body executes. It records a reflection, defines association methods in a generated module (so you can override them and call `super`), and registers callbacks for options such as `dependent:`.

**Q64. Which hooks does Ruby provide?**
`included`, `extended` and `prepended` on modules; `inherited` and `method_added` on classes; `const_missing` for unknown constants. Call `super` inside them.

**Q65. Why is `params[:type].constantize` dangerous?**
The client chooses which class your code loads and instantiates. Map allowed names to classes explicitly instead.

**Q66. How do you find where a method is defined?**
`obj.method(:name).owner` for the defining module or class, `.source_location` for file and line, and `Klass.ancestors` for the lookup order.

## Internals, errors and concurrency (see [06-advanced-internals.md](06-advanced-internals.md))

**Q67. What is the GVL, and does Ruby have real threads?**
CRuby threads are real OS threads, but the Global VM Lock lets one run Ruby code at a time per process. It is released during blocking I/O, so threads speed up I/O-bound work but not CPU-bound work.

**Q68. How do Rails apps get parallelism despite the GVL?**
Multiple processes: Puma cluster mode forks workers, each with its own GVL and thread pool; Sidekiq scales by running more processes.

**Q69. Why is class-level mutable state dangerous in Rails?**
Every Puma or Sidekiq thread in the process shares it, so concurrent requests race or leak data (for example the current tenant). Use `CurrentAttributes`, locals or per-request objects.

**Q70. What does a bare `rescue` catch?**
`StandardError` and its subclasses, not every `Exception`.

**Q71. Why should you never `rescue Exception`?**
It also catches `Interrupt`, `SignalException`, `SystemExit`, `NoMemoryError` and `ScriptError`, so processes ignore shutdown signals and real bugs disappear.

**Q72. `else` vs `ensure` in a `begin` block?**
`else` runs only when no exception was raised; `ensure` runs no matter what, including after `return`.

**Q73. How does `retry` work, and what is the classic bug?**
Inside `rescue`, `retry` re-runs the `begin` block. Without a counter defined outside the block (and backoff), it can loop forever.

**Q74. How do you define custom exceptions?**
Inherit from `StandardError`, usually through one base class per library (`PriceSync::Error`) with specific subclasses that carry data. Raising inside `rescue` keeps the original as `cause`.

**Q75. What are `catch` and `throw` for?**
Non-exception control flow that unwinds to a matching `catch` with a value, for example to escape nested loops. Rails halts callback chains with `throw(:abort)`.

**Q76. What is wrong with `@flag ||= compute` when `compute` returns `false`?**
`||=` assigns whenever the value is falsy, so `false` and `nil` are never cached and `compute` runs on every call. Use `return @flag if defined?(@flag)`.

**Q77. How does CRuby's garbage collector work?**
Mark and sweep from roots, generational (minor GCs scan young objects, major GCs scan everything), incremental marking to shorten pauses, and optional compaction.

**Q78. What are fibers and Ractors?**
Fibers are cooperative coroutines on one thread, used by enumerators and, with the fiber scheduler, for async I/O. Ractors (Ruby 3.0, experimental) run Ruby code in parallel with their own GVL and share only immutable objects.

**Q79. What is `deconstruct_keys` for?**
It lets objects take part in hash patterns: `case obj in { status: :paid }` calls `obj.deconstruct_keys([:status])`. `deconstruct` does the same for array patterns.

**Q80. What is YJIT?**
CRuby's built-in JIT compiler (3.1, production-ready 3.2), enabled with `--yjit`. Rails 7.2 enables it by default on supported Ruby versions; it trades extra memory for faster Ruby code.

## Ecosystem and tooling (see [06-advanced-internals.md](06-advanced-internals.md) and [Rails testing](../../05-ruby-on-rails/13-testing-rspec.md))

**Q81. What does `gem "rails", "~> 7.1"` allow?**
Any version `>= 7.1` and `< 8.0`. `~> 7.1.3` would allow `>= 7.1.3` and `< 7.2`.

**Q82. Why commit `Gemfile.lock`, and why use `bundle exec`?**
The lockfile pins exact versions so every machine and deploy installs the same dependency tree. `bundle exec` runs a command with those locked versions instead of globally installed ones.

**Q83. How do you debug Ruby code?**
Read the backtrace from your own frames, add a `debugger` breakpoint (the `debug` gem ships with Ruby 3.1+) or `binding.irb`, inspect with `p`/`pp`, check logs, and use `method(:x).source_location` to find the code that actually runs.

**Q84. What tools keep Ruby code consistent?**
RuboCop (configurable linter and formatter) or Standard (zero-config RuboCop setup); Rails 7.2+ generates `rubocop-rails-omakase`. Brakeman scans Rails apps for security issues.

**Q85. RSpec vs Minitest?**
Minitest ships with Ruby and is Rails' default: plain classes and assertions. RSpec is a DSL with `describe`, `it`, `let` and matchers, very common in Rails teams.

**Q86. How do you manage Ruby versions per project?**
A `.ruby-version` file read by a version manager such as rbenv, asdf, chruby or rvm, plus the Ruby version in the Dockerfile or CI configuration.

**Q87. What is Rake?**
Ruby's task runner. Rails exposes tasks such as `db:migrate` through `bin/rails`, and you can write your own in `lib/tasks/*.rake` for maintenance jobs like backfills.

**Q88. `length` vs `bytesize` on a string?**
`length` counts characters; `bytesize` counts bytes in the string's encoding (UTF-8 by default), so `"é".length` is 1 but `"é".bytesize` is 2.

## What's the output and spot the bug (see all topic files)

**Q89. What's the output?**
```ruby
h = Hash.new([])
h[:a] << 1
h[:b] << 2
p h
p h[:c]
```
`{}` then `[1, 2]`. The default array is one shared object that `<<` mutates, and no key is ever stored. Use `Hash.new { |hash, key| hash[key] = [] }`.

**Q90. How many times is "checking" printed?**
```ruby
def enabled?
  puts "checking"
  false
end

def cached_enabled?
  @cached ||= enabled?
end

cached_enabled?
cached_enabled?
```
Twice. `||=` re-evaluates whenever the stored value is `false` or `nil`, so the falsy result is never cached.

**Q91. What's the output?**
```ruby
a = "hello"
b = a
b << " world"
c = a
c += "!"
puts a
puts c
```
`hello world` then `hello world!`. `b` and `a` reference the same object, so `<<` mutates both; `+=` builds a new string and rebinds only `c`.

**Q92. What's the output?**
```ruby
def run_proc
  pr = proc { return 10 }
  pr.call
  20
end

def run_lambda
  la = -> { return 10 }
  la.call
  20
end

p run_proc
p run_lambda
```
`10` then `20`. `return` in a proc exits the enclosing method; in a lambda it exits only the lambda.

**Q93. What's the output?**
```ruby
p 7 / 2
p(-7 / 2)
p(-7 % 3)
p 7 / 2.0
```
`3`, `-4`, `2`, `3.5`. Integer division floors toward negative infinity and the remainder takes the sign of the divisor.

**Q94. What's the output?**
```ruby
require "bigdecimal"
p 0.1 + 0.2
p 0.1 + 0.2 == 0.3
p BigDecimal("0.1") + BigDecimal("0.2") == BigDecimal("0.3")
```
`0.30000000000000004`, `false`, `true`. Binary floats cannot represent 0.1 exactly; BigDecimal uses decimal arithmetic.

**Q95. What's the output?**
```ruby
x = puts "hi"
y = p "hi"
p x
p y
```
`hi`, `"hi"`, `nil`, `"hi"`. `puts` returns `nil`; `p` prints the inspected value and returns its argument.

**Q96. What's the output?**
```ruby
p :ruby.object_id == :ruby.object_id
p "ruby".object_id == "ruby".object_id
p "ruby".equal?("ruby")
p "ruby" == "ruby"
```
`true`, `false`, `false`, `true`. Each symbol name is a single object; each string literal evaluation creates a new object. With `# frozen_string_literal: true`, identical literals are deduplicated and the middle two typically become `true`.

**Q97. What's the output?**
```ruby
# frozen_string_literal: true
s = "abc"
t = s.upcase
u = s.dup << "d"
p t, u, s.frozen?, u.frozen?
s << "x"
```
`"ABC"`, `"abcd"`, `true`, `false`, then `FrozenError` on the last line. Non-mutating methods and `dup` copies are fine; mutating the frozen literal is not.

**Q98. What's the output?**
```ruby
module A; def hi; "A > #{super}"; end; end
module B; def hi; "B > #{super}"; end; end
class Base; def hi; "Base"; end; end

class Child < Base
  include A
  prepend B
  def hi; "Child > #{super}"; end
end

puts Child.new.hi
p Child.ancestors.take(4)
```
`B > Child > A > Base` and `[B, Child, A, Base]`. Prepended modules come before the class, included modules after it, then the superclass.

**Q99. What's the output?**
```ruby
puts [1, 2, 3].map do |x|
  x * 2
end
```
`#<Enumerator: [1, 2, 3]:map>`. `do ... end` binds to `puts`, so `map` receives no block and returns an Enumerator. Braces would print 2, 4 and 6.

**Q100. What's the output?**
```ruby
procs = []
for i in 1..3
  procs << -> { i }
end
p procs.map(&:call)
p [1, 2, 3].map { |j| -> { j } }.map(&:call)
```
`[3, 3, 3]` then `[1, 2, 3]`. `for` shares one variable across iterations; each block call gets its own parameter.

**Q101. What's the output?**
```ruby
x = true and false
y = true && false
p x, y
```
`true` then `false`. `and` binds more loosely than `=`, so the first line is `(x = true) and false`.

**Q102. What's the output?**
```ruby
p [1, 2, 2, 3].uniq!
p [1, 2, 3].uniq!
p "x".strip!
```
`[1, 2, 3]`, `nil`, `nil`. Many bang methods return `nil` when nothing changed, which is why chaining them is a bug.

**Q103. What's the output?**
```ruby
grid = Array.new(2, [])
grid[0] << :x
p grid
```
`[[:x], [:x]]`. Both slots reference the same array; `Array.new(2) { [] }` creates one array per slot.

**Q104. What's the output?**
```ruby
nums = [1, 2, 3, 4]
nums.each { |n| nums.delete(n) }
p nums
```
`[2, 4]`. Deleting during iteration shifts elements under the iterator, so every other element is skipped. Use `reject!` or `delete_if`.

**Q105. What's the output?**
```ruby
if false
  ghost = 1
end
p ghost
p defined?(ghost)
```
`nil` then `"local-variable"`. The parser saw the assignment, so `ghost` exists as a local even though the line never ran.

**Q106. What's the output?**
```ruby
p [].all?
p [].any?
p [].sum
p [].max
```
`true`, `false`, `0`, `nil`. `all?` is vacuously true on an empty collection.

**Q107. Spot the bug.**
```ruby
def average(prices)
  prices.sum / prices.size
end
average([1, 2]) # => 1
```
Integer division truncates the result (and an empty array raises `ZeroDivisionError`). Use `prices.sum.fdiv(prices.size)` with an empty guard, or BigDecimal for money.

**Q108. Spot the bug.**
```ruby
class Settings
  def initialize(values)
    @values = values
  end

  def method_missing(name, *args)
    @values[name]
  end
end
```
No `super` fallback, so typos such as `settings.timout` return `nil` silently, and no `respond_to_missing?`, so `respond_to?` lies. Handle known keys, call `super` otherwise, and define `respond_to_missing?`.

**Q109. Spot the bug.**
```ruby
class CurrentTenant
  class << self
    attr_accessor :organization
  end
end
# before_action { CurrentTenant.organization = current_user.organization }
```
The class-level accessor is shared by every thread in the Puma process, so concurrent requests overwrite each other's tenant and can leak data across organizations. Use `ActiveSupport::CurrentAttributes`, which isolates and resets values per request.
