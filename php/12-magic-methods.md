# Magic Methods in PHP

Magic methods are PHP's hooks into the runtime: special methods whose names start with two underscores (`__`) that the engine calls *automatically* in response to certain events — constructing an object, reading an undefined property, calling a missing method, converting an object to a string, serializing it, and more. They are the machinery behind Laravel's Eloquent (`$user->name`, `User::where(...)`), Carbon, collections, and most "magical-feeling" PHP libraries. Understanding them turns that magic into something you can reason about, debug, and build yourself.

> **Jargon check.** A *magic method* is not magic you write a spell for — it is a callback the PHP engine invokes for you at well-defined moments. You define the method; PHP decides when to call it.

---

## What you'll learn

- What magic methods are, the `__` naming convention, and why those names are reserved
- The object lifecycle hooks: `__construct`, `__destruct`, and `__clone`
- Property overloading with `__get`, `__set`, `__isset`, `__unset` — and their pitfalls
- Method overloading with `__call` and `__callStatic`
- Turning objects into strings (`__toString` + `Stringable`) and into callables (`__invoke`)
- Debug and export hooks: `__debugInfo` and `__set_state`
- The two serialization pairs: `__sleep`/`__wakeup` and the modern `__serialize`/`__unserialize`
- Performance trade-offs of property overloading, and how Eloquent uses all of this in the real world

---

## Why magic methods exist

PHP objects are normally rigid: you read a declared property, call a declared method, and if it doesn't exist you get an error. That's good for safety but bad for *dynamic* APIs.

Consider an ORM. A `users` table can have any columns. You can't declare a `name` property for every possible column in advance — the schema is decided at runtime. Magic methods let an object say: "If someone reads a property I don't have, ask *me* what to do." That single idea — intercept the access and run your own code — is what powers `$user->name`, `Config::get(...)`-style facades, fluent query builders, and value objects that print themselves.

The trade-off is that this indirection is **slower and less discoverable** than real properties/methods (no autocomplete, no static analysis without extra annotations). Magic is a tool for *framework-shaped* problems, not your everyday domain code.

### The naming convention

Method names beginning with `__` are **reserved** by PHP. The engine only treats a fixed, documented set as magic; defining your own `__foo` won't get auto-invoked, but PHP reserves the prefix and may warn or break in future versions. The complete magic set is:

`__construct`, `__destruct`, `__call`, `__callStatic`, `__get`, `__set`, `__isset`, `__unset`, `__sleep`, `__wakeup`, `__serialize`, `__unserialize`, `__toString`, `__invoke`, `__set_state`, `__clone`, `__debugInfo`.

> Magic method names are **case-insensitive** (`__GET` works), but always write them in the canonical `camelCase`-after-underscores form shown above.

---

## The object lifecycle

### `__construct` — the constructor

`__construct` runs immediately after an object is created with `new`. It is the one magic method you use constantly. Modern PHP (8.0+) supports **constructor property promotion**, which declares and assigns properties in the signature:

```php
<?php

final class Money
{
    public function __construct(
        public readonly int $amountCents,
        public readonly string $currency = 'USD',
    ) {
        if ($amountCents < 0) {
            throw new InvalidArgumentException('Amount cannot be negative.');
        }
    }
}

$price = new Money(amountCents: 1999); // named args, PHP 8.0+
echo $price->amountCents; // 1999
echo $price->currency;    // USD
```

Output:
```
1999
USD
```

Notes:
- `readonly` properties (PHP 8.1+) can be assigned exactly once, typically in the constructor — perfect for immutable value objects. PHP 8.2 added `readonly` *classes* (`final readonly class Money {}`), which makes every (typed) instance property readonly in one keyword.
- PHP 8.4 also adds **asymmetric visibility** (e.g. `public private(set) int $amount;`), letting a property be publicly readable but only writable from inside the class — a lighter-weight alternative to a `__get`/`__set` wrapper when you only need a read-only-from-outside field.
- Unlike many languages, PHP does **not** support constructor overloading (one signature per class). The idiom is named static factory methods:

