# Routing in Laravel 12

Routing is the front door of every Laravel application. A **route** is a rule that maps an incoming HTTP request — defined by its *HTTP verb* (GET, POST, etc.) and its *URI* (the path like `/users/5`) — to a piece of code that produces a response (a closure or a controller method). Before any of your business logic runs, the router decides *what* runs. Get routing right and the rest of the framework falls into place; get it wrong and you'll fight 404s, wrong middleware, and broken URLs for weeks.

This module takes you from "what is a route" all the way to scoped model binding, rate limiters, and signed URLs — the things interviewers actually probe.

**What you'll learn**

- How and where routes are defined in Laravel 12, and why `web.php` and `api.php` behave differently.
- Every HTTP verb helper plus `match`, `any`, `redirect`, and `view`.
- Route parameters: required, optional, defaults, and regex constraints (`where`, `whereNumber`, global patterns).
- Named routes and how to generate URLs safely with the `route()` helper.
- Route groups for sharing prefixes, middleware, names, controllers, and domains.
- Route model binding — implicit, custom keys, scoped, and explicit — and how it works under the hood.
- Resource and API resource routes, including nesting and `only`/`except`.
- Production concerns: fallback routes, rate limiting, signed URLs, CSRF, and route caching.

---

## 1. Why routing exists and where routes live

Every web request that hits your app is just text: a method (`GET`), a path (`/dashboard`), some headers, maybe a body. Something has to translate that into "call this function." That something is the **router**. Laravel's router is one of the framework's fastest, most battle-tested components: it compiles your route definitions into an optimized data structure and matches each request against it.

### The route files

In a fresh Laravel 12 app, routes live in the `routes/` directory:

```text
routes/
├── web.php       # Browser-facing routes: sessions, cookies, CSRF
├── api.php       # Stateless API routes (opt-in in L11+)
└── console.php   # Artisan/closure commands (not HTTP)
```

The critical difference between `web.php` and `api.php` is the **middleware group** applied to each:

- Routes in `web.php` get the `web` middleware group: session state, cookies, and **CSRF protection**. This is what lets `Auth::user()` and flash messages work.
- Routes in `api.php` get the `api` middleware group: stateless, no sessions, and they are automatically prefixed with `/api`. They are typically protected with token auth (Sanctum) instead of CSRF.

> **Laravel 11/12 change (important for interviews):** The old `app/Http/Kernel.php` is **gone**. Bootstrapping now happens in `bootstrap/app.php`. Also, `api.php` is **not created by default** in a fresh app — you opt in by running `php artisan install:api`, which installs Sanctum and registers the API routes.

Here is the routing wiring in `bootstrap/app.php`:

```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',                 // built-in health-check endpoint
        apiPrefix: 'api',              // change the /api prefix here
        // then: function () { ... }   // register extra route files
    )
    ->withMiddleware(function ($middleware) {
        // register/append middleware here (replaces Kernel.php)
    })
    ->create();
```

If you want a third route file (say `routes/admin.php`), register it in the `then` callback. Note the `Illuminate\Support\Facades\Route` import — the closure needs the facade in scope:

```php
use Illuminate\Support\Facades\Route;

->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
    then: function () {
        Route::middleware('web')
            ->prefix('admin')
            ->name('admin.')
            ->group(base_path('routes/admin.php'));
    },
)
```

> **`then` vs `using`:** `then` *adds* route files on top of the framework's default registration. If you pass a `using` closure instead, Laravel registers **no** HTTP routes automatically and you take full control of all route registration.

---

## 2. HTTP verb methods

The simplest possible route maps a verb + URI to a closure that returns a response. Laravel converts strings, arrays, and `Arrayable` returns into proper HTTP responses automatically (arrays become JSON).

```php
<?php
// routes/web.php
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return 'Hello, world';        // 200 text/html
});

Route::get('/users', function () {
    return ['name' => 'Ada'];     // 200 application/json
});
// Output (GET /users): {"name":"Ada"}
```

There is one helper per common verb:

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```

**Why so many verbs?** HTTP semantics. `GET` reads, `POST` creates, `PUT` replaces a whole resource, `PATCH` updates part of it, `DELETE` removes it. Following these conventions makes your API predictable and lets caches/proxies behave correctly.

### `match` and `any`

When one URI should respond to several verbs, use `match`. When it should respond to *all* verbs, use `any`:

```php
Route::match(['get', 'post'], '/contact', function () {
    // handles GET (show form) and POST (submit)
});

Route::any('/legacy', function () {
    // responds to every HTTP verb — use sparingly
});
```

> **Gotcha:** Browsers can only send `GET` and `POST` from HTML forms. To send `PUT`, `PATCH`, or `DELETE` from a form, add a hidden `_method` field. Blade's `@method` directive does this:

```blade
<form action="/posts/5" method="POST">
    @csrf
    @method('DELETE')
    <button>Delete</button>
