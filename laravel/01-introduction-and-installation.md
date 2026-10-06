# Laravel: Introduction & Installation

> **Module 01** — Your first stop on the road from zero to expert. By the end you'll understand *what* Laravel is, *why* it dominates PHP web development, how to install it cleanly, and you'll have a working "Hello, World" served through the full request lifecycle.

**What you'll learn**

- What Laravel is, the problems it solves, and why it's the most popular PHP framework.
- What "MVC" means and how Laravel maps onto it.
- The exact system requirements (PHP version, extensions, Composer) and how to install Laravel two different ways.
- A guided tour of every top-level directory and what belongs where.
- How `.env`, `config()`, and `env()` work — plus the single most common production bug (`env()` after config caching).
- Artisan: the command-line companion, and the commands you'll type every day.
- How a request travels from the browser through `public/index.php` to a controller and back as a view.
- Composer autoloading, Laravel's versioning/LTS philosophy, Vite for assets, and Tinker for live experimentation.

---

## 1. What is Laravel, and why is it everywhere?

**Laravel** is a free, open-source **web application framework** for PHP, created by Taylor Otwell in 2011. A *framework* is a pre-built skeleton of reusable code that handles the boring, repetitive plumbing of web apps — routing requests, talking to databases, managing sessions, validating input, sending email — so you can focus on the logic that makes *your* app unique.

You could build a web app in raw PHP. People did for years. But you'd hand-roll routing, SQL escaping, CSRF protection, password hashing, templating, and a hundred other things — and you'd get half of them subtly wrong (security holes especially). A framework gives you battle-tested, audited solutions for all of that.

### Why Laravel specifically?

There are many PHP frameworks (Symfony, CodeIgniter, CakePHP). Laravel won the popularity contest for a few concrete reasons:

- **Expressive, readable syntax.** Laravel optimizes for *developer happiness*. Code reads almost like English: `Route::get('/users', [UserController::class, 'index'])`. This is not cosmetic — readable code is cheaper to maintain.
- **Batteries included.** Out of the box you get an ORM (Eloquent), a templating engine (Blade), authentication scaffolding, queues, task scheduling, caching, mail, file storage abstraction, validation, and more. You rarely reach for third-party glue.
- **A huge first-party ecosystem.** Official packages cover almost every common need: **Breeze**/**Jetstream** (auth), **Sanctum**/**Passport** (API tokens/OAuth), **Horizon** (queue dashboard), **Telescope** (debugging), **Cashier** (Stripe/Paddle billing), **Scout** (full-text search), **Forge**/**Vapor**/**Envoyer** (deployment), and **Nova** (admin panels).
- **Excellent documentation** and a massive community, which means answers to almost any question are a search away.
- **Convention over configuration.** Follow Laravel's conventions (naming, folder layout) and almost everything "just works" with zero config. You only configure the exceptions.

> **Jargon check — ORM:** *Object-Relational Mapper.* A layer that lets you work with database rows as PHP objects (`$user->name`) instead of writing raw SQL. Laravel's ORM is called **Eloquent**.

### Laravel is an MVC framework

**MVC** stands for **Model–View–Controller**, an architectural pattern that separates an application into three responsibilities:

- **Model** — the data and business rules. In Laravel these are usually Eloquent models (e.g. `App\Models\User`) representing database tables.
- **View** — what the user sees. In Laravel these are **Blade** templates (`.blade.php` files) that produce HTML.
- **Controller** — the coordinator. It receives the request, asks Models for data, and hands that data to a View.

The point of MVC is **separation of concerns**: your HTML doesn't contain database queries, and your database logic doesn't contain HTML. This makes code easier to test, reason about, and change.

```text
Browser ──HTTP request──▶  Route  ──▶  Controller  ──asks──▶  Model (database)
                                          │
                                          ▼ passes data
                                         View (Blade)  ──HTML──▶  Browser
```

> Laravel is technically more flexible than strict MVC — you can put logic in route closures, form requests, jobs, service classes, etc. But MVC is the mental model that frames everything, and interviewers expect you to name it.

---

## 2. System requirements

Before installing Laravel **12** (the version we target), make sure your machine has the prerequisites.

### PHP version

- **Laravel 12** supports **PHP 8.2 – 8.5**. It runs great on **PHP 8.4** (our target). PHP 8.2 and 8.3 are also fully supported, and PHP 8.5 is supported as of the Laravel 12 line.
- For reference: **Laravel 11** supported PHP 8.2 – 8.4, and **Laravel 10** supported PHP 8.1 – 8.3. If you're reading older tutorials that assume PHP 8.1, that's a Laravel-10-era assumption.
- Tip: the official `php.new` installer (`curl -fsSL https://php.new/install/mac/8.4 | bash`, and equivalents for Windows/Linux) sets up PHP, Composer, and the Laravel installer in one step.

Check your version:

```bash
php -v
```

```text
PHP 8.4.1 (cli) (built: Dec 17 2024 ...)
Copyright (c) The PHP Group
Zend Engine v4.4.1 ...
```

### Required PHP extensions

Laravel needs a set of common extensions, almost all of which ship with a standard PHP build:

- Ctype, cURL, DOM, Fileinfo, Filter, Hash, Mbstring, OpenSSL, PCRE, PDO, Session, Tokenizer, XML.

