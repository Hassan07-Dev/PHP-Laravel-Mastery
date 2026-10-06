# Eloquent ORM: Basics

Eloquent is Laravel's built-in **ORM** (Object-Relational Mapper). It lets you talk to your database tables using PHP objects and expressive method calls instead of writing raw SQL by hand. This module takes you from "what even is an ORM" to confidently performing CRUD, casting attributes, defining accessors/mutators, scoping queries, and using soft deletes — the bread-and-butter skills you'll be drilled on in any Laravel interview.

> **Versions covered:** Laravel 12 (with notes for Laravel 10/11) and PHP 8.4 (with notes for 8.1–8.3). The Eloquent API has been remarkably stable across these versions, so the vast majority of this applies unchanged from Laravel 9 onward.

---

## **What you'll learn**

- What an ORM is and how the **Active Record** pattern shapes Eloquent's design.
- How to generate and configure models, including the table / primary-key / timestamp conventions and how to override them.
- **Mass assignment** — `$fillable` vs `$guarded` — and why the `MassAssignmentException` exists.
- The full CRUD surface: `create`, `save`, `find`/`findOrFail`, `first`/`firstOrFail`, `all`/`get`, `update`, `delete`/`destroy`.
- "Find or make" helpers: `firstOrCreate`, `firstOrNew`, `updateOrCreate`.
- **Attribute casting** (arrays, JSON, dates, booleans, decimals, enums, encrypted) and **accessors & mutators** (modern `Attribute` + legacy style).
- **Query scopes** (local & global), **soft deletes**, and utility methods: `refresh`, `fresh`, `replicate`.

---

## 1. What is an ORM, and why use one?

A relational database stores data in **tables** (rows and columns). Your application code, however, thinks in **objects**. An **ORM (Object-Relational Mapper)** is a library that maps rows in a table to objects in your code, so you can write `$user->email` instead of parsing the result of `SELECT email FROM users WHERE id = 1`.

**Why bother?**

- **Less boilerplate.** No hand-writing repetitive SQL strings.
- **Safety.** Queries are built through a parameterized query builder, which protects you from SQL injection by default.
- **Portability.** The same Eloquent code runs against MySQL, PostgreSQL, SQLite, or SQL Server.
- **Expressiveness.** Relationships, casting, events, and scopes let you encode business rules on the model itself.

The trade-off: an ORM hides SQL, so it's easy to write code that triggers slow or excessive queries (the classic **N+1 problem**, covered in the relationships module). A good engineer knows both the ORM *and* the SQL it generates.

### The Active Record pattern

Eloquent implements the **Active Record** pattern, popularized by Ruby on Rails. The core idea: **a single object both carries the data of one row AND knows how to persist itself.** The model is the row, and the row has methods like `save()` and `delete()`.

```php
$user = new User();      // an object representing one (not-yet-saved) row
$user->name = 'Ada';     // setting a column value
$user->save();           // the object persists ITSELF to the database
```

Contrast this with the **Data Mapper** pattern (used by Doctrine, the other major PHP ORM), where a separate "entity manager" handles persistence and the entity is a plain object. Active Record is more convenient and concise; Data Mapper enforces a cleaner separation of concerns. **For interviews, be ready to name the pattern (Active Record) and contrast it with Data Mapper / Doctrine.**

---

## 2. Defining a model

Models live in `app/Models` by default. Generate one with Artisan:

```bash
php artisan make:model Post
```

The `-m` (or `--migration`) flag also creates a matching migration — extremely common because a model usually needs a table:

```bash
php artisan make:model Post -m
```

Other handy flags you'll see together (`-mfsc` is a popular combo):

```bash
# migration, factory, seeder, and resource controller in one shot
php artisan make:model Post -mfsc

# everything: migration, factory, seeder, policy, controller, and form requests
php artisan make:model Post --all     # -a is the short form

# a pivot (many-to-many) model:
php artisan make:model RoleUser --pivot
```

A freshly generated model is almost empty:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;
}
```

That tiny class already knows how to run full CRUD. The magic is **convention over configuration** — Eloquent infers everything it needs from the class name.

### Convention: table name

By default Eloquent assumes the table is the **snake_case, pluralized** form of the class name:

| Model class | Inferred table |
|-------------|----------------|
| `Post`      | `posts`        |
| `User`      | `users`        |
| `BlogPost`  | `blog_posts`   |
| `Category`  | `categories`   |

Override it with the `$table` property:

```php
class Post extends Model
{
    protected $table = 'blog_articles';
}
```

### Convention: primary key

Eloquent assumes a primary key column named `id` that is an auto-incrementing integer. Override as needed:

```php
class Post extends Model
{
    protected $primaryKey = 'post_id';   // non-default key column

    public $incrementing = false;        // e.g. for UUID / string keys

