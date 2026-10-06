# Advanced PHP: Generators, Iterators, SPL & Regex

> Module 19 — "Zero to Expert" PHP/Laravel course. Target: **PHP 8.4** (with notes for 8.1–8.3) and **Laravel 12** (with notes for Laravel 10/11).

This module is about the machinery that lets PHP work with *streams of data* instead of *blobs of data*, the standard library data structures most developers never learn (and therefore reinvent badly), the pattern-matching engine baked into the language, and a handful of low-level features (output buffering, variable variables, references) that show up in framework internals and interview questions.

## **What you'll learn**

- How `Traversable`, `Iterator`, and `IteratorAggregate` make any object usable in `foreach`, and when to pick each.
- Generators (`yield`, `yield key => value`, `yield from`, return values, `send()` coroutines) and *why* they slash memory usage versus building arrays.
- The SPL data structures (`SplStack`, `SplQueue`, `SplDoublyLinkedList`, `SplPriorityQueue`, `SplObjectStorage`, `SplFixedArray`, `ArrayObject`, `ArrayIterator`) and when each beats a plain array.
- The SPL interfaces `Countable` and `ArrayAccess` to make your objects behave like built-ins.
- PCRE regex end-to-end: `preg_match`, `preg_match_all`, `preg_replace`, `preg_replace_callback`, `preg_split`, delimiters, modifiers, anchors, classes, quantifiers (greedy vs lazy), groups, backreferences, named groups, and lookaround.
- Output buffering (`ob_start` / `ob_get_clean`), variable variables, and how PHP references *really* work.

---

## 1. Iterators: making objects walk in `foreach`

### Why this exists

`foreach` is PHP's universal looping construct, but out of the box it only knows how to walk **arrays** and objects' **public properties**. If you build a custom collection class, looping over its *internal* data should not require exposing that data publicly. Iterators solve this: they are a contract that says "here is how to walk me," and `foreach` honors that contract.

The base type is the marker interface **`Traversable`**. You cannot implement `Traversable` directly — it has no methods. Instead you implement one of its two children:

- **`Iterator`** — you write the cursor logic yourself (`current`, `key`, `next`, `rewind`, `valid`).
- **`IteratorAggregate`** — you delegate iteration to *another* traversable by returning it from `getIterator()`.

Everything `foreach` can loop is, under the hood, `Traversable` (arrays are special-cased in the engine, but conceptually the same).

### The `Iterator` interface (you own the cursor)

```php
<?php

/**
 * @implements Iterator<int, string>
 */
final class FibonacciIterator implements Iterator
{
    private int $key = 0;
    private int $current = 0;
    private int $previous = 1;

    public function __construct(private readonly int $limit) {}

    public function rewind(): void      // called once, at the start of foreach
    {
        $this->key = 0;
        $this->current = 0;
        $this->previous = 1;
    }

    public function valid(): bool       // is the current position valid? (controls loop end)
    {
        return $this->key < $this->limit;
    }

    public function current(): int      // the value for the current position
    {
        return $this->current;
    }

    public function key(): int          // the key for the current position
    {
        return $this->key;
    }

    public function next(): void        // advance the cursor
    {
        [$this->previous, $this->current] = [$this->current, $this->current + $this->previous];
        $this->key++;
    }
}

foreach (new FibonacciIterator(7) as $i => $n) {
    echo "$i => $n\n";
}
```

`foreach` calls these methods in a fixed order: `rewind()` once, then a loop of `valid()` → `current()` / `key()` → `next()`.

Output:

```
0 => 0
1 => 1
2 => 1
3 => 2
4 => 3
5 => 5
6 => 8
```

### `IteratorAggregate` (delegate the cursor)

Writing five methods by hand is tedious and error-prone. Most of the time your class *already wraps* an array or another iterable. `IteratorAggregate` lets you return a ready-made iterator (an `ArrayIterator`, or a generator — more on that soon):

```php
<?php

/**
 * @implements IteratorAggregate<int, string>
 */
final class TagCollection implements IteratorAggregate, Countable
{
    /** @param list<string> $tags */
    public function __construct(private array $tags = []) {}

    public function add(string $tag): void
    {
        $this->tags[] = $tag;
    }

    public function getIterator(): Iterator
    {
        return new ArrayIterator($this->tags);
    }

    public function count(): int
    {
        return count($this->tags);
    }
}

$tags = new TagCollection(['php', 'laravel']);
$tags->add('pcre');

foreach ($tags as $tag) {
    echo "#$tag ";
}
echo "\nTotal: " . count($tags);
```

Output:

```
#php #laravel #pcre 
Total: 3
```

> **Laravel connection:** `Illuminate\Support\Collection` implements `IteratorAggregate` (its `getIterator()` returns an `ArrayIterator` over the underlying items) plus `Countable`, `ArrayAccess`, and `JsonSerializable`. That is *exactly* why you can `foreach` a collection, `count()` it, and use `$collection['key']` syntax. Eloquent's `LazyCollection` instead wraps a **generator** to stream rows — covered next.

---

## 2. Generators: streaming data without building arrays

### Why this exists

Imagine reading a 2 GB log file. The naive approach — `file()` returns an array of every line — must hold the entire file in memory. A **generator** lets you produce values **one at a time, on demand**, so memory stays flat regardless of dataset size. A generator function looks like a normal function but contains the `yield` keyword; calling it does *not* run the body — it returns a `Generator` object (which implements `Iterator`).

