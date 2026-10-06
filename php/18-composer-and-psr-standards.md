# Composer & PSR Standards

Composer is the dependency manager that the entire modern PHP ecosystem is built on, and the **PSRs** (PHP Standards Recommendations) are the shared conventions that let code from thousands of independent authors interoperate. Laravel itself is just a collection of Composer packages glued together with PSR-compliant interfaces. If you understand Composer and the PSRs, you understand the *plumbing* of every PHP project you will ever touch.

> **What you'll learn**
> - What Composer is, the problem it solves, and how it differs from a package *repository* like Packagist
> - The full anatomy of `composer.json` — `require`, `require-dev`, `autoload`, `scripts`, `config`
> - Semantic versioning and every constraint operator (`^`, `~`, `*`, ranges, `@stable`)
> - `install` vs `update`, why you **commit** `composer.lock`, and how the `vendor/` directory and autoloader actually work
> - The autoload mechanisms — PSR-4, classmap, and `files` — plus `dump-autoload --optimize`
> - Platform requirements, global packages, and the basics of publishing your own package
> - The PSR landscape: PSR-1/PSR-12 (style), PSR-4 (autoloading), PSR-3 (logging), PSR-7 (HTTP messages), PSR-17 (HTTP factories), PSR-11 (container), PSR-15 (middleware)
> - Quality tooling: PHP-CS-Fixer, PHPStan, and Psalm

---

## 1. Why Composer Exists (the WHY before the HOW)

Before Composer (released 2012), reusing PHP code meant manually downloading `.zip` files, copying folders into your project, and writing your own `require_once` statements for every single file — in the right order, because a class that `extends` another must be loaded *after* its parent. If library A needed library B at version 2, and library C needed B at version 1, you were stuck resolving that conflict by hand. This was fragile, unscalable misery.

**Composer** is a *per-project dependency manager*. You declare *what* you want ("I need `monolog/monolog` version 3-ish") in a single file, and Composer figures out *which exact versions* of everything (including dependencies-of-dependencies, called **transitive dependencies**) are mutually compatible, downloads them, and generates an **autoloader** so you never write a manual `require` for a class again.

Two pieces of jargon to separate up front:

- **Composer** = the *tool* (a CLI program) that resolves and installs dependencies.
- **Packagist** (packagist.org) = the default public *repository* where packages are published, so Composer knows where to download them from. Composer is the shopper; Packagist is the warehouse.

It is "per-project" by design: unlike a system-wide package manager (`apt`, `brew`), each project gets its *own* `vendor/` folder. Two projects on the same machine can use different versions of the same library without conflict.

---

## 2. Getting Started

```bash
# Check it's installed (install from https://getcomposer.org if not)
composer --version
# Output: Composer version 2.8.x 2025-xx-xx ...

# Start a brand new project interactively
composer init

# Create a new project FROM an existing package (this is how you scaffold Laravel)
composer create-project laravel/laravel my-app
```

`composer create-project laravel/laravel my-app` downloads the `laravel/laravel` skeleton, runs `composer install` inside it, and gives you a ready-to-run Laravel 12 app. Note: the *application* skeleton is `laravel/laravel`; the actual framework code lives in the `laravel/framework` package that the skeleton requires.

---

## 3. Anatomy of `composer.json`

`composer.json` is the manifest — the human-authored source of truth. Here is a realistic, annotated example:

```json
{
    "name": "acme/blog",
    "description": "A demo blog application.",
    "type": "project",
    "license": "MIT",
    "keywords": ["blog", "demo"],
    "require": {
        "php": "^8.4",
        "laravel/framework": "^12.0",
        "monolog/monolog": "^3.7"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^2.0",
        "friendsofphp/php-cs-fixer": "^3.64"
    },
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/"
        },
        "files": [
            "app/helpers.php"
        ]
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "post-autoload-dump": [
            "@php artisan package:discover --ansi"
        ],
        "test": "phpunit",
        "lint": "php-cs-fixer fix --dry-run --diff",
        "analyse": "phpstan analyse"
    },
    "config": {
        "optimize-autoloader": true,
        "preferred-install": "dist",
        "sort-packages": true,
        "allow-plugins": {
            "pestphp/pest-plugin": true
        }
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

Section by section:

- **`require`** — runtime dependencies. Your app needs these in production. Note that `php` and `ext-*` (e.g. `ext-mbstring`) are valid "packages" here — these are **platform requirements**.
- **`require-dev`** — tools needed only for development/testing (test runners, static analysers, linters). These are *not* installed in production when you run `composer install --no-dev`.
- **`autoload`** — how your *own* code maps namespaces to directories (covered in depth in §7).
- **`autoload-dev`** — autoload rules used only in dev (your `Tests\` namespace), kept out of the production autoloader.
- **`scripts`** — named command shortcuts and **lifecycle hooks**. `post-autoload-dump` runs automatically after the autoloader is regenerated — this is how Laravel auto-discovers package service providers. `@php` is a Composer placeholder for the PHP binary; `@test` would reference another script.
- **`config`** — Composer's own behaviour. `sort-packages` keeps `require` alphabetised; `optimize-autoloader` builds an optimized classmap; `allow-plugins` is a security gate (since Composer 2.2, plugins must be explicitly trusted or they won't run).
- **`minimum-stability` / `prefer-stable`** — `stable` means don't pull `-dev`/`-beta` versions unless explicitly asked; `prefer-stable: true` says "prefer a stable version even if a constraint *could* match a pre-release."

---

## 4. Semantic Versioning & Constraints

PHP packages follow **SemVer** (Semantic Versioning): versions are `MAJOR.MINOR.PATCH` (e.g. `3.7.2`).

- **MAJOR** — breaking changes (your code may need updating).
- **MINOR** — new features, backward-compatible.
- **PATCH** — backward-compatible bug fixes.

Your constraints in `require` tell Composer how much it's allowed to upgrade. Master these operators:

| Constraint | Matches | Equivalent range | Use when |
|---|---|---|---|
| `^3.7` (caret) | `>=3.7.0 <4.0.0` | up to next MAJOR | **Default choice.** Trusts SemVer. |
| `~3.7` (tilde, 2 parts) | `>=3.7.0 <4.0.0` | up to next MAJOR | **same as `^3.7`** — with 2 parts, tilde does *not* pin minor |
| `~3.7.2` (tilde, 3 parts) | `>=3.7.2 <3.8.0` | patches within `3.7.x` | conservative: allow patch updates, block the next minor |
| `3.7.*` (wildcard) | `>=3.7.0 <3.8.0` | one MINOR line | like `~3.7.0` (note: lower bound is `.0`, unlike `~3.7.2`) |
| `3.7.2` (exact) | exactly `3.7.2` | — | rarely; too rigid |
| `>=3.7 <4.0` (range) | explicit bounds | — | fine-grained control |
| `^3.7 \|\| ^4.0` (or) | either range | — | supporting two majors |
| `*` | anything | — | **avoid** — dangerous |
| `dev-main` | the `main` git branch tip | — | unreleased/forked code |

**The caret vs tilde subtlety** is a favourite interview question:

```bash
^1.2.3   →  >=1.2.3 <2.0.0     # caret locks the leftmost non-zero digit (MAJOR here)
~1.2.3   →  >=1.2.3 <1.3.0     # tilde with 3 parts locks MINOR
~1.2     →  >=1.2.0 <2.0.0     # tilde with 2 parts locks MAJOR (acts like ^1.2)
^0.3.0   →  >=0.3.0 <0.4.0     # CARET TREATS 0.x SPECIALLY: locks MINOR, not MAJOR
```

That last line trips people up: for `0.x` versions, caret treats the *minor* as the breaking digit, because pre-1.0 packages are considered unstable and any minor bump may break things.

You almost always want `^` for libraries (you get bug fixes and features automatically, no breaking changes), and that is exactly what `composer require` writes for you by default.

---

## 5. `install` vs `update`, and the Lock File

This is the single most important operational concept in Composer.

```bash
composer install   # Reads composer.lock, installs EXACT versions listed there.
composer update    # Re-resolves composer.json, finds newest allowed versions,
                   # REWRITES composer.lock, then installs.
