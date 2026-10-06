# Loops in PHP (and Laravel)

Loops let you run the same block of code repeatedly without copying and pasting it. They are the backbone of almost every real program: rendering a list of users, summing invoice totals, retrying a failed API call, paginating millions of database rows. PHP gives you several loop constructs, each tuned for a different situation. Knowing **which** to reach for — and the subtle traps each one hides — separates a junior from a senior backend engineer.

> **Jargon check.** An *iteration* is one pass through the loop body. A *control variable* (or *counter*) is the variable that changes each iteration to eventually end the loop. An *iterable* is anything you can loop over: an array, or an object that implements the `Traversable` interface (e.g. a `Generator` or an `ArrayIterator`).

---

## **What you'll learn**

- The four loop constructs — `for`, `while`, `do-while`, `foreach` — and exactly when each one is the right tool.
- How `foreach` works *by value* vs *by reference*, and the infamous **reference-after-foreach bug** that bites nearly everyone.
- Iterating associative arrays with `key => value`, and destructuring with `[$a, $b]` / `list()` inside `foreach`.
- Controlling flow with `break`, `continue`, and the multi-level `break N` / `continue N`.
- Looping over objects and any `Traversable` (Generators, iterators) — and why `foreach` is special there.
- The alternative colon syntax (`foreach: ... endforeach`) used heavily in plain-PHP and Blade-adjacent templates.
- Performance gotchas (like calling `count()` in a `for` condition) and the danger of modifying an array while iterating it.
- Laravel-specific iteration: `@foreach`/`@forelse` in Blade, `$loop`, and memory-safe DB iteration with `chunk`, `chunkById`, `cursor`, and `lazy`.

---

## 1. Why loops exist (the WHY before the HOW)

Imagine you must greet five users. Without loops:

```php
echo "Hi, Ana\n";
echo "Hi, Ben\n";
echo "Hi, Cy\n";
// ...and so on — brittle, unscalable, impossible if the count is unknown at write time
```

The data is dynamic — it comes from a database, a form, an API. You don't know the count when you write the code. A loop says "do this **for each** item, however many there are":

```php
$users = ['Ana', 'Ben', 'Cy'];

foreach ($users as $name) {
    echo "Hi, $name\n";
}
// Output:
// Hi, Ana
// Hi, Ben
// Hi, Cy
```

Every loop is built from three ideas: **initialization** (set up a starting state), a **condition** (when to keep going / when to stop), and **progress** (move toward the stopping condition each pass). If any of these is wrong you get either an infinite loop or a loop that never runs. Keep this mental model and the syntax becomes easy.

---

## 2. `for` — when you know the shape of the iteration

`for` is the explicit, "I control everything" loop. Its three clauses are separated by semicolons:

```php
for (init; condition; afterthought) {
    // body
}
```

- **init** runs once, before the first iteration.
- **condition** is checked **before** each iteration; the loop runs only while it is truthy.
- **afterthought** runs **after** each iteration.

```php
for ($i = 0; $i < 5; $i++) {
    echo $i;
}
// Output: 01234
```

Counting down or stepping by more than one:

```php
for ($i = 10; $i > 0; $i -= 2) {
    echo "$i ";
}
// Output: 10 8 6 4 2
```

### Multiple expressions per clause

Each clause may hold several comma-separated expressions. This is occasionally handy but usually hurts readability:

```php
for ($i = 0, $j = 10; $i < $j; $i++, $j--) {
    echo "$i-$j ";
}
// Output: 0-10 1-9 2-8 3-7 4-6
```

> **The condition is the *last* listed expression.** In `for ($i = 0; foo(), $i < 5; $i++)`, only `$i < 5` decides whether to continue; `foo()` is evaluated for its side effect each time and discarded.

### Empty clauses

Any clause can be empty. An empty condition is treated as `true` — this is a deliberate infinite loop (covered in §10):

```php
for (;;) {
    // runs forever unless you break
    break;
}
```

**Use `for` when** you iterate a numeric range, need the index for arithmetic (every other row, columns of a grid), or must walk an array by position. **Prefer `foreach`** when you simply want each element.

---

## 3. `while` — loop until a condition flips

`while` checks the condition **before** every iteration, so the body may run **zero** times.

```php
$countdown = 3;

while ($countdown > 0) {
    echo "$countdown ";
    $countdown--;
}
// Output: 3 2 1
```

`while` shines when the number of iterations is **unknown in advance** and depends on runtime state — reading lines from a file, polling a queue, consuming a stream:

```php
$handle = fopen('data.txt', 'r');

while (($line = fgets($handle)) !== false) {
    echo trim($line) . "\n";
}

fclose($handle);
```

