# OOP Fundamentals in PHP

Object-Oriented Programming (OOP) is the dominant way modern PHP — and all of Laravel — is written. Every controller, model, service, and middleware you will ever touch is a *class*. If you understand classes deeply, the rest of Laravel stops looking like magic and starts looking like ordinary objects calling methods on each other. This module builds that foundation from zero.

> **Jargon up front:** *OOP* is a programming style where you bundle **data** (variables) and **behavior** (functions that act on that data) together into units called **objects**. A *class* is the blueprint; an *object* (or *instance*) is a concrete thing built from that blueprint.

---

## **What you'll learn**

- What classes and objects are, and how to create (instantiate) them — including the modern `new (Foo::class)` style.
- How `$this`, properties, and methods work together, and what visibility (`public` / `protected` / `private`) really protects you from.
- Constructors, **constructor property promotion**, destructors, **typed properties**, and **`readonly` properties & classes** (PHP 8.1–8.4).
- Class constants (with visibility), **static** properties/methods, and the crucial difference between `self`, `static`, and `parent` (late static binding).
- How PHP compares objects (`==` vs `===`) and copies them (cloning, `__clone`, shallow vs deep).
- Object **handle/reference semantics** when passing objects to functions — the #1 source of "why did my object change?" bugs.
- `instanceof`, proper **encapsulation** with getters/setters, and working with `stdClass` / casting arrays to objects.

---

## 1. Why OOP? The problem it solves

Before OOP, a program was a pile of loose functions and global variables. Imagine modeling a bank account with plain arrays:

```php
<?php
$account = ['owner' => 'Ada', 'balance' => 100];

function deposit(array &$account, int $amount): void {
    $account['balance'] += $amount;
}

deposit($account, 50);
echo $account['balance']; // Output: 150
```

This works, but nothing stops a careless teammate from writing `$account['balance'] = -99999;` directly, or misspelling `'balnace'`. The data and the rules that protect it live in different places. OOP fixes this by **putting the data and its rules in one tamper-resistant unit** — an object. That principle is called **encapsulation**, and it's the heart of why OOP exists.

---

## 2. Classes and objects

A **class** declares what kind of data an object holds (**properties**) and what it can do (**methods**). An **object** is a live instance built from that class in memory.

```php
<?php

class BankAccount
{
    public string $owner;       // a property (typed)
    public int $balance = 0;    // a property with a default value

    // a method
    public function deposit(int $amount): void
    {
        $this->balance += $amount;
    }
}

// Instantiation: create an object from the class with `new`
$account = new BankAccount();
$account->owner = 'Ada';
$account->deposit(50);

echo $account->balance; // Output: 50
```

Key syntax:

- `new BankAccount()` builds the object. The parentheses are optional **only** when there's no constructor or no required arguments — but always writing them is the modern, lint-clean convention.
- `->` is the **object operator**: `$object->property` and `$object->method()`. Note there is **no `$`** on the member name (`$account->balance`, not `$account->$balance`).

### Instantiating with the class constant

`::class` resolves to the **fully-qualified class name as a string**. It's compile-time, IDE-checkable, and rename-safe — strongly preferred over hard-coded class-name strings.

```php
<?php
echo BankAccount::class; // Output: BankAccount  (or App\Models\BankAccount if namespaced)

$className = BankAccount::class;
$account   = new $className();        // instantiate from a string variable

// Or instantiate straight from the class-constant expression (parens required around it):
$account   = new (BankAccount::class)();

// PHP 8.4+: you can instantiate directly without a temp variable — no wrapping parens needed:
$balance = new BankAccount()->balance; // valid in PHP 8.4, syntax error in 8.3 and earlier
```

> **PHP 8.4 note:** PHP 8.4 lets you chain off `new Foo()` **without** wrapping it in parentheses: `new Foo()->method()`. In PHP 8.1–8.3 you must write `(new Foo())->method()`.

In Laravel you'll see `::class` constantly:

```php
<?php
// Laravel 12 — the container resolves a class by its name
$service = app(PaymentGateway::class);

// Route to a controller action — rename-safe, IDE-navigable
Route::get('/users', [UserController::class, 'index']);
```

---

## 3. `$this` — the current object

