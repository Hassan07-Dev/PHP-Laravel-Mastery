# Laravel Request Lifecycle

> **Module 02 — Laravel track.** Target stack: **PHP 8.4** and **Laravel 12**. Differences for PHP 8.1–8.3 and Laravel 10/11 are called out inline.

Every HTTP request that reaches a Laravel app travels through a precise, predictable sequence of stages before a response is sent back to the browser. This sequence is the **request lifecycle**. If you treat the framework as a black box, debugging becomes guesswork: you don't know *where* a header was added, *why* a route returns 404, or *where* you could hook in to mutate the response. Once you can name every stage and point to the file responsible for it, Laravel stops being magic and becomes a tool you can reason about. That is exactly what senior interviewers probe for.

---

## **What you'll learn**

- The full journey of a request: web server → `public/index.php` → kernel → router → controller → response → terminate.
- How `bootstrap/app.php` builds the **service container** (the `Application` object) and why everything else depends on it.
- The Laravel 11+ **fluent `bootstrap/app.php`** configuration model versus the older `app/Http/Kernel.php` class.
- What **service providers** do, and the difference between the `register` and `boot` phases.
- How the **middleware pipeline** works, and where **global** vs **route** vs **terminable** middleware execute.
- How **bootstrappers** load environment variables, configuration, and facades before your code ever runs.
- Why this knowledge is the single most useful debugging and "where do I extend it?" tool in your kit.

---

## Why the lifecycle matters before the "how"

Imagine three real bugs:

1. A response is missing a `Cache-Control` header you swear you set.
2. A route returns `404` locally but works in production.
3. A queued job runs *after* the response is sent and you don't understand why the user already saw "Done".

Each of these is a *lifecycle* question. The header is added by middleware (which runs at a specific stage). The 404 is a routing-stage decision. The post-response work is **terminable middleware** running after the response is flushed. Without a mental model of the stages, you'd be adding `dd()` calls blindly. With one, you go straight to the right file.

The lifecycle is also where you find **extension points**: service providers (to bind things), middleware (to inspect/modify requests and responses), and the router (to add routes). Knowing the order tells you *which* hook is the right one.

---

## The 30,000-foot text diagram

Here is the entire journey on one screen. Refer back to it as we walk through each stage.

```text
                    ┌─────────────────────────────────────────────┐
   Browser ───────► │  Web server (Nginx / Apache / `artisan serve`)│
                    └─────────────────────────────────────────────┘
                                      │  (all non-file URLs rewritten here)
                                      ▼
                    ┌─────────────────────────────────────────────┐
                    │  public/index.php  (the single entry point)  │
                    └─────────────────────────────────────────────┘
                                      │ 1. require autoloader (vendor/autoload.php)
                                      │ 2. require bootstrap/app.php  ──► $app (Application = the container)
                                      ▼
                    ┌─────────────────────────────────────────────┐
                    │  HTTP Kernel  (resolved from the container)  │
                    │                                              │
                    │  handle(Request):                            │
                    │    ├─ run BOOTSTRAPPERS:                      │
                    │    │    • load env (.env)                    │
                    │    │    • load config (config/*.php)         │
                    │    │    • handle exceptions                  │
                    │    │    • register facades                   │
                    │    │    • register() all service providers   │
                    │    │    • boot() all service providers       │
                    │    │                                         │
                    │    └─ send Request through the PIPELINE:     │
                    │         global middleware ──► router         │
                    │            └─ route middleware (groups)      │
                    │                 └─ controller / closure      │
                    └─────────────────────────────────────────────┘
                                      │ returns a Response
                                      ▼  (back out through every middleware, in reverse)
                    ┌─────────────────────────────────────────────┐
                    │  $response->send()  ──► bytes to the browser │
                    └─────────────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────────────┐
                    │  $kernel->terminate(Request, Response)       │
                    │   • terminable middleware (e.g. session save)│
                    │   • deferred work AFTER the user has bytes   │
                    └─────────────────────────────────────────────┘
```

Two ideas to anchor everything:

- The **service container** (`$app`) is created *first* and underpins *every* later stage. Everything is resolved out of it.
- The request goes **into** the pipeline through middleware, hits the route, and the response comes **back out** through the same middleware in reverse — like a sandwich (an "onion model").

