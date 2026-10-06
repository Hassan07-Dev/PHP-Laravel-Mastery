# Laravel Interview Questions & Answers (PHP 8.4 / Laravel 12)

A categorized bank of 60+ Laravel interview questions with concise, correct answers and short, runnable code. Targets **Laravel 12** and **PHP 8.4**, with notes where **Laravel 10/11** or **PHP 8.1–8.3** differ. Use the question-then-answer format to rehearse out loud.

**What you'll learn**

- How the Laravel request lifecycle, service container, providers, facades, and contracts actually work.
- Routing, middleware ordering, groups, and route model binding.
- Eloquent in depth: relationships, the N+1 problem, scopes, casts, soft deletes, chunking, and lazy collections.
- Validation, form requests, authentication (Sanctum vs Passport), and authorization (gates vs policies).
- Collections, events vs jobs vs queues, caching, sessions, testing, and Artisan.
- Performance, security, Blade, API resources, and how to answer "how would you design…" questions.

---

## How to use this bank

Each entry is **Q → A**. Read the question, answer aloud, then check. The "under the hood" answers are what separate strong candidates from memorizers. Jargon is defined on first use.

---

## 1. Lifecycle & Service Container

### Q1. Walk me through the Laravel request lifecycle.

**A.** Every request flows through these stages:

1. **`public/index.php`** — the single entry point. It loads Composer's autoloader and `bootstrap/app.php`.
2. **Bootstrap** — `bootstrap/app.php` creates and returns the **Application** (the service container).
3. **Kernel** — the framework's HTTP kernel (`Illuminate\Foundation\Http\Kernel`, used internally — there is **no** `app/Http/Kernel.php` in Laravel 11/12) receives the request. The kernel bootstraps the framework: loads environment, config, registers facades, and **registers + boots all service providers**.
4. **Global middleware** — the request passes through the global middleware stack (the "onion").
5. **Routing** — the router matches the request to a route, runs route/group middleware, resolves the controller.
6. **Controller / action** — runs your business logic, returns a `Response`.
7. **Response** — travels back out through middleware (in reverse), and `index.php` sends it to the browser.

```php
// public/index.php (Laravel 12, simplified)
use Illuminate\Http\Request;

require __DIR__.'/../vendor/autoload.php';
$app = require_once __DIR__.'/../bootstrap/app.php';
$app->handleRequest(Request::capture());
```

> **Laravel 11/12 note:** the kernel is now configured fluently in `bootstrap/app.php` via `Application::configure(...)->withMiddleware(...)->withRouting(...)`. Laravel 10 used dedicated `app/Http/Kernel.php` and `app/Console/Kernel.php` classes plus five route files.

### Q2. What is the service container?

**A.** A **container** is an object that manages **class dependencies** and performs **dependency injection (DI)** — it knows how to build objects and supply their constructor arguments automatically. It is the heart of the framework.

```php
// Resolving out of the container
$service = app(PaymentGateway::class);   // or resolve(PaymentGateway::class)
```

### Q3. What is dependency injection and why use it?

**A.** Instead of a class creating its own dependencies (`new Mailer()`), they are **passed in** (usually via the constructor). This decouples classes, makes them testable (you can inject mocks), and lets the container swap implementations.

```php
class OrderController
{
    public function __construct(private PaymentGateway $gateway) {}
}
// Laravel auto-resolves PaymentGateway when constructing the controller.
```

### Q4. `bind` vs `singleton` vs `instance` vs `scoped`?

**A.**

- **`bind`** — registers a resolver; a **new instance every time** it's resolved.
- **`singleton`** — resolves **once**, then returns the same instance for the lifetime of the request.
- **`instance`** — register an **already-created** object.
- **`scoped`** — singleton **per request/job lifecycle** (reset at the start of a new Octane request or queue-worker job). Added in Laravel 9 (alongside Octane).

```php
$this->app->bind(Reporter::class, fn () => new Reporter());
$this->app->singleton(Cache::class, fn () => new RedisCache());
$this->app->instance(Config::class, $config);
$this->app->scoped(RequestContext::class, fn () => new RequestContext());
```

### Q5. How does automatic resolution / autowiring work under the hood?

**A.** When asked for a class, the container uses **PHP Reflection** to inspect the constructor's typed parameters. For each parameter it recursively resolves the type from the container; scalars without defaults must be bound explicitly or passed contextually. This recursion is why you rarely call `new` yourself.

```php
$this->app->when(PhotoController::class)
    ->needs(Filesystem::class)
    ->give(fn () => Storage::disk('s3'));
```

### Q6. `register()` vs `boot()` in a service provider?

**A.** A **service provider** is the central place to configure/bind things into the container.

- **`register()`** — only **bind** things into the container. Do **not** use other services here; they may not be registered yet.
- **`boot()`** — runs **after all providers are registered**, so every binding is available. Put event listeners, view composers, route/macro/policy registration, validators, etc. here.

```php
class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, StripeGateway::class);
    }

    public function boot(): void
    {
        Gate::define('refund', fn ($user) => $user->isAdmin());
    }
}
```

### Q7. How do facades work under the hood?

**A.** A **facade** is a static-looking proxy to an object resolved from the container. Each facade extends `Illuminate\Support\Facades\Facade` and implements `getFacadeAccessor()`, returning a container binding key. Calls like `Cache::get()` hit `Facade::__callStatic()`, which resolves the underlying instance and forwards the call. So `Cache::get()` ≈ `app('cache')->get()`. Because the real object comes from the container, facades are still mockable in tests via `Cache::shouldReceive(...)`.