Verify what's compiled in:

```bash
php -m
```

```text
# Output: a list of loaded modules, one per line. Confirm you see
# openssl, mbstring, pdo, tokenizer, ctype, json, etc.
```

If something is missing on Linux you'd install it via your package manager, e.g. `sudo apt install php8.4-mbstring`.

### Composer

**Composer** is PHP's dependency manager (think `npm` for PHP). Laravel uses it to install the framework and every package. You need it installed globally.

```bash
composer --version
```

```text
Composer version 2.8.4 2024-12-11 ...
```

If you don't have it, install from [getcomposer.org](https://getcomposer.org). On macOS, `brew install composer` is the easy path.

### Node.js (for front-end assets)

You don't *need* Node to run Laravel's PHP, but you need it to compile CSS/JS via **Vite** (covered in §8). Install Node 18+ and you get `npm` alongside it.

---

## 3. Installing Laravel

There are two standard ways. Both produce an identical project.

### Method A — the Laravel installer (recommended)

The **Laravel installer** is a small global tool that scaffolds a new project and (in Laravel 12) interactively asks which **starter kit** and testing framework you want.

Install the installer once:

```bash
composer global require laravel/installer
```

> Make sure Composer's global `bin` directory is on your `PATH`, or the `laravel` command won't be found. Typically `~/.composer/vendor/bin` or `~/.config/composer/vendor/bin` on Linux/macOS.

Create a project:

```bash
laravel new blog
```

```text
# In Laravel 12 the installer prompts you (exact wording/order can vary by version):
#   Which starter kit would you like to install? [None, React, Vue, Livewire]
#   Which testing framework do you prefer? [Pest, PHPUnit]   (asked when no starter kit)
#   Which database will your application use? [SQLite, MySQL, MariaDB, PostgreSQL, SQL Server]
#   Would you like to run `npm install` and `npm run build`? (yes/no)
# Then it scaffolds the app, runs composer install, copies .env.example -> .env,
# generates the APP_KEY, and (for SQLite) creates database/database.sqlite and runs migrations.
```

### Method B — `composer create-project`

This works anywhere Composer is installed, no separate installer needed. It's the universal fallback.

```bash
composer create-project laravel/laravel blog
```

```text
# Composer downloads the laravel/laravel skeleton, installs all
# dependencies into vendor/, copies .env.example to .env, and
# generates an application key automatically.
```

### What happens right after creating a project

Whichever method you used, do this:

```bash
# 1. Generate the APP_KEY if it wasn't already (installer/create-project usually does it)
php artisan key:generate

# 2. Start the dev server
php artisan serve
```

```text
   INFO  Server running on [http://127.0.0.1:8000].

  Press Ctrl+C to stop the server
```

Open `http://127.0.0.1:8000` and you'll see Laravel's welcome page. You're running Laravel.

> **`APP_KEY`** is a random 32-byte string used to encrypt session data and cookies. Without it, encryption throws an exception. `key:generate` writes it into your `.env` file. **Never share it or commit it** — and never reuse one across environments.

---

## 4. A tour of the directory structure

When you open the new project, here's what each top-level folder is for. Understanding this layout is half of "knowing Laravel."

```text
blog/
├── app/            ← YOUR application code (models, controllers, etc.)
├── bootstrap/      ← framework bootstrap + cached files
├── config/         ← all configuration files
├── database/       ← migrations, seeders, factories, (optional) sqlite db
├── public/         ← web server document root; the front controller lives here
├── resources/      ← Blade views, raw CSS/JS, language files
├── routes/         ← route definitions (web.php, console.php, etc.)
├── storage/        ← logs, compiled Blade, file uploads, caches
├── tests/          ← automated tests
├── vendor/         ← Composer-installed dependencies (never edit, never commit)
├── .env            ← environment-specific secrets/settings (never commit)
├── artisan         ← the command-line entry point
├── composer.json   ← PHP dependencies + autoload config
└── vite.config.js  ← front-end asset build config
```

### `app/` — your code lives here

This is where you spend most of your time. In **Laravel 11 and 12** the structure was slimmed down considerably compared to Laravel 10.

```text
app/
├── Http/
│   └── Controllers/   ← controllers
├── Models/            ← Eloquent models (User.php ships by default)
└── Providers/         ← service providers (AppServiceProvider.php)
```

> **Laravel 10 vs 11/12:** In Laravel 10 the `app/` directory also contained `app/Http/Middleware/`, `app/Http/Kernel.php`, `app/Console/Kernel.php`, and `app/Exceptions/Handler.php`. Laravel 11 **removed these app-level files** — middleware, exceptions, and bootstrapping are now configured fluently in `bootstrap/app.php`. Laravel 12 keeps this slim structure. If an interviewer asks "where's the HTTP Kernel?", the precise answer is: *the user-editable `app/Http/Kernel.php` class is gone; you now configure middleware/exceptions/routing in `bootstrap/app.php`.* The HTTP kernel itself still exists inside the framework (`Illuminate\Foundation\Http\Kernel`) — it was just removed from your application's editable code.

