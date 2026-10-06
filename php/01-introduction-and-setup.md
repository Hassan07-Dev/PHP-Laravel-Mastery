# Introduction to PHP & Environment Setup

> Module 01 of the **Zero to Expert** PHP/Laravel track. This is the foundation everything else stands on: what PHP actually *is*, how it runs your code, and how to get a clean, modern dev environment working on any OS.

**What you'll learn**

- What PHP is, where it runs, and why it still dominates the server-side web
- The journey from PHP 5 → 7 → 8, the big performance/feature wins, and what the JIT does
- How PHP executes your code under the hood (the Zend Engine, opcodes, SAPIs, the HTTP request lifecycle)
- How to install PHP 8.4 on macOS, Linux, and Windows — and how to verify it
- The most important `php.ini` directives and how to safely change them
- Running scripts from the CLI, spinning up the built-in dev server, and embedding PHP in HTML
- Core syntax: tags, the short echo tag, comments, statements, semicolons, and PHP's case-sensitivity rules
- A first "Hello World", a tiny dynamic page, and a 60-second tour of Composer

---

## 1. What PHP Is (and Why It Won the Web)

**PHP** stands for **PHP: Hypertext Preprocessor** (a recursive acronym; it originally meant "Personal Home Page"). It is a **server-side, general-purpose scripting language** whose original — and still primary — job is to generate dynamic web content.

The key word is **server-side**. When you visit a PHP-powered page, your browser never sees PHP code. The web server runs the PHP, the PHP produces HTML (or JSON, or a PDF, or whatever), and *that output* is what travels back over the network. Contrast this with JavaScript, which (in the browser) runs *client-side*, after the page arrives.

Why does this matter? Because server-side means:

- **Secrets stay secret.** Database passwords, API keys, and business logic never reach the client.
- **You control the environment.** One known PHP version on the server, instead of dozens of browser quirks.
- **It's "interpreted on demand."** Save a `.php` file, refresh the browser, see the change — no compile-and-deploy step during development.

**What people build with PHP today:**

