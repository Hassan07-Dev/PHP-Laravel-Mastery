# Service Container & Service Providers

> **Module 14 — Zero to Expert: Laravel**
> Target: **PHP 8.4** (notes for 8.1–8.3) · **Laravel 12** (notes for Laravel 10/11)

The service container is the beating heart of Laravel. Almost every feature you've ever used — routing, the database, queues, mail, validation, even the `request()` you type without thinking — is wired together by this one object. Once you truly understand the container, Laravel stops feeling like magic and starts feeling like a set of predictable, composable tools. This module takes you from "what is dependency injection?" all the way to deferred providers and how facades resolve under the hood.

---

## **What you'll learn**

- What **Inversion of Control (IoC)** and **Dependency Injection (DI)** are, and why a *container* is the natural solution.
- How Laravel **automatically resolves** concrete classes with zero configuration (autowiring via reflection).
- Every flavor of **binding**: `bind`, `singleton`, `scoped`, `instance`, and binding **interfaces to implementations**.
- **Contextual binding** (`when` / `needs` / `give`), **tagging**, **extending** bindings, and binding **primitives**.
- How to **resolve** services: `make`, `resolve()`, the `app()` helper, and automatic injection via type-hints.
- **Service Providers**: the difference between `register()` and `boot()`, creating them with `make:provider`, and registering them (`config/app.php` vs. Laravel 11+ `bootstrap/providers.php`).
- **Contextual attributes** (`#[Config]`, `#[Tag]`, `#[Storage]`, …) and the Laravel 12 class-level `#[Bind]` / `#[Singleton]` / `#[Scoped]` attributes.
- **Deferred providers**, what the built-in providers do, and how **facades and helpers** are powered by the container.
- How **testing** swaps container bindings (Pest), the default test runner in Laravel 11/12.

---

## 1. The "Why": Inversion of Control & Dependency Injection

Before any Laravel-specific API, you must understand the problem the container solves. Let's build up from plain PHP.

### 1.1 The problem: hard-coded dependencies

A **dependency** is simply an object that another object needs to do its job. Consider a class that sends order confirmations:

```php
<?php

class OrderService
{
    private MailgunClient $mailer;

    public function __construct()
    {
        // The class CREATES its own dependency.
        $this->mailer = new MailgunClient('SECRET_API_KEY', 'us-east-1');
    }

    public function confirm(int $orderId): void
    {
        $this->mailer->send("Order {$orderId} confirmed.");
    }
}
```

This looks fine until you try to actually use it well. Problems:

1. **Tight coupling** — `OrderService` is welded to `MailgunClient`. Switching to SES means editing this class.
2. **Untestable** — every test sends a *real* email, because you cannot substitute a fake mailer.
3. **Configuration leak** — the API key and region are buried inside the class.
4. **Duplication** — every class that needs a mailer repeats this construction logic.

### 1.2 The fix: Dependency Injection

**Dependency Injection (DI)** means a class *receives* its dependencies from the outside instead of creating them. First we extract an interface (the *contract*) so the class can depend on an abstraction rather than `MailgunClient`:

```php
<?php

interface Mailer
{
    public function send(string $message): void;
}
```

The most common form of DI is **constructor injection**:

```php
<?php

class OrderService
{
    // PHP 8.0+ constructor property promotion: declares + assigns in one line.
    public function __construct(private Mailer $mailer) {}

    public function confirm(int $orderId): void
    {
        $this->mailer->send("Order {$orderId} confirmed.");
    }
}
```

Notice we now type-hint the **interface** (`Mailer`), not a concrete class. The class no longer knows *or cares* which mailer it gets. This is **Inversion of Control (IoC)**: the responsibility of choosing and wiring dependencies is *inverted* — moved out of the class and up to the caller.

### 1.3 But now... who builds everything?

DI moves the wiring problem; it doesn't delete it. Someone, somewhere, must write:

```php
<?php

$mailer = new MailgunClient(
    config('services.mailgun.secret'),
    config('services.mailgun.region'),
);
$orderService = new OrderService($mailer);
```

In a real app this dependency graph is enormous — services depend on repositories, which depend on connections, which depend on config. Hand-wiring it everywhere is exactly the boilerplate we wanted to avoid.

**The solution is a *container*.** A **service container** (a.k.a. **IoC container** or **DI container**) is an object that knows *how to build other objects and their entire dependency graph*. You ask it for an `OrderService`, and it figures out it needs a `Mailer`, builds the right implementation, injects it, and hands you a finished object.

That container, in Laravel, is `Illuminate\Container\Container` (the application instance `Illuminate\Foundation\Application` extends it), accessible as `$app` or via the `app()` helper. It implements the **PSR-11** `Psr\Container\ContainerInterface` standard, so you can also type-hint `ContainerInterface` and call `$container->get(Foo::class)` / `$container->has(Foo::class)`. An unresolvable id throws a `Psr\Container\NotFoundExceptionInterface` (never bound) or `ContainerExceptionInterface` (bound but failed to build).

---

## 2. Automatic Resolution (Zero-Config Autowiring)

Here's the part that surprises newcomers: for **concrete classes with no special needs, you don't bind anything at all.** The container uses PHP's **Reflection API** to inspect a class's constructor and recursively build whatever it asks for. This is called **autowiring**.

```php
<?php

namespace App\Services;

class Engine
{
    public function start(): string
    {
        return 'vroom';
    }
}

class Car
{
    // The container sees this type-hint and builds an Engine automatically.
    public function __construct(private Engine $engine) {}

    public function drive(): string
    {
        return $this->engine->start();
    }
}
```

Resolve it:

```php
<?php

use App\Services\Car;

$car = app(Car::class);   // Container builds Engine, then injects it into Car.
echo $car->drive();
// Output: vroom
```

