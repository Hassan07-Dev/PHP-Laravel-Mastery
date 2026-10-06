# Error & Exception Handling in PHP & Laravel

Handling failure well is one of the clearest signals of a senior engineer. Anyone can write the happy path; the question is what your code does when the database is down, the input is garbage, the file is missing, or a third-party API times out. This module takes you from "what even is an `Error` vs an `Exception`" to designing your own exception hierarchies and wiring up production-grade error handling in PHP 8.4 and Laravel 12.

> **What you'll learn**
> - The difference between **errors** and **exceptions** in modern PHP, and how `Throwable` unifies them.
> - PHP's **error levels** (notice, warning, deprecated, fatal) and the `error_reporting` / `display_errors` / `log_errors` settings — and how dev and prod should differ.
> - Global hooks: `set_error_handler`, `set_exception_handler`, and `register_shutdown_function`.
> - Everything about `try` / `catch` / `finally`: multi-catch with `|`, non-capturing catch, `throw` as an expression, and the subtle `finally` + `return` interplay.
> - The **SPL exception hierarchy** (`LogicException` vs `RuntimeException` and their subclasses) and how to choose the right one.
> - Designing **custom exception classes** and using **exception chaining** (`previous`).
> - Converting plain PHP errors into exceptions with `ErrorException`.
> - How **Laravel 12** centralizes exception handling via `bootstrap/app.php` and renders errors for web vs API.

---

## 1. Why exception handling exists (the WHY)

Before objects, PHP code dealt with failure by returning special values: `false`, `null`, `-1`, or an error code. The caller was *trusted* to check. They rarely did. A function deep in your call stack would return `false`, the immediate caller ignored it, and the program limped on with corrupt state until something blew up far away from the real cause.

**Exceptions invert the default.** When something goes wrong, the normal flow *stops* and control jumps up the call stack until someone explicitly says "I know how to handle this" (a `catch` block). If nobody does, the program halts loudly with a stack trace pointing at the real source. This is the principle of **fail fast**: surface problems immediately and visibly rather than silently continuing in a broken state.

> **Jargon:** A *stack trace* (or *backtrace*) is the chain of function calls that were active when the exception was thrown — it tells you *how* execution got to the failure point.

---

## 2. Errors vs Exceptions: the `Throwable` hierarchy

This is one of the most common interview questions, so get it crisp.

In PHP 5, "errors" (like calling an undefined function) and "exceptions" were completely separate worlds. You could `try/catch` an exception, but a fatal error just killed the script — uncatchable. **PHP 7 unified them** by introducing the `Throwable` interface at the top of the tree.

```
Throwable (interface)
├── Error            ← engine-level problems (formerly "fatal errors")
│   ├── TypeError
│   ├── ValueError            (PHP 8.0+)
│   ├── ArgumentCountError
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── UnhandledMatchError   (PHP 8.0+)
│   ├── FiberError            (PHP 8.1+)
│   └── CompileError
│       └── ParseError
└── Exception        ← application-level problems you create/handle
    ├── ErrorException
    ├── JsonException         (PHP 7.3+; thrown with JSON_THROW_ON_ERROR)
    ├── LogicException        (+ subclasses, see §7)
    └── RuntimeException      (+ subclasses, see §7)
```

> **Both `Error` and `Exception` also implement `Stringable`** (so `(string) $e` works) and `Throwable` itself extends `Stringable`. `Throwable` is a sealed interface — see the next rule.

Key rules:

- **`Error`** represents problems with the program *itself* — type mismatches, calling a method on `null`, dividing by zero. These usually mean a *bug*, not a recoverable runtime condition.
- **`Exception`** is the base class *you* extend for application-level conditions you anticipate and want to handle.
- **Both implement `Throwable`.** You **cannot** `implements Throwable` on your own class directly — the engine forbids it; you must extend `Exception` or `Error`.
- To catch *anything*, catch `Throwable`. To catch only application exceptions, catch `Exception` (this will **not** catch a `TypeError`).

```php
<?php
declare(strict_types=1);

function divide(int $a, int $b): int {
    return intdiv($a, $b); // throws DivisionByZeroError when $b === 0
}

try {
    echo divide(10, 0);
} catch (Exception $e) {
    echo "Caught Exception\n";
} catch (Throwable $e) {
    echo "Caught Throwable: " . $e::class . "\n";
}
// Output:
// Caught Throwable: DivisionByZeroError
```

Notice the `Exception` catch was skipped — `DivisionByZeroError` is an `Error`, not an `Exception`. Only the `Throwable` catch matched.

```php
<?php
function greet(string $name): string {
    return "Hello, {$name}";
}

try {
    // For a USER-DEFINED function, passing literal null to a non-nullable
    // scalar parameter is always a TypeError — even without declare(strict_types=1).
    // (Internal functions like strlen() only *deprecated* this in 8.1 in coercive mode.)
    greet(null);
} catch (TypeError $e) {
    echo "TypeError: " . $e->getMessage() . "\n";
}
// Output (message abbreviated):
// TypeError: greet(): Argument #1 ($name) must be of type string, null given
```

