# Architecture Patterns in Laravel

Laravel is famously forgiving: you can put everything in a controller, ship it, and it works. That freedom is also a trap. As an app grows from a weekend prototype to a system that several engineers touch every day, "it works" stops being enough — you need code that is *changeable* without fear. This module is about the patterns that keep a Laravel codebase maintainable as it scales, when to reach for each one, and — just as important — when **not** to.

> **Audience note:** This is an opinionated tour. The Laravel community genuinely disagrees about some of this (especially the Repository pattern). I will flag the debates honestly so you can defend a position in an interview rather than parrot a dogma.

---

**What you'll learn**

- Why "fat controllers" rot a codebase and what to do instead
- The spectrum from **fat models → service layer → action classes**, and when each fits
- The **Repository pattern** in Laravel, the interface-plus-binding mechanics, and the famous "Eloquent is already a repository" debate
- **DTOs** (Data Transfer Objects), Form Requests, and API Resources as boundary objects
- **Events/Listeners, Observers, and Jobs** for side effects and async work
- Where business logic actually belongs, and a pragmatic take on **DDD** and the **modular monolith**
- **Dependency injection, SOLID, and the service container** applied to real Laravel code
- Why you should **avoid facades in domain code**, the testing payoff, and how to **avoid over-engineering**

---

## 1. The core problem: change cost grows faster than features

When an app is small, every change is cheap because you can hold the whole thing in your head. As it grows, two things happen:

1. **Coupling** — code that knows too much about other code. Change one thing, break three others.
2. **Low cohesion** — related logic scattered across controllers, models, blade files, and helpers, so a single feature lives in ten places.

Architecture patterns are not academic decoration. They are tools to **reduce coupling** and **increase cohesion** so that the cost of the next change stays roughly flat instead of exploding. Every pattern below earns its place only if it reduces that change cost for *your* app at *your* size.

> **Jargon:** *Coupling* = how much one unit depends on the internals of another. *Cohesion* = how much the things inside one unit belong together. The goal is **low coupling, high cohesion**.

---

## 2. The starting point: skinny controller, and where logic goes

A controller's job is to handle an HTTP concern: take a request, hand off work, return a response. That's it. The moment a controller starts validating, querying, computing, charging a card, and sending email, it has become four classes wearing a trench coat.

### The "fat controller" smell

```php
<?php

namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Mail;

class CheckoutController extends Controller
{
    public function store(Request $request)
    {
        // validation
        $data = $request->validate([
            'user_id'  => 'required|exists:users,id',
            'items'    => 'required|array|min:1',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.qty'        => 'required|integer|min:1',
        ]);

        // business logic mixed with persistence
        $user  = User::findOrFail($data['user_id']);
        $total = 0;
        foreach ($data['items'] as $item) {
            $total += $item['qty'] * \App\Models\Product::find($item['product_id'])->price;
        }

        if ($user->wallet_balance < $total) {
            return response()->json(['error' => 'Insufficient funds'], 422);
        }

        $order = Order::create([
            'user_id' => $user->id,
            'total'   => $total,
            'status'  => 'paid',
        ]);

        $user->decrement('wallet_balance', $total);

        // side effect
        Mail::to($user)->send(new \App\Mail\OrderConfirmation($order));

        return response()->json($order, 201);
    }
}
```

This works. It is also untestable without HTTP, unreusable from a queue/CLI/another controller, and impossible to reason about at a glance. The N+1 query in the loop (`Product::find(...)` once per item) is hiding in plain sight.

> **⚠️ Security / correctness note:** This snippet also has a classic **lost-update race condition**. Two concurrent checkouts both read the same `wallet_balance`, both pass the `< $total` check, and both `decrement` it — overspending the wallet. Money flows like this must run inside a transaction *and* lock the row (`User::whereKey($id)->lockForUpdate()->first()`) or use an atomic conditional update. The fat controller hides this; pushing the operation into a service (next sections) is where you fix it properly. Never ship balance/inventory mutations without locking or atomicity.

### The question that drives all of this: *where does business logic go?*

There is no single right answer, but there is a useful default hierarchy. Try them in this order and stop at the simplest one that keeps the code clean:

| Stage | Put logic in… | Good when |
|-------|---------------|-----------|
| 1 | **The model** (fat model, skinny controller) | Logic is *about one entity* and is simple |
| 2 | **A service class** | Logic spans multiple models / coordinates steps |
| 3 | **An action class** | One specific use case, invoked from many entry points |
| 4 | **A domain layer / module** | Large app, multiple bounded contexts, strict boundaries |

You almost never need to jump straight to stage 4. Most successful Laravel apps live happily at stage 1–3.

---

## 3. Fat model, skinny controller

