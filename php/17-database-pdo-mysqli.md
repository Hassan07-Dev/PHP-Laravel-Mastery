# Databases with PDO & MySQLi

Talking to a relational database is the single most common thing a backend application does. In this module you'll learn how raw PHP connects to MySQL/MariaDB, how to do it *safely* (no SQL injection), and why almost every serious project standardizes on **PDO**. We finish by showing how Laravel wraps all of this so you understand what's happening beneath Eloquent.

> Targets: **PHP 8.4** (notes for 8.1–8.3 where relevant) and **Laravel 12** (notes for Laravel 10/11 where behavior differs).

---

**What you'll learn**

- Relational database fundamentals you need before writing a query
- How to connect with **PDO** (DSN, options, error mode) and why PDO is preferred over **mysqli**
- **Prepared statements** and **parameter binding** (named & positional) to stop SQL injection cold
- Fetch modes (`assoc`, `object`, into a class), plus `fetchAll`, `rowCount`, and `lastInsertId`
- **Transactions**: `beginTransaction`, `commit`, `rollBack`, and when they matter
- The subtle difference between `bindValue` and `bindParam`, and **emulated vs native** prepares
- A full CRUD example, `LIKE` with parameters, and a dynamic `IN (...)` clause
- Connection best practices, persistent connections, and how Laravel abstracts all of this

---

## 1. Relational databases in 90 seconds

A **relational database** stores data in **tables** (think spreadsheets). Each table has **columns** (typed fields like `id`, `email`) and **rows** (individual records). Tables relate to each other via **keys**:

- **Primary key (PK)**: a column (usually `id`) that uniquely identifies each row.
- **Foreign key (FK)**: a column that points to a PK in another table, e.g. `posts.user_id` references `users.id`.

You manipulate data with **SQL** (Structured Query Language). The four core operations are called **CRUD**: **C**reate (`INSERT`), **R**ead (`SELECT`), **U**pdate (`UPDATE`), **D**elete (`DELETE`).

```sql
-- The schema we'll use throughout this module
CREATE TABLE users (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    age        INT UNSIGNED NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Jargon:** a **database driver** is the C extension that lets PHP speak a particular database's wire protocol (e.g. the `pdo_mysql` or `mysqli` extension for MySQL). A **DBMS** (Database Management System) is the server itself — MySQL, MariaDB, PostgreSQL, SQLite, etc.

---

## 2. The two native APIs: PDO vs MySQLi

PHP ships two modern ways to talk to MySQL:

| | **PDO** (`PDO`) | **MySQLi** (`mysqli`) |
|---|---|---|
| Database support | 12+ drivers (MySQL, PostgreSQL, SQLite, SQL Server…) | MySQL/MariaDB **only** |
| API style | OOP only | OOP **and** procedural |
| Named placeholders (`:name`) | ✅ Yes | ❌ No (positional `?` only) |
| Prepared statements | ✅ | ✅ |
| Stored procedures | ✅ | ✅ |
| Maturity / ecosystem | Used by Laravel, Symfony, Doctrine | Common in legacy/WordPress code |

**Why PDO over mysqli?**

1. **Driver-agnostic.** Switch from MySQL to PostgreSQL by changing the **DSN** string, not your code. Vendor lock-in is reduced.
2. **Named placeholders** (`:email`) make complex queries readable and self-documenting.
3. **Consistent OOP API** that frameworks build on. Laravel's database layer is built on PDO.

`mysqli` is faster for a handful of MySQL-specific features and is fine for MySQL-only apps, but **for new code, default to PDO.** We cover PDO in depth and show mysqli briefly so you recognize it in interviews and legacy code.

---

## 3. Connecting with PDO

### 3.1 The DSN

A **DSN** (Data Source Name) is a string that tells PDO *which driver, which host, which database, and which charset* to use. Format: `driver:key=value;key=value`.

```php
<?php
$host    = '127.0.0.1';
$db      = 'app';
$user    = 'root';
$pass    = 'secret';
$port    = 3306;
$charset = 'utf8mb4'; // full Unicode incl. emoji; NEVER use plain 'utf8' in MySQL