</form>
```

Laravel reads the hidden `_method` field at the HTTP layer — when it builds the `Request`, it calls Symfony's `Request::enableHttpMethodParameterOverride()`, so the spoofed verb is in effect *before* the router matches. The POST therefore routes to your `Route::delete('/posts/{post}', ...)`. (Method spoofing only works on `POST` requests carrying a `_method` field.)

### Pointing routes at controllers

Closures are great for demos, but real apps use controllers. The modern idiom uses the `::class` constant and a `[Controller::class, 'method']` tuple — no magic strings:

```php
use App\Http\Controllers\PostController;

Route::get('/posts', [PostController::class, 'index']);

// Single-action (invokable) controller — no method name needed:
Route::post('/reports', GenerateReportController::class);
```

---

## 3. Route parameters

A **route parameter** is a placeholder in the URI captured and passed to your handler. Wrap the segment in braces.

### Required parameters

```php
Route::get('/users/{id}', function (string $id) {
    return "User {$id}";
});
// GET /users/42  -> "User 42"
// GET /users     -> 404 (parameter is required)
```

Route parameters are injected **by order**, not by name — the names of the callback/controller arguments do not have to match the URI segment names (matching them anyway is good for readability). The one rule that matters: if you also type-hint services for the container to resolve (like `Request`), list those **dependencies first** and your route parameters **after** them:

```php
use Illuminate\Http\Request;

Route::get('/posts/{post}/comments/{comment}', function (Request $request, string $post, string $comment) {
    return "Post {$post}, comment {$comment}";
});
```

> **Note:** The "by order" rule applies to plain scalar parameters. Route *model binding* (Section 6) is the exception — there the type-hint and the variable name **do** matter, because Laravel matches the `{post}` segment to a `Post $post` parameter by name.

### Optional parameters and defaults

Add a `?` to make a parameter optional, and give the matching argument a default value. The argument should be nullable (`?string`) or have a default — when the segment is absent, Laravel passes the default:

```php
Route::get('/greet/{name?}', function (?string $name = 'guest') {
    return "Hello, {$name}";
});
// GET /greet        -> "Hello, guest"
// GET /greet/Ada    -> "Hello, Ada"

// A plain default of null is also valid:
Route::get('/greet/{name?}', function (?string $name = null) {
    return "Hello, " . ($name ?? 'guest');
});
```

> **Gotcha:** An optional parameter `{name?}` **must** have a default on the argument, or PHP throws an `ArgumentCountError` when the segment is absent. Note that an optional parameter can only be the **last** segment of the URI.

### Regex constraints with `where`

By default a parameter matches *anything except a slash*. Use `where` to constrain it with a regular expression — this both validates input and prevents route collisions.

```php
Route::get('/users/{id}', function (string $id) {
    return "User {$id}";
})->where('id', '[0-9]+');

Route::get('/users/{name}', function (string $name) {
    return "User {$name}";
})->where('name', '[A-Za-z]+');
```

Now `/users/42` hits the first route and `/users/ada` hits the second — the regex disambiguates them. (Route order still matters: the first **matching** route wins.)

Constrain multiple parameters at once with an array:

```php
Route::get('/posts/{post}/{slug}', $handler)->where([
    'post' => '[0-9]+',
    'slug' => '[a-z\-]+',
]);
```

### Fluent constraint helpers

Laravel ships readable shortcuts so you rarely write raw regex:

```php
Route::get('/orders/{id}', $h)->whereNumber('id');           // \d+
Route::get('/tags/{tag}', $h)->whereAlpha('tag');            // [a-zA-Z]+
Route::get('/codes/{c}', $h)->whereAlphaNumeric('c');        // [a-zA-Z0-9]+
Route::get('/u/{ulid}', $h)->whereUlid('ulid');
Route::get('/u/{uuid}', $h)->whereUuid('uuid');
Route::get('/status/{s}', $h)->whereIn('s', ['active', 'archived']);
```

### Global patterns

If a parameter name *always* means the same thing across your whole app (e.g. `id` is always numeric), set a global pattern once in a service provider's `boot` method instead of repeating `where`:

```php
<?php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Route;

public function boot(): void
{
    Route::pattern('id', '[0-9]+');
}
```

Now every `{id}` parameter in every route is automatically constrained to digits.

### Encoded slashes

To allow a slash *inside* a single parameter (rare — e.g. file paths), you must explicitly opt in with `.*`, and it only works on the **last** segment:

```php
Route::get('/files/{path}', fn (string $path) => $path)->where('path', '.*');
// GET /files/docs/2026/report.pdf -> "docs/2026/report.pdf"
```

---

## 4. Named routes and the `route()` helper

Hardcoding URLs like `'/users/'.$id` everywhere is fragile: change the path and you must hunt down every reference. **Named routes** decouple the *name* (stable) from the *URL* (may change). You then generate URLs from the name.

```php
Route::get('/user/profile', [ProfileController::class, 'show'])->name('profile');
```

Generate URLs and redirects from the name:

```php
$url = route('profile');                 // http://app.test/user/profile
return redirect()->route('profile');
return to_route('profile');              // shorthand redirect (L9+)
```

Pass parameters as an array. Extra keys become query strings automatically:

```php
Route::get('/users/{id}/posts/{post}', $h)->name('users.posts.show');

