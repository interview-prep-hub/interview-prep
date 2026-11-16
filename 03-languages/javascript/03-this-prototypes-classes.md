# this, Prototypes and Classes

> **Tier:** T1 (know cold) · **Related:** [Scope and closures](02-scope-closures-hoisting.md), [ES6+ features](05-es6-plus.md), [Implement from scratch](07-implement-from-scratch.md), [OOP](../../01-cs-fundamentals/02-oop.md), [Ruby OOP, modules and mixins](../ruby/03-oop-modules-mixins.md)
> **On your resume:** event handlers and components in your React and React Native work, classes in TypeScript backends, and the contrast between JavaScript's prototype model and the Ruby classes and modules you use in Rails.

## Quick revision
- `this` is decided by how a function is called, not where it is defined (arrow functions are the exception).
- Rules: `new` -> the new object; `call`/`apply`/`bind` -> the given object; `obj.method()` -> `obj`; plain call -> `undefined` in strict mode (`globalThis` in sloppy mode).
- Precedence: `new` > explicit (`bind`/`call`/`apply`) > implicit (`obj.`) > default.
- Arrow functions have no own `this`: they use the surrounding scope's `this`, and `call`/`bind` cannot change it.
- `call(thisArg, a, b)` and `apply(thisArg, [a, b])` invoke immediately; `bind(thisArg, a)` returns a new function with `this` fixed for good (a second `bind` does nothing; `new` still overrides it).
- Extracting a method (`const f = obj.method`, `setTimeout(obj.method)`, `onClick={this.handle}`) loses `this`. Fix with an arrow wrapper, `bind` or an arrow class field.
- Every object has a hidden `[[Prototype]]` link (`Object.getPrototypeOf(obj)`; `__proto__` is the legacy accessor). Functions have a `.prototype` object that becomes the `[[Prototype]]` of instances created with `new`.
- Lookup: own properties, then up the chain to `Object.prototype`, then `null` -> `undefined`. Assignment creates an own property that shadows the inherited one.
- `new F()`: create an empty object, link it to `F.prototype`, run `F` with `this` = that object, return it (unless `F` returns another object).
- Classes are mostly syntax over prototypes (methods live on the prototype), but they must be called with `new`, have strict bodies, sit in the TDZ until defined, and have non-enumerable methods.
- `#private` members are enforced by the engine; `static` members belong to the class; derived constructors must call `super()` before touching `this`.
- `obj instanceof C` walks `obj`'s prototype chain looking for `C.prototype`. It fails across realms (iframes); prefer `Array.isArray`.
- Single inheritance only; share behavior with `Object.assign(Class.prototype, mixin)` or subclass factories `(Base) => class extends Base {}`.
- `Object.freeze` (no add, delete or change), `Object.seal` (no add or delete), `Object.preventExtensions` (no add). All shallow.
- Descriptors: `value`, `writable`, `enumerable`, `configurable` (or `get`/`set`). `Object.defineProperty` defaults every flag to `false`.

## 1. How `this` is determined

`this` is a hidden parameter filled in at call time. Ask "how was this function called?":

| Call form | `this` | Example |
|---|---|---|
| `new Fn()` | the newly created object | `new User('Sara')` |
| `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)()` | `obj` | `greet.call(admin)` |
| `obj.fn()` | the object before the dot at call time | `user.greet()` |
| `fn()` | `undefined` in strict mode, `globalThis` in sloppy mode | `const f = user.greet; f()` |
| arrow function | inherited from the enclosing scope, never rebound | `() => this.count` |

Top-level `this` is `window` in a classic browser script, `undefined` in an ES module, and `module.exports` in a Node CommonJS file.

```js
'use strict';

function whoAmI() {
  return this;
}
const admin = { role: 'admin', whoAmI };

whoAmI();                    // undefined (default binding, strict)
admin.whoAmI();              // admin (implicit)
whoAmI.call({ role: 'x' });  // { role: 'x' } (explicit)

const bound = whoAmI.bind(admin);
bound();                     // admin
bound.call({ role: 'y' });   // still admin: bind is permanent
```

