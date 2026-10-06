# Caching & Sessions in Laravel 12

Caching and sessions are two of the most misunderstood features in Laravel — partly because they *look* similar (both store key/value data, both have pluggable "drivers"), but they solve fundamentally different problems. **Cache** is about *speed*: store the result of expensive work so you don't redo it. **Sessions** are about *identity and state*: remember who a user is and what they've done across stateless HTTP requests. This module unpacks both, the subtle traps between them, and how to choose the right tool.

## **What you'll learn**

- The mental model for *why* caching exists and when it actually helps (and hurts).
- Every important `Cache` facade method: `get`, `put`, `add`, `forever`, `remember`, `rememberForever`, `flexible`, `increment`/`decrement`, `has`, `missing`, `forget`, `flush`, `pull`, and friends.
- Cache drivers (`file`, `database`, `redis`, `memcached`, `array`, `null`), cache **tags**, **TTL**, and **atomic locks** (`Cache::lock`).
- The crucial difference between *application caching* and *bootstrap caching* (`route:cache`, `config:cache`, `view:cache`).
- How sessions work over stateless HTTP, the session drivers, and the full session API (`get`/`put`/`flash`/`reflash`/`regenerate`, CSRF).
- **Session security**: fixation, hijacking, and how Laravel defends against them.
- **Cache invalidation** strategies and how to avoid the classic "two hard problems in CS" trap.

---

## 1. Why Cache At All? The Core Idea

Every web request that hits a database, calls an external API, or renders a heavy view costs *time* and *resources*. If the answer doesn't change often, recomputing it on every request is waste. **Caching** stores the result of expensive work in fast storage (memory or local disk) so subsequent requests read the cheap copy instead of redoing the work.

The trade-off is **staleness**: a cached value can be out of date. Caching is therefore always a deliberate bet — "this data is read far more often than it changes, and a few seconds/minutes of staleness is acceptable." If that bet is wrong, caching causes bugs (users see old prices, deleted posts, wrong permissions).

> **Jargon:** *TTL (Time To Live)* is how long a cached value stays valid before it's considered expired and is recomputed. *Cache hit* = the value was found in cache. *Cache miss* = it wasn't, so you compute it fresh.

### Configuring the cache

Laravel's cache config lives in `config/cache.php`, driven by environment variables. The default store is set via `CACHE_STORE` (Laravel 11/12; in Laravel 10 and earlier this was `CACHE_DRIVER`).

```env
# .env
CACHE_STORE=redis        # Laravel 11/12 key. (Laravel 10: CACHE_DRIVER=redis)
CACHE_PREFIX=myapp_cache  # prefixes every key to avoid collisions on shared stores
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

```php
// config/cache.php (abridged)
return [
    'default' => env('CACHE_STORE', 'database'), // L11/12 default is 'database'
    'stores' => [
        'array'    => ['driver' => 'array', 'serialize' => false],
        'database' => ['driver' => 'database', 'connection' => null, 'table' => 'cache'],
        'file'     => ['driver' => 'file', 'path' => storage_path('framework/cache/data')],
        'redis'    => ['driver' => 'redis', 'connection' => 'cache'],
        'memcached'=> ['driver' => 'memcached', /* servers... */],
    ],
];
```

> **Version note:** In Laravel 11 and 12 the fresh-install default `CACHE_STORE` is `database` (it was `file` in Laravel 10). The default *session* driver also moved to `database` in Laravel 11+.

Because the default cache store is `database`, a fresh Laravel 12 app needs a `cache` table. It ships in the default `0001_01_01_000001_create_cache_table.php` migration; if it's missing, generate it:

```bash
php artisan make:cache-table   # Laravel 11/12 (older: php artisan cache:table)
php artisan migrate
```

### The cache drivers — and when to use each

| Driver | Storage | Speed | Shared across servers? | Use when |
|---|---|---|---|---|
| `array` | PHP array in memory, **per-request** | Fastest | No | **Tests** — wiped after the request, never persists |
| `file` | Files under `storage/framework/cache` | Slow-ish | No (local disk) | Single small server, dev |
| `database` | A `cache` table | Slow (DB round-trip) | Yes | No Redis available, low volume |
| `redis` | Redis server (in-memory) | Very fast | Yes | Production default, supports tags + locks |
| `memcached` | Memcached server | Very fast | Yes | Legacy / pure key-value caching |
| `null` | Nowhere — discards every write | N/A | N/A | **Tests** — turn caching off entirely (every read is a miss) |
| `dynamodb` | AWS DynamoDB table | Fast | Yes | Serverless / AWS-native; supports locks, **not** tags |

The `array` driver is special: it stores data in a PHP array that lives only for the current process, so it's perfect for tests where you want cache behavior without persistence between test cases. (Use `null` instead when you want caching effectively disabled — every read is a miss.)

A fresh Laravel 12 app uses **Pest** as the default test runner. Both the cache and session drivers are forced to `array` for tests via `phpunit.xml`, so state never leaks between test cases:

```php
// tests/Feature/CacheTest.php (Pest)
use Illuminate\Support\Facades\Cache;

