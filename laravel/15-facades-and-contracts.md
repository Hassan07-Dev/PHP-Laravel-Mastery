# Facades & Contracts

Laravel gives you two complementary ways to reach the powerful machinery sitting inside its **service container**: **Facades** (a terse, static-looking syntax) and **Contracts** (the interfaces that define what that machinery promises to do). On the surface they feel like opposites — one is a convenience, the other is an abstraction — but they are two views of the same underlying system. Understanding both, and *when to reach for which*, is one of the clearest signals of a developer who actually understands how Laravel is wired together rather than just memorizing syntax.

> A **service container** (a.k.a. IoC container, "Inversion of Control" container) is the object that knows how to build and hand out your application's services. When you ask for `Cache`, the container is the thing that constructs the right cache store and gives it to you. Both facades and contracts ultimately resolve through this container.

---

## **What you'll learn**

- What a facade *actually is* — a static proxy to an object resolved from the service container — and the exact mechanism (`getFacadeAccessor` + `__callStatic`) that makes `Cache::get()` work.
- **Real-time facades** (the `Facades\` prefix) and when they save you boilerplate.
- The full roster of common facades (`Route`, `DB`, `Cache`, `Auth`, `Log`, `Storage`, `Mail`, `Queue`, `Event`, `Validator`, `Config`, `Session`, `Gate`, `URL`) and what each proxies to.
- **Helper functions vs facades** — why both exist and which to pick.
- **Contracts**: Laravel's interfaces, why "coding to a contract" gives you swappability, and how they pair with dependency injection (DI).
- The **facade vs DI** debate — the real tradeoffs, not dogma.
- **Facade aliases** and how the `class_alias` magic lets you write `Cache` instead of `Illuminate\Support\Facades\Cache`.
- **Testing with fakes** — `Mail::fake`, `Queue::fake`, `Event::fake`, `Storage::fake`, `Bus::fake`, and the `shouldReceive` mocking approach.

---

## 1. Why facades exist (the WHY before the HOW)

Laravel resolves almost everything through its service container. Without facades, calling the cache might look like this:

```php
<?php

use Illuminate\Contracts\Cache\Repository as CacheRepository;

class ReportController
{
    public function __construct(
        private CacheRepository $cache,
    ) {}

    public function show(): string
    {
        return $this->cache->get('report:daily', 'no data yet');
    }
}
```

That is perfectly correct — and in many cases *preferable* (see the DI debate below). But it is verbose: you must inject the dependency, name it, and type-hint the contract. For quick reach-ins, Laravel offers a shorthand:

```php
<?php

use Illuminate\Support\Facades\Cache;

class ReportController
{
    public function show(): string
    {
        return Cache::get('report:daily', 'no data yet');
    }
}
```

`Cache::get()` *looks* static, but it is not a static method on a static class. It is a **facade**: a thin class that forwards every static call to a real object instance pulled from the container. The "WHY" is ergonomics — facades give you expressive, memorable, IDE-discoverable syntax without sacrificing the testability and swappability of a container-resolved object. The trick is that the object behind the facade is still a fully mockable, swappable, container-managed instance.

> **Jargon:** a **proxy** is an object that stands in for another and forwards calls to it. A facade is specifically a *static proxy* — you call it statically, it forwards to an instance.

---

## 2. How facades work under the hood

Every facade extends `Illuminate\Support\Facades\Facade` and implements exactly **one** required method: `getFacadeAccessor()`. That method returns the *key* used to resolve the underlying object from the container.

```php
<?php

namespace Illuminate\Support\Facades;

class Cache extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'cache'; // the container binding key
    }
}
```

When you write `Cache::get('key')`, PHP looks for a static method `get()` on the `Cache` class. There isn't one. So PHP triggers the magic method **`__callStatic`**, which the base `Facade` class implements:

```php
<?php
// Simplified from Illuminate\Support\Facades\Facade

public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot(); // resolve the real object

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args); // forward the call
}
```

`getFacadeRoot()` calls `resolveFacadeInstance(getFacadeAccessor())`, which essentially does `app()->make('cache')` (with a small static cache so repeated calls reuse the same instance). So the full chain for `Cache::get('key')` is:

```
Cache::get('key')
  -> __callStatic('get', ['key'])           // magic method on base Facade
  -> getFacadeRoot()                          // resolve instance
  -> resolveFacadeInstance('cache')           // getFacadeAccessor() returned 'cache'
  -> app()->make('cache')                      // container builds CacheManager
  -> $cacheManager->get('key')                 // real method call
