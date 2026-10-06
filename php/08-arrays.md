# Arrays in PHP (and How Laravel Builds on Them)

Arrays are the workhorse data structure of PHP. Unlike most languages, PHP has **one** array type that doubles as a list, a dictionary (hash map), a stack, a queue, and a set. Mastering arrays — and the ~80 `array_*` functions around them — is the single highest-leverage PHP skill for backend work, and it's exactly what Laravel's `Collection`, `Arr`, and query-builder result handling are built on top of.

> **Jargon up front:** an **array** in PHP is an *ordered map* — an ordered collection of key → value pairs. "Ordered" means it remembers insertion order. "Map" means every value has a key (an integer or a string). There is no separate "list" type at the language level; a "list" is just an array whose keys happen to be `0, 1, 2, …`.

---

## What you'll learn

- The three mental models of PHP arrays — indexed, associative, and multidimensional — and why they're all the *same* type under the hood.
- How to create, nest, add to, remove from, and safely access array elements (`isset` vs `array_key_exists` vs `in_array` vs `array_search`).
- The transformation trio every interview loves: `array_map`, `array_filter`, `array_reduce` (plus `array_walk`), and the **new PHP 8.4** predicate helpers `array_find`, `array_find_key`, `array_any`, `array_all`.
- Combining and merging — and the classic `array_merge` vs the `+` operator trap.
- Slicing, splicing, chunking, and the **entire** sorting family, including which sorts are *stable*.
- Modern syntax: the spread operator (including string-key spread, new in 8.1), `list()` / short destructuring with keys, and references in arrays.
- Using arrays as stacks and queues, plus the **copy-on-write** behavior that makes PHP arrays cheap to pass around.
- Where this maps to Laravel 12 (`Arr`, `collect()`, `data_get`).

---

## 1. The three faces of one array type

### Why: one structure, many uses

PHP collapses what other languages split into `array` / `List` / `Dictionary` / `Map` into a single ordered map. This is convenient but means you must understand keys deeply, because subtle key behavior (string-to-int coercion, key collisions, gaps) causes most array bugs.

### Indexed arrays (numeric keys)

An **indexed array** uses sequential integer keys starting at `0`. This is PHP's "list."

```php
<?php
$fruits = ['apple', 'banana', 'cherry'];   // short syntax (preferred since PHP 5.4)
$legacy = array('apple', 'banana');         // old long syntax — still valid, rarely used now

echo $fruits[0];   // apple
echo $fruits[2];   // cherry

$fruits[] = 'date'; // append — key becomes 3 automatically
// $fruits is now ['apple','banana','cherry','date']
```

Keys are assigned automatically as the **largest integer key seen so far, plus one**:

```php
<?php
$a = [5 => 'x'];
$a[] = 'y';          // key is 6, not 1
var_dump(array_keys($a));
// Output: array(2) { [0]=> int(5) [1]=> int(6) }
```

### Associative arrays (string keys)

An **associative array** uses string keys — this is your hash map / dictionary.

```php
<?php
$user = [
    'name'  => 'Ada',
    'email' => 'ada@example.com',
    'age'   => 36,
];

echo $user['name'];        // Ada
$user['role'] = 'admin';   // add a new pair
```

### Multidimensional arrays (arrays of arrays)

A **multidimensional array** is just an array whose values are themselves arrays. This models tables, trees, JSON, etc.

```php
<?php
$users = [
    ['id' => 1, 'name' => 'Ada',   'roles' => ['admin', 'editor']],
    ['id' => 2, 'name' => 'Linus', 'roles' => ['editor']],
];

echo $users[0]['name'];        // Ada
echo $users[0]['roles'][1];    // editor
$users[1]['roles'][] = 'viewer'; // nest deeper / append into a sub-array
```

### Key normalization — the rule that trips everyone up

PHP coerces keys before storing them. Memorize these:

- Strings containing a **canonical integer** become integers: `"8"` → `8`, but `"08"`, `"8.0"`, `" 8"` stay strings.
- **Floats** are truncated to int: `8.7` → `8`.
- **Booleans** become ints: `true` → `1`, `false` → `0`.
- **`null`** becomes the empty string `""`.
- These coercions can cause silent **key collisions** (last write wins).

```php
<?php
$a = [];
$a["1"]  = 'string one';
$a[1]    = 'int one';      // SAME key as "1" → overwrites
$a[(int) 1.9] = 'float';   // we truncate ON PURPOSE → key 1 → overwrites again
$a[true] = 'bool';         // true → 1 → overwrites again
$a[null] = 'null key';     // key is ""
var_dump($a);
/* Output:
array(2) {
  [1]=> string(4) "bool"
  [""]=> string(8) "null key"
}
*/
```

> **PHP 8.1+ deprecation — float keys:** Writing a *raw float* as a key, e.g. `$a[1.9] = 'x'`, still truncates to `1`, but since **PHP 8.1** it also emits
> `Deprecated: Implicit conversion from float 1.9 to int loses precision`.
> That's why the example above casts with `(int)` to make the truncation explicit. Don't rely on implicit float→int key conversion; cast deliberately.

> **PHP 8.0+ note:** A *numeric string float* like `"1.5"` is kept as a **string** key (it is NOT a canonical integer string), whereas the float `1.5` as a key is truncated to `1` (with the deprecation above). The two are not interchangeable — be deliberate.

---

## 2. Adding and removing elements