```

- **`composer.json`** says "I want `monolog ^3.7`" — a *range*.
- **`composer.lock`** says "we resolved that to exactly `monolog 3.7.2`, sha `abc...`" — a *snapshot* of the entire dependency tree, including transitive deps.

When you run `composer install`:

1. If `composer.lock` exists → install the exact pinned versions (it ignores the *ranges* in `composer.json` for resolution). Composer compares a content hash of `composer.json` against the one stored in the lock; if they differ, `composer install` prints a **warning** — *"The lock file is not up to date with the latest changes in composer.json. You may be getting outdated dependencies. Run update to update them."* — but it **does not fail**: it proceeds and installs the versions already in the lock. To make a stale lock a hard error in CI (exit code 1), run `composer validate --strict` (which includes a `--check-lock` step) *before* `composer install` as a separate, blocking pipeline stage.
2. If no lock exists → behaves like `update` and creates one.

When you run `composer update` → Composer ignores the lock, recomputes the newest versions allowed by your constraints, and writes a fresh lock.

**Why you MUST commit `composer.lock` to git (for applications):**

Because it guarantees every developer, your CI server, and production all install *byte-for-byte identical* dependencies. Without it, a teammate running `composer install` next month could silently get `monolog 3.9.0` (released after you) while you have `3.7.2`, producing the dreaded "works on my machine" bug. Committing the lock makes builds **reproducible**.

> **Library exception:** if you are publishing a *library* (a reusable package others depend on), you typically **gitignore** the lock file, because the lock of *your* library shouldn't dictate the resolved versions in the *consuming* application. Commit the lock for applications; ignore it for libraries.

```bash
# Update only ONE package (and its deps), leaving everything else pinned — the safe daily move:
composer update monolog/monolog

# Update only the lock hash without changing versions (after a manual composer.json edit):
composer update --lock

# See what WOULD change, without doing it:
composer update --dry-run

# Production deploy: deterministic, no dev tools, optimized autoloader:
composer install --no-dev --optimize-autoloader --no-interaction
```

---

## 6. The `vendor/` Directory & Autoloading Basics

When you install, Composer downloads everything into `vendor/` and generates `vendor/autoload.php`. Your application's entry point includes that one file:

```php
<?php
require __DIR__ . '/vendor/autoload.php';

// From here on, ANY class is loaded automatically the first time you reference it.
$logger = new Monolog\Logger('app');   // Composer finds & includes the file lazily.
```

`vendor/` is a **build artifact** — it is *gitignored*. You regenerate it from `composer.lock` with `composer install`. Never edit files inside `vendor/`; your changes vanish on the next install. (If you must patch a dependency, use a tool like `cweagans/composer-patches`.)

Under the hood, Composer registers a function with PHP's `spl_autoload_register()`. When PHP encounters a class it doesn't know, it calls that function with the class name; Composer's autoloader maps the name to a file path and `require`s it. This is *lazy loading* — files are only read from disk when actually needed.

---

## 7. Autoload Types: PSR-4, classmap, files

Composer supports three ways to map code to files. You declare them under `autoload`:

### PSR-4 (the modern default)

**PSR-4** is the standard that maps a **namespace prefix** to a **base directory**. The rule: replace the namespace separator `\` with a directory separator `/`, append `.php`.

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Acme\\Blog\\": "src/"
        }
    }
}
```

With this mapping:

```
App\Http\Controllers\PostController   →  app/Http/Controllers/PostController.php
Acme\Blog\Models\Post                 →  src/Models/Post.php
```

Rules that matter: the file's `namespace` must match, the class name must match the **file name exactly** (case-sensitive on Linux), and one file = one class. No central registry needs updating when you add a new class — it just works because the path is computed.

### classmap

A **classmap** scans given directories/files and builds an explicit `class name → file path` table at dump time. Use it for legacy code that *doesn't* follow PSR-4 (e.g. multiple classes per file, no namespaces).

```json
{
    "autoload": {
        "classmap": ["database/seeders", "legacy/lib/"]
    }
}
```

Because the map is precomputed, classmap is fast to *resolve* but you **must re-run `dump-autoload`** every time you add a class, or it won't be found.

### files

`files` are **eagerly loaded** on every request — they're `require`d when `vendor/autoload.php` is included, not lazily. Use this *only* for procedural code that defines functions or constants (which can't be autoloaded), like a global helpers file:

```json
{
    "autoload": {
        "files": ["app/helpers.php"]
    }
}
```

```php
<?php
// app/helpers.php — global helper functions can't be autoloaded, so they live in a files entry.
if (! function_exists('money')) {
    function money(int $cents): string {
        return '$' . number_format($cents / 100, 2);
    }
}
```

> After editing the `autoload` block in `composer.json`, you must regenerate the autoloader (`composer dump-autoload`) or new mappings won't take effect.

---

## 8. `dump-autoload` & Optimization

`composer dump-autoload` (alias `composer dumpautoload`) regenerates `vendor/autoload.php` and its support files **without** touching downloaded packages.