```

**This is the single most important thing to understand about facades:** they are not magic statics; they are syntactic sugar over `app()->make(...)->method(...)`. Because the object is resolved from the container, you can rebind it — which is exactly what fakes and mocks do.

```php
<?php
// Proof: these two lines are functionally equivalent
Cache::get('key');
app('cache')->get('key');
```

### Why facades are testable (the key consequence)

The base `Facade` class exposes a `swap()` method (and helpers like `shouldReceive()`). When you call `Cache::shouldReceive('get')->andReturn('x')`, the facade swaps the container's `'cache'` binding for a Mockery mock. Every subsequent `Cache::get()` in the code under test now hits the mock. No global static state to reset by hand — the container is reset between tests.

```php
<?php

Cache::shouldReceive('get')
    ->once()
    ->with('report:daily')
    ->andReturn('cached value');

// Now anywhere in the code under test:
// Cache::get('report:daily') === 'cached value'
```

> Under the hood `shouldReceive` is defined on the base `Facade` via `createFreshMockInstance()`, which builds a `Mockery` mock and `swap()`s it into the container. This is why facades — unlike plain static classes — are mockable.

---

## 3. Facade aliases (why you can type `Cache`, not the FQCN)

You may have noticed you write `use Illuminate\Support\Facades\Cache;`. But in Blade views and Tinker you can often write just `Cache` with no import. That is the **aliasing** system.

Laravel registers class aliases via PHP's `class_alias()` function. The registration happens during boot in `Illuminate\Foundation\Bootstrap\RegisterFacades`, which builds the `AliasLoader` from three merged sources:

```php
<?php
// Illuminate\Foundation\Bootstrap\RegisterFacades::bootstrap() — Laravel 11/12
AliasLoader::getInstance(array_merge(
    Facade::defaultAliases()->all(),                          // framework defaults
    $app->make('config')->get('app.aliases', []),            // your config/app.php aliases (defaults to [])
    $app->make(PackageManifest::class)->aliases()             // package-provided aliases
))->register();
```

The **framework defaults** (`Cache`, `DB`, `Auth`, `Route`, …) are NOT stored in `config/app.php` or `bootstrap/app.php` — in Laravel 11/12 they live in the static method `Illuminate\Support\Facades\Facade::defaultAliases()`. (In Laravel 10 and earlier they were a big literal `aliases` array inside `config/app.php`; the slimmed Laravel 11/12 `config/app.php` no longer ships that key, so `config('app.aliases', [])` falls back to an empty array unless you add it back.)

The `AliasLoader` then registers an SPL autoloader that, when PHP first references the unqualified class `Cache`, transparently aliases it to `Illuminate\Support\Facades\Cache`.

```php
<?php
// Conceptually, the AliasLoader does this lazily the first time PHP hits an unknown class:
class_alias(\Illuminate\Support\Facades\Cache::class, 'Cache');
```

**In PHP source files you should still `use` the fully-qualified facade** — relying on the global alias in app code couples you to that config and confuses IDEs. The global alias is mostly a convenience for Blade and Tinker.

To add your own alias in Laravel 12, add an `aliases` array to the existing `config/app.php` (it ships by default, just without that key):

```php
<?php
// config/app.php — add this array (the slim default does not include it)
'aliases' => [
    'Pdf' => App\Support\Facades\Pdf::class,
],
```

> The merge above means you only list *your own* aliases here — the framework defaults from `Facade::defaultAliases()` are added automatically, so you do not (and should not) re-list `Cache`, `DB`, etc.

---

## 4. Real-time facades

Sometimes you want facade-style syntax for *your own* class without writing a facade class for it. **Real-time facades** let you do this by prefixing the class's FQCN with `Facades\` in the `use` statement.

```php
<?php

namespace App\Services;

class Publisher
{
    public function publish(string $title): string
    {
        return "Published: {$title}";
    }
}
```

```php
<?php

namespace App\Models;

use Facades\App\Services\Publisher; // note the leading "Facades\"
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    public function publish(): string
    {
        // Calls Publisher::publish() statically, but Laravel resolves
        // a real Publisher instance from the container behind the scenes.
        return Publisher::publish($this->title);
    }
}
```

When PHP tries to load `Facades\App\Services\Publisher`, Laravel's bootstrapper intercepts the class load, generates a facade on the fly whose accessor returns `App\Services\Publisher`, and resolves it from the container. The benefit: you keep your real class injectable/testable, but you can call it statically.

```php
<?php
// And you can still fake it in tests, because it's a real facade:
use Facades\App\Services\Publisher;