```php
<?php

final class Temperature
{
    private function __construct(public readonly float $celsius) {}

    public static function fromCelsius(float $c): self    { return new self($c); }
    public static function fromFahrenheit(float $f): self { return new self(($f - 32) * 5 / 9); }
}

$t = Temperature::fromFahrenheit(212.0);
echo round($t->celsius); // 100
```

### `__destruct` — the destructor

`__destruct` runs when an object has no more references and is garbage-collected, or at script shutdown. Use it to release resources (file handles, sockets). You rarely need it — PHP frees memory automatically — but it matters for non-memory resources.

```php
<?php

final class TempFile
{
    private string $path;

    public function __construct()
    {
        $this->path = tempnam(sys_get_temp_dir(), 'tf_');
    }

    public function write(string $data): void
    {
        file_put_contents($this->path, $data);
    }

    public function __destruct()
    {
        if (is_file($this->path)) {
            unlink($this->path); // clean up when the object dies
        }
    }
}

$f = new TempFile();
$f->write('hello');
unset($f); // __destruct fires here, temp file removed
```

> **Gotcha.** Do not rely on destructor *ordering* or on exceptions thrown inside `__destruct` (they cannot be caught normally and trigger a fatal error during shutdown). Treat destructors as best-effort cleanup.

### `__clone` — customizing copies

`clone $obj` makes a **shallow** copy: scalar properties are copied, but object properties still point at the *same* nested objects. `__clone` runs on the new copy right after the shallow copy, letting you deep-copy what needs isolating.

```php
<?php

final class Address
{
    public function __construct(public string $city) {}
}

final class Customer
{
    public function __construct(
        public string $name,
        public Address $address,
    ) {}

    public function __clone(): void
    {
        // Without this, both customers would share the SAME Address object.
        $this->address = clone $this->address;
    }
}

$a = new Customer('Ada', new Address('London'));
$b = clone $a;
$b->address->city = 'Paris';

echo $a->address->city; // London  (isolated thanks to __clone)
echo $b->address->city; // Paris
```

Output:
```
London
Paris
```

Remove the `__clone` method and `$a->address->city` would also become `Paris` — the classic shared-reference bug.

---

## Property overloading: `__get`, `__set`, `__isset`, `__unset`

> **Jargon check.** "Overloading" in the PHP manual means *dynamically creating* properties/methods at access time — **not** the compile-time overloading from Java/C++. These hooks fire only for properties that are **inaccessible or non-existent**: undeclared properties, or `private`/`protected` ones accessed from outside the class. They never fire for accessible public properties.

| Trigger | Magic method |
|---|---|
| Reading an inaccessible/undefined property | `__get($name)` |
| Writing an inaccessible/undefined property | `__set($name, $value)` |
| `isset()` / `empty()` on one | `__isset($name)` |
| `unset()` on one | `__unset($name)` |

```php
<?php

final class Config
{
    private array $items = [];

    public function __get(string $name): mixed
    {
        echo "[get $name] ";
        return $this->items[$name] ?? null;
    }

    public function __set(string $name, mixed $value): void
    {
        echo "[set $name] ";
        $this->items[$name] = $value;
    }

    public function __isset(string $name): bool
    {
        return isset($this->items[$name]);
    }

    public function __unset(string $name): void
    {
        unset($this->items[$name]);
    }
}

$c = new Config();
$c->timeout = 30;          // [set timeout]
echo $c->timeout, "\n";    // [get timeout] 30
var_dump(isset($c->timeout)); // bool(true)  -> __isset
unset($c->timeout);           // __unset
var_dump(isset($c->timeout)); // bool(false)
```

Output:
```
[set timeout] [get timeout] 30
bool(true)
bool(false)
```

### Critical pitfalls of `__get`/`__set`

1. **They only fire for inaccessible properties.** If you declare `public mixed $timeout;`, the magic methods are bypassed and you read/write the real property directly. This surprises everyone once.

   > **Subtle gotcha (typed properties).** A *declared-but-uninitialized typed* property does **not** fall through to `__get`. Reading it throws `Error: Typed property X::$y must not be accessed before initialization` — the property exists, it just has no value yet. `__get` fires only when the property is genuinely **undeclared** or **inaccessible from the current scope**, not merely "unset".

