# Namespaces & Autoloading

Namespaces and autoloading are the two mechanisms that let a modern PHP application made of thousands of classes — yours plus dozens of third‑party packages — coexist without name collisions and without a single `require` statement at the top of your files. Understanding them is the difference between "PHP scripts" and "PHP applications", and it is the foundation that Composer, PSR‑4, and the entire Laravel framework are built on.

> **What you'll learn**
> - **Why namespaces exist** and the class‑collision problem they solve.
> - How to **declare namespaces**, sub‑namespaces, and the difference between **fully‑qualified, qualified, and unqualified** names.
> - The **`use` keyword**: importing classes, aliasing with `use ... as`, grouped imports, and `use function` / `use const`.
> - The **global namespace**, the **leading backslash**, the `namespace` keyword, and the `__NAMESPACE__` magic constant.
> - How **autoloading** works: `spl_autoload_register`, the **PSR‑4** standard, and how **Composer** maps namespaces to directories.
> - The role of the `autoload` block in **`composer.json`**, `composer dump-autoload`, **classmaps**, and the generated `vendor/autoload.php`.
> - How to **debug a `Class "X" not found`** error systematically.

---

## 1. Why namespaces exist: the collision problem

Before namespaces (PHP < 5.3), every class, function, and constant lived in one **flat global space**. If you used two libraries that both defined a `Logger` class, PHP would throw a fatal error the moment both files were loaded:

```php
// library_a.php
class Logger { /* ... */ }

// library_b.php
class Logger { /* ... */ }   // Fatal error: Cannot declare class Logger, because the name is already in use
```

The community's only workaround was **prefixing**: `Acme_Mailer_Logger`, `Zend_Db_Adapter`, `Swift_Mailer`. This produced long, ugly, brittle names and was a convention, not a language feature.

A **namespace** is a named container for class, interface, trait, enum, function, and constant names. It lets `App\Logger` and `Monolog\Logger` both exist because their *fully‑qualified* names differ, even though the short name `Logger` is the same in both. Think of it like a filesystem: two files can both be named `index.php` as long as they live in different folders.

```php
namespace App;
class Logger {}      // fully-qualified name: \App\Logger

namespace Monolog;
class Logger {}      // fully-qualified name: \Monolog\Logger
```

Namespaces only affect **classes, interfaces, traits, enums, functions, and constants**. They do *not* affect variables (`$x` is always just `$x`).

---

## 2. Declaring a namespace

You declare a namespace with the `namespace` keyword. **It must be the very first statement in the file** (a `declare(strict_types=1);` may precede it, and comments are allowed, but no other code or even whitespace output).

```php
<?php

declare(strict_types=1);

namespace App\Services;   // must come before any class/function/output

class PaymentService
{
    public function charge(int $cents): bool
    {
        return true;
    }
}
```

The fully‑qualified name of that class is `\App\Services\PaymentService`.

### Sub‑namespaces

Namespaces are hierarchical. Separate levels with a backslash `\`. There is no separate "sub‑namespace" keyword — `App\Services\Payment` is simply a deeper namespace than `App\Services`.

```php
namespace App\Services\Payment;   // three levels deep

class StripeGateway {}
// fully-qualified: \App\Services\Payment\StripeGateway
```

### One namespace per file (the rule everyone follows)

PHP technically allows multiple namespaces in one file (with braces), but **autoloading requires one namespace and one class per file**, so in practice you always write one:

```php
// LEGAL but never do this in real projects:
namespace A { class Foo {} }
namespace B { class Bar {} }
```

PSR‑4 and Composer cannot autoload classes packed multiple‑per‑file, so stick to one class per file matching its namespace.

---

## 3. Name resolution: fully‑qualified, qualified, unqualified

This is the single most misunderstood part of namespaces. PHP classifies every name into three kinds, and resolves each differently. The **leading backslash** is what makes a name absolute.

| Kind | Example | How PHP resolves it |
|------|---------|---------------------|
| **Fully‑qualified** | `\App\Services\Mailer` | Absolute. Starts from the global root. No guessing. |
| **Qualified** | `Services\Mailer` | Relative to the current namespace (and `use` imports applied to the first segment). |
| **Unqualified** | `Mailer` | A single name; checks `use` imports first, then current namespace, then (for functions/consts) falls back to global. |

The key mental model: **a leading `\` means "start from the absolute root"**, exactly like `/` in a Unix path.

```php
namespace App\Http;

