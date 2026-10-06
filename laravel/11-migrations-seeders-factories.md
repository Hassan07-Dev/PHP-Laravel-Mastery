# Migrations, Seeders & Factories

Database schema and data are not something you click together in phpMyAdmin and hope to remember. In a professional Laravel project they live in code, in version control, reproducible on any machine with one command. This module teaches the three tools that make that possible: **migrations** (version-controlled schema changes), **seeders** (scripts that insert known data), and **factories** (blueprints that generate realistic fake records on demand).

> **Why this matters for interviews:** "How do you manage database schema across a team?" and "How do you set up test data?" are extremely common backend questions. Migrations/seeders/factories are the canonical Laravel answer, and they tie directly into testing, CI, and deployments.

## **What you'll learn**

- Why migrations exist and how `up()`/`down()` make schema changes reversible and version-controlled
- The full `Blueprint` vocabulary: column types, modifiers, indexes, and foreign keys
- Every schema command you need: `migrate`, `rollback`, `refresh`, `fresh`, `status`, plus how **batches** work
- How `schema:dump` (squashing) keeps a long migration history fast
- How to write seeders, wire them through `DatabaseSeeder`, and run them in tests
- How to build factories with Faker, **states**, **sequences**, and relationships (`for`/`has`)
- The crucial difference between `make()` and `create()` and when to reach for each

---

## 1. Migrations: schema as version-controlled code

### Why migrations exist (the WHY)

Imagine a three-person team. You add a `phone` column to `users`. How do your teammates get that column? You could send them an SQL script in Slack, but then everyone runs it (or forgets to) at different times, in different orders, and production drifts from staging drifts from your laptop. Nobody can answer "what does the schema look like right now?" without inspecting a live database.

A **migration** is a PHP class that *describes* a schema change. It is committed to Git alongside your code. Running `php artisan migrate` applies any migrations that haven't run yet, in order, on whatever machine you're on. The schema becomes a deterministic function of your commit history. That is the entire point: **reproducibility and shared history**.

Laravel tracks which migrations have run in a table called `migrations`. Each row records the migration filename and a **batch** number (more on batches later). Anything not in that table is "pending" and will run on the next `migrate`.

### Creating a migration

```bash
php artisan make:migration create_posts_table
```

This creates a timestamped file like `database/migrations/2026_06_18_101500_create_posts_table.php`. The timestamp prefix is how Laravel orders migrations — they run oldest-first, which is why dependencies (a table you reference in a foreign key) must be created in an *earlier* migration.

Laravel infers intent from the name. `create_posts_table` scaffolds a `Schema::create('posts', ...)`. A name like `add_phone_to_users_table` scaffolds a `Schema::table('users', ...)` (modify existing). You can also be explicit:

```bash
php artisan make:migration add_phone_to_users_table --table=users
php artisan make:migration create_posts_table --create=posts
```

> **Laravel 11/12 note:** New Laravel apps ship with a *much* smaller default set of migrations than Laravel 10 did. The old `2014_..._create_users_table`, `create_password_reset_tokens_table`, etc. are consolidated. The mechanics below are unchanged.

### Anatomy: `up()` and `down()`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Apply the change.
     */
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();                       // bigint unsigned auto-increment PK named "id"
            $table->string('title');            // VARCHAR(255) NOT NULL
            $table->text('body');               // TEXT NOT NULL
            $table->boolean('is_published')->default(false);
            $table->timestamps();               // created_at + updated_at (nullable)
        });
    }

    /**
     * Reverse the change.
     */
    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

- **`up()`** runs on `migrate`. It moves the schema *forward*.
- **`down()`** runs on `rollback`. It must *undo* exactly what `up()` did, in reverse. If `up()` creates a table, `down()` drops it. If `up()` adds a column, `down()` drops that column.

> **Modern idiom:** Since Laravel 8, migrations are **anonymous classes** (`return new class extends Migration`). This avoids class-name collisions when two migrations are named similarly. You'll still see named classes in old codebases; both work.

A migration that can't be cleanly reversed should still implement `down()` honestly — even if that means doing nothing — but a missing/empty `down()` is a real liability because `rollback`, `refresh`, and test resets all rely on it.

---

## 2. The `Blueprint`: defining columns

Inside `Schema::create()` and `Schema::table()`, the `$table` argument is a `Blueprint`. Its methods declare columns. Here is the working vocabulary.

### Common column types