$dsn = "mysql:host=$host;port=$port;dbname=$db;charset=$charset";
```

> **Why `utf8mb4`?** MySQL's legacy `utf8` is a broken 3-byte encoding that can't store 4-byte characters (emoji, some CJK). `utf8mb4` is real UTF-8. Always use it.

### 3.2 Connection options (the critical part)

The constructor's 4th argument is an **options array**. Three options should be set on *every* production connection:

```php
<?php
$options = [
    // 1. Throw exceptions on errors instead of silently returning false.
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,

    // 2. Return associative arrays by default (no duplicated numeric keys).
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,

    // 3. Use REAL prepared statements, not PHP-side emulation (see §9).
    PDO::ATTR_EMULATE_PREPARES   => false,
];

try {
    $pdo = new PDO($dsn, $user, $pass, $options);
} catch (PDOException $e) {
    // Don't leak credentials/host to the user. Log the real message.
    error_log($e->getMessage());
    throw new RuntimeException('Database connection failed.', (int) $e->getCode(), $e);
}
```

**Why `ERRMODE_EXCEPTION` matters:** by default (`ERRMODE_SILENT`), a failed query just returns `false` and you must check return values everywhere. With exceptions, a failed query *throws*, so you can't accidentally ignore an error.

> **PHP 8.1+ default:** Since **PHP 8.1**, PDO's default error mode is already `ERRMODE_EXCEPTION` (it was `ERRMODE_SILENT` in 8.0 and earlier). Set it explicitly anyway — it documents intent and is portable across versions.

### 3.3 PHP 8.4: driver-specific subclasses (`PDO::connect()`)

PHP 8.4 added **driver-specific PDO subclasses** (`Pdo\Mysql`, `Pdo\Sqlite`, `Pdo\Pgsql`, …) and a static factory, `PDO::connect()`, that returns the right subclass for the DSN. The classic `new PDO(...)` constructor still works and is fully backward-compatible; the subclasses just expose driver-specific helpers in a type-safe place.

```php
<?php
// PHP 8.4+: returns a Pdo\Mysql instance (a PDO subclass), not a generic PDO.
$pdo = PDO::connect($dsn, $user, $pass, $options);

if ($pdo instanceof Pdo\Mysql) {
    echo $pdo->getWarningCount(); // MySQL-specific helper, only on the subclass
}
```

> **Looking ahead (PHP 8.5):** accessing *driver-specific constants* via the base `PDO` class (e.g. `PDO::MYSQL_ATTR_SSL_CA`, `PDO::ATTR_TIMEOUT` is generic and stays) is **deprecated** in 8.5 in favor of the subclass constants (`Pdo\Mysql::ATTR_SSL_CA`). Plain `new PDO(...)` and the generic attributes used in this module are unaffected.

---

## 4. Prepared statements & parameter binding (the heart of safety)

### 4.1 Why: SQL injection

**Never** build queries by concatenating user input:

```php
<?php
// ❌ CATASTROPHIC. Do not do this. Ever.
$email = $_GET['email']; // e.g. "x' OR '1'='1"
$pdo->query("SELECT * FROM users WHERE email = '$email'");
// Resulting SQL: SELECT * FROM users WHERE email = 'x' OR '1'='1'
// -> returns EVERY user. This is SQL injection.
```

A **prepared statement** sends the query *template* and the *data* to the database **separately**. The database compiles the template once, then treats your data strictly as values — it can never become executable SQL. This is the correct, complete defense against SQL injection.

### 4.2 How: `prepare()` + `execute()`

**Named placeholders** (`:name`) — recommended for readability:

```php
<?php
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND age > :age');
$stmt->execute([':email' => 'ada@example.com', ':age' => 18]);
$user = $stmt->fetch();
// The leading colon in the keys is optional: ['email' => ...] also works.
```

**Positional placeholders** (`?`) — order matters, no names:

```php
<?php
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ? AND age > ?');
$stmt->execute(['ada@example.com', 18]);
$user = $stmt->fetch();
```

> **Rule:** placeholders stand in for **values only** — never for table names, column names, or SQL keywords (`ASC`/`DESC`). Those must be validated against an allow-list and inlined.

### 4.3 `bindValue` vs `bindParam`

Instead of passing an array to `execute()`, you can bind explicitly. There are two methods, and the difference trips people up.

```php
<?php
$stmt = $pdo->prepare('SELECT * FROM users WHERE age > :age LIMIT :lim');

// bindValue: binds the VALUE at the moment of the call (copy by value).
$stmt->bindValue(':age', 18, PDO::PARAM_INT);