class Controller
{
    public function index(): void
    {
        // Unqualified: resolves to \App\Http\Response (current namespace)
        $r = new Response();

        // Qualified: relative -> \App\Http\Api\Response
        $a = new Api\Response();

        // Fully-qualified: absolute, ignores current namespace
        $g = new \App\Services\Mailer();
    }
}
```

### A critical gotcha for classes vs functions

For **classes**, an unqualified name like `new DateTime()` inside `namespace App;` resolves to `\App\DateTime` — and PHP does **NOT** fall back to the global `\DateTime`. If you want the global class you must write `new \DateTime()` (leading backslash) or `use DateTime;`.

```php
namespace App;

$d = new DateTime();   // ❌ Error: Class "App\DateTime" not found
$d = new \DateTime();  // ✅ global class
```

For **functions and constants**, PHP *does* fall back to global if no namespaced version exists (see §6).

---

## 4. The `use` keyword: importing names

Writing `\App\Services\Payment\StripeGateway` everywhere is exhausting. The `use` statement **imports** a fully‑qualified name into the current file so you can refer to it by its short name. `use` statements go after the `namespace` declaration and before any code.

```php
namespace App\Http\Controllers;

use App\Services\Payment\StripeGateway;   // import
use App\Models\User;

class CheckoutController
{
    public function store(): void
    {
        $gateway = new StripeGateway();   // short name works now
        $user    = User::first();
    }
}
```

Two important facts about `use`:

1. **`use` does not include or load any file.** It is purely a compile‑time alias — "when I write `StripeGateway`, I mean `\App\Services\Payment\StripeGateway`". The actual loading is the autoloader's job (§7).
2. **`use` names are always fully‑qualified**, with or without a leading backslash. `use App\Models\User;` and `use \App\Models\User;` are identical. The convention is to **omit** the leading backslash in `use` statements.

### Aliasing with `use ... as`

When two imported classes share a short name, or you just want a clearer local name, alias with `as`:

```php
namespace App\Reports;

use App\Pdf\Generator as PdfGenerator;
use App\Excel\Generator as ExcelGenerator;

class ReportBuilder
{
    public function build(): void
    {
        $pdf   = new PdfGenerator();
        $excel = new ExcelGenerator();
    }
}
```

### Grouped `use` (PHP 7+)

Import several names from the same parent namespace in one statement:

```php
use App\Models\{User, Post, Comment};

// Equivalent to:
use App\Models\User;
use App\Models\Post;
use App\Models\Comment;
```

You can alias individual entries inside a group, and PHP 7.2+ allows a **trailing comma** in the group:

```php
use App\Support\{
    Str,
    Arr as ArrayHelper,   // alias an individual entry
    Collection,           // trailing comma is fine (PHP 7.2+)
};
```

> **⚠️ You cannot mix classes, functions, and constants in one group.** A single grouped `use` imports **only classes** (and interfaces/traits/enums). Writing `use App\Support\{Str, function slugify, const MAX_LENGTH};` is a **parse error**. Functions and constants each need their own keyword on the group — and the keyword goes *before* the namespace, not inside the braces:
>
> ```php
> use App\Support\{Str, Arr as ArrayHelper};   // classes
> use function App\Support\{slugify, truncate}; // functions (own statement)
> use const App\Support\{MAX_LENGTH, MIN_LENGTH}; // constants (own statement)
> ```

### Importing namespaces, not just classes

You can `use` a namespace itself and then write qualified names from it:

```php
namespace App\Http;

use App\Services;   // import the namespace

