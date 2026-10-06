# Middleware in Laravel

Middleware is one of the most important concepts in any HTTP framework. If you understand middleware deeply, you understand *how an HTTP request actually flows through your application* — and that knowledge unlocks authentication, authorization, rate limiting, logging, localization, CORS, and dozens of cross-cutting concerns. This module takes you from the absolute basics to the internals and the interview-grade details.

> Target stack: **PHP 8.4** and **Laravel 12**. Where Laravel 10/11 or PHP 8.1–8.3 behave differently, it's called out explicitly.

---

## **What you'll learn**

- What middleware *is* and why it exists (filtering and decorating HTTP requests/responses).
- How to generate middleware with `php artisan make:middleware` and what the `handle($request, $next)` signature means.
- The difference between **before** middleware and **after** middleware.
- How to register middleware three ways: **global**, **middleware groups** (`web`/`api`), and **aliased route middleware** — using the Laravel 11+ `bootstrap/app.php` `withMiddleware()` approach *and* the older `app/Http/Kernel.php` approach.
- How to pass **parameters** to middleware (e.g. a role: `role:admin`).
- **Terminable** middleware (the `terminate()` method) and **middleware priority**.
- Built-in middleware you'll use constantly: `auth`, `throttle`, `verified`, `signed`, plus a note on CORS.
- Practical, runnable examples: auth checks, request logging, locale switching.

---

## 1. Why does middleware exist? (the WHY)

Every web request to a Laravel app needs the *same* repetitive checks before your controller does its real work:

- "Is this user logged in?"
- "Has this user verified their email?"
- "Are they sending too many requests (rate limiting)?"
- "What language should the response be in?"
- "Is the CSRF token valid?"

If you put all of that logic *inside every controller method*, you'd duplicate it hundreds of times and forget it somewhere eventually. **Middleware** is the framework's answer: a series of layers that wrap your application. Each layer can inspect or modify the request on the way *in*, and inspect or modify the response on the way *out*.

The classic mental model is an **onion** (also called the "pipeline" or "decorator" pattern). The request travels inward through each middleware layer until it reaches your route/controller (the core), then the response travels back outward through the same layers in reverse:

```
  Request ─►  [ Middleware A ] ─► [ Middleware B ] ─► [ Route/Controller ]
                                                              │
  Response ◄─ [ Middleware A ] ◄─ [ Middleware B ] ◄──────────┘
```

Because each layer wraps the next, a middleware can run code **before** the request reaches the controller, **after** the response comes back, or **both**. It can also **short-circuit** the whole thing — for example, `auth` middleware redirects an unauthenticated user to the login page and the controller is never called.

> **Jargon check.** A *cross-cutting concern* is logic that applies across many parts of your app (auth, logging, etc.) rather than to one specific feature. Middleware is Laravel's primary tool for cross-cutting concerns at the HTTP layer.

---

## 2. Creating middleware with `make:middleware`

Use the Artisan generator:

```bash
php artisan make:middleware EnsureTokenIsValid
```

Output:

```
   INFO  Middleware [app/Http/Middleware/EnsureTokenIsValid.php] created successfully.
```

This creates `app/Http/Middleware/EnsureTokenIsValid.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }
}
```

A freshly generated middleware does *nothing* useful yet — it simply passes the request along by calling `$next($request)`. Your job is to add logic around that call.

---

## 3. The `handle(Request $request, Closure $next)` signature

This is the heart of middleware. Let's dissect it.

```php
public function handle(Request $request, Closure $next): Response
{
    return $next($request);
}
```

- **`Request $request`** — the incoming HTTP request (an `Illuminate\Http\Request` instance). You can read headers, query params, the body, the authenticated user, etc.
- **`Closure $next`** — a closure (anonymous function) that represents *"the rest of the pipeline"*: all the remaining middleware plus your route/controller. Calling `$next($request)` hands control to the next layer.
- **Return type `Response`** — middleware must return an HTTP response. Usually you return whatever `$next($request)` gives you (the response produced by the controller), but you can return your *own* response to short-circuit.

The key insight: **`$next($request)` is the line that "goes deeper" into the onion.** Anything before it runs on the way *in*; anything after it runs on the way *out*.

> **PHP 8.4 note.** The signature uses constructor-injectable type hints and a return type. PHP 8.1–8.4 all support this identically; nothing version-specific here. Where PHP version matters is inside your logic (e.g. enums, `match`, readonly properties), shown later.

---

## 4. Before vs After middleware

### Before middleware

