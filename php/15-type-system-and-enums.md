# Type System & Enums

PHP began life as a dynamically and weakly typed scripting language: a variable could hold anything, and the engine would happily juggle a string into a number when it felt like it. Over the last decade PHP has grown a genuinely powerful **gradual type system** — you opt into as much type safety as you want, function by function, file by file. PHP 8.1 then added **enums**, finally giving the language a first-class way to model "a value that is one of a fixed set of options." Mastering both is the difference between writing PHP that *runs* and writing PHP that the engine, your IDE, and static analysers (PHPStan, Psalm) all help you keep correct.

> **Gradual typing** = a type system where type declarations are optional. Code with annotations is checked; code without them runs with the old dynamic rules. The two can coexist in the same program.

---

## **What you'll learn**

- How `declare(strict_types=1)` flips PHP between **coercive** and **strict** type checking, and why you should almost always turn it on.
- The full vocabulary of type declarations: scalars, nullable, **union**, **intersection**, and **DNF** types, plus the special types `mixed`, `void`, `never`, `self`, `static`, `parent`, `iterable`, `object`, and the standalone `false`/`true`/`null`.
- **Typed properties** and the dreaded "must not be accessed before initialization" error — what it means and how to avoid it.
- **`readonly`** properties and classes, and how they pair with constructor promotion to model immutable value objects.
- **Enums** end to end: pure vs backed, `cases()`, `from()`/`tryFrom()`, methods, constants, interfaces, and `match`.
- When an **enum** beats a bag of `const`s, and how Laravel 12 uses enums (casts, route binding, validation).
- The interview-grade "under the hood" details: variance, the `gettype`/`settype` distinction, and why enums are singletons.

---

## 1. Why types at all? The two modes of PHP

PHP has always *had* types at runtime — every value is internally an `int`, `string`, `bool`, `float`, `array`, `object`, `null`, or `resource`. What changed is your ability to **declare** the types you expect and have the engine enforce them.

Consider a function with no declarations:

```php
<?php

function add($a, $b) {
    return $a + $b;
}

echo add(2, 3);       // 5
echo add("2", "3");   // 5  — numeric strings coerced for the `+` operator
echo add("apples", 3);   // PHP 8: TypeError — "Unsupported operand types: string + int"
echo add("2 apples", 3); // PHP 8: Warning "A non-numeric value encountered" → still 5
```

> **Careful:** the last two lines are about *arithmetic on an untyped value*, which is a separate mechanism from *type-declaration coercion*. A leading-numeric string (`"2 apples"`) only triggers a `Warning` and uses the `2`; a fully non-numeric string (`"apples"`) throws a `TypeError`. Once you add a parameter type (`int $a`), the rules change — see the typed examples below, where `"10abc"` is rejected outright.

Without declarations you have no guarantees: callers can pass anything, bugs surface deep inside the function, and your IDE can't autocomplete. Adding a type declaration documents intent *and* enforces it:

```php
<?php

function add(int $a, int $b): int {
    return $a + $b;
}
```

The behaviour of that `int` declaration depends entirely on which **mode** the calling file is in.

### Coercive mode (the default)

By default PHP runs in **coercive mode**: when a value of the "wrong" but convertible scalar type is passed, the engine quietly *coerces* (converts) it to the declared type.

```php
<?php
// No declare(strict_types=1) at the top → coercive mode

function priceLabel(float $price): string {
    return '$' . number_format($price, 2);
}

echo priceLabel(10);     // "$10.00" — int 10 coerced to float 10.0
echo priceLabel("9.5");  // "$9.50"  — numeric string coerced to float 9.5
echo priceLabel(true);   // "$1.00"  — bool true coerced to float 1.0 (!)
```

That last line is the danger of coercion: `true` silently becomes `1.0`. Coercion only happens for **scalar types** (`int`, `float`, `string`, `bool`) and only when the value is sensibly convertible. A non-numeric string into an `int` parameter throws a `TypeError`, and passing an array where a scalar is expected always fails.

> One asymmetry worth memorising: in coercive mode `int → float` is allowed (10 becomes 10.0), but passing a `float` with a fractional part to an `int` parameter emits `Deprecated: Implicit conversion from float … to int loses precision` (since PHP 8.1) and still truncates — it does **not** throw a `TypeError` in coercive mode (this deprecation becomes a hard error in PHP 9). Fully numeric strings coerce to `int`/`float`; leading-numeric strings like `"10abc"` passed to a *typed* `int`/`float` parameter throw a `TypeError` in PHP 8.

### Strict mode

Put this as the **very first statement** of a file (before any other code, after only an optional opening tag):

```php
<?php

declare(strict_types=1);
```

Now, in *this file*, no coercion happens for scalar arguments. The type must match exactly — with the single, sanctioned exception that an `int` is still accepted where a `float` is declared (because every integer is exactly representable as a float; this is the **widening** rule).