---

## Stage 1 — The web server and `public/index.php`

PHP cannot run a URL on its own; a **web server** (Nginx, Apache, or the dev server from `php artisan serve`) accepts the TCP connection and decides what to do. Laravel ships a `public/` directory as the **document root**. Static files (images, compiled CSS) are served directly. Everything else is rewritten to a single PHP file: `public/index.php`. This is the **front controller** pattern — one entry point for the whole app.

A minimal Nginx rule that performs this rewrite:

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

> **Gotcha:** the document root must point at `public/`, not the project root. Pointing it one level up exposes your `.env` and source code to the internet.

`public/index.php` in Laravel 11/12 is intentionally tiny:

```php
<?php
// public/index.php

use Illuminate\Foundation\Application;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true)); // used for timing/debug

// 1) Maintenance mode short-circuit (php artisan down)
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}

// 2) Composer's PSR-4 autoloader: makes `use App\...` work.
require __DIR__.'/../vendor/autoload.php';

// 3) Build the application (the container) and grab the HTTP kernel.
$app = require_once __DIR__.'/../bootstrap/app.php';

$app->handleRequest(Request::capture());
```

Reading top to bottom:

- **Maintenance mode** is checked *before* the framework boots, so `php artisan down` responds fast even if the app is broken.
- **`vendor/autoload.php`** registers Composer's class autoloader. This is what makes `App\Http\Controllers\UserController` load automatically when first referenced.
- **`bootstrap/app.php`** returns the `$app` object — the **container**.
- **`Request::capture()`** builds an `Illuminate\Http\Request` from PHP superglobals (`$_GET`, `$_POST`, `$_SERVER`, …). `handleRequest()` is a Laravel 11+ convenience on the `Application` object that internally resolves the HTTP kernel, calls `handle($request)`, **sends** the response with `$response->send()`, and then calls `$kernel->terminate($request, $response)`. So a single call still covers all the explicit phases of the verbose form below.

> **Laravel 10 difference:** the old `public/index.php` was more verbose. It manually did:
> ```php
> $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
> $response = $kernel->handle($request = Request::capture());
> $response->send();
> $kernel->terminate($request, $response);
> ```
> Laravel 11/12 collapse this into `$app->handleRequest(...)`, but the underlying steps are identical. Knowing the verbose form is interview gold because it shows the four explicit calls — **`capture` → `handle` → `send` → `terminate`** — i.e. build the request, produce a response, flush it to the browser, then run post-response work.

---

## Stage 2 — `bootstrap/app.php` builds the container

The **service container** (also called the IoC container — Inversion of Control) is an object that knows how to construct other objects and their dependencies. The `Illuminate\Foundation\Application` class *is* that container, plus framework-specific extras (paths, environment detection, etc.). Almost everything you "ask Laravel for" is resolved out of it.

In **Laravel 11/12**, `bootstrap/app.php` uses a **fluent builder** to configure the app:

```php
<?php
// bootstrap/app.php (Laravel 11 / 12)

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up', // built-in health-check endpoint
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Configure the middleware pipeline fluently (see Stage 5).
        $middleware->web(append: [
            \App\Http\Middleware\EnsureTokenIsValid::class,
        ]);
        $middleware->alias([
            'subscribed' => \App\Http\Middleware\EnsureUserIsSubscribed::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReport(\App\Exceptions\PodcastException::class);
    })
    ->create();
```

`Application::configure()` returns an `ApplicationBuilder`. Each `->with...()` call registers configuration, and `->create()` finalizes and returns the `Application` instance. Critically, this also **binds the HTTP kernel, console kernel, and exception handler** into the container behind the scenes — that is why Laravel 11/12 no longer need `app/Http/Kernel.php`, `app/Console/Kernel.php`, or `app/Exceptions/Handler.php` files.

> **Laravel 10 / pre-11 difference:** `bootstrap/app.php` was minimal — it just created the `Application`, then bound three kernels by class name:
> ```php
> $app = new Illuminate\Foundation\Application(dirname(__DIR__));
> $app->singleton(Illuminate\Contracts\Http\Kernel::class, App\Http\Kernel::class);
> $app->singleton(Illuminate\Contracts\Console\Kernel::class, App\Console\Kernel::class);
> $app->singleton(Illuminate\Contracts\Debug\ExceptionHandler::class, App\Exceptions\Handler::class);
> return $app;
> ```
> All the middleware lists lived in `app/Http/Kernel.php`. In an interview, mentioning *both* models — "L10 used a `Kernel.php` class; L11+ moved that config into the fluent `bootstrap/app.php`" — signals you've worked across versions.