    protected $keyType = 'string';       // tell Eloquent the key is a string
}
```

> **Tip:** For UUID keys, use the `Illuminate\Database\Eloquent\Concerns\HasUuids` trait (Laravel 9+) — it sets `$incrementing = false`, `$keyType = 'string'`, and auto-generates a key on create. As of Laravel 11/12 it generates ordered **UUIDv7** values (lexicographically sortable, far better for B-tree index locality than random UUIDv4). There's also `HasUlids` for sortable ULIDs. Pair either with `$table->uuid('id')->primary()` (or `$table->ulid(...)`) in the migration.

### Convention: timestamps

Eloquent expects `created_at` and `updated_at` columns and maintains them automatically on insert/update. Disable or customize:

```php
class Post extends Model
{
    public $timestamps = false;                   // turn timestamps off entirely

    const CREATED_AT = 'created_on';              // rename the columns
    const UPDATED_AT = 'last_modified';

    protected $dateFormat = 'U';                  // storage format (here: UNIX timestamp)
}
```

The migration helper `$table->timestamps()` creates the two nullable `TIMESTAMP` columns these features rely on.

Bump only `updated_at` without changing other columns:

```php
$post->touch();              // sets updated_at = now() and saves
$post->touch('published_at'); // (Laravel 9+) touch a specific timestamp column
```

A child model can also auto-bump a parent's `updated_at` on save/delete via the `$touches` property (e.g. a `Comment` touching its `Post`) — handy for cache-busting:

```php
class Comment extends Model
{
    protected $touches = ['post'];   // names of relationships to touch
}
```

### Convention: database connection

Use a connection other than the default by setting `$connection`:

```php
class Post extends Model
{
    protected $connection = 'analytics';   // matches a key in config/database.php
}
```

---

## 3. Mass assignment: `$fillable` vs `$guarded`

**Mass assignment** means setting many attributes at once from an array — typically straight from request input:

```php
// ⚠️ ANTI-PATTERN — do NOT pass $request->all() blindly. Shown to illustrate the risk.
$post = Post::create($request->all());   // assigns every key in the array
```

This is convenient but dangerous: a malicious user could submit an extra field (say `is_admin=1` or `user_id=999`) and quietly overwrite a column you never intended to expose. Prefer validated input — `Post::create($request->validated())` (from a Form Request) or `$request->only([...])` — so only intended keys ever reach the model. As a second line of defense, Eloquent **blocks mass assignment by default** and forces you to opt in.

You declare intent with **exactly one** of these properties:

```php
class Post extends Model
{
    // ALLOW-LIST: only these columns may be mass-assigned.
    protected $fillable = ['title', 'body', 'published_at'];
}
```

```php
class Post extends Model
{
    // BLOCK-LIST: everything EXCEPT these may be mass-assigned.
    protected $guarded = ['id'];

    // Unguard everything (use with extreme caution):
    // protected $guarded = [];
}
```

- **`$fillable`** is an allow-list (preferred — explicit and safe).
- **`$guarded`** is a block-list. `protected $guarded = []` means "nothing is guarded," allowing *all* attributes — only safe when you fully control the input.

Two distinct behaviors trip people up here — get them straight:

1. **You configured `$fillable`/`$guarded`, but pass an attribute that isn't allowed.** By default Eloquent **silently discards** it (in *all* environments, including local). No exception. To turn that silence into a loud error during development, opt in via `Model::preventSilentlyDiscardingAttributes()` (typically in `AppServiceProvider::boot()`, gated to non-production):

```php
// app/Providers/AppServiceProvider.php — boot()
use Illuminate\Database\Eloquent\Model;

Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

2. **The model is "totally guarded"** — i.e. you have *not* set `$fillable` and `$guarded` is still its default `['*']` — and you call `create()`/`fill()` with any attribute. This throws, in **every** environment:

```
Illuminate\Database\Eloquent\MassAssignmentException:
Add [title] to fillable property to allow mass assignment on [App\Models\Post].
```

> **Common misconception:** The `MassAssignmentException` is *not* an automatic "non-production only" safety net for unlisted attributes — that quieter, opt-in behavior is `preventSilentlyDiscardingAttributes()`. The exception above fires specifically when the model is totally guarded. The fix for both is the same: declare `$fillable` (or guard appropriately).

**Bypassing the protection deliberately** — setting attributes one at a time is *never* mass assignment, so it's always allowed:

```php
$post = new Post();
$post->title = $request->title;   // direct assignment — never throws
$post->save();
```

You can also force-fill, which ignores guarding entirely (use sparingly, e.g. in trusted seeders):

```php
$post->forceFill(['is_featured' => true])->save();
```

---

## 4. CRUD operations

### Create

There are two idioms. **Mass-assignment create** (one call, returns a saved model):

```php
$post = Post::create([
    'title' => 'Hello',
    'body'  => 'My first post',
]);
// INSERT INTO posts (title, body, created_at, updated_at) VALUES (...);
echo $post->id;   // Output: e.g. 1  (the new auto-increment id)
```

**Make + save** (build first, persist later — handy when you need to set extra properties between):

```php
$post = new Post();
$post->title = 'Hello';
$post->body  = 'My first post';
$post->save();          // performs the INSERT

// Or use make() to instantiate WITHOUT saving (respects mass-assignment rules):
$post = Post::make(['title' => 'Draft']);
$post->save();
```

> `create()` = `make()` + `save()`. `make()` was added precisely so you could build a fillable model instance without immediately hitting the database.

