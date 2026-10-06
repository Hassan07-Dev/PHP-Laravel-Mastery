# Query Builder & Database

Laravel ships with two complementary ways to talk to your database: the **Query Builder** (a fluent, expressive PHP API that produces raw SQL) and **Eloquent** (an Object-Relational Mapper, or ORM, that maps rows to model objects). This module is about the Query Builder and the lower-level `DB` facade that powers everything underneath. Master this and you'll understand exactly what SQL Laravel generates, how to keep it safe from injection, and how to scale it from a 10-row demo to a 10-million-row job.

> **Facade** = a static-looking proxy class (e.g. `DB::`) that resolves to a concrete service object out of Laravel's service container. `DB::table(...)` is really calling a method on the singleton `Illuminate\Database\DatabaseManager`.

---

**What you'll learn**

- The two layers of the `DB` facade (raw `DB::select`/`statement` vs the fluent builder)
- When to reach for the Query Builder vs Eloquent, and why the choice matters
- The full `where` family, ordering, grouping, `having`, limits, and joins
- Aggregates, inserts (including `upsert`/`insertOrIgnore`), updates, and deletes
- Raw expressions (`DB::raw`, `selectRaw`, `whereRaw`) and how to keep bindings safe from SQL injection
- Subqueries, subquery joins, and subquery selects
- Streaming large datasets safely with `chunk`, `chunkById`, `lazy`, and `cursor`
- Pagination (`paginate`, `simplePaginate`, `cursorPaginate`) and the trade-offs
- Transactions, deadlock retries, multiple connections, and debugging with `toSql`/`DB::listen`/Telescope

---

## Why two tools? Query Builder vs Eloquent

Eloquent is wonderful for **domain logic**: a `User` model with relationships, casts, accessors, events, and scopes reads like business language. But that convenience has a cost. Hydrating a row into a full model object allocates memory, runs casts, fires events, and tracks the attribute "dirty" state. When you fetch 200,000 rows just to sum a column, that overhead is wasted.

The Query Builder returns **plain `stdClass` objects** (or arrays), does no hydration, fires no model events, and is the right tool for:

- Bulk reporting / analytics aggregates
- Heavy bulk inserts and updates where you don't need model events
- Joins across tables that don't map cleanly to a single model
- Anywhere you want to see and control the exact SQL

Rule of thumb: **Eloquent for domain reads/writes where behavior matters; Query Builder for set-based, performance-sensitive, or reporting work.** And remember — Eloquent *is* a Query Builder underneath. Every `where`, `orderBy`, `join` you learn here works identically on an Eloquent query (`User::where(...)`), because `Illuminate\Database\Eloquent\Builder` forwards unknown calls to the underlying `Illuminate\Database\Query\Builder`.

```php
// Query Builder — returns Illuminate\Support\Collection of stdClass
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->where('active', true)->get();
echo $users->first()->email; // property access on stdClass

// Eloquent — returns a Collection of App\Models\User instances
use App\Models\User;

$users = User::where('active', true)->get();
echo $users->first()->email; // attribute access on a model
```

---

## The `DB` facade: raw queries vs the fluent builder

The `DB` facade has two layers. The low-level layer runs **raw SQL strings** directly; the high-level layer (`DB::table(...)`) is the **fluent query builder**. Both go through the same PDO connection and both support bindings.

```php
use Illuminate\Support\Facades\DB;

// Raw SELECT — always returns a plain PHP array of stdClass, never a Collection
$users = DB::select('select * from users where active = ?', [1]);

// Named bindings work too
$users = DB::select('select * from users where id = :id', ['id' => 1]);

// Single scalar value (Laravel 9+)
$count = DB::scalar('select count(*) from users where active = ?', [1]);

// Raw write statements — insert/update/delete return affected-row info
DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
$affected = DB::update('update users set votes = 100 where name = ?', ['Anita']);
$deleted  = DB::delete('delete from users where votes < ?', [100]);

// statement() for DDL / anything that returns no rows
DB::statement('drop table users');

// unprepared() binds NOTHING — never pass user input to it (injection risk)
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> Note the return-type difference that trips people up: `DB::select(...)` returns a plain **`array`** of `stdClass`, whereas `DB::table(...)->get()` returns an **`Illuminate\Support\Collection`** of `stdClass`. `DB::statement()` *does* accept a bindings array, but `DB::unprepared()` binds nothing — never pass any user-controlled string to `unprepared()`. Inside a transaction, also avoid statements that cause an implicit commit (e.g. `CREATE TABLE`), which silently ends the transaction.

The rest of this module uses the fluent builder (`DB::table(...)`), which collects bindings for you automatically.

---

## Getting a query: `DB::table()` and `select`

Every query starts from a table. From there you choose columns; the default is `*`.

```php
use Illuminate\Support\Facades\DB;

// Retrieve all columns of all rows
$rows = DB::table('users')->get();
// Output: Illuminate\Support\Collection of stdClass {id, name, email, ...}