Inside a method, `$this` refers to **the specific object the method was called on**. It's how an object reads and mutates its own properties.

```php
<?php
class Counter
{
    public int $count = 0;

    public function increment(): static  // returns the same object → enables chaining
    {
        $this->count++;
        return $this;
    }
}

$c = new Counter();
$c->increment()->increment()->increment();
echo $c->count; // Output: 3
```

Returning `$this` (typed as `static`) is the **fluent interface** pattern — exactly how Laravel's query builder lets you write `User::where(...)->orderBy(...)->limit(...)`.

---

## 4. Visibility: `public`, `protected`, `private`

**Visibility** (a.k.a. *access modifiers*) controls *who* can touch a member. This is how encapsulation is enforced by the language.

| Modifier | Accessible from... |
|----------|--------------------|
| `public` | anywhere (the object, subclasses, outside code) |
| `protected` | the class itself **and** its subclasses |
| `private` | **only** the exact class that declares it |

```php
<?php
class Account
{
    public string $owner;        // anyone can read/write
    protected float $balance = 0; // this class + subclasses
    private string $pin;          // only Account itself

    public function setPin(string $pin): void
    {
        $this->pin = $pin;        // OK: same class
    }
}

class SavingsAccount extends Account
{
    public function showBalance(): float
    {
        return $this->balance;    // OK: protected, accessible in subclass
        // return $this->pin;     // FATAL ERROR: private, invisible to subclass
    }
}

$a = new Account();
// echo $a->balance; // FATAL ERROR: Cannot access protected property
```

**Why bother?** Visibility lets you change private internals later without breaking callers, and it documents intent: "touch this through my methods, not directly." Default to the **most restrictive** level that still works (usually `private` for state, `public` for the deliberate API).

> **PHP 8.4 note:** PHP 8.4 adds **asymmetric visibility** — you can make a property publicly *readable* but only privately *writable*: `public private(set) int $balance;`. Readers see it; only the class can change it. This often removes the need for a manual getter.

```php
<?php
// PHP 8.4 asymmetric visibility
class Wallet
{
    public function __construct(
        public private(set) int $balance = 0  // read anywhere, write only inside Wallet
    ) {}

    public function add(int $n): void { $this->balance += $n; }
}

$w = new Wallet(100);
echo $w->balance; // Output: 100  (read OK)
// $w->balance = 5; // Error: Cannot modify private(set) property Wallet::$balance from global scope
```

---

## 5. Constructors and constructor property promotion

The **constructor** is a special method named `__construct()` that PHP runs automatically the moment an object is created. Use it to require and validate the data an object needs to exist.

```php
<?php
class User
{
    public string $name;
    public string $email;

    public function __construct(string $name, string $email)
    {
        $this->name  = $name;
        $this->email = $email;
    }
}

$u = new User('Ada', 'ada@example.com');
echo $u->name; // Output: Ada
```

That `$this->x = $x;` boilerplate gets old fast. **Constructor property promotion** (PHP 8.0+) declares and assigns the property in one move — just add a visibility keyword to the constructor parameter:

```php
<?php
class User
{
    public function __construct(
        public string $name,
        public string $email,
        private ?string $phone = null,   // promoted + default
    ) {}
}

$u = new User('Ada', 'ada@example.com');
echo $u->name; // Output: Ada
```

This is the idiom you'll see throughout Laravel 12 (controllers, jobs, DTOs, services). You can mix promoted and non-promoted parameters, and add a body for validation:

```php
<?php
class Price
{
    public function __construct(public readonly int $cents)
    {
        if ($cents < 0) {
            throw new InvalidArgumentException('Price cannot be negative.');
        }
    }
}
```

### Named arguments + constructors

**Named arguments** (PHP 8.0+) let you pass constructor args by name in any order, which pairs beautifully with promotion and many optional params:

```php
<?php
$u = new User(
    email: 'ada@example.com',
    name: 'Ada',           // order doesn't matter when named
);
```

---

## 6. Destructors

A **destructor**, `__destruct()`, runs when an object is destroyed — either when the last reference to it goes away or when the script ends. It's used for cleanup (closing files/connections). It's rare in everyday Laravel code because the framework and PHP's garbage collector handle most lifecycles.

