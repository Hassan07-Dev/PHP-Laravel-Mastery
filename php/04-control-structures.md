# Control Structures: `if` / `switch` / `match`

Control structures are the decision-makers of your code. They let a program choose *which* statements to run based on the state of your data — the difference between "send the invoice" and "block the fraudulent order." In a Laravel backend you reach for them constantly: routing logic, validation branches, Blade templates, permission checks, and state machines. This module takes you from the humble `if` all the way to PHP's modern `match` expression, and equips you to answer the "`switch` vs `match`" question that interviewers love.

> Target versions: **PHP 8.4** (with notes for 8.1–8.3) and **Laravel 12** (with notes for Laravel 10/11). Examples assume `declare(strict_types=1);` unless stated.

---

**What you'll learn**

- How `if` / `elseif` / `else` work, how to nest them cleanly, and when nesting becomes a smell.
- The **alternative colon syntax** (`if: … endif;`) and why Blade templates rely on it.
- `switch` semantics: **fallthrough**, `break`, `default`, and the **loose-comparison** trap.
- The `match` **expression** — strict comparison, no fallthrough, returns a value, must be exhaustive, and multiple conditions per arm.
- Using the **ternary** and **null coalescing** operators as compact control flow.
- What `declare(strict_types=1)` does and how it changes comparison behavior.
- How `include` / `require` (and their `_once` variants) affect program flow.
- Why `goto` exists, why you should avoid it, and a crisp **`switch` vs `match`** comparison for interviews.

---

## 1. Why control structures matter (the WHY)

A program without branching can only do one fixed thing. Control structures introduce **conditional execution**: evaluate a *condition* (an expression that resolves to a boolean) and run different code accordingly. Getting them right is about three things:

1. **Correctness** — picking comparison semantics that match your intent (strict vs loose).
2. **Readability** — a reader should grasp the branches at a glance.
3. **Exhaustiveness** — handling every case so unexpected input fails loudly instead of silently doing the wrong thing.

PHP gives you a ladder of tools, from the flexible-but-verbose `if` to the strict-and-safe `match`. Knowing which rung to stand on is the skill.

---

## 2. `if` / `elseif` / `else`

The `if` statement runs a block only when its condition is **truthy**. Add `elseif` for additional mutually exclusive conditions and `else` for the fallback.

```php
<?php
declare(strict_types=1);

$score = 82;

if ($score >= 90) {
    echo "Grade: A";
} elseif ($score >= 80) {
    echo "Grade: B";
} elseif ($score >= 70) {
    echo "Grade: C";
} else {
    echo "Grade: F";
}
// Output: Grade: B
```

Conditions are evaluated **top to bottom**; the first truthy branch wins and the rest are skipped. Order matters — if you wrote `>= 70` first, an A student would be graded C.

### Truthiness: what counts as `false`

PHP coerces the condition to boolean. These values are **falsy**: `false`, `0`, `0.0`, `"0"`, `""` (empty string), `[]` (empty array), and `null`. **Everything else is truthy** — including the string `"0.0"`, `"false"`, and `-1`.

```php
<?php
if ("0") {
    echo "truthy";
} else {
    echo "falsy";   // "0" is falsy
}
// Output: falsy

if ("0.0") {        // NOT the same as "0"
    echo "truthy";  // this runs
}
// Output: truthy
```

> Gotcha: `"0"` is falsy but `"0.0"` and `" "` (a space) are truthy. This bites people checking form input.

### `elseif` vs `else if`

In curly-brace syntax, `elseif` (one word) and `else if` (two words) behave identically. **But** in the alternative colon syntax (next section) you *must* use `elseif` — `else if` is a parse error there.

### Nesting and when to stop

You can nest `if` inside `if`, but deep nesting hurts readability. Prefer **guard clauses** (early `return`) to flatten logic.

```php
<?php
// ❌ Arrow-shaped, hard to read
function discount(?User $user): float
{
    if ($user !== null) {
        if ($user->isActive()) {
            if ($user->isPremium()) {
                return 0.20;
            }
        }
    }
    return 0.0;
}

// ✅ Flattened with guard clauses
function discount(?User $user): float
{
    if ($user === null)       return 0.0;
    if (!$user->isActive())   return 0.0;
    if (!$user->isPremium())  return 0.0;

    return 0.20;
}
```