```php
<?php
$list = ['a', 'b', 'c'];

// Add
$list[] = 'd';                  // append at end (re-uses largest int + 1)
$list[10] = 'jump';             // explicit key, leaves a "gap" in indices
array_push($list, 'e', 'f');    // append one or more (function form)
array_unshift($list, 'start');  // prepend (re-indexes numeric keys!)

// Remove
unset($list[1]);                // remove a specific key — does NOT re-index
$last  = array_pop($list);      // remove + return last element
$first = array_shift($list);    // remove + return first element (re-indexes)
```

> **Gotcha:** `unset()` leaves a hole. After `unset($x[1])` the keys might be `0, 2, 3`. To get a clean `0,1,2…` list back, call `array_values($x)`.

```php
<?php
$x = ['a', 'b', 'c'];
unset($x[1]);
var_dump($x);                   // [0=>'a', 2=>'c']  (gap at 1)
$x = array_values($x);
var_dump($x);                   // [0=>'a', 1=>'c']  (re-indexed)
```

---

## 3. Accessing & checking safely

There are four checks people confuse constantly. Knowing the difference is a **classic interview question**.

| Function | Question it answers | Watch out for |
|---|---|---|
| `isset($a['k'])` | Does key `k` exist **and** is its value not `null`? | Returns `false` if value is `null` |
| `array_key_exists('k', $a)` | Does key `k` exist (even if value is `null`)? | Slightly slower; the correct "does the key exist" test |
| `in_array($v, $a)` | Is **value** `$v` present? | Loose `==` by default — pass `true` for strict |
| `array_search($v, $a)` | Returns the **key** of value `$v` (or `false`) | Loose by default; use `===` on the result |

```php
<?php
$config = ['timeout' => null, 'retries' => 3];

isset($config['timeout']);                // false  ← value is null!
array_key_exists('timeout', $config);     // true   ← key really exists

in_array('3', $config);                   // true (loose: '3' == 3)
in_array('3', $config, true);             // false (strict: '3' !== 3)

$key = array_search(3, $config);          // 'retries'
$key = array_search(99, $config);         // false
// Always compare with === because a valid key could be 0:
if (array_search('x', ['x']) !== false) { /* found at key 0 */ }
```

Two safer modern accessors:

```php
<?php
$user = ['name' => 'Ada'];

// Null coalescing — return a default if the key is missing or null:
$role = $user['role'] ?? 'guest';         // 'guest', no warning

// Null coalescing assignment — set only if not already set:
$user['role'] ??= 'guest';                // adds 'role' => 'guest'
```

> **PHP 8 change:** Accessing an undefined array key (`$a['missing']`) now raises an `E_WARNING` (it was a less-severe `E_NOTICE` before 8.0). Use `??` to avoid noise.

---

## 4. Counting

```php
<?php
$a = ['x', 'y', 'z'];
count($a);                       // 3
sizeof($a);                      // 3 — alias, prefer count()

// Recursive count of a nested structure:
$nested = [1, [2, 3], [4, [5, 6]]];
count($nested);                  // 3 (top level only)
count($nested, COUNT_RECURSIVE); // 9 — counts the sub-arrays themselves PLUS their elements
// Breakdown: 3 top-level + 2 (in [2,3]) + 2 (the 4 and the inner array in [4,[5,6]]) + 2 (in [5,6]) = 9

// Frequency table — count occurrences of each value:
$votes = ['yes', 'no', 'yes', 'yes', 'no'];
var_dump(array_count_values($votes));
// ['yes' => 3, 'no' => 2]

empty($a);                       // false (has elements)
$a === [];                       // true only if exactly an empty array
```

---

## 5. Iterating

```php
<?php
$user = ['name' => 'Ada', 'age' => 36];

// foreach with key + value — the idiomatic loop:
foreach ($user as $key => $value) {
    echo "$key: $value\n";
}
// Output:
// name: Ada
// age: 36

// Values only:
foreach (['a', 'b'] as $letter) { /* ... */ }

// Modify in place with a reference (note the &):
$nums = [1, 2, 3];
foreach ($nums as &$n) { $n *= 10; }
unset($n);                       // ALWAYS unset the reference after the loop
var_dump($nums);                 // [10, 20, 30]
```

> **Gotcha (the most common PHP array bug ever):** after `foreach ($x as &$ref)`, `$ref` still points at the *last element*. A later `foreach ($x as $ref)` will clobber that last element on its second-to-last iteration. **Always `unset($ref)` immediately after a by-reference foreach.**

```php
<?php
// Nested destructuring directly in foreach (PHP 7.1+):
$points = [[1, 2], [3, 4]];
foreach ($points as [$x, $y]) {
    echo "($x, $y)\n";           // (1, 2) then (3, 4)
}

// Destructure associative rows by key:
$rows = [['id' => 1, 'name' => 'Ada'], ['id' => 2, 'name' => 'Linus']];
foreach ($rows as ['id' => $id, 'name' => $name]) {
    echo "$id => $name\n";
}
```

---

## 6. Key / value functions

```php
<?php
$prices = ['apple' => 100, 'banana' => 50, 'cherry' => 100];

array_keys($prices);             // ['apple','banana','cherry']
array_keys($prices, 100);        // ['apple','cherry'] — keys whose value is 100
array_values($prices);           // [100, 50, 100]

// array_flip swaps keys and values (values must be int|string and unique-ish):
array_flip(['a' => 1, 'b' => 2]);     // [1 => 'a', 2 => 'b']
array_flip(['a', 'b', 'c']);          // ['a'=>0, 'b'=>1, 'c'=>2]
// Collisions: later duplicate values overwrite earlier keys.

// Fast O(1) membership set: flip values into keys, then isset:
$allowed = array_flip(['read', 'write', 'delete']);
isset($allowed['write']);        // true — faster than in_array on big arrays
```