// Select specific columns (and alias one)
$rows = DB::table('users')
    ->select('id', 'name', 'email as contact')
    ->get();

// addSelect appends to an existing select list
$query = DB::table('users')->select('id');
$rows  = $query->addSelect('name')->get();
```

### Retrieval methods you must know

```php
DB::table('users')->get();                  // Collection of all matching rows
DB::table('users')->first();                // first row or null
DB::table('users')->find(7);                // row where id = 7 (shortcut for where('id',7)->first())
DB::table('users')->value('email');         // single scalar: the email of the first row
DB::table('users')->pluck('email');         // Collection of one column: ['a@x.com', ...]
DB::table('users')->pluck('email', 'id');   // keyed Collection: [7 => 'a@x.com', ...]
DB::table('users')->count();                // int
DB::table('users')->exists();               // bool — far cheaper than count() > 0
DB::table('users')->doesntExist();          // bool
```

> `value()` and `pluck()` are interview favorites: they avoid hydrating whole rows when you only need one column. `exists()` issues a `SELECT EXISTS(...)` and stops at the first match — prefer it over `count() > 0`, which scans/counts every matching row.

### `distinct`

```php
DB::table('orders')->distinct()->pluck('status');
// SQL: select distinct "status" from "orders"
```

---

## Filtering: the `where` family

`where` is the workhorse. The 3-argument form is `where($column, $operator, $value)`; the 2-argument form assumes `=`.

```php
DB::table('users')->where('votes', '=', 100);
DB::table('users')->where('votes', 100);          // identical, = is implied
DB::table('users')->where('votes', '>=', 100);
DB::table('users')->where('name', 'like', 'T%');
```

### Multiple conditions: AND vs OR

Chaining `where` joins with **AND**. Use `orWhere` for **OR**.

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->where('active', true);          // votes > 100 AND active = 1

DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'Abigail');     // votes > 100 OR name = 'Abigail'
```

**Gotcha:** mixing `where` and `orWhere` without grouping produces operator-precedence bugs. Group with a closure to wrap conditions in parentheses:

```php
DB::table('users')
    ->where('active', true)
    ->where(function ($query) {
        $query->where('votes', '>', 100)
              ->orWhere('name', 'Abigail');
    })
    ->get();
// SQL: ... where "active" = 1 and ("votes" > 100 or "name" = 'Abigail')
```

You can also pass an array to `where` for several AND equality/operator conditions at once:

```php
DB::table('users')->where([
    ['status', '=', 'active'],
    ['subscribed', '=', true],
])->get();
```

### Specialized `where` clauses

```php
// IN / NOT IN
DB::table('users')->whereIn('id', [1, 2, 3]);
DB::table('users')->whereNotIn('id', [1, 2, 3]);

// NULL checks
DB::table('users')->whereNull('deleted_at');
DB::table('users')->whereNotNull('email_verified_at');

// BETWEEN
DB::table('users')->whereBetween('votes', [1, 100]);
DB::table('users')->whereNotBetween('votes', [1, 100]);

// Compare two columns to each other
DB::table('users')->whereColumn('updated_at', '>', 'created_at');
DB::table('users')->whereColumn([
    ['first_name', '=', 'last_name'],
    ['updated_at', '>', 'created_at'],
]); // joined with AND

// Date / time helpers (database-portable extraction)
DB::table('users')->whereDate('created_at', '2026-06-18');
DB::table('users')->whereMonth('created_at', '06');
DB::table('users')->whereDay('created_at', '18');
DB::table('users')->whereYear('created_at', '2026');
DB::table('users')->whereTime('created_at', '<=', '11:00:00');
```

> `whereDate('created_at', '2026-06-18')` generates `where date("created_at") = ?`. Convenient, but wrapping the column in a function (`date(...)`) usually **defeats an index** on `created_at`. For hot paths prefer a range: `->whereBetween('created_at', ['2026-06-18 00:00:00', '2026-06-18 23:59:59'])`, which can use the index.

### Newer `where` conveniences (Laravel 10/11/12)

```php
// String helpers (Laravel 11+): a database-agnostic LIKE that is
// case-INSENSITIVE by default (Laravel normalizes per driver).
DB::table('users')->whereLike('name', '%Tay%');                          // case-insensitive
DB::table('users')->whereLike('name', '%tay%', caseSensitive: true);     // opt in to case-sensitive
DB::table('users')->orWhereLike('name', '%Tay%');
DB::table('users')->whereNotLike('name', '%spam%');

// Compare against any-of / all-of a set of columns (Laravel 10.47+)
DB::table('users')->whereAny(['title', 'body'], 'like', '%laravel%'); // OR across columns
DB::table('users')->whereAll(['name', 'email'], 'like', '%@%');       // AND across columns
DB::table('orders')->whereNone(['status'], '=', 'cancelled');         // none match (L11+)
```

---

## Ordering, grouping, `having`, and limits