route('users.posts.show', ['id' => 1, 'post' => 99]);
// -> /users/1/posts/99

route('users.posts.show', ['id' => 1, 'post' => 99, 'page' => 2]);
// -> /users/1/posts/99?page=2
```

In Blade you reference names too:

```blade
<a href="{{ route('profile') }}">My Profile</a>
```

> **Why this matters in interviews:** named routes make refactoring safe and let you check the *current* route by name (`$request->routeIs('profile')`) rather than parsing the path. `route()` will **throw** a `RouteNotFoundException` if the name doesn't exist — a typo fails loudly instead of silently building a wrong URL.

---

## 5. Route groups

When many routes share attributes — a URL prefix, middleware, a name prefix, a controller, or a subdomain — wrap them in a **group** so you write the shared config once. Groups can be nested; attributes merge.

```php
use App\Http\Controllers\Admin\DashboardController;

Route::middleware(['auth', 'verified'])      // shared middleware
    ->prefix('admin')                        // URL prefix:  /admin/...
    ->name('admin.')                         // name prefix: admin....
    ->group(function () {

        Route::get('/dashboard', [DashboardController::class, 'index'])
            ->name('dashboard');             // full name: admin.dashboard, URL: /admin/dashboard

        Route::get('/users', [DashboardController::class, 'users'])
            ->name('users');                 // admin.users, /admin/users
    });
```

### Controller groups

When every route in a group points at the same controller, hoist it with `controller()` and reference only the method names:

```php
Route::controller(OrderController::class)->group(function () {
    Route::get('/orders/{id}', 'show');
    Route::post('/orders', 'store');
});
```

### Domain / subdomain routing

`domain()` matches against the host, and host segments can themselves be parameters:

```php
Route::domain('{tenant}.myapp.com')->group(function () {
    Route::get('/dashboard', function (string $tenant) {
        return "Tenant: {$tenant}";
    });
});
// GET https://acme.myapp.com/dashboard -> "Tenant: acme"
```

> **Gotcha:** Domain routes are matched **before** path, so register them carefully and make sure your web server / local `/etc/hosts` actually resolves the subdomains.

---

## 6. Route model binding

Almost every controller starts the same way: take an ID from the URL, look up the model, 404 if missing. **Route model binding** automates this. Laravel resolves the model *before* your handler runs and injects the instance.

### Implicit binding

Type-hint a parameter with an Eloquent model whose variable name matches the route segment:

```php
use App\Models\Post;

Route::get('/posts/{post}', function (Post $post) {
    return $post->title;     // already fetched, or auto-404 if not found
});
```

Laravel sees the `{post}` segment, sees the `Post $post` type-hint, and resolves the model via the model's `resolveRouteBinding()` method — effectively `Post::where($post->getRouteKeyName(), $value)->firstOrFail()` (the route key defaults to `id`). If nothing is found it raises a `ModelNotFoundException`, which the exception handler converts to a **404**. No manual `find()`, no manual 404.

### Customizing the key

By default binding uses the `id` column. To resolve by another column (e.g. a slug), either specify it in the URI or override `getRouteKeyName()` on the model:

```php
// Per-route: bind by slug
Route::get('/posts/{post:slug}', fn (Post $post) => $post);
// GET /posts/hello-world -> resolves WHERE slug = 'hello-world'
```

```php
// Model-wide default key
class Post extends Model
{
    public function getRouteKeyName(): string
    {
        return 'slug';
    }
}
```

> **PHP 8.1+ enums tie-in:** You can also type-hint a *backed enum* on a route parameter. Laravel matches the segment against the enum's backing values and 404s if it doesn't match — a clean way to constrain to a fixed set:

```php
enum Category: string { case Tech = 'tech'; case Life = 'life'; }

Route::get('/feed/{category}', function (Category $category) {
    return $category->value;
});
// GET /feed/tech -> "tech";  GET /feed/sports -> 404
```

### Scoped bindings (nested resources)

When a route has a parent and child, you usually want the child scoped *to the parent* — e.g. "the comment must belong to this post." There are two ways to get scoping:

- **Automatic** — use a *custom key* on the child (`{comment:slug}`). When a nested parameter has a custom key, Laravel automatically scopes the child to the parent, guessing the relationship name from the plural of the child parameter (`{comment}` -> `comments()`).
- **Explicit** — call `->scopeBindings()` to force scoping even when the child uses its default key.

```php
Route::get('/posts/{post}/comments/{comment}', function (Post $post, Comment $comment) {
    // $comment guaranteed to belong to $post
    return $comment->body;
})->scopeBindings();
```

Under the hood this resolves the child via `$post->comments()->where(...)->firstOrFail()` (using the relationship and the child's route key) instead of a global `Comment` lookup — so a comment from a *different* post yields 404, not a data leak. You can force scoping on a whole group with `Route::scopeBindings()->group(...)`, or opt a specific route out with `->withoutScopedBindings()`.

### Explicit binding

For full control (custom query, soft-delete handling, non-conventional names), register a resolver in a service provider:

```php
use App\Models\User;
use Illuminate\Support\Facades\Route;