Guard clauses read like a checklist and keep the "happy path" un-indented at the bottom.

---

## 3. The alternative colon syntax (`if: … endif;`)

PHP offers a second syntax for `if`, `for`, `foreach`, `while`, and `switch`: replace the opening `{` with `:` and the closing `}` with `endif;` / `endforeach;` / etc. **Why does this exist?** It interleaves cleanly with HTML — you can close a block far away from where it opened and still read what it closes.

```php
<?php $loggedIn = true; ?>

<?php if ($loggedIn): ?>
    <p>Welcome back!</p>
<?php elseif ($guest ?? false): ?>
    <p>Browsing as guest.</p>
<?php else: ?>
    <p>Please log in.</p>
<?php endif; ?>
```

Output (when `$loggedIn` is `true`):

```html
    <p>Welcome back!</p>
```

> Note: in colon syntax you **must** use `elseif`, not `else if`. The two-word form causes a parse error here.

### Blade is built on this idea

Laravel's **Blade** templating engine (`.blade.php` files) compiles its `@if` directives down to exactly this colon syntax. You rarely write raw `<?php if: ?>` in Laravel — you write Blade:

```blade
@if ($user->isAdmin())
    <span class="badge">Admin</span>
@elseif ($user->isEditor())
    <span class="badge">Editor</span>
@else
    <span class="badge">Member</span>
@endif

{{-- Blade also has the inverse @unless --}}
@unless ($user->hasVerifiedEmail())
    <div class="alert">Please verify your email.</div>
@endunless
```

Behind the scenes Blade transforms `@if (...)` into `<?php if (...): ?>` and `@endif` into `<?php endif; ?>`, then caches the compiled PHP under `storage/framework/views/`. So mastering the colon syntax means understanding what Blade actually does. (This compilation behavior is identical across Laravel 10, 11, and 12.)

---

## 4. The ternary operator (`?:`) and null coalescing (`??`)

The **ternary operator** is `if`/`else` compressed into an expression that *returns a value*:

```php
<?php
$age = 20;
$label = $age >= 18 ? "adult" : "minor";
echo $label;
// Output: adult
```

Use it for simple value selection, **not** to run side-effectful statements. Nesting ternaries is notoriously confusing — PHP 8 actually made **unparenthesized nested ternaries a fatal error** to stop the bleeding:

```php
<?php
// ❌ Fatal error in PHP 8.0+ : "Unparenthesized `a ? b : c ? d : e` is not supported"
// $x = true ? 1 : false ? 2 : 3;

// ✅ Add parentheses to clarify intent
$x = true ? 1 : (false ? 2 : 3);
echo $x;
// Output: 1
```

### The Elvis operator `?:`

`$a ?: $b` returns `$a` if it is **truthy**, otherwise `$b`. It is shorthand for `$a ? $a : $b`.

```php
<?php
$name = "" ?: "Anonymous";
echo $name;
// Output: Anonymous   ("" is falsy)
```

### Null coalescing `??` (this is the one you want most of the time)

`$a ?? $b` returns `$a` only if it is **set and not null** — it does *not* trigger a warning on undefined variables/keys. Compare:

```php
<?php
$config = ['timeout' => 0];

// ?: treats 0 as falsy → wrong default applied
echo $config['timeout'] ?: 30;   // Output: 30  ❌ (0 was a valid value!)

// ?? only falls back on null/unset → correct
echo $config['timeout'] ?? 30;   // Output: 0   ✅

echo $config['missing'] ?? 30;   // Output: 30  (key absent, no warning)
```

**Rule of thumb:** use `??` when you mean "if not provided," and `?:` only when you genuinely mean "if falsy." The null-coalescing assignment `$x ??= 'default';` assigns only when `$x` is null/unset.

---

## 5. `switch`

`switch` compares one subject against many candidate values. Historically it was the go-to for "dispatch on a value," but it carries sharp edges.