### Read

```php
// By primary key — returns the model or null
$post = Post::find(1);

// By primary key, throwing ModelNotFoundException (=> HTTP 404) if missing
$post = Post::findOrFail(1);

// Find several at once — returns a Collection
$posts = Post::find([1, 2, 3]);

// First row matching a constraint
$post = Post::where('title', 'Hello')->first();

// First row or throw if none
$post = Post::where('title', 'Hello')->firstOrFail();

// ALL rows — returns a Collection of models
$posts = Post::all();

// get() runs the built query and returns a Collection
$posts = Post::where('published', true)->orderBy('created_at', 'desc')->get();
```

**`all()` vs `get()`** — a favorite interview gotcha:

- `Post::all()` retrieves **every** row (no constraints possible) and is shorthand for `Post::query()->get()`.
- `get()` is called at the end of a *query builder chain* and respects all your `where`/`orderBy`/`limit` clauses.
- Both return an `Illuminate\Database\Eloquent\Collection`.

```php
// ❌ Wrong — you cannot constrain all()
Post::all()->where(...);   // this filters IN PHP after loading EVERY row

// ✅ Right — constrain in the database
Post::where(...)->get();
```

`findOrFail` / `firstOrFail` throw `ModelNotFoundException`, which Laravel's exception handler automatically renders as a **404** response — perfect for controllers.

### Update

```php
// Pattern 1: fetch, mutate, save
$post = Post::find(1);
$post->title = 'Updated title';
$post->save();   // UPDATE only fires if attributes are "dirty" (changed)

// Pattern 2: mass-update a single model (fetch + fill + save)
$post = Post::find(1);
$post->update(['title' => 'Updated title']);

// Pattern 3: bulk update WITHOUT loading models (no events, no timestamps unless you set them)
Post::where('published', false)->update(['published' => true]);
// Output: returns the number of affected rows, e.g. 7
```

> **Subtlety:** `$model->update([...])` works on a *loaded* model, fires events, and touches `updated_at`. The query-builder `update()` on a `where(...)` chain is a single bulk SQL `UPDATE` — it does **not** load models, fire model events, or update `updated_at` (it updates exactly the columns you pass).

### Delete

```php
// Delete a model you already have
$post = Post::find(1);
$post->delete();

// destroy() deletes by primary key(s) without you fetching first
Post::destroy(1);
Post::destroy(1, 2, 3);
Post::destroy([1, 2, 3]);   // accepts an array too
// Output: number of deleted records

// Bulk delete via query (no model events for the matched rows)
Post::where('published', false)->delete();
```

`destroy()` *does* load each model and fire delete events (it loops internally), whereas `where(...)->delete()` is a single bulk SQL `DELETE` that fires **no** per-model events.

---

## 5. firstOrCreate / firstOrNew / updateOrCreate

These "upsert-ish" helpers save you from writing `if (exists) { ... } else { ... }`.

```php
// firstOrCreate: find by the FIRST array; if not found, create using BOTH arrays.
$user = User::firstOrCreate(
    ['email' => 'ada@example.com'],          // search attributes
    ['name'  => 'Ada Lovelace']              // extra attrs used ONLY on create
);
// If a matching row exists -> returns it (no write).
// If not -> INSERTs with email + name and returns the new (saved) model.

// firstOrNew: same lookup, but if not found returns an UNSAVED instance.
$user = User::firstOrNew(
    ['email' => 'ada@example.com'],
    ['name'  => 'Ada Lovelace']
);
$user->save();   // you decide when to persist; check $user->exists or ->wasRecentlyCreated

// updateOrCreate: find by first array; if found UPDATE with second array, else CREATE.
$user = User::updateOrCreate(
    ['email' => 'ada@example.com'],          // match on this
    ['name'  => 'Ada L.', 'votes' => 10]     // set/overwrite these
);
```

Useful properties on the returned model:

- `$model->exists` — `true` if the row exists in the DB.
- `$model->wasRecentlyCreated` — `true` if *this* call inserted it (vs. found an existing row).

### `createOrFirst` and the race-condition story

This used to be a simple "`firstOrCreate` is not atomic" gotcha, but the story changed in **Laravel 10.20+** (so it applies to 11/12). `firstOrCreate` and `updateOrCreate` now use **`createOrFirst()`** under the hood:

1. SELECT to find the row.
2. If missing, attempt the INSERT inside a savepoint/transaction.
3. If the INSERT hits a **unique-constraint violation** (because a concurrent request won the race), catch it and SELECT again to return the existing row.

```php
// Attempts the INSERT first; on a duplicate-key error, fetches the existing row instead.
$user = User::createOrFirst(
    ['email' => 'ada@example.com'],
    ['name'  => 'Ada Lovelace']
);
```

