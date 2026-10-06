# Laravel Collections — From Zero to Expert

Collections are one of the most-used and most-loved features in Laravel. Once you internalize them, you stop writing `for` loops and start *describing* data transformations as readable, chainable pipelines. This module takes you from "what is a Collection?" all the way to lazy collections, higher-order messages, and the Eloquent-specific extras that show up constantly in real applications and interviews.

---

## **What you'll learn**

- What the `Illuminate\Support\Collection` class is, why it exists, and how it differs from plain PHP arrays.
- How to create collections with `collect()` and how Eloquent's `get()` returns one automatically.
- The core toolkit: `map`, `filter`, `reject`, `reduce`, `pluck`, `groupBy`, `sortBy`, `unique`, `where`, `chunk`, and dozens more — with output for every non-trivial example.
- **Higher-order messages** — the elegant `$users->each->markAsActive()` shorthand.
- **Lazy collections** (`LazyCollection`) and Eloquent's `cursor()` for processing huge datasets without exhausting memory.
- The extra methods Eloquent Collections add on top: `find`, `modelKeys`, `load`, and keyed `contains`/`only`/`except`.
- The mental model of *immutability* and *fluent chaining* that makes collections click.

---

## 1. Why collections exist (the WHY before the HOW)

PHP already gives you arrays and a pile of `array_*` functions. So why does Laravel ship its own data structure?

Three reasons:

1. **Inconsistent argument order.** PHP's native functions are famously inconsistent: `array_map($callback, $array)` puts the callback first, but `array_filter($array, $callback)` puts the array first. `in_array($needle, $haystack)` vs `array_search($needle, $haystack)`... It's hard to memorize and easy to get wrong.

2. **No fluent chaining.** With raw arrays you end up nesting calls inside-out, which reads backwards:

   ```php
   // Read this from the inside out — awkward.
   $names = array_values(
       array_map(
           fn ($u) => strtoupper($u['name']),
           array_filter($users, fn ($u) => $u['active'])
       )
   );
   ```

3. **Discoverability and expressiveness.** A `Collection` wraps your array and exposes well over 100 well-named methods (180+ in Laravel 12) that each return a *new* collection, so you can chain them left-to-right in the order the data flows:

   ```php
   $names = collect($users)
       ->filter(fn ($u) => $u['active'])   // keep active users
       ->map(fn ($u) => strtoupper($u['name'])) // transform
       ->values();                          // re-index 0,1,2...
   ```

That second version reads like a sentence. That is the entire selling point: **collections turn data manipulation into a readable, chainable pipeline.**

### The key mental model: immutability + fluent chaining

