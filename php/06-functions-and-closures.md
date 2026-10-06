# Functions & Closures

Functions are the fundamental unit of reuse in PHP. Master them and you understand most of what makes Laravel's expressive APIs possible — routes that take closures, collection pipelines, middleware, service container bindings, and event listeners are all built on functions, closures, and callables. This module takes you from defining a basic function all the way to first-class callable syntax, `Closure::bind`, and higher-order programming.

> Target versions: **PHP 8.4** (with notes for 8.1–8.3) and **Laravel 12** (with notes for Laravel 10/11).

## **What you'll learn**

- How to define functions and declare parameter types, return types, and the special `void`/`never` return types
- Required vs. default parameters, and how **named arguments** make call sites self-documenting
- The difference between **passing by value** and **passing by reference**, and when to use each
- **Variadic functions** (`...`) and the spread/unpacking operator, plus how they interact with named arguments
- **Closures** (anonymous functions): capturing variables with `use` (by value vs. by reference), and binding `$this`
- **Arrow functions** (`fn`) with automatic by-value capture, and when to prefer them
- **First-class callable syntax** (`strlen(...)`), callables/callbacks, and writing **higher-order functions**
- Variable **scope**, the `global` and `static` keywords, recursion, and how this all maps onto everyday Laravel code

---

## 1. Why functions exist (the WHY before the HOW)

A **function** is a named, reusable block of code that takes zero or more inputs (**arguments**), does some work, and optionally returns a value. The reasons we use them are timeless:

- **DRY (Don't Repeat Yourself):** write logic once, call it many times.
- **Abstraction:** a caller uses `formatPrice($cents)` without caring *how* the formatting works.
- **Testability:** small functions with clear inputs/outputs are easy to unit test.
- **Composition:** functions can be passed to other functions, enabling pipelines (the heart of Laravel Collections).

PHP has two flavors you'll meet constantly:

1. **Named functions** — declared with `function name() {}`. Globally available once defined.
2. **Anonymous functions / closures** — values you can store in variables and pass around.

We build from the first to the second.

---

## 2. Defining functions and declaring parameters

### 2.1 The basics

```php
<?php

function greet(string $name): string
{
    return "Hello, {$name}!";
}

echo greet('Ada');
// Output: Hello, Ada!
```

Read the signature left to right: the keyword `function`, the name `greet`, a parameter `$name` **typed** as `string`, and a **return type** `string` after the colon.

> **Jargon:** a *type declaration* (sometimes called a "type hint") tells PHP what kind of value a parameter or return value must be. PHP enforces it at runtime and throws a `TypeError` on mismatch (subject to `strict_types`, below).

### 2.2 `declare(strict_types=1)` — turn off silent coercion

By default PHP is in **coercive mode**: it tries to convert scalar values to the declared type. `"3"` becomes `3`, `3.0` becomes `3`, etc. This hides bugs. In **strict mode** PHP requires the exact type (with one exception: an `int` is always accepted where a `float` is declared, because widening is safe).

```php
<?php
declare(strict_types=1); // MUST be the very first statement in the file

function double(int $n): int
{
    return $n * 2;
}

echo double(5);     // Output: 10
echo double("5");   // TypeError in strict mode (would be 10 in coercive mode)
```

> **Best practice:** put `declare(strict_types=1);` at the top of every PHP file. It catches type errors early.

### 2.3 Required parameters, default values, and optional parameters

A parameter with a **default value** is optional. If the caller omits it, the default is used.

```php
<?php

function paginate(int $page = 1, int $perPage = 15): string
{
    $offset = ($page - 1) * $perPage;
    return "LIMIT {$perPage} OFFSET {$offset}";
}

echo paginate();        // Output: LIMIT 15 OFFSET 0
echo paginate(3);       // Output: LIMIT 15 OFFSET 30
echo paginate(3, 50);   // Output: LIMIT 50 OFFSET 100
```

**Rule:** required parameters must come before optional ones (mostly). Putting a required parameter after a defaulted one is deprecated:

```php
<?php
// Deprecated since PHP 8.0: an optional param declared before a required one.
function bad(int $a = 1, int $b) {}
// Deprecated: Optional parameter $a declared before required parameter $b
// is implicitly treated as a required parameter

// Fix: reorder so required comes first, or give both defaults.
function good(int $b, int $a = 1) {}
```

> Because the "optional" parameter is implicitly treated as required anyway, the default on `$a` is meaningless here. Named arguments (Section 6) make ordering less painful, but the deprecation still applies — keep required parameters first.

> **PHP 8.4 change — implicitly nullable parameters are deprecated.** Historically, giving a typed parameter a `null` default silently made the type nullable. As of PHP 8.4 you must write the `?` (or `|null`) explicitly:
>
> ```php
> <?php
> function old(string $name = null) {}   // PHP 8.4: Deprecated — implicitly nullable
> function modern(?string $name = null) {} // Correct: explicitly nullable
> ```
>
> This is one of the most common deprecation warnings you'll hit when upgrading an older Laravel app to run on PHP 8.4.

### 2.4 Type declarations you can use

PHP supports a rich type system on parameters and return types:

```php
<?php

function example(
    int $i,                 // scalar
    float $f,
    string $s,
    bool $b,
    array $a,               // array
    ?string $maybe,         // nullable: string OR null
    int|string $either,     // union type (PHP 8.0+)
    iterable $list,         // arrays OR Traversable
    callable $cb,           // anything callable
    User $user,             // class/interface type
    mixed $anything,        // any type at all (PHP 8.0+)
): void {
    // ...
}
```

- **Nullable** `?T` is shorthand for `T|null`.
- **Union types** `A|B` (PHP 8.0+) accept any of the listed types.
- **Intersection types** `A&B` (PHP 8.1+) require a value implementing *all* listed interfaces, e.g. `Countable&Traversable`.
- **DNF types** (Disjunctive Normal Form, PHP 8.2+) combine unions and intersections with parentheses, e.g. `(Countable&Traversable)|array`.
- **Standalone `null`, `false`, and `true`** are usable as types since PHP 8.2 (before that, `false`/`true` could only appear inside a union).
- **`mixed`** accepts everything (it's the widest type). Note: `mixed` itself already includes `null`, so `?mixed` is a compile error.
- **`object`** matches any object; **`self`**, **`static`**, and **`parent`** are usable as return types for methods (`static` is the modern choice for fluent setters so subclasses return their own type).

### 2.5 Constructor property promotion (modern idiom)

When a function is a class constructor, PHP 8.0+ lets you declare and assign properties in the signature. You'll see this everywhere in Laravel 11/12 code.

```php
<?php

final class Money
{
    public function __construct(
        public readonly int $amountInCents,
        public readonly string $currency = 'USD',
    ) {}

    public function format(): string
    {
        return number_format($this->amountInCents / 100, 2) . " {$this->currency}";
    }
}

echo (new Money(1599))->format();        // Output: 15.99 USD
echo (new Money(50000, 'EUR'))->format();// Output: 500.00 EUR
```

> `readonly` (PHP 8.1+) makes a property immutable after initialization — great for value objects and DTOs.

---

## 3. Return types: `void` and `never`

### 3.1 `void` — "returns nothing useful"

A `void` function performs a side effect and returns no value. You may use a bare `return;` to exit early, but you must not return a value.

```php
<?php

function logMessage(string $message): void
{
    error_log($message);
    // return $message;  // Fatal error: void function must not return a value
}
```

Calling a `void` function still technically yields `null` if you read its result, but you should never rely on that.

### 3.2 `never` — "never returns at all" (PHP 8.1+)

A `never` return type means the function **always** throws an exception or terminates the script (e.g., `exit`/`die`). Control never returns to the caller. This helps static analyzers reason about code flow.

```php
<?php

function abort404(): never
{
    throw new RuntimeException('Not Found', 404);
    // No return is possible — and a `return;` here is a compile error.
}
```

> **Difference from `void`:** `void` returns control (with no value); `never` guarantees control will *not* return. Laravel's `abort()` and `dd()` are conceptually `never`-style helpers.

---

## 4. Passing by value vs. by reference

### 4.1 By value (the default)

By default, scalars and arrays are passed **by value** — the function receives a *copy*. Modifying the parameter inside the function does not affect the caller's variable.

```php
<?php

function addOne(int $n): void
{
    $n = $n + 1; // local copy only
}

$x = 10;
addOne($x);
echo $x; // Output: 10  (unchanged)
```

> **Important nuance — objects:** objects are *not* passed by reference, but the value passed is a **handle/identifier** to the object. So a function can mutate the object's properties (the caller sees the change), but reassigning the parameter to a new object does not affect the caller. This trips up many candidates in interviews.

```php
<?php

class Box { public int $value = 0; }

function mutate(Box $b): void   { $b->value = 99; }   // affects caller
function reassign(Box $b): void { $b = new Box(); }   // does NOT affect caller

$box = new Box();
mutate($box);
echo $box->value;   // Output: 99

reassign($box);
echo $box->value;   // Output: 99  (still the mutated one)
```

### 4.2 By reference with `&`

Prefix the parameter with `&` to pass **by reference**. Now the function operates on the caller's actual variable.

```php
<?php

function addOneRef(int &$n): void
{
    $n = $n + 1;
}

$x = 10;
addOneRef($x);
echo $x; // Output: 11  (changed)
```

A real-world example is PHP's own `sort()`, which sorts the array in place:

```php
<?php
$nums = [3, 1, 2];
sort($nums);            // sort() takes its array by reference
print_r($nums);
// Output: Array ( [0] => 1 [1] => 2 [2] => 3 )
```

> **Gotcha:** you do *not* repeat the `&` at the call site — `addOneRef($x)`, not `addOneRef(&$x)`. Call-time pass-by-reference was removed in PHP 5.4.

> **Best practice:** prefer returning new values over mutating arguments by reference. References make code harder to reason about and to test. Reserve `&` for performance-critical paths over huge arrays, or when you're implementing an API that genuinely needs in-place mutation.

### 4.3 Returning by reference (brief, advanced)

You can also return a reference using `function &name()` and `$x = &fn()`. It's rare and usually a code smell outside of building container-like data structures.

```php
<?php

class Registry
{
    private array $items = ['count' => 0];

    public function &get(string $key)
    {
        return $this->items[$key]; // note the & on the method
    }
}

$r = new Registry();
$ref = &$r->get('count');
$ref = 5;                 // mutates the array element inside $r through the reference
echo $r->get('count');    // Output: 5
```

> Both the function definition (`&get`) and the assignment (`= &`) need the ampersand for a true reference return. Use sparingly.

---

## 5. Variadic functions and argument unpacking (spread)

### 5.1 Variadic parameters: collect "the rest" into an array

A **variadic** parameter uses `...` and gathers all remaining arguments into an array. It must be the **last** parameter.

```php
<?php

function sum(int ...$numbers): int
{
    return array_sum($numbers); // $numbers is a plain array of ints
}

echo sum(1, 2, 3);          // Output: 6
echo sum(10, 20, 30, 40);   // Output: 100
echo sum();                 // Output: 0
```

You can have fixed parameters before the variadic:

```php
<?php

function join_with(string $glue, string ...$parts): string
{
    return implode($glue, $parts);
}

echo join_with('-', 'a', 'b', 'c'); // Output: a-b-c
```

### 5.2 Argument unpacking: spread an array INTO a call

The same `...` operator, used at the **call site**, unpacks an array (or Traversable) into individual arguments.

```php
<?php

function point(int $x, int $y, int $z): string
{
    return "({$x}, {$y}, {$z})";
}

$coords = [1, 2, 3];
echo point(...$coords); // Output: (1, 2, 3)
```

You can unpack **string-keyed** arrays as named arguments (PHP 8.1+):

```php
<?php
$args = ['z' => 3, 'x' => 1, 'y' => 2];
echo point(...$args);   // Output: (1, 2, 3)  — keys map to parameter names
```

> **Pre-8.1 note:** unpacking string-keyed arrays into named arguments works from PHP 8.1. Before that, unpacking only worked positionally with integer keys.

### 5.3 How named args and variadics interact

This is a favorite interview detail. When you pass **named** arguments that don't match any declared parameter, they are collected into the variadic parameter **with their keys preserved**.

```php
<?php

function tag(string $name, ...$attributes): string
{
    $attrs = '';
    foreach ($attributes as $key => $value) {
        $attrs .= " {$key}=\"{$value}\"";
    }
    return "<{$name}{$attrs}>";
}

echo tag('a', href: '/home', class: 'btn');
// Output: <a href="/home" class="btn">

echo tag('img', src: 'x.png', alt: 'X');
// Output: <img src="x.png" alt="X">
```

> **⚠️ Security note (XSS):** this `tag()` builder interpolates attribute values *raw* — it's fine for the fixed literals above, but **never** feed it user-controlled input as written. A value like `" onmouseover="steal()` would break out of the attribute and inject script. In real code, escape with `htmlspecialchars($value, ENT_QUOTES)` (or in Blade, rely on `{{ }}` auto-escaping / `e()`), and use the URL-aware escaping for `href`/`src`. The example is about argument mechanics, not safe HTML generation.

So a variadic parameter can hold a mix: positional extras land with integer keys, named extras land with string keys.

---

## 6. Named arguments (PHP 8.0+)

**Named arguments** let you pass arguments by parameter name instead of position. They make call sites self-documenting and let you skip optional parameters you don't care about.

```php
<?php

function createUser(
    string $name,
    bool $admin = false,
    bool $verified = false,
    string $locale = 'en',
): string {
    return "{$name} admin={$admin} verified={$verified} locale={$locale}";
}

// Skip the middle defaults; only set what you need:
echo createUser('Sam', verified: true);
// Output: Sam admin= verified=1 locale=en   (note: false prints as empty string)

// Mix positional then named (positional MUST come first):
echo createUser('Ada', locale: 'fr');
// Output: Ada admin= verified= locale=fr
```

Rules and gotchas:

- **Positional arguments must precede named ones.** `f(1, x: 2)` is fine; `f(x: 2, 1)` is a fatal error.
- You **cannot** pass the same parameter both positionally and by name.
- Named arguments are tied to **parameter names**, so renaming a parameter is a *breaking change* for callers. Library authors should treat parameter names as part of the public API.

```php
<?php
// Combining with the spread of a string-keyed array (PHP 8.1+):
$opts = ['admin' => true, 'locale' => 'de'];
echo createUser('Lee', ...$opts);
// Output: Lee admin=1 verified= locale=de
```

---

## 7. Scope, `global`, and `static`

### 7.1 Variable scope

PHP functions have their **own local scope**. Variables outside the function are *not* visible inside it (unlike, say, JavaScript closures over outer variables — PHP requires explicit capture, which we'll cover with closures).

```php
<?php
$message = 'outside';

function show(): void
{
    // echo $message; // Warning: Undefined variable $message
    echo 'inside';
}
show(); // Output: inside
```

### 7.2 The `global` keyword

`global` pulls a variable from the global scope into the function. Use it rarely — global mutable state is a maintenance hazard and is essentially banned in clean Laravel code (use the service container or dependency injection instead).

```php
<?php
$counter = 0;

function bump(): void
{
    global $counter; // bind to the global $counter
    $counter++;
}
bump(); bump();
echo $counter; // Output: 2
```

> Equivalent (and equally discouraged) is `$GLOBALS['counter']`.

### 7.3 The `static` keyword inside functions

A **static** local variable keeps its value between calls but stays private to the function. It's initialized once.

```php
<?php

function nextId(): int
{
    static $id = 0; // initialized only on the first call
    return ++$id;
}

echo nextId(); // Output: 1
echo nextId(); // Output: 2
echo nextId(); // Output: 3
```

This is handy for memoization or sequence generation without a global. (Don't confuse this with `static` properties/methods on classes — same keyword, different context.)

---

## 8. Anonymous functions (closures) and `use`

An **anonymous function** is a function with no name that you assign to a variable or pass as an argument. In PHP, every anonymous function is an instance of the built-in `Closure` class.

```php
<?php

$square = function (int $n): int {
    return $n * $n;
};

echo $square(5); // Output: 25
var_dump($square instanceof Closure); // Output: bool(true)
```

### 8.1 Capturing variables with `use` (by value)

Unlike named functions, an anonymous function can **capture** variables from the enclosing scope — but only the ones you explicitly list in `use`. By default capture is **by value** (a snapshot at definition time).

```php
<?php

$tax = 0.2;

$addTax = function (float $price) use ($tax): float {
    return $price * (1 + $tax);
};

$tax = 0.5;            // changed AFTER the closure was created
echo $addTax(100);     // Output: 120  (captured the old value 0.2)
```

### 8.2 Capturing by reference with `use (&$var)`

Add `&` to capture **by reference**, so the closure sees later changes and can write back to the outer variable.

```php
<?php

$total = 0;

$accumulate = function (int $n) use (&$total): void {
    $total += $n;
};

$accumulate(5);
$accumulate(10);
echo $total; // Output: 15
```

A classic use is building a list of callbacks where each must remember its own data — a place where by-value capture is what you want:

```php
<?php
$callbacks = [];
for ($i = 1; $i <= 3; $i++) {
    $callbacks[] = fn() => $i; // arrow fn captures $i BY VALUE each iteration
}
echo $callbacks[0]() . $callbacks[1]() . $callbacks[2](); // Output: 123
```

> If you captured `$i` by reference in a loop, all three closures would share one variable and print `444` after the loop ends. By-value capture (the default, and what arrow functions always do) avoids that.

### 8.3 Closures in Laravel

You meet closures the moment you open `routes/web.php`:

```php
<?php
// routes/web.php (Laravel 12)
use Illuminate\Support\Facades\Route;

Route::get('/ping', function () {
    return response()->json(['pong' => true]);
});
```

And in Eloquent query scopes, transactions, collection pipelines, etc.:

```php
<?php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    // everything here runs in one transaction; throw to roll back
});

$names = collect([1, 2, 3, 4])
    ->filter(fn ($n) => $n % 2 === 0) // arrow fn closure
    ->map(fn ($n) => "#{$n}")
    ->values()
    ->all();
// $names === ['#2', '#4']
```

---

## 9. Binding `$this`: `Closure::bind` and `bindTo`

A closure can be bound to an object, giving its body access to `$this` (and even the object's private members, depending on the bound *scope*). Laravel uses this extensively for its **macro** system (`Collection::macro`, `Str::macro`, etc.), view composers, and Blade directives — anywhere a user-supplied closure must run as though it were a method of the framework's class.

> **Note:** plain *route* closures (`Route::get('/', function () { ... })`) are **not** bound to a useful `$this`; relying on `$this` inside a route closure won't work the way it does in a controller method. Binding is an explicit, deliberate act — it doesn't happen automatically just because a closure is passed to a framework method.

```php
<?php

class Counter
{
    private int $count = 0;
}

$increment = function (): int {
    return ++$this->count; // needs $this AND access to a private property
};

// bindTo($newThis, $scope): returns a NEW closure bound to the object.
// Passing the class name (or object) as scope grants private/protected access.
$bound = Closure::bind($increment, new Counter(), Counter::class);

echo $bound(); // Output: 1
echo $bound(); // Output: 2
```

`Closure::bind($closure, $object, $scope)` is the static form; `$closure->bindTo($object, $scope)` is the instance form — they do the same thing.

There is also `Closure::call($object, ...$args)` which binds and invokes in one step:

```php
<?php
class Point { private int $x = 7; }

$getX = function () { return $this->x; };
echo $getX->call(new Point()); // Output: 7
```

> **Why it matters in Laravel:** the `Collection::macro()` / `Str::macro()` system explicitly binds your closure to the target instance, so inside a macro body `$this` refers to the collection (or string builder) and you can call its other methods:
>
> ```php
> <?php
> use Illuminate\Support\Collection;
>
> Collection::macro('sumWhere', function (callable $filter) {
>     // $this is the Collection — bound by Laravel via bindTo()
>     return $this->filter($filter)->sum();
> });
>
> $total = collect([1, 2, 3, 4])->sumWhere(fn ($n) => $n % 2 === 0); // 6
> ```

---

## 10. Arrow functions (PHP 7.4+)

**Arrow functions** are a concise closure syntax with **automatic by-value capture** of any outer variables they reference. No `use` clause is needed.

```php
<?php

$multiplier = 3;

$triple = fn (int $n): int => $n * $multiplier; // captures $multiplier by value, automatically

echo $triple(4); // Output: 12
```

Key facts:

- Syntax: `fn (params) => expression`. The body is a **single expression**; its value is returned implicitly (no `return`, no braces, no statements).
- Capture is **always by value** and **automatic** — you cannot capture by reference, and you cannot list a `use` clause.
- They can be nested, and each level captures what it references.

```php
<?php
$base = 10;
$adder = fn ($x) => fn ($y) => $base + $x + $y; // currying via nested arrow fns
echo $adder(1)(2); // Output: 13
```

**When to choose which:**

| Use an **arrow function** when | Use a full **closure** when |
| --- | --- |
| The body is one expression | You need multiple statements |
| You want concise inline callbacks (`map`, `filter`) | You need `use (&$ref)` by-reference capture |
| Automatic capture is convenient | You want explicit control over what's captured |

---

## 11. First-class callable syntax (PHP 8.1+)

You can turn any function, method, or static method into a `Closure` using `(...)`. This is the **first-class callable syntax** and it's type-safe, IDE-friendly, and refactor-safe — a modern replacement for string/array callable forms.

```php
<?php

$len = strlen(...);            // a Closure wrapping strlen
echo $len('hello');            // Output: 5

$nums = ['3', '1', '2'];
$ints = array_map(intval(...), $nums);
print_r($ints);
// Output: Array ( [0] => 3 [1] => 1 [2] => 2 )
```

It works on methods too:

```php
<?php
class Greeter
{
    public function hello(string $name): string { return "Hi {$name}"; }
    public static function bye(string $name): string { return "Bye {$name}"; }
}

$g = new Greeter();
$instanceFn = $g->hello(...);       // bound to $g
$staticFn   = Greeter::bye(...);

echo $instanceFn('Ada'); // Output: Hi Ada
echo $staticFn('Lee');   // Output: Bye Lee
```

> **Compare the old forms** — all of these are "callables" PHP understands, but the `(...)` form is preferred because it's checked at compile time:
>
> | Form | Example |
> | --- | --- |
> | String (function name) | `'strlen'` |
> | String (static method) | `'Greeter::bye'` |
> | Array (instance method) | `[$g, 'hello']` |
> | Array (static method) | `[Greeter::class, 'bye']` |
> | First-class callable | `strlen(...)`, `$g->hello(...)` |
> | Invokable object | any object with `__invoke()` |

---

## 12. Callables, callbacks, and higher-order functions

A **callable** is anything PHP can invoke: a closure, an arrow function, a function name string, a `[$object, 'method']` array, a static-method string, or an object implementing `__invoke()`. A **callback** is just a callable you pass to another function so it can be called later.

```php
<?php

class Multiplier
{
    public function __construct(private int $factor) {}
    public function __invoke(int $n): int { return $n * $this->factor; } // makes the object callable
}

$double = new Multiplier(2);
echo $double(21);                 // Output: 42  (object invoked like a function)
print_r(array_map($double, [1, 2, 3]));
// Output: Array ( [0] => 2 [1] => 4 [2] => 6 )
```

### 12.1 Higher-order functions

A **higher-order function** is one that takes a function as an argument and/or returns a function. PHP's standard library is full of them: `array_map`, `array_filter`, `array_reduce`, `usort`, `array_walk`.

```php
<?php

$numbers = [1, 2, 3, 4, 5, 6];

$evens   = array_filter($numbers, fn ($n) => $n % 2 === 0); // keep matching
$squares = array_map(fn ($n) => $n * $n, $numbers);         // transform each
$total   = array_reduce($numbers, fn ($carry, $n) => $carry + $n, 0); // fold to one value

print_r(array_values($evens)); // Output: Array ( [0] => 2 [1] => 4 [2] => 6 )
print_r($squares);             // Output: Array ( [0] => 1 [1] => 4 ... [5] => 36 )
echo $total;                   // Output: 21
```

> **`array_filter` gotcha:** it **preserves keys**. After filtering you often want `array_values()` to reindex. (Laravel's `Collection` solves this with `->values()`.)

A function that *returns* a function (a factory / closure builder):

```php
<?php

function makeMultiplier(int $factor): callable
{
    return fn (int $n): int => $n * $factor; // arrow fn captures $factor
}

$triple = makeMultiplier(3);
echo $triple(10); // Output: 30
```

### 12.2 The Laravel connection: pipelines and `Collection`

Laravel's `Collection` is essentially a fluent wrapper around higher-order functions, which is why understanding callbacks pays off immediately:

```php
<?php
use Illuminate\Support\Collection;

$report = collect([
    ['name' => 'A', 'amount' => 30],
    ['name' => 'B', 'amount' => 50],
    ['name' => 'C', 'amount' => 20],
])
    ->filter(fn ($row) => $row['amount'] >= 25)
    ->sortByDesc('amount')
    ->map(fn ($row) => "{$row['name']}: {$row['amount']}")
    ->values()
    ->all();

print_r($report);
// Output: Array ( [0] => B: 50 [1] => A: 30 )
```

Collections also support **higher-order messages**, a sugar that uses first-class-callable-like proxies:

```php
<?php
// Instead of ->each(fn ($u) => $u->markAsVerified())
$users->each->markAsVerified();
// Instead of ->sum(fn ($o) => $o->total)
$revenue = $orders->sum->total;
```

---

## 13. Recursion

A **recursive** function calls itself, reducing the problem toward a **base case** that stops the recursion. Always define the base case first to avoid infinite loops (and a stack overflow / `Maximum function nesting` error).

```php
<?php

function factorial(int $n): int
{
    if ($n <= 1) {          // base case
        return 1;
    }
    return $n * factorial($n - 1); // recursive case
}

echo factorial(5); // Output: 120
```

Recursion shines for tree-shaped data — for example, rendering a nested category menu (a common Laravel task):

```php
<?php

function renderTree(array $nodes, int $depth = 0): string
{
    $out = '';
    foreach ($nodes as $node) {
        $out .= str_repeat('  ', $depth) . "- {$node['name']}\n";
        if (!empty($node['children'])) {
            $out .= renderTree($node['children'], $depth + 1); // recurse into children
        }
    }
    return $out;
}

echo renderTree([
    ['name' => 'Electronics', 'children' => [
        ['name' => 'Phones', 'children' => []],
        ['name' => 'Laptops', 'children' => []],
    ]],
    ['name' => 'Books', 'children' => []],
]);
/* Output:
- Electronics
  - Phones
  - Laptops
- Books
*/
```

> **Recursion vs. iteration:** PHP does not optimize tail calls, so deep recursion can exhaust the call stack. For very deep or unbounded structures prefer an explicit stack/loop. For modest tree depths, recursion is clearest.

---

## ⚠️ Common Mistakes & Gotchas

1. **Expecting closures to auto-capture (like JavaScript).**
   PHP closures need an explicit `use` clause; only arrow functions auto-capture.
   ```php
   $tax = 0.2;
   $f = function ($p) { return $p * $tax; }; // BUG: $tax is undefined inside
   ```
   **Fix:** add `use ($tax)`, or use an arrow function `fn ($p) => $p * $tax`.

2. **By-reference capture in loops surprising you.**
   ```php
   $cbs = [];
   foreach ([1, 2, 3] as $v) {
       $cbs[] = function () use (&$v) { return $v; }; // all return 3!
   }
   ```
   **Fix:** capture by value (`use ($v)`) or use an arrow function — each iteration snapshots `$v`.

3. **Confusing "objects are references" with pass-by-reference.**
   Reassigning an object parameter inside a function does *not* change the caller's variable (Section 4.1). Mutating its properties *does*.
   **Fix:** if you need to swap the object itself, return the new one, or declare the parameter `&$obj`.

4. **Forgetting `array_filter` preserves keys.**
   ```php
   $odds = array_filter([1,2,3,4], fn ($n) => $n % 2); // keys 0 and 2 kept
   json_encode($odds); // {"0":1,"2":3} — an object, not [1,3]!
   ```
   **Fix:** wrap in `array_values()` (or `->values()` on a Collection) when you need a clean list.

5. **Putting a required parameter after an optional one.**
   Deprecated since PHP 8.0 and confusing for callers.
   **Fix:** keep required parameters first; rely on named arguments to skip optionals.

6. **Using string-keyed array unpacking on PHP < 8.1.**
   `point(...['x' => 1, 'y' => 2])` only maps keys to parameter names from PHP 8.1.
   **Fix:** upgrade, or build a positional array in the right order.

7. **Trying to put statements in an arrow function.**
   `fn ($x) => { $y = $x + 1; return $y; }` is a syntax error — arrow bodies are single expressions.
   **Fix:** use a full `function () { ... }` closure when you need multiple statements.

8. **Relying on global mutable state via `global`.**
   It makes code untestable and order-dependent.
   **Fix:** pass dependencies as arguments, or in Laravel use the service container / dependency injection.

9. **Implicitly nullable parameters on PHP 8.4.**
   ```php
   function f(string $name = null) {} // PHP 8.4: Deprecated — implicit nullable
   ```
   **Fix:** make nullability explicit: `function f(?string $name = null) {}`. This is the single most common deprecation when moving a codebase to PHP 8.4.

10. **Assuming a closure passed to a framework method is bound to `$this`.**
    A route closure (`Route::get('/', fn () => ...)`) is **not** bound to a controller instance. Only mechanisms that explicitly call `bindTo()`/`Closure::bind` (like macros) give your closure a meaningful `$this`.
    **Fix:** don't rely on `$this` inside ordinary callbacks; inject what you need or bind explicitly.

11. **Mutating an array while iterating it by reference (`foreach ($arr as &$v)`) and forgetting to `unset($v)`.**
    The reference outlives the loop, so a later write to `$v` silently corrupts the last element.
    **Fix:** `unset($v);` right after the loop, or avoid `&` in `foreach` altogether.

---

## ✅ Best Practices

- Start every file with `declare(strict_types=1);` and **type every parameter and return value**.
- Prefer **arrow functions** for one-liner callbacks; use full closures only when you need multiple statements or by-reference capture.
- Prefer **first-class callable syntax** (`strlen(...)`, `$svc->handle(...)`) over string/array callables — it's compile-time checked and refactor-safe.
- Default to **pass-by-value**; reach for `&` references only with a clear, documented reason.
- Use **named arguments** to make boolean-heavy calls readable (`createUser('Sam', verified: true)`).
- Keep functions **small and single-purpose**; if a function needs many flags, consider a value object / DTO instead.
- Use **`void`** for side-effecting functions and **`never`** for functions that always throw or exit — it documents intent and helps static analysis.
- Avoid `global`; inject dependencies. In Laravel, let the container resolve them.
- Treat **parameter names as public API** if external code uses named arguments — renaming them is a breaking change.
- On **PHP 8.4**, always write nullable types explicitly (`?string $x = null`); implicit nullability via a `null` default is deprecated.
- **Escape output** when a function builds HTML/SQL from its arguments — function mechanics (variadics, named args) don't make raw interpolation safe.
- For tree/graph traversal use recursion only when depth is bounded; otherwise iterate with an explicit stack.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between an anonymous function and an arrow function in PHP?**
Both are `Closure` instances. An anonymous function (`function () use (...) {}`) requires an explicit `use` clause and can have a multi-statement body and by-reference capture. An arrow function (`fn () => expr`) automatically captures referenced outer variables **by value**, has a single-expression body, and cannot capture by reference or use a `use` clause.

**Q2. By value vs. by reference — and are objects passed by reference?**
By value passes a copy; by reference (`&`) lets the function mutate the caller's variable. Objects are *not* passed by reference — what's passed is a copy of the **object handle**. So mutating the object's properties affects the caller, but reassigning the parameter to a new object does not.

**Q3. How do named arguments interact with variadic parameters? (under the hood)**
Named arguments not matching a declared parameter are collected into the variadic `...$args` array **with their string keys preserved**, while positional extras get integer keys. Internally PHP resolves positional arguments to their parameter slots first, then maps named arguments to remaining named parameters, and finally funnels leftovers into the variadic. Positional arguments must always precede named ones in a call.

**Q4. What's the difference between `void` and `never` return types?**
`void` means the function returns no usable value but control *does* return to the caller. `never` (PHP 8.1+) means the function never returns at all — it always throws or exits — so any code after a call to it is unreachable. `never` is a "bottom type" useful for static analysis.

**Q5. Explain `Closure::bind` / `bindTo`. Why does Laravel use it?**
They return a *new* closure bound to a given object (so `$this` works inside) and an optional scope (so the closure can access private/protected members of that class). Laravel uses binding for Collection/Str/etc. **macros**, view composers, and other extension points so a user-supplied closure can act as if it were a method of the target class.

**Q6. What is a higher-order function? Give PHP examples.**
A function that takes a function as an argument and/or returns one. Examples: `array_map`, `array_filter`, `array_reduce`, `usort`. Returning a function is how factories like `makeMultiplier()` work. Laravel Collections are a fluent API built on this idea.

**Q7. What does the first-class callable syntax `f(...)` do, and why prefer it?**
It produces a `Closure` wrapping the function/method without calling it. Prefer it over `'f'` strings or `[$obj, 'm']` arrays because it's resolved at compile time, supports static analysis and IDE refactoring, and binds the instance automatically.

**Q8. How does `static` inside a function differ from a local variable?**
A `static` local variable is initialized once and **retains its value across calls** to that function, but remains private to it. A normal local variable is re-created on every call. Useful for counters, caches, and memoization without globals.

**Q9. Why does capturing a loop variable by reference inside a closure cause bugs?**
By-reference capture means all closures share the same variable; after the loop they all see its final value. By-value capture (and arrow functions) snapshot the value at definition time, giving each closure its own copy.

**Q10. When would you pass by reference in real code?**
Rarely — e.g., in-place sorting of a large array to avoid copying, or implementing an API like `preg_match($pattern, $subject, $matches)` where `$matches` is filled by reference. Default to returning values; references reduce readability and testability.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Definition & types ---
declare(strict_types=1);
function f(int $a, string $b = 'x', ?float $c = null): string { /* ... */ }
function g(): void   { /* side effect, returns nothing */ }
function h(): never  { throw new Exception(); } // PHP 8.1+

// --- Parameters ---
function req(int $a, int $b = 2) {}     // required first, optional after
function uni(int|string $x) {}          // union type (8.0+)
function inter(Countable&Traversable $x) {} // intersection (8.1+)

// --- By reference ---
function inc(int &$n): void { $n++; }   // caller's var changes
inc($x);                                // no & at call site

// --- Variadics & spread ---
function sum(int ...$nums): int { return array_sum($nums); }
sum(...[1, 2, 3]);                      // unpack array into args (positional)
function pt(int $x, int $y) {}
pt(...['x' => 1, 'y' => 2]);            // string-keyed unpack -> named args (8.1+)

// --- Named arguments (8.0+) ---
createUser('Sam', verified: true);      // positional then named

// --- Closures ---
$byVal = function ($p) use ($tax) { return $p * $tax; };
$byRef = function ($n) use (&$total) { $total += $n; };

// --- Arrow functions (7.4+): auto by-value capture, single expression ---
$fn = fn ($x) => $x * $multiplier;

// --- Binding $this ---
$bound = Closure::bind($cl, $obj, MyClass::class);
$bound = $cl->bindTo($obj, MyClass::class);
$result = $cl->call($obj, ...$args);

// --- First-class callable (8.1+) ---
$len = strlen(...);
$m   = $obj->method(...);
$s   = MyClass::staticMethod(...);

// --- Higher-order built-ins ---
array_map($fn, $arr);
array_filter($arr, $fn);                // PRESERVES KEYS -> often array_values()
array_reduce($arr, $fn, $initial);
usort($arr, $compareFn);                // sorts in place

// --- Scope helpers ---
function c(): int { static $n = 0; return ++$n; } // persists across calls
function bump(): void { global $x; $x++; }         // avoid in practice

// --- Callable forms ---
'strlen'                 // function name string
'Cls::method'            // static method string
[$obj, 'method']         // instance method array
[Cls::class, 'method']   // static method array
$obj                     // object with __invoke()
strlen(...)              // first-class callable (preferred)
```

---

## 🧪 Mini Exercises

1. **Compose-it.** Write `compose(callable ...$fns): callable` that returns a new function applying the given functions right-to-left (so `compose($a, $b)($x) === $a($b($x))`). Test it with `compose(fn ($n) => $n + 1, fn ($n) => $n * 2)` applied to `5` and explain the result.

2. **Memoize.** Write a function `memoize(callable $fn): callable` that caches results keyed by the serialized arguments, so repeated calls with the same inputs skip recomputation. Use a `static` variable or a by-reference captured array for the cache. Demonstrate it on a slow `fibonacci` function.

3. **Tag builder with named/variadic.** Implement `el(string $tag, string $content = '', ...$attrs): string` that produces `<tag a="1" b="2">content</tag>`. Call it three ways: positional only, with named attributes, and by unpacking a string-keyed attributes array.

4. **Reference vs. value.** Write two functions, `appendCopy(array $a)` and `appendRef(array &$a)`, that each push `99` onto the array. Call both on the same array and print the array after each to demonstrate the difference. Then add an object example showing property mutation vs. reassignment.

5. **Macro binding.** Outside of Laravel, simulate a macro: create a class `Box` with a private `$items` array, then use `Closure::bind` to attach an external closure that can read `$this->items`. Show that without passing the scope argument the closure cannot access the private property.