---

## 3. PHP error levels & the engine's "errors"

Separate from the `Error` *class* hierarchy, PHP has a legacy system of **error levels** — bitmask constants the engine raises for non-exception problems. You'll still meet them constantly.

| Level constant | Meaning | Fatal? |
|---|---|---|
| `E_ERROR` | Fatal runtime error, script halts | Yes |
| `E_WARNING` | Non-fatal runtime warning (e.g. `fopen` on missing file) | No |
| `E_NOTICE` | Minor issue (legacy; many promoted to warnings in PHP 8) | No |
| `E_DEPRECATED` | Feature still works but will be removed | No |
| `E_PARSE` | Compile-time syntax error | Yes |
| `E_USER_ERROR` / `E_USER_WARNING` / `E_USER_NOTICE` / `E_USER_DEPRECATED` | Triggered by your code via `trigger_error()` | Varies |
| `E_ALL` | All of the above | — |

> **PHP 8 shift:** Many things that were `E_NOTICE` or `E_WARNING` in PHP 7 became `Error` *exceptions* in PHP 8. Examples: accessing an undefined array key is now an `E_WARNING` (it was an `E_NOTICE` in 7.x), reading an undefined variable was also promoted from notice to warning, but calling a method on `null` now throws an `Error`, and a non-matching `match` throws `UnhandledMatchError`. Note too that PHP 8.0's *default* `error_reporting` is `E_ALL` (it previously excluded `E_NOTICE`/`E_DEPRECATED`).

> **PHP 8.4 note:** The `E_STRICT` level was removed (its constant is deprecated and no longer raised), so don't reference it. Separately, an *implicitly* nullable parameter such as `function f(Type $x = null)` now emits an `E_DEPRECATED`; always write it explicitly as `?Type $x = null` (every example in this module already does).

### Triggering your own errors

```php
<?php
trigger_error("Config value missing, using default", E_USER_WARNING);
// Emits a warning; script continues.
```

### The three INI settings you must know

These control *whether* errors are reported, *where* they go, and at *what level*.

```ini
; php.ini

; WHICH levels get reported at all (a bitmask)
error_reporting = E_ALL

; SHOW errors in the output/browser? On in dev, NEVER On in prod.
; (This example shows the SAFE production value.)
display_errors = Off

; WRITE errors to the log file? (keep On in BOTH dev and prod)
log_errors = On

; WHERE the log goes (empty = SAPI default, e.g. Apache/FPM error log)
error_log = /var/log/php/error.log
```

You can also set these at runtime:

```php
<?php
error_reporting(E_ALL & ~E_DEPRECATED); // everything except deprecations
ini_set('display_errors', '0');
ini_set('log_errors', '1');
```

> **Note:** `error_reporting` accepts a *bitmask*. `E_ALL & ~E_DEPRECATED` means "all levels, with the deprecated bit removed." `~` flips bits; `&` ANDs them together.

### Dev vs Prod: the golden rule