---

## 7. Transforming: map / filter / reduce / walk

This quartet is the heart of functional-style PHP and a guaranteed interview topic.

### `array_map` — transform every value

```php
<?php
$nums = [1, 2, 3, 4];
$squares = array_map(fn($n) => $n * $n, $nums);
// [1, 4, 9, 16]

// Multiple arrays — callback gets one element from each, in parallel:
$a = [1, 2, 3];
$b = [10, 20, 30];
$sums = array_map(fn($x, $y) => $x + $y, $a, $b);  // [11, 22, 33]
```

> **Two subtle `array_map` rules:**
> 1. With a **single** array, keys are **preserved**. With **multiple** arrays, keys are **discarded** (result is re-indexed `0,1,2…`).
> 2. Passing `null` as the callback **zips** arrays together: `array_map(null, $a, $b)` → `[[1,10],[2,20],[3,30]]`.

### `array_filter` — keep elements that pass a test

```php
<?php
$nums = [1, 2, 3, 4, 5, 6];
$even = array_filter($nums, fn($n) => $n % 2 === 0);
// [1 => 2, 3 => 4, 5 => 6]   ← KEYS ARE PRESERVED (gaps!)
$even = array_values($even);   // re-index → [2, 4, 6]

// No callback → removes "falsy" values (0, '', '0', null, false, []):
array_filter([0, 1, '', 'a', null, '0', false]);   // [1 => 1, 3 => 'a']

// ARRAY_FILTER_USE_KEY / ARRAY_FILTER_USE_BOTH filter on the key too:
$data = ['a1' => 1, 'b2' => 2, 'a3' => 3];
array_filter($data, fn($k) => str_starts_with($k, 'a'), ARRAY_FILTER_USE_KEY);
// ['a1' => 1, 'a3' => 3]
```

> **Gotcha:** `array_filter` preserves keys, so the result of filtering an indexed array is often *not* a clean list. If you then `json_encode` it, PHP emits a JSON **object** `{"1":2,"3":4}` instead of an array `[2,4]`. Wrap with `array_values()` when you need a JSON array.

### `array_reduce` — fold an array down to one value

```php
<?php
$nums = [1, 2, 3, 4];

// Sum:
$total = array_reduce($nums, fn($carry, $n) => $carry + $n, 0);   // 10

// Build a string:
$csv = array_reduce(['a', 'b', 'c'],
    fn($carry, $x) => $carry === '' ? $x : "$carry,$x", '');       // "a,b,c"

// Group into buckets (carry is an array):
$people = [['team' => 'A', 'name' => 'Ada'], ['team' => 'B', 'name' => 'Bob'], ['team' => 'A', 'name' => 'Al']];
$byTeam = array_reduce($people, function ($acc, $p) {
    $acc[$p['team']][] = $p['name'];
    return $acc;
}, []);
// ['A' => ['Ada','Al'], 'B' => ['Bob']]
```

### `array_walk` — apply a callback for side effects (in place)

`array_map` returns a new array; `array_walk` mutates the original (when you take the value by reference) and returns `true`/`false`.

```php
<?php
$prices = [10, 20, 30];
array_walk($prices, function (&$price, $key) {
    $price = $price * 1.1;       // add 10% tax, IN PLACE
});
var_dump($prices);               // [11.0, 22.0, 33.0]
```

> **map vs walk:** use `array_map` when you want a *new* transformed array; use `array_walk` when you want to *mutate in place* or need the key inside the callback while keeping the original keys.

### `array_find` / `array_find_key` / `array_any` / `array_all` — NEW in PHP 8.4

PHP 8.4 finally added first-class predicate search/quantifier functions, so you no longer need a `foreach` (or a `array_filter(...)[0]` hack) to answer "find the first match / does any / do all".

```php
<?php
// PHP 8.4+ ONLY
$users = [
    ['name' => 'Ada',   'active' => true],
    ['name' => 'Bob',   'active' => false],
    ['name' => 'Linus', 'active' => true],
];

// array_find — first VALUE matching the predicate, or null if none:
array_find($users, fn($u) => !$u['active']);          // ['name'=>'Bob','active'=>false]

// array_find_key — KEY of the first match, or null:
array_find_key($users, fn($u) => !$u['active']);      // 1

// array_any — true if AT LEAST ONE element matches (short-circuits):
array_any($users, fn($u) => $u['active']);            // true

// array_all — true only if EVERY element matches (short-circuits):
array_all($users, fn($u) => $u['active']);            // false
```

> **Versioning:** all four are **PHP 8.4+**. On 8.3 and earlier they don't exist (`Call to undefined function array_find()`). The pre-8.4 equivalents: `array_find` ≈ `array_filter(...)` then `reset()`/`array_key_first()`; `array_any`/`array_all` ≈ a `foreach` with early `return`. Laravel's `Collection` has had `first(fn)`, `contains(fn)`, and `every(fn)` for years if you're on an older runtime.

---

## 8. Combining arrays

### `array_merge` vs the `+` operator — the #1 array trap

Both join two arrays, but they resolve conflicts **oppositely** and treat numeric keys differently.

```php
<?php
$a = ['x' => 1, 'y' => 2];
$b = ['y' => 99, 'z' => 3];

array_merge($a, $b);
// ['x' => 1, 'y' => 99, 'z' => 3]   ← RIGHT side wins on string-key conflicts

$a + $b;
// ['x' => 1, 'y' => 2,  'z' => 3]   ← LEFT side wins (keeps existing keys)
```