```php
// ORDER BY
DB::table('users')->orderBy('name', 'asc')->orderBy('email', 'desc');
DB::table('users')->orderByDesc('created_at');
DB::table('users')->latest();          // orderBy('created_at', 'desc')
DB::table('users')->oldest();          // orderBy('created_at', 'asc')
DB::table('users')->latest('updated_at');
DB::table('users')->inRandomOrder();   // ORDER BY RANDOM()/RAND() — expensive on big tables
DB::table('users')->reorder();         // strip all existing order clauses

// GROUP BY + HAVING
DB::table('orders')
    ->select('user_id', DB::raw('SUM(total) as revenue'))
    ->groupBy('user_id')
    ->having('revenue', '>', 1000)
    ->get();

DB::table('orders')->groupByRaw('user_id, status'); // when you need raw grouping

// LIMIT / OFFSET
DB::table('users')->limit(10)->offset(20)->get();
DB::table('users')->skip(20)->take(10)->get();   // skip = offset, take = limit (aliases)
```

> **WHERE filters rows before grouping; HAVING filters groups after aggregation.** You cannot use an aggregate (`SUM`, `COUNT`) in `where`, only in `having`. That distinction is a classic SQL interview question.

---

## Joins

```php
// INNER JOIN — only rows with a match in both tables
DB::table('users')
    ->join('contacts', 'users.id', '=', 'contacts.user_id')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'contacts.phone', 'orders.price')
    ->get();

// LEFT JOIN — all users, contacts where present, NULLs otherwise
DB::table('users')->leftJoin('posts', 'users.id', '=', 'posts.user_id');

// RIGHT JOIN — all posts, users where present
DB::table('users')->rightJoin('posts', 'users.id', '=', 'posts.user_id');

// CROSS JOIN — cartesian product (every row with every row); no ON clause
DB::table('sizes')->crossJoin('colours')->get();
```

### Advanced join closures

Pass a closure as the second argument to build multi-condition `ON` clauses. Inside the closure use `on`, `orOn`, and `where`:

```php
DB::table('users')
    ->join('contacts', function ($join) {
        $join->on('users.id', '=', 'contacts.user_id')
             ->where('contacts.is_primary', '=', true)
             ->whereNull('contacts.deleted_at');
    })
    ->get();
// SQL: ... inner join "contacts"
//       on "users"."id" = "contacts"."user_id"
//      and "contacts"."is_primary" = ?
//      and "contacts"."deleted_at" is null
```

> Inside a join closure, use `on()` to compare two **columns** and `where()` to compare a column to a **value/binding**. Mixing them up (`on('contacts.is_primary', '=', true)`) treats `true` as a column name and breaks the SQL.

### Subquery joins (`joinSub`)

Join against a derived table (a subquery aliased as a table):

```php
$latestPosts = DB::table('posts')
    ->select('user_id', DB::raw('MAX(created_at) as last_post_at'))
    ->groupBy('user_id');

DB::table('users')
    ->joinSub($latestPosts, 'latest_posts', function ($join) {
        $join->on('users.id', '=', 'latest_posts.user_id');
    })
    ->get();
// leftJoinSub and rightJoinSub also exist
```

---

## Aggregates

```php
DB::table('orders')->count();            // int
DB::table('orders')->count('coupon_id'); // counts non-null coupon_id values
DB::table('orders')->sum('total');       // numeric
DB::table('orders')->avg('total');       // numeric (alias: average)
DB::table('orders')->min('total');
DB::table('orders')->max('total');

// Conditional existence (cheap — stops at first row)
if (DB::table('orders')->where('user_id', 7)->exists()) {
    // ...
}
```

Aggregates execute the query immediately and return a scalar (not a Collection). Combine with `groupBy` + `select(DB::raw(...))` for per-group aggregates, as shown earlier.

---

## Inserts

```php
// Single row
DB::table('users')->insert([
    'email' => 'kayla@example.com',
    'votes' => 0,
]);
// Output: true on success

// Bulk insert — one SQL statement, many value tuples
DB::table('users')->insert([
    ['email' => 'picard@example.com', 'votes' => 0],
    ['email' => 'janeway@example.com', 'votes' => 0],
]);

// Insert and get the new auto-increment id
$id = DB::table('users')->insertGetId([
    'email' => 'sisko@example.com',
    'votes' => 0,
]);
// Output: 1234 (int)

// Insert, silently skipping rows that violate unique constraints / errors
DB::table('users')->insertOrIgnore([
    ['id' => 1, 'email' => 'sisko@example.com'],
    ['id' => 2, 'email' => 'kira@example.com'],
]);
// Output: number of rows actually inserted
```

> `insertGetId` accepts a single row only. For PostgreSQL, pass the sequence column name as the second argument if it isn't `id`: `insertGetId([...], 'sequence_column')`.

### `upsert` — insert-or-update in one statement