| Setting | Development | Production |
|---|---|---|
| `display_errors` | `On` (see problems fast) | **`Off`** (never leak stack traces to users) |
| `log_errors` | `On` | `On` |
| `error_reporting` | `E_ALL` | `E_ALL` (log everything; just don't display) |

Showing a stack trace to a user in production is both an awful UX and a **security risk** — traces leak file paths, library versions, SQL, and sometimes credentials. In Laravel this maps to the `APP_DEBUG` env var:

```env
# .env — development
APP_DEBUG=true

# .env — production (CRITICAL)
APP_DEBUG=false
```

With `APP_DEBUG=false`, Laravel shows a generic error page; with `true`, it shows the rich Ignition/Whoops debug screen.

---

## 4. Global handlers: `set_error_handler`, `set_exception_handler`, `register_shutdown_function`

These three functions let you intercept failures *globally* — they're how frameworks like Laravel implement their error pages.

### `set_exception_handler` — last resort for *uncaught* exceptions

If a `Throwable` propagates all the way up without being caught, this handler fires (just before the script dies).

```php
<?php
set_exception_handler(function (Throwable $e): void {
    error_log("Uncaught: " . $e->getMessage());
    http_response_code(500);
    echo "Something went wrong. Please try again later.\n";
});

throw new RuntimeException("Boom");
// Output:
// Something went wrong. Please try again later.
// (and "Uncaught: Boom" written to the log)
```

> After the exception handler runs, the script **terminates**. You cannot resume execution. The handler's job is graceful shutdown / logging, not recovery.

### `set_error_handler` — intercept engine errors

This catches the *level-based* errors (warnings, notices, deprecations) — **not** exceptions, and **not** fatal `E_ERROR`/`E_PARSE` (those can't be caught here).

```php
<?php
set_error_handler(function (int $severity, string $message, string $file, int $line): bool {
    // Respect the error_reporting() setting (e.g. @ silencing)
    if (!(error_reporting() & $severity)) {
        return false; // let PHP's default handler run
    }
    // Convert the error into an exception (see §9)
    throw new ErrorException($message, 0, $severity, $file, $line);
});

echo $undefined; // raises E_WARNING -> becomes ErrorException
```

> **Return value matters:** returning `false` tells PHP to *also* run its built-in handler; returning `true` (or nothing) suppresses it.

### `register_shutdown_function` — catch the *fatal* ones

Fatal errors (`E_ERROR`, out-of-memory, `E_PARSE`) bypass `set_error_handler`. The only way to react is a shutdown function combined with `error_get_last()`.

```php
<?php
register_shutdown_function(function (): void {
    $err = error_get_last();
    if ($err !== null && in_array($err['type'], [E_ERROR, E_PARSE, E_CORE_ERROR, E_COMPILE_ERROR], true)) {
        error_log("FATAL: {$err['message']} in {$err['file']}:{$err['line']}");
        // last chance to flush, close connections, send 500, etc.
    }
});
```

This trio — error handler, exception handler, shutdown function — is exactly what Laravel registers internally to turn every kind of failure into a consistent response.

---

## 5. `try` / `catch` / `finally`

The core construct.

```php
<?php
try {
    $data = riskyOperation();          // code that may throw
} catch (RuntimeException $e) {
    // runs only if a RuntimeException (or subclass) was thrown
    logIt($e);
} finally {
    // ALWAYS runs — exception or not, return or not
    cleanup();
}
```

- `try` wraps code that might throw.
- `catch` handles a specific `Throwable` type (and its subclasses).
- `finally` runs **no matter what** — even if the `try` or `catch` returns, or rethrows. Perfect for releasing resources (file handles, locks, DB transactions).

### Multiple catch blocks (first match wins, order from specific to general)

```php
<?php
try {
    process();
} catch (InvalidArgumentException $e) {
    // most specific
} catch (LogicException $e) {
    // InvalidArgumentException is a LogicException, so order matters:
    // putting this first would shadow the block above
} catch (Throwable $e) {
    // catch-all safety net
}
```

### Catching multiple types with the pipe `|` (PHP 7.1+)

When two unrelated exception types need the *same* handling, list them with `|`:

```php
<?php
try {
    importFile($path);
} catch (JsonException | UnexpectedValueException $e) {
    echo "Bad data: " . $e->getMessage();
}
```

### Non-capturing catch (PHP 8.0+)

If you don't need the exception object, you can omit the variable:

```php
<?php
try {
    config('cache')->flush();
} catch (Throwable) {           // no $e — we just want to ignore failure here
    // (use sparingly — see Common Mistakes)
}
```

### `throw` as an expression (PHP 8.0+)

`throw` is now an *expression*, so it can appear where only expressions are allowed — inside the `?:`, `??`, arrow functions, and `match`:

```php
<?php
$user = findUser($id) ?? throw new UserNotFoundException($id);

$role = $user->role ?: throw new InvalidArgumentException('Role required');

$status = match ($code) {
    200 => 'ok',
    404 => 'missing',
    default => throw new UnexpectedValueException("Unknown code: {$code}"),
};
```

This is a hugely idiomatic modern pattern — interviewers love seeing `?? throw`.

---

## 6. `finally` semantics & the `return` interplay

This is a favorite "gotcha" question. Understand the exact ordering.

**Rule 1 — `finally` runs even when `try`/`catch` returns.** The return value is computed and *staged*, then `finally` runs, then the function returns the staged value:

```php
<?php
function demo(): string {
    try {
        return "from try";
    } finally {
        echo "finally ran\n";
    }
}
echo demo();
// Output:
// finally ran
// from try
```

**Rule 2 — a `return` inside `finally` OVERRIDES the try/catch return.** This is a trap; avoid returning from `finally`:

```php
<?php
function trap(): string {
    try {
        return "from try";
    } finally {
        return "from finally"; // overrides!
    }
}
echo trap();
// Output:
// from finally
```

**Rule 3 — `finally` can swallow exceptions if it returns or throws.** If the `try` throws and `finally` `return`s, the exception is *silently discarded*:

```php
<?php
function swallow(): string {
    try {
        throw new RuntimeException("lost!");
    } finally {
        return "no exception escaped"; // the RuntimeException vanishes — BAD
    }
}
echo swallow();
// Output:
// no exception escaped   (the exception is gone — a debugging nightmare)
```

**Takeaway:** Use `finally` for cleanup only. **Never** `return` or `throw` from a `finally` block unless you fully intend to override what `try` produced.

---

## 7. The SPL exception hierarchy: `LogicException` vs `RuntimeException`

The SPL (Standard PHP Library) ships a set of base exception classes. Choosing the *right* one communicates intent. The top-level split is the most important interview point.

### `LogicException` — "this is a programming bug"

A `LogicException` means the error *should have been prevented in the code* — a precondition was violated, an impossible state was reached. These typically indicate a bug to be fixed, not a runtime condition to recover from.

| Subclass | Use when |
|---|---|
| `InvalidArgumentException` | An argument is the wrong type/value (most common) |
| `DomainException` | A value is outside the valid *domain* (e.g. an unknown enum-like string) |
| `LengthException` | A length is invalid (too short/long) |
| `OutOfRangeException` | An *illegal* index used at *compile-logic* time (key that shouldn't exist) |
| `BadFunctionCallException` | An undefined callback was called |
| `BadMethodCallException` | An undefined/uncallable method was called (common in `__call`) |

### `RuntimeException` — "this can only be detected at runtime"

A `RuntimeException` means something went wrong that you couldn't have prevented by reading the code — the network failed, a file disappeared, data didn't match expectations. These are often *recoverable*.

| Subclass | Use when |
|---|---|
| `OutOfBoundsException` | An invalid *index* requested at runtime (e.g. user-supplied key) |
| `RangeException` | A runtime value is out of range (the runtime sibling of `DomainException`) |
| `OverflowException` | Adding to a full container |
| `UnderflowException` | Removing from an empty container |
| `UnexpectedValueException` | A value didn't match expectations (e.g. a function returned junk) |

> **The classic confusion:** `OutOfRangeException` (Logic — a key you should *know* is wrong, a bug) vs `OutOfBoundsException` (Runtime — an index that turned out invalid at runtime). Mnemonic: **Range = at coding time, Bounds = at run time.**

### Choosing in practice

```php
<?php
final class TemperatureConverter
{
    public function celsiusToFahrenheit(float $celsius): float
    {
        // Below absolute zero is physically impossible -> bad argument = Logic
        if ($celsius < -273.15) {
            throw new InvalidArgumentException(
                "Temperature {$celsius}°C is below absolute zero."
            );
        }
        return $celsius * 9 / 5 + 32;
    }
}
```

```php
<?php
function readConfig(string $path): array
{
    // The file missing is a runtime condition, not a code bug -> Runtime
    if (!is_readable($path)) {
        throw new RuntimeException("Cannot read config at: {$path}");
    }
    $json = file_get_contents($path);
    return json_decode($json, true, flags: JSON_THROW_ON_ERROR); // throws JsonException
}
```

---

## 8. Custom exception classes & the `Throwable` API

Custom exceptions make `catch` blocks meaningful and let you attach domain data. Extend the SPL class that best matches the *category* of failure.

```php
<?php
declare(strict_types=1);

final class InsufficientFundsException extends RuntimeException
{
    public function __construct(
        public readonly int $balanceCents,
        public readonly int $requestedCents,
        ?Throwable $previous = null,
    ) {
        $deficit = $requestedCents - $balanceCents;
        parent::__construct(
            message: "Insufficient funds: short by {$deficit} cents.",
            code: 402,
            previous: $previous,
        );
    }
}
```

Notice:
- We extend `RuntimeException` (a withdrawal failing is a runtime condition).
- **Constructor property promotion** stores domain data (`$balanceCents`, `$requestedCents`) as `readonly` properties.
- We use **named arguments** to pass `message`, `code`, and `previous` to the parent.

Now callers can react precisely:

```php
<?php
try {
    $account->withdraw(5000);
} catch (InsufficientFundsException $e) {
    echo $e->getMessage() . "\n";
    echo "You need {$e->requestedCents} but have {$e->balanceCents}.\n";
}
```

### The `Throwable` API (memorize these)

Every `Throwable` exposes:

| Method | Returns |
|---|---|
| `getMessage(): string` | Human-readable message |
| `getCode(): int` (or mixed) | Error code you assigned |
| `getFile(): string` | File where it was thrown |
| `getLine(): int` | Line number |
| `getTrace(): array` | Stack trace as a structured array |
| `getTraceAsString(): string` | Stack trace formatted as text |
| `getPrevious(): ?Throwable` | The chained "cause" exception (see §8.1) |
| `__toString(): string` | Full string representation (message + trace) |

```php
<?php
try {
    throw new LogicException("Bad state", 99);
} catch (LogicException $e) {
    echo $e->getMessage() . "\n"; // Bad state
    echo $e->getCode() . "\n";    // 99
    echo $e->getLine() . "\n";    // line of the throw
}
```

> **Gotcha:** `Exception::__construct`'s `$code` parameter is typed `int`. `PDOException` famously breaks this — its code can be a string SQLSTATE — so don't assume `getCode()` is always numerically meaningful for third-party exceptions.

### 8.1 Exception chaining (`previous`) — preserve the root cause

When you catch a low-level exception and rethrow a higher-level one, **pass the original as `$previous`**. This preserves the full causal chain so your logs show *both* the friendly wrapper and the technical root cause.

```php
<?php
function loadUserProfile(int $id): array
{
    try {
        return $db->fetchProfile($id); // may throw PDOException
    } catch (PDOException $e) {
        // Wrap the low-level DB error in a domain-meaningful one,
        // but keep the original as the cause.
        throw new ProfileUnavailableException(
            "Could not load profile for user {$id}.",
            previous: $e
        );
    }
}
```

Walking the chain later:

```php
<?php
catch (ProfileUnavailableException $e) {
    $current = $e;
    while ($current !== null) {
        echo $current::class . ": " . $current->getMessage() . "\n";
        $current = $current->getPrevious();
    }
}
// Output:
// ProfileUnavailableException: Could not load profile for user 42.
// PDOException: SQLSTATE[HY000] [2002] Connection refused
```

**Never** discard the original exception when wrapping — you'll lose the actual cause.

---

## 9. Converting errors to exceptions with `ErrorException`

Many built-in functions raise *warnings* instead of throwing (e.g. `file_get_contents`, `fopen`, `json_decode` pre-8). You can promote those warnings to catchable exceptions with `ErrorException`.

```php
<?php
set_error_handler(function (int $severity, string $message, string $file, int $line): bool {
    if (!(error_reporting() & $severity)) {
        return false; // honor @ and error_reporting level
    }
    throw new ErrorException($message, 0, $severity, $file, $line);
});

try {
    $contents = file_get_contents('/does/not/exist'); // raises E_WARNING
} catch (ErrorException $e) {
    echo "Handled: " . $e->getMessage() . "\n";
} finally {
    restore_error_handler(); // pop our handler off the stack
}
// Output:
// Handled: file_get_contents(/does/not/exist): Failed to open stream: No such file or directory
```

> **Modern alternative:** Prefer functions that *throw natively*. `json_decode($s, true, flags: JSON_THROW_ON_ERROR)` throws `JsonException`; `intdiv`/`%` throw `DivisionByZeroError`. Reach for the error-handler trick only when a function offers no throwing mode.

> **`@` silencing:** The `@` operator suppresses warnings but is a code smell — it hides errors and has a performance cost. Note that even `@`-suppressed errors still invoke a custom `set_error_handler` (which is why the `error_reporting() & $severity` guard above matters).

---

## 10. Exception handling in Laravel 12

Laravel wraps every request in a `try/catch`. Any uncaught `Throwable` is funneled to the framework's exception handler, which **reports** it (logs/Sentry/etc.) and **renders** it (HTML page or JSON).

### 10.1 Configuration lives in `bootstrap/app.php` (Laravel 11+)

This is the **biggest change from Laravel 10**. In Laravel 10 you edited `app/Exceptions/Handler.php`. In **Laravel 11 and 12**, that file is gone by default — you configure exception behavior fluently in `bootstrap/app.php`:

```php
<?php
// bootstrap/app.php (Laravel 11/12)
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use App\Exceptions\ProfileUnavailableException;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(/* ... */)
    ->withExceptions(function (Exceptions $exceptions): void {
        // Don't flood logs with these
        $exceptions->dontReport([
            ProfileUnavailableException::class,
        ]);

        // Add context to every reported exception
        $exceptions->context(fn () => [
            'tenant_id' => tenant()?->id,
        ]);

        // Custom reporting for a specific type
        $exceptions->report(function (ProfileUnavailableException $e): void {
            logger()->channel('profiles')->warning($e->getMessage());
        });

        // Custom rendering for a specific type
        $exceptions->render(function (ProfileUnavailableException $e, $request) {
            return response()->json(['error' => 'Profile unavailable'], 503);
        });
    })->create();
```

> **Laravel 10 equivalent:** the same logic went in the `register()` method of `app/Exceptions/Handler.php` using `$this->reportable(...)` and `$this->renderable(...)`. The behavior is identical; only the *location* moved.

### 10.2 HTTP exceptions & `abort()`

Laravel maps certain exceptions to HTTP status codes. Use the `abort()` helper to throw them:

```php
<?php
abort(404);                          // throws NotFoundHttpException -> 404 page
abort(403, 'You shall not pass.');   // 403 with a message
abort_if($post->is_private && !$user, 403);
abort_unless($user->isAdmin(), 403);

throw new \Symfony\Component\HttpKernel\Exception\HttpException(503, 'Maintenance');
```

Common framework exceptions and their status codes:

| Exception | HTTP status |
|---|---|
| `ModelNotFoundException` (e.g. `findOrFail`) | 404 |
| `AuthenticationException` | 401 |
| `AuthorizationException` / `AccessDeniedHttpException` | 403 |
| `ValidationException` | 422 |
| `ThrottleRequestsException` | 429 |
| `NotFoundHttpException` | 404 |

### 10.3 Reportable & renderable exceptions

A custom exception can define its own `report()` and `render()` methods — Laravel calls them automatically:

```php
<?php
namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class PaymentFailedException extends Exception
{
    public function report(): bool
    {
        logger()->channel('payments')->error($this->getMessage());

        // IMPORTANT: even with a custom report() method, Laravel STILL writes to
        // the default logging stack as well — unless you return false here.
        return false; // we've logged it our way; stop the default log entry
    }

    public function render(Request $request): JsonResponse
    {
        return response()->json([
            'message' => 'Payment could not be processed.',
        ], 402);
    }
}
```

> **Reportable nuance:** Returning `void` from `report()` lets Laravel *also* log through the default stack (you get two entries). Return `false` to suppress the default; return `true` when you specifically want the default logging in addition to your custom logic. The same is true for `withExceptions()->report(...)` callbacks (use `->stop()` or return `false`).

> **Renderable nuance:** If your exception extends an already-renderable framework/Symfony exception, you can return `false` from `render()` to fall back to Laravel's default response. A `render()` method may also type-hint dependencies for injection.

### 10.4 Content negotiation (web vs API)

Laravel decides between an HTML and a JSON response by inspecting the request's `Accept` header (effectively `$request->expectsJson()`) — **not** by the route path. So an XHR/`fetch` call sending `Accept: application/json` gets JSON; a plain browser request gets the HTML page. You can override this rule with `$exceptions->shouldRenderJsonWhen(fn (Request $r, Throwable $e) => $r->is('api/*') || $r->expectsJson())` in `bootstrap/app.php`. With `APP_DEBUG=true` a JSON error includes the message, exception class, file, line, and trace; with `APP_DEBUG=false` it returns a clean `{"message": "Server Error"}`. This is why `APP_DEBUG=false` in production is non-negotiable.

```json
// APP_DEBUG=false, API request, uncaught error
{
    "message": "Server Error"
}
```

```json
// A ValidationException always returns 422 with field errors (regardless of debug)
{
    "message": "The email field is required.",
    "errors": {
        "email": ["The email field is required."]
    }
}
```

### 10.5 Reporting without stopping: `report()`

Sometimes you want to log an exception but keep going. The `report()` helper sends it through the exception handler without halting:

```php
<?php
try {
    $this->syncToCrm($user);
} catch (\Throwable $e) {
    report($e);   // log it, then continue gracefully
}
// ... request continues
```

### 10.6 Other `withExceptions` tools worth knowing (Laravel 12)

These round out the fluent API and come up in real apps and interviews:

```php
<?php
// bootstrap/app.php
use Illuminate\Support\Lottery;
use Illuminate\Cache\RateLimiting\Limit;
use Psr\Log\LogLevel;
use Throwable;
use PDOException;

->withExceptions(function (Exceptions $exceptions): void {
    // Mark an exception as never-reported via an interface instead of dontReport():
    //   class FooException extends Exception implements \Illuminate\Contracts\Debug\ShouldntReport {}

    // Conditionally ignore reporting
    $exceptions->dontReportWhen(fn (Throwable $e) => $e instanceof PaymentFailedException);

    // Force a specific log level for a type
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);

    // Throttle / sample noisy exceptions (return a Lottery or a Limit)
    $exceptions->throttle(fn (Throwable $e) => Lottery::odds(1, 100));

    // De-duplicate the same instance reported multiple times
    $exceptions->dontReportDuplicates();

    // Last-resort hook to rewrite the final Symfony Response
    $exceptions->respond(function (\Symfony\Component\HttpFoundation\Response $response) {
        return $response;
    });
})->create();
```

> `dontReport()` / `ShouldntReport` only suppress *reporting* (logging) — a custom `render()` still runs. To stop a *built-in* ignored type (404, CSRF 419) from being ignored, use `$exceptions->stopIgnoring(SomeException::class)`.

---

## ⚠️ Common Mistakes & Gotchas

1. **Swallowing exceptions silently.**
   ```php
   // BAD — the error disappears with no trace
   try { doWork(); } catch (\Throwable $e) { /* nothing */ }
   ```
   **Fix:** At minimum log it (`report($e)` in Laravel, or `error_log`). Only swallow when you have a *deliberate*, documented reason, and prefer a non-capturing `catch (SpecificType)` so you're explicit about *what* you're ignoring.

2. **Catching `\Exception` and expecting to catch `Error`s.**
   ```php
   try { $obj->method(); }      // $obj is null -> Error, NOT Exception
   catch (\Exception $e) { ... } // never runs!
   ```
   **Fix:** Catch `\Throwable` when you genuinely want everything; catch `\Exception` only when you specifically don't want engine errors.

3. **Returning (or throwing) from `finally`.**
   ```php
   try { return compute(); } finally { return $fallback; } // overrides + may hide exceptions
   ```
   **Fix:** Use `finally` only for cleanup. Never `return`/`throw` from it.

4. **Catch order from general to specific (dead code).**
   ```php
   try { ... }
   catch (\Throwable $e) { ... }        // catches everything...
   catch (InvalidArgumentException $e) { ... } // ...so this is unreachable
   ```
   **Fix:** Order `catch` blocks from **most specific to most general**.

5. **Losing the root cause when rethrowing.**
   ```php
   catch (PDOException $e) { throw new AppException("DB failed"); } // $e lost!
   ```
   **Fix:** Always chain: `throw new AppException("DB failed", previous: $e);`.

6. **Leaving `APP_DEBUG=true` / `display_errors=On` in production.**
   **Fix:** `APP_DEBUG=false` and `display_errors=Off` in prod; keep `log_errors=On` so you still capture everything server-side.

7. **Using exceptions for ordinary control flow.**
   Throwing to break out of a loop or signal a normal "not found" on the happy path is slow (trace capture) and confusing.
   **Fix:** Reserve exceptions for *exceptional* conditions; return values/`null`/enums for expected outcomes.

---

## ✅ Best Practices

- **Fail fast.** Validate inputs at the boundary and throw immediately on violated preconditions rather than propagating bad state.
- **Throw the most specific type** that fits (custom > SPL subclass > `RuntimeException`/`LogicException` > `Exception`). Specific types make `catch` precise.
- **Catch narrowly, handle deliberately.** Catch the specific types you can actually do something about; let everything else bubble to the global handler.
- **Always chain with `previous`** when wrapping a lower-level exception.
- **Never swallow** — log at minimum. Silence hides production fires.
- **Use `finally` for cleanup** (closing files, releasing locks, rolling back transactions) — and never `return` from it.
- **Centralize** logging/reporting in the global handler (or Laravel's `withExceptions`), not in every `catch`.
- **Prefer throwing built-ins** (`JSON_THROW_ON_ERROR`, `intdiv`) over the `@`-suppress-and-check pattern.
- **Keep messages actionable but safe** — enough to debug from logs, never leaking secrets, and never shown raw to end users in production.
- **Don't over-catch in Laravel** — let the framework handle rendering/reporting; only catch when you'll genuinely recover or transform the error.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between an `Error` and an `Exception` in PHP 7+?**
A. Both implement `Throwable`. `Error` represents engine-level problems (type errors, calling methods on null, division by zero) that usually indicate bugs; `Exception` is the base you extend for application-level conditions. Catching `\Exception` does *not* catch an `Error` — you need `\Throwable` for that.

**Q2. How does PHP's exception mechanism work under the hood?**
A. When `throw` executes, the engine unwinds the call stack frame by frame, running any `finally` blocks it passes, until it finds a `catch` whose declared type (or supertype) matches the thrown object via `instanceof`. If none matches, the engine invokes the handler registered by `set_exception_handler` (Laravel registers one), then terminates. The exception object captures the file, line, and a backtrace at throw time — which is why throwing is comparatively expensive and shouldn't be used for normal control flow.

**Q3. When would you use `LogicException` vs `RuntimeException`?**
A. `LogicException` for errors that should have been caught in code/review — violated preconditions, bad arguments (a *bug*). `RuntimeException` for conditions only detectable while running — missing files, network failures, unexpected data (often *recoverable*). The classic pair: `OutOfRangeException` (logic, a key you should know is wrong) vs `OutOfBoundsException` (runtime, an index that turned out invalid).

**Q4. What's exception chaining and why does it matter?**
A. Passing the caught exception as the `$previous` argument when throwing a new one. It preserves the full causal chain (`getPrevious()` walks it), so logs show both the friendly wrapper and the technical root cause. Without it you lose the actual reason something failed.

**Q5. Explain `finally` and its interaction with `return`.**
A. `finally` always runs — after `try`/`catch`, even when they `return` (the return value is staged, then `finally` runs, then it's returned). But a `return` *inside* `finally` overrides the staged value, and a `return`/`throw` in `finally` can silently swallow a pending exception. So: cleanup only, never return from `finally`.

**Q6. How do you catch a fatal error in PHP?**
A. You can't catch true fatals (`E_ERROR`, `E_PARSE`) with `try/catch` or `set_error_handler`. The only hook is `register_shutdown_function` combined with `error_get_last()` to inspect and react during shutdown. (Note many former fatals in PHP 7+ are now catchable `Error` exceptions.)

**Q7. What do `error_reporting`, `display_errors`, and `log_errors` control, and how should prod differ from dev?**
A. `error_reporting` is a bitmask of which levels are reported; `display_errors` sends them to output; `log_errors` writes them to a file. In dev: display on, log on, report `E_ALL`. In prod: **display off** (security/UX), log on, still report `E_ALL`. In Laravel this is governed by `APP_DEBUG`.

**Q8. How do you convert a PHP warning into a catchable exception?**
A. Register a `set_error_handler` that throws an `ErrorException($message, 0, $severity, $file, $line)`, guarded by `error_reporting() & $severity` so it respects silencing. Prefer native throwing modes (`JSON_THROW_ON_ERROR`) where available.

**Q9. Where does exception handling configuration live in Laravel 12 vs Laravel 10?**
A. Laravel 11/12 configures it fluently in `bootstrap/app.php` via `->withExceptions(fn (Exceptions $e) => ...)` with `report()`, `render()`, `dontReport()`, and `context()`. Laravel 10 used the `register()` method of `app/Exceptions/Handler.php` with `reportable()`/`renderable()`.

**Q10. What's `throw` as an expression, and give an idiomatic use.**
A. Since PHP 8.0 `throw` can appear in expression positions, so `$user = find($id) ?? throw new NotFoundException($id);` and `match` arms can throw in their `default`. It makes "this must exist or fail" a one-liner.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Hierarchy ---
Throwable                 // catch everything
├─ Error                  // engine bugs: TypeError, ValueError, DivisionByZeroError, UnhandledMatchError...
└─ Exception              // app-level
   ├─ ErrorException      // wraps engine errors
   ├─ LogicException      // BUG: InvalidArgumentException, DomainException, OutOfRangeException, BadMethodCallException...
   └─ RuntimeException    // RUNTIME: OutOfBoundsException, UnexpectedValueException, RangeException...

// --- try/catch/finally ---
try { ... }
catch (TypeA | TypeB $e) { ... }   // multi-catch (7.1+)
catch (TypeC) { ... }               // non-capturing (8.0+)
catch (\Throwable $e) { ... }       // safety net (most general LAST)
finally { /* cleanup only — no return! */ }

// --- throw as expression (8.0+) ---
$x = maybe() ?? throw new RuntimeException('missing');

// --- Throwable API ---
$e->getMessage(); $e->getCode(); $e->getFile(); $e->getLine();
$e->getTrace(); $e->getTraceAsString(); $e->getPrevious();

// --- Chaining ---
throw new AppException('high level', previous: $low);

// --- Custom exception ---
final class FooException extends RuntimeException {
    public function __construct(public readonly int $id, ?\Throwable $previous = null) {
        parent::__construct("Foo {$id} failed", 0, $previous);
    }
}
```

```php
// --- Global handlers ---
set_error_handler(fn($sev,$msg,$f,$l) =>
    error_reporting() & $sev
        ? throw new ErrorException($msg, 0, $sev, $f, $l)
        : false);
set_exception_handler(fn(\Throwable $e) => error_log((string) $e));
register_shutdown_function(fn() => /* check error_get_last() for fatals */ null);
restore_error_handler(); restore_exception_handler();
```

```ini
; --- dev ---            ; --- prod ---
error_reporting=E_ALL    error_reporting=E_ALL
display_errors=On        display_errors=Off
log_errors=On            log_errors=On
```

```php
// --- Laravel 12 (bootstrap/app.php) ---
->withExceptions(function (Exceptions $e) {
    $e->dontReport([MyException::class]);                 // never log this type
    $e->report(fn (MyException $x) => logger()->error($x->getMessage()))
      ->stop();                                           // ->stop() (or return false) = skip default log
    $e->render(fn (MyException $x, $req) => response()->json([/* ... */], 503));
    $e->context(fn () => ['user_id' => auth()->id()]);    // user id auto-added anyway
    $e->level(PDOException::class, \Psr\Log\LogLevel::CRITICAL);
    $e->throttle(fn ($x) => \Illuminate\Support\Lottery::odds(1, 100));
    $e->shouldRenderJsonWhen(fn ($req, $x) => $req->expectsJson());
})

// --- Laravel helpers ---
abort(404); abort_if($cond, 403); abort_unless($cond, 403);
report($e);          // log without halting
findOrFail($id);     // -> ModelNotFoundException -> 404
```

---

## 🧪 Mini Exercises

1. **Custom hierarchy.** Create a base `OrderException extends RuntimeException` and two subclasses, `OutOfStockException` and `PaymentDeclinedException`. `OutOfStockException` should accept and store a `readonly int $sku` and `readonly int $requested`. Write a `placeOrder()` function that throws the appropriate one, and a caller that handles each with a separate `catch`.

2. **Error-to-exception bridge.** Register a `set_error_handler` that converts warnings into `ErrorException`. Call `fopen('/no/such/file', 'r')`, catch the resulting `ErrorException`, print its message, and use `restore_error_handler()` in a `finally`. Verify a *suppressed* call (`@fopen(...)`) is NOT converted.

3. **`finally` proof.** Write a function with `return "A"` in `try` and `echo` in `finally`; then add a second version that *also* `return "B"` in `finally`. Predict and confirm the output of each, and explain the difference in a comment.

4. **Chaining.** Write `parseConfig(string $json)` that calls `json_decode(..., flags: JSON_THROW_ON_ERROR)`, catches the `JsonException`, and rethrows a `DomainException("Invalid config")` with the original as `previous`. In the caller, walk and print the full `getPrevious()` chain.

5. **Laravel rendering.** In a fresh Laravel 12 app, create `App\Exceptions\QuotaExceededException`, give it a `render()` method returning a 429 JSON response with a `Retry-After` header, throw it from a controller, and confirm the response for both a browser request and an `Accept: application/json` request.