### Why the container is created first

Because *the kernel itself is resolved from the container*. You cannot run middleware, route, or boot providers without first having something that knows how to build all those objects with their dependencies wired in. The container is the foundation; the rest of the lifecycle is built on top of it.

A tiny illustration of what the container does for you:

```php
class OrderController
{
    // Constructor promotion (PHP 8.0+). The container reads these type hints
    // and AUTOMATICALLY constructs and injects a PaymentGateway. You never
    // call `new` yourself — this is "automatic dependency injection".
    public function __construct(
        private readonly PaymentGateway $gateway,
    ) {}
}
```

When the router dispatches to `OrderController`, the container inspects the constructor, sees it needs a `PaymentGateway`, builds one (recursively resolving *its* dependencies), and hands you a ready-to-use controller. This resolution happens at the dispatch stage, but it only works because the container was built in Stage 2.

---

## Stage 3 — The HTTP kernel and the bootstrappers

The **kernel** is the object that orchestrates handling one request from start to finish. `Illuminate\Foundation\Http\Kernel` implements the `Illuminate\Contracts\Http\Kernel` contract; its `handle($request)` returns a `Response`. The first thing `handle()` does (inside `sendRequestThroughRouter()`, via `$this->bootstrap()`) is run a fixed list of **bootstrappers** — small classes that each prepare one subsystem. They run **in order**, and order matters:

> **Version note:** in Laravel 10 you had an `app/Http/Kernel.php` that *extended* the framework kernel, but even then the `$bootstrappers` array lived on the parent `Illuminate\Foundation\Http\Kernel`. In Laravel 11/12 there is no app-level kernel at all — the framework's `Kernel` is resolved directly from the container (bound by `Application::configure()`), and this same `$bootstrappers` list still applies.

```php
// The default bootstrapper sequence (conceptual — defined on the Kernel).
protected $bootstrappers = [
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class, // 1. parse .env
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,        // 2. merge config/*.php
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,         // 3. error/exception handlers
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,          // 4. set up facade aliases
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,        // 5. register() providers
    \Illuminate\Foundation\Bootstrap\BootProviders::class,            // 6. boot() providers
];
```

Walking the order:

1. **`LoadEnvironmentVariables`** reads `.env` into `$_ENV`/`$_SERVER` so `env()` works. This must run first because config files call `env()`.
2. **`LoadConfiguration`** loads and merges every file under `config/` into one repository accessible via `config('app.timezone')`.
3. **`HandleExceptions`** registers PHP error/exception/shutdown handlers so failures become proper responses instead of white screens.
4. **`RegisterFacades`** wires up facade aliases (e.g. the `Cache` facade → the container's `cache` binding).
5. **`RegisterProviders`** calls `register()` on every service provider.
6. **`BootProviders`** calls `boot()` on every service provider.

> **Critical interview point:** `env()` should **only** be called inside `config/*.php` files. Once config is cached with `php artisan config:cache`, the `.env` file is *not read at all* at runtime, and `env()` returns `null` everywhere except config files (whose values were baked in). This trips up countless developers.

```bash
# In production you cache config for speed. After this, env() outside config = null.
php artisan config:cache

# Clear it when debugging "why is my env var null?"
php artisan config:clear
```

> **HTTP kernel vs console kernel.** Everything in this module follows the **HTTP** path: `Illuminate\Foundation\Http\Kernel`, driven by `public/index.php`. There is a parallel **console** kernel (`Illuminate\Foundation\Console\Kernel`) driven by the `artisan` script for CLI commands and the scheduler. Both run *almost the same* bootstrappers (the console kernel adds command-discovery and uses `LoadEnvironmentVariables` with CLI args), both resolve out of the same container, and both end in `terminate()`. When an interviewer asks "what about artisan commands?", the answer is: same container, same providers, a different kernel and a different entry point — no HTTP request, no middleware pipeline.

---

## Stage 4 — Service providers: `register` then `boot`

A **service provider** is a class that tells Laravel how to wire a feature into the container. It's the central place for bootstrapping. Every provider has up to two methods, and the lifecycle runs them in **two separate passes** across *all* providers:

- **`register()`** — *only* bind things into the container. Do **not** resolve other services here, because other providers may not be registered yet.
- **`boot()`** — runs after *every* provider has registered. Safe to resolve and use other services here (e.g., register routes, view composers, validators, event listeners).

```php
<?php

namespace App\Providers;

use App\Services\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // PHASE 1: bind only. `singleton` = one shared instance per container
        // lifetime. On classic FPM that's effectively one per request (a new
        // app is built each request); under Laravel Octane the container is
        // reused, so a singleton persists across requests — watch for leaked state.
        $this->app->singleton(PaymentGateway::class, function ($app) {
            return new StripeGateway(
                secret: config('services.stripe.secret'), // config is loaded by now
            );
        });
    }

    public function boot(): void
    {
        // PHASE 2: everything is registered, so it's safe to use other services.
        \Illuminate\Support\Facades\Gate::define('refund', fn ($user) => $user->isAdmin());
    }
}
```

**Why two phases?** If you tried to *use* the `PaymentGateway` inside another provider's `register()`, that binding might not exist yet (provider order isn't guaranteed for resolution). By splitting into `register` (declare) then `boot` (use), Laravel guarantees that by the time any `boot()` runs, every binding is already in place. This is a classic ordering problem solved by a two-pass approach — a great "why does it work this way?" answer.