`upsert(rows, uniqueBy, update)`: inserts each row; if a row collides on the columns in `uniqueBy` (which must be a primary key or unique index), it updates the columns listed in `update`.

```php
DB::table('flights')->upsert(
    [
        ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
        ['departure' => 'Chicago', 'destination' => 'New York',  'price' => 150],
    ],
    ['departure', 'destination'], // unique-by columns (need a unique index!)
    ['price']                     // columns to update on conflict
);
```

> If you omit the third argument, **all columns are updated** on conflict. On MySQL the third argument is effectively advisory (it always updates all matched columns under the hood via `ON DUPLICATE KEY UPDATE`), but on PostgreSQL/SQLite the `uniqueBy` index is mandatory. Without a matching unique index, `upsert` throws or silently misbehaves depending on driver.

---

## Updates

```php
// Update all matching rows; returns the number of affected rows
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1, 'name' => 'Kayla']);
// Output: 1

// JSON column update (-> syntax)
DB::table('users')->where('id', 1)->update(['options->enabled' => true]);

// increment / decrement (atomic, single UPDATE)
DB::table('users')->where('id', 1)->increment('votes');       // +1
DB::table('users')->where('id', 1)->increment('votes', 5);    // +5
DB::table('users')->where('id', 1)->decrement('votes', 5);    // -5

// increment/decrement AND set other columns in the same statement
DB::table('users')->where('id', 1)->increment('votes', 1, ['name' => 'Kayla']);

// increment several columns at once (Laravel 9+)
DB::table('users')->incrementEach(['votes' => 5, 'balance' => 100]);
```

> `increment` runs `SET votes = votes + 1` in the database — it is **atomic** and avoids the read-modify-write race you'd hit with `$row->votes + 1` in PHP under concurrency. This is a common "why" interview question.

### `updateOrInsert`

Update the first matching row; if none exists, insert a new one.

```php
DB::table('users')->updateOrInsert(
    ['email' => 'john@example.com'],   // attributes to find by
    ['votes' => 2]                     // values to set/insert
);
// Returns bool. Not atomic by itself — wrap in a transaction or rely on a unique index if racing.
```

---

## Deletes & truncate

```php
DB::table('users')->where('votes', '<', 100)->delete(); // returns affected row count
DB::table('users')->delete();                            // deletes ALL rows (DELETE FROM users)
DB::table('users')->truncate();                          // TRUNCATE — resets auto-increment, no events, can't rollback in MySQL
```

> `delete()` with no `where` deletes every row but is a logged, transactional `DELETE`. `truncate()` is faster, resets the auto-increment counter, but in MySQL it implicitly commits and **cannot be rolled back** inside a transaction. Choose deliberately.

---

## Raw expressions and binding safety

When the fluent API can't express something (DB functions, vendor-specific SQL), drop to raw. But raw SQL is where **SQL injection** lives, so always separate *static SQL* from *user data (bindings)*.

> **Binding** = a `?` placeholder whose value the database driver sends separately from the SQL text, so the value can never be interpreted as SQL. This is what makes parameterized queries injection-proof.

```php
use Illuminate\Support\Facades\DB;

// DB::raw — inject a literal SQL fragment into a clause
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->groupBy('status')
    ->get();

// selectRaw — same, but lets you pass BINDINGS as the 2nd arg (SAFE)
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();

// whereRaw / orWhereRaw with bindings
DB::table('orders')->whereRaw('price > IF(state = "TX", ?, 100)', [200])->get();

// havingRaw
DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();

// orderByRaw / groupByRaw
DB::table('orders')->orderByRaw('updated_at - created_at DESC')->get();
```

**The cardinal rule:**

```php
// 🔴 DANGEROUS — user input concatenated into raw SQL = injection
$status = request('status');
DB::table('orders')->whereRaw("status = '$status'")->get();

// 🟢 SAFE — value passed as a binding
DB::table('orders')->whereRaw('status = ?', [request('status')])->get();

// 🟢 Even better — just use the fluent API, which always binds
DB::table('orders')->where('status', request('status'))->get();
```

`DB::raw()` (and the `DB::raw`-equivalent `Illuminate\Database\Query\Expression`) does **not** escape anything — never put user input inside it. If you need a value, use the `*Raw` methods' binding array or the fluent API.

---

## Subqueries

### Subquery in `select` (`selectSub`)

```php
$query = DB::table('users')
    ->select('users.*')
    ->selectSub(function ($q) {
        $q->from('posts')
          ->selectRaw('count(*)')
          ->whereColumn('posts.user_id', 'users.id');
    }, 'post_count')
    ->get();
// each row gains a "post_count" column
```

### Subquery as a value in `where`

