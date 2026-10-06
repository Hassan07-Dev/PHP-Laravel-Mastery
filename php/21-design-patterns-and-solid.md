# Design Patterns & SOLID Principles

> **Module 21 — Zero to Expert PHP/Laravel**
> Target: PHP **8.4** (notes for 8.1–8.3 where relevant), Laravel **12** (notes for 10/11 where behavior differs).

Design patterns are **reusable solutions to recurring design problems**. They are not libraries you install; they are *shapes* your code can take. SOLID is a set of five principles that keep object-oriented code flexible and maintainable. Together they form the vocabulary senior engineers use in code reviews and interviews ("let's extract a strategy here", "this violates the dependency inversion principle"). Laravel itself is built almost entirely out of these patterns, so learning them makes the framework feel obvious instead of magic.

---

## **What you'll learn**

- The five **SOLID** principles, each with a "bad → good" PHP refactor.
- The most common **creational, structural, and behavioral** patterns — intent, a minimal PHP example, and the real Laravel feature that uses each.
- Why **Singleton** is usually an anti-pattern, and what to use instead.
- How **Dependency Injection (DI)** and **Inversion of Control (IoC)** actually work, and what a **DI container** does under the hood (tied to Laravel's `Container`).
- How **MVC**, the **service layer**, and **action classes** organize a real application.
- **When NOT to reach for a pattern** — over-engineering is a real cost.
- Interview-ready answers, common mistakes, and a cheat sheet.

---

## 1. Why patterns and SOLID exist (the WHY)

Code is read far more often than it is written, and it *changes* constantly. The enemy is **coupling**: when changing one thing forces you to change ten others. Patterns and SOLID are tools to **localize change** — to make "I need to add a new payment provider" a small, additive edit rather than a risky surgery across the codebase.

Two ideas recur throughout this module:

- **Program to an interface, not an implementation.** Depend on *what* something does, not *how* it does it.
- **Favor composition over inheritance.** Build behavior by *combining* objects rather than building tall, brittle inheritance trees.

Keep these in mind; almost every pattern is one of these two ideas wearing a costume.

---

## 2. SOLID Principles

SOLID is an acronym coined by Robert C. Martin: **S**ingle Responsibility, **O**pen/Closed, **L**iskov Substitution, **I**nterface Segregation, **D**ependency Inversion.

### 2.1 S — Single Responsibility Principle (SRP)

> **A class should have one, and only one, reason to change.**

"Reason to change" means *one actor / concern*. A class that parses a CSV, validates it, saves it to the DB, and emails a report has four reasons to change.

```php
<?php
// ❌ BAD: one class doing parsing, persistence, and notification
class ReportManager
{
    public function generate(array $rows): void
    {
        $csv = $this->toCsv($rows);          // formatting concern
        file_put_contents('report.csv', $csv); // persistence concern
        mail('boss@co.com', 'Report', $csv);   // notification concern
    }

    private function toCsv(array $rows): string { /* ... */ return ''; }
}
```

```php
<?php
// ✅ GOOD: each class has a single reason to change
final class CsvFormatter
{
    public function format(array $rows): string { /* ... */ return ''; }
}

final class FileStorage
{
    public function save(string $path, string $contents): void
    {
        file_put_contents($path, $contents);
    }
}

final class ReportMailer
{
    public function send(string $to, string $body): void
    {
        mail($to, 'Report', $body);
    }
}

// A thin coordinator wires them together (this is the "service layer", see §7)
final class ReportService
{
    public function __construct(
        private CsvFormatter $formatter,
        private FileStorage $storage,
        private ReportMailer $mailer,
    ) {}

    public function run(array $rows): void
    {
        $csv = $this->formatter->format($rows);
        $this->storage->save('report.csv', $csv);
        $this->mailer->send('boss@co.com', $csv);
    }
}
```

**Laravel analogue:** Form Requests (validation), Mailables (email), and Jobs (background work) each isolate one responsibility instead of cramming everything into a controller.

### 2.2 O — Open/Closed Principle (OCP)

> **Software entities should be open for extension, but closed for modification.**

You should be able to add new behavior **without editing existing, tested code**. The classic smell is a growing `switch`/`match` on a type.

```php
<?php
// ❌ BAD: every new shape forces you to edit this method
class AreaCalculator
{
    public function area(object $shape): float
    {
        return match (true) {
            $shape instanceof Circle    => 3.14159 * $shape->r ** 2,
            $shape instanceof Rectangle => $shape->w * $shape->h,
            // add Triangle? edit here again...
        };
    }
}
```

```php
<?php
// ✅ GOOD: extend by adding a new class, not editing AreaCalculator
interface Shape
{
    public function area(): float;
}

final class Circle implements Shape
{
    public function __construct(public float $r) {}
    public function area(): float { return 3.14159 * $this->r ** 2; }
}

final class Rectangle implements Shape
{
    public function __construct(public float $w, public float $h) {}
    public function area(): float { return $this->w * $this->h; }
}

final class AreaCalculator
{
    public function total(Shape ...$shapes): float
    {
        return array_sum(array_map(fn (Shape $s) => $s->area(), $shapes));
    }
}

echo (new AreaCalculator())->total(new Circle(2), new Rectangle(3, 4));
// Output: 24.56636
```

**Laravel analogue:** Notification *channels*. Adding an SMS channel doesn't require editing the notification system — you implement the channel contract and register it.

### 2.3 L — Liskov Substitution Principle (LSP)

> **Subtypes must be substitutable for their base types without breaking the program.**

If `B extends A`, any code that works with an `A` must keep working when handed a `B`. Violations usually show up as a subclass that *strengthens preconditions* or *throws* where the parent didn't (the infamous `Square extends Rectangle`).

```php
<?php
// ❌ BAD: Square breaks the contract callers expect from Rectangle
class Rectangle
{
    public function __construct(protected int $w, protected int $h) {}
    public function setWidth(int $w): void  { $this->w = $w; }
    public function setHeight(int $h): void { $this->h = $h; }
    public function area(): int { return $this->w * $this->h; }
}

class Square extends Rectangle
{
    public function setWidth(int $w): void  { $this->w = $this->h = $w; } // surprise!
    public function setHeight(int $h): void { $this->w = $this->h = $h; }
}

function resize(Rectangle $r): void
{
    $r->setWidth(5);
    $r->setHeight(4);
    assert($r->area() === 20); // holds for Rectangle, FAILS for Square (returns 16)
}
```

```php
<?php
// ✅ GOOD: don't force a false "is-a". Model the real abstraction.
interface Shape
{
    public function area(): int;
}

final class Rectangle implements Shape
{
    public function __construct(private int $w, private int $h) {}
    public function area(): int { return $this->w * $this->h; }
}

final class Square implements Shape
{
    public function __construct(private int $side) {}
    public function area(): int { return $this->side ** 2; }
}
```

**Rule of thumb:** if overriding a method requires you to *throw* an exception or quietly *ignore* arguments, you are probably violating LSP. **Laravel analogue:** Eloquent's query builder returns the same `Builder` contract whether you query MySQL, PostgreSQL, or SQLite — drivers are substitutable.

### 2.4 I — Interface Segregation Principle (ISP)

> **No client should be forced to depend on methods it does not use.**

Prefer many small, focused interfaces over one fat interface. A fat interface forces implementers to write empty/`throw` stubs.

```php
<?php
// ❌ BAD: a "robot worker" is forced to implement eat()/sleep()
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class Robot implements Worker
{
    public function work(): void  { /* ok */ }
    public function eat(): void   { throw new \LogicException('robots do not eat'); }
    public function sleep(): void { throw new \LogicException('robots do not sleep'); }
}
```

```php
<?php
// ✅ GOOD: split responsibilities into role interfaces
interface Workable { public function work(): void; }
interface Feedable { public function eat(): void; }

final class Human implements Workable, Feedable
{
    public function work(): void { /* ... */ }
    public function eat(): void  { /* ... */ }
}

final class Robot implements Workable
{
    public function work(): void { /* ... */ }
}
```

**Laravel analogue:** Laravel ships tiny contracts (`Illuminate\Contracts\Cache\Store`, `Queue`, `Mail\Mailer`) instead of one giant `Framework` interface. You depend only on the slice you need.

### 2.5 D — Dependency Inversion Principle (DIP)

> **High-level modules should not depend on low-level modules. Both should depend on abstractions.** And: abstractions should not depend on details; details should depend on abstractions.

This is the principle that makes DI containers useful. Don't `new` a concrete class inside business logic; depend on an interface and let it be injected.

```php
<?php
// ❌ BAD: OrderService is welded to a specific gateway
final class StripeGateway
{
    public function charge(int $cents): void { /* ... */ }
}

final class OrderService
{
    private StripeGateway $gateway;
    public function __construct() { $this->gateway = new StripeGateway(); } // hard dependency

    public function checkout(int $cents): void { $this->gateway->charge($cents); }
}
```

```php
<?php
// ✅ GOOD: depend on an abstraction; the concrete is injected
interface PaymentGateway
{
    public function charge(int $cents): void;
}

final class StripeGateway implements PaymentGateway
{
    public function charge(int $cents): void { /* ... */ }
}

final class FakeGateway implements PaymentGateway // trivially testable
{
    public function charge(int $cents): void { /* record for assertions */ }
}

final class OrderService
{
    public function __construct(private PaymentGateway $gateway) {}
    public function checkout(int $cents): void { $this->gateway->charge($cents); }
}
```

**Laravel analogue:** This is *exactly* what the service container does. You bind `PaymentGateway::class` to `StripeGateway::class` once, and every constructor that type-hints `PaymentGateway` gets it automatically (see §6).

> **DI vs DIP:** Dependency *Injection* is the mechanism (passing dependencies in). Dependency *Inversion* is the principle (depend on abstractions). DI is one way to achieve DIP.

---

## 3. Creational Patterns

These deal with **how objects are created**, decoupling client code from concrete construction.

### 3.1 Singleton — and why it's often an anti-pattern

**Intent:** Ensure a class has exactly one instance and provide a global access point.

```php
<?php
final class Config
{
    private static ?Config $instance = null;
    private array $data = [];

    private function __construct() {}          // can't `new` from outside
    private function __clone() {}              // can't clone
    public function __wakeup(): void           // can't recreate via unserialize()
    {
        throw new \LogicException('Cannot unserialize a singleton');
    }

    public static function instance(): self
    {
        return self::$instance ??= new self(); // null-coalescing assignment (PHP 7.4+)
    }

    public function set(string $k, mixed $v): void { $this->data[$k] = $v; }
    public function get(string $k): mixed { return $this->data[$k] ?? null; }
}

Config::instance()->set('env', 'prod');
echo Config::instance()->get('env'); // Output: prod
```

**Why it's usually an anti-pattern:**

- It's **global mutable state** in disguise — hard to reason about, order-dependent.
- It **hides dependencies**: a class calling `Config::instance()` doesn't declare that it needs config in its constructor.
- It **wrecks testability**: the static instance leaks between tests; you can't easily swap a fake.
- It **violates SRP** (the class manages both its job *and* its lifecycle) and **DIP** (callers depend on the concrete class).

**What to use instead:** Let a DI container manage a *single shared instance* (a "singleton scope") and **inject it**. You get one-instance semantics without global access. In Laravel:

```php
<?php
// Container-managed singleton — same single instance, but injectable & swappable
$this->app->singleton(Config::class, fn () => new Config());
// Now type-hint Config::class in any constructor; tests can rebind it.
```

> **Key distinction for interviews:** "Singleton the *pattern*" (static `getInstance`) is the anti-pattern. "Singleton the *lifetime/scope*" managed by a container is fine and common.

### 3.2 Factory Method

**Intent:** Define an interface for creating an object, but let the chosen *subclass/branch* decide which concrete class to instantiate. It centralizes `new` so the rest of the code depends only on the interface.

```php
<?php
interface Notifier { public function send(string $msg): void; }

final class EmailNotifier implements Notifier {
    public function send(string $msg): void { /* ... */ }
}
final class SmsNotifier implements Notifier {
    public function send(string $msg): void { /* ... */ }
}

enum Channel: string { case Email = 'email'; case Sms = 'sms'; }

final class NotifierFactory
{
    public function make(Channel $channel): Notifier
    {
        return match ($channel) {
            Channel::Email => new EmailNotifier(),
            Channel::Sms   => new SmsNotifier(),
        };
    }
}

$notifier = (new NotifierFactory())->make(Channel::Sms);
$notifier->send('Hi');
```

**Laravel analogue:** `Cache::store('redis')`, `Storage::disk('s3')`, `DB::connection('reporting')` — managers are factories that hand back the right concrete driver by name. Eloquent **model factories** (`UserFactory`) are a database-seeding application of the same idea.

### 3.3 Abstract Factory

**Intent:** Create **families of related objects** without specifying their concrete classes. Use it when products must be used together (e.g. a UI theme's button + checkbox must match).

```php
<?php
interface Button { public function render(): string; }
interface Checkbox { public function render(): string; }

final class DarkButton   implements Button   { public function render(): string { return '[dark btn]'; } }
final class DarkCheckbox implements Checkbox { public function render(): string { return '[dark chk]'; } }
final class LightButton  implements Button   { public function render(): string { return '[light btn]'; } }
final class LightCheckbox implements Checkbox{ public function render(): string { return '[light chk]'; } }

interface UiFactory
{
    public function button(): Button;
    public function checkbox(): Checkbox;
}

final class DarkUiFactory implements UiFactory
{
    public function button(): Button     { return new DarkButton(); }
    public function checkbox(): Checkbox { return new DarkCheckbox(); }
}

function renderForm(UiFactory $ui): string
{
    return $ui->button()->render() . $ui->checkbox()->render();
}

echo renderForm(new DarkUiFactory()); // Output: [dark btn][dark chk]
```

**Difference from Factory Method:** Factory Method makes *one* product; Abstract Factory makes a *family* of related products. **Laravel analogue:** database connection factories produce a matched set (connector + query grammar + schema grammar) per driver.

### 3.4 Builder

**Intent:** Construct a complex object **step by step**, especially when there are many optional parameters. Avoids "telescoping constructors" with 8 positional args.

```php
<?php
final class Query
{
    public function __construct(
        public readonly string $table,
        public readonly array $wheres,
        public readonly ?int $limit,
    ) {}
}

final class QueryBuilder
{
    private array $wheres = [];
    private ?int $limit = null;

    public function __construct(private string $table) {}

    public function where(string $col, string $op, mixed $val): static
    {
        $this->wheres[] = [$col, $op, $val];
        return $this; // fluent: return self for chaining
    }

    public function limit(int $n): static { $this->limit = $n; return $this; }

    public function build(): Query
    {
        return new Query($this->table, $this->wheres, $this->limit);
    }
}

$q = (new QueryBuilder('users'))
        ->where('active', '=', true)
        ->limit(10)
        ->build();
```

**Laravel analogue:** The **Eloquent / Query Builder** itself (`User::where('active', true)->limit(10)->get()`) is the canonical builder. So is the `Mail` fluent API and `Http::withHeaders()->post()`.

---

## 4. Structural Patterns

These deal with **how objects are composed** into larger structures.

### 4.1 Adapter

**Intent:** Convert one interface into another that clients expect. Wrap a third-party/legacy class so it fits *your* contract.

```php
<?php
interface Logger { public function log(string $level, string $message): void; }

// Third-party class with an incompatible API we can't change:
final class MonologLike
{
    public function addRecord(int $severity, string $text): void { /* ... */ }
}

final class MonologAdapter implements Logger
{
    public function __construct(private MonologLike $monolog) {}

    public function log(string $level, string $message): void
    {
        $severity = ['info' => 200, 'error' => 400][$level] ?? 100;
        $this->monolog->addRecord($severity, $message); // translate the call
    }
}
```

**Laravel analogue:** Filesystem disks adapt different backends (local, S3, FTP) to a single `Storage` interface via Flysystem adapters. Cache stores adapt Redis/Memcached/file to one `Store` contract.

### 4.2 Decorator

**Intent:** Add behavior to an object **dynamically** by wrapping it, without changing its class. Each decorator implements the same interface and delegates to the wrapped instance — composition, not inheritance.

```php
<?php
interface DataSource { public function read(): string; }

final class FileSource implements DataSource
{
    public function __construct(private string $contents) {}
    public function read(): string { return $this->contents; }
}

final class UppercaseDecorator implements DataSource
{
    public function __construct(private DataSource $inner) {}
    public function read(): string { return strtoupper($this->inner->read()); }
}

final class ExclaimDecorator implements DataSource
{
    public function __construct(private DataSource $inner) {}
    public function read(): string { return $this->inner->read() . '!!!'; }
}

$source = new ExclaimDecorator(new UppercaseDecorator(new FileSource('hello')));
echo $source->read(); // Output: HELLO!!!
```

**Laravel analogue:** **Middleware** is essentially a decorator pipeline around the request/response — each middleware wraps the next, adding behavior (auth, throttling, CORS) layer by layer. The underlying `Illuminate\Pipeline\Pipeline` class is the generic engine that composes these wrappers.

### 4.3 Facade

**Intent:** Provide a **simple, unified interface** to a complex subsystem. Hide many moving parts behind one convenient entry point.

```php
<?php
// A plain (GoF) facade: one simple method hides several subsystems
final class OrderFacade
{
    public function __construct(
        private InventoryService $inventory,
        private PaymentService $payment,
        private ShippingService $shipping,
    ) {}

    public function placeOrder(int $productId, int $cents): void
    {
        $this->inventory->reserve($productId);
        $this->payment->charge($cents);
        $this->shipping->schedule($productId);
    }
}
```

**Laravel analogue:** Laravel **Facades** (`Cache::get()`, `Route::get()`, `DB::table()`) are a *specific implementation* of this idea. They are static-looking proxies that resolve the real object from the container under the hood — `Cache::get()` is roughly `app('cache')->get()`. They are technically a Facade + Service Locator hybrid. (See the Service Locator caveat in *Common Mistakes #6* — overusing them in core logic hides dependencies.)

```php
<?php
// What a Laravel facade does behind the scenes:
abstract class Facade
{
    abstract protected static function getFacadeAccessor(): string;

    public static function __callStatic(string $method, array $args): mixed
    {
        $instance = app(static::getFacadeAccessor()); // resolve from container
        return $instance->$method(...$args);
    }
}
```

### 4.4 Proxy

**Intent:** Provide a **stand-in** for another object to control access — for lazy loading, caching, logging, or access control. The proxy implements the same interface as the real subject.

```php
<?php
interface Image { public function display(): string; }

final class RealImage implements Image
{
    public function __construct(private string $file)
    {
        // expensive load happens here
    }
    public function display(): string { return "showing {$this->file}"; }
}

final class LazyImageProxy implements Image
{
    private ?RealImage $real = null;
    public function __construct(private string $file) {}

    public function display(): string
    {
        $this->real ??= new RealImage($this->file); // create only on first use
        return $this->real->display();
    }
}
```

**Laravel analogue:** Eloquent **lazy-loaded relationships** behave like proxies (the query fires only when you access `$user->posts`). Laravel can also build a true lazy proxy from the container via `Container::factory()` / a closure binding, and **lazy collections** (`Illuminate\Support\LazyCollection`, plus the query builder's `->lazy()` / `cursor()`) defer work item-by-item in the same spirit.

### 4.5 Repository (as used in Laravel)

**Intent:** Abstract data access behind a collection-like interface so business logic doesn't know *how* data is stored (Eloquent, raw SQL, an API, an in-memory array for tests).

```php
<?php
interface UserRepository
{
    public function find(int $id): ?User;
    public function all(): array;
    public function save(User $user): void;
}

// Production implementation backed by Eloquent
final class EloquentUserRepository implements UserRepository
{
    public function find(int $id): ?User { return User::find($id); }
    public function all(): array { return User::all()->all(); }
    public function save(User $user): void { $user->save(); }
}

// Test implementation — no database needed
final class InMemoryUserRepository implements UserRepository
{
    private array $users = [];
    public function find(int $id): ?User { return $this->users[$id] ?? null; }
    public function all(): array { return array_values($this->users); }
    public function save(User $user): void { $this->users[$user->id] = $user; }
}
```

Bind it in a service provider:

```php
<?php
// app/Providers/AppServiceProvider.php
public function register(): void
{
    $this->app->bind(UserRepository::class, EloquentUserRepository::class);
}
```

**Caveat (important for interviews):** Eloquent **already is** an implementation of the Active Record + Repository ideas. Wrapping every model in a repository purely "for the pattern" is often over-engineering and fights the framework. Use a repository when you need to (a) swap storage backends, (b) hide complex queries behind a named method, or (c) decouple domain code from Eloquent for a richer domain model. Otherwise, query scopes and dedicated query/action classes are usually enough.

---

## 5. Behavioral Patterns

These deal with **how objects communicate and distribute responsibility**.

### 5.1 Strategy

**Intent:** Define a family of interchangeable algorithms, encapsulate each one, and make them swappable at runtime. This is OCP in action — add a new strategy without touching the context.

```php
<?php
interface ShippingStrategy { public function cost(float $weight): float; }

final class FlatRate implements ShippingStrategy {
    public function cost(float $weight): float { return 5.00; }
}
final class WeightBased implements ShippingStrategy {
    public function cost(float $weight): float { return $weight * 1.50; }
}

final class Cart
{
    public function __construct(private ShippingStrategy $shipping) {}
    public function shippingCost(float $weight): float
    {
        return $this->shipping->cost($weight);
    }
}

echo (new Cart(new WeightBased()))->shippingCost(4); // Output: 6
```

**Laravel analogue:** Auth **guards** and **drivers** (session vs token), queue connections, broadcast drivers, and password hashers (bcrypt vs argon) are selectable strategies.

### 5.2 Observer

**Intent:** Define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically. Decouples the "thing that happened" from the "things that react".

```php
<?php
interface Observer { public function handle(string $event): void; }

final class Subject
{
    /** @var Observer[] */
    private array $observers = [];

    public function subscribe(Observer $o): void { $this->observers[] = $o; }

    public function fire(string $event): void
    {
        foreach ($this->observers as $o) {
            $o->handle($event);
        }
    }
}

final class AuditLog implements Observer {
    public function handle(string $event): void { /* write log */ }
}
```

**Laravel analogue:** The **Events & Listeners** system (`Event::dispatch()`, listeners), and **Eloquent model observers** (`created`, `updated`, `deleted` hooks). `UserObserver` reacting to `User::created` is the textbook Observer pattern.

### 5.3 Template Method

**Intent:** Define the **skeleton** of an algorithm in a base class, deferring specific steps to subclasses. The base controls the order; subclasses fill in the blanks.

```php
<?php
abstract class ReportExporter
{
    // The template method — fixed sequence, not overridable.
    final public function export(array $data): string
    {
        $header = $this->header();
        $body   = $this->body($data);
        return $header . "\n" . $body;
    }

    abstract protected function header(): string;        // step left to subclasses
    abstract protected function body(array $data): string;
}

final class CsvExporter extends ReportExporter
{
    protected function header(): string { return 'id,name'; }
    protected function body(array $data): string
    {
        return implode("\n", array_map(fn ($r) => "{$r['id']},{$r['name']}", $data));
    }
}
```

**Laravel analogue:** The base `Illuminate\Console\Command` defines the run lifecycle and you override `handle()`. Test case `setUp()`/`tearDown()` and the base `Mailable`/`Notification` lifecycle follow the same shape.

### 5.4 Command

**Intent:** Encapsulate a request as an object, so you can queue it, log it, undo it, or pass it around. Decouples the *invoker* from the *receiver*.

```php
<?php
interface Command { public function execute(): void; }

final class SendWelcomeEmail implements Command
{
    public function __construct(private int $userId) {}
    public function execute(): void { /* send the email */ }
}

final class CommandBus
{
    public function dispatch(Command $command): void
    {
        $command->execute(); // could log, queue, wrap in a transaction, etc.
    }
}

(new CommandBus())->dispatch(new SendWelcomeEmail(42));
```

**Laravel analogue:** **Queued Jobs** are commands — a job object holds its data and a `handle()` method, and the queue can store, retry, and run it later. The framework also ships a literal **command bus** (`Bus::dispatch()`). Artisan **commands** are the same pattern applied to the CLI.

---

## 6. Dependency Injection & Inversion of Control

### 6.1 The concepts

- **Inversion of Control (IoC):** Instead of your code calling into a framework/creating its own dependencies, the framework calls your code and *gives* it what it needs. "Don't call us, we'll call you."
- **Dependency Injection (DI):** A concrete way to do IoC — dependencies are **passed in** (usually via the constructor) rather than created inside the class.
- **DI Container (a.k.a. IoC container / service container):** An object that knows how to **build and wire** your objects automatically.

```php
<?php
// Constructor injection — the standard, preferred form
final class InvoiceService
{
    public function __construct(
        private PaymentGateway $gateway, // an abstraction
        private Logger $logger,
    ) {}
}
```

Other (less preferred) forms: **setter injection** (`setLogger()`) and **method injection** (Laravel injects type-hinted dependencies into controller *methods*). Prefer constructor injection because it makes dependencies **required and explicit** and the object **immutable** once built.

### 6.2 How a DI container works under the hood

A container is, at its core: a map of `id → "how to build it"`, plus **autowiring** via reflection. Here is a minimal, real container to demystify the magic:

```php
<?php
final class Container
{
    private array $bindings = [];   // id => closure factory
    private array $instances = [];  // id => shared singleton instance

    public function bind(string $id, Closure $factory): void
    {
        $this->bindings[$id] = $factory;
    }

    public function singleton(string $id, Closure $factory): void
    {
        $this->bind($id, function (Container $c) use ($id, $factory) {
            return $this->instances[$id] ??= $factory($c);
        });
    }

    public function make(string $id): object
    {
        if (isset($this->bindings[$id])) {
            return ($this->bindings[$id])($this);
        }
        return $this->resolve($id); // autowire by reflection
    }

    private function resolve(string $class): object
    {
        $ref = new ReflectionClass($class);
        $ctor = $ref->getConstructor();

        if ($ctor === null) {
            return new $class();               // no dependencies
        }

        $args = [];
        foreach ($ctor->getParameters() as $param) {
            $type = $param->getType();
            if ($type instanceof ReflectionNamedType && ! $type->isBuiltin()) {
                $args[] = $this->make($type->getName()); // recurse into dependencies
            } elseif ($param->isDefaultValueAvailable()) {
                $args[] = $param->getDefaultValue();
            } else {
                throw new RuntimeException("Cannot resolve \${$param->getName()}");
            }
        }
        return $ref->newInstanceArgs($args);
    }
}
```

Walkthrough: `make(InvoiceService::class)` → reflection sees the constructor needs `PaymentGateway` and `Logger` → it recursively `make()`s each (following any bindings, e.g. `PaymentGateway → StripeGateway`) → builds the whole object graph for you. **This is exactly the shape of Laravel's `Illuminate\Container\Container`** (with far more features: contextual bindings, tagging, scoped/singleton lifetimes, method injection, etc.).

### 6.3 In Laravel

```php
<?php
// Binding in a service provider's register() method
$this->app->bind(PaymentGateway::class, StripeGateway::class);      // new each resolve
$this->app->singleton(Clock::class, fn () => new SystemClock());     // one shared instance
$this->app->scoped(RequestId::class, fn () => new RequestId());      // one per request/job (L8.47+)

// Contextual binding: different concrete depending on who needs it
$this->app->when(ReportController::class)
          ->needs(PaymentGateway::class)
          ->give(PayPalGateway::class);

// Resolving (you rarely call this directly — the container injects for you)
$service = app(InvoiceService::class);
```

```php
<?php
// Method injection: Laravel reads the controller method's type-hints and injects them
class OrderController
{
    public function store(Request $request, InvoiceService $invoices)
    {
        // both $request and $invoices are resolved from the container automatically
    }
}
```

> **L11/L12 note:** Application bootstrapping moved to a streamlined `bootstrap/app.php`, and there are far fewer default service providers, but the container's binding/resolving API (`bind`, `singleton`, `scoped`, `when`/`needs`/`give`) is unchanged from Laravel 10.

---

## 7. MVC, the Service Layer, and Action Classes

### 7.1 MVC

**MVC** (Model–View–Controller) separates concerns:

- **Model** — data and the rules about it (Eloquent models, plus domain logic).
- **View** — presentation (Blade templates, or a JSON resource for APIs).
- **Controller** — the thin coordinator: receive request → invoke business logic → return a response.

```php
<?php
// Controller stays THIN — it orchestrates, it doesn't compute.
class InvoiceController
{
    public function __construct(private InvoiceService $invoices) {}

    public function store(StoreInvoiceRequest $request) // validation lives in the FormRequest
    {
        $invoice = $this->invoices->create($request->validated());
        return redirect()->route('invoices.show', $invoice);
    }
}
```

```blade
{{-- resources/views/invoices/show.blade.php — the View --}}
<h1>Invoice #{{ $invoice->id }}</h1>
<p>Total: {{ $invoice->total }}</p>
```

**Fat controllers are the most common architectural smell.** Push logic down into services/actions and models.

### 7.2 Service Layer

A **service** is a class that holds a chunk of business logic, coordinating models and other services. It keeps controllers thin and logic reusable (a controller *and* an Artisan command can both call the same service).

```php
<?php
final class InvoiceService
{
    public function __construct(
        private InvoiceRepository $repo,
        private PaymentGateway $gateway,
    ) {}

    public function create(array $data): Invoice
    {
        return DB::transaction(function () use ($data) {
            $invoice = $this->repo->save(new Invoice($data));
            $this->gateway->charge($invoice->total_cents);
            return $invoice;
        });
    }
}
```

### 7.3 Action Classes (single-purpose services)

An **action** (a.k.a. single-action class or "invokable") is a class that does *exactly one thing*. It's SRP taken to its logical end and is very popular in modern Laravel. Easy to name, test, and queue.

```php
<?php
final class PublishPost
{
    public function __construct(private Clock $clock) {}

    public function __invoke(Post $post): Post   // invokable: call it like a function
    {
        $post->published_at = $this->clock->now();
        $post->save();
        event(new PostPublished($post));
        return $post;
    }
}

// Usage — the container resolves dependencies even for __invoke:
$publish = app(PublishPost::class);
$publish($post);
```

**Service vs Action:** A *service* groups related operations (`InvoiceService::create/cancel/refund`); an *action* is one operation per class (`CreateInvoice`, `CancelInvoice`). Many teams use actions for write operations and keep services for broader coordination. Both are correct — consistency within a codebase matters more than which you pick.

---

## 8. When NOT to use a pattern

Patterns have a cost: indirection, more files, more cognitive overhead. Misapplied, they make code *harder* to follow.

- **YAGNI ("You Aren't Gonna Need It").** Don't add an interface + factory + strategy for code that has exactly one implementation and no concrete plan for a second. Add the seam when the second case actually arrives.
- **Don't wrap Eloquent in a repository reflexively.** Eloquent is already a data-access abstraction; an unnecessary repository layer often just forwards calls and adds friction.
- **Don't reach for Singleton** when a container-managed shared instance does the job.
- **Don't build an Abstract Factory** for two unrelated objects — that's just two `new`s.
- **Premature abstraction is worse than duplication.** "Rule of three": tolerate a little duplication; extract a pattern once you've seen the same shape ~3 times and the abstraction is *clear*.
- **A `match`/`switch` on a type is fine** for small, stable, rarely-changing sets. Only refactor to Strategy/Polymorphism when it's churning or growing.

> The mature instinct isn't "which pattern do I use here?" — it's "does this change actually need a seam, and is the seam worth the indirection?"

---

## ⚠️ Common Mistakes & Gotchas

1. **Confusing the Singleton *pattern* with a container *singleton scope*.**
   - *Mistake:* Using a static `getInstance()` for shared state, then struggling to mock it in tests.
   - *Fix:* Bind it as `$app->singleton(...)` and inject it. Same single-instance behavior, but swappable and testable.

2. **Liskov violations via "is-a" inheritance.**
   - *Mistake:* `Square extends Rectangle`, or a subclass that `throw`s in an overridden method, breaking callers that worked with the base type.
   - *Fix:* Model the real abstraction (share an `interface`), and prefer composition. If overriding forces a throw/ignore, the inheritance is wrong.

3. **Fat controllers / fat models doing everything.**
   - *Mistake:* Validation, business rules, payment calls, and email all inside a controller method.
   - *Fix:* Validation → Form Request; logic → service/action; side effects → events/jobs. Controller just orchestrates.

4. **Newing dependencies inside business logic (`new StripeGateway()` in a service).**
   - *Mistake:* Hard-coded concrete dependency → can't test, can't swap.
   - *Fix:* Depend on an interface, inject it (DIP). Let the container wire the concrete.

5. **Repository-everything over-engineering.**
   - *Mistake:* A repository class per model that just forwards `find`/`all`/`save` to Eloquent.
   - *Fix:* Only introduce a repository for a real reason (swap backend, encapsulate complex queries, decouple domain). Otherwise use query scopes/actions.

6. **Facades everywhere = hidden dependencies (Service Locator smell).**
   - *Mistake:* Calling `Cache::`, `Auth::`, `DB::` deep inside services, hiding what the class needs and making unit tests rely on global state.
   - *Fix:* Inject the underlying contracts (`Repository`/`Guard`/`ConnectionInterface`) into the constructor. Reserve facades for controllers/quick scripts. (Laravel facades are testable via `Cache::fake()` etc., but explicit injection is cleaner for core logic.)

---

## ✅ Best Practices

- **Program to interfaces** for dependencies that have (or may plausibly have) multiple implementations or that you need to fake in tests.
- **Prefer constructor injection.** Required dependencies become explicit and the object is fully formed once constructed.
- **Favor composition over inheritance.** Reach for inheritance only for genuine, stable "is-a" relationships.
- **Keep controllers thin**; push logic into services/actions; keep models focused on data + domain rules.
- **Use enums + `match`** for closed, stable type sets; switch to Strategy/polymorphism when the set grows or churns.
- **Name patterns in your code** (`...Repository`, `...Factory`, `...Strategy`, `...Action`) so intent is self-documenting.
- **Mark classes `final`** by default; open them for extension deliberately. Use `readonly` properties for value objects.
- **Let the container manage lifetimes** (`bind` vs `singleton` vs `scoped`) instead of hand-rolling lifecycle code.
- **Apply YAGNI:** add the seam when the requirement is real, not speculative.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What does SOLID stand for, and which principle do DI containers most directly support?**
A: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion. DI containers most directly support **Dependency Inversion** — high-level code depends on abstractions, and the container supplies the concrete implementation.

**Q2. Why is the Singleton pattern often called an anti-pattern? When is "singleton" actually fine?**
A: The classic static-`getInstance` Singleton is global mutable state: it hides dependencies, couples callers to a concrete class, and leaks between tests. It's "fine" as a **lifetime/scope** managed by a DI container (`$app->singleton(...)`) — you still get one shared instance, but it's injected and swappable.

**Q3. (Under the hood) How does Laravel's service container resolve a class you never explicitly bound?**
A: **Autowiring via reflection.** It inspects the constructor with `ReflectionClass`/`ReflectionParameter`, reads each parameter's type-hint, and recursively resolves each non-builtin dependency (honoring any registered bindings), then calls `newInstanceArgs`. Builtins with defaults use their default; unresolvable parameters throw a `BindingResolutionException`.

**Q4. Difference between Factory Method and Abstract Factory?**
A: Factory Method creates **one** product (decides which concrete class to return). Abstract Factory creates a **family of related products** that are meant to be used together, behind a factory interface.

**Q5. Strategy vs Template Method — both vary behavior. How do they differ?**
A: **Strategy** uses *composition* — you inject an interchangeable algorithm object at runtime. **Template Method** uses *inheritance* — a base class fixes the algorithm's skeleton and subclasses override specific steps. Strategy is more flexible (swap at runtime, no inheritance); Template Method is simpler when steps vary but the sequence is fixed.

**Q6. Is a Laravel Facade the same as the GoF Facade pattern?**
A: Related but not identical. The GoF Facade is a simple interface over a complex subsystem. A Laravel facade is a static **proxy** that resolves the real service from the container (`__callStatic` → `app(accessor)->method()`), so it's a Facade + Service Locator hybrid. It provides convenience but can hide dependencies if overused in core logic.

**Q7. Give a concrete LSP violation and how you'd fix it.**
A: `Square extends Rectangle`: setting width on a `Square` also changes height, so code that sets width=5, height=4 and expects area=20 breaks. Fix: don't force the inheritance — both implement a `Shape` interface with their own `area()`.

**Q8. How do you decide whether to add a Repository in a Laravel app?**
A: Add it only for a real need: swapping storage backends, encapsulating complex/named queries, or decoupling a rich domain model from Eloquent for testability. If it would just forward `find/all/save` to Eloquent, skip it — that's over-engineering.

**Q9. What's the difference between DI and IoC?**
A: IoC is the broad principle — control of flow/creation is inverted to the framework. DI is a specific technique to achieve IoC: dependencies are passed in (typically via the constructor) rather than created internally.

**Q10. Where does Laravel use the Observer pattern, and how does it relate to events?**
A: Eloquent **model observers** (`created`, `updated`, `deleted`) and the **Events/Listeners** system. Both are Observer: a subject (model/event) notifies many subscribers (observer methods/listeners) without knowing who they are, decoupling the cause from the reactions.

---

## 📋 Quick Reference / Cheat Sheet

**SOLID**

| Letter | Principle | One-liner |
|---|---|---|
| S | Single Responsibility | One reason to change per class |
| O | Open/Closed | Extend without modifying |
| L | Liskov Substitution | Subtypes must be drop-in for base |
| I | Interface Segregation | Many small interfaces > one fat one |
| D | Dependency Inversion | Depend on abstractions, inject concretes |

**Patterns → Laravel analogue**

| Pattern | Type | Laravel example |
|---|---|---|
| Singleton (scope) | Creational | `$app->singleton()` |
| Factory Method | Creational | `Cache::store()`, model factories |
| Abstract Factory | Creational | DB connection factories |
| Builder | Creational | Eloquent/Query Builder, `Http::` |
| Adapter | Structural | Filesystem/Cache drivers (Flysystem) |
| Decorator | Structural | Middleware, `Pipeline` |
| Facade | Structural | `Cache::`, `Route::`, `DB::` |
| Proxy | Structural | Lazy-loaded relations, lazy bindings |
| Repository | Structural | Custom `*Repository` over Eloquent |
| Strategy | Behavioral | Auth guards, queue/broadcast drivers |
| Observer | Behavioral | Events/Listeners, model observers |
| Template Method | Behavioral | base `Command`, `setUp()/tearDown()` |
| Command | Behavioral | Queued Jobs, `Bus::dispatch()`, Artisan |

**Container API**

```php
$app->bind(I::class, C::class);                 // new instance each resolve
$app->singleton(I::class, fn () => new C());     // one shared instance
$app->scoped(I::class, fn () => new C());         // one per request/job
$app->when(A::class)->needs(I::class)->give(B::class); // contextual
app(Service::class);                              // resolve (usually auto-injected)
```

**Decision quick-rules**

- Need to swap behavior at runtime → **Strategy**
- Need to add behavior by wrapping → **Decorator**
- React to "something happened" → **Observer / Events**
- Hide complex construction → **Builder / Factory**
- One class, one job → **Action class**
- Only one implementation, no real need → **no pattern (YAGNI)**

---

## 🧪 Mini Exercises

1. **SRP refactor.** Take a `UserController@register` method that validates input, hashes the password, saves the user, and sends a welcome email. Refactor it into: a Form Request (validation), a `RegisterUser` action class (logic), and an event + listener (welcome email). Inject the password hasher rather than calling a static.

2. **Strategy + OCP.** Build a `DiscountCalculator` with three swappable strategies (`PercentageDiscount`, `FixedDiscount`, `NoDiscount`) implementing a `Discount` interface. Make the calculator accept the strategy via constructor injection. Prove that adding a fourth strategy requires zero edits to existing classes.

3. **DI container.** Extend the minimal `Container` from §6.2 to support `singleton()` semantics *and* a `when()->needs()->give()` contextual binding. Write a small object graph (`A` depends on interface `B`, which has two implementations) and resolve `A` two different ways.

4. **Decorator.** Given a `PriceProvider` interface returning a base price, write a `TaxDecorator` and a `DiscountDecorator` that each wrap a `PriceProvider`. Stack them so tax is applied after the discount, and verify the final number by hand.

5. **Spot the violation.** Find a real piece of code (yours or open-source) where a subclass overrides a parent method to throw `NotSupportedException`. Explain which SOLID principle it violates and rewrite it using interface segregation so no client depends on a method it can't use.