2. **`isset()` requires `__isset`.** Without `__isset`, `isset($c->timeout)` returns `false` even when the data exists — because PHP can't see your backing array. Likewise `$c->timeout ?? 'x'` and `empty()` depend on `__isset`.

3. **Nested writes need a reference return.** `$obj->arr['k'] = 1` where `arr` is a magic property will trigger a notice/error unless `__get` returns *by reference* (`public function &__get(...)`). Most implementations store the value back through `__set` instead of mutating in place.

4. **No type safety and no IDE help.** Magic properties are invisible to static analysis. Document them with `@property` PHPDoc so tools and humans know they exist:

```php
<?php

/**
 * @property string $timeout
 * @property-read string $appName
 */
final class Config { /* ... */ }
```

### PHP 8.4: property hooks (the modern alternative for *known* properties)

PHP 8.4 introduced **property hooks**, which let a *declared* property run code on read/write without `__get`/`__set`. This is the right tool when the field is known at design time — you keep type safety, IDE autocomplete, and static analysis, and you avoid the magic-method overhead entirely.

```php
<?php

final class Temperature
{
    // A "virtual" property with a get hook — no backing field, computed on read.
    public float $fahrenheit {
        get => $this->celsius * 9 / 5 + 32;
    }

    public function __construct(public float $celsius) {}
}

$t = new Temperature(100.0);
echo $t->fahrenheit; // 212
```

A more realistic example with both hooks and validation:

```php
<?php

final class User
{
    public string $name {
        set (string $value) {
            $this->name = ucfirst(trim($value)); // normalize on write
        }
    }

    public string $slug {
        get => strtolower(str_replace(' ', '-', $this->name)); // computed on read
    }

    public function __construct(string $name)
    {
        $this->name = $name; // runs the set hook
    }
}

$u = new User('  ada lovelace');
echo $u->name; // Ada lovelace
echo $u->slug; // ada-lovelace
```

**When to use which:**
- **Property hooks (8.4+)** → known, named fields that need computed/validated access. Typed, fast, statically analyzable.
- **`__get`/`__set`** → genuinely *dynamic* schemas where you don't know the field names in advance (ORM rows, config bags). Property hooks cannot replace this because they require a declared property name.

> Property hooks do **not** make `__get`/`__set` obsolete — they cover the opposite case (fixed schema). Eloquent still leans on `__get`/`__set` precisely because columns are dynamic.

---

## Method overloading: `__call` and `__callStatic`

- `__call($name, $args)` fires when you call an **inaccessible or undefined instance method**.
- `__callStatic($name, $args)` fires for an inaccessible or undefined **static** method.

This is how fluent builders and Laravel facades work.

```php
<?php

final class QueryBuilder
{
    private array $wheres = [];

    /** @return $this */
    public function __call(string $name, array $args): static
    {
        if (str_starts_with($name, 'where')) {
            // whereName('Ada') -> column "name"
            $column = lcfirst(substr($name, 5));
            $this->wheres[] = [$column, $args[0]];
            return $this;
        }

        throw new BadMethodCallException("Unknown method {$name}().");
    }

    public function toSql(): string
    {
        $parts = array_map(fn ($w) => "{$w[0]} = '{$w[1]}'", $this->wheres);
        return 'SELECT * FROM users WHERE ' . implode(' AND ', $parts);
    }
}

echo (new QueryBuilder())
    ->whereName('Ada')
    ->whereCity('London')
    ->toSql();
```

Output:
```
SELECT * FROM users WHERE name = 'Ada' AND city = 'London'
```

`__callStatic` is identical but static. A simplified facade:

```php
<?php

final class Cache
{
    private array $store = [];

    public function set(string $k, mixed $v): void { $this->store[$k] = $v; }
    public function get(string $k): mixed { return $this->store[$k] ?? null; }
}

final class CacheFacade
{
    private static ?Cache $instance = null;

    public static function __callStatic(string $name, array $args): mixed
    {
        self::$instance ??= new Cache();
        return self::$instance->{$name}(...$args); // argument unpacking
    }
}

CacheFacade::set('user:1', 'Ada');
echo CacheFacade::get('user:1'); // Ada
```