$mailer = new Services\Mailer();      // -> \App\Services\Mailer
$payment = new Services\Payment\Stripe();  // -> \App\Services\Payment\Stripe
```

---

## 5. The global namespace and the leading backslash

Built‑in PHP classes (`Exception`, `DateTime`, `PDO`, `ArrayObject`), built‑in functions (`count`, `strlen`), and built‑in constants (`PHP_EOL`, `PHP_INT_MAX`) all live in the **global namespace** — the root, written as just `\`.

> **Note:** `true`, `false`, and `null` are *reserved keywords*, not ordinary constants. They are case‑insensitive language literals and are **never** affected by namespaces — you never need a leading backslash for them. Genuine global constants like `PHP_EOL` are real constants and *do* participate in the global fallback described in §6.

Inside a namespaced file you reach global classes with a leading backslash:

```php
namespace App\Exceptions;

class PaymentFailed extends \Exception {}   // global \Exception

try {
    // ...
} catch (\Throwable $e) {                   // global \Throwable
    throw new \RuntimeException('boom');     // global \RuntimeException
}
```

Alternatively import them once at the top, which most teams prefer for readability:

```php
namespace App\Exceptions;

use Exception;
use Throwable;
use RuntimeException;

class PaymentFailed extends Exception {}
```

Note: `use Exception;` has **no leading backslash** even though `Exception` is global — because, as noted, `use` targets are *always* absolute.

---

## 6. `use function` and `use const`

By default `use` imports **classes** (and interfaces/traits/enums). To import a namespaced **function** or **constant** you must say so explicitly:

```php
namespace App\Text;

function slugify(string $s): string { return strtolower(trim($s)); }
const VERSION = '1.0';
```

```php
namespace App\Http;

use function App\Text\slugify;   // import the function
use const App\Text\VERSION;      // import the constant

echo slugify('Hello World');  // hello world
echo VERSION;                 // 1.0
```

### The global fallback for functions and constants

Unlike classes, **unqualified function and constant calls fall back to the global namespace** if no matching name exists in the current namespace. This is why `count()`, `strlen()`, and `is_array()` "just work" inside namespaced code:

```php
namespace App;

$len = strlen('hi');   // ✅ no \App\strlen exists, so PHP uses global \strlen
```

This fallback has a small performance cost (PHP checks the local namespace first, then global) and can be ambiguous if your namespace ever defines a function of the same name. The clearest practice is to prefix built‑ins with `\` (`\strlen(...)`, `\count($x)`), which resolves to global immediately with no fallback lookup. You will often see `\count($x)` written deliberately in hot loops for this reason.

```php
namespace App;

$len = \strlen('hi');   // explicit global call — no namespace fallback, no ambiguity
```

You *can* also `use function` a name — this is most useful when importing a **namespaced** helper so you can call it by its short name. Importing an already‑global name like `use function strlen;` is legal but a near no‑op (it just aliases the global into the current file). Real‑world example with a namespaced helper:

```php
namespace App\Http;

use function App\Support\str_slug;   // import a namespaced function

echo str_slug('Hello World');         // calls \App\Support\str_slug
```

> **Modern reality check:** since PHP 7+, OPcache and the engine make the function/constant fallback cost negligible for the vast majority of applications. Reach for `\`‑prefixing or `use function` for *clarity and correctness* (avoiding accidental shadowing), not as a blanket performance optimization.

---

## 7. The `namespace` keyword and `__NAMESPACE__` constant

Beyond declaring a namespace, the `namespace` keyword can be used as an **operator** to write a name relative to the current namespace explicitly — useful when an imported alias might otherwise shadow what you mean:

```php
namespace App\Http;

use Vendor\Controller;   // imported "Controller"

$mine = new namespace\Controller();   // forces \App\Http\Controller, ignoring the import
```

The `__NAMESPACE__` magic constant is a **string** containing the current namespace name (empty string in global scope). It is handy for building class names dynamically:

```php
namespace App\Jobs;

echo __NAMESPACE__;   // "App\Jobs"