```php
// These are equivalent:
Cache::put('k', 'v', 60);
app('cache')->put('k', 'v', 60);
```

### Q8. What are contracts, and facade vs contract vs helper?

**A.** **Contracts** are the framework's **interfaces** (e.g. `Illuminate\Contracts\Cache\Repository`). Code to the contract for loose coupling and easy swapping.

- **Facade** — convenient static access; great in app code, slightly harder to reason about for DI.
- **Contract** — type-hint in constructors for explicit, testable dependencies.
- **Helper** — terse global functions (`cache()`, `view()`, `now()`).

```php
public function __construct(private \Illuminate\Contracts\Cache\Repository $cache) {}
```

### Q9. What is deferred provider loading?

**A.** A provider implementing `DeferrableProvider` and listing `provides()` is **only loaded when one of its bindings is actually resolved**, saving boot time. Use for providers that aren't needed on every request.

```php
class ReportServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void { $this->app->singleton(Report::class); }
    public function provides(): array { return [Report::class]; }
}
```

---

## 2. Routing & Middleware

### Q10. What is route model binding?

**A.** Laravel can automatically inject a model resolved from a route parameter.

- **Implicit binding** — type-hint a model; Laravel queries by primary key (404 if missing).
- **Custom key** — `{user:slug}` resolves by `slug`.
- **Explicit binding** — define resolution logic in a provider via `Route::model()` / `Route::bind()`.

```php
Route::get('/posts/{post}', fn (Post $post) => $post);          // by id
Route::get('/posts/{post:slug}', fn (Post $post) => $post);     // by slug
```

```php
// Explicit binding (boot of a provider)
Route::bind('post', fn ($value) => Post::where('slug', $value)->firstOrFail());
```

> Scoped bindings: `/users/{user}/posts/{post}` with `->scopeBindings()` ensures `post` belongs to `user`.

### Q11. What is middleware, and how is order determined?

**A.** **Middleware** filters HTTP requests entering/leaving the app (auth, CSRF, throttling). Order matters: the request passes top-to-bottom, the response bottom-to-top (the "onion").

- **Global** middleware runs first, in registration order.
- **Group** middleware (`web`, `api`) next.
- **Route** middleware last.
- Laravel sorts certain middleware via a **priority list** (`$middlewarePriority` in older versions; `withMiddleware(fn ($m) => $m->priority([...]))` in L11/12) so e.g. `SubstituteBindings` runs after auth when needed.

```php
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware(['auth', 'verified', 'can:access-admin']);
```

### Q12. Middleware: `handle` "before" vs "after" logic?

**A.** Code **before** `$next($request)` runs on the way in; code **after** runs on the way out.

```php
public function handle(Request $request, Closure $next): Response
{
    // before
    $response = $next($request);
    // after — can inspect/modify response
    $response->headers->set('X-Trace', Str::uuid());
    return $response;
}
```

### Q13. Route groups — what can you share?

**A.** Groups share `prefix`, `middleware`, `name` prefix, `controller`, `domain`, and namespace.

```php
Route::middleware('auth')->prefix('admin')->name('admin.')->group(function () {
    Route::get('/users', [UserController::class, 'index'])->name('users'); // admin.users
});
```

### Q14. Middleware aliases, groups, and parameters?

**A.** Register aliases and pass parameters with a colon.

```php
// bootstrap/app.php (Laravel 11/12)
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias(['role' => EnsureUserHasRole::class]);
})
```

```php
Route::get('/reports', fn () => ...)->middleware('role:admin'); // 'admin' => $role
```

### Q15. `throttle` middleware — how does rate limiting work?

**A.** Define named limiters (usually in a provider or `bootstrap/app.php`) returning a `Limit`; apply via `throttle:name`. Backed by the cache.

```php
RateLimiter::for('api', fn (Request $r) =>
    Limit::perMinute(60)->by($r->user()?->id ?: $r->ip()));
```

---

## 3. Eloquent ORM

### Q16. Eloquent vs the Query Builder — when to use which?

**A.** **Query Builder** is a fluent SQL builder returning `stdClass`/arrays — lightweight, great for reports/aggregates and bulk operations. **Eloquent** is an Active Record ORM returning model objects with relationships, events, casts, and accessors — best for domain logic and CRUD. Eloquent is built on top of the Query Builder, so you can drop down anytime.

```php
DB::table('orders')->where('status', 'paid')->sum('total');   // Query Builder
Order::where('status', 'paid')->get();                        // Eloquent
```

### Q17. Explain the N+1 problem and how to fix it.

**A.** **N+1** means one query to fetch parents, then **one extra query per parent** to fetch a relation (1 + N). **Eager loading** with `with()` collapses this to a constant number of queries (typically 2) using a `WHERE IN`.

```php
// BAD: 1 + N queries
foreach (Post::all() as $post) {
    echo $post->author->name; // queries authors one-by-one
}

// GOOD: 2 queries
foreach (Post::with('author')->get() as $post) {
    echo $post->author->name;
}
```

To catch it in dev, enable strict mode: `Model::preventLazyLoading(! app()->isProduction());` — accessing an un-eager-loaded relation throws `LazyLoadingViolationException`.

### Q18. `with` vs `load` vs `withCount` vs `loadMissing`?

**A.**

- `with('rel')` — eager load **at query time**.
- `load('rel')` — **lazy eager load** on an already-fetched model/collection.
- `loadMissing('rel')` — load only if not already loaded.
- `withCount('rel')` — adds a `rel_count` attribute without loading rows.

```php
$posts = Post::withCount('comments')->get();
echo $posts->first()->comments_count;   // e.g. 12
$posts->load('author');                 // add author later
```