> **Why `!== false` and not just `while ($line = fgets($handle))`?** A line containing `"0"` (or an empty line read as `""`) is *falsy* in PHP, so the loose check would stop early. `fgets()` returns the boolean `false` only at end-of-file, so the strict `!== false` comparison is correct. This same care applies to any function that signals "done"/"not found" with `false` while also being able to return a falsy success value — e.g. `readdir()` (a directory entry named `"0"`), `fread()`, `stream_get_line()`, and `strpos()` (a match at offset `0`).
>
> *(Historical note: the old `each()` function used in this pattern in PHP 5/7 was **removed in PHP 8.0** — do not use it on PHP 8.x.)*

---

## 4. `do-while` — run the body at least once

`do-while` checks the condition **after** the body, guaranteeing **at least one** execution. Note the trailing semicolon.

```php
$attempt = 0;

do {
    $attempt++;
    echo "Attempt $attempt\n";
} while ($attempt < 3);
// Output:
// Attempt 1
// Attempt 2
// Attempt 3
```

Contrast: if `$attempt` started at `99`, a `while` would print nothing, but `do-while` still prints "Attempt 100" once.

Classic use cases: input validation prompts ("ask, then re-ask until valid"), and retry-with-backoff logic where you always make the first attempt:

```php
$tries = 0;
$maxTries = 5;

do {
    $response = callFlakyApi();
    $tries++;
} while ($response === null && $tries < $maxTries);
```

> A lesser-known trick: `do { ... } while (false);` is sometimes used as a "run once" block you can `break` out of early instead of using nested `if`s. It works, but a guard clause or early `return` is usually clearer.

---

## 5. `foreach` — the workhorse for arrays and objects

`foreach` is the idiomatic way to iterate **arrays** and **`Traversable` objects**. It manages the cursor for you — no off-by-one errors, no `count()` calls.

### 5.1 By value (the default)

```php
$prices = [10, 20, 30];

foreach ($prices as $price) {
    echo $price * 2, " ";
}
// Output: 20 40 60
```

Here `$price` is a **copy** of each element. Mutating `$price` inside the loop does **not** change the original array:

```php
$prices = [10, 20, 30];

foreach ($prices as $price) {
    $price *= 2;          // changes only the local copy
}

print_r($prices);
// Output: Array ( [0] => 10 [1] => 20 [2] => 30 )   <-- unchanged
```

### 5.2 With keys: `key => value`

To get the index/key as well as the value:

```php
$capitals = [
    'France'  => 'Paris',
    'Japan'   => 'Tokyo',
    'Egypt'   => 'Cairo',
];

foreach ($capitals as $country => $capital) {
    echo "$capital is the capital of $country\n";
}
// Output:
// Paris is the capital of France
// Tokyo is the capital of Japan
// Cairo is the capital of Egypt
```

This works on **list arrays** too, where the key is the integer index:

```php
$letters = ['a', 'b', 'c'];

foreach ($letters as $index => $letter) {
    echo "$index=$letter ";
}
// Output: 0=a 1=b 2=c
```

### 5.3 By reference (`&`) — and the trap

Prefix the value variable with `&` to make it an **alias** of the real array element. Now mutations stick:

```php
$prices = [10, 20, 30];

foreach ($prices as &$price) {
    $price *= 2;
}
unset($price);            // <-- ESSENTIAL: break the reference (see below)

print_r($prices);
// Output: Array ( [0] => 20 [1] => 40 [2] => 60 )   <-- changed!
```

#### The classic reference-after-foreach bug

After a by-reference `foreach`, `$price` **still points at the last element** of the array. If you then reuse that variable (or run a second `foreach`), you silently corrupt your data. This is one of the most-asked PHP interview "gotchas."

```php
$items = [1, 2, 3];

foreach ($items as &$item) {
    // ... do something
}
// $item is now a reference to $items[2]

foreach ($items as $item) {   // by VALUE this time
    // each pass assigns to $item, which is STILL aliased to $items[2]!
}

print_r($items);
// Output: Array ( [0] => 1 [1] => 2 [2] => 2 )   <-- last element clobbered!
```

Walkthrough of why `$items[2]` ends up as `2` (final array `[1, 2, 2]`). Remember `$item` is **still an alias of `$items[2]`**, and the value `foreach` reads each pass is the element at the current index:

- **Pass 1** reads `$items[0]` (`1`) and assigns it into `$item`, i.e. into `$items[2]`. Array is now `[1, 2, 1]`.
- **Pass 2** reads `$items[1]` (`2`) and assigns it into `$items[2]`. Array is now `[1, 2, 2]`.
- **Pass 3** reads `$items[2]` — but that slot was just overwritten to `2`, so it reads `2` (not the original `3`) and writes `2` back into itself. Array stays `[1, 2, 2]`.

The original `3` is destroyed on pass 2 before the loop ever reaches it. The net effect is that the last element gets clobbered with the second-to-last value.

**The fix is always the same: `unset()` the reference variable immediately after the loop.**

```php
foreach ($items as &$item) {
    // ...
}
unset($item);   // now $item is a free variable again — safe to reuse
```

> **When does `&` actually do anything?** Only when you loop over an array (or by-reference-yielding iterable) **stored in a real variable**. PHP *accepts* `foreach ([1, 2, 3] as &$x)` and `foreach (getArray() as &$x)` without error on PHP 8.x, but the mutations are **useless**: the array literal / function return value is a temporary that is thrown away when the loop ends, so nothing persists. Always loop over a **named variable** when you use `&`, otherwise the reference has no observable effect.

### 5.4 Destructuring inside `foreach` (`list()` / `[]`)

If each element is itself an array, you can unpack it in the loop head. The short `[]` syntax (PHP 7.1+) is preferred over the older `list()`.

```php
$points = [
    [1, 2],
    [3, 4],
    [5, 6],
];

foreach ($points as [$x, $y]) {
    echo "($x, $y) ";
}
// Output: (1, 2) (3, 4) (5, 6)
```

You can destructure by **string keys** too — order-independent and self-documenting:

```php
$users = [
    ['id' => 1, 'name' => 'Ana'],
    ['id' => 2, 'name' => 'Ben'],
];

foreach ($users as ['id' => $id, 'name' => $name]) {
    echo "#$id $name\n";
}
// Output:
// #1 Ana
// #2 Ben
```

The verbose equivalent uses `list()`. It is fully equivalent to `[]` — including **keyed** destructuring (`list()` has supported string keys since PHP 7.1) — just wordier, so the short form is preferred:

```php
foreach ($points as list($x, $y)) {                 // identical to [$x, $y]
    // ...
}

foreach ($users as list('id' => $id, 'name' => $name)) {  // keyed list() also works
    // ...
}
```

> Nested destructuring works: `foreach ($data as [$id, [$lat, $lng]]) { ... }`. Skipping elements with empty slots also works: `foreach ($rows as [, $second]) { ... }` ignores the first column.

### 5.5 `foreach` does not move the internal array pointer (PHP 7+)

In PHP 5, `foreach` operated on the array's internal pointer and could interfere with `current()`/`next()`. Since **PHP 7**, `foreach` iterates over a **copy of the iteration state** (for by-value loops over arrays), so it does not advance the array's internal pointer. This makes nested `foreach` over the same array safe and predictable — a frequent "what changed in PHP 7" interview note.

---

## 6. Iterating objects and `Traversable`

### 6.1 Plain objects

`foreach` over a plain object iterates its **visible (accessible) properties**. From outside the class you see only `public` properties; from inside a method you also see `private`/`protected` ones.

```php
class Profile
{
    public string $name = 'Ada';
    public int $age = 36;
    private string $secret = 'hidden';
}

foreach (new Profile() as $prop => $value) {
    echo "$prop = $value\n";
}
// Output (from outside the class — private is skipped):
// name = Ada
// age = 36
```

### 6.2 `Iterator` and `IteratorAggregate`

To control exactly how an object is iterated, implement `Iterator` (5 methods) or, more commonly, `IteratorAggregate` (1 method that returns a `Traversable`). Both extend the `Traversable` marker interface that `foreach` checks for.

```php
class NumberRange implements IteratorAggregate
{
    public function __construct(
        private int $start,
        private int $end,
    ) {}

    public function getIterator(): Iterator
    {
        // A generator is the simplest Traversable to return
        for ($i = $this->start; $i <= $this->end; $i++) {
            yield $i;
        }
    }
}

foreach (new NumberRange(3, 6) as $n) {
    echo "$n ";
}
// Output: 3 4 5 6
```

### 6.3 Generators — lazy, memory-cheap loops

A function containing `yield` is a **generator**: it produces values one at a time, computing each only when the loop asks for it. This means you can "loop over" a billion items using almost no memory, because they are never all in an array at once.

```php
function naturals(): Generator
{
    $n = 1;
    while (true) {
        yield $n++;
    }
}

foreach (naturals() as $n) {
    if ($n > 5) break;
    echo "$n ";
}
// Output: 1 2 3 4 5
```