The oldest MVC advice: push logic *down* into the model where the data lives. For entity-scoped behavior this is genuinely good and very Laravel-idiomatic.

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Model
{
    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }

    // Domain behavior that is genuinely about a User:
    public function canAfford(int $cents): bool
    {
        return $this->wallet_balance >= $cents;
    }

    public function charge(int $cents): void
    {
        $this->decrement('wallet_balance', $cents);
    }
}
```

The payoff is at the call site, where the domain reads like a sentence:

```php
if ($user->canAfford($total)) {
    $user->charge($total);
}
```

**Where fat models break down:** when logic involves *several* models, external services, or a multi-step workflow (charge card → create order → deduct stock → email). Cramming "checkout" into the `User` model or the `Order` model violates the **Single Responsibility Principle** — the model now changes for reasons unrelated to being a user/order. That's the signal to extract a **service** or **action**.

> **Rule of thumb:** If the method needs to know about *more than one* model or talks to the outside world, it probably doesn't belong on the model.

---

## 4. The service layer

A **service class** is a plain PHP class that coordinates a business operation. It is the most common "next step" past fat models in real Laravel apps.

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\User;
use App\Exceptions\InsufficientFundsException;
use Illuminate\Support\Facades\DB;

class CheckoutService
{
    public function checkout(User $user, array $items): Order
    {
        $total = $this->calculateTotal($items);

        // Cheap early-out before opening a transaction (NOT the source of
        // truth — the authoritative, locked check happens below).
        if (! $user->canAfford($total)) {
            throw new InsufficientFundsException();
        }

        // Atomic: either all of this happens or none of it.
        return DB::transaction(function () use ($user, $items, $total) {
            // Lock the row so two concurrent checkouts can't both overspend
            // the wallet (prevents the lost-update race from Section 2).
            $user = User::whereKey($user->id)->lockForUpdate()->firstOrFail();

            if (! $user->canAfford($total)) {
                throw new InsufficientFundsException();
            }

            $order = Order::create([
                'user_id' => $user->id,
                'total'   => $total,
                'status'  => 'paid',
            ]);

            $order->items()->createMany($items);
            $user->charge($total);

            return $order;
        });
    }

    private function calculateTotal(array $items): int
    {
        return collect($items)->sum(fn (array $i) => $i['qty'] * $i['price']);
    }
}
```

The controller shrinks to its real job:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\CheckoutRequest;
use App\Http\Resources\OrderResource;
use App\Services\CheckoutService;

class CheckoutController extends Controller
{
    public function __construct(private readonly CheckoutService $checkout) {}

    public function store(CheckoutRequest $request): OrderResource
    {
        $order = $this->checkout->checkout(
            user:  $request->user(),
            items: $request->validated('items'),
        );

        return new OrderResource($order);
    }
}
```

Note the **constructor property promotion** (`private readonly CheckoutService $checkout`, PHP 8.0+) and **named arguments** (`user:`, `items:`) — modern idioms that make the call site self-documenting. Laravel's service container **auto-resolves** `CheckoutService` and injects it; you never `new` it.

**Pros:** reusable from controllers, jobs, commands, and tests; testable in isolation; keeps controllers and models thin.
**Cons:** services can become "god classes" if you let one service own a whole subsystem. Keep them focused, or split into actions.

---

## 5. Action / single-purpose classes

An **action** is a service taken to its logical extreme: one class, one public method, one use case. The community convention is an `__invoke()` method or a descriptively named `execute()`/`handle()`.

```php
<?php

namespace App\Actions;

use App\Models\User;
use App\Models\Order;
use Illuminate\Support\Facades\DB;

final class PlaceOrder
{
    public function __construct(
        private readonly CalculateOrderTotal $calculateTotal,
    ) {}

    public function __invoke(User $user, array $items): Order
    {
        $total = ($this->calculateTotal)($items);

        return DB::transaction(function () use ($user, $items, $total) {
            $order = Order::create([
                'user_id' => $user->id,
                'total'   => $total,
                'status'  => 'paid',
            ]);
            $order->items()->createMany($items);
            $user->charge($total);
            return $order;
        });
    }
}
```

```php
// Called like a function thanks to __invoke:
$order = ($placeOrder)($user, $items);
```

**Why actions over services?** They obey the **S** in SOLID rigorously — each class has exactly one reason to change. They compose (one action calls another), they read like a list of verbs in your domain, and they're trivial to test. The downside is class proliferation: a big app can have hundreds of action classes. That's a feature for some teams and noise for others.

> Packages like `lorisleiva/laravel-actions` let one class serve as controller, job, listener, and command simultaneously. Useful, but know that it blurs boundaries — mention it in interviews as a known option, not a default.

---

## 6. DTOs — Data Transfer Objects

A **DTO** is a simple, typed object whose only job is to carry data across a boundary (HTTP → service, service → service). The problem it solves: passing untyped `array $data` everywhere means no autocomplete, no type safety, and "what keys are in here?" archaeology.

Plain PHP DTO with a readonly class (PHP 8.2+ for `readonly` *classes*; readonly *properties* arrived in 8.1):

```php
<?php