> **Race-condition caveat (still important):** This safety net only works if there is actually a **unique index** on the lookup column(s). Without one, two concurrent requests can still both "not find" and both insert, creating duplicates — there's nothing for the database to reject. So the rule stands: **back every `firstOrCreate`/`updateOrCreate`/`createOrFirst` lookup with a unique index.** For high-volume *bulk* inserts/updates, prefer the separate **`upsert()`** method, which issues a single `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL) / `ON CONFLICT` (PostgreSQL) statement:
>
> ```php
> Post::upsert(
>     [['slug' => 'a', 'views' => 1], ['slug' => 'b', 'views' => 2]],
>     uniqueBy: ['slug'],   // columns that uniquely identify a row
>     update: ['views'],     // columns to update on conflict
> );
> ```
>
> Note `upsert()` does **not** fire model events and (on most drivers) requires the `uniqueBy` columns to have a primary or unique index.

---

## 6. Attribute casting

By default every column comes back as a **string** (because that's how the PDO driver hands it over for most types). **Casting** tells Eloquent to convert attributes to/from native PHP types automatically, on both read and write.

In Laravel 11/12 the idiomatic place is the `casts()` **method** (added in Laravel 11). The older `protected $casts = [...]` **property** still works in all versions and is fine for Laravel 10.

```php
use Illuminate\Database\Eloquent\Model;
use App\Enums\PostStatus;

class Post extends Model
{
    // Laravel 11/12 idiom — a method, so you can use ::class and computed casts:
    protected function casts(): array
    {
        return [
            'is_published' => 'boolean',
            'meta'         => 'array',        // JSON column <-> PHP array
            'options'      => 'json',         // similar; returns array (or object via 'object')
            'published_at' => 'datetime',     // <-> Carbon instance
            'release_date' => 'date',         // <-> Carbon (date only)
            'price'        => 'decimal:2',     // string with fixed precision, e.g. "9.99"
            'status'       => PostStatus::class, // native enum cast
            'secret'       => 'encrypted',     // transparently encrypt at rest
            'tags'         => 'encrypted:array', // encrypt + JSON
        ];
    }
}
```

```php
// Laravel 10 (or any version) — the property form:
class Post extends Model
{
    protected $casts = [
        'is_published' => 'boolean',
        'meta'         => 'array',
        'published_at' => 'datetime',
        'price'        => 'decimal:2',
        'status'       => PostStatus::class,
    ];
}
```

### What each cast does

```php
$post = Post::find(1);

$post->is_published;   // bool true (DB stored 1) — not "1"
$post->meta;           // array ['views' => 10] — decoded from JSON text
$post->published_at;   // Carbon\Carbon instance -> $post->published_at->diffForHumans()
$post->price;          // "9.99" string, formatted to 2 decimals (decimal cast returns a STRING)
$post->status;         // PostStatus enum case, e.g. PostStatus::Draft
```

Writing works in reverse — assign a native value and Eloquent serializes it for storage:

```php
$post->meta = ['views' => 11, 'likes' => 3];   // stored as JSON text
$post->is_published = true;                     // stored as 1
$post->status = PostStatus::Published;          // stored as the enum's backing value
$post->save();
```

### Date casts

`date` and `datetime` return **Carbon** objects (Carbon extends PHP's `DateTime`). You can specify a storage/serialization format:

```php
protected function casts(): array
{
    return [
        'published_at' => 'datetime:Y-m-d H:i:s',  // controls JSON/array serialization format
    ];
}
```

> **Removed:** The legacy `$dates` property (a flat list of columns to treat as dates) was deprecated in Laravel 8 and **removed in Laravel 10**. In Laravel 11/12 use the `datetime`/`date` casts instead. Note: `created_at`/`updated_at` are cast to Carbon automatically — you do not list them.
>
> **`date` vs `datetime`:** both return Carbon, but the `date` cast calls `->startOfDay()` (the time component is zeroed), while `datetime` preserves the time.

### Enum casts (PHP 8.1+)

Backed enums map cleanly to a column. Define the enum:

```php
namespace App\Enums;

enum PostStatus: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';
}
```

With `'status' => PostStatus::class`, the DB stores `'draft'` and reads back `PostStatus::Draft`. This is one of the strongest examples of "modern idioms" interviewers love — it gives you type-safety end to end.

### Custom casts

For complex types, implement `CastsAttributes`:

```php
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

/** @implements CastsAttributes<Money, Money> */
class MoneyCast implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return new Money($value);                 // DB -> object
    }

    public function set(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return $value->amountInCents;             // object -> DB
    }
}
// Usage in casts(): 'wallet' => MoneyCast::class
```

> `set()` may also return an **array** to write several columns at once (e.g. an address cast writing `street`, `city`, `zip`). That's how multi-column value objects are persisted.

---

## 7. Accessors & mutators

**Accessor** = transform an attribute when you *read* it. **Mutator** = transform when you *write* it. These are great for derived/formatted values without changing how data is stored.

### Modern approach (Laravel 9+): the `Attribute` return type

A single method returns an `Attribute` describing both get and set. This is the preferred style in Laravel 10/11/12.

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    // Accessor only: a computed, virtual "full_name"
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) =>
                "{$attributes['first_name']} {$attributes['last_name']}",
        );
    }

    // Accessor + mutator on a real column
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }
}
```

```php
$user = User::find(1);     // first_name stored as "ada", last_name "lovelace"
echo $user->full_name;     // Output: ada lovelace  (note: uses raw $attributes here)
echo $user->first_name;    // Output: Ada  (accessor ucfirst'd it on read)