```php
<?php
$role = "editor";

switch ($role) {
    case "admin":
        $level = 3;
        break;
    case "editor":
        $level = 2;
        break;
    case "viewer":
        $level = 1;
        break;
    default:
        $level = 0;
}

echo $level;
// Output: 2
```

### Fallthrough and `break`

A `switch` case does **not** stop on its own — execution "falls through" into the next case until it hits a `break` (or the end of the switch). Forgetting `break` is the classic `switch` bug:

```php
<?php
$role = "admin";

switch ($role) {
    case "admin":
        echo "admin ";
        // ❌ no break — falls through!
    case "editor":
        echo "editor ";
        break;
    case "viewer":
        echo "viewer ";
        break;
}
// Output: admin editor
```

Intentional fallthrough is occasionally useful — grouping multiple labels that share a body:

```php
<?php
switch ($day) {
    case "Sat":
    case "Sun":
        echo "Weekend";
        break;
    default:
        echo "Weekday";
}
```

### `default`

`default` runs when no case matches. It does **not** have to be last (though placing it last is conventional and clearest), and it also needs a `break` if more cases follow it.

### `continue` inside a `switch` is a trap

A `switch` counts as a loop-level target for `break`/`continue`. Inside a loop, writing `continue` to "skip this iteration" actually targets the `switch`, not the loop — it behaves like `break`, and PHP emits a warning:

```php
<?php
foreach ([1, 2, 3] as $n) {
    switch ($n) {
        case 2:
            continue;   // ❌ Warning: "continue" targeting switch is equivalent to "break"
        default:
            echo $n;
    }
}
// Output: 13   (the warning fires; the loop is NOT skipped the way you intended)
```

To skip the enclosing loop iteration from inside a `switch`, use `continue 2;` (the level count includes the switch). `match`, being an expression rather than a loop-level construct, has no such pitfall.

### The loose-comparison gotcha (critical)

`switch` compares with **loose equality (`==`)**, not strict (`===`). This causes type juggling surprises:

```php
<?php
$value = 0;   // an integer

switch ($value) {
    case "hello":          // "hello" == 0 ... ?
        echo "matched string!";
        break;
    default:
        echo "default";
}
```

On **PHP 7.x** this printed `matched string!` because `0 == "hello"` was `true` (the string was cast to `0`). **PHP 8 changed string↔int comparison rules**, so `0 == "hello"` is now `false`, and the snippet prints `default`. But subtler cases still bite:

```php
<?php
$value = "1abc";

switch ($value) {
    case 1:                // "1abc" == 1 → still true-ish on numeric-leading strings? 
        echo "matched 1";  // In PHP 8, "1abc" == 1 is FALSE (non-numeric string)
        break;
    case "1abc":
        echo "matched exact";
        break;
}
// Output (PHP 8): matched exact
```

The reliable takeaway: **`switch` uses `==`, and `==` is full of edge cases.** When you need predictable, type-safe matching, reach for `match`.

### Alternative colon syntax for `switch`

```php
<?php switch ($status): case "ok": ?>
    <p>All good</p>
<?php break; default: ?>
    <p>Unknown</p>
<?php endswitch; ?>
```

> Historical gotcha: in PHP 7 you could not emit any output (not even whitespace) between `switch (...):` and the first `case` — doing so was a fatal error. PHP 8 is more lenient, but the rule of thumb stands: you'll almost never write hand-rolled colon `switch`, because it's awkward and error-prone. Let Blade generate it.

Blade's `@switch` / `@case` / `@break` / `@default` directives compile to this colon form:

```blade
@switch($role)
    @case('admin')
        <p>Administrator</p>
        @break
    @case('editor')
        <p>Editor</p>
        @break
    @default
        <p>Guest</p>
@endswitch
```

---

## 6. The `match` expression (PHP 8.0+)

`match` is the modern answer to `switch`'s problems. It is an **expression** (it produces a value you can assign or return), it uses **strict comparison (`===`)**, it has **no fallthrough**, and it **must be exhaustive** — an unmatched subject throws `\UnhandledMatchError` instead of silently doing nothing.