> **Where do new `app/` subfolders come from?** Only `Http/`, `Models/`, and `Providers/` ship by default. Others (`Console/`, `Events/`, `Listeners/`, `Jobs/`, `Mail/`, `Notifications/`, `Policies/`, `Rules/`, `Exceptions/`, `Broadcasting/`) are created on demand the first time you run the matching `make:` command (e.g. `php artisan make:job` creates `app/Jobs/`).

### `routes/` — URL → code mappings

```text
routes/
├── web.php        ← browser routes (gets session, CSRF protection)
├── console.php    ← custom Artisan commands defined as closures
└── api.php        ← (optional) API routes; added via `php artisan install:api`
```

> In Laravel 11/12, `api.php` and `channels.php` are **not** present by default. Run `php artisan install:api` or `install:broadcasting` to add them. In Laravel 10 they shipped by default.

### `resources/` — front-end source

```text
resources/
├── views/    ← Blade templates (.blade.php)
├── css/      ← source CSS (app.css)
└── js/       ← source JS (app.js)
```

### `config/` — configuration files

Each file returns a PHP array of settings (`app.php`, `database.php`, `mail.php`, etc.). These are the **only** place you should call `env()` (more on that in §5).

### `database/` — schema and data

```text
database/
├── migrations/   ← version-controlled schema changes
├── seeders/      ← scripts that insert sample/default data
├── factories/    ← blueprints for generating fake model data (for tests)
└── database.sqlite  ← (Laravel 11/12 default) a local SQLite file
```

> **Laravel 11/12 default database is SQLite.** The fresh `.env` points at a `database/database.sqlite` file, so a brand-new app works with zero database setup. Older Laravel defaulted to MySQL.

### `public/` — the web root (and the *only* publicly exposed folder)

```text
public/
├── index.php   ← THE FRONT CONTROLLER (every request enters here)
├── .htaccess   ← Apache rewrite rules (pretty URLs)
└── (compiled assets land here after `npm run build`)
```

Your web server's document root must point at `public/`, **not** at the project root. This keeps `.env`, `vendor/`, and source code unreachable from the web. (We'll trace `index.php` in §9.)

### `storage/` — generated files

```text
storage/
├── app/         ← user file uploads, app-generated files
├── framework/   ← compiled Blade views, sessions, caches
└── logs/        ← laravel.log (your default log destination)
```

`storage/` and `bootstrap/cache/` must be **writable** by the web server, or you'll hit permission errors.

### `bootstrap/` — framework startup

Contains `app.php` (where the application instance is created and, since Laravel 11, where middleware/routing/exceptions are configured), `providers.php` (the list of your application's service providers — in Laravel 11/12 this replaced the `providers` array that used to live in `config/app.php`), and a `cache/` folder for framework-generated cache files.

### `tests/` — automated tests

Ships with **Pest** by default in Laravel 11/12 (PHPUnit is selectable). Contains `Feature/` (test full requests) and `Unit/` (test isolated classes) subfolders.

### `vendor/` — dependencies

Every Composer package, including Laravel's own framework code (`vendor/laravel/framework`). **Never edit files here** (changes get overwritten) and **never commit it to Git** (`.gitignore` already excludes it). You regenerate it with `composer install`.

---

## 5. The `.env` file, `config()`, and `env()`

### Why `.env` exists

Your app needs settings that differ per environment: the database password on your laptop is different from production; the mail driver in dev is `log`, in prod it's a real SMTP server. You must **not** hard-code these into source code (you'd leak secrets and couple code to one environment).

The solution is the **`.env` file** — a simple `KEY=VALUE` text file at the project root, listing environment-specific values. Laravel loads it at startup (via the `vlucas/phpdotenv` library) into the `$_ENV` super-global, where the `env()` helper can read it. Note that real, system-level environment variables (exported by your shell, server, or container) **override** values in `.env`.

```env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:abcd1234...........................=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=sqlite
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=laravel
# DB_USERNAME=root
# DB_PASSWORD=

MAIL_MAILER=log
CACHE_STORE=database
SESSION_DRIVER=database
QUEUE_CONNECTION=database
```

Key rules:

- `.env` is **git-ignored**. You never commit it. Instead you commit **`.env.example`** (the same keys with safe/blank values) so teammates know what to fill in. (Laravel can also **encrypt** an env file for safe storage in source control via `php artisan env:encrypt`.)
- Values are strings, but a few reserved words are coerced by `env()`: `true`/`(true)` → `true`, `false`/`(false)` → `false`, `null`/`(null)` → `null`, and `empty`/`(empty)` → `''`.
- Quote values containing spaces: `APP_NAME="My App"`.

> ⚠️ **Security: `APP_DEBUG` must be `false` in production.** `APP_DEBUG=true` is correct for local development, but in production it makes Laravel render full stack traces and dump environment/config values on any error — a serious information-disclosure leak. Always set `APP_DEBUG=false` (and `APP_ENV=production`) on production servers.

### `env()` — read a raw environment value

The `env()` helper reads a value from the environment (the parsed `.env`), with an optional default:

```php
$debug = env('APP_DEBUG', false);   // true (boolean), thanks to coercion
$key   = env('APP_KEY');            // the base64 string
```

### `config()` — read a resolved configuration value

The `config()` helper reads from the `config/*.php` files using **dot notation**: `config('file.key')`.