No binding required. The container reflected on `Car::__construct`, saw it needs an `Engine`, reflected on `Engine` (no constructor args), instantiated it, and injected it.

### 2.1 Where autowiring happens for you

You rarely call `app(Car::class)` manually. Laravel resolves things for you at framework boundaries:

```php
<?php

namespace App\Http\Controllers;

use App\Services\Car;
use Illuminate\Http\Request;

class CarController extends Controller
{
    // Both arguments are type-hinted -> container injects both.
    public function drive(Car $car, Request $request)
    {
        return ['result' => $car->drive(), 'ip' => $request->ip()];
    }
}
```

Controllers, route closures, queued jobs, event listeners, console commands, middleware — Laravel resolves their type-hinted dependencies through the container automatically. **Method injection** also works on controller actions (as above): Laravel reads the method's signature and injects each type-hinted parameter.

### 2.2 What autowiring can NOT do alone

Reflection can build a class only when every constructor parameter is either:

- a **type-hinted class/interface** the container can resolve, or
- a parameter with a **default value**, or
- nullable / variadic.

It **cannot** guess a scalar like `private string $apiKey`. For those you need explicit binding (Section 4 and 7.5). And it cannot decide *which* implementation to use for an **interface** — interfaces can't be instantiated, so you must tell the container the mapping. That's where binding comes in.

---

## 3. Binding Interfaces to Implementations

This is the single most important practical use of the container: **"when something asks for this interface, give it that class."**

Define the contract and an implementation:

```php
<?php

namespace App\Contracts;

interface PaymentGateway
{
    public function charge(int $cents): string; // returns a transaction id
}
```

```php
<?php

namespace App\Services;

use App\Contracts\PaymentGateway;

class StripeGateway implements PaymentGateway
{
    public function charge(int $cents): string
    {
        return 'stripe_txn_' . $cents;
    }
}
```