Output:
```
Ada
```

That `forwardCallTo`-style pattern is exactly how Laravel's `Facade` base class proxies `Cache::get()` to the underlying singleton resolved from the service container.

> **Gotcha.** If a real method with the same name exists but is `private`, calling it from outside routes to `__call` — not an error. This can silently hide bugs. Prefer explicit `throw new BadMethodCallException` for unknown names.

---

## `__toString` and the `Stringable` interface

`__toString` defines how an object converts to a string — in `echo`, string interpolation, `(string)` casts, and concatenation.

```php
<?php

final class Money implements Stringable
{
    public function __construct(
        public readonly int $cents,
        public readonly string $currency = 'USD',
    ) {}

    public function __toString(): string
    {
        return sprintf('%s %.2f', $this->currency, $this->cents / 100);
    }
}

$m = new Money(1999);
echo $m;                 // USD 19.99
echo "Total: {$m}\n";    // Total: USD 19.99
$s = (string) $m;        // explicit cast
```

Output:
```
USD 19.99Total: USD 19.99
```

Key facts:
- **`Stringable` is auto-implemented.** Since PHP 8.0, any class with `__toString` is *implicitly* treated as `Stringable`, so `$x instanceof Stringable` is `true` even without an explicit `implements`. Declaring it anyway documents intent and lets you type-hint `string|Stringable`.
- **Throwing exceptions from `__toString` has been allowed since PHP 7.4** (before 7.4, doing so caused a fatal error). PHP 7.4 also began requiring a `string` return; PHP 8.0 relaxed that to standard type coercion. The distinctly *PHP 8.0* change is the **implicit `Stringable`** behavior above.

---

## `__invoke` — callable objects

`__invoke` lets you call an *object* like a function: `$obj(...)`. The object is then a valid `callable`, so it works with `array_map`, route handlers, middleware, etc. These are sometimes called "functors" or single-action classes.

```php
<?php

final class Multiplier
{
    public function __construct(private int $factor) {}

    public function __invoke(int $n): int
    {
        return $n * $this->factor;
    }
}

$double = new Multiplier(2);
echo $double(21);                          // 42
print_r(array_map($double, [1, 2, 3]));    // object used as a callable
var_dump(is_callable($double));            // bool(true)
```

Output:
```
42
Array
(
    [0] => 2
    [1] => 4
    [2] => 6
)
bool(true)
```

In Laravel, an invokable controller (a class with a single `__invoke`) can be registered without naming a method: `Route::get('/report', GenerateReport::class);`.

---

## `__debugInfo` — controlling `var_dump`

`__debugInfo` returns the array that `var_dump()` shows for an object. Use it to hide secrets or summarize complex internals.

```php
<?php

final class ApiClient
{
    public function __construct(
        private string $endpoint,
        private string $apiKey,
    ) {}

    public function __debugInfo(): array
    {
        return [
            'endpoint' => $this->endpoint,
            'apiKey'   => '***redacted***', // never dump the real key
        ];
    }
}

var_dump(new ApiClient('https://api.example.com', 'sk-secret-123'));
```

Output:
```
object(ApiClient)#1 (2) {
  ["endpoint"]=>
  string(23) "https://api.example.com"
  ["apiKey"]=>
  string(14) "***redacted***"
}
```

> Note: `__debugInfo` only affects `var_dump`. `print_r` and `var_export` ignore it.

---

## `__set_state` — re-creating objects from `var_export`

`var_export()` can output runnable PHP that reconstructs a value. For objects, it emits a `\ClassName::__set_state([...])` call, and your static `__set_state` method rebuilds the instance from the property array. This is how compiled config/cache files (e.g. `php artisan config:cache`, which `var_export`s the merged config to a PHP file) can round-trip data to disk as executable PHP. Note that Laravel's config is almost entirely arrays/scalars — `__set_state` only matters if an *object* lands in exported PHP, which is why putting objects in cached config is discouraged unless they implement `__set_state`.