```php
<?php
declare(strict_types=1);

$role = "editor";

$level = match ($role) {
    "admin"  => 3,
    "editor" => 2,
    "viewer" => 1,
    default  => 0,
};

echo $level;
// Output: 2
```

Note the differences from `switch`:

- Arms use `=>` and are **comma-separated** (the whole thing is one statement ending in `;`). A **trailing comma** after the last arm is allowed.
- No `break` needed — each arm is self-contained, exactly one runs.
- It returns a value, so you can assign it directly.
- The **subject is evaluated exactly once**, then compared against each arm's value(s) top to bottom with `===`. The first matching arm's expression is evaluated and becomes the result; the rest are never touched.

### Strict comparison means no type juggling

```php
<?php
$value = "1";   // a string

echo match ($value) {
    1   => "int one",
    "1" => "string one",
};
// Output: string one   (=== distinguishes "1" from 1)
```

With a `switch`, the `case 1:` could have matched `"1"` via `==`. `match` won't.

### Multiple conditions per arm

Separate values with commas to share one result — this replaces the "stacked case labels" pattern:

```php
<?php
$day = "Sun";

$type = match ($day) {
    "Sat", "Sun"                       => "Weekend",
    "Mon", "Tue", "Wed", "Thu", "Fri" => "Weekday",
};

echo $type;
// Output: Weekend
```

### Exhaustiveness: handle every case or throw

If no arm matches and there is no `default`, PHP throws `\UnhandledMatchError`:

```php
<?php
$status = "archived";

try {
    echo match ($status) {
        "draft"     => "Editing",
        "published" => "Live",
    };
} catch (\UnhandledMatchError $e) {
    echo "Unhandled: " . $e->getMessage();
}
// Output: Unhandled: Unhandled match case 'archived'
```

This "fail loud" behavior is a **feature**: when you add a new status to your enum, any `match` without a `default` will throw on the new case during testing, reminding you to handle it. With `switch`, the missing case would silently slip into `default` (or do nothing).

### `match (true)` for range/boolean logic

`match` matches against discrete values, not ranges. To express `if`/`elseif` chains, switch the subject to `true` and put boolean expressions in the arms — the first arm that `=== true` wins:

```php
<?php
$score = 82;

$grade = match (true) {
    $score >= 90 => "A",
    $score >= 80 => "B",
    $score >= 70 => "C",
    default      => "F",
};

echo $grade;
// Output: B
```

This is the idiomatic modern replacement for a long `if`/`elseif` ladder that produces a value.

### `match` with enums (very interview-relevant)

`match` pairs beautifully with PHP 8.1 **enums**, and because enum cases are a fixed set, you often omit `default` to get compile-time-ish exhaustiveness pressure:

```php
<?php
enum OrderStatus: string
{
    case Pending  = 'pending';
    case Shipped  = 'shipped';
    case Delivered = 'delivered';

    public function label(): string
    {
        return match ($this) {
            OrderStatus::Pending   => 'Awaiting shipment',
            OrderStatus::Shipped   => 'On its way',
            OrderStatus::Delivered => 'Delivered',
        };
    }
}

echo OrderStatus::Shipped->label();
// Output: On its way
```

If you later add `case Cancelled` and forget to handle it in `label()`, the `match` throws `UnhandledMatchError` the first time a cancelled order hits that method — a loud, testable failure.

### In Laravel

`match` shines in controllers, model accessors, and policies. A real Eloquent accessor in Laravel 9–12 uses the `Illuminate\Database\Eloquent\Casts\Attribute` class (the legacy `getStatusColorAttribute()` magic-method style still works but is no longer the documented approach):

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class Payment extends Model
{
    // Modern accessor: access as $payment->status_color
    protected function statusColor(): Attribute
    {
        return Attribute::make(
            get: fn (): string => match ($this->status) {
                'pending' => 'warning',
                'paid'    => 'success',
                'failed'  => 'danger',
                default   => 'secondary',
            },
        );
    }
}
```

A plain helper method works too when you don't need it exposed as a virtual attribute:

```php
<?php
public function statusColor(): string
{
    return match ($this->status) {
        'pending' => 'warning',
        'paid'    => 'success',
        'failed'  => 'danger',
        default   => 'secondary',
    };
}
```

Blade has **no** `@match` directive — use `match` inside `{{ }}` echoes or (better) in the controller/model/accessor and pass the result to the view.

---

## 7. `declare(strict_types=1)` and how it affects comparisons

`declare(strict_types=1);` must be the **very first statement** in a file (before any other code, after an optional opening `<?php`). It changes how PHP handles **scalar type coercion at function boundaries**: with strict types on, passing a `string` where an `int` is declared throws a `TypeError` instead of silently casting.

```php
<?php
declare(strict_types=1);