public function boot(): void
{
    // Bind '{user}' to User by id (custom logic possible)
    Route::model('user', User::class);

    // Or fully custom resolution:
    Route::bind('user', function (string $value) {
        return User::where('username', $value)->firstOrFail();
    });
}
```

By default, soft-deleted models are **not** found by binding. To include them, chain `->withTrashed()`:

```php
Route::get('/users/{user}', fn (User $user) => $user)->withTrashed();
```

> **How it works under the hood:** implicit binding is performed by the `SubstituteBindings` middleware (part of both the `web` and `api` groups). It inspects the matched route's handler signature via reflection, finds parameters whose type implements `UrlRoutable`, and replaces the raw string value with a resolved instance — *before* your controller is invoked. If you remove `SubstituteBindings` from the group, model binding silently stops working and you'll get raw strings.

---

## 7. Resource routes

A typical CRUD resource needs seven routes (index, create, store, show, edit, update, destroy). `Route::resource` generates all of them with conventional names and URIs from a single line.

```php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

This produces:

| Verb        | URI                    | Controller method | Route name        |
|-------------|------------------------|-------------------|-------------------|
| GET         | `/photos`              | `index`           | `photos.index`    |
| GET         | `/photos/create`       | `create`          | `photos.create`   |
| POST        | `/photos`              | `store`           | `photos.store`    |
| GET         | `/photos/{photo}`      | `show`            | `photos.show`     |
| GET         | `/photos/{photo}/edit` | `edit`            | `photos.edit`     |
| PUT/PATCH   | `/photos/{photo}`      | `update`          | `photos.update`   |
| DELETE      | `/photos/{photo}`      | `destroy`         | `photos.destroy`  |

Scaffold the matching controller with the right method stubs:

```bash
php artisan make:controller PhotoController --resource --model=Photo
```

### API resources

APIs don't need the HTML-form routes (`create`, `edit`). `apiResource` omits them, leaving the five JSON-relevant actions:

```php
Route::apiResource('photos', PhotoController::class);
// index, store, show, update, destroy  (no create/edit)

// Register many at once:
Route::apiResources([
    'photos'   => PhotoController::class,
    'comments' => CommentController::class,
]);
```

```bash
php artisan make:controller PhotoController --api --model=Photo
```

### `only` / `except` and renaming

Limit which actions are generated:

```php
Route::resource('photos', PhotoController::class)->only(['index', 'show']);
Route::resource('photos', PhotoController::class)->except(['destroy']);
```

Customize names and the parameter name:

```php
Route::resource('users', UserController::class)
    ->names(['index' => 'users.list'])
    ->parameters(['users' => 'admin_user'])     // {admin_user} instead of {user}
    ->where(['user' => '[0-9]+']);
```

### Nested resources

Express parent/child relationships with dot notation:

```php
Route::resource('posts.comments', CommentController::class);
// e.g. GET /posts/{post}/comments/{comment}  -> name posts.comments.show
```

To avoid deeply nested URLs, use **shallow nesting** — index/create/store stay nested, but show/edit/update/destroy use only the child key:

```php
Route::resource('posts.comments', CommentController::class)->shallow();
// GET /posts/{post}/comments       (posts.comments.index)
// GET /comments/{comment}          (comments.show)
```

Combine with scoped bindings for safe nested lookups. Calling `->scoped()` with no arguments enables scoping using each child's default route key; pass an array to also pick the child's binding field:

```php
Route::resource('posts.comments', CommentController::class)->scoped();
Route::resource('posts.comments', CommentController::class)->scoped(['comment' => 'slug']);
// -> /posts/{post}/comments/{comment:slug}, scoped to the parent post
```

### Customizing the "model not found" behavior

By default a missing implicitly-bound resource model yields a 404. Override that for *all* of a resource's routes with `missing()` — for example, redirect back to the index instead of 404ing:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redirect;

Route::resource('photos', PhotoController::class)
    ->missing(fn (Request $request) => Redirect::route('photos.index'));