```php
<?php
class FileLogger
{
    // Left untyped on purpose: fopen() returns a `resource`, and PHP has
    // no `resource` type declaration, so this is the one place you skip a type.
    private $handle;

    public function __construct(string $path)
    {
        $this->handle = fopen($path, 'a');
    }

    public function __destruct()
    {
        if (is_resource($this->handle)) {
            fclose($this->handle); // guaranteed cleanup
        }
    }
}
```

> **Gotcha:** You can't reliably predict the *exact* moment a destructor fires under garbage collection, and you shouldn't depend on order between objects. Use it for safety nets, not core logic.

---

## 7. Typed properties

Since PHP 7.4, properties can declare a **type**. PHP then enforces it: assigning the wrong type throws a `TypeError`. This catches bugs early and documents intent.

```php
<?php
class Product
{
    public int $id;
    public string $name;
    public ?string $description = null; // nullable: string OR null
    public float $price;
    public array $tags = [];
}

$p = new Product();
$p->price = 9.99;
// $p->price = 'free'; // TypeError: Cannot assign string to property of type float
```

> **Uninitialized typed properties:** A typed property **without a default** is in a special "uninitialized" state — not `null`. Reading it before assignment throws:
> `Error: Typed property Product::$name must not be accessed before initialization`.
> This is *different* from old untyped properties, which silently returned `null`. Initialize via the constructor to avoid it.

---

## 8. `readonly` properties and `readonly` classes

A **`readonly`** property (PHP 8.1+) can be assigned **exactly once** — typically inside the constructor — and never again. After that it's locked. This gives you cheap **immutability**: ideal for value objects and DTOs (Data Transfer Objects — simple objects that just carry data).

```php
<?php
class Money
{
    public function __construct(
        public readonly int $amount,
        public readonly string $currency,
    ) {}

    // To "change" a readonly object, return a NEW one
    public function add(int $n): static
    {
        return new static($this->amount + $n, $this->currency);
    }
}

$m = new Money(100, 'USD');
echo $m->amount; // Output: 100
// $m->amount = 200; // Error: Cannot modify readonly property Money::$amount

$m2 = $m->add(50);
echo $m2->amount; // Output: 150
echo $m->amount;  // Output: 100  (original untouched)
```

Rules to remember:
- `readonly` requires a **type**; it can't be untyped.
- It can be written **only from within the declaring class's scope**, and only **once**.
- A `readonly` property **cannot have a default value on the property declaration** (`public readonly int $x = 0;` is a fatal error — "Readonly property cannot have default value"). It must be initialized at runtime, normally in the constructor. (A *promoted* constructor parameter may still carry a default, e.g. `public readonly int $x = 0` in the parameter list — that default lives on the parameter, not the property, which is why the `Money`/`Wallet` examples above are legal.)
- `static` properties cannot be `readonly`.

### `readonly` classes (PHP 8.2+)

Mark the **whole class** `readonly` and every property becomes readonly automatically — and the class may declare only typed properties.

```php
<?php
// PHP 8.2+
readonly class Point
{
    public function __construct(
        public int $x,
        public int $y,
    ) {}
}
```

> **PHP 8.3 note:** PHP 8.3 made `readonly` properties **re-initializable inside `__clone()`**, which finally makes cloning-with-changes possible on immutable objects (see §13). Before 8.3, cloning a readonly object and tweaking it in `__clone` failed.

---

## 9. Class constants and their visibility

A **class constant** is a fixed value tied to the class, not to any instance. Declared with `const`, accessed with `::`. Use them for fixed configuration, enum-like flags (though real enums are better — see the enums module), and named magic numbers.

```php
<?php
class HttpClient
{
    public const int DEFAULT_TIMEOUT = 30;     // typed const (PHP 8.3+)
    protected const string BASE = 'https://';  // visibility on constants (PHP 7.1+)
    private const array RETRY_CODES = [500, 502, 503];

    public function timeout(): int
    {
        return self::DEFAULT_TIMEOUT; // access via self:: inside the class
    }
}

echo HttpClient::DEFAULT_TIMEOUT; // Output: 30  (no instance needed, no $)
```