function addOne(int $n): int {
    return $n + 1;
}

echo addOne(5);     // Output: 6
echo addOne("5");   // ❌ TypeError: addOne(): Argument #1 ($n) must be of type int, string given
```

Without `declare(strict_types=1)`, PHP runs in **coercive mode** and `addOne("5")` would quietly become `addOne(5)`.

**How this relates to control structures:** strict types do **not** change `==`, `===`, `match`, or `switch` comparison rules directly — those operators behave the same regardless. The connection is *indirect but important*: strict types stop sloppy coercion from sneaking the wrong type into the value you later branch on. If a function guarantees it returns a real `int`, your `match ($n)` arms can trust they're comparing ints. **Always put `declare(strict_types=1)` at the top of every PHP file** — it is standard in modern Laravel codebases.

> Scope note: `strict_types` is per-file and affects only calls *made from* that file, based on where the call originates.

---

## 8. `include` / `require` and `_once` — how they affect flow

These constructs pull another PHP file's contents into the current execution at that point — they are a control-of-*flow* mechanism, not a conditional, but they belong in this discussion because they determine *what code runs*.

| Construct      | On failure to find file | Re-runs if called again? |
|----------------|-------------------------|--------------------------|
| `include`      | Emits a **warning**, continues | Yes |
| `require`      | Emits a **fatal error**, halts | Yes |
| `include_once` | Warning, continues      | **No** (skipped if already included) |
| `require_once` | Fatal error, halts      | **No** |

```php
<?php
// config.php returns a value:
//   <?php return ['debug' => true];
$config = require __DIR__ . '/config.php';  // captures the returned array
var_dump($config['debug']);
// Output: bool(true)
```

- Use **`require`** for files your program *cannot run without* (a class, critical config) — you want it to die loudly if missing.
- Use **`include`** only when the file is optional (e.g. a non-essential template) and continuing without it is acceptable.
- The **`_once`** variants guard against double-inclusion, which would otherwise cause "Cannot redeclare class/function" fatal errors. They track included paths and silently skip repeats.

A file can `return` a value (as above), which is how Laravel's config files (`config/app.php`) and route files work under the hood. In day-to-day Laravel you almost never call these yourself — **Composer's autoloader** (`vendor/autoload.php`, loaded once in `public/index.php`) lazily `require`s class files for you via PSR-4. You'll mostly see `require __DIR__.'/../vendor/autoload.php';` once at the entry point.

> Performance note: `include`/`require` of the *same* path repeatedly is cheap on opcode-cached servers (OPcache), but `_once` adds a small bookkeeping cost. For class loading, rely on the autoloader, not manual `require`.

---

## 9. `goto` — mentioned, and discouraged

PHP has a `goto` operator that jumps execution to a labeled point in the same file/function scope:

```php
<?php
$i = 0;
loop:
echo $i;
$i++;
if ($i < 3) goto loop;
// Output: 012
```

**Don't use it.** `goto` produces "spaghetti code" — control flow that's hard to follow, debug, and refactor. It cannot jump *into* a loop or function, only within constraints, and every legitimate use is better expressed with loops, functions, `break`/`continue` (with levels), or exceptions. Interviewers ask about it to confirm you *know it exists and know to avoid it*. The correct answer is: "PHP has `goto`, but I never use it; structured control flow and early returns are clearer."

---

## 10. `switch` vs `match` — the interview comparison

| Aspect                      | `switch`                              | `match`                                  |
|-----------------------------|---------------------------------------|------------------------------------------|
| Statement or expression?    | Statement (no return value)           | **Expression** (returns a value)         |
| Comparison                  | **Loose** (`==`) — type juggling      | **Strict** (`===`) — no juggling         |
| Fallthrough                 | Yes — needs `break`                   | **No** — each arm isolated               |
| Unmatched + no default      | Silently does nothing                 | Throws `\UnhandledMatchError`            |
| Multiple values per branch  | Stacked `case` labels                 | Comma-separated in one arm               |
| Bodies                      | Multiple statements per case          | Single expression per arm                |
| Syntax weight               | Verbose (`case`, `:`, `break`)        | Concise (`=>`, `,`)                      |
| Available since             | Always                                | PHP 8.0                                  |

**When to use which:**

- Use **`match`** for value selection / dispatch when each branch yields a value — the default choice in modern PHP. Its strictness and exhaustiveness catch bugs.
- Use **`switch`** when a branch needs to run **multiple statements** with side effects (and you don't need a return value), or when you genuinely want loose `==` matching (rare), or you're maintaining pre-8.0 code.
- Use **`if`/`elseif`** for arbitrary, unrelated boolean conditions (ranges, multiple variables) — though `match (true)` is a clean alternative when producing a value.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `break` in a `switch`.**
   Execution falls through to the next case, running code you didn't intend.
   **Fix:** Add `break` to every case (or use `match`, which has no fallthrough).

2. **Relying on `switch`'s loose `==` comparison.**
   `switch ($x) { case 0: … }` can match unexpected values, and the rules changed between PHP 7 and 8 (`0 == "abc"` flipped from `true` to `false`).
   **Fix:** Use `match` (strict `===`), or compare explicitly with `===` in `if`.

3. **Using `?:` (or truthiness) when you mean "if not provided."**
   `$config['timeout'] ?: 30` wrongly replaces a legitimate `0`, `""`, or `false` with the default.
   **Fix:** Use `??` (null coalescing), which only falls back on `null`/unset.

4. **A `match` with no matching arm and no `default`.**
   This throws `\UnhandledMatchError` at runtime and can crash a request.
   **Fix:** Either add `default =>`, or guarantee exhaustiveness (e.g. an enum subject covering all cases) — and *deliberately* omit `default` only when you want the loud failure.

5. **Putting code before `declare(strict_types=1)`.**
   It must be the first statement; otherwise you get `Fatal error: strict_types declaration must be the very first statement in the script.`
   **Fix:** Place it immediately after `<?php` on its own line.

6. **Nested ternaries without parentheses.**
   `a ? b : c ? d : e` is a fatal error in PHP 8.
   **Fix:** Parenthesize, or refactor to `match (true)` / `if`.

7. **Using `else if` (two words) in colon/Blade-style syntax.**
   `<?php if (...): ?> … <?php else if (...): ?>` is a parse error.
   **Fix:** Use `elseif` (one word) in colon syntax.

8. **`continue` inside a `switch` that's inside a loop.**
   `continue` targets the `switch` (acting like `break`) and warns — it does **not** skip the loop iteration.
   **Fix:** Use `continue 2;` to target the loop, or restructure with `if`/`match`.

9. **Forgetting that `match` arm bodies must be single expressions.**
   You cannot put multiple statements or a loop in a `match` arm. Trying to do procedural work there won't parse.
   **Fix:** Call a function/method from the arm, or use `switch`/`if` when you need a statement block.

---

## ✅ Best Practices

- Put `declare(strict_types=1);` at the top of **every** PHP file.
- Prefer **`match`** over `switch` for value selection; reserve `switch` for multi-statement, side-effectful branches.
- Use **guard clauses / early returns** to flatten nested `if`s and keep the happy path un-indented.
- Use **`??`** for "default if absent," and reserve `?:` for genuine truthiness checks.
- Keep ternaries simple and single-level; reach for `match (true)` instead of nesting them.
- Pair `match` with **enums** and omit `default` when you want exhaustiveness to enforce that every case is handled.
- In Laravel, do branching logic in **controllers/models**, not Blade — keep templates declarative with `@if` / `@switch`.
- Order `if`/`elseif` and `match (true)` arms from **most specific to least specific**; the first match wins.
- Use `require`/`require_once` for mandatory files (fail loud), `include` only for optional ones; otherwise let the **autoloader** handle class loading.
- Never use `goto`.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `switch` and `match`?**
A: `switch` is a statement using loose `==`, with fallthrough (needs `break`), and silently does nothing on no match. `match` is an expression using strict `===`, has no fallthrough, returns a value, and throws `\UnhandledMatchError` if no arm matches and there's no `default`. `match` is the safer, modern choice for value dispatch.

**Q2. How does `match` work under the hood / why is it "exhaustive"?**
A: `match` evaluates the subject once, then compares it with strict `===` against each arm's value(s) top to bottom. The first match's expression is evaluated and returned. If none match and there's no `default` arm, PHP throws `\UnhandledMatchError`. There's no implicit fallthrough because each arm is a discrete expression, not a labeled statement block — so no `break` is needed. This "fail loud" design means adding a new enum case surfaces unhandled branches at runtime/test time instead of silently misbehaving.

**Q3. Why is `0 == "foo"` `false` in PHP 8 but was `true` in PHP 7?**
A: PHP 8 changed number↔string comparison. Now, when comparing a number to a *non-numeric* string, the **number is cast to a string** (so `0 == "foo"` becomes `"0" == "foo"` → `false`). In PHP 7 the string was cast to a number (`"foo"` → `0`), making it `true`. This directly affects `switch`, which uses `==`. It's a strong argument for using `match`.

**Q4. What does `declare(strict_types=1)` do, and does it change `===`?**
A: It enforces strict scalar type checks at function call boundaries in that file — passing a `string` to an `int` parameter throws `TypeError` instead of coercing. It does **not** change `==`, `===`, `match`, or `switch` directly; those operators behave the same. Its value to control flow is indirect: it keeps wrong types from coercing into the values you branch on.

**Q5. When would you still use `switch` over `match`?**
A: When a branch must execute **multiple statements with side effects** (and you don't need a returned value), when you intentionally want loose `==` matching, or when maintaining a pre-PHP-8 codebase. `match` arms are single expressions, so heavy procedural branches read better in `switch` or `if`.

**Q6. What's the difference between `??`, `?:`, and a full ternary?**
A: A full ternary `a ? b : c` returns `b`/`c` based on `a`'s truthiness. `?:` (Elvis) is `a ?: b` → returns `a` if truthy else `b`. `??` (null coalescing) is `a ?? b` → returns `a` only if it's **set and not null** (no warning on undefined), else `b`. Use `??` for "default if missing"; it does *not* treat `0`/`""`/`false` as missing.

**Q7. Difference between `require` and `require_once`? When does it matter?**
A: Both fatal-error if the file is missing. `require_once` tracks already-included paths and **skips re-inclusion**, preventing "Cannot redeclare class/function" errors. Use `_once` for files defining classes/functions. For plain templates that should render each time, plain `include`/`require` is fine.

**Q8. Does PHP have `goto`? Should you use it?**
A: Yes, PHP has `goto` with labels, but it's discouraged — it creates unreadable, hard-to-debug control flow. Every real use case is better served by loops, functions, `break`/`continue` (with levels), or exceptions. The expected answer is "I know it exists and I avoid it."

**Q9. How does Blade's `@if` relate to raw PHP?**
A: Blade compiles `@if (...)` / `@elseif` / `@else` / `@endif` to PHP's **alternative colon syntax** (`<?php if (...): ?> … <?php endif; ?>`), caching the result in `storage/framework/views/`. Likewise `@switch` maps to `switch`. There is no `@match` directive — use `match` in the controller or inside `{{ }}`.

**Q10. How would you replace a long `if/elseif` chain that returns a value?**
A: Use `match (true)` with boolean expressions in the arms; the first arm evaluating to `true` (strict) wins, e.g. grading by score thresholds. It's more concise and forces you to provide a `default`.

**Q11. Inside a `foreach` loop you have a `switch`, and you write `continue` in a case to skip the iteration. What happens?**
A: It doesn't skip the loop iteration. A `switch` counts as a loop-level construct, so `continue` (with no level) targets the `switch` and behaves like `break`, plus PHP emits a warning: *"continue" targeting switch is equivalent to "break"*. To skip the enclosing loop iteration you must write `continue 2;`. `match` has no equivalent trap because it is an expression, not a loop-level statement.

**Q12. Can a `match` arm contain multiple statements (e.g. log something *and* return a value)?**
A: No. Each `match` arm body is a **single expression**, evaluated and returned. If you need side-effectful, multi-statement logic, call a function/method from the arm (the call is one expression) or fall back to `switch`/`if`. This single-expression rule is part of why `match` reads so cleanly.

---

## 📋 Quick Reference / Cheat Sheet

```php
// if / elseif / else
if ($a) { … } elseif ($b) { … } else { … }