```

To allow soft-deleted models on the binding-aware actions (`show`, `edit`, `update`), chain `->withTrashed()` (optionally scoped to specific actions, e.g. `->withTrashed(['show'])`).

### Singleton resources

For a resource that only ever has one instance (a user's `profile`, an image's `thumbnail`), use `Route::singleton` / `Route::apiSingleton`. These register `show`, `edit`, `update` (no `index`/`create`/`store` and **no `{id}` segment**), and you can add creation/deletion with `->creatable()` or `->destroyable()`:

```php
Route::singleton('profile', ProfileController::class);     // GET /profile, GET /profile/edit, PUT/PATCH /profile
Route::apiSingleton('profile', ProfileController::class);  // show + update only (no edit)
```

---

## 8. Convenience routes

Some routes don't need a controller at all.

### `Route::redirect`

```php
Route::redirect('/here', '/there');            // 302 by default
Route::redirect('/here', '/there', 301);       // permanent
Route::permanentRedirect('/old', '/new');      // 301 shortcut
```

### `Route::view`

Render a Blade view directly, passing optional data — perfect for static-ish pages:

```php
Route::view('/welcome', 'welcome');
Route::view('/about', 'pages.about', ['team' => 'Platform']);
```

> **Gotcha — reserved parameter names (and they differ per method):**
> - In `Route::redirect` URIs, the parameter names `destination` and `status` are reserved by Laravel and cannot be used.
> - In `Route::view` URIs, the reserved names are `view`, `data`, `status`, and `headers`.
>
> These are convenience helpers — you can still attach middleware to them by chaining `->middleware(...)`, but you can't put request-handling logic inside them; that's what controllers are for.

### Fallback routes

A **fallback** runs when *no other route matches* — your custom 404 handler at the routing layer. Define it **last**:

```php
Route::fallback(function () {
    return response()->view('errors.404', [], 404);
});
```

It only fires for routes that fail to match; it does not catch exceptions thrown inside matched routes.

---

## 9. Rate limiting

**Rate limiting** caps how many requests a client may make in a time window — protecting against abuse, brute force, and runaway costs. Laravel applies it through the `throttle` middleware backed by a named **rate limiter**.

### Inline throttle

```php
Route::middleware('throttle:60,1')->group(function () {
    // max 60 requests per 1 minute per client
    Route::get('/feed', [FeedController::class, 'index']);
});
```

### Named rate limiters (the modern way)

Define limiters centrally so the logic (per-user vs per-IP, tiers, etc.) lives in one place. In Laravel 12 the conventional home for these definitions is the `boot()` method of `App\Providers\AppServiceProvider`:

```php
<?php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(100)->by($request->user()->id)   // logged-in: by user
            : Limit::perMinute(20)->by($request->ip());          // guest: by IP
    });

    // Multiple limits evaluated in order.
    // IMPORTANT: when stacking limits, give each `by()` a UNIQUE key
    // (e.g. prefix it) so the per-minute and per-day buckets don't collide:
    RateLimiter::for('uploads', function (Request $request) {
        $key = $request->user()?->id ?: $request->ip();

        return [
            Limit::perMinute(10)->by('minute:'.$key),
            Limit::perDay(100)->by('day:'.$key),
        ];
    });

    RateLimiter::for('login', function (Request $request) {
        return Limit::perMinute(5)->by($request->ip())
            // The response closure receives ($request, array $headers);
            // pass $headers through so X-RateLimit-* / Retry-After survive.
            ->response(fn (Request $request, array $headers) =>
                response('Slow down', 429, $headers));
    });
}
```

Attach by name:

```php
Route::middleware('throttle:api')->group(function () { /* ... */ });
Route::middleware('throttle:uploads')->prefix('uploads')->group(/* ... */);
```

When the limit is exceeded the client receives **HTTP 429 Too Many Requests**. Laravel adds `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers (note the `X-` prefix) to responses while attempts remain, and on a 429 it also sends `Retry-After` and `X-RateLimit-Reset` so clients know when to retry. (These header names come from `Illuminate\Routing\Middleware\ThrottleRequests::getHeaders()`.)

> **Response-based rate limiting (Laravel 12):** you can chain `->after(fn (Response $response) => $response->status() === 404)` onto a `Limit` so that only certain responses count toward the limit. This is handy for throttling consecutive 404s to block resource-enumeration attacks without penalizing legitimate traffic.

> **Throttling with Redis:** by default `throttle` uses `Illuminate\Routing\Middleware\ThrottleRequests` (your cache store). If your cache driver is Redis, opt into the Redis-optimized implementation in `bootstrap/app.php` with `$middleware->throttleWithRedis();` so all app servers share atomic counts.

> **Under the hood:** the limiter increments a counter in the cache store keyed by your `by()` value with a TTL equal to the window. `Limit::perMinute` is sugar over the underlying counter; `by()` defines the bucket key. Because it's cache-backed, in production you want a fast shared store (Redis) so all app servers share counts.

---

## 10. Signed URLs

A **signed URL** carries a cryptographic signature in its query string, so the server can verify it wasn't tampered with — without a session. Perfect for "unsubscribe" links, email verification, and public-but-protected links.

```php
use Illuminate\Support\Facades\URL;

// Generate
$url       = URL::signedRoute('unsubscribe', ['user' => 1]);
$temporary = URL::temporarySignedRoute('unsubscribe', now()->addMinutes(30), ['user' => 1]);
```