You can also yield keys: `yield $key => $value;`. Generators are `Traversable`, so `foreach` accepts them directly — but note a generator can usually be iterated **only once**.

> **`iterator_to_array()`** converts any `Traversable` into a real array when you genuinely need one (e.g. to `count()` or re-iterate). Use it sparingly — it defeats the memory benefit of generators.

---

## 7. `break`, `continue`, and the `N` variants

### 7.1 `break` — leave the loop entirely

```php
$haystack = [4, 8, 15, 16, 23, 42];

foreach ($haystack as $n) {
    if ($n === 16) {
        echo "Found it!\n";
        break;            // stop searching
    }
}
```

### 7.2 `continue` — skip to the next iteration

```php
for ($i = 1; $i <= 6; $i++) {
    if ($i % 2 === 0) {
        continue;         // skip even numbers
    }
    echo "$i ";
}
// Output: 1 3 5
```

> **`continue` in a `for` loop still runs the afterthought.** `continue` jumps to the condition check, and in a `for` that means the `$i++` part *does* execute. Beginners sometimes fear `continue` causes an infinite loop in `while` — it can, if the increment is *inside* the skipped region. Put progress logic before any `continue`.

### 7.3 `break N` and `continue N` — multi-level control

These take an optional integer telling PHP **how many enclosing loops** to break out of / continue. `break 1` equals `break`. This is genuinely useful for nested loops and avoids "flag variable" hacks.

```php
for ($row = 0; $row < 3; $row++) {
    for ($col = 0; $col < 3; $col++) {
        if ($row === 1 && $col === 1) {
            echo "Hit center, bailing out of BOTH loops\n";
            break 2;       // exits inner AND outer
        }
        echo "($row,$col) ";
    }
}
// Output:
// (0,0) (0,1) (0,2) (1,0) Hit center, bailing out of BOTH loops
```

`continue 2` skips to the next iteration of the **outer** loop:

```php
foreach (['a', 'b'] as $letter) {
    foreach ([1, 2, 3] as $num) {
        if ($num === 2) {
            continue 2;    // jump to next $letter
        }
        echo "$letter$num ";
    }
}
// Output: a1 b1
```

> **`break`/`continue` and `switch`/`match`.** A `switch` counts as one "loop level" for `break`/`continue`. So inside a loop, a bare `continue` written *inside* a `switch` targets the `switch`, **not** the loop — and since **PHP 7.3** this emits the warning `"continue" targeting switch is equivalent to "break". Did you mean to use "continue 2"?`. Use `continue 2` to reach the enclosing loop. `match` is an expression, not a statement, so `break`/`continue` cannot target it and this pitfall does not arise — one more reason to prefer `match`.

> **`break`/`continue` cannot cross function boundaries.** Inside a closure passed to `array_map`/`array_filter` you cannot `break` the outer loop — those are function calls, not loop bodies. Use a real `foreach` if you need to break early.

---

## 8. Nested loops

Loops inside loops handle grids, matrices, combinations, and joins. Keep the variable names distinct (`$i`/`$j`, or meaningful names) and watch the complexity — two nested loops over `n` items is `O(n²)`.

```php
$matrix = [
    [1, 2, 3],
    [4, 5, 6],
];

foreach ($matrix as $rowIndex => $row) {
    foreach ($row as $colIndex => $cell) {
        echo "M[$rowIndex][$colIndex]=$cell ";
    }
    echo "\n";
}
// Output:
// M[0][0]=1 M[0][1]=2 M[0][2]=3
// M[1][0]=4 M[1][1]=5 M[1][2]=6
```

> **Performance reflex:** if you find yourself searching an inner array on every outer iteration (an `O(n*m)` lookup), build a hash map (`array keyed by id`) once before the loop and look up in `O(1)`. This single change is a common interview "optimize this" answer.

---

## 9. The alternative colon syntax (`endfor`, `endwhile`, `endforeach`)

Every loop has a **colon form** that replaces the opening `{` with `:` and the closing `}` with `endfor;` / `endwhile;` / `endforeach;`. It exists to make PHP-in-HTML templates readable, because the explicit `endforeach` keyword is far easier to match against opening tags than a lonely `}` buried among markup.

```php
<ul>
<?php foreach ($users as $user): ?>
    <li><?= htmlspecialchars($user['name']) ?></li>
<?php endforeach; ?>
</ul>
```

The same applies to `for`/`endfor`, `while`/`endwhile`, and `if`/`endif`. `do-while` has **no** colon form.

```php
<?php for ($i = 1; $i <= 3; $i++): ?>
    <p>Item <?= $i ?></p>
<?php endfor; ?>
```