Register the mapping (we'll cover *where* to put this — a service provider — in Section 7):

```php
<?php

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;

$this->app->bind(PaymentGateway::class, StripeGateway::class);
```

Now any class can depend on the **interface** and stay ignorant of Stripe:

```php
<?php

class CheckoutService
{
    public function __construct(private PaymentGateway $gateway) {}

    public function pay(int $cents): string
    {
        return $this->gateway->charge($cents);
    }
}

echo app(CheckoutService::class)->pay(500);
// Output: stripe_txn_500
```

Swapping to a different provider is now a **one-line change** in a service provider — no application code touches it. This is the "program to an interface, not an implementation" principle, enforced by the container.

> **Laravel 12 shortcut:** instead of registering the mapping in a provider, you can annotate the interface itself with `#[Bind(StripeGateway::class)]` (see Section 6.8). The provider approach is still the most explicit and the most common in real codebases, so learn it first.

---

## 4. Binding: All the Flavors

Bindings are registered on the container, almost always inside a service provider's `register()` method (Section 7). Here are all the binding types.

### 4.1 `bind` — a new instance every time

`bind` registers a **resolver closure** (or a class name). The closure runs **every time** the binding is resolved, producing a *fresh* object each time.

```php
<?php

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Contracts\Foundation\Application;

// Closure form: full control over construction.
$this->app->bind(PaymentGateway::class, function (Application $app) {
    return new StripeGateway(
        secret: config('services.stripe.secret'),
    );
});

// Shorthand: bind interface to a concrete class name (container autowires it).
$this->app->bind(PaymentGateway::class, StripeGateway::class);
```

The closure receives the container instance, so you can resolve other dependencies inside it.

```php
<?php

$a = app(PaymentGateway::class);
$b = app(PaymentGateway::class);
var_dump($a === $b);
// Output: bool(false)   <- bind() gives a new object each resolve
```

### 4.2 `singleton` — one shared instance

`singleton` resolves the closure **once**, caches the result, and returns that *same* object for every subsequent resolve. Use it for stateless services or anything expensive to build (clients, connections).

```php
<?php

$this->app->singleton(PaymentGateway::class, StripeGateway::class);

$a = app(PaymentGateway::class);
$b = app(PaymentGateway::class);
var_dump($a === $b);
// Output: bool(true)    <- same shared instance
```

> **`singletonIf` / `bindIf`:** register a binding only if one isn't already registered. Handy in packages that want to let the host app override.

### 4.3 `scoped` — one instance per request/job lifecycle

`scoped` behaves like `singleton`, **except** the cached instance is **flushed** at the start of a new "lifecycle" — a new HTTP request in **Laravel Octane**, or a new job in a long-running queue worker. Use it for per-request state that must NOT leak between requests in a persistent runtime.

```php
<?php

// One instance for the duration of a single request/job, then discarded.
$this->app->scoped(RequestContext::class);

// scopedIf() (register only if not already bound) was added in Laravel 11.
$this->app->scopedIf(RequestContext::class);
```

> **Why it matters:** under a traditional PHP-FPM setup, every request boots a fresh container, so `singleton` and `scoped` look identical. Under Octane/Swoole/RoadRunner the container *persists*, so a `singleton` would leak per-request data across requests — `scoped` prevents that. **Available since Laravel 8.31** (`scoped`); `scopedIf` arrived in Laravel 11.

### 4.4 `instance` — bind an already-built object

If you already have an object, register it directly. The container returns that exact instance forever.

```php
<?php

$api = new \App\Services\ApiClient('https://api.example.com');
$this->app->instance(\App\Services\ApiClient::class, $api);

var_dump(app(\App\Services\ApiClient::class) === $api);
// Output: bool(true)
```

This is also how you swap implementations in tests (`$this->app->instance(...)` or `$this->mock(...)`).

### 4.5 Comparison table

| Method | New object each resolve? | Shared instance? | Reset per request/job? | Typical use |
|---|---|---|---|---|
| `bind` | ✅ yes | ❌ no | n/a | Cheap, stateful-per-use objects |
| `singleton` | ❌ no | ✅ yes (app-wide) | ❌ no | Clients, stateless services |
| `scoped` | ❌ no | ✅ yes (per lifecycle) | ✅ yes | Per-request state (Octane-safe) |
| `instance` | ❌ no | ✅ yes (the given object) | ❌ no | Pre-built / mocked objects |

---

## 5. Resolving Out of the Container

You've seen `app(...)`. Here are all the ways to pull objects out.

```php
<?php

use App\Contracts\PaymentGateway;

// 1. The app() helper with a class/abstract name.
$gw = app(PaymentGateway::class);

// 2. app() with no argument returns the container itself.
$container = app();
$gw = $container->make(PaymentGateway::class);

// 3. The resolve() helper (identical to app($abstract)).
$gw = resolve(PaymentGateway::class);

// 4. Inside a service provider, $this->app is the container.
$gw = $this->app->make(PaymentGateway::class);

// 5. Automatic — just type-hint it (the preferred way 95% of the time).
public function __construct(private PaymentGateway $gw) {}
```

> **Prefer injection over `make()`.** Reaching into the container with `app()`/`make()` is the **Service Locator** anti-pattern — it hides dependencies and hurts testability. Use it only where you truly cannot inject (e.g., inside a static helper, or to resolve something conditionally).

### 5.1 Passing extra parameters with `makeWith`

You can pass additional primitive arguments that the container can't autowire:

```php
<?php

class Report
{
    public function __construct(
        private \App\Services\Pdf $pdf,   // autowired
        private string $title,            // must be supplied
    ) {}
}

// makeWith (a.k.a. make with a second array arg) injects $pdf, you supply $title.
$report = app()->makeWith(Report::class, ['title' => 'Q2 Sales']);
// In recent Laravel you can also write: app()->make(Report::class, ['title' => 'Q2 Sales']);
```

### 5.2 `bound()`, `resolved()`, and the array interface

```php
<?php

app()->bound(PaymentGateway::class);     // bool: has a binding been registered?
app()->resolved(PaymentGateway::class);  // bool: has it been resolved at least once?

// The container implements ArrayAccess:
$gw = app()[PaymentGateway::class];      // same as app()->make(...)
```

### 5.3 Calling methods with dependency injection

`Container::call()` invokes any callable and injects its type-hinted parameters:

```php
<?php

class Stats
{
    public function generate(\App\Services\Db $db, int $days = 7): array
    {
        return ['days' => $days];
    }
}

// $db is injected from the container; $days uses extra params or its default.
$result = app()->call([new Stats, 'generate'], ['days' => 30]);
// You can also pass 'App\Services\Stats@generate' as a string.
```

---

## 6. Advanced Binding Techniques

### 6.1 Contextual binding: `when` / `needs` / `give`

Sometimes two classes need the *same interface* resolved to *different implementations*. **Contextual binding** expresses "when **X** needs **Y**, give it **Z**."

```php
<?php

use App\Contracts\Filesystem;
use App\Services\LocalFilesystem;
use App\Services\S3Filesystem;
use App\Jobs\BackupJob;
use App\Http\Controllers\UploadController;

$this->app->when(BackupJob::class)
    ->needs(Filesystem::class)
    ->give(S3Filesystem::class);

$this->app->when(UploadController::class)
    ->needs(Filesystem::class)
    ->give(function () {
        return new LocalFilesystem(storage_path('uploads'));
    });
```

Now `BackupJob` gets S3, `UploadController` gets local — same interface, different wiring. You can also pass an array of consumers to `when([A::class, B::class])`.

**Laravel 11+** adds `giveTagged()` and `giveConfig()` for contextual convenience:

```php
<?php

// Inject all services tagged 'reports' into a specific consumer.
$this->app->when(ReportManager::class)
    ->needs('$reporters')          // a variadic/array constructor param named $reporters
    ->giveTagged('reports');

// Inject a config value directly (Laravel 11+).
$this->app->when(GitHubClient::class)
    ->needs('$token')
    ->giveConfig('services.github.token');
```

### 6.2 Binding primitives (scalars)

Reflection can't guess a `string`/`int`. Use contextual binding with a `$variableName` to supply primitives:

```php
<?php

class GitHubClient
{
    public function __construct(private string $token, private int $timeout = 10) {}
}

$this->app->when(GitHubClient::class)
    ->needs('$token')                       // note the leading $ for primitives
    ->give(fn () => config('services.github.token'));
```

### 6.3 Tagging — resolve a group of bindings

**Tagging** labels a set of bindings so you can resolve them all at once. Great for plugin-style architectures (e.g., a collection of report generators).

```php
<?php

use App\Reports\CsvReport;
use App\Reports\PdfReport;

$this->app->bind(CsvReport::class, fn () => new CsvReport);
$this->app->bind(PdfReport::class, fn () => new PdfReport);

// Tag both bindings with the label 'reports'.
$this->app->tag([CsvReport::class, PdfReport::class], 'reports');

// Resolve every binding carrying that tag (returns an iterable of instances).
$this->app->bind(ReportManager::class, function ($app) {
    return new ReportManager($app->tagged('reports'));
});
```

```php
<?php

class ReportManager
{
    public function __construct(private iterable $reports) {}

    public function runAll(): array
    {
        return array_map(fn ($r) => $r->generate(), iterator_to_array($this->reports));
    }
}
```

### 6.4 Binding typed variadics

When a constructor takes a **typed variadic** (`Filter ...$filters`), reflection can't know which implementations to pass. Use contextual binding with a closure returning an array — or, more concisely, an array of class names the container will resolve for you:

```php
<?php

use App\Services\Firewall;
use App\Filters\NullFilter;
use App\Filters\ProfanityFilter;
use App\Filters\TooLongFilter;

class Firewall
{
    /** @var Filter[] */
    protected array $filters;

    public function __construct(protected Logger $logger, Filter ...$filters)
    {
        $this->filters = $filters;
    }
}

// Closure form — full control:
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->give(fn ($app) => [
        $app->make(NullFilter::class),
        $app->make(ProfanityFilter::class),
    ]);

// Array-of-class-names form — the container resolves each:
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->give([NullFilter::class, ProfanityFilter::class, TooLongFilter::class]);

// Or feed a tag straight into the variadic dependency:
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->giveTagged('filters');
```

### 6.5 Extending a binding

`extend` lets you **wrap or modify** a resolved object after the container builds it — useful for decorating a third-party binding without redefining it.

```php
<?php

use App\Contracts\PaymentGateway;
use App\Services\LoggingGatewayDecorator;

$this->app->extend(PaymentGateway::class, function ($service, $app) {
    // $service is the originally-resolved gateway. Wrap it.
    return new LoggingGatewayDecorator($service, $app->make('log'));
});
```

The closure receives the resolved instance and the container, and returns the (possibly wrapped) object.

### 6.6 Rebinding & resolving callbacks

```php
<?php

// Run a callback every time something is resolved from the container.
$this->app->resolving(PaymentGateway::class, function ($gateway, $app) {
    // e.g. configure the object after construction
});

// React when a binding is replaced after it was first resolved.
$this->app->rebinding(PaymentGateway::class, function ($app, $gateway) {
    // ...
});
```

### 6.7 Contextual attributes (Laravel 11.32+ / 12)

Recent Laravel supports **PHP attributes** to drive contextual injection directly on the consumer, no provider code needed:

```php
<?php

use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\Tag;
use Illuminate\Container\Attributes\Storage;

class GitHubClient
{
    public function __construct(
        #[Config('services.github.token')] private string $token,
        #[Tag('reports')] private iterable $reports,
        #[Storage('s3')] private \Illuminate\Contracts\Filesystem\Filesystem $disk,
    ) {}
}
```

The full set of built-in contextual attributes (all in `Illuminate\Container\Attributes\`) is:

| Attribute | Injects |
|---|---|
| `#[Auth('web')]` | A specific auth guard (`Illuminate\Contracts\Auth\Guard`) |
| `#[Cache('redis')]` | A specific cache store (`Illuminate\Contracts\Cache\Repository`) |
| `#[Config('app.timezone')]` | A config value |
| `#[Context('uuid')]` | A value from the Laravel **context** repository |
| `#[CurrentUser]` | The currently authenticated user model |
| `#[DB('mysql')]` | A specific database connection |
| `#[Give(DatabaseRepository::class)]` | A specific implementation for an interface |
| `#[Log('daily')]` | A specific log channel (`Psr\Log\LoggerInterface`) |
| `#[RouteParameter('photo')]` | A bound route parameter |
| `#[Storage('s3')]` | A specific filesystem disk |
| `#[Tag('reports')]` | All bindings carrying a tag (as `iterable`) |

```php
<?php

use Illuminate\Container\Attributes\CurrentUser;
use App\Models\User;

// #[CurrentUser] works on route closures and controller methods too.
Route::get('/me', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```

> You can build your own with the `Illuminate\Contracts\Container\ContextualAttribute` contract — implement a static `resolve(self $attribute, Container $container)` method. (PHP 8 attribute syntax; works on PHP 8.1–8.4.)

### 6.8 Class-level binding attributes (Laravel 12)

Laravel 12 also adds attributes you put **on the interface or class itself**, removing the need for any provider registration:

```php
<?php

namespace App\Contracts;

use App\Services\FakeEventPusher;
use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\Singleton;

// "Whenever EventPusher is requested, give RedisEventPusher" — no provider needed.
#[Bind(RedisEventPusher::class)]
// You can target specific environments; multiple #[Bind] attributes are allowed.
#[Bind(FakeEventPusher::class, environments: ['local', 'testing'])]
// Resolve the binding as a singleton (also: #[Scoped] for per-lifecycle).
#[Singleton]
interface EventPusher
{
    public function push(string $event): void;
}
```

> `#[Bind]`, `#[Singleton]`, and `#[Scoped]` are **Laravel 12** additions. `#[Singleton]`/`#[Scoped]` may be placed directly on a concrete class too, replacing a `singleton()`/`scoped()` call in a provider.

---

## 7. Service Providers

Bindings have to be registered *somewhere*, early in the request lifecycle. That "somewhere" is a **Service Provider** — the central place where you tell the container how to wire your application. Every Laravel feature (and most packages) ships its services via a provider. They are the **bootstrapping** mechanism of the framework.

### 7.1 Creating one

```bash
php artisan make:provider PaymentServiceProvider
# Creates: app/Providers/PaymentServiceProvider.php
```

A provider extends `Illuminate\Support\ServiceProvider` and has two key methods:

```php
<?php

namespace App\Providers;

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    /**
     * register(): ONLY bind things into the container.
     * Do NOT resolve services here — other providers may not be registered yet.
     */
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, function ($app) {
            return new StripeGateway(config('services.stripe.secret'));
        });
    }

    /**
     * boot(): runs AFTER all providers have registered.
     * Everything in the container is now available — wire up routes,
     * events, view composers, validators, publishing, etc.
     */
    public function boot(): void
    {
        // e.g. register a custom validation rule, gate, macro, observer...
    }
}
```

### 7.2 `register()` vs `boot()` — the crucial distinction

This is a **top interview question.** The framework boots in two phases:

1. **Register phase** — Laravel calls `register()` on *every* provider, in order. Its only job is to *put bindings into the container*. You must **not resolve** anything here, because a service you depend on might be registered by a provider that hasn't run yet.
2. **Boot phase** — Laravel calls `boot()` on every provider. By now **all** bindings exist, so it's safe to resolve services and use them.

```php
<?php

// ❌ WRONG: resolving in register() — Event dispatcher may not exist yet.
public function register(): void
{
    $events = $this->app->make('events'); // risky: ordering dependent
    $events->listen(/* ... */);
}

// ✅ RIGHT: bind in register(), use in boot().
public function register(): void
{
    $this->app->singleton(PaymentGateway::class, StripeGateway::class);
}

public function boot(): void
{
    // Safe: everything is registered now.
    \Illuminate\Support\Facades\Validator::extend('luhn', /* ... */);
}
```

> **Mnemonic:** *register = bind, boot = use.*

`boot()` itself supports **method injection** — type-hint dependencies and the container injects them:

```php
<?php

use Illuminate\Contracts\Routing\ResponseFactory;

public function boot(ResponseFactory $response): void
{
    $response->macro('caps', fn ($value) => $response->make(strtoupper($value)));
}
```

### 7.3 Registering your provider

Two mechanisms, depending on Laravel version:

**Laravel 11 and 12** — providers live in `bootstrap/providers.php`, a plain returned array. `make:provider` adds yours automatically.

```php
<?php
// bootstrap/providers.php  (Laravel 11+)

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\PaymentServiceProvider::class,  // <- added automatically
];
```

**Laravel 10 and earlier** — providers are listed in the `providers` array in `config/app.php`:

```php
<?php
// config/app.php  (Laravel 10 and earlier)

'providers' => [
    // ...framework providers...
    App\Providers\AppServiceProvider::class,
    App\Providers\PaymentServiceProvider::class,
],
```

> **Laravel 11+ change:** the old fixed set of app providers (`Route`, `Event`, `Broadcast`, `Auth`, etc.) was consolidated. New apps ship with just `AppServiceProvider`; routing/middleware/exceptions are now configured in `bootstrap/app.php` via the `Application::configure()` fluent builder. You add behavior in `AppServiceProvider` or your own providers.

### 7.4 Package auto-discovery

Packages don't need manual registration. Laravel reads their `composer.json` `extra.laravel.providers` key and auto-registers them. You can opt out via the `dont-discover` array in your app's `composer.json`.

```json
{
    "extra": {
        "laravel": {
            "providers": ["VendorName\\Package\\PackageServiceProvider"]
        }
    }
}
```

### 7.5 The `$bindings` and `$singletons` shortcut properties

For simple class-name bindings you can skip `register()` entirely and use array properties:

```php
<?php

class PaymentServiceProvider extends ServiceProvider
{
    // Equivalent to $this->app->bind(...) for each pair.
    public array $bindings = [
        \App\Contracts\PaymentGateway::class => \App\Services\StripeGateway::class,
    ];

    // Equivalent to $this->app->singleton(...) for each pair.
    public array $singletons = [
        \App\Contracts\Clock::class => \App\Services\SystemClock::class,
    ];
}
```

---

## 8. Deferred Providers

If a provider **only registers bindings** (no `boot()` work, no side effects), loading it on *every* request is wasteful. A **deferred provider** is loaded **lazily** — only when one of the services it provides is actually resolved. This is a real performance win for apps with many providers.

To defer, implement `DeferrableProvider` and list the abstracts your provider provides:

```php
<?php

namespace App\Providers;

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Contracts\Support\DeferrableProvider;
use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, fn ($app) =>
            new StripeGateway(config('services.stripe.secret'))
        );
    }

    /**
     * The bindings that, when resolved, trigger loading this provider.
     */
    public function provides(): array
    {
        return [PaymentGateway::class];
    }
}
```

How it works: Laravel builds a manifest mapping each provided abstract to its (deferred) provider class. When `app(PaymentGateway::class)` is called and no binding exists yet, the container looks up the manifest, loads & registers the provider on the spot, then resolves the service.

> **Constraints:** a deferred provider must **not** have a `boot()` method that needs to run on every request (it only boots when triggered), and every service it offers must be listed in `provides()`. If you need always-on bootstrapping, don't defer.

---

## 9. What the Built-in Providers Do

Laravel itself is assembled from providers. These live **inside the framework** (`Illuminate\*`) and are registered automatically — you never see them in `bootstrap/providers.php`:

| Framework provider | Responsibility |
|---|---|
| `DatabaseServiceProvider` | Binds the DB manager, connection factory, Eloquent. |
| `QueueServiceProvider`, `MailServiceProvider`, `CacheServiceProvider`, etc. | Bind each subsystem's manager into the container. |
| `FilesystemServiceProvider`, `RedisServiceProvider`, `ValidationServiceProvider`, … | Bind their respective managers/factories. |

The pattern is consistent: a **Manager** class (e.g., `CacheManager`, `MailManager`) is bound as a singleton and acts as a factory for "drivers." This is the **Manager pattern**, and it's why `Cache::store('redis')` vs `Cache::store('file')` works.

### 9.1 The app-level providers (and what changed in Laravel 11+)

Older Laravel (≤ 10) scaffolded several editable providers into `app/Providers/`. Knowing what happened to each is a common interview check:

| App provider | Pre-11 role | Laravel 11/12 |
|---|---|---|
| `AppServiceProvider` | General bindings/boot logic | **Still here** — the one you'll edit most. |
| `AuthServiceProvider` | Registered policies + gates | **Removed.** Policies are auto-discovered; define gates in `AppServiceProvider::boot()`. |
| `EventServiceProvider` | Mapped events → listeners | **Removed.** Listeners are auto-discovered; add `Event::listen(...)` in `AppServiceProvider::boot()` for manual mappings, or point discovery at extra dirs via `->withEvents(discover: [...])` in `bootstrap/app.php`. |
| `RouteServiceProvider` | Loaded route files + rate limiters | **Removed.** Routes/middleware/rate limiters are configured in `bootstrap/app.php` via `->withRouting(...)`. |
| `BroadcastServiceProvider` | Registered broadcast routes | **Removed.** Enable with `->withBroadcasting()` in `bootstrap/app.php`, or `Broadcast::routes()` in `AppServiceProvider`. |

> In a fresh Laravel 11/12 app, `AppServiceProvider` is the **only** provider in `app/Providers/`. The behavior of the deleted providers didn't disappear — it moved into the `bootstrap/app.php` fluent builder and into framework-level auto-discovery.

---

## 10. How Facades & Helpers Use the Container

This ties everything together and is a guaranteed interview topic.

### 10.1 Facades

A **facade** (e.g., `Cache`, `DB`, `Route`) is a class providing a `static` proxy to an object **resolved from the container**. `Cache::get('key')` is *not* a real static method — it's intercepted.

```php
<?php

namespace Illuminate\Support\Facades;

class Cache extends Facade
{
    // This string is a key registered in the container.
    protected static function getFacadeAccessor(): string
    {
        return 'cache';
    }
}
```

When you call `Cache::get('x')`:

1. PHP can't find static `get()`, so it triggers the **`__callStatic`** magic method on the base `Facade` class.
2. `Facade::__callStatic` calls `getFacadeAccessor()` → `'cache'`.
3. It resolves that key from the container: `app()->make('cache')` → the `CacheManager` singleton.
4. It forwards the call: `$cacheManager->get('x')`.

So `Cache::get('x')` is effectively `app('cache')->get('x')`. **Facades are just sugar over container resolution.** This is also why facades are easy to fake in tests (`Cache::fake()` swaps the container binding).

```php
<?php
// These two lines are functionally equivalent:
\Illuminate\Support\Facades\Cache::get('x');
app('cache')->get('x');
```

> **Real-time facades:** prefix any class with `Facades\` in a `use` statement and Laravel resolves it from the container on the fly — e.g. `use Facades\App\Services\PaymentGateway;` then call `PaymentGateway::charge(...)`. The container still does the resolving.

### 10.2 Helpers

Global helpers are similarly thin wrappers around the container:

```php
<?php

// Each helper resolves a binding under the hood:
request();   // app('request')
session();   // app('session')
view();      // app('view')
cache();     // app('cache')
auth();      // app('auth')
config();    // app('config')
response();  // app(ResponseFactory::class)
```

That's the whole secret: **facades and helpers are convenience layers; the container is doing the real work underneath.**

### 10.3 Swapping bindings in tests (Pest)

Because everything resolves from the container, tests swap real services for fakes by **rebinding** — never by editing production code. Laravel 11/12 ship with **Pest** as the default test runner:

```php
<?php
// tests/Feature/CheckoutTest.php

use App\Contracts\PaymentGateway;
use Mockery\MockInterface;

it('charges the gateway during checkout', function () {
    // 1. Swap the binding with a Mockery double via the TestCase helper.
    $this->mock(PaymentGateway::class, function (MockInterface $mock) {
        $mock->shouldReceive('charge')->once()->with(500)->andReturn('fake_txn');
    });

    // 2. Or bind a pre-built fake object directly.
    // $this->app->instance(PaymentGateway::class, new FakeGateway);

    $txn = app(\App\Services\CheckoutService::class)->pay(500);

    expect($txn)->toBe('fake_txn');
});

it('fakes a facade', function () {
    \Illuminate\Support\Facades\Cache::shouldReceive('get')
        ->with('key')->andReturn('value');

    expect(cache('key'))->toBe('value');   // helper resolves the faked binding too
});
```

> `Cache::fake()` / `Cache::shouldReceive(...)` and `$this->mock(...)` all work by replacing the container binding — the same mechanism you learned in this module.

---

## ⚠️ Common Mistakes & Gotchas

1. **Resolving services inside `register()`.**
   *Symptom:* intermittent "target is not instantiable" or null services depending on provider order.
   *Fix:* `register()` may only *bind*. Move any `make()`/use of a service into `boot()`, where all providers are registered.

2. **Using `singleton` for per-request state under Octane.**
   *Symptom:* one user sees another user's data; state bleeds across requests on a persistent runtime.
   *Fix:* use `scoped` (reset each request/job) instead of `singleton` for anything holding request-specific state. Under PHP-FPM you won't notice the bug locally — it only appears under Octane/Swoole/RoadRunner.

3. **Type-hinting a concrete class instead of the interface.**
   *Symptom:* contextual binding / swapping implementations doesn't take effect; hard to mock in tests.
   *Fix:* depend on the **interface** (`PaymentGateway`), bind it in a provider, and let contextual binding/test doubles do their job. Type-hinting `StripeGateway` directly bypasses the binding.

4. **Forgetting to bind primitives (scalars).**
   *Symptom:* `Unresolvable dependency resolving [Parameter #0 $token]` / `BindingResolutionException`.
   *Fix:* the container can't autowire a `string`/`int`. Provide it via contextual `->needs('$token')->give(...)`, a default value, or the `#[Config(...)]` attribute.

5. **Expecting a deferred provider's `boot()` to always run.**
   *Symptom:* macros/listeners registered in a deferred provider's `boot()` silently never fire.
   *Fix:* deferred providers only load when a `provides()` service is resolved. If you need always-on boot logic, don't defer (don't implement `DeferrableProvider`).

6. **Calling `make()` everywhere (Service Locator anti-pattern).**
   *Symptom:* hidden dependencies, brittle tests, classes that lie about what they need.
   *Fix:* prefer constructor/method injection. Reserve `app()`/`make()` for the few places injection is impossible (static helpers, conditional resolution).

7. **Forgetting to register a non-package provider.**
   *Symptom:* your bindings simply don't exist; container falls back to autowiring or throws.
   *Fix:* add it to `bootstrap/providers.php` (L11+) or `config/app.php` (L10−). `make:provider` does this for you in L11+; manual providers must be added by hand.

---

## ✅ Best Practices

- **Bind to interfaces**, depend on interfaces. Reserve concrete type-hints for value objects and DTOs.
- **`register()` = bind only. `boot()` = everything else.** Never resolve in `register()`.
- **Prefer constructor injection** over `app()`/`make()`. Make dependencies explicit and visible.
- Use **`singleton`** for stateless/expensive services, **`scoped`** for per-request state (Octane-safe), **`bind`** when you genuinely need a fresh object each time.
- **Group related bindings** into a dedicated provider (`PaymentServiceProvider`) rather than dumping everything in `AppServiceProvider`.
- **Defer providers** that only register bindings, to shave bootstrap cost — but list *every* provided abstract in `provides()`.
- Use **`#[Config]`, `#[Tag]`, `#[Storage]` attributes** (L11.32+/12) for cleaner contextual injection where they fit. For a global interface→implementation mapping, the L12 `#[Bind]` attribute is an option — but a service provider keeps wiring in one discoverable place, which most teams prefer.
- Keep providers **thin and focused**; push real logic into the services themselves.
- In tests, swap implementations with `$this->app->instance()`, `$this->mock()`, or facade fakes (`Cache::fake()`), not by editing production bindings.
- Use `$bindings` / `$singletons` array properties for trivial mappings to reduce boilerplate.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is the service container and why does Laravel need one?**
It's an IoC/DI container — an object that knows how to construct classes and resolve their entire dependency graph automatically. It exists to implement Inversion of Control: classes declare *what* they need (via type-hints), and the container decides *how* to build and inject it. This decouples classes, centralizes wiring, and makes testing/swapping implementations trivial.

**Q2. What's the difference between `bind`, `singleton`, `scoped`, and `instance`?**
`bind` runs its resolver every time → a new object per resolve. `singleton` resolves once and caches the result app-wide. `scoped` is a singleton that's reset at the start of each request/job (matters under Octane). `instance` registers a pre-built object you already have.

**Q3. What's the difference between `register()` and `boot()` in a service provider, and why does the order matter?**
Laravel runs all providers' `register()` first (binding phase), then all `boot()` methods (use phase). `register()` must only add bindings — resolving a service there is unsafe because the provider that supplies it may not have registered yet. `boot()` runs after every binding exists, so it's safe to resolve and use services. Mnemonic: *register = bind, boot = use.*

**Q4. How does autowiring work? (under the hood)**
When you ask the container for a class with no explicit binding, it uses PHP's **Reflection API** to inspect the constructor's parameters. For each type-hinted class/interface, it recursively resolves that dependency from the container; for parameters with defaults it uses the default. It then instantiates the class with the resolved arguments. It throws `BindingResolutionException` if it hits an interface with no binding or an un-resolvable scalar.

**Q5. How do facades actually work under the hood?**
A facade extends `Illuminate\Support\Facades\Facade` and defines `getFacadeAccessor()`, returning a container key. Calling a "static" method that doesn't exist triggers `__callStatic`, which resolves that key from the container (`app()->make($accessor)`) and forwards the call to the real object. So `Cache::get()` ≈ `app('cache')->get()`. Facades are syntactic sugar over container resolution, which is also why they're easy to fake in tests.

**Q6. When would you use contextual binding?**
When the same interface must resolve to different implementations depending on the consumer — e.g., `BackupJob` needs S3 storage but `UploadController` needs local storage. You express it with `when(Consumer::class)->needs(Interface::class)->give(Implementation::class)`.

**Q7. What is a deferred provider and when should you use one?**
A provider implementing `DeferrableProvider` plus a `provides()` list. Laravel loads it lazily — only when one of its provided services is first resolved — instead of on every request. Use it for providers that *only* register bindings (no always-on boot work) to reduce bootstrap overhead.

**Q8. How do you bind a primitive/scalar value the container can't autowire?**
Use contextual binding with a `$`-prefixed variable name: `when(GitHubClient::class)->needs('$token')->give(fn () => config('services.github.token'))`. In Laravel 11+ you can also use the `#[Config('...')]` attribute on the constructor parameter, or `->giveConfig('...')`.

**Q9. What's the difference between `singleton` and `scoped`, concretely?**
Functionally identical under PHP-FPM (fresh container per request). The difference appears in long-running runtimes like **Octane**: the container persists across requests, so a `singleton` keeps its instance (and any per-request state leaks), whereas `scoped` is flushed at the start of each new request/job — making it the safe choice for request-specific state.

**Q10. What changed about service providers in Laravel 11?**
Apps now register providers in `bootstrap/providers.php` (a plain array) instead of `config/app.php`. The default provider set was slimmed to mostly just `AppServiceProvider`; routing, middleware, and exception handling moved to the `bootstrap/app.php` fluent `Application::configure()` builder. Event listeners are auto-discovered. Laravel 12 keeps this structure.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
// ── BINDING (inside a provider's register()) ──────────────────────────
$this->app->bind(Abstract::class, Concrete::class);        // new each resolve
$this->app->bind(Abstract::class, fn ($app) => new C());   // closure form
$this->app->singleton(Abstract::class, Concrete::class);   // one shared instance
$this->app->scoped(Abstract::class, Concrete::class);      // one per request/job
$this->app->instance(Abstract::class, $object);            // bind existing object
$this->app->bindIf(...); $this->app->singletonIf(...);     // only if not bound

// ── INTERFACE → IMPLEMENTATION ────────────────────────────────────────
$this->app->bind(PaymentGateway::class, StripeGateway::class);

// ── CONTEXTUAL BINDING ────────────────────────────────────────────────
$this->app->when(Consumer::class)
          ->needs(Interface::class)        // or ->needs('$scalarName')
          ->give(Implementation::class);   // or ->give(fn () => ...)
          // L11+: ->giveTagged('tag'), ->giveConfig('key')
$this->app->when(Consumer::class)         // typed variadic: array of class names
          ->needs(Filter::class)
          ->give([NullFilter::class, ProfanityFilter::class]);

// ── CONTEXTUAL ATTRIBUTES (on constructor params) ─────────────────────
// #[Config], #[Tag], #[Storage], #[Auth], #[Cache], #[DB], #[Log],
// #[Context], #[Give], #[RouteParameter], #[CurrentUser]
// ── CLASS/INTERFACE ATTRIBUTES (L12) ──────────────────────────────────
// #[Bind(Impl::class)], #[Singleton], #[Scoped]

// ── TAGGING ───────────────────────────────────────────────────────────
$this->app->tag([A::class, B::class], 'group');
$this->app->tagged('group');               // iterable of instances

// ── EXTENDING ─────────────────────────────────────────────────────────
$this->app->extend(Abstract::class, fn ($svc, $app) => new Decorator($svc));

// ── RESOLVING ─────────────────────────────────────────────────────────
app(Abstract::class);                       // helper
resolve(Abstract::class);                   // helper (same thing)
app()->make(Abstract::class);               // explicit
app()->make(C::class, ['title' => 'X']);    // with extra params
app()->makeWith(C::class, ['title' => 'X']);
app()->call([$obj, 'method'], ['x' => 1]);  // method injection
app()->bound(Abstract::class);              // is it bound?

// ── PROVIDER SKELETON ─────────────────────────────────────────────────
class FooServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public array $singletons = [Clock::class => SystemClock::class]; // shortcut
    public function register(): void { /* bind only */ }
    public function boot(): void { /* use services */ }
    public function provides(): array { return [Clock::class]; }     // deferred
}
```

```php
<?php
// Register a provider:
// L11/12 -> bootstrap/providers.php   |   L10- -> config/app.php 'providers'
```

```bash
php artisan make:provider FooServiceProvider   # scaffold a provider
php artisan about                              # see registered providers/services
```

| Concept | API |
|---|---|
| New instance every time | `bind` |
| Shared instance | `singleton` |
| Per-request instance | `scoped` |
| Pre-built object | `instance` |
| Interface → impl | `bind(Interface, Impl)` |
| Per-consumer impl | `when→needs→give` |
| Scalar injection | `when→needs('$x')→give` or `#[Config]` |
| Group resolution | `tag` / `tagged` |
| Wrap a binding | `extend` |
| Interface → impl (attribute, L12) | `#[Bind(Impl::class)]` on the interface |
| Singleton (attribute, L12) | `#[Singleton]` on the class/interface |
| Lazy provider | `implements DeferrableProvider` + `provides()` |

---

## 🧪 Mini Exercises

1. **Interface swap.** Create a `NotificationChannel` interface with a `send(string $message): void` method, plus `SlackChannel` and `LogChannel` implementations. Bind `NotificationChannel` to `SlackChannel` in a new `NotificationServiceProvider`. Inject the interface into a controller action and confirm Slack is used. Then change one line so `LogChannel` is used instead — without touching the controller.

2. **Contextual + primitive binding.** Build an `ApiClient` whose constructor needs `private string $baseUrl` and `private int $timeout = 30`. Using contextual binding, make `ReportSyncJob` receive `https://reports.internal` and `WebhookController` receive `https://api.public`, each pulling the base URL from `config()`. Verify the timeout default still works when not overridden.

3. **Tagging a plugin set.** Create three `Exporter` implementations (`JsonExporter`, `CsvExporter`, `XmlExporter`). Tag all three as `'exporters'`. Build an `ExportManager` that receives `iterable $exporters` via the tag and exposes `exportAll(array $data): array`. Resolve `ExportManager` from the container and call `exportAll`.

4. **Singleton vs scoped.** Bind a `RequestId` service that generates a random UUID in its constructor. Register it once as a `singleton` and once (separately) as `scoped`. Resolve each twice in the same request and compare identities. In one sentence, explain how the results would differ across two requests under Laravel Octane.

5. **Deferred provider.** Convert your `NotificationServiceProvider` from Exercise 1 into a deferred provider (`implements DeferrableProvider` + `provides()`). Then run `php artisan about` (or dump the deferred services manifest) and confirm the provider is not loaded until `NotificationChannel` is resolved. Note what would break if you'd put always-on logic in its `boot()`.

6. **Contextual attributes (L12).** Rewrite Exercise 2's `ApiClient` so the base URL and timeout are injected with `#[Config(...)]` attributes on the constructor parameters instead of provider-side contextual bindings. Then inject a `#[Storage('s3')]` filesystem disk into a class and confirm it resolves the S3 disk without any provider code.

7. **`#[Bind]` attribute (L12).** Take your `NotificationChannel` interface and bind it to `SlackChannel` using a class-level `#[Bind(SlackChannel::class)]` attribute on the interface — deleting the provider binding entirely. Add a second `#[Bind(LogChannel::class, environments: ['testing'])]` and verify that test runs receive `LogChannel`.

8. **Facade under the hood.** Without using any facade, replicate `Cache::put('k', 'v')` using only `app('cache')`. Then write a Pest test that uses `Cache::shouldReceive('put')` and explain, in one sentence, why mocking the facade also affects the `cache()` helper.