Publisher::shouldReceive('publish')->andReturn('faked!');
```

> **Trade-off:** real-time facades hide a dependency (you no longer see `Publisher` in the constructor). Use them for terse call sites in models or simple flows, not as a blanket replacement for DI. The cleaner long-term design is usually constructor injection.

---

## 5. The common facades (quick tour)

Each facade proxies to a concrete service. Knowing the *binding key* and the *contract* behind each is interview gold.

| Facade | Container binding | What it does |
| --- | --- | --- |
| `Route` | `router` | Define routes (usually in route files) |
| `DB` | `db` | Query builder, raw SQL, transactions |
| `Cache` | `cache` | Read/write the cache store |
| `Auth` | `auth` | Authentication, current user |
| `Log` | `log` | Write to log channels (PSR-3) |
| `Storage` | `filesystem` | Filesystem / cloud disk access |
| `Mail` | `mailer` | Send email |
| `Queue` | `queue` | Push jobs onto queues |
| `Event` | `events` | Dispatch / listen to events |
| `Validator` | `validator` | Build validators manually |
| `Config` | `config` | Read/write config at runtime |
| `Session` | `session` | Read/write session data |
| `Gate` | (Gate contract) | Authorization checks |
| `URL` | `url` | Generate URLs |

```php
<?php

use Illuminate\Support\Facades\{Route, DB, Cache, Auth, Log, Storage, Mail, Queue, Event, Validator, Config, Session, Gate, URL};

// Routing
Route::get('/posts', [PostController::class, 'index'])->name('posts.index');

// Database
$users = DB::table('users')->where('active', true)->get();
DB::transaction(fn () => DB::table('orders')->insert(['total' => 100]));

// Cache (remember = get-or-compute-and-store)
$value = Cache::remember('stats', now()->addMinutes(10), fn () => expensiveStats());

// Auth
$user = Auth::user();
if (Auth::check()) { /* logged in */ }

// Log (PSR-3 levels)
Log::info('Order placed', ['order_id' => 42]);
Log::channel('slack')->critical('Payment gateway down');

// Storage
Storage::disk('s3')->put('avatars/1.png', $contents);
$url = Storage::disk('public')->url('avatars/1.png');

// Mail
Mail::to($user)->send(new \App\Mail\WelcomeMail($user));

// Queue
Queue::push(new \App\Jobs\ProcessPodcast($podcast));

// Event
Event::dispatch(new \App\Events\OrderShipped($order));

// Validator (manual)
$validator = Validator::make($data, ['email' => 'required|email']);
if ($validator->fails()) { /* ... */ }

// Config (read & runtime-set)
$timezone = Config::get('app.timezone');
Config::set('services.feature.enabled', true);

// Session
Session::put('cart_id', 99);
$cartId = Session::get('cart_id');

// Gate
if (Gate::allows('update-post', $post)) { /* ... */ }