```php
<?php

declare(strict_types=1);

function priceLabel(float $price): string {
    return '$' . number_format($price, 2);
}

echo priceLabel(9.5);   // "$9.50"  ✅
echo priceLabel(10);    // "$10.00" ✅ — int→float widening still allowed
echo priceLabel("9.5"); // TypeError: must be of type float, string given ❌
echo priceLabel(true);  // TypeError ❌
```

**Two things every developer trips on with `strict_types`:**

1. **It is per-file, and it applies to the call site, not the declaration site.** Whether a call is checked strictly depends on the file where the *call* is written, not where the function is *defined*. A strict file calling a function in a coercive file is checked strictly; a coercive file calling a strict function is checked coercively.
2. **It only affects scalar coercion.** Return types and internal-function argument checking in strict mode also follow it, but class/array/callable types were never coerced anyway.

```bash
# strict_types is set in code, not via php.ini or a CLI flag.
# It MUST be the first statement; this is a fatal parse error otherwise:
#   "strict_types declaration must be the very first statement in the script"
```

**Recommendation:** turn `declare(strict_types=1)` on in every PHP file you write. Laravel's own framework files use it, modern starter kits add it, and tools like Laravel Pint / PHP-CS-Fixer can insert it automatically. Strictness surfaces bugs at the boundary instead of letting bad data sink into your domain logic.

### Inspecting a runtime type: `gettype()` vs `get_debug_type()` (and `settype()`)

When you need to *read* a value's runtime type, two functions exist — and they disagree, which is a classic interview gotcha:

```php
<?php
declare(strict_types=1);

gettype(1);          // "integer"  ← legacy long name
gettype(1.5);        // "double"   ← NOT "float"!
gettype(true);       // "boolean"
gettype(null);       // "NULL"
gettype(new stdClass); // "object" (never the class name)

get_debug_type(1);          // "int"      ← matches type-declaration syntax
get_debug_type(1.5);        // "float"
get_debug_type(new stdClass); // "stdClass" ← the actual class FQN
get_debug_type([]);         // "array"
```

`gettype()` returns the historical names (`integer`, `double`, `boolean`, `NULL`) that do **not** match what you write in a type declaration. `get_debug_type()` (PHP 8.0+) returns the modern names (`int`, `float`, `bool`, `null`) and the real class name for objects — prefer it for error messages and logging.

`settype(&$var, $type)` *mutates* a variable's type in place (`settype($x, 'integer')`), returning `bool`. It's a relic of weakly-typed PHP; in modern code prefer an explicit cast (`$x = (int) $x;`) or a typed boundary, both of which are clearer and friendlier to static analysers.

---

## 2. Scalar, nullable, and default-value types

### The four scalar types

`int`, `float`, `string`, `bool`. These are the only types that participate in coercion.

```php
<?php
declare(strict_types=1);

function register(string $name, int $age, float $score, bool $active): void {
    // ...
}
```

### Nullable types

Prefix any type with `?` to also allow `null`. `?T` is exactly equivalent to the union `T|null`.

```php
<?php
declare(strict_types=1);

function findUser(?int $id): ?string {
    return $id === null ? null : "User #$id";
}

echo findUser(5);    // "User #5"
var_dump(findUser(null)); // NULL
```

### Null defaults and the implicit-nullable trap

A parameter with a default value of `null` used to be *implicitly* nullable even without `?`:

```php
<?php
// Legacy / DEPRECATED in PHP 8.4
function legacy(string $name = null) { /* ... */ }
```

In **PHP 8.4 this implicit nullability is deprecated**. The compiler now warns:
`Deprecated: Implicitly marking parameter $name as nullable is deprecated, the explicit nullable type must be used instead`. The fix is to be explicit:

```php
<?php
declare(strict_types=1);

function modern(?string $name = null): void { /* ... */ }
```

> **Why the change?** The implicit rule made `string $x = null` silently mean `?string`, which was surprising and inconsistent with every other default. Explicit `?string` is clearer and future-proof. Make this change now — in PHP 9 it will become an error.

A non-null default does **not** widen the type:

```php
function greet(string $name = 'guest'): string { /* ... */ }
greet();        // "guest" — fine, default used
greet(null);    // TypeError — string does not accept null
```

---

## 3. Composite types: union, intersection, DNF

### Union types (PHP 8.0+)

A **union type** `A|B` means "a value of type A *or* type B." Use it when a value genuinely can be more than one type.

```php
<?php
declare(strict_types=1);

function parseId(int|string $id): string {
    return is_int($id) ? "numeric:$id" : "uuid:$id";
}

echo parseId(42);        // "numeric:42"
echo parseId("a1b2");    // "uuid:a1b2"
```