### Deferred providers

Providers can implement `\Illuminate\Contracts\Support\DeferrableProvider` and a `provides()` method. Laravel then *delays* registering them until one of their bindings is actually resolved — a performance optimization so you don't boot, say, the full mail stack on a request that never sends mail.

```php
class FastServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(HeavyService::class, fn () => new HeavyService());
    }

    public function provides(): array
    {
        return [HeavyService::class]; // only loaded when HeavyService is requested
    }
}
```

> **Laravel 11/12 note:** new apps auto-discover package providers and use `bootstrap/providers.php` (an array of *your* app providers) instead of the old `config/app.php` `'providers'` array. Conceptually, the register-then-boot two-pass behavior is unchanged.

---

## Stage 5 — The middleware pipeline

After bootstrapping, the kernel sends the `Request` through a **pipeline** of **middleware**. A middleware is a class with a `handle(Request $request, Closure $next)` method that wraps the rest of the application. It can inspect or modify the request *on the way in*, decide whether to pass control onward (`$next($request)`), and inspect or modify the response *on the way out*.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AddResponseTime
{
    public function handle(Request $request, Closure $next): Response
    {
        // --- BEFORE: runs on the way IN, before the controller ---
        $start = microtime(true);

        $response = $next($request); // hand off to the next layer (eventually the controller)

        // --- AFTER: runs on the way OUT, after the controller produced a response ---
        $response->headers->set('X-Response-Time', (string) (microtime(true) - $start));

        return $response;
    }
}
```

The pipeline is an **onion**: the request travels inward through each "before" block, hits the route/controller at the center, and the response travels back outward through each "after" block **in reverse order**.

```text
Request →  [Global MW 1] [Global MW 2] [Route MW] →  Controller
                                                        │
Response ← [Global MW 1] [Global MW 2] [Route MW] ←  (returns)
   (after-blocks run in REVERSE order on the way back out)
```

### Three kinds of middleware — and where each runs

| Kind | Runs on | Examples | Configure (L11/12) |
|------|---------|----------|--------------------|
| **Global** | *Every* HTTP request | `TrimStrings`, `HandleCors`, `ValidatePostSize` | `$middleware->append(...)` / `prepend(...)` |
| **Group** | All routes in `web` or `api` | `web`: `StartSession`, `ValidateCsrfToken`, `ShareErrorsFromSession`, `SubstituteBindings`; `api`: `SubstituteBindings` (+ optional throttling) | `$middleware->web(append: ...)` / `$middleware->api(...)` |
| **Route (alias)** | Only routes that opt in | `auth`, `throttle`, `verified` | `$middleware->alias([...])` then `->middleware('auth')` on a route |

In **Laravel 11/12** you configure all of this fluently in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    // Global: runs on EVERY request.
    $middleware->append(\App\Http\Middleware\AddResponseTime::class);

    // Group: add to the `web` group only.
    $middleware->web(append: [\App\Http\Middleware\EnsureTokenIsValid::class]);

    // Route alias: opt-in per route.
    $middleware->alias([
        'subscribed' => \App\Http\Middleware\EnsureUserIsSubscribed::class,
    ]);

    // Control ordering when it matters.
    $middleware->priority([
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        // ...
    ]);
})
```