$user->first_name = 'ADA'; // mutator lowercases on write
$user->save();             // stored as "ada"
```

> **Naming rule:** the method name is camelCase (`fullName`), the attribute you access is snake_case (`full_name`). Eloquent maps between them.

**Caching nuance:** Accessors that return **objects** (e.g. a value object, a Carbon instance) are cached by Eloquent automatically — access the attribute twice and you get the same instance. Accessors that return **primitives** (strings, booleans, ints) are **not** cached by default; if such a value is expensive to compute, opt in with `shouldCache()`:

```php
protected function fullName(): Attribute
{
    return Attribute::make(
        get: fn () => /* expensive */ $this->compute(),   // returns a string
    )->shouldCache();   // needed only because the value is a primitive
}
```

Conversely, if a cached object-accessor depends on mutable state and you want it recomputed every access, disable caching with `->withoutObjectCaching()`.

### Legacy approach: `getXAttribute` / `setXAttribute`

You'll still see (and may be asked about) the original style. It works in all versions:

```php
class User extends Model
{
    // Accessor: getFullNameAttribute -> $user->full_name
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }

    // Mutator: setFirstNameAttribute -> assignment to $user->first_name
    public function setFirstNameAttribute(string $value): void
    {
        $this->attributes['first_name'] = strtolower($value);
    }
}
```

The naming convention: `get` + StudlyCase attribute + `Attribute` for accessors, `set` + StudlyCase + `Attribute` for mutators. The modern `Attribute` class was introduced to consolidate these two methods into one and reduce method clutter.

### Appending virtual attributes to JSON

A computed accessor isn't included when the model is serialized to JSON/array unless you append it:

```php
class User extends Model
{
    protected $appends = ['full_name'];   // include the accessor in toArray()/toJson()
}
```

---

## 8. Query scopes

A **scope** packages a reusable set of query constraints so you don't repeat `where(...)` chains everywhere.

### Local scopes — opt-in, called by name

Define a method prefixed with `scope`; call it (minus the prefix, lowercased first letter):

```php
use Illuminate\Database\Eloquent\Builder;

class Post extends Model
{
    public function scopePublished(Builder $query): void
    {
        $query->where('published', true);
    }

    // Scope with an argument
    public function scopeOfStatus(Builder $query, string $status): void
    {
        $query->where('status', $status);
    }
}
```

```php
Post::published()->get();                 // calls scopePublished
Post::ofStatus('draft')->latest()->get(); // chainable + arguments
Post::published()->ofStatus('archived')->get();
```

> **Laravel 12.4+ note:** You can now declare a local scope with the `#[Scope]` **attribute** (`Illuminate\Database\Eloquent\Attributes\Scope`) on a regular method, dropping the `scope` prefix. The attributed method should be `protected`:
>
> ```php
> use Illuminate\Database\Eloquent\Attributes\Scope;
> use Illuminate\Database\Eloquent\Builder;
>
> #[Scope]
> protected function published(Builder $query): void
> {
>     $query->where('published', true);
> }
> // Called exactly the same way: Post::published()->get();
> ```
>
> This shipped in Laravel v12.4; the `scopeXxx` convention remains fully supported and is what you'll see in most codebases.

### Global scopes — automatic, applied to every query

A global scope is applied to **all** queries for the model automatically (soft deletes are implemented as a global scope). Two ways to define one.

**1) A dedicated class implementing `Scope`:**

```php
use Illuminate\Database\Eloquent\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class PublishedScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('published', true);
    }
}
```

```php
class Post extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope(new PublishedScope);
    }
}
```

**2) An anonymous/closure scope, or the `#[ScopedBy]` attribute (Laravel 10+):**

```php
use Illuminate\Database\Eloquent\Attributes\ScopedBy;

#[ScopedBy([PublishedScope::class])]
class Post extends Model
{
    //
}
```

```php
// Closure form inside booted():
static::addGlobalScope('published', function (Builder $builder) {
    $builder->where('published', true);
});
```

**Removing a global scope for a single query:**

```php
Post::withoutGlobalScope(PublishedScope::class)->get();
Post::withoutGlobalScopes()->get();                 // remove ALL
Post::withoutGlobalScope('published')->get();       // by closure name/key
```

> **Why it matters:** Global scopes are powerful but sneaky — every query is silently filtered. Multi-tenancy and soft deletes use them. In interviews you may be asked *how soft deletes hide trashed rows* — the answer is a global scope (`SoftDeletingScope`).

---

## 9. Soft deletes

A **soft delete** marks a row as deleted (by setting a `deleted_at` timestamp) instead of physically removing it. The row stays in the table but is hidden from normal queries — letting you "undelete" or keep an audit trail.

### Setup

Add the trait and the column:

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;   // adds a global scope + casts deleted_at to datetime
}
```

Migration:

```php
Schema::table('posts', function (Blueprint $table) {
    $table->softDeletes();   // adds a nullable `deleted_at` TIMESTAMP column
});
```

### Behavior

```php
$post->delete();        // sets deleted_at = now(); row stays in DB but is hidden
$post->trashed();       // true — convenience check