The bigger difference is with **numeric keys**:

```php
<?php
$a = ['a', 'b'];          // keys 0,1
$b = ['c', 'd'];          // keys 0,1

array_merge($a, $b);      // ['a','b','c','d'] — numeric keys RENUMBERED, appended
$a + $b;                  // ['a','b']         — keys 0,1 already exist, $b dropped!
```

**Rule of thumb:**
- Use `array_merge` to **concatenate lists** (it renumbers integer keys).
- Use `+` (the *union* operator) to **apply defaults** to an associative array (left side wins, keys preserved):

```php
<?php
$options  = ['timeout' => 30];
$defaults = ['timeout' => 10, 'retries' => 3];
$final = $options + $defaults;   // ['timeout' => 30, 'retries' => 3]
```

### `array_combine` — keys from one array, values from another

```php
<?php
$keys   = ['id', 'name', 'role'];
$values = [1, 'Ada', 'admin'];
array_combine($keys, $values);
// ['id' => 1, 'name' => 'Ada', 'role' => 'admin']
// Both arrays MUST have the same count, or it throws ValueError (PHP 8+).
```

### `array_replace` — like merge but keeps keys for numeric arrays too

```php
<?php
$base     = ['a', 'b', 'c'];
$override = [1 => 'B'];
array_replace($base, $override);   // ['a', 'B', 'c']  ← replaces by key, no renumbering
array_merge($base, $override);     // ['a', 'b', 'c', 'B']  ← would append instead
```

> **`array_merge` vs `array_replace`:** merge *appends* numeric keys; replace *overwrites by key*. For string keys they behave the same. Use `array_replace_recursive` / `array_merge_recursive` for deep structures (they have their own quirks — `array_merge_recursive` turns colliding scalars into arrays, which is rarely what you want).

---

## 9. Slicing, splicing, chunking

```php
<?php
$a = ['a', 'b', 'c', 'd', 'e'];

// array_slice(array, offset, length, preserve_keys) — does NOT modify original
array_slice($a, 1, 2);            // ['b', 'c']        (re-indexed)
array_slice($a, -2);              // ['d', 'e']        (negative offset from end)
array_slice($a, 1, 2, true);     // [1 => 'b', 2 => 'c']  (preserve keys)

// array_splice — DOES modify the original; removes and optionally inserts
$b = ['a', 'b', 'c', 'd'];
$removed = array_splice($b, 1, 2, ['X', 'Y', 'Z']);
// $b       = ['a', 'X', 'Y', 'Z', 'd']
// $removed = ['b', 'c']

// array_chunk — split into fixed-size groups (great for pagination/batching)
array_chunk([1,2,3,4,5], 2);          // [[1,2],[3,4],[5]]
array_chunk(['a'=>1,'b'=>2,'c'=>3], 2, true);  // [['a'=>1,'b'=>2],['c'=>3]] (keep keys)
```

> **slice vs splice memory hook:** *sl**i**ce* returns a copy and leaves the source intact; *spl**i**ce* surgically removes from the original (think "splice a film reel").

---

## 10. The full sorting family + stability

PHP has a sort function for every combination of (sort by value vs key) × (ascending vs descending) × (keep keys vs reindex) × (custom comparator). **All sort functions mutate the array in place and return `bool`, not the sorted array.**

| Function | Sorts by | Order | Keeps key→value association? |
|---|---|---|---|
| `sort` | value | asc | No (reindexes 0,1,2…) |
| `rsort` | value | desc | No (reindexes) |
| `asort` | value | asc | **Yes** |
| `arsort` | value | desc | **Yes** |
| `ksort` | key | asc | Yes |
| `krsort` | key | desc | Yes |
| `usort` | value (custom) | custom | No (reindexes) |
| `uasort` | value (custom) | custom | Yes |
| `uksort` | key (custom) | custom | Yes |
| `natsort` | value (natural) | asc | Yes |
| `natcasesort` | value (natural, case-insensitive) | asc | Yes |

```php
<?php
$n = [3, 1, 2];
sort($n);                 // $n is now [1, 2, 3]; returns true (NOT the array!)

$ages = ['Ada' => 36, 'Bob' => 28, 'Cy' => 41];
asort($ages);             // ['Bob'=>28, 'Ada'=>36, 'Cy'=>41]  (by value, keys kept)
arsort($ages);            // by value desc, keys kept
ksort($ages);             // ['Ada'=>36, 'Bob'=>28, 'Cy'=>41] (by key asc)

// usort with a custom comparator: return <0, 0, or >0
$people = [['name'=>'Ada','age'=>36], ['name'=>'Bob','age'=>28]];
usort($people, fn($a, $b) => $a['age'] <=> $b['age']);  // by age asc
// The <=> "spaceship" operator returns -1/0/1 — perfect for comparators.

// Multi-key sort: age asc, then name asc:
usort($people, fn($a, $b) => [$a['age'], $a['name']] <=> [$b['age'], $b['name']]);

// natsort: "img10" comes AFTER "img2" (humans expect this; plain sort doesn't):
$files = ['img12.png', 'img10.png', 'img2.png', 'img1.png'];
sort($files);             // ['img1.png','img10.png','img12.png','img2.png']  ← wrong order
natsort($files);          // ['img1.png','img2.png','img10.png','img12.png'] (keys kept)
```

### Stability — important since PHP 8.0

A sort is **stable** if elements that compare *equal* keep their original relative order. **Before PHP 8.0, all PHP sorts were unstable.** Since PHP 8.0, **all sort functions are guaranteed stable.** This matters when you sort by one field and expect ties to preserve a prior order.