Applying a route alias:

```php
// routes/web.php
use App\Http\Controllers\DashboardController;

Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware(['auth', 'subscribed']); // route-level middleware
```

> **Laravel 10 difference:** the same three lists lived in `app/Http/Kernel.php` as `$middleware` (global), `$middlewareGroups` (`web`/`api`), and `$routeMiddleware`/`$middlewareAliases` (route). The fluent API in L11/12 is just a different surface over the same pipeline.

> **CSRF rename (L11+):** the CSRF middleware class was renamed from `Illuminate\Foundation\Http\Middleware\VerifyCsrfToken` (Laravel ≤10) to `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` (Laravel 11/12). The old name remains as a deprecated subclass alias for backward compatibility, but reference the new name in fresh code. To exclude URIs from CSRF in L11/12 you no longer edit a `$except` array on a kernel-registered class; instead use `$middleware->validateCsrfTokens(except: ['stripe/*'])` inside `->withMiddleware(...)`.

> **Lean default `api` group (L11+):** a fresh Laravel 11/12 app's `api` group contains only `SubstituteBindings` — there is **no** `throttle:api` by default unless you opt in (e.g. `$middleware->api(append: ['throttle:api'])` or installing Sanctum, which also adds `EnsureFrontendRequestsAreStateful`). The `web` group's defaults are `EncryptCookies`, `AddQueuedCookiesToResponse`, `StartSession`, `ShareErrorsFromSession`, `ValidateCsrfToken`, and `SubstituteBindings`.