namespace App\DataTransferObjects;

final readonly class OrderItemData
{
    public function __construct(
        public int $productId,
        public int $qty,
        public int $price,   // in cents
    ) {}
}
```

```php
<?php

namespace App\DataTransferObjects;

use App\Http\Requests\CheckoutRequest;

final readonly class CheckoutData
{
    /** @param array<int, OrderItemData> $items */
    public function __construct(
        public int $userId,
        public array $items,
    ) {}

    public static function fromRequest(CheckoutRequest $request): self
    {
        return new self(
            userId: $request->user()->id,
            items:  array_map(
                fn (array $i) => new OrderItemData($i['product_id'], $i['qty'], $i['price']),
                $request->validated('items'),
            ),
        );
    }
}
```

Now a service signature becomes honest: `checkout(CheckoutData $data)` instead of `checkout(array $data)`. The compiler and your IDE enforce the shape.

### spatie/laravel-data

Writing DTO boilerplate by hand gets old. The `spatie/laravel-data` package gives you DTOs that double as validators, can be built from requests, and cast to/from arrays and JSON automatically.

```php
<?php

namespace App\Data;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\DataCollectionOf;
use Spatie\LaravelData\Attributes\Validation\Min;

class CheckoutData extends Data
{
    public function __construct(
        public int $userId,
        // #[DataCollectionOf] tells the package which Data class fills the
        // array so nested items are cast correctly; #[Min(1)] enforces a
        // non-empty array when validating.
        #[DataCollectionOf(OrderItemData::class), Min(1)]
        public array $items,
    ) {}
}
```

> **Note:** with spatie/laravel-data, an array of nested data objects must be typed either with the `#[DataCollectionOf(...)]` attribute (shown) or a precise generic annotation (`@var OrderItemData[]`); a bare `array` type won't cast the inner items. `OrderItemData` would itself extend `Spatie\LaravelData\Data`.

```php
// Build straight from the request — validation runs automatically:
$data = CheckoutData::from($request);
```

```bash
composer require spatie/laravel-data
```

**When to use DTOs:** at boundaries where data shape matters and crosses layers — API payloads, queue job arguments, service inputs. **When not to:** trivial CRUD where an Eloquent model already *is* your data object. Don't wrap every array in a DTO reflexively.

---

## 7. Form Requests (input boundary) and API Resources (output boundary)

These are first-party Laravel patterns that do the same job as DTOs at the HTTP edge: give your input and output a defined shape.

### Form Requests — validation + authorization in one place

```bash
php artisan make:request CheckoutRequest
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CheckoutRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null;
    }

    public function rules(): array
    {
        return [
            'items'              => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'integer', 'exists:products,id'],
            'items.*.qty'        => ['required', 'integer', 'min:1'],
            'items.*.price'      => ['required', 'integer', 'min:0'],
        ];
    }
}
```

Type-hint it in the controller and validation runs *before* your method body. Failed validation throws automatically (422 JSON for APIs, redirect-back for web). This keeps validation out of controllers and services entirely.

### API Resources — shaping output

An **API Resource** transforms a model (or collection) into a JSON structure, decoupling your DB columns from your public API contract.

```bash
php artisan make:resource OrderResource
```

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'        => $this->id,
            'total'     => $this->total / 100,            // cents -> dollars for the API
            'status'    => $this->status,
            'items'     => OrderItemResource::collection($this->whenLoaded('items')),
            'placed_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

```json
{
  "data": {
    "id": 42,
    "total": 59.99,
    "status": "paid",
    "items": [],
    "placed_at": "2026-06-18T10:30:00+00:00"
  }
}
```

`whenLoaded('items')` only includes the relationship if it was eager-loaded — a built-in N+1 guard. Form Requests + Resources together mean your controller never touches a raw array on the way in or a raw model on the way out.

---

## 8. The Repository pattern (and the great Laravel debate)

A **Repository** is an object that mediates between your domain and your data store, exposing collection-like methods (`find`, `save`, `all`) while hiding *how* persistence happens. Classic enterprise pattern, born in the .NET/Java world.

### The interface + binding mechanics

The point of a repository is the **interface**, so callers depend on an abstraction (Dependency Inversion — the **D** in SOLID), not on Eloquent.

```php
<?php

namespace App\Repositories\Contracts;

use App\Models\Order;

interface OrderRepository
{
    public function find(int $id): ?Order;
    public function save(Order $order): Order;
    /** @return \Illuminate\Support\Collection<int, Order> */
    public function forUser(int $userId): \Illuminate\Support\Collection;
}
```