// Guard clause pattern
if ($invalid) return;

// Alternative (colon) syntax — used by Blade
if ($a): … elseif ($b): … else: … endif;

// Ternary / Elvis / null coalescing
$x = $cond ? $yes : $no;     // standard ternary
$x = $a ?: $b;               // Elvis: $a if truthy, else $b
$x = $a ?? $b;               // null coalescing: $a if set & not null
$x ??= $default;             // assign only if $x is null/unset

// switch — loose (==), needs break, default optional
switch ($v) {
    case 'a': … break;
    case 'b':                // fallthrough (shared body)
    case 'c': … break;
    default:  … 
}
// Inside a loop: `continue` in a switch == break (+warning).
// Use `continue 2;` to skip the enclosing loop iteration.

// match — strict (===), expression, no break, must be exhaustive
$r = match ($v) {
    'a'        => 1,
    'b', 'c'   => 2,         // multiple values per arm
    default    => 0,         // omit to enforce exhaustiveness (throws UnhandledMatchError)
};

// match(true) — replaces if/elseif chains that yield a value
$grade = match (true) {
    $s >= 90 => 'A',
    $s >= 80 => 'B',
    default  => 'F',
};

// File flow
require   'f.php';   // fatal if missing, re-runs
include   'f.php';   // warning if missing, re-runs
require_once 'f.php';// fatal if missing, runs once
include_once 'f.php';// warning if missing, runs once