> **Laravel/Blade tie-in.** Blade's `@foreach ... @endforeach`, `@for ... @endfor`, and `@while ... @endwhile` directives compile down to exactly this PHP colon syntax. Understanding the raw form demystifies what Blade generates (see §12).

---

## 10. Infinite loops and guards

An **infinite loop** never satisfies its exit condition. Sometimes intentional (an event loop, a queue worker), often a bug.

Intentional, with a clear exit path:

```php
while (true) {
    $job = $queue->pop();
    if ($job === null) {
        break;          // nothing left to do
    }
    $job->handle();
}
```

The dangerous accidental version usually comes from **forgetting to make progress**:

```php
$i = 0;
while ($i < 10) {
    echo $i;
    // forgot $i++  -> runs forever, pegs CPU, eventually times out
}
```

### Always add a guard for unbounded loops

When looping until an external condition (API readiness, retries), cap the iterations so a bug or outage cannot hang the request forever:

```php
$maxIterations = 1000;
$guard = 0;

while (! $done) {
    if (++$guard > $maxIterations) {
        throw new RuntimeException('Loop exceeded safety limit');
    }
    $done = doWork();
}
```

> PHP's `max_execution_time` (default 30s for web, 0/unlimited for CLI) will eventually kill a runaway web request, but you should not rely on it — by then you may have leaked memory or left data half-written. Explicit guards are defensive engineering.

---

## 11. Performance notes and the array-mutation pitfall

### 11.1 Don't recompute `count()` every iteration

```php
// BAD: count($items) is called on EVERY iteration
for ($i = 0; $i < count($items); $i++) {
    // ...
}

// GOOD: compute once
for ($i = 0, $n = count($items); $i < $n; $i++) {
    // ...
}

// BEST when you don't need the index at all:
foreach ($items as $item) {
    // ...
}
```

For small arrays the difference is negligible, but the habit matters: `count()` in the condition is `O(n)` per check, turning a clean `O(n)` loop into `O(n²)` in the worst case. (PHP arrays cache their size, so modern `count()` is `O(1)` for plain arrays — but the *call overhead* and the principle of hoisting invariant work out of loops still stand, and for `Countable` objects `count()` can be genuinely expensive.)

### 11.2 Hoist invariant work out of the loop

```php
// BAD: prepares the same regex / config every pass
foreach ($lines as $line) {
    $pattern = '/\d+/';        // rebuilt each time
    preg_match($pattern, $line, $m);
}

// GOOD
$pattern = '/\d+/';
foreach ($lines as $line) {
    preg_match($pattern, $line, $m);
}
```

### 11.3 Modifying an array while iterating it

`foreach` (by value) iterates over an **internal copy of the array's iteration order**, so adding/removing keys mid-loop usually does not crash — but the results are confusing and version-dependent. The safe rules:

```php
// RISKY: removing during a by-value foreach — operates on a copy,
// so you may delete from the original while the loop keeps going over old data
$nums = [1, 2, 3, 4];
foreach ($nums as $i => $n) {
    if ($n % 2 === 0) {
        unset($nums[$i]);   // affects $nums, not the iterated copy
    }
}
print_r($nums);
// Output: Array ( [0] => 1 [2] => 3 )  -- works here, but fragile to reason about
```

Two robust patterns instead:

```php
// 1) Build a NEW array (functional, clear, no mutation surprises)
$odds = array_filter($nums, fn ($n) => $n % 2 !== 0);

// 2) Collect keys to remove, then delete after the loop
$toRemove = [];
foreach ($nums as $i => $n) {
    if ($n % 2 === 0) {
        $toRemove[] = $i;
    }
}
foreach ($toRemove as $i) {
    unset($nums[$i]);
}
```

> **Iterating by reference while mutating structure is especially dangerous.** Appending to an array you're looping over by reference can produce an infinite loop or undefined behavior. Rule of thumb: **don't change an array's shape (add/remove keys) while iterating it.** Changing *values in place* (via `&` or by key) is fine.

---

## 12. Looping in Laravel

### 12.1 Blade directives

Blade compiles to the PHP colon syntax under the hood. The core loop directives:

```blade
@foreach ($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@for ($i = 0; $i < 3; $i++)
    <p>{{ $i }}</p>
@endfor

@while ($condition)
    {{-- ... --}}
@endwhile
```

`@forelse` is Laravel's sugar for "loop, but show something if empty":

```blade
@forelse ($orders as $order)
    <tr><td>{{ $order->id }}</td></tr>
@empty
    <tr><td>No orders yet.</td></tr>
@endforelse
```