Post::all();            // excludes soft-deleted rows (thanks to the global scope)
Post::withTrashed()->get();   // include soft-deleted rows
Post::onlyTrashed()->get();   // ONLY soft-deleted rows

$post->restore();       // sets deleted_at back to NULL (undelete)
$post->forceDelete();   // permanently DELETE the row from the database
```

Restore many at once:

```php
Post::onlyTrashed()->where('author_id', 5)->restore();
```

> **Gotcha:** `find()` / `findOrFail()` will **not** return a soft-deleted record unless you prefix with `withTrashed()`:
> ```php
> Post::withTrashed()->findOrFail($id);
> ```
> Also, unique constraints don't "know" about soft deletes — a soft-deleted `email` still occupies the unique index, so re-inserting the same value fails. Consider a composite unique index including `deleted_at`, or handle it in app logic.

---

## 10. Default attributes, refresh/fresh, replicate

### Default attribute values

Give new model instances default values with `$attributes`:

```php
class Post extends Model
{
    protected $attributes = [
        'status'    => 'draft',
        'views'     => 0,
        'is_public' => false,
    ];
}
```

```php
$post = new Post();
echo $post->status;   // Output: draft  (default applied before any DB write)
```

> These defaults apply at the **model layer** (when instantiating). For DB-level defaults, also set them in the migration with `->default(...)`. Both layers together is the safest setup.

### `fresh()` vs `refresh()`

Both re-read from the database, but differ in mutation:

```php
$post = Post::find(1);

// fresh(): returns a NEW instance from the DB; $post itself is unchanged.
$fresh = $post->fresh();

// refresh(): re-hydrates the SAME instance in place, discarding unsaved changes.
$post->title = 'unsaved change';
$post->refresh();          // re-reads from DB; the unsaved title is lost
echo $post->title;         // Output: the DB value, not "unsaved change"

// You can eager-load relations while refreshing:
$post->fresh(['comments']);
```

Use `refresh()` after a raw/bulk update where the in-memory model may be stale.

### `replicate()`

Clone a model into a new, **unsaved** instance (primary key and timestamps are reset):

```php
$original = Post::find(1);

$copy = $original->replicate();        // not yet in DB; no id
$copy->title = 'Copy of ' . $original->title;
$copy->save();                          // INSERTs a new row

// Exclude specific attributes from the copy:
$copy = $original->replicate(['slug', 'published_at']);
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `$fillable`/`$guarded` and hitting `MassAssignmentException`.**
   `Post::create([...])` throws because Eloquent won't mass-assign without an allow/block list.
   **Fix:** declare `protected $fillable = ['title', 'body', ...];` (preferred) or set `$guarded`. Never set `$guarded = []` on a model populated directly from `$request->all()`.

2. **Using `all()` when you meant `get()`.**
   `Post::all()->where('published', true)` loads **every** row into memory, then filters in PHP — a performance disaster at scale.
   **Fix:** filter in the database: `Post::where('published', true)->get()`.

3. **Expecting `find()` to return a soft-deleted record.**
   Once a row is soft-deleted, the global scope hides it from `find`, `first`, `all`, etc., so you get `null` and may wrongly conclude the row is gone.
   **Fix:** `Post::withTrashed()->find($id)` (or `onlyTrashed()`).

4. **Assuming `firstOrCreate` is race-safe *without* a unique index.**
   Since Laravel 10.20+, `firstOrCreate`/`updateOrCreate` use `createOrFirst()` internally, which catches a duplicate-key error and re-fetches — but **only if the database actually rejects the duplicate.** With no unique index, two concurrent requests can still both insert, creating duplicates.
   **Fix:** add a **unique index** on the lookup column(s). For bulk operations, use `upsert()`.

5. **`where(...)->update([...])` not firing model events or touching `updated_at` as expected.**
   The query-builder `update` is a single SQL statement — it doesn't instantiate models, so `saving`/`updated` events and casts/mutators don't run, and `updated_at` is only changed if you pass it.
   **Fix:** if you need events/timestamps, load the models and call `$model->update([...])`, or explicitly include `'updated_at' => now()`.

6. **Comparing a cast attribute to the wrong type.**
   Without a `boolean` cast, `if ($post->is_published === true)` is `false` because the DB returns the string `"1"`.
   **Fix:** cast it: `'is_published' => 'boolean'`.

7. **`decimal` cast returns a string, not a float.**
   `$post->price` with `decimal:2` is the string `"9.99"`. Doing strict float math/comparisons can surprise you.
   **Fix:** be aware it's a string for precision reasons; cast to float deliberately only when you accept the precision trade-off.

---

## ✅ Best Practices