```bash
composer dump-autoload                 # rebuild after editing autoload rules or adding classmap classes

composer dump-autoload -o              # --optimize: pre-scan PSR-4 dirs into a static classmap (PSR-4 kept as fallback)
composer dump-autoload -a              # --classmap-authoritative: classmap is the ONLY source; no filesystem fallback (implies -o)
```

Why optimize in production? Normally a PSR-4 lookup does a filesystem `file_exists()` check to find the class file — that's I/O on every uncached class load. The `-o` flag pre-scans every PSR-4/PSR-0 directory into one big classmap (`vendor/composer/autoload_classmap.php`), turning each *known* lookup into a fast in-memory array read — but it **keeps the PSR-4 rules as a fallback** so classes added after the dump (or generated at runtime) still resolve via the filesystem. `--classmap-authoritative` (`-a`, which implies `-o`) goes further: it drops that fallback entirely — "the classmap is complete; if a class isn't in it, it doesn't exist." Use `-a` only when you're certain no classes are generated or added at runtime. In Laravel, `composer install --optimize-autoloader --no-dev` on deploy handles this for you.

---

## 9. Adding, Removing, Inspecting Packages

```bash
# Add a runtime dependency (writes ^constraint into composer.json, updates lock, installs)
composer require guzzlehttp/guzzle

# Add a dev-only dependency
composer require --dev phpstan/phpstan

# Pin a specific version range at add time
composer require monolog/monolog:^3.7

# Remove a package (cleans composer.json, lock, and vendor/)
composer remove guzzlehttp/guzzle

# Inspect the installed tree
composer show                       # all installed packages + versions
composer show monolog/monolog       # details for one package
composer show --tree                # dependency tree
composer why monolog/monolog        # WHO requires this package (reverse lookup)
composer why-not monolog/monolog 4  # explain why you can't upgrade to 4.x

# Find outdated packages
composer outdated                   # all with newer versions
composer outdated --direct          # only your direct deps (ignores transitive)

# Validate your manifest before committing
composer validate

# Security audit against known CVEs
composer audit
```

`composer why` and `composer why-not` are gold for debugging "I can't update X" situations — they explain the constraint conflict precisely.

---

## 10. Global Packages

Some packages are *developer tools* you want available everywhere, not per-project. Install them globally:

```bash
composer global require laravel/installer

# Make global binaries runnable — add this to your shell profile (~/.zshrc / ~/.bashrc).
# The global home differs by platform: macOS + legacy *nix use ~/.composer; modern Linux
# following the XDG spec uses ~/.config/composer. Don't guess — ask Composer:
export PATH="$PATH:$(composer global config bin-dir --absolute 2>/dev/null)"
# (Equivalently, hardcode ~/.composer/vendor/bin on macOS or ~/.config/composer/vendor/bin on XDG Linux.)
```

Now `laravel new my-app` works anywhere. **Caution:** global installs share one dependency tree, so two global tools needing incompatible versions of the same library will conflict. For project-bound tools (PHPUnit, PHPStan), prefer `require-dev` over global — that keeps each project self-contained and reproducible.

---

## 11. Platform Requirements

You can declare the runtime your code needs, and Composer enforces it:

```json
{
    "require": {
        "php": "^8.4",
        "ext-mbstring": "*",
        "ext-pdo": "*"
    }
}
```

If a developer on PHP 8.1 runs `composer install`, Composer **refuses** with a clear platform error rather than letting them install code that will crash at runtime.