Almost every Collection method **returns a brand-new Collection instead of mutating the original**. (A handful — `push`, `put`, `pop`, `shift`, `prepend`, `transform`, `forget` — mutate in place; we'll flag those explicitly.) This immutability is what makes long chains safe: each step hands a fresh collection to the next.

```php
$original = collect([1, 2, 3]);
$doubled  = $original->map(fn ($n) => $n * 2);

$original->all(); // [1, 2, 3]  — unchanged
$doubled->all();  // [2, 4, 6]
```

---

## 2. Creating collections

### `collect()` — the universal helper

The `collect()` global helper is the most common way to make one:

```php
use Illuminate\Support\Collection;

$c = collect([1, 2, 3]);
$c = collect(['name' => 'Ada', 'role' => 'admin']); // associative
$c = collect('single value'); // collect(['single value'])  -> wraps non-array
$c = collect();               // empty collection
$c = new Collection([1, 2, 3]); // equivalent, explicit class

$c instanceof Collection; // true
```

`collect(null)` gives you an empty collection — handy for guarding against null without `if` statements.

### `Collection::make()` and `Collection::times()`

```php
use Illuminate\Support\Collection;

Collection::make([1, 2, 3]);

// times(): build N items from a callback (1-indexed)
Collection::times(3, fn ($i) => $i * 10);
// Output: [10, 20, 30]
```

### `Collection::range()` and `wrap()`

```php
// range() is a STATIC method (Collection::range), not an instance method:
Collection::range(1, 5)->all();     // [1, 2, 3, 4, 5]

Collection::wrap('x')->all();       // ['x']
Collection::wrap(['x'])->all();     // ['x']  — already an array, not double-wrapped
Collection::wrap(collect([1]))->all(); // [1]  — already a collection, returned as-is
```

`wrap()` is the safe "give me a collection no matter what you passed me" helper.

### Eloquent `get()` returns a Collection automatically

This is the connection that makes collections matter day-to-day. Any Eloquent query that returns multiple rows hands you a collection (specifically `Illuminate\Database\Eloquent\Collection`, a subclass — more on that in section 9):

```php
use App\Models\User;

$users = User::where('active', true)->get();

$users instanceof \Illuminate\Database\Eloquent\Collection; // true
$users instanceof \Illuminate\Support\Collection;           // also true (subclass)

// So you can immediately chain collection methods on query results:
$emails = User::all()->pluck('email');
```

> **Note:** A single-model query like `User::find(1)` or `User::first()` returns a *model*, not a collection. Only the "many" methods (`get`, `all`, `cursor`) return collections.

---

## 3. Transforming: `map`, `mapWithKeys`, `flatMap`, `transform`

### `map` — transform each item, keys preserved

```php
collect([1, 2, 3])->map(fn ($n) => $n * 2)->all();
// Output: [2, 4, 6]

// The callback receives ($value, $key):
collect(['a' => 1, 'b' => 2])
    ->map(fn ($value, $key) => "$key=$value")
    ->all();
// Output: ['a' => 'a=1', 'b' => 'b=2']
```

### `mapWithKeys` — produce new keys

Return a single `[key => value]` pair from the callback to re-key the collection:

```php
$users = collect([
    ['id' => 10, 'name' => 'Ada'],
    ['id' => 20, 'name' => 'Linus'],
]);

$users->mapWithKeys(fn ($u) => [$u['id'] => $u['name']])->all();
// Output: [10 => 'Ada', 20 => 'Linus']
```

### `flatMap` — map then flatten one level

```php
collect([
    ['name' => 'Ada', 'tags' => ['php', 'sql']],
    ['name' => 'Linus', 'tags' => ['c', 'git']],
])->flatMap(fn ($u) => $u['tags'])->all();
// Output: ['php', 'sql', 'c', 'git']
```

### `transform` — like `map`, but MUTATES in place

```php
$c = collect([1, 2, 3]);
$c->transform(fn ($n) => $n * 10);
$c->all(); // [10, 20, 30]  — $c itself changed
```

Use `transform` only when you deliberately want to mutate the source; prefer `map` otherwise.

---

## 4. Filtering: `filter`, `reject`, `where*`, `only`, `except`, `unique`

### `filter` and `reject`

`filter` keeps items where the callback is truthy; `reject` keeps items where it's falsy — they are exact opposites.

```php
collect([1, 2, 3, 4])->filter(fn ($n) => $n % 2 === 0)->all();
// Output: [1 => 2, 3 => 4]  — note keys are PRESERVED

collect([1, 2, 3, 4])->reject(fn ($n) => $n % 2 === 0)->all();
// Output: [0 => 1, 2 => 3]
```

> **Gotcha:** `filter`/`reject` keep original keys. Chain `->values()` if you need a clean `0,1,2` re-index (common before JSON-encoding to get an array, not an object).

`filter()` with no argument removes all *falsy* values (`null`, `false`, `0`, `''`, `[]`):

```php
collect([0, 1, 2, null, 3, '', 'x'])->filter()->values()->all();
// Output: [1, 2, 3, 'x']
```

### `where`, `whereIn`, `whereNotIn`, `whereBetween` — SQL-like filtering on arrays/objects

```php
$products = collect([
    ['name' => 'Desk',  'price' => 200, 'type' => 'office'],
    ['name' => 'Chair', 'price' => 100, 'type' => 'office'],
    ['name' => 'Lamp',  'price' => 50,  'type' => 'home'],
]);

$products->where('type', 'office')->values()->all();
// Output: [['name'=>'Desk',...], ['name'=>'Chair',...]]

// Operator form:
$products->where('price', '>', 75)->pluck('name')->all();
// Output: ['Desk', 'Chair']

$products->whereIn('type', ['home'])->pluck('name')->all();
// Output: ['Lamp']

$products->whereNotIn('type', ['home'])->pluck('name')->all();
// Output: ['Desk', 'Chair']

$products->whereBetween('price', [60, 250])->pluck('name')->all();
// Output: ['Desk', 'Chair']
```

> **Gotcha:** `where()` uses loose comparison (`==`) by default. Use `whereStrict()` (or pass `===` style behavior via `where($key, '===', $value)`) when type matters, e.g. distinguishing `0` from `'0'` or `false`.

### `only` and `except` — pick/drop by key

```php
$data = collect(['name' => 'Ada', 'role' => 'admin', 'password' => 'secret']);

$data->only(['name', 'role'])->all();
// Output: ['name' => 'Ada', 'role' => 'admin']

$data->except(['password'])->all();
// Output: ['name' => 'Ada', 'role' => 'admin']
```

> **Tip:** `except`/`only` are a clean way to strip sensitive fields (passwords, tokens, internal flags) before returning data to a client. For real API output prefer an **API Resource** (`JsonResource`) or `$model->makeHidden([...])`, which give you a single, auditable place to control exposure rather than scattering `except()` calls. Never hand a raw model/collection that still contains secrets to the response.

### `unique` and `duplicates`

```php
collect([1, 1, 2, 2, 3])->unique()->values()->all();
// Output: [1, 2, 3]

// On a key:
collect([
    ['name' => 'Ada', 'team' => 'A'],
    ['name' => 'Bob', 'team' => 'A'],
    ['name' => 'Cy',  'team' => 'B'],
])->unique('team')->pluck('name')->all();
// Output: ['Ada', 'Cy']  — first of each team kept
```

> `unique()` uses loose comparison; `uniqueStrict()` uses strict.

---

## 5. Plucking, keying, and grouping

### `pluck` — extract a column

```php
$users = collect([
    ['id' => 1, 'name' => 'Ada'],
    ['id' => 2, 'name' => 'Bob'],
]);

$users->pluck('name')->all();
// Output: ['Ada', 'Bob']

// Pluck a value keyed by another column:
$users->pluck('name', 'id')->all();
// Output: [1 => 'Ada', 2 => 'Bob']

// Dot notation reaches into nested data:
collect([['user' => ['name' => 'Ada']]])->pluck('user.name')->all();
// Output: ['Ada']
```

### `keyBy` — re-index by a column

```php
$users->keyBy('id')->all();
// Output: [1 => ['id'=>1,'name'=>'Ada'], 2 => ['id'=>2,'name'=>'Bob']]

// keyBy with a callback for computed keys:
$users->keyBy(fn ($u) => 'user_' . $u['id'])->keys()->all();
// Output: ['user_1', 'user_2']
```

### `groupBy` — bucket items

```php
$people = collect([
    ['name' => 'Ada',   'dept' => 'eng'],
    ['name' => 'Bob',   'dept' => 'eng'],
    ['name' => 'Cleo',  'dept' => 'sales'],
]);

$people->groupBy('dept')->map->count()->all();
// Output: ['eng' => 2, 'sales' => 1]

// Multi-level grouping:
$grouped = $people->groupBy(['dept', fn ($p) => substr($p['name'], 0, 1)]);
```

### `countBy` — count occurrences

```php
collect(['a', 'b', 'a', 'c', 'b', 'a'])->countBy()->all();
// Output: ['a' => 3, 'b' => 2, 'c' => 1]

collect([1, 2, 3, 4, 5])->countBy(fn ($n) => $n % 2 === 0 ? 'even' : 'odd')->all();
// Output: ['odd' => 3, 'even' => 2]
```

---

## 6. Sorting and ordering

```php
$c = collect([3, 1, 2]);

$c->sort()->values()->all();        // [1, 2, 3]  (ascending)
$c->sortDesc()->values()->all();    // [3, 2, 1]

// sortBy on a key (ascending):
$people = collect([
    ['name' => 'Bob', 'age' => 40],
    ['name' => 'Ada', 'age' => 30],
]);
$people->sortBy('age')->pluck('name')->all();      // ['Ada', 'Bob']
$people->sortByDesc('age')->pluck('name')->all();  // ['Bob', 'Ada']

// sortBy with a callback (computed sort key):
$people->sortBy(fn ($p) => strlen($p['name']))->values();

// Sort by KEYS (for associative data):
collect(['c' => 3, 'a' => 1, 'b' => 2])->sortKeys()->all();
// Output: ['a' => 1, 'b' => 2, 'c' => 3]
```

> **Gotcha:** `sort` and `sortBy` **preserve keys** (so you usually want `->values()` afterward). Laravel 10+ also supports multi-column sorting: `sortBy([['age', 'asc'], ['name', 'desc']])`.

---

## 7. Reshaping: `values`, `keys`, `flatten`, `merge`, `concat`, `combine`, `zip`

```php
$c = collect([5 => 'a', 9 => 'b']);
$c->values()->all(); // ['a', 'b']     — drop keys, re-index
$c->keys()->all();   // [5, 9]         — just the keys

// flatten() collapses nested arrays. Default = all levels:
collect([1, [2, 3], [4, [5]]])->flatten()->all();    // [1, 2, 3, 4, 5]
collect([1, [2, [3]]])->flatten(1)->all();           // [1, 2, [3]]  — depth 1

// merge: associative keys overwrite, numeric keys append:
collect(['a' => 1, 'b' => 2])->merge(['b' => 9, 'c' => 3])->all();
// Output: ['a' => 1, 'b' => 9, 'c' => 3]

// concat: ALWAYS appends, re-indexing numeric keys (never overwrites):
collect([1, 2])->concat([3, 4])->all();   // [1, 2, 3, 4]

// combine: use this collection as keys, the argument as values:
collect(['name', 'age'])->combine(['Ada', 30])->all();
// Output: ['name' => 'Ada', 'age' => 30]

// zip: pair up by position:
collect([1, 2, 3])->zip(['a', 'b', 'c'])->all();
// Output: [[1,'a'], [2,'b'], [3,'c']]  (each is a Collection)
```

---

## 8. Slicing and taking: `take`, `skip`, `slice`, `chunk`, `splice`, `nth`

```php
collect([1, 2, 3, 4, 5])->take(2)->all();    // [1, 2]
collect([1, 2, 3, 4, 5])->take(-2)->all();   // [4, 5]  — from the end
collect([1, 2, 3, 4, 5])->skip(2)->values()->all(); // [3, 4, 5]
collect([1, 2, 3, 4, 5])->slice(1, 2)->values()->all(); // [2, 3]

// chunk: split into groups of N (great for grid layouts / batch jobs):
collect([1, 2, 3, 4, 5])->chunk(2)->toArray();
// Output: [[1, 2], [3, 4], [5]]

// nth: every Nth element:
collect(['a', 'b', 'c', 'd', 'e', 'f'])->nth(2)->all();
// Output: ['a', 'c', 'e']
```

`takeWhile` / `takeUntil` / `skipWhile` / `skipUntil` stop or start based on a predicate:

```php
collect([1, 2, 3, 4, 1])->takeWhile(fn ($n) => $n < 3)->all();
// Output: [1, 2]  — stops at the first 3, even though a 1 appears later
```

In Blade, `chunk` is the classic way to build rows of columns:

```blade
@foreach ($products->chunk(3) as $row)
    <div class="row">
        @foreach ($row as $product)
            <div class="col">{{ $product->name }}</div>
        @endforeach
    </div>
@endforeach
```

---

## 9. Mutating helpers: `push`, `put`, `prepend`, `pop`, `shift`, `pull`

These **mutate the collection in place** (most return the affected value, not the collection — so they're not chainable the same way):

```php
$c = collect([1, 2, 3]);

$c->push(4);             // [1, 2, 3, 4]            (returns $c — chainable)
$c->prepend(0);          // [0, 1, 2, 3, 4]         (returns $c — chainable)
$c->put('key', 'val');   // adds ['key' => 'val']   (returns $c — chainable)

$c->pop();               // returns 'val', removes last
$c->shift();             // returns 0, removes first
$c->pull('key');         // returns value at 'key' and removes it
```

`push` accepts multiple values in Laravel 9+: `$c->push(4, 5, 6)`.

---

## 10. Aggregating & reducing

```php
$nums = collect([1, 2, 3, 4]);

$nums->sum();      // 10
$nums->avg();      // 2.5   (alias: average())
$nums->min();      // 1
$nums->max();      // 4
$nums->count();    // 4
$nums->median();   // 2.5
$nums->mode();     // [...] most frequent value(s)

// sum/avg/min/max over a key or callback:
$orders = collect([
    ['total' => 100], ['total' => 250], ['total' => 50],
]);
$orders->sum('total');                      // 400
$orders->avg(fn ($o) => $o['total']);       // 133.33...
```

### `reduce` — fold to a single value

`reduce` walks the collection accumulating a result. The callback receives `($carry, $item)`, where `$carry` is the running result (seeded by the 2nd argument):

```php
collect([1, 2, 3, 4])->reduce(fn ($carry, $n) => $carry + $n, 0);
// Output: 10

// Build a string:
collect(['a', 'b', 'c'])->reduce(fn ($carry, $c) => $carry . strtoupper($c), '');
// Output: 'ABC'
```

### `implode` / `join`

```php
collect([1, 2, 3])->implode('-');             // '1-2-3'
collect([['name' => 'Ada'], ['name' => 'Bob']])->implode('name', ', '); // 'Ada, Bob'

// join() adds a special "final glue":
collect(['Ada', 'Bob', 'Cleo'])->join(', ', ' and ');
// Output: 'Ada, Bob and Cleo'
```

---

## 11. Searching & boolean checks

```php
$c = collect([1, 2, 3, 4]);

$c->contains(3);                          // true
$c->contains(fn ($n) => $n > 3);          // true
$c->containsStrict('3');                  // false (strict type check)

$c->first();                              // 1
$c->first(fn ($n) => $n > 2);            // 3
$c->last();                               // 4
$c->last(fn ($n) => $n < 3);            // 2

// firstWhere: shorthand for first() + where():
$users = collect([['name' => 'Ada', 'admin' => false], ['name' => 'Bob', 'admin' => true]]);
$users->firstWhere('admin', true);        // ['name' => 'Bob', 'admin' => true]

// every: do ALL items satisfy the predicate?
collect([2, 4, 6])->every(fn ($n) => $n % 2 === 0);  // true

// isEmpty / isNotEmpty
collect([])->isEmpty();                   // true
```

> **Gotcha:** On an *Eloquent* collection, `contains()` is smarter — see section 13. `contains($modelInstance)` and `contains($id)` both work by primary key.

---

## 12. Flow control: `tap`, `pipe`, `when`, `unless`, `partition`, `dd`, `dump`

### `tap` — peek without breaking the chain

`tap` hands the collection to a callback (for a side effect like logging) and then returns **the same collection** so the chain continues:

```php
collect([1, 2, 3])
    ->tap(fn ($c) => logger('before map: ' . $c->sum())) // side effect, returns original
    ->map(fn ($n) => $n * 2)
    ->all();
// Output: [2, 4, 6]  (and "before map: 6" gets logged)
```

### `pipe` — pass the whole collection to a callback and use its return

```php
collect([1, 2, 3])->pipe(fn ($c) => $c->sum() * 10);
// Output: 60
```

### `when` / `unless` — conditional chaining

These run a callback only when a condition is true (`when`) or false (`unless`) — the cleanest way to apply conditional steps without breaking the fluent chain:

```php
$includeAdmins = false;

$users = collect($allUsers)
    ->when($includeAdmins, fn ($c) => $c->push(['name' => 'root', 'admin' => true]))
    ->reject(fn ($u) => $u['admin'] ?? false);
```

A great real-world pattern is conditional sorting from request input:

```php
$sorted = $products->when(
    request('sort') === 'price',
    fn ($c) => $c->sortBy('price'),
    fn ($c) => $c->sortBy('name')   // optional "else" callback (3rd arg)
);
```

### `partition` — split into two by a predicate

```php
[$active, $inactive] = collect($users)->partition(fn ($u) => $u['active']);
// $active   = collection of active users
// $inactive = the rest
```

### `dd` and `dump` — debug

```php
collect([1, 2, 3])
    ->map(fn ($n) => $n * 2)
    ->dump()          // dumps [2,4,6] and CONTINUES the chain
    ->filter(fn ($n) => $n > 2)
    ->dd();           // dumps and DIES (stops execution)
```

`dump` prints and continues; `dd` ("dump and die") prints and halts — invaluable for inspecting a chain mid-flight.

---

## 13. Higher-order messages

A **higher-order message** is syntactic sugar that lets you call a method or access a property on *every item* without writing a closure. Instead of `->each(fn ($u) => $u->markAsVip())`, you write `->each->markAsVip()`.

Supported "proxy" methods include: `average`, `avg`, `contains`, `each`, `every`, `filter`, `first`, `flatMap`, `groupBy`, `keyBy`, `map`, `max`, `min`, `partition`, `reject`, `skipWhile`, `skipUntil`, `some`, `sortBy`, `sortByDesc`, `sum`, `takeWhile`, `takeUntil`, and `unique`.

```php
// Call a method on every model:
$users->each->markAsActive();

// map calling a method:
$names = $users->map->getFullName();

// Access a property in a where-style sum:
$totalAge = $users->sum->age;

// filter on a boolean accessor:
$admins = $users->filter->isAdmin();
```

These read beautifully and are extremely common in real Laravel code. Under the hood, `->each` returns a `HigherOrderCollectionProxy` whose `__call`/`__get` forwards to each element.

---

## 14. Lazy collections — processing huge data without blowing memory

A regular `Collection` holds the **entire dataset in memory** at once. If you load 1,000,000 rows, you have 1,000,000 array entries (and hydrated models) resident in RAM. For big jobs that's a problem.

`Illuminate\Support\LazyCollection` solves this using **PHP generators** (`yield`). A lazy collection produces items *one at a time, on demand*, so only one item is in memory during iteration.

### Creating a lazy collection

```php
use Illuminate\Support\LazyCollection;

$lazy = LazyCollection::make(function () {
    $handle = fopen('huge-log.txt', 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;   // one line at a time — file never fully loaded
    }
});

$errors = $lazy
    ->filter(fn ($line) => str_contains($line, 'ERROR'))
    ->take(10)        // stops reading the file after 10 matches!
    ->all();
```

The magic: because everything is lazy, `take(10)` lets the pipeline **stop early** — it never reads the rest of the file.

### Eloquent `cursor()` — the database equivalent

`cursor()` returns a `LazyCollection` and keeps only **one Eloquent model** in memory at a time, fetching rows from the DB driver one by one:

```php
use App\Models\User;

// BAD for 1M rows — loads everything into memory:
foreach (User::all() as $user) { /* ... */ }

// GOOD — one model in memory at a time:
foreach (User::cursor() as $user) {
    // process $user
}

// Still chainable like any collection:
User::cursor()
    ->filter(fn ($u) => $u->isEligible())
    ->each(fn ($u) => $u->notify());
```

> **Caveat:** `cursor()` runs a *single* DB query and streams results, so eager loading relationships (`with()`) is limited and you can't go back. For batching with eager loading, prefer `chunk()` or `chunkById()` / `lazyById()` on the query builder, which run multiple bounded queries.

`chunk` on the query (different from collection `chunk`) processes N records per query:

```php
User::where('active', true)->chunkById(1000, function ($users) {
    foreach ($users as $user) {
        // 1000 at a time; memory stays flat
    }
});
```

### When to use which

| Need | Use |
|------|-----|
| Small/medium result already in memory | `Collection` (default `get()`) |
| Iterate millions of DB rows, one query | `Model::cursor()` (LazyCollection) |
| Iterate millions of DB rows, with eager loading | `chunkById()` / `lazyById()` |
| Stream a huge file/API without loading it all | `LazyCollection::make(generator)` |

---

## 15. Eloquent Collection extras

When Eloquent returns rows, you get `Illuminate\Database\Eloquent\Collection`, which extends the base `Collection` and adds **model-aware** methods.

### `find` — get a model by primary key from the loaded set

```php
$users = User::all();          // already in memory
$users->find(5);               // the User with id 5, or null — no DB query
$users->find([5, 7]);          // a collection of those models
$users->find(5, $defaultUser); // 2nd arg = default if not found
```

### `modelKeys` — array of primary keys

```php
User::all()->modelKeys();
// Output: [1, 2, 3, 4, ...]  — equivalent to ->pluck('id')->all() but model-aware
```

### `load` — lazy eager-load a relationship onto already-fetched models

```php
$users = User::all();          // posts NOT loaded
$users->load('posts');         // ONE extra query loads posts for all users (no N+1)
$users->loadMissing('profile');// only loads if not already loaded
$users->loadCount('posts');    // adds posts_count without loading the posts
```

This is the cure for the classic N+1 problem when you didn't know you'd need a relationship at query time.

### Keyed `contains`, `except`, `only` — work by primary key

On an Eloquent collection these accept model instances or primary keys:

```php
$users = User::all();

$users->contains(3);            // true if a user with id 3 is present
$users->contains($someUser);    // true if that model (by key) is present

$users->only([1, 2, 3]);        // collection of models with those ids
$users->except([1, 2]);         // all models EXCEPT ids 1 and 2

$users->fresh();                // re-fetch all models from the DB
$users->diff($otherUsers);      // models not in the other collection (by key)
```

The base `Collection`'s `only`/`except` work on *array keys*; the Eloquent version works on *primary keys* — an easy point of confusion and a common interview gotcha.

### A few more Eloquent-only helpers worth knowing

```php
$users->findOrFail(5);          // like find(5) but throws ModelNotFoundException if absent
$users->intersect($otherUsers); // models present in BOTH collections (by key)

// makeHidden / makeVisible — control serialization per request, on every model:
$users->makeHidden(['password', 'remember_token']);  // strip before toJson()/toArray()
$users->makeVisible(['email']);                       // reveal normally-$hidden attrs
$users->append('full_name');                          // include an accessor in output

// toQuery() — turn the collection back into a query (whereIn on the keys):
$users->toQuery()->update(['status' => 'archived']);  // one UPDATE for all loaded models
```

> `find($key, $default = null)` on an Eloquent collection also accepts a default (returned when no model matches), and `find([$ids])` returns a collection of the matches.

---

## 16. Difference from plain arrays (summary)

| Aspect | Plain array | Collection |
|--------|-------------|------------|
| Chaining | No (nest calls) | Yes (fluent, left-to-right) |
| Return type | Mutates / mixed | New Collection (mostly immutable) |
| Method naming | Inconsistent (`array_map`, `array_filter`) | Consistent, discoverable |
| Higher-order msgs | No | Yes (`->each->save()`) |
| Lazy evaluation | No | `LazyCollection` |
| Convert | `(array)` cast | `->all()` / `->toArray()` |
| Memory | Array overhead | Same + wrapper (negligible) |

Collections **wrap** an array; `->all()` and `->toArray()` get the raw data back out. Collections are also `Countable`, `IteratorAggregate`, `ArrayAccess`, and `JsonSerializable`, so you can `count($c)`, `foreach ($c as ...)`, `$c[0]`, and `json_encode($c)` directly.

```php
$c = collect(['a' => 1, 'b' => 2]);
$c->all();       // ['a' => 1, 'b' => 2]  — top level only
$c->toArray();   // recursively converts nested collections/Arrayable too
$c->toJson();    // '{"a":1,"b":2}'
count($c);       // 2
$c['a'];         // 1
```

> **`all()` vs `toArray()`:** `all()` returns the underlying items untouched (nested collections stay collections; models stay models). `toArray()` recursively converts everything — nested collections, Eloquent models (via their own `toArray`), and any `Arrayable` — into plain arrays. Use `toArray()` for API output, `all()` when you want the raw items.

---

## ⚠️ Common Mistakes & Gotchas

**1. Forgetting that `filter`/`where`/`sort` preserve keys.**
After filtering, your keys become non-sequential (`[1 => ..., 3 => ...]`). When you `json_encode` that, you get a JSON **object** `{"1":...,"3":...}` instead of an **array**. *Fix:* call `->values()` before encoding when you want a JSON array.

```php
collect([1, 2, 3, 4])->filter(fn ($n) => $n > 2)->toJson(); // {"2":3,"3":4}  ❌
collect([1, 2, 3, 4])->filter(fn ($n) => $n > 2)->values()->toJson(); // [3,4]  ✅
```

**2. Thinking every method returns a new collection.**
`push`, `put`, `prepend`, `pop`, `shift`, `pull`, `transform`, `forget`, and `splice` **mutate in place**. *Fix:* don't rely on the original being unchanged after these, and remember `pop`/`shift`/`pull` return the *value*, not the collection — so they can't be chained.

**3. Calling `count()` or `all()` on a `cursor()`/`LazyCollection` and killing the memory benefit.**
`LazyCollection::all()` or `count()` forces the entire dataset into memory, defeating the whole point. *Fix:* keep lazy chains lazy — iterate or use early-terminating ops like `take`, `first`, `each`.

**4. Loose comparison surprises in `where`/`contains`/`unique`.**
`where('active', false)` will also match `0`, `''`, and `null` because of `==`. *Fix:* use the strict variants (`whereStrict`, `containsStrict`, `uniqueStrict`) or `where('key', '===', $value)`.

**5. Confusing base-`Collection` `only`/`except`/`contains` with the Eloquent versions.**
On a base collection these operate on *array keys/values*; on an Eloquent collection they operate on *model primary keys*. *Fix:* know which collection type you hold (`get()` → Eloquent; `collect([...])` → base).

**6. N+1 inside a collection loop.**
`User::all()->each(fn ($u) => $u->posts)` fires a query per user. *Fix:* eager load up front (`User::with('posts')->get()`) or `->load('posts')` on the collection afterward.

**7. `map` vs `each` confusion.**
`each` is for side effects and returns the original collection; `map` builds and returns a new one. Using `each` and expecting the transformed result back is a silent bug. *Fix:* use `map` when you want the transformed data.

---

## ✅ Best Practices

- **Prefer collection pipelines over manual loops** for readability — but don't force a 6-method chain when a simple `foreach` is clearer.
- **Add `->values()`** after filtering/sorting when the result feeds JSON or numeric-index assumptions.
- **Use higher-order messages** (`->each->save()`, `->sum->total`) when the closure would just call one method/property.
- **Reach for `when()`/`unless()`** instead of breaking a chain with `if` statements for conditional transformations.
- **Use `cursor()` / `lazyById()` / `chunkById()`** for any job touching large tables; never `Model::all()` on big data.
- **Use strict variants** (`whereStrict`, `containsStrict`) whenever `0`, `'0'`, `false`, `null` could collide.
- **Use `toArray()` for API responses**, `all()` when you need the raw underlying items.
- **Prefer DB-level filtering/sorting** when possible: `User::where(...)->orderBy(...)->get()` lets the database do the heavy lifting instead of pulling everything into a collection and filtering in PHP.
- **Type-hint with `Collection`** in signatures for clarity, and consider generics-style docblocks (`@return Collection<int, User>`) for IDE support.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between a Collection and a plain PHP array?**
A Collection is an object wrapper (`Illuminate\Support\Collection`) around an array that provides 100+ consistent, chainable methods, mostly returning new (immutable) instances. It implements `ArrayAccess`, `Countable`, `IteratorAggregate`, and `JsonSerializable`, so it behaves array-like while enabling fluent pipelines. Plain arrays rely on inconsistent `array_*` functions and can't be chained.

**Q2. Does Eloquent's `get()` return an array or a collection? What about `find()`?**
`get()`, `all()`, and `cursor()` return collections (`Illuminate\Database\Eloquent\Collection`, a subclass of the base one). `find($id)` and `first()` return a single model (or null). `find([$ids])` returns a collection.

**Q3. Which collection methods mutate vs. return new instances?**
The vast majority return a new collection (immutable). The mutating ones are `push`, `put`, `prepend`, `pop`, `shift`, `pull`, `transform`, `forget`, and `splice`. Knowing this prevents bugs where you expect the original to be untouched.

**Q4. What is a LazyCollection and when would you use one? (under-the-hood)**
`LazyCollection` is backed by a **PHP generator** (`yield`). Instead of materializing all items in memory, it produces them one at a time on demand. Because operations are lazy, terminal/early-stopping methods like `take`/`first` can short-circuit the source. Use it for huge datasets — e.g. `Model::cursor()` streams one model at a time, and `LazyCollection::make($generator)` can stream a multi-GB file. The trade-off: you can iterate only forward, and calling `all()`/`count()` collapses the laziness.

**Q5. How does `cursor()` differ from `chunk()`?**
`cursor()` runs a single query and uses the DB driver to stream rows, hydrating one model at a time (lowest memory, but limited eager loading and single pass). `chunk()`/`chunkById()` run multiple bounded queries (e.g. 1000 rows each) and hand you a full collection per chunk — slightly more memory per batch but supports eager loading and is safer when modifying rows during iteration (`chunkById` avoids skipping rows).

**Q6. What are higher-order messages? (under-the-hood)**
They're a shorthand that removes the closure: `$users->each->activate()` instead of `$users->each(fn ($u) => $u->activate())`. Accessing a supported method name (e.g. `->each`) returns a `HigherOrderCollectionProxy`; its `__call`/`__get` then applies the method/property to each element. Supported on `map`, `each`, `filter`, `sum`, `reject`, `sortBy`, and more.

**Q7. After `filter()`, why does `json_encode` sometimes return an object instead of an array?**
`filter()` preserves original keys, so they may become non-sequential. JSON only encodes sequential 0-based integer keys as an array; anything else becomes an object. Call `->values()` to re-index and get a JSON array.

**Q8. `all()` vs `toArray()` — what's the difference?**
`all()` returns the underlying items as-is (nested collections/models stay as objects). `toArray()` recursively converts everything (including Eloquent models and `Arrayable`s) into plain arrays — what you want for API output.

**Q9. How would you avoid N+1 queries when you already have a collection of models?**
Use `$collection->load('relation')` (or `loadMissing`) to lazy eager-load with a single extra query, or fetch with `with()` up front. `loadCount()` adds counts without loading the related models.

**Q10. What do the Eloquent collection's `contains`, `only`, `except`, and `modelKeys` do that the base ones don't?**
The Eloquent versions are keyed by **primary key**: `contains($id)` / `contains($model)`, `only([$ids])`, `except([$ids])`, and `modelKeys()` returns the array of primary keys. The base collection's `only`/`except`/`contains` operate on array keys/values instead.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ── CREATE ────────────────────────────────────────────
collect([1, 2, 3]);              Collection::make([...]);
Collection::times(3, fn ($i) => $i);  Collection::wrap($x);
User::all(); User::get(); User::cursor();   // Eloquent → collections

// ── TRANSFORM ─────────────────────────────────────────
->map(fn ($v, $k) => ...)        // transform, keep keys
->mapWithKeys(fn ($v) => [$k => $v])
->flatMap(fn ($v) => [...])      // map + flatten 1 level
->transform(...)                 // MUTATES in place

// ── FILTER ────────────────────────────────────────────
->filter(fn ($v) => ...)         ->reject(fn ($v) => ...)
->where('k', '>', 5)             ->whereIn('k', [...])
->whereNotIn(...)                ->whereBetween('k', [a, b])
->only([...])  ->except([...])   ->unique('k')  ->filter() // drop falsy

// ── EXTRACT / GROUP ───────────────────────────────────
->pluck('name', 'id')            ->keyBy('id')
->groupBy('dept')                ->countBy()

// ── SORT ──────────────────────────────────────────────
->sort() ->sortDesc()            ->sortBy('k') ->sortByDesc('k')
->sortKeys()                     ->values() ->keys()

// ── COMBINE / RESHAPE ─────────────────────────────────
->merge([...])  ->concat([...])  ->combine([...])  ->zip([...])
->flatten($depth)

// ── SLICE ─────────────────────────────────────────────
->take(n) ->skip(n) ->slice(off, len) ->chunk(n) ->nth(n)
->takeWhile(...) ->skipUntil(...)

// ── MUTATE (in place) ─────────────────────────────────
->push($v) ->prepend($v) ->put($k, $v) ->pop() ->shift() ->pull($k)

// ── AGGREGATE ─────────────────────────────────────────
->sum('k') ->avg() ->min() ->max() ->count() ->countBy()
->reduce(fn ($carry, $v) => ..., $initial)
->implode('k', ', ') ->join(', ', ' and ')

// ── SEARCH / BOOL ─────────────────────────────────────
->first() ->firstWhere('k', $v) ->last()
->contains($v|fn) ->containsStrict() ->every(fn) ->isEmpty()

// ── FLOW ──────────────────────────────────────────────
->tap(fn ($c) => ...)            ->pipe(fn ($c) => ...)
->when($cond, fn, $else)         ->unless($cond, fn)
->partition(fn) → [$pass, $fail] ->dump() ->dd()

// ── HIGHER-ORDER MESSAGES ─────────────────────────────
->each->save()  ->map->getName()  ->sum->total  ->filter->isActive()

// ── LAZY (huge data) ──────────────────────────────────
LazyCollection::make($generator)  User::cursor()
Query: ->chunkById(1000, fn) ->lazyById()

// ── ELOQUENT EXTRAS ───────────────────────────────────
->find($id) ->find([$ids])  ->modelKeys()  ->fresh()
->load('rel') ->loadMissing() ->loadCount('rel')
->contains($id|$model) ->only([$ids]) ->except([$ids])

// ── OUT ───────────────────────────────────────────────
->all()  ->toArray()  ->toJson()  count($c)  $c[0]
```

---

## 🧪 Mini Exercises

1. **Pipeline practice.** Given `collect([['name' => 'Ada', 'score' => 90], ['name' => 'Bob', 'score' => 55], ['name' => 'Cleo', 'score' => 75]])`, produce an array of names (re-indexed `0,1,...`) of everyone with a score of 70 or higher, sorted alphabetically. Hint: `filter` → `sortBy` → `pluck` → `values`.

2. **Grouping + aggregation.** Given a collection of orders, each `['customer' => 'X', 'total' => 120]`, build a map of `customer => total spent` and then find the single biggest-spending customer. Use `groupBy`, `map`, `sum`, and a sorting method.

3. **Re-keying.** Take `collect([['sku' => 'A1', 'qty' => 3], ['sku' => 'B2', 'qty' => 5]])` and turn it into `['A1' => 3, 'B2' => 5]` using exactly one method.

4. **Lazy memory.** Write a `LazyCollection` that reads a (hypothetical) `access.log`, keeps only lines containing `"500"`, and returns the first 5 — without ever loading the whole file. Explain in a comment why `take(5)` makes this efficient.

5. **Eloquent extras.** Given `$users = User::all()`, (a) get the array of primary keys, (b) lazy-load their `posts` relationship without an N+1, and (c) return only the users whose id is in `[2, 4, 6]` — using the keyed Eloquent methods, not `filter`.