```php
<?php

final class Point
{
    public function __construct(public int $x = 0, public int $y = 0) {}

    public static function __set_state(array $state): static
    {
        return new self($state['x'], $state['y']);
    }
}

$code = var_export(new Point(3, 4), true);
echo $code, "\n";

// The exported code is runnable PHP:
$restored = eval('return ' . $code . ';');
echo "{$restored->x},{$restored->y}\n";
```

Output:
```
\Point::__set_state(array(
   'x' => 3,
   'y' => 4,
))
3,4
```

Without `__set_state`, evaluating that exported code throws an `Error`, because PHP has no default way to populate a fresh object from the array.

> **Note on `self` vs `static`.** The example returns `new self(...)`, which always builds a `Point` even if a subclass is exported. The `: static` return type is honest only for a `final` class (where `self === static`). If the class can be extended, prefer `new static(...)` so a subclass round-trips as itself. Also be aware that the `$state` array `var_export` produces uses the **declaring class's property names**, including private ones, so your `__set_state` must map exactly those keys.

> **Security note.** `__set_state` only runs when you `eval`/`include` `var_export` output. Treat that output exactly like serialized data: never `eval()` it from an untrusted source.

---

## Serialization

PHP can turn an object into a storable string (`serialize()`) and back (`unserialize()`). There are **two** magic-method pairs for customizing this. Since PHP 7.4 the newer pair is preferred.

### The modern pair: `__serialize` / `__unserialize` (PHP 7.4+)

- `__serialize(): array` returns exactly the data you want stored.
- `__unserialize(array $data): void` receives that array and restores state.