### Q19. What relationship types does Eloquent support?

**A.** `hasOne`, `hasMany`, `belongsTo`, `belongsToMany` (pivot), `hasManyThrough`/`hasOneThrough`, and polymorphic: `morphOne`, `morphMany`, `morphToMany`, `morphTo`.

```php
class User extends Model {
    public function posts(): HasMany { return $this->hasMany(Post::class); }
    public function roles(): BelongsToMany { return $this->belongsToMany(Role::class); }
}
```

### Q20. How do you work with pivot table data?

**A.** Use `withPivot`, `withTimestamps`, and access via `$model->pivot`. For richer behavior, use a custom pivot model with `using()`.

```php
$this->belongsToMany(Role::class)->withPivot('expires_at')->withTimestamps();
$user->roles->first()->pivot->expires_at;
$user->roles()->attach($roleId, ['expires_at' => now()->addYear()]);
$user->roles()->sync([1, 2, 3]);   // replace set
$user->roles()->toggle([1, 4]);
```

### Q21. Local vs global scopes?

**A.** A **scope** encapsulates reusable query constraints.

- **Local scope** — a `scopeXxx()` method called as `Model::xxx()`.
- **Global scope** — applied automatically to **every** query for the model (e.g. multi-tenancy, soft deletes).

```php
class Post extends Model {
    public function scopePublished($query) { return $query->where('published', true); }
}
Post::published()->get();
```

```php
// Global scope via attribute (Laravel 11/12)
#[ScopedBy([TenantScope::class])]
class Invoice extends Model {}
Invoice::withoutGlobalScope(TenantScope::class)->get(); // bypass when needed
```

### Q22. Mass assignment — what is it and why the guard?

**A.** **Mass assignment** is filling many attributes at once from request input (`Model::create($request->all())`). Without protection, a malicious user could set fields like `is_admin`. Eloquent guards with **`$fillable`** (allowlist) or **`$guarded`** (denylist).

```php
class User extends Model {
    protected $fillable = ['name', 'email', 'password'];
}
User::create($request->only('name', 'email', 'password'));
```

> Prefer `$fillable`. `protected $guarded = [];` disables protection entirely — only safe with validated, explicit data.

### Q23. Soft deletes — how do they work?

**A.** The `SoftDeletes` trait sets a `deleted_at` timestamp instead of removing the row and adds a global scope to exclude "deleted" records.

```php
use Illuminate\Database\Eloquent\SoftDeletes;
class Post extends Model { use SoftDeletes; } // needs nullable deleted_at column

$post->delete();                       // soft delete
Post::withTrashed()->get();            // include deleted
Post::onlyTrashed()->get();            // only deleted
$post->restore();
$post->forceDelete();                  // permanent
```

### Q24. Accessors, mutators, and casts — what's the difference?

**A.**

- **Accessor** — transform an attribute when **reading**.
- **Mutator** — transform when **writing**.
- **Casts** — declarative type conversion (`int`, `bool`, `datetime`, `array`, `encrypted`, enums, custom).

```php
// Laravel 9+ unified accessor/mutator
protected function name(): Attribute
{
    return Attribute::make(
        get: fn ($value) => ucfirst($value),
        set: fn ($value) => strtolower($value),
    );
}

protected function casts(): array
{
    return [
        'is_admin' => 'boolean',
        'meta'     => 'array',
        'status'   => OrderStatus::class,   // enum cast
        'options'  => AsCollection::class,
    ];
}
```

> **Laravel 11/12 note:** prefer the `casts()` **method** over the `$casts` property — it supports parameterized casts and is the current idiom.

### Q25. How do you process large datasets without exhausting memory?

**A.** Don't load everything into memory.

- **`chunk` / `chunkById`** — process N rows at a time. Use `chunkById` when mutating rows in the loop (stable cursor).
- **`cursor`** — one model at a time via a generator, but the underlying PDO buffer still holds the result set.
- **`lazy` / `lazyById`** — returns a **`LazyCollection`** that chunks under the hood and yields models lazily — best of both worlds.

```php
User::where('active', true)->chunkById(500, function ($users) {
    foreach ($users as $user) { /* ... */ }
});

User::lazy()->each(fn ($user) => /* ... */);   // LazyCollection
```

### Q26. What is a LazyCollection?

**A.** A collection backed by **PHP generators** that keeps only one item in memory at a time, enabling work on huge or streamed datasets with the familiar collection API.

```php
LazyCollection::make(fn () => yield from readHugeFile('log.txt'))
    ->filter(fn ($line) => str_contains($line, 'ERROR'))
    ->take(100)
    ->each(fn ($line) => Log::warning($line));
```

### Q27. `firstOrCreate` vs `firstOrNew` vs `updateOrCreate`?

**A.**

- `firstOrNew` — find or **instantiate** (not saved).
- `firstOrCreate` — find or **insert**.
- `updateOrCreate` — find and update, or insert.

```php
User::updateOrCreate(['email' => $email], ['name' => $name]);
```

### Q28. What are model events / observers?

**A.** Eloquent fires events (`creating`, `created`, `updating`, `saved`, `deleting`, `deleted`, `restored`, etc.). Centralize handlers in an **observer**.

```php
class UserObserver {
    public function creating(User $user): void { $user->uuid = (string) Str::uuid(); }
}
// Laravel 11/12
#[ObservedBy([UserObserver::class])]
class User extends Model {}
```

> Gotcha: `saveQuietly()`, `updateQuietly()`, and bulk operations (`Model::query()->update()`) do **not** fire model events.

---

## 4. Validation & Form Requests