// URL
$url = URL::route('posts.index');
$signed = URL::signedRoute('unsubscribe', ['user' => $user->id]);
```

> **Note on `Route`:** although `Route::get()` is a facade call, routes are normally defined in `routes/web.php`/`routes/api.php` rather than scattered around your code. Treat `Route` as a registration tool, not a runtime helper.

---

## 6. Helper functions vs facades

Laravel ships **global helper functions** that often mirror facades. They are usually thinner — many resolve the same binding and call a method.

```php
<?php
// These pairs are essentially equivalent:
Cache::get('k');            cache('k');           // helper, no args = the store
Config::get('app.name');    config('app.name');
Session::get('cart');       session('cart');
URL::to('/home');           url('/home');
Auth::user();               auth()->user();
Log::info('hi');            logger('hi');
Response::json([]);         response()->json([]);   // both exist (Response facade + response() helper)
View::make('home');         view('home');           // both exist (View facade + view() helper)
```

How to choose:

- **Helpers** read naturally in templates and quick code: `config('app.name')`, `route('home')`, `now()`.
- **Facades** are more discoverable in IDEs (autocomplete of methods) and clearer when chaining: `Cache::tags(['a'])->put(...)`.
- For **testing**, the facade form is easier to swap because you can call `Cache::shouldReceive(...)` / `Cache::spy()` on it. (Note: not every facade has a `fake()` — only `Mail`, `Queue`, `Event`, `Bus`, `Notification`, `Storage`, and `Http` ship dedicated fakes. `Cache` has **no** `Cache::fake()`; for cache you use the in-memory `array` driver, `shouldReceive`, or `spy`.) Because helpers like `cache()` resolve the *same* `cache` binding, a `shouldReceive`/`swap` on the facade also affects the helper.

Both ultimately hit the container, so it is mostly a style choice. Be consistent within a file.

---

## 7. Contracts (interfaces) — the "code to an abstraction" idea

A **contract** in Laravel is simply an **interface** under the `Illuminate\Contracts\*` namespace that defines what a service can do, without saying *how*. Facades and helpers give you a concrete object; contracts let you depend on the *capability* instead of the *implementation*.

> **Jargon:** "coding to an interface (contract), not an implementation" means your class declares it needs *something that can cache* (the `Cache\Repository` contract) rather than *the specific Redis cache class*. You can then swap implementations without touching the consumer.

Common contracts (the interface → typical facade/helper pairing):

| Contract | Facade |
| --- | --- |
| `Illuminate\Contracts\Cache\Repository` | `Cache` |
| `Illuminate\Contracts\Auth\Authenticatable` | (the user model) |
| `Illuminate\Contracts\Auth\Guard` | `Auth` |
| `Illuminate\Contracts\Mail\Mailer` | `Mail` |
| `Illuminate\Contracts\Queue\Queue` | `Queue` |
| `Illuminate\Contracts\Events\Dispatcher` | `Event` |
| `Illuminate\Contracts\Filesystem\Filesystem` | `Storage` |
| `Illuminate\Contracts\Config\Repository` | `Config` |
| `Psr\Log\LoggerInterface` | `Log` |
| `Illuminate\Contracts\Validation\Factory` | `Validator` |

Because Laravel binds these contracts in the container, you can type-hint the **interface** and the container injects the right concrete class:

```php
<?php

namespace App\Services;

use Illuminate\Contracts\Cache\Repository as Cache;

class StatsService
{
    // Type-hint the CONTRACT. Laravel resolves the concrete CacheManager
    // store automatically because the binding exists.
    public function __construct(
        private Cache $cache,
    ) {}

    public function dailyActiveUsers(): int
    {
        return $this->cache->remember('dau', 600, fn () => $this->compute());
    }

    private function compute(): int
    {
        return \App\Models\User::whereDate('last_seen_at', today())->count();
    }
}
```

### Swappability: bind your own contract

The real payoff is *your own* contracts. Define an interface, write implementations, and bind one in a service provider.

```php
<?php

namespace App\Contracts;

interface PaymentGateway
{
    public function charge(int $amountInCents, string $token): string; // returns charge id
}
```

```php
<?php

namespace App\Services;

use App\Contracts\PaymentGateway;

final class StripeGateway implements PaymentGateway
{
    public function charge(int $amountInCents, string $token): string
    {
        // ... call Stripe ...
        return 'ch_123';
    }
}
```

```php
<?php

namespace App\Providers;

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind the contract to a concrete implementation.
        $this->app->bind(PaymentGateway::class, StripeGateway::class);
    }
}
```

```php
<?php

namespace App\Http\Controllers;

use App\Contracts\PaymentGateway;

class CheckoutController
{
    // Consumer depends only on the contract. Swap StripeGateway for
    // PaddleGateway in the provider and nothing here changes.
    public function __construct(
        private PaymentGateway $gateway,
    ) {}