```php
config('app.name');                  // 'Laravel'
config('app.timezone');              // 'UTC' (the default in a fresh app)
config('database.default');          // 'sqlite'
config('app.timezone', 'UTC');       // second arg = default if key is missing

// Set a value at runtime (rare, but valid — only affects the current request):
config(['app.timezone' => 'Europe/London']);
```

You can also use the `Config` facade, which (since Laravel 11) offers **typed accessors** that throw if the value isn't the expected type — handy for static analysis:

```php
use Illuminate\Support\Facades\Config;

Config::get('app.name');             // same as config('app.name')
Config::string('app.name');          // returns string, or throws
Config::boolean('app.debug');        // returns bool, or throws
Config::array('app.maintenance');    // returns array, or throws
```

Inspect resolved config from the CLI without writing code:

```bash
php artisan config:show app          # dump the resolved config/app.php values
```

Inside `config/app.php` you'll see config values *pull from* env:

```php
// config/app.php (excerpt)
return [
    'name'  => env('APP_NAME', 'Laravel'),
    'env'   => env('APP_ENV', 'production'),
    'debug' => (bool) env('APP_DEBUG', false),
    // ...
];
```

So the data flow is: **`.env` → read by `env()` inside `config/*.php` → exposed everywhere via `config()`.**

### ⚠️ The config-caching gotcha (the #1 rule about `env()`)

**Never call `env()` outside of `config/*.php` files.** Here's the precise reason.

In production you run `php artisan config:cache`. This **serializes all config arrays into one cached PHP file** for speed. As an important side effect, **once config is cached, Laravel stops loading the `.env` file at all** during requests and Artisan commands. After that, `env()` no longer sees anything from `.env`: it will **only return real, system-level environment variables** (those exported into the actual process environment by your server/container). For a key that lived only in `.env` — which is the usual case — `env('SOME_KEY')` therefore returns **`null`** (or the default you passed).

```php
// ❌ In a controller — BREAKS after `php artisan config:cache`
$stripeKey = env('STRIPE_SECRET');   // null in production (key lived only in .env)!

// ✅ Define it in config/services.php
// 'stripe' => ['secret' => env('STRIPE_SECRET')],
$stripeKey = config('services.stripe.secret');   // works always
```

Because the config file is read *while it's being cached*, the `env()` call *inside* `config/services.php` is captured at cache time. Calls from controllers/models run at request time, after caching, when `.env` is no longer loaded.

**Rule of thumb:** `env()` belongs *only* in `config/*.php`. Everywhere else in your app, use `config()`. This is one of the most common interview "gotcha" questions and one of the most common real-world production bugs.

---

## 6. Artisan — the command-line companion

**Artisan** is Laravel's command-line interface (CLI). It lives at the project root as the `artisan` file and is invoked with `php artisan <command>`. It automates scaffolding, database work, caching, and more — and you can write your own commands.

List every available command:

```bash
php artisan list
```

Get help for a specific command:

```bash
php artisan help make:controller
# or
php artisan make:controller --help
```

### Commands you'll use constantly

```bash
# Serving & info
php artisan serve                 # start the dev server (http://127.0.0.1:8000)
php artisan about                 # environment + package overview
php artisan route:list            # show every registered route

# Scaffolding (the `make:` family)
php artisan make:controller PostController
php artisan make:model Post -mfc   # model + Migration + Factory + Controller
php artisan make:migration create_posts_table
php artisan make:middleware EnsureTokenIsValid

# Database
php artisan migrate                # run pending migrations
php artisan migrate:fresh --seed   # drop all tables, re-migrate, seed
php artisan db:seed                # run seeders

# Caching (production performance)
php artisan config:cache           # cache config (remember the env() gotcha!)
php artisan route:cache            # cache routes
php artisan view:cache             # precompile Blade views
php artisan event:cache            # cache event/listener discovery
php artisan optimize               # cache config + routes + views + events at once
php artisan optimize:clear         # clear ALL caches (config, route, view, event, compiled)

# Maintenance
php artisan key:generate           # generate APP_KEY
php artisan storage:link           # symlink public/storage -> storage/app/public
php artisan tinker                 # interactive REPL (see §10)
```

> **Tip:** `php artisan about` is the fastest way to see your PHP version, Laravel version, cache drivers, and which caches are active — great for debugging "why is my env change not taking effect" (answer: config is cached).

---

## 7. Serving the application with `artisan serve`

For local development, `php artisan serve` boots PHP's built-in web server pointed at `public/`:

```bash
php artisan serve
# Server running on [http://127.0.0.1:8000].

# Change the port or host:
php artisan serve --port=8080
php artisan serve --host=0.0.0.0 --port=8000   # expose on your LAN
```

`artisan serve` is **for development only** — it's single-threaded and not built for production. In production you run PHP-FPM behind **Nginx** or **Apache** (or use Laravel **Octane** with Swoole/FrankenPHP for high performance). For a richer local environment you can use **Laravel Sail** (Docker) or **Laravel Herd** (a fast native macOS/Windows environment).

> **The `dev` Composer script (modern shortcut).** A fresh Laravel 12 app ships with a `composer.json` script that boots the PHP server, queue worker, log tailer, and the Vite dev server together with one command. This is what the official docs now recommend for local development:
>
> ```bash
> composer run dev   # runs `artisan serve` + queue:listen + pail + `npm run dev` concurrently
> ```
>
> It saves you from juggling several terminal tabs. You can still run the pieces individually (`php artisan serve`, `npm run dev`) when you prefer.