```php
<?php

namespace App\Repositories\Eloquent;

use App\Models\Order;
use App\Repositories\Contracts\OrderRepository;
use Illuminate\Support\Collection;

class EloquentOrderRepository implements OrderRepository
{
    public function find(int $id): ?Order
    {
        return Order::find($id);
    }

    public function save(Order $order): Order
    {
        $order->save();
        return $order;
    }

    public function forUser(int $userId): Collection
    {
        return Order::where('user_id', $userId)->latest()->get();
    }
}
```

Bind the interface to the implementation in a service provider so the container knows what to inject:

```php
<?php

namespace App\Providers;

use App\Repositories\Contracts\OrderRepository;
use App\Repositories\Eloquent\EloquentOrderRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
    }
}
```

> **Laravel 11/12 note:** there is no `app/Providers/RouteServiceProvider` boilerplate to copy anymore and providers are registered in `bootstrap/providers.php` (Laravel 11+), not `config/app.php` (Laravel 10 and earlier). Run `php artisan make:provider RepositoryServiceProvider` and it's auto-registered.

Now any consumer just type-hints the interface:

```php
public function __construct(private readonly OrderRepository $orders) {}
```

If you later swap Eloquent for a different store (or a fake in tests), you change *one binding line* and nothing else.

### The debate: "Eloquent is already a repository"

This is the part interviewers love. The honest take:

**Arguments against repositories in Laravel:**
- Eloquent models already abstract the database. Wrapping them in a repository often just **re-implements the query builder with fewer features** (`forUser`, `findActive`, `findActiveWithPosts`… method explosion).
- You lose Eloquent's ergonomics: relationships, eager loading, scopes, pagination — or you leak them back through the interface, defeating the purpose.
- It adds layers and indirection that most apps never benefit from.

**Arguments for repositories:**
- Genuine **persistence-agnosticism**: if you might back the same domain with Eloquent *and* an external API *and* a cache, an interface is real value.
- **Testability without a database**: bind a fake in-memory repo in tests.
- **Enforcing a query boundary**: stops ad-hoc `Model::where(...)` queries from spreading everywhere.

**The pragmatic position to state in an interview:** "I don't use the Repository pattern by default in Laravel, because Eloquent already abstracts persistence and a repository often duplicates the query builder. I reach for it only when I need a real abstraction over multiple data sources or want to test domain logic with no database. Otherwise I keep query logic in **query scopes** or thin **query objects**, which give most of the benefit without the ceremony." That answer shows you know the pattern *and* know when it's over-engineering.

A lightweight middle ground — **query scopes** on the model:

```php
// In the model:
public function scopeForUser($query, int $userId)
{
    return $query->where('user_id', $userId);
}

// Usage reads cleanly, no extra class:
Order::forUser($userId)->latest()->get();
```

---

## 9. Events, Listeners, and Observers — decoupling side effects

When *the main thing happened*, other things often need to react: send email, write an audit log, notify Slack, update a cache. Hard-coding those into the service couples your core flow to every side effect. **Events** invert that.

### Events + Listeners

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;

class OrderPlaced
{
    use Dispatchable;

    public function __construct(public readonly Order $order) {}
}
```

```php
<?php

namespace App\Listeners;

use App\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderConfirmation;

class SendOrderConfirmation implements ShouldQueue   // runs on the queue, not in-request
{
    public function handle(OrderPlaced $event): void
    {
        Mail::to($event->order->user)->send(new OrderConfirmation($event->order));
    }
}
```

Dispatch from the service after the core work:

```php
use App\Events\OrderPlaced;

OrderPlaced::dispatch($order);
```

> **Laravel 11/12 note:** listeners are **auto-discovered** by default — Laravel scans `app/Listeners` and registers any class method named `handle` or `__invoke` against the event type-hinted in its signature. You no longer need an `EventServiceProvider` `$listen` array (that provider doesn't exist by default in Laravel 11+). If you keep listeners elsewhere (e.g. in feature modules), point the scanner at those directories with `->withEvents(discover: [...])` in `bootstrap/app.php`. You can still register manually with `Event::listen()` for explicitness, and in production you should run `php artisan event:cache` (part of `php artisan optimize`) so discovery isn't re-run on every request.

The service no longer knows or cares who listens. Add a `WriteAuditLog` listener tomorrow without touching `CheckoutService`. That's the **Observer pattern** at the framework level and a clean application of the **Open/Closed Principle** (open for extension, closed for modification).

### Model Observers

For lifecycle hooks tied to a single model (creating, created, updating, deleted…), an **Observer** is cleaner than scattering logic in events:

```php
<?php

namespace App\Observers;

use App\Models\Order;
use Illuminate\Support\Str;