```php
Schema::create('products', function (Blueprint $table) {
    $table->id();                          // BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
    $table->uuid('public_id');             // CHAR(36) — for public-facing IDs
    $table->string('name');                // VARCHAR(255)
    $table->string('sku', 64);             // VARCHAR(64) — second arg = length
    $table->text('description');           // TEXT (~64KB)
    $table->longText('spec_sheet');        // LONGTEXT (~4GB)
    $table->integer('view_count');         // INT
    $table->bigInteger('downloads');       // BIGINT
    $table->unsignedBigInteger('owner_id');// BIGINT UNSIGNED (manual FK column)
    $table->boolean('active');             // TINYINT(1)
    $table->decimal('price', 8, 2);        // DECIMAL(8,2) — 8 total digits, 2 after the point
    $table->float('weight', precision: 53);// FLOAT/DOUBLE — approximate; never use for money
    $table->json('metadata');              // JSON column
    $table->enum('status', ['draft', 'active', 'archived']);
    $table->date('released_on');           // DATE
    $table->dateTime('reviewed_at');       // DATETIME
    $table->timestamp('seen_at');          // TIMESTAMP
    $table->timestamps();                  // created_at, updated_at (both nullable TIMESTAMP)
    $table->softDeletes();                 // deleted_at (nullable) for SoftDeletes trait
});
```

**Why `decimal`, never `float`, for money.** Floating-point types store binary approximations — `0.1 + 0.2` famously isn't `0.3`. `DECIMAL` stores exact base-10 values. Use `decimal(8, 2)` for currency and accept the storage cost.

> **Laravel 11/12 note — `float()` signature changed.** In Laravel 10 and earlier you wrote `$table->float('weight', 8, 3)` (total, places). **Laravel 11 rewrote `float`/`double` to be consistent across all databases**, so `float()` no longer accepts total/places — it takes an optional `precision` (`$table->float('weight', precision: 53)`, where precision 0–24 is a 4-byte single and 25–53 an 8-byte double), and `double()` takes no precision/scale at all (`$table->double('amount')`). Only `decimal` still takes `total`/`places`: `$table->decimal('price', total: 8, places: 2)`. Passing the old three-argument form to `float()` on Laravel 11/12 is an error. (This is exactly the kind of "did you keep up with the framework?" detail an interviewer may probe.)

**`enum` caveat.** A database `ENUM` is rigid: changing the allowed values requires another migration that alters the column, and not all databases handle `ENUM` identically. Many teams prefer a plain `string` column validated against a **PHP enum** in the model (casting). It's a legitimate interview talking point — see Best Practices.

### Column modifiers

Modifiers are chained after the column definition. They tweak how the column behaves.

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('email')->unique();              // UNIQUE index
    $table->string('phone')->nullable();            // allows NULL
    $table->string('country')->default('US');       // DEFAULT 'US'
    $table->unsignedInteger('login_count')->default(0);
    $table->string('referral_code')->nullable()->unique();
    $table->integer('age')->unsigned();             // no negatives, doubles positive range
    $table->string('nickname')->index();            // non-unique index
    $table->timestamps();
});
```

| Modifier | Effect |
|---|---|
| `->nullable()` | Column accepts `NULL`. Without it, columns are `NOT NULL`. |
| `->default($value)` | Default value when none supplied on insert. |
| `->unique()` | Adds a unique index (no two rows share the value). |
| `->index()` | Adds a normal (non-unique) index to speed up lookups. |
| `->unsigned()` | No negative values; doubles the positive range of integers. |
| `->after('col')` | (MySQL) Places the new column right after `col` when altering a table. |
| `->comment('...')` | Stores a column comment in the schema. |
| `->useCurrent()` | `DEFAULT CURRENT_TIMESTAMP` for timestamps. |

**`->after()` only matters when altering an existing table** (column order rarely matters logically, but it's nice in `DESCRIBE` output). It's MySQL-specific.

---

## 3. Foreign keys & relationships

A **foreign key (FK)** is a column whose value must match a primary key in another table — the database enforces that link. This prevents *orphan rows* (a `post` pointing at a `user_id` that doesn't exist).

### The modern way: `foreignId` + `constrained`

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();      // -> users.id
    $table->string('title');
    $table->timestamps();
});
```

`foreignId('user_id')` creates an `unsignedBigInteger` (the right type to point at an `id()` PK). `constrained()` adds the actual FK constraint, *inferring* the table name from the column: `user_id` -> `users`. If the table can't be inferred, pass it explicitly:

```php
$table->foreignId('author_id')->constrained('users');
// or, Laravel 9+:
$table->foreignId('author_id')->constrained(table: 'users', column: 'id');
```