```php
<?php
// PHP 8.0+: equal "age" entries keep insertion order — predictable.
$rows = [['age'=>30,'id'=>1],['age'=>20,'id'=>2],['age'=>30,'id'=>3]];
usort($rows, fn($a,$b) => $a['age'] <=> $b['age']);
// id order among the two age=30 rows is 1 then 3 (stable). Pre-8.0 this was undefined.
```

> **Spaceship + arrays:** `<=>` compares arrays element-by-element, which is why `[$a->x, $a->y] <=> [$b->x, $b->y]` gives you clean multi-key sorting without nested `if`s.

---

## 11. The spread operator in arrays

The spread (`...`) operator unpacks one array into a literal.

```php
<?php
$a = [1, 2, 3];
$b = [0, ...$a, 4];          // [0, 1, 2, 3, 4]

// Spread into a function call (argument unpacking):
function add($x, $y, $z) { return $x + $y + $z; }
add(...[1, 2, 3]);           // 6
```

### String-key spread — PHP 8.1+

Before PHP 8.1, spreading arrays with **string keys** was a fatal error. Since 8.1 it works, and later keys win (like `array_merge`):

```php
<?php
// PHP 8.1+ only:
$defaults = ['color' => 'red', 'size' => 'M'];
$custom   = ['size' => 'L'];
$merged   = [...$defaults, ...$custom];   // ['color' => 'red', 'size' => 'L']

// Numeric keys are still renumbered when spread:
[...['a' => 1], ...['b' => 2]];           // ['a' => 1, 'b' => 2]
[...[5 => 'x'], ...[5 => 'y']];           // [0 => 'x', 1 => 'y']  (numeric → renumbered)
```

> **8.0 vs 8.1:** if you `[...$assoc]` on PHP 8.0 or earlier with string keys you get `Fatal error: Cannot unpack array with string keys`. On 8.1+ it just works. This is a common "which version are you on?" gotcha.

---

## 12. Destructuring: `list()` and `[]` with keys

**Destructuring** assigns array elements to multiple variables at once.

```php
<?php
// Positional (long form list() and short form [] are equivalent):
[$a, $b, $c] = [1, 2, 3];        // $a=1, $b=2, $c=3
list($a, $b) = [10, 20];         // same thing, older syntax

// Skip elements with empty slots:
[, , $third] = ['x', 'y', 'z'];  // $third = 'z'

// Key destructuring (PHP 7.1+) — pull specific keys, any order:
['name' => $name, 'age' => $age] = ['age' => 36, 'name' => 'Ada'];
// $name='Ada', $age=36

// Nested:
[[$a, $b], [$c, $d]] = [[1, 2], [3, 4]];   // 1,2,3,4

// Swap two variables with no temp:
[$x, $y] = [$y, $x];

// Default-ish: there are NO inline defaults in list()/[]. Destructuring a key that is
// MISSING assigns null AND raises an E_WARNING (since PHP 8.0):
$user = ['name' => 'Ada'];
['role' => $role] = $user;   // ⚠️ Warning: Undefined array key "role"; $role is null
$role ??= 'guest';           // apply the default afterward

// To avoid the warning entirely, don't destructure an optional key — read it with ??:
$role = $user['role'] ?? 'guest';   // clean, no warning
```