### Basic `yield`

```php
<?php

function countTo(int $n): Generator
{
    for ($i = 1; $i <= $n; $i++) {
        yield $i;       // pause here, hand $i to the caller, resume on next iteration
    }
}

foreach (countTo(3) as $value) {
    echo "$value ";
}
// Output: 1 2 3
```

Each `yield` *suspends* the function, preserving all local state, and resumes exactly where it left off when the consumer asks for the next value. Nothing after a `yield` runs until iteration continues.

### Memory: array vs generator

```php
<?php

function rangeArray(int $n): array
{
    $out = [];
    for ($i = 0; $i < $n; $i++) {
        $out[] = $i;
    }
    return $out;                         // entire array materialized in RAM
}

function rangeGen(int $n): Generator
{
    for ($i = 0; $i < $n; $i++) {
        yield $i;                        // one int alive at a time
    }
}

// Building 1,000,000 ints as an array costs tens of MB.
// The generator's footprint is roughly constant.
echo memory_get_peak_usage();
```

Reading a large file line-by-line is the canonical real-world case:

```php
<?php

function readLines(string $path): Generator
{
    $handle = fopen($path, 'rb');
    if ($handle === false) {
        throw new RuntimeException("Cannot open {$path}");
    }
    try {
        while (($line = fgets($handle)) !== false) {
            yield rtrim($line, "\r\n");
        }
    } finally {
        fclose($handle);                 // runs even if the consumer stops early
    }
}

foreach (readLines('huge.log') as $line) {
    if (str_contains($line, 'ERROR')) {
        echo $line, "\n";
    }
}
```

> **Trade-off:** generators are **forward-only and single-pass**. You cannot `count()` a generator, index into it, or rewind it (calling `rewind()` after iteration has begun throws `Exception: Cannot rewind a generator that was already run`). If you need random access or multiple passes, use an array or buffer it.

### `yield key => value`

You control the key, not just the value:

```php
<?php

function envPairs(): Generator
{
    yield 'driver' => 'mysql';
    yield 'host'   => '127.0.0.1';
    yield 'port'   => 3306;
}

foreach (envPairs() as $key => $value) {
    echo "$key = $value\n";
}
```

Output:

```
driver = mysql
host = 127.0.0.1
port = 3306
```

If you never specify a key, generators auto-number from 0 like a list — **but** mixing `yield $v;` and `yield $k => $v;` continues the integer sequence independently, which can surprise you. Be consistent.

### `yield from` (delegation)

`yield from` flattens another iterable (array, generator, or any `Traversable`) into the current generator. It is the composition primitive for generators:

```php
<?php

function evens(): Generator
{
    yield 2;
    yield 4;
}

function numbers(): Generator
{
    yield 1;
    yield from evens();   // delegate: re-yield everything evens() produces
    yield 5;
}

echo implode(' ', iterator_to_array(numbers(), false));
// Output: 1 2 4 5
```

> **Key gotcha:** `yield from` an **array** or another generator *preserves the inner keys*. With integer keys this means duplicates and overwrites if you collect with `iterator_to_array(..., true)`. Pass `false` as the second argument to `iterator_to_array` to renumber and avoid clobbering. (The auto-numbering of the *outer* generator is also not advanced by `yield from`.)

### Returning a value from a generator

A generator's `return` value is separate from its yielded values. Retrieve it with `getReturn()` *after* iteration completes:

```php
<?php

function parse(array $rows): Generator
{
    $valid = 0;
    foreach ($rows as $row) {
        if ($row !== null) {
            $valid++;
            yield $row;
        }
    }
    return $valid;                       // summary, available after the loop
}

$gen = parse(['a', null, 'b']);
foreach ($gen as $r) { /* ... */ }
echo $gen->getReturn();                  // Output: 2
```

`yield from` *also* captures the inner generator's return value as the expression result: `$total = yield from parse($rows);`.

### `send()` — generators as coroutines

`yield` is a two-way street. The consumer can push a value *back into* the generator with `send()`, and that value becomes the result of the `yield` expression inside the generator. This turns a generator into a lightweight **coroutine** (a function that can pause and be resumed with new input).

```php
<?php

function runningTotal(): Generator
{
    $total = 0;
    while (true) {
        $delta = yield $total;           // emit current total, wait for next delta
        $total += $delta;
    }
}

$acc = runningTotal();
$acc->current();          // prime: runs to the first `yield`. Returns 0 (not echoed here).
echo $acc->send(10);      // send 10 -> $total becomes 10, next yield emits 10
echo ' ';
echo $acc->send(5);       // 15
echo ' ';
echo $acc->send(100);     // 115
```

Output:

```
10 15 115
```

> **Priming rule:** before the first `send()`, the generator must be advanced to its first `yield` (via `current()`, `next()`, or `rewind()`); otherwise the first sent value is discarded. This pattern underpins async runtimes (ReactPHP, Amp pre-fibers). **PHP 8.1 added Fibers**, which are a more general primitive for cooperative multitasking and have largely superseded `send()`-based coroutines in modern async libraries — but `send()` still appears in interviews and legacy async code.

---

## 3. SPL: the Standard PHP Library

The SPL ships data structures implemented in C that are faster and more memory-efficient than hand-rolled array equivalents, and that express intent (a queue *is* FIFO; an array is whatever you treat it as). No `composer require` needed — they are part of core.