In sloppy mode, `call(null)` makes `this` the global object and primitives get boxed; in strict mode `this` is exactly what you pass. Class bodies and ES modules are always strict.

### call vs apply vs bind

| | Invokes now? | Arguments | Returns |
|---|---|---|---|
| `fn.call(thisArg, a, b)` | yes | one by one | the function's result |
| `fn.apply(thisArg, [a, b])` | yes | as an array | the function's result |
| `fn.bind(thisArg, a)` | no | pre-filled, more can follow later | a new bound function |

```js
function formatPrice(currency, decimals) {
  return `${currency} ${this.amount.toFixed(decimals)}`;
}
const item = { amount: 1250 };

formatPrice.call(item, 'PKR', 0);     // 'PKR 1250'
formatPrice.apply(item, ['USD', 2]);  // 'USD 1250.00'
const toPkr = formatPrice.bind(item, 'PKR');
toPkr(1);                             // 'PKR 1250.0'
```

`apply` was the old way to spread an array into arguments (`Math.max.apply(null, nums)`); spread syntax (`Math.max(...nums)`) replaced it.

### Losing `this` in callbacks

```js
'use strict';
const user = {
  name: 'Ali',
  greet() {
    return `Hi ${this.name}`;
  },
};

const greet = user.greet;   // copies the function, not its object
// greet();                 // TypeError: Cannot read properties of undefined (reading 'name')
// setTimeout(user.greet, 0) fails the same way: the timer calls it as a plain function

setTimeout(() => console.log(user.greet()), 0); // fix 1: arrow wrapper
setTimeout(user.greet.bind(user), 0);           // fix 2: bind
```

Class methods passed as callbacks behave the same:

```js
class SearchBox {
  query = '';

  // Arrow class field: one function per instance, `this` captured lexically
  handleInput = (event) => {
    this.query = event.target.value;
  };

  // Regular method: lives on the prototype, `this` depends on the call
  submit() {
    return this.query;
  }
}

const box = new SearchBox();
const { handleInput, submit } = box;
handleInput({ target: { value: 'pikachu' } }); // works: this is box
// submit();                                    // TypeError: this is undefined
```

React class components hit exactly this with `onClick={this.handleClick}`, which is why old code has `this.handleClick = this.handleClick.bind(this)` in constructors or arrow class fields. Function components with hooks avoid `this` altogether.

### Arrow functions and `this`

```js
const counter = {
  count: 0,
  incArrow: () => {
    // `this` here is the outer scope's this (undefined in a module), not counter
  },
  incLater() {
    setTimeout(() => {
      this.count++; // the arrow inherits incLater's this, which is counter
    }, 0);
  },
};
```

Use arrows for callbacks inside methods. Do not use them as object or prototype methods that need the object, or as DOM handlers that rely on `this` being the element (`event.currentTarget` is clearer anyway).

## 2. Prototypes and the prototype chain

Objects inherit directly from other objects. Each object has an internal `[[Prototype]]` link to another object (or `null`). Reading a property the object does not own follows that link, again and again, until the property is found or the chain ends at `null`.

```js
const animal = {
  eats: true,
  describe() {
    return `${this.name} eats: ${this.eats}`;
  },
};

const rabbit = Object.create(animal);     // rabbit's [[Prototype]] is animal
rabbit.name = 'Bugs';

rabbit.describe();                        // 'Bugs eats: true' (found on animal, this = rabbit)
Object.getPrototypeOf(rabbit) === animal; // true
Object.hasOwn(rabbit, 'eats');            // false: inherited
'eats' in rabbit;                         // true: `in` searches the whole chain

rabbit.eats = false;                      // own property shadows animal.eats
animal.eats;                              // still true
```

```text
[1, 2, 3] -> Array.prototype (map, filter, push) -> Object.prototype (hasOwnProperty, toString) -> null
```

### `__proto__` vs `prototype`

| | `obj.__proto__` | `Fn.prototype` |
|---|---|---|
| Exists on | every object (legacy accessor inherited from `Object.prototype`) | regular functions and classes (not arrows or methods) |
| Means | the object this object inherits from | the object future instances of `new Fn()` inherit from |
| Modern API | `Object.getPrototypeOf` / `Object.setPrototypeOf` | the same property |