When your *local* PHP differs from production (e.g. you're on 8.4 but deploy to 8.2), pin the target so resolution matches production:

```json
{
    "config": {
        "platform": {
            "php": "8.2.0"
        }
    }
}
```

```bash
# Check what Composer thinks the platform provides
composer check-platform-reqs

# Escape hatch — install despite a missing/wrong platform (USE SPARINGLY)
composer install --ignore-platform-req=ext-gd
```

---

## 12. Packagist & Publishing Your Own Package

**Packagist.org** is the central public repository. When you `composer require vendor/name`, Composer queries Packagist's metadata to find the package, then downloads the actual code (a "dist" zip) usually from GitHub.

To create and publish your own package:

**1. Structure it as a library** with a `composer.json` of `type: library`:

```json
{
    "name": "acme/calculator",
    "description": "A tiny calculator library.",
    "type": "library",
    "license": "MIT",
    "require": {
        "php": "^8.4"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "autoload": {
        "psr-4": {
            "Acme\\Calculator\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Acme\\Calculator\\Tests\\": "tests/"
        }
    }
}
```

```php
<?php
// src/Calculator.php
namespace Acme\Calculator;

final class Calculator
{
    public function add(int|float ...$numbers): int|float
    {
        return array_sum($numbers);
    }
}
```

**2. Tag a release.** Versions come from git tags, not `composer.json` — never hardcode a `version` field:

```bash
git tag v1.0.0
git push --tags
```

**3. Submit the repo URL to Packagist** (one-time), then set up the **GitHub webhook** Packagist provides so new tags auto-update.

> **If your package ships a command-line tool**, declare it with the top-level `bin` key (an array of script paths). Composer symlinks each entry into the consumer's `vendor/bin/`, making it runnable as `vendor/bin/your-tool`:
>
> ```json
> { "bin": ["bin/calculator"] }
> ```
>
> This is exactly how `phpunit`, `php-cs-fixer`, and `phpstan` end up in your `vendor/bin/`.

To use a package *before* it's on Packagist (private repo, fork, local dev), add a custom `repositories` entry:

```json
{
    "repositories": [
        { "type": "vcs", "url": "https://github.com/acme/calculator" },
        { "type": "path", "url": "../calculator" }
    ],
    "require": {
        "acme/calculator": "@dev"
    }
}
```

`type: path` symlinks a sibling folder — invaluable when developing a package and an app side by side.

---

## 13. The PSR Landscape

The **PSRs** are published by **PHP-FIG** (the PHP Framework Interop Group). Each is a numbered recommendation. Their power is *interoperability*: a PSR is usually an **interface** living in a `psr/*` package, so any library can implement it and any consumer can type-hint against it without coupling to a concrete vendor. "Program to an interface, not an implementation" — at the ecosystem scale.

### Style: PSR-1 & PSR-12

- **PSR-1** — basic coding standard: files use `<?php`, classes in `StudlyCaps`, methods in `camelCase`, one class per file.
- **PSR-12** — the extended coding style (supersedes the deprecated PSR-2): 4-space indentation, opening brace on its own line for classes/methods, visibility declared on all properties/methods, one blank line after `namespace`, etc.

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Psr\Log\LoggerInterface;

final class ReportService
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function generate(int $userId): string
    {
        $this->logger->info('Generating report', ['user' => $userId]);

        return match (true) {
            $userId > 0 => "report-{$userId}",
            default     => 'report-anon',
        };
    }
}
```

> PSR-12 is a *style* spec — you don't import it. Tools like **PHP-CS-Fixer** and **PHP_CodeSniffer** enforce it automatically (see §14).

### PSR-4 — Autoloading

Covered in §7. It's the standard that makes `App\Http\Controllers\PostController` resolve to `app/Http/Controllers/PostController.php`.

### PSR-3 — Logger Interface

`Psr\Log\LoggerInterface` defines the eight RFC-5424 severity methods (`emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info`, `debug`) plus a generic `log()`. Monolog implements it; Laravel's `Log` facade resolves to a PSR-3 logger. Because the interface is standard, you type-hint `LoggerInterface` and swap implementations freely:

```php
<?php
use Psr\Log\LoggerInterface;

final class PaymentProcessor
{
    public function __construct(private readonly LoggerInterface $log) {}

    public function charge(int $cents): void
    {
        // Note the {placeholder} context interpolation defined by PSR-3:
        $this->log->info('Charging {amount} cents', ['amount' => $cents]);
    }
}
// Inject Monolog, Laravel's logger, or a test spy — the class never changes.
```

### PSR-7 — HTTP Message Interfaces

Defines value objects for HTTP: `RequestInterface`, `ResponseInterface`, `ServerRequestInterface`, `UriInterface`, `StreamInterface`. They are **immutable** — "modifying" returns a *new* instance via `with*()` methods. Guzzle and many frameworks speak PSR-7. (Laravel's native `Illuminate\Http\Request` is *not* PSR-7, but you can bridge it via `symfony/psr-http-message-bridge`.)

```php
<?php
use Psr\Http\Message\ResponseInterface;

function addHeader(ResponseInterface $response): ResponseInterface
{
    // Immutable: withHeader() returns a NEW response; $response is unchanged.
    return $response->withHeader('X-Powered-By', 'PSR-7');
}
```

### PSR-11 — Container Interface

`Psr\Container\ContainerInterface` standardises **dependency injection containers** with two methods: `get(string $id)` and `has(string $id): bool`. Laravel's service container implements it, so framework-agnostic libraries can resolve services from *any* compliant container.

```php
<?php
use Psr\Container\ContainerInterface;

