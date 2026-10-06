# Coding Challenges (with Solutions) — PHP & Laravel

A curated set of 20+ practical coding problems you are likely to face in a backend/full-stack interview, each with a problem statement, an approach, idiomatic PHP 8.4 / Laravel 12 solution code, expected output, and a complexity/notes section. The goal is not just to memorize answers but to learn the *reasoning patterns* so you can adapt under pressure.

> **What you'll learn**
> - How to reason about and articulate **time/space complexity** (Big-O) for everyday problems.
> - Idiomatic, modern PHP 8.4 solutions using **enums, `match`, constructor promotion, generators, named arguments, and the `::class` constant**.
> - The difference between **recursion, iteration, and generators** — and when each is the right tool.
> - **Multibyte-safe** string handling (so your code doesn't break on emoji or accented characters).
> - How to design small systems: a **router, event dispatcher, and rate limiter** from scratch.
> - Laravel-specific data problems: **top-N-per-group, N+1 fixes, Form Requests, queued batch jobs, and feature tests**.
> - How to talk through trade-offs out loud the way interviewers want to hear.

---

## How to Approach Any Coding Challenge

Before the problems, internalize a repeatable framework. Interviewers grade your *process* as much as your answer.

1. **Clarify** — restate the problem, ask about edge cases (empty input, duplicates, Unicode, negatives, huge input).
2. **Examples** — write 1–2 input→output pairs by hand. This catches misunderstandings early.
3. **Approach first, code second** — say the algorithm out loud before typing. State the complexity you expect.
4. **Code cleanly** — small functions, meaningful names, type declarations.
5. **Test** — walk through your examples, then edge cases.
6. **Analyze** — state final time and space complexity, and mention how you'd scale it.

**Big-O cheat:** O(1) constant, O(log n) halving, O(n) linear, O(n log n) good sorts, O(n²) nested loops, O(2ⁿ) naive recursion. A hash map (PHP associative array) gives **average O(1)** lookups and is the single most common optimization in these problems.

All examples assume `declare(strict_types=1);` at the top of the file unless noted. Strict types make PHP reject silent type coercion (e.g., passing `"5"` where an `int` is required), which is what you want in production and in interviews.

---

## Part 1 — Strings

### Problem 1: Reverse a String Without `strrev` (Multibyte-Aware)

**Problem.** Reverse a string. You may not use `strrev`. It must work for multibyte text like `"café"` or `"héllo 🌍"`.

**Approach.** `strrev` works on *bytes*, not *characters*. A UTF-8 character can be 1–4 bytes, so reversing bytes corrupts multibyte text (it produces invalid UTF-8). The correct approach is to split the string into an array of **grapheme/characters** using `mb_str_split` (PHP 7.4+), reverse that array, and join it back.

```php
<?php
declare(strict_types=1);

function reverseString(string $str): string
{
    // mb_str_split splits on character boundaries, not byte boundaries.
    $chars = mb_str_split($str);          // ["c", "a", "f", "é"]
    return implode('', array_reverse($chars));
}

echo reverseString("café"), PHP_EOL;
echo reverseString("hello"), PHP_EOL;
// Output:
// éfac
// olleh
```

A naive byte-based version for comparison (do NOT use for Unicode):

```php
<?php
// WRONG for multibyte: reverses bytes, corrupting "é" (2 bytes).
function reverseBytes(string $s): string
{
    $out = '';
    for ($i = strlen($s) - 1; $i >= 0; $i--) {
        $out .= $s[$i];
    }
    return $out;
}
// reverseBytes("café") => broken UTF-8: the 2 bytes of "é" (0xC3 0xA9) get
// reversed to 0xA9 0xC3, an invalid sequence that renders as mojibake (the
// exact glyph shown depends on your terminal/editor).
```

> **Note on emoji with skin tones / flags:** Some "characters" are actually multiple Unicode code points joined (grapheme clusters). For *fully* correct reversal of those, use the `grapheme_*` functions from the `intl` extension (`grapheme_str_split` exists in PHP 8.4+). For interviews, `mb_str_split` is the expected answer.

**Complexity.** O(n) time, O(n) space (we build an array of characters).

---

### Problem 2: Palindrome Check

**Problem.** Return `true` if a string reads the same forwards and backwards, ignoring case and non-alphanumeric characters. Multibyte-aware.

**Approach.** Normalize first: lowercase and strip everything that isn't a letter or digit. Then compare with the reversed version, or use a **two-pointer** walk (more efficient — no second string allocation).

```php
<?php
declare(strict_types=1);

function isPalindrome(string $str): bool
{
    // mb_strtolower + a Unicode-aware regex (\p{L} letters, \p{N} numbers).
    $normalized = preg_replace('/[^\p{L}\p{N}]/u', '', mb_strtolower($str));
    $chars = mb_str_split($normalized);

    $left = 0;
    $right = count($chars) - 1;
    while ($left < $right) {
        if ($chars[$left] !== $chars[$right]) {
            return false;
        }
        $left++;
        $right--;
    }
    return true;
}

var_dump(isPalindrome("A man, a plan, a canal: Panama")); // bool(true)
var_dump(isPalindrome("race a car"));                      // bool(false)
var_dump(isPalindrome("Ële vele"));                        // bool(false)
```

The `/u` modifier tells PCRE to treat the pattern and subject as UTF-8. Without it, `\p{L}` is meaningless and multibyte bytes get mangled.

**Complexity.** O(n) time, O(n) space for the normalized copy.

---

### Problem 3: Anagram Check

**Problem.** Are two strings anagrams (same characters, same counts, order irrelevant)? E.g. `"listen"` / `"silent"`.

**Approach.** Two strings are anagrams iff their character-frequency maps are equal. Two common methods:
1. **Sort both** and compare — O(n log n), simple.
2. **Count frequencies** in a hash map — O(n), the optimal answer.

```php
<?php
declare(strict_types=1);

function isAnagram(string $a, string $b): bool
{
    $a = mb_strtolower($a);
    $b = mb_strtolower($b);

    if (mb_strlen($a) !== mb_strlen($b)) {
        return false; // quick reject — different lengths can't be anagrams
    }

    $counts = [];
    foreach (mb_str_split($a) as $ch) {
        $counts[$ch] = ($counts[$ch] ?? 0) + 1;
    }
    foreach (mb_str_split($b) as $ch) {
        // If a char is missing or already depleted, not an anagram.
        if (empty($counts[$ch])) {
            return false;
        }
        $counts[$ch]--;
    }
    return true; // all counts balanced to zero
}

var_dump(isAnagram("listen", "silent")); // bool(true)
var_dump(isAnagram("hello", "world"));   // bool(false)
```

**Complexity.** Counting approach: O(n) time, O(k) space where k is the alphabet size. Sorting approach: O(n log n).

---

## Part 2 — Arrays

### Problem 4: FizzBuzz

**Problem.** Print numbers 1..n. For multiples of 3 print `"Fizz"`, multiples of 5 `"Buzz"`, multiples of both `"FizzBuzz"`.

**Approach.** Classic warm-up. The trick interviewers watch for: **check the combined condition (15) first**, or build the string by concatenation so you never duplicate logic. `match(true)` is the cleanest modern form.

```php
<?php
declare(strict_types=1);

function fizzBuzz(int $n): array
{
    $out = [];
    for ($i = 1; $i <= $n; $i++) {
        $out[] = match (true) {
            $i % 15 === 0 => 'FizzBuzz',
            $i % 3 === 0  => 'Fizz',
            $i % 5 === 0  => 'Buzz',
            default       => (string) $i,
        };
    }
    return $out;
}

print_r(fizzBuzz(15));
// Output: Array ( [0] => 1 [1] => 2 [2] => Fizz [3] => 4 [4] => Buzz
//   [5] => Fizz [6] => 7 [7] => 8 [8] => Fizz [9] => Buzz [10] => 11
//   [11] => Fizz [12] => 13 [13] => 14 [14] => FizzBuzz )
```

`match` uses strict (`===`) comparison and throws `\UnhandledMatchError` if nothing matches — but our `default` arm guarantees a result. Compared to `switch`, `match` is an expression (returns a value) and has no fall-through bugs.

**Complexity.** O(n) time, O(n) space (or O(1) extra if you `echo` directly).

---

### Problem 5: Find Duplicates in an Array

**Problem.** Given an array, return the values that appear more than once.

**Approach.** Use a frequency map (associative array), then filter to counts > 1. `array_count_values` does the counting for you in one call (works on int/string values).

```php
<?php
declare(strict_types=1);

function findDuplicates(array $items): array
{
    $counts = array_count_values($items); // ['a'=>2, 'b'=>1, 'c'=>3]
    // array_keys with a search value returns keys whose count is > 1.
    return array_keys(array_filter($counts, fn (int $c) => $c > 1));
}

print_r(findDuplicates(['a', 'b', 'a', 'c', 'c', 'c']));
// Output: Array ( [0] => a [1] => c )
```

If values can be mixed types or objects (where `array_count_values` won't work), build the map manually keyed by a serialized identity:

```php
<?php
$seen = [];
$dupes = [];
foreach ($items as $item) {
    $key = is_scalar($item) ? $item : serialize($item);
    if (isset($seen[$key])) {
        $dupes[$key] = $item; // dedupe the dupes themselves
    }
    $seen[$key] = true;
}
$dupes = array_values($dupes);
```

**Complexity.** O(n) time, O(n) space. (Sorting first would be O(n log n) and let you find dupes with O(1) extra space.)

---

### Problem 6: Two-Sum

**Problem.** Given an array of integers and a target, return the **indices** of the two numbers that add up to the target. Assume exactly one solution.

**Approach.** Brute force is two nested loops, O(n²). The optimal trick: for each number `x`, the partner you need is `target - x`. Store numbers you've seen in a hash map (value → index); if the complement is already in the map, you're done. One pass, O(n).

```php
<?php
declare(strict_types=1);

function twoSum(array $nums, int $target): array
{
    $seen = []; // value => index
    foreach ($nums as $i => $num) {
        $complement = $target - $num;
        if (isset($seen[$complement])) {
            return [$seen[$complement], $i];
        }
        $seen[$num] = $i;
    }
    return []; // no pair found
}

print_r(twoSum([2, 7, 11, 15], 9));  // Array ( [0] => 0 [1] => 1 )  (2 + 7)
print_r(twoSum([3, 2, 4], 6));       // Array ( [0] => 1 [1] => 2 )  (2 + 4)
```

> **Gotcha:** `isset($seen[$complement])` checks for *key existence*. If a value could legitimately be `0` mapped to index `0`, `isset` still works because the *value stored* is the index, and index `0` is a valid array value. Be careful: use `array_key_exists` instead of `isset` if you ever store `null` values, because `isset` returns `false` for `null`.

**Complexity.** O(n) time, O(n) space. This "complement in a hash map" trick generalizes to many array problems.

---

### Problem 7: Group / Aggregate an Array by a Key

**Problem.** Given a list of records (associative arrays), group them by a field and compute an aggregate (e.g., sum, count) per group.

**Approach.** Iterate once, bucket each row under `$row[$key]`. For aggregation, accumulate as you go. In Laravel, the `Collection::groupBy()` method does this elegantly.

**Plain PHP:**

```php
<?php
declare(strict_types=1);

$orders = [
    ['customer' => 'Ada', 'total' => 30],
    ['customer' => 'Bob', 'total' => 10],
    ['customer' => 'Ada', 'total' => 20],
];

// Group rows by customer
$grouped = [];
foreach ($orders as $order) {
    $grouped[$order['customer']][] = $order;
}

// Aggregate: total spend per customer
$totals = [];
foreach ($orders as $order) {
    $totals[$order['customer']] = ($totals[$order['customer']] ?? 0) + $order['total'];
}
print_r($totals);
// Output: Array ( [Ada] => 50 [Bob] => 10 )
```

**Laravel Collections** (much terser):

```php
<?php
use Illuminate\Support\Collection;

$totals = collect($orders)
    ->groupBy('customer')
    ->map(fn (Collection $rows) => $rows->sum('total'));

// $totals->toArray() => ['Ada' => 50, 'Bob' => 10]
```

**Complexity.** O(n) time, O(n) space.

---

### Problem 8: Flatten a Nested Array (Recursion and a Generator)

**Problem.** Turn `[1, [2, [3, 4]], 5]` into `[1, 2, 3, 4, 5]` for arbitrary depth.

**Approach 1 — Recursion.** For each element, if it's an array, recurse and merge; otherwise append.

```php
<?php
declare(strict_types=1);

function flattenRecursive(array $arr): array
{
    $result = [];
    foreach ($arr as $item) {
        if (is_array($item)) {
            // Spread the recursively-flattened sub-array into the result.
            array_push($result, ...flattenRecursive($item));
        } else {
            $result[] = $item;
        }
    }
    return $result;
}

print_r(flattenRecursive([1, [2, [3, 4]], 5]));
// Output: Array ( [0] => 1 [1] => 2 [2] => 3 [3] => 4 [4] => 5 )
```

**Approach 2 — Generator.** A generator (`yield`) produces values lazily, so memory stays low even for huge/deep structures — you never build the full intermediate array. `yield from` delegates to a sub-generator.

```php
<?php
declare(strict_types=1);

function flattenLazy(array $arr): \Generator
{
    foreach ($arr as $item) {
        if (is_array($item)) {
            yield from flattenLazy($item); // delegate to the recursive generator
        } else {
            yield $item;
        }
    }
}

foreach (flattenLazy([1, [2, [3, 4]], 5]) as $value) {
    echo $value; // prints 12345
}
echo PHP_EOL;
// If you need an array: iterator_to_array(flattenLazy($data), false)
```

**Laravel one-liner** for completeness: `collect($nested)->flatten()->all();`

**Complexity.** Both O(n) total time across all elements. The recursive version uses O(n) extra memory for the result; the generator uses O(depth) stack memory and streams output — better for very large inputs.

---

### Problem 9: Sort an Array of Objects by Multiple Keys

**Problem.** Sort users by `age` ascending, then by `name` ascending as a tiebreaker.

**Approach.** `usort` with a comparator. The **spaceship operator** `<=>` returns an `int` (negative / `0` / positive). Two idioms for multi-key sorting: (a) compare arrays of keys — `[$a->k1, $a->k2] <=> [$b->k1, $b->k2]` — which `<=>` evaluates element-by-element (lexicographic); or (b) chain with `?:` — `($a->k1 <=> $b->k1) ?: ($a->k2 <=> $b->k2)` — where a `0` (tie) on the first key falls through to the next. The array form is used below.

```php
<?php
declare(strict_types=1);

final class User
{
    public function __construct(
        public string $name,
        public int $age,
    ) {}
}

$users = [
    new User('Charlie', 30),
    new User('Alice', 25),
    new User('Bob', 30),
];

usort($users, function (User $a, User $b): int {
    // <=> compares the arrays element-by-element: age first, then name as
    // a tiebreaker when ages are equal. No need for an explicit ?: chain.
    return [$a->age, $a->name] <=> [$b->age, $b->name];
});

foreach ($users as $u) {
    echo "$u->name ($u->age)" . PHP_EOL;
}
// Output:
// Alice (25)
// Bob (30)
// Charlie (30)
```

The neat trick `[$a->age, $a->name] <=> [$b->age, $b->name]` works because `<=>` compares arrays **element by element**, giving you lexicographic multi-key ordering for free. For descending on a key, negate it: `[-$a->age, $a->name] <=> [-$b->age, $b->name]`.

**Laravel:** `collect($users)->sortBy([['age', 'asc'], ['name', 'asc']])->values();`

**Complexity.** O(n log n) time (PHP's `usort` uses an introsort/hybrid). `usort` is **not stable** before PHP 8.0; from PHP 8.0+ all sort functions are **stable** (equal elements keep their original order).

---

### Problem 10: Paginate an Array

**Problem.** Given an array and a page number + page size, return that page's slice plus pagination metadata.

**Approach.** `array_slice` with a computed offset. Metadata: total items, total pages (`ceil`), current page clamped to valid range.

```php
<?php
declare(strict_types=1);

function paginate(array $items, int $page = 1, int $perPage = 10): array
{
    $total = count($items);
    $lastPage = max(1, (int) ceil($total / $perPage));
    $page = max(1, min($page, $lastPage));      // clamp into [1, lastPage]
    $offset = ($page - 1) * $perPage;

    return [
        'data'         => array_slice($items, $offset, $perPage),
        'current_page' => $page,
        'per_page'     => $perPage,
        'total'        => $total,
        'last_page'    => $lastPage,
    ];
}

print_r(paginate(range(1, 25), page: 3, perPage: 10));
// Output: data => [21,22,23,24,25], current_page => 3,
//         per_page => 10, total => 25, last_page => 3
```

Note the **named arguments** (`page:`, `perPage:`) — they make the call self-documenting. In Laravel you'd usually reach for `LengthAwarePaginator` or `Collection::forPage()`, but knowing the manual math shows you understand what those abstractions do.

**Complexity.** O(k) time where k = perPage (`array_slice` copies the slice). O(1) for the metadata.

---

## Part 3 — Functions & Algorithms

### Problem 11: Memoize with a Closure

**Problem.** Cache the results of an expensive pure function so repeated calls with the same arguments are instant.

**Approach.** Wrap the function in a closure that holds a cache array by reference (via `use (&$cache)`). Build a cache key from the arguments. This is a classic demonstration of closures capturing state.

```php
<?php
declare(strict_types=1);

function memoize(callable $fn): callable
{
    $cache = [];
    return function (...$args) use (&$cache, $fn) {
        // Serialize args to form a stable cache key.
        $key = serialize($args);
        return $cache[$key] ??= $fn(...$args); // compute once, reuse forever
    };
}

$slowSquare = function (int $n): int {
    usleep(100_000);      // simulate expensive work
    return $n * $n;
};

$fastSquare = memoize($slowSquare);
echo $fastSquare(5); // 25 (slow first time)
echo $fastSquare(5); // 25 (instant — served from cache)
// Output: 2525
```

The `??=` (null coalescing assignment) operator only evaluates and assigns the right side if `$cache[$key]` is unset/null — perfect for "compute once" semantics. `&$cache` captures the array **by reference** so mutations persist between calls.

> **Gotcha:** Because `??=` treats a stored `null` as "absent," this caches everything *except* a legitimate `null` result — a function returning `null` would be recomputed every call. If `$fn` can return `null`, switch to `array_key_exists`:
> ```php
> if (! array_key_exists($key, $cache)) {
>     $cache[$key] = $fn(...$args);
> }
> return $cache[$key];
> ```

**Complexity.** First call: cost of `$fn`. Subsequent identical calls: O(cost of serialize) ≈ O(size of args). Space: O(number of distinct argument sets cached).

---

### Problem 12: Fibonacci — Recursive vs Iterative vs Generator

**Problem.** Compute the nth Fibonacci number (0, 1, 1, 2, 3, 5, 8, …).

**Approach.** Three flavors, each teaching a lesson.

**Naive recursion — exponential, avoid for large n:**

```php
<?php
declare(strict_types=1);

function fibRecursive(int $n): int
{
    if ($n < 2) {
        return $n;
    }
    return fibRecursive($n - 1) + fibRecursive($n - 2);
}
// fibRecursive(10) => 55 ; but fibRecursive(45) takes seconds — O(2^n)!
```

**Iterative — the production answer, O(n) time, O(1) space:**

```php
<?php
declare(strict_types=1);

function fibIterative(int $n): int
{
    [$a, $b] = [0, 1];
    for ($i = 0; $i < $n; $i++) {
        [$a, $b] = [$b, $a + $b]; // simultaneous swap via list destructuring
    }
    return $a;
}
echo fibIterative(10); // 55
```

**Generator — stream an infinite sequence lazily:**

```php
<?php
declare(strict_types=1);

function fibGenerator(): \Generator
{
    [$a, $b] = [0, 1];
    while (true) {                 // infinite, but only computed on demand
        yield $a;
        [$a, $b] = [$b, $a + $b];
    }
}

$gen = fibGenerator();
$first10 = [];
foreach ($gen as $f) {
    $first10[] = $f;
    if (count($first10) === 10) {
        break;
    }
}
print_r($first10);
// Output: Array ( [0] => 0 [1] => 1 [2] => 1 [3] => 2 [4] => 3
//   [5] => 5 [6] => 8 [7] => 13 [8] => 21 [9] => 34 )
```

> **Interview gold:** Explain *why* naive recursion is O(2ⁿ) — it recomputes the same subproblems exponentially (fib(5) computes fib(3) twice, fib(2) three times, …). You'd fix it with **memoization** (top-down DP) or the **iterative** bottom-up approach.

**Complexity.** Recursive O(2ⁿ) / O(n) stack; iterative O(n) / O(1); generator O(1) per value produced, O(1) space.

---

## Part 4 — Mini Systems (Design)

### Problem 13: Parse CSV into Objects

**Problem.** Read a CSV file with a header row and map each data row to a typed object.

**Approach.** Use `fgetcsv` to read line by line (memory-efficient — doesn't load the whole file). Treat the first row as headers, then `array_combine(headers, row)` to get associative rows, and hydrate a DTO. Yield rows from a generator so multi-GB files don't blow up memory.

```php
<?php
declare(strict_types=1);

final class Product
{
    public function __construct(
        public string $name,
        public float $price,
        public int $stock,
    ) {}
}

/** @return \Generator<Product> */
function parseCsv(string $path): \Generator
{
    $handle = fopen($path, 'rb');
    if ($handle === false) {
        throw new \RuntimeException("Cannot open: $path");
    }

    try {
        // PHP 8.4: pass escape: "" explicitly — see note below.
        $headers = fgetcsv($handle, escape: '');   // ['name','price','stock']
        if ($headers === false) {
            return;                                 // empty file
        }
        while (($row = fgetcsv($handle, escape: '')) !== false) {
            // array_combine throws if column counts differ — guard ragged rows.
            if (count($headers) !== count($row)) {
                continue; // or log/throw, depending on your strictness
            }
            $assoc = array_combine($headers, $row);
            yield new Product(
                name:  $assoc['name'],
                price: (float) $assoc['price'],
                stock: (int) $assoc['stock'],
            );
        }
    } finally {
        fclose($handle); // always close, even if an exception is thrown
    }
}

// Usage:
// foreach (parseCsv('products.csv') as $product) { /* ... */ }
```

> **PHP 8.4 note (important):** Starting in **PHP 8.4**, calling `fgetcsv`/`str_getcsv`/`fputcsv` **without** an explicit `$escape` argument raises a deprecation: *"the $escape parameter must be provided as its default value will change."* The legacy default is `"\\"` (backslash escaping), which is **not** RFC 4180-compliant and breaks round-tripping; the value will change (toward `""`) no earlier than PHP 9.0. The fix is to **always pass `escape: ""` explicitly** (as the code above does). An empty string disables backslash escaping and gives correct RFC 4180 behavior (the enclosure `"` is still escaped by doubling). This was *not* deprecated in PHP 8.1 — only in 8.4.

**Complexity.** O(n) over rows, O(1) extra memory per row thanks to streaming.

---

### Problem 14: Build a Simple Router

**Problem.** Map HTTP method + URI patterns (with `{params}`) to handlers, then dispatch a request.

**Approach.** Store routes as (method, regex, handler). Convert `/users/{id}` to a regex `#^/users/(?P<id>[^/]+)$#` with named capture groups. On dispatch, find the first matching route and call its handler with extracted params.

```php
<?php
declare(strict_types=1);

final class Router
{
    /** @var array<int, array{method:string, pattern:string, handler:callable}> */
    private array $routes = [];

    public function add(string $method, string $path, callable $handler): void
    {
        // Turn {id} into a named regex group; escape slashes via '#' delimiter.
        $pattern = preg_replace('#\{(\w+)\}#', '(?P<$1>[^/]+)', $path);
        $this->routes[] = [
            'method'  => strtoupper($method),
            'pattern' => "#^{$pattern}$#",
            'handler' => $handler,
        ];
    }

    public function dispatch(string $method, string $uri): mixed
    {
        foreach ($this->routes as $route) {
            if ($route['method'] !== strtoupper($method)) {
                continue;
            }
            if (preg_match($route['pattern'], $uri, $matches)) {
                // Keep only named params (string keys).
                $params = array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
                return ($route['handler'])(...array_values($params));
            }
        }
        // Throw, don't *return*, the exception — returning an exception object
        // means callers would treat the error as a successful result.
        throw new \RuntimeException('404 Not Found');
    }
}

$router = new Router();
$router->add('GET', '/users/{id}', fn (string $id) => "User #$id");
$router->add('GET', '/', fn () => 'Home');

echo $router->dispatch('GET', '/users/42'), PHP_EOL; // User #42
echo $router->dispatch('GET', '/'), PHP_EOL;         // Home
```

This is a teaching model of what Laravel's router does (Laravel adds middleware, named routes, model binding, and a compiled route cache). Mentioning that comparison scores points.

**Complexity.** O(r) per dispatch where r = number of routes (linear scan). Production routers compile routes into a single combined regex or trie for faster matching.

---

### Problem 15: Implement a Basic Event Dispatcher

**Problem.** Let code subscribe listeners to named events and dispatch events to all listeners — the **observer pattern**.

**Approach.** Keep a map of `eventName => listeners[]`. `listen` appends; `dispatch` calls each listener with the payload. This decouples emitters from handlers.

```php
<?php
declare(strict_types=1);

final class EventDispatcher
{
    /** @var array<string, list<callable>> */
    private array $listeners = [];

    public function listen(string $event, callable $listener): void
    {
        $this->listeners[$event][] = $listener;
    }

    public function dispatch(string $event, mixed $payload = null): void
    {
        foreach ($this->listeners[$event] ?? [] as $listener) {
            $listener($payload);
        }
    }
}

$events = new EventDispatcher();
$events->listen('user.registered', fn ($user) => print("Welcome email to {$user}\n"));
$events->listen('user.registered', fn ($user) => print("Audit log: {$user}\n"));

$events->dispatch('user.registered', 'ada@example.com');
// Output:
// Welcome email to ada@example.com
// Audit log: ada@example.com
```

Extensions an interviewer may probe: priorities (sort listeners), **stoppable events** (a listener returns `false` to halt propagation), wildcard listeners, and using `::class` of event objects as the event name (Laravel's approach: `Event::listen(UserRegistered::class, ...)`).

**Complexity.** `listen` O(1); `dispatch` O(L) where L = listeners for that event.

---

### Problem 16: Design a Simple Rate Limiter

**Problem.** Allow at most N actions per key (e.g., per IP) within a time window — the **fixed window** algorithm.

**Approach.** Keep, per key, a count and the window's expiry timestamp. On each hit: if the window expired, reset; otherwise increment and reject once the count exceeds N. (A more accurate variant is the **sliding window** or **token bucket** — mention these.)

```php
<?php
declare(strict_types=1);

final class RateLimiter
{
    /** @var array<string, array{count:int, resetAt:int}> */
    private array $buckets = [];

    public function __construct(
        private readonly int $maxAttempts,
        private readonly int $windowSeconds,
    ) {}

    public function allow(string $key): bool
    {
        $now = time();
        $bucket = $this->buckets[$key] ?? null;

        // Start a fresh window if none exists or the old one expired.
        if ($bucket === null || $now >= $bucket['resetAt']) {
            $this->buckets[$key] = ['count' => 1, 'resetAt' => $now + $this->windowSeconds];
            return true;
        }

        if ($bucket['count'] >= $this->maxAttempts) {
            return false; // limit hit
        }

        $this->buckets[$key]['count']++;
        return true;
    }
}

$limiter = new RateLimiter(maxAttempts: 3, windowSeconds: 60);
$results = array_map(fn () => $limiter->allow('1.2.3.4'), range(1, 4));
print_r(array_map(fn (bool $b) => $b ? 'allow' : 'block', $results));
// Output: Array ( [0] => allow [1] => allow [2] => allow [3] => block )
```

> **Production note:** An in-memory array only works within one process. Real apps store counters in **Redis** (`INCR` + `EXPIRE` is the canonical fixed-window implementation) so all servers share state. Laravel ships `Illuminate\Support\Facades\RateLimiter` and the `throttle` middleware that do exactly this.

**Trade-off to mention:** Fixed windows allow bursts at boundaries (up to 2N requests around the reset). **Sliding window log** or **token bucket** smooth this out at the cost of more state.

**Complexity.** O(1) per check, O(k) space for k distinct keys.

---

## Part 5 — Laravel-Specific Challenges

These assume Laravel 12 with Eloquent models `User` (hasMany `posts`) and `Post` (belongsTo `user`, has `created_at`, `category_id`).

### Problem 17: Top-N-Per-Group with Eloquent

**Problem.** Get the 3 most recent posts **for each** category.

**Approach.** "Top-N-per-group" is hard in plain SQL without window functions. Modern MySQL 8 / PostgreSQL support `ROW_NUMBER() OVER (PARTITION BY ...)`. With Laravel you can either use a raw window-function subquery, or — simplest and very readable — group in PHP after fetching, or use a lateral/correlated subquery.

**Window function approach (MySQL 8+/Postgres):**

```php
<?php
use Illuminate\Support\Facades\DB;

$topPosts = DB::table(DB::raw('(
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY created_at DESC) AS rn
    FROM posts
) ranked'))
    ->where('rn', '<=', 3)
    ->orderBy('category_id')
    ->get();
```

`PARTITION BY category_id` restarts the row numbering for each category; `ORDER BY created_at DESC` makes `rn = 1` the newest. We keep rows where `rn <= 3`.

**Eloquent collection approach (small datasets, very readable):**

```php
<?php
use App\Models\Post;

$topPerCategory = Post::query()
    ->latest()              // ORDER BY created_at DESC
    ->get()
    ->groupBy('category_id')
    ->map(fn ($posts) => $posts->take(3))
    ->flatten(1);
```

> **Trade-off:** The collection approach fetches *all* posts into memory — fine for thousands, bad for millions. The window-function approach pushes the work to the database. Say this out loud.

**Complexity.** Window function: handled in-DB, typically O(n log n) for the sort. Collection: O(n) fetch + O(n) group, but O(n) memory.

---

### Problem 18: Find Users with No Posts

**Problem.** List users who have never created a post.

**Approach.** Eloquent's `doesntHave` / `whereDoesntHave` generates an efficient `NOT EXISTS` subquery — the idiomatic answer.

```php
<?php
use App\Models\User;

// Users with zero posts:
$usersWithoutPosts = User::doesntHave('posts')->get();

// SQL generated (roughly):
// SELECT * FROM users
// WHERE NOT EXISTS (SELECT * FROM posts WHERE posts.user_id = users.id)
```

A constrained variant (no *published* posts):

```php
<?php
$users = User::whereDoesntHave('posts', function ($query) {
    $query->where('status', 'published');
})->get();
```

> **Why not `LEFT JOIN ... WHERE posts.id IS NULL`?** That also works and can be faster on some schemas, but `NOT EXISTS` is clearer, avoids row duplication from the join, and short-circuits. Mention both.

**Complexity.** Depends on indexes; with an index on `posts.user_id`, the `NOT EXISTS` is efficient (semi-join).

---

### Problem 19: Fixing an N+1 Query

**Problem.** This code is slow under load. Why, and how do you fix it?

```php
<?php
// PROBLEM: N+1 queries
$posts = Post::all();                 // 1 query
foreach ($posts as $post) {
    echo $post->user->name;           // +1 query PER post → N extra queries
}
// 100 posts => 101 queries total
```

**Approach.** This is the **N+1 problem**: one query to load the parents, then one *additional* query per parent to lazily load a relation. The fix is **eager loading** with `with()`, which loads all related records in a single extra `WHERE IN (...)` query.

```php
<?php
// FIX: eager load — 2 queries total regardless of post count
$posts = Post::with('user')->get();   // SELECT * FROM posts;
                                      // SELECT * FROM users WHERE id IN (...)
foreach ($posts as $post) {
    echo $post->user->name;           // no extra queries — already loaded
}
```

Related tools to name-drop:
- `with(['user', 'comments'])` — eager load multiple/nested (`with('comments.author')`).
- `loadMissing('user')` — eager load only if not already loaded.
- `withCount('comments')` — get counts without loading the rows.
- **`Model::preventLazyLoading()`** in `AppServiceProvider::boot()` — throws an exception whenever lazy loading happens in non-production, so N+1 bugs surface during development. This is the modern best practice.

```php
<?php
// In AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Model;

Model::preventLazyLoading(! $this->app->isProduction());
```

**Complexity.** N+1 is O(n) queries (catastrophic latency); eager loading is O(1) queries (2). Query *count*, not row count, dominates real-world latency.

---

### Problem 20: Form Request + Controller for a Resource

**Problem.** Validate and store a `Post`, keeping validation out of the controller.

**Approach.** Generate a Form Request (`php artisan make:request StorePostRequest`) for `authorize()` + `rules()`. The controller stays thin; Laravel auto-validates before the controller method runs and injects the validated data.

```php
<?php
declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Only logged-in users may create posts.
        return $this->user() !== null;
    }

    /** @return array<string, mixed> */
    public function rules(): array
    {
        return [
            'title'       => ['required', 'string', 'max:255'],
            'body'        => ['required', 'string', 'min:10'],
            'category_id' => ['required', 'integer', 'exists:categories,id'],
            'published'   => ['sometimes', 'boolean'],
        ];
    }
}
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Models\Post;
use Illuminate\Http\JsonResponse;

final class PostController extends Controller
{
    public function store(StorePostRequest $request): JsonResponse
    {
        // $request->validated() returns only the validated fields.
        $post = $request->user()->posts()->create($request->validated());

        return response()->json($post, 201); // 201 Created
    }
}
```

Route. In Laravel 11/12 the `routes/api.php` file is **no longer scaffolded by default** — run `php artisan install:api` to create it (this also installs Sanctum and registers the API routes in `bootstrap/app.php`). Without it, `auth:sanctum` and the `api` route group won't exist.

```php
<?php
// routes/api.php (created by `php artisan install:api`)
use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

Route::post('/posts', [PostController::class, 'store'])
    ->middleware('auth:sanctum');
```

> **Why Form Requests?** They centralize validation + authorization, are reusable, keep controllers single-responsibility, and a failed `authorize()` returns 403 while failed `rules()` returns 422 with structured errors automatically.

---

### Problem 21: A Queued Job That Processes a Batch

**Problem.** Send a newsletter to thousands of users without blocking the web request, and track overall progress.

**Approach.** Use **job batching** (`Bus::batch`). Each job handles a chunk of users. The batch gives you `progress()`, `then()`, `catch()`, and `finally()` callbacks. Jobs implement `ShouldQueue` so they run on a worker.

```php
<?php
declare(strict_types=1);

namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Batchable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Mail;

final class SendNewsletterChunk implements ShouldQueue
{
    use Batchable, Queueable;

    /** @param list<int> $userIds */
    public function __construct(private readonly array $userIds) {}

    public function handle(): void
    {
        if ($this->batch()?->cancelled()) {
            return; // batch was cancelled — bail early
        }

        foreach (User::findMany($this->userIds) as $user) {
            Mail::to($user)->send(new \App\Mail\Newsletter($user));
        }
    }
}
```

Dispatching the batch (e.g., from a controller or command):

```php
<?php
use App\Jobs\SendNewsletterChunk;
use App\Models\User;
use Illuminate\Support\Facades\Bus;
use Throwable;

$batch = Bus::batch(
    User::query()
        ->pluck('id')
        ->chunk(500)                                   // 500 ids per job
        ->map(fn ($ids) => new SendNewsletterChunk($ids->all()))
        ->all()
)->name('Newsletter Blast')
 ->allowFailures()
 ->then(fn () => logger('Newsletter complete'))
 ->catch(fn (\Illuminate\Bus\Batch $b, Throwable $e) => logger()->error($e->getMessage()))
 ->dispatch();

// $batch->progress() => 0..100 ; track via $batch->id
```

> **Notes:** `Queueable` (Laravel 11+) bundles `Dispatchable`, `InteractsWithQueue`, `Queueable`, and `SerializesModels`. Always queue **IDs, not full models**, in big batches to keep payloads small. Run workers with `php artisan queue:work`. Batching requires the `job_batches` table (`php artisan make:queue-batches-table` then migrate).

**Complexity.** Work is O(users) but spread across workers and off the request path. Chunking bounds per-job memory.

---

### Problem 22: A Feature Test for an Endpoint

**Problem.** Test that `POST /posts` creates a post for an authenticated user and rejects invalid input.

**Approach.** Use Laravel's feature-testing helpers: `actingAs`, `postJson`, and assertions on the HTTP response and the database. `RefreshDatabase` gives each test a clean schema.

```php
<?php
declare(strict_types=1);

namespace Tests\Feature;

use App\Models\Category;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class CreatePostTest extends TestCase
{
    use RefreshDatabase;

    public function test_authenticated_user_can_create_a_post(): void
    {
        $user = User::factory()->create();
        $category = Category::factory()->create();

        $response = $this->actingAs($user)->postJson('/posts', [
            'title'       => 'My First Post',
            'body'        => 'This is the body of the post.',
            'category_id' => $category->id,
        ]);

        $response->assertCreated()                        // 201
                 ->assertJsonPath('title', 'My First Post');

        $this->assertDatabaseHas('posts', [
            'title'   => 'My First Post',
            'user_id' => $user->id,
        ]);
    }

    public function test_validation_fails_without_a_title(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->postJson('/posts', [
            'body' => 'Body without a title.',
        ]);

        $response->assertStatus(422)                      // Unprocessable
                 ->assertJsonValidationErrors(['title', 'category_id']);
    }

    public function test_guests_cannot_create_posts(): void
    {
        $this->postJson('/posts', [])->assertUnauthorized(); // 401
    }
}
```

Run with `php artisan test` or `php artisan test --filter=CreatePostTest`.

**Pest version** — Laravel 11/12 ship **Pest as the default test runner**, so know this syntax too. The same suite in Pest is far terser (no class, no method names):

```php
<?php

use App\Models\Category;
use App\Models\User;

use function Pest\Laravel\{actingAs, postJson};

uses(\Illuminate\Foundation\Testing\RefreshDatabase::class);

it('lets an authenticated user create a post', function () {
    $user = User::factory()->create();
    $category = Category::factory()->create();

    actingAs($user)
        ->postJson('/posts', [
            'title'       => 'My First Post',
            'body'        => 'This is the body of the post.',
            'category_id' => $category->id,
        ])
        ->assertCreated()
        ->assertJsonPath('title', 'My First Post');

    $this->assertDatabaseHas('posts', [
        'title'   => 'My First Post',
        'user_id' => $user->id,
    ]);
});

it('rejects a post with no title', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->postJson('/posts', ['body' => 'Body without a title.'])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['title', 'category_id']);
});

it('forbids guests from creating posts', function () {
    postJson('/posts', [])->assertUnauthorized(); // 401
});
```

> **Tip:** `uses(RefreshDatabase::class)` in a test file (or in `tests/Pest.php` for the whole suite) is the Pest equivalent of the `use RefreshDatabase;` trait. The `actingAs`/`postJson` helpers come from the `Pest\Laravel` namespace. Knowing both PHPUnit and Pest syntax is a plus.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `strlen`/`strrev`/`substr` on multibyte text.** These count *bytes*, not characters, so `strlen("café")` is 5, not 4, and `strrev` corrupts UTF-8. **Fix:** use the `mb_*` family (`mb_strlen`, `mb_substr`, `mb_str_split`) and `/u` regex modifiers.

2. **`isset()` returning false for stored `null`.** In hash-map problems (two-sum, memoization), `isset($map[$key])` is `false` if the value is `null`, causing missed hits. **Fix:** use `array_key_exists($key, $map)` when `null` is a legitimate stored value.

3. **Naive recursive Fibonacci / unbounded recursion.** O(2ⁿ) blows up and deep recursion can hit memory limits (PHP has no tail-call optimization). **Fix:** memoize or iterate; for huge sequences use a generator.

4. **N+1 queries hidden behind clean-looking loops.** `$post->user->name` inside a loop silently fires a query per row. **Fix:** eager load with `with()`, and enable `Model::preventLazyLoading()` in dev to catch it automatically.

5. **Mutating an array while iterating it.** Adding/removing keys inside a `foreach` over the same array leads to surprising results. **Fix:** build a new array, or iterate over a copy / over `array_keys()`.

6. **Forgetting `usort` stability assumptions.** Before PHP 8.0, equal elements could be reordered. **Fix:** if you target ≥8.0 you're safe (all sorts are stable); otherwise add a tiebreaker key. Also remember `usort` **re-indexes** keys (use `uasort` to preserve them).

7. **Loading huge files/datasets into memory.** `file()` or `Post::all()` on millions of rows exhausts memory. **Fix:** stream with generators / `fgetcsv`, or Eloquent's `chunk()` / `lazy()` / `cursor()`.

---

## ✅ Best Practices

- **State complexity out loud.** Always end with "this is O(n) time, O(n) space because…". Interviewers want to hear it.
- **Reach for a hash map first.** Most "find / dedupe / pair / group" problems collapse from O(n²) to O(n) with an associative array.
- **Prefer `match` over `switch`** for value-returning branches — no fall-through, strict comparison, exhaustive.
- **Use generators for large or infinite sequences** to keep memory flat.
- **Type everything** — parameter, return, and property types plus `declare(strict_types=1)` catch bugs early and document intent.
- **Use constructor property promotion and `readonly`** for immutable DTOs (PHP 8.1+).
- **In Laravel, lean on the framework**: Collections for in-memory transforms, eager loading for relations, Form Requests for validation, batches for bulk work, factories + `RefreshDatabase` for tests.
- **Write the brute force first if stuck**, then optimize — a working O(n²) beats a broken O(n).
- **Name things well and keep functions small** — readability is graded.

---

## 🎯 Interview Tips & Likely Questions

**Q1. How would you reverse a Unicode string correctly, and why is `strrev` wrong?**
A. `strrev` reverses bytes; a UTF-8 char is 1–4 bytes, so byte reversal produces invalid sequences. Split into characters with `mb_str_split`, reverse the array, `implode`. For grapheme clusters (emoji with modifiers, flags) use the `grapheme_*`/intl functions.

**Q2. Two-sum: brute force vs optimal?**
A. Brute force is two nested loops, O(n²). Optimal: one pass with a hash map of `value → index`; for each number check if its complement (`target - num`) was already seen — O(n) time, O(n) space. The complement-in-a-map trick generalizes widely.

**Q3. Recursion vs iteration vs generators — when each?**
A. Recursion is natural for tree/divide-and-conquer problems but risks deep stacks and recomputation. Iteration is memory-efficient and the default for linear sequences. Generators (`yield`) stream values lazily — ideal for large/infinite data where you don't want to materialize everything. Naive recursive Fibonacci is O(2ⁿ); iterative is O(n)/O(1).

**Q4. What is the N+1 problem and how do you detect and fix it? (Under the hood.)**
A. Lazily accessing a relation inside a loop issues one query per parent row (N), plus the initial query (1). Under the hood, `with('user')` runs **two** queries: load posts, then `SELECT * FROM users WHERE id IN (<collected ids>)`, and Eloquent stitches the related models onto the parents in PHP by matching keys. Detect with Laravel Telescope/Debugbar or `Model::preventLazyLoading()`, which throws when lazy loading occurs in non-prod.

**Q5. How does a `match` expression differ from `switch`? (Under the hood.)**
A. `match` is an *expression* (returns a value), uses strict `===` comparison, has no fall-through, allows multiple conditions per arm, and throws `\UnhandledMatchError` if nothing matches and there's no `default`. `switch` uses loose `==`, falls through without `break`, and is a statement. `match` compiles to a chain of strict comparisons.

**Q6. How do generators work internally?**
A. A function containing `yield` returns a `Generator` object instead of running immediately. Each `foreach`/`->next()` resumes the function from where it last yielded, preserving local state on a separate execution frame. This makes iteration lazy and memory-constant. `yield from` delegates to an inner iterable/generator and can also forward keys and a return value.

**Q7. Design a rate limiter. What algorithms exist?**
A. **Fixed window** (counter + reset time) — simple but allows boundary bursts. **Sliding window log** — store timestamps, count those within the window — accurate but more memory. **Sliding window counter** — weighted blend of two fixed windows. **Token bucket / leaky bucket** — refill tokens over time, allowing controlled bursts. In distributed systems, store counters in Redis (`INCR`+`EXPIRE`) so all nodes share state; Laravel's `RateLimiter`/`throttle` middleware does this.

**Q8. How do you achieve top-N-per-group in SQL/Eloquent?**
A. Best: a window function — `ROW_NUMBER() OVER (PARTITION BY group ORDER BY sort DESC)` and filter `rn <= N` (MySQL 8+/Postgres). Alternatives: correlated subquery counting "how many are ranked above this row," or fetch+`groupBy()->map->take(N)` in a Collection for small datasets (but that loads everything into memory).

**Q9. Why use Form Requests instead of validating in the controller?**
A. They separate validation and authorization from controller logic (single responsibility), are reusable, auto-return 422/403 with structured errors, and run before the controller method so it only sees valid data via `validated()`.

**Q10. Why queue + batch a bulk operation, and what goes in the payload?**
A. Queueing moves slow work off the web request for fast responses and resilience (retries). Batching tracks aggregate progress and provides `then/catch/finally` hooks. Put **IDs, not full models**, in the payload to keep it small and avoid stale serialized state; rehydrate inside `handle()`.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Multibyte strings ---
mb_strlen($s); mb_substr($s, 0, 3); mb_str_split($s); mb_strtolower($s);
preg_match('/\p{L}+/u', $s);            // /u for UTF-8

// --- Hash-map patterns ---
$counts = array_count_values($arr);     // frequency map (int/string vals)
$map[$key] ??= compute();               // compute-once / memoize cell
isset($map[$k]) // false for null!  →  array_key_exists($k, $map)

// --- Array toolbox ---
array_filter($a, $cb); array_map($cb, $a); array_reduce($a, $cb, $init);
array_slice($a, $offset, $len); array_keys($a, $searchVal);
usort($a, fn($x,$y) => [$x->k1,$x->k2] <=> [$y->k1,$y->k2]); // multi-key
array_push($out, ...$spread);           // spread into

// --- Control flow ---
$r = match(true) { $n%15===0 => 'FB', $n%3===0 => 'F', default => "$n" };

// --- Generators ---
function gen(){ yield 1; yield from [2,3]; }
iterator_to_array(gen(), false);

// --- Complexity quick map ---
hash lookup O(1) | sort O(n log n) | nested loop O(n²) | naive fib O(2ⁿ)
```

```php
// --- Laravel cheat sheet ---
Post::with('user')->get();                       // fix N+1 (2 queries)
Post::withCount('comments')->get();              // counts, no rows
User::doesntHave('posts')->get();                // NOT EXISTS
collect($x)->groupBy('k')->map->sum('v');        // group + aggregate
Model::preventLazyLoading(!app()->isProduction()); // catch N+1 in dev

// Window function (top-N-per-group): ROW_NUMBER() OVER (PARTITION BY g ORDER BY s DESC)

// Validation: php artisan make:request StorePostRequest  → rules()/authorize()
// Queue:      implements ShouldQueue; use Queueable, Batchable;
//             Bus::batch([...])->then()->catch()->dispatch();
// Testing:    $this->actingAs($u)->postJson('/posts', [...])->assertCreated();
//             $this->assertDatabaseHas('posts', [...]);
//             use RefreshDatabase;
```

```bash
# Useful artisan commands
php artisan install:api                 # scaffolds routes/api.php + Sanctum (L11/12)
php artisan make:request StorePostRequest
php artisan make:job SendNewsletterChunk
php artisan make:test CreatePostTest     # --pest forces a Pest test; --unit for tests/Unit
php artisan make:queue-batches-table    # then: php artisan migrate
php artisan queue:work
php artisan test --filter=CreatePostTest
```

> **Laravel 11/12 structure reminder (common interview trap):** there is **no** `app/Http/Kernel.php` or `app/Console/Kernel.php`. Middleware and exception handling are configured fluently in **`bootstrap/app.php`**; service providers are listed in **`bootstrap/providers.php`**; scheduled tasks live in **`routes/console.php`**; and **Pest** is the default test runner. `routes/api.php` is created on demand via `php artisan install:api`.

---

## 🧪 Mini Exercises

1. **Longest unique substring.** Given a string, return the length of the longest substring with no repeated characters. Use the sliding-window + hash-set technique. State your complexity (target O(n)).

2. **Merge & sort intervals.** Given `[[1,3],[2,6],[8,10]]`, merge overlapping intervals into `[[1,6],[8,10]]`. Sort by start, then sweep. What's the time complexity, and why does sorting dominate?

3. **Eloquent: most active commenters.** Write a single query returning the top 5 users by number of comments in the last 30 days, including the count, with no N+1. (Hint: `withCount` + a constrained closure + `orderByDesc`.)

4. **LRU cache.** Implement a fixed-capacity Least-Recently-Used cache with O(1) `get` and `put`. (Hint: an ordered associative array — PHP arrays preserve insertion order; re-insert on access.)

5. **Streaming CSV aggregation.** Using a generator-based CSV reader, compute the total revenue per `region` from a 2 GB file without loading it into memory. Verify memory stays roughly constant.