class OrderObserver
{
    public function creating(Order $order): void
    {
        $order->reference ??= 'ORD-' . Str::upper(Str::random(8));
    }
}
```

```php
// Laravel 11/12: attribute-based registration on the model
use App\Observers\OrderObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy(OrderObserver::class)]
class Order extends Model { /* ... */ }
```

**Gotcha to internalize:** Observers fire on Eloquent *model* events. Mass operations like `Order::where(...)->update([...])` or `->delete()` **bypass model events entirely** — they run a single SQL statement, so `updating`/`deleting` observers never fire. Don't hide business-critical logic in an observer if mass updates are possible.

---

## 10. Jobs — async and queued work

Some work shouldn't happen inside the request: sending email, calling slow third-party APIs, generating PDFs, processing uploads. **Queued Jobs** move it to a background worker so the user gets a fast response.

```bash
php artisan make:job ProcessOrderFulfillment
```

```php
<?php

namespace App\Jobs;

use App\Models\Order;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessOrderFulfillment implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;                 // retry up to 3 times
    public int $backoff = 30;              // wait 30s between retries

    public function __construct(public readonly Order $order) {}

    public function handle(): void
    {
        // slow work: talk to warehouse API, etc.
    }
}
```

> **Laravel 11/12 note:** the default `make:job` stub now uses the single `Illuminate\Foundation\Queue\Queueable` trait, which bundles what used to be four separate traits (`Dispatchable`, `InteractsWithQueue`, `Illuminate\Bus\Queueable`, `SerializesModels`). If you are reading older tutorials you will see all four imported individually — that still works, but the consolidated trait is the modern idiom. (`Batchable` is still added separately when you need batching.)

```php
ProcessOrderFulfillment::dispatch($order);            // queued
ProcessOrderFulfillment::dispatch($order)->onQueue('fulfillment')->delay(now()->addMinutes(5));
```

```bash
php artisan queue:work --queue=fulfillment,default
```

**Architectural point:** `SerializesModels` stores only the model's *key* and re-fetches it when the job runs — so jobs stay small and always work with fresh data, but the record must still exist at run time. Jobs, listeners, and actions overlap heavily; a common clean design is **service does core work → dispatches event → queued listener (or job) does the slow side effect.**

---

## 11. Dependency Injection, the service container, and SOLID

Everything above rests on Laravel's **service container** (the "IoC container"). DI means a class declares what it needs in its constructor and the container *provides* it, rather than the class `new`-ing its own dependencies.

```php
// BAD: hard dependency, untestable, violates Dependency Inversion
class CheckoutService
{
    public function checkout()
    {
        $gateway = new StripeGateway();  // can't substitute a fake in tests
    }
}

// GOOD: depend on an abstraction, container injects the concrete
class CheckoutService
{
    public function __construct(private readonly PaymentGateway $gateway) {}
}
```

Bind the interface once:

```php
$this->app->bind(PaymentGateway::class, StripeGateway::class);
// or a singleton (one shared instance for the whole request lifecycle):
$this->app->singleton(PaymentGateway::class, StripeGateway::class);
```

**SOLID applied to Laravel:**

- **S — Single Responsibility:** action classes; services that do one workflow; thin controllers.
- **O — Open/Closed:** events/listeners let you extend behavior without editing the publisher.
- **L — Liskov Substitution:** any `PaymentGateway` implementation must be drop-in interchangeable.
- **I — Interface Segregation:** small, role-specific interfaces (`OrderRepository`, not `MegaRepository`).
- **D — Dependency Inversion:** depend on `PaymentGateway`, not `StripeGateway`; the container wires it.

---

## 12. Avoid facades in domain code

Facades (`Mail::`, `DB::`, `Cache::`, `Auth::`) are static-looking accessors to container services. They're convenient in controllers and routes, but in **domain/service code** they hurt you:

```php
// Hidden dependency — nothing in the signature tells you this class needs the mailer:
class CheckoutService
{
    public function checkout(): void
    {
        \Illuminate\Support\Facades\Mail::to(/* ... */)->send(/* ... */);
    }
}
```

```php
// Better — dependency is explicit, injectable, and trivially mockable:
use Illuminate\Contracts\Mail\Mailer;