---

## 8. Vite for front-end assets

Laravel uses **Vite** (a modern JavaScript build tool) to compile and bundle your CSS and JavaScript from `resources/css` and `resources/js` into optimized files in `public/build`.

During development you run the Vite dev server, which provides **hot module replacement** (instant browser updates when you edit CSS/JS):

```bash
npm install      # once, to install front-end deps
npm run dev      # start the Vite dev server (keep it running alongside artisan serve)
npm run build    # compile a production bundle into public/build
```

In your Blade layout you reference assets with the `@vite` directive, which automatically points to the dev server (in dev) or the built files (in prod):

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    @yield('content')
</body>
</html>
```

> **Why Vite and not Laravel Mix/Webpack?** Laravel switched its default from **Laravel Mix (Webpack)** to **Vite** starting with Laravel 9.2. Vite is dramatically faster for development because it serves source modules natively rather than rebundling on every change. If you see `webpack.mix.js` in a codebase, it's a pre-Vite (older) project.

---

## 9. The front controller: `public/index.php`

Every web request to a Laravel app — `/`, `/users`, `/posts/42`, *everything* — is routed by the web server to a **single PHP file**: `public/index.php`. This is the **front controller pattern**: one entry point that bootstraps the framework and dispatches the request. (Pretty URLs are produced by Apache's `.htaccess` or an Nginx `try_files` rule that funnels all non-file requests to `index.php`.)

Here is the request lifecycle in order:

1. **Web server** receives the HTTP request and hands it to `public/index.php`.
2. `index.php` loads Composer's autoloader (`vendor/autoload.php`) so all classes are available.
3. It loads `bootstrap/app.php`, which **creates the application (service container)** instance and registers core bindings, middleware, and routing (configured fluently since Laravel 11).
4. `Request::capture()` builds an `Illuminate\Http\Request` from the PHP super-globals, and `$app->handleRequest(...)` hands it to the framework's **HTTP kernel** (`Illuminate\Foundation\Http\Kernel`).
5. The kernel runs its **bootstrappers** (load env, load config, register & boot all **service providers** — the heart of Laravel's bootstrap), then sends the request through **global middleware** (e.g. maintenance-mode check, trim strings, session start, CSRF).
6. The **router** matches the URL to a route, runs any route/group middleware, and invokes its controller/closure.
7. The controller returns a **response** (HTML, JSON, redirect), which travels back out through the middleware and is sent to the browser via the response's `send()` method.

A simplified view of `public/index.php`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

// 1. If the app is in maintenance mode, serve the maintenance page.
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}

// 2. Register the Composer autoloader.
require __DIR__.'/../vendor/autoload.php';

// 3. Bootstrap the application and handle the request.
/** @var Application $app */
$app = require_once __DIR__.'/../bootstrap/app.php';

$app->handleRequest(Request::capture());
```

> **Interview-grade takeaway:** "Where does a Laravel request start?" → `public/index.php`, the front controller. It loads the autoloader, boots the app from `bootstrap/app.php`, and dispatches the captured `Request` through middleware and the router.

---

## 10. Composer & autoloading in Laravel

Laravel relies on Composer for two things:

1. **Dependency management** — `composer.json` lists packages; `composer install` downloads them into `vendor/` and writes a `composer.lock` recording exact versions.
2. **Autoloading** — you never write `require 'SomeClass.php'`. Composer generates an **autoloader** that loads classes on demand based on their namespace.

Laravel uses the **PSR-4** autoloading standard: a namespace prefix maps to a directory. Look at `composer.json`:

```json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/",
      "Database\\Factories\\": "database/factories/",
      "Database\\Seeders\\": "database/seeders/"
    }
  }
}
```