- Content platforms — **WordPress** alone powers a large share of all websites on the internet.
- E-commerce — Magento and WooCommerce are the big PHP players. (Note: Shopify is *not* PHP — its backend is Ruby on Rails — a common interview trap, so don't list it as a PHP example.)
- Frameworks for serious applications — **Laravel** (this course's focus), Symfony, Slim.
- APIs and microservices, CLI tooling, queue workers, scheduled jobs, and more.

> **Jargon check.** A *scripting language* is one where you typically run source files directly rather than compiling them into a standalone binary first. *Dynamic* in "dynamic web content" means the page is generated per-request (e.g., showing *your* shopping cart), as opposed to a *static* `.html` file that is identical for everyone.

---

## 2. A Short History & The Versions That Matter

You don't need every release date, but you must understand the **eras**, because interviewers ask "what changed in PHP 8?" and because legacy codebases force you to know the differences.

| Era | Headline |
|---|---|
| PHP 5.x | The OOP era — real classes, interfaces, namespaces (5.3), traits (5.4). Slow by modern standards. |
| PHP 7.0 (2015) | **The performance revolution.** ~2x faster than 5.6, far lower memory use, thanks to a rewritten Zend Engine internal data structure (`zval`). Scalar type hints, return types, the `<=>` spaceship operator, null coalescing `??`. |
| PHP 7.1–7.4 | Nullable types, void return, `[$a, $b] = ...` destructuring, typed properties (7.4), arrow functions `fn()`, the null coalescing assignment `??=`. |
| PHP 8.0 (2020) | **The modern language.** The **JIT** compiler, named arguments, attributes, constructor property promotion, `match`, nullsafe `?->`, union types, `throw` as an expression, stricter type juggling. |
| PHP 8.1 | **Enums**, readonly properties, fibers, pure intersection types, `never` return type, `new` in initializers, first-class callable syntax `strlen(...)`. |
| PHP 8.2 | `readonly` classes, disjunctive normal form (DNF) types, deprecation of dynamic properties, standalone `null`/`false`/`true` types. |
| PHP 8.3 | Typed class constants, `#[\Override]` attribute, `json_validate()`, dynamic class constant fetch, deep-cloning of readonly properties. |
| **PHP 8.4 (2024)** | **Property hooks** (computed/guarded properties without boilerplate getters/setters), **asymmetric visibility** (`public private(set)`), new-without-parentheses (`new Foo()->bar()`), `array_find`/`array_any`/`array_all`, lazy objects, a new (opt-in) DOM API. |

### The 7 → 8 leap, in plain terms

PHP 7 made PHP *fast*. PHP 8 made PHP *expressive and safe*. The features that show up constantly in modern code:

```php
<?php
// PHP 8 idioms you'll use every day:

// 1. Constructor property promotion — declare + assign in one line
class Money {
    public function __construct(
        public readonly int $amount,      // readonly (8.1): set once, in constructor
        public readonly string $currency = 'USD',
    ) {}
}

// 2. Named arguments — order-independent, self-documenting
$price = new Money(amount: 999, currency: 'EUR');

// 3. match — like switch but returns a value and uses strict (===) comparison
$label = match ($price->currency) {
    'USD' => 'US Dollars',
    'EUR' => 'Euros',
    default => 'Unknown',
};

// 4. Nullsafe operator — short-circuits to null instead of erroring
$country = $user?->getAddress()?->country;   // no "call on null" error

echo $label; // Output: Euros
```

### What the JIT actually is

**JIT** = **Just-In-Time** compilation. Normally PHP compiles your source to **opcodes** (a low-level bytecode the Zend Engine executes — more on this below). The JIT goes a step further: at *runtime*, it compiles hot opcode paths directly into **native machine code**, skipping the engine's interpreter loop for those parts.

The honest truth interviewers like to hear:

- The JIT gives huge gains on **CPU-bound** work — number crunching, image processing, Mandelbrot-style benchmarks.
- It gives **modest or negligible** gains on **typical web apps**, which are dominated by I/O (database, network, disk), not raw CPU. For most Laravel apps, **OPcache** (caching the compiled opcodes so they aren't recompiled every request) matters far more than the JIT.

```ini
; Enabling the JIT in php.ini (it's off by default)
opcache.enable=1
opcache.jit_buffer_size=128M
opcache.jit=tracing   ; "tracing" is the recommended mode for most workloads
```

> **Version note (PHP 8.4):** the default of `opcache.jit` changed from `tracing` to `disable`. Before 8.4, simply setting a non-zero `opcache.jit_buffer_size` was enough to turn the JIT on; from 8.4 you **must** explicitly set `opcache.jit=tracing` (or another mode) as well. The JIT only runs in the engine that has OPcache loaded — for web traffic that's your FPM/web SAPI, not the CLI (the CLI JIT is gated separately by `opcache.enable_cli=1`, which you rarely need).

---

## 3. How PHP Executes Your Code (Under the Hood)

This section is gold for interviews. Understanding the pipeline separates juniors from people who can debug a production issue.

### The Zend Engine and the compilation pipeline

PHP source code is **not** run line-by-line as raw text. The runtime engine — the **Zend Engine** — processes every script in stages:

```
Your .php file
   │  1. Lexing (tokenizing): text → tokens  (T_ECHO, T_STRING, ...)
   ▼
   │  2. Parsing: tokens → an Abstract Syntax Tree (AST)
   ▼
   │  3. Compilation: AST → opcodes (Zend bytecode)
   ▼
   │  4. Execution: the Zend VM runs the opcodes  (+ optional JIT → machine code)
   ▼
Output (HTML / JSON / text)
```

People call PHP "interpreted," and that's a fair shorthand — but the precise statement is: **PHP compiles each script to opcodes on the fly, then executes those opcodes in a virtual machine.** There is no separate `.exe` you ship.

**Why OPcache matters:** without it, steps 1–3 run on *every single request*. OPcache stores the compiled opcodes in shared memory so that for subsequent requests PHP jumps straight to step 4. This is one of the single biggest performance wins for any real PHP site, and it's enabled by default in production-grade setups.

```bash
# Inspect what your opcache is doing
php -i | grep opcache.enable
# Output (example):
# opcache.enable => On => On
```

### SAPIs: the same language, different front doors

A **SAPI** (**Server API**) is the interface between PHP and whatever is invoking it. The *same* PHP code can run through different SAPIs:

| SAPI | What it is | When you use it |
|---|---|---|
| **CLI** | Command-Line Interface | Scripts, cron jobs, Laravel Artisan, queue workers, this course's exercises |
| **FPM** | FastCGI Process Manager — a pool of persistent PHP worker processes | The modern standard for serving web apps (paired with Nginx or Apache) |
| **Apache module** (`mod_php`) | PHP embedded directly inside Apache | Classic shared-hosting setup; simpler but less flexible than FPM |
| **Built-in server** | The dev server from `php -S` | Local development only — never production |

Find out which SAPI is active:

```php
<?php
echo php_sapi_name() . PHP_EOL;
// Output when run from terminal:  cli
// Output via PHP-FPM:             fpm-fcgi
```

### The web request lifecycle (FPM example)

Here's what happens when a browser hits a Laravel page served by Nginx + PHP-FPM:

```
1. Browser sends:   GET /dashboard  →  Nginx (port 80/443)
2. Nginx sees it's a .php request and forwards it over FastCGI to PHP-FPM.
3. PHP-FPM hands the request to an idle worker process.
4. The worker bootstraps PHP, runs your script (public/index.php for Laravel),
   compiling to opcodes (or pulling them from OPcache).
5. Your code queries the DB, builds an HTML response string.
6. PHP returns the output to FPM → Nginx → the browser.
7. The worker resets its per-request state and waits for the next request.
```

> **Key mental model: "shared nothing."** Each request starts with a clean slate. Global variables, static state — everything is torn down when the request ends. This is *why* PHP is famously easy to reason about and hard to crash with one bad request, and *why* you persist anything important in a database, cache, or session store rather than in memory between requests.

---

## 4. Installing PHP 8.4

> Always confirm with `php -v` after installing. If your terminal shows an old version, a different PHP is earlier in your `PATH` — fix the `PATH` rather than fighting the installer.

### macOS

The cleanest route is **Homebrew**:

```bash
# Install Homebrew first if you don't have it (see brew.sh), then:
brew install php          # installs the current stable (8.4 at time of writing)

# Pin a specific version if you need it:
brew install php@8.4
brew link --overwrite --force php@8.4

php -v
# Output (example):
# PHP 8.4.x (cli) (built: ...) (NTS)
# Copyright (c) The PHP Group
# Zend Engine v4.4.x, Copyright (c) Zend Technologies
#     with Zend OPcache v8.4.x, Copyright (c), by Zend Technologies
```

> **NTS vs ZTS.** `NTS` = *Non-Thread-Safe* (the normal choice for CLI and FPM). `ZTS` = *Zend Thread Safe*, only needed for threaded SAPIs. Pick NTS unless you have a specific reason not to.

Popular all-in-one alternatives for macOS: **Laravel Herd** (zero-config PHP + Nginx + dnsmasq, purpose-built for Laravel) and **DBngin**/**Valet**.

### Linux (Ubuntu/Debian)

The distro's default repos often lag behind. Use Ondřej Surý's well-maintained PPA:

```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt update

# CLI + the FPM SAPI + extensions Laravel needs
sudo apt install -y php8.4-cli php8.4-fpm \
  php8.4-mbstring php8.4-xml php8.4-bcmath \
  php8.4-curl php8.4-mysql php8.4-zip php8.4-intl php8.4-gd

php -v
```

On RHEL/Fedora/Alma, the **Remi repository** plays the same role as the PPA.

> **Extensions matter.** Laravel will refuse to boot without certain extensions (`mbstring`, `openssl`, `pdo`, `tokenizer`, `xml`, `ctype`, `json`, `bcmath`, `fileinfo`, `curl`). Check what's loaded with `php -m`.

### Windows

Three good options, easiest first:

1. **Laravel Herd for Windows** — bundles everything; recommended for this course.
2. **WSL2** (Windows Subsystem for Linux) — then follow the Linux instructions above inside Ubuntu. This is what most professionals do, because production is Linux.
3. **Manual** — download the Non-Thread-Safe x64 zip from `windows.php.net`, unzip to `C:\php`, add `C:\php` to your `PATH`, and copy `php.ini-development` to `php.ini`.

```powershell
# After installing (any method), verify in PowerShell:
php -v
```

---

## 5. `php.ini` — Configuration That Bites You

`php.ini` is PHP's main configuration file. The single most useful debugging command in your toolkit is finding *which* `php.ini` is actually in effect — there can be several (one for CLI, one for FPM):

```bash
php --ini
# Output (example):
# Configuration File (php.ini) Path: /opt/homebrew/etc/php/8.4
# Loaded Configuration File:         /opt/homebrew/etc/php/8.4/php.ini
# Scan for additional .ini files in: /opt/homebrew/etc/php/8.4/conf.d
# Additional .ini files parsed:      ...
```

> **Gotcha already:** the CLI uses a *different* `php.ini` than PHP-FPM. Changing the CLI ini will **not** affect your web app. Always edit the one whose path matches your SAPI, and **restart PHP-FPM** after editing its ini (`sudo systemctl restart php8.4-fpm`).

### The directives you must know

```ini
; ── Error handling (the #1 source of "why is my page blank?") ──
display_errors = Off          ; SHOW errors in browser? Off in production, On in dev
display_startup_errors = Off
error_reporting = E_ALL       ; WHICH errors to report. E_ALL = everything (use in dev)
log_errors = On               ; write errors to a log even when not displayed
error_log = /var/log/php/error.log

; ── Memory & time ──
memory_limit = 128M           ; max RAM one script may use; -1 = unlimited
max_execution_time = 30       ; seconds a script may run (CLI is exempt: 0/unlimited)

; ── File uploads ──
file_uploads = On
upload_max_filesize = 2M      ; max size of ONE uploaded file
post_max_size = 8M            ; max size of the ENTIRE POST body — MUST be >= upload_max_filesize
max_file_uploads = 20

; ── Other common ones ──
default_charset = "UTF-8"
date.timezone = "UTC"         ; set this, or date functions emit warnings
```

**`display_errors` vs `error_reporting` — don't confuse them:**

- `error_reporting` decides **which severity levels** PHP cares about (notices, warnings, errors…).
- `display_errors` decides **whether** the ones PHP cares about get printed to the output stream.

So in production you typically keep `error_reporting = E_ALL` *but* `display_errors = Off` and `log_errors = On` — you record everything, but show users nothing.

**The upload trap:** to allow a 10 MB upload you must raise **both** `upload_max_filesize` *and* `post_max_size` (which must be larger, since the body also carries other form fields). Forgetting `post_max_size` is a classic "my big file silently fails" bug.

### Reading and overriding settings from code

```php
<?php
echo ini_get('memory_limit') . PHP_EOL;   // read a directive  → e.g. 128M

ini_set('display_errors', '1');           // override at runtime (some directives only)
error_reporting(E_ALL);                    // override which errors are reported

// PHP_INT_MAX, PHP_VERSION, etc. are built-in constants, not ini settings:
echo PHP_VERSION . PHP_EOL;                 // 8.4.x
```

> Some directives (like `memory_limit`) *can* be changed with `ini_set()` at runtime; others (like `post_max_size`) are read so early that runtime changes have no effect — you must edit `php.ini`.

---

## 6. Running PHP: CLI, the Dev Server, and Inside HTML

### From the command line

```bash
# Run a script:
php hello.php

# Run a one-liner without a file:
php -r 'echo 1 + 2;'
# Output: 3

# Check syntax WITHOUT executing (lint) — great in CI/pre-commit hooks:
php -l hello.php
# Output: No syntax errors detected in hello.php

# Start an interactive shell (REPL):
php -a
```

### The built-in development server

PHP ships a tiny web server — perfect for quick local work, **never** for production (it's single-threaded by default and not hardened).

```bash
# Serve the current directory at http://localhost:8000
php -S localhost:8000

# Serve a specific document root:
php -S localhost:8000 -t public/

# Output (example):
# [Wed Jun 18 10:00:00 2026] PHP 8.4.x Development Server (http://localhost:8000) started
```

Now `http://localhost:8000/index.php` runs your script and returns its output. (Laravel's `php artisan serve` is a friendly wrapper around exactly this.)

### Embedding PHP in HTML

PHP's superpower is that it can be sprinkled directly into an HTML file. Everything **outside** `<?php ... ?>` is sent to the browser verbatim; everything **inside** is executed.

```php
<!DOCTYPE html>
<html>
<head><title>Greeting</title></head>
<body>
    <h1>Welcome</h1>
    <?php
        $name = 'Ada';
        $hour = (int) date('G');           // 0–23
    ?>
    <p>Hello, <?= htmlspecialchars($name) ?>!</p>
    <?php if ($hour < 12): ?>
        <p>Good morning.</p>
    <?php else: ?>
        <p>Good afternoon.</p>
    <?php endif; ?>
</body>
</html>
```

Two things to notice:

- `<?= ... ?>` is the **short echo tag** — shorthand for `<?php echo ... ?>`. It is **always available** (since PHP 5.4) regardless of the `short_open_tag` ini setting, so it's safe to use everywhere.
- Always run user-facing strings through `htmlspecialchars()` to prevent **XSS** (cross-site scripting). (Laravel's Blade `{{ }}` does this for you automatically — a major reason to use a templating engine.)

> **Pure-PHP files don't close the tag.** In files that contain *only* PHP (like a class or a config file), **omit the closing `?>`**. A stray newline after `?>` can be sent to the browser and break headers/redirects. This is a near-universal coding standard (PSR-12).

---

## 7. Core Syntax Essentials

### Open/close tags

```php
<?php   // standard opening tag — ALWAYS use this one
// ... code ...
?>      // closing tag (omit in pure-PHP files)
```

Avoid the *short open tag* `<?` (without `php`) — it depends on the `short_open_tag` ini setting and isn't portable. The short *echo* tag `<?=` is the one exception that's always safe.

### Statements & semicolons

Every PHP **statement** ends with a semicolon `;`. The closing `?>` implies a semicolon for the last statement before it, but relying on that is poor style.

```php
<?php
$x = 10;            // statement 1
$y = $x * 2;        // statement 2
echo $y;            // statement 3 → 20
```

### Comments

```php
<?php
// single-line comment (C++ style)
# single-line comment (shell style — valid but less common)

/*
   multi-line
   block comment
*/

/**
 * Docblock comment — read by IDEs, documentation tools, and PHP attributes parsers.
 * @param int $n
 * @return int
 */
function square(int $n): int { return $n * $n; }
```

### Case-sensitivity rules (memorize this — it's a favorite gotcha)

| Element | Case-sensitive? | Example |
|---|---|---|
| **Variables** | **Yes** | `$name` and `$Name` are *different* variables |
| Function names | No | `STRLEN()` == `strlen()` |
| Class/method/property names | No (for resolution) | but **always** write them in their declared case |
| Keywords (`if`, `echo`, `class`, `true`…) | No | `IF`, `If`, `if` all work |
| Constants (default) | **Yes** | `define('FOO', 1)` → `FOO`, not `foo` |
| `true` / `false` / `null` | No (they're keywords) | `TRUE` == `true` |

```php
<?php
$user = 'Ada';
echo $User ?? 'undefined';   // $User is a DIFFERENT, undefined variable → "undefined"

echo STRLEN('hi');           // 2  (function names are case-insensitive)
```

> **Best practice:** ignore the leniency. Write functions, classes, and keywords in their canonical case (`strlen`, `App\Models\User`, `if`). It reads better and avoids surprises when you move to case-sensitive contexts.

### Two constants you'll use constantly

```php
<?php
echo "Line one" . PHP_EOL . "Line two";
// PHP_EOL = the correct End-Of-Line for the current OS ("\n" on Unix, "\r\n" on Windows).
// Use it in CLI output so your scripts are portable.

echo PHP_VERSION;        // "8.4.x" — the running version as a string
echo PHP_MAJOR_VERSION;  // 8       — the major version as an int

// Comparing versions the right way:
if (version_compare(PHP_VERSION, '8.4.0', '>=')) {
    echo 'Property hooks available!' . PHP_EOL;
}
```

---

## 8. Your First Programs

### Hello, World (CLI)

```php
<?php
// hello.php
echo 'Hello, World!' . PHP_EOL;
```

```bash
php hello.php
# Output:
# Hello, World!
```

### A tiny dynamic page

Save as `time.php`, then run `php -S localhost:8000` and open `http://localhost:8000/time.php`:

```php
<?php
$now   = new DateTimeImmutable('now');
$hour  = (int) $now->format('G');

$greeting = match (true) {
    $hour < 12 => 'Good morning',
    $hour < 18 => 'Good afternoon',
    default    => 'Good evening',
};
?>
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8"><title>Clock</title></head>
<body>
    <h1><?= $greeting ?>!</h1>
    <p>The server time is <?= $now->format('H:i:s') ?>.</p>
    <p>Running on PHP <?= PHP_VERSION ?> via the <?= php_sapi_name() ?> SAPI.</p>
</body>
</html>
```

```
Output in the browser (rendered HTML), e.g.:
  Good afternoon!
  The server time is 14:32:07.
  Running on PHP 8.4.x via the cli-server SAPI.
```

> Note the SAPI name when using `php -S` is `cli-server` — proof that the *same* engine serves you through a different front door.

---

## 9. Choosing an Editor

- **VS Code** — free, ubiquitous. Install the **Intelephense** extension (excellent autocomplete, go-to-definition, type checking) plus **PHP Debug** (Xdebug integration). This is the recommended starting point.
- **PhpStorm** — the gold-standard PHP IDE (paid). Deep Laravel/Blade/Composer awareness, top-tier refactoring and debugging. If you're going pro on PHP, it pays for itself.

Whatever you pick, also set up:

- **Xdebug** — a step debugger and profiler. Vastly better than scattering `var_dump()` everywhere.
- **PHP CS Fixer** or **Laravel Pint** — auto-format code to the PSR-12 standard.

---

## 10. Composer in 60 Seconds

**Composer** is PHP's **dependency manager** — the equivalent of npm (Node) or pip (Python). It downloads the libraries your project needs and **autoloads** them so you never write `require` for vendor code.

```bash
# Install Composer globally (see getcomposer.org for the secure installer), then:
composer --version

# Start a new project's dependency manifest:
composer init

# Add a package:
composer require guzzlehttp/guzzle

# Install everything listed in composer.json:
composer install
```

Two files you'll see everywhere:

- **`composer.json`** — *your* declaration of which packages (and version ranges) you want.
- **`composer.lock`** — the *exact* resolved versions actually installed. **Commit this** so every teammate and server gets identical versions.

The autoloader follows the **PSR-4** standard (a namespace-to-directory mapping). Including it once unlocks every installed class:

```php
<?php
require __DIR__ . '/vendor/autoload.php';   // the one require you DO write
// Now any class from any installed package "just works" — no further require calls.
```

Creating a brand-new Laravel app is literally one Composer command:

```bash
composer create-project laravel/laravel my-app
cd my-app
php artisan serve   # → http://127.0.0.1:8000
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Blank white page / "nothing happens."**
   The script errored, but `display_errors` is `Off` (the default in many setups). **Fix:** during development set `display_errors = On` and `error_reporting = E_ALL` in the *correct* `php.ini`, or temporarily add `ini_set('display_errors', '1'); error_reporting(E_ALL);` at the top of the script. In production, leave display off and read `error_log` instead.

2. **Editing the wrong `php.ini`.**
   You change a setting and nothing happens because the **CLI** and **FPM** SAPIs load *different* ini files. **Fix:** run `php --ini` (for CLI) and check your FPM config path separately; after editing FPM's ini, **restart PHP-FPM**.

3. **`$Name` is not `$name`.**
   Variable names are case-sensitive; function and class names are not. A typo'd variable becomes a brand-new (undefined) variable instead of an error. **Fix:** enable `E_ALL` so "Undefined variable" warnings surface, and use an IDE that flags them.

4. **Large file uploads "silently fail."**
   You raised `upload_max_filesize` but forgot `post_max_size` (which must be *larger* and gates the whole request body). **Fix:** set `post_max_size` ≥ `upload_max_filesize`, and remember the web server (Nginx `client_max_body_size`) may also cap it.

5. **A stray closing `?>` breaks headers/redirects.**
   Whitespace or a newline after `?>` gets sent to the browser, so any later `header()` or `session_start()` throws "headers already sent." **Fix:** **omit the closing `?>`** in pure-PHP files (PSR-12 rule).

6. **Using the built-in dev server in production.**
   `php -S` is single-threaded and not hardened. **Fix:** use Nginx/Apache + PHP-FPM in production; reserve `php -S` (and `artisan serve`) for local dev.

7. **Expecting variables to persist between requests.**
   PHP is "shared-nothing" — every request starts fresh. A global you set in one request is gone in the next. **Fix:** persist state in a database, cache (Redis), or session.

---

## ✅ Best Practices

- **Develop on the same major.minor PHP version as production** (8.4 here). Subtle behavior differs across versions.
- **In dev:** `display_errors = On`, `error_reporting = E_ALL`. **In prod:** `display_errors = Off`, `log_errors = On`, errors to a file you actually monitor.
- **Always** escape output with `htmlspecialchars()` (or let Blade's `{{ }}` do it). Never echo raw user input into HTML.
- **Enable OPcache** in production; it dwarfs the JIT for typical web workloads. Consider the JIT only if you're CPU-bound and have measured it.
- **Use the canonical case** for everything, even where PHP is lenient.
- **Omit the closing `?>`** in pure-PHP files.
- **Commit `composer.lock`**; run `composer install` (not `update`) on servers for reproducible builds.
- **Set `date.timezone`** explicitly to avoid warnings and ambiguous timestamps.
- **Prefer `php -l` linting and Pint formatting** in a pre-commit hook to catch issues before they ship.
- **Use `DateTimeImmutable`** over `DateTime` to avoid accidental mutation bugs.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Is PHP compiled or interpreted?**
A: Both, in a sense. PHP **compiles each script to opcodes** (Zend bytecode) at runtime, then a virtual machine executes those opcodes. With **OPcache**, the compiled opcodes are cached in shared memory so they aren't recompiled on every request. The **JIT** can further compile hot opcodes to native machine code at runtime. So it's "compiled to bytecode, then interpreted, with optional JIT."

**Q2. (Under the hood) Walk me through what happens from `php script.php` to output.**
A: The CLI SAPI bootstraps the engine → the Zend Engine **lexes** the source into tokens → **parses** tokens into an **AST** → **compiles** the AST into **opcodes** → the **Zend VM executes** the opcodes (pulling from OPcache if available, optionally JIT-compiling hot paths) → output is written to STDOUT → per-request state is torn down.

**Q3. What changed between PHP 7 and 8?**
A: PHP 7 was the **performance** jump (engine rewrite, ~2x faster, scalar types, `??`, `<=>`). PHP 8 was the **language modernization**: JIT, named arguments, attributes, constructor promotion, `match`, nullsafe `?->`, union types, enums (8.1), readonly (8.1), and so on.

**Q4. Does the JIT make my Laravel app faster?**
A: Usually only marginally. Web apps are **I/O-bound** (database, network), and the JIT helps **CPU-bound** work. For typical apps, **OPcache** (and good queries/caching) matters far more. The JIT shines on computational workloads.

**Q5. What's a SAPI? Name a few.**
A: A **Server API** — the interface between PHP and its caller. **CLI** (terminal/cron/Artisan), **FPM** (FastCGI process pool, the modern web standard), **Apache module** (`mod_php`), and the **built-in dev server** (`cli-server`). The same code runs through any of them.

**Q6. Difference between `display_errors` and `error_reporting`?**
A: `error_reporting` sets **which severities** PHP tracks; `display_errors` sets **whether** they're printed to output. Production: report everything (`E_ALL`), display nothing, log instead.

**Q7. Why omit the closing `?>` tag?**
A: To prevent trailing whitespace/newlines after `?>` from being sent to the client, which causes "headers already sent" errors and corrupts redirects, downloads, and sessions. PSR-12 mandates omitting it in pure-PHP files.

**Q8. What does "shared-nothing architecture" mean for PHP?**
A: Each request runs in isolation and all in-memory state is destroyed when it ends. Nothing leaks between requests, which makes PHP robust and easy to reason about, but means you must persist anything cross-request in a DB, cache, or session.

**Q9. What is Composer and why `composer.lock`?**
A: Composer is PHP's dependency manager and PSR-4 autoloader. `composer.json` declares desired version ranges; `composer.lock` records the exact installed versions. Committing the lock file guarantees identical dependencies across machines and deploys.

**Q10. Which is case-sensitive in PHP — variables or functions?**
A: **Variables are case-sensitive** (`$a` ≠ `$A`); function and class names are **not** (but you should still always use their declared case).

---

## 📋 Quick Reference / Cheat Sheet

```bash
# ── CLI ──
php -v                         # version
php -m                         # loaded modules/extensions
php --ini                      # which php.ini files are loaded
php -i                         # full phpinfo() in the terminal
php script.php                 # run a file
php -r 'echo 1+2;'             # run a one-liner
php -l file.php                # lint (syntax check, no execute)
php -a                         # interactive REPL
php -S localhost:8000 -t public/   # dev server with doc root

# ── Composer ──
composer require vendor/pkg    # add a dependency
composer install               # install from composer.lock
composer update                # update + rewrite the lock file
composer dump-autoload         # rebuild the autoloader
```

```text
# ── Tags (shown as literal markup, not nested code) ──
<?php echo 'x'; ?>     standard open/echo/close
<?= $x ?>             short echo tag (always available, since 5.4)
<?php ... (no ?>)     pure-PHP files: omit the closing tag
```

```php
<?php
// ── Constants ──
echo PHP_EOL;           // OS-correct newline ("\n" on Unix, "\r\n" on Windows)
echo PHP_VERSION;       // "8.4.x"
echo PHP_INT_MAX;       // largest int on this build
echo PHP_OS_FAMILY;     // "Darwin" | "Linux" | "Windows" | "BSD" | "Solaris" | "Unknown"

// ── Runtime info ──
echo php_sapi_name();   // "cli" | "fpm-fcgi" | "cli-server" | "apache2handler"
echo phpversion();      // version string (same as PHP_VERSION)
var_dump(version_compare(PHP_VERSION, '8.4.0', '>='));  // bool(true) on 8.4+

// ── Errors at runtime ──
ini_set('display_errors', '1');
error_reporting(E_ALL);
```

```ini
; ── php.ini essentials ──
display_errors = On            ; Off in production
error_reporting = E_ALL
log_errors = On
memory_limit = 256M
max_execution_time = 30
upload_max_filesize = 10M
post_max_size = 12M            ; must be >= upload_max_filesize
date.timezone = "UTC"
opcache.enable = 1
```

---

## 🧪 Mini Exercises

1. **Environment audit.** Write a CLI script `env.php` that prints, each on its own line using `PHP_EOL`: the PHP version, the active SAPI name, the current `memory_limit`, and whether OPcache is enabled (`ini_get`/`extension_loaded`). Run it with `php env.php`.

2. **Two SAPIs, one script.** Write a script that prints `php_sapi_name()`. Run it (a) directly with `php` and (b) by serving its directory with `php -S localhost:8000` and visiting it in the browser. Note how the SAPI name differs, and write a one-sentence explanation of why.

3. **Dynamic greeting page.** Build an HTML page (served with `php -S`) that reads a `?name=...` query parameter (`$_GET['name']`), greets the visitor by name, and falls back to "stranger" if it's missing. Escape the name with `htmlspecialchars()`. Then test what happens if you pass `?name=<script>alert(1)</script>` — confirm your escaping neutralizes it.

4. **Break and fix it.** Create a pure-PHP file that ends with a closing `?>` followed by a blank line, then tries to call `header('Location: /')` *after* some output. Observe the "headers already sent" error, then fix it by removing the closing tag. Write down why this happened.

5. **Version gate.** Using `version_compare()` and `PHP_VERSION`, write a script that prints which PHP 8.x features are available on the running version (e.g., "enums: yes (8.1+)", "property hooks: yes (8.4+)"). Hard-code the minimum version for at least four features.
