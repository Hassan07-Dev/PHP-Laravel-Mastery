# Localization, Packages, Deployment & Optimization

> **Module 28 — Zero to Expert (Laravel 12 / PHP 8.4)**
> This module ties together the "ship it and keep it fast" half of backend work: speaking your users' language, packaging reusable code, getting your app safely onto a production server, and squeezing performance out of it. These are exactly the topics that separate a junior who *can build a feature* from an engineer who *can run a system*.

**What you'll learn**

- How Laravel localization works: PHP-array vs JSON lang files, `__()`, `trans()`, `trans_choice()`, pluralization, parameter replacement, and locale switching with a fallback.
- How to build a minimal reusable Laravel package: a service provider, publishing config/views/migrations, and package auto-discovery.
- A real production deployment recipe: `.env` handling, `composer install --no-dev --optimize-autoloader`, the full `optimize` cache stack, `storage:link`, migrations, and maintenance mode with a bypass secret.
- How to run background work reliably: queue workers under **Supervisor** and the scheduler via a single **cron** entry.
- Zero-downtime deploys (Forge/Envoyer) and why naive `git pull` deploys break.
- Performance: fixing **N+1** queries with eager loading, response/data caching, database indexing, **Laravel Octane** (Swoole / RoadRunner / FrankenPHP), **OPcache**, lazy collections, and profiling with Debugbar/Telescope.
- A concrete production security checklist.

---

## 1. Localization (i18n)

### Why localization matters

**Localization** (often abbreviated **l10n**) means presenting your app's text — labels, emails, validation messages — in the user's language and regional format. **Internationalization** (**i18n**) is the prior step of *designing your code so localization is possible*: never hard-code user-facing strings, always pull them from a translation layer. If you scatter `return "User created!";` across controllers, you can never translate them. If you write `return __('users.created');`, you can.

The "locale" is a short code identifying a language (and optionally region): `en`, `fr`, `de`, `es`, `pt_BR`, `zh_CN`. Laravel keeps a **current locale** (what we render now) and a **fallback locale** (what we use when a key is missing in the current locale).

### Where translations live

In Laravel 12, the language directory is **not published by default** — a fresh app has no `lang/` folder until you run:

```bash
php artisan lang:publish
```

This creates the `lang/` directory at the project root (Laravel 9+ moved it from `resources/lang/` to `lang/`). Two storage formats coexist:

```
lang/
├── en/
│   ├── auth.php          # PHP-array file: keys grouped by "namespace" (the filename)
│   ├── pagination.php
│   └── validation.php
├── fr/
│   ├── auth.php
│   └── validation.php
├── en.json               # JSON file: full English sentences as keys
└── fr.json
```

**PHP-array files** are best for *short, structured keys* you reference by a dotted path (`auth.failed`). **JSON files** are best for *full sentences* used as their own key (`__('Welcome back, :name')`) — common when the source language is English and you don't want to invent key names.

### PHP-array lang files

```php
// lang/en/messages.php
<?php

return [
    'welcome'      => 'Welcome to our application',
    'greeting'     => 'Hello, :name!',          // :name is a placeholder
    'apples'       => 'There is one apple|There are many apples', // pluralization
    'profile'      => [
        'updated'  => 'Your profile was updated.',
    ],
];
```

```php
// lang/fr/messages.php
<?php

return [
    'welcome'      => 'Bienvenue dans notre application',
    'greeting'     => 'Bonjour, :name !',
    'apples'       => 'Il y a une pomme|Il y a plusieurs pommes',
    'profile'      => [
        'updated'  => 'Votre profil a été mis à jour.',
    ],
];
```

### The `__()` and `trans()` helpers

`__()` and `trans()` both retrieve a translation. The differences are subtle but worth knowing:

- `__($key, $replace, $locale)` — the modern, preferred helper. If the key is missing, it **returns the key string unchanged**.
- `trans($key, $replace, $locale)` — the older helper; behaves the same for retrieval. Called with **no arguments** it returns the translator instance.

```php
// Dotted key into a PHP-array file (file.key.subkey):
echo __('messages.welcome');
// Output: Welcome to our application

echo __('messages.profile.updated');
// Output: Your profile was updated.

// JSON key — the whole sentence is the key:
echo __('Welcome back, friend!');
// Output: Welcome back, friend!  (or the fr.json value if locale is fr)
```

In Blade, the `{{ }}` echo plus `__()` is idiomatic:

```blade
<h1>{{ __('messages.welcome') }}</h1>
<p>{{ __('Welcome back, :name', ['name' => $user->name]) }}</p>
```

> **Missing key behavior:** `__('foo.bar')` when `foo.bar` doesn't exist returns the literal string `"foo.bar"`. That's intentional — a visible broken key on the page is a clear signal to translators, and it never throws.

### Parameter replacement

Placeholders start with `:` in the source string. The replacement is **case-aware**:

```php
echo __('messages.greeting', ['name' => 'sarah']);
// Output: Hello, sarah!

// Capitalize the placeholder name to control output casing:
echo __('Hello, :NAME',  ['name' => 'sarah']); // Output: Hello, SARAH
echo __('Hello, :Name',  ['name' => 'sarah']); // Output: Hello, Sarah
```

| Placeholder style | Input `sarah` | Output |
|---|---|---|
| `:name` | sarah | sarah |
| `:Name` | sarah | Sarah |
| `:NAME` | sarah | SARAH |

