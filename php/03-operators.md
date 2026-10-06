# Operators in PHP (8.4) — From Zero to Expert

Operators are the verbs of a programming language. Everything you compute — adding numbers, comparing values, deciding which branch to take, defaulting a missing value — is expressed with operators. PHP has a *lot* of them, and several have subtle, interview-favorite behaviors (loose `==`, the `and`/`or` precedence trap, the nullsafe `?->`, the at-sign `@`). This module teaches every operator you'll be asked about, why it behaves the way it does, and the modern idioms that senior engineers reach for.

> **Jargon up front.** An **operator** is a symbol (or keyword) that performs an action on one or more values. The values it acts on are called **operands**. An **expression** is anything that evaluates to a value (e.g. `2 + 3`). **Precedence** decides which operator binds tighter when several appear in one expression; **associativity** decides the direction (left-to-right or right-to-left) when operators share the same precedence.

---

**What you'll learn**

- Every arithmetic operator including `**` (exponent) and the `intdiv()` function for integer division
- Assignment vs. compound assignment, and why `=` returns a value
- The full comparison family: `==` vs `===`, `!=`/`<>`/`!==`, and the spaceship `<=>`
- The exact loose-comparison rules and the **PHP 8** change to string-to-number comparison (a top interview question)
- Logical operators and the infamous `and`/`or` precedence trap
- Bitwise operators, increment/decrement (pre vs post, and string increment)
- Ternary, short ternary `?:`, null coalescing `??`, its assignment form `??=`, and the nullsafe `?->`
- `instanceof`, the error-control `@`, and a complete precedence & associativity table

---

## 1. Arithmetic operators

These are the operators you already know from math, plus two PHP-specific ones.

```php
<?php
echo 7 + 3;   // 10   addition
echo 7 - 3;   // 4    subtraction
echo 7 * 3;   // 21   multiplication
echo 7 / 3;   // 2.3333333333333   division (ALWAYS returns float unless evenly divisible)
echo 7 % 3;   // 1    modulo (remainder) — operates on INTEGERS
echo 2 ** 10; // 1024 exponentiation
```

**The WHY behind a few traps:**