### Cascade options

What happens to a post when its user is deleted? You decide:

```php
$table->foreignId('user_id')
      ->constrained()
      ->onDelete('cascade')      // delete posts when user is deleted
      ->onUpdate('cascade');     // update FK if users.id changes

// Fluent shorthands:
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->foreignId('category_id')->nullable()->constrained()->nullOnDelete(); // set NULL
$table->foreignId('tenant_id')->constrained()->restrictOnDelete();           // block delete
```

| Method | On parent delete... |
|---|---|
| `cascadeOnDelete()` | delete the child rows too |
| `nullOnDelete()` | set the FK to `NULL` (column must be `nullable()`) |
| `restrictOnDelete()` | refuse to delete the parent if children exist |
| `noActionOnDelete()` | DB default; behaves like restrict on most engines |

### The classic verbose form

You'll meet this in older code and other databases. `constrained()` is sugar over it:

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id');
    $table->string('title');
    $table->timestamps();

    $table->foreign('user_id')
          ->references('id')
          ->on('users')
          ->onDelete('cascade');
});
```

### Dropping a foreign key

FK constraints get an auto-generated name: `<table>_<column>_foreign`. To drop one:

```php
public function down(): void
{
    Schema::table('posts', function (Blueprint $table) {
        $table->dropForeign(['user_id']);   // pass the COLUMN in an array — Laravel derives the name
        $table->dropColumn('user_id');
    });
}
```

> **Gotcha:** On many engines you must drop the foreign key *before* dropping the column it sits on, or the operation errors. Order matters in `down()`.

---

## 4. Modifying existing tables

Use `Schema::table()` to alter a table that already exists.

```php
// add_phone_to_users_table
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('phone')->nullable()->after('email');
    });
}

public function down(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('phone');
    });
}
```

### Renaming columns

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('phone', 'mobile_number');
});
```

> **Version note:** Renaming/dropping columns historically required the `doctrine/dbal` package. **Since Laravel 9 (`renameColumn`) and Laravel 10 (`change`/`dropColumn` on more drivers), `doctrine/dbal` is no longer required** for most operations — Laravel uses native DB features. On Laravel 12 you generally don't need it at all. If you're maintaining a Laravel 8 codebase, you may still see `composer require doctrine/dbal`.

### Changing a column's type or attributes

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 100)->nullable()->change();   // redefine the column
});
```

> **Big gotcha (Laravel 11+):** `->change()` redefines the column **completely**. Any modifier you *don't* repeat is dropped. If the column was `unsigned()` and you `change()` without repeating `unsigned()`, it becomes signed. Always restate every attribute you want to keep.

### Dropping columns and tables

```php
Schema::table('posts', function (Blueprint $table) {
    $table->dropColumn('subtitle');               // one
    $table->dropColumn(['subtitle', 'old_slug']); // many
});