$class = __NAMESPACE__ . '\\SendEmail';   // "App\Jobs\SendEmail"
$job   = new $class();
```

Note the **doubled backslash** `'\\'` inside a string — in PHP a backslash is an escape character, so a literal backslash is written `\\`. (In single‑quoted strings only `\\` and `\'` are special, so `'\\'` produces one backslash; in double‑quoted strings the same `\\` rule applies.)

> **⚠️ Security:** Dynamically building a class name and passing it to `new $class()` (or `$class::method()`) is only safe when the name comes from **your own trusted code**, as above. **Never** feed user‑controlled input straight into a dynamic class name — an attacker could instantiate arbitrary classes (an "object injection" / arbitrary‑instantiation vector). If a class name must come from outside, validate it against an explicit allow‑list of permitted classes before instantiating.

```php
// ❌ DANGEROUS: attacker controls which class gets instantiated
$job = new $_GET['handler']();

// ✅ SAFE: constrain to a known allow-list
$allowed = [
    'email' => \App\Jobs\SendEmail::class,
    'sms'   => \App\Jobs\SendSms::class,
];
$class = $allowed[$_GET['handler']] ?? throw new \InvalidArgumentException('Unknown handler');
$job   = new $class();
```

---

## 8. Autoloading: from `require` to magic

Namespaces solve *naming*. Autoloading solves *loading*. Without it, you'd write a `require` for every class:

```php
require __DIR__ . '/src/Models/User.php';
require __DIR__ . '/src/Services/PaymentService.php';
require __DIR__ . '/vendor/monolog/.../Logger.php';
// ...hundreds more
```

**Autoloading** lets PHP load the file for a class *on demand*, the first moment that class is referenced. The hook is `spl_autoload_register`, which registers a callback PHP invokes whenever it encounters a class/interface/trait/enum name it doesn't yet know.

### A minimal hand‑written autoloader

```php
spl_autoload_register(function (string $class): void {
    // $class is the fully-qualified name WITHOUT a leading backslash,
    // e.g. "App\Services\PaymentService"

    $prefix  = 'App\\';
    $baseDir = __DIR__ . '/src/';

    // Only handle classes in our prefix
    if (!str_starts_with($class, $prefix)) {
        return;   // let another registered autoloader try
    }

    // Strip the prefix, convert namespace separators to directory separators
    $relative = substr($class, strlen($prefix));        // "Services\PaymentService"
    $file = $baseDir . str_replace('\\', '/', $relative) . '.php';
    // -> ".../src/Services/PaymentService.php"

    if (is_file($file)) {
        require $file;
    }
});

$p = new App\Services\PaymentService();   // triggers the autoloader -> requires the file
```

Key points:
- The callback receives the FQCN **without** a leading backslash.
- You can register **many** autoloaders; PHP tries each in order until one defines the class.
- If none define it, PHP throws `Error: Class "X" not found`.
- `spl_autoload_register` replaced the old single `__autoload()` function (removed in PHP 8.0).

This hand‑rolled loader is essentially what **PSR‑4** standardizes and what **Composer** generates for you.

---

## 9. PSR‑4: the autoloading standard

