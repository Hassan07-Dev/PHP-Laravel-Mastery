# PHP Interview Questions & Answers — A Categorized Bank (50+)

> Target: **PHP 8.4** (with notes on 8.1–8.3 where relevant). This is the language-fundamentals bank; framework questions live in the Laravel modules.

**What you'll learn**

- The precise, defensible answers to the PHP questions interviewers actually ask
- How loose vs strict comparison, `isset`/`empty`/`is_null`, and type juggling really behave
- How PHP manages memory: **copy-on-write**, **reference counting**, and the **cycle collector**
- The OOP distinctions that trip people up: `self` vs `static`, abstract vs interface, traits, late static binding
- Error-handling, security (SQLi/XSS/CSRF, `password_hash`), and Composer/PSR essentials
- What changed in PHP 8: JIT, enums, `match`, named args, `readonly`, nullsafe, fibers
- Several "explain it under the hood" answers you can deliver with confidence

---

## How to use this bank

Each entry is a **question heading** followed by a **bold one-line answer**, then a short explanation and (where useful) runnable code with expected output. Read the bold line first; if you can expand it into the paragraph below, you're interview-ready. The "under the hood" entries are flagged — those separate junior from senior candidates.

---

## 1. Basics & Types

### Q1. What is the difference between `==` and `===`?

**`==` compares values after type juggling (loose); `===` compares value *and* type (strict) with no coercion.**

```php
<?php
var_dump(0 == "a");      // PHP 8+: false  (was true before PHP 8!)
var_dump("1" == "01");   // true  — both look numeric, compared as numbers
var_dump("10" == "1e1"); // true  — numeric strings, 10 == 10
var_dump(100 == "1e2");  // true
var_dump("1" === 1);     // false — different types
var_dump(null == false); // true
var_dump(null === false);// false
```

The single biggest change in PHP 8: when comparing a number to a **non-numeric string**, PHP now casts the *number* to a string instead of casting the string to `0`. So `0 == "a"` is `false` in PHP 8 (it was `true` in PHP 7). Always prefer `===` unless you have a specific reason.

### Q2. Explain `isset()` vs `empty()` vs `is_null()`.

**`isset()` is true when a variable exists and is not `null`; `empty()` is true when a value is "falsy" (and never errors on undefined); `is_null()` is true only when the value is exactly `null`.**

```php
<?php
$a = 0;
$b = "";
$c = null;
// $d undefined

var_dump(isset($a));    // true
var_dump(isset($c));    // false — null counts as "not set"
var_dump(isset($d));    // false — no notice
var_dump(empty($a));    // true  — 0 is falsy
var_dump(empty($b));    // true  — "" is falsy
var_dump(empty($d));    // true  — no notice
var_dump(is_null($c));  // true
// is_null($d) would raise a "Undefined variable" warning
```

Falsy values for `empty()`: `false`, `0`, `0.0`, `""`, `"0"`, `null`, `[]`, and unset. Note the surprise: `empty("0")` is `true`. Use `isset()` to test existence; use `array_key_exists()` when a key may legitimately hold `null` (`isset()` returns `false` for a key whose value is `null`, but `array_key_exists()` returns `true`).

### Q3. Pass by value vs pass by reference — how does PHP pass arguments?

**By default PHP passes by value (a logical copy via copy-on-write); prefix the parameter with `&` to pass by reference so the function can mutate the caller's variable.**

```php
<?php
function addByValue(array $a): void { $a[] = 99; }
function addByRef(array &$a): void  { $a[] = 99; }

$x = [1, 2];
addByValue($x);
print_r($x);   // [1, 2]      — unchanged
addByRef($x);
print_r($x);   // [1, 2, 99]  — mutated
```

Objects are special: the *variable* still holds a value, but that value is a **handle** (identifier) pointing to the object instance. So passing an object by value still lets you mutate the same underlying object — you just can't reassign the caller's variable to a *different* object without `&`.

```php
<?php
class Box { public int $n = 0; }
function bump(Box $b): void { $b->n++; $b = new Box(); } // reassignment is local
$box = new Box();
bump($box);
echo $box->n; // 1 — the property change stuck; the reassignment did not
```

### Q4. What is type juggling? Give surprising examples.

**Type juggling is PHP's automatic conversion of a value's type based on context (arithmetic, comparison, string operations).**

```php
<?php
var_dump("5 apples" + 3);   // int(8) + E_WARNING "A non-numeric value encountered"
var_dump("5" + 3);          // int(8)  — fully numeric string, no warning
var_dump("5" . 3);          // string(2) "53"
var_dump(true + true);      // int(2)
var_dump((int) "12abc");    // int(12) — leading-numeric cast
var_dump(0.1 + 0.2 == 0.3); // false   — IEEE-754 floating point
```

Two leading-numeric vs non-numeric rules matter in PHP 8: a *leading-numeric* string like `"5 apples"` in arithmetic yields the number plus an `E_WARNING`; a fully non-numeric string like `"apples"` throws a `TypeError`. For money never use floats — use integers (cents) or a decimal library.

### Q5. What are the eight PHP types, and which are scalar?

**Four scalars: `bool`, `int`, `float`, `string`. Two compound: `array`, `object`. Two special: `resource`, `null`. (Plus `callable` and `iterable` as pseudo-types.)**

PHP 8 also adds `union types` (`int|string`), `intersection types` (`Countable&Traversable`, 8.1+), `never` (8.1), `mixed` (8.0), and `true`/`false`/`null` as standalone types (8.2+). `int` width depends on the platform (`PHP_INT_MAX`); overflow promotes silently to `float`.

### Q6. What does the spaceship operator `<=>` return?

**It returns `-1`, `0`, or `1` for less-than, equal, greater-than — ideal for comparison callbacks.**

```php
<?php
$nums = [3, 1, 2];
usort($nums, fn($a, $b) => $a <=> $b);
print_r($nums); // [1, 2, 3]
```

### Q7. Difference between `null coalescing` `??` and the ternary `?:`?