They meet here: `new User().__proto__ === User.prototype`.

```js
function User(name) {
  this.name = name;                     // own property, one per instance
}
User.prototype.greet = function () {    // shared by all instances, stored once
  return `Hi, I am ${this.name}`;
};

const u = new User('Sara');
u.greet();                                   // 'Hi, I am Sara'
Object.getPrototypeOf(u) === User.prototype; // true
User.prototype.constructor === User;         // true
```

- `Object.create(null)` makes an object with no prototype (no inherited `toString`, `constructor` or `__proto__`), a safe dictionary for arbitrary keys. A `Map` is usually better still.
- `for...in` includes inherited enumerable properties; `Object.keys` returns only own enumerable keys.
- Changing a prototype after creation (`Object.setPrototypeOf`) is slow; set it at creation time.
- Never add methods to built-in prototypes in application code: they can clash with future standard methods.

## 3. What `new` does

`new User('Sara')` performs four steps:
1. Create a new empty object.
2. Set its `[[Prototype]]` to `User.prototype`.
3. Call `User` with `this` bound to the new object and the given arguments.
4. If `User` returns an object, use that object; otherwise (no return, or a primitive) return the new object.

```js
function myNew(Constructor, ...args) {
  const obj = Object.create(Constructor.prototype);            // steps 1 and 2
  const result = Constructor.apply(obj, args);                 // step 3
  const isObject =
    result !== null && (typeof result === 'object' || typeof result === 'function');
  return isObject ? result : obj;                              // step 4
}

const u2 = myNew(User, 'Zara');
u2.greet();          // 'Hi, I am Zara'
u2 instanceof User;  // true
```

Arrow functions and methods cannot be used with `new` (`TypeError: x is not a constructor`); they have no `prototype` property and no construct behavior. Calling a constructor function without `new` in sloppy mode writes properties onto the global object, one reason classes refuse to run without `new`.

## 4. ES6 classes

```js
class Account {
  static #nextId = 1;          // private static field
  static currency = 'PKR';     // public static field

  #balanceCents = 0;           // private instance field
  status = 'active';           // public field: own property on each instance

  constructor(owner) {
    this.id = Account.#nextId++;
    this.owner = owner;
  }

  deposit(cents) {             // lives on Account.prototype
    if (cents <= 0) throw new RangeError('Deposit must be positive');
    this.#balanceCents += cents;
    return this;               // enables chaining
  }

  get balance() {              // getter: account.balance, no parentheses
    return this.#balanceCents / 100;
  }

  static isAccount(value) {    // brand check with `#field in obj` (ES2022)
    return #balanceCents in value;
  }
}

class SavingsAccount extends Account {
  constructor(owner, rate) {
    super(owner);              // must run before `this` is used
    this.rate = rate;
  }

  deposit(cents) {
    super.deposit(cents);      // parent implementation
    console.log(`Saved ${cents} cents`);
    return this;
  }
}

const acc = new SavingsAccount('Sara', 0.05).deposit(5000);
acc.balance;                   // 50
// acc.#balanceCents           // SyntaxError outside the class body
Account.isAccount(acc);        // true
```

A class is a function (`typeof Account === 'function'`) and its methods live on `Account.prototype`, but classes are more than sugar:

| Difference from a constructor function | Effect |
|---|---|
| must be called with `new` | `Account()` throws `TypeError: Class constructor Account cannot be invoked without 'new'` |
| strict-mode body | default `this` inside methods is `undefined` |
| TDZ | using the class before its declaration throws a ReferenceError |
| non-enumerable methods | `for...in` and `Object.keys` skip them |
| `super`, fields, `#private`, static blocks | no direct ES5 equivalent |

- Fields are assigned to each instance before the constructor body runs (base class) or right after `super()` returns (derived class), so a base constructor cannot see a subclass's fields.
- Using `this` before `super()` in a derived constructor throws `ReferenceError: Must call super constructor in derived class...`.
- Arrow function fields bind `this` but create one function per instance and are not on the prototype, so a subclass cannot call them with `super.handle()`.
- `#private` is enforced at runtime and invisible to `Object.keys`, `JSON.stringify` and reflection; TypeScript's `private` exists only at compile time.