### `SplStack` — LIFO (last in, first out)

```php
<?php

$stack = new SplStack();
$stack->push('a');
$stack->push('b');
$stack->push('c');

echo $stack->top();   // 'c' (peek without removing)
echo $stack->pop();   // 'c' (remove and return)
echo $stack->pop();   // 'b'
echo count($stack);   // 1   (SplStack is Countable)
```

### `SplQueue` — FIFO (first in, first out)

```php
<?php

$queue = new SplQueue();
$queue->enqueue('first');
$queue->enqueue('second');

echo $queue->dequeue();   // 'first'
echo $queue->dequeue();   // 'second'
```

Both `SplStack` and `SplQueue` extend **`SplDoublyLinkedList`**, which gives O(1) inserts/removes at *both* ends — use it directly when you need a deque (double-ended queue).

### `SplPriorityQueue` — highest priority first

```php
<?php

$pq = new SplPriorityQueue();
$pq->insert('low task', 1);
$pq->insert('urgent task', 10);
$pq->insert('normal task', 5);

while (!$pq->isEmpty()) {
    echo $pq->extract(), "\n";
}
```

Output:

```
urgent task
normal task
low task
```

> **Gotcha:** `SplPriorityQueue` is **not stable** — two items with equal priority have an undefined relative order. To force FIFO-within-priority, make the priority a composite like `[$priority, -$insertionSequence]` (SPL compares arrays element-by-element).

### `SplObjectStorage` — a set/map keyed by object identity

A plain array cannot use objects as keys. `SplObjectStorage` can, keying by **object identity** (`spl_object_id`), and lets you attach arbitrary data to each object. It is ideal for "have I seen this object?" sets and object→metadata maps.

```php
<?php

$visited = new SplObjectStorage();

$user1 = new stdClass();
$user2 = new stdClass();

$visited->attach($user1, ['seen_at' => 'login']);
$visited->attach($user2);

var_dump($visited->contains($user1));  // bool(true)
echo $visited[$user1]['seen_at'];      // 'login'  (ArrayAccess to the attached data)
echo count($visited);                  // 2

$visited->detach($user1);
var_dump($visited->contains($user1));  // bool(false)
```

> **Laravel uses this internally** — e.g. tracking observed model instances and avoiding duplicate work on the same object.

### `SplFixedArray` — fixed-size, integer-indexed, low overhead

A regular PHP array is a flexible ordered hash map, which costs memory. `SplFixedArray` is a fixed-length array with **integer indices only**, using far less memory per element — useful for large numeric datasets.

```php
<?php

$arr = new SplFixedArray(3);
$arr[0] = 'x';
$arr[1] = 'y';
$arr[2] = 'z';

echo $arr->getSize();   // 3
$arr->setSize(5);       // resize (new slots are null)
// $arr[10] = 'oops';   // RuntimeException: Index invalid or out of range
```

> Since PHP 7.0 the Zend engine stores list-like (sequential integer-keyed) arrays as compact "packed" arrays, which narrowed much of the memory gap, but `SplFixedArray` still wins for very large fixed numeric arrays (it stores raw values with no hash-table bucket overhead) and signals intent.

### `ArrayObject` and `ArrayIterator` — arrays that are objects

`ArrayObject` wraps an array (or public object properties) so it can be passed **by handle** (object semantics: shared, not copied) while still supporting `[]` access, `count()`, and iteration. `ArrayIterator` is the iterator it hands out, and you can use it standalone.

```php
<?php

$config = new ArrayObject(['debug' => false, 'cache' => true]);
$config['debug'] = true;

function disableCache(ArrayObject $c): void
{
    $c['cache'] = false;                 // mutates the original — objects pass by handle
}
disableCache($config);

foreach ($config as $key => $value) {
    echo "$key: " . var_export($value, true) . "\n";
}
echo count($config);                     // 2
```

Output:

```
debug: true
cache: false
2
```

> **Why it matters:** plain arrays are **value types** — passing one to a function copies it (copy-on-write). `ArrayObject` gives you reference-like sharing without `&`, which is cleaner when you genuinely want shared mutable state.

### SPL interfaces: `Countable` and `ArrayAccess`

These let *your* objects behave like built-ins.

**`Countable`** — makes `count($obj)` work via a `count()` method (you saw it on `TagCollection` above).

**`ArrayAccess`** — makes `$obj['key']` work via four methods:

```php
<?php

/**
 * @implements ArrayAccess<string, mixed>
 */
final class Config implements ArrayAccess, Countable
{
    /** @param array<string, mixed> $items */
    public function __construct(private array $items = []) {}

    public function offsetExists(mixed $offset): bool
    {
        return isset($this->items[$offset]);
    }

    public function offsetGet(mixed $offset): mixed
    {
        return $this->items[$offset] ?? null;
    }

    public function offsetSet(mixed $offset, mixed $value): void
    {
        if ($offset === null) {          // $config[] = 'x' appends
            $this->items[] = $value;
        } else {
            $this->items[$offset] = $value;
        }
    }

    public function offsetUnset(mixed $offset): void
    {
        unset($this->items[$offset]);
    }

    public function count(): int
    {
        return count($this->items);
    }
}

$c = new Config(['env' => 'prod']);
$c['debug'] = false;
echo $c['env'];              // 'prod'
var_dump(isset($c['x']));    // bool(false)  -> calls offsetExists
unset($c['debug']);
echo count($c);              // 1
```