it('caches the computed value', function () {
    $value = Cache::remember('answer', 60, fn () => 42);

    expect($value)->toBe(42)
        ->and(Cache::get('answer'))->toBe(42);
});
```

---

## 2. The `Cache` Facade — Reading and Writing

All examples use the facade; the `cache()` helper does the same thing more tersely (covered below).

### `put`, `get`, and the cheap defaults

```php
use Illuminate\Support\Facades\Cache;

// put($key, $value, $ttl) — $ttl in seconds, or a DateTime/Carbon, or null = forever-ish
Cache::put('user:42:name', 'Ada', 600); // expires in 600 seconds (10 min)
Cache::put('flag', true, now()->addHour()); // Carbon TTL also accepted

// get($key, $default) — returns $default (or null) on a miss
$name = Cache::get('user:42:name');          // 'Ada'
$missing = Cache::get('nope', 'fallback');   // 'fallback'
$lazy = Cache::get('nope', fn () => expensiveDefault()); // closure default, lazily run
// Output: 'Ada', then 'fallback', then the closure's return
```

> **Gotcha:** A *cached* value of `null` or `false` is indistinguishable from a miss via `get`. Use `Cache::has()` (which is true only when the key exists *and* is non-null) carefully, or store a sentinel. More on this in Gotchas.

### `add` — write only if absent (atomic)

```php
// add() returns true if the key did NOT already exist (and was written), false otherwise.
$wrote = Cache::add('once-per-hour-job', true, 3600);
if ($wrote) {
    dispatchTheJob(); // guaranteed to run at most once per hour, race-safe on redis/memcached
}
// Output: true on first call within the hour, false afterwards
```

`add` is **atomic** on Redis and Memcached (check-and-set in one operation), which makes it a lightweight lock for "do this once" semantics.

### `forever` — no expiry

```php
Cache::forever('app:settings', $settings);
// Stays until you forget() it or flush() — survives no TTL. Must be manually invalidated.
```

> Even `forever` items can be evicted by Redis/Memcached if memory pressure triggers the eviction policy. "Forever" means "no TTL," not "guaranteed present." Don't treat the cache as a source of truth.

### `remember` and `rememberForever` — the workhorses

This is the pattern you'll use 90% of the time. `remember` returns the cached value if present; otherwise it runs the closure, stores the result, and returns it ("cache-aside" / "read-through" pattern).

```php
$users = Cache::remember('users:active', 300, function () {
    return User::where('active', true)->get(); // only runs on a miss
});

// rememberForever — same, but no TTL
$config = Cache::rememberForever('site:config', fn () => SiteConfig::all());
```

There is also `Cache::flexible()` (Laravel 11+), which implements **stale-while-revalidate**: serve a slightly stale value instantly while refreshing in the background.

```php
// flexible($key, [$fresh, $stale], $callback)
// "fresh" for 5s; between 5s and 20s, return the stale value AND refresh after the response is sent
// (via a deferred function); after 20s it's a hard miss and the closure runs synchronously.
$value = Cache::flexible('weather', [5, 20], fn () => Weather::fetch());
```

> **Laravel 12 addition — `Cache::memo()`:** the `memo` driver memoizes resolved values *in memory for the current request/job*, so repeated reads of the same key within one execution don't re-hit Redis/the DB. It decorates another store and transparently forgets its in-memory copy on any write:
>
> ```php
> $a = Cache::memo()->get('key');        // hits the underlying store
> $b = Cache::memo()->get('key');        // returns the in-memory copy, no round-trip
> $c = Cache::memo('redis')->get('key'); // decorate a specific store
> ```

### `increment` / `decrement` — atomic counters

```php
Cache::put('visits', 0);
Cache::increment('visits');      // 1
Cache::increment('visits', 10);  // 11
Cache::decrement('visits', 3);   // 8
// Atomic on redis/memcached — safe under concurrency for things like rate counters.
// Output: 1, 11, 8
```

### `has`, `forget`, `pull`, `flush`

```php
Cache::has('visits');   // true  — key exists and is not null
Cache::forget('visits'); // true  — remove a single key
$val = Cache::pull('one-time-token', 'default'); // get + forget in one call
Cache::flush();          // ⚠️ wipes the ENTIRE store (all keys, all apps sharing it)
```

> **Danger:** `Cache::flush()` on a shared Redis instance clears *everything in that database* — including other apps and Laravel's own queue/session data if they share the connection. Prefer tags or prefixed `forget` over `flush`. Use a dedicated Redis DB number for cache.

### Picking a non-default store on the fly

```php
Cache::store('redis')->put('key', 'val', 60);
Cache::store('array')->get('key'); // explicitly target a store
```

---

## 3. The `cache()` Helper

The global `cache()` helper is a convenience wrapper:

```php
cache(['user:1' => 'Ada'], 600); // put: pass an array + TTL
$value = cache('user:1');         // get
$value = cache('missing', 'def'); // get with default
cache()->remember('k', 60, fn () => heavy()); // call any facade method via cache()
cache()->forget('user:1');
```

Use whichever reads cleaner in context. The facade is more discoverable (IDE autocomplete on methods); the helper is terser.

---

## 4. Cache Tags — Grouped Invalidation

Sometimes you want to invalidate a *group* of related keys at once — e.g., "all cached data for user 42." **Tags** let you label entries and flush by label.

```php
// Only supported on TAGGABLE stores: redis, memcached, array.
// NOT supported on file, database, or dynamodb drivers (they throw a BadMethodCallException).