> **Gotcha:** you **cannot mix** positional and keyed destructuring in the same `[]` (it's a fatal `Cannot mix keyed and unkeyed array entries` error), and there are no inline defaults inside `list()`. Destructuring a key that doesn't exist emits an `E_WARNING` (PHP 8.0+) and yields `null` — so for genuinely optional keys, prefer `$x = $a['k'] ?? default` over destructure-then-`??`.

---

## 13. Set-like and generator helpers

```php
<?php
// array_column — pluck one column out of a list of rows (the SQL-result workhorse):
$users = [
    ['id' => 1, 'name' => 'Ada'],
    ['id' => 2, 'name' => 'Bob'],
];
array_column($users, 'name');          // ['Ada', 'Bob']
array_column($users, 'name', 'id');    // [1 => 'Ada', 2 => 'Bob']  (id as key)
array_column($users, null, 'id');      // re-key whole rows by id

// array_unique — remove duplicate VALUES (keeps first key, preserves keys):
array_unique([1, 2, 2, 3, 3, 3]);      // [0=>1, 1=>2, 3=>3]
// Uses loose string comparison by default; pass SORT_REGULAR for value-type compare.

// array_fill / array_fill_keys — pre-populate:
array_fill(0, 3, 'x');                 // ['x', 'x', 'x']
array_fill_keys(['a', 'b'], 0);        // ['a' => 0, 'b' => 0]

// range — generate a sequence:
range(1, 5);                           // [1, 2, 3, 4, 5]
range('a', 'e');                       // ['a','b','c','d','e']
range(0, 10, 2);                       // [0, 2, 4, 6, 8, 10]  (step)

// Set operations (compare by VALUE):
array_diff([1, 2, 3, 4], [2, 4]);      // [0=>1, 2=>3]  (in first, not in others)
array_intersect([1, 2, 3], [2, 3, 4]); // [1=>2, 2=>3]  (in both)
// _key variants compare by KEY: array_diff_key, array_intersect_key
array_intersect_key(['a'=>1,'b'=>2], ['a'=>9]);  // ['a' => 1]
```

> **PHP 8.3 note:** `range()` got stricter/more consistent — e.g. it now throws on invalid step values and handles float/edge cases more predictably than in 8.2 and earlier. Behavior for the common integer/char cases above is unchanged.

---

## 14. Arrays as stacks and queues

PHP gives you all four mutators, so an array *is* a stack and a queue.

```php
<?php
// STACK (LIFO — last in, first out): push/pop at the END
$stack = [];
array_push($stack, 'a');     // ['a']
array_push($stack, 'b');     // ['a', 'b']
array_pop($stack);           // returns 'b'; $stack = ['a']

// QUEUE (FIFO — first in, first out): push at END, shift from FRONT
$queue = [];
array_push($queue, 'job1');  // ['job1']
array_push($queue, 'job2');  // ['job1', 'job2']
array_shift($queue);         // returns 'job1'; $queue = ['job2']
```

| Operation | End of array | Front of array |
|---|---|---|
| Add | `array_push` / `$a[] =` | `array_unshift` |
| Remove | `array_pop` | `array_shift` |

> **Performance note:** `array_push`/`array_pop` are O(1). `array_shift`/`array_unshift` are O(n) because every remaining element must be re-indexed. For a hot, large FIFO queue, prefer `SplQueue` (from the SPL — Standard PHP Library) or `SplDoublyLinkedList` to avoid the re-indexing cost.

---

## 15. References in arrays & copy-on-write

### Arrays are value types — assigned by copy

```php
<?php
$a = [1, 2, 3];
$b = $a;          // $b is a COPY (logically)
$b[] = 4;
var_dump($a);     // [1, 2, 3] — unchanged
var_dump($b);     // [1, 2, 3, 4]
```

This is the opposite of objects, which assign by handle (reference-like). Arrays copy.

### Copy-on-write (COW) — the optimization that makes copies cheap

If arrays truly copied on every assignment, PHP would be slow. Instead, the Zend engine uses **copy-on-write**: `$b = $a` does **not** duplicate the data immediately. Both names point to the same internal array with a reference-counter. PHP only performs the real (deep) copy at the moment one of them is **written to**. So reading is free; the copy cost is paid lazily, only when you mutate.

```php
<?php
$a = range(1, 1_000_000);   // big array
$b = $a;                    // O(1): no copy yet, refcount goes to 2
// ... reading $b is free ...
$b[0] = 99;                 // NOW PHP duplicates the array (the "write" triggers COW)
```

> This is why passing a large array to a function by value is **not** automatically expensive — as long as the function only *reads* it, no copy happens. The copy is triggered only on mutation. You can pass `array &$x` by reference to also avoid the eventual write-copy, but only do so when you genuinely intend to mutate the caller's array.

### Explicit references inside arrays — handle with care

```php
<?php
$x = 10;
$arr = ['ref' => &$x];   // store a reference
$arr['ref'] = 20;
echo $x;                 // 20 — the original variable changed!

// References survive copies in surprising ways and break COW.
// Avoid binding references into arrays unless you have a specific reason.
```

> **Gotcha:** an element that is a **reference** persists through `$b = $a` copies — it is *not* broken by the copy. Combined with the dangling `foreach (... as &$v)` reference, this is a frequent source of "why did my array mutate?" bugs. Rule: `unset()` references when done, and avoid storing `&` in arrays.

---

## 16. How this maps to Laravel 12

Laravel wraps raw arrays in two ergonomic layers. Knowing the raw functions makes the Laravel ones obvious.

```php
<?php
use Illuminate\Support\Arr;

// Arr:: static helpers operate on plain arrays with "dot" notation:
$data = ['user' => ['name' => 'Ada', 'roles' => ['admin']]];
Arr::get($data, 'user.name');                 // 'Ada'   (safe deep access)
Arr::get($data, 'user.phone', 'n/a');         // 'n/a'   (default)
Arr::has($data, 'user.roles');                // true
Arr::set($data, 'user.age', 36);              // deep set
data_get($data, 'user.roles.0');              // 'admin' (global helper)
Arr::pluck($users ?? [], 'name', 'id');       // like array_column
Arr::flatten([[1, 2], [3, [4]]]);             // [1, 2, 3, 4]
Arr::only($data['user'], ['name']);           // ['name' => 'Ada']
Arr::except($data['user'], ['roles']);        // ['name' => 'Ada']
```

```php
<?php
// Collections — a fluent, chainable OO wrapper. Same operations, method chaining:
$result = collect([1, 2, 3, 4, 5, 6])
    ->filter(fn ($n) => $n % 2 === 0)   // [2,4,6]  (filter, like array_filter)
    ->map(fn ($n) => $n * 10)           // [20,40,60]
    ->values()                          // re-index (like array_values)
    ->all();                            // back to a plain array → [20, 40, 60]

// Eloquent query results are Collections, so map/filter/reduce/groupBy/sortBy
// all "just work" on DB rows. groupBy/keyBy mirror array_reduce grouping +
// array_column re-keying.
```

> **Laravel 10/11 vs 12 note:** the `Arr` and `Collection` APIs covered here are stable across Laravel 10, 11, and 12 — these are core helpers that haven't changed in those versions. Newer Collection methods get *added* over time (e.g. `Collection::after`/`before`), but the fundamentals above behave identically. Always check the version-specific docs for the newest methods.

---

## ⚠️ Common Mistakes & Gotchas

1. **Expecting sort functions to return the sorted array.**
   `$sorted = sort($a);` sets `$sorted` to `true`, not the array! Sorts mutate in place and return a bool.
   **Fix:** `sort($a); /* use $a */` — or use `collect($a)->sort()->values()->all()` if you want a return value.

2. **`array_filter` / `unset` leaving holes, then breaking `json_encode`.**
   Filtering an indexed array preserves keys, so you get `[0=>'a', 2=>'c']`, which `json_encode`s to an *object* `{"0":"a","2":"c"}`.
   **Fix:** wrap with `array_values()` before encoding when you need a JSON array.

3. **Dangling `foreach` reference clobbering the last element.**
   ```php
   foreach ($a as &$v) { /*...*/ }    // forgot to unset
   foreach ($a as $v) { /*...*/ }     // SILENTLY overwrites $a's last element
   ```
   **Fix:** `unset($v);` immediately after any by-reference foreach.

4. **Using `isset` to check for a key whose value can be `null`.**
   `isset($a['k'])` returns `false` when `$a['k']` is `null`, even though the key exists.
   **Fix:** use `array_key_exists('k', $a)` when `null` is a meaningful value.

5. **Confusing `array_merge` and `+` (and getting silently wrong data).**
   `+` keeps left-side keys and silently drops right-side numeric-key elements; `array_merge` renumbers numeric keys and lets the right side win on string keys.
   **Fix:** lists → `array_merge`; defaults on associative arrays → `$opts + $defaults`.

6. **Forgetting `array_search`/`in_array` are loose by default.**
   Before PHP 8.0, `in_array(0, ['a', 'b'])` returned `true` because `0 == 'a'` was `true` (string→0). PHP 8.0 changed number↔non-numeric-string comparison, so it now returns `false` — but subtle bugs remain with other mixed types (`'1e3' == '1000'`, `'0' == false`, leading/trailing whitespace, etc.).
   **Fix:** pass the strict flag: `in_array($needle, $a, true)` and `array_search($needle, $a, true)`.

7. **Comparing `array_search` result with `==` instead of `===`.**
   A real match at key `0` is falsy.
   **Fix:** `if (($k = array_search($v, $a, true)) !== false)`.

---

## ✅ Best Practices

- **Prefer the short syntax** `[]` over `array()`.
- **Reach for `array_map`/`filter`/`reduce`** over manual loops when transforming data — it states intent and avoids index bugs. Use a plain `foreach` when you need early exit, side effects, or readability with complex bodies.
- **Use `??` and `??=`** instead of `isset()`/ternary towers for defaults.
- **Build a lookup set** with `array_flip(...)` + `isset()` for repeated membership tests on large arrays (O(1) vs `in_array`'s O(n)).
- **Use the spaceship operator `<=>`** (and array comparison for multi-key) in `usort` comparators.
- **Call `array_values()`** after `array_filter`/`unset`/`array_unique` whenever downstream code or JSON expects a clean list.
- **Use `array_column`/`Arr::pluck`/`keyBy`** to reshape SQL/JSON rows instead of hand-rolling loops.
- **Don't store references in arrays**, and always `unset()` a by-reference `foreach` variable.
- **Pass big arrays by value freely** for read-only use — COW makes it cheap; only use `&` when you intend to mutate the caller's array.
- **In Laravel**, prefer `data_get`/`Arr::get` for deep optional access instead of chained `??` on nested arrays.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `isset()` and `array_key_exists()`?**
A. `isset()` returns `false` if the key is missing **or** the value is `null`. `array_key_exists()` returns `true` as long as the key exists, even when its value is `null`. Use `array_key_exists` when `null` is a valid stored value; otherwise `isset` is faster and usually fine.

**Q2. Explain `array_merge` vs the `+` operator.**
A. `array_merge` renumbers integer keys (appends lists) and, on string-key collisions, the **later** array wins. The `+` (union) operator keeps the **left** array's keys — for any key already present on the left, the right side is ignored, including numeric keys (so `['a','b'] + ['c','d']` is just `['a','b']`). Use `array_merge` to concatenate lists, `+` to apply defaults.

**Q3. Which sort functions preserve key→value association, and are PHP sorts stable?**
A. The `a*` (asort/arsort) and `k*` (ksort/krsort) families, plus `uasort`/`uksort` and `natsort`, preserve keys; `sort`/`rsort`/`usort` reindex. **Since PHP 8.0 all sorts are guaranteed stable** (equal elements keep insertion order); before 8.0 they were unstable.

**Q4. (Under the hood) How are PHP arrays implemented, and what is copy-on-write?**
A. A PHP array is an **ordered hash table** (`HashTable` in the Zend engine) that stores both an integer-indexed bucket layout and string-key hashing, while preserving insertion order via a doubly-ordered structure. Assignment (`$b = $a`) uses **copy-on-write**: PHP shares one internal `zval`/`HashTable` and bumps a refcount instead of copying. The actual deep copy ("separation") is deferred until one alias is **written to**. This makes reads and pass-by-value cheap; mutation is what triggers the copy. Storing a `&` reference inside an array disables COW for that element.

**Q5. Why does `array_filter` sometimes produce a JSON object instead of an array?**
A. `array_filter` preserves the original keys, so filtering `[0,1,2,3]` might yield `[1=>1, 3=>3]`. `json_encode` only outputs a JSON array when keys are a continuous `0..n` sequence; with gaps it outputs an object. Fix with `array_values()`.

**Q6. What's the difference between `array_map` and `array_walk`?**
A. `array_map` returns a **new** array and (with one input) preserves keys but doesn't pass the key to the callback (by default). `array_walk` mutates the array **in place** (when the value is taken by reference), passes both value and key to the callback, and returns a bool. Use map for transformation pipelines, walk for in-place mutation or when you need the key.

**Q7. When would you use `SplQueue`/`SplStack` over a plain array?**
A. `array_shift`/`array_unshift` are O(n) because they re-index. For a large, high-throughput FIFO queue, `SplQueue` (or `SplDoublyLinkedList`) gives O(1) enqueue/dequeue and clearer intent. For a stack, plain arrays (`push`/`pop`) are already O(1), so the win is mostly semantic.

**Q8. How does the spread operator handle string keys, and since when?**
A. Since **PHP 8.1**, `[...$assoc]` works with string keys (later keys win, like `array_merge`); before 8.1 it was a fatal error. Numeric keys are always renumbered when spread.

**Q9. How do you sort an array of records by two fields?**
A. Use `usort` with the spaceship operator on tuples: `usort($rows, fn($a,$b) => [$a['age'],$a['name']] <=> [$b['age'],$b['name']]);`. `<=>` compares arrays element-by-element, giving lexicographic multi-key ordering, and stability (8.0+) handles remaining ties.

**Q10. What does `array_column` do and where is it used?**
A. It extracts a single column from a list of rows (arrays or objects), optionally re-keying by another column: `array_column($users, 'name', 'id')`. It's the idiomatic way to reshape DB/JSON result sets; Laravel's `Collection::pluck` / `Arr::pluck` are the equivalents.

**Q11. What did PHP 8.4 add for searching arrays with a predicate?**
A. Four functions: `array_find` (first matching *value* or `null`), `array_find_key` (first matching *key* or `null`), `array_any` (true if at least one element matches — short-circuits), and `array_all` (true only if every element matches — short-circuits). They replace common `foreach`/`array_filter(...)` idioms. They are 8.4+ only; on older runtimes use a `foreach` with early `return`, or Laravel's `Collection::first(fn)`/`contains(fn)`/`every(fn)`.

---

## 📋 Quick Reference / Cheat Sheet

```php
// CREATE
$a = ['x', 'y'];                  $m = ['k' => 'v'];

// ADD / REMOVE
$a[] = 'z';       array_push($a, 'z');     array_unshift($a, 'first');
unset($a[1]);     array_pop($a);           array_shift($a);

// CHECK
isset($a['k']);            array_key_exists('k', $a);
in_array($v, $a, true);    array_search($v, $a, true);   // strict!
$v = $a['k'] ?? 'default'; $a['k'] ??= 'default';

// COUNT
count($a);   count($a, COUNT_RECURSIVE);   array_count_values($a);

// KEYS / VALUES
array_keys($a);   array_values($a);   array_flip($a);   array_column($rows,'c','id');

// TRANSFORM
array_map(fn($x)=>$x*2, $a);                 // new array
array_filter($a, fn($x)=>$x>0);              // preserves keys → array_values()!
array_reduce($a, fn($c,$x)=>$c+$x, 0);       // fold to one value
array_walk($a, fn(&$v,$k)=>$v++);            // mutate in place

// SEARCH / QUANTIFY (PHP 8.4+)
array_find($a, fn($x)=>$x>0);                // first matching VALUE or null
array_find_key($a, fn($x)=>$x>0);            // KEY of first match or null
array_any($a, fn($x)=>$x>0);                 // true if ≥1 matches
array_all($a, fn($x)=>$x>0);                 // true if all match

// COMBINE
array_merge($a, $b);     // lists; right wins on string keys; renumbers ints
$a + $b;                 // defaults; left wins; keeps keys
array_combine($keys, $vals);   array_replace($base, $over);

// SLICE / SPLICE / CHUNK
array_slice($a, 1, 2);        // copy
array_splice($a, 1, 2, [..]); // in place, returns removed
array_chunk($a, 3);

// SORT (all in place, return bool)
sort  rsort                    // value, reindex
asort arsort                   // value, keep keys
ksort krsort                   // key, keep keys
usort uasort uksort            // custom comparator (use <=>)
natsort                        // natural order
// PHP 8.0+: all stable.

// SPREAD / DESTRUCTURE
[$x, ...$rest] = ...;   $c = [...$a, ...$b];   // string-key spread: 8.1+
['name' => $n] = $row;  [, , $third] = $a;

// SET OPS
array_unique($a);  array_diff($a,$b);  array_intersect($a,$b);  range(1,10);

// LARAVEL
Arr::get($a,'x.y',$def);  data_get($a,'x.y');  Arr::pluck($rows,'n','id');
collect($a)->filter()->map()->values()->all();
```

---

## 🧪 Mini Exercises

1. **Group & count.** Given `$orders = [['user'=>'a','total'=>10],['user'=>'b','total'=>5],['user'=>'a','total'=>7]]`, use `array_reduce` to produce `['a'=>17,'b'=>5]` (sum of totals per user). Then re-key the result by user with the highest total first.

2. **Safe deep config.** Write a function `getConfig(array $cfg, string $dotPath, $default = null)` that resolves `'mail.smtp.host'` against a nested array without warnings (re-implement a tiny `data_get`). Verify it returns the default for missing paths and works when an intermediate value is literally `null`.

3. **Natural-order file sort + chunk.** Given `['file10.txt','file2.txt','file1.txt','file20.txt']`, sort them in human order and split into pages of 2 using `array_chunk`. Explain in a comment why plain `sort()` gives the wrong order.

4. **Merge vs plus.** Build two associative arrays that produce *different* results under `array_merge($a,$b)` vs `$a + $b`, and two indexed arrays that also differ. Print both results and annotate which rule (right-wins / left-wins / renumber) caused each.

5. **De-dup pipeline.** Given a list of user rows with possible duplicate emails, produce a clean re-indexed list with one row per email (keep the first occurrence), using `array_column`, `array_unique`, and `array_values`. Confirm the output `json_encode`s as a JSON array, not an object.