Runs its logic **before** the controller. Use it to *guard* or *mutate the incoming request*.

```php
public function handle(Request $request, Closure $next): Response
{
    // --- BEFORE: runs on the way in ---
    if (! $request->hasValidSignature()) {
        abort(403, 'Invalid signature.');
    }

    return $next($request); // continue into the app
}
```

### After middleware

Runs its logic **after** the controller, by capturing the response first and modifying it before returning.

```php
public function handle(Request $request, Closure $next): Response
{
    $response = $next($request); // go in, get the response back

    // --- AFTER: runs on the way out ---
    $response->headers->set('X-Frame-Options', 'DENY');

    return $response;
}
```

### Both (a single middleware can do before *and* after)

```php
public function handle(Request $request, Closure $next): Response
{
    $start = microtime(true);          // BEFORE

    $response = $next($request);       // descend into the app

    $ms = round((microtime(true) - $start) * 1000, 2); // AFTER
    $response->headers->set('X-Response-Time', "{$ms}ms");

    return $response;
}
```

If a request hits this middleware and the controller returns a `200 OK`, the response will include a header like:

```
X-Response-Time: 12.84ms
```

> **Short-circuiting.** A before middleware that calls `abort()`, `redirect()`, or returns a `Response` *without* calling `$next($request)` stops the pipeline. The controller — and all inner middleware — never run.

---

## 5. Registering middleware (the big one)

There are **three scopes** where middleware can run, and **two registration styles** depending on your Laravel version.

| Scope | When it runs | Example |
|---|---|---|
| **Global** | On *every* HTTP request | Force HTTPS, log all requests |
| **Group** | On every request in a group (`web` or `api`) | Session, CSRF (web); rate limit (api) |
| **Aliased / route** | Only on routes you explicitly attach it to | `auth`, `verified`, custom role check |

### 5a. The Laravel 11+ way: `bootstrap/app.php` + `withMiddleware()`

**Laravel 11 removed `app/Http/Kernel.php`.** All middleware configuration now lives in `bootstrap/app.php` via the fluent `withMiddleware()` callback. This is the approach for **Laravel 11 and 12**.

```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use App\Http\Middleware\EnsureTokenIsValid;
use App\Http\Middleware\EnsureUserHasRole;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // 1) GLOBAL — runs on every request
        $middleware->append(EnsureTokenIsValid::class);

        // 2) APPEND/PREPEND to a GROUP
        $middleware->web(append: [
            \App\Http\Middleware\SetLocale::class,
        ]);
        $middleware->api(prepend: [
            \App\Http\Middleware\ForceJsonResponse::class,
        ]);

        // 3) ALIAS — a short name you attach to routes
        $middleware->alias([
            'role' => EnsureUserHasRole::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

Key methods inside the `Middleware $middleware` callback:

| Method | Purpose |
|---|---|
| `$middleware->append(Class::class)` | Add to the **global** stack (runs last among globals) |
| `$middleware->prepend(Class::class)` | Add to the **global** stack (runs first) |
| `$middleware->web(append: [...])` | Add to the **web** group (also supports `prepend:`, `replace:`, `remove:`) |
| `$middleware->api(prepend: [...])` | Add to the **api** group (also supports `append:`, `replace:`, `remove:`) |
| `$middleware->appendToGroup('name', [...])` | Add to **any** group (the general form; `web()`/`api()` are convenience wrappers) |
| `$middleware->prependToGroup('name', [...])` | Add to the front of **any** group |
| `$middleware->alias(['name' => Class::class])` | Register a route alias (formerly `$routeMiddleware`) |
| `$middleware->group('name', [...])` | Define (or fully redefine) a **named** middleware group |
| `$middleware->priority([...])` | Set explicit ordering (see §8) |
| `$middleware->use([...])` | Redefine the **entire global** stack manually |
| `$middleware->remove(Class::class)` | Remove a middleware from the global stack |
| `$middleware->replace(Old::class, New::class)` | Swap a global middleware for your own |

> **`web()`/`api()` vs `appendToGroup()`.** `$middleware->web(append: [...])` is just a convenience alias for `$middleware->appendToGroup('web', [...])`. Use the named `web()`/`api()` helpers for the two default groups; use `appendToGroup()`/`prependToGroup()` for your own custom groups. You can also swap or drop a single default entry per group, e.g. `$middleware->web(remove: [\Illuminate\Session\Middleware\StartSession::class])` or `$middleware->web(replace: [StartSession::class => StartCustomSession::class])`.

> **Important Laravel 11/12 detail.** In Laravel 11+, the framework's default middleware (e.g. `EncryptCookies`, `VerifyCsrfToken`, `StartSession`) are *no longer listed in your app's code*. They live inside the framework and are applied automatically. You only interact with them when you want to customize — e.g. `$middleware->validateCsrfTokens(except: ['stripe/*'])` or `$middleware->encryptCookies(except: ['theme'])`.

### 5b. The older way: `app/Http/Kernel.php` (Laravel 10 and earlier)

If you're on **Laravel 10 or earlier** (or maintaining a legacy app), middleware is configured in `app/Http/Kernel.php`:

```php
<?php
// app/Http/Kernel.php  (Laravel 10 and earlier)

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    // GLOBAL middleware
    protected $middleware = [
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\EnsureTokenIsValid::class, // your custom global
    ];

    // GROUP middleware
    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \App\Http\Middleware\SetLocale::class, // your custom in the web group
        ],
        'api' => [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    // ROUTE middleware (renamed $routeMiddleware → $middlewareAliases in L9.19+)
    protected $middlewareAliases = [
        'auth'     => \App\Http\Middleware\Authenticate::class,
        'role'     => \App\Http\Middleware\EnsureUserHasRole::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
    ];

    // Priority
    protected $middlewarePriority = [ /* ... */ ];
}
```

> **Migration mental map.** `$middleware` → `append()`/`prepend()`. `$middlewareGroups` → `web()`/`api()`/`group()`. `$middlewareAliases` (formerly `$routeMiddleware`) → `alias()`. `$middlewarePriority` → `priority()`.

---

## 6. Assigning middleware to routes and groups

Once a middleware has an **alias** (e.g. `'role'`) or you reference it by class, attach it to routes.

### By alias

```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\DashboardController;

Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware('auth');
```

### Multiple middleware

```php
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware(['auth', 'verified', 'role:admin']);
```

### By class name (no alias needed)

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/secure', SecureController::class)
    ->middleware(EnsureTokenIsValid::class);
```