> **Gotcha:** `isset($obj['x'])` calls `offsetExists`, **not** `offsetGet`. If `offsetGet` returns `null` for a key that *does* exist with a `null` value, `isset()` will still report `false` unless your `offsetExists` is written carefully. This is the same quirk as `isset()` on arrays with `null` values. Laravel's `Collection`, request bag, and `Fluent` all lean on `ArrayAccess`.

---

## 4. PCRE Regex in PHP

PHP's regex engine is **PCRE** (Perl Compatible Regular Expressions). A regular expression is a pattern that describes a *set of strings*; the engine tells you whether/where a subject string matches the pattern.

### Delimiters

Unlike JavaScript, PHP regex patterns are **strings**, so the pattern must be wrapped in delimiters. The most common is `/`, but any non-alphanumeric, non-backslash, non-whitespace character works — pick one that doesn't appear in the pattern to avoid escaping:

```php
<?php

preg_match('/https?/', $url);        // / delimiter — but URLs contain slashes...
preg_match('#https?://#', $url);     // # delimiter — no need to escape the slashes
preg_match('~\d+~', $text);          // ~ delimiter
```

Paired delimiters `()`, `{}`, `[]`, `<>` are also allowed: `'{\d+}'`.

### `preg_match` — find the first match

Returns `1` (match), `0` (no match), or `false` (error). The third argument captures groups by reference.

```php
<?php

$subject = 'Order #4815 shipped on 2026-06-18';

if (preg_match('/#(\d+)/', $subject, $m)) {
    echo $m[0];   // '#4815'  (full match)
    echo $m[1];   // '4815'   (first capture group)
}
```

> **Always check the return value, not the match array.** `preg_match` returns `false` on a *malformed pattern or engine error* (which is falsy but distinct from "no match" = `0`). Use `=== 1` if you need to distinguish, and check `preg_last_error()` / `preg_last_error_msg()` (PHP 8.0+) when it returns `false`.

### `preg_match_all` — find every match

```php
<?php

$html = '<a href="/a">A</a> <a href="/b">B</a>';
preg_match_all('/href="([^"]+)"/', $html, $matches);

print_r($matches[1]);
// Output: Array ( [0] => /a  [1] => /b )
```

By default `$matches` is grouped **by capture group** (`PREG_PATTERN_ORDER`): `$matches[0]` = all full matches, `$matches[1]` = all first-group captures. Pass `PREG_SET_ORDER` to instead get one sub-array per match.

### `preg_replace` and `preg_replace_callback`

```php
<?php

// Backreference $1 (or \1) in the replacement refers to capture group 1.
echo preg_replace('/(\d{4})-(\d{2})-(\d{2})/', '$3/$2/$1', '2026-06-18');
// Output: 18/06/2026

// When the replacement needs logic, use a callback:
echo preg_replace_callback(
    '/\b\w/',
    fn(array $m): string => strtoupper($m[0]),
    'hello world'
);
// Output: Hello World
```

> Never build replacement logic by concatenating user input into a `preg_replace` *pattern*. Use `preg_replace_callback` for anything conditional — it is also safer than the long-removed `/e` modifier (deleted in PHP 7.0).

### `preg_split` — split on a pattern

```php
<?php

// Split on one-or-more whitespace, dropping empty pieces.
print_r(preg_split('/\s+/', "one  two\tthree\nfour", -1, PREG_SPLIT_NO_EMPTY));
// Output: Array ( [0] => one [1] => two [2] => three [3] => four )
```

### `preg_quote` — escape dynamic input before embedding it in a pattern