Rules:
- You cannot use redundant members — `int|int` is a fatal `Duplicate type int is redundant`.
- `void` and `never` are "standalone-only" types and may **never** appear in a union (`void|int` is a fatal error).
- `callable` **is** allowed in a *union* (e.g. `callable|string` is legal), but it is **disallowed in intersection types** and **cannot be used as a property type** (it isn't a fully reified type the engine can store/check there).
- `false`/`true`/`null` can be union members (e.g. `int|false`, common for functions like `strpos`).

```php
function indexOf(string $haystack, string $needle): int|false {
    return strpos($haystack, $needle); // returns int position, or false
}
```

### Intersection types (PHP 8.1+)

An **intersection type** `A&B` means "a value that satisfies type A *and* type B simultaneously." Members must be **class or interface** names (you cannot intersect scalars — `int&string` is meaningless and disallowed).

```php
<?php
declare(strict_types=1);

interface Countable2 { public function count(): int; }
interface Iterator2 { public function current(): mixed; }

function process(Countable2&Iterator2 $collection): void {
    // $collection is guaranteed to implement BOTH interfaces
    echo $collection->count();
    $collection->current();
}
```

A real-world example: `function f(Traversable&Countable $x)` accepts only objects that are both traversable and countable (like an `ArrayIterator`).

### DNF types (Disjunctive Normal Form, PHP 8.2+)

**DNF types** let you combine unions *and* intersections, as long as you write them in disjunctive normal form: an OR of AND-groups, with each intersection group wrapped in parentheses.

```php
<?php
declare(strict_types=1);

// "An (A AND B), OR null"
function handle((Countable&Traversable)|null $x): void { /* ... */ }

// "An (A AND B), OR a plain C"
function render((HasTitle&HasBody)|string $content): string {
    return is_string($content) ? $content : $content->getTitle();
}
```

The parentheses are required around each intersection. You cannot nest unions inside the parentheses; the whole expression must be one level of OR over groups of AND. `(A|B)&C` is **not** valid DNF — you'd rewrite it as `(A&C)|(B&C)`.

---

## 4. Special and "bottom/top" types

These types don't name concrete classes; they describe *kinds* of values or control flow.

| Type | Meaning | Valid as param? | Valid as return? | Valid as property? |
|------|---------|:---:|:---:|:---:|
| `mixed` | Any value at all (incl. null) | ✅ | ✅ | ✅ |
| `void` | Function returns nothing | ❌ | ✅ | ❌ |
| `never` | Function never returns (throws or exits) | ❌ | ✅ | ❌ |
| `null` (standalone) | Only `null` | ✅ (8.2+) | ✅ (8.2+) | ✅ (8.2+) |
| `false` (standalone) | Only `false` | ✅ (8.2+) | ✅ (8.2+) | ✅ (8.2+) |
| `true` (standalone) | Only `true` | ✅ (8.2+) | ✅ (8.2+) | ✅ (8.2+) |
| `object` | Any object instance | ✅ | ✅ | ✅ |
| `iterable` | `array` or `Traversable` | ✅ | ✅ | ✅ |
| `self` | The current class | ✅ | ✅ | ✅ |
| `static` | The called (late-bound) class | ❌ (never a param) | ✅ (return type added in 8.0) | ❌ |
| `parent` | The parent class | ✅ | ✅ | ✅ |

### `mixed`

`mixed` is the **top type** — it is the union of every type plus `null`. It documents "anything goes," but it disables most type safety, so prefer a precise type when you can.

```php
function jsonDecode(string $json): mixed {
    return json_decode($json, true); // could be array, scalar, or null
}
```

### `void` vs `never`

`void` means the function **returns control** but yields no usable value (`return;` with no value, or falling off the end). `never` (PHP 8.1+) is the **bottom type**: it promises the function **never returns control at all** — it always throws or terminates.

```php
<?php
declare(strict_types=1);

function log_it(string $msg): void {
    error_log($msg);
    // implicit "return;" — fine for void
}

function fail(string $msg): never {
    throw new RuntimeException($msg);
    // A `return;` here is a compile-time Fatal error:
    //   "A never-returning function must not return".
    // Falling off the end at runtime throws a TypeError.
}
```

`never` is genuinely useful: static analysers know that code after a `never` call is unreachable, so a `match` arm calling `fail(...)` doesn't need its own return.

### `self` vs `static` (late static binding)

`self` resolves to the class where the method is *written*. `static` resolves to the class that was actually *called* — this is **late static binding**. The difference matters for fluent APIs and factory methods on base classes.

```php
<?php
declare(strict_types=1);

class Model {
    public static function make(): static {   // returns the called class
        return new static();
    }
    public function self_clone(): self {       // always returns a Model
        return clone $this;
    }
}

class User extends Model {}

$u = User::make();          // instance of User, because of `static`
var_dump($u instanceof User); // bool(true)
```

Had `make()` returned `self`, the declared return type would be `Model`, losing the subtype information. Return `static` from base-class factories/fluent setters so subclasses keep their own type.

### `iterable` and `object`

`iterable` is sugar for `array|Traversable` — anything you can `foreach`. `object` accepts any object regardless of class.

```php
function sum(iterable $nums): int {
    $total = 0;
    foreach ($nums as $n) { $total += $n; }
    return $total;
}
sum([1, 2, 3]);                       // works with array
sum(new ArrayIterator([1, 2, 3]));    // works with Traversable
```

---

## 5. Typed properties & the "uninitialized" error

Since PHP 7.4 properties can carry type declarations. A typed property without a default value starts in a special **uninitialized** state — *not* `null`, but genuinely "no value yet."

```php
<?php
declare(strict_types=1);

class Profile {
    public string $bio;        // typed, NO default → uninitialized
    public ?string $nickname = null; // explicitly null
}

$p = new Profile();
var_dump($p->nickname);  // NULL — fine
echo $p->bio;            // Error: Typed property Profile::$bio
                         // must not be accessed before initialization
```

This is one of the most-asked-about runtime errors. The rationale: a `string` property must hold a string. The engine refuses to pretend it's `null` (which would violate the type), so it tracks a third state and throws if you read before writing.

**Fixes / ways to avoid it:**

```php
<?php
declare(strict_types=1);

class Profile {
    public string $bio = '';            // 1. give a default
    public function __construct(
        public string $name,            // 2. require it in the constructor
    ) {}
}

// 3. Make the property nullable if "no value" is legitimate:
class Profile2 {
    public ?string $bio = null;
}

// 4. Check before reading if assignment is conditional:
$p = new Profile('Ada');
if (isset($p->bio)) { /* ... */ }   // isset() returns false for uninitialized
```

> **Gotcha:** `isset($obj->typedProp)` returns `false` for an uninitialized typed property *and* for a property holding `null`. Use `array_key_exists`-style reasoning carefully; for objects, `isset` cannot distinguish "uninitialized" from "null."

### Constructor property promotion

PHP 8.0 lets you declare and assign constructor parameters as properties in one stroke. Combine it with types for concise, fully-typed classes.

```php
<?php
declare(strict_types=1);

// Verbose, pre-8.0 style:
class MoneyOld {
    public int $amount;
    public string $currency;
    public function __construct(int $amount, string $currency) {
        $this->amount = $amount;
        $this->currency = $currency;
    }
}

// Promoted, 8.0+:
class Money {
    public function __construct(
        public int $amount,
        public string $currency = 'USD',
    ) {}
}

$m = new Money(500);
echo $m->currency;  // "USD"
```

---

## 6. `readonly` — properties and classes

A **`readonly`** property (PHP 8.1+) may be written **exactly once**, and only from within the scope of the declaring class — typically in the constructor. After that any write throws an `Error`. This is the idiomatic way to build **immutable value objects**.

```php
<?php
declare(strict_types=1);

final class Coordinate {
    public function __construct(
        public readonly float $lat,
        public readonly float $lng,
    ) {}

    // Immutable "wither" — return a new instance instead of mutating
    public function withLat(float $lat): self {
        return new self($lat, $this->lng);
    }
}

$c = new Coordinate(51.5, -0.12);
echo $c->lat;          // 51.5
$c->lat = 0.0;         // Error: Cannot modify readonly property Coordinate::$lat
$c2 = $c->withLat(52.0); // OK: brand-new object
```

Rules and gotchas:

- `readonly` requires a **type declaration** (`public readonly $x;` without a type is a fatal error).
- It can't have a **default value** in the declaration — it must be initialized at runtime.
- "Readonly" is **shallow**: a `readonly array` can't be reassigned, but if it held an object that object's internals can still change. (PHP doesn't deep-freeze.)
- You cannot make a property `readonly` *and* later un-set or re-clone-modify it pre-8.3. **PHP 8.3** added the ability to re-initialize readonly properties during `__clone()`, enabling proper deep cloning of immutable objects.

**PHP 8.2+ readonly classes:** mark the whole class `readonly` and every property is implicitly readonly (and you can't add non-readonly or untyped/static properties).

```php
<?php
declare(strict_types=1);

readonly class Dto {
    public function __construct(
        public string $id,
        public int $version,
    ) {}
}
```

---

## 7. Variance (covariance & contravariance)

**Variance** governs how types may change when you override a method in a subclass. PHP supports **covariant return types** and **contravariant parameter types** (since 7.4), which makes overrides type-safe (this is the Liskov Substitution Principle expressed in the type system).

- **Covariant return type:** an override may return a *more specific* (narrower/subtype) type than the parent.
- **Contravariant parameter type:** an override may accept a *more general* (wider/supertype) type than the parent.

```php
<?php
declare(strict_types=1);

class Animal {}
class Dog extends Animal {}

class AnimalShelter {
    public function adopt(): Animal { return new Animal(); }
}

class DogShelter extends AnimalShelter {
    // ✅ Covariant return: Dog is a subtype of Animal
    public function adopt(): Dog { return new Dog(); }
}
```

Why is this safe? Anyone holding an `AnimalShelter` and calling `adopt()` expects *an* `Animal`; a `Dog` is one, so the contract still holds. The reverse — returning a *wider* type — would break callers, so it's forbidden.

```php
class Feeder {
    public function feed(Dog $d): void {}
}
class GenericFeeder extends Feeder {
    // ✅ Contravariant param: accepts the wider Animal
    public function feed(Animal $a): void {}
}
```

Property types, by contrast, are **invariant** — a typed property's type must match exactly across the hierarchy (you can't narrow or widen it), because properties are read *and* written.

---

## 8. Enums

Before enums, developers modelled fixed sets with class constants or bare strings — error-prone and untyped. An **enum** (enumeration) is a special class whose instances are a fixed, known set of named values. Each case is a real object, type-checkable, autocompletable, and impossible to forge.

> Mental model: an enum is a class, each case is a *singleton instance* of that class. `Status::Active === Status::Active` is always `true`, and there is exactly one `Status::Active` object in memory.

### Pure enums

A **pure enum** has cases with no underlying scalar value.

```php
<?php
declare(strict_types=1);

enum Direction {
    case North;
    case South;
    case East;
    case West;
}

$d = Direction::North;
var_dump($d);                       // enum(Direction::North)
var_dump($d === Direction::North);  // bool(true)
var_dump($d instanceof Direction);  // bool(true)
echo $d->name;                      // "North" (read-only `name` property)
```

Every case exposes a read-only `name` property (the case identifier as a string).

### Backed enums

A **backed enum** ties each case to a scalar **`int` or `string`** value (the "backing"). Declare the backing type after a colon. Backed cases expose a read-only `value` property in addition to `name`.

```php
<?php
declare(strict_types=1);

enum HttpStatus: int {
    case OK = 200;
    case NotFound = 404;
    case ServerError = 500;
}

echo HttpStatus::NotFound->value;  // 404
echo HttpStatus::NotFound->name;   // "NotFound"

enum Suit: string {
    case Hearts = 'H';
    case Spades = 'S';
    case Clubs  = 'C';
    case Diamonds = 'D';
}
```

The backing must be exactly `int` or `string` (no `float`, no `bool`), every case must have a literal value, and values must be **unique**.

### `cases()`, `from()`, `tryFrom()`

Every enum gets a static `cases()` returning all cases in declaration order. Backed enums additionally get `from()` and `tryFrom()` for converting a scalar *back* into a case.

```php
<?php
declare(strict_types=1);

// cases() — works on pure AND backed enums
foreach (HttpStatus::cases() as $case) {
    echo "{$case->name} = {$case->value}\n";
}
// Output:
// OK = 200
// NotFound = 404
// ServerError = 500

// from() — backed only; throws ValueError on no match
$s = HttpStatus::from(404);        // HttpStatus::NotFound
$x = HttpStatus::from(999);        // ValueError: 999 is not a valid backing value for enum HttpStatus

// tryFrom() — backed only; returns null on no match (no exception)
$ok   = HttpStatus::tryFrom(200);  // HttpStatus::OK
$none = HttpStatus::tryFrom(999);  // null
```

**Rule of thumb:** use `tryFrom()` for untrusted input (form data, query params) so you can handle the miss gracefully; use `from()` when a miss is a programmer error you *want* to crash on.

### Methods, constants, and interfaces on enums

Enums can declare methods (including static ones), constants, and implement interfaces. They **cannot** have non-constant *properties* or *state* (cases are the only data), and they can't be instantiated with `new` or extended.

```php
<?php
declare(strict_types=1);

interface HasLabel {
    public function label(): string;
}

enum Priority: int implements HasLabel {
    case Low    = 1;
    case Medium = 2;
    case High   = 3;

    // Enum constant
    const DEFAULT = self::Medium;

    // Instance method — note `$this` is the current case
    public function label(): string {
        return match ($this) {
            Priority::Low    => 'Low priority',
            Priority::Medium => 'Normal',
            Priority::High   => 'Urgent!',
        };
    }

    public function isCritical(): bool {
        return $this === self::High;
    }

    // Static factory / helper
    public static function fromScore(int $score): self {
        return match (true) {
            $score >= 80 => self::High,
            $score >= 40 => self::Medium,
            default      => self::Low,
        };
    }
}

echo Priority::High->label();        // "Urgent!"
var_dump(Priority::Low->isCritical()); // bool(false)
echo Priority::fromScore(90)->name;  // "High"
echo Priority::DEFAULT->name;        // "Medium"
```

### Enums in `match`

`match` (PHP 8.0+) uses **strict `===` comparison** and is exhaustive-friendly, making it the natural companion to enums. Because enum cases are singletons, `===` is exactly right.

```php
<?php
declare(strict_types=1);

function colorFor(Priority $p): string {
    return match ($p) {
        Priority::Low    => 'green',
        Priority::Medium => 'amber',
        Priority::High   => 'red',
    };
    // No `default` needed if all cases are covered. If a future case
    // is added and not handled, match throws UnhandledMatchError —
    // a useful safety net.
}
```

### Enums vs class constants

| | Class constants | Enums |
|---|---|---|
| Type safety | ❌ `const A = 1;` is just an `int` | ✅ A param typed `Status` rejects raw ints |
| Forgeable | Any `int`/`string` passes | Only the defined cases exist |
| Iterate all values | Manual / reflection | `Status::cases()` |
| Attach behaviour | Not really | Methods, interfaces |
| IDE autocomplete | Weak | Full |
| Switch safety | `switch` falls through, no exhaustiveness | `match` + `UnhandledMatchError` |

```php
<?php
// Old way — nothing stops you passing 99 or "banana":
class OrderStatusConst {
    const PENDING = 'pending';
    const SHIPPED = 'shipped';
}
function ship(string $status) { /* any string accepted */ }

// Enum way — the type IS the constraint:
enum OrderStatus: string {
    case Pending = 'pending';
    case Shipped = 'shipped';
}
function shipBetter(OrderStatus $status) { /* only valid cases compile */ }
```

---

## 9. Enums in Laravel 12

Laravel embraces enums across the framework. Highlights you should know for interviews:

**Eloquent attribute casting** — back a column with an enum and Laravel converts both ways automatically:

```php
<?php
// app/Models/Order.php
namespace App\Models;

use App\Enums\OrderStatus;
use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    protected function casts(): array   // Laravel 11/12 method form
    {
        return [
            'status' => OrderStatus::class,
        ];
    }
}

// Usage:
$order->status = OrderStatus::Pending;   // stored as 'pending' in DB
$order->status instanceof OrderStatus;   // true on retrieval
```

> In Laravel 10 you'd use the `protected $casts = [...]` property; Laravel 11+ prefers the `casts()` method. Both still work in 12.

**Validation** with the `Enum` rule:

```php
<?php
use Illuminate\Validation\Rule;

$request->validate([
    'status' => ['required', Rule::enum(OrderStatus::class)],
]);
```

**Implicit route model binding** for backed enums — type-hint an enum on a route/controller and Laravel resolves the path segment via `tryFrom`, returning 404 on an invalid value:

```php
<?php
// routes/web.php
use App\Enums\OrderStatus;

Route::get('/orders/{status}', function (OrderStatus $status) {
    return $status->name;
});
// GET /orders/pending → "Pending"
// GET /orders/banana  → 404
```

In Blade, enum cases work naturally:

```blade
@foreach (App\Enums\OrderStatus::cases() as $status)
    <option value="{{ $status->value }}">{{ $status->name }}</option>
@endforeach
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Putting `declare(strict_types=1)` anywhere but the top.**
   It must be the *first statement* (after only the opening `<?php`). A blank line or a comment is fine; any executable statement before it is a fatal `Fatal error: strict_types declaration must be the very first statement`. **Fix:** make it line 1 (after `<?php`).

2. **Assuming `strict_types` is global.**
   It's per-file and governed by the *calling* file. A strict library function called from a coercive file is still coerced. **Fix:** add `declare(strict_types=1)` to *every* file; don't rely on one file to protect another.

3. **Reading a typed property before initializing it.**
   `Error: Typed property X::$y must not be accessed before initialization` — this is NOT the same as `null`. **Fix:** give the property a default, initialize it in the constructor (promotion is easiest), or make it nullable if "no value" is a legitimate state. Use `isset()` to test safely.

4. **Using `from()` on user input.**
   `OrderStatus::from($_GET['status'])` throws an uncaught `ValueError` and a 500 the moment someone tweaks the URL. **Fix:** use `tryFrom()` and handle the `null`, or wrap `from()` in validation.

5. **Expecting `float` parameters to accept numeric strings in strict mode.**
   `f("9.5")` where `f(float $x)` throws `TypeError` under `strict_types=1`. Only `int→float` widening survives strict mode. **Fix:** cast explicitly (`(float) $input`) or validate/convert at the boundary.

6. **Forgetting `readonly` is shallow.**
   `public readonly array $items;` stops reassignment of the array, but `public readonly Cart $cart;` does not stop mutating the `Cart`'s internals. **Fix:** make the nested objects immutable too, or clone in a wither.

7. **Trying to add state or `new` an enum.**
   `new Status()` is a fatal error; enums can't have mutable properties. People try to store per-instance data on a case and are surprised. **Fix:** model variable data outside the enum; use methods + `match` for case-dependent behaviour.

8. **Implicit-nullable defaults in PHP 8.4.**
   `function f(string $x = null)` now emits a deprecation. **Fix:** write `?string $x = null`.

9. **Confusing arithmetic coercion with type-declaration coercion.**
   `"2 apples" + 3` is *operator* behaviour (a `Warning`, result `5`), not parameter coercion. The moment you declare `int $a`, that same `"2 apples"`/`"10abc"` is a hard `TypeError`. Don't reason about one from the other. **Fix:** validate/cast at the boundary; never rely on PHP "doing the right thing" with mixed strings.

10. **Putting `callable` in the wrong place.**
    `callable` is fine in a *union* (`callable|string`) but is a fatal error inside an *intersection type* and as a *property type* (`Property cannot have type callable`). **Fix:** use `Closure` as a property/intersection-friendly first-class callable type when you need to store one.

11. **Reaching for `gettype()` and comparing to `"float"`.**
    `gettype(1.5)` returns `"double"`, not `"float"`, and `gettype($obj)` is always `"object"` (never the class). **Fix:** use `get_debug_type()`, whose output matches type-declaration names and reports the real class.

---

## ✅ Best Practices

- **Always `declare(strict_types=1)`** at the top of every file. Let Pint/PHP-CS-Fixer enforce it.
- **Type everything you can**: parameters, return types, and properties. Reserve `mixed` for genuine "any" cases (e.g. JSON decode results) and narrow as soon as possible.
- **Prefer the narrowest accurate type.** `iterable` over `array` if you accept generators; a precise union over `mixed`; a specific class over `object`.
- **Use `?T` explicitly** rather than relying on a `null` default to imply nullability.
- **Return `static` (not `self`)** from base-class factories and fluent setters so subclasses keep their concrete type.
- **Use `readonly` + promotion** for value objects and DTOs; combine with a `with*()` wither for controlled "mutation."
- **Reach for enums** instead of class-constant bags whenever a value is one of a fixed set; type your parameters with the enum so the type *is* the validation.
- **`tryFrom()` for untrusted input, `from()` for trusted/internal** conversions.
- **Let `match` enforce exhaustiveness** — omit `default` when handling all enum cases so a newly added case surfaces as an `UnhandledMatchError` instead of silently slipping through.
- **Put behaviour on the enum** (label, color, permissions) via methods rather than scattering `switch` statements across the codebase.
- **Prefer `get_debug_type()` over `gettype()`** for logging and error messages — its names match what you write in type declarations and it reports the real class for objects.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between coercive and strict typing in PHP, and how do you switch modes?**
A: Coercive (default) silently converts convertible scalar arguments to the declared scalar type (e.g. `"9.5"` → `9.5`, `true` → `1`). Strict mode (`declare(strict_types=1)`) disables that — types must match exactly, with the sole exception of `int→float` widening. You enable strict mode per-file with `declare(strict_types=1)` as the first statement; it applies to the file where the *call* is made.

**Q2. Is `declare(strict_types=1)` global or per-file? Which file's setting wins?**
A: Per-file. The mode that applies to a function call is the mode of the file containing the **call site**, not the file defining the function. So a strict file calling a coercive-defined function is still checked strictly, and vice versa.

**Q3. What does "Typed property must not be accessed before initialization" mean, and how is it different from null?**
A: A typed property with no default starts *uninitialized* — a distinct third state, not `null`. Reading it before any write throws an `Error`, because the engine won't fabricate a `null` that would violate the declared type. Fix by defaulting it, initializing in the constructor, or declaring it nullable. `isset()` returns `false` for both uninitialized and null.

**Q4. Explain union, intersection, and DNF types with an example of each.**
A: Union `A|B` = A *or* B (`int|string`). Intersection `A&B` = satisfies both, class/interface members only (`Countable&Traversable`). DNF = unions of parenthesised intersection groups, e.g. `(Countable&Traversable)|null`, introduced in 8.2 to allow mixing the two.

**Q5. `void` vs `never` — what's the distinction?**
A: `void` means the function returns control but no value. `never` (8.1+) means the function *never* returns control at all — it always throws or exits. `never` is the bottom type; analysers treat code after a `never` call as unreachable, which is handy in `match` arms and guard clauses.

**Q6. How does `static` differ from `self` as a return type?**
A: `self` resolves to the class where the method is written; `static` uses late static binding to resolve to the actually-called class. Return `static` from base-class factory/fluent methods so a subclass call returns the subclass type, not the base type.

**Q7. (Under the hood) How are enum cases represented at runtime, and why is `===` safe for them?**
A: Each enum case is a *singleton object* — an instance of the enum class created once and cached by the engine. There's exactly one `Status::Active` object, so identity comparison (`===`) is reliable and cheap; that's also why `match` (which uses `===`) is the idiomatic way to branch on enums. Enums can't be instantiated with `new`, cloned, or serialized as fresh objects (they serialize by name), preserving the singleton guarantee. Backed enums also maintain an internal value→case map that powers `from()`/`tryFrom()`.

**Q8. When would you choose an enum over class constants?**
A: Whenever the value is one of a fixed set. Enums give type safety (a parameter typed as the enum rejects raw scalars), prevent forging invalid values, allow iteration via `cases()`, support methods/interfaces, and pair with `match` for exhaustiveness. Class constants are just scalars with no type guarantees.

**Q9. What is variance, and which kinds does PHP support?**
A: Variance describes how method types may change in overrides. PHP supports **covariant return types** (override may return a subtype) and **contravariant parameter types** (override may accept a supertype), enforcing Liskov substitution. Property types are invariant.

**Q10. What does `readonly` actually guarantee, and what are its limits?**
A: A `readonly` property can be assigned once, from within the declaring class's scope, and never again — enforced at runtime with an `Error` on a second write. Limits: it requires a type, can't have a declaration default, and is *shallow* (nested objects remain mutable). PHP 8.3 added re-initialization during `__clone()` for deep-copy support, and 8.2 added whole-class `readonly`.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
declare(strict_types=1);          // ALWAYS first; per-file; governs call site

// ── Scalar & nullable ──────────────────────────────
function f(int $a, ?string $b = null): float {}

// ── Composite ──────────────────────────────────────
int|string                         // union (OR)
Countable&Traversable              // intersection (AND), classes/interfaces only
(A&B)|null                         // DNF: OR of parenthesised AND-groups

// ── Special types ──────────────────────────────────
mixed     // any value incl null (top type)
void      // returns no value           (return only)
never     // never returns; throws/exits(return only, bottom type)
object    // any object
iterable  // array|Traversable
self      // class where written
static    // late-bound called class    (return: covariant)
null|false|true   // standalone literal types (8.2 for params/props)

// ── Properties ─────────────────────────────────────
class C {
    public int $x;                 // typed; UNINITIALIZED until written
    public ?int $y = null;         // nullable, defaults to null
    public readonly string $id;    // write-once, needs a type, no default
}
readonly class Dto { /* all props readonly */ }   // 8.2+

// ── Enums ──────────────────────────────────────────
enum Suit {}                        // pure: ->name only
enum Status: string {               // backed: ->name and ->value
    case Active = 'active';
    case Done   = 'done';
    const DEFAULT = self::Active;    // enum constant
    public function label(): string { return match($this) {/*...*/}; }
}
Status::cases()                      // [] of all cases (pure & backed)
Status::from('active')              // case or ValueError       (backed)
Status::tryFrom('nope')            // case or null             (backed)
Status::Active->name               // "Active"
Status::Active->value              // "active"                 (backed)
Status::Active === Status::Active  // true (singletons)

// ── Mode behaviour ─────────────────────────────────
// strict:   "9" -> int param  => TypeError; int -> float OK
// coercive: "9" -> int param  => 9; true -> int => 1

// ── Inspecting runtime types ───────────────────────
gettype(1.5)         // "double"  (legacy names: integer/double/boolean/NULL)
get_debug_type(1.5)  // "float"   (8.0+; matches type-declaration names + class FQN)
```

```php
// Variance recap
class P { function get(): Animal {} function set(Dog $d): void {} }
class C extends P {
    function get(): Dog {}          // ✅ covariant return (narrower)
    function set(Animal $a): void {}// ✅ contravariant param (wider)
}
```

---

## 🧪 Mini Exercises

1. **Mode detective.** Write two files: `lib.php` defines `function half(int $n): float`, and `app.php` (with `declare(strict_types=1)`) requires it and calls `half("10")` and `half(10)`. Predict and explain the result of each call. Then remove `strict_types` from `app.php` and predict again.

2. **Immutable money.** Build a `readonly` value object `Money` with promoted `int $cents` and `string $currency` properties, an `add(Money $other): self` method that throws a `never`-returning helper `mismatch()` when currencies differ, and an `amount(): string` formatter. Ensure attempting `$m->cents = 0` throws.

3. **Status enum with behaviour.** Create a backed `enum TicketStatus: string` (`Open`, `InProgress`, `Closed`) implementing an interface `HasColor { public function color(): string; }`. Add `label()` and a static `default(): self`. Write a function `transition(TicketStatus $from): TicketStatus` using `match` that returns the next logical status with no `default` arm.

4. **DNF in practice.** Declare interfaces `Serializable2` and `Cacheable`, and a function `store((Serializable2&Cacheable)|string $item): void`. Implement a class that satisfies the intersection and call `store()` both with an instance and with a plain string; explain why a class implementing only one interface is rejected.

5. **Enum vs constants refactor.** Take this snippet and refactor it to an enum, then update the function signature so invalid values are impossible to pass:
   ```php
   class Role { const ADMIN = 'admin'; const EDITOR = 'editor'; const VIEWER = 'viewer'; }
   function canPublish(string $role): bool {
       return in_array($role, [Role::ADMIN, Role::EDITOR], true);
   }
   ```