    public function pay(): string
    {
        return $this->gateway->charge(2_000, 'tok_visa');
    }
}
```

This is the essence of contracts: **decoupling**. The consumer is welded to behavior, not to a vendor.

---

## 8. The facade vs dependency-injection debate

This is a classic interview topic. Both reach the container; the difference is *how the dependency is expressed*.

**Facades / helpers:**
- Pros: terse, no constructor boilerplate, great for one-off calls, fully mockable in tests via fakes.
- Cons: dependencies are **hidden** (you can't tell what a class needs by reading its constructor), which can mask how much a class actually does (a code-smell indicator for over-large classes).

**Constructor injection (with contracts):**
- Pros: dependencies are **explicit and visible**; encourages small, focused classes; trivial to unit-test with plain mocks; the standard for library/package code.
- Cons: more verbose; can feel heavy for trivial reach-ins.

**Pragmatic guidance (what to say in an interview):**
- In **application code** (controllers, jobs, services), facades are idiomatic Laravel and perfectly fine — they are testable. Use them freely.
- In **reusable packages/libraries**, prefer injecting **contracts** so consumers can swap implementations and you don't depend on the app's facade aliases.
- If a class accumulates many facade calls, that's a hint it's doing too much — that "hidden dependency" smell is actually useful feedback.

> The honest answer: "Facades are not anti-patterns in Laravel because they resolve from the container and are mockable. I reach for facades in app code for ergonomics, and inject contracts in packages or where I want explicit, swappable dependencies."

---

## 9. Testing with facade fakes

The fake methods replace the real service with an in-memory spy that records calls and lets you assert on them — **without** sending real emails, dispatching real jobs, or touching the real disk.

> The examples below use **Pest**, the default test runner in a fresh Laravel 12 install (`it('...', function () { ... })`). The exact same facade calls work in classic PHPUnit — just move them into a `public function test_...()` method on a class extending `Tests\TestCase`. Fakes require the application to be booted, which both styles get for free through Laravel's base `TestCase`.

```php
<?php

use Illuminate\Support\Facades\{Mail, Queue, Event, Storage, Bus};
use App\Mail\OrderShipped;
use App\Jobs\ProcessPodcast;
use App\Events\OrderPlaced;

it('sends the shipped email', function () {
    Mail::fake();

    // ...run the code that should send mail...
    (new \App\Actions\ShipOrder())->handle($order);

    Mail::assertSent(OrderShipped::class);
    Mail::assertSent(OrderShipped::class, fn ($mail) => $mail->order->is($order));
    Mail::assertNothingSent();      // or this, if nothing should send
    Mail::assertSentCount(1);
});

it('queues podcast processing', function () {
    Queue::fake();

    ProcessPodcast::dispatch($podcast);

    Queue::assertPushed(ProcessPodcast::class);
    Queue::assertPushedOn('audio', ProcessPodcast::class);
    Queue::assertNothingPushed();
});

it('dispatches the order event', function () {
    Event::fake();

    OrderPlaced::dispatch($order);

    Event::assertDispatched(OrderPlaced::class);
    Event::assertDispatchedTimes(OrderPlaced::class, 1);
    // Optionally fake only specific events:
    // Event::fake([OrderPlaced::class]);
});

it('stores the upload', function () {
    Storage::fake('avatars'); // fakes the "avatars" disk in memory

    $file = \Illuminate\Http\UploadedFile::fake()->image('me.jpg');
    $file->storeAs('', 'me.jpg', 'avatars');

    Storage::disk('avatars')->assertExists('me.jpg');
    Storage::disk('avatars')->assertMissing('nope.jpg');
});

it('dispatches a job batch', function () {
    Bus::fake();

    \App\Jobs\BigImport::dispatch();

    Bus::assertDispatched(\App\Jobs\BigImport::class);
    Bus::assertBatched(fn ($batch) => $batch->jobs->count() === 3);
});
```

### `shouldReceive` (Mockery) vs `fake`

- Use **`fake()`** when you want a real-ish spy you can make *assertions* against after the fact (`assertSent`, `assertPushed`). Best for outgoing side-effects.
- Use **`shouldReceive()`** when you need to *control return values* of a dependency the code reads back (e.g. `Cache::shouldReceive('get')->andReturn(...)`).

```php
<?php
// Controlling a return value with a partial mock:
Cache::partialMock()
    ->shouldReceive('get')
    ->with('flag')
    ->andReturn(true);