If you build a pattern from a variable (e.g. a user-supplied search term), any regex metacharacter in that input (`.`, `*`, `(`, `[`, `\`, …) changes the pattern's meaning — at best a wrong match, at worst a denial-of-service via catastrophic backtracking. `preg_quote` escapes those metacharacters. **Always pass your delimiter as the second argument** so it gets escaped too.

```php
<?php

$term = 'a.b+c (test)';
$pattern = '/' . preg_quote($term, '/') . '/i';   // delimiter '/' escaped too
// $pattern is now: /a\.b\+c \(test\)/i  — metacharacters escaped, literal text preserved
echo preg_match($pattern, 'A.B+C (TEST) found'), "\n";   // 1
```

> **Security:** never concatenate untrusted input straight into a pattern. Unescaped input lets an attacker inject regex constructs (a so-called *ReDoS* attack) that hang the request. `preg_quote` is the fix when you need a *literal* match; for non-pattern logic, prefer `str_contains` / `str_replace`.

### Modifiers (flags after the closing delimiter)

| Flag | Name | Effect |
|------|------|--------|
| `i` | caseless | case-insensitive matching |
| `m` | multiline | `^`/`$` match at line breaks, not just string ends |
| `s` | dotall | `.` also matches newline |
| `u` | unicode | treat pattern & subject as UTF-8; `\w`, `\d` etc. become Unicode-aware |
| `x` | extended | ignore whitespace in the pattern, allow `#` comments — for readability |

```php
<?php

preg_match('/error/i', 'ERROR');        // matches (case-insensitive)
preg_match('/café/u', $utf8Text);       // correct UTF-8 handling

// Extended mode keeps complex patterns readable:
$pattern = '/
    ^(\d{4})    # year
    -(\d{2})    # month
    -(\d{2})$   # day
/x';
preg_match($pattern, '2026-06-18', $m);
```

> **Use `u` whenever the subject may contain multibyte text.** Without it, `.` matches single *bytes*, which can split a UTF-8 character and corrupt output. An invalid UTF-8 subject under `/u` makes `preg_match` return `false` with `preg_last_error() === PREG_BAD_UTF8_ERROR`.

### Anchors, character classes, quantifiers

```
^        start of string (or line with /m)
$        end of string (or before trailing \n; line with /m)
\b       word boundary           \B  non-boundary
\d \w \s digit / word-char / whitespace   (uppercase = negation: \D \W \S)
[abc]    any one of a, b, c      [^abc] none of them
[a-z0-9] ranges
.        any char except newline (unless /s)
```

**Quantifiers** control repetition:

```
*    0 or more      +    1 or more     ?    0 or 1
{3}  exactly 3      {2,} 2 or more     {2,5} between 2 and 5
```

### Greedy vs lazy

By default quantifiers are **greedy** — they match as much as possible, then backtrack. Adding `?` makes them **lazy** — match as little as possible.

```php
<?php

$html = '<b>one</b><b>two</b>';

preg_match('/<b>(.*)<\/b>/', $html, $g);    // greedy
echo $g[1];   // 'one</b><b>two'   <- swallowed everything to the LAST </b>

preg_match('/<b>(.*?)<\/b>/', $html, $l);   // lazy
echo $l[1];   // 'one'              <- stops at the FIRST </b>
```

This greedy/lazy distinction is one of the most common regex bugs in the wild.

### Groups, backreferences, named groups

```php
<?php

// Backreference inside the PATTERN: \1 matches the same text group 1 captured.
preg_match('/(\w)\1/', 'balloon', $m);
echo $m[0];   // 'll'  (a doubled letter)

// Named groups: (?P<name>...) or (?<name>...)
preg_match('/(?<year>\d{4})-(?<month>\d{2})/', '2026-06', $m);
echo $m['year'];    // '2026'
echo $m['month'];   // '06'

// Non-capturing group (?:...) — group for alternation/quantifying without capturing.
preg_match('/(?:https?|ftp):\/\//', 'ftp://x', $m);
echo $m[0];   // 'ftp://'  (no extra capture slot consumed)
```

> Named groups are far more maintainable than numeric `$m[3]` indices. Prefer them in any pattern with more than one or two groups.

### Lookahead and lookbehind (zero-width assertions)

Lookaround checks that something does (or does not) appear *next to* the match **without consuming characters**.

```
(?=...)   positive lookahead   — followed by ...
(?!...)   negative lookahead   — NOT followed by ...
(?<=...)  positive lookbehind  — preceded by ...
(?<!...)  negative lookbehind  — NOT preceded by ...
```

```php
<?php

// Password rule: at least one digit AND one uppercase, min 8 chars — using lookaheads.
$strong = '/^(?=.*\d)(?=.*[A-Z]).{8,}$/';
var_dump(preg_match($strong, 'Hello123') === 1);   // bool(true)
var_dump(preg_match($strong, 'hello123') === 1);   // bool(false) — no uppercase

// Extract the number after "$" using lookbehind (the "$" is not part of the match).
preg_match('/(?<=\$)\d+(?:\.\d+)?/', 'Total: $42.50', $m);
echo $m[0];   // '42.50'
```

> PCRE lookbehind must be **fixed-width** in classic mode (no `*`/`+` of variable length directly), though alternations of fixed lengths are allowed. PCRE2 (used since PHP 7.3) relaxes some of this with `\K` and variable-length lookbehind support in newer builds, but for portability assume fixed-width.

### Common production-ready patterns

```php
<?php

// These are pragmatic, not RFC-exhaustive. For real email validation prefer filter_var().
$slug      = '/^[a-z0-9]+(?:-[a-z0-9]+)*$/';        // url-slug
$ipv4octet = '/^(?:25[0-5]|2[0-4]\d|1?\d?\d)$/';    // 0-255
$uuid      = '/^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i';

// In Laravel, regex shows up in validation rules:
// $request->validate(['handle' => ['required', 'regex:/^@\w{1,15}$/']]);
```

> **For emails/URLs, prefer `filter_var($v, FILTER_VALIDATE_EMAIL)` / `FILTER_VALIDATE_URL`** over hand-written regex — the truly correct email regex is monstrous and `filter_var` is battle-tested.

---

## 5. Output buffering

### Why this exists

Normally `echo` sends bytes straight to the client (or stdout). **Output buffering** captures that output into an in-memory buffer instead, so you can inspect, modify, discard, or capture it as a string. It is how template engines render to a variable, and it lets you set HTTP headers *after* you've already "printed" content (because nothing was actually sent yet).

```php
<?php

ob_start();                  // start capturing
echo "<h1>Hello</h1>";
$captured = ob_get_clean();  // get the buffer contents AND stop+discard the buffer

echo strtoupper($captured);  // Output: <H1>HELLO</H1>
```

Related calls:

- `ob_get_contents()` — read buffer without clearing it.
- `ob_get_clean()` — read buffer **and** end buffering (most common for "render to string").
- `ob_end_clean()` — discard buffer and stop (throw output away).
- `ob_end_flush()` / `ob_flush()` — send the buffer onward.
- `ob_get_level()` — current nesting depth (buffers stack).

A classic "render a PHP template to a string" helper (exactly what Blade compiles down toward):

```php
<?php

function renderTemplate(string $file, array $data = []): string
{
    extract($data, EXTR_SKIP);
    ob_start();
    try {
        include $file;           // the template's echoes go into the buffer
        return ob_get_clean();
    } catch (\Throwable $e) {
        ob_end_clean();          // don't leak a half-rendered buffer on error
        throw $e;
    }
}
```

> **Gotcha:** every `ob_start()` must be balanced by an `ob_get_clean()`/`ob_end_*()`. An unbalanced buffer can swallow output or leak memory. On error, end the buffer in a `catch`/`finally` (as above).

---

## 6. Variable variables

A **variable variable** uses the *value* of one variable as the *name* of another. Syntax: `$$name`.

```php
<?php

$field = 'email';
$$field = 'a@b.com';     // creates $email
echo $email;             // 'a@b.com'

// Curly braces are REQUIRED when the name is an expression or for array members:
$key = 'user_id';
${$key} = 42;            // creates $user_id
echo $user_id;           // 42
```

> **Use sparingly.** Variable variables defeat static analysis, IDE autocompletion, and refactoring tools, and are a common source of bugs and even security holes (e.g. the old `register_globals`-style mass assignment). In modern code, an associative array or object property is almost always clearer: `$data[$field] = ...`. They remain worth *recognizing* in interviews and legacy code.

---

## 7. References in depth

### Why this matters

PHP variables are *names bound to values*. A **reference** (`&`) makes two names point at the **same** underlying value — assigning through one changes the other. Misunderstanding references (especially the `foreach` + `&` trap) causes some of the nastiest, hardest-to-spot PHP bugs.

### Reference assignment

```php
<?php

$a = 1;
$b = &$a;     // $b is now an alias of $a
$b = 99;
echo $a;      // 99  — they share one value
```

### Objects are *not* references — they are handles

A frequent confusion: object variables hold a **handle** (an identifier) to the object, not the object itself and not a reference.

```php
<?php

$o1 = new stdClass();
$o2 = $o1;          // copies the HANDLE, not the object
$o2->x = 5;
echo $o1->x;        // 5  — same object, two handles

$o2 = new stdClass(); // reassigns $o2's handle only
echo isset($o1->x) ? 'set' : 'unset';  // 'set' — $o1 still points at the original
```

So you rarely need `&` with objects; you only need it to make one variable *follow reassignments* of another.

### References in `foreach` — the #1 trap

```php
<?php

$nums = [1, 2, 3];

foreach ($nums as &$n) {     // $n is a reference into the array
    $n *= 2;
}
// $nums is now [2, 4, 6] — good, this was intentional.

unset($n);                   // <-- CRUCIAL: break the reference

foreach ($nums as $n) {      // without the unset above, this loop corrupts $nums!
    // ...
}
```

Without the `unset($n)`, `$n` still references the **last** element. The second `foreach` then assigns each iterated value *into that last slot*, leaving `$nums` as `[2, 4, 4]` (the last element gets overwritten by the second-to-last value, then itself). Always `unset()` a foreach reference variable immediately after the loop.

### References as function parameters

```php
<?php

function addTax(float &$price, float $rate = 0.2): void
{
    $price += $price * $rate;   // mutates the caller's variable
}

$total = 100.0;
addTax($total);
echo $total;   // 120
```

Built-ins use this too: `sort(&$array)`, `preg_match($p, $s, &$matches)`, `array_push(&$arr, ...)`.

> **Gotcha:** you cannot pass a literal or an expression to a by-reference parameter (`addTax(100.0)` is a fatal error in modern PHP) — only a variable, since there must be something to alias.

### References and `unset`

`unset()` on a reference only breaks *that* binding; the other name keeps the value.

```php
<?php

$x = 5;
$y = &$x;
unset($y);    // breaks $y's binding only
echo $x;      // 5 — still there
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `unset($v)` after `foreach (... as &$v)`.** The dangling reference makes the *next* loop silently corrupt the array. **Fix:** always `unset()` the reference immediately after the loop, or avoid `&` and assign back by index.

2. **Trying to rewind/recount a generator.** Generators are single-pass and forward-only; `count($gen)` is a `TypeError` and re-iterating throws "Cannot rewind a generator that was already run." **Fix:** if you need multiple passes or a count, collect with `iterator_to_array($gen)` (which loses the lazy benefit) or redesign so one pass suffices. In Laravel, `LazyCollection` makes the single-pass nature explicit.

3. **Treating `preg_match`'s return like a boolean for errors.** It returns `0` for "no match" and `false` for "pattern error" — both falsy. **Fix:** compare `=== 1` / `=== 0` / `=== false` and inspect `preg_last_error_msg()` when `false`.

4. **Greedy quantifiers swallowing too much.** `/<.*>/` on `<a><b>` matches the whole `<a><b>`. **Fix:** use the lazy form `/<.*?>/`, or a negated class `/<[^>]*>/` which is both correct and faster (no catastrophic backtracking).

5. **Missing the `/u` modifier on multibyte text.** Without it, `.`, `\w`, and quantifiers operate on bytes, corrupting UTF-8. **Fix:** add `u` whenever the subject can contain non-ASCII; validate that the subject is valid UTF-8 (a `/u` match on invalid UTF-8 returns `false`).

6. **Unbalanced output buffers.** An `ob_start()` without a matching close swallows later output or leaks memory; an exception mid-buffer leaves a half-rendered buffer open. **Fix:** pair every `ob_start()` with `ob_get_clean()`/`ob_end_*()`, and clean up in `catch`/`finally`.

7. **Assuming `SplPriorityQueue` is stable.** Equal-priority items come out in arbitrary order. **Fix:** encode an insertion counter into the priority to break ties deterministically.

8. **`isset()` vs `offsetExists` with null values.** `isset($obj['k'])` calls `offsetExists`, not `offsetGet`; a key whose value is `null` can read as "not set." **Fix:** implement `offsetExists` with `array_key_exists` semantics if `null` values must be distinguishable from missing keys.

9. **Concatenating raw input into a regex pattern.** `'/' . $userTerm . '/'` lets metacharacters in `$userTerm` change the pattern — wrong matches at best, ReDoS (catastrophic backtracking that hangs the request) at worst. **Fix:** `preg_quote($userTerm, '/')` for literal matches; or skip regex entirely and use `str_contains` / `str_replace`.

10. **Passing a literal/expression to a by-reference parameter.** `addTax(100.0)` where `addTax(float &$price)` throws `Error: ... cannot be passed by reference` — there is nothing to alias. **Fix:** pass a variable, or drop `&` and return the new value instead.

---

## ✅ Best Practices

- **Reach for generators** any time you process a large or unbounded sequence (file lines, paginated API pages, DB cursors). In Laravel use `Model::cursor()` / `lazy()` and `LazyCollection`, which are generator-backed.
- **Prefer `IteratorAggregate` + a generator in `getIterator()`** over hand-writing the five `Iterator` methods — less code, no cursor bugs.
- **Pick the SPL structure that names your intent**: `SplQueue` for FIFO, `SplStack` for LIFO, `SplPriorityQueue` for scheduling, `SplObjectStorage` for object sets/maps. It documents the code and is C-fast.
- **Implement `Countable` / `ArrayAccess`** to give value-object collections an ergonomic, array-like API — but keep them immutable where you can.
- **Compile-once regex**: keep patterns as constants. PCRE caches compiled patterns internally, but constants also aid readability. Use named groups and `/x` for any non-trivial pattern.
- **Validate with the right tool**: `filter_var` for emails/URLs/IPs; regex for *structural* matching and extraction.
- **Escape dynamic input with `preg_quote($input, $delimiter)`** before embedding it in a pattern, or avoid regex entirely (`str_contains`/`str_replace`) when you only need a literal match — this closes the ReDoS injection door.
- **Avoid variable variables and `extract()` on untrusted data**; they erase static analysis and invite bugs/security issues.
- **Default to value semantics**; reach for references only when you have a measured reason, and always `unset` foreach reference variables.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `Iterator` and `IteratorAggregate`?**
A. `Iterator` requires you to implement the cursor (`current/key/next/rewind/valid`). `IteratorAggregate` only requires `getIterator()` returning a `Traversable` (often an `ArrayIterator` or a generator), delegating the cursor. Both extend `Traversable`, which `foreach` requires but you can't implement directly.

**Q2. How does a generator save memory compared to building an array?**
A. A generator produces values lazily, one at a time, suspending execution at each `yield`. Only the current value and the generator's local state live in memory, so footprint is roughly constant regardless of how many values flow through — versus an array, which materializes every element at once.

**Q3. (Under the hood) What actually happens when PHP hits `yield`?**
A. Calling a generator function doesn't run the body; it returns a `Generator` object implementing `Iterator`. The engine keeps the function's execution frame (instruction pointer + local variable table) on the heap. Each `valid()`/`current()`/`next()` from `foreach` resumes that frame, runs until the next `yield`, snapshots state, and suspends. `send()` resumes the frame and injects its argument as the value of the `yield` expression — that's the coroutine mechanism. Fibers (8.1+) generalize this to arbitrary stack switching.

**Q4. What does `yield from` do, and what's the catch with keys?**
A. It delegates: re-yields every value from an inner iterable into the outer generator, and evaluates to the inner generator's `getReturn()`. The catch: it preserves the inner iterable's keys, so integer keys can collide/overwrite when collected with `iterator_to_array($gen, true)`. Pass `false` to renumber.

**Q5. When would you use `SplObjectStorage` over an array?**
A. When you need objects as keys or a set of objects keyed by identity — plain arrays can't key by object. It's O(1) for attach/detach/contains and can store metadata per object. Used for "seen this object?" checks, observer registries, and graph traversal visited-sets.

**Q6. Explain greedy vs lazy quantifiers with an example.**
A. Greedy (`*`, `+`) matches as much as possible then backtracks; lazy (`*?`, `+?`) matches as little as possible. On `<a><b>`, `<.*>` greedily matches the whole string, while `<.*?>` matches just `<a>`. Negated classes like `<[^>]*>` are often the best of both.

**Q7. What's the difference between a capturing group, non-capturing group, and named group?**
A. `(...)` captures into a numbered slot you can backreference (`\1`) or read (`$m[1]`). `(?:...)` groups for alternation/quantifying without consuming a capture slot (faster, cleaner). `(?<name>...)` captures into a named slot readable as `$m['name']` — more maintainable.

**Q8. What's the difference between a reference and an object handle in PHP?**
A. A reference (`&`) makes two *variable names* alias one value, so reassigning either changes both. An object variable holds a *handle* — copying it (`$b = $a`) copies the handle, so both point at the same object (mutations are visible through both), but reassigning one variable to a new object doesn't affect the other. References change reassignment behavior; handles don't.

**Q9. Why does this loop corrupt the array, and how do you fix it?**
```php
foreach ($a as &$v) {}
foreach ($a as $v) {}   // bug
```
A. After the first loop `$v` is still a reference to the last element. The second loop assigns each value into that last slot, overwriting it. Fix: `unset($v)` after the first loop (or don't use `&`).

**Q10. Where is output buffering used in real frameworks?**
A. Template engines render PHP templates to a string by capturing `echo` output via `ob_start`/`ob_get_clean` (Blade compiles to PHP that's executed this way). It also lets middleware capture/modify a response body and lets code set headers after generating "output," since nothing is sent until the buffer flushes.

**Q11. A search feature builds `'/' . $userInput . '/'` and matches it. What's wrong?**
A. Two problems. (1) Correctness: metacharacters in `$userInput` (`.`, `(`, `*`, the `/` delimiter) change the pattern, so the match misbehaves. (2) Security: a crafted input like `(a+)+$` causes catastrophic backtracking — a ReDoS that pins a CPU and hangs the request. Fix: `preg_quote($userInput, '/')` for a literal match, or use `str_contains`/`stripos` when no regex features are needed.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ---- Iterators ----
class C implements IteratorAggregate { public function getIterator(): Iterator { return new ArrayIterator($this->items); } }
// Iterator methods: rewind(), valid(), current(), key(), next()

// ---- Generators ----
function g() { yield $v; yield $k => $v; yield from $iterable; return $final; }
$gen->getReturn();          // value returned after iteration
$gen->send($x);             // coroutine: inject value (prime first!)
iterator_to_array($gen, false);  // collect; false => renumber keys

// ---- SPL structures ----
new SplStack();             // push/pop/top      (LIFO)
new SplQueue();             // enqueue/dequeue   (FIFO)
new SplDoublyLinkedList();  // push/pop/shift/unshift (deque)
new SplPriorityQueue();     // insert($v,$prio)/extract  (NOT stable)
new SplObjectStorage();     // attach/detach/contains; objects as keys
new SplFixedArray($n);      // fixed size, int index, low memory
new ArrayObject($arr);      // array w/ object (handle) semantics
new ArrayIterator($arr);    // standalone array iterator

// ---- SPL interfaces ----
Countable:   count(): int
ArrayAccess: offsetExists/offsetGet/offsetSet/offsetUnset

// ---- Regex functions ----
preg_match($p, $s, $m);          // first match; returns 1|0|false
preg_match_all($p, $s, $m);      // all matches (PREG_PATTERN_ORDER default)
preg_replace($p, $repl, $s);     // $1 / \1 backreferences in $repl
preg_replace_callback($p, fn, $s);
preg_split($p, $s, -1, PREG_SPLIT_NO_EMPTY);
preg_quote($input, '/');         // escape dynamic input + delimiter before embedding
preg_last_error_msg();           // diagnose a false return

// ---- Regex syntax ----
^ $ \b           anchors
\d \w \s         + negations \D \W \S
[abc] [^abc] [a-z]   classes
* + ? {n} {n,} {n,m} quantifiers   (+? *? => lazy)
( ) (?:...) (?<name>...)           groups
(?=) (?!) (?<=) (?<!)              lookahead / lookbehind
modifiers: i m s u x

// ---- Output buffering ----
ob_start(); echo '...'; $s = ob_get_clean();

// ---- References ----
$b = &$a;                    // alias
foreach ($a as &$v) {} unset($v);   // ALWAYS unset after
function f(&$x) {}           // by-reference param (variable only)
```

---

## 🧪 Mini Exercises

1. **Lazy CSV reader.** Write a generator `csvRows(string $path): Generator` that opens a CSV file, `yield`s each row as an associative array keyed by the header row, and `return`s the total number of data rows. Prove with `getReturn()` that it counted correctly, and confirm it only ever holds one row in memory.

2. **Custom collection.** Build a `Stack` value object that implements `IteratorAggregate`, `Countable`, and `ArrayAccess`. Iterating it should yield items top-to-bottom, `count()` should work, and `$stack[0]` should peek the top. Internally back it with an `SplStack` or `SplDoublyLinkedList`.

3. **Template renderer.** Implement `render(string $template, array $vars): string` using output buffering that includes a `.php` template, injects `$vars` as local variables, returns the rendered string, and never leaves a dangling buffer even if the template throws.

4. **Log parser with named groups.** Given lines like `[2026-06-18 14:03:55] production.ERROR: Disk full {context}`, write one regex with named groups (`date`, `env`, `level`, `message`) and use `preg_match_all` with `PREG_SET_ORDER` to return a list of structured arrays. Make it `/u`-safe.

5. **Coroutine accumulator.** Write a `send()`-based generator `collector()` that receives strings via `send()` and yields back the running concatenation, then returns the final string when sent `null`. Demonstrate priming and termination, and explain in a comment why the first sent value would be lost without priming.