class CheckoutService
{
    public function __construct(private readonly Mailer $mailer) {}
}
```

**Why it matters:**
- **Explicitness:** the constructor lists every dependency. No surprises.
- **Testability:** inject a mock instead of relying on facade fakes/aliasing.
- **Honest coupling:** a class with ten injected dependencies is *telling you* it does too much. A facade hides that smell.

Facades are fine in controllers, commands, and quick scripts. The discipline is: **the deeper into your domain, the more you prefer injected contracts over static facades.** Note that Laravel facades *are* testable via `Mail::fake()` etc. — the objection is about hidden coupling and clarity, not impossibility.

---

## 13. Light DDD, the modular monolith, and folder structure

You do **not** need full Domain-Driven Design for most apps. But two ideas from it scale beautifully.

### Bounded contexts → modules

A **bounded context** is a part of the system with its own language and rules (Billing, Catalog, Shipping). As an app grows, grouping code by *feature/domain* instead of by *technical type* keeps related code together.

**Type-based (default Laravel) — fine until it isn't:**

```text
app/
├── Http/Controllers/      ← every controller in the app
├── Models/                ← every model
├── Services/              ← every service
```

**Modular monolith (group by domain):**

```text
app/
├── Modules/
│   ├── Billing/
│   │   ├── Actions/
│   │   ├── Data/          ← DTOs
│   │   ├── Models/
│   │   ├── Http/
│   │   └── Events/
│   ├── Catalog/
│   └── Shipping/
└── Support/               ← shared kernel
```

A **modular monolith** is one deployable app whose internals are split into well-isolated modules with clear boundaries (ideally, modules talk via events/interfaces, not by reaching into each other's models). It gives you most of the maintainability of microservices with none of the distributed-systems pain — and you can extract a true service later if a module truly needs it. This is the sweet spot for most growing Laravel apps.

> Adjust autoloading by adding the namespace to `composer.json`'s PSR-4 map, then `composer dump-autoload`. Some teams use packages like `nwidart/laravel-modules`; others wire it by hand.

**A full DDD layout** (Domain / Application / Infrastructure layers, value objects, aggregates) is appropriate only for genuinely complex business domains with multiple teams. For a CRUD-heavy app it's pure overhead — say so in an interview.

---

## 14. Testing implications

Good architecture is *defined by* how testable it is. Every pattern above pays off here. (Laravel 11/12 ship with **Pest** as the default test runner, so the examples below use Pest's `it()`/`expect()` syntax — the same ideas apply verbatim to PHPUnit's `class … extends TestCase` style if your project uses it.)

```php
<?php

use App\Models\User;
use App\Services\CheckoutService;
use App\Exceptions\InsufficientFundsException;

it('rejects checkout when the user cannot afford it', function () {
    $user = User::factory()->create(['wallet_balance' => 100]);

    $service = app(CheckoutService::class);

    expect(fn () => $service->checkout($user, [
        ['product_id' => 1, 'qty' => 1, 'price' => 5000],
    ]))->toThrow(InsufficientFundsException::class);
});
```

```php
// Faking framework side effects so tests stay fast and isolated:
use Illuminate\Support\Facades\{Event, Queue, Mail};

Event::fake();
Queue::fake();
Mail::fake();

// ... act ...