- **Prefer `$fillable` (allow-list) over `$guarded`.** Explicit is safer than implicit, especially with request input.
- **Use `findOrFail`/`firstOrFail` in controllers** so missing records become clean 404s instead of null-pointer errors.
- **Push filtering into the database** (`where(...)->get()`), not into PHP collections (`all()->filter(...)`).
- **Cast everything that isn't a plain string** — booleans, dates, JSON, enums. It eliminates a whole class of type bugs and documents the column's intent.
- **Use the modern `Attribute` accessor/mutator style and the `casts()` method** in Laravel 11/12; reach for backed **enums** for status-like columns.
- **Keep query logic in scopes**, not scattered across controllers, so business rules ("what 'published' means") live in one place.
- **Reach for `updateOrCreate`/`firstOrCreate`** instead of hand-rolled `if (exists)` blocks — but back lookups with a unique index.
- **Set defaults in both the model (`$attributes`) and the migration (`->default()`)** for consistency at every layer.
- **Be deliberate about soft deletes** — they change every query's semantics. Document it and remember unique constraints don't respect `deleted_at`.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is an ORM, and which pattern does Eloquent use?**
An ORM maps database rows to PHP objects so you work with objects instead of raw SQL. Eloquent uses the **Active Record** pattern — each model instance represents a row *and* knows how to persist itself (`save`, `delete`). Contrast: Doctrine uses **Data Mapper**, separating entities from a persistence manager.

**Q2. `$fillable` vs `$guarded` — what's the difference and why do they exist?**
`$fillable` is an allow-list of mass-assignable columns; `$guarded` is a block-list. They exist to prevent **mass-assignment vulnerabilities**, where a request injects unexpected fields (e.g. `is_admin`). Without either configured, `create()`/`fill()` throws a `MassAssignmentException` (in non-production). Setting individual attributes directly (`$model->x = ...`) is never mass assignment and always allowed.

**Q3. What's the difference between `all()` and `get()`?**
`all()` is a static shortcut that fetches every row (`Post::query()->get()` under the hood) and can't take constraints. `get()` terminates a query-builder chain, respecting your `where`/`orderBy`/`limit`. Both return an Eloquent `Collection`. Filtering with `all()->where(...)` loads everything then filters in PHP — avoid it.

**Q4. `find` vs `first` vs `firstOrFail` vs `findOrFail`?**
`find` looks up by primary key (or an array of keys) and returns the model or `null`. `first` returns the first row of a query or `null`. The `*OrFail` variants throw `ModelNotFoundException` (auto-rendered as HTTP 404) instead of returning `null`.

**Q5. Explain `firstOrCreate` vs `firstOrNew` vs `updateOrCreate` (and `createOrFirst`).**
All look up by the first array. `firstOrCreate` returns the match or **creates and saves** a new row. `firstOrNew` returns the match or a **new unsaved** instance (you call `save()`). `updateOrCreate` updates the matched row with the second array, or creates one if none matched. Since Laravel 10.20+, `firstOrCreate`/`updateOrCreate` are backed by **`createOrFirst()`**, which tries the INSERT first and, on a unique-constraint violation from a concurrent request, re-fetches the existing row. That makes them race-safe **only if a unique index exists** on the lookup column(s) — always add one.

**Q6. How does attribute casting work, and what's special about the `decimal` and `enum` casts?**
Casting converts DB values (usually strings) to native PHP types on read and back on write — booleans, arrays/JSON, Carbon dates, enums, encrypted. `decimal:2` returns a **string** formatted to fixed precision (to avoid float rounding errors). An enum cast maps a column to a backed PHP 8.1 enum, giving end-to-end type safety. In Laravel 11/12, prefer the `casts()` method; the `$casts` property still works.

**Q7. Modern accessor/mutator syntax vs the legacy one?**
Modern (Laravel 9+): a single method returning `Attribute::make(get: ..., set: ...)`. Legacy: separate `getXAttribute()` / `setXAttribute()` methods. The `Attribute` class consolidates both into one method and supports caching (`->shouldCache()`). Method names are camelCase; the accessed attribute is snake_case.

**Q8. (Under the hood) How do soft deletes actually hide records, and how do global scopes work?**
The `SoftDeletes` trait registers a **global scope** (`SoftDeletingScope`) in the model's `booted()`/boot lifecycle. That scope adds `WHERE deleted_at IS NULL` to every query automatically. `delete()` is overridden to set `deleted_at = now()` instead of issuing a `DELETE`. `withTrashed()` removes the scope for that query; `onlyTrashed()` flips it to `WHERE deleted_at IS NOT NULL`. More broadly, **global scopes** implement the `Scope` contract's `apply(Builder, Model)` method and are merged into the query when the builder is constructed. (Bonus: `__call`/`__callStatic` and the magic `__get`/`__set` on `Model` are how method-name conventions like `scopeXxx` and attribute access are resolved at runtime.)

**Q9. Does `Model::create()` differ from `new Model + save()`?**
`create()` is essentially `make()` (fill a new instance respecting mass-assignment rules) followed by `save()`. `new Model; $m->x = ...; $m->save();` sets attributes individually (bypassing mass-assignment checks) and persists. `make()` exists to build a fillable instance without saving.