Validate either with the `signed` middleware (cleanest) or manually:

```php
// Middleware approach — auto-aborts 403 if invalid/expired
Route::get('/unsubscribe/{user}', UnsubscribeController::class)
    ->name('unsubscribe')
    ->middleware('signed');

// Manual approach
Route::get('/unsubscribe/{user}', function (Request $request) {
    if (! $request->hasValidSignature()) {
        abort(403);
    }
    // ...
})->name('unsubscribe');
```

When the `signed` middleware rejects a request it aborts with a **403** response. To show a friendly page instead of the generic 403, register a render handler for `InvalidSignatureException` in `bootstrap/app.php`:

```php
use Illuminate\Routing\Exceptions\InvalidSignatureException;

->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (InvalidSignatureException $e) {
        return response()->view('errors.link-expired', status: 403);
    });
})
```

> **Relative (host-less) signatures:** if you generate the URL with `URL::signedRoute('name', [...], absolute: false)`, you must validate it with `->middleware('signed:relative')` (or `hasValidSignature(absolute: false)`), otherwise the host mismatch invalidates it.

> **Gotcha:** Signed URLs are tied to `APP_KEY` and the exact host/path/params. If `APP_KEY` differs between environments, or a proxy rewrites the host, validation fails (use relative signatures to sidestep host issues). Use `hasValidSignatureWhileIgnoring(['page', 'order'])` to allow extra query params your front-end appends — but remember those ignored params can then be tampered with freely.

---

## 11. Current-route info, `route:list`, and CSRF

### Inspecting the current route

```php
use Illuminate\Support\Facades\Route;

Route::current();              // Illuminate\Routing\Route instance
Route::currentRouteName();     // 'photos.show'
Route::currentRouteAction();   // 'App\Http\Controllers\PhotoController@show'

// From a Request (preferred, testable):
$request->route()->getName();
$request->routeIs('admin.*');  // true if name matches pattern -> great for nav highlighting
$request->route('post');       // value of the {post} parameter
```

In Blade, highlight the active nav item:

```blade
<a class="{{ request()->routeIs('dashboard') ? 'active' : '' }}" href="{{ route('dashboard') }}">
    Dashboard
</a>
```

### `php artisan route:list`

Your single best debugging tool for routing. It prints every registered route with its verb, URI, name, action, and middleware.

```bash
php artisan route:list

# Useful filters:
php artisan route:list --name=user        # only routes whose name contains "user"
php artisan route:list --path=api         # only URIs containing "api"
php artisan route:list --method=POST
php artisan route:list -v                 # show middleware
php artisan route:list --except-vendor    # hide package routes
```

### CSRF on web routes

**CSRF (Cross-Site Request Forgery)** is an attack where a malicious site tricks a logged-in user's browser into submitting a request to your app. Laravel's `web` group includes CSRF protection: every "unsafe" verb (`POST`, `PUT`, `PATCH`, `DELETE`) must include a valid token, or Laravel returns **419 Page Expired**.

Add the token to forms with `@csrf`:

```blade
<form method="POST" action="{{ route('posts.store') }}">
    @csrf
    <input name="title">
    <button>Save</button>
</form>
```

For AJAX, send the token via the `X-CSRF-TOKEN` header (read from a `<meta>` tag) or use the `XSRF-TOKEN` cookie that Laravel sets automatically (Axios reads it for you).

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