function resolveMailer(ContainerInterface $container): object
{
    if ($container->has('mailer')) {
        return $container->get('mailer');
    }
    throw new RuntimeException('No mailer registered.');
}
```

### PSR-15 — HTTP Server Request Handlers (Middleware)

Builds on PSR-7. Defines `MiddlewareInterface` and `RequestHandlerInterface` — the standard middleware "onion" pattern: each middleware gets the request and a handler, and decides whether to pass through or short-circuit.

```php
<?php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

final class AuthMiddleware implements MiddlewareInterface
{
    // PSR-7 only defines INTERFACES — there is no concrete `Response` class in the
    // standard. To build a response you inject a PSR-17 ResponseFactoryInterface
    // (e.g. from nyholm/psr7 or guzzlehttp/psr7), not `new Response(...)`.
    public function __construct(
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler,
    ): ResponseInterface {
        if (! $request->hasHeader('Authorization')) {
            // Short-circuit without calling the next handler.
            $response = $this->responseFactory->createResponse(401);
            $response->getBody()->write('Unauthorized');

            return $response;
        }

        return $handler->handle($request);   // Pass to the next layer.
    }
}
```

> **PSR-17** (`Psr\Http\Message\ResponseFactoryInterface`, `RequestFactoryInterface`, `StreamFactoryInterface`, …) is the companion "HTTP Factories" spec that exists precisely *because* PSR-7 ships no concrete classes. Creating a PSR-7 message means either instantiating a concrete implementation (`new Nyholm\Psr7\Response(...)`) or — the decoupled way shown above — depending on a PSR-17 factory.

Laravel's own middleware predates PSR-15 and uses a `handle($request, Closure $next)` signature instead — conceptually identical, but not PSR-15-typed. Knowing both is interview gold.

---

## 14. Quality Tooling (install via `require-dev`)

These aren't PSRs, but they enforce/verify the standards above and are expected on any serious team.

```bash
composer require --dev friendsofphp/php-cs-fixer   # auto-formats to PSR-12 (and beyond)
composer require --dev phpstan/phpstan             # static analysis: finds type bugs without running code
composer require --dev vimeo/psalm                 # alternative static analyser, strong on type inference
```

```bash
# PHP-CS-Fixer: a FORMATTER — rewrites code to match a rule set
vendor/bin/php-cs-fixer fix                # apply fixes
vendor/bin/php-cs-fixer fix --dry-run --diff   # CI mode: show what's wrong, change nothing

# PHPStan: a static ANALYSER — levels 0 (lax) to 10 (strict in v2)
vendor/bin/phpstan analyse src --level=6