Schema::dropIfExists('posts');   // safe — no error if table is missing
Schema::drop('posts');           // errors if table is missing
```

### Indexes after the fact

```php
Schema::table('posts', function (Blueprint $table) {
    $table->index('published_at');                 // single column
    $table->index(['user_id', 'created_at']);      // composite (order matters!)
    $table->unique(['team_id', 'slug']);           // compound uniqueness
    $table->index('title', 'idx_posts_title');     // custom index name

    // dropping:
    $table->dropIndex(['user_id', 'created_at']);  // by columns
    $table->dropUnique('posts_team_id_slug_unique'); // by name
});
```

**Why indexes?** An index is a sorted lookup structure (typically a B-tree). Without one, finding rows by a column requires scanning the whole table — O(n). With one, it's roughly O(log n). Index columns you filter (`WHERE`), join, or sort by often. The cost: indexes consume space and slow down writes slightly, so don't index everything.

**Composite index order matters.** An index on `(user_id, created_at)` helps queries filtering by `user_id` alone or `user_id` + `created_at`, but *not* queries filtering by `created_at` alone — this is the "leftmost prefix" rule.

---

## 5. The migration commands

```bash
php artisan migrate            # run all pending migrations
php artisan migrate:status     # table of each migration + Ran?/batch
php artisan migrate:rollback   # undo the LAST batch (calls down())
php artisan migrate:rollback --step=1   # undo just one migration
php artisan migrate:reset      # roll back ALL migrations
php artisan migrate:refresh    # reset + migrate (rebuild via up/down)
php artisan migrate:fresh      # DROP all tables, then migrate (ignores down())
```

### `refresh` vs `fresh` — a common interview trap

- **`migrate:refresh`** runs every `down()` in reverse, then every `up()`. It relies on your `down()` methods being correct. Slower, but exercises rollback logic.
- **`migrate:fresh`** literally **drops every table** (`DROP TABLE`) and re-runs `up()` only. It ignores `down()` entirely. Faster and bulletproof against broken `down()` methods, but obviously destructive.

Both wipe data. Both have a `--seed` flag to re-seed afterward:

```bash
php artisan migrate:fresh --seed
```

> **`--pretend`** prints the SQL a migration would run without executing it — great for reviewing a risky change:
> ```bash
> php artisan migrate --pretend
> ```

### Batches

When you run `migrate`, every migration applied in that run shares one **batch number**. The `migrations` table looks like:

```
+----+-------------------------------+-------+
| id | migration                     | batch |
+----+-------------------------------+-------+
|  1 | 0001_01_01_create_users_table |     1 |
|  2 | 2026_06_01_create_posts_table |     1 |
|  3 | 2026_06_10_add_phone_to_users |     2 |
+----+-------------------------------+-------+
```

`migrate:rollback` (no flags) undoes the **most recent batch** — here, batch 2 (one migration). Run `migrate` again and the *next* set gets batch 3. This is why "rollback" doesn't always undo just one file: it undoes everything that went out together. Use `--step=N` to roll back a precise number of migrations regardless of batch.

```bash
php artisan migrate:rollback --batch=2   # roll back a specific batch (Laravel 11+)
```

---

## 6. `schema:dump` — squashing migrations

After a couple of years, a project can have 300+ migration files. Running them all on a fresh database (e.g., in CI for every test run) gets slow, and tooling has to parse hundreds of files. **Squashing** collapses the entire current schema into a single SQL dump.

```bash
php artisan schema:dump                 # write the current schema to a .sql file
php artisan schema:dump --prune         # write the dump AND delete the existing migration files
```

This produces `database/schema/mysql-schema.sql` (named per connection). On the next `migrate` or `migrate:fresh`, Laravel loads that SQL file in one shot, then runs only the migrations created *after* the dump.

Key facts to state in an interview:

- Squashing is purely an optimization; it doesn't change your final schema.
- The dump uses your DB's native CLI (`mysqldump`, `pg_dump`), so that binary must be available.
- `--prune` deletes the squashed migration files, so **commit the `.sql` dump to Git** — it now *is* your early history.
- New migrations after the dump are unaffected and run normally on top of the loaded schema.

---

## 7. Seeders: inserting known data

### Why seeders exist

Migrations define *structure*. Seeders insert *data* — and specifically **known, deterministic data** you need every time: an admin account, the list of countries, default roles/permissions, lookup tables. Anyone can spin up a working app with `migrate --seed` instead of hand-entering reference rows.

### Creating and writing a seeder

```bash
php artisan make:seeder UserSeeder
```

```php
<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        User::create([
            'name'     => 'Admin',
            'email'    => 'admin@example.com',
            'password' => Hash::make('secret'),
        ]);

        // Bulk insert (skips Eloquent events/casts/timestamps — fast, but blunt):
        User::insert([
            ['name' => 'Bot', 'email' => 'bot@example.com', 'password' => Hash::make('x')],
        ]);
    }
}
```

> **`create()` vs `insert()` in a seeder:** `create()` goes through Eloquent — it fills `created_at`/`updated_at`, applies casts/mutators, fires model events. `insert()` is a raw query builder call: faster for thousands of rows but skips all of that (you must supply timestamps yourself if the column is `NOT NULL`).

### `DatabaseSeeder` — the entry point

`database/seeders/DatabaseSeeder.php` is the seeder `db:seed` runs by default. Chain others from its `run()` with `$this->call()`:

```php
class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            RoleSeeder::class,
            UserSeeder::class,
            PostSeeder::class,   // order matters: roles/users must exist before posts FK them
        ]);
    }
}
```

### Running seeders

```bash
php artisan db:seed                          # runs DatabaseSeeder
php artisan db:seed --class=UserSeeder       # run one seeder directly
php artisan migrate:fresh --seed             # rebuild schema, then seed
php artisan db:seed --force                  # required to seed in production
```

> **Safety:** `db:seed` and `migrate:fresh` refuse to run in `production` without `--force`, because they can destroy data. This guard exists for a reason — never script `--force` casually.

---

## 8. Factories: generating fake data on demand

### Why factories exist

Seeders are for *specific* rows. **Factories** are templates for *generating many realistic-but-fake rows* — exactly what you want for development data ("give me 200 users") and especially **tests** ("a published post belonging to an admin"). They lean on **Faker**, a library that produces fake names, emails, sentences, dates, etc.

### The `HasFactory` trait and definition

Models opt in with the `HasFactory` trait (already on the default `User` model). The trait resolves the factory by convention (`App\Models\Post` -> `Database\Factories\PostFactory`). When your names don't follow that convention, Laravel 11.39+ adds a `#[UseFactory(...)]` attribute you can place on the model instead of overriding `newFactory()`:

```php
use Database\Factories\Admin\PostFactory;
use Illuminate\Database\Eloquent\Attributes\UseFactory;

#[UseFactory(PostFactory::class)]
class Post extends Model
{
    use HasFactory;   // still required to call Post::factory()
}
```

```bash
php artisan make:factory PostFactory
```

```php
<?php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<\App\Models\Post>
 */
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title'        => fake()->sentence(4),
            'slug'         => fake()->unique()->slug(),
            'body'         => fake()->paragraphs(3, asText: true),
            'is_published' => fake()->boolean(70),     // 70% chance true
            'user_id'      => User::factory(),         // creates a related user lazily
        ];
    }
}
```

`definition()` returns the default attributes. `fake()` is a global helper returning the Faker generator. `fake()->unique()->slug()` guarantees no duplicates within one factory run.

### Using factories: `make` vs `create`

This distinction is asked constantly.

```php
use App\Models\Post;

Post::factory()->make();              // builds a Post instance in memory — NOT saved to DB
Post::factory()->create();            // builds AND inserts into the DB (returns the saved model)
Post::factory()->count(50)->create(); // a Collection of 50 saved posts

// Override defaults:
Post::factory()->create(['title' => 'Fixed Title']);

// raw() gives an attribute array, no model at all:
Post::factory()->raw();   // ['title' => '...', 'body' => '...', ...]
```

- **`make()`** — in-memory only. Use it when you don't need persistence (testing validation, serialization, a method that doesn't touch the DB). Fast, no DB hit.
- **`create()`** — persisted. Use it when the code under test reads from the database or you need an `id`.

### States

A **state** is a named variation of the defaults — e.g., a "published" post or an "admin" user. Define a method that calls `state()`:

```php
public function published(): static
{
    return $this->state(fn (array $attributes) => [
        'is_published' => true,
        'published_at' => now(),
    ]);
}

public function unpublished(): static
{
    return $this->state(['is_published' => false, 'published_at' => null]);
}
```

```php
Post::factory()->published()->create();
Post::factory()->count(10)->unpublished()->create();
```

The default `UserFactory` ships with an `unverified()` state as a built-in example.

### Sequences

A **sequence** cycles attribute values across the records being created — useful when you want a *spread* rather than identical or fully random values.

```php
use Illuminate\Database\Eloquent\Factories\Sequence;

User::factory()
    ->count(6)
    ->state(new Sequence(
        ['role' => 'admin'],
        ['role' => 'editor'],
        ['role' => 'viewer'],
    ))
    ->create();
// Output: 6 users cycling admin, editor, viewer, admin, editor, viewer

// Closure form gives you the index:
Post::factory()
    ->count(100)
    ->state(new Sequence(fn (Sequence $s) => ['position' => $s->index + 1]))
    ->create();
```

### Relationships: `for` (belongsTo) and `has` (hasMany)

```php
use App\Models\User;
use App\Models\Post;

// has: create a user WITH 3 posts (hasMany side)
User::factory()
    ->has(Post::factory()->count(3))
    ->create();

// magic method form (pluralized relationship name):
User::factory()->hasPosts(3)->create();

// for: create a post BELONGING TO a specific/new user (belongsTo side)
Post::factory()->for(User::factory()->state(['name' => 'Jane']))->create();

// magic method form (singular relationship name):
Post::factory()->forUser(['name' => 'Jane'])->create();

// attach attributes to the related child:
User::factory()
    ->has(Post::factory()->count(3)->state(['is_published' => true]), 'posts')
    ->create();
```

In `definition()`, writing `'user_id' => User::factory()` means: if the caller doesn't supply a user, lazily create one. If they pass `->for($existingUser)`, that overrides it.

### Factories in seeders and tests

In a seeder, factories replace dozens of hand-written `create()` calls:

```php
class PostSeeder extends Seeder
{
    public function run(): void
    {
        User::factory()
            ->count(10)
            ->has(Post::factory()->count(5))
            ->create();
        // Output: 10 users, each with 5 posts => 50 posts total
    }
}
```