**`??` returns the right side only when the left is `null` or unset (no notice); `?:` returns the right side when the left is *falsy*.**

```php
<?php
$config = ['rows' => 0];
echo $config['rows'] ?? 10;  // 0  — key exists and is not null
echo $config['rows'] ?: 10;  // 10 — 0 is falsy
echo $config['missing'] ?? 10; // 10 — no warning
```

`??=` is the null-coalescing assignment: `$x ??= 'default';` assigns only if `$x` is null/unset.

---

## 2. Strings & Arrays

### Q8. Name the array functions every PHP dev should know.

**Map/filter/reduce: `array_map`, `array_filter`, `array_reduce`. Keys/values: `array_keys`, `array_values`, `array_column`. Merge/combine: `array_merge`, `array_combine`, `array_diff`, `array_intersect`. Search: `in_array`, `array_search`, `array_key_exists`.**

```php
<?php
$users = [
    ['id' => 1, 'name' => 'Ada'],
    ['id' => 2, 'name' => 'Linus'],
];
print_r(array_column($users, 'name', 'id'));
// [1 => 'Ada', 2 => 'Linus']

$sum = array_reduce([1, 2, 3, 4], fn($carry, $n) => $carry + $n, 0);
echo $sum; // 10
```

Gotcha: `array_merge` **renumbers** integer keys but preserves string keys; the `+` union operator keeps the *left* array's keys and ignores duplicates from the right.

```php
<?php
print_r([0 => 'a'] + [0 => 'b', 1 => 'c']); // ['a', 'c'] — left wins on key 0
print_r(array_merge(['a'], ['b']));          // ['a', 'b'] — reindexed
```

### Q9. How do the sort functions differ?

**`sort`/`rsort` reindex and sort by value; `asort`/`arsort` keep keys and sort by value; `ksort`/`krsort` sort by key; `usort`/`uasort`/`uksort` use a comparator.**

```php
<?php
$fruit = ['banana' => 3, 'apple' => 1, 'cherry' => 2];
asort($fruit);
print_r($fruit); // ['apple'=>1, 'cherry'=>2, 'banana'=>3] — keys kept
ksort($fruit);
print_r($fruit); // ['apple'=>1, 'banana'=>3, 'cherry'=>2]
```

PHP's sort is **not stable** before PHP 8.0; **as of PHP 8.0 all sorts are stable** (equal elements keep their original order). Worth mentioning in interviews.

### Q10. When does a PHP array get copied? (copy-on-write)

**Assigning or passing an array does *not* copy it immediately; PHP shares the same internal buffer (with a refcount) and only performs a deep copy the moment one side writes to it — "copy-on-write" (COW).**

```php
<?php
$a = range(1, 1_000_000); // big array
$b = $a;                   // O(1): no copy yet, refcount = 2
$b[0] = 'x';               // NOW the array is duplicated (separation)
```

This is why "PHP arrays are passed by value" is cheap in practice — the copy is deferred until mutation. A `&` reference disables COW because both names must see the same buffer.

### Q11. What is the difference between an indexed, associative, and multidimensional array?

**Indexed arrays use integer keys (often `0..n`), associative arrays use string keys, multidimensional arrays nest arrays as values — but internally PHP has only one type: an ordered hash map.**

Every PHP array is an *ordered map*. Even `[0,1,2]` is a hashmap that happens to have integer keys; insertion order is preserved, which is why JSON encoding can differ from what you expect when keys are non-sequential.

### Q12. How do you handle multibyte (UTF-8) strings safely?

**Use the `mb_*` functions (`mb_strlen`, `mb_substr`, `mb_strtoupper`) with an explicit encoding; `strlen` counts *bytes*, not characters.**

```php
<?php
$s = "café";
echo strlen($s);            // 5 — é is 2 bytes in UTF-8
echo mb_strlen($s, 'UTF-8');// 4 — characters
```

### Q13. How does string interpolation work, and when do you need braces?

**Double-quoted strings and heredocs interpolate `$var`; use `{$expr}` for array/object access or complex expressions. Single quotes and nowdocs do not interpolate.**

```php
<?php
$user = ['name' => 'Ada'];
echo "Hi {$user['name']}";   // Hi Ada
echo 'Hi $user';             // Hi $user (literal)
```

---

## 3. Functions & Closures

### Q14. What is a closure, and how does it capture variables?

**A closure is an anonymous function object (instance of `Closure`) that explicitly captures outer variables via the `use` clause — by value by default, or by reference with `&`.**

```php
<?php
$counter = 0;
$byValue = function () use ($counter) { return $counter; };
$byRef   = function () use (&$counter) { return $counter; };
$counter = 5;
echo $byValue(); // 0 — captured the value at definition time
echo $byRef();   // 5 — captured the variable itself
```

### Q15. Closures vs arrow functions — what's the difference?

**Arrow functions (`fn() => ...`) auto-capture outer variables *by value* (no `use`), are single-expression only, and cannot capture by reference; closures (`function() {}`) need explicit `use` but support multi-statement bodies and `use (&$x)`.**

```php
<?php
$tax = 0.2;
$withTax = fn(float $p) => $p * (1 + $tax); // $tax captured automatically
echo $withTax(100); // 120
```

Arrow functions cannot mutate the outer scope. If you need reference capture or multiple statements, use a full closure.

### Q16. What are variadics and the spread operator?

**`...$args` in a signature collects extra arguments into an array (variadic); `...$array` at a call site spreads an array into individual arguments (spread/unpacking).**

```php
<?php
function sum(int ...$nums): int { return array_sum($nums); }
echo sum(1, 2, 3);          // 6
$parts = [4, 5, 6];
echo sum(...$parts);        // 15
```

Since PHP 8.1 you can also spread **string-keyed** arrays into named arguments.

### Q17. What are named arguments and why use them?

**Named arguments let you pass parameters by name regardless of order, so you can skip optional ones and make calls self-documenting.**