### On a route group

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/profile', [ProfileController::class, 'show']);
    Route::put('/profile', [ProfileController::class, 'update']);
    Route::resource('orders', OrderController::class);
});
```

### Applying a named group to routes

```php
Route::middleware('web')->group(base_path('routes/web.php'));

// Or a custom group you defined with $middleware->group('admin', [...])
Route::middleware('admin')->prefix('admin')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
});
```

### Excluding middleware from specific routes

```php
Route::middleware(['auth'])->group(function () {
    Route::get('/account', [AccountController::class, 'show']);

    Route::get('/public-status', [StatusController::class, 'show'])
        ->withoutMiddleware(['auth']); // skip auth just for this one
});
```

> `withoutMiddleware()` can only remove **route-level** middleware, not global middleware.

---

## 7. Middleware parameters (e.g. a role)

Middleware can accept extra arguments after `$next`. The classic example is a role check. You pass parameters in the route using `alias:value` syntax, comma-separated for multiple values.

Generate it:

```bash
php artisan make:middleware EnsureUserHasRole
```

Implement it (note the modern PHP idioms — variadic params, `match` could be used, etc.):

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    /**
     * Usage in routes: ->middleware('role:admin')
     *                  ->middleware('role:admin,editor')
     *
     * @param  string  ...$roles  one or more allowed roles
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (! $request->user() || ! in_array($request->user()->role, $roles, true)) {
            abort(403, 'You do not have the required role.');
        }

        return $next($request);
    }
}
```

Register the alias (Laravel 11/12 `bootstrap/app.php`):

```php
$middleware->alias([
    'role' => \App\Http\Middleware\EnsureUserHasRole::class,
]);
```

Use it on routes:

```php
// Single role
Route::get('/admin', AdminController::class)->middleware('role:admin');

// Multiple allowed roles — both become elements of $roles
Route::get('/content', ContentController::class)->middleware('role:admin,editor');
```

For `role:admin,editor`, the `handle` method receives `$roles = ['admin', 'editor']` thanks to PHP's variadic `...$roles`. A user whose `role` is not in that list gets:

```
403 Forbidden — You do not have the required role.
```

> **Tip:** parameters are always passed as **strings**. `role:1,2` gives you `['1', '2']` (strings), not integers. Cast if you need numbers.

### Using an enum for roles (PHP 8.1+ idiom)

Backed enums make role logic safer and self-documenting:

```php
<?php

namespace App\Enums;

enum Role: string
{
    case Admin  = 'admin';
    case Editor = 'editor';
    case Viewer = 'viewer';
}
```