```php
DB::table('users')
    ->where('amount', '<', function ($q) {
        $q->selectRaw('avg(i.amount)')->from('incomes as i');
    })
    ->get();

// whereIn with a subquery
DB::table('users')
    ->whereIn('id', function ($q) {
        $q->select('user_id')->from('orders')->where('total', '>', 1000);
    })
    ->get();

// whereExists — efficient existence check, no data returned from the subquery
DB::table('users')
    ->whereExists(function ($q) {
        $q->select(DB::raw(1))
          ->from('orders')
          ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

### Ordering by a subquery (`fromSub` / subquery select)

```php
// Order users by the date of their most recent post
$lastPost = DB::table('posts')
    ->whereColumn('user_id', 'users.id')
    ->orderByDesc('created_at')
    ->limit(1);

DB::table('users')
    ->orderByDesc($lastPost->select('created_at'))
    ->get();
```

---

## Processing large result sets without exhausting memory

Calling `->get()` on a million-row table loads every row into memory at once — a fast way to hit a PHP memory limit. Laravel offers four streaming strategies.

### `chunk` — process in fixed-size batches

```php
DB::table('users')->orderBy('id')->chunk(1000, function ($users) {
    foreach ($users as $user) {
        // process $user
    }
    // return false; // returning false stops further chunks
});
```

Each chunk runs its own `SELECT ... LIMIT 1000 OFFSET n`. **Danger:** if you *modify* the rows you're chunking over (e.g. flipping a flag you also filter on), the OFFSET windows shift and you'll skip rows.

### `chunkById` — safe when updating during iteration

```php
DB::table('users')
    ->where('active', false)
    ->chunkById(1000, function ($users) {
        foreach ($users as $user) {
            DB::table('users')->where('id', $user->id)->update(['active' => true]);
        }
    });
// Uses WHERE id > lastId instead of OFFSET — stable even as rows change
```

`chunkById` paginates by the last-seen primary key (`where id > ?`) rather than `OFFSET`, so concurrent or in-loop modifications don't cause skips. Prefer it for any read-then-write loop. (`lazyById` is the lazy-collection equivalent.)

### `lazy` and `cursor` — `LazyCollection` streaming

```php
// lazy(): runs chunked queries under the hood, yields one model/row at a time
DB::table('users')->orderBy('id')->lazy()->each(function ($user) {
    // ...
});

DB::table('users')->lazyById(1000, 'id'); // chunk-by-id variant

// cursor(): ONE query, streams rows from the DB driver via a PHP generator
DB::table('users')->orderBy('id')->cursor()->each(function ($user) {
    // ...
});
```

> **`lazy` vs `cursor`:** `lazy()` runs multiple chunked queries but keeps only one chunk's-worth of state; memory is bounded regardless of dataset size. `cursor()` issues a single query and uses a generator, so PHP-side memory stays tiny — but with MySQL's default buffered queries the **driver** may still buffer the whole result set, and a long-lived cursor holds a DB connection open. For huge tables, `lazyById` is usually the safest, most predictable choice.

---

## Pagination

```php
// Length-aware: runs an extra COUNT(*) so you get total pages & numbered links
$users = DB::table('users')->paginate(15);
// In Blade: {{ $users->links() }}

// Simple: only "next/previous" — skips the COUNT, fetches perPage+1 rows
$users = DB::table('users')->simplePaginate(15);

// Cursor: keyset pagination — uses WHERE on the ordered column, no OFFSET
$users = DB::table('users')->orderBy('id')->cursorPaginate(15);
```

| Method | Total count query? | Random page jump? | Best for |
|---|---|---|---|
| `paginate` | Yes (extra `COUNT(*)`) | Yes (numbered pages) | Admin tables, anything needing total/last page |
| `simplePaginate` | No | No (next/prev only) | Long lists where total is irrelevant |
| `cursorPaginate` | No | No (next/prev only) | Huge / infinite-scroll feeds; stable under inserts |

> **Why `cursorPaginate` scales:** OFFSET-based pagination forces the DB to scan and discard all preceding rows, so deep pages get slower and slower (the "deep pagination problem"). Cursor pagination uses `where id > ?`, which an index serves in constant time regardless of how deep you go. The trade-off: you must order by a unique, sequential column and you lose numbered page links. The cursor is an opaque, encoded string in the query (`?cursor=...`), not a page number.

---

## Transactions

A **transaction** groups statements so they all commit together or all roll back — atomicity. Use them whenever a logical operation spans multiple writes (e.g. debit one account, credit another).

### The closure form (recommended) — with automatic deadlock retries

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::table('accounts')->where('id', 1)->decrement('balance', 100);
    DB::table('accounts')->where('id', 2)->increment('balance', 100);
}, attempts: 3); // retry up to 3 times if a deadlock occurs
```