Auth::shouldReceive('id')->andReturn(7);
```

> **Important Laravel 12 note:** `Event::fake()` stops listeners (including model observers) from running, which can surprise you if a listener performs setup your test relies on. Use `Event::fake([SpecificEvent::class])` to fake selectively, or `Event::fakeExcept([...])`.

---

## ⚠️ Common Mistakes & Gotchas

1. **Calling a facade before the application is booted.**
   In code that runs before the container is ready (some early bootstrap paths, certain static initializers), `Cache::get()` throws `RuntimeException: A facade root has not been set.` because `getFacadeRoot()` found no application.
   **Fix:** ensure the framework is booted, or resolve lazily inside a method that runs after boot. In tests, extend Laravel's `TestCase` so the app is bootstrapped.

2. **Putting `Foo::fake()` *after* the code that uses `Foo`.**
   Fakes only intercept calls made *after* the swap. If you dispatch the job and *then* call `Queue::fake()`, the real queue already ran.
   **Fix:** call `Mail::fake()` / `Queue::fake()` / etc. at the **top** of the test, before exercising the code.

3. **`Event::fake()` silently disabling observers/listeners.**
   Faking all events stops model observers from firing, so records that a listener was supposed to create won't exist.
   **Fix:** fake only the events you assert on: `Event::fake([OrderPlaced::class])`.

4. **Type-hinting a facade class in a constructor.**
   `public function __construct(private Cache $cache)` where `Cache` is the *facade* (`Illuminate\Support\Facades\Cache`) does **not** inject the cache repository — the facade is not a resolvable service.
   **Fix:** inject the **contract** `Illuminate\Contracts\Cache\Repository`, not the facade.

5. **Relying on the global alias (`\Cache`) inside namespaced source files without importing.**
   In a namespaced file, `Cache::get()` resolves to `App\Whatever\Cache`, not the facade, unless you `use Illuminate\Support\Facades\Cache;`. This produces confusing "class not found" errors.
   **Fix:** always `use` the fully-qualified facade in PHP source; reserve unqualified aliases for Blade/Tinker.

6. **Forgetting that `Storage::fake('disk')` only fakes the named disk.**
   `Storage::fake()` with no argument fakes the default disk; asserting on `Storage::disk('s3')` then hits the real S3 config.
   **Fix:** fake the exact disk name your code uses, and assert against the same disk.

---

## ✅ Best Practices

- **Inject contracts in packages and reusable libraries; use facades freely in app code.** Both are testable; choose based on visibility needs.
- **Always import the FQCN facade** (`use Illuminate\Support\Facades\Cache;`). Don't lean on global aliases in source.
- **Call `fake()` first** in tests, then assert with `assert*` helpers afterward.
- **Prefer `Event::fake([...])` selectively** to avoid disabling observers you rely on.
- **Bind your own contracts in a service provider** so you can swap implementations (fake gateways, alternate drivers) without touching consumers.
- **Watch for facade overload** — a class littered with many different facade calls is usually doing too much; extract a service.
- **Use real-time facades sparingly** — they hide dependencies; prefer constructor injection unless terseness genuinely wins.
- **Be consistent within a file**: pick facade *or* helper style and stick with it.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is a facade in Laravel?**
A: A class providing a static-looking interface to an object resolved from the service container. It is a *static proxy*, not a true static class — calls are forwarded to a real, mockable instance.

**Q2. (Under the hood) How does `Cache::get()` actually work?**
A: `Cache` has no static `get()`, so PHP invokes the base `Facade::__callStatic('get', $args)`. That calls `getFacadeRoot()`, which resolves `getFacadeAccessor()` (`'cache'`) from the container via `app()->make('cache')`, then forwards `->get(...)` to that `CacheManager` instance. So `Cache::get()` ≡ `app('cache')->get()`.

**Q3. If facades are static, why aren't they considered an anti-pattern like global static state?**
A: Because the underlying object comes from the container and can be swapped. `Facade::swap()`/`shouldReceive()` rebind it for tests, so there's no untestable hard-coded dependency — the usual complaint about statics doesn't apply.

**Q4. What's the difference between a facade and a contract?**
A: A facade is a concrete access shortcut (gives you an object); a contract is an interface defining a capability. You depend on a contract to stay implementation-agnostic and swappable; you use a facade for convenient access to the concrete service.

**Q5. Facade vs dependency injection — when do you use each?**
A: Facades for ergonomic reach-ins in app code (still testable via fakes); constructor injection of contracts when you want explicit, visible, swappable dependencies — especially in packages. Many facade calls in one class signal it's doing too much.

**Q6. What is a real-time facade?**
A: Prefixing a class's namespace with `Facades\` in the `use` statement lets you call your own class statically; Laravel generates a facade on the fly and resolves the instance from the container. Keeps the class injectable/fakeable while allowing terse static calls.

**Q7. How do facade aliases let you write `Cache` instead of the full path?**
A: During boot, `RegisterFacades` builds the `AliasLoader` from `Facade::defaultAliases()` (framework defaults) merged with `config('app.aliases', [])` and package aliases. The loader registers an autoloader that `class_alias()`es the unqualified `Cache` to `Illuminate\Support\Facades\Cache` the first time PHP references it. In Laravel 11/12 the framework defaults live in `Facade::defaultAliases()`, not in `config/app.php`. Used mainly in Blade/Tinker.

**Q8. How do you test that an email was sent without sending it?**
A: `Mail::fake()` before the code, then `Mail::assertSent(WelcomeMail::class)`. The fake swaps the mailer for an in-memory spy that records sends.

**Q9. What's the difference between `Foo::fake()` and `Foo::shouldReceive()`?**
A: `fake()` installs a spy you assert against afterward (good for side-effects like mail/queue/events). `shouldReceive()` installs a Mockery expectation that *controls return values* (good when the code reads a value back, e.g. `Cache::get`).

**Q10. Why can't you type-hint the `Cache` facade in a constructor for injection?**
A: The facade class isn't a container binding — it's a proxy. Inject the contract `Illuminate\Contracts\Cache\Repository` instead, which the container can resolve.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
// --- Mechanism ---
// Facade::method()  ==  app(getFacadeAccessor())->method()
//   __callStatic -> getFacadeRoot -> resolveFacadeInstance -> app()->make(key)

// --- Defining a custom facade ---
class Pdf extends Illuminate\Support\Facades\Facade {
    protected static function getFacadeAccessor(): string { return 'pdf'; }
}

// --- Real-time facade (no facade class needed) ---
use Facades\App\Services\Publisher;   // leading "Facades\"
Publisher::publish($title);

// --- Inject a contract (NOT the facade) ---
use Illuminate\Contracts\Cache\Repository as Cache;
public function __construct(private Cache $cache) {}

// --- Bind your own contract ---
$this->app->bind(App\Contracts\PaymentGateway::class, App\Services\StripeGateway::class);

// --- Common facade -> binding ---
// Route->router  DB->db  Cache->cache  Auth->auth  Log->log
// Storage->filesystem  Mail->mailer  Queue->queue  Event->events
// Validator->validator  Config->config  Session->session  URL->url  Gate->(Gate)

// --- Helper vs facade ---
Cache::get('k');  cache('k');     Config::get('x');  config('x');
Auth::user();     auth()->user(); Log::info('m');    logger('m');
```