```php
public function handle(Request $request, Closure $next, string ...$roles): Response
{
    // Guard FIRST: an unauthenticated request has no role at all.
    if (! $request->user()) {
        abort(401);
    }

    $userRole = Role::tryFrom($request->user()->role); // null if column holds an unknown value

    // Use tryFrom() — never from() — on untrusted/route-supplied strings.
    // Role::from('typo') throws \ValueError, which surfaces as a 500, not a clean 403.
    $allowed = array_filter(array_map(Role::tryFrom(...), $roles));

    if ($userRole === null || ! in_array($userRole, $allowed, true)) {
        abort(403);
    }

    return $next($request);
}
```

> **Why `tryFrom()` and not `from()`?** `Role::from($value)` throws a `\ValueError` when `$value` is not a valid case. Because the route parameters (`role:admin,...`) and the user's stored role are effectively untrusted input, a typo in a route definition or a stale DB value would otherwise blow up the request with a 500 instead of a controlled 403/401. `Role::tryFrom()` returns `null` instead, which you handle explicitly. (`Role::tryFrom(...)` here uses PHP 8.1+ first-class callable syntax.)

---

## 8. Middleware priority (ordering)

Group middleware run in the order they're listed — **except** when the framework forces a specific order via the **priority list**. Some middleware *must* run before others to function correctly. For example, `StartSession` must run before `Authenticate`, because authentication reads the session.

By default Laravel defines a sensible priority. You override it in `bootstrap/app.php`. The snippet below is the **actual Laravel 12 default priority list** (copy it verbatim and reorder/trim as needed):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->priority([
        \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
        \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        \Illuminate\Routing\Middleware\ThrottleRequests::class,
        \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
        \Illuminate\Auth\Middleware\Authorize::class,
    ]);
})
```

> **Accuracy note.** The authentication entry in the real list is the **interface** `Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests` (which `Illuminate\Auth\Middleware\Authenticate` implements), not the concrete class — Laravel resolves any middleware implementing it to this slot. Likewise, CSRF protection in Laravel 11/12 is `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` (the old `VerifyCsrfToken` name was used in Laravel 10 and earlier).

Any middleware in this list is sorted to its listed position regardless of where it was attached. Middleware *not* in the list keeps its natural attachment order. You rarely need to touch priority unless you write middleware that depends on session/auth/binding ordering.

> **Interview-grade nuance:** priority only re-orders middleware that appear in the priority array. It does not affect the relative order of middleware absent from the list.

---

## 9. Terminable middleware (`terminate()`)

Sometimes you want work to happen **after the HTTP response has already been sent to the browser** — e.g. writing logs, flushing analytics. That's what **terminable middleware** is for. Add a `terminate()` method:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class LogRequestDuration
{
    public function handle(Request $request, Closure $next): Response
    {
        $request->attributes->set('start', microtime(true));

        return $next($request);
    }

    /**
     * Called AFTER the response is sent to the browser.
     */
    public function terminate(Request $request, Response $response): void
    {
        $ms = round((microtime(true) - $request->attributes->get('start')) * 1000, 2);

        Log::info('Request handled', [
            'uri'    => $request->getRequestUri(),
            'status' => $response->getStatusCode(),
            'ms'     => $ms,
        ]);
    }
}
```

Important details:

- `terminate()` receives **both** the request *and* the final response.
- It runs only when the server supports it. With **PHP-FPM** (FastCGI), Laravel calls `fastcgi_finish_request()` so the response is flushed to the client first, then `terminate()` runs. The built-in `php artisan serve` uses PHP's own development web server, which is **not** FastCGI and has no `fastcgi_finish_request()` — so the response is *not* truly sent before `terminate()` runs there.
- By default, Laravel resolves a **fresh instance** of the middleware from the service container for the `terminate()` call — it is **not** the same object as the one that ran `handle()`. To guarantee the *same* instance (so you can keep state on `$this`), register it as a singleton in a service provider:

```php
// app/Providers/AppServiceProvider.php → register()
$this->app->singleton(\App\Http\Middleware\LogRequestDuration::class);
```

That's why the example above stores state on `$request->attributes` rather than on `$this` — `$request` is the same object across both calls.

> **Octane caveat.** Under Laravel Octane (long-running workers), be careful with state on `$this`; the instance can be reused across requests. Prefer request-scoped storage.

---

## 10. Built-in middleware you'll use constantly

### `auth` — require an authenticated user

```php
Route::get('/dashboard', DashboardController::class)->middleware('auth');

// Specify a guard:
Route::get('/admin', AdminController::class)->middleware('auth:admin');
```