### Q29. Where should validation live, and what is a Form Request?

**A.** A **Form Request** is a dedicated class encapsulating authorization + validation rules, keeping controllers thin. Laravel auto-validates it on injection and redirects back with errors (web) or returns 422 JSON (API).

```php
class StorePostRequest extends FormRequest
{
    public function authorize(): bool { return $this->user()->can('create', Post::class); }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required', 'string'],
            'tags'  => ['array'],
            'tags.*' => ['string', 'distinct'],
        ];
    }
}

public function store(StorePostRequest $request) {
    $post = Post::create($request->validated());
}
```

### Q30. Inline validation and accessing validated data?

**A.**

```php
$validated = $request->validate([
    'email' => ['required', 'email', Rule::unique('users')->ignore($user)],
]);
$data = $request->safe()->only(['email']);   // only validated subset
```

### Q31. How do you write a custom validation rule?

**A.** Implement `ValidationRule` (Laravel 10+).

```php
class Uppercase implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (strtoupper($value) !== $value) {
            $fail("The {$attribute} must be uppercase.");
        }
    }
}
// usage: 'code' => [new Uppercase]
```

### Q32. `sometimes`, `bail`, `nullable`, conditional rules?

**A.**

- `nullable` — allow null.
- `bail` — stop on first failure for that field.
- `sometimes` — validate only if present; `$validator->sometimes()` adds conditional rules.
- `required_if`, `prohibited_unless`, etc. for cross-field logic.

```php
'discount' => ['nullable', 'numeric', 'required_if:type,promo'],
```

---

## 5. Auth & Authorization

### Q33. Authentication vs authorization?

**A.** **Authentication** = *who are you?* (login). **Authorization** = *what may you do?* (permissions). Auth runs first.

### Q34. What is a guard? Web vs API?

**A.** A **guard** defines how users are authenticated per request. `web` uses **session + cookies** (stateful). `api`/`sanctum` uses **tokens** (stateless). Config in `config/auth.php`.

```php
Auth::guard('web')->user();
$request->user();   // current guard's user
```

### Q35. Sanctum vs Passport — which and when?

**A.**

- **Sanctum** — lightweight. Two modes: (1) **API tokens** (personal access tokens stored in DB) and (2) **SPA auth** using the session/CSRF cookie for first-party SPAs and mobile. Default choice for most apps.
- **Passport** — full **OAuth2** server (authorization codes, client credentials, refresh tokens). Use only when you need true OAuth2 (third-party clients, token scopes across orgs).

```php
$token = $user->createToken('mobile', ['posts:read'])->plainTextToken;
// protect routes:
Route::middleware('auth:sanctum')->get('/me', fn (Request $r) => $r->user());
```

### Q36. Gates vs Policies?

**A.** Both authorize actions.

- **Gate** — a closure for a simple, standalone ability (not tied to a model).
- **Policy** — a class grouping authorization methods for a **specific model**. Methods map to abilities (`view`, `update`, `delete`).

```php
Gate::define('access-admin', fn (User $u) => $u->is_admin);

class PostPolicy {
    public function update(User $user, Post $post): bool {
        return $user->id === $post->user_id;
    }
}
// usage
$this->authorize('update', $post);      // controller
@can('update', $post) ... @endcan       // Blade
$user->can('access-admin');             // gate
```

### Q37. How does `before` work in a policy/gate?

**A.** A `before()` method runs **before** all other checks. Return `true` to grant everything (e.g. super-admin), `false` to deny all, `null` to fall through to the specific method.

```php
public function before(User $user): ?bool {
    return $user->isSuperAdmin() ? true : null;
}
```

### Q38. How does Laravel hash and verify passwords?

**A.** `Hash::make()` uses **bcrypt** by default (argon2id available), salted automatically. `Hash::check()` verifies. Never store plaintext; the `password` cast (`'hashed'`) auto-hashes on set.

```php
protected function casts(): array { return ['password' => 'hashed']; }
```

---

## 6. Collections

### Q39. What is a Collection and why prefer it over arrays?

**A.** A `Collection` is a fluent, chainable wrapper around arrays with 100+ expressive methods (`map`, `filter`, `reduce`, `groupBy`, `pluck`, `sum`). Eloquent returns collections. It improves readability and avoids manual loops.

```php
$names = collect($users)
    ->filter(fn ($u) => $u->active)
    ->sortBy('name')
    ->pluck('name')
    ->values();
```

### Q40. Useful collection methods to know?

**A.** `map`/`mapWithKeys`, `filter`/`reject`, `reduce`, `groupBy`, `keyBy`, `flatMap`, `partition`, `chunk`, `tap`, `when`, `unless`, `pipe`, `each`, `every`, `contains`, `firstWhere`, `whereIn`, `zip`, `collapse`, `flatten`, `unique`, `duplicates`.

```php
[$active, $inactive] = $users->partition(fn ($u) => $u->active);
$byRole = $users->groupBy('role');
```

### Q41. Collection vs LazyCollection?

**A.** `Collection` is eager (holds everything in memory). `LazyCollection` is lazy (generator-backed, one item at a time) for large/streamed data. Same API where possible.

### Q42. Higher-order messages?

**A.** Shorthand for common loops: `$users->each->markAsRead();`, `$users->sum->points;`.

---

## 7. Events vs Jobs vs Queues

### Q43. Events vs Jobs vs Queues — distinguish them.

**A.**

- **Event** — a "something happened" signal. **Listeners** react. Great for decoupling (one event, many listeners) and observer-style design.
- **Job** — a **unit of work**, especially work you want to run **on a queue** (asynchronously) like sending email or processing video.
- **Queue** — the **infrastructure/transport** (database, Redis, SQS) where queued jobs/listeners wait for a **worker** to process them.

