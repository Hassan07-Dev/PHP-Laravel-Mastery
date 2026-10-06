# Syntax, Variables & Data Types in PHP

> Module 02 of the "Zero to Expert" PHP/Laravel track. Targets **PHP 8.4** (with notes for 8.1–8.3) and **Laravel 12** (with notes for Laravel 10/11). Everything here is the bedrock that every later module — Eloquent, validation, queues — silently relies on.

**What you'll learn**

- How PHP variables work: the `$` prefix, naming rules, case-sensitivity, and what "dynamic typing" really means.
- Variable **scope** (local, global, the `global` keyword, `static` variables) and why functions can't see your outer variables.
- **Constants** (`define` vs `const`), the new `final` class constants, enum constants, and magic constants like `__LINE__` / `__FILE__`.
- Every PHP **data type** — scalar, compound, and special — plus the `callable` / `iterable` pseudo-types.
- **Type juggling**, coercion, truthy/falsy rules, casting, and the float-precision and integer-overflow traps that bite people in interviews and production.
- How to **inspect types** correctly with `var_dump`, `gettype`, the `is_*` family, and the modern `get_debug_type()`.
- `null` semantics and the `isset` / `empty` / `is_null` distinction.
- **Variable variables** and **references** (the `&` operator) — what they do and when (not) to use them.

---

## 1. Why this module matters

Almost every bug a junior backend dev ships traces back to a misunderstanding here: `"0"` being falsy, `0.1 + 0.2 !== 0.3`, a function that "can't see" a variable, or `empty()` reporting `true` for the string `"0"`. PHP is a **dynamically typed, weakly typed** language with decades of backward-compatibility baggage. The rules are learnable and mostly consistent — but only if you learn them deliberately instead of by trial and error. This module makes the rules explicit so you stop guessing.

A quick vocabulary check before we start:

- **Dynamically typed**: a variable's type is determined at runtime by the value it holds, not declared ahead of time. The *variable* has no type; the *value* does.
- **Weakly typed (a.k.a. coercive)**: PHP will silently convert between types in many contexts (e.g. `"5" + 3 === 8`). The opposite is *strongly typed*, where such mixing is an error.

---

## 2. PHP file syntax in 60 seconds

PHP is embedded in text. Code lives between `<?php` and `?>` tags; everything outside is emitted verbatim.

```php
<?php
$name = "Ada";
echo "Hello, $name\n";   // inside double quotes, $name is interpolated
echo 'Hello, $name\n';   // single quotes: literal, prints: Hello, $name\n
```

```
Output:
Hello, Ada
Hello, $name\n
```

Key syntax facts:

- Statements end with a semicolon `;`.
- The closing `?>` is **optional** and you should **omit it in pure-PHP files** (no HTML). A stray newline after `?>` becomes output and breaks `header()` calls and Laravel responses with the dreaded "headers already sent" error. PSR-12 mandates omitting it.
- PHP is **case-insensitive for keywords, function names, and class names** (`ECHO`, `Echo`, `echo` all work; `MyClass` and `myclass` refer to the same class) — but **case-sensitive for variable names and constants** (`$user` ≠ `$User`). Convention: write keywords lowercase.

```php
<?php
ECHO "works but ugly\n";   // valid — echo is case-insensitive
$User = 1;
$user = 2;
echo $User, $user;          // 12 — two different variables
```

---

## 3. Variables

### 3.1 The rules

A variable is a named box for a value. In PHP every variable name begins with `$`.

```php
<?php
$count   = 10;          // valid
$_total  = 0;           // valid — may start with underscore
$user2   = "Ada";       // valid — digits allowed after first char
// $2user = "bad";      // INVALID — cannot start with a digit
// $my-var = 1;         // INVALID — no hyphens; that's "minus"
```

Rules, precisely:

1. Starts with `$`.
2. First character after `$` must be a **letter (a–z, A–Z) or underscore `_`**.
3. Subsequent characters may be letters, digits, or underscores.
4. Names are **case-sensitive**.
5. Technically bytes `0x80–0xff` are allowed too (so non-ASCII names work), but never rely on this.

**Why `$`?** It lets the parser distinguish variables from constants and bare words without ambiguity, and it's what makes *string interpolation* (`"Hello $name"`) and *variable variables* (covered later) possible.

### 3.2 Assignment and dynamic typing

Assignment uses `=`. There is **no type declaration** on the variable itself — the same variable can hold different types over its lifetime.

```php
<?php
$x = 42;            // int
$x = "forty-two";   // now a string — perfectly legal
$x = [1, 2, 3];     // now an array
echo gettype($x);   // array
```

PHP variables are assigned **by value** by default — assigning copies the value:

```php
<?php
$a = 5;
$b = $a;   // $b is a copy
$b = 99;
echo $a;   // 5 — $a is untouched
```

> Performance note: PHP uses **copy-on-write (COW)** internally. `$b = $a` does *not* immediately duplicate large arrays in memory; the copy happens lazily only when one side is modified. So the by-value model is safe *and* efficient. (Objects are the exception — see §5.6.)

### 3.3 Scope: where a variable is visible

A variable's **scope** is the region of code where it's accessible. PHP has function-level scope (not block-level like C/JS `let`):

```php
<?php
$message = "outer";

function show() {
    echo $message;   // Warning: Undefined variable $message  →  prints nothing/null
}
show();
```