// LIMIT must be a real integer; with emulation off, type matters.
$limit = 10;
$stmt->bindValue(':lim', $limit, PDO::PARAM_INT);
$stmt->execute();
```

`bindParam` binds **by reference** — it remembers the *variable*, and reads its value only when `execute()` runs. This matters in loops:

```php
<?php
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (:name, :email)');
$stmt->bindParam(':name', $name);   // bind the variables ONCE
$stmt->bindParam(':email', $email);

foreach ([['Ada', 'ada@x.com'], ['Alan', 'alan@x.com']] as [$name, $email]) {
    $stmt->execute(); // reads CURRENT $name/$email each iteration
}
// Output: two rows inserted, "Ada" and "Alan".
```

If you'd used `bindValue` here, both inserts would use the *first* loop's values. Conversely, you **cannot** `bindParam` a literal: `bindParam(':age', 18)` is a fatal error because you can't reference a constant.

| | `bindValue` | `bindParam` |
|---|---|---|
| Binds | the value, immediately | a variable, by reference |
| Read | now | at `execute()` time |
| Accepts literals | ✅ | ❌ (needs a variable) |
| Use when | one-shot queries | re-executing in a loop |

> **The `PARAM_INT` gotcha with `LIMIT`/`OFFSET`:** When `ATTR_EMULATE_PREPARES => false`, MySQL is strict — a `LIMIT` bound as a string can error. Always pass `PDO::PARAM_INT` for integer placeholders in `LIMIT`/`OFFSET`. (When emulation is *on*, PDO quotes things itself and is more forgiving, which is one reason people leave it on — but native prepares are safer overall.)

---

## 5. Fetch modes

After a `SELECT`, you retrieve rows. `fetch()` returns the **next** row (or `false` when done); `fetchAll()` returns **all** rows as an array.

```php
<?php
$stmt = $pdo->query('SELECT id, name, email FROM users LIMIT 1');

// 1. Associative array (column => value) — set as our default in §3.2
$row = $stmt->fetch(PDO::FETCH_ASSOC);
// ['id' => 1, 'name' => 'Ada', 'email' => 'ada@x.com']

// 2. Anonymous stdClass object — access with ->
$stmt = $pdo->query('SELECT id, name FROM users LIMIT 1');
$obj = $stmt->fetch(PDO::FETCH_OBJ);
echo $obj->name; // Ada

// 3. Numeric array (column index => value)
$stmt = $pdo->query('SELECT id, name FROM users LIMIT 1');
$num = $stmt->fetch(PDO::FETCH_NUM); // [0 => 1, 1 => 'Ada']
```

### 5.1 Fetching into a class

`FETCH_CLASS` hydrates each row into an instance of a class, matching columns to **public** properties by name.

```php
<?php
class User
{
    public int $id;
    public string $name;
    public string $email;

    public function greet(): string
    {
        return "Hi, I'm {$this->name}";
    }
}

$stmt = $pdo->query('SELECT id, name, email FROM users');
$users = $stmt->fetchAll(PDO::FETCH_CLASS, User::class);
echo $users[0]->greet(); // Hi, I'm Ada
```

> **Property order vs constructor:** With plain `FETCH_CLASS`, PDO assigns the columns to properties **first**, then calls the constructor with any args you passed (the optional 3rd arg of `fetchAll`). Add `PDO::FETCH_PROPS_LATE` to reverse this — `PDO::FETCH_CLASS | PDO::FETCH_PROPS_LATE` runs the constructor first, then overwrites with the column values. This matters when the constructor sets defaults you don't want clobbered.

> **Constructor-promoted properties are NOT populated by `FETCH_CLASS`.** Promoted params (`public function __construct(public int $id) {}`) are *constructor parameters*, so PDO won't write them as plain properties. If your model uses constructor promotion, either fetch with `FETCH_ASSOC` and build the object yourself, or map each row through a callback with `fetchAll(PDO::FETCH_FUNC, fn (...$cols) => new User(...$cols))`. `FETCH_FUNC` is still supported in PHP 8.4.

### 5.2 `fetchAll`, `rowCount`, `lastInsertId`

```php
<?php
// fetchAll — all rows at once (careful with huge result sets; loads into memory)
$all = $pdo->query('SELECT * FROM users')->fetchAll(); // array of assoc rows

// rowCount — number of rows AFFECTED by the last INSERT/UPDATE/DELETE
$stmt = $pdo->prepare('DELETE FROM users WHERE age < :age');
$stmt->execute([':age' => 13]);
echo $stmt->rowCount(); // e.g. 3 rows deleted