Cache::tags(['users', 'user:42'])->put('profile', $profile, 600);
Cache::tags(['users', 'user:42'])->put('settings', $settings, 600);

// Read must specify the same tags:
$profile = Cache::tags(['user:42'])->get('profile');

// Flush everything tagged 'user:42' (both keys above, in one call):
Cache::tags(['user:42'])->flush();
// 'users'-tagged data for OTHER users is untouched.
```

> **Key fact for interviews:** Tags are *only* available on `redis`, `memcached`, and `array`. They are **not** available on the `file`, `database`, or `dynamodb` stores. If you tag on a non-taggable store you get a `BadMethodCallException`. Also, you must read with the *same* tag set you wrote with — a tagged item is not visible via a plain `Cache::get()`.

How it works under the hood (Redis): each tag maps to a namespace whose version is bumped on flush. Flushing a tag effectively orphans the old entries (they become unreachable and expire/are GC'd), rather than deleting each key individually — this is why tag flush is O(1)-ish regardless of how many keys were tagged.

---

## 5. Atomic Locks — `Cache::lock`

A **lock** prevents two processes from running the same critical section simultaneously (e.g., two queue workers processing the same record). Locks require a driver that implements the `LockProvider` contract. As of Laravel 12 the supported lock drivers are: `memcached`, `redis`, `dynamodb`, `database`, `file`, and `array`. (Note this list differs from cache *tags*, which exclude `file`, `database`, and `dynamodb`.) For a truly **distributed** lock across multiple app servers, use a centralized store such as `redis` — the `file` and `array` drivers only lock within a single server/process.

```php
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('process-order:1001', 10); // name, TTL=10s (auto-released after 10s)

if ($lock->get()) {                 // non-blocking: true if acquired, false if held elsewhere
    try {
        processOrder(1001);         // critical section — only one process at a time
    } finally {
        $lock->release();           // always release in finally
    }
}
```

The cleaner closure form auto-releases (even on exception):

```php
Cache::lock('process-order:1001', 10)->get(function () {
    processOrder(1001); // lock auto-released when the closure returns/throws
});
```

**Blocking** acquisition — wait up to N seconds for the lock:

```php
use Illuminate\Contracts\Cache\LockTimeoutException;

$lock = Cache::lock('report', 120);
try {
    $lock->block(5); // wait up to 5s; throws LockTimeoutException if still locked
    generateReport();
} catch (LockTimeoutException $e) {
    // someone else is generating it — back off
} finally {
    optional($lock)->release();
}
```

Cross-process release uses an **owner token**, so worker A cannot accidentally release worker B's lock:

```php
$lock = Cache::lock('migrate', 600);
$token = $lock->owner(); // store this token (e.g., pass to a queued job)

// Later, in a different process — re-instantiate the lock with the token, then release:
Cache::restoreLock('migrate', $token)->release();