> **Gotcha:** it's the casing of the *placeholder in the source string* that drives the output, not the casing of the array key. The rule (from the docs) is: if the placeholder is all-uppercase (`:NAME`) the value is uppercased; if only its first letter is capitalized (`:Name`) the value's first letter is capitalized; otherwise (`:name`) the value is inserted verbatim. The replacement-array key (`'name'`) is matched **case-sensitively**, so it must match the placeholder name — conventionally lowercase.

### Pluralization with `trans_choice`

Many languages change the noun form by count. Laravel handles this with **pipe-delimited** strings and `trans_choice($key, $count, $replace)`:

```php
// Simple two-form (singular | plural):
echo trans_choice('messages.apples', 1);  // Output: There is one apple
echo trans_choice('messages.apples', 5);  // Output: There are many apples
```

For finer control, use **range syntax** `{n}` (exact) and `[a,b]` / `[a,*]` (inclusive ranges):

```php
// lang/en/messages.php
'notifications' => '{0} You have no notifications|{1} You have one notification|[2,*] You have :count notifications',
```

```php
echo trans_choice('messages.notifications', 0); // You have no notifications
echo trans_choice('messages.notifications', 1); // You have one notification
echo trans_choice('messages.notifications', 9); // You have 9 notifications
```

> **`:count` is special and auto-filled.** The built-in `:count` placeholder is automatically replaced with the integer you pass to `trans_choice` — you do **not** pass `['count' => 9]` yourself. Any *other* placeholder you reference in a plural string you must supply via the third argument, e.g. `'{1} :value minute ago|[2,*] :value minutes ago'` with `trans_choice('time.ago', 5, ['value' => 5])`.

> **Under the hood:** `trans_choice` uses the `MessageSelector` class. For pipe-only strings (no `{}`/`[]`), it applies language-specific plural rules — many Slavic languages have *three* forms, and the selector picks the right segment by the count. Range/exact prefixes (`{0}`, `[2,*]`) short-circuit those rules with an explicit match.

### Setting and reading the locale

```php
use Illuminate\Support\Facades\App;

App::setLocale('fr');        // change current locale for this request
$current  = App::currentLocale();   // 'fr'
$isFrench = App::isLocale('fr');    // true

// Read/override the fallback (config/app.php: 'fallback_locale' => 'en')
$fallback = config('app.fallback_locale'); // 'en'
```

`setLocale()` affects only the **current request/process** — it is not persisted. To make locale "sticky" per user, set it on every request via **middleware**:

```php
// app/Http/Middleware/SetLocale.php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\App;
use Symfony\Component\HttpFoundation\Response;

class SetLocale
{
    private const SUPPORTED = ['en', 'fr', 'de'];

    public function handle(Request $request, Closure $next): Response
    {
        $locale = $request->user()?->locale
            ?? $request->session()->get('locale')
            ?? $request->getPreferredLanguage(self::SUPPORTED) // from Accept-Language header
            ?? config('app.locale');

        if (in_array($locale, self::SUPPORTED, true)) {
            App::setLocale($locale);
        }

        return $next($request);
    }
}
```

Register it in `bootstrap/app.php` (Laravel 11/12 style — there is no `Kernel.php`):

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        \App\Http\Middleware\SetLocale::class,
    ]);
})
```

> **Laravel 10 vs 11/12:** in Laravel 10 you'd register middleware in `app/Http/Kernel.php`'s `$middlewareGroups`. Laravel 11 removed the HTTP Kernel; middleware is configured in `bootstrap/app.php`.

### The fallback locale

Set the current and fallback locales in your environment / config:

```env
APP_LOCALE=fr
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=en_US
```

```php
// config/app.php (Laravel 12 reads these env vars by default)
'locale'          => env('APP_LOCALE', 'en'),
'fallback_locale' => env('APP_FALLBACK_LOCALE', 'en'),
```

If `App::currentLocale()` is `fr` and a key is missing in `fr`, Laravel transparently looks it up in `en`. If both miss, you get the raw key back.

### Publishing vendor (package) lang files

When a package ships translations, you override them by publishing into a namespaced folder. A package registers its translations with a **namespace** (e.g. `courier`); you reference them as `__('courier::messages.welcome')`. To customize, publish:

```bash
php artisan vendor:publish --tag=courier-lang
```

This copies the package's strings to `lang/vendor/courier/{locale}/...`, which take precedence over the package's bundled files. The package registers them in its service provider:

```php
// In a package's service provider boot()
$this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
$this->loadJsonTranslationsFrom(__DIR__.'/../lang'); // for JSON keys

$this->publishes([
    __DIR__.'/../lang' => $this->app->langPath('vendor/courier'),
], 'courier-lang');
```

---

## 2. Package Development Basics

### Why build a package

When the same auth gateway, billing logic, or UI kit appears in three apps, copy-paste rots fast. A **Composer package** lets you version that code once, install it with `composer require you/thing`, and let Laravel wire it up automatically. Even if you never publish to Packagist, structuring shared code as a package (loaded via a `path` repository) keeps boundaries clean.

### Anatomy of a Laravel package

```
courier/
├── composer.json
├── config/
│   └── courier.php
├── database/
│   └── migrations/
│       └── 2026_01_01_000000_create_couriers_table.php
├── resources/
│   └── views/
│       └── dashboard.blade.php
├── routes/
│   └── web.php
└── src/
    └── CourierServiceProvider.php