// lastInsertId — the auto-increment id of the most recent INSERT on this connection
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
$stmt->execute(['Grace', 'grace@x.com']);
echo $pdo->lastInsertId(); // e.g. "42" (a string)
```

> **`rowCount()` on SELECT is unreliable** for portability — many drivers don't return the SELECT row count. To count selected rows, either `count(fetchAll())` or run a `SELECT COUNT(*)`. `rowCount()` is meant for write operations.

---

## 6. Full CRUD example

A small repository class tying it together with named parameters and exception handling.

```php
<?php
declare(strict_types=1);

final class UserRepository
{
    public function __construct(private readonly PDO $pdo) {}

    // CREATE
    public function create(string $name, string $email, ?int $age = null): int
    {
        $sql = 'INSERT INTO users (name, email, age) VALUES (:name, :email, :age)';
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute([
            ':name'  => $name,
            ':email' => $email,
            ':age'   => $age, // null binds as SQL NULL
        ]);
        return (int) $this->pdo->lastInsertId();
    }

    // READ (one)
    public function find(int $id): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = :id');
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch();
        return $row === false ? null : $row;
    }

    // READ (many)
    public function all(): array
    {
        return $this->pdo->query('SELECT * FROM users ORDER BY id')->fetchAll();
    }

    // UPDATE
    public function updateEmail(int $id, string $email): bool
    {
        $stmt = $this->pdo->prepare('UPDATE users SET email = :email WHERE id = :id');
        $stmt->execute([':email' => $email, ':id' => $id]);
        return $stmt->rowCount() > 0; // true if a row actually changed
    }

    // DELETE
    public function delete(int $id): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM users WHERE id = :id');
        $stmt->execute([':id' => $id]);
        return $stmt->rowCount() > 0;
    }
}

// Usage:
$repo = new UserRepository($pdo);
$id   = $repo->create('Ada Lovelace', 'ada@x.com', 36);
$user = $repo->find($id);
$repo->updateEmail($id, 'ada.lovelace@x.com');
$repo->delete($id);
```

---

## 7. `LIKE` with parameters

A common mistake is putting `%` wildcards *inside* the placeholder. They must go in the **value**, not the SQL template.

```php
<?php
// ❌ Wrong — '%:term%' is a literal string, the placeholder is never seen.
$stmt = $pdo->prepare("SELECT * FROM users WHERE name LIKE '%:term%'");

// ✅ Correct — wildcards live in the bound value.
$term = 'ada';
$stmt = $pdo->prepare('SELECT * FROM users WHERE name LIKE :term');
$stmt->execute([':term' => '%' . $term . '%']);
$matches = $stmt->fetchAll();
```

> **Escape user wildcards** if `%` or `_` typed by the user should be literal:
> `$safe = str_replace(['%', '_'], ['\%', '\_'], $term);` then `LIKE :term ESCAPE '\\'`.

---

## 8. Dynamic `IN (...)` clause

You can't bind an array to a single placeholder. Build one placeholder per element.

```php
<?php
$ids = [3, 7, 11, 15];

// Make "?, ?, ?, ?" with the right count.
$placeholders = implode(',', array_fill(0, count($ids), '?'));
$sql = "SELECT * FROM users WHERE id IN ($placeholders)";