Event::assertDispatched(OrderPlaced::class);
Queue::assertPushed(ProcessOrderFulfillment::class);
```

Because the service is plain PHP with injected dependencies, you can unit-test it without HTTP. Because side effects go through events/jobs, you can assert "it dispatched X" instead of testing email delivery. **If a piece of code is hard to test, that's architectural feedback — it's too coupled.**

---

## 15. When NOT to over-engineer

This is the most important section, and the one juniors get wrong. Patterns are a cost: more files, more indirection, more for a new dev to learn. Apply them **when the pain they solve is real**, not preemptively.

- A 3-controller CRUD admin panel does **not** need repositories, DTOs, and a domain layer. Eloquent + Form Requests + Resources is plenty.
- Don't add an interface with exactly one implementation "in case." That's **speculative generality** (YAGNI — You Aren't Gonna Need It). Add the interface when the *second* implementation actually appears.
- Don't wrap every array in a DTO or every query in a repository on reflex.
- **Refactor toward** patterns when you feel the pain (a fat controller, duplicated logic, an untestable knot), not before.

The mark of a senior engineer is not knowing the most patterns — it's choosing the *least* structure that keeps the code changeable. Architecture is a means, not the goal.

---

## ⚠️ Common Mistakes & Gotchas

1. **Repository that just wraps the query builder.**
   *Symptom:* `findActive()`, `findActiveWithPosts()`, `findActiveWithPostsOrderedByDate()` — the method count explodes and you've reinvented Eloquent badly.
   *Fix:* Either use query scopes / thin query objects, or commit to the repository fully with a genuine abstraction. Don't half-do it.

2. **Putting multi-model business logic in a model.**
   *Symptom:* `Order::checkout()` that also charges the user and emails them. The model now changes for unrelated reasons (SRP violation).
   *Fix:* Extract a service or action. Keep models focused on data and single-entity behavior.

3. **Observers/events fail to fire on mass operations.**
   *Symptom:* `User::where(...)->update([...])` doesn't trigger your `updating` observer or `UserUpdated` event.
   *Fix:* Mass query-builder updates/deletes bypass model events. Loop and save individually if you need events, or move the logic somewhere that always runs (a service method).

4. **Facade-heavy services that are secretly untestable / over-coupled.**
   *Symptom:* A service littered with `DB::`, `Cache::`, `Http::` so its real dependencies are invisible.
   *Fix:* Inject contracts (`Mailer`, `Repository`, `CacheRepository`) via the constructor. The signature should reveal every dependency.

5. **Side effects inside a DB transaction that can't be rolled back.**
   *Symptom:* Sending email or dispatching a non-deferred job *inside* `DB::transaction()`; if the transaction rolls back, the email already went out.
   *Fix:* Dispatch events/jobs *after* the transaction commits. Options: chain `->afterCommit()` on the dispatch (`SomeJob::dispatch($x)->afterCommit()`), set `'after_commit' => true` on the queue connection in `config/queue.php`, mark a job/listener with the `ShouldDispatchAfterCommit`/`ShouldHandleEventsAfterCommit` interface (or the `$afterCommit = true` property), or register a callback with `DB::afterCommit(fn () => ...)`. Note: `DB::afterCommit()` only *defers* when called **inside** an open transaction — called with no active transaction it runs the callback immediately.

6. **Over-engineering a small app.**
   *Symptom:* Five layers of abstraction for a 200-line CRUD feature.
   *Fix:* Start simple. Add structure when change becomes painful, not on day one.

---

## ✅ Best Practices

- **Skinny controllers:** request in, delegate, response out. No business logic.
- **Validate at the edge** with Form Requests; **shape output** with API Resources.
- **Put single-entity logic on the model; multi-step/multi-model logic in services or actions.**
- **Depend on abstractions** (interfaces) where substitution is real; bind them in a provider.
- **Use DTOs at boundaries** that matter (API, jobs, cross-layer), not everywhere.
- **Decouple side effects** with events/listeners and observers; move slow work to jobs.
- **Dispatch jobs/events after commit** so rolled-back transactions don't leak side effects.
- **Inject contracts instead of facades** in domain code.
- **Group by feature/module** once type-based folders get crowded.
- **Let testability guide design** — hard to test means too coupled.
- **Default to the simplest structure** that keeps the code changeable; refactor toward patterns under pressure, not preemptively.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Where does business logic belong in a Laravel app?**
A: There's no single home; I use a hierarchy. Single-entity logic goes on the model. Multi-model workflows go in a service or single-purpose action class. Validation goes in Form Requests, output shaping in API Resources, side effects in events/jobs. Controllers stay thin. I pick the simplest level that keeps the code cohesive and testable.

**Q2. Do you use the Repository pattern in Laravel? Why or why not?**
A: Not by default. Eloquent already abstracts persistence, so a repository often just re-implements the query builder with a worse API and method explosion. I reach for it only when I need a genuine abstraction over multiple data sources or want to test domain logic with zero database. Otherwise query scopes or thin query objects give most of the benefit without the ceremony. (This balanced answer scores higher than a dogmatic "always" or "never.")

**Q3. Service class vs. action class — what's the difference?**
A: A service typically groups several related operations for a subsystem; an action is one class for one use case (often `__invoke()`). Actions enforce SRP more strictly and compose well but proliferate; services are coarser-grained. Both are plain, injectable PHP and serve the same goal of getting logic out of controllers and models.

**Q4. How does Laravel's service container resolve and inject dependencies under the hood?**
A: The container uses PHP **Reflection** to inspect a class's constructor, reads the type-hints, and recursively resolves each dependency — auto-wiring concretes and looking up bindings for interfaces. `bind` produces a new instance per resolve; `singleton` caches one instance for the container's lifetime. When you type-hint an interface, it consults the bindings registered (usually in a service provider's `register()` method) to know which concrete to build. Contextual bindings let different consumers get different implementations.

**Q5. Why avoid facades in domain code if Laravel ships with them?**
A: Facades hide dependencies — nothing in a class's signature reveals it uses the mailer or cache. That hurts readability, makes coupling invisible (a class with ten facade calls *looks* simple but isn't), and complicates reasoning about tests. Injecting contracts makes every dependency explicit and a constructor that's getting crowded is honest feedback that the class does too much. Facades are fine in controllers and quick scripts.

**Q6. What's the difference between an Observer and an Event/Listener?**
A: An Observer hooks Eloquent model lifecycle events (creating, updated, deleted) for one model — great for entity-scoped concerns like setting a slug. Events/Listeners are general-purpose: any code can dispatch a domain event and many listeners can react, even on the queue. Both apply the Observer pattern; events are more decoupled and reusable, observers are tighter to a model. Watch out: both miss mass query-builder operations.

**Q7. When would you NOT introduce these patterns?**
A: For small or CRUD-heavy apps. Adding interfaces with one implementation, DTOs around every array, or a domain layer to a 3-screen admin tool is speculative generality (YAGNI). I start simple and refactor toward patterns when change becomes painful. The goal is the least structure that keeps the code changeable.

**Q8. What is a modular monolith and why prefer it over microservices?**
A: One deployable app whose internals are isolated into domain modules (Billing, Catalog…) with clear boundaries, ideally communicating via events/interfaces rather than reaching into each other's internals. You get strong cohesion and maintainability without distributed-system pain (network failures, eventual consistency, ops overhead). If a module genuinely needs independent scaling later, the clean boundary makes extracting a real service much easier.

**Q9. Why dispatch jobs/events after a transaction commits?**
A: If you dispatch a non-deferred job or send an email *inside* a transaction that later rolls back, the side effect already happened against data that no longer exists. Use `DB::afterCommit()`, queued listeners, or the `after_commit` queue config so side effects only fire on successful commit.

**Q10. How do DTOs differ from Eloquent models, and when do you use each?**
A: A DTO is an immutable, typed data carrier with no persistence behavior; an Eloquent model is an Active Record tied to a table with relationships and lifecycle. I use DTOs at boundaries (API input, job payloads, cross-layer calls) to get type safety and decouple the wire format from the DB schema. For plain CRUD where the model *is* the data, an extra DTO is just overhead.

---

## 📋 Quick Reference / Cheat Sheet

```text
LAYER / RESPONSIBILITY MAP
  Controller .......... HTTP only: receive request, delegate, return response
  Form Request ........ input validation + authorization
  API Resource ........ output shaping (model -> JSON contract)
  Model ............... data + single-entity behavior + scopes/relationships
  Service ............. multi-step / multi-model business workflow
  Action .............. one use case, one class (__invoke)
  DTO ................. typed data carrier across boundaries
  Repository .......... persistence abstraction (use sparingly in Laravel)
  Event/Listener ...... decoupled side effects (sync or queued)
  Observer ............ Eloquent model lifecycle hooks
  Job ................. async / background / slow work