// To force a release WITHOUT the owner token (e.g., a stuck lock), use forceRelease():
Cache::lock('migrate')->forceRelease();
```

> Locks are the basis of the `WithoutOverlapping` middleware for queued jobs and `Schedule::...->withoutOverlapping()` for the scheduler.

---

## 6. Application Cache vs Bootstrap Cache — Don't Confuse Them

This trips up *everyone*. There are two completely different things called "cache" in Laravel.

**A) Application cache** — the `Cache` facade above. Stores your data (query results, API responses). Cleared with `php artisan cache:clear`.

**B) Bootstrap/optimization caches** — compile framework files for faster boot. These have *nothing* to do with the `Cache` facade:

```bash
php artisan config:cache   # merges all config/*.php into one cached file
php artisan route:cache    # serializes all routes into one file (huge speedup at scale)
php artisan view:cache     # precompiles all Blade templates to PHP
php artisan event:cache    # caches event->listener mappings
php artisan optimize       # runs config + route + view + event caching together
php artisan optimize:clear # clears ALL of the above + the application cache
```

```bash
# Clearing individual bootstrap caches:
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

Key differences and traps:

- **Query/view "caching" (application)** stores *data/results* and is invalidated by TTL or by you. **Route/config caching (bootstrap)** stores *compiled framework state* and is invalidated by *re-running the command* after a deploy.
- `config:cache` **breaks `env()` calls outside config files.** Once config is cached, `env()` returns `null` everywhere except inside `config/*.php`. **Always read config via `config('services.x')`, never `env('X')`, in app code.** This is the #1 production "it works locally" bug.
- Run `config:cache`/`route:cache` in **production deploys**, never blindly in local dev (you'll forget to clear them after editing routes/config).

---

## 7. Sessions — Remembering Users Across Stateless HTTP

HTTP is **stateless**: each request is independent and carries no memory of the last. So how does a site remember you're logged in across page loads? **Sessions.**

The mechanism: the server generates a random **session ID**, sends it to the browser in a cookie (default name `laravel_session`), and stores the actual session *data* server-side keyed by that ID. On each request the browser sends the cookie back, the server looks up the data, and "remembers" you.

> **Cache vs Session — the distinction:** Cache is **global and shared** (one cached `users:active` list serves everyone). A session is **per-user, per-browser** (your cart is yours alone). Cache is disposable optimization; session data is user state you'd be sad to lose mid-checkout. Never store per-user state in the cache keyed only by a user ID if you actually mean "this browser session," and never store global shared data in the session.

### Session config and drivers

```env
SESSION_DRIVER=database   # L11/12 default (was 'file' in L10)
SESSION_LIFETIME=120      # minutes of inactivity before expiry
SESSION_ENCRYPT=false
SESSION_SECURE_COOKIE=true  # only send cookie over HTTPS (set true in prod!)
SESSION_SAME_SITE=lax       # CSRF mitigation: lax | strict | none
SESSION_HTTP_ONLY=true      # JS cannot read the cookie (XSS mitigation)
```

| Session driver | Where data lives | Shared across servers? | Notes |
|---|---|---|---|
| `file` | `storage/framework/sessions` | No | Simple; fine for one server |
| `cookie` | In an **encrypted cookie** client-side | N/A | No server storage; ~4KB limit; data leaves your control |
| `database` | A `sessions` table | Yes | L11/12 default; good for multi-server |
| `redis` | Redis | Yes | Fast multi-server; production favorite |
| `array` | Per-request array | No | Tests only — not persisted |

### Creating the database sessions table

```bash
php artisan make:session-table   # Laravel 11/12 (older: php artisan session:table)
php artisan migrate
```

The migration creates a `sessions` table roughly like:

```php
Schema::create('sessions', function (Blueprint $table) {
    $table->string('id')->primary();
    $table->foreignId('user_id')->nullable()->index();
    $table->string('ip_address', 45)->nullable();
    $table->text('user_agent')->nullable();
    $table->longText('payload');     // serialized + base64 session data
    $table->integer('last_activity')->index();
});
```

---

## 8. Reading and Writing Session Data

Two equivalent entry points: the `session()` helper and the `Request` object. Prefer injecting `Request` in controllers for testability.

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    // Writing
    $request->session()->put('cart.items', [101, 102]);
    session(['theme' => 'dark']);          // helper: pass an array to write

    // Reading
    $items = $request->session()->get('cart.items', []); // default [] on miss
    $theme = session('theme', 'light');     // helper: get with default

    // Dot notation works for nested data:
    session(['user.prefs.lang' => 'en']);
    $lang = session('user.prefs.lang');     // 'en'

    // Existence & retrieval helpers
    $request->session()->has('cart.items');     // true only if present AND not null
    $request->session()->exists('cart.items');  // true even if value is null
    $request->session()->missing('discount');   // inverse of exists

    $all  = $request->session()->all();          // entire session as array
    $one  = $request->session()->pull('flash_id', null); // get + forget

    // Removing
    $request->session()->forget('theme');        // remove one (or pass array)
    $request->session()->forget(['a', 'b']);
    $request->session()->flush();                // remove ALL session data

    // Incrementing (handy for counters in session)
    $request->session()->increment('page_views');
    $request->session()->increment('page_views', 5);
}
```

> `has()` vs `exists()`: `has` returns `false` if the key is present but `null`; `exists` returns `true` as long as the key is set, even when its value is `null`. Mirror of the cache `has` gotcha.

### Flash data — survive exactly one request

**Flash data** lives for the *next* request only, then auto-deletes. This is how "success" banners after a redirect work.

```php
// In the controller before a redirect:
$request->session()->flash('status', 'Profile updated!');
return redirect('/dashboard');
// On /dashboard, session('status') === 'Profile updated!'; on the request AFTER that, it's gone.