**Q10. What's the difference between `fresh()`, `refresh()`, and `replicate()`?**
`fresh()` returns a brand-new instance reloaded from the DB (original untouched). `refresh()` re-hydrates the *same* instance in place, discarding unsaved changes. `replicate()` clones the model into a new **unsaved** instance with the primary key and timestamps reset — useful for "duplicate this record."

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate models
php artisan make:model Post              # model only
php artisan make:model Post -m           # + migration
php artisan make:model Post -mfsc        # + migration, factory, seeder, controller
php artisan make:model Post --all        # everything
```

```php
// --- Conventions / overrides ---
protected $table = 'blog_articles';
protected $primaryKey = 'post_id';
public    $incrementing = false;
protected $keyType = 'string';
public    $timestamps = false;
protected $connection = 'analytics';
protected $attributes = ['status' => 'draft'];     // default attribute values

// --- Mass assignment ---
protected $fillable = ['title', 'body'];           // allow-list (preferred)
protected $guarded  = ['id'];                       // block-list

// --- Create ---
Post::create([...]);                                // make + save
$p = Post::make([...]); $p->save();                 // build, then save
$p = new Post; $p->title = 'x'; $p->save();         // direct assignment

// --- Read ---
Post::find($id);            Post::findOrFail($id);
Post::find([1,2,3]);        Post::where(...)->first();
Post::where(...)->firstOrFail();
Post::all();                Post::where(...)->get();

// --- Update ---
$p->update([...]);                                  // loaded model + events
Post::where(...)->update([...]);                    // bulk, no model events

// --- Delete ---
$p->delete();   Post::destroy($id);   Post::destroy([1,2,3]);
Post::where(...)->delete();                         // bulk

// --- Find-or helpers (back lookups with a UNIQUE index!) ---
Post::firstOrCreate([...lookup...], [...extra...]);
Post::firstOrNew([...lookup...],  [...extra...]);   // unsaved
Post::createOrFirst([...lookup...], [...extra...]); // INSERT-first; race-safe w/ unique idx
Post::updateOrCreate([...lookup...], [...values...]);
Post::upsert([[...], [...]], uniqueBy: ['slug'], update: ['views']); // bulk, no events

// --- Casting (Laravel 11/12 method form) ---
protected function casts(): array {
    return [
        'is_published' => 'boolean',
        'meta'         => 'array',          // also: 'json', 'object', 'collection'
        'published_at' => 'datetime',       // also: 'date', 'datetime:Y-m-d'
        'price'        => 'decimal:2',       // returns a STRING
        'status'       => PostStatus::class, // backed enum
        'secret'       => 'encrypted',       // also: 'encrypted:array'
    ];
}

// --- Accessor / mutator (modern) ---
protected function fullName(): Attribute {
    return Attribute::make(
        get: fn ($v, $attrs) => "{$attrs['first_name']} {$attrs['last_name']}",
        set: fn ($v) => strtolower($v),
    );
}
protected $appends = ['full_name'];          // include accessor in JSON

// --- Scopes ---
public function scopePublished(Builder $q): void { $q->where('published', true); } // classic
#[Scope] protected function published(Builder $q): void { ... }  // Laravel 12.4+ attribute
Post::published()->get();
static::addGlobalScope(new PublishedScope);  // in booted()
Post::withoutGlobalScope(PublishedScope::class)->get();

// --- Soft deletes ---
use SoftDeletes;                              // trait on the model
$table->softDeletes();                        // migration: deleted_at column
$p->delete();  $p->trashed();  $p->restore();  $p->forceDelete();
Post::withTrashed()->get();   Post::onlyTrashed()->get();

// --- Utility ---
$p->fresh();        // new instance from DB
$p->refresh();      // re-hydrate same instance
$p->replicate();    // unsaved clone (id/timestamps reset)
$p->exists;  $p->wasRecentlyCreated;
$p->isDirty();  $p->isClean();  $p->wasChanged();  $p->getChanges();  $p->getOriginal();
```

---

## 🧪 Mini Exercises

1. **Model setup.** Generate a `Product` model with a migration in one command. Configure it to use the table `catalog_products`, a string primary key `sku` (non-incrementing), and `$fillable` for `name`, `price`, and `status`. Add model-level default attributes so new products start as `status = 'draft'` and `price = 0`.

2. **Casting + enum.** Create a backed enum `ProductStatus` (`draft`, `active`, `discontinued`). Add a `casts()` method to `Product` that casts `status` to the enum, `price` to `decimal:2`, `is_featured` to `boolean`, and `metadata` to `array`. Then write code that creates a product, reads `$product->status`, and asserts it is a `ProductStatus` enum instance.

3. **Find-or helpers.** Write a single statement that finds a product by `sku = 'ABC-123'`, updating its `price` to `19.99` if it exists or creating it (with `name = 'Widget'`) if it doesn't. Then explain in a comment why this is *not* safe under concurrency and what database-level safeguard you'd add.

4. **Scope + soft deletes.** Add the `SoftDeletes` trait and a `deleted_at` column. Write a local scope `scopeActive` that returns only products with `status = ProductStatus::Active`. Then write queries to (a) list active products, (b) list active products including trashed ones, and (c) restore all trashed products whose `name` contains "Widget".

5. **Accessor + appends.** Add a virtual `display_name` accessor that returns the product name in title case prefixed with its SKU (e.g. `"ABC-123: Super Widget"`), using the modern `Attribute` style with caching enabled. Make sure `display_name` appears when the model is serialized to JSON.