$stmt = $pdo->prepare($sql);
$stmt->execute($ids); // pass the array as positional params
$rows = $stmt->fetchAll();
// SQL sent: SELECT * FROM users WHERE id IN (?,?,?,?)
```

With **named** placeholders, generate unique names:

```php
<?php
$ids = [3, 7, 11];
$params = [];
$names  = [];
foreach ($ids as $i => $id) {
    $key = ":id$i";
    $names[]      = $key;
    $params[$key] = $id;
}
$sql = 'SELECT * FROM users WHERE id IN (' . implode(',', $names) . ')';
$stmt = $pdo->prepare($sql);
$stmt->execute($params);
```

> **Guard the empty case:** `IN ()` is a syntax error. If `$ids` is empty, short-circuit and return `[]` instead of running the query.

---

## 9. Emulated vs native prepares

This is a favorite "under the hood" interview topic.

- **Native (real) prepared statements** (`ATTR_EMULATE_PREPARES => false`): PHP sends the SQL template to MySQL once via the binary protocol. MySQL parses and plans it. Then values are sent separately and bound server-side. The template and data **never mix** in text form.

- **Emulated prepares** (`ATTR_EMULATE_PREPARES => true`, the **default**): PDO does the substitution **itself in PHP**, properly quoting/escaping each value, then sends one finished SQL string to MySQL. It's still injection-safe (PDO escapes correctly), but there's no server-side prepared statement.

```php
<?php
// Inspect / set the mode:
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false); // native
```

| | Native prepares | Emulated prepares |
|---|---|---|
| Who substitutes values | MySQL server | PDO in PHP |
| Round trips | 2 (prepare, then execute) | 1 (one finished string) |
| Reuse benefit | Faster if executed many times | None server-side |
| Type strictness | Strict (mind `PARAM_INT` for `LIMIT`) | Lenient (PDO quotes) |
| Multiple statements in one `query` | Blocked | Possible (a risk) |
| Recommendation | ✅ Default for safety/correctness | Only for niche compatibility |

> **Why turn emulation OFF?** Native prepares give true server-side parameterization, better type safety, and protection against certain edge-case injection vectors involving multi-statement queries. The tiny extra round-trip is negligible for most apps. Set `ATTR_EMULATE_PREPARES => false`.

---

## 10. Transactions

A **transaction** groups multiple statements so they either **all** succeed or **all** fail — the **atomicity** guarantee (the "A" in ACID). Classic example: transferring money — you must not debit one account without crediting the other.

```php
<?php
try {
    $pdo->beginTransaction(); // turns OFF autocommit

    $debit  = $pdo->prepare('UPDATE accounts SET balance = balance - :amt WHERE id = :id');
    $debit->execute([':amt' => 100, ':id' => 1]);

    $credit = $pdo->prepare('UPDATE accounts SET balance = balance + :amt WHERE id = :id');
    $credit->execute([':amt' => 100, ':id' => 2]);

    $pdo->commit(); // make all changes permanent
} catch (Throwable $e) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack(); // undo everything since beginTransaction()
    }
    throw $e;
}
```

**When to use transactions:**

- Multiple writes that must be consistent together (the transfer above).
- Insert into a parent + child tables (order + order items).
- "Read then conditionally write" where another process could interfere (combine with `SELECT ... FOR UPDATE`).

> **Gotchas:** (1) Check `inTransaction()` before `rollBack()` to avoid an exception if no transaction is active. (2) In MySQL, **DDL** statements (`CREATE TABLE`, `ALTER`, `DROP`) cause an **implicit commit** — they can't be rolled back. (3) Transactions only work on transactional engines like **InnoDB**, not **MyISAM**.

---

## 11. PDOException handling

Every PDO error (when `ERRMODE_EXCEPTION` is set) throws a `PDOException`. It carries a SQLSTATE code you can branch on.

```php
<?php
try {
    $stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
    $stmt->execute(['Ada', 'ada@x.com']);
} catch (PDOException $e) {
    // SQLSTATE 23000 = integrity constraint violation (e.g. duplicate UNIQUE email)
    if ($e->getCode() === '23000') {
        throw new RuntimeException('That email is already registered.', 0, $e);
    }
    error_log($e->getMessage()); // log full detail server-side
    throw new RuntimeException('A database error occurred.', 0, $e); // generic to user
}
```

> **Security:** never echo `$e->getMessage()` to end users — it can reveal table names, column names, and query structure. Log it; show a generic message. `getCode()` returns the **SQLSTATE string** (e.g. `'23000'`), while `$e->errorInfo[1]` holds the **driver-specific** code (e.g. `1062` for MySQL duplicate entry).

---

## 12. MySQLi, briefly

So you recognize it. Equivalent "insert one user" with mysqli's OOP API:

```php
<?php
$mysqli = new mysqli('127.0.0.1', 'root', 'secret', 'app', 3306);
$mysqli->set_charset('utf8mb4');
// PHP 8.1+: mysqli throws exceptions by default (mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT))

$stmt = $mysqli->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
// "ss" = two string params; mysqli requires you to declare types up front.
// Like PDO's bindParam, bind_param binds BY REFERENCE — values are read at
// execute() time, so assigning $name/$email after the bind is fine.
$stmt->bind_param('ss', $name, $email);
$name  = 'Grace';
$email = 'grace@x.com';
$stmt->execute();