A listener can itself be queued (`implements ShouldQueue`), blurring the line — events announce, jobs do, queues defer.

```php
event(new OrderShipped($order));      // fire event
ProcessPodcast::dispatch($podcast);   // queue a job
```

### Q44. How do you create and dispatch a queued job?

**A.**

```php
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;   // provides the static ::dispatch()
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;        // re-fetches models by id on the worker

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Podcast $podcast) {}
    public function handle(): void { /* heavy work */ }
}

ProcessPodcast::dispatch($podcast)->onQueue('media')->delay(now()->addMinutes(5));
```

> The four default traits matter: `Dispatchable` adds the static `::dispatch()`, `Queueable` adds `->onQueue()`/`->delay()`, `InteractsWithQueue` lets the job `release()`/`fail()` itself, and `SerializesModels` stores only the model **id** (re-fetching fresh on the worker, so don't rely on unsaved in-memory state).

Run a worker: `php artisan queue:work --queue=media,default`.

### Q45. `dispatch` vs `dispatchSync` vs `dispatchAfterResponse`?

**A.** `dispatch` queues it; `dispatchSync` runs immediately (no queue); `dispatchAfterResponse` runs after the HTTP response is sent (good for quick, fire-and-forget tasks without a worker).

### Q46. How do you handle failed jobs and retries?

**A.** Configure `$tries`, `$backoff`, `$timeout`; implement `failed(Throwable $e)`. Failed jobs go to the `failed_jobs` table (`php artisan queue:retry all`). Use `release()` to requeue, and **idempotency** so retries are safe.

```php
public int $tries = 5;
public array $backoff = [10, 30, 60];
public function failed(\Throwable $e): void { Log::error($e->getMessage()); }
```

### Q47. What are job batches and chains?

**A.** **Chain** runs jobs sequentially (stop on failure). **Batch** runs many jobs in parallel with completion callbacks and progress tracking.

```php
Bus::chain([new A, new B])->dispatch();
Bus::batch([new Import(1), new Import(2)])
    ->then(fn () => Log::info('done'))
    ->dispatch();
```

### Q48. Why use `ShouldQueue` on a listener vs a job?

**A.** Queue a listener when the reaction is naturally tied to an event and you want it async without a separate dispatch. Use a job when the work is invoked from multiple places or needs its own lifecycle (chains/batches/retries config).

---

## 8. Caching & Sessions

### Q49. How does caching work in Laravel?

**A.** A unified `Cache` API over drivers (redis, memcached, database, file, array). Use `remember` to compute-and-store on miss.

```php
$users = Cache::remember('users:active', now()->addMinutes(10), fn () =>
    User::where('active', true)->get()
);
Cache::forget('users:active');
Cache::tags(['posts'])->flush();   // tags: redis/memcached only
```

### Q50. `remember` vs `rememberForever` vs `flexible`?

**A.** `remember` expires after TTL; `rememberForever` never expires (evict manually). **`flexible`** (Laravel 11+) implements stale-while-revalidate: serve stale data while refreshing in the background.

```php
Cache::flexible('stats', [60, 120], fn () => expensiveStats());
```

### Q51. How do sessions work and which driver in production?

**A.** Sessions persist data across requests, keyed by a cookie. Drivers: `file`, `cookie`, `database`, `redis`. In production use `redis` or `database` (scales across servers; `file` doesn't in multi-server setups).

```php
session(['cart_id' => $id]);
$id = session('cart_id');
```

### Q52. Cache stampede — what is it and how to mitigate?

**A.** When a hot key expires, many requests recompute it simultaneously, hammering the DB. Mitigate with `Cache::lock()` (atomic locks), `flexible`/stale-while-revalidate, or staggered TTLs.

```php
Cache::lock('report:lock', 10)->block(5, function () {
    return Cache::remember('report', 600, fn () => buildReport());
});
```

---

## 9. Testing

### Q53. Feature vs Unit tests in Laravel?

**A.** **Unit** tests isolate a class/method (no framework boot ideally). **Feature** tests exercise an HTTP route/full stack through the container, DB, and middleware. In **Laravel 11/12 new apps scaffold with Pest by default** (you can opt into PHPUnit with `--pest=false` / `laravel new app --phpunit`); Pest runs on top of PHPUnit, so the assertions and Laravel test helpers are the same under either.

```php
// Pest (default in new Laravel 11/12 apps)
it('creates users', function () {
    $this->postJson('/api/users', ['name' => 'Ada', 'email' => 'a@b.com', 'password' => 'secret123'])
        ->assertCreated();
    $this->assertDatabaseHas('users', ['email' => 'a@b.com']);
});
```

```php
// The PHPUnit equivalent (still fully supported):
public function test_users_can_be_created(): void
{
    $response = $this->postJson('/api/users', ['name' => 'Ada', 'email' => 'a@b.com', 'password' => 'secret123']);
    $response->assertCreated();
    $this->assertDatabaseHas('users', ['email' => 'a@b.com']);
}
```

### Q54. How do you reset the database between tests?

**A.** Use the `RefreshDatabase` trait (migrates once, wraps each test in a transaction and rolls back). Use **factories** for test data.

```php
use RefreshDatabase;
User::factory()->count(3)->create();
```

### Q55. How do you fake external services?

**A.** Laravel ships fakes that swap real implementations and add assertions.

```php
Mail::fake();    Queue::fake();    Event::fake();    Http::fake();    Storage::fake();
Bus::fake();     Notification::fake();

Mail::assertSent(WelcomeMail::class);
Queue::assertPushed(ProcessPodcast::class);
Http::assertSent(fn ($r) => $r->url() === 'https://api.test/x');
```

### Q56. How do you test time-dependent code?

**A.** `travel`/`freeze` time, then assert.

```php
$this->travelTo(now()->addDay(), function () { /* assertions at +1 day */ });
$this->freezeTime();
```

---

## 10. Artisan & Config/Env

### Q57. Artisan commands you should know?

**A.**

```bash
php artisan migrate              # run migrations
php artisan migrate:fresh --seed # drop, re-migrate, seed
php artisan make:model Post -mfc # model + migration + factory + controller
php artisan route:list           # inspect routes
php artisan tinker               # REPL
php artisan queue:work           # process jobs
php artisan schedule:work        # run the scheduler locally
php artisan db:seed
php artisan optimize             # cache config/routes/events/views
```

### Q58. How do you write a custom Artisan command?

**A.**

```php
class SendReminders extends Command
{
    protected $signature = 'reminders:send {--days=1}';
    protected $description = 'Send reminders';
    public function handle(): int
    {
        $this->info("Sending for {$this->option('days')} day(s)");
        return self::SUCCESS;
    }
}
```

### Q59. config vs env — and the BIG caching gotcha.

**A.** `env()` reads `.env`; `config()` reads cached config arrays. **Once you run `php artisan config:cache`, the `.env` file is no longer loaded at runtime, so any `env()` call outside `config/*.php` returns `null`.** Rule: **only call `env()` inside config files**; everywhere else call `config()`.

```php
// config/services.php
'stripe' => ['key' => env('STRIPE_KEY')],   // OK
// app code:
config('services.stripe.key');               // correct
env('STRIPE_KEY');                            // null after config:cache — BUG
```

Clear caches after deploy/config changes: `php artisan config:clear` (or `optimize:clear`).

### Q60. How does the scheduler work?

**A.** Define schedules (in `routes/console.php` for Laravel 11/12, or `app/Console/Kernel.php` for 10). A single server cron entry runs `schedule:run` every minute; Laravel decides what's due.

```php
// routes/console.php (Laravel 11/12)
Schedule::command('reminders:send')->dailyAt('07:00')->withoutOverlapping();
```

```bash
* * * * * cd /app && php artisan schedule:run >> /dev/null 2>&1
```

---

## 11. Performance & Optimization

### Q61. How do you optimize a Laravel app for production?

**A.**

1. **Eager load** to kill N+1; add DB indexes; select only needed columns.
2. `php artisan optimize` (config/route/event/view caches).
3. Use `composer install --no-dev --optimize-autoloader`.
4. Cache expensive reads (Redis); queue slow work.
5. Use `chunk`/`lazy` for big datasets; paginate APIs.
6. Consider **Octane** (Swoole/FrankenPHP) to keep the app booted in memory.
7. Use `Model::preventLazyLoading()` and DB query logging in dev to catch issues early.

### Q62. What is Laravel Octane and the main caveat?

**A.** Octane boots the app once and keeps it in memory across requests (huge speedup). Caveat: **state leaks** — singletons and static/global state persist between requests, so avoid storing request-specific data in singletons; use `scoped` bindings.

### Q63. `select` and chunking for performance?

**A.** Fetch only needed columns and avoid loading whole tables.

```php
User::select(['id', 'email'])->where('active', true)->cursor();
```

---

## 12. Security in Laravel

### Q64. What built-in protections does Laravel provide?

**A.**

- **CSRF** — `@csrf` token + the `ValidateCsrfToken` middleware (renamed from `VerifyCsrfToken` in Laravel 11; the old name is kept as an alias) for stateful POST/PUT/PATCH/DELETE. Exclude routes via `->withMiddleware(fn ($m) => $m->validateCsrfTokens(except: ['stripe/*']))` in `bootstrap/app.php`.
- **SQL injection** — query bindings/parameterization by default (don't interpolate user input into `whereRaw`).
- **XSS** — Blade `{{ }}` escapes output; only use `{!! !!}` for trusted HTML.
- **Mass assignment** — `$fillable`/`$guarded`.
- **Password hashing** — bcrypt/argon2.
- **Encryption** — `Crypt::encryptString()`, the `encrypted` cast (uses `APP_KEY`).
- **Signed URLs** — `URL::signedRoute()` to prevent tampering.

### Q65. How do you prevent SQL injection with raw queries?

**A.** Always use bindings, never string concatenation.

```php
DB::select('select * from users where email = ?', [$email]);          // safe
User::whereRaw('age > ?', [$min])->get();                            // safe
// NEVER: ->whereRaw("email = '$email'")  -- injectable
```

### Q66. `{{ }}` vs `{!! !!}` in Blade?

**A.** `{{ $x }}` escapes HTML entities (XSS-safe). `{!! $x !!}` outputs raw HTML — only for content you fully trust/sanitize.

### Q67. How do you handle authorization safely on APIs?

**A.** Use `auth:sanctum`, policies (`$this->authorize()`), token abilities/scopes (`$user->tokenCan('posts:write')`), rate limiting, and never trust client-supplied IDs without ownership checks.

---

## 13. Blade

### Q68. Key Blade features?

**A.** Template inheritance (`@extends`/`@section`/`@yield`), **components** (`<x-alert/>`), slots, `@props`, control structures (`@if`, `@foreach`, `@forelse`), `@auth`/`@guest`/`@can`, `@include`, `@once`, `@push`/`@stack`, and `@php`.

```blade
{{-- components/alert.blade.php --}}
@props(['type' => 'info'])
<div class="alert alert-{{ $type }}">{{ $slot }}</div>
```

```blade
<x-alert type="error">Something failed.</x-alert>
@forelse ($posts as $post)
    <p>{{ $post->title }}</p>
@empty
    <p>No posts.</p>
@endforelse
```

### Q69. How are Blade templates compiled (under the hood)?

**A.** Blade is **compiled to plain PHP** files cached in `storage/framework/views`. Directives become PHP (`@if` → `<?php if ...`). Compilation happens on first render or when the source changes, so there's no runtime parsing overhead afterward.

### Q70. Anonymous vs class-based components?

**A.** Anonymous = just a Blade file in `components/`. Class-based = a PHP class + view, used when you need logic, computed props, or dependency injection in the constructor.

---

## 14. API Resources

### Q71. What are API Resources and why use them?

**A.** **API Resources** are a transformation layer between Eloquent models and JSON responses — control exactly which fields/relationships are exposed, shape the payload, and add metadata. Prevents leaking internal columns.

```php
class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'    => $this->id,
            'name'  => $this->name,
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'admin' => $this->when($request->user()?->isAdmin(), $this->is_admin),
        ];
    }
}
return UserResource::collection(User::with('posts')->paginate());
```

### Q72. `whenLoaded`, `when`, and `additional` — purpose?

**A.** `whenLoaded` includes a relation only if eager-loaded (avoids N+1). `when` conditionally includes a field. `additional([...])` adds top-level metadata. Resource collections preserve pagination meta automatically.

---

## 15. "How would you design…" Questions

### Q73. How would you design a multi-tenant app?

**A.** Choose a strategy: **single DB + tenant_id column** (global scope auto-filters; simplest), **schema-per-tenant**, or **database-per-tenant** (strongest isolation). For column-based: a global scope + a `BelongsToTenant` trait setting `tenant_id` on `creating`, plus middleware that resolves the current tenant and binds it into the container (`scoped`). Cache and queue keys must be tenant-aware.

### Q74. How would you design a job that imports a 1M-row CSV?

**A.** Stream the file (don't load all rows), dispatch a **batch** of jobs each handling a chunk (e.g. 1k rows) via `chunkById`/`LazyCollection`, use `upsert()` for bulk writes, make jobs **idempotent**, set retries/backoff, track progress via the batch, and notify on completion. Run dedicated queue workers.

### Q75. How would you design a rate-limited public API?

**A.** Define named `RateLimiter` rules keyed by token/IP, apply `throttle:` middleware, return proper `429` + `Retry-After`, use Sanctum tokens with **abilities/scopes**, version routes (`/api/v1`), wrap responses in API Resources, and cache idempotent GETs.

### Q76. How would you keep controllers thin?

**A.** Push validation into **Form Requests**, business logic into **Service classes** or **Actions**, data shaping into **API Resources**, side effects into **Events/Jobs**, and query logic into **scopes/repositories**. Controller just orchestrates.

### Q77. How would you implement soft real-time notifications?

**A.** Use Laravel's **notification** system with the **broadcast** channel over **Reverb** (or Pusher) + Echo on the client; queue the broadcast so HTTP stays fast; also persist to the `database` channel for an inbox.

---

## ⚠️ Common Mistakes & Gotchas

1. **Calling `env()` outside config files.** After `php artisan config:cache`, `env()` returns `null` in app code. **Fix:** only use `env()` in `config/*.php`; read everything else via `config()`.

2. **N+1 queries.** Looping over models and touching relations without eager loading. **Fix:** `with()`/`load()`/`withCount()`; enable `Model::preventLazyLoading()` in dev to fail loudly.

3. **`protected $guarded = [];` with `$request->all()`.** Opens you to mass-assignment of `is_admin` and friends. **Fix:** use `$fillable`, or always pass `$request->validated()` / `->only([...])`.

4. **Bulk updates skip events.** `Post::where(...)->update([...])`, `saveQuietly()`, and `forceDelete` may bypass observers/`updated_at`. **Fix:** iterate when you need events, or move logic into the DB-level operation deliberately.

5. **Forgetting `chunkById` when mutating in `chunk`.** Plain `chunk` uses OFFSET; modifying rows inside shifts the result set and skips records. **Fix:** use `chunkById` (or `lazyById`) when the loop changes the filtered column.

6. **Stale caches after deploy.** Config/route/view caches not cleared → old values served. **Fix:** run `php artisan optimize:clear` (or rebuild with `optimize`) in your deploy pipeline.

7. **Logic in `register()` that uses other services.** They may not be bound yet. **Fix:** move it to `boot()`.

8. **Using `{!! !!}` on user input.** XSS hole. **Fix:** use `{{ }}`; sanitize before any raw output.

---

## ✅ Best Practices

- Type-hint dependencies (constructor injection); code to **contracts** where it aids testing.
- Keep controllers thin: Form Requests + Services/Actions + API Resources + Jobs/Events.
- Always eager load known relations; add DB indexes for `where`/`join`/`orderBy` columns.
- Use `config()` everywhere except inside `config/*.php`; cache config/routes in production.
- Validate everything; never mass-assign raw request input.
- Queue slow work; make jobs idempotent with sane `$tries`/`$backoff` and a `failed()` handler.
- Use `$fillable`, `casts()` method, enums for status fields, and observers for cross-cutting model logic.
- Prefer Sanctum unless you genuinely need OAuth2 (Passport).
- Write feature tests with `RefreshDatabase` + factories; fake external services.
- Enable `preventLazyLoading`, `preventSilentlyDiscardingAttributes`, and `preventAccessingMissingAttributes` in non-production via `Model::shouldBeStrict()`.

---

## 🎯 Interview Tips & Likely Questions

**Q. (Under the hood) How does the service container resolve a class with dependencies?**
A. It uses PHP **Reflection** to read the constructor's typed parameters and recursively resolves each from the container; singletons are cached, contextual bindings override defaults, and unresolvable scalars must be bound explicitly. This recursive autowiring is why DI "just works."

**Q. (Under the hood) How do facades stay testable despite being static?**
A. `Facade::__callStatic` forwards to a **container-resolved instance** (via `getFacadeAccessor`). `Cache::shouldReceive()` swaps that instance with a Mockery mock in the container, so the static call hits the mock.

**Q. Explain the N+1 problem and prove you can detect it in production code.**
A. Define it (1 + N queries), fix with `with()`, and mention `preventLazyLoading()` / Laravel Debugbar / `DB::listen()` to detect.

**Q. When would you choose a Job over an Event listener?**
A. Use a Job when the work is invoked from many places or needs its own lifecycle (chains, batches, retry config); use a (queued) listener when the reaction is naturally tied to a single domain event.

**Q. Sanctum vs Passport — pick and justify for a mobile app + first-party SPA.**
A. Sanctum: token mode for mobile, SPA cookie mode for the web app; lighter, no OAuth2 overhead. Passport only if third parties need OAuth2.

**Q. Gate vs Policy?**
A. Gate = simple standalone ability (closure); Policy = class of abilities for one model, auto-discovered, supports `before()`.

**Q. What breaks when you run `config:cache`?**
A. `.env` stops loading at runtime; `env()` outside config returns `null`. Always read via `config()`.

**Q. How does Octane change how you write code?**
A. The app stays booted in memory across requests, so avoid leaking request state into singletons/statics; use `scoped` bindings and reset stateful services.

**Q. register() vs boot()?**
A. `register` = bindings only; `boot` = everything else (it runs after all providers registered).

**Q. How do you safely process millions of rows?**
A. `chunkById`/`lazyById`/`LazyCollection`, `select` minimal columns, `upsert`, batched queued jobs, idempotency.

---

## 📋 Quick Reference / Cheat Sheet

```php
// Container
app(Foo::class); resolve(Foo::class);
$app->bind / singleton / scoped / instance(...);

// Routing
Route::get('/x/{model}', ...);           // implicit binding
Route::get('/x/{m:slug}', ...);          // custom key
->middleware(['auth','can:update,post'])->name('x');

// Eloquent
Model::with('rel')->withCount('rel')->get();   // eager + count
$m->load('rel'); $m->loadMissing('rel');
Model::published()->get();                     // scope
Model::withTrashed() / onlyTrashed() / restore() / forceDelete();
Model::chunkById(500, fn ($rows) => ...);      // big data
Model::lazy()->each(fn ($m) => ...);           // LazyCollection
Model::updateOrCreate([...], [...]);
protected function casts(): array { return ['x' => 'array']; }

// Validation
$request->validate([...]); $request->validated(); $request->safe();

// Auth
auth()->user(); $request->user();
$user->createToken('name', ['ability'])->plainTextToken;
Route::middleware('auth:sanctum');
$this->authorize('update', $post); @can('update', $post)

// Queues
Job::dispatch($x)->onQueue('q')->delay(now()->addMin(5));
Bus::chain([...])->dispatch(); Bus::batch([...])->then(...)->dispatch();

// Cache / Session
Cache::remember($k, $ttl, fn () => ...); Cache::flexible($k, [60,120], ...);
session(['k' => $v]); session('k');

// Testing
use RefreshDatabase; Model::factory()->create();
Mail::fake(); Queue::fake(); Http::fake();

// Artisan
php artisan make:model Post -mfc
php artisan migrate:fresh --seed
php artisan optimize / optimize:clear / route:list / tinker / queue:work
```

| Concept | Use |
|---|---|
| `bind` / `singleton` / `scoped` | new each time / once / once-per-request |
| `register()` / `boot()` | bind only / use services |
| `with` / `load` / `withCount` | eager / lazy-eager / count |
| `chunkById` / `lazy` | memory-safe iteration |
| `$fillable` / `$guarded` | mass-assignment control |
| Gate / Policy | ability closure / model policy class |
| Sanctum / Passport | tokens & SPA / full OAuth2 |
| Event / Job / Queue | announce / do / defer |
| `env()` / `config()` | config files only / everywhere |

---

## 🧪 Mini Exercises

1. **Container & providers:** Create a `ReportGenerator` interface with two implementations. Bind the default as a `singleton` in a provider's `register()`, and use a **contextual binding** so one controller gets the alternate implementation. Verify with `php artisan tinker`.

2. **N+1 hunt:** Build `Author hasMany Post` with factories seeding 50 authors and 500 posts. Write a route that prints each post's author name, observe the query count with `DB::listen()`, then fix it with eager loading and confirm it drops to 2 queries.

3. **Form Request + Policy:** Create `UpdatePostRequest` (authorize via a `PostPolicy::update`) and rules for `title`/`body`. Return the updated model through a `PostResource`, exposing the author only when eager-loaded.

4. **Queue + retries:** Write a `ChargeCustomer` job that is idempotent, has `$tries = 3` with exponential `$backoff`, logs failures in `failed()`, and dispatch a `Bus::batch` of 100 of them with a completion callback.

5. **Config gotcha:** Add a custom value to `config/services.php` from `.env`. Read it via `config()` in a controller. Then run `php artisan config:cache`, replace the call with `env()`, and observe the `null` — then fix it back to `config()`.