Custom errors are the most common class you will write:

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

try {
  throw new ValidationError('Email is invalid', 'email');
} catch (err) {
  if (err instanceof ValidationError) console.log(err.field); // 'email'
}
```

## 5. instanceof internals

`obj instanceof C` asks whether `C.prototype` appears anywhere in `obj`'s prototype chain:

```js
function myInstanceof(obj, Constructor) {
  if (obj === null || (typeof obj !== 'object' && typeof obj !== 'function')) return false;
  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === Constructor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}

myInstanceof(acc, Account);  // true (SavingsAccount.prototype -> Account.prototype)
myInstanceof(5, Number);     // false: primitives are not instances
```

- It checks the chain, not how the object was built: reassigning `C.prototype` breaks checks for existing objects.
- It fails across realms (an array from an iframe has a different `Array.prototype`) and with two copies of the same library in `node_modules`. Use `Array.isArray` or duck typing.
- A class can customize it with `static [Symbol.hasInstance](value)`.
- Duck typing checks capabilities instead: promises detect thenables with `typeof value.then === 'function'`.

## 6. Mixins and composition

Classes have single inheritance. To share behavior across unrelated classes, use mixins:

```js
// 1. Copy methods onto a prototype
const Serializable = {
  serialize() {
    return JSON.stringify(this);
  },
};
class Invoice {
  constructor(total) { this.total = total; }
}
Object.assign(Invoice.prototype, Serializable);
new Invoice(500).serialize(); // '{"total":500}'

// 2. Subclass factories: stackable and compatible with super
const Timestamped = (Base) =>
  class extends Base {
    constructor(...args) {
      super(...args);
      this.createdAt = new Date().toISOString();
    }
  };
const SoftDeletable = (Base) =>
  class extends Base {
    softDelete() { this.deletedAt = new Date().toISOString(); }
  };

class Model {
  constructor(id) { this.id = id; }
}
class Booking extends SoftDeletable(Timestamped(Model)) {}
new Booking(7).softDelete();
```

This is Ruby's `include Module` done by hand: Ruby inserts the module into the ancestors chain, and a subclass factory inserts a class into the prototype chain. In practice, prefer composition (objects that hold helpers, or plain functions) over deep hierarchies; React itself moved from mixins to hooks for this reason.

## 7. Property descriptors, freeze and seal

Every property has a descriptor. Data properties have `value`, `writable`, `enumerable`, `configurable`; accessor properties have `get`, `set`, `enumerable`, `configurable`.

```js
const config = {};
Object.defineProperty(config, 'apiVersion', {
  value: 'v2',
  writable: false,     // cannot be reassigned
  enumerable: false,   // hidden from Object.keys and JSON.stringify
  configurable: false, // cannot be deleted or redefined
});
// Flags you omit default to false with defineProperty (true for normal assignment).
Object.getOwnPropertyDescriptor(config, 'apiVersion');
// { value: 'v2', writable: false, enumerable: false, configurable: false }

const temperature = {
  celsius: 25,
  get fahrenheit() { return this.celsius * 1.8 + 32; },
  set fahrenheit(f) { this.celsius = (f - 32) / 1.8; },
};
temperature.fahrenheit = 212; // setter runs
temperature.celsius;          // 100
```

| | Add | Delete | Change values | Reconfigure |
|---|---|---|---|---|
| `Object.preventExtensions` | no | yes | yes | yes |
| `Object.seal` | no | no | yes | no |
| `Object.freeze` | no | no | no | no |

- All three are **shallow**: nested objects stay mutable unless frozen recursively.
- Violations throw a TypeError in strict mode and fail silently in sloppy mode.
- `const` stops reassignment of the variable; `Object.freeze` stops changes to the object. Different problems.
- Check state with `Object.isFrozen`, `Object.isSealed`, `Object.isExtensible`.

```js
function deepFreeze(obj) {
  for (const value of Object.values(obj)) {
    if (value && typeof value === 'object' && !Object.isFrozen(value)) deepFreeze(value);
  }
  return Object.freeze(obj);
}
const ROLES = deepFreeze({ admin: { canDelete: true } });
```

## Common pitfalls
- Passing methods as callbacks (`setTimeout(obj.method)`, `items.map(obj.method)`, `onClick={this.handle}`) and losing `this`.
- Arrow functions as object or prototype methods, then wondering why `this` is not the object.
- Using `this` before `super()` in a derived constructor.
- Putting arrays or objects on a constructor's `prototype`: every instance shares them, so one push is visible everywhere.
- Assuming `Object.freeze` is deep, or that `const` makes an object immutable.
- `instanceof` across iframes or across duplicate library copies.
- Treating TypeScript `private` as runtime privacy; use `#private` when it must be enforced.
- Thousands of instances with arrow-function fields where one prototype method would do.