If the user isn't authenticated: web requests **redirect** to the `login` route; API/JSON requests get a **401 Unauthorized** JSON response. The branch is decided by whether the request `expectsJson()`.

### `throttle` — rate limiting

```php
// 60 requests per minute (the named 'api' limiter, defined in code)
Route::middleware('throttle:api')->group(/* ... */);

// Inline: 5 attempts per 1 minute
Route::post('/login', [LoginController::class, 'store'])
    ->middleware('throttle:5,1');
```

Named limiters are defined (Laravel 11/12) in the `boot()` method of `app/Providers/AppServiceProvider.php` or a dedicated provider:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

> **Laravel 11/12 nuance.** A brand-new Laravel app does **not** ship `routes/api.php` *or* a pre-defined `api` rate limiter — you opt in by running `php artisan install:api`. That command creates `routes/api.php`, registers the `api` route file in `bootstrap/app.php`, installs Sanctum, and scaffolds the `api` limiter (the 60/min definition above) in a provider. So if `throttle:api` errors with "Rate limiter [api] is not defined," you either haven't run `install:api` or haven't defined the limiter yourself.

Exceeding the limit returns:

```
429 Too Many Requests
Retry-After: 37
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
```

### `verified` — require a verified email

```php
Route::get('/billing', BillingController::class)
    ->middleware(['auth', 'verified']);
```

Unverified users are redirected to the `verification.notice` route. Your `User` model must implement `Illuminate\Contracts\Auth\MustVerifyEmail`.

### `signed` — verify a signed URL

Signed URLs carry a tamper-proof signature in the query string (great for unsubscribe links, email confirmations).

```php
use Illuminate\Support\Facades\URL;

// Generate a temporary signed URL valid for 30 minutes
$url = URL::temporarySignedRoute('unsubscribe', now()->addMinutes(30), ['user' => 1]);

// Protect the route
Route::get('/unsubscribe/{user}', UnsubscribeController::class)
    ->name('unsubscribe')
    ->middleware('signed');
```

A tampered or expired URL yields `403 Invalid signature.`

### A note on **CORS**

CORS (Cross-Origin Resource Sharing) controls which external origins (domains) may call your API from a browser. Laravel ships `Illuminate\Http\Middleware\HandleCors` as a **global** middleware, configured via `config/cors.php`:

```php
// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    'allowed_methods' => ['*'],
    'allowed_origins' => ['https://app.example.com'],
    'allowed_headers' => ['*'],
    'supports_credentials' => true,
];
```

You usually **don't** write CORS middleware yourself — configure the file. To publish it if it's missing:

```bash
php artisan config:publish cors
```

> **Security warning.** Never combine `'allowed_origins' => ['*']` with `'supports_credentials' => true`. The CORS spec forbids a wildcard origin when credentials (cookies, `Authorization` headers) are allowed, and browsers will reject the response — but more importantly, a permissive wildcard exposes authenticated endpoints to any site. List explicit origins (`['https://app.example.com']`) whenever you allow credentials. Treat `'*'` as a public, unauthenticated-only setting. `HandleCors` is registered as a global middleware automatically in Laravel 11/12, so you do not append it yourself.

### Default middleware aliases you get for free

You don't have to register aliases for the framework's own middleware — Laravel 11/12 ships these out of the box, so you can use them on any route immediately:

| Alias | Class | Purpose |
|---|---|---|
| `auth` | `Illuminate\Auth\Middleware\Authenticate` | Require an authenticated user |
| `auth.basic` | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth` | HTTP Basic auth |
| `auth.session` | `Illuminate\Session\Middleware\AuthenticateSession` | Protect the session: log the user out of *other* devices when their password changes |
| `cache.headers` | `Illuminate\Http\Middleware\SetCacheHeaders` | Set `Cache-Control` headers (e.g. `cache.headers:public;max_age=3600`) |
| `can` | `Illuminate\Auth\Middleware\Authorize` | Run a Gate/Policy authorization check (`can:update,post`) |
| `guest` | `Illuminate\Auth\Middleware\RedirectIfAuthenticated` | Redirect *away* if already logged in |
| `password.confirm` | `Illuminate\Auth\Middleware\RequirePassword` | Require recent password confirmation |
| `precognitive` | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests` | Laravel Precognition support |
| `signed` | `Illuminate\Routing\Middleware\ValidateSignature` | Verify a signed URL |
| `throttle` | `Illuminate\Routing\Middleware\ThrottleRequests` | Rate limiting |
| `verified` | `Illuminate\Auth\Middleware\EnsureEmailIsVerified` | Require a verified email |