echo $mysqli->insert_id; // last auto-increment id
$stmt->close();
$mysqli->close();
```

Key differences from PDO: positional `?` only (no names), explicit type-string in `bind_param` (`i`, `d`, `s`, `b`), and it's MySQL-only. For new code, prefer PDO.

> **PHP 8.1+ note:** `mysqli` now reports errors as exceptions by default (`MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT`). Before 8.1 it returned `false` silently like old PDO.

---

## 13. Connection best practices & persistent connections

```php
<?php
// One connection per request, injected where needed (don't reconnect per query).
// Centralize config; never hardcode credentials — read from environment.
$pdo = new PDO($dsn, getenv('DB_USER'), getenv('DB_PASS'), $options);
```

**Persistent connections** (`PDO::ATTR_PERSISTENT => true`) keep the TCP connection open across requests in the connection pool, skipping the connect handshake.

```php
<?php
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_PERSISTENT => true,
    PDO::ATTR_ERRMODE    => PDO::ERRMODE_EXCEPTION,
]);
```

> **Use with caution.** Persistent connections can leak state across requests (lingering transactions, temp tables, session vars) and exhaust `max_connections` under load. For most apps, plain connections are fine; reach for persistent only after measuring a real connect-cost bottleneck.

**Other best practices:** set `utf8mb4`; keep secrets in env/secret manager; use one connection per request lifecycle; close large statements with `closeCursor()` if reusing; avoid `fetchAll()` on unbounded result sets (stream with `fetch()` in a loop, or paginate).

---

## 14. Frameworks wrap all of this (Laravel)

You will rarely write raw PDO in a Laravel app — but Laravel **is built on PDO**. The query builder and Eloquent generate parameterized PDO statements for you, so injection protection is automatic.

```php
<?php
use Illuminate\Support\Facades\DB;

// Query builder — bindings are parameterized PDO under the hood.
$users = DB::table('users')->where('age', '>', 18)->get();

// Raw with bindings (still safe — uses PDO placeholders):
$users = DB::select('SELECT * FROM users WHERE age > ?', [18]);

// Transactions — Laravel wraps begin/commit/rollBack and retries on deadlock:
DB::transaction(function () {
    DB::table('accounts')->where('id', 1)->decrement('balance', 100);
    DB::table('accounts')->where('id', 2)->increment('balance', 100);
}); // auto-commits on success, auto-rollBack on exception

// Reach the raw PDO instance if you ever need it:
$pdo = DB::connection()->getPdo();
```

```php
<?php
// Eloquent — even higher level, still parameterized PDO beneath.
$user = App\Models\User::where('email', $email)->first();
```

> **Laravel 12 / 11 / 10 notes:** Database config lives in `config/database.php`, driven by `.env`. Laravel 11 introduced a slimmer skeleton but the DB layer behaves the same; Laravel 12 continues this. Importantly, Laravel's base `Connector` class **already sets `PDO::ATTR_EMULATE_PREPARES => false`** (along with `ATTR_ERRMODE => ERRMODE_EXCEPTION`, `ATTR_STRINGIFY_FETCHES => false`, `ATTR_CASE => CASE_NATURAL`, and `ATTR_ORACLE_NULLS => NULL_NATURAL`) — so Laravel gives you native prepares by default. You can still override any of these per connection via the `'options'` key in `config/database.php`. `DB::transaction()` accepts a second arg for **deadlock retry attempts**, e.g. `DB::transaction($cb, attempts: 3)`.

```php
<?php
// config/database.php — overriding connection options in Laravel.
// (Native prepares are ALREADY the default; this shows how to set extra
//  options, e.g. a longer timeout, or to re-assert emulation off explicitly.)
'mysql' => [
    // ...
    'options' => extension_loaded('pdo_mysql')
        ? array_filter([
            PDO::ATTR_EMULATE_PREPARES => false,
            PDO::ATTR_TIMEOUT          => 5,
        ])
        : [],
],
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Concatenating user input into SQL.** This is SQL injection.
   **Fix:** always use prepared statements with bound parameters. Placeholders are for *values* only — validate column/table names against an allow-list.

2. **Putting `%` inside the placeholder for `LIKE`** (`LIKE '%:term%'`).
   The placeholder is never recognized inside a quoted string.
   **Fix:** keep `LIKE :term` and bind `'%' . $term . '%'` as the value.