This says: any class in the `App\` namespace lives under `app/`. So `App\Http\Controllers\PostController` is expected at `app/Http/Controllers/PostController.php`. When PHP encounters that class name, Composer's autoloader finds and loads the file automatically.

If you add a new class and PHP can't find it (a "class not found" error), or you edit `composer.json`'s autoload section, regenerate the autoloader:

```bash
composer dump-autoload
```

> **`use ... ::class`:** Modern Laravel prefers the `::class` constant — `[PostController::class, 'index']` resolves to the fully-qualified string `"App\Http\Controllers\PostController"` at compile time, so it survives refactors and IDE renames. Avoid the old string style `'PostController@index'`: the implicit-namespace form (`'PostController@index'`) was removed back in Laravel 8, and even the fully-qualified string (`'App\Http\Controllers\PostController@index'`) is discouraged because it isn't refactor-safe.

---

## 11. Versioning, LTS, and upgrade philosophy

- Laravel follows **semantic versioning** and ships a **major version roughly once a year** in Q1 (Laravel 10 on Feb 14 2023, 11 on March 12 2024, 12 on Feb 24 2025). Minor and patch releases ship far more often and **never** contain breaking changes.
- **Support window:** For a given major version, **bug fixes are provided for 18 months** and **security fixes for 2 years** from its release. For Laravel 12 that means bug fixes through ~Aug 2026 and security fixes through ~Feb 2027.
- **"LTS" today:** Historically Laravel had explicit Long-Term Support releases (e.g. Laravel 6 LTS). The current policy drops the separate LTS label — **every major release gets the same 18-month bug-fix / 2-year security-fix window**, so each release is treated uniformly.
- **Upgrade philosophy:** Laravel deliberately keeps **breaking changes small** between majors (Laravel 12 itself was a "maintenance release" — most apps upgraded with no code changes), publishes a detailed **upgrade guide** for each, and offers **Laravel Shift** (a paid automated upgrade service). The team explicitly aims for upgrades to take **a day or less**, not a rewrite.

Check your installed version:

```bash
php artisan --version
```

```text
Laravel Framework 12.0.0
```

In `composer.json`, the framework is pinned with a caret constraint, which allows compatible updates within the major:

```json
{
  "require": {
    "php": "^8.2",
    "laravel/framework": "^12.0"
  }
}
```

> `^12.0` means "12.x but below 13.0" — you get minor/patch updates safely, never an unintended major jump. Run `composer update` to pull the latest allowed versions.

---

## 12. Tinker — the interactive REPL

**Tinker** is a **REPL** (Read–Eval–Print Loop) for your Laravel app — an interactive PHP shell where the entire framework (Eloquent, config, helpers, your classes) is already booted. It's the fastest way to poke at your app without writing throwaway routes.

```bash
php artisan tinker
```

```php
>>> 2 + 2
=> 4

>>> config('app.name')
=> "Laravel"

>>> Str::slug('Hello World, Laravel!')
=> "hello-world-laravel"

>>> App\Models\User::count()
=> 0

>>> $u = new App\Models\User(['name' => 'Ada']);
>>> $u->name
=> "Ada"

>>> exit   // leave the shell
```

> Tinker is invaluable for trying an Eloquent query, checking a config value, or testing a helper before wiring it into real code. It uses the **PsySH** shell under the hood.

---

## 13. Your first route → controller → view ("Hello, World")

Let's wire the full MVC path: a URL, handled by a controller, rendering a Blade view.

### Step 1 — define a route

Routes for the browser live in `routes/web.php`. The simplest route returns a string directly:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;

Route::get('/hello', function () {
    return 'Hello, World!';
});
```

Visit `http://127.0.0.1:8000/hello` → the page shows `Hello, World!`.

But MVC says routes should delegate to controllers. Let's do that properly.

### Step 2 — generate a controller

```bash
php artisan make:controller GreetingController
```

```text
   INFO  Controller [app/Http/Controllers/GreetingController.php] created successfully.
```

Edit it to return a view, passing data:

```php
<?php
// app/Http/Controllers/GreetingController.php

namespace App\Http\Controllers;

class GreetingController extends Controller
{
    public function show(string $name = 'World')
    {
        // Pass data to the view as an associative array.
        return view('greeting', ['name' => $name]);
    }
}
```

### Step 3 — point the route at the controller

```php
// routes/web.php
use App\Http\Controllers\GreetingController;
use Illuminate\Support\Facades\Route;

// {name?} is an optional route parameter.
Route::get('/hello/{name?}', [GreetingController::class, 'show']);
```

### Step 4 — create the view

Views live in `resources/views/`. The name `'greeting'` maps to `resources/views/greeting.blade.php`.

```blade
{{-- resources/views/greeting.blade.php --}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Greeting</title>
</head>
<body>
    {{-- {{ }} echoes a value with automatic HTML-escaping (XSS-safe). --}}
    <h1>Hello, {{ $name }}!</h1>
</body>
</html>
```

### Step 5 — see it work

```text
http://127.0.0.1:8000/hello        →  <h1>Hello, World!</h1>
http://127.0.0.1:8000/hello/Ada    →  <h1>Hello, Ada!</h1>
```

You just completed the round trip: **Route → Controller → View**, with a parameter flowing from the URL all the way into rendered, escaped HTML. That's the heartbeat of every Laravel page.

> **Why `{{ }}` and not `<?= ?>`:** Blade's `{{ $name }}` runs the value through `htmlspecialchars()` automatically, preventing **XSS** (cross-site scripting) attacks. Use `{!! $html !!}` only when you *intentionally* want raw, trusted HTML.

---

## ⚠️ Common Mistakes & Gotchas

1. **Calling `env()` outside config files.**
   *Symptom:* works locally, returns `null` in production.
   *Cause:* `php artisan config:cache` stops `.env` from loading; afterward `env()` only sees real system-level environment variables, so any key that lived only in `.env` reads as `null` everywhere except where it was captured inside `config/*.php`.
   *Fix:* Read values via `config('services.x.key')`. Put the `env()` call only inside a `config/*.php` file. If you change `.env` after caching, run `php artisan config:clear` (or `optimize:clear`).

2. **Pointing the web server at the project root instead of `public/`.**
   *Symptom:* visitors can download `.env`, see `vendor/`, or get a raw directory listing.
   *Fix:* Set the document root to the `public/` directory. Only `public/` should be web-accessible.