```php
// Authorization via the `can` alias (delegates to a Gate/Policy):
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('can:update,post');
```

---

## 11. Practical example A: an auth-check middleware (custom)

For learning, here's a hand-rolled "must be logged in" middleware that returns JSON for API clients and redirects browsers:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Symfony\Component\HttpFoundation\Response;

class EnsureAuthenticated
{
    public function handle(Request $request, Closure $next): Response
    {
        if (Auth::guest()) {
            return $request->expectsJson()
                ? response()->json(['message' => 'Unauthenticated.'], 401)
                : redirect()->guest(route('login'));
        }

        return $next($request);
    }
}
```

In real apps prefer the framework's `auth` middleware; this is illustrative.

## 12. Practical example B: request logging middleware

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class LogIncomingRequests
{
    public function handle(Request $request, Closure $next): Response
    {
        Log::channel('requests')->info('Incoming', [
            'method' => $request->method(),
            'uri'    => $request->path(),
            'ip'     => $request->ip(),
            'user'   => $request->user()?->id,
        ]);

        return $next($request);
    }
}
```

A request to `GET /orders/42` writes a line to `storage/logs/requests.log` similar to:

```
[2026-06-18 09:14:02] local.INFO: Incoming {"method":"GET","uri":"orders/42","ip":"127.0.0.1","user":7}
```

## 13. Practical example C: setting the locale

A very common real-world middleware: read the user's preferred language and set the app locale.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\App;
use Symfony\Component\HttpFoundation\Response;

class SetLocale
{
    private const SUPPORTED = ['en', 'fr', 'de', 'ur'];
    private const FALLBACK  = 'en';

    public function handle(Request $request, Closure $next): Response
    {
        // Priority: authenticated user's saved pref → ?lang query → Accept-Language header
        $locale = $request->user()?->locale
            ?? $request->query('lang')
            ?? $request->getPreferredLanguage(self::SUPPORTED);

        $locale = in_array($locale, self::SUPPORTED, true) ? $locale : self::FALLBACK;

        App::setLocale($locale);

        return $next($request);
    }
}
```

Register it in the `web` group so it applies to browser traffic:

```php
$middleware->web(append: [
    \App\Http\Middleware\SetLocale::class,
]);
```

Now `__('messages.welcome')` and `@lang(...)` will resolve translations in the chosen language. Visiting `/?lang=fr` makes `App::getLocale()` return `'fr'` for that request.

---

## 14. How middleware works under the hood (the internals)

When the kernel handles a request, it builds an `Illuminate\Pipeline\Pipeline`. The pipeline takes the request, an array of middleware, and your route handler as the "destination". It composes them using `array_reduce` over the middleware list, wrapping each one around the next as a closure. Conceptually:

```php
$pipeline = array_reduce(
    array_reverse($middlewares),
    fn ($next, $middleware) => fn ($request) => $middleware->handle($request, $next),
    fn ($request) => $router->dispatch($request) // the core
);