**PSR‑4** (PHP Standard Recommendation #4, "Autoloader") defines a deterministic mapping from a **namespace prefix** to a **base directory**. The rule:

> Take the fully‑qualified class name, remove the registered namespace prefix, replace the remaining `\` with directory separators, append `.php`, and look in the prefix's base directory.

So if the prefix `App\` maps to `src/`:

| Fully‑qualified class name | File path |
|----------------------------|-----------|
| `App\User` | `src/User.php` |
| `App\Services\PaymentService` | `src/Services/PaymentService.php` |
| `App\Http\Controllers\HomeController` | `src/Http/Controllers/HomeController.php` |

PSR‑4 rules to remember:
- The mapping is **case‑sensitive** for the trailing portion, because file paths on Linux are case‑sensitive. `App\User` must live in `User.php`, not `user.php`. (Class names themselves are case‑insensitive in PHP, but the *file lookup* uses the exact case — this bites people moving from macOS/Windows to Linux servers.)
- The namespace prefix maps to the base directory; **the prefix itself does not become a folder.** With `App\ => src/`, `App\Foo` is `src/Foo.php`, *not* `src/App/Foo.php`.

---

## 10. Composer: how it wires up autoloading

You almost never write `spl_autoload_register` yourself. **Composer** — PHP's dependency manager — generates a PSR‑4 autoloader from the `autoload` section of your `composer.json`.

### The `autoload` block

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "Database\\Seeders\\": "database/seeders/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

- The key is the **namespace prefix** (note the escaped trailing `\\` because JSON strings escape backslashes — `App\\` is the literal `App\`).
- The value is the **directory** relative to `composer.json`.
- `autoload-dev` is loaded only in dev (e.g. test classes) so it isn't shipped to production.

After editing `composer.json`'s autoload section you must regenerate the autoloader:

```bash
composer dump-autoload
# or the short alias:
composer dumpautoload
```

`dump-autoload` rescans the mappings and rewrites the generated files in `vendor/composer/`. **If you add a new namespace mapping (or a new classmap file) and forget to run it, your new classes won't be found** — a classic "it works on my machine" trap.

### Classmap autoloading

PSR‑4 needs files to follow the namespace→path convention. For legacy code, helper files, or files that don't follow PSR‑4, use **`classmap`**. Composer scans the listed files/directories, finds every class/interface/trait/enum declaration, and builds a literal `class => file` lookup table.

```json
{
    "autoload": {
        "psr-4": { "App\\": "src/" },
        "classmap": [
            "database/factories",
            "app/Legacy"
        ]
    }
}
```

Classmaps are **fastest at runtime** (a direct array lookup, no path computation) but require a `dump-autoload` every time you add a new class, since the map is static. PSR‑4 is more flexible: new files in a mapped directory are found without re‑dumping (in dev mode, where Composer falls back to filesystem checks).

### `files` autoloading

For files with no class to autoload — e.g. global helper functions — use `files`. These are `require`d eagerly on every request:

```json
{
    "autoload": {
        "files": [
            "src/helpers.php"
        ]
    }
}
```

This is exactly how Laravel loads its global helper functions (`now()`, `collect()`, `dd()`).

---

## 11. How `vendor/autoload.php` works

The single line at the top of every Composer project's entry point:

```php
require __DIR__ . '/vendor/autoload.php';
```

This file returns the Composer **ClassLoader** instance and registers it with `spl_autoload_register`. Under the hood Composer generates several files in `vendor/composer/`:

- `autoload_psr4.php` — array of `namespace prefix => [base dirs]`.
- `autoload_classmap.php` — array of `class => file`. In a default (non‑optimized) build this holds only classes from explicit `classmap` entries plus class lists packages ship; it is **not** a full scan of your PSR‑4 code until you run with `-o`/`--optimize` (which adds the scanned PSR‑4 classes here).
- `autoload_namespaces.php` — legacy **PSR‑0** prefixes (present but usually empty in modern projects).
- `autoload_files.php` — list of files to eager‑load.
- `autoload_static.php` — a single optimized static class holding all of the above for fast loading.
- `ClassLoader.php` — the engine that does the lookup.

When you reference `App\Models\User`, PHP calls Composer's `ClassLoader::loadClass()`, which checks the classmap first (O(1)), then walks the PSR‑4 prefixes, computes the path, and `require`s the file.

### Optimizing the autoloader for production

In production you want everything in a classmap for speed (no filesystem stat calls per class):

```bash
composer install --optimize-autoloader --no-dev
# or after the fact:
composer dump-autoload --optimize        # -o, scans PSR-4 dirs into a classmap
composer dump-autoload --classmap-authoritative  # -a, classmap is the ONLY source
```

- `--optimize` / `-o`: converts PSR‑4 into a classmap but still falls back to filesystem for classes not found.
- `--classmap-authoritative` / `-a`: if it's not in the classmap, it doesn't exist (no filesystem fallback) — fastest, but you MUST re‑dump after adding classes.

---

## 12. Namespaces & autoloading in Laravel

Laravel ships with this PSR‑4 mapping in its `composer.json`:

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

So `App\` maps to the `app/` directory. A controller at `app/Http/Controllers/UserController.php` must declare:

```php
namespace App\Http\Controllers;

class UserController extends Controller {}
```

Artisan generators get this right automatically. `php artisan make:controller Admin/UserController` creates `app/Http/Controllers/Admin/UserController.php` with `namespace App\Http\Controllers\Admin;`.

### The `::class` constant

Always reference classes with `::class` rather than string literals. It produces the fully‑qualified name, is checked by the IDE/static analyzers, and survives refactors:

```php
use App\Jobs\SendWelcomeEmail;

dispatch(new SendWelcomeEmail($user));

// In config or route definitions:
Route::get('/users', [UserController::class, 'index']);

// Get the FQCN as a string:
echo SendWelcomeEmail::class;   // "App\Jobs\SendWelcomeEmail"
echo User::class;               // "App\Models\User"
```

> **Laravel 11/12 note:** In Laravel 11 the default app skeleton was slimmed down (no `app/Http/Kernel.php`, no `app/Console/Kernel.php`, fewer middleware files) — but the namespace/autoload mapping (`App\` → `app/`) is unchanged from Laravel 10. The route‑model‑binding and provider registration moved to `bootstrap/app.php` and `bootstrap/providers.php`, but autoloading itself behaves identically across Laravel 10/11/12.

---

## ⚠️ Common Mistakes & Gotchas

**1. Forgetting `composer dump-autoload` after adding a new namespace or classmap entry.**
You create `app/Support/Money.php`, reference `App\Support\Money`, and get `Class not found`. PSR‑4 dirs that already exist usually work without re‑dumping, but **new top‑level namespace mappings, new classmap entries, and new `files` entries require a re‑dump.**
**Fix:** `composer dump-autoload`. When in doubt, run it — it's cheap.

**2. File path / case mismatch with the namespace.**
`namespace App\Services;` in a file named `app/services/Mailer.php` (lowercase `services`) works on macOS/Windows (case‑insensitive filesystems) but **fails on Linux production servers**. Same with `Mailer` vs `mailer.php`.
**Fix:** Make the directory and file names match the namespace and class name **exactly**, including case. PSR‑4 path lookup is case‑sensitive.

**3. Expecting global classes to resolve without a backslash.**
Inside `namespace App;`, writing `throw new Exception(...)` looks for `\App\Exception` and fails.
**Fix:** Either `use Exception;` at the top, or write `new \Exception(...)` with a leading backslash.

**4. Putting code or output before the `namespace` declaration.**
A stray blank line, BOM, `echo`, or `<?php ... ?>` whitespace before `namespace` causes `Fatal error: Namespace declaration statement has to be the very first statement` (or `headers already sent`).
**Fix:** `namespace` must be the first statement (only `declare(strict_types=1);` and comments may precede it). Remove trailing `?>` from pure‑PHP files.

**5. Confusing the namespace prefix with a directory.**
With `"App\\": "app/"`, people put a class in `app/App/Foo.php` expecting `App\Foo`. The prefix is *stripped*, so `App\Foo` maps to `app/Foo.php`.
**Fix:** Remember PSR‑4 strips the prefix — the prefix maps to the base dir, it is not duplicated as a folder.

**6. `use` doesn't load anything, so a typo in the FQCN fails silently until use.**
`use App\Modls\User;` (typo) compiles fine; you only get `Class not found` when you actually instantiate it.
**Fix:** Rely on `::class` and IDE/static analysis (PHPStan/Psalm) to catch bad imports early.

---

## ✅ Best Practices

- **One class per file**, file named exactly after the class, directory structure mirroring the namespace (PSR‑4).
- **Always `use` imports** at the top instead of inline FQCNs; it makes dependencies visible and refactor‑friendly.
- **Omit leading backslashes in `use` statements** (`use App\Models\User;`), and **import global classes** rather than sprinkling `\Exception` everywhere — pick one style per team and be consistent.
- Use **`::class`** everywhere you need a class name as a string — never hand‑type `"App\\Models\\User"`.
- For namespaced built‑in **functions/constants in hot paths**, either `use function`/`use const` them or prefix with `\` to skip the global fallback lookup.
- Keep your `composer.json` `autoload` block clean: PSR‑4 for your app code, `classmap` for legacy/factory code, `files` for helper functions; put test‑only namespaces in `autoload-dev`.
- In **production**, deploy with `composer install --no-dev --optimize-autoloader` (or `--classmap-authoritative` for read‑only deploys) for the fastest class loading.
- Run **`composer dump-autoload`** as part of your build/CI step so missing mappings never reach production.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What problem do namespaces solve?**
A: Name collisions in PHP's flat global symbol table. Two libraries can each define `Logger` because their fully‑qualified names (`App\Logger`, `Monolog\Logger`) differ. They replaced the old underscore‑prefix convention (`Zend_Db_Adapter`).

**Q2. What's the difference between fully‑qualified, qualified, and unqualified names?**
A: A *fully‑qualified* name has a leading `\` (`\App\Foo`) and is absolute. A *qualified* name has at least one `\` but no leading one (`Sub\Foo`) and is resolved relative to the current namespace (with `use` applied to the first segment). An *unqualified* name (`Foo`) is a bare identifier resolved against `use` imports, then the current namespace.

**Q3. Inside a namespace, does `new Exception()` use the global `Exception`?**
A: No. For *classes*, an unqualified name resolves only within the current namespace — it would look for `CurrentNs\Exception` and fail. You need `\Exception` or `use Exception;`. (Functions and constants *do* fall back to global.)

**Q4. Does `use` include/require the file?**
A: No. `use` is a compile‑time alias only. The actual file loading is done by the autoloader (`spl_autoload_register` / Composer) when the class is first referenced at runtime.

**Q5. Explain PSR‑4. How does a class name become a file path?**
A: PSR‑4 maps a namespace prefix to a base directory. Remove the prefix from the FQCN, replace remaining `\` with `/`, append `.php`, look in the base dir. E.g. with `App\ => src/`, `App\Http\Kernel` → `src/Http/Kernel.php`. The lookup is case‑sensitive.

**Q6. (Under the hood) What actually happens when you reference an undefined class?**
A: PHP invokes every callback registered via `spl_autoload_register`, in order, passing the FQCN (no leading backslash). Composer's `ClassLoader::loadClass` checks its classmap first (O(1)), then iterates PSR‑4 prefixes, computes the path, and `require`s it. If no autoloader defines the class, PHP throws `Error: Class "X" not found`.

**Q7. Difference between PSR‑4 and classmap autoloading?**
A: PSR‑4 computes the file path from the class name at runtime and tolerates new files without re‑dumping (in dev). Classmap is a precomputed `class => file` array — fastest at runtime but must be regenerated (`dump-autoload`) whenever a class is added. Production typically converts PSR‑4 into a classmap with `-o`/`-a`.

**Q8. What does `composer dump-autoload` do, and when must you run it?**
A: It rescans the `autoload` config and regenerates the files in `vendor/composer/`. You must run it after adding a new namespace mapping, a new classmap path, a new `files` entry, or when using authoritative/optimized classmaps and adding classes.

**Q9. What is `vendor/autoload.php`?**
A: The Composer‑generated bootstrap that instantiates the `ClassLoader`, registers it with `spl_autoload_register`, and eager‑loads `files` autoloads. Including it once wires up autoloading for your whole app and all dependencies.

**Q10. Why is `User::class` preferred over the string `"App\Models\User"`?**
A: `::class` is resolved at compile time using the current `use` imports/namespace, so it's refactor‑safe, checked by tooling, and avoids backslash‑escaping bugs in strings.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Declaring ---
namespace App\Services;            // first statement; one per file

// --- Importing (compile-time aliases; load nothing) ---
use App\Models\User;               // class (leading \ omitted by convention)
use App\Pdf\Generator as PdfGen;   // alias
use App\Models\{Post, Comment};    // grouped
use function App\Text\slugify;     // function
use const App\Text\VERSION;        // constant

// --- Name kinds ---
new \DateTime();        // fully-qualified (absolute)
new Sub\Thing();        // qualified (relative to current ns)
new Thing();            // unqualified (use-imports, then current ns)

// --- Global from inside a namespace ---
new \Exception();       // leading backslash reaches global
\strlen($s);            // explicit global function (skips fallback)

// --- Magic ---
echo __NAMESPACE__;     // current namespace as string
echo User::class;       // "App\Models\User"
new namespace\Foo();    // force current-namespace resolution
```

```json
// composer.json
{
  "autoload": {
    "psr-4": { "App\\": "src/" },
    "classmap": ["database/factories"],
    "files": ["src/helpers.php"]
  },
  "autoload-dev": { "psr-4": { "Tests\\": "tests/" } }
}
```

```bash
composer dump-autoload                       # regenerate autoloader
composer dump-autoload -o                    # optimize (PSR-4 -> classmap)
composer dump-autoload -a                    # classmap authoritative
composer install --no-dev --optimize-autoloader   # production
```

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Class "X" not found` | No autoload mapping / wrong path | Check namespace↔path, run `dump-autoload` |
| Works locally, fails on Linux | Case mismatch in dir/file | Match case exactly |
| `namespace must be first statement` | Output/code before `namespace` | Move `namespace` up; remove `?>`/BOM |
| New helper function not callable | `files` entry not dumped | `composer dump-autoload` |

### Debugging a `Class not found` — checklist

1. Is the **namespace in the file** exactly correct, including case?
2. Does the **file path** match PSR‑4 (prefix stripped, `\`→`/`, `.php`)?
3. Is the **prefix→dir mapping** present in `composer.json`?
4. Did you run **`composer dump-autoload`**?
5. Is `vendor/autoload.php` actually **required** in your entry point?
6. Check `vendor/composer/autoload_psr4.php` / `autoload_classmap.php` — is your class/prefix listed?
7. Typo in the `use` statement or the FQCN? Confirm with `class_exists('App\\Foo')` or `composer dump-autoload --strict-psr` (dedicated flag that reports — and fails on — classes whose file path doesn't match their namespace; plain `-o`/`--optimize` also prints these warnings but always exits 0).

---

## 🧪 Mini Exercises

1. **Build a hand‑rolled PSR‑4 autoloader.** Create a `src/` folder with `App\Math\Calculator` and `App\Math\Geometry\Circle`. Write a `spl_autoload_register` callback (no Composer) that loads them, and prove both classes instantiate from a single `index.php`.

2. **Composer mapping.** Initialize a project with `composer init`, add the PSR‑4 mapping `"Acme\\": "lib/"`, create `Acme\Greeter` in the right file, run `composer dump-autoload`, and call it from a script that requires `vendor/autoload.php`.

3. **Name resolution drill.** In a file with `namespace App;`, write code that (a) uses the global `DateTime`, (b) calls the global `count()`, (c) instantiates a local `App\Helper`, and (d) references `App\Sub\Thing` via a qualified name — each correctly, without any `use` statements. Then rewrite it using `use` imports.

4. **Classmap vs PSR‑4.** Add a legacy file `legacy/OldReport.php` defining a class `OldReport` (no namespace) and register it via `classmap`. Verify it loads, then add a second class to the same file and observe what happens before vs after `composer dump-autoload`.

5. **Aliasing collision.** Import two different `Generator` classes from two namespaces into the same file using `as` aliases, instantiate both, and confirm there's no collision. Then add a `use function` import and call a namespaced function by its short name.