```php
<?php
function makeCoffee(string $size = 'medium', bool $oat = false, int $shots = 1) {
    return "$size, oat=$oat, shots=$shots";
}
echo makeCoffee(shots: 2, oat: true);
// "medium, oat=1, shots=2"
```

Pair with `match`/enums for clean, readable APIs. Caveat: named args couple callers to *parameter names*, so renaming a parameter is now a breaking change.

### Q18. What is `static` in a function and what is a first-class callable?

**A `static` local variable retains its value across calls; the first-class callable syntax `strlen(...)` (PHP 8.1+) creates a `Closure` from any callable.**

```php
<?php
function tick(): int { static $n = 0; return ++$n; }
echo tick(), tick(), tick(); // 123

$len = strlen(...);          // Closure
echo $len("hello");          // 5
```

---

## 4. Object-Oriented PHP

### Q19. Abstract class vs interface — when do you use which?

**An abstract class can hold state, constructors, and concrete methods and is single-inheritance; an interface is a pure contract (signatures + constants, no state) that a class can implement many of. Use an interface for "can-do" capability, an abstract class for shared partial implementation.**

```php
<?php
interface Payable { public function pay(int $cents): void; }

abstract class Employee implements Payable {
    public function __construct(protected string $name) {}
    abstract protected function rate(): int;       // subclass must define
    public function describe(): string { return $this->name; } // shared
}
```

Interfaces have grown richer over time — they can declare **typed constants** (PHP 8.3) and require **property hooks** / `public/protected(set)` property declarations (PHP 8.4) — but the rule of thumb stays: interface = contract, abstract = partial base.

### Q20. `self` vs `static` vs `parent` — and what is late static binding?

**`self::` resolves to the class where the code is *written*; `static::` resolves to the class that was *called at runtime* (late static binding); `parent::` calls the parent's version.**

```php
<?php
class Base {
    public static function create(): static { return new static(); }
    public static function whoSelf(): string  { return self::class; }
    public static function whoStatic(): string { return static::class; }
}
class Child extends Base {}

echo get_class(Child::create()); // Child  — `new static()`
echo Child::whoSelf();           // Base   — bound at compile time
echo Child::whoStatic();         // Child  — bound at call time (LSB)
```

**Under the hood:** `self` is bound at compile time to the lexical class; `static` is resolved at runtime from the class used in the call, which PHP tracks as the "called class." This is why factory methods return `new static()`.

### Q21. What are traits, and how are conflicts resolved?

**Traits are reusable method/property bundles copied into a class at compile time to work around single inheritance; conflicts between two traits are resolved with `insteadof` and `as`.**

```php
<?php
trait Logs   { public function hello() { return "log"; } }
trait Audits { public function hello() { return "audit"; } }

class Service {
    use Logs, Audits {
        Logs::hello insteadof Audits;   // pick Logs' version
        Audits::hello as auditHello;    // alias the other
    }
}
$s = new Service();
echo $s->hello();      // log
echo $s->auditHello(); // audit
```

Traits are *horizontal reuse*: think "copy-paste at compile time," not inheritance. They can declare abstract methods (forcing the using class to implement them) and `static` members.

### Q22. Explain the common magic methods.

**Methods PHP calls automatically: `__construct`/`__destruct`, `__get`/`__set`/`__isset`/`__unset` (overloading inaccessible properties), `__call`/`__callStatic` (inaccessible methods), `__toString`, `__invoke` (callable objects), `__clone`, `__sleep`/`__wakeup` and `__serialize`/`__unserialize`.**

```php
<?php
class Bag {
    private array $data = [];
    public function __get($k) { return $this->data[$k] ?? null; }
    public function __set($k, $v) { $this->data[$k] = $v; }
    public function __isset($k) { return isset($this->data[$k]); }
    public function __toString(): string { return json_encode($this->data); }
}
$b = new Bag();
$b->color = 'red';          // __set
echo $b->color;             // red — __get
echo isset($b->color) ? 'y':'n'; // y — __isset
echo $b;                    // {"color":"red"} — __toString
```

Magic methods are convenient but slower and hide intent; prefer explicit properties/methods unless building a flexible container or proxy.

### Q23. What do `public`, `protected`, and `private` mean?

**`public` = accessible anywhere; `protected` = within the class and its subclasses; `private` = only within the declaring class itself.**

Note: a `private` member is private to the *class that declares it*, not the object — two instances of the same class can read each other's private members. PHP 8.4 adds **asymmetric visibility** (`public private(set)`) so a property can be read publicly but written only inside the class.

```php
<?php
class Account {
    public private(set) int $balance = 0; // read anywhere, write only inside
    public function deposit(int $n): void { $this->balance += $n; }
}
```

### Q24. What does `final` do?

**`final` on a method prevents overriding; on a class prevents extending. Final constants (8.1+) can't be overridden by child constants.**

Use `final` to make inheritance intentional and to lock down value objects. It signals "this is not an extension point" and can enable optimizations.

### Q25. What is constructor property promotion?

**A shorthand (PHP 8.0+) that declares and assigns properties directly in the constructor signature, eliminating boilerplate.**

```php
<?php
final class Money {
    public function __construct(
        public readonly int $amount,
        public readonly string $currency = 'USD',
    ) {}
}
$m = new Money(500);
echo $m->amount;   // 500
// $m->amount = 1; // Error: Cannot modify readonly property
```

### Q26. `readonly` properties and classes — what do they guarantee?

**A `readonly` property can be initialized once (typically in the constructor) and never reassigned; a `readonly class` (8.2+) makes *all* its instance properties readonly automatically.**

This gives you immutable value objects. To "change" one, `clone` it and apply the new value inside `__clone()` (or expose a `with*()` method that clones and reassigns). Note: a plain `clone $obj` *can* reinitialize `readonly` properties from within `__clone()` (allowed since PHP 8.3); the `clone($obj, [...])` syntax that overrides properties at the clone site is a **PHP 8.5** feature, not 8.4.

### Q27. How does object cloning work and what is `__clone`?