- **`/` does not floor.** `10 / 4` is `2.5`, not `2`. If both operands divide evenly you get an `int` (`8 / 4 === 2`), otherwise a `float`. Division by zero throws a `DivisionByZeroError` in PHP 8 (in PHP 7 it emitted a warning and returned `false`).
- **`%` is integer modulo.** Operands are cast to `int` first, so `7.8 % 3` is computed as `7 % 3 === 1`. Note: in **PHP 8.1+** passing a float that loses precision (like `7.8`) to `%` emits an `E_DEPRECATED` ("Implicit conversion from float ... loses precision"); cast explicitly with `(int)` to silence it. The sign of the result follows the **dividend** (left operand): `-7 % 3 === -1`, but `7 % -3 === 1`. For floating-point remainder use `fmod()` — e.g. `fmod(7.8, 3.0)` returns `1.7999999999999998` (binary floating point can't represent `1.8` exactly), so never compare its result with `==`.
- **`**` is right-associative.** `2 ** 3 ** 2` means `2 ** (3 ** 2)` = `2 ** 9` = `512`, *not* `(2 ** 3) ** 2` = `64`.

### `intdiv()` — true integer division

There is no `//` operator in PHP. To get the integer quotient (dropping the fractional part), use the `intdiv()` function:

```php
<?php
echo intdiv(10, 3);   // 3
echo 10 / 3;          // 3.3333333333333
echo (int)(10 / 3);   // 3  — works here, but `/` makes a float first, which loses precision for very large operands (see note below)

intdiv(PHP_INT_MIN, -1); // throws ArithmeticError (result would overflow int)
intdiv(1, 0);            // throws DivisionByZeroError
```

> **Why `intdiv` over `(int)(a / b)`?** For very large operands, `a / b` produces a float, and floats lose integer precision beyond `2**53`. `intdiv()` stays in integer space the whole way, so it's both faster and exact.

### Unary plus / minus and numeric strings

```php
<?php
$x = -5;
echo -$x;       // 5   unary minus
echo +"42";     // 42  unary plus coerces a numeric string to int
echo +"3.14";   // 3.14 coerces to float
```

---

## 2. Assignment & compound assignment

The basic assignment operator is `=`. A subtle but important fact: **assignment is an expression that returns the assigned value.** That's why `$a = $b = 5;` works — `$b = 5` evaluates to `5`, which is then assigned to `$a`.

```php
<?php
$a = $b = 5;   // both are 5
echo $a, $b;   // 55
```

**By value vs. by reference.** `=` copies the value for scalars/arrays. `=&` binds two variables to the *same* underlying storage:

```php
<?php
$a = 1;
$b = &$a;   // reference: $b is now an alias of $a
$b = 99;
echo $a;    // 99  — changing $b changed $a
```

### Compound assignment

Compound operators combine an operation with assignment. `$x += 5` is shorthand for `$x = $x + 5`.

| Operator | Equivalent to | Notes |
|---|---|---|
| `+= -= *= /= %= **=` | `$x = $x + …` etc. | arithmetic |
| `.=` | `$x = $x . …` | string concatenation |
| `??=` | `$x = $x ?? …` | null coalescing assignment (PHP 7.4+) |
| `&= \|= ^= <<= >>=` | bitwise | |

```php
<?php
$total = 100;
$total -= 30;       // 70
$total *= 2;        // 140

$name = "Jane";
$name .= " Doe";    // "Jane Doe"

$config = [];
$config['debug'] ??= true;   // only sets it if not already set / not null
```

---

## 3. String concatenation

PHP uses the **dot** `.` to join strings — *not* `+`. Using `+` on strings would attempt arithmetic.

```php
<?php
$first = "Ada";
$last  = "Lovelace";
echo $first . " " . $last;   // "Ada Lovelace"

echo "5" + "5";   // 10   (numeric strings → arithmetic addition!)
echo "5" . "5";   // "55" (concatenation)
```

> **PHP 8 precedence fix:** Before PHP 8, `.` had the *same* precedence as `+`/`-`, so `echo "sum: " . 1 + 2;` parsed surprisingly. Since **PHP 8.0**, `.` has **lower** precedence than `+`/`-`, so `"sum: " . 1 + 2` now correctly evaluates the arithmetic first → `"sum: 3"`. Still, when mixing, use parentheses for clarity.

For building strings, interpolation is usually cleaner than concatenation:

```php
<?php
$user = "ada";
echo "Hello, {$user}!";            // "Hello, ada!"  (double quotes interpolate)
echo 'Hello, ' . $user . '!';      // same result, single quotes do NOT interpolate
```

---

## 4. Comparison operators

This is the single most interview-relevant section. PHP has **loose** comparison (`==`) and **strict** comparison (`===`).

| Operator | Name | True when… |
|---|---|---|
| `==` | Equal (loose) | values are equal *after type juggling* |
| `===` | Identical (strict) | equal **and** same type |
| `!=` or `<>` | Not equal (loose) | `==` is false |
| `!==` | Not identical (strict) | `===` is false |
| `<` `>` `<=` `>=` | Less/greater | numeric/string ordering |
| `<=>` | Spaceship | `-1`, `0`, or `1` |

```php
<?php
var_dump(1 == "1");    // bool(true)   loose: "1" juggled to 1
var_dump(1 === "1");   // bool(false)  strict: int vs string
var_dump(0 == "");     // bool(false) in PHP 8 (was true in PHP 7!)
var_dump(null == false); // bool(true)
var_dump([1,2] === [1,2]); // bool(true) same keys, values, order, types
```

### The PHP 8 string-to-number comparison change (KNOW THIS)

This is *the* classic "what changed in PHP 8" question.

**Before PHP 8:** when comparing a number to a string with `==`, PHP cast the **string to a number**. This meant `0 == "foo"` was **true**, because `"foo"` became `0`. This caused real security bugs (e.g. comparing a hashed token to `0`).

**PHP 8.0 onward:** when comparing a number to a **non-numeric string**, PHP now casts the **number to a string** instead and compares as strings. When the string *is* numeric, it still compares numerically.

```php
<?php
// --- PHP 8 behavior ---
var_dump(0 == "foo");    // bool(false)  ✅ (was TRUE before PHP 8)
var_dump(0 == "");       // bool(false)  ✅ (was TRUE before PHP 8)
var_dump(0 == "0");      // bool(true)   "0" is numeric → numeric compare
var_dump("1" == "01");   // bool(true)   both numeric strings → 1 == 1
var_dump("10" == "1e1"); // bool(true)   both numeric → 10 == 10
var_dump(100 == "1e2");  // bool(true)   "1e2" numeric → 100 == 100
var_dump("abc" == 0);    // bool(false)  ✅ non-numeric string vs number → string compare
```

The simplified PHP 8 rule: **if one operand is a number and the other a numeric string, compare as numbers; otherwise compare as strings.**

> **🔒 Security: never compare secrets with `==`/`===`.** Comparing tokens, password hashes, HMAC signatures, or API keys with `===` is vulnerable to **timing attacks** — `===` returns as soon as it finds the first differing byte, so an attacker can measure response time to recover the value byte by byte. Use the constant-time `hash_equals($known, $userSupplied)` instead. (And historically, loose `==` was worse still: a `"0e..."` "magic hash" could equal `0` or another magic hash. Always use `===` or, for secrets, `hash_equals()`.)
>
> ```php
> // ❌ timing-attack vulnerable
> if ($expectedToken === $request->header('X-Token')) { /* ... */ }
>
> // ✅ constant-time comparison
> if (hash_equals($expectedToken, (string) $request->header('X-Token'))) { /* ... */ }
> ```

### The spaceship `<=>` (combined comparison)

Returns `-1` if left is less, `0` if equal, `1` if greater. Its main job is powering sort callbacks.

```php
<?php
echo 1 <=> 2;     // -1
echo 2 <=> 2;     // 0
echo 3 <=> 2;     // 1
echo "a" <=> "b"; // -1
echo [1,2,3] <=> [1,2,1]; // 1  (arrays compared element by element)
```

A sort comparator must return an int that's negative/zero/positive — exactly what `<=>` gives you:

```php
<?php
$users = [
    ['name' => 'Zoe', 'age' => 30],
    ['name' => 'Ann', 'age' => 25],
    ['name' => 'Max', 'age' => 30],
];

// Sort by age ascending, then name ascending — concise with spaceship
usort($users, fn($a, $b) =>
    [$a['age'], $a['name']] <=> [$b['age'], $b['name']]
);
// Result order: Ann(25), Max(30), Zoe(30)

// Descending? Flip the operands:
usort($users, fn($a, $b) => $b['age'] <=> $a['age']);
```

> **Laravel note:** Collections give you higher-level sorting (`$collection->sortBy('age')`, `->sortByDesc()`, `->sortBy([['age','asc'],['name','asc']])`), so you rarely write `<=>` by hand in Laravel code — but interviewers still ask about it, and it's exactly what those methods use internally.

---

## 5. Logical operators & the precedence trap

PHP has two sets of logical operators that *look* equivalent but differ in **precedence**.

| Symbol form | Word form | Meaning |
|---|---|---|
| `&&` | `and` | logical AND |
| `\|\|` | `or` | logical OR |
| `!` | — | logical NOT |
| `xor` | — | exclusive OR (true if exactly one is true) |

Both `&&` and `||` **short-circuit**: in `A && B`, `B` is never evaluated if `A` is false; in `A || B`, `B` is skipped if `A` is true. This is widely used for guard clauses.

```php
<?php
$user && $user->isAdmin() && grantAccess();   // isAdmin() runs only if $user is truthy
$config['cache'] ?? $defaultCache;             // (covered below)
isLoggedIn() || redirectToLogin();             // redirect only if NOT logged in
```

### The `and` / `or` trap

`and`/`or` have **much lower precedence than `=`**, while `&&`/`||` have **higher** precedence than `=`. This produces a famous bug:

```php
<?php
$result = true && false;   // && binds tighter than = →  $result = (true && false) = false ✅
var_dump($result);         // bool(false)

$result = true and false;  // = binds tighter than 'and' → ($result = true) and false
var_dump($result);         // bool(true)   ❌ surprising!
```

In the second line, `=` runs first (`$result = true`), and the `and false` part is evaluated but its result is thrown away. The fix: **prefer `&&` and `||` everywhere.** The only common legitimate use of `or` is the old idiom `$fp = fopen($file, 'r') or die('cannot open');` — and even there, `&&`/exceptions are cleaner.

```php
<?php
// ❌ subtle bug
$ok = doThing() and logResult();   // $ok gets only doThing()'s return value

// ✅ clear
$ok = doThing() && logResult();
```

---

## 6. Bitwise operators

These operate on the individual **bits** of integers. You'll see them in flags/permissions (e.g. file modes, feature toggles).

| Operator | Name | Description |
|---|---|---|
| `&` | AND | 1 if both bits are 1 |
| `\|` | OR | 1 if either bit is 1 |
| `^` | XOR | 1 if bits differ |
| `~` | NOT | flips every bit |
| `<<` | left shift | shift bits left (× 2 per shift) |
| `>>` | right shift | shift bits right (÷ 2 per shift) |

```php
<?php
echo 6 & 3;   // 2    0110 & 0011 = 0010
echo 6 | 3;   // 7    0110 | 0011 = 0111
echo 6 ^ 3;   // 5    0110 ^ 0011 = 0101
echo ~5;      // -6   (two's complement)
echo 1 << 4;  // 16   1 shifted left 4 places = 2**4
echo 32 >> 2; // 8    32 / 4
```

A classic real-world pattern is **bit flags**:

```php
<?php
const PERM_READ  = 1;   // 0001
const PERM_WRITE = 2;   // 0010
const PERM_EXEC  = 4;   // 0100

$perms = PERM_READ | PERM_WRITE;        // 3 — combine flags
$canWrite = (bool)($perms & PERM_WRITE); // true — test a flag
$perms &= ~PERM_WRITE;                    // remove the write flag → 1
```

> **Watch out:** `&`/`|`/`^` on **strings** do bitwise operations on the bytes, which is almost never what you want. And don't confuse bitwise `&`/`|` with logical `&&`/`||` — bitwise operators do **not** short-circuit.

---

## 7. Increment & decrement

`++` adds one, `--` subtracts one. **Position matters:**

- **Pre-increment** `++$x`: increment first, *then* return the new value.
- **Post-increment** `$x++`: return the current value first, *then* increment.

```php
<?php
$a = 5;
echo $a++;  // 5  (returns old value, then $a becomes 6)
echo $a;    // 6

$b = 5;
echo ++$b;  // 6  (increments first, returns new value)
echo $b;    // 6
```

### String increment (a PHP curiosity interviewers love)

`++` on a string performs Perl-style alphanumeric increment. `--` on a string does **nothing** (no decrement for strings).

```php
<?php
$s = "a";  $s++;  echo $s;   // "b"
$s = "Az"; $s++;  echo $s;   // "Ba"  (carries like odometer)
$s = "Zz"; $s++;  echo $s;   // "AAa"
$s = "a9"; $s++;  echo $s;   // "b0"

$s = "z";  $s--;  echo $s;   // "z"   (-- on string is a no-op)
```

> **PHP 8.3+ note:** Perl-style increment of purely *alphanumeric* strings (like `"a9"` → `"b0"`) still works as shown above. What changed in **PHP 8.3** is that `++`/`--` on a **non-alphanumeric** string (e.g. `""`, `"foo!"`, `"-1"` treated as a string) is now `E_DEPRECATED` — previously such strings were silently treated as `0`/`-1`. Separately, incrementing `null` gives `int(1)` while decrementing `null` leaves it `null` (a frequent gotcha). Bottom line: never rely on `++`/`--` for anything but plain integers (or deliberate odometer-style alphanumeric IDs).

---

## 8. Ternary & short ternary

The **ternary operator** `?:` is a compact `if/else` that returns a value.

```php
<?php
$status = $isActive ? "active" : "inactive";
// equivalent to:
// if ($isActive) { $status = "active"; } else { $status = "inactive"; }
```

### Short ternary (the "Elvis" operator) `?:`

`$a ?: $b` returns `$a` if `$a` is **truthy**, otherwise `$b`. It's `$a ? $a : $b` without repeating `$a`.

```php
<?php
$name = $input ?: "Guest";   // "Guest" if $input is falsy ("", 0, null, [], false)
```

> **Nesting is forbidden without parentheses.** Since PHP 8.0, *unparenthesized* nested ternaries are a **fatal error** (it was deprecated in 7.4). Always parenthesize:

```php
<?php
// ❌ Fatal error in PHP 8
// $x = $a ? "a" : $b ? "b" : "c";

// ✅ explicit
$x = $a ? "a" : ($b ? "b" : "c");
```

**Short ternary vs null coalescing:** `?:` checks *truthiness* and warns on undefined variables; `??` checks for *null/undefined* only and never warns. They are NOT interchangeable:

```php
<?php
$v = "0";
echo $v ?: "fallback";   // "fallback"  ("0" is falsy)
echo $v ?? "fallback";   // "0"         ("0" is not null)
```

---

## 9. Null coalescing `??` and `??=`

`??` returns its left operand if it **exists and is not null**, otherwise the right operand. Crucially, it does **not** emit a warning if the left side is an undefined variable/array key — making it the go-to for safe defaults.

```php
<?php
$data = ['name' => 'Ada'];

echo $data['name'] ?? 'Anonymous';  // "Ada"
echo $data['email'] ?? 'no email';  // "no email"  (no "undefined key" warning)

// Chainable: first non-null wins
$city = $_GET['city'] ?? $user->city ?? $defaultCity ?? 'Unknown';
```

### Null coalescing assignment `??=`

`$x ??= $y` assigns `$y` to `$x` only if `$x` is currently null/unset. The right side is **not evaluated** if assignment isn't needed (useful for lazy/expensive defaults).

```php
<?php
function getConfig(array &$config): array {
    $config['timeout'] ??= 30;       // set default only if missing
    $config['retries'] ??= computeExpensiveDefault(); // NOT called if 'retries' set
    return $config;
}
```

---

## 10. The nullsafe operator `?->` (PHP 8.0+)

The nullsafe operator lets you call a method or access a property on something that **might be null**, short-circuiting the whole chain to `null` instead of crashing.

```php
<?php
// Old way — verbose null checks
$country = null;
if ($user !== null) {
    $address = $user->getAddress();
    if ($address !== null) {
        $country = $address->getCountry();
    }
}

// PHP 8 nullsafe — same result, one line
$country = $user?->getAddress()?->getCountry();
// If $user is null, or getAddress() returns null, $country is null. No error.
```

**Key rules and limits:**

- It short-circuits: as soon as a `?->` hits `null`, the rest of the chain is skipped and the whole expression is `null`.
- It works for **method calls and property access**, but **not** for array access (`?[]` does not exist) and **not** as an assignment target (`$a?->b = 1;` is a fatal error — you can't write *to* a maybe-null path).
- It is **not** a replacement for `??`. `$user?->name ?? 'Guest'` is a common, idiomatic combo: nullsafe to traverse, `??` to default.

```php
<?php
// Laravel: avoid crashes when a relationship is missing
$companyName = $order?->customer?->company?->name ?? 'Unknown';
```

> **Laravel caveat:** Eloquent's `optional()` helper predates `?->` and does similar work (`optional($user)->name`). With PHP 8, prefer the native `?->` — it's faster (no helper object) and clearer. `optional()` still has a niche use: `optional($value, fn($v) => ...)` with a callback.

---

## 11. `instanceof`

`instanceof` tests whether an object is an instance of a class, a subclass, or implements an interface. It returns a boolean.

```php
<?php
interface PaymentGateway {}
class StripeGateway implements PaymentGateway {}

$gateway = new StripeGateway();

var_dump($gateway instanceof StripeGateway);   // bool(true)
var_dump($gateway instanceof PaymentGateway);  // bool(true)  — implements the interface
var_dump($gateway instanceof \Stringable);     // bool(false)

// Use the ::class constant for the right-hand side instead of a magic string:
$class = StripeGateway::class;
var_dump($gateway instanceof $class);           // bool(true)  — variable holding a class name works
```

Notes:
- The right operand may be a class name (bareword), a string variable, or another object.
- `instanceof` does **not** throw if the class doesn't exist when given a *string* variable — it returns `false`. Use the `::class` constant to get compile-time safety.
- For closed sets of types, PHP 8's `match(true)` pairs nicely with `instanceof`:

```php
<?php
$label = match (true) {
    $shape instanceof Circle    => "round",
    $shape instanceof Rectangle => "boxy",
    default                     => "unknown",
};
```

---

## 12. The error-control operator `@` (and why to avoid it)

Prefixing an expression with `@` **suppresses** any error/warning/notice messages that expression would generate. The error still *occurs* — it's just silenced.

```php
<?php
$contents = @file_get_contents('maybe-missing.txt');  // no warning if file is missing
if ($contents === false) {
    // handle failure
}
```

**Why avoid it:**

1. It's a performance and debugging nightmare — it silences *everything*, including typos and fatal-precursor warnings, so real bugs hide.
2. In **PHP 8.0+**, `@` no longer suppresses errors that are severe enough to halt execution (fatal errors), so it gives a false sense of safety.
3. Better tools exist: check return values, use `try/catch`, set proper error handlers, or use functions that report failure via return value.

```php
<?php
// ❌ silences everything
$json = @json_decode($raw, true);

// ✅ explicit and safe (PHP 7.3+)
try {
    $json = json_decode($raw, true, 512, JSON_THROW_ON_ERROR);
} catch (\JsonException $e) {
    // handle malformed JSON
}
```

> **Rule of thumb:** if you find yourself reaching for `@`, there's almost always a return-value check or exception-based API that does the job correctly.

---

## 13. Operator precedence & associativity table

When an expression mixes operators, **precedence** (higher binds tighter) and **associativity** (tie-breaker direction) decide the grouping. The list below goes from **highest** to **lowest** precedence (PHP 8.x).

| Associativity | Operators | Notes |
|---|---|---|
| (n/a) | `clone` `new` | highest |
| right | `**` | exponent is right-associative |
| right | `++` `--` `~` `(int)`/`(float)`/`(string)`/`(array)`/`(object)`/`(bool)` `@` | unary, casts, error-control |
| non-assoc | `instanceof` | type check |
| (n/a) | `!` | logical not (unary) |
| left | `*` `/` `%` | |
| left | `+` `-` | arithmetic add/sub |
| left | `<<` `>>` | bit shifts |
| left | `.` | concatenation (lowered below +/- in PHP 8) |
| non-assoc | `<` `<=` `>` `>=` | |
| non-assoc | `==` `!=` `===` `!==` `<=>` `<>` | |
| left | `&` | bitwise AND |
| left | `^` | bitwise XOR |
| left | `\|` | bitwise OR |
| left | `&&` | logical AND (symbol) |
| left | `\|\|` | logical OR (symbol) |
| right | `??` | null coalescing |
| non-assoc | `? :` | ternary |
| right | `=` `+=` `-=` `*=` `**=` `/=` `.=` `%=` `&=` `\|=` `^=` `<<=` `>>=` `??=` | assignment |
| left | `and` | low-precedence AND |
| left | `xor` | |
| left | `or` | lowest |

**The two precedence facts that win interviews:**

1. `**` is **right-associative** → `2 ** 2 ** 3 == 2 ** 8 == 256`.
2. `and`/`or` sit **below** assignment → `$x = true and false;` sets `$x` to `true`. Use `&&`/`||`.

When in doubt, **add parentheses.** Readable code beats clever precedence every time.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `and`/`or` with assignment.**
   `$ok = save() and notify();` assigns only `save()`'s result to `$ok`. **Fix:** use `&&`/`||`, which bind tighter than `=`: `$ok = save() && notify();`.

2. **Assuming pre-PHP-8 loose comparison.**
   Writing `if ($value == 0)` to detect an empty/invalid string. In PHP 8, `"foo" == 0` is **false** (it was true in PHP 7). **Fix:** use `===` for type-safe checks, and validate numeric strings with `is_numeric()` before numeric comparison.

3. **Nesting ternaries without parentheses.**
   `$x = $a ? 1 : $b ? 2 : 3;` is a **fatal error** in PHP 8. **Fix:** parenthesize — `$a ? 1 : ($b ? 2 : 3)` — or use `match`.

4. **Confusing `?:` with `??`.**
   `$n = $count ?: 10;` returns `10` even when `$count` is a legitimate `0`. **Fix:** use `??` when only null/undefined should trigger the default: `$count ?? 10`.

5. **Reaching for `?->` on array keys.**
   `$user?->roles?[0]` is invalid — there's no nullsafe array access. **Fix:** combine: `($user?->roles ?? [])[0] ?? null`, or use Laravel's `data_get($user, 'roles.0')`.

6. **Using `@` to hide problems.**
   `@$arr['key']` silences the warning but masks the real bug. **Fix:** use `$arr['key'] ?? $default`.

7. **Integer division pitfalls.**
   `(int)(9999999999999999 / 3)` loses precision because the division produces a float. **Fix:** use `intdiv()`. Also remember `7 / 2` is `3.5`, not `3`.

---

## ✅ Best Practices

- **Default to strict comparison (`===`/`!==`).** Reach for `==` only when you intentionally want type juggling, and even then prefer an explicit cast or `is_numeric()` check.
- **Never compare secrets with `==`/`===`.** Use the constant-time `hash_equals()` for tokens, signatures, and hashes to avoid timing attacks (and verify passwords with `password_verify()`, which is also constant-time).
- **Always use `&&`/`||`, never `and`/`or`** (except the rare deliberate low-precedence idiom — and even then, comment it).
- **Use `??` for "missing value" defaults and `?:` for "falsy" defaults** — pick deliberately based on whether `0`/`""` should count.
- **Combine `?->` with `??`**: `$obj?->prop ?? $default` is the idiomatic safe-traverse-then-default.
- **Parenthesize anything non-obvious.** Don't make the reader recall the precedence table.
- **Prefer `intdiv()` over `(int)($a / $b)`** for exact integer division.
- **Never use `@`.** Use return-value checks and exceptions (`JSON_THROW_ON_ERROR`, `try/catch`).
- **Use the spaceship `<=>` in `usort` comparators**, and prefer Laravel's `sortBy`/`sortByDesc` collection methods in app code.
- **Use `::class` instead of string class names** with `instanceof` for refactor-safe, IDE-checkable code.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `==` and `===`?**
`==` is loose equality: it compares values after type juggling (`1 == "1"` is true). `===` is strict identity: values must be equal **and** the same type (`1 === "1"` is false). Always default to `===` to avoid surprises.

**Q2. What changed about string-to-number comparison in PHP 8?**
Before PHP 8, comparing a number to a string cast the *string* to a number, so `0 == "foo"` was **true** (a security hazard). In PHP 8, if the string is **non-numeric**, the *number* is cast to a string and they compare as strings, so `0 == "foo"` is now **false**. Numeric strings (`"10"`, `"1e2"`) still compare numerically.

**Q3. Explain the `and`/`or` precedence trap.**
`and`/`or` have lower precedence than `=`, while `&&`/`||` have higher. So `$x = true and false;` parses as `($x = true) and false` — `$x` becomes `true`. `$x = true && false;` parses as `$x = (true && false)` — `$x` becomes `false`. Use `&&`/`||`.

**Q4. Difference between `?:`, `??`, and `?->`?**
`?:` (short ternary) returns the left side if it's **truthy**. `??` (null coalescing) returns the left side if it **exists and isn't null** (no warning on undefined). `?->` (nullsafe) safely traverses a method/property chain, yielding `null` instead of an error if any link is null. They solve different problems and are often combined.

**Q5. How does the nullsafe operator work under the hood?**
`?->` short-circuits the **entire** chain. When evaluating `$a?->b()?->c()`, PHP evaluates `$a`; if it's `null`, the whole expression immediately becomes `null` and `b()`/`c()` are never called. It is purely null-checking — it does not catch exceptions or handle non-object values like `false`. It cannot be used as an assignment target or for array access.

**Q6. What does the spaceship operator return and where is it used?**
`<=>` returns `-1`, `0`, or `1` for less-than, equal, greater-than. It's designed for sort comparators (`usort`, `uasort`), since those require a negative/zero/positive int. You can sort by multiple keys by comparing arrays: `[$a->x, $a->y] <=> [$b->x, $b->y]`.

**Q7. Why prefer `intdiv()` over casting `/` to int?**
`/` always produces a float when the division isn't exact, and floats lose integer precision past `2**53`. `intdiv()` stays in integer arithmetic, giving exact results and throwing `ArithmeticError`/`DivisionByZeroError` on overflow/zero rather than silently producing wrong values.

**Q8. What does `2 ** 3 ** 2` evaluate to, and why?**
`512`. `**` is **right-associative**, so it groups as `2 ** (3 ** 2)` = `2 ** 9` = `512`, not `(2 ** 3) ** 2` = `64`.

**Q9. Why should you avoid the `@` operator?**
It suppresses *all* diagnostics for an expression, hiding real bugs and hurting debuggability, and since PHP 8 it doesn't even silence fatal-level errors — giving false safety. Use return-value checks, `??` for missing data, and exception-based APIs (`JSON_THROW_ON_ERROR`) instead.

**Q10. What's the result of `"5" + "5"` vs `"5" . "5"`?**
`"5" + "5"` is `10` — `+` is arithmetic, so numeric strings are added. `"5" . "5"` is `"55"` — `.` is string concatenation. PHP uses `.`, not `+`, to join strings.

**Q11. How should you compare a user-supplied token to a secret, and why not `===`?**
Use `hash_equals($known, $userSupplied)`. `===` returns at the first differing byte, so its run time leaks how many leading bytes matched — a timing side-channel an attacker can exploit to recover the secret. `hash_equals()` compares in constant time regardless of where the mismatch is. For passwords, use `password_verify()` (also constant-time) against a `password_hash()` digest — never compare raw or naively-hashed passwords with `==`/`===`.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ---- Arithmetic ----
$a + $b   $a - $b   $a * $b   $a / $b   $a % $b   $a ** $b
intdiv(10, 3); // 3   fmod(7.8, 3.0); // 1.7999999999999998 (float remainder)

// ---- Assignment ----
$x = 5;  $x += 2;  $x .= "!";  $x **= 2;  $x ??= $default;  $b = &$a;

// ---- Comparison ----
$a == $b     // loose (type juggling)
$a === $b    // strict (value + type)
$a != $b     $a <> $b   // loose not-equal
$a !== $b    // strict not-equal
$a <=> $b    // -1 / 0 / 1  (spaceship)

// ---- Logical (USE THESE) ----
$a && $b   $a || $b   !$a   $a xor $b
// AVOID:  and  or  (low precedence — assignment trap)

// ---- Bitwise ----
$a & $b   $a | $b   $a ^ $b   ~$a   $a << $n   $a >> $n

// ---- Increment / Decrement ----
$x++  ++$x  $x--  --$x        // string ++: "az"++ → "ba";  string -- is a no-op

// ---- Conditional / Null handling ----
$cond ? $yes : $no            // ternary
$a ?: $b                      // short ternary (truthy?)
$a ?? $b                      // null coalescing (not-null?)
$a ??= $b                     // null coalescing assignment
$obj?->prop?->method()        // nullsafe (null instead of error)

// ---- Type / Misc ----
$obj instanceof SomeClass     // type check (use ::class for the name)
@expr                         // error suppression — AVOID
```

**Precedence (high → low), the parts you must remember:**
`**` (right-assoc) > `* / %` > `+ -` > `.` > comparisons > `&&` > `||` > `??` > ternary > `= += …` > `and` > `xor` > `or`

---

## 🧪 Mini Exercises

1. **Comparison forensics.** Without running it, predict the output of each line, then explain *why* using the PHP 8 rules:
   `var_dump(0 == "");`, `var_dump(0 == "0");`, `var_dump("abc" == 0);`, `var_dump("1e3" == "1000");`, `var_dump(null == false);`, `var_dump([] == false);`.

2. **Fix the precedence bug.** This function is supposed to set `$success` to whether *both* operations worked, but it always returns `true` from the first call. Find the bug and fix it:
   ```php
   function process(): bool {
       $success = validate() and persist();
       return $success;
   }
   ```

3. **Default-value drill.** You receive `$input` which may be `null`, `""`, `"0"`, or a real string. Write three expressions — one with `?:`, one with `??`, and one with `??=` — and describe exactly which inputs each treats as "missing." Explain when you'd choose each.

4. **Safe traversal.** Given a possibly-null `$order` whose chain is `$order->customer->company->name`, write a single expression that returns the company name or the string `"N/A"` if any link is null. Then write the equivalent in Laravel using `data_get()`.

5. **Spaceship sort.** Given `$people = [['name'=>'Bo','age'=>40], ['name'=>'Al','age'=>40], ['name'=>'Cy','age'=>22]]`, write a `usort` callback using `<=>` that sorts by `age` ascending and, for equal ages, by `name` ascending. Then rewrite it as a Laravel collection chain.