$response = $pipeline($request);
```

That's the onion. The **last** middleware's `$next` is the route dispatch itself. Each `handle()` decides whether to call `$next` (continue) or return early (short-circuit). After the response is generated and sent, the kernel iterates any middleware exposing a `terminate()` method and calls it. This is why middleware order matters and why `$next($request)` is the pivot between "before" and "after" logic.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting to `return $next($request)`.**
   If your `handle()` runs logic but never returns the result of `$next($request)`, the controller may not run or the response is lost (you'll get a blank/`null` response error).
   **Fix:** unless you're deliberately short-circuiting, always `return $next($request);`.

2. **Putting "after" logic before `$next()`.**
   Trying to read or modify the *response* before calling `$next($request)` doesn't work — the response doesn't exist yet.
   **Fix:** capture `$response = $next($request);` first, then modify `$response`, then `return $response;`.

3. **Registering global middleware when you meant route middleware.**
   Calling `$middleware->append(MyMiddleware::class)` makes it run on **every** request (including health checks, assets, webhooks). People do this and then wonder why their role check fires on the login page.
   **Fix:** use `$middleware->alias()` and attach via `->middleware('role:admin')`, or add to a specific group with `web()`/`api()`.

4. **Expecting `terminate()` to use the same instance / state from `handle()`.**
   By default Laravel resolves a *fresh* instance from the container for `terminate()`, so `$this->startTime` set in `handle()` is lost.
   **Fix:** store request-scoped data on `$request->attributes`, or bind the middleware as a `singleton` in a service provider.

5. **Relying on session/auth in API middleware.**
   The `api` group has no session by default, so `$request->user()` via session won't work for stateless APIs.
   **Fix:** use token auth (Sanctum/Passport) and the `auth:sanctum` guard, or include session middleware deliberately.

6. **Assuming `php artisan serve` truly defers `terminate()`.**
   Terminable middleware only flushes the response *before* terminating under PHP-FPM/proper SAPIs. The dev server runs it inline.
   **Fix:** test terminable behavior against FPM or accept it runs inline locally; don't rely on timing in dev.

7. **Middleware ordering surprises.**
   Two middleware where one depends on the other (e.g. set-locale needs the authenticated user) can run in the "wrong" order.
   **Fix:** understand group order and the `priority()` list; place dependent middleware after its dependency.

---

## ✅ Best Practices

- **Keep middleware focused.** One concern per middleware (auth, locale, logging) — easier to test, reorder, and reuse.
- **Prefer route/group scope over global.** Global middleware taxes *every* request, including assets and health checks. Reach for global only for truly universal concerns (HTTPS enforcement, CORS, proxy trust).
- **Return early to short-circuit, clearly.** When denying access, `abort(403)` / `redirect()` instead of letting the request continue.
- **Use aliases for readability.** `->middleware('role:admin')` reads better in routes than a long FQCN.
- **Pass configuration as parameters, not hardcoded constants**, when the behavior varies per route (`throttle:5,1`, `role:editor`).
- **Use enums and `match`** for role/permission logic instead of magic strings.
- **Don't do heavy work in `handle()` that could go in `terminate()`** (logging, analytics) — let the user get their response first.
- **Be Octane-safe:** avoid mutable instance state; use request attributes or fresh resolution.
- **Write feature tests** that hit routes and assert on middleware effects (`$this->get('/admin')->assertForbidden();`).

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is middleware and why use it?**
A: Middleware is a layer that sits between the incoming HTTP request and your application logic. It lets you run code before and/or after the controller to handle cross-cutting concerns (auth, logging, rate limiting, CORS, localization) without duplicating that logic in every controller. It implements the pipeline/decorator pattern — the "onion" model.

**Q2. Walk me through the `handle($request, $next)` signature.**
A: `$request` is the HTTP request; `$next` is a closure representing the rest of the pipeline (remaining middleware + the route). Code before `$next($request)` runs on the way in; code after runs on the way out. You must return a `Response`. Not calling `$next` short-circuits the pipeline.

**Q3. Difference between before and after middleware?**
A: Before middleware runs logic *before* `$next($request)` — used to guard or mutate the request. After middleware captures `$response = $next($request)` then modifies/inspects the response before returning it. A single middleware can do both.

**Q4. Global vs group vs route middleware — how do you register each in Laravel 11/12?**
A: In `bootstrap/app.php` inside `withMiddleware()`: global via `append()`/`prepend()`; group via `web(append: [...])`/`api(prepend: [...])`/`group()`; route aliases via `alias([...])`, then attach with `->middleware('alias')`. Pre-Laravel 11 this lived in `app/Http/Kernel.php` (`$middleware`, `$middlewareGroups`, `$middlewareAliases`).

**Q5. How do you pass parameters to middleware?**
A: After `$next`, declare extra params (often variadic `string ...$roles`). In the route use `alias:value` or `alias:v1,v2`, e.g. `->middleware('role:admin,editor')`. Values arrive as strings.

**Q6. What is terminable middleware and when does `terminate()` run?**
A: Middleware with a `terminate($request, $response)` method runs *after* the response is sent to the client (under PHP-FPM/FastCGI via `fastcgi_finish_request()`). Ideal for logging/analytics so the user isn't blocked. Laravel resolves a *fresh* instance for `terminate()` by default, so bind the middleware as a singleton in the container if you need the same instance/state across `handle()` and `terminate()`.

**Q7. (Under the hood) How does Laravel actually execute the middleware stack?**
A: The HTTP kernel builds an `Illuminate\Pipeline\Pipeline` and uses `array_reduce` over the (reversed) middleware array to compose nested closures, each wrapping the next, with the route dispatch as the innermost core. Calling the composed closure invokes each `handle()`, where `$next` is the next layer. Terminable middleware are collected and called after the response is sent.

**Q8. What's middleware priority and when do you need it?**
A: A framework-defined ordering that forces certain middleware (e.g. `StartSession` before `Authenticate`) into a fixed sequence regardless of attachment order. Override via `$middleware->priority([...])`. Only middleware listed are re-ordered.

**Q9. What changed about middleware in Laravel 11?**
A: `app/Http/Kernel.php` was removed; all configuration moved to `bootstrap/app.php`'s `withMiddleware()` callback. Default framework middleware are applied internally and customized via fluent methods like `validateCsrfTokens(except: [...])` rather than editing a Kernel array.

**Q10. How does `auth` middleware decide between redirect and 401?**
A: It checks `$request->expectsJson()` (based on `Accept`/`X-Requested-With`). JSON/API clients get a `401 Unauthenticated`; browser requests are redirected to the `login` route.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate middleware
php artisan make:middleware EnsureUserHasRole

# Inspect which middleware are attached to each route (there is NO `middleware:list`
# command — use route:list with -v to reveal the middleware column)
php artisan route:list -v

# Hide third-party/vendor routes while inspecting
php artisan route:list -v --except-vendor
```