## How to talk about this in an interview
- Lead with the rule ("`this` depends on the call site, except for arrow functions"), then walk the four rules in precedence order with one-line examples. Most questions here are "predict the output".
- Use Ruby as a contrast: Ruby has classes, modules and an ancestors chain; JavaScript has objects linked to objects, with classes as friendlier syntax over that chain. Your Rails work makes this comparison natural.
  > **Fill in:** where you used classes in TypeScript or JavaScript at work (service classes, custom errors, API clients) versus plain functions and hooks.
- If you mostly write function components and hooks, say so honestly, and show you know why class components needed `bind` in the constructor; that is the classic follow-up.
  > **Fill in:** did you maintain any class components in React on Rails or older React code? What did you change?

## Interview questions

**Q1. How is `this` determined in JavaScript?**
By the call site, through four rules in precedence order: `new` binds the new object; `call`/`apply`/`bind` bind the given object; `obj.method()` binds `obj`; a plain call binds `undefined` in strict mode or `globalThis` in sloppy mode. Arrow functions skip the rules and use the enclosing scope's `this`.
*Follow-up:* What is `this` at the top of an ES module? `undefined`.

**Q2. How does `this` differ between arrow functions and regular functions?**
A regular function gets `this` from how it is called. An arrow function has no own `this` (nor `arguments`, `super` or `new.target`): it captures `this` lexically when created, and `call`, `apply` and `bind` cannot change it. That makes arrows ideal for callbacks inside methods and wrong for object methods.
*Follow-up:* Can you `new` an arrow function? No, it has no `prototype` and throws a TypeError.

**Q3. What is the difference between `call`, `apply` and `bind`?**
`call` and `apply` invoke the function immediately with a chosen `this`; `call` takes arguments individually, `apply` as an array. `bind` does not invoke; it returns a new function with `this` (and optionally leading arguments) fixed.
*Follow-up:* What if you `bind` a bound function again? The first binding wins; only `new` can override a bound `this`.

**Q4. Why does `setTimeout(user.greet, 1000)` lose `this`, and how do you fix it?**
`user.greet` evaluates to the bare function with no object attached, and the timer calls it as a plain function, so `this` is `undefined` (strict) or `window`. Fix with `setTimeout(() => user.greet(), 1000)`, `user.greet.bind(user)`, or an arrow class field.
*Follow-up:* Where else does this bite? Passing a class method as a React `onClick` without binding.

**Q5. Explain the prototype chain and prototypal inheritance.**
Every object has a hidden `[[Prototype]]` link. Property lookup checks the object, then its prototype, then that prototype's prototype, up to `Object.prototype` and then `null`. Inheritance means linking objects so they share methods through the chain instead of copying them.
*Follow-up:* What happens when you assign to an inherited property? An own property is created that shadows it, unless the prototype has a setter or a read-only property.

**Q6. What is the difference between `__proto__` and `prototype`?**
`obj.__proto__` (properly `Object.getPrototypeOf(obj)`) is the object `obj` inherits from. `Fn.prototype` is a property on constructor functions and classes: the object that becomes the `[[Prototype]]` of instances made with `new Fn()`. So `new Fn().__proto__ === Fn.prototype`.
*Follow-up:* Do arrow functions have a `prototype`? No.

**Q7. What does the `new` keyword do?**
Creates an empty object, links its `[[Prototype]]` to `Constructor.prototype`, runs the constructor with `this` set to that object, and returns it unless the constructor returns a different object (a returned primitive is ignored).
*Follow-up:* Implement it: `Object.create(C.prototype)`, then `C.apply(obj, args)`, then return the result if it is an object, else `obj`.