### 12.2 The `$loop` variable

Inside any Blade `@foreach`/`@for`, Laravel injects a `$loop` object with iteration metadata — no manual counters needed:

```blade
@foreach ($users as $user)
    <li class="{{ $loop->even ? 'striped' : '' }}">
        {{ $loop->iteration }}. {{ $user->name }}
        @if ($loop->first) (first!) @endif
        @if ($loop->last) (last!) @endif
    </li>
@endforeach
```

Key `$loop` properties: `index` (0-based), `iteration` (1-based), `count`, `first`, `last`, `even`, `odd`, `remaining`, `depth`, and `parent` (the outer loop's `$loop` in nested loops). Access the outer loop from an inner one via `$loop->parent->iteration`.

### 12.3 Memory-safe database iteration

Loading 1,000,000 Eloquent models with `User::all()` then `foreach`-ing them will exhaust memory. Laravel provides loop-friendly, memory-bounded alternatives:

```php
// chunk: load N rows at a time (each chunk is a Collection)
User::chunk(500, function ($users) {
    foreach ($users as $user) {
        // process...
    }
});

// chunkById: like chunk but uses the id for pagination — SAFE when you
// modify rows inside the loop in a way that could shift offsets
User::where('active', true)->chunkById(500, function ($users) {
    foreach ($users as $user) {
        $user->update(['active' => false]);
    }
});

// cursor: one model in memory at a time via a generator (single DB query,
// streams rows) — great for read-only passes
foreach (User::cursor() as $user) {
    // very low memory
}

// lazy / lazyById: generator + chunked queries — best of both worlds (L8+)
foreach (User::lazy() as $user) {
    // ...
}
```

> **`chunk` vs `chunkById`:** if your loop body changes the column you're ordering/filtering on, plain `chunk`'s offset pagination can **skip records**. `chunkById` keys off the (stable, increasing) primary key, avoiding the skip. This is a favorite senior-level Laravel interview distinction. Return `false` from a `chunk` callback to stop chunking early — the equivalent of `break`.

> **`cursor` vs `lazy`:** `cursor()` runs a single query and streams the result set (low PHP memory, but the DB connection holds the whole result), while `lazy()` runs many smaller queries under the hood. For huge tables, `lazy()`/`lazyById()` is generally the safest.

### 12.4 Collections favor pipelines over manual loops

Idiomatic Laravel often replaces explicit loops with Collection methods, which are more declarative:

```php
$names = collect($users)
    ->filter(fn ($u) => $u->active)
    ->map(fn ($u) => $u->name)
    ->values()
    ->all();
```

Under the hood these still loop — but they read as a transformation pipeline and avoid temporary variables. Use a real loop when you need `break`, side effects, or complex control flow.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `unset($ref)` after a by-reference `foreach`.**
   The leftover reference silently corrupts the last array element on the next use of that variable.
   **Fix:** always `unset($value);` immediately after `foreach ($arr as &$value) { ... }`. Better yet, avoid `&` entirely when a `map`/`array_map` or rebuilding a new array will do.

2. **Calling `count()` (or other expensive work) inside the loop condition.**
   `for ($i = 0; $i < count($a); $i++)` re-evaluates `count($a)` every pass.
   **Fix:** hoist it — `for ($i = 0, $n = count($a); $i < $n; $i++)` — or use `foreach` when the index isn't needed.

3. **Modifying an array's shape while iterating it.**
   Adding/removing keys during a `foreach` produces version-dependent, hard-to-reason-about results (and can infinite-loop with references).
   **Fix:** build a new array (`array_filter`/`array_map`), or collect keys to delete and `unset()` them *after* the loop.

4. **Loose truthy checks in `while` reads.**
   `while ($line = fgets($f))` stops on a line that is `"0"` or empty.
   **Fix:** compare strictly against the sentinel: `while (($line = fgets($f)) !== false)`. Same lesson for `strpos`, `array_search`, `next`, etc.

5. **Off-by-one in `for` bounds.**
   `for ($i = 0; $i <= count($a); $i++)` reads one past the end → undefined-array-key warning / `null`.
   **Fix:** use `< $n` for zero-based indexing, and prefer `foreach` to sidestep the boundary entirely.

6. **`break`/`continue` from inside a closure or a `switch`.**
   You can't `break` an outer loop from inside an `array_map` callback; a bare `continue` inside a `switch` targets the `switch`, not the loop.
   **Fix:** use a real `foreach` for early exits; use `continue 2` to skip past a `switch` to the loop, or prefer `match`.

7. **Forgetting to make progress → infinite loop.**
   A `while`/`for` whose control variable never changes (or whose increment sits inside a `continue`-skipped region) hangs the process.
   **Fix:** ensure the body always moves toward the exit condition; add a guard counter for unbounded loops.

8. **Using `User::all()` then looping over millions of rows.**
   Loads everything into memory and OOMs.
   **Fix:** use `chunkById`, `cursor`, or `lazyById` to stream rows.

---

## ✅ Best Practices

- **Reach for `foreach` by default** for arrays and iterables; use `for` only when you genuinely need the numeric index for arithmetic.
- **Prefer `foreach` by value**; introduce `&` only when in-place mutation is the clearest option, and always `unset()` afterward.
- **Hoist invariant work** (counts, regex compilation, config lookups, DB queries) out of the loop body.
- **Don't mutate the structure you're iterating** — produce a new collection instead. It's clearer and avoids version-specific surprises.
- **Use `break N` / `continue N`** instead of boolean "flag" variables to escape nested loops; it expresses intent directly.
- **Use destructuring** (`foreach ($rows as ['id' => $id, 'name' => $name])`) to make loop bodies self-documenting.
- **Guard unbounded loops** with an explicit iteration cap; never trust `max_execution_time` as your only safety net.
- **In Laravel**, stream large result sets with `chunkById`/`lazyById`/`cursor`, prefer Collection pipelines for transformations, and use `@forelse` + `$loop` in Blade instead of hand-rolled counters and empty checks.
- **Keep loop bodies small** — extract complex per-item logic into a well-named method/function for testability.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `while` and `do-while`?**
A. `while` checks the condition *before* the body, so it may run zero times. `do-while` checks *after* the body, guaranteeing at least one execution. Use `do-while` when the first attempt should always happen (input prompts, first API call before deciding to retry).

**Q2. Explain the reference-after-foreach bug.**
A. After `foreach ($arr as &$v)`, `$v` remains a reference to the last element. Reusing `$v` (e.g. in a later by-value `foreach`) writes through that reference and clobbers the last array element. The fix is `unset($v);` right after the loop. It's a favorite gotcha because it's silent and data-corrupting.

**Q3. How does `foreach` work under the hood? Does it move the array's internal pointer?**
A. Since PHP 7, `foreach` over an array works on a *copy of the iteration state* (using copy-on-write so it's cheap unless you mutate), and it does **not** advance the array's internal pointer the way PHP 5 did. By reference (`&`), it operates on the live array. For objects, `foreach` checks for the `Traversable` interface: if the object implements `Iterator`/`IteratorAggregate`, PHP calls `rewind`/`valid`/`current`/`key`/`next` (or the aggregated iterator's methods); otherwise it iterates the object's accessible properties.

**Q4. What do `break 2` and `continue 2` do?**
A. The integer is the number of nested loop levels to act on. `break 2` exits two enclosing loops at once; `continue 2` skips to the next iteration of the loop two levels out. They replace flag-variable workarounds in nested loops. `break`/`continue` also count `switch` as a level.

**Q5. Why is `for ($i = 0; $i < count($arr); $i++)` considered bad?**
A. `count($arr)` is re-evaluated on every iteration. Even though PHP caches array length (so it's O(1)), the repeated call is wasteful and for `Countable` objects it can be genuinely expensive. Hoist it: `for ($i = 0, $n = count($arr); $i < $n; $i++)`, or just use `foreach`.

**Q6. Can you modify an array while looping over it with `foreach`?**
A. You can change values in place (by reference or by key), but changing the *structure* (adding/removing keys) is unsafe and confusing because by-value `foreach` iterates a copy of the iteration state. Prefer building a new array with `array_filter`/`array_map`, or collect keys and `unset()` them after the loop.

**Q7. What is the alternative (colon) syntax and where is it used?**
A. `foreach (...): ... endforeach;` (and `endfor`, `endwhile`, `endif`). It makes PHP-in-HTML templates readable by giving each block a named closing keyword. Blade's `@foreach`/`@endforeach` compile down to exactly this. `do-while` has no colon form.

**Q8. In Laravel, how do you loop over a huge table without running out of memory? `chunk` vs `chunkById` vs `cursor` vs `lazy`?**
A. `User::all()` loads everything → OOM. `chunk(n, ...)` pulls `n` rows per query using offset pagination. `chunkById(n, ...)` paginates by primary key — safer when the loop modifies the filtered/ordered column, because offset chunking can skip rows. `cursor()` returns a generator backed by a single query that streams rows (low PHP memory). `lazy()`/`lazyById()` returns a generator backed by many chunked queries — the most robust for very large tables. Return `false` from a `chunk` callback to stop early.

**Q9. What's `$loop` in Blade?**
A. An object Laravel injects into `@foreach`/`@for` with metadata: `index`, `iteration`, `count`, `first`, `last`, `even`, `odd`, `remaining`, `depth`, and `parent` (for nested loops). It removes the need for manual counters and first/last flags.

**Q10. Generators and `yield` — how do they relate to loops?**
A. A function with `yield` returns a `Generator` (a `Traversable`), letting `foreach` consume values lazily, one at a time, with O(1) memory regardless of total count. Ideal for streaming files, ranges, or DB rows. A generator is typically single-use; `iterator_to_array()` materializes it into an array if needed.

---

## 📋 Quick Reference / Cheat Sheet

```php
// for — known/indexed iteration
for ($i = 0, $n = count($a); $i < $n; $i++) { /* ... */ }

// while — condition checked BEFORE (may run 0 times)
while ($cond) { /* ... */ }

// do-while — condition checked AFTER (runs >= 1 time)
do { /* ... */ } while ($cond);

// foreach by value (copy)
foreach ($arr as $value) { /* ... */ }

// foreach with key
foreach ($arr as $key => $value) { /* ... */ }

// foreach by reference — REMEMBER to unset!
foreach ($arr as &$value) { $value *= 2; }
unset($value);

// destructuring in foreach
foreach ($rows as [$x, $y]) { /* ... */ }
foreach ($rows as ['id' => $id, 'name' => $name]) { /* ... */ }

// flow control
break;       // exit current loop
break 2;     // exit two nested levels
continue;    // next iteration of current loop
continue 2;  // next iteration two levels out

// colon syntax (templates)
foreach ($a as $v): /* ... */ endforeach;
for (...): endfor;   while (...): endwhile;

// intentional infinite loop with exit
while (true) { if ($done) break; }

// safe file read
while (($line = fgets($fh)) !== false) { /* ... */ }
```

```blade
{{-- Blade --}}
@foreach ($items as $item) ... @endforeach
@forelse ($items as $item) ... @empty No items. @endforelse
@for ($i = 0; $i < 3; $i++) ... @endfor
@while ($cond) ... @endwhile

{{-- $loop metadata --}}
{{ $loop->index }} {{ $loop->iteration }} {{ $loop->count }}
{{ $loop->first }} {{ $loop->last }} {{ $loop->even }} {{ $loop->odd }}
{{ $loop->remaining }} {{ $loop->depth }} {{ $loop->parent->iteration }}
```

```php
// Laravel memory-safe DB iteration
User::chunk(500, fn ($users) => /* ... */);
User::where(...)->chunkById(500, fn ($users) => /* ... */);  // safe under mutation
foreach (User::cursor() as $u) { /* single query, streamed */ }
foreach (User::lazy() as $u)   { /* chunked queries, generator */ }
```

| Construct  | Condition checked | Min runs | Best for                                  |
|------------|-------------------|----------|-------------------------------------------|
| `for`      | before            | 0        | numeric ranges, index arithmetic          |
| `while`    | before            | 0        | unknown count, stream/poll until done     |
| `do-while` | after             | 1        | "act first, then decide", input prompts   |
| `foreach`  | per element       | 0        | arrays & `Traversable` (the default tool) |

---

## 🧪 Mini Exercises

1. **FizzBuzz, two ways.** Print numbers 1–30, replacing multiples of 3 with "Fizz", multiples of 5 with "Buzz", and multiples of both with "FizzBuzz". Implement it once with a `for` loop and once with a `foreach` over `range(1, 30)`, using `match(true)` for the branching.

2. **Reference bug detective.** Write a snippet that triggers the reference-after-foreach bug (last element clobbered), then fix it. In a comment, explain *exactly* why `$arr`'s last element ends up wrong before the fix.

3. **Multiplication table with `break N`.** Print a 9×9 multiplication grid, but stop *all* looping entirely the moment a product first exceeds 50 — using `break 2` (no flag variables).

4. **Group and skip with `continue N`.** Given `$matrix = [[1,2,3],[4,5,6],[7,8,9]]`, iterate every cell; whenever you hit a value divisible by 4, skip the rest of that row using `continue 2`. Print each cell you actually visit.

5. **Laravel streaming.** Assume a `posts` table with millions of rows. Write code that marks every post older than one year as `archived = true`, processing in batches of 1,000 *without* loading all rows into memory, and choosing the chunking method that won't skip rows given that you're modifying a filtered column. Justify your choice in a comment.