```

### The service provider

The **service provider** is the package's entry point — Laravel calls its `register()` (bind things into the container) and `boot()` (everything else: load routes, views, migrations, publish assets).

```php
// src/CourierServiceProvider.php
<?php

namespace Acme\Courier;

use Illuminate\Support\ServiceProvider;

class CourierServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Merge package defaults so the host app's published config wins,
        // but missing keys fall back to ours.
        $this->mergeConfigFrom(__DIR__.'/../config/courier.php', 'courier');

        // Bind a singleton service into the container.
        $this->app->singleton(CourierClient::class, function ($app) {
            return new CourierClient(config('courier.api_key'));
        });
    }

    public function boot(): void
    {
        // Load resources directly from the package:
        $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
        $this->loadMigrationsFrom(__DIR__.'/../database/migrations');
        $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');

        // Make resources publishable (copy into the host app), each tagged:
        $this->publishes([
            __DIR__.'/../config/courier.php' => config_path('courier.php'),
        ], 'courier-config');

        $this->publishes([
            __DIR__.'/../resources/views' => resource_path('views/vendor/courier'),
        ], 'courier-views');

        // Note: loadMigrationsFrom already runs them; publishing is optional
        // and lets users edit a copy before running.
        $this->publishes([
            __DIR__.'/../database/migrations' => database_path('migrations'),
        ], 'courier-migrations');
    }
}
```

> **`loadViewsFrom` vs `publishes`:** `loadViewsFrom` makes `courier::dashboard` *resolvable* using the package's own files. `publishes` lets the host app copy those views into `resources/views/vendor/courier` to *override* them. Published files always win because Laravel checks the vendor path first.

> **`mergeConfigFrom` is shallow:** it only merges *top-level* keys. Nested arrays in the host's published config fully replace the package's nested array; missing top-level keys fall back. Don't rely on deep merging.

### Package auto-discovery

You do **not** want users to manually register your provider. **Package discovery** lets Composer announce it. Add an `extra.laravel` block to the package's `composer.json`:

```json
{
    "name": "acme/courier",
    "description": "A courier integration for Laravel.",
    "type": "library",
    "require": {
        "php": "^8.2",
        "illuminate/support": "^11.0 || ^12.0"
    },
    "autoload": {
        "psr-4": {
            "Acme\\Courier\\": "src/"
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "Acme\\Courier\\CourierServiceProvider"
            ],
            "aliases": {
                "Courier": "Acme\\Courier\\Facades\\Courier"
            }
        }
    }
}
```

When the host runs `composer require acme/courier`, Laravel's `package:discover` (triggered by Composer's `post-autoload-dump`) reads `extra.laravel` and registers the provider into `bootstrap/cache/packages.php`. No manual config.

To **opt a single app out** of discovering one package, add to the *host app's* `composer.json`:

```json
"extra": { "laravel": { "dont-discover": ["acme/courier"] } }
```

Develop a package locally without publishing by using a **path repository** in the host app:

```json
"repositories": [
    { "type": "path", "url": "../packages/courier" }
],
"require": { "acme/courier": "*" }
```

```bash
composer require acme/courier
# Discovered Package: acme/courier
```

---

## 3. Deployment

### Environment setup and `.env` in production

Laravel reads runtime configuration from environment variables, conventionally loaded from a `.env` file. **The `.env` file is never committed** (it's in `.gitignore`); production gets its own, created out-of-band (server provisioning, secrets manager, or your platform's env UI).

A production `.env` differs from local in critical ways:

```env
APP_NAME="Acme"
APP_ENV=production
APP_KEY=base64:...        # MUST be set; generate once with `php artisan key:generate`
APP_DEBUG=false           # NEVER true in production — leaks stack traces & env
APP_URL=https://acme.com

LOG_LEVEL=warning
LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=10.0.0.5
DB_DATABASE=acme
DB_USERNAME=acme
DB_PASSWORD=__strong_secret__

CACHE_STORE=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_SECURE_COOKIE=true   # cookies only over HTTPS
```

> **`APP_DEBUG=true` in production is the single most common — and most dangerous — Laravel mistake.** It exposes full stack traces, file paths, and (via the error page) loaded environment values to any visitor.

> **Caution:** once you run `config:cache` (below), the framework **stops reading `.env` at runtime** and reads only the compiled config. Any `env()` call *outside* a config file returns `null`. Always read env via `config('...')`, and re-cache after any `.env` change.

### Building for production with Composer

```bash
composer install --no-dev --optimize-autoloader --no-interaction --prefer-dist
```

- `--no-dev` — skip dev dependencies (PHPUnit, Faker, Debugbar). Smaller, safer, faster.
- `--optimize-autoloader` (alias `-o`) — converts PSR-4 rules into a flat **classmap** so the autoloader does a hash lookup instead of hitting the filesystem per class. Meaningfully faster cold class loading.

For maximum effect, generate an **authoritative classmap** (also a security win — classes not in the map won't autoload):

```bash
composer install --no-dev --classmap-authoritative
```

Build frontend assets separately (Vite):

```bash
npm ci && npm run build
```

### The optimization cache stack

Laravel reads many small PHP/JSON files on each request: config, route definitions, Blade compilation, the event-listener map. Caching pre-compiles these into single fast files.

```bash
php artisan config:cache   # merges all config/*.php into bootstrap/cache/config.php
php artisan route:cache    # serializes routes into bootstrap/cache/routes-v7.php
php artisan view:cache     # precompiles all Blade templates
php artisan event:cache    # caches the discovered event→listener map
```

Or do it all (and clear) with the umbrella commands:

```bash
php artisan optimize         # runs config:cache, route:cache, view:cache, event:cache + more
php artisan optimize:clear   # clears all of the above
```

```bash
$ php artisan optimize
   INFO  Caching framework bootstrap, configuration, and metadata.

  config ......................................... DONE
  events ......................................... DONE
  routes ......................................... DONE
  views .......................................... DONE