3. **Forgetting `php artisan key:generate` / missing `APP_KEY`.**
   *Symptom:* `RuntimeException: No application encryption key has been specified.`
   *Fix:* Run `php artisan key:generate`. `composer create-project` and `laravel new` normally do this for you; manual clones from Git won't (because `.env` isn't committed) — copy `.env.example` to `.env`, then generate the key.

4. **Editing files inside `vendor/` or committing `vendor/` to Git.**
   *Symptom:* your fix vanishes after `composer install`/`update`, or your repo is bloated and merge-conflict-prone.
   *Fix:* Never edit `vendor/`. Override behavior properly (config, service providers, package config publishing). Keep `vendor/` git-ignored; run `composer install` to rebuild it.

5. **Wondering why a config or route change "doesn't take effect" in production.**
   *Symptom:* you edit `.env`/config/routes but the app behaves as before.
   *Cause:* cached config/routes (`config:cache`, `route:cache`).
   *Fix:* `php artisan optimize:clear` to wipe all caches, or re-run the relevant `*:cache` command after the change.

6. **Forgetting to run a Vite process for assets.**
   *Symptom:* unstyled page, or `Unable to locate file in Vite manifest`.
   *Fix:* run `npm run dev` during development, or `npm run build` to compile a production bundle before deploying.

---

## ✅ Best Practices

- **`env()` only in `config/*.php`.** Everywhere else, use `config()`. This is the single most important rule from this module.
- **Commit `.env.example`, never `.env`.** Keep secrets out of version control; document required keys in the example file. If you must version an env file, encrypt it with `php artisan env:encrypt`.
- **Set `APP_DEBUG=false` and `APP_ENV=production` in production.** Leaving debug on leaks stack traces, config, and env values to end users.
- **Document root = `public/`.** Never expose the project root to the web.
- **Run `php artisan optimize` in production** (config/route/view caching) and `optimize:clear` whenever you change cached inputs.
- **Use the `::class` constant** for controllers, models, and bindings instead of magic strings.
- **Follow Laravel's conventions** (singular `Post` model → `posts` table; controller actions named `index/show/store/update/destroy`). Conventions unlock auto-wiring.
- **Keep controllers thin.** Push business logic into models, form requests, jobs, or service classes; controllers should coordinate, not compute.
- **Pin the framework with `^12.0`** and read the official upgrade guide each major release; upgrades are intentionally small.
- **Use Tinker to experiment** before committing code, and `php artisan about` to sanity-check your environment.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is Laravel and why would you choose it over raw PHP or another framework?**
A. Laravel is an expressive, "batteries-included" PHP MVC framework. It bundles routing, the Eloquent ORM, Blade templating, auth, queues, caching, and a huge first-party ecosystem (Sanctum, Horizon, Telescope, Cashier, Forge). It favors convention over configuration and developer ergonomics, with excellent docs and community — so you ship features instead of reinventing plumbing, with security best practices baked in.

**Q2. Explain MVC and how Laravel implements it.**
A. Model–View–Controller separates concerns: Models (Eloquent) hold data/business rules, Views (Blade) render output, Controllers coordinate by taking the request, querying models, and returning a view/response. It keeps SQL out of templates and HTML out of data logic, improving testability and maintainability.

**Q3. What's the difference between `env()` and `config()`, and why must you avoid `env()` outside config files? (under-the-hood)**
A. `env()` reads raw values from the parsed `.env` (or real system env vars); `config()` reads resolved values from the `config/*.php` arrays. In production you run `php artisan config:cache`, which serializes all config into one file and **stops loading `.env` entirely** at request time. After that, `env()` only returns true system-level environment variables — so a key that lived only in `.env` reads as `null` outside config files, while `config()` still works because that env value was captured into the cache when the config was built. So: `env()` only inside `config/*.php`; `config()` everywhere else.

**Q4. Walk me through what happens when a request hits a Laravel app. (under-the-hood)**
A. The web server funnels every request to the front controller `public/index.php`. It loads `vendor/autoload.php`, then `bootstrap/app.php` to create the application/service-container instance (which configures middleware and routing — in Laravel 11/12 fluently in that file). `Request::capture()` builds the `Request`, and `$app->handleRequest()` passes it to the framework's HTTP kernel, which boots the service providers, runs the request through global and route middleware, lets the router match a route and invoke its controller, and then sends the returned response back out through middleware to the browser.

**Q5. What are Laravel's installation requirements?**
A. PHP 8.2–8.5 for Laravel 12 (8.4 recommended/current), Composer, and a set of standard PHP extensions (Mbstring, OpenSSL, PDO, Tokenizer, Ctype, cURL, XML, etc.). Node/npm (or Bun) if you need to build front-end assets with Vite. Laravel 11/12 default to a SQLite database so a fresh app runs with no DB setup.