In a test, combine `RefreshDatabase` with factories. **New Laravel 11/12 apps scaffold tests with Pest by default** (`php artisan test` runs Pest; `vendor/bin/pest` works too), though PHPUnit is fully supported and a `phpunit.xml` ships in both. Here is the **Pest** form (the default), using the `uses()` helper to apply the `RefreshDatabase` trait:

```php
<?php

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);   // migrate a fresh DB + wrap each test in a transaction

it('shows a user their published posts', function () {
    $user = User::factory()
        ->has(Post::factory()->count(3)->published())
        ->create();

    $this->actingAs($user)
        ->get('/posts')
        ->assertOk();

    expect(Post::all())->toHaveCount(3);
});
```

The same test as a classic **PHPUnit** class (what you'll still see in many codebases and in upgraded apps):

```php
<?php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;   // migrates a fresh DB and wraps each test in a transaction

    public function test_a_user_can_see_their_published_posts(): void
    {
        $user = User::factory()
            ->has(Post::factory()->count(3)->published())
            ->create();

        $response = $this->actingAs($user)->get('/posts');

        $response->assertOk();
        $this->assertCount(3, Post::all());
    }
}
```

> **`RefreshDatabase`** gives every test a clean, migrated database. It runs migrations once, then wraps each test in a DB **transaction** that's rolled back at the end — fast and isolated. (`DatabaseMigrations`, by contrast, runs `migrate:fresh` before every test: slower, but necessary if your test commits transactions.)

> **Seeding inside tests:** call `$this->seed()` for `DatabaseSeeder` or `$this->seed(RoleSeeder::class)` for one. In a **PHPUnit** class you can set `protected bool $seed = true;` to auto-seed after each refresh; in **Pest**, do it per file with `uses(RefreshDatabase::class); beforeEach(fn () => $this->seed(RoleSeeder::class));`. Inside a Pest test `$this->seed(...)` still works because the test closure is bound to the underlying `TestCase`.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting or breaking `down()`.** If `up()` adds three columns but `down()` only drops one, `migrate:rollback` and `migrate:refresh` leave the schema in a corrupt state. **Fix:** make `down()` the exact inverse of `up()`, or default to `migrate:fresh` in dev where `down()` is ignored — but never ship a half-written `down()`.

2. **Foreign key type mismatch.** `foreign('user_id')->references('id')->on('users')` fails with a vague constraint error if `user_id` is a plain `integer` while `users.id` is `bigint unsigned`. **Fix:** use `foreignId('user_id')->constrained()`, which creates the correct `unsignedBigInteger` type automatically; or manually match `unsignedBigInteger`.

3. **Editing an already-run migration instead of writing a new one.** Changing a migration file that teammates/CI/production already ran does nothing on their machines — Laravel sees it in the `migrations` table and skips it. Worse, you get silent drift. **Fix:** schema changes after a migration has shipped go in a *new* migration. Treat run migrations as immutable history.

4. **`->change()` silently dropping modifiers (Laravel 11+).** `$table->string('email')->change()` on a previously-`unique`, `nullable` column drops both unless you repeat them. **Fix:** restate every modifier you want to keep: `$table->string('email', 191)->nullable()->unique()->change();` — and review with `migrate --pretend`.

5. **Using `make()` when the code under test hits the database.** `Post::factory()->make()` returns an unsaved model with no `id`. A test that then calls `GET /posts/{id}` 404s. **Fix:** use `create()` whenever persistence or an `id` is needed; reserve `make()` for in-memory-only assertions.

6. **Seeder order vs foreign keys.** Calling `PostSeeder` before `UserSeeder` violates the `posts.user_id` FK. **Fix:** order `$this->call([...])` so parents seed before children, or let factories create dependencies lazily (`'user_id' => User::factory()`).

7. **`fake()->unique()` exhausting its pool.** Asking for 1,000 unique two-letter values from a tiny set throws `OverflowException`. **Fix:** widen the generator (`unique()->numerify()`, `unique()->slug()`) or reduce the count.

---

## ✅ Best Practices

- **One logical change per migration.** Small, focused migrations are easier to review, roll back, and reason about than one giant file.
- **Treat shipped migrations as immutable.** New changes = new migrations. Never rewrite history teammates have already applied.
- **Always provide a correct `down()`** even though dev usually uses `fresh`. CI, deployments, and code review may depend on it.
- **Use `foreignId(...)->constrained()`** over the verbose `foreign/references/on` form unless you need a non-conventional table/column. Less to get wrong.
- **Prefer `string` + a PHP enum cast over a DB `enum` column** when the value set may evolve — altering DB enums is painful and inconsistent across engines.
- **Use `decimal` for money, never `float`.** Approximation errors compound.
- **Index what you query**, not everything. Filter/join/sort columns benefit; respect the leftmost-prefix rule on composite indexes.
- **Keep deterministic data in seeders, random data in factories.** Admin user = seeder. "200 demo posts" = factory.
- **Squash with `schema:dump --prune`** once the migration list gets long, and commit the `.sql` dump.
- **Use `RefreshDatabase` + factories in tests**; don't depend on a manually-populated database.
- **Wrap multi-step structural migrations carefully** — note that MySQL DDL is not transactional, so a mid-migration failure can leave a partial schema; PostgreSQL/SQLite are transactional for DDL.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What problem do migrations solve, and how does Laravel know which have run?**
Migrations make schema changes version-controlled and reproducible across environments. Laravel records each applied migration (filename + batch) in a `migrations` table; on `migrate`, it runs only files not present there, in timestamp order.

**Q2. `migrate:refresh` vs `migrate:fresh` — what's the difference and when do you use each?**
`refresh` rolls everything back via `down()` then re-runs `up()`; it tests your rollback logic but breaks if `down()` is wrong. `fresh` drops all tables outright and re-runs `up()`, ignoring `down()` — faster and more reliable for local resets. Both wipe data and accept `--seed`.

**Q3. How do batches relate to rollbacks?**
Every migration in a single `migrate` run shares one batch number. `migrate:rollback` (no flags) undoes the most recent *batch* — which may be several files. Use `--step=N` to roll back an exact number, or `--batch=N` (Laravel 11+) for a specific batch.

**Q4. `make()` vs `create()` on a factory?**
`make()` instantiates the model in memory without touching the DB (no `id`); `create()` persists it and returns the saved model. Use `make()` for in-memory tests, `create()` when the code reads from the DB or needs an `id`. `raw()` returns just an attributes array.

**Q5. How do you model a one-to-many relationship in factories?**
From the parent: `User::factory()->has(Post::factory()->count(3))->create()` or the magic `->hasPosts(3)`. From the child: `Post::factory()->for(User::factory())->create()` or `->forUser(...)`. In `definition()`, `'user_id' => User::factory()` lazily creates a parent if none is supplied.

**Q6. What does `schema:dump` do, and why? (under the hood)**
It collapses the entire current schema into a native SQL dump (via `mysqldump`/`pg_dump`) at `database/schema/<conn>-schema.sql`. On `migrate`/`migrate:fresh`, Laravel loads that file in one statement instead of replaying hundreds of migrations, then runs only migrations created after the dump. `--prune` deletes the squashed files, so the dump must be committed. It's a pure performance optimization, mostly for CI speed.

**Q7. (Under the hood) What actually happens when you run `php artisan migrate`?**
Laravel reads the `migrations` table to find applied files, diffs against `database/migrations/`, sorts pending files by their timestamp prefix, then for each: instantiates the migration class and calls `up()`. The `Blueprint` accumulates column/index commands, which a grammar object compiles into the SQL for your specific driver (MySQL/PostgreSQL/SQLite/SQL Server). Each successfully-run file is inserted into `migrations` with the current batch number. With `--pretend`, it compiles and prints the SQL instead of executing it.

**Q8. When would you choose a DB `enum` vs a string column with a PHP enum cast?**
A DB `enum` enforces the value set at the storage layer but is rigid — changing values needs another `ALTER`, and engines vary. A `string` column cast to a PHP `enum` keeps validation in the app where it's easy to evolve and works uniformly across databases. Most modern Laravel teams pick the latter.

**Q9. How do you reset the database between tests, and what's the performance trade-off?**
`RefreshDatabase` migrates once then wraps each test in a transaction rolled back afterward — fast and isolated, but breaks if the test commits a transaction. `DatabaseMigrations` runs `migrate:fresh` before each test — slower but handles committed transactions. `DatabaseTruncation` truncates tables instead of migrating, a middle ground.

**Q10. Why `decimal` over `float` for currency?**
`float`/`double` store binary approximations, so values like `0.1` aren't exact and rounding errors accumulate over arithmetic. `decimal(p, s)` stores exact base-10 numbers, which is mandatory for money. (Bonus version detail: in Laravel 11+ `float()` no longer takes total/places — it takes an optional `precision`, and `double()` takes no arguments; only `decimal` keeps `total`/`places`.)

**Q11. What testing framework do new Laravel apps use, and how do you get a fresh database per test?**
Since Laravel 11, new apps scaffold tests with **Pest** by default (PHPUnit still works and `php artisan test` runs either). For database isolation you apply the `RefreshDatabase` trait — in Pest via `uses(RefreshDatabase::class)`, in a PHPUnit class via `use RefreshDatabase;`. It migrates once and wraps each test in a rolled-back transaction. Pair it with factories rather than relying on a hand-seeded DB.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate
php artisan make:migration create_posts_table --create=posts
php artisan make:migration add_phone_to_users_table --table=users
php artisan make:seeder PostSeeder
php artisan make:factory PostFactory --model=Post

# Run / inspect schema
php artisan migrate                 # apply pending
php artisan migrate --pretend       # print SQL, don't run
php artisan migrate:status          # what's applied + batches
php artisan migrate:rollback        # undo last batch
php artisan migrate:rollback --step=1
php artisan migrate:reset           # roll back everything
php artisan migrate:refresh         # reset + migrate (uses down())
php artisan migrate:fresh           # drop all tables + migrate (ignores down())
php artisan migrate:fresh --seed    # ...and seed

# Seed
php artisan db:seed                       # DatabaseSeeder
php artisan db:seed --class=UserSeeder
php artisan db:seed --force               # required in production

# Squash
php artisan schema:dump                   # dump current schema
php artisan schema:dump --prune           # dump + delete old migration files
```

```php
// Blueprint columns
$table->id();                          $table->foreignId('user_id')->constrained();
$table->string('name', 120);           $table->foreignId('cat_id')->nullable()->constrained()->nullOnDelete();
$table->text('body');                  $table->boolean('active')->default(false);
$table->integer('n'); $table->bigInteger('big');   $table->decimal('price', 8, 2);
$table->json('meta'); $table->enum('s', ['a','b']); $table->timestamps(); $table->softDeletes();

// Modifiers
->nullable() ->default($v) ->unique() ->index() ->unsigned() ->after('col') ->comment('..')

// FK cascade
->cascadeOnDelete() ->nullOnDelete() ->restrictOnDelete()
$table->dropForeign(['user_id']); $table->dropColumn('user_id');

// Alter
$table->renameColumn('a', 'b');  $table->string('x')->nullable()->change();
$table->index(['user_id','created_at']);  $table->dropIndex(['user_id','created_at']);
```

```php
// Factories
Post::factory()->make();                       // memory only
Post::factory()->create();                     // persisted
Post::factory()->count(50)->create();
Post::factory()->create(['title' => 'X']);     // override
Post::factory()->published()->create();        // state
Post::factory()->count(3)->state(new Sequence(['r'=>'a'], ['r'=>'b']))->create();
User::factory()->has(Post::factory()->count(3))->create();   // hasMany
Post::factory()->for(User::factory())->create();             // belongsTo

// In tests (Pest = default since Laravel 11; PHPUnit also supported)
uses(RefreshDatabase::class);          // Pest
use RefreshDatabase;                   // PHPUnit class
$this->seed();  $this->seed(RoleSeeder::class);
```

---

## 🧪 Mini Exercises

1. **Schema design.** Write a single migration that creates a `comments` table with: an `id`, a `body` text column, a `foreignId('post_id')` that cascades on delete, a nullable `foreignId('parent_id')` (self-reference for threaded replies) that sets null on delete, an `approved` boolean defaulting to `false`, `timestamps`, and a composite index on `(post_id, created_at)`. Implement a correct `down()`.

2. **Reversible alter.** Add a `migration` that gives the existing `posts` table a `nullable` `published_at` timestamp placed `after('is_published')` and a `unique` `slug` string. Make `down()` drop both cleanly (mind the unique index).

3. **Factory with states & relationships.** Create a `CommentFactory`. Add a `spam()` state (`approved => false`, body from `fake()->sentences(2, asText: true)`) and an `approved()` state. Then write one line that creates a post with 5 approved comments and 2 spam comments.

4. **Seeder orchestration.** Write a `DatabaseSeeder::run()` that seeds 3 specific admin users via a `UserSeeder` (deterministic emails), then uses factories to create 20 regular users each owning between 1 and 4 posts. Ensure FK order is respected.

5. **Squash drill.** Describe (in commands and one sentence each) the exact steps to squash a project's 150 migrations into a schema dump, what file gets created, what you must commit, and how a brand-new teammate's `php artisan migrate` behaves afterward.