```

> **`route:cache` requires all routes to be cacheable.** Closure-based routes throw `LogicException: Unable to prepare route ... for serialization because it uses a closure.` Use controller actions (`[UserController::class, 'index']`) so caching works.

> **Order matters in CI/CD:** run `config:cache` **after** the production `.env` is in place, not before. And always `optimize:clear` (or re-`optimize`) as part of deploy, never leave a stale cache from the previous release.

### `storage:link`

Files saved to the `local`/`public` disk live in `storage/app/public`, which isn't web-accessible. The symlink exposes them under `public/storage`:

```bash
php artisan storage:link
# INFO  The [public/storage] link has been created.
```

This is a one-time setup per server (or per release dir in zero-downtime setups). Without it, uploaded images 404.

### Running migrations on deploy

```bash
php artisan migrate --force
```

`--force` is **mandatory** in production: `migrate` refuses to run in a `production` environment without it (a guard against accidental schema changes). Run migrations as part of the deploy script, after pulling new code.

> **Zero-downtime caveat:** if old and new code run simultaneously during a rollout, a destructive migration (dropping a column the old code still reads) breaks live requests. Use **expand/contract**: deploy the additive migration and code that tolerates both shapes first, then a later release removes the old column.

### Maintenance mode (`down` / `up`) with a secret

For migrations that *can't* be done online, put the app into maintenance mode. Visitors get a 503; you bypass with a secret token:

```bash
php artisan down --secret="a1b2c3-deploy-token" --render="errors::503" --retry=60
```

- Visit `https://acme.com/a1b2c3-deploy-token` once — Laravel sets a cookie so **your** browser sees the live site while everyone else sees the 503.
- `--render` pre-renders a Blade view (so the maintenance page still works even if the app can't boot).
- `--retry=60` sets the `Retry-After` header.

Laravel 11+ adds secret-with-redirect convenience:

```bash
php artisan down --with-secret   # auto-generates and prints a secret
```

Bring it back:

```bash
php artisan up
```

> **Why pre-render?** During deploy you might be replacing vendor code mid-flight. `php artisan down` stores its state in `storage/framework/`. With `--render`, the 503 page is captured up front so it doesn't depend on a fully-bootable app.

### Queue workers with Supervisor

`php artisan queue:work` is a long-running process. If it crashes (fatal error, OOM, deploy), it must restart automatically. **Supervisor** is a Linux process manager that does exactly that.

```ini
; /etc/supervisor/conf.d/acme-worker.conf
[program:acme-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/acme/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4                  ; run 4 worker processes
redirect_stderr=true
stdout_logfile=/var/www/acme/storage/logs/worker.log
stopwaitsecs=3600           ; give a job time to finish before SIGKILL
```

```bash
sudo supervisorctl reread        # read new config
sudo supervisorctl update        # apply it
sudo supervisorctl start acme-worker:*
```

> **You MUST restart workers on every deploy.** A worker is a single long-lived PHP process that loads your code *once* at boot — it keeps running the **old** code until restarted. The graceful way:
> ```bash
> php artisan queue:restart   # signals workers to exit after the current job
> ```
> Supervisor then relaunches them with the new code. Use `--max-time` / `--max-jobs` so workers also recycle on their own (helps memory).

### The scheduler cron

Laravel's scheduler (defined in `routes/console.php` or `bootstrap/app.php` in v11/12) is driven by **one** system cron entry that runs every minute:

```cron
* * * * * cd /var/www/acme && php artisan schedule:run >> /dev/null 2>&1
```

`schedule:run` checks which tasks are due *this minute* and runs them. You define the schedule in code:

```php
// routes/console.php (Laravel 11/12)
use Illuminate\Support\Facades\Schedule;

Schedule::command('reports:generate')->dailyAt('02:00');
Schedule::command('telescope:prune')->daily();
Schedule::job(new \App\Jobs\SyncInventory)->everyFifteenMinutes()->withoutOverlapping();
```

> **Laravel 10 vs 11/12:** in Laravel 10 you scheduled inside `app/Console/Kernel.php`'s `schedule()` method. Laravel 11 removed the console Kernel; use `routes/console.php` or `->withSchedule()` in `bootstrap/app.php`.

### Zero-downtime deploys (Forge / Envoyer)

A naive deploy — `git pull` + `composer install` + `migrate` in the live directory — has a window where files are half-updated and requests fail. **Zero-downtime** deploys build a *new release directory* and atomically flip a symlink:

```
/var/www/acme/
├── releases/
│   ├── 20260618093000/   ← previous
│   └── 20260618101500/   ← new (built fully, then activated)
├── storage/              ← shared across releases (persisted)
├── .env                  ← shared
└── current -> releases/20260618101500   ← atomic symlink swap
```

- **Laravel Forge** provisions/manages the server and runs a deploy script you control.
- **Laravel Envoyer** specializes in *zero-downtime* deploys: it clones into a fresh timestamped folder, installs deps, links shared paths (`.env`, `storage/`), then atomically repoints `current`. If a step fails, the live symlink never changes — no broken deploy.

A typical Envoyer/Forge deploy hook:

```bash
composer install --no-dev --optimize-autoloader --no-interaction
php artisan migrate --force
php artisan optimize         # rebuild config/route/view/event caches
php artisan queue:restart    # cycle workers onto new code
# (symlink flip happens automatically after hooks succeed)
```

> Because each release is its own directory, `storage:link` and `.env` are **symlinked from a shared location** so user uploads and secrets survive across releases.

---

## 4. Optimization & Performance

### Fixing N+1 with eager loading

The **N+1 problem**: you load N parent records, then trigger one extra query *per parent* to load a relation — 1 + N queries instead of 2.

```php
// ❌ N+1: 1 query for posts, then 1 query PER post for its author
$posts = Post::all();              // SELECT * FROM posts
foreach ($posts as $post) {
    echo $post->author->name;      // SELECT * FROM users WHERE id = ? (×N)
}
```

**Eager loading** with `with()` collapses this to 2 queries (a single `WHERE id IN (...)` for all authors):

```php
// ✅ 2 queries total
$posts = Post::with('author')->get();
// SELECT * FROM posts
// SELECT * FROM users WHERE id IN (1, 2, 3, ...)

foreach ($posts as $post) {
    echo $post->author->name;      // no query — already loaded
}
```

Nested and constrained eager loads:

```php
Post::with(['author', 'comments.user'])->get();          // nested
Post::with(['comments' => fn ($q) => $q->latest()->limit(5)])->get(); // constrained
Post::withCount('comments')->get();                       // adds comments_count, no N+1
```

Catch N+1 in development by making lazy loading throw:

```php
// app/Providers/AppServiceProvider.php  boot()
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

Now an accidental lazy access throws `LazyLoadingViolationException` locally — but never breaks production.

### Caching

The **cache** stores expensive results (a heavy query, an API response) in fast storage (Redis/Memcached) keyed by a string, with a TTL.

```php
use Illuminate\Support\Facades\Cache;

// Compute on miss, store for 600s, return cached on hit:
$stats = Cache::remember('dashboard:stats', 600, function () {
    return [
        'users'   => User::count(),
        'revenue' => Order::sum('total'),
    ];
});

Cache::forget('dashboard:stats');         // invalidate
$forever = Cache::rememberForever('config:flags', fn () => Flag::all());
```

**Cache tags** (Redis/Memcached only) let you invalidate groups:

```php
Cache::tags(['users', 'reports'])->put('user:1:report', $data, 3600);
Cache::tags(['users'])->flush();   // clears everything tagged 'users'
```

Use the right driver per concern in production:

```env
CACHE_STORE=redis      # app cache
SESSION_DRIVER=redis   # sessions
QUEUE_CONNECTION=redis # jobs
```

> **`file`/`database` cache drivers are fine for dev but slow under load.** Redis is the standard production choice. Don't put sessions on `file` across multiple app servers — requests hitting different boxes won't share session data.

### Database indexing

An **index** is a sorted lookup structure (typically a B-tree) the database maintains so it can find rows without scanning the whole table. Index the columns you **filter, join, or sort by**.

```php
// In a migration
Schema::table('orders', function (Blueprint $table) {
    $table->index('user_id');                       // single-column
    $table->index(['status', 'created_at']);        // composite (order matters!)
    $table->unique('email');                        // enforces uniqueness + indexes
    $table->fullText('body');                       // MySQL/Postgres full-text search
});
```

Composite-index column order follows the **leftmost-prefix** rule: an index on `(status, created_at)` helps `WHERE status = ?` and `WHERE status = ? AND created_at > ?`, but **not** `WHERE created_at > ?` alone.

Verify the optimizer actually uses an index:

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- Look for type=ref/range and a named key; type=ALL means a full table scan (bad).
```

> **Trade-offs:** indexes speed reads but slow writes (each `INSERT`/`UPDATE` must maintain them) and use disk. Don't index low-cardinality columns (e.g. a boolean) or columns you never query. Foreign keys created via `$table->foreignId('user_id')->constrained()` do **not** auto-index in all databases — add the index explicitly when you'll filter by it.

### Laravel Octane

A normal PHP request **boots the entire framework from scratch** (read config, register providers, build the container) on *every* request, then throws it all away. **Laravel Octane** keeps the application **booted in memory** in a long-lived worker, serving many requests from that warm state. This removes per-request bootstrap cost and can multiply throughput.

Supported application servers (per the Octane docs):

- **FrankenPHP** — a modern PHP app server written in Go (built on Caddy); HTTP/2-3, modern compression, worker mode, easy TLS. The default Octane suggests, and the only one whose binary `octane:install` downloads for you.
- **Swoole** / **Open Swoole** — a PHP extension (`pecl install swoole` or `openswoole`); coroutine-based; the only servers that offer Octane's concurrent **task workers**, ticks, and the built-in Octane cache table.
- **RoadRunner** — a Go-based application server; runs PHP as worker processes; no PHP extension needed.

```bash
composer require laravel/octane
php artisan octane:install        # choose FrankenPHP / Swoole / RoadRunner
php artisan octane:start --server=frankenphp --workers=4 --max-requests=500

# --task-workers is Swoole/Open Swoole ONLY (drives Octane::concurrently):
php artisan octane:start --server=swoole --workers=4 --task-workers=6 --max-requests=500
```

On deploy you must reload workers so the new code is loaded into memory — the dedicated command is `octane:reload` (not a plain `octane:start`):

```bash
php artisan octane:reload   # gracefully restarts Octane workers onto new code
```

> **The Octane mental-model shift: state persists between requests.** Because the framework isn't torn down, anything you stash in a static property, a singleton, or a long-lived object **leaks across requests/users**. Classic example — appending to a static array in a controller grows unbounded across requests (a memory leak); injecting the container, `config`, or the `Request` into a *singleton's constructor* freezes a stale copy. Symptoms: one user seeing another's data, "stale" config. Rules: don't hold request/auth state in singletons; inject resolver closures (`fn () => Container::getInstance()`) or use the global `app()`/`request()`/`config()` helpers, which always return the current instance; never store the current `Request`/auth user in a static. Octane resets first-party framework state between requests, but it cannot reset *your* global state. `--max-requests` recycles workers periodically to bound leaks you miss.

### OPcache

**OPcache** is a PHP extension that caches the *compiled bytecode* of your PHP files in shared memory, so PHP doesn't re-parse and re-compile source on every request. It's a pure win for any PHP app — enable it in production:

```ini
; php.ini (production)
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0   ; don't stat files each request — fastest, but you
                                 ; MUST reset opcache on deploy (new code won't load otherwise)
```

```bash
# On deploy, when validate_timestamps=0, clear the bytecode cache:
php artisan optimize        # then also reset opcache via your reload, e.g.:
# (FrankenPHP/Octane restart, or `cachetool opcache:reset`, or php-fpm reload)
```

> **PHP 8.4 note:** OPcache's **JIT** (`opcache.jit`) is most useful for CPU-heavy workloads; typical I/O-bound web apps benefit modestly. The big, reliable win is the bytecode cache itself. With `validate_timestamps=0`, a deploy that doesn't reset OPcache will keep serving the old code — bake the reset into your deploy.

### Lazy collections

Loading a million rows into a normal `Collection` loads them all into memory at once. A **`LazyCollection`** uses PHP generators to process records one at a time, keeping memory flat:

```php
use App\Models\Order;

// ❌ Loads every row into memory — can OOM:
Order::all()->each(fn ($o) => $o->export());

// ✅ Streams rows from the DB cursor, one at a time:
Order::cursor()->each(fn ($o) => $o->export());

// ✅ Or chunk to balance round-trips vs memory:
Order::chunk(1000, function ($orders) {
    foreach ($orders as $order) { $order->export(); }
});
```

`LazyCollection` supports the familiar chainable API, evaluated lazily:

```php
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $handle = fopen(storage_path('app/huge.log'), 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;
    }
})
->filter(fn ($line) => str_contains($line, 'ERROR'))
->take(100)            // stops reading the file after 100 matches
->each(fn ($line) => Log::warning($line));
```

### Profiling: Debugbar & Telescope

You can't optimize what you can't measure. Two dev tools:

- **Laravel Debugbar** (`barryvdh/laravel-debugbar`) — an in-browser overlay showing queries (with duplicate/N+1 flags), timings, memory, views, and route info.
- **Laravel Telescope** (`laravel/telescope`) — a dashboard logging requests, queries, jobs, mail, exceptions, cache hits, and scheduled tasks; great for debugging async/background work.

```bash
composer require barryvdh/laravel-debugbar --dev
composer require laravel/telescope --dev
php artisan telescope:install && php artisan migrate
```

> **Never ship these to production.** Install with `--dev`, gate Telescope behind an authorization gate, and prune its data (`telescope:prune`). Debugbar in production leaks queries and config to visitors and slows every request. (Telescope *can* run in prod if locked down and pruned, but most teams keep it to staging.)

### Production security checklist

```text
[ ] APP_DEBUG=false and APP_ENV=production
[ ] APP_KEY set (php artisan key:generate, once) — encryption depends on it
[ ] .env not committed; file perms 600; never web-accessible
[ ] Only public/ is the web root (never expose project root / vendor / storage)
[ ] HTTPS enforced; SESSION_SECURE_COOKIE=true; HSTS header set
[ ] config:cache / route:cache / view:cache / event:cache run on deploy
[ ] composer install --no-dev (no Debugbar/Telescope/Faker in prod deps)
[ ] DB user has least privilege; strong, rotated DB password
[ ] CSRF protection on all state-changing web routes (on by default)
[ ] Validation + mass-assignment guarded ($fillable/$guarded) on all models
[ ] Rate limiting on auth + API endpoints (throttle middleware)
[ ] Security headers (CSP, X-Frame-Options, X-Content-Type-Options)
[ ] Dependencies patched: composer audit; npm audit
[ ] migrate --force in deploy; backups + tested restore
[ ] Logs to a level (warning+) that doesn't leak PII; centralized
[ ] Queue/scheduler restarted on deploy (queue:restart)
```

```bash
php artisan key:generate    # one-time, sets APP_KEY
composer audit              # report known CVEs in dependencies
php artisan about           # quick env/cache/driver sanity check before/after deploy
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `env()` outside config files after `config:cache`.**
   Once config is cached, `.env` is *not loaded* at runtime, so `env('FOO')` in a controller returns `null`. **Fix:** put all `env()` reads inside `config/*.php` and read via `config('foo')` everywhere else. Re-run `php artisan config:cache` after any `.env` change.

2. **`route:cache` fails with closures.**
   `php artisan route:cache` throws `Unable to prepare route [...] for serialization because it uses a closure`. **Fix:** convert closure routes to controller actions (`Route::get('/x', [XController::class, 'show'])`). Only then is the route serializable.

3. **Forgetting to restart queue workers / reset OPcache on deploy.**
   Workers are long-lived PHP processes that loaded the *old* code at boot; with `opcache.validate_timestamps=0`, FPM serves old bytecode too. Both keep running stale code after you deploy. **Fix:** `php artisan queue:restart` (Supervisor relaunches) and reset OPcache (FPM reload / Octane restart) as deploy steps.

4. **`APP_DEBUG=true` in production.**
   Leaks stack traces, file paths, and environment variables on any error. **Fix:** `APP_DEBUG=false`, `APP_ENV=production`; verify with `php artisan about`.

5. **Holding per-request/user state in singletons under Octane.**
   Because the app stays booted in memory, a static or singleton populated in request A is still there in request B — cross-request data bleed. **Fix:** keep request/auth state out of singletons; rebind or reset per-request services; use `--max-requests` to recycle workers.

6. **Assuming `mergeConfigFrom` deep-merges, or that `migrate` runs without `--force` in prod.**
   `mergeConfigFrom` only merges top-level keys; and `migrate` refuses to run in `production` without `--force`. **Fix:** document the full config array for users; always `php artisan migrate --force` in deploy scripts.

---

## ✅ Best Practices

- **Never hard-code user-facing strings** — wrap them in `__()` / `trans_choice()` from day one, even if you ship only one language.
- **Read config via `config()`, set values via `env()` only inside `config/*.php`.** This is what makes `config:cache` safe.
- **Make deploys atomic and idempotent.** Build a new release dir (Envoyer/Forge), run `optimize`, migrate with `--force`, then flip the symlink. Re-runnable without side effects.
- **Always run the cache stack in production and clear it on deploy** (`php artisan optimize` / `optimize:clear`). Never commit `bootstrap/cache/*.php`.
- **Restart background processes on every deploy:** `queue:restart` + reset OPcache. Use Supervisor for resilience, one cron line for the scheduler.
- **Profile before optimizing.** Use Debugbar/Telescope to *find* N+1s and slow queries; eager-load and index based on evidence, not guesses. Enable `Model::preventLazyLoading()` outside production.
- **Index columns you filter/join/sort by; respect the leftmost-prefix rule;** verify with `EXPLAIN`. Don't over-index.
- **Stream large datasets** with `cursor()`/`chunk()`/`LazyCollection`; never `->all()` a huge table.
- **Treat the security checklist as a deploy gate**, not an afterthought. Run `composer audit` in CI.
- **For packages:** auto-discovery, tagged `publishes`, `mergeConfigFrom` defaults, and namespaced views/lang so consumers can override cleanly.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `__()`, `trans()`, and `trans_choice()`?**
`__()` and `trans()` both retrieve a single translation; `__()` is the modern preferred helper and returns the key unchanged if missing. `trans()` with no args returns the translator instance. `trans_choice($key, $count)` selects the correct pluralized segment from a pipe-delimited string based on `$count` and the locale's plural rules.

**Q2. How does the fallback locale work?**
Laravel keeps a current locale and a `fallback_locale`. When a key is missing in the current locale's files, the translator transparently looks it up in the fallback locale. If both miss, you get the literal key string back (it never throws).

**Q3. How does Laravel package auto-discovery work under the hood?**
Each package's `composer.json` declares providers/aliases under `extra.laravel`. On Composer's `post-autoload-dump` event, Laravel runs `package:discover`, which scans installed packages' `extra.laravel`, and writes the resolved provider/alias list to `bootstrap/cache/packages.php`. The framework loads that manifest at boot — so no manual registration. Apps can opt out via `dont-discover`.

**Q4. Why is `APP_DEBUG=true` dangerous in production, and what does `config:cache` change about `env()`?**
Debug mode renders detailed stack traces and exposes file paths and environment values on errors. `config:cache` compiles all `config/*.php` into one file and stops loading `.env` at runtime — so `env()` calls *outside* config return `null`. You must read via `config()` and re-cache after `.env` changes.

**Q5. (Under the hood) How does Laravel Octane achieve higher throughput, and what's the main risk?**
A standard request boots the whole framework (config, providers, container) then discards it. Octane boots the app once inside a long-lived worker (Swoole/RoadRunner/FrankenPHP) and reuses that warm state across many requests, eliminating per-request bootstrap. The risk is shared state: statics, singletons, and long-lived objects persist between requests and can leak data across users. Mitigations: stateless request handling, per-request rebinding, and `--max-requests` worker recycling.

**Q6. How do you fix an N+1 query problem, and how do you detect it?**
Eager-load the relation with `with()` (or `load()` after the fact), turning 1+N queries into 2 via a `WHERE IN`. Use `withCount()` for counts and constrained closures for filtered relations. Detect with Debugbar/Telescope, or fail fast in dev via `Model::preventLazyLoading(! app()->isProduction())`.

**Q7. Walk me through a safe production deploy.**
Put new code in a fresh release dir; `composer install --no-dev --optimize-autoloader`; `npm ci && npm run build`; ensure shared `.env` and `storage/` are symlinked; `php artisan migrate --force`; `php artisan optimize`; `php artisan queue:restart`; reset OPcache; atomically flip the `current` symlink. Use `down --secret`/`up` only if a migration can't be done online. Forge manages the server; Envoyer specializes in the zero-downtime symlink flip with rollback.

**Q8. Why must queue workers be restarted on deploy, and how does Supervisor help?**
A worker is a single PHP process that loads your code once at boot and keeps it in memory — it'll run the old code indefinitely. `queue:restart` signals workers to exit gracefully after the current job; Supervisor (with `autorestart=true`) relaunches them on the new code and restarts them if they crash or OOM.

**Q9. How does the Laravel scheduler run with only one cron entry?**
A single `* * * * * php artisan schedule:run` cron line runs every minute. `schedule:run` evaluates your code-defined schedule, determines which tasks are due *this* minute, and dispatches them. So you express complex schedules (hourly, daily-at, weekday-only, `withoutOverlapping`) purely in PHP.

**Q10. When does an index help, and what does composite-index column order mean?**
Index columns used in `WHERE`, `JOIN`, `ORDER BY`. A composite index on `(a, b)` helps queries filtering by `a` or `a AND b` (leftmost prefix), but not by `b` alone. Indexes speed reads but slow writes and consume disk — index based on `EXPLAIN`, not guesses.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Localization ---
__('messages.welcome');                       // PHP-array key
__('Welcome, :name', ['name' => $u->name]);   // JSON key + replacement (:Name / :NAME for case)
trans_choice('messages.apples', $count);       // pluralization
App::setLocale('fr'); App::currentLocale();    // set / read locale
config('app.fallback_locale');                 // 'en'
```

```bash
# --- Localization / vendor publish ---
php artisan lang:publish
php artisan vendor:publish --tag=courier-lang
```

```bash
# --- Build for production ---
composer install --no-dev --optimize-autoloader --no-interaction
npm ci && npm run build

# --- Cache stack ---
php artisan optimize           # config + route + view + event cache
php artisan optimize:clear     # clear them all
php artisan config:cache | route:cache | view:cache | event:cache

# --- One-time / per-release ---
php artisan key:generate
php artisan storage:link
php artisan migrate --force

# --- Maintenance mode ---
php artisan down --secret="token" --render="errors::503" --retry=60
php artisan up

# --- Background work ---
php artisan queue:work redis --tries=3 --max-time=3600
php artisan queue:restart
sudo supervisorctl reread && sudo supervisorctl update

# --- Octane ---
php artisan octane:install            # FrankenPHP | Swoole | RoadRunner
php artisan octane:start --server=frankenphp --workers=4 --max-requests=500
php artisan octane:reload             # graceful worker restart on deploy

# --- Health / audit ---
php artisan about
composer audit
```

```cron
# Scheduler (one entry, every minute)
* * * * * cd /var/www/acme && php artisan schedule:run >> /dev/null 2>&1
```

```php
// --- Performance ---
Post::with(['author', 'comments.user'])->withCount('comments')->get();  // no N+1
Cache::remember('k', 600, fn () => heavy());                            // cache w/ TTL
Order::cursor()->each(fn ($o) => $o->export());                        // lazy/streaming
Model::preventLazyLoading(! app()->isProduction());                    // catch N+1 in dev
```

---

## 🧪 Mini Exercises

1. **Three-language greeting.** Create `lang/en/messages.php`, `lang/fr/messages.php`, and `lang/de/messages.php` plus a `SetLocale` middleware that reads `?lang=` from the query string (whitelisting `en`/`fr`/`de`). Add a route that returns `__('messages.greeting', ['name' => 'Sam'])` and verify the fallback locale by deleting one key from the German file.

2. **Pluralized cart.** Define a `cart.items` key using range syntax (`{0}`, `{1}`, `[2,*] :count items in your cart`) so it renders "Your cart is empty", "1 item in your cart", and "N items in your cart". Render it in a Blade view with `trans_choice('messages.cart.items', $count)` — and confirm the `:count` placeholder fills in automatically *without* you passing it in the replacements array.

3. **Tiny package.** Build a `path`-repository package with a service provider that `mergeConfigFrom` a `config/greeter.php`, exposes a `greeter::hello` view, and is tagged so `vendor:publish --tag=greeter-config` works. Wire it via auto-discovery (`extra.laravel`) and confirm it loads without manual registration.

4. **Deploy script.** Write a single bash deploy script that: installs prod deps, builds assets, runs `migrate --force`, runs `php artisan optimize`, restarts queue workers, and uses `down --secret`/`up` around the migration. Add a comment explaining why `config:cache` must run *after* `.env` is in place.

5. **Hunt the N+1.** Enable `Model::preventLazyLoading()` outside production, write a controller that lists posts with their authors and comment counts *without* eager loading, observe the violation, then fix it with `with()` + `withCount()`. Bonus: add the right index to the `comments.post_id` column and confirm with `EXPLAIN`.