```php
<?php
// --- Fakes (call BEFORE the code under test) ---
Mail::fake();    Mail::assertSent(WelcomeMail::class);  Mail::assertNothingSent();
Queue::fake();   Queue::assertPushed(Job::class);       Queue::assertPushedOn('q', Job::class);
Event::fake();   Event::assertDispatched(Evt::class);   Event::fake([Evt::class]); // selective
Storage::fake('disk'); Storage::disk('disk')->assertExists('f.png');
Bus::fake();     Bus::assertDispatched(Job::class);     Bus::assertBatched(fn($b) => ...);

// --- Mocking return values ---
Cache::shouldReceive('get')->with('k')->andReturn('v');
Auth::shouldReceive('id')->andReturn(7);
```

---

## 🧪 Mini Exercises

1. **Trace the chain.** Without running code, write out every step `Storage::disk('s3')->put('a.txt', 'hi')` goes through, from `__callStatic` to the concrete filesystem object. Name the `getFacadeAccessor()` value and the container binding involved.

2. **Build a custom facade.** Create a `Greeter` service with a `greet(string $name): string` method, bind it in a service provider under the key `greeter`, write a `Greeter` facade whose accessor returns `'greeter'`, and call `Greeter::greet('Sam')` from a route. Then rewrite the call using a **real-time facade** instead of the custom facade class.

3. **Code to a contract.** Define a `NotificationChannel` interface with `send(string $to, string $message): void`, implement `SmsChannel` and `LogChannel`, bind one in a provider, and inject the *contract* into a controller. Show how switching the binding swaps behavior without changing the controller.

4. **Test outgoing side-effects.** For an action that emails a receipt and queues a `GenerateInvoice` job, write a test using `Mail::fake()` and `Queue::fake()` that asserts exactly one email and one queued job, asserting the job lands on the `invoices` queue.

5. **Mock a read.** Write a test where a service reads a feature flag via `Cache::get('beta_enabled')`. Use `Cache::shouldReceive('get')` to force it to `true`, and assert the service takes the "beta" code path. Explain in a comment why `fake()` would be the wrong tool here.