**Q6. What changed structurally between Laravel 10 and 11/12?**
A. Laravel 11 introduced a slimmed skeleton: the *app-level* HTTP/Console Kernel classes, default middleware classes, and the exception handler were removed from `app/`; their configuration moved into `bootstrap/app.php` (note the framework's own HTTP kernel still exists internally). The service-provider list moved from `config/app.php` to `bootstrap/providers.php`. `api.php`/`channels.php` are no longer present by default (added via `php artisan install:api` / `install:broadcasting`), Pest is the default test runner, and SQLite is the default DB. Laravel 12 keeps this structure and adds new React/Vue/Svelte/Livewire starter kits (Breeze/Jetstream are no longer updated).

**Q7. How does autoloading work in Laravel?**
A. Composer generates a PSR-4 autoloader from the `autoload.psr-4` map in `composer.json` (`App\` → `app/`). Classes load on demand by namespace, so you never `require` files manually. After adding classes or changing the map, run `composer dump-autoload`.

**Q8. What is Artisan and what are common commands?**
A. Artisan is Laravel's CLI (`php artisan ...`). Common: `serve`, `make:controller/model/migration`, `migrate`, `migrate:fresh --seed`, `route:list`, `config:cache`/`optimize`/`optimize:clear`, `key:generate`, `tinker`, `about`. You can also author custom commands.

**Q9. What is Tinker and when do you use it?**
A. A REPL (built on PsySH) that boots your whole app, letting you run Eloquent queries, read config, and test helpers interactively — ideal for quick experiments and debugging without writing temporary routes.

**Q10. How does Laravel handle versioning and upgrades?**
A. A major release roughly yearly with semantic versioning; ~18 months of bug fixes and ~2 years of security fixes per major. Breaking changes are kept small, each release has an upgrade guide, and tools like Laravel Shift automate upgrades. You pin `laravel/framework` as `^12.0` so minor/patch updates are safe.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# --- Install ---
composer global require laravel/installer   # one-time
laravel new myapp                            # via installer (interactive in L12)
composer create-project laravel/laravel myapp  # via Composer

# --- First run ---
php artisan key:generate        # create APP_KEY (installer usually does this)
composer run dev                # serve + queue + logs + Vite, all at once (recommended)
php artisan serve               # OR just the PHP dev server (http://127.0.0.1:8000)
npm install && npm run dev      # OR just the Vite dev server

# --- Daily Artisan ---
php artisan about               # environment overview
php artisan config:show app     # dump resolved config for a file
php artisan route:list          # list routes
php artisan make:controller XController
php artisan make:model X -mfc   # model + migration + factory + controller
php artisan migrate             # run migrations
php artisan migrate:fresh --seed
php artisan tinker              # REPL

# --- Caching (prod) ---
php artisan optimize            # cache config + routes + views + events
php artisan optimize:clear      # clear everything (use after .env/config edits)
php artisan config:clear        # clear just config cache

# --- Composer ---
composer install                # install from composer.lock
composer update                 # update within version constraints
composer dump-autoload          # rebuild the autoloader
```

| Concept | Key fact |
|---|---|
| Laravel 12 PHP requirement | PHP **8.2 – 8.5** (8.4 current target) |
| Laravel 12 support | Bug fixes ~18 months, security fixes ~2 years |
| Entry point | `public/index.php` (front controller) |
| HTTP kernel | Still exists in framework (`Illuminate\Foundation\Http\Kernel`); no app-level `Http/Kernel.php` |
| Service providers list | `bootstrap/providers.php` (L11/12; was `config/app.php` in L10) |
| Your code | `app/` (Models, Http/Controllers, Providers) |
| Routes | `routes/web.php`, `console.php`, (opt) `api.php` |
| Config | `config/*.php` — the **only** place for `env()` |
| Secrets/env | `.env` (git-ignored); commit `.env.example` |
| Read config in code | `config('file.key')` — never `env()` |
| Default DB (L11/12) | SQLite at `database/database.sqlite` |
| Default test runner (L11/12) | Pest (PHPUnit selectable) |
| Asset bundler | Vite (`@vite`, `npm run dev`/`build`) |
| Dependency manager | Composer (PSR-4 autoload, `App\` → `app/`) |
| REPL | `php artisan tinker` |
| Version pin | `"laravel/framework": "^12.0"` |

```text
.env → env() inside config/*.php → config() everywhere in your app
Browser → public/index.php → bootstrap/app.php → middleware → router → Controller → View → response
```

---

## 🧪 Mini Exercises

1. **Fresh install.** Create a new Laravel 12 project with `laravel new` (or `composer create-project`), run `php artisan serve`, and confirm the welcome page loads. Then run `php artisan about` and note your PHP and Laravel versions.

2. **The env gotcha, proven.** Add a key `GREETING_PREFIX="Hi"` to `.env`. Expose it through `config/app.php` as `'greeting_prefix' => env('GREETING_PREFIX', 'Hello')`. In a route, echo `config('app.greeting_prefix')`. Now run `php artisan config:cache`, change the `.env` value, refresh — observe it does **not** change until you run `php artisan config:clear`. Write one sentence explaining why.

3. **Full MVC round trip.** Build a `/profile/{username}` route → `ProfileController@show` → a `profile.blade.php` view that prints the username inside an `<h1>`, with a default of `guest` when no username is given. Confirm both `/profile` and `/profile/ada` render correctly.

4. **Directory scavenger hunt.** Without looking back at this module, write down what each of these is for: `bootstrap/app.php`, `storage/logs/`, `database/migrations/`, `resources/views/`, `vendor/`. Then verify against §4.

5. **Tinker drills.** Open `php artisan tinker` and (a) print `config('app.timezone')`, (b) generate a URL-friendly slug from a sentence using `Str::slug(...)`, and (c) count the rows in the `users` table with `App\Models\User::count()`. Note what each returns.