```php
// Minimal middleware
public function handle(Request $request, Closure $next): Response
{
    // before...
    $response = $next($request);
    // after...
    return $response;
}

// With parameters
public function handle(Request $request, Closure $next, string ...$roles): Response { }

// Terminable
public function terminate(Request $request, Response $response): void { }
```

```php
// bootstrap/app.php  (Laravel 11/12)
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(GlobalMw::class);                 // global
    $middleware->prepend(FirstMw::class);                 // global, first
    $middleware->web(append: [SetLocale::class]);         // web group
    $middleware->api(prepend: [ForceJson::class]);        // api group
    $middleware->appendToGroup('admin', [Role::class]);   // add to custom group
    $middleware->group('admin', [Auth::class, Role::class]); // define custom group
    $middleware->alias(['role' => Role::class]);          // route alias
    $middleware->priority([/* ordered list */]);          // ordering
    $middleware->remove(SomeDefault::class);              // remove from global stack
    $middleware->validateCsrfTokens(except: ['stripe/*']);// CSRF excludes
})
```

```php
// Routes
Route::get('/x', C::class)->middleware('auth');
Route::get('/x', C::class)->middleware(['auth','verified','role:admin']);
Route::get('/x', C::class)->middleware(EnsureTokenIsValid::class);
Route::middleware(['auth'])->group(function () { /* ... */ });
Route::get('/x', C::class)->withoutMiddleware(['auth']);

// Built-ins
->middleware('auth');             // require login
->middleware('auth:sanctum');     // token guard
->middleware('verified');         // verified email
->middleware('signed');           // signed URL
->middleware('can:update,post');  // gate/policy authorization
->middleware('throttle:60,1');    // 60 req / 1 min
->middleware('throttle:api');     // named limiter
```

| Concept | Key API |
|---|---|
| Continue pipeline | `return $next($request);` |
| Short-circuit | `abort()`, `redirect()`, return a `Response` |
| Global | `append()` / `prepend()` |
| Group | `web()` / `api()` / `group()` |
| Alias | `alias([...])` then `->middleware('name')` |
| Params | `handle(..., $next, ...$args)` + `name:a,b` |
| Run after send | `terminate($request, $response)` |
| Ordering | `priority([...])` |

---

## 🧪 Mini Exercises

1. **ForceJson API middleware.** Write `ForceJsonResponse` that sets the request's `Accept` header to `application/json` *before* the controller runs, so all API responses (including validation errors) come back as JSON. Register it on the `api` group via `prepend`.

2. **Role + permission parameter.** Extend `EnsureUserHasRole` to accept a permission too, used as `->middleware('role:admin')` and `->middleware('role:editor,viewer')`. Back the roles with a PHP enum and use `match` to decide access. Add a 403 with a helpful message.

3. **Timing + terminate.** Build `MeasureResponseTime` that records the start time in `handle()` (on `$request->attributes`) and, in `terminate()`, logs the total milliseconds and the HTTP status. Bind it as a singleton and confirm via a feature test that a slow route logs a duration.

4. **Maintenance window.** Create `BlockDuringMaintenance` that returns a `503` JSON response (`{"message":"Down for maintenance"}`) for all `api/*` routes between two configured times, but allows requests carrying a valid bypass token header. Register it globally but ensure it doesn't affect the `/up` health route.

5. **Locale from subdomain.** Write a middleware that reads the locale from a subdomain (`fr.example.com` → `fr`), validates it against a supported list, falls back to `en`, and sets `App::setLocale()`. Add a feature test asserting `App::getLocale()` for two different hosts.