# Psalm: similar, with --show-info and taint analysis
vendor/bin/psalm
```

**The distinction matters:** PHP-CS-Fixer changes *how code looks* (style). PHPStan/Psalm analyse *whether code is correct* (types, null-safety, dead code) — without executing it. They are complementary; many teams run all three in CI. For Laravel specifically, **Larastan** (`larastan/larastan`) extends PHPStan to understand Eloquent magic, and **Laravel Pint** (`laravel/pint`, ships with Laravel 12) is an opinionated PHP-CS-Fixer wrapper preconfigured for the framework.

---

## ⚠️ Common Mistakes & Gotchas

1. **Committing `vendor/` or NOT committing `composer.lock`.**
   `vendor/` is a regenerable artifact — gitignore it. `composer.lock` is the reproducibility guarantee — **commit it** (for applications). The classic blunder is reversing these, leading to bloated repos and non-reproducible builds.
   **Fix:** `.gitignore` should contain `/vendor`, and `composer.lock` should be tracked.

2. **Running `composer update` on production / during routine deploys.**
   `update` re-resolves to the newest allowed versions, defeating the entire point of the lock file and potentially shipping untested upgrades.
   **Fix:** Deploy with `composer install --no-dev --optimize-autoloader`. Only run `update` deliberately, locally, then commit the new lock and let CI test it.

3. **Adding a new class to a `classmap` (or editing `autoload` rules) and getting "class not found".**
   Classmaps and the optimized autoloader are precomputed snapshots; new classes aren't auto-detected.
   **Fix:** Run `composer dump-autoload` after adding classmap classes or changing the `autoload` section.

4. **Misunderstanding caret on `0.x` versions.**
   Developers assume `^0.3` allows up to `<1.0`, but it actually resolves to `>=0.3.0 <0.4.0` because caret treats the first non-zero segment as the breaking one.
   **Fix:** For `0.x` packages, read constraints carefully and pin conservatively; expect minor bumps to break.

5. **Using `*` or `dev-main` constraints in production.**
   `*` accepts *any* version (including future breaking majors); `dev-main` tracks a moving branch with no stability guarantee.
   **Fix:** Use `^` against a tagged stable release. Reserve `dev-*` for short-lived development of your own forks.

6. **Putting test/lint tools in `require` instead of `require-dev`.**
   This ships PHPUnit, PHPStan, etc. to production, bloating the install and widening the attack surface.
   **Fix:** Test/analysis/format tooling always goes in `require-dev`; production deploys use `--no-dev`.

---

## ✅ Best Practices

- **Commit `composer.lock` for apps; gitignore it for libraries.** Apps want reproducibility; libraries must not dictate consumers' resolution.
- **Default to `^` constraints** against the latest stable release — you get patches and features, no breaking changes.
- **Deploy with `composer install --no-dev --optimize-autoloader --no-interaction`** for fast, lean, deterministic production installs.
- **Run `composer outdated --direct`, `composer audit`, and `composer validate --strict` in CI** to catch stale deps, known CVEs, and a malformed or out-of-sync manifest/lock early. (Plain `composer install` only *warns* on a stale lock; `validate --strict` makes it a build failure.)
- **Update one package at a time** (`composer update vendor/pkg`) rather than blanket updates, so blast radius is small and reviewable.
- **Type-hint PSR interfaces** (`LoggerInterface`, `ContainerInterface`) rather than concrete classes — your code stays swappable and testable.
- **Set `config.platform.php`** to your production PHP version so local resolution matches production.
- **Enforce PSR-12 with Pint/PHP-CS-Fixer and correctness with PHPStan/Larastan in CI** — make `--dry-run`/`analyse` a required check, not a suggestion.
- **Pin `allow-plugins` explicitly** and review what each plugin does — Composer plugins run arbitrary code at install time.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `composer install` and `composer update`?**
`install` reads `composer.lock` and installs those exact versions (reproducible). `update` ignores the lock, re-resolves the constraints in `composer.json` to the newest allowed versions, rewrites the lock, then installs. You run `install` everywhere; `update` only deliberately when bumping deps.

**Q2. Why commit `composer.lock`, and when would you *not*?**
Commit it for applications so every environment installs identical versions — reproducible builds, no "works on my machine." Don't commit it for *libraries*, because the library's lock shouldn't override the resolved versions in the consuming app.

**Q3. Explain caret vs tilde.**
`^1.2.3` → `>=1.2.3 <2.0.0` (locks MAJOR, allows minors+patches). `~1.2.3` → `>=1.2.3 <1.3.0` (locks MINOR, patches only). `~1.2` → `>=1.2.0 <2.0.0`. Gotcha: `^0.3.0` → `>=0.3.0 <0.4.0` because caret treats `0.x` minors as breaking.

**Q4. How does Composer autoloading work under the hood?**
Composer generates `vendor/autoload.php`, which registers a callback via `spl_autoload_register()`. When PHP hits an unknown class, it invokes the callback; Composer's `ClassLoader` consults its PSR-4 prefix map (or a precomputed classmap) to compute the file path, then `require`s it — lazily, on first reference. `dump-autoload -o` precomputes everything into a static classmap to skip filesystem `file_exists` checks; `-a` (authoritative) makes that classmap the sole source of truth.

**Q5. PSR-4 vs classmap vs files — when do you use each?**
PSR-4 for modern namespaced code (computed path, no re-dump needed). Classmap for legacy code that doesn't follow PSR-4 (must re-dump on new classes). `files` for procedural code defining functions/constants that *can't* be autoloaded — loaded eagerly on every request (e.g. a helpers file).

**Q6. What problem do PSRs solve, and how are they usually delivered?**
Interoperability. A PSR like PSR-3/PSR-7/PSR-11 ships as an *interface* in a tiny `psr/*` package. Libraries implement the interface; consumers type-hint it — so you can swap Monolog for another logger, or Guzzle for another HTTP client, without changing dependent code.

**Q7. Difference between PHP-CS-Fixer and PHPStan?**
PHP-CS-Fixer is a *formatter* — it rewrites code to satisfy a *style* standard (PSR-12). PHPStan/Psalm are *static analysers* — they inspect types and logic to find *bugs* without running the code. Style vs correctness; teams run both.

**Q8. What's Packagist, and how is it different from Composer?**
Composer is the CLI dependency-resolution *tool*; Packagist is the default public *repository* of package metadata it queries. You can also add private/VCS/path repositories via the `repositories` key.

**Q9. How would you optimize a Composer install for production?**
`composer install --no-dev --optimize-autoloader --no-interaction`. `--no-dev` drops `require-dev`; `--optimize-autoloader` (or `dump-autoload -o`) builds a static classmap so class lookups are array reads, not filesystem checks. Add `--classmap-authoritative` if no classes are generated at runtime.

**Q10. How do platform requirements work, and how do you handle a local/prod PHP mismatch?**
Declaring `"php": "^8.4"` (and `ext-*`) makes Composer refuse to install on an incompatible runtime. If local PHP differs from prod, set `config.platform.php` to the production version so resolution targets prod; verify with `composer check-platform-reqs`.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# --- Lifecycle ---
composer init                              # create composer.json interactively
composer install                           # install exact versions from lock
composer update                            # re-resolve + rewrite lock
composer update vendor/pkg                 # update one package only
composer install --no-dev -o --no-interaction   # production deploy

# --- Packages ---
composer require vendor/pkg                # add runtime dep (^ by default)
composer require --dev vendor/pkg          # add dev dep
composer require vendor/pkg:^3.7           # add with constraint
composer remove vendor/pkg                 # remove
composer global require vendor/pkg         # install globally

# --- Inspect / debug ---
composer show [--tree]                     # list installed
composer why vendor/pkg                     # who requires it
composer why-not vendor/pkg 4              # why can't I upgrade
composer outdated --direct                 # stale direct deps
composer audit                             # security CVEs
composer validate --strict                 # lint composer.json + fail on out-of-sync lock
composer check-platform-reqs               # platform check

# --- Autoload ---
composer dump-autoload                     # regenerate autoloader
composer dump-autoload -o                  # optimized classmap (PSR-4 kept as fallback)
composer dump-autoload -a                  # classmap authoritative (no fallback; implies -o)
```

```text
# --- Version constraints ---
^3.7      >=3.7.0 <4.0.0     (default; locks MAJOR)
~3.7.2    >=3.7.2 <3.8.0     (patch-only)
~3.7      >=3.7.0 <4.0.0     (locks MAJOR)
3.7.*     >=3.7.0 <3.8.0     (one minor line)
^0.3.0    >=0.3.0 <0.4.0     (0.x: locks MINOR!)
>=3 <5    explicit range
^3 || ^4  multiple majors
*         anything (avoid)
dev-main  branch tip (dev only)
```

```text
# --- Key PSRs ---
PSR-1   basic coding standard
PSR-12  extended coding style (PSR-2 deprecated)
PSR-3   LoggerInterface
PSR-4   namespace -> directory autoloading
PSR-7   HTTP message interfaces (immutable)
PSR-17  HTTP factories (build PSR-7 messages)
PSR-11  ContainerInterface (get/has)
PSR-15  MiddlewareInterface / RequestHandlerInterface
```

---

## 🧪 Mini Exercises

1. **Constraint reading.** Given `"laravel/framework": "^12.3", "monolog/monolog": "~3.5.0", "acme/x": "^0.2"`, write the exact min/max version range each one resolves to. Explain why the `acme/x` line is the odd one out.

2. **Build a package.** Create a `vendor/name` library with a `composer.json` exposing a PSR-4 namespace `Acme\\Utils\\` → `src/`, add one class with a static method, and write the exact commands to (a) tag `v1.0.0` and (b) consume it locally from a sibling app using a `type: path` repository.

3. **Autoload type triage.** You have: (a) a namespaced controller, (b) a legacy file with three classes and no namespace, (c) a file of global helper functions. State which autoload mechanism each needs and whether adding a new item of that type requires re-running `dump-autoload`.

4. **PSR in practice.** Write a class that depends on `Psr\Log\LoggerInterface` via constructor promotion and logs an `info` message using PSR-3 context interpolation (`{placeholder}`). Then describe two different concrete loggers you could inject without changing the class.

5. **Production hardening.** Write the single Composer command for a deterministic production deploy, then explain what each flag does and why `composer update` must never appear in a deploy script.