To exempt specific URIs (e.g. an incoming webhook that can't have a token), register them in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [
        'stripe/webhook',
        'webhooks/*',
    ]);
})
```

> **Why API routes don't need CSRF:** `api.php` routes are stateless and authenticated by token, not session cookie. CSRF only matters when the browser *automatically* attaches a credential (the session cookie). No auto-attached credential, no CSRF risk — so the `api` group omits the CSRF middleware.

### A note on route caching (production)

In production, cache the compiled routes so they aren't re-parsed on every request:

```bash
php artisan route:cache    # build the cache (run on deploy)
php artisan route:clear    # remove it
```

> **Critical gotcha:** `route:cache` requires that **all** route handlers be controllers — **closures cannot be serialized**. If any route uses a closure, the command throws a `LogicException` whose message is roughly *"Unable to prepare route [...] for serialization. Uses Closure."* Convert closures to controllers (or invokable controllers) before caching. (A separately worded `LogicException: Unable to prepare route for serialization. Another route has the same name.` is a *different* error — it means two routes share the same name; route names must be unique.)

---

## ⚠️ Common Mistakes & Gotchas

1. **Route order matters — specific routes must come first.**
   `Route::get('/users/{id}')` will swallow `/users/create` because `create` matches the `{id}` wildcard.
   **Fix:** Define static/specific routes *before* parameterized ones, or constrain the parameter: `->whereNumber('id')`.

2. **Forgetting `@csrf` (or `@method`) in Blade forms.**
   POST forms without `@csrf` return **419 Page Expired**; "edit" forms without `@method('PUT')` hit your `store` instead of `update`.
   **Fix:** Always add `@csrf`, and `@method('PUT'|'PATCH'|'DELETE')` for non-POST verbs.

3. **`route:cache` breaks because of closure routes.**
   Caching fails the moment one route uses a closure.
   **Fix:** Move all logic into controllers/invokables. Run `php artisan route:list` to find stragglers.

4. **Implicit binding by the wrong column, or scoping not applied.**
   `/posts/{post:slug}` works, but a nested `/posts/{post}/comments/{comment}` without `scopeBindings()` lets a comment from *another* post resolve — a data-leak bug.
   **Fix:** Use `->scopeBindings()` (or `->scoped()` on resources) for nested routes; set `getRouteKeyName()` for non-`id` keys.

5. **Optional parameter without a default argument.**
   `Route::get('/greet/{name?}', fn (string $name) => ...)` throws `ArgumentCountError` when the segment is missing.
   **Fix:** Give the argument a default: `fn (string $name = 'guest') => ...`.

6. **Assuming `api.php` exists.**
   In Laravel 11/12 it's not created by default, so your `/api/...` calls 404.
   **Fix:** Run `php artisan install:api` to scaffold it (and Sanctum).

7. **`route()` typo silently shipping the wrong URL.**
   Actually it doesn't ship silently — `route('porfile')` throws `RouteNotFoundException`. The real mistake is catching/ignoring it.
   **Fix:** Let it fail loudly in dev; run your test suite which hits the routes.

---

## ✅ Best Practices

- **Always name your routes.** Reference them via `route()`/`to_route()` so URL changes don't break links.
- **Group shared attributes** (middleware, prefix, name, controller, domain) instead of repeating them per route.
- **Keep `web.php` and `api.php` separate** by concern; opt into `api.php` deliberately.
- **Push logic into controllers**, not closures — required for `route:cache` and far more testable.
- **Constrain parameters** with `whereNumber`/`whereUuid`/global patterns to prevent collisions and reject garbage early.
- **Use route model binding** (and scoped/explicit binding) instead of manual `find()` + 404 boilerplate.
- **Centralize rate limits** with named `RateLimiter::for(...)` definitions; back them with Redis in production.
- **Cache routes and config on deploy** (`route:cache`, `config:cache`), and clear the cache on rollback.
- **Use `Route::fallback`** for a friendly 404 and `signed` URLs for tamper-proof public links.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `web.php` and `api.php`?**
A. They apply different middleware groups. `web` adds sessions, cookies, and CSRF protection (browser-facing, stateful). `api` is stateless, has no CSRF, is auto-prefixed with `/api`, and is typically token-authenticated (Sanctum). In Laravel 11/12, `api.php` is opt-in via `php artisan install:api`.

**Q2. How does route model binding work under the hood?**
A. The `SubstituteBindings` middleware runs after route matching. It reflects on the route handler's signature, finds parameters typed as Eloquent models (which implement `UrlRoutable`), and resolves each via `resolveRouteBinding()` — effectively `Model::where(getRouteKeyName(), $value)->firstOrFail()`. The resolved instance is injected into the controller. Remove that middleware and binding stops working.

**Q3. Why do you sometimes need `scopeBindings()`?**
A. For nested routes like `/posts/{post}/comments/{comment}`, default binding resolves each model independently — a comment from a different post would still resolve, leaking data. Scoped binding constrains the child to the parent (`$post->comments()->...`), 404ing mismatches. Resource routes use `->scoped()`.

**Q4. How does Laravel handle PUT/PATCH/DELETE from an HTML form?**
A. Browsers only send GET/POST from forms, so Laravel uses **method spoofing**: a hidden `_method` field (emitted by `@method('PUT')`) tells Laravel to treat the POST request as the spoofed verb during routing.

**Q5. Why don't API routes need CSRF protection?**
A. CSRF exploits credentials the browser *auto-attaches* (the session cookie). API routes are stateless and authenticated by an explicit token (header/bearer), which the browser does not auto-send cross-site — so there's nothing to forge. Hence the `api` group omits CSRF middleware.

**Q6. What breaks `route:cache` and why?**
A. Closure-based routes. The route cache serializes route definitions to PHP, and closures can't be serialized. All routes must use controllers/invokables to cache. This is why production code avoids closure routes.

**Q7. `Route::match` vs `Route::any` vs multiple verb helpers — when each?**
A. Use specific helpers (`get`, `post`, ...) for clarity and correct semantics. `match(['get','post'], ...)` when one URI legitimately handles a small set of verbs (e.g. show + submit). `any()` only for catch-alls; it's a code smell otherwise because it muddies HTTP semantics.

**Q8. How would you rate-limit login attempts per IP and return a custom 429?**
A. Define a named limiter: `RateLimiter::for('login', fn (Request $r) => Limit::perMinute(5)->by($r->ip())->response(fn ($r, array $h) => response('Slow down', 429, $h)));` then attach `throttle:login`. The `response` closure receives the rate-limit `$headers` array — pass it through so `X-RateLimit-*`/`Retry-After` survive. It's cache-backed (Redis in prod) keyed by the `by()` value.

**Q9. How do signed URLs stay secure without a session?**
A. Laravel appends an HMAC `signature` query param computed from the full URL + `APP_KEY`. On request, `hasValidSignature()` recomputes and compares; any tampering changes the hash. `temporarySignedRoute` also bakes in an `expires` timestamp that's part of the signed payload.

**Q10. Given two routes `/users/{id}` and `/users/create`, why might `/users/create` 404 or hit the wrong handler, and how do you fix it?**
A. If `/users/{id}` is declared first and unconstrained, `create` matches `{id}`. Fix by ordering `create` first, or constraining `{id}` with `->whereNumber('id')` so `create` (non-numeric) falls through correctly.

---

## 📋 Quick Reference / Cheat Sheet

```php
// Verbs
Route::get|post|put|patch|delete|options($uri, $action);
Route::match(['get','post'], $uri, $action);
Route::any($uri, $action);