**`clone $obj` makes a shallow copy; nested objects remain shared references unless you deep-copy them inside `__clone`.**

```php
<?php
class Profile { public array $tags = []; }
class User {
    public Profile $profile;
    public function __construct() { $this->profile = new Profile(); }
    public function __clone() { $this->profile = clone $this->profile; } // deep copy
}
```

Without the `__clone` method, both the original and the clone would point at the *same* `Profile` object.

### Q28. What is dependency injection and why prefer it?

**DI means a class receives its collaborators from outside (usually via the constructor) instead of creating them itself, which decouples code and makes it testable.**

```php
<?php
interface Mailer { public function send(string $to, string $body): void; }

final class Welcome {
    public function __construct(private Mailer $mailer) {} // injected
    public function greet(string $email): void {
        $this->mailer->send($email, 'Welcome!');
    }
}
```

In tests you pass a fake `Mailer`; in production a real one. Frameworks like Laravel wire this automatically through a **service container**.

---

## 5. Error & Exception Handling

### Q29. What is the `Throwable` hierarchy? `Error` vs `Exception`.

**`Throwable` is the root interface. Two branches implement it: `Error` (engine-level problems like `TypeError`, `ParseError`, `DivisionByZeroError`) and `Exception` (application-level, e.g. `RuntimeException`, `InvalidArgumentException`). Catch `Throwable` to catch both.**

```php
<?php
try {
    intdiv(1, 0);
} catch (\DivisionByZeroError $e) {     // an Error, not an Exception
    echo "math: " . $e->getMessage();
} catch (\Throwable $e) {
    echo "anything: " . $e->getMessage();
}
// Output: math: Division by zero
```

Before PHP 7, fatal errors couldn't be caught. Now most are `Error` objects you *can* catch — though catching `Error` usually means a bug to fix, not recover from.

### Q30. How does `try / catch / finally` behave, including with `return`?

**`finally` always runs — even after a `return`, `throw`, or `break` in the try/catch. A `return` (or value) inside `finally` overrides any earlier return.**

```php
<?php
function demo(): string {
    try {
        return 'try';
    } finally {
        echo "[finally] ";
    }
}
echo demo(); // [finally] try

function override(): string {
    try { return 'A'; }
    finally { return 'B'; } // overrides
}
echo override(); // B  (avoid this — confusing)
```

### Q31. How do you catch multiple exception types, and what is exception chaining?

**Use the union catch `catch (TypeA | TypeB $e)`; chain via the third constructor argument `previous` to preserve the original cause.**

```php
<?php
try {
    try {
        throw new RuntimeException('low-level');
    } catch (RuntimeException $e) {
        throw new LogicException('high-level', 0, $e); // chain
    }
} catch (LogicException $e) {
    echo $e->getMessage();              // high-level
    echo $e->getPrevious()->getMessage(); // low-level
}
```

### Q32. What is a custom exception and when do you make one?

**A class extending `Exception` (or a specific subclass) to represent a domain-specific error you can catch precisely.**

```php
<?php
final class InsufficientFundsException extends \RuntimeException {}
// catch (InsufficientFundsException $e) { ... } — targeted handling
```

### Q33. What's the difference between an error, a warning, and a notice?

**They are `E_*` severity levels: `E_ERROR` halts execution; `E_WARNING` is non-fatal; `E_NOTICE`/`E_DEPRECATED` are informational. PHP 8 raised the severity of several conditions — e.g. accessing an undefined array key went from `E_NOTICE` to `E_WARNING`, and using an undefined variable is now an `E_WARNING`.**

Control reporting with `error_reporting(E_ALL)` and `display_errors`. In production, log errors and never display them to users.

### Q34. What is `set_error_handler` vs `set_exception_handler`?

**`set_error_handler` intercepts traditional PHP errors/warnings (you can convert them to `ErrorException`); `set_exception_handler` is the last-resort handler for uncaught exceptions.**

```php
<?php
set_error_handler(function ($severity, $msg, $file, $line) {
    throw new \ErrorException($msg, 0, $severity, $file, $line);
});
```

---

## 6. Namespaces & Autoloading (PSR-4)

### Q35. What problem do namespaces solve, and how do `use`/aliasing work?

**Namespaces prevent name collisions between classes/functions/constants from different libraries; `use` imports a fully-qualified name into the current file, optionally aliased with `as`.**

```php
<?php
namespace App\Services;

use App\Models\User;
use App\Models\Order as Purchase; // alias to avoid clash
use function App\Helpers\format;
```