return redirect('/dashboard')->with('status', 'Profile updated!'); // shorthand: ->with() flashes
```

```blade
{{-- In a Blade view --}}
@if (session('status'))
    <div class="alert">{{ session('status') }}</div>
@endif
```

Controlling the flash lifetime:

```php
$request->session()->reflash();          // keep ALL flash data for one more request
$request->session()->keep(['status']);   // keep specific flash keys one more request
$request->session()->now('status', 'X'); // flash for the CURRENT request only (no redirect)
```

`reflash`/`keep` matter when a redirect chains through an intermediate request (e.g., redirect → middleware redirect → final page) and you'd otherwise lose the flash too early.

---

## 9. Session Security: Fixation, Hijacking, and CSRF

Sessions are a prime attack target because they *are* the user's authenticated identity. Three threats and Laravel's defenses:

### a) Session fixation → `regenerate()`

**Session fixation:** an attacker tricks a victim into using a session ID the attacker already knows; after the victim logs in, the attacker reuses that same ID and is now "logged in" as the victim.

**Defense:** issue a *new* session ID at any privilege change (login). Laravel's `Auth` does this for you on login, but if you build custom auth, call it explicitly:

```php
// Regenerate the session ID but KEEP the data — call on login/privilege change.
$request->session()->regenerate();

// Regenerate ID AND wipe all data — call on logout.
$request->session()->invalidate();
```

### b) Session hijacking → secure cookie flags

**Hijacking:** stealing the session cookie (via XSS, network sniffing). Defenses are cookie flags set in `config/session.php`:

- `'http_only' => true` — JavaScript (`document.cookie`) cannot read the cookie, blunting XSS theft.
- `'secure' => true` — cookie only sent over HTTPS, preventing sniffing on the wire.
- `'same_site' => 'lax'` — cookie not sent on most cross-site requests, mitigating CSRF.
- `'encrypt' => true` — encrypt session payload (especially important for the `cookie` driver, where data lives client-side).

### c) CSRF → the token

**CSRF (Cross-Site Request Forgery):** a malicious site makes your logged-in browser submit a request to your app using your cookies. Laravel defends with a per-session **CSRF token** that must accompany every state-changing request (`POST`/`PUT`/`PATCH`/`DELETE`). An attacker's site can't read your token (same-origin policy), so forged requests fail.

```blade
<form method="POST" action="/profile">
    @csrf  {{-- emits <input type="hidden" name="_token" value="..."> --}}
    ...