declare(strict_types=1);  // FIRST line; strict scalar type checks
```

**Falsy values:** `false`, `0`, `0.0`, `"0"`, `""`, `[]`, `null`. (Note: `"0.0"` and `" "` are **truthy**.)

**Blade equivalents:** `@if/@elseif/@else/@endif`, `@unless/@endunless`, `@switch/@case/@break/@default/@endswitch`. (No `@match`.)

---

## 🧪 Mini Exercises

1. **Refactor to `match`:** Take a `switch` that maps HTTP status codes (`200`, `201`, `404`, `500`, default) to short messages and convert it to a `match` expression. Then change the subject so a `string` `"200"` is passed and explain (in a comment) what now happens and why.

2. **Guard clauses:** Given a function `canCheckout(?Cart $cart): bool` that currently uses three levels of nested `if` (cart not null, cart not empty, user verified), rewrite it using early-return guard clauses.

3. **`match (true)` grading:** Write a function `shippingTier(float $weightKg): string` returning `"light"` (< 1), `"standard"` (1–10 inclusive), `"heavy"` (> 10) using a single `match (true)`. Add `declare(strict_types=1)` and test what happens if you call it with an `int` argument.

4. **`??` vs `?:` trap:** Given `$settings = ['retries' => 0, 'name' => '']`, write two echoes — one demonstrating where `?:` gives the wrong default and one where `??` gives the right one. Explain the difference in a comment.

5. **Blade + enum:** Create an `OrderStatus` string-backed enum with at least four cases and a `label()` method using `match`. In a Blade snippet, render a colored badge for an order's status using `@switch` on `$order->status->value`. Then describe what would happen if you added a fifth enum case but forgot to add it to `label()`.