A leading `\` means "fully qualified from the global root" (`\strlen`, `\DateTime`). Inside a namespace, unqualified function calls fall back to the global function if not found locally — but unqualified *class* names do not.

### Q36. What is PSR-4 autoloading and how does Composer implement it?

**PSR-4 is a standard mapping a namespace prefix to a base directory so each class lives in a predictably named file; Composer generates an autoloader that lazily `require`s the right file the first time a class is used.**

```json
{
  "autoload": {
    "psr-4": { "App\\": "src/" }
  }
}
```

With this, `App\Services\Mailer` must live in `src/Services/Mailer.php`. After editing `composer.json` run `composer dump-autoload`. **Under the hood:** Composer registers a callback with `spl_autoload_register`; when PHP hits an unknown class, it invokes registered autoloaders in order, the PSR-4 one converts the namespace to a path and `require`s it. Use `composer dump-autoload --optimize` (a classmap) in production to skip filesystem lookups.

### Q37. Difference between `require`, `include`, `require_once`, `include_once`?

**`require` fatals if the file is missing; `include` only warns; the `_once` variants ensure the file is loaded at most once.**

In modern code you rarely call these directly — Composer's autoloader handles class loading. Use them for config files or bootstrap.

---

## 7. Memory & Performance (Under the Hood)

### Q38. How does PHP manage memory / garbage collection? (under the hood)

**PHP uses reference counting as the primary mechanism: every value (`zval`) tracks a `refcount`; when it drops to zero the memory is freed immediately. A separate cycle collector handles reference cycles that refcounting alone can't reclaim.**

```php
<?php
$a = "hello"; // refcount of the string = 1
$b = $a;      // refcount = 2 (still shared, COW)
unset($a);    // refcount = 1
unset($b);    // refcount = 0 -> freed
```

The problem: two objects referencing each other keep each other's refcount above zero even when nothing else points to them — a *cycle*. PHP's **cycle collector** (the "garbage collector," `gc_collect_cycles()`) periodically scans the *possible roots* buffer to find and free such cycles. It runs automatically when the root buffer fills (default 10,000 roots) or on demand. Most short-lived web requests finish before GC ever needs to run, because all memory is released when the request ends.

### Q39. What is copy-on-write at the engine level?

**A value's `zval` is shared (refcount > 1) until someone writes; the write triggers "separation" — duplicating the value so the writer gets its own copy without affecting other holders.**

This is why returning or assigning big arrays is cheap until mutation. References (`&`) opt out: a referenced `zval` is marked `is_ref` and writes go straight through, shared by all aliases.

### Q40. What are generators and why use them?

**A generator is a function containing `yield` that produces values lazily one at a time, so you can iterate huge or infinite sequences with near-constant memory instead of building a full array.**

```php
<?php
function readLines(string $file): Generator {
    $fh = fopen($file, 'r');
    while (($line = fgets($fh)) !== false) {
        yield rtrim($line);
    }
    fclose($fh);
}
foreach (readLines('huge.log') as $line) {
    // processes one line at a time; file never fully loaded into memory
}
```

Generators implement `Iterator`. `yield $key => $value` yields keys; `yield from` delegates to another iterable. They can also receive values via `->send()`, making them coroutine-like.

### Q41. When should you use references, and why are they often discouraged?

**Use `&` for in-place mutation of large structures or to share a single value across names; avoid them generally because they disable copy-on-write, make data flow hard to follow, and cause subtle aliasing bugs.**

A classic foot-gun: a leftover reference from a `foreach ($arr as &$v)` loop.

```php
<?php
$arr = [1, 2, 3];
foreach ($arr as &$v) {}   // $v still references the last element!
foreach ($arr as $v) {}    // this overwrites $arr[2] with each value
print_r($arr);             // [1, 2, 2] — the bug
// Fix: unset($v); after the first loop.
```

### Q42. What is OPcache and the JIT? (under the hood)

**OPcache stores compiled bytecode (opcodes) in shared memory so PHP skips re-parsing/compiling files on every request; the JIT (PHP 8.0+) compiles hot opcodes to native machine code at runtime for CPU-bound work.**

OPcache is the single biggest production performance win and should always be on. JIT helps math/CPU-heavy code (image processing, simulations) but gives little benefit to typical I/O-bound web apps. Configure JIT via `opcache.jit` and `opcache.jit_buffer_size` in `php.ini`.

---

## 8. Configuration, SAPIs, Sessions & Cookies

### Q43. What is `php.ini` and what are commonly tuned directives?

**`php.ini` is PHP's main configuration file controlling runtime behavior; common directives include `memory_limit`, `max_execution_time`, `upload_max_filesize`, `post_max_size`, `display_errors`, `error_reporting`, `date.timezone`, and the `opcache.*` settings.**

```ini
memory_limit = 256M
max_execution_time = 30
display_errors = Off          ; production
error_reporting = E_ALL
opcache.enable = 1
```

Find the active file with `php --ini` or `phpinfo()`. Some values can be set per-request with `ini_set()`; others are `PHP_INI_SYSTEM` (only changeable in the ini file).

### Q44. What is a SAPI? Name a few.

**SAPI = Server API: the interface between PHP and its host environment. Examples: `cli` (command line), `fpm-fcgi` (PHP-FPM behind Nginx), `apache2handler` (mod_php), `cgi-fcgi`, and embedded.**

The same script behaves differently per SAPI — e.g. `max_execution_time` is ignored on CLI by default, and headers/sessions behave differently. Check with `php_sapi_name()` or the `PHP_SAPI` constant.

### Q45. Sessions vs cookies — what's the difference?

**A cookie is a small piece of data stored *in the browser* and sent with each request; a session stores data *on the server*, identified by a session ID that is itself typically carried in a cookie.**

```php
<?php
// Cookie: visible/editable by the client
setcookie('theme', 'dark', time() + 86400, '/', '', true, true); // Secure, HttpOnly

// Session: data lives server-side, only the ID travels
session_start();
$_SESSION['user_id'] = 42;
```

Use cookies for small, non-sensitive client state; use sessions for anything sensitive (auth state). Protect cookies with `Secure`, `HttpOnly`, and `SameSite` flags. Regenerate the session ID on login (`session_regenerate_id(true)`) to prevent session fixation.

### Q46. How does PHP's request lifecycle work for a typical web request? (under the hood)

**For each request a SAPI like PHP-FPM hands off to the PHP engine: it loads `php.ini`, checks OPcache for compiled bytecode (compiling on miss), executes the script, sends output, then tears down all per-request memory. PHP is "shared-nothing" — no state persists between requests except what you explicitly store (DB, cache, session).**

This shared-nothing model is why PHP scales horizontally so easily and why memory leaks rarely matter for short requests.

---

## 9. Security

### Q47. How do you prevent SQL injection?

**Never concatenate user input into SQL; use parameterized queries (prepared statements) via PDO or mysqli so values are sent separately from the query text.**

```php
<?php
$pdo = new PDO('mysql:host=localhost;dbname=app;charset=utf8mb4', $user, $pass, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false, // real server-side prepares
]);
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $userInput]);   // safe — input never parsed as SQL
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

Set `PDO::ATTR_EMULATE_PREPARES => false` so binding happens on the database server. Identifiers (table/column names) can't be parameterized — whitelist them.

### Q48. How do you prevent XSS?

**Escape all dynamic output for its context: HTML-encode with `htmlspecialchars($v, ENT_QUOTES, 'UTF-8')`, and escape differently for HTML attributes, JS, and URLs. Treat all user input as untrusted.**

