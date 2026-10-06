# OOP Advanced: Inheritance, Abstraction, Interfaces & Traits

> Target: **PHP 8.4** (notes for 8.1–8.3 where it matters) and **Laravel 12** (notes for Laravel 10/11 where behavior differs).

This module takes you from "I know what a class is" to "I can reason about inheritance hierarchies, design with interfaces, reuse code with traits, and explain late static binding in an interview." Every concept is grounded in a runnable example and tied back to how Laravel itself uses these features.

## **What you'll learn**

- How `extends` works, how to **override** methods/properties, and how to call back into the parent with `parent::`.
- When to seal a design with **`final`** classes and methods, and what that buys you.
- The difference between **abstract classes** and **interfaces** — and a decision rule for picking one.
- **Polymorphism**: writing code against an interface so it works with any implementation.
- **Traits** for horizontal code reuse, including conflict resolution (`insteadof` / `as`), abstract/static methods, and trait properties.
- **Late static binding** — what `static::` and `new static()` actually resolve to, and why `self::` is different.
- **Covariance & contravariance** in return and parameter types, and the rules PHP enforces.
- Why senior engineers reach for **composition over inheritance**, with a concrete refactor.

---

## 1. Inheritance: reusing and specializing behavior

**Inheritance** lets a class (the *child* or *subclass*) take on the properties and methods of another class (the *parent* or *superclass*) and then add to or change them. The keyword is `extends`.

**Why it exists:** to model an "is-a" relationship and avoid repeating shared behavior. A `Manager` *is an* `Employee`; a `PdfReport` *is a* `Report`. The child gets everything the parent has for free.

```php
<?php

class Employee
{
    public function __construct(
        protected string $name,
        protected float $baseSalary,
    ) {}

    public function monthlyPay(): float
    {
        return $this->baseSalary / 12;
    }

    public function describe(): string
    {
        return "{$this->name} earns {$this->monthlyPay()}/mo";
    }
}

class Manager extends Employee
{
    public function __construct(
        string $name,
        float $baseSalary,
        private float $bonus,
    ) {
        parent::__construct($name, $baseSalary); // run the parent's setup
    }

    // Override: change behavior inherited from Employee
    public function monthlyPay(): float
    {
        return parent::monthlyPay() + ($this->bonus / 12);
    }
}

$m = new Manager('Ada', 120_000, 24_000);
echo $m->describe();
// Output: Ada earns 12000/mo
```

Notice `describe()` was never redefined in `Manager`, yet it printed the *manager's* pay (12000, not 10000). That is the heart of inheritance and polymorphism working together: `describe()` calls `$this->monthlyPay()`, and `$this` is a `Manager`, so the **overridden** version runs. (Constructor promotion — declaring properties right in the constructor signature with `protected`/`private` — has been available since PHP 8.0.)

### Visibility and what gets inherited

- `public` and `protected` members are inherited and accessible in the child.
- `private` members are **not** accessible in the child (they belong to the declaring class only).
- Use `protected` when a child needs access; use `private` to keep an implementation detail sealed inside one class.

```php
<?php

class Base
{
    public string $pub = 'public';
    protected string $prot = 'protected';
    private string $priv = 'private';

    public function show(): void
    {
        // All three are visible inside the declaring class:
        echo "$this->pub $this->prot $this->priv";
    }
}

class Child extends Base
{
    public function peek(): void
    {
        echo $this->pub;   // OK
        echo $this->prot;  // OK
        // echo $this->priv; // Error: cannot access private property
    }
}
```

### Overriding properties

You can redeclare a property in a child to give it a different default. Since PHP 8.4, you can also have **asymmetric visibility** and **property hooks**, but the classic rule still holds: the child's redeclaration replaces the parent's default for instances of that child.

```php
<?php

class Animal
{
    public string $sound = 'generic noise';
}

class Dog extends Animal
{
    public string $sound = 'woof'; // override the default
}

echo (new Dog)->sound; // Output: woof
```

> **PHP 8.4 note:** PHP 8.4 added typed property *hooks* (`get`/`set`) and asymmetric visibility (e.g. `public private(set) string $x`). These interact with inheritance — a child can refine a hook — but the core override rules above are unchanged. Treat hooks as an advanced add-on, not a prerequisite.

---

## 2. `final`: sealing classes and methods

`final` is the opposite of "extend me." It tells the compiler (and other developers): *this is not meant to be subclassed or overridden.*