If the closure throws, the transaction is rolled back automatically and the exception re-thrown. If it returns normally, it commits. The second argument is the number of **attempts** — on a deadlock or serialization failure, Laravel rolls back and re-runs the *entire* closure (so keep it idempotent; don't send emails inside it).

### Manual control

```php
DB::beginTransaction();
try {
    DB::table('accounts')->where('id', 1)->decrement('balance', 100);
    DB::table('accounts')->where('id', 2)->increment('balance', 100);
    DB::commit();
} catch (\Throwable $e) {
    DB::rollBack();
    throw $e;
}
```

> **Deadlock** = two transactions each hold a lock the other needs, so neither can proceed; the DB kills one with a deadlock error. The closure form's `attempts` parameter is the idiomatic fix — it re-runs after the loser rolls back. Manual transactions get no automatic retry. Note that nested `DB::transaction()` calls use **savepoints**, not real nested transactions (most engines don't support true nesting).

---

## Multiple connections

Define connections in `config/database.php`; select one with `DB::connection('name')`.

```php
// Query a specific connection
$users = DB::connection('reporting')->table('users')->get();

// Read/write splitting: Laravel auto-routes SELECTs to read hosts and
// writes to the write host when you configure 'read' and 'write' in the config.
DB::connection('mysql')->table('users')->insert([...]); // goes to write host
DB::connection('mysql')->table('users')->get();         // goes to a read host
```

```php
// config/database.php (read/write split example)
'mysql' => [
    'read'  => ['host' => ['192.168.1.2']],
    'write' => ['host' => ['192.168.1.1']],
    'sticky' => true, // after a write in this request, reads also hit the write host
    // ... driver, database, username, etc.
],
```

> `'sticky' => true` solves replication lag: after you write in a request, subsequent reads in that *same request* go to the write connection so you see your own write. Without it, a read replica that hasn't caught up could return stale data.

---

## Debugging & query logging

```php
// See the SQL a builder will run (with ? placeholders)
$sql = DB::table('users')->where('votes', '>', 100)->toSql();
// Output: select * from "users" where "votes" > ?

// See SQL with bindings substituted in (Laravel 10.15+) — for humans, not execution
$raw = DB::table('users')->where('votes', '>', 100)->toRawSql();
// Output: select * from "users" where "votes" > 100

// dump/die helpers on the builder
DB::table('users')->where('votes', '>', 100)->dd();   // dump SQL+bindings and die
DB::table('users')->where('votes', '>', 100)->dump(); // dump and continue

// Inspect the bindings array
DB::table('users')->where('votes', '>', 100)->getBindings(); // [100]
```

### Listening to every query

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Database\Events\QueryExecuted;

// In a service provider's boot() method:
DB::listen(function (QueryExecuted $query) {
    logger()->info($query->sql, [
        'bindings' => $query->bindings,
        'time_ms'  => $query->time,
        'connection' => $query->connectionName,
    ]);
});

// Or enable the built-in query log (dev only — it stores every query in memory)
DB::enableQueryLog();
DB::table('users')->get();
dump(DB::getQueryLog()); // [['query' => ..., 'bindings' => ..., 'time' => ...]]
DB::flushQueryLog();
```

### Catching slow queries automatically

```php
use Illuminate\Database\Connection;
use Illuminate\Database\Events\QueryExecuted;