```

```php
// Bind interface -> implementation (in a ServiceProvider::register)
$this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
$this->app->singleton(PaymentGateway::class, StripeGateway::class);

// Inject (constructor) — container auto-resolves
public function __construct(private readonly OrderRepository $orders) {}

// Dispatch event / job
OrderPlaced::dispatch($order);
ProcessOrderFulfillment::dispatch($order)->onQueue('fulfillment');

// Run side effect only after DB commit (call INSIDE the transaction to defer;
// outside an open transaction it fires immediately)
DB::transaction(function () use ($order) {
    // ...write...
    DB::afterCommit(fn () => OrderPlaced::dispatch($order));
});

// Or defer the dispatch itself:
ProcessOrderFulfillment::dispatch($order)->afterCommit();

// spatie/laravel-data DTO from request
$data = CheckoutData::from($request);
```

```bash
php artisan make:request CheckoutRequest
php artisan make:resource OrderResource
php artisan make:job ProcessOrderFulfillment
php artisan make:event OrderPlaced
php artisan make:listener SendOrderConfirmation --event=OrderPlaced
php artisan make:observer OrderObserver --model=Order
php artisan make:provider RepositoryServiceProvider
php artisan queue:work
```

| Version note | What changed |
|--------------|--------------|
| Laravel 11/12 | Providers in `bootstrap/providers.php`; listeners auto-discovered; no `EventServiceProvider`/`RouteServiceProvider` boilerplate |
| Laravel 11/12 | `#[ObservedBy(...)]` and `#[ScopedBy(...)]` attribute registration |
| Laravel 12 | Default `make:job` stub uses the single `Illuminate\Foundation\Queue\Queueable` trait |
| Laravel 10 | Providers/listeners registered in `config/app.php` and `EventServiceProvider::$listen` |
| PHP 8.4 | Property hooks and asymmetric visibility (`public private(set)`) — useful for richer DTOs/value objects without full getters/setters |
| PHP 8.2+ | `readonly` classes for immutable DTOs |
| PHP 8.1+ | Enums, readonly *properties*, first-class callable syntax, `never` return type |
| PHP 8.0+ | Constructor property promotion, named arguments, `match` |

---

## 🧪 Mini Exercises

1. **Refactor a fat controller.** Take the `CheckoutController::store` from Section 2 and refactor it into: a `CheckoutRequest`, a `CheckoutService` (wrapping the write in a transaction), and an `OrderResource`. The controller body should be three lines or fewer. Fix the hidden N+1 in the total calculation along the way.

2. **Interface + binding.** Define a `NotificationChannel` interface with a `send(string $to, string $message): void` method. Write two implementations (`SlackChannel`, `EmailChannel`), bind one in a service provider, and inject it into a service. Then write a test that binds a *fake* implementation and asserts it was called.

3. **Events for side effects.** Make your `CheckoutService` dispatch an `OrderPlaced` event after the order commits (use `DB::afterCommit`). Add two queued listeners — one that emails the customer, one that writes an audit log — without modifying the service after the first event is added.

4. **DTO boundary.** Introduce a `CheckoutData` DTO (plain readonly class *or* spatie/laravel-data) so your service signature is `checkout(CheckoutData $data): Order` instead of taking a raw array. Build it from the Form Request.

5. **Decide: repository or not?** Given a feature that reads orders from Eloquent today but must *also* read archived orders from an external REST API next quarter, sketch whether you'd introduce an `OrderRepository` interface now or wait. Write one paragraph justifying your choice in terms of YAGNI vs. Dependency Inversion.