// Controllers
Route::get('/x', [Ctrl::class, 'method']);
Route::post('/x', InvokableCtrl::class);

// Parameters & constraints
Route::get('/u/{id}', $h)->whereNumber('id');
Route::get('/u/{name?}', fn ($name = 'guest') => $name);
Route::get('/u/{id}', $h)->where('id', '[0-9]+');
->whereAlpha | whereAlphaNumeric | whereUuid | whereUlid | whereIn('p',[...]);
Route::pattern('id', '[0-9]+');             // global, in a provider boot()

// Named routes & URLs
Route::get('/p', $h)->name('profile');
route('profile', ['id' => 1, 'page' => 2]); // -> /.../1?page=2
to_route('profile');                         // redirect helper

// Groups
Route::middleware([...])->prefix('admin')->name('admin.')
    ->controller(Ctrl::class)->domain('{t}.app.com')->group(fn () => /* ... */);

// Model binding
Route::get('/p/{post}', fn (Post $post) => $post);          // implicit by id
Route::get('/p/{post:slug}', fn (Post $post) => $post);     // custom key
Route::get('/p/{post}/c/{comment}', $h)->scopeBindings();   // scoped
Route::model('user', User::class);                          // explicit (provider)
Route::bind('user', fn ($v) => User::firstWhere('name', $v));

// Resources
Route::resource('photos', Ctrl::class)->only([...])->except([...]);
Route::apiResource('photos', Ctrl::class);
Route::resource('posts.comments', Ctrl::class)->shallow()->scoped();

// Convenience
Route::redirect('/a', '/b', 301);
Route::view('/about', 'pages.about', ['k' => 'v']);
Route::fallback(fn () => abort(404));

// Rate limiting
Route::middleware('throttle:60,1')->group(...);
RateLimiter::for('api', fn (Request $r) => Limit::perMinute(100)->by($r->user()?->id ?: $r->ip()));

// Signed URLs
URL::signedRoute('name', [...]); URL::temporarySignedRoute('name', now()->addHour(), [...]);
Route::get('/x', $h)->middleware('signed');

// Current route
Route::currentRouteName(); $request->routeIs('admin.*'); $request->route('post');
```

```bash
# Artisan
php artisan route:list -v --except-vendor
php artisan route:list --name=user --method=POST
php artisan route:cache        # production (no closure routes!)
php artisan route:clear
php artisan install:api        # scaffold api.php + Sanctum (L11/12)
php artisan make:controller PhotoController --api --model=Photo
```

---

## 🧪 Mini Exercises

1. **CRUD + constraints.** Build a `posts` resource using `Route::resource` pointed at a `PostController`, but expose only `index`, `show`, `store`, and `destroy`. Constrain the `{post}` parameter to numeric IDs and rename the `index` route to `posts.list`. Verify with `php artisan route:list`.

2. **Scoped nested binding.** Create routes for `/projects/{project}/tasks/{task:slug}` so a task is resolved by slug *and* scoped to its project. Add a route that returns 404 if you pass a task slug that belongs to a different project. Prove the scoping works.

3. **Tiered rate limiting.** Define a named rate limiter `reports` that allows 30 requests/minute for authenticated users (keyed by user id) and 5/minute for guests (keyed by IP), and returns a JSON `{"error":"rate limited"}` body with a 429 status when exceeded. Attach it to a `GET /reports` route.

4. **Signed unsubscribe link.** Add a named route `unsubscribe` that takes a `{user}` model binding, protected by a *temporary* signed URL valid for 24 hours. Write the line that generates the link and confirm the route 403s when the signature is missing or expired.

5. **Active-nav helper.** Given an admin route group prefixed `admin` with name prefix `admin.`, write a Blade snippet that adds an `active` class to the "Users" link only when the current route name matches `admin.users.*`.