**Ordering insight:** global middleware run *before* the router even resolves a route, so they cannot know which controller will run. Route/group middleware run *after* the route is matched (they're attached to the route). This is why authentication tied to a route (`auth`) is route middleware, while request-size validation that should apply to literally everything is global.

---

## Stage 6 — Routing and route resolution

Once the request exits the global middleware, control reaches the **router**. The router matches the request's **method + URI** against the routes registered in `routes/web.php` and `routes/api.php` (loaded by `->withRouting(...)`).

```php
// routes/web.php
use App\Http\Controllers\PostController;

Route::get('/posts/{post}', [PostController::class, 'show'])
    ->middleware('auth')
    ->name('posts.show');
```

When a request for `GET /posts/42` arrives:

1. The router finds the matching `Route` object.
2. It runs that route's **group + route middleware** (e.g. the `web` group, then `auth`).
3. **Route-model binding** happens inside the `SubstituteBindings` middleware (which is part of the `web` and `api` groups), *not* in the router's matching core. The `{post}` segment plus the `Post $post` type hint cause Laravel to query the model by its route key — by default `Post::where($post->getRouteKeyName(), 42)->firstOrFail()`, i.e. effectively `findOrFail(42)` since the default route key is the primary key. Override `getRouteKeyName()` (e.g. return `'slug'`) to bind by another column, or use `Route::get('/posts/{post:slug}', ...)` for a per-route key. (Because binding lives in middleware, a route with neither group skips it and you receive the raw string `"42"` instead of a model.)
4. The dispatcher resolves the controller out of the container (injecting constructor dependencies) and calls the method (resolving method parameters too).

```php
class PostController
{
    // {post} in the URI is resolved to a Post model automatically.
    public function show(Post $post): \Illuminate\View\View
    {
        return view('posts.show', ['post' => $post]);
    }
}
// GET /posts/42  → loads Post #42, or returns 404 if it doesn't exist.
```

If **no** route matches, the router throws `NotFoundHttpException`, which the exception handler turns into a **404** response. If the method is wrong (e.g. `POST` to a `GET`-only route), you get a **405 Method Not Allowed**. Both happen at *this* stage — useful to know when "route works in browser, fails from a form."

> **Performance note:** `php artisan route:cache` serializes all routes into a single file so the router skips re-registering closures. Cached routes require that no route uses a `Closure` handler (only `[Controller::class, 'method']` array syntax), because closures can't be serialized.

---

## Stage 7 — Controller/closure dispatch and the response

The matched route points to either a **closure** or a **controller method**. Whatever it returns is converted into a **`Response`** object:

```php
// 1) A string → wrapped in a 200 HTML response.
Route::get('/hello', fn () => 'Hi!');

// 2) An array or Eloquent model → automatically JSON-encoded.
Route::get('/api/user', fn () => ['name' => 'Ada', 'role' => 'admin']);
// Output: {"name":"Ada","role":"admin"}  with Content-Type: application/json

// 3) A view → rendered HTML response.
Route::get('/home', fn () => view('home'));

// 4) An explicit response with status + headers.
Route::get('/teapot', fn () => response('No coffee', 418)
    ->header('X-Brew', 'tea'));
```

The framework normalizes all of these into a `Symfony\Component\HttpFoundation\Response` (the class `Illuminate\Http\Response` extends). The response then travels **back out** through the middleware pipeline — every middleware's "after" block gets a chance to modify it (add headers, compress, etc.), in reverse order.

Finally, back in `index.php`, the response is **sent**:

```php
$response->send();
// Internally: sends the HTTP status line, sends all headers,
// then echoes the body to the output buffer → bytes go to the browser.
```

At this moment the user has the full response. The PHP process, however, is **not done yet**.

---

## Stage 8 — Termination and terminable middleware

After `send()`, the kernel calls `terminate(Request, Response)`. This runs any middleware that implements a `terminate()` method — **terminable middleware**. The key property: this work happens **after the response has already been sent to the browser** (when the server setup supports it, e.g. PHP-FPM with `fastcgi_finish_request`). So you can do "after the user is happy" work without making them wait.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class StoreRequestMetrics
{
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    // Runs AFTER the response is sent. The user already has their bytes.
    public function terminate(Request $request, Response $response): void
    {
        Metrics::record($request->path(), $response->getStatusCode());
    }
}
```

The framework's own `StartSession` middleware uses this to **persist the session to storage** after the response is sent — that's why session writes don't block the user. After all `terminate()` methods run, the application calls registered "terminating" callbacks and the PHP process ends.

> **Important caveat:** terminable middleware only run *after the response is flushed* when using PHP-FPM (which provides `fastcgi_finish_request`). With the built-in `php artisan serve` or some setups, the work still runs, but the response may not be flushed early. For genuinely long tasks, use **queued jobs**, not terminable middleware.

---

## How the container underpins every stage (the through-line)

Notice the container kept reappearing:

- **Stage 2** built it.
- **Stage 3** resolved the **kernel** from it and ran bootstrappers that populate it (config repository, facades).
- **Stage 4** filled it with bindings via providers.
- **Stage 5/6/7** resolved **middleware** and **controllers** (with auto-injected dependencies) from it.

So the lifecycle is really: *build the container → fill the container → resolve everything from the container → tear down.* If you remember nothing else, remember that the container is the spine the whole lifecycle hangs from.

---

## ⚠️ Common Mistakes & Gotchas

**1. Calling `env()` outside config files.**
`env()` returns `null` whenever `php artisan config:cache` has run (production), because `.env` is not read at runtime after caching.
**Fix:** read environment values *only* inside `config/*.php`, then use `config('services.stripe.secret')` everywhere in your app.

**2. Resolving services inside a provider's `register()` method.**
Other providers may not have registered their bindings yet, so you get "binding not found" or stale objects.
**Fix:** only **bind** in `register()`; do all **using/resolving** (routes, gates, listeners) in `boot()`.

**3. Confusing global middleware with route middleware ordering.**
Trying to read the matched route or the authenticated user inside *global* middleware fails — global middleware runs **before** routing, so there is no route yet (and `auth` hasn't run).
**Fix:** move logic that depends on the route or auth state into **route/group middleware**, which run after route resolution.

**4. Pointing the web server document root at the project root.**
This exposes `.env`, `composer.json`, and source. It can also break asset URLs.
**Fix:** set the document root to the `public/` directory; only `public/` should be web-accessible.

**5. Expecting terminable middleware to run for long-running tasks reliably.**
On `php artisan serve` or non-FPM setups, the "after the user gets bytes" guarantee may not hold, and a slow `terminate()` still delays process completion.
**Fix:** use **queued jobs** for anything slow; reserve terminable middleware for fast cleanup like writing the session or recording a metric.

**6. Caching routes that use closures.**
`php artisan route:cache` throws `LogicException: Unable to prepare route ... for serialization` if any route uses a closure handler.
**Fix:** convert closure routes to `[Controller::class, 'method']` array syntax before caching, or invokable controllers.

---

## ✅ Best Practices

- **Read `.env` only in `config/*.php`.** Everywhere else use `config()`. This keeps `config:cache` safe.
- **Keep `register()` pure** (bindings only); put side-effects in `boot()`.
- **Pick the right middleware scope.** Global for things that truly apply to every request (CORS, trim strings); group for `web`/`api` concerns; route alias for opt-in (`auth`, `throttle`).
- **Cache config and routes in production** (`php artisan config:cache`, `route:cache`, `event:cache`, and `optimize`) — and clear them when debugging surprising `null`s.
- **Use the container, don't fight it.** Type-hint dependencies in constructors and methods; let auto-resolution wire them. Avoid `app()->make()` sprinkled through business logic.
- **Use `DeferrableProvider`** for heavy services that aren't needed on every request.
- **Use queued jobs** (not terminable middleware) for slow post-response work.
- **Know your version.** State whether you're on the L11/12 fluent `bootstrap/app.php` model or the L10 `Kernel.php` model when discussing config.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Walk me through what happens from the moment a request hits the server to the response being sent.**
A: Web server rewrites the URL to `public/index.php` → it loads the Composer autoloader and `bootstrap/app.php`, which builds the `Application` (the container) → the HTTP kernel is resolved and `handle()` runs the **bootstrappers** (env, config, exceptions, facades, register providers, boot providers) → the request goes through the **middleware pipeline** (global, then group/route after routing) → the **router** matches a route and dispatches to a controller/closure → the return value becomes a `Response` that travels back out through middleware → `$response->send()` flushes bytes → `$kernel->terminate()` runs **terminable middleware** after the user has the response.

**Q2. (Under the hood) How does Laravel resolve a controller and its dependencies?**
A: The router resolves the controller class out of the **service container**. The container uses PHP **reflection** to read the constructor's type-hinted parameters, recursively builds each dependency (using any registered bindings/singletons, or auto-resolving concrete classes), and injects them. Method parameters are resolved the same way (method injection), and route-model binding fills typed model parameters from URI segments.

**Q3. What's the difference between the `register` and `boot` methods of a service provider, and why are there two?**
A: `register` only binds things into the container; `boot` runs after *all* providers have registered, so it can safely use other services (define routes, gates, view composers). The two-pass design guarantees that by the time any `boot()` runs, every binding exists — avoiding ordering bugs.

**Q4. Where do global vs route middleware run, and why does it matter?**
A: Global middleware run on every request **before** the router resolves a route, so they can't know the route or the authenticated user. Route/group middleware are attached to the matched route and run **after** routing. That's why `auth` is route middleware while `TrimStrings`/`HandleCors` are global.

**Q5. How did the lifecycle change from Laravel 10 to Laravel 11/12?**
A: L10 had `app/Http/Kernel.php`, `app/Console/Kernel.php`, and `app/Exceptions/Handler.php`, and registered them in a minimal `bootstrap/app.php`. L11/12 removed those files; configuration moved into a fluent `bootstrap/app.php` via `Application::configure()->withRouting()->withMiddleware()->withExceptions()->create()`. The actual stages — bootstrap, providers, middleware, routing, dispatch, terminate — are unchanged.

**Q6. What is terminable middleware and when would you use it?**
A: Middleware with a `terminate(Request, Response)` method, called after the response is sent (on FPM, after `fastcgi_finish_request` flushes bytes). Use it for fast post-response cleanup — Laravel's session middleware persists the session this way. Use queues for anything slow.

**Q7. Why does `env('APP_NAME')` return `null` in production but work locally?**
A: In production you've likely run `php artisan config:cache`, after which `.env` isn't read at runtime; `env()` returns `null` outside config files. Use `config('app.name')` instead, and only call `env()` inside `config/*.php`.

**Q8. What is the service container and why is it created first?**
A: It's the IoC container (`Illuminate\Foundation\Application`) that constructs objects and resolves dependencies. It's created first because the kernel, providers, middleware, and controllers are all *resolved from it* — it's the foundation every later stage depends on.

**Q9. What happens if no route matches?**
A: The router throws `NotFoundHttpException`; the exception handler (registered by the `HandleExceptions` bootstrapper / `withExceptions`) converts it to a 404 response. A wrong HTTP method yields a 405.

**Q10. Where are the best extension points in the lifecycle?**
A: Service providers (to bind/replace services), middleware (to inspect/modify request and response), the router (routes, route-model binding), and `withExceptions` (custom error rendering/reporting). Choosing the right one comes from knowing the order things run.

---

## 📋 Quick Reference / Cheat Sheet

```text
ENTRY        public/index.php  → autoload → bootstrap/app.php → $app (container)
DISPATCH     $app->handleRequest(Request::capture())   [L11/12]
             (= kernel->handle() → $response->send() → kernel->terminate() under the hood)

BOOTSTRAPPERS (run in order, inside kernel->handle):
  1 LoadEnvironmentVariables   .env  → env()
  2 LoadConfiguration          config/*.php → config()
  3 HandleExceptions           error/exception handlers
  4 RegisterFacades            facade aliases
  5 RegisterProviders          provider register()  ← BIND ONLY
  6 BootProviders              provider boot()       ← SAFE TO USE SERVICES

PIPELINE (onion / sandwich):
  Request → [global mw] → router → [group mw] → [route mw] → controller
  Response ← (same middleware, "after" blocks, REVERSE order) ←

MIDDLEWARE SCOPES:
  global  every request, before routing     append()/prepend()
  group   web / api                          ->web(append: ...) / ->api(...)
  route   opt-in per route                   ->alias([...]) + ->middleware('auth')

RESPONSE     return value → Response → $response->send() → bytes to browser
TERMINATE    $kernel->terminate() → terminable middleware (e.g. save session)
```

```bash
# Performance caches (production)               # Clear when debugging
php artisan config:cache                        php artisan config:clear
php artisan route:cache                         php artisan route:clear
php artisan event:cache                         php artisan event:clear
php artisan optimize                            php artisan optimize:clear

php artisan about            # shows env, cached states, providers, drivers
php artisan route:list       # every registered route + its middleware
```

| File (L11/12) | Responsibility |
|---------------|----------------|
| `public/index.php` | Front controller / entry point |
| `bootstrap/app.php` | Builds container, configures routing/middleware/exceptions |
| `bootstrap/providers.php` | Your app's service providers |
| `routes/web.php`, `routes/api.php` | Route definitions |
| `app/Http/Middleware/*` | Your middleware classes |
| `config/*.php` | The only place `env()` should be called |

---

## 🧪 Mini Exercises

1. **Trace it yourself.** Open `public/index.php` and `bootstrap/app.php` in a fresh Laravel 12 app. Write down, in order, the exact lines responsible for: loading the autoloader, building the container, and dispatching the request. Then find where the HTTP kernel is bound (hint: it's inside the framework's `ApplicationBuilder`, triggered by `Application::configure()`).

2. **Prove the pipeline order.** Create two global middleware, `First` and `Second`, each logging `"<name> before"` before `$next()` and `"<name> after"` after it. Register both globally, hit any route, and predict the log order *before* you look. Confirm the "after" lines appear in reverse.

3. **Break and fix `env()`.** Add a custom value to `.env`, reference it via `env()` directly inside a controller, and confirm it works. Then run `php artisan config:cache` and reload — observe it become `null`. Fix it by moving the value into a `config/*.php` file and using `config()`.

4. **Write terminable middleware.** Build a middleware that records the request path and response status *after* the response is sent, using a `terminate()` method. Register it and verify (via a log) that it runs after the controller returns.

5. **Register-vs-boot ordering.** Create two providers, `A` and `B`. In `A::register()`, attempt to resolve a binding that `B` defines, and observe the failure. Then move the resolution into `A::boot()` and confirm it now succeeds. Explain why in one sentence.