</form>
```

```php
$token = $request->session()->token(); // or csrf_token() helper
```

```blade
{{-- For JS/AJAX, expose it in a meta tag and send as the X-CSRF-TOKEN header --}}
<meta name="csrf-token" content="{{ csrf_token() }}">
```

> **Version note:** In Laravel 11/12, CSRF protection is registered via the middleware in `bootstrap/app.php` (`$middleware->validateCsrfTokens(except: [...])`). In Laravel 10 it was the `VerifyCsrfToken` middleware class in `app/Http/Middleware`. The `@csrf` directive and `csrf_token()` are unchanged.

### Where the cookie itself is configured

```env
SESSION_COOKIE=myapp_session   # cookie name
SESSION_DOMAIN=.example.com     # cookie domain (subdomain sharing)
SESSION_PATH=/
```

---

## 10. Cache Invalidation — Choosing a Strategy

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

Stale cache is the source of most cache-related bugs. Strategies, simplest first:

1. **TTL-only (time-based).** Let entries expire after N seconds; tolerate up to N seconds of staleness. Simplest, no invalidation logic. Good for dashboards, counts, "trending" lists.

2. **Write-through / event-based invalidation.** When the underlying data changes, forget the relevant cache keys. Use model events:

```php
// app/Models/Post.php
protected static function booted(): void
{
    static::saved(fn (Post $p) => Cache::forget("post:{$p->id}"));
    static::deleted(fn (Post $p) => Cache::forget("post:{$p->id}"));
}
```

3. **Tag-based bulk invalidation.** Tag related entries; flush the tag on change (Redis/Memcached only) — see §4. Good when one write should invalidate many derived keys.

4. **Versioned keys.** Bake a version/timestamp into the key so old keys are simply never read again (they expire on their own). No explicit delete needed:

```php
$key = "post:{$post->id}:v{$post->updated_at->timestamp}";
$html = Cache::remember($key, 3600, fn () => render($post));
// When the post is updated, updated_at changes → a brand-new key → guaranteed fresh.
```

**Choosing:** Use TTL when staleness is acceptable and writes are frequent (invalidation churn isn't worth it). Use event/versioned invalidation when correctness matters (prices, permissions, balances). Combine TTL as a *safety net* with event invalidation as the *primary* mechanism — so even a missed invalidation self-heals within the TTL window.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `env()` in app code after `config:cache`.** Once you cache config, `env()` returns `null` outside `config/*.php`, so `env('STRIPE_KEY')` in a service class becomes `null` in production and the integration silently breaks.
   **Fix:** Reference config only: define it in `config/services.php` and read `config('services.stripe.key')` everywhere in app code.

2. **Calling `Cache::tags()` on `file`, `database`, or `dynamodb` stores.** Tags throw `BadMethodCallException` on non-taggable drivers; also, tagged writes are invisible to plain `Cache::get()`.
   **Fix:** Use `redis`, `memcached`, or `array` for tags; always read with the same tag set you wrote with.

3. **`Cache::has()` / `session()->has()` returning false for stored `null`/`false`.** `has` is false when the value is `null`, so a cached `null` looks like a miss.
   **Fix:** Store a sentinel (e.g., wrap in an array), or use `session()->exists()`. Note that `false` *is* a distinguishable value — only `null` reads as a miss.

4. **Assuming `Cache::remember` caches a `null` returned by the closure — it does not.** If the closure returns `null`, `remember` treats it like a miss and re-runs the closure on *every* subsequent request, defeating the cache (the classic "cache stampede on missing rows" bug). The same applies to `rememberForever` and `Cache::get` (a stored `null` is indistinguishable from a miss).
   **Fix:** To cache a *negative* result (e.g., "user not found"), return a non-null sentinel such as `false` or an empty collection from the closure, then translate it back in your code.

5. **`Cache::flush()` nuking a shared Redis/Memcached instance.** It wipes *every* entry in the store (and note: `flush` ignores your configured `CACHE_PREFIX`) — other apps, sessions, queues sharing the connection included.
   **Fix:** Use tags or targeted `forget`; isolate cache on its own Redis DB index (`'cache'` connection in `config/database.php`); reserve `flush` for `cache:clear` in deploys.

6. **Forgetting `regenerate()` on custom login → session fixation.** If you roll your own auth and reuse the pre-login session ID, you're vulnerable.
   **Fix:** Call `$request->session()->regenerate()` on login and `invalidate()` on logout (Laravel's built-in `Auth`, the starter kits, and Fortify already do this).

7. **Acquiring a lock but not releasing it on exception.** A thrown exception inside the critical section leaves the lock held until its TTL expires, stalling other workers.
   **Fix:** Use the closure form `Cache::lock(...)->get(fn () => ...)` (auto-releases), or always `release()` in a `finally`.

8. **Storing huge per-user data in the `cookie` session driver.** Cookies cap at ~4KB and the data travels on *every* request, and lives client-side.
   **Fix:** Use `redis`/`database` session drivers for anything non-trivial; keep cookie sessions tiny.

9. **Running `config:cache` / `route:cache` in local dev and forgetting to clear them.** Edits to `.env`, `config/*.php`, or routes then appear to "have no effect" until you run `optimize:clear`.
   **Fix:** Treat bootstrap caching as a *deploy* step. In local dev, leave it off (or always pair it with `php artisan optimize:clear`).

---

## ✅ Best Practices

- **Read config via `config()`, never `env()`** outside of `config/*.php` — so `config:cache` stays safe.
- **Default to `redis`** for cache *and* sessions in production: fast, shared across servers, supports tags and locks.
- **Isolate stores:** put cache on a dedicated Redis DB so `cache:clear`/`flush` can't clobber sessions/queues.
- **Prefer `Cache::remember`** over manual `has`/`get`/`put` — it's atomic-ish, readable, and caches the computed value for you.
- **Combine TTL with event-based invalidation:** event invalidation for correctness, a TTL as a self-healing safety net.
- **Use the `array` driver in tests** (`phpunit.xml`: `CACHE_STORE=array`, `SESSION_DRIVER=array`) so state never leaks between tests.
- **Always set `SESSION_SECURE_COOKIE=true`, `http_only`, and an appropriate `same_site`** in production.
- **Call `regenerate()` on privilege escalation** and `invalidate()` on logout.
- **Release locks in `finally`** (or use the closure form); set a sane lock TTL as a deadlock backstop.
- **Cache the *expensive* thing, not the cheap thing** — caching a trivial query can be slower than just running it (cache round-trip + serialization overhead).

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between cache and session, and when would you use each?**
A. Cache is *shared, global, disposable* optimization — store expensive-to-compute data that's read more than written (a few seconds of staleness is fine). Session is *per-user, per-browser* state tied to identity (cart, "remember me", flash messages) that you'd hate to lose. Cache can vanish without correctness impact; session loss logs users out.

**Q2. How does `Cache::remember` work, and how is it different from `get` + `put`?**
A. `remember($key, $ttl, $cb)` returns the cached value on a hit; on a miss it runs the closure, stores the result for `$ttl`, and returns it (cache-aside pattern). It encapsulates the check/compute/store dance in one call. **Watch out:** if the closure returns `null`, `remember` treats it as a miss and re-runs the closure on every subsequent request — so to cache a "not found" result you must return a non-null sentinel (e.g. `false`).

**Q3. Which cache drivers support tags, and why not all of them?**
A. Only `redis`, `memcached`, and `array`. Tags need a mechanism to group keys and bump a tag "version" atomically (Redis/Memcached have the right primitives). `file`, `database`, and `dynamodb` have no efficient way to track tag membership, so Laravel disallows tags there — calling `tags()` throws `BadMethodCallException`.

**Q4. (Under the hood) How does tag flushing actually invalidate entries?**
A. Each tag is associated with a namespace identifier stored in the cache. Tagged keys are stored under a composite namespace derived from those tag IDs. Flushing a tag generates a *new* namespace ID for it, so all previously written keys (built from the old ID) become unreachable and are eventually evicted/expired — rather than the cache deleting each member key one by one. That's why a tag flush is cheap regardless of member count.

**Q5. How does a session persist across stateless HTTP requests?**
A. The server stores session *data* server-side (file/db/redis) under a random session ID, and sends only that ID to the browser in a cookie. The browser returns the cookie each request; Laravel reads the ID, loads the data, and exposes it via `session()`. The cookie carries the *pointer*, not the data (except the `cookie` driver, which encrypts the data into the cookie itself).

**Q6. What is session fixation and how does Laravel prevent it?**
A. Fixation is when an attacker fixes a known session ID onto a victim, then reuses it post-login to impersonate them. Defense: regenerate the session ID at login so the pre-login ID becomes useless. Laravel's `Auth` calls `regenerate()` automatically; custom auth must call `$request->session()->regenerate()` itself.

**Q7. Explain `config:cache` vs `cache:clear`. Are they related?**
A. No — different subsystems sharing the word "cache." `config:cache` compiles all config files into one cached PHP file to speed boot (a *bootstrap* optimization); `cache:clear` empties the *application* cache (`Cache` facade data). `optimize:clear` clears both kinds.

**Q8. How do `Cache::lock` atomic locks work and what backs them?**
A. They use a driver's atomic set-if-not-exists primitive (Redis `SET NX PX`, etc.) to claim a named key with a TTL and an owner token. `get()` is non-blocking; `block($seconds)` waits. The owner token means only the acquiring process can release it, preventing one worker from releasing another's lock. They power `WithoutOverlapping` job middleware.

**Q9. A user reports seeing a deleted product still listed. What's your debugging approach?**
A. Suspect stale cache. Check whether the listing is cached (`remember`/tags), confirm the delete path forgets/invalidates the relevant keys (model `deleted` event, tag flush, or versioned key), verify the TTL, and check you're not on a shared store where another process repopulated it. Fix: add event-based invalidation with a TTL safety net.

**Q10. Why must you avoid `env()` in production app code?**
A. After `php artisan config:cache`, the `.env` file is no longer parsed at runtime; `env()` returns `null` outside config files. Reading credentials via `env()` in a service then silently yields `null` in production. Always go through `config()`.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ---------- CACHE ----------
Cache::put('k', $v, 60);                 // write with 60s TTL
Cache::get('k', $default);               // read (default/closure on miss)
Cache::add('k', $v, 60);                 // write only if absent (atomic) → bool
Cache::forever('k', $v);                 // no TTL
Cache::remember('k', 60, fn () => …);    // get-or-compute-and-store
Cache::rememberForever('k', fn () => …);
Cache::flexible('k', [5, 20], fn () => …); // stale-while-revalidate (L11+)
Cache::increment('k'[, $n]);             // atomic +
Cache::decrement('k'[, $n]);             // atomic -
Cache::has('k');                         // exists & not null
Cache::missing('k');                     // !has
Cache::forget('k');                      // delete one
Cache::pull('k', $default);              // get + forget
Cache::flush();                          // ⚠️ wipe whole store (ignores prefix)
Cache::store('redis')->...;              // pick a store
Cache::memo()->get('k');                 // in-memory memoize for this request (L12)
Cache::tags(['t'])->put/get/flush(...);  // redis/memcached/array only

// Locks (drivers: redis, memcached, dynamodb, database, file, array)
Cache::lock('name', 10)->get(fn () => …);          // auto-release closure
$l = Cache::lock('name', 10); $l->block(5); …; $l->release();
Cache::restoreLock('name', $token)->release();      // cross-process release (with token)
Cache::lock('name')->forceRelease();                // force-release without token

cache('k');  cache(['k' => $v], 60);  cache()->remember(...); // helper
```

```php
// ---------- SESSION ----------
session(['k' => $v]);  session('k', $default);     // helper write/read
$request->session()->put('k', $v);
$request->session()->get('k', $default);
$request->session()->has('k');      // present & not null
$request->session()->exists('k');   // present (even if null)
$request->session()->missing('k');
$request->session()->all();
$request->session()->pull('k');     // get + forget
$request->session()->forget('k');   // or ->forget(['a','b'])
$request->session()->flush();       // remove all
$request->session()->increment('k'[, $n]);

// Flash
$request->session()->flash('k', $v);   // next request only
session()->now('k', $v);               // current request only
session()->reflash();                  // keep all flash 1 more req
session()->keep(['k']);                // keep specific flash 1 more req
return redirect('/x')->with('k', $v);  // flash shorthand

// Security
$request->session()->regenerate();     // new ID, keep data (on login)
$request->session()->invalidate();     // new ID, wipe data (on logout)
csrf_token();  $request->session()->token();
```

```bash
# ---------- ARTISAN ----------
php artisan cache:clear        # clear APPLICATION cache
php artisan config:cache       # compile config (bootstrap)
php artisan route:cache        # compile routes (bootstrap)
php artisan view:cache         # precompile Blade
php artisan optimize           # config + route + view + event
php artisan optimize:clear     # clear ALL caches incl. application
php artisan make:cache-table   # create cache migration (L11/12; old: cache:table)
php artisan make:session-table # create sessions migration (L11/12; old: session:table)
php artisan migrate
```

```env
# ---------- ENV ----------
CACHE_STORE=redis          # L11/12 (L10: CACHE_DRIVER)
SESSION_DRIVER=database    # L11/12 default
SESSION_LIFETIME=120
SESSION_SECURE_COOKIE=true
SESSION_SAME_SITE=lax
SESSION_HTTP_ONLY=true
```

---

## 🧪 Mini Exercises

1. **Cache-aside dashboard.** Build a `StatsController@index` that returns a JSON payload of three expensive aggregate queries, cached for 5 minutes with `Cache::remember`. Then add a route `?fresh=1` that bypasses and refreshes the cache (forget + recompute).

2. **Tag-based invalidation.** Configure the `redis` store. Cache a user's profile and settings under tags `['users', "user:{id}"]`. Write an Eloquent `saved` model observer that flushes only that user's tag when their record changes. Prove that another user's cache survives.

3. **Atomic "run once" job.** Using `Cache::add` (or `Cache::lock`), guarantee that a `daily-report` routine runs at most once even if triggered concurrently by three queue workers. Log which worker won.

4. **Session cart + flash.** Build add-to-cart and view-cart endpoints backed by the session (`cart.items`). After adding, redirect with a flash `status` message and render it in Blade. Add a "clear cart" action that `forget`s only the cart key (not the whole session).

5. **Fixation check.** Write a custom login action (no `Auth::attempt`) that validates credentials, calls `session()->regenerate()`, and stores `user_id`. Then write a logout that calls `invalidate()`. Inspect the `laravel_session` cookie value before and after login to confirm the ID changed.