// In a service provider's boot(): fire a callback when this request's
// cumulative query time crosses a threshold (in milliseconds).
DB::whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
    // Notify the dev team, log a warning, etc.
});
```

> **Laravel Telescope** is the heavyweight option: an installable dev dashboard (`composer require laravel/telescope --dev`) that records queries, requests, jobs, cache hits, mail, exceptions, and more, with a slow-query highlight. Great for spotting N+1 problems and slow queries in development; never enable it unguarded in production.

---

## ⚠️ Common Mistakes & Gotchas

**1. SQL injection via string interpolation in raw methods.**
Concatenating request data into `whereRaw`/`DB::raw` is the #1 security bug here.
```php
DB::table('u')->whereRaw("name = '".request('name')."'"); // 🔴 injectable
```
**Fix:** use bindings or the fluent API: `->whereRaw('name = ?', [request('name')])` or `->where('name', request('name'))`.

**2. `orWhere` without grouping breaks your logic.**
```php
DB::table('users')->where('active', true)->where('age', '>', 18)->orWhere('admin', true);
// = (active AND age>18) OR admin  → admins of any age/status slip through
```
**Fix:** wrap the OR group in a closure so it becomes `active AND age>18 AND (... OR ...)` or whatever you intended.

**3. `chunk` while modifying the filtered column.**
Using `chunk()` with `OFFSET` and updating a column you also filter on silently skips rows.
**Fix:** use `chunkById()` (or `lazyById`), which paginates by primary key instead of OFFSET.

**4. Wrapping an indexed column in a function kills the index.**
`whereDate('created_at', $d)` becomes `where date(created_at) = ?`, which can't use a normal index on `created_at`.
**Fix:** use a half-open range: `->whereBetween('created_at', [$start, $end])`.

**5. Expecting `truncate()` to roll back inside a transaction.**
In MySQL, `TRUNCATE` causes an implicit commit and can't be undone.
**Fix:** use `->delete()` (no where) when you need it to participate in a transaction.

**6. `upsert`/`insertOrIgnore` without the required unique index.**
On PostgreSQL/SQLite, `upsert`'s `uniqueBy` columns must back a unique or primary index, or it errors / doesn't dedupe.
**Fix:** add the unique index in a migration before relying on `upsert`.

**7. `count() > 0` instead of `exists()`.**
`count()` aggregates every matching row; `exists()` stops at the first.
**Fix:** `->exists()` for "is there at least one?".

---

## ✅ Best Practices

- **Prefer the fluent API; reach for raw only when necessary, and always with bindings.** Never interpolate user input into SQL strings.
- **Use the Query Builder for bulk and reporting work**, Eloquent where model behavior (events, casts, relations) matters.
- **Select only the columns you need.** `select('id', 'name')` beats `*` for wide tables and reduces memory and I/O.
- **Stream large datasets** with `chunkById`/`lazyById`/`cursor` — never `->get()` a huge table.
- **Wrap multi-write operations in `DB::transaction(..., attempts: 3)`** and keep the closure free of side effects (emails, HTTP calls) so retries are safe.
- **Use atomic `increment`/`decrement`** instead of read-modify-write in PHP for counters and balances.
- **Prefer `cursorPaginate` for huge or infinite-scroll lists** and `paginate` only when you genuinely need total counts / page numbers.
- **Add the right indexes** for your `where`, `join`, and `orderBy` columns; verify with `EXPLAIN` and `toSql()`/Telescope.
- **Enable `'sticky' => true`** on read/write-split connections to avoid replication-lag bugs.
- **Inspect generated SQL with `toSql()`/`toRawSql()`/`dd()`** before trusting a complex builder chain.

---

## 🎯 Interview Tips & Likely Questions

**Q1. When would you use the Query Builder over Eloquent?**
For set-based, performance-sensitive, or reporting work — bulk inserts/updates, aggregates over many rows, complex joins, or any time you want explicit control over the SQL and don't need model events, casts, or relationships. Eloquent's hydration and event overhead is wasteful for those. Eloquent itself sits on top of the Query Builder, so the query methods are shared.

**Q2. What's the difference between `WHERE` and `HAVING`?**
`WHERE` filters individual rows *before* grouping/aggregation and cannot reference aggregate functions. `HAVING` filters *groups* after aggregation and can use `SUM`, `COUNT`, etc. In the builder: `where()`/`whereRaw()` vs `having()`/`havingRaw()`.

**Q3. How does Laravel prevent SQL injection?**
By using **parameterized queries (bindings)**: the fluent methods collect values into a bindings array and send them to the PDO driver separately from the SQL text with `?` placeholders, so values are never parsed as SQL. The only way to bypass this is to concatenate input into `DB::raw`/`*Raw` strings yourself — which you must avoid.

**Q4. (Under the hood) How does `chunkById` differ from `chunk`, and why is it safer?**
`chunk` uses `LIMIT/OFFSET`. If rows you've already processed are deleted or move out of the filtered set, the OFFSET windows shift and later rows get skipped. `chunkById` instead tracks the maximum primary key seen and queries `WHERE id > ? ORDER BY id LIMIT n` for the next batch, so the window is anchored to actual IDs and stays correct even while you modify rows mid-iteration.

**Q5. (Under the hood) Why is `increment('votes')` better than `$row->votes + 1`?**
`increment` emits `UPDATE ... SET votes = votes + 1` executed atomically by the database, so concurrent requests can't clobber each other. Reading the value into PHP, adding one, and writing it back is a read-modify-write with a race condition — two concurrent requests can both read the same value and one increment is lost.

**Q6. Explain `paginate` vs `simplePaginate` vs `cursorPaginate`.**
`paginate` runs an extra `COUNT(*)` to know the total, enabling numbered pages and last-page links. `simplePaginate` skips the count (fetches `perPage + 1` rows to detect a next page) — cheaper, only next/prev. `cursorPaginate` is keyset pagination using `WHERE column > ?` instead of `OFFSET`; it's O(1) regardless of depth and stable under inserts, but requires ordering by a unique sequential column and offers only next/prev navigation.

**Q7. What does `DB::transaction`'s second argument do, and how are deadlocks handled?**
It's the number of attempts. If the closure throws a deadlock/serialization exception, Laravel rolls back and re-runs the whole closure up to that many times before giving up. Because the closure may run multiple times, it must be idempotent — no side effects like emails inside it. Manual `beginTransaction`/`commit`/`rollBack` gives you no automatic retry.

**Q8. How do you debug what SQL a query produces?**
`->toSql()` shows the SQL with `?` placeholders, `->getBindings()` shows the values, `->toRawSql()` shows the interpolated SQL (for reading, not running), and `->dd()`/`->dump()` dump both. Globally, `DB::listen()` or `DB::enableQueryLog()` capture every executed query; Telescope provides a UI for the same in development.

**Q9. What's the deep-pagination problem and how does cursor pagination solve it?**
With `OFFSET`, the database must scan and discard every row before the offset, so page 10,000 is far slower than page 1. Cursor pagination replaces `OFFSET` with a `WHERE id > last_seen_id` condition that an index satisfies in constant time, making deep pages just as fast as shallow ones.

**Q10. What's the difference between `delete()` and `truncate()`?**
`delete()` is a row-by-row, logged, transaction-safe `DELETE` that returns affected rows and fires no auto-increment reset. `truncate()` is a fast `TRUNCATE TABLE` that resets the auto-increment counter and, in MySQL, implicitly commits and can't be rolled back.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Raw DB facade (returns array of stdClass / affected count) ---
DB::select('select * from t where id = ?', [1]);   DB::scalar('select count(*) from t');
DB::insert('...', [...]); DB::update('...', [...]); DB::delete('...', [...]); DB::statement('...');

// --- Start & retrieve (fluent builder → Collection of stdClass) ---
DB::table('t')->get();                 first();  find($id);  value('col');
DB::table('t')->pluck('col', 'key');   count();  exists();   doesntExist();
DB::table('t')->select('a','b as c')->distinct();

// --- Where family ---
->where('c', '>', 5)  ->orWhere(...)   ->where(fn($q) => $q->...)   // grouped
->whereIn('id',[1,2]) ->whereNotIn(...) ->whereNull('c') ->whereNotNull('c')
->whereBetween('c',[1,9]) ->whereColumn('a','>','b') ->whereDate('c','2026-06-18')
->whereLike('c','%x%') ->whereAny(['a','b'],'like','%x%') ->whereExists(fn($q)=>...)

// --- Order / group / limit ---
->orderBy('c')->orderByDesc('c')->latest()->oldest()->inRandomOrder()->reorder()
->groupBy('c')->having('agg','>',1)->havingRaw('SUM(x) > ?',[1])
->limit(10)->offset(20)   // == ->take(10)->skip(20)

// --- Joins ---
->join('b','a.id','=','b.a_id') ->leftJoin(...) ->rightJoin(...) ->crossJoin('b')
->join('b', fn($j)=>$j->on('a.id','=','b.a_id')->where('b.flag',true))
->joinSub($sub,'alias', fn($j)=>$j->on(...))

// --- Aggregates ---
->count() ->sum('c') ->avg('c') ->min('c') ->max('c') ->exists()

// --- Write ---
->insert([...]); ->insertGetId([...]); ->insertOrIgnore([...]);
->upsert($rows, ['uniq_col'], ['update_col']);
->update([...]); ->increment('c', 1, ['other'=>x]); ->updateOrInsert([find],[set]);
->delete(); ->truncate();

// --- Raw (always bind!) ---
->selectRaw('c * ? as x', [1.1]) ->whereRaw('c > ?', [5]) ->havingRaw('SUM(c) > ?', [1])
DB::raw('count(*) as n')   // NO bindings — never put user input here

// --- Big data ---
->chunkById(1000, fn($rows)=>...) ->lazyById(1000) ->cursor()->each(...)

// --- Pagination ---
->paginate(15) ->simplePaginate(15) ->cursorPaginate(15)

// --- Transactions ---
DB::transaction(fn()=>..., attempts: 3);
DB::beginTransaction(); DB::commit(); DB::rollBack();

// --- Connections & debug ---
DB::connection('reporting')->table('t')->get();
->toSql(); ->toRawSql(); ->getBindings(); ->dd(); ->dump();
DB::listen(fn($q)=>...); DB::enableQueryLog(); DB::getQueryLog();
DB::whenQueryingForLongerThan(500, fn($conn,$event)=>...); // slow-query alert
```

---

## 🧪 Mini Exercises

1. **Reporting query.** Using only the Query Builder, write a query that returns each `user_id` with their total order revenue and order count, but only for users whose total revenue exceeds 5,000, ordered by revenue descending. Use `select`, `DB::raw`/`selectRaw`, `groupBy`, and `having`.

2. **Safe raw filter.** A controller receives `request('term')` and `request('min_price')`. Build a query against an `orders` table that matches rows where `notes` is `LIKE` the term *and* `price` is above `min_price`, using `whereRaw` with **bindings** (no string interpolation). Then print the generated SQL with `toSql()` and the bindings with `getBindings()`.

3. **Backfill safely.** Write a routine that sets `status = 'archived'` on every `posts` row older than one year, processing the table in batches of 500 in a way that won't skip rows even though you're updating the `status` you might filter on. (Hint: which chunk method?)

4. **Atomic transfer.** Implement a money transfer between two `accounts` rows inside a transaction that retries up to 3 times on deadlock, decrements the sender and increments the receiver atomically, and throws if the sender's balance would go negative.

5. **Pagination choice.** You're building an infinite-scroll activity feed over a 50-million-row `events` table ordered by `id`. Write the query that paginates it efficiently, and in a one-line comment explain why this method beats `paginate()` here.