```php
<?php
echo htmlspecialchars($comment, ENT_QUOTES, 'UTF-8');
```

Output encoding (not input filtering) is the real defense. In Blade, `{{ $x }}` auto-escapes; `{!! $x !!}` does not (dangerous). Add a `Content-Security-Policy` header for defense in depth.

### Q49. What is CSRF and how do you stop it?

**CSRF (Cross-Site Request Forgery) tricks a logged-in user's browser into submitting an unwanted authenticated request. Defend with an unpredictable per-session CSRF token validated on state-changing requests, plus `SameSite` cookies.**

```php
<?php
// Generate once per session
$_SESSION['csrf'] ??= bin2hex(random_bytes(32));
// Embed in forms, then verify on POST:
if (!hash_equals($_SESSION['csrf'], $_POST['csrf'] ?? '')) {
    http_response_code(419);
    exit('CSRF token mismatch');
}
```

Use `hash_equals()` for constant-time comparison (prevents timing attacks). Laravel handles this automatically with `@csrf` and its CSRF middleware (`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` — renamed from `VerifyCsrfToken` in Laravel 11; in Laravel 11/12 you tweak its exceptions fluently in `bootstrap/app.php` rather than editing a `Kernel.php`).

### Q50. How do you hash passwords correctly?

**Use `password_hash()` (which defaults to bcrypt, or Argon2id if requested) and verify with `password_verify()`. Never use `md5`/`sha1`, never store plaintext, and re-hash when the algorithm/cost is outdated via `password_needs_rehash()`.**

```php
<?php
$hash = password_hash($plain, PASSWORD_DEFAULT);     // bcrypt by default
// store $hash...

if (password_verify($plain, $hash)) {
    if (password_needs_rehash($hash, PASSWORD_DEFAULT)) {
        $hash = password_hash($plain, PASSWORD_DEFAULT); // upgrade
    }
    // login OK
}
```

`password_hash` salts automatically and the salt is embedded in the output string — you never manage salts yourself. Use `PASSWORD_ARGON2ID` for the strongest modern option if the build supports it.

### Q51. How do you generate cryptographically secure random values?

**Use `random_bytes()` and `random_int()` (CSPRNG-backed); never use `rand()`/`mt_rand()` for tokens, passwords, or anything security-sensitive.**

```php
<?php
$token = bin2hex(random_bytes(32)); // 64-char hex token
$otp   = random_int(100000, 999999);
```

### Q52. What is mass assignment / over-posting and other common web risks?

**Mass assignment is blindly trusting all request fields when creating/updating a record (e.g. a user setting `is_admin=1`). Defend by whitelisting fields. Other staples: insecure file uploads (validate type/size, store outside webroot), path traversal (canonicalize paths), and leaking errors (disable `display_errors` in production).**

---

## 10. Composer & PSR

### Q53. What is Composer and what's the difference between `require` and `require --dev`?

**Composer is PHP's dependency manager; `composer require` adds a runtime dependency to `require`, while `composer require --dev` adds a development-only dependency (tests, static analysis) to `require-dev`, which isn't installed in production with `--no-dev`.**

```bash
composer require guzzlehttp/guzzle
composer require --dev phpunit/phpunit
composer install --no-dev --optimize-autoloader  # production
```

### Q54. `composer.json` vs `composer.lock` — what's the difference?