```php
<?php

final class Money
{
    public function __construct(public readonly int $cents) {}
}

// class Wallet extends Money {} // Fatal error: Class Wallet cannot extend final class Money

class Repository
{
    final public function connect(): void { /* fixed connection logic */ }
}

class UserRepository extends Repository
{
    // public function connect(): void {} // Fatal error: cannot override final method
}
```

**Why use it?**

- **Correctness guarantees:** value objects like `Money` rely on their invariants; subclassing could break them.
- **Refactoring freedom:** a `final` class can be changed internally without worrying about breaking unknown subclasses.
- **Performance & clarity:** the engine and readers both know the method binding is fixed.

A common idiom: *"design for inheritance or prohibit it."* If you don't have a clear extension story, mark the class `final`. You can always remove `final` later — but you can never safely add it once people subclass.

> **Version note:** `final` for **class constants** (e.g. `final public const X = 1;`) was added in **PHP 8.1**, not 8.4 — it prevents a child class or interface from redefining the constant. (Typed class constants like `final public const int X = 1;` are the part that needs **PHP 8.3+**.)

---

## 3. Abstract classes: partial implementations you must complete

An **abstract class** is a class you cannot instantiate directly. It exists to be extended. It can mix **concrete** methods (with bodies) and **abstract** methods (signatures only — the child *must* implement them).

**Why it exists:** to capture shared behavior *and* enforce that subclasses fill in the gaps. It is a template with holes.

```php
<?php

abstract class Report
{
    // Concrete: shared by all reports
    final public function generate(): string
    {
        return $this->header() . "\n" . $this->body();
    }

    protected function header(): string
    {
        return "=== Report generated " . date('Y-m-d') . " ===";
    }

    // Abstract: every concrete report MUST supply this
    abstract protected function body(): string;
}

class SalesReport extends Report
{
    protected function body(): string
    {
        return "Total sales: \$42,000";
    }
}

// $r = new Report();          // Fatal error: Cannot instantiate abstract class Report
$r = new SalesReport();
echo $r->generate();
// Output:
// === Report generated 2026-06-18 ===
// Total sales: $42,000
```

The `generate()` method here is the **Template Method** pattern: the parent defines the skeleton (header + body), the child fills in the variable step (`body()`). Marking `generate()` `final` stops a subclass from breaking the skeleton.

Rules to remember:

- A class with **at least one** abstract method must itself be declared `abstract`.
- Abstract methods have **no body** (just a semicolon). The child's implementation must be **compatible** (same or wider visibility, compatible signature — see covariance below).
- Abstract classes **can** have constructors, properties, constants, and concrete methods.

Laravel leans on base classes and the Template Method pattern throughout. For example, `Illuminate\Http\Resources\Json\JsonResource` and `Illuminate\Foundation\Http\FormRequest` are base classes you extend and complete (you fill in `toArray()` / `rules()` while the framework runs the surrounding skeleton). `Illuminate\Database\Eloquent\Model` is itself a concrete base class you extend per table.