Inside `show()`, `$message` is **not** visible — functions get a fresh, isolated scope. This trips up newcomers from JavaScript. There are three ways to reach across the boundary:

**(a) Pass it as a parameter (preferred):**

```php
<?php
function show(string $message): void {
    echo $message;
}
show("outer");   // outer
```

**(b) The `global` keyword** (use sparingly — it's a code smell):

```php
<?php
$config = "prod";
function env(): void {
    global $config;       // bind the global into local scope
    echo $config;         // prod
}
env();
// Equivalent: echo $GLOBALS['config'];  — the superglobal array
```

**(c) Closures with `use`** (the modern, controlled way to capture):

```php
<?php
$tax = 0.2;
$withTax = fn(float $price) => $price * (1 + $tax);   // arrow fn auto-captures $tax by value
echo $withTax(100);   // 120

$multiplier = 3;
$scale = function (int $n) use ($multiplier) {          // classic closure: explicit capture
    return $n * $multiplier;
};
echo $scale(4);       // 12
```

Note: **arrow functions** (`fn`) capture outer variables automatically **by value**; classic `function () use (...)` requires you to list captures explicitly (and you can capture **by reference** with `use (&$x)`).

### 3.4 Static variables

A `static` variable inside a function keeps its value **between calls** but stays private to that function. Initialization runs once.

```php
<?php
function counter(): int {
    static $n = 0;   // initialized only on the first call
    return ++$n;
}
echo counter();   // 1
echo counter();   // 2
echo counter();   // 3
```

> **PHP 8.1+**: the static initializer may use a `new` expression (`static $cache = new Cache();`). **PHP 8.3+**: static variables in methods are now shared across inheritance correctly, and the *initializer can reference dynamic expressions* more freely. Use static vars for memoization, but be aware they make functions stateful and harder to test.

### 3.5 Variable variables

A `$$` lets you use the *value* of one variable as the *name* of another.

```php
<?php
$key = "color";
$$key = "red";       // creates a variable named $color
echo $color;          // red
echo ${$key};         // red — explicit/brace form (use this; it's clearer)
```

This is occasionally handy for metaprogramming but usually a sign you actually want an **array** (`$data['color']`). Avoid in application code; arrays are clearer, safer, and faster. **Note:** with arrays you must use braces — `$$arr['x']` is ambiguous, write `${$arr['x']}`.

> ⚠️ **Security:** never build a variable name from untrusted input — `$$_GET['k'] = $_GET['v']` (or the related `extract($_GET)`) lets an attacker overwrite arbitrary local variables, a classic *variable-injection* vulnerability. Inside a Laravel controller this could clobber `$user`, `$isAdmin`, etc. Keep request data in arrays (`$request->input('k')`), never splat it into the symbol table.

### 3.6 References (the `&` operator)

A reference makes two names point at the **same underlying value** — change one, the other changes. This is *not* a pointer (no address arithmetic); it's an alias.

```php
<?php
$a = 1;
$b = &$a;   // $b is an alias of $a
$b = 99;
echo $a;    // 99 — they share the same storage

unset($b);  // breaks the reference, does NOT destroy the value
echo $a;    // 99
```

References by parameter (mutate the caller's variable):

```php
<?php
function addOne(int &$n): void {   // & makes $n a reference to the argument
    $n++;
}
$count = 5;
addOne($count);
echo $count;   // 6
```

A classic gotcha — leftover reference in a `foreach`:

```php
<?php
$nums = [1, 2, 3];
foreach ($nums as &$v) { $v *= 2; }   // $v is a reference to each element
// $nums is now [2, 4, 6]
unset($v);   // ALWAYS unset the reference variable after the loop!

foreach ($nums as $v) { /* ... */ }   // without the unset above, the LAST element gets clobbered
```

> Rule of thumb: prefer return values and immutability over references. They exist for the rare case where you genuinely need shared mutable state or want to avoid copying a huge structure that you must modify in place.

---

## 4. Constants

A **constant** is a name bound to a value that **cannot change** after definition and has no `$`. Use them for fixed configuration, magic numbers, and flags.

### 4.1 `const` vs `define()`

```php
<?php
const MAX_USERS = 100;                 // compile-time, must be top-level or in a class
define('API_VERSION', 'v2');           // runtime function call

echo MAX_USERS;     // 100
echo API_VERSION;   // v2
```

| Feature | `const` | `define()` |
|---|---|---|
| Evaluated | Compile time | Runtime |
| Can be conditional / inside `if` | No (must be top-level or class member) | Yes |
| Dynamic name | No | Yes (`define($name, $val)`) |
| Works in classes | Yes (class constants) | No |
| Case-insensitive option | No | Removed in PHP 8.0 (was deprecated) |

Modern guidance: use `const` for almost everything (faster, scoped, supports class constants). Reach for `define()` only when the constant name or condition is dynamic.

```php
<?php
// define() can be conditional — const cannot:
if (!defined('APP_ENV')) {
    define('APP_ENV', getenv('APP_ENV') ?: 'production');
}
```

Constants can be arrays, and class constants can have visibility:

```php
<?php
const ROLES = ['admin', 'editor', 'viewer'];   // array constant — fine

class Http {
    public const int OK = 200;         // PHP 8.3+: typed class constants
    protected const string BASE = '/api';
    final public const RETRIES = 3;    // PHP 8.1+: final prevents override in subclasses
}
echo Http::OK;   // 200
```

> **PHP 8.3** added **typed class constants** (`public const int OK = 200;`). **PHP 8.1** added `final` class constants and `enum` (see below). **PHP 8.4** allows new expressions as default constant/parameter values in more places.

### 4.2 Enums (the modern alternative to scattered constants)

Since PHP 8.1, **enums** group related constant values into a type. Prefer them over loose `const` flags.

```php
<?php
enum Status: string {       // "backed" enum — each case has a scalar value
    case Active   = 'active';
    case Inactive = 'inactive';
    case Banned   = 'banned';

    public function label(): string {
        return match ($this) {
            Status::Active   => 'Active user',
            Status::Inactive => 'Inactive user',
            Status::Banned   => 'Banned user',
        };
    }
}

echo Status::Active->value;          // active
echo Status::from('banned')->label(); // Banned user
var_dump(Status::tryFrom('nope'));    // NULL  (tryFrom doesn't throw; from() would)
```

### 4.3 Magic constants

These are special tokens resolved by the compiler based on **where they appear**. They start and end with double underscores (`__`).

```php
<?php
namespace App\Demo;

class Widget {
    public function info(): void {
        echo __LINE__, "\n";      // current line number, e.g. 8
        echo __FILE__, "\n";      // absolute path of this file
        echo __DIR__, "\n";       // directory of this file (no trailing slash)
        echo __FUNCTION__, "\n";  // info
        echo __METHOD__, "\n";    // App\Demo\Widget::info
        echo __CLASS__, "\n";     // App\Demo\Widget
        echo __NAMESPACE__, "\n"; // App\Demo
    }
}
```

Full list: `__LINE__`, `__FILE__`, `__DIR__`, `__FUNCTION__`, `__CLASS__`, `__METHOD__`, `__NAMESPACE__`, `__TRAIT__`, and `ClassName::class` (the `::class` constant, which yields the fully-qualified class name as a string and is the idiomatic, refactor-safe way to reference a class).

```php
<?php
use App\Models\User;
echo User::class;   // App\Models\User  — survives renames, IDE-friendly
```

In Laravel you'll see `__DIR__` constantly for path building (`require __DIR__.'/../vendor/autoload.php';`) and `::class` everywhere (`User::class`, route actions, event listeners, `config('auth.providers.users.model')` defaults).

---

## 5. The data types

PHP has **eight concrete runtime types** (the ones `get_debug_type()` / `gettype()` can actually report), plus a set of pseudo-types that only appear in type declarations. The eight split into three families:

- **Scalar** (a single value): `int`, `float`, `string`, `bool`.
- **Compound** (hold multiple values): `array`, `object`.
- **Special**: `null`, `resource`.

**Pseudo-types** are *not* real runtime types — you'll never see them from `get_debug_type()`; they only show up in type hints to accept a *family* of values: `callable`, `iterable` (= `array|Traversable`), `mixed`, `void`, `never`, `self`, `static`, `parent`, and the standalone literal types `false` / `true` / `null` (allowed as standalone types since **PHP 8.2**). (Older PHP manuals listed `callable`/`iterable` under "compound types"; modern PHP treats them as pseudo-types, which is how we cover them in §5.9.)

### 5.1 Integer (`int`)

Whole numbers. Platform-dependent size — **64-bit** on virtually all modern systems (range roughly ±9.2 × 10¹⁸, exposed as `PHP_INT_MAX` / `PHP_INT_MIN` / `PHP_INT_SIZE`).

```php
<?php
$dec  = 255;
$hex  = 0xFF;        // 255
$oct  = 0o17;        // 15  (PHP 8.1+ explicit octal prefix; older: 017)
$bin  = 0b1010;      // 10
$big  = 1_000_000;   // underscores as digit separators (PHP 7.4+), value is 1000000
echo PHP_INT_MAX;    // 9223372036854775807 on 64-bit
```

**Integer overflow becomes float** — PHP does **not** wrap around; it silently promotes to float (losing exact-integer precision):

```php
<?php
$n = PHP_INT_MAX;          // 9223372036854775807
var_dump($n);              // int(9223372036854775807)
var_dump($n + 1);          // float(9.223372036854776E+18)  ← became a float!
```

### 5.2 Float (`float`)

Double-precision (IEEE 754 64-bit) real numbers. Also called "double". Has special values `INF`, `-INF`, and `NAN`.

```php
<?php
$pi   = 3.14159;
$sci  = 1.5e3;     // 1500.0
$inf  = INF;
$nan  = NAN;
var_dump(is_nan($nan));   // bool(true)
var_dump(NAN === NAN);    // bool(false) — NaN is never equal to anything, even itself
```

**Float precision pitfall** — the single most-asked-about gotcha:

```php
<?php
var_dump(0.1 + 0.2 === 0.3);   // bool(false)
echo 0.1 + 0.2;                 // 0.3 (echo rounds for display!)
printf("%.17f\n", 0.1 + 0.2);  // 0.30000000000000004
```

Binary floating point cannot represent most decimal fractions exactly. **Never** use `==`/`===` to compare floats and **never store money as float**. Fixes:

```php
<?php
// (a) Compare within an epsilon tolerance:
$eps = 1e-9;
var_dump(abs((0.1 + 0.2) - 0.3) < $eps);   // bool(true)

// (b) Use integers (store money in cents): 1050 instead of 10.50
// (c) Use BCMath / GMP for arbitrary precision:
echo bcadd('0.1', '0.2', 1);   // 0.3  (string math, exact)
```

> In **Laravel**, use the `decimal:2` Eloquent cast for currency columns (returns a string to preserve precision) or a value object, and a `DECIMAL(10,2)` column — never `FLOAT`/`DOUBLE` for money.

### 5.3 String (`string`)

A sequence of bytes (not Unicode code points — a `string` is byte-oriented; `strlen()` returns *bytes*, use `mb_strlen()` for characters). Four syntaxes:

```php
<?php
$single = 'No $interpolation here, only \\ and \' escapes';
$double = "Has $name interpolation and \n \t escapes";
$heredoc = <<<TXT
Behaves like double quotes: $name interpolated.
TXT;
$nowdoc = <<<'TXT'
Behaves like single quotes: $name is literal.
TXT;
```

Interpolation supports simple and complex (brace) forms:

```php
<?php
$user = ['name' => 'Ada'];
echo "Hi {$user['name']}";       // Hi Ada — braces needed for array/object access
$obj = new stdClass; $obj->n = 1;
echo "Val: {$obj->n}";           // Val: 1
```

Concatenation uses `.` (dot), not `+`:

```php
<?php
echo "foo" . "bar";   // foobar
$s = "a"; $s .= "b";  // $s is now "ab"
```

### 5.4 Boolean (`bool`)

Just `true` and `false` (case-insensitive keywords). The crucial knowledge is **what counts as falsy** (see §6).

```php
<?php
var_dump(true, false);   // bool(true) bool(false)
```

### 5.5 Array (`array`)

PHP arrays are **ordered maps** — a single type that serves as list, dictionary, stack, and queue. Keys are `int` or `string`; values are anything.

```php
<?php
$list  = [1, 2, 3];                        // int keys 0,1,2 (a "list")
$assoc = ['name' => 'Ada', 'age' => 36];   // associative (string keys)
$mixed = [0 => 'a', 'x' => 'b', 1 => 'c']; // mixed keys allowed

echo $list[0];          // 1
echo $assoc['name'];     // Ada
$list[] = 4;             // append → [1,2,3,4]

// Spread + named-key spread (PHP 8.1+ allows string keys in spread):
$more = [...$list, 5];
$cfg  = ['a' => 1]; $merged = [...$cfg, 'b' => 2];
```

Key coercion gotcha: `"1"` (numeric string) becomes int `1`; `true` becomes `1`; `null` becomes `""`; floats are truncated to int.

```php
<?php
$a = ["1" => 'x', 1 => 'y'];
var_dump(count($a));   // int(1) — "1" and 1 collide; 'y' overwrote 'x'
```

> PHP 8.1+ added `array_is_list()` to check if an array is a sequential 0-indexed list — important because Laravel/JSON serialize lists as `[...]` and maps as `{...}`.

### 5.6 Object (`object`)

An instance of a class. Unlike scalars/arrays, objects are handled **by handle** — assigning an object copies the *handle*, not the object, so both names see the same instance.

```php
<?php
class Box { public int $n = 0; }
$a = new Box();
$b = $a;          // copies the handle, not the object
$b->n = 5;
echo $a->n;       // 5 — same underlying object!

$c = clone $a;    // clone makes a (shallow) copy
$c->n = 99;
echo $a->n;       // 5 — $c is independent
```

The generic object type is `stdClass`; `(object)['a'=>1]` casts an array to one, and `json_decode($json)` returns `stdClass` (or arrays with the second arg `true`).

```php
<?php
$o = (object) ['name' => 'Ada'];
echo $o->name;   // Ada
var_dump($o instanceof stdClass);   // bool(true)
```

Modern class quick-reference (you'll use these constantly in Laravel):

```php
<?php
class User {
    public function __construct(
        public string $name,           // constructor property promotion (PHP 8.0+)
        public readonly int $id = 0,   // readonly: assign once, then immutable (PHP 8.1+)
    ) {}
}
$u = new User(name: 'Ada', id: 1);     // named arguments (PHP 8.0+)
echo $u->name;                          // Ada
// $u->id = 2;                          // Error: Cannot modify readonly property
```

### 5.7 `null` (the unit type)

`null` represents "no value." A variable is `null` if assigned `null`, never set, or `unset()`. There is exactly one value: `null` (case-insensitive).

```php
<?php
$x = null;
var_dump(is_null($x));   // bool(true)
var_dump($x === null);    // bool(true) — preferred check
```

Working with null safely:

```php
<?php
$data = ['user' => null];

echo $data['user'] ?? 'guest';      // null coalescing → guest
$data['count'] ??= 0;                // null-coalescing assignment (PHP 7.4+): set only if null/unset

$obj = null;
echo $obj?->profile?->email ?? 'n/a'; // nullsafe operator (PHP 8.0+): short-circuits to null, no error
```

`isset` vs `empty` vs `is_null` (memorize this table):

| Expression | `$x` undefined | `$x = null` | `$x = 0` | `$x = ""` | `$x = "0"` | `$x = "a"` | `$x = []` |
|---|---|---|---|---|---|---|---|
| `isset($x)` | false | **false** | true | true | true | true | true |
| `empty($x)` | true | true | **true** | true | **true** | false | true |
| `is_null($x)` | (warning)→true | true | false | false | false | false | false |

The two killers: **`empty("0")` is `true`** (the string zero!), and **`isset()` returns `false` for a variable that exists but holds `null`**.

### 5.8 `resource`

A special type holding a handle to an external resource — an open file, database connection, image, or stream. You don't construct these directly; functions like `fopen()`, `curl_init()` return them.

```php
<?php
$fh = fopen('php://temp', 'r+');
var_dump(get_debug_type($fh));   // resource (stream)
fwrite($fh, 'hi');
fclose($fh);                      // free it
```

> Modern PHP is migrating many resources to **opaque objects** (e.g. cURL handles are `CurlHandle` objects since 8.0, GD images are `GdImage` since 8.0). So `resource` is a shrinking category, but file/stream handles remain resources.

### 5.9 Pseudo-types: `callable` and `iterable`

These appear in type declarations to accept a *family* of values, not a single concrete type.

```php
<?php
function apply(callable $fn, int $x): int {   // anything you can call()
    return $fn($x);
}
echo apply(fn($n) => $n * 2, 5);        // 10  — closure passed as callable
echo apply('strlen', 12345);            // 5   — a function-name string is callable; strlen('12345') = 5

function sum(iterable $items): int {     // arrays OR Traversable (generators, collections)
    $t = 0;
    foreach ($items as $i) { $t += $i; }
    return $t;
}
echo sum([1, 2, 3]);                      // 6
echo sum((function () { yield 1; yield 2; })());   // 3 — a Generator is iterable
```

`callable` accepts closures, function-name strings, `[$object, 'method']` arrays, and invokable objects (classes with `__invoke`). `iterable` is shorthand for `array|Traversable`. (Laravel's `Collection` is `Traversable`, hence `iterable`.)

---

## 6. Type juggling, coercion & truthiness

### 6.1 Truthy / falsy

In a boolean context (`if`, `while`, `&&`, `!`, ternary), values are coerced to bool. **These are the ONLY falsy values:**

```text
false
0          (int)
0.0, -0.0  (float)
""         (empty string)
"0"        (the string containing a single zero)
[]         (empty array)
null
```

Everything else is truthy — including `"0.0"`, `"false"`, `" "` (space), `"00"`, `-1`, `[0]` (array with one falsy element), any object, and — surprisingly — **`NAN` is truthy** (`(bool) NAN === true`), even though `NAN` famously isn't equal to anything in numeric comparison.

```php
<?php
var_dump((bool) "0");      // bool(false)  ← the famous one
var_dump((bool) "0.0");    // bool(true)   ← NOT "0", so truthy
var_dump((bool) "false");  // bool(true)   ← non-empty string
var_dump((bool) []);       // bool(false)
var_dump((bool) [0]);      // bool(true)   ← non-empty array
var_dump((bool) "0.00");   // bool(true)
```

### 6.2 Comparison: `==` vs `===`

- `===` (and `!==`) is **identity**: same type *and* same value. Use this by default.
- `==` (and `!=`) is **loose**: coerces types first. Source of countless bugs.

```php
<?php
var_dump(0 == "a");     // PHP 8: bool(false)   (PHP 7: was true!)
var_dump("1" == "01");  // bool(true)  — both numeric strings → compared as numbers
var_dump("10" == "1e1");// bool(true)  — 1e1 == 10 numerically
var_dump(100 == "1e2"); // bool(true)
var_dump(null == false);// bool(true)
var_dump(null === false);// bool(false)
var_dump([] == false);  // bool(true)
```

> **Major PHP 8.0 change**: comparing a number to a non-numeric string now casts the *number to string* (so `0 == "foo"` is `false`). In PHP 7 the string was cast to `0`, making `0 == "foo"` *true* — a notorious security footgun. Mention this in interviews; it shows version awareness.

> ⚠️ **Security — type-juggling auth bypass ("magic hashes"):** never compare secrets with `==`. Two strings that *look* numeric still compare as numbers even in PHP 8, so `"0e123" == "0e456"` is `true` (both parse to `0 × 10ⁿ = 0`). If you do `if (md5($input) == $storedHash)` and both hashes happen to be `0e…`-form, the check passes for the wrong input. Always use `===` for tokens/hashes, and for secret comparison use the constant-time `hash_equals($known, $user)` to also avoid timing attacks. In Laravel, use `Hash::check()` for passwords — never roll your own `==` comparison.

```php
<?php
var_dump("0e1234" == "0e5678");                 // bool(true)  ← magic-hash bypass
var_dump("0e1234" === "0e5678");                // bool(false) ← safe
var_dump(hash_equals("0e1234", "0e5678"));      // bool(false) ← safe + constant-time
```

### 6.3 Arithmetic & string coercion

```php
<?php
var_dump(5 + "5");        // int(10)   — clean numeric string coerced, no warning
var_dump(5 + "5abc");     // int(10)   — leading-numeric string: parses 5, BUT emits
                          //            Warning: "A non-numeric value encountered" (PHP 8.x)
var_dump(5 . 5);          // string(2) "55"   — . concatenates
var_dump("5" <=> 5);      // int(0)    — spaceship: -1/0/1
var_dump(true + true);    // int(2)    — bools coerce to 1/0
// var_dump(5 + "abc");   // TypeError: Unsupported operand types: int + string  (fully non-numeric)
```

Three distinct behaviors in PHP 8, and you should know all three:

- **Clean numeric string** (`"5"`, `" 5 "` with surrounding whitespace) → coerced silently, no warning.
- **Leading-numeric string** (`"5abc"`) → the leading number is used (`5`), and PHP raises a **`Warning: A non-numeric value encountered`**. (In PHP 7 this was a milder `E_NOTICE`; PHP 8.0 promoted it to `E_WARNING`. The historical wording differs by version, so match it against your runtime rather than memorizing it.)
- **Fully non-numeric string** (`"abc"`) → a **`TypeError: Unsupported operand types`** is thrown (this was a non-fatal warning in PHP 7).

The safe move is to validate/cast explicitly rather than relying on any of this coercion.

### 6.4 Explicit casting

```php
<?php
$s = "42px";
var_dump((int) $s);       // int(42)    — parses leading number
var_dump((int) "abc");    // int(0)
var_dump((float) "3.14");  // float(3.14)
var_dump((bool) 0);        // bool(false)
var_dump((string) 3.14);   // string(4) "3.14"
var_dump((array) "x");     // array(1) { [0] => "x" }   — wraps scalar
var_dump((array) (object)['a'=>1]); // array(1) { ["a"] => 1 }
$obj = (object) ['k' => 'v'];        // array → stdClass; $obj->k === 'v'
settype($s, "integer");    // mutates $s ("42px") in place to int(42); returns true
```

Casts: `(int)`/`(integer)`, `(float)`/`(double)`, `(string)`, `(bool)`/`(boolean)`, `(array)`, `(object)`. The `(unset)` cast was removed in PHP 8.0 — don't use it.

For robust user-input parsing, prefer dedicated parsers over casts:

```php
<?php
var_dump(filter_var("42", FILTER_VALIDATE_INT));      // int(42)
var_dump(filter_var("4.2x", FILTER_VALIDATE_FLOAT));  // bool(false) — rejects junk
var_dump(filter_var("yes", FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE)); // bool(true)
```

### 6.5 `declare(strict_types=1)`

By default PHP **coerces** scalar arguments to match a function's typed parameters (so passing `"5"` to `function f(int $x)` works). Add `declare(strict_types=1)` as the **first statement** of a file to forbid this — mismatches throw `TypeError`. This is per-*calling*-file.

```php
<?php
declare(strict_types=1);

function double(int $n): int { return $n * 2; }
echo double(5);     // 10
echo double("5");   // TypeError: must be of type int, string given
```

> Best practice (and Laravel skeleton convention in many files): put `declare(strict_types=1);` at the top of every PHP file you author. It turns silent coercion bugs into loud, early errors.

---

## 7. Inspecting types

```php
<?php
$x = [1, 2];

var_dump($x);        // structured dump WITH types — your #1 debugging tool
//  array(2) { [0]=> int(1) [1]=> int(2) }

echo gettype($x);    // array   (legacy; returns "double" for floats, "integer" for ints — odd names)
echo get_debug_type($x);   // array   (PHP 8.0+, returns canonical names: int, bool, App\User, etc.)

var_dump(is_int(1), is_string("a"), is_array($x), is_callable('strlen'),
         is_iterable($x), is_null(null), is_numeric("3.14"));
// bool(true) bool(true) bool(true) bool(true) bool(true) bool(true) bool(true)
```

Why `get_debug_type()` over `gettype()`: `gettype()` returns historical names (`"double"`, `"integer"`, `"boolean"`) and just `"object"` for any object. `get_debug_type()` returns the modern, programmer-expected names (`"float"`, `"int"`, `"bool"`) and the **fully-qualified class name** for objects — ideal for error messages and type checks.

```php
<?php
var_dump(gettype(3.14));          // string(6) "double"
var_dump(get_debug_type(3.14));   // string(5) "float"
var_dump(get_debug_type(new stdClass)); // string(8) "stdClass"
```

In **Laravel**, the equivalents you'll reach for are `dd($var)` (dump-and-die), `dump($var)` (dump-and-continue), `ray()` (with the Ray app), and `Log::debug(['type' => get_debug_type($x)])`.

---

## 8. ⚠️ Common Mistakes & Gotchas

1. **`empty("0")` is `true`.** The string `"0"` is falsy, so `empty()` reports a field as empty even though the user typed something.
   *Fix:* check explicitly — `if ($value === '' || $value === null)` or use `isset()` plus an emptiness check that matches your intent. In Laravel, prefer validation rules (`'required'` treats `"0"` as present).

2. **Comparing floats with `==`/`===`.** `0.1 + 0.2 === 0.3` is `false`.
   *Fix:* compare with an epsilon (`abs($a - $b) < 1e-9`), or use integers/`bcmath`/`decimal:2` casts for money.

3. **`isset()` returns `false` for a variable set to `null`.** People use `isset($data['key'])` to test "did they send this key?" but a key explicitly set to `null` reads as not-set.
   *Fix:* use `array_key_exists('key', $data)` when you must distinguish "absent" from "present-but-null."

4. **Forgetting to `unset()` the `foreach (... as &$v)` reference.** The reference lingers; the next loop or operation that touches `$v` corrupts the array's last element.
   *Fix:* `unset($v);` immediately after the by-reference loop — or avoid by-reference loops entirely.

5. **Objects/arrays confusion on assignment.** `$b = $a` copies arrays (value) but shares objects (handle). Mutating `$b` mutates `$a` for objects.
   *Fix:* use `clone` for an independent object copy; know that `readonly` properties and DTOs help avoid accidental shared mutation.

6. **Loose-comparison surprises in `switch` and `in_array`.** `switch` uses `==`, and `in_array($needle, $hay)` defaults to loose comparison — so `in_array(0, ['a', 'b'])` was a classic `true` on PHP 7.
   *Fix:* pass the strict flag: `in_array(0, $arr, true)`, or use `match` (which compares with `===`).

7. **Integer overflow silently becoming float.** Large counters or IDs past `PHP_INT_MAX` turn into imprecise floats.
   *Fix:* keep big numeric identifiers as strings, or use GMP/BCMath.

8. **A trailing `?>` or whitespace before `<?php`.** Emits stray output and breaks headers/JSON responses.
   *Fix:* omit the closing `?>` in pure-PHP files.

9. **Comparing secrets/hashes with `==` (type-juggling auth bypass).** Loose comparison treats two `0e…`-form strings as equal numbers, so `"0e1" == "0e9"` is `true` — an attacker can forge a matching hash.
   *Fix:* use `===` for tokens, `hash_equals()` for secret comparison (also constant-time), and `Hash::check()` / `Password` rules in Laravel.

10. **Relying on the exact text of arithmetic coercion warnings.** `5 + "5abc"` parses to `10` but raises a warning whose wording changed between PHP versions (and `5 + "abc"` is now a `TypeError`, not a warning).
    *Fix:* don't depend on the message; validate input with `filter_var()`/`is_numeric()` before doing math on it.

---

## 9. ✅ Best Practices

- Put **`declare(strict_types=1);`** at the top of every file you write; it catches type bugs at the boundary.
- **Type everything**: parameters, return types, and (PHP 7.4+) properties. Use union types (`int|string`), nullable (`?int`), and `mixed` only as a last resort.
- Use **`===`/`!==`** by default; reach for `==` only when you deliberately want coercion (rare). For secrets/tokens/hashes use **`hash_equals()`** (constant-time), never `==`.
- Prefer **enums** over loose `const` flags, and `const` over `define()`.
- Use **`get_debug_type()`** in error messages and logs; reserve `var_dump`/`dd` for debugging, never ship them.
- Never store money as `float`. Use integer cents, `bcmath`, or Laravel's `decimal:2` cast.
- Avoid `global` and variable variables; pass dependencies explicitly. Prefer arrays/DTOs/enums over `$$var` tricks.
- Use **null coalescing (`??`)**, **`??=`**, and the **nullsafe operator (`?->`)** instead of nested `isset()`/`is_null()` ladders.
- Use `array_key_exists()` (not `isset()`) when `null` is a legitimate value you must detect.
- Name with intent: `snake_case` for variables/functions is PHP-historic, but Laravel/PSR favor `camelCase` for variables and methods, `PascalCase` for classes, `SCREAMING_SNAKE_CASE` for constants. Be consistent with the codebase.

---

## 10. 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `==` and `===`?**
`===` is identity — same type and value, no coercion. `==` is loose — it coerces operands to a common type before comparing. Default to `===`. Bonus: mention PHP 8 changed number-vs-non-numeric-string comparison so `0 == "foo"` is now `false` (was `true` in PHP 7).

**Q2. Why does `0.1 + 0.2 == 0.3` return false, and how would you fix it?**
Floats are IEEE-754 binary; most decimal fractions (like 0.1) have no exact binary representation, so rounding error accumulates. Fix by comparing within an epsilon, or by using integers (cents), `bcmath`, or `gmp` for exact decimal arithmetic.

**Q3. Explain `isset()` vs `empty()` vs `is_null()`.**
`isset()` → true if the variable exists and is not null. `empty()` → true if the variable is falsy *or* unset (and notably `empty("0")` is true). `is_null()` → true only if the value is exactly `null` (and warns on undefined). To tell "absent" from "present-but-null" in arrays, use `array_key_exists()`.

**Q4. What are the falsy values in PHP?**
`false`, `0`, `0.0`/`-0.0`, `""`, `"0"`, `[]`, and `null`. Everything else is truthy — including `"0.0"`, `"false"`, `" "`, and `[0]`.

**Q5. How does variable scope work? How do you access an outer variable inside a function?**
PHP has function-level scope; functions don't inherit the enclosing scope. Options: pass as a parameter (best), `global $x` / `$GLOBALS['x']` (avoid), or closures with `use ($x)` (arrow functions capture automatically by value).

**Q6. (Under the hood) How does PHP copy variables — is `$b = $a` expensive for a big array?**
No. PHP uses **copy-on-write**: `$b = $a` shares the same internal `zval`/array buffer and only duplicates it when one side is modified (refcount > 1 and a write occurs). So assignment is O(1) until a mutation forces the split. Objects differ: the variable holds a *handle* to the object, so `$b = $a` shares the same object instance; use `clone` for a copy.

**Q7. What happens on integer overflow in PHP?**
PHP does not wrap; it promotes the result to `float`, which loses integer precision beyond 2^53. `PHP_INT_MAX + 1` becomes a float. For exact big integers, use strings with GMP/BCMath.

**Q8. `const` vs `define()` — when do you use each?**
`const` is compile-time, supports class constants and (8.3+) typed constants, but must be top-level/static. `define()` is a runtime call, allowing conditional or dynamically-named constants. Prefer `const`; use `define()` only when the name/condition is dynamic.

**Q9. What's the difference between a reference and an object handle?**
A reference (`$b = &$a`) makes two *variable names* alias the same value slot — `unset`ing one doesn't destroy the value. An object handle is what every object variable holds; copying the variable copies the handle, so both see the same instance, but they are still independent variable slots (you can rebind one without affecting the other). References are an aliasing mechanism; handles are how objects are passed around.

**Q10. What does `declare(strict_types=1)` do and where does it go?**
It must be the very first statement of a file and disables scalar type *coercion* for calls made *from that file*, turning type mismatches into `TypeError`. It's caller-side, file-scoped, and recommended on every file.

**Q11. (Security) Why is `==` dangerous for comparing password hashes or tokens?**
Loose comparison coerces numeric-looking strings to numbers, so two distinct strings of the form `"0e…"` (which evaluate to `0`) compare as equal — the "magic hash" / type-juggling bypass. Even in PHP 8, `"0e1" == "0e9"` is `true`. Use `===` for tokens and `hash_equals($known, $user)` for secret comparison (constant-time, also defeats timing attacks); in Laravel use `Hash::check()`.

---

## 11. 📋 Quick Reference / Cheat Sheet

```php
// ----- Variables & scope -----
$x = 1;                       // by value; PHP uses copy-on-write internally
$b = &$a;                     // reference (alias)
function f() { global $g; }   // import global (avoid)
function f() { static $n=0; } // persists across calls
$fn = fn($n) => $n + $g;      // arrow fn: auto-captures by value
$fn = function() use ($g){};  // closure: explicit capture (use (&$g) for ref)

// ----- Constants -----
const MAX = 10;               // compile-time, supports class consts
define('K', 'v');             // runtime, conditional/dynamic OK
class C { public const int N = 1; final const M = 2; }  // 8.3 typed, 8.1 final
__LINE__ __FILE__ __DIR__ __FUNCTION__ __CLASS__ __METHOD__ __NAMESPACE__
Foo::class                    // FQCN string, refactor-safe

// ----- Types -----
int  0xFF 0o17 0b101 1_000    PHP_INT_MAX (overflow -> float)
float 1.5e3 INF NAN           never compare with == ; use epsilon
string 'lit' "interp $v" <<<H heredoc H  <<<'N' nowdoc N
bool true false               falsy: false 0 0.0 "" "0" [] null
array [1,2] ['k'=>'v'] [...$a] array_is_list($a)
object new C  (object)[...]    clone for copy; assignment shares handle
null  $x ?? d   $x ??= d   $o?->p   is_null($x)  $x===null
resource fopen()              (many now objects: CurlHandle, GdImage)
callable / iterable           pseudo-types in signatures

// ----- Inspect -----
var_dump($x); gettype($x);    get_debug_type($x)  // prefer this one
is_int is_float is_string is_bool is_array is_object is_null
is_callable is_iterable is_numeric  array_key_exists($k,$a)

// ----- Coercion & casting -----
===  !==  (use these)         ==  != (coercive)   <=>  (spaceship)
hash_equals($a,$b)            // compare secrets/hashes — NEVER use == ("0e.." bypass)
(int)(float)(string)(bool)(array)(object)   settype($x,'int')
filter_var($s, FILTER_VALIDATE_INT|FLOAT|BOOLEAN)
declare(strict_types=1);      // first line; disables scalar coercion
```

```bash
# Inspect your PHP at the CLI
php -v                          # version
php -r 'var_dump(PHP_INT_MAX);' # run a one-liner
php -i | grep -i precision      # float display precision ini setting
```

---

## 12. 🧪 Mini Exercises

1. **Falsy audit.** Write a function `describe(mixed $v): string` that returns `"falsy"` or `"truthy"`, then call it on `0`, `"0"`, `"0.0"`, `[]`, `[0]`, `null`, `" "`, and `"false"`. Predict each result *before* running, then check with `var_dump`.

2. **Money math.** Implement `addPrices(string $a, string $b): string` that adds two decimal price strings exactly (e.g. `"19.99"` + `"0.01"` → `"20.00"`) using `bcadd`. Then write a deliberately buggy float version and demonstrate, with `printf("%.17f")`, where it diverges.

3. **Scope & counters.** Build a `tick()` function using a `static` counter and a second `makeCounter(): callable` that returns a closure capturing its own private count by reference. Show that two counters from `makeCounter()` are independent but the `static` one is global to the function.

4. **Type reporter.** Write `report(mixed ...$vals): void` that prints, for each value, both `gettype()` and `get_debug_type()` side by side. Feed it an int, float, string, bool, array, `stdClass`, a closure, `null`, and an open file handle, and note every place the two functions disagree.

5. **Safe input.** Given `$_GET`-style input `['age' => '0', 'name' => '', 'admin' => 'false']`, write validation that correctly treats `"0"` as a provided age, rejects the empty name, and parses `"false"` as boolean `false` — using `array_key_exists`, `filter_var`, and explicit checks rather than `empty()`.