3. **Binding `LIMIT`/`OFFSET` as a string with native prepares.** MySQL errors because `LIMIT` needs an integer.
   **Fix:** `bindValue(':lim', $limit, PDO::PARAM_INT)` (and validate it's a non-negative int).

4. **Calling `rollBack()` when no transaction is active.** Throws "There is no active transaction".
   **Fix:** guard with `if ($pdo->inTransaction()) { $pdo->rollBack(); }`.

5. **Relying on `rowCount()` after a `SELECT`.** It's not portable and may return 0.
   **Fix:** use `count($stmt->fetchAll())` or a `SELECT COUNT(*)` query.

6. **Confusing `bindParam` with literals.** `bindParam(':age', 18)` fails — it needs a variable reference.
   **Fix:** use `bindValue` for literals; reserve `bindParam` for variables you re-execute in a loop.

7. **Leaking exception messages to users.** Reveals schema and aids attackers.
   **Fix:** log `$e->getMessage()`; show a generic message.

8. **Using legacy `utf8` charset.** Can't store emoji/4-byte chars.
   **Fix:** use `utf8mb4` in the DSN and table collation.

9. **Comparing `PDOException::getCode()` against an integer.** `getCode()` returns the **string** SQLSTATE (e.g. `"23000"`), so `=== 23000` (int) is always false.
   **Fix:** compare against the string (`=== '23000'`), or use the driver code in `$e->errorInfo[1]` (an int like `1062`).

10. **Assuming Laravel still emulates prepares.** Laravel's `Connector` sets `ATTR_EMULATE_PREPARES => false` by default — you get native prepares without doing anything.
    **Fix:** don't add config to "turn it on"; only override `'options'` when you genuinely need different behavior.

---

## ✅ Best Practices

- Set `ATTR_ERRMODE => ERRMODE_EXCEPTION`, `ATTR_DEFAULT_FETCH_MODE => FETCH_ASSOC`, and `ATTR_EMULATE_PREPARES => false` on every connection.
- **Always** parameterize. No exceptions for "trusted" input.
- Use **named placeholders** for readability in non-trivial queries.
- Keep credentials in environment variables / secret managers, never in code.
- Use `utf8mb4` everywhere (DSN, table, column collation).
- Wrap multi-step writes in **transactions**; guard `rollBack()` with `inTransaction()`.
- Stream large result sets with `fetch()` in a loop or paginate — don't `fetchAll()` millions of rows.
- Validate dynamic identifiers (column/sort/direction) against an **allow-list**.
- Inject the `PDO` instance (DI) rather than creating connections ad hoc.
- In Laravel, prefer the query builder/Eloquent; drop to `DB::select` with bindings only when needed. Note Laravel already ships native prepares and exception error mode — don't assume you must turn them on.
- On PHP 8.4+, `PDO::connect()` returns a driver-specific subclass (`Pdo\Mysql`, …); plain `new PDO(...)` is still perfectly fine and the most portable choice.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Why prefer PDO over mysqli?**
PDO is **driver-agnostic** (12+ databases via one API), supports **named placeholders**, and has a consistent OOP interface that frameworks like Laravel build on. mysqli is MySQL-only and positional-only. For new code, PDO is the default.

**Q2. How do prepared statements prevent SQL injection — under the hood?**
The query *template* and the *data* travel to the database **separately**. With **native** prepares, the server parses and plans the template first, then binds values server-side; user data is never interpreted as SQL. With **emulated** prepares, PDO escapes/quotes values itself in PHP before sending one finished string — still safe, but no server-side statement.

**Q3. What's the difference between emulated and native prepared statements?**
Native: server-side parameterization, two round trips, strict typing, blocks multi-statements. Emulated (PDO's default): PDO substitutes in PHP, one round trip, lenient. Turn emulation **off** for true parameterization and type safety.

**Q4. `bindValue` vs `bindParam`?**
`bindValue` binds the value immediately (by copy) and accepts literals. `bindParam` binds a **variable by reference** and reads it at `execute()` time — ideal for re-executing a statement in a loop with changing variables; it cannot bind a literal.

**Q5. How do you safely build an `IN (...)` clause from an array?**
You can't bind an array to one placeholder. Generate one placeholder per element (`implode(',', array_fill(0, count($ids), '?'))`), then pass the array to `execute()`. Guard against an empty array (`IN ()` is a syntax error).

**Q6. What does `lastInsertId()` return, and what's a caveat?**
The auto-increment id of the most recent INSERT **on that connection**, as a **string**. Caveat: it's per-connection, so concurrent requests don't interfere — but with persistent connections or bulk inserts, understand it returns the *first* id of a multi-row insert in MySQL.

**Q7. When do you need a transaction?**
When multiple statements must succeed or fail together (atomicity): money transfers, order + order-items, any "all-or-nothing" write. Remember DDL implicitly commits and MyISAM ignores transactions.

**Q8. How does Laravel relate to PDO?**
Laravel's database layer is built **on PDO**. The query builder and Eloquent produce parameterized PDO statements; `DB::transaction()` wraps begin/commit/rollBack with deadlock retries. Laravel's `Connector` sets sensible PDO defaults for you — including `ATTR_ERRMODE => ERRMODE_EXCEPTION` and `ATTR_EMULATE_PREPARES => false` (native prepares). You can fetch the raw instance via `DB::connection()->getPdo()`.

**Q9. Why set the charset to `utf8mb4`?**
MySQL's legacy `utf8` is a 3-byte subset that can't store 4-byte characters (emoji, some scripts). `utf8mb4` is full UTF-8.

**Q10. How should you handle a `PDOException` from a duplicate key?**
Catch `PDOException`, check `getCode() === '23000'` (SQLSTATE integrity violation) or `errorInfo[1] === 1062` (MySQL duplicate). Log the detail server-side, return a friendly message — never expose the raw message.

---

## 📋 Quick Reference / Cheat Sheet

```php
<?php
// CONNECT
$dsn = 'mysql:host=127.0.0.1;port=3306;dbname=app;charset=utf8mb4';
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
]);

// PREPARE + EXECUTE (named)
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute([':email' => $email]);

// PREPARE + EXECUTE (positional)
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ?');
$stmt->execute([$email]);

// BIND
$stmt->bindValue(':lim', 10, PDO::PARAM_INT); // literal, by value
$stmt->bindParam(':name', $name);             // variable, by reference

// FETCH
$row  = $stmt->fetch();                         // next row (default FETCH_ASSOC)
$row  = $stmt->fetch(PDO::FETCH_OBJ);           // stdClass
$all  = $stmt->fetchAll();                      // all rows
$objs = $stmt->fetchAll(PDO::FETCH_CLASS, User::class);

// WRITE RESULTS
$pdo->lastInsertId();   // last auto-increment id (string)
$stmt->rowCount();      // rows affected by INSERT/UPDATE/DELETE

// LIKE
$stmt = $pdo->prepare('SELECT * FROM users WHERE name LIKE :t');
$stmt->execute([':t' => "%$term%"]);

// IN (...)
$ph  = implode(',', array_fill(0, count($ids), '?'));
$pdo->prepare("SELECT * FROM users WHERE id IN ($ph)")->execute($ids);

// TRANSACTION
$pdo->beginTransaction();
try { /* writes */ $pdo->commit(); }
catch (Throwable $e) { if ($pdo->inTransaction()) $pdo->rollBack(); throw $e; }

// LARAVEL
DB::select('SELECT * FROM users WHERE age > ?', [18]);
DB::transaction(fn () => /* ... */, attempts: 3);
DB::connection()->getPdo();
```

**SQLSTATE quick hits:** `23000` = integrity violation (duplicate/foreign key); `42S02` = table not found; `42000` = syntax error. MySQL native: `1062` duplicate entry, `1452` FK constraint fails.

---

## 🧪 Mini Exercises

1. **Safe search endpoint.** Write a function `searchUsers(PDO $pdo, string $term): array` that returns users whose `name` *or* `email` matches the term using `LIKE`, parameterized, with wildcards in the bound value. Escape user-supplied `%`/`_`.

2. **Bulk insert in a transaction.** Given an array of `[name, email]` pairs, insert them all inside a single transaction using one prepared statement and `bindParam` in a loop. Roll back the whole batch if any insert fails (e.g. duplicate email) and report how many would have been inserted.

3. **Dynamic `IN` with safety.** Write `findByIds(PDO $pdo, array $ids): array` that builds the placeholder list dynamically, casts each id to int, returns `[]` for an empty array, and binds every id as `PDO::PARAM_INT`.

4. **Paginator.** Write `page(PDO $pdo, int $limit, int $offset): array` that returns a page of users ordered by `id`, binding `LIMIT`/`OFFSET` as integers, and rejects negative inputs.

5. **Emulation experiment (conceptual).** Explain, in writing, what bytes go over the wire for the same parameterized query with `ATTR_EMULATE_PREPARES` set to `true` vs `false`, and why the native mode is generally preferred.