> **Accuracy note:** Not every "base" in Laravel is an *abstract* class. The console kernel, `Illuminate\Foundation\Console\Kernel`, is a **concrete** class that **implements** the interface `Illuminate\Contracts\Console\Kernel` — it is not abstract and does not extend itself. (In Laravel 11+/12's slimmed application skeleton, an `App\Console\Kernel` is no longer published by default; console bootstrapping moved into `bootstrap/app.php` and `routes/console.php`.) The lesson: Laravel mixes abstract classes, concrete base classes, and interfaces deliberately — look at the actual `class`/`abstract class`/`interface` keyword rather than assuming.

---

## 4. Interfaces: contracts without implementation

An **interface** is a pure contract: a list of method signatures (and optionally constants) with **no implementation**. A class that `implements` an interface promises to provide every method.

**Why it exists:** to decouple *what* from *how*. Callers depend on the interface; you can swap implementations freely. This is the foundation of dependency injection and testability.

```php
<?php

interface PaymentGateway
{
    public function charge(int $cents, string $token): string; // returns a transaction id
}

class StripeGateway implements PaymentGateway
{
    public function charge(int $cents, string $token): string
    {
        return "stripe_txn_" . substr($token, 0, 6);
    }
}

class FakeGateway implements PaymentGateway
{
    public function charge(int $cents, string $token): string
    {
        return "fake_txn_123"; // perfect for tests
    }
}
```

### A class can implement many interfaces

```php
<?php

interface Loggable { public function logLine(): string; }
interface Jsonable { public function toJson(): string; }

class Order implements Loggable, Jsonable
{
    public function __construct(private int $id, private int $total) {}

    public function logLine(): string { return "Order #{$this->id}"; }
    public function toJson(): string  { return json_encode(['id' => $this->id, 'total' => $this->total]); }
}
```

A class has **one** parent (single inheritance) but can implement **any number** of interfaces. This is how PHP gets the benefits of multiple inheritance for *types* without the diamond-problem ambiguity of multiple inheritance of *implementation*.

### Interface constants

Interfaces may declare constants. They are implicitly `public`. Constants are accessed via the interface name or any implementing class name.

The overriding rules changed over time, so be precise in an interview:

- **Before PHP 8.1:** an implementing class could **not** override an interface constant at all.
- **PHP 8.1 and later:** an implementing class **may** override an interface constant — *unless* the constant is declared `final` (the `final` modifier for constants also arrived in 8.1).
- **PHP 8.3 and later:** constants can be **typed** (`const int OK = 200;`).

```php
<?php

interface HttpStatus
{
    const int OK = 200;            // typed class constants: PHP 8.3+
    const int NOT_FOUND = 404;
}

class Response implements HttpStatus {}

echo HttpStatus::OK;     // 200
echo Response::NOT_FOUND; // 404 (inherited constant)
```

> **Version note:** *Typed* class constants (`const int OK = 200;`) require **PHP 8.3+**. On 8.1–8.2 write `const OK = 200;`.

Overriding an interface constant (PHP 8.1+), and locking one with `final`:

```php
<?php

interface Config
{
    const TIMEOUT = 30;              // overridable by implementers (8.1+)
    final const VERSION = '1.0';     // locked: implementers may NOT redefine it
}

class FastConfig implements Config
{
    const TIMEOUT = 5;               // OK on PHP 8.1+
    // const VERSION = '2.0';        // Fatal error: cannot override final constant
}

echo FastConfig::TIMEOUT; // 5
echo FastConfig::VERSION;  // 1.0 (inherited)
```

### Interfaces can extend multiple interfaces

Unlike classes, an interface may `extend` **several** interfaces at once, composing a larger contract:

```php
<?php

interface Readable { public function read(): string; }
interface Writable { public function write(string $data): void; }

interface ReadWrite extends Readable, Writable {}

class File implements ReadWrite
{
    private string $buffer = '';
    public function read(): string { return $this->buffer; }
    public function write(string $data): void { $this->buffer .= $data; }
}
```

Any class implementing `ReadWrite` must satisfy both `read()` and `write()`.

---

## 5. Abstract class vs interface: which one?

This is one of the most common interview questions. Here is the mental model.

| Question | Abstract class | Interface |
|---|---|---|
| Can it contain implemented methods? | Yes | No (only signatures) |
| Can it hold state (properties)? | Yes | No (constants only) |
| How many can a class take? | One (single inheritance) | Many |
| Relationship it models | "is-a" + shared code | "can-do" capability / contract |
| Can it have a constructor? | Yes | No |
| Constants? | Yes | Yes |

**Decision rule:**

- Reach for an **interface** when you only need to define a *capability* (`Comparable`, `Cacheable`, `PaymentGateway`) and multiple unrelated classes should be able to provide it. Interfaces maximize flexibility and are the default for dependency injection.
- Reach for an **abstract class** when implementations share *real code and state* and there is a genuine "is-a" hierarchy (`AbstractController`, `Report`).
- They compose: a very common, clean design is *interface for the contract + abstract base class that partially implements it.*

```php
<?php

interface Shape
{
    public function area(): float;
    public function name(): string;
}

abstract class AbstractShape implements Shape
{
    // Shared default; concrete shapes still must define area()
    public function name(): string
    {
        return static::class; // late static binding — see §8
    }
}

class Circle extends AbstractShape
{
    public function __construct(private float $r) {}
    public function area(): float { return M_PI * $this->r ** 2; }
}

echo (new Circle(2))->name(); // Output: Circle
```

---

## 6. Polymorphism via interface type hints

**Polymorphism** ("many forms") means code written against a *type* works with any object that satisfies that type. By type-hinting the **interface**, your function neither knows nor cares about the concrete class.

```php
<?php

function describeShapes(Shape ...$shapes): void
{
    foreach ($shapes as $shape) {
        printf("%s has area %.2f\n", $shape->name(), $shape->area());
    }
}

class Square extends AbstractShape
{
    public function __construct(private float $side) {}
    public function area(): float { return $this->side ** 2; }
}

describeShapes(new Circle(1), new Square(3));
// Output:
// Circle has area 3.14
// Square has area 9.00
```

`describeShapes()` will accept *any* future `Shape` implementation without modification — this is the **Open/Closed Principle** in action (open for extension, closed for modification).

In Laravel this is everywhere. The container resolves interfaces to concrete classes:

```php
<?php
// In a service provider (Laravel 12, same as 10/11):
$this->app->bind(PaymentGateway::class, StripeGateway::class);

// Anywhere you type-hint the interface, the container injects Stripe:
class CheckoutController
{
    public function __construct(private PaymentGateway $gateway) {}

    public function pay()
    {
        return $this->gateway->charge(4200, request('token'));
    }
}
```

Swap `StripeGateway` for `FakeGateway` in tests with one binding change — the controller never knows.

---

## 7. Traits: horizontal code reuse

A **trait** is a bundle of methods (and properties) you can "mix in" to a class with `use`. Because PHP has single inheritance, traits solve a real problem: **sharing behavior across classes that don't share a parent**. Think of a trait as copy-paste that the compiler does for you, at compile time.

```php
<?php

trait HasTimestamps
{
    public ?string $createdAt = null;

    public function touch(): void
    {
        $this->createdAt = date('c');
    }
}

trait Sluggable
{
    public function slug(string $title): string
    {
        return strtolower(str_replace(' ', '-', $title));
    }
}

class Post
{
    use HasTimestamps, Sluggable;
}

$p = new Post();
$p->touch();
echo $p->createdAt;             // e.g. 2026-06-18T10:00:00+00:00
echo $p->slug('Hello World');   // Output: hello-world
```

### Traits vs inheritance

- Inheritance models an **is-a** relationship and gives you one parent.
- A trait is **horizontal**: unrelated classes (`Post`, `Invoice`, `User`) can all `use HasTimestamps` without sharing a base class.
- A trait is **not a type** — you cannot type-hint against a trait. If you need polymorphism, pair the trait with an interface (the trait supplies the implementation, the interface supplies the type). Laravel does exactly this: `Illuminate\Support\Traits\Macroable`, the `Notifiable` trait, etc., often accompany an interface.

```php
<?php
// Common Laravel-style pairing:
interface Auditable { public function auditLabel(): string; }

trait ProvidesAudit
{
    public function auditLabel(): string
    {
        return static::class . '#' . ($this->id ?? '?');
    }
}

class Invoice implements Auditable
{
    use ProvidesAudit;
    public int $id = 7;
}

echo (new Invoice)->auditLabel(); // Output: Invoice#7
```

### Conflict resolution: `insteadof` and `as`

If two traits define a method with the same name, PHP raises a fatal error **unless** you resolve the conflict. Use `insteadof` to pick a winner, and `as` to alias the other (or to change visibility).

```php
<?php

trait FileLogger
{
    public function log(string $m): string { return "FILE: $m"; }
}

trait DbLogger
{
    public function log(string $m): string { return "DB: $m"; }
    public function flush(): string { return "flushed"; }
}

class Service
{
    use FileLogger, DbLogger {
        FileLogger::log insteadof DbLogger;   // FileLogger wins for log()
        DbLogger::log as logToDb;             // keep DbLogger's version under a new name
        DbLogger::flush as protected;         // change visibility WITHOUT renaming
    }

    public function run(): string
    {
        return $this->flush(); // flush() is now protected, callable from inside only
    }
}

$s = new Service();
echo $s->log('hi');     // Output: FILE: hi
echo $s->logToDb('hi'); // Output: DB: hi
echo $s->run();         // Output: flushed
```

> **Common mistake:** to change *only* the visibility of a trait method, write `DbLogger::flush as protected;` — **do not repeat the method name** (`DbLogger::flush as protected flush;`). Repeating the same name is treated as creating a second `flush()` alongside the original and triggers a fatal collision: *"Trait method DbLogger::flush has not been applied as Service::flush, because of collision with DbLogger::flush."* You only repeat a name when you genuinely want a *new* name (e.g. `as logToDb`).

Without the `insteadof` line, PHP fatals with: *"Trait method log has not been applied as Service::log, because of collision with ... "*.

### Abstract and static methods in traits

A trait can declare **abstract** methods to *require* the using class to implement something (a contract on the host class), and **static** methods/properties that become static members of the host.

```php
<?php

trait Identifiable
{
    abstract public function id(): int;     // host MUST provide this

    public function tag(): string
    {
        return static::prefix() . $this->id(); // calls host's static via late static binding
    }

    abstract protected static function prefix(): string;
}

class Customer
{
    use Identifiable;
    public function __construct(private int $cid) {}
    public function id(): int { return $this->cid; }
    protected static function prefix(): string { return 'CUST-'; }
}

echo (new Customer(99))->tag(); // Output: CUST-99
```

### Static properties in traits — a gotcha

Each class that uses a trait gets its **own copy** of the trait's static properties; they are *not* shared across all users of the trait.

```php
<?php

trait Counter
{
    public static int $count = 0;
    public static function inc(): void { static::$count++; }
}

class A { use Counter; }
class B { use Counter; }

A::inc(); A::inc();
B::inc();

echo A::$count; // Output: 2
echo B::$count; // Output: 1  (separate copy!)
```

### Trait properties — and the conflicting-property gotcha

Traits can carry instance properties (like `$createdAt` above), which become regular properties of the host. But if the **host class** (or another trait) declares a property of the **same name** with a *different* type or default value, PHP raises a fatal error — trait properties are not silently overridden the way methods can be.

```php
<?php

trait HasId
{
    public int $id = 0;
}

class Widget
{
    use HasId;
    // public int $id = 1;     // Fatal: "C and T define the same property ($id) ...
                               //         the definition differs and is considered incompatible"
    // public string $id = 'x'; // Fatal: incompatible type as well
}
```

An **identical** redeclaration (same type and default) is allowed, but it's noise — prefer to declare the property in exactly one place. This is one reason to keep state out of traits unless it's genuinely self-contained.

### Trait order and precedence

When a method name collides, precedence is: **the host class's own method** > **trait method** > **inherited (parent) method**. So a method defined directly in the class always wins over a trait method of the same name, and a trait method overrides one inherited from a parent.

---

## 8. Late static binding (LSB): `static::` vs `self::`

This is the deepest sub-topic here and a favorite "how does it work under the hood" question.

**The problem:** `self::` is resolved at *compile time* to the class where it's written. `static::` is resolved at *runtime* to the class that was actually **called** — the "called class." That runtime resolution is **late static binding**.

```php
<?php

class Model
{
    public static function create(): static  // return type `static` = the called class
    {
        return new static();   // LSB: instantiate the called class, not Model
    }

    public function who(): string
    {
        return self::class . " vs " . static::class;
    }
}

class User extends Model {}

$u = User::create();
var_dump($u instanceof User); // bool(true)  — new static() built a User, not a Model
echo $u->who();               // Output: Model vs User
```

Walk through `who()`:

- `self::class` is bound at compile time inside `Model`, so it's **always** `Model`.
- `static::class` is bound at runtime to whatever class the method was invoked through — here, `User`.

`new static()` is the engine behind factory/fluent APIs. It's why `User::query()`, `User::create()`, and Eloquent's static finders return the *right* model subclass rather than the base class. The `static` return type (PHP 8.0+) lets the type system follow along.

```php
<?php

class QueryBuilder
{
    private array $wheres = [];

    public static function start(): static { return new static(); }

    public function where(string $cond): static
    {
        $this->wheres[] = $cond;
        return $this; // fluent
    }

    public function toSql(): string
    {
        return 'WHERE ' . implode(' AND ', $this->wheres);
    }
}

echo QueryBuilder::start()->where('a = 1')->where('b = 2')->toSql();
// Output: WHERE a = 1 AND b = 2
```

**Under the hood:** the engine keeps two pieces of context for every method call — the *defining class* (used to resolve `self`, `parent`, and private members) and the *called/static class* (used to resolve `static::`). When you call `User::create()`, the called class is `User`; that value is forwarded through `static::` and `new static()`. A non-forwarding call like `Model::create()` resets the called class to `Model`. This is why `static::` is "late": it's not known until the call actually happens.

---

## 9. Covariance and contravariance

These describe how method signatures may change when overriding. PHP enforces the **Liskov Substitution Principle**: a subclass object must be usable anywhere the parent is expected.

- **Covariance (return types):** a child may **narrow** (return a more specific type) than the parent. Supported since PHP 7.4.
- **Contravariance (parameter types):** a child may **widen** (accept a more general type) than the parent. Supported since PHP 7.4.

```php
<?php

class Animal {}
class Dog extends Animal {}

class AnimalShelter
{
    public function adopt(): Animal { return new Animal(); }
}

class DogShelter extends AnimalShelter
{
    // COVARIANT return: Dog is a subtype of Animal — allowed (narrowing)
    public function adopt(): Dog { return new Dog(); }
}
```

```php
<?php

interface Comparator
{
    public function compare(Dog $a, Dog $b): int;
}

class FlexibleComparator implements Comparator
{
    // CONTRAVARIANT parameters: Animal is wider than Dog — allowed (widening)
    public function compare(Animal $a, Animal $b): int
    {
        return 0;
    }
}
```

**Intuition:** a `DogShelter` that *only ever returns Dogs* is still a valid `AnimalShelter` (callers expecting an `Animal` get one). A comparator that *accepts any Animal* is still a valid `Comparator` for `Dog` (it can handle Dogs and more). Going the other way is **forbidden**: a child cannot return a *wider* type or accept a *narrower* parameter type, because that would break callers relying on the parent's contract.

```php
<?php
// ILLEGAL — would not compile:
class BadShelter extends AnimalShelter
{
    // public function adopt(): object {} // wider return = Fatal error
}
```

The `static` and `self` return types are also covariant tools; `static` is the most specific possible return and is fully covariant with the parent's `self`/class return type.

---

## 10. Composition over inheritance

Inheritance is seductive but rigid: it couples the child to the parent's internals and locks you into one hierarchy. **Composition** — building behavior by *holding* other objects and delegating to them — is more flexible and the senior default. The guideline "favor composition over inheritance" (from *Design Patterns*, GoF) is heavily favored in interviews.

**Problem with deep inheritance:**

```php
<?php
// Fragile: every variation needs a new subclass, combinations explode.
class Notification {}
class EmailNotification extends Notification {}
class UrgentEmailNotification extends EmailNotification {} // and SmsUrgent? SlackUrgent?
```

**Composition refactor — inject the channel:**

```php
<?php

interface Channel
{
    public function send(string $to, string $message): void;
}

class EmailChannel implements Channel
{
    public function send(string $to, string $message): void
    {
        echo "Email to $to: $message\n";
    }
}

class SmsChannel implements Channel
{
    public function send(string $to, string $message): void
    {
        echo "SMS to $to: $message\n";
    }
}

class Notifier
{
    /** @param Channel[] $channels */
    public function __construct(private array $channels) {}

    public function notify(string $to, string $message): void
    {
        foreach ($this->channels as $channel) {
            $channel->send($to, $message);
        }
    }
}

$notifier = new Notifier([new EmailChannel(), new SmsChannel()]);
$notifier->notify('ada@example.com', 'Build passed');
// Output:
// Email to ada@example.com: Build passed
// SMS to ada@example.com: Build passed
```

Now adding a Slack channel is a new *class*, not a new branch in a brittle hierarchy, and you can mix channels per notification at runtime. **Rule of thumb:** use inheritance for true "is-a" with shared code; use composition (with interfaces) for "has-a"/"uses-a" and for varying behavior.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `self::` when you meant `static::`.**
   `self::` is frozen to the defining class at compile time, so `new self()` in a base class always builds the *base*, breaking subclass factories.
   **Fix:** use `new static()` and the `static` return type for any factory/fluent method meant to be inherited.

2. **Forgetting `parent::__construct()` in a child constructor.**
   When a child declares its own `__construct`, the parent's constructor does **not** run automatically. Promoted properties / setup in the parent silently never execute.
   **Fix:** explicitly call `parent::__construct(...)` (usually first) in the child's constructor.

3. **Unresolved trait method collisions.**
   Using two traits that both define `log()` is a fatal error, not a silent "last one wins."
   **Fix:** resolve with `Trait::method insteadof OtherTrait;` and optionally `OtherTrait::method as alias;`.

   **Sub-gotcha — changing only visibility:** to re-expose a trait method at a different visibility *without* renaming it, write `Trait::m as protected;` (no name after the modifier). Writing `Trait::m as protected m;` repeats the name and fatals with a self-collision. Repeat a name only when you actually want a *new* alias (`Trait::m as renamed;`).

4. **Trying to instantiate an abstract class or treat a trait as a type.**
   `new Report()` on an abstract class is a fatal error, and `function f(SomeTrait $x)` is invalid — traits are not types.
   **Fix:** instantiate a concrete subclass; for type-hinting, define an interface alongside the trait and hint the interface.

5. **Assuming `private` members are visible to children.**
   A child cannot see a parent's `private` property/method; a same-named `private` in the child is a *separate* member, which can produce surprising results.
   **Fix:** use `protected` when subclasses legitimately need access; keep truly internal details `private` and don't rely on inheriting them.

6. **Illegal variance: widening a return type or narrowing a parameter type when overriding.**
   `public function adopt(): object` overriding `: Animal` won't compile.
   **Fix:** narrow returns (covariant) and only widen parameters (contravariant); never the reverse.

7. **Expecting trait static properties to be shared across all users.**
   Each `use`-ing class gets its own copy of a trait's `static` property.
   **Fix:** if you need a single shared counter, put the static state in a dedicated class and reference it, not in a trait.

8. **Redeclaring a trait property with a different type or default in the host class.**
   `trait T { public int $x = 1; }` combined with `class C { use T; public int $x = 2; }` is a fatal error ("the definition differs and is considered incompatible") — not a silent override.
   **Fix:** declare the property in one place. If the host needs a different default, set it in the constructor rather than redeclaring the property.

---

## ✅ Best Practices

- **Program to interfaces, not implementations.** Type-hint contracts so the container/tests can swap implementations.
- **Mark classes `final` by default**; open them for inheritance only with a deliberate extension design.
- **Prefer composition over inheritance** for varying behavior; reserve inheritance for genuine "is-a" with shared code.
- **Keep inheritance hierarchies shallow** (ideally ≤ 2–3 levels). Deep trees are hard to reason about.
- **Pair traits with interfaces**: trait = reusable implementation, interface = the type for polymorphism.
- **Use the Template Method pattern with abstract classes**: concrete `final` skeleton + abstract steps for subclasses.
- **Use `static::` / `new static()` / `: static`** in any base method intended to be inherited by factories or fluent builders.
- **Make the smallest visibility that works** the default: `private` first, `protected` only when a subclass truly needs it.
- **Don't put state in traits** unless it's clearly per-instance and self-contained; shared state belongs in a real object.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between an abstract class and an interface, and when do you pick each?**
A: An abstract class can hold implemented methods, properties/state, and constructors, and a class can extend only one. An interface is a pure contract (signatures + constants), and a class can implement many. Use an interface for a capability/contract that unrelated classes provide; use an abstract class when implementations share real code and state in an "is-a" hierarchy. They combine well: interface for the type, abstract base for shared default behavior.

**Q2. (Under the hood) How does late static binding work, and how is `static::` different from `self::`?**
A: `self::` is resolved at compile time to the class in which the code is written. `static::` is resolved at runtime to the *called class*. The engine tracks two contexts per call — the defining class (for `self`/`parent`/private access) and the called/static class (for `static::`). Forwarding calls (`static::`, `parent::`, `self::`) preserve the called class; a direct call like `Base::method()` resets it. This is what makes `new static()` build the correct subclass in factory methods.

**Q3. Why does PHP have traits when it has interfaces and inheritance?**
A: PHP has single inheritance, so a class can't extend two parents to reuse code from both. Traits provide *horizontal* reuse — mixing concrete methods/properties into classes that don't share an ancestor. They're compile-time copy-paste, not a type, so they complement (not replace) interfaces.

**Q4. How do you resolve a method-name conflict between two traits?**
A: With `insteadof` to choose which trait's method wins, and `as` to expose the other under an alias (which can also change visibility). Without resolution, PHP throws a fatal collision error.

**Q5. Explain covariance and contravariance in PHP method overriding.**
A: Return types are covariant — a child may return a more specific (narrower) type. Parameter types are contravariant — a child may accept a more general (wider) type. Both have been allowed since PHP 7.4 and enforce the Liskov Substitution Principle so a subtype is always usable where the supertype is expected.

**Q6. What does `final` do and why use it?**
A: `final` on a class prevents subclassing; on a method prevents overriding; on a class/interface constant (since PHP 8.1) prevents redefinition. It guarantees invariants, frees you to refactor internals safely, and signals intent. The maxim: "design for inheritance or prohibit it."

**Q7. Can you type-hint a trait? Can you instantiate an abstract class?**
A: No to both. Traits are not types — to get polymorphism, define an interface and have the trait's host class implement it. Abstract classes cannot be instantiated; you instantiate a concrete subclass.

**Q8. What is polymorphism and how does Laravel use it?**
A: Polymorphism lets one piece of code operate on many concrete types via a shared interface/parent type. Laravel's service container binds interfaces to implementations, so type-hinting an interface (e.g. a `PaymentGateway`) lets you swap the concrete class (Stripe vs a fake in tests) without touching the consumer.

**Q9. If a class uses a trait and also inherits a method of the same name from its parent, which wins?**
A: Precedence is: the class's own method > the trait method > the inherited parent method. So the trait overrides the parent, but a method defined directly in the class beats the trait.

**Q10. Why "composition over inheritance"?**
A: Inheritance tightly couples a child to a parent's implementation and forces a single rigid hierarchy where feature combinations explode. Composition injects collaborators behind interfaces, letting behavior vary at runtime, improving testability, and following the Open/Closed and Single Responsibility principles.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
// INHERITANCE
class Child extends Base {}              // single inheritance ('Parent' is reserved)
parent::method();                        // call overridden parent method
parent::__construct(...);                // must call manually in child ctor

// FINAL
final class C {}                         // no subclassing
final public function m() {}             // no overriding
final public const X = 1;                // PHP 8.1+: no constant redefinition

// ABSTRACT
abstract class A {
    abstract public function must(): T;  // no body; subclass must implement
    public function shared(): void {}     // concrete, inherited
}                                         // cannot `new A()`

// INTERFACE
interface I { public function m(): T; }  // signatures only
interface J extends I1, I2 {}            // extend MANY interfaces
class C implements I, J {}               // implement MANY interfaces
const int CODE = 200;                     // typed const: PHP 8.3+

// TRAITS
trait T { public function m() {} }
class C {
    use T1, T2 {
        T1::m insteadof T2;              // pick winner on conflict
        T2::m as mAlias;                 // alias the loser
        T2::m as protected;              // change visibility via alias
    }
}
// trait may declare: abstract methods (host must implement), static members

// LATE STATIC BINDING
self::class      // compile-time: defining class
static::class    // runtime: called class
new static()     // build the called subclass (factories)
public function f(): static {}           // covariant "called class" return

// VARIANCE (PHP 7.4+)
// return type:    child may NARROW (covariant)
// parameter type: child may WIDEN  (contravariant)
```

**One-line picks:**
- Need a capability many unrelated classes provide → **interface**.
- Need shared code + state + "is-a" → **abstract class**.
- Need to share methods across unrelated classes → **trait** (pair with interface for typing).
- Need behavior to vary at runtime → **composition**, not inheritance.

---

## 🧪 Mini Exercises

1. **Template Method:** Create an abstract `Exporter` class with a `final` method `export(array $rows): string` that calls abstract `formatRow(array $row): string` and `extension(): string`. Implement `CsvExporter` and `JsonExporter`. Verify you cannot `new Exporter()`.

2. **Interfaces + polymorphism:** Define an interface `Discount { public function apply(int $cents): int; }`. Implement `PercentageDiscount` and `FlatDiscount`. Write a `Cart::total(Discount $d): int` method that works with either, then add a third discount type *without* modifying `Cart`.

3. **Trait conflict resolution:** Write two traits `JsonSerializerTrait` and `XmlSerializerTrait`, each with a `serialize()` method. Use both in a `Document` class so that `serialize()` defaults to JSON but XML is still callable as `serializeXml()`. Confirm the unresolved version fatals before you fix it.

4. **Late static binding:** Build a base `Entity` with `public static function make(): static { return new static(); }` and a `protected static function table(): string` that returns `static::class`. Subclass it as `Product` and `Order`; show that `Product::make()` returns a `Product` and that `table()` reflects the subclass name.

5. **Composition refactor:** Take a small inheritance chain (`Logger` → `FileLogger` → `TimestampedFileLogger`) and refactor it into a `Logger` that composes a `Formatter` interface and a `Writer` interface. Demonstrate swapping the formatter at construction time without subclassing.