**Q8. Are ES6 classes just syntactic sugar?**
Mostly: a class is a function and its methods live on its `prototype`. But classes add real semantics: they require `new`, have strict bodies, are in the TDZ until declared, have non-enumerable methods, and support `super`, fields, `#private` members and static blocks.
*Follow-up:* What does `typeof SomeClass` return? `'function'`.

**Q9. How do private class fields compare to closures and TypeScript `private`?**
`#field` is enforced by the engine: using it outside the class body is a SyntaxError, and it never shows up in `Object.keys` or `JSON.stringify`. Closures also give real privacy but create functions per instance. TypeScript `private` disappears at compile time, so the property is readable at runtime.
*Follow-up:* How do you check whether an object has a private field? `#field in obj` (ES2022).

**Q10. How does `instanceof` work, and when does it fail?**
It walks the left operand's prototype chain looking for the right operand's `prototype`. It fails across realms (iframes, VM contexts), after `prototype` is reassigned, with duplicate copies of a library, and for primitives. Use `Array.isArray` or duck typing there.
*Follow-up:* Can a class customize it? Yes, with `static [Symbol.hasInstance]`.

**Q11. What is the difference between `Object.freeze`, `Object.seal` and `const`?**
`const` prevents reassigning the variable but not mutating the object. `seal` blocks adding and deleting properties but allows changing values. `freeze` blocks adding, deleting and changing. `seal` and `freeze` are shallow, and violations only throw in strict mode.
*Follow-up:* How do you deep freeze? Freeze every nested object recursively.

**Q12. What happens when you call a bound function with `new`?**
`new` wins over `bind`: the bound `this` is ignored and a new object is created using the original function's `prototype`, while pre-bound arguments are still prepended. That is why `new` tops the precedence list.
*Follow-up:* Does a bound function have its own `prototype` property? No; `new` uses the target function's.

**Q13. What does `Object.create(null)` create, and why use it?**
An object with no prototype, so no inherited keys like `toString`, `constructor` or `__proto__`. It is a safe dictionary for arbitrary keys; a `Map` is often the better choice today.
*Follow-up:* What is the risk of a normal object as a dictionary? Keys like `'__proto__'` collide with inherited properties and can enable prototype pollution.

**Q14. How do you share behavior between classes without multiple inheritance?**
Mixins: copy methods onto a prototype with `Object.assign`, or stack subclass factories `(Base) => class extends Base { ... }`. Plain composition (pass helpers in, use functions or hooks) is often simpler.
*Follow-up:* What is the Ruby equivalent? `include` a module, which inserts it into the ancestors chain.

**Q15. What does this log (in an ES module), and why?**
```js
const obj = {
  name: 'obj',
  regular() { return this.name; },
  arrow: () => this?.name,
};
const fn = obj.regular;
console.log(obj.regular(), obj.arrow(), fn.call({ name: 'other' }));
```
`'obj'`, `undefined`, `'other'`. `obj.regular()` uses implicit binding. The arrow captured the module's top-level `this`, which is `undefined`. `call` sets `this` explicitly on the extracted function.
*Follow-up:* What would plain `fn()` do? Throw a TypeError, because modules are strict and `this` is `undefined`.

**Q16. What are property descriptors, and how do you make a read-only property?**
Metadata on every property: `value`, `writable`, `enumerable`, `configurable`, or `get`/`set` for accessors. Use `Object.defineProperty(obj, 'id', { value: 1, writable: false })`, or a getter without a setter. Omitted flags default to `false`.
*Follow-up:* Why is such a property missing from `JSON.stringify` output? `enumerable` defaulted to `false`.

**Q17. Why must a derived class call `super()` before using `this`?**
In a derived class, the parent constructor creates and initializes the object; until `super()` returns there is no `this`, so touching it throws a ReferenceError. Derived class fields are initialized right after `super()` returns.
*Follow-up:* Does a subclass need a constructor at all? No; the default one forwards all arguments to `super`.

## Further reading
- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [MDN: Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