**`composer.json` declares your version *constraints* (what's allowed); `composer.lock` records the *exact resolved versions* installed, so every environment gets identical dependencies. Commit the lock file for applications; libraries usually don't.**

`composer install` installs from the lock; `composer update` re-resolves constraints and rewrites the lock.

### Q55. Explain semantic versioning constraints (`^`, `~`).

**SemVer is `MAJOR.MINOR.PATCH`. `^1.2.3` allows changes that don't change the leftmost non-zero digit (up to `<2.0.0`); `~1.2.3` allows only the last specified digit to move (up to `<1.3.0`).**

```json
{ "require": { "monolog/monolog": "^3.0", "vendor/lib": "~1.4.2" } }
```

`^` is the common default. Be cautious with `0.x` versions — under SemVer, minor bumps there may break.

### Q56. What are PSRs? Name the important ones.

**PSR = PHP Standards Recommendations from PHP-FIG. Key ones: PSR-1/PSR-12 (coding style), PSR-4 (autoloading), PSR-3 (logging interface), PSR-7 (HTTP messages), PSR-11 (container interface), PSR-15 (HTTP middleware), PSR-6/PSR-16 (caching).**

Implementing PSR interfaces lets libraries interoperate — e.g. any PSR-3 logger can be swapped into any code that type-hints `Psr\Log\LoggerInterface`.

---

## 11. What's New in PHP 8 (and 8.1–8.4)

### Q57. What is the JIT compiler?

**Just-In-Time compilation (PHP 8.0) translates hot opcodes into native machine code at runtime, benefiting CPU-bound workloads; it has little effect on typical I/O-bound web apps.**

### Q58. What are enums and how do backed enums differ from pure enums?

**Enums (PHP 8.1) are a type with a fixed set of named instances. *Pure* enums have only cases; *backed* enums assign each case a scalar (`int`/`string`) value and gain `from()`/`tryFrom()`.**

```php
<?php
enum Status: string {
    case Active   = 'active';
    case Archived = 'archived';

    public function label(): string {
        return match ($this) {
            Status::Active   => 'Active',
            Status::Archived => 'Archived',
        };
    }
}
echo Status::Active->value;           // active
echo Status::from('archived')->name;  // Archived
var_dump(Status::tryFrom('nope'));    // NULL (from() would throw)
```

Enums can implement interfaces and hold methods/constants, but not instance state. They're singletons — `Status::Active === Status::Active` is always true.

### Q59. What does the `match` expression do that `switch` doesn't?

**`match` returns a value, uses strict (`===`) comparison, requires no `break`, has no fall-through, and throws `UnhandledMatchError` when nothing matches (no silent default).**

```php
<?php
$code = 404;
$text = match ($code) {
    200, 201 => 'OK',
    404      => 'Not Found',
    default  => 'Unknown',
};
echo $text; // Not Found
```

`switch` uses loose comparison and falls through without `break` — a classic bug source.

### Q60. What is the nullsafe operator `?->`?

**`?->` short-circuits a method/property chain to `null` if the left side is `null`, instead of erroring — avoiding nested null checks.**

```php
<?php
$country = $user?->getAddress()?->country; // null if user or address is null
```

### Q61. What are fibers?

**Fibers (PHP 8.1) are full-stack, interruptible functions that let you pause and resume execution, enabling cooperative multitasking and powering async runtimes (e.g. ReactPHP/Amp). They are low-level primitives, not async-by-default.**

```php
<?php
$fiber = new Fiber(function (): void {
    $value = Fiber::suspend('paused'); // hands control back to caller
    echo "resumed with $value";
});
echo $fiber->start();    // paused
$fiber->resume('done');  // resumed with done
```

### Q62. Quick-fire: other notable 8.x features?

**`readonly` properties (8.1) and classes (8.2); pure intersection types (8.1); `never` return type (8.1); first-class callable syntax `foo(...)` (8.1); `new` in initializers (8.1); enums (8.1); `#[Attributes]` (8.0); DNF (disjunctive normal form) types (8.2); `#[\Override]`, typed class constants, and `json_validate()` (8.3); property hooks and asymmetric visibility (8.4); chaining directly off `new` without the extra wrapping parentheses — `new Foo()->bar()` instead of `(new Foo())->bar()` (8.4); implicitly-nullable parameters deprecated — write `?int $x = null` instead of `int $x = null` (8.4).**

```php
<?php
// PHP 8.4 property hooks — computed/validated properties without boilerplate
class Temperature {
    public float $celsius = 0.0;
    public float $fahrenheit {
        get => $this->celsius * 9 / 5 + 32;
        set => $this->celsius = ($value - 32) * 5 / 9;
    }
}
$t = new Temperature();
$t->fahrenheit = 212;
echo $t->celsius; // 100
```

### Q63. What are PHP 8 Attributes?

**Attributes (`#[...]`) are native, structured metadata attached to classes, methods, properties, or parameters, readable at runtime via Reflection — replacing docblock annotations.**

```php
<?php
#[Attribute]
class Route {
    public function __construct(public string $path) {}
}

class Controller {
    #[Route('/users')]
    public function index() {}
}
// Read with (new ReflectionMethod(...))->getAttributes(Route::class)
```

Frameworks use these for routing, validation, ORM mapping, etc.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `==` where `===` is meant.** Loose comparison causes subtle bugs (`"0" == false` is `true`; before PHP 8, `0 == "abc"` was `true`). **Fix:** default to `===` and explicit casts; only use `==` deliberately.

2. **The dangling `foreach` reference.** `foreach ($a as &$v) { ... }` leaves `$v` referencing the last element; a later loop or assignment silently corrupts the array. **Fix:** `unset($v);` right after a by-reference loop, or avoid `&` entirely.

3. **Confusing `self` and `static`.** Returning `new self()` from a parent factory breaks subclass factories. **Fix:** use `new static()` (late static binding) so the called subclass is instantiated.

4. **Floating-point money math.** `0.1 + 0.2 !== 0.3`, and rounding errors accumulate. **Fix:** store money as integer cents, or use `bcmath`/a decimal library; never compare floats with `==`.

5. **`isset()` vs `array_key_exists()` on null values.** `isset($arr['k'])` is `false` when the key exists but holds `null`, so you can miss legitimately-null entries. **Fix:** use `array_key_exists('k', $arr)` when `null` is a valid value.

6. **Building SQL by string concatenation.** Even "internal" inputs become injection vectors over time. **Fix:** always use parameterized prepared statements; whitelist identifiers.

7. **Echoing user input without escaping.** Leads to XSS. **Fix:** `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` per output context; in Blade prefer `{{ }}` over `{!! !!}`.

8. **Using `rand()`/`md5()` for tokens or passwords.** Predictable/weak. **Fix:** `random_bytes()`/`random_int()` for tokens, `password_hash()`/`password_verify()` for passwords.

---

## ✅ Best Practices

- **Prefer strict comparison (`===`)** and enable `declare(strict_types=1);` at the top of every file to disable silent scalar coercion on typed parameters.
- **Type everything:** parameters, returns, and properties. Use union/intersection types and enums to make illegal states unrepresentable.
- **Favor immutability:** `readonly` properties and value objects; clone-with-changes instead of mutating.
- **Inject dependencies** through constructors; program to interfaces, not concretions.
- **Catch narrow exception types**, chain causes with `previous`, and never swallow `Throwable` silently.
- **Escape on output, validate on input,** parameterize all queries, hash passwords with `password_hash`.
- **Turn on OPcache in production** and `composer install --no-dev --optimize-autoloader`.
- **Use generators** for large datasets/streams to keep memory flat.
- **Follow PSR-12** for style and PSR-4 for autoloading; run static analysis (PHPStan/Psalm) and tests (PHPUnit, or Pest — the default test runner in new Laravel 12 apps).

---

## 🎯 Interview Tips & Likely Questions

**Q: How does PHP's garbage collection work under the hood?**
A: Primary mechanism is **reference counting** on each `zval` — when the count hits zero the value is freed immediately. Because refcounting can't reclaim **reference cycles** (objects pointing at each other), PHP runs a separate **cycle collector** that buffers possible roots and scans them when the buffer fills (default 10,000) or when `gc_collect_cycles()` is called. In short-lived requests, GC rarely matters because everything is released at request teardown (shared-nothing model).

**Q: Why is `static::` needed when `self::` exists?**
A: `self::` binds at *compile time* to the class where the code is written, so a parent's `new self()` always makes a parent. `static::` uses **late static binding** — it resolves to the *runtime called class*, so a subclass factory returns the subclass. This is the canonical use of `new static()`.

**Q: Are PHP arrays passed by value or reference, and is that expensive?**
A: By value semantically, but cheap because of **copy-on-write** — the underlying buffer is shared (refcount) and only duplicated when one side writes. So you get value safety with reference-like performance until mutation.

**Q: What actually changed about `==` in PHP 8?**
A: Number-to-non-numeric-string comparison now casts the *number to a string* rather than the string to `0`. So `0 == "foo"` is `false` in PHP 8 (`true` in PHP 7). Numeric strings still compare numerically (`"10" == "1e1"` is `true`).

**Q: When would you use an interface vs an abstract class?**
A: Interface for a capability/contract a class can advertise (and a class can implement many); abstract class to share partial implementation/state through single inheritance. "Can-do" → interface; "is-a with shared base" → abstract class.

**Q: How does Composer autoloading work?**
A: It registers a callback via `spl_autoload_register`. On first use of an unknown class, PHP calls the autoloader, which (for PSR-4) maps the namespace prefix to a directory, builds the file path, and `require`s it. Production uses an optimized **classmap** (`--optimize-autoloader`) to skip filesystem scans.

**Q: How do you store a password securely, and why not SHA-256?**
A: Use `password_hash()` (bcrypt/Argon2id) — it's *deliberately slow* and salts automatically, resisting brute force. Fast hashes like SHA-256/MD5 are designed for speed, making them crackable at scale. Verify with `password_verify()` and upgrade via `password_needs_rehash()`.

**Q: Explain copy-on-write at the engine level.**
A: A `zval` is shared while `refcount > 1`. A write triggers **separation**: the engine duplicates the value so the writer mutates its own copy, leaving other holders untouched. References (`&`) set the `is_ref` flag and bypass separation so all aliases share one mutable value.

**Q: What's the difference between `Error` and `Exception`?**
A: Both implement `Throwable`. `Error` represents engine/runtime faults (`TypeError`, `DivisionByZeroError`) — usually bugs; `Exception` represents recoverable application conditions. Catch `Throwable` to handle both; catch specific types to handle precisely.

**Q: How would you process a 5 GB file without exhausting memory?**
A: Stream it with a **generator** that `yield`s one line/chunk at a time (`fgets` in a loop), keeping memory roughly constant instead of loading the whole file into an array.

---

## 📋 Quick Reference / Cheat Sheet

```text
COMPARISON
  ===   value + type, no coercion (prefer)
  ==    loose; PHP8: 0 == "a" is FALSE; "10" == "1e1" TRUE
  <=>   spaceship: -1 / 0 / 1
  ??    null-coalesce (null/unset only)   ??= assign-if-null
  ?:    elvis (falsy)

EXISTENCE
  isset($x)            true if set AND not null
  empty($x)            true if falsy (no notice); empty("0") is true
  is_null($x)          true only if exactly null
  array_key_exists()   true even if value is null

OOP RESOLUTION
  self::    compile-time lexical class
  static::  runtime called class (late static binding) → new static()
  parent::  parent implementation
  final     no override / no extend
  readonly  write-once property; readonly class = all props readonly
  public private(set)  asymmetric visibility (8.4)

ARRAYS
  copy-on-write: copies only on write; & disables COW
  array_map / array_filter / array_reduce / array_column
  sort/rsort (reindex) · asort/arsort (keep keys, by value)
  ksort/krsort (by key) · usort (comparator)
  sorts STABLE since PHP 8.0

ERRORS              Throwable ─┬─ Error (TypeError, DivisionByZeroError...)
                               └─ Exception (RuntimeException...)
  try / catch (A|B $e) / finally  (finally always runs; its return wins)
  chain: new X($msg, 0, $previous)

MEMORY
  refcount → free at 0; cycle collector for cycles; gc_collect_cycles()
  generators (yield) for lazy/streaming; OPcache + JIT for speed
  shared-nothing: state cleared at request end

SECURITY
  SQLi  → PDO prepared statements, EMULATE_PREPARES=false
  XSS   → htmlspecialchars($v, ENT_QUOTES, 'UTF-8')
  CSRF  → per-session token + hash_equals() + SameSite cookie
  Pwd   → password_hash / password_verify / password_needs_rehash
  Rand  → random_bytes() / random_int()  (never rand/mt_rand)

COMPOSER
  composer.json = constraints · composer.lock = exact versions (commit it)
  ^1.2.3 → <2.0.0   |   ~1.2.3 → <1.3.0
  install --no-dev --optimize-autoloader   (production)

PHP 8.x HIGHLIGHTS
  8.0 JIT · match · named args · nullsafe ?-> · attributes · union types · mixed
  8.1 enums · readonly props · fibers · first-class callable foo(...) · never · intersection
  8.2 readonly classes · DNF types · true/false/null types
  8.3 typed class constants · #[\Override] · json_validate()
  8.4 property hooks · asymmetric visibility · new Foo()->m() · implicit ?-nullable deprecated
```

---

## 🧪 Mini Exercises

1. **Comparison trap.** Without running it, predict the output of: `var_dump(0 == "a", "1" == "01", "abc" == 0, null == [], [] == false);` under PHP 8.4. Then explain each result in one sentence.

2. **Late static binding.** Write a base class `Model` with a static `make(): static` factory and a subclass `User`. Show that `User::make()` returns a `User` instance, and explain why `new self()` would have failed.

3. **Copy-on-write & references.** Create a large array, assign it to a second variable, then mutate the second. Explain at which line the array is actually duplicated. Then introduce a `&` reference and describe how that changes the behavior.

4. **Safe data layer.** Write a function that fetches a user row by email using a PDO **prepared statement** with exception mode on, and a second function that hashes a password and verifies it, upgrading the hash when `password_needs_rehash()` is true.

5. **Streaming with generators.** Implement a generator `evenNumbers(int $limit): Generator` that yields even numbers up to `$limit` lazily, and a consumer that sums the first 1,000 of them. Explain why this uses constant memory regardless of `$limit`.