This is cleaner and more flexible than the old pair because *you control the entire payload* (keys can be anything; you don't have to expose real property names).

> **Important:** `unserialize()` does **not** run `__construct`. PHP creates the instance without invoking the constructor and then calls `__unserialize()` (or `__wakeup()` for the legacy pair) to populate it. So any setup you put only in the constructor — default values, dependency wiring, validation — will be skipped on restore. Re-establish that state inside `__unserialize`.

```php
<?php

final class Session
{
    private ?PDO $db = null; // a live resource we must NOT serialize

    public function __construct(public string $userId, public array $data = []) {}

    public function __serialize(): array
    {
        // Only persist plain data; drop the live connection.
        return ['userId' => $this->userId, 'data' => $this->data];
    }

    public function __unserialize(array $state): void
    {
        $this->userId = $state['userId'];
        $this->data   = $state['data'];
        $this->db     = null; // reconnect lazily later
    }
}

$s = new Session('u-42', ['theme' => 'dark']);
$blob = serialize($s);
$copy = unserialize($blob);

echo $copy->userId; // u-42
print_r($copy->data);
```

Output:
```
u-42
Array
(
    [theme] => dark
)
```

### The legacy pair: `__sleep` / `__wakeup`

- `__sleep(): array` returns a list of **property names** to keep; everything else is dropped.
- `__wakeup(): void` runs after unserialization to re-establish resources.

```php
<?php

final class LegacySession
{
    private $db; // not serializable

    public function __construct(public string $userId) {}

    public function __sleep(): array
    {
        return ['userId']; // keep only this property
    }

    public function __wakeup(): void
    {
        $this->db = null; // rebuild connection on demand
    }
}
```

**Rules and precedence:**
- If **both** pairs are defined, PHP uses `__serialize`/`__unserialize` and ignores `__sleep`/`__wakeup`.
- Prefer the modern pair in new code: it's clearer, can return arbitrary keys, and avoids the property-name coupling of `__sleep`.
- For objects you simply don't want serialized at all, implement the `__serialize`/`__unserialize` to throw, or mark with appropriate guards.

> **Security note.** Never `unserialize()` untrusted input — it can instantiate arbitrary classes and trigger `__wakeup`/`__destruct` "gadget chains" (object-injection attacks). Use JSON for external data. If you must, pass `['allowed_classes' => [...]]` as the second argument to `unserialize()`.

---

## Performance considerations of `__get`/`__set`

Magic property access is meaningfully slower than real property access. Every read/write of a magic property is a *function call* plus (usually) an array lookup, versus a direct memory read for a declared property. In tight loops over millions of accesses this is measurable.

Guidance:
- Use magic properties where the schema is genuinely dynamic (ORM rows, config bags). Don't use them as a lazy substitute for declaring known properties.
- If you have a fixed set of fields, declare them — you get speed, type safety, and IDE support for free.
- `readonly` declared properties + a real constructor beat magic for value objects on every axis.
- Eloquent accepts the overhead deliberately because the convenience of `$user->any_column` is worth more than raw speed for typical request volumes — but it caches attribute logic and casts to keep it reasonable.

---

## Real-world: how Eloquent uses magic methods

Laravel's Eloquent is a tour of nearly every magic method at once:

```php
<?php

use App\Models\User;

$user = User::find(1);     // __callStatic on the model -> new query builder
echo $user->name;          // __get -> reads from the $attributes array (+ casts/accessors)
$user->email = 'a@b.com';  // __set -> writes to $attributes (+ mutators)
isset($user->name);        // __isset -> checks attributes/relations
unset($user->name);        // __unset -> removes from attributes
echo (string) $user;       // __toString -> JSON (via toJson())
$user->where('active', 1); // __call -> forwards to the underlying query builder
```

What's happening under the hood:
- **`__get`/`__set`** read/write the model's internal `$attributes` array. They also run **accessors/mutators** and **casts**. In Laravel 9+ the modern syntax is a single method returning an `Attribute` object:

```php
<?php

use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }
}
```

  When you read `$user->name`, `__get` notices a `name()` accessor and routes through it. (The older `getNameAttribute()` / `setNameAttribute()` style still works.)
- **`__call`** forwards unknown methods (like `where`, `orderBy`) to a fresh `Builder`. **`__callStatic`** does the same for static calls (`User::where(...)`) by spinning up an instance first.
- **`__isset`/`__unset`** make `isset($user->name)` and `unset($user->relation)` behave intuitively across attributes *and* loaded relationships.
- **`__toString`** serializes the model to JSON (its body is `return $this->escapeWhenCastingToString ? e($this->toJson()) : $this->toJson();`), which is why `return $user;` from a controller produces a JSON response.
- **`__sleep`/`__wakeup`** — interestingly, Eloquent still uses the *legacy* serialization pair (not `__serialize`/`__unserialize`). `__sleep` clears cached casts and resolved-relation callbacks before serializing; `__wakeup` re-boots the model. So "prefer the modern pair in new code" is the rule for *your* classes, but you'll see the legacy pair alive and well in framework source.

This single example explains why mastering magic methods is the key to *reading Laravel source* and debugging "where did this property come from?" mysteries.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `__isset`, so `isset()`/`??` lie.**
   With only `__get`/`__set`, `isset($obj->prop)` returns `false` and `$obj->prop ?? 'x'` returns `'x'` even when the data exists.
   **Fix:** Always implement `__isset` (and `__unset`) alongside `__get`/`__set` for any backing store.

2. **Declaring the property you meant to make magic.**
   If a property is declared `public`, `__get`/`__set` never fire for it — the real property wins. People add a magic layer and wonder why it's bypassed.
   **Fix:** Keep magic-backed data in a private array (e.g. `$attributes`), not as declared public properties.

3. **Routing private/missing methods to `__call` and swallowing typos.**
   A misspelled or `private` method silently lands in `__call`. If `__call` doesn't validate the name, bugs hide.
   **Fix:** In `__call`/`__callStatic`, `throw new BadMethodCallException("...")` for names you don't recognize.

4. **Throwing or relying on order in `__destruct`.**
   Exceptions in `__destruct` during shutdown cause fatal errors you can't catch; destruction order is not guaranteed.
   **Fix:** Keep destructors to simple, exception-free cleanup. Use explicit `close()`/`try-finally` for critical teardown.

5. **Shallow `clone` sharing nested objects.**
   `clone $obj` copies object properties *by reference*; mutating the copy mutates the original's nested object.
   **Fix:** Implement `__clone` to `clone` each nested object you need isolated.

6. **`unserialize()` on untrusted input.**
   This enables object-injection / gadget-chain attacks via `__wakeup`/`__destruct`.
   **Fix:** Use JSON for external data, or pass `['allowed_classes' => false]` (or a whitelist) to `unserialize()`.

7. **Expecting `__toString` to be called by `json_encode` or `print_r`.**
   It isn't — `__toString` only handles *string* conversions. `json_encode` uses `JsonSerializable::jsonSerialize()`.
   **Fix:** Implement `JsonSerializable` for JSON output; `__toString` for string contexts.

---

## ✅ Best Practices

- **Prefer real, typed properties and methods.** Reach for magic only when the API is genuinely dynamic (ORM, config, facades).
- **Document magic members** with `@property`, `@property-read`, and `@method` PHPDoc so IDEs and static analyzers (PHPStan/Psalm) understand your class.
- **Validate names in `__call`/`__callStatic`** and throw `BadMethodCallException` for the unknown.
- **Implement the full property quartet** (`__get`/`__set`/`__isset`/`__unset`) together for consistent behavior.
- **Use the modern serialization pair** (`__serialize`/`__unserialize`) over `__sleep`/`__wakeup` in new code.
- **Use `Stringable` and `JsonSerializable` explicitly** to declare the contracts you support.
- **Redact secrets in `__debugInfo`** so dumps and error pages never leak credentials.
- **Deep-copy in `__clone`** whenever a copy must be independent of the original's nested objects.
- **Keep destructors trivial** and never throw from them.
- **Measure before optimizing**, but be aware magic property access is slower than direct access in hot paths.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What are magic methods, and how does PHP decide to call them?**
A. They're reserved `__`-prefixed methods the engine invokes automatically on specific events (construction, missing-property access, missing-method calls, string/serialize conversion, etc.). You define them; PHP calls them at the right moment. They only fire when the normal path fails (e.g. an *inaccessible* property triggers `__get`).

**Q2. When exactly do `__get`/`__set` fire — and when do they NOT?**
A. Only for **inaccessible or undefined** properties: undeclared names, or `private`/`protected` accessed from outside the class scope. They never fire for accessible (e.g. public) declared properties — those are read/written directly.

**Q3. Difference between `__call` and `__callStatic`?**
A. `__call` handles missing/inaccessible **instance** method calls (`$obj->foo()`); `__callStatic` handles missing/inaccessible **static** calls (`Class::foo()`). Both receive `(string $name, array $args)`. Laravel facades use `__callStatic` to proxy to a container-resolved instance.

**Q4. `__sleep`/`__wakeup` vs `__serialize`/`__unserialize` — which wins and why?**
A. If both are defined, `__serialize`/`__unserialize` (PHP 7.4+) win. The modern pair returns/consumes an arbitrary array — you fully control the payload — whereas `__sleep` only returns a list of property names. Prefer the modern pair in new code.

**Q5. How does `__toString` relate to `Stringable`, and what's the PHP 8 change?**
A. Any class with `__toString` is *automatically* `Stringable` (implicit since PHP 8.0). Throwing exceptions from `__toString` and the `string` return requirement actually landed earlier, in PHP 7.4 (PHP 8.0 then relaxed the return to type coercion); the headline 8.0 change is the implicit `Stringable`. `json_encode` does **not** use `__toString` — that's `JsonSerializable`.

**Q6. (Under the hood) How does Eloquent make `$user->name` work for arbitrary columns?**
A. Eloquent stores row data in a private `$attributes` array. Accessing `$user->name` is an undefined property, so PHP calls Eloquent's `__get`, which looks up `name` in `$attributes`, applies any cast and accessor (`name()` returning an `Attribute`, or legacy `getNameAttribute`), and returns the result. Writes go through `__set` into `$attributes`; query methods like `where` route via `__call`/`__callStatic` to a `Builder`. So a single magic-method layer adapts a static class to a dynamic schema.

**Q7. What does `clone` do by default and how do you change it?**
A. Default `clone` is a **shallow** copy: scalars copied, object properties shared by reference. Implement `__clone` on the class; it runs on the new copy, where you `clone` nested objects to make a deep, independent copy.

**Q8. Why is `unserialize()` on user input dangerous?**
A. It can instantiate arbitrary classes and invoke their `__wakeup`/`__destruct`, enabling PHP object-injection "gadget chains" to achieve code execution or other side effects. Use JSON for external data, or restrict with `unserialize($s, ['allowed_classes' => false])`.

**Q9. What are the performance implications of `__get`/`__set`?**
A. Each access is a method call (plus typically an array lookup) instead of a direct memory read — noticeably slower in tight loops. Use them for dynamic schemas; declare real properties when the field set is known.

**Q10. What is `__invoke` and where is it used?**
A. It makes an object callable as `$obj(...)`, so the instance satisfies `callable` and `is_callable()`. Used for functors, strategy objects, `array_map` callbacks, and Laravel single-action (`__invoke`) controllers.

---

## 📋 Quick Reference / Cheat Sheet

| Method | Signature | Fires when |
|---|---|---|
| `__construct` | `(...): void` | Object created with `new` |
| `__destruct` | `(): void` | Object GC'd / script end |
| `__get` | `(string $name): mixed` | Read inaccessible/undefined property |
| `__set` | `(string $name, mixed $value): void` | Write inaccessible/undefined property |
| `__isset` | `(string $name): bool` | `isset()`/`empty()` on such a property |
| `__unset` | `(string $name): void` | `unset()` on such a property |
| `__call` | `(string $name, array $args): mixed` | Inaccessible/undefined instance method |
| `__callStatic` | `static (string $name, array $args): mixed` | Inaccessible/undefined static method |
| `__toString` | `(): string` | `echo`, cast, interpolation, concat |
| `__invoke` | `(...$args): mixed` | Object called like a function `$o(...)` |
| `__clone` | `(): void` | After `clone $obj` (runs on the copy) |
| `__debugInfo` | `(): array` | `var_dump()` |
| `__set_state` | `static (array $state): object` | `eval`'d output of `var_export()` |
| `__serialize` | `(): array` | `serialize()` (modern, preferred) |
| `__unserialize` | `(array $data): void` | `unserialize()` (modern, preferred) |
| `__sleep` | `(): array` | `serialize()` (legacy; returns prop names) |
| `__wakeup` | `(): void` | `unserialize()` (legacy) |

**Rules of thumb**
- Property quartet `__get`/`__set`/`__isset`/`__unset` → implement together.
- Magic only fires when the normal path *fails* (inaccessible/undefined member).
- `__serialize`/`__unserialize` override `__sleep`/`__wakeup` when both exist.
- `__toString` ⇒ implicit `Stringable`; for JSON use `JsonSerializable`.
- Default `clone` is shallow; deep-copy nested objects in `__clone`.
- Never `unserialize()` untrusted input.

---

## 🧪 Mini Exercises

1. **Typed config bag.** Build a `Settings` class backed by a private `$items` array. Implement `__get`, `__set`, `__isset`, and `__unset` so that `isset()`, the `??` operator, and `unset()` all behave correctly. Add a `@property` PHPDoc block for three known keys and verify your IDE/PHPStan recognizes them.

2. **Fluent builder via `__call`.** Create an `HtmlBuilder` where `->divClass('x')->spanId('y')` builds nested HTML through `__call`, parsing the tag and attribute from the method name. Throw `BadMethodCallException` for any method that doesn't match your `tagAttr` pattern.

3. **Deep-clone a graph.** Model a `Team` that holds an array of `Member` objects (each `Member` holds an `Address`). Write `__clone` so cloning a `Team` produces fully independent members and addresses. Prove it by mutating the clone and checking the original is untouched.

4. **Safe serialization.** Write a `DbConnection` value object that holds a live PDO-like resource plus a DSN string. Implement `__serialize`/`__unserialize` to persist only the DSN and reconnect lazily on `__unserialize`. Confirm `serialize()` output contains no resource and that a round-trip restores the DSN.

5. **Invokable rate limiter.** Implement a `RateLimiter` callable (`__invoke($key): bool`) that allows N calls per key per run. Use it directly (`$limiter('ip:1')`) and as an `array_filter` callback over a list of keys, returning only those still under the limit.