- Constants can be `public` / `protected` / `private` (since PHP 7.1) — same meaning as for properties.
- **Typed constants** are a PHP 8.3 feature: `const int DEFAULT_TIMEOUT = 30;`. In 8.1–8.2 omit the type.
- Constants are accessed **without** a `$`: `Class::CONST`, not `Class::$CONST`.

> **PHP 8.3 note:** PHP 8.3 also added **dynamic class-constant fetch**: `$class::{$name}` where `$name` is a variable.

---

## 10. Static properties and methods

A **static** member belongs to the **class itself**, shared across all instances — there is exactly one copy. Access it with `::` and `self::`/`static::`, never `$this` (a static method has no `$this`).

```php
<?php
class IdGenerator
{
    private static int $next = 1;          // one shared counter

    public static function nextId(): int   // static method
    {
        return self::$next++;
    }
}

echo IdGenerator::nextId(); // Output: 1
echo IdGenerator::nextId(); // Output: 2
echo IdGenerator::nextId(); // Output: 3
```

Note that for a **static property** you *do* keep the `$`: `self::$next`. For a **constant** you don't: `self::CONST`.

**When to use static:** factory methods, utility helpers, counters/registries, and named constructors. **When to avoid:** static *mutable state* makes code hard to test and reason about (it's effectively a global). Laravel hides a lot of static-looking calls behind **facades** (`Cache::get(...)`, `DB::table(...)`) — but those actually proxy to real objects in the service container, so they're testable; they're not true static state.

---

## 11. `self` vs `static` vs `parent` (late static binding)

These three keywords all reference classes inside class code, but they resolve differently. This is a classic interview trap.

- **`parent::`** — the parent class (used to call an overridden method's original).
- **`self::`** — the class where the code is **literally written** (resolved at *compile time*).
- **`static::`** — the class that was **actually called at runtime** (resolved at *runtime*). This runtime resolution is called **Late Static Binding (LSB)**.

```php
<?php
class Model
{
    public static function create(): static
    {
        // `new self()`   → always a Model
        // `new static()` → whatever subclass was called
        return new static();
    }

    public function who(): string
    {
        return self::class . ' / ' . static::class;
    }
}

class User extends Model {}

$u = User::create();
echo $u::class;     // Output: User   (thanks to `new static()`)

echo (new User())->who();
// Output: Model / User
//          ^self    ^static
```

`new self()` would have returned a `Model` even when called as `User::create()`. `new static()` honors the real runtime class — which is exactly why Eloquent's `User::create([...])` returns a `User`, not a base `Model`. The return type `static` (PHP 8.0+) documents this contract.

`parent::` example:

```php
<?php
class Animal
{
    public function speak(): string { return 'some sound'; }
}

class Dog extends Animal
{
    public function speak(): string
    {
        return parent::speak() . ' → woof'; // extend, don't replace
    }
}

echo (new Dog())->speak(); // Output: some sound → woof
```

---

## 12. Object comparison: `==` vs `===`

PHP compares objects in two distinct ways:

- **`==` (loose / equality):** `true` when **both objects are instances of the same class AND all their properties are equal** (compared with `==`, recursively).
- **`===` (strict / identity):** `true` **only when both variables point to the exact same object instance** (same handle in memory).

```php
<?php
class Color
{
    public function __construct(public string $hex) {}
}

$a = new Color('#fff');
$b = new Color('#fff');
$c = $a;

var_dump($a == $b);  // Output: bool(true)   — same class, equal properties
var_dump($a === $b); // Output: bool(false)  — different instances
var_dump($a === $c); // Output: bool(true)   — $c is literally $a
```

Mnemonic: **`==` asks "are they alike?"**, **`===` asks "are they the same one?"**

> **Caveat about `==`:** the loose comparison compares *all* properties, including private/protected ones, and recurses into nested objects. If two objects hold different but "equivalent" data (say a `DateTime` of `2024-01-01 00:00` vs `2024-01-01 00:00:00.000001`), `==` may surprise you. For domain "value equality" you usually want an explicit method that compares only the fields that matter:

```php
<?php
final class Color
{
    public function __construct(public readonly string $hex) {}

    public function equals(self $other): bool
    {
        return strtolower($this->hex) === strtolower($other->hex);
    }
}

$a = new Color('#FFF');
$b = new Color('#fff');
var_dump($a == $b);        // Output: bool(false)  — raw property strings differ
var_dump($a->equals($b));  // Output: bool(true)   — your rule: case-insensitive
```

---

## 13. Object cloning: shallow vs deep, and `__clone`

Assigning an object (`$b = $a`) does **not** copy it — both names point to the same object (see §14). To get a genuine second object, use **`clone`**.

```php
<?php
$a = new Color('#fff');
$b = clone $a;       // a separate object with copied property values
$b->hex = '#000';

echo $a->hex; // Output: #fff  (unaffected)
echo $b->hex; // Output: #000
```

By default `clone` does a **shallow copy**: scalar properties are duplicated, but if a property *holds another object*, both the original and the clone end up pointing at that **same** nested object.

```php
<?php
class Engine { public int $hp = 100; }

class Car
{
    public function __construct(public Engine $engine) {}
}

$a = new Car(new Engine());
$b = clone $a;             // shallow: $a->engine and $b->engine are the SAME Engine
$b->engine->hp = 250;

echo $a->engine->hp; // Output: 250  ← surprise! the original changed too
```

To get a **deep copy** (clone nested objects too), implement the magic method **`__clone()`**, which PHP calls automatically *on the new object* right after a clone:

```php
<?php
class Car
{
    public function __construct(public Engine $engine) {}

    public function __clone(): void
    {
        $this->engine = clone $this->engine; // duplicate the nested object
    }
}

$a = new Car(new Engine());
$b = clone $a;
$b->engine->hp = 250;

echo $a->engine->hp; // Output: 100  ← original now safe
echo $b->engine->hp; // Output: 250
```

> **PHP 8.3 note:** Inside `__clone()` you may now reassign `readonly` properties (`$this->engine = clone $this->engine;` even if `$engine` is readonly). This is the supported pattern for "modify-on-clone" with immutable objects.

> **Version note:** In PHP 8.4 and earlier, `clone` is a language construct, **not** a function — you cannot write `clone(...)` as a first-class callable or pass it to `array_map()`. That changed in **PHP 8.5** (the *clone-with* RFC), which turns `clone` into a function-like construct, adds an optional second argument for overriding properties (`clone($obj, ['hp' => 250])`), and makes `clone(...)` usable as a first-class callable. On PHP 8.4, deep-copy a collection with an explicit closure instead: `array_map(fn($o) => clone $o, $list)`.

---

## 14. Object handle / reference semantics when passing

This trips up nearly everyone coming from "pass by value" thinking. In PHP, **objects are passed by handle**: when you assign or pass an object, you copy a *reference (handle) to the same object*, not the object's contents. Scalars (int, string, bool, float, array) are copied by value.

```php
<?php
function rename(User $u): void
{
    $u->name = 'Changed';   // mutates the caller's actual object
}

$user = new User('Ada', 'ada@example.com');
rename($user);
echo $user->name; // Output: Changed   ← the original was modified
```

But reassigning the parameter to a *new* object inside the function does **not** affect the caller, because the handle itself is a copy:

```php
<?php
function replace(User $u): void
{
    $u = new User('New', 'new@example.com'); // only the local handle is swapped
}

$user = new User('Ada', 'ada@example.com');
replace($user);
echo $user->name; // Output: Ada   ← caller's variable still points to the original
```

Contrast with a scalar, which is copied:

```php
<?php
function addOne(int $n): void { $n++; }

$x = 5;
addOne($x);
echo $x; // Output: 5   (unchanged — passed by value)
```

**Practical takeaway:** if a function receives an object and you don't want it mutated, `clone` it first, or design the object as `readonly`/immutable.

---

## 15. `instanceof`

`instanceof` tests whether an object is an instance of a class, a subclass, or implements an interface. It returns a `bool` and is the safe way to type-check before acting.

```php
<?php
interface Shape {}
class Circle implements Shape {}
class Square implements Shape {}

$s = new Circle();

var_dump($s instanceof Circle); // Output: bool(true)
var_dump($s instanceof Shape);  // Output: bool(true)  (implements the interface)
var_dump($s instanceof Square); // Output: bool(false)

// Works with a class-name string or ::class too:
$class = Circle::class;
var_dump($s instanceof $class); // Output: bool(true)
```

In modern code, prefer a `match`/typed-parameter design over long `instanceof` chains, but `instanceof` is perfect for guard clauses:

```php
<?php
if (! $value instanceof Shape) {
    throw new InvalidArgumentException('Expected a Shape.');
}
```

---

## 16. Encapsulation with getters and setters

Encapsulation means exposing a **controlled API** and hiding raw state. Make state `private`, then offer methods that enforce rules.

```php
<?php
class Temperature
{
    private float $celsius;

    public function __construct(float $celsius)
    {
        $this->setCelsius($celsius);
    }

    public function getCelsius(): float
    {
        return $this->celsius;
    }

    public function setCelsius(float $value): void
    {
        if ($value < -273.15) {
            throw new InvalidArgumentException('Below absolute zero!');
        }
        $this->celsius = $value;
    }

    // a derived/computed getter — no stored field needed
    public function getFahrenheit(): float
    {
        return $this->celsius * 9 / 5 + 32;
    }
}

$t = new Temperature(25);
echo $t->getFahrenheit(); // Output: 77
$t->setCelsius(100);
echo $t->getFahrenheit(); // Output: 212
```

**Don't over-do it.** A getter/setter for *every* field that does nothing but read/write is just a verbose public property. Add accessors when you need **validation, computation, or the freedom to change internals later**. In PHP 8.4, **property hooks** can also replace trivial getters/setters:

```php
<?php
// PHP 8.4 property hooks — logic attached directly to a property
class Person
{
    public string $fullName {
        get => trim($this->first . ' ' . $this->last);
    }

    public function __construct(
        private string $first,
        private string $last,
    ) {}
}

$p = new Person('Ada', 'Lovelace');
echo $p->fullName; // Output: Ada Lovelace
```

---

## 17. `stdClass` and casting arrays to objects

**`stdClass`** is PHP's built-in, empty, generic class — an object with no methods and no predefined properties. You get one when you cast an array to `(object)`, decode JSON without `true`, or fetch DB rows as objects.

```php
<?php
// 1. Casting an array to an object
$data = (object) ['name' => 'Ada', 'role' => 'admin'];
echo $data->name; // Output: Ada
echo $data->role; // Output: admin

// 2. JSON decode → stdClass by default
$obj = json_decode('{"id": 7, "active": true}');
echo $obj->id; // Output: 7
var_dump($obj instanceof stdClass); // Output: bool(true)

// pass true to get an associative array instead
$arr = json_decode('{"id": 7}', true);
echo $arr['id']; // Output: 7

// 3. Casting back: object → array
$back = (array) $data;
echo $back['name']; // Output: Ada

// 4. Creating one ad hoc
$row = new stdClass();
$row->x = 1;
```

> **Caveat:** Casting an array to an object only works cleanly with **string keys**. Numeric keys become property names you can't access with normal `->` syntax (e.g. `$obj->0` is invalid). And `stdClass` gives you **no type safety or behavior** — prefer a real class (or a `readonly` DTO) for anything important. In Laravel, `DB::table('users')->get()` returns a collection of `stdClass` rows by default, while Eloquent (`User::all()`) returns typed model objects.

---

## ⚠️ Common Mistakes & Gotchas

1. **Assuming `$b = $a` copies the object.**
   It doesn't — both point to the same object, so mutating one mutates "both."
   **Fix:** use `clone` for a real copy; implement `__clone()` if it has nested objects you also need duplicated.

2. **Confusing `==` and `===` for objects.**
   `==` is `true` for two *different* objects with equal properties; `===` is `true` only for the *same* instance. Using `===` to check "are these the same data?" will silently fail.
   **Fix:** use `==` (or a custom `equals()` method) for value equality, `===` for identity.

3. **Reading a typed property before initializing it.**
   `public string $name;` with no default is *uninitialized*, not `null`. Touching it throws `Error: ... must not be accessed before initialization`.
   **Fix:** assign it in the constructor, give it a default, or make it nullable (`?string $name = null`).

4. **Mixing up `$` rules across static members.**
   Static *property*: `self::$count` (keep the `$`). Class *constant*: `self::COUNT` (no `$`). Instance property: `$this->count` (no `$` on the name). Getting these wrong yields "undefined constant/property" errors.
   **Fix:** memorize — `$` only on **variables/static properties**, never on constants or member *names* after `->`.

5. **Using `new self()` in a base class that's meant to be subclassed.**
   `self` is locked to the declaring class, so `User::create()` would return a `Model`, not a `User`.
   **Fix:** use `new static()` and a `static` return type for late static binding.

6. **Shallow-clone surprises.**
   A `clone` shares nested objects with the original; changing the clone's nested object changes the original's too.
   **Fix:** implement `__clone()` to `clone` each nested object (deep copy) — or make the nested object immutable.

7. **Forgetting that mutating a passed-in object affects the caller.**
   Functions receive a handle to your *actual* object.
   **Fix:** `clone` before mutating, or accept/return immutable (`readonly`) objects.

---

## ✅ Best Practices

- **Default to the most restrictive visibility.** Make properties `private` (or `protected`); expose only a deliberate `public` API. This is what lets you refactor internals safely.
- **Use constructor property promotion** for concise, single-source-of-truth class definitions — the Laravel 12 standard.
- **Type everything**: properties, parameters, return types. Let the engine catch bugs you'd otherwise find in production.
- **Prefer immutability** with `readonly` properties/classes for value objects and DTOs. Immutable objects can't be mutated out from under you, so they're far easier to reason about and safe to share freely.
- **Make value objects `final`.** A value object's whole point is its invariants; marking the class `final` stops a subclass from quietly breaking them and makes `new static()` vs `new self()` a non-issue.
- **Use `::class`** instead of class-name strings everywhere — rename-safe, IDE-navigable, no typos.
- **Use `new static()` + `static` return types** in base classes intended for inheritance; reserve `self` for "always exactly this class."
- **Don't write reflexive getters/setters.** Add accessors only for validation, computation, or future-proofing; consider PHP 8.4 property hooks / asymmetric visibility instead.
- **Avoid mutable static state** — it's a global in disguise and a testing nightmare. Use dependency injection (Laravel's container) instead.
- **Reach for real classes over `stdClass`** for anything beyond throwaway data; they give you types, behavior, and IDE help.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between a class and an object?**
A class is the blueprint/template that defines properties and methods; an object is a concrete instance of that class living in memory. One class → many independent objects.

**Q2. Explain `public` vs `protected` vs `private`. Why does it matter?**
`public` = accessible anywhere; `protected` = the class and its subclasses; `private` = only the declaring class. It enforces encapsulation: you hide internals behind a stable API so you can change implementation without breaking callers, and you prevent invalid state from being set directly.

**Q3. What is constructor property promotion and why use it?**
A PHP 8.0 shorthand where you add a visibility modifier to a constructor parameter (`public function __construct(private int $id)`) and PHP declares the property and assigns it automatically — eliminating boilerplate. It's the modern Laravel idiom.

**Q4. `self` vs `static` vs `parent` — and what is late static binding?**
`parent::` calls the parent class; `self::` resolves to the class where the code is *written* (compile-time); `static::` resolves to the class *actually called at runtime* (late static binding). LSB is why `new static()` in a base `Model::create()` returns the correct subclass (e.g. `User`). `new self()` would always return the base class.

**Q5. How does PHP compare objects with `==` versus `===`?**
`==` is `true` when both are the same class with equal properties (recursive value comparison). `===` is `true` only when both variables reference the exact same instance (identity). "Alike" vs "the same one."

**Q6. (Under the hood) How are objects passed to functions — by value or by reference?**
Neither, exactly. PHP passes an object **by handle**: it copies a *reference/identifier* that points to the same underlying object. So mutating the object's properties inside a function affects the caller's object, but reassigning the parameter to a new object only changes the local copy of the handle. True pass-by-reference (`&$obj`) would also let you swap the caller's variable itself. Internally, an object variable holds a `zend_object` handle (a pointer with refcounting); the engine increments the refcount on copy and decrements it when a reference goes away. When the count drops to zero the object is freed and `__destruct()` runs; objects trapped in reference cycles are instead collected later by the cycle-collecting garbage collector (so destructor timing for those is not guaranteed).

**Q7. What's the difference between shallow and deep cloning? How do you implement deep cloning?**
`clone` does a shallow copy: scalars are duplicated but nested *objects* are shared between original and clone. For a deep copy, implement the `__clone()` magic method and `clone` each nested object inside it. (Since PHP 8.3 you can even reassign `readonly` properties inside `__clone()`.)

**Q8. What are `readonly` properties and when would you use them?**
Properties (PHP 8.1+) that can be assigned once — usually in the constructor — then never changed. Great for immutable value objects/DTOs. PHP 8.2 adds whole `readonly` classes. To "change" one, you construct a new instance.

**Q9. What happens if you read a typed property before initializing it?**
You get a fatal `Error: Typed property must not be accessed before initialization` — it's an "uninitialized" state, distinct from `null`. Initialize it in the constructor or make it nullable/defaulted.

**Q10. What is `stdClass` and when do you encounter it?**
PHP's generic empty class. You get `stdClass` objects from casting an array `(object)`, from `json_decode()` without the `true` flag, and from `DB::table()->get()` rows in Laravel. It has no type safety or methods — prefer a real class for anything meaningful.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
// --- Declaration & instantiation ---
class Foo {}
$o = new Foo();
$o = new (Foo::class)();         // from class constant
$o = new Foo()->bar();           // PHP 8.4: no wrapping parens needed

// --- Members ---
public int $x;                   // typed property
public readonly int $id;         // readonly (PHP 8.1+); assign once
public const int MAX = 10;       // typed constant (PHP 8.3+); access: Foo::MAX
private static int $n = 0;       // static property; access: self::$n

// --- Constructor promotion + named args ---
public function __construct(
    public string $name,
    private ?int $age = null,
) {}
new Foo(name: 'Ada', age: 30);

// --- Access operators ---
$o->prop;        // instance property/method (no $ on name)
Foo::CONST;      // class constant (no $)
Foo::$staticProp;// static property (keep $)
self::method();  // declaring class (compile-time)
static::method();// runtime class (late static binding)
parent::method();// parent class

// --- Comparison / type checks ---
$a == $b;        // same class + equal props
$a === $b;       // same instance
$a instanceof Foo;

// --- Copying ---
$b = $a;         // SAME object (handle copy)
$b = clone $a;   // shallow copy; define __clone() for deep

// --- stdClass ---
$d = (object) ['k' => 'v'];      // array → object
json_decode($json);              // → stdClass
json_decode($json, true);        // → associative array
```

| Keyword | Resolves to | When |
|---------|-------------|------|
| `self::` | class where code is written | compile time |
| `static::` | class called at runtime | runtime (LSB) |
| `parent::` | parent class | — |
| `$this->` | current object instance | runtime |

| Compare | Meaning |
|---------|---------|
| `==` | same class **and** equal properties |
| `===` | same exact instance |

---

## 🧪 Mini Exercises

1. **Immutable Money.** Build a `readonly class Money` with `int $amount` and `string $currency`. Add `add(Money $other): static` that throws if currencies differ and otherwise returns a *new* `Money`. Prove the original is unchanged after `add()`.

2. **ID factory with LSB.** Write a base class `Entity` with a `public static int $count` and a static `create(): static` using `new static()`. Subclass it as `Post` and `Comment`. Show that `Post::create()` returns a `Post` (use `::class`) and explain what would break if you used `new self()` instead.

3. **Deep clone a graph.** Create `Team` holding an array of `Member` objects. Implement `__clone()` so cloning a `Team` also clones each `Member`. Demonstrate that mutating a member on the clone does not affect the original.

4. **Encapsulated bank account.** Implement `Account` with a `private float $balance`, a `deposit(int)` and `withdraw(int)` that reject invalid amounts (negative, or overdraft), and a read-only balance. Try at least two approaches to "read-only": a `getBalance()` getter, and PHP 8.4 `public private(set)`.

5. **Reference semantics quiz (then verify mentally).** Write a function `mutate(stdClass $o)` that sets `$o->changed = true`, and a function `reassign(stdClass $o)` that sets `$o = new stdClass()`. Predict the caller-side result of each before running, then confirm your model of handle-passing is correct.
