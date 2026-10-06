# Eloquent Relationships

Relational databases store data in separate tables, but real-world entities are connected: a user *has* posts, a post *belongs to* a category, a video *has* comments, a project *involves* many users. **Eloquent relationships** are the mechanism Laravel's ORM (Object–Relational Mapper) gives you to express those connections in PHP — so instead of hand-writing `JOIN`s and `WHERE` clauses, you write `$user->posts` and Eloquent does the SQL for you.

This module assumes you already know the basics of Eloquent models and migrations. Everything here targets **Laravel 12** on **PHP 8.4**, with notes where Laravel 10/11 or PHP 8.1–8.3 differ.

> **Jargon check.** *ORM*: a layer that maps database rows to PHP objects (models). *Pivot table*: an intermediate table that joins two other tables in a many-to-many relationship. *Eager loading*: fetching related records up front in a small number of queries. *N+1*: a performance bug where you accidentally run one query per parent row.

---

**What you'll learn**

- The six core relationship types: `hasOne`, `belongsTo`, `hasMany`, `belongsToMany`, `hasManyThrough`/`hasOneThrough`, and the polymorphic family.
- Foreign-key naming conventions — and how to override every one of them.
- How to query relationships: dynamic property vs. method call, and lazy vs. eager loading.
- Eager loading deeply: `with()`, `load()`, nested/constrained loads, `loadCount()`.
- The N+1 problem — how to *detect* it with `preventLazyLoading()` and how to fix it.
- Constraining by relationship existence: `whereHas`, `withWhereHas`, `doesntHave`.
- Aggregating relationships: `withCount`, `withSum`, `withExists`.
- Writing through relationships: `save`, `create`, `associate`, `attach`/`sync`/`toggle`, and `push`.

---

## 1. Why relationships exist (the "why" before the "how")

Imagine you `SELECT * FROM posts` and now want each post's author name. Without relationships you'd loop over the posts, and for each one run `SELECT * FROM users WHERE id = ?`. That's the manual, error-prone, slow way.

Eloquent lets you declare the connection **once**, on the model, as a method:

```php
class Post extends Model
{
    public function author(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Then access it as if it were a property:

```php
$post = Post::find(1);
echo $post->author->name; // "Ada Lovelace"
```

The payoff: the relationship is **reusable**, **testable**, **chainable as a query builder**, and **eager-loadable** to avoid performance traps. That last point is what separates juniors from seniors in interviews.

A relationship method *always* returns a relationship object (e.g. `BelongsTo`, `HasMany`). Accessing it as a **property** (`$post->author`) triggers the query and returns a **model or collection**. Calling it as a **method** (`$post->author()`) returns the relationship instance, which you can keep chaining query constraints onto.

---

## 2. Foreign-key conventions

Eloquent guesses column names so you write less code. Memorize these — interviewers love them.

| Relationship | Default foreign key | Default local/owner key | Lives on |
|---|---|---|---|
| `belongsTo` | `<method>_id` (e.g. `user_id`) | `id` of the related model | the child table |
| `hasOne` / `hasMany` | `<parent>_id` (e.g. `user_id`) | `id` of the parent | the child table |
| `belongsToMany` | `<model>_id` for both | — | the pivot table |

The default pivot table name is the two related model names, **singular**, **snake_case**, in **alphabetical order**, joined by `_`. So `User` + `Role` → `role_user` (r before u). Override any default by passing extra arguments:

```php
// belongsTo(related, foreignKey, ownerKey)
return $this->belongsTo(User::class, 'author_id', 'id');

// hasMany(related, foreignKey, localKey)
return $this->hasMany(Post::class, 'writer_id', 'id');
```

---

## 3. One-to-one: `hasOne` and `belongsTo`

A `User` has one `Profile`; a `Profile` belongs to a `User`. The **foreign key lives on the child** (`profiles.user_id`).

```php
class User extends Model
{
    public function profile(): \Illuminate\Database\Eloquent\Relations\HasOne
    {
        return $this->hasOne(Profile::class);
    }
}

class Profile extends Model
{
    public function user(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

The migration that backs it:

```php
Schema::create('profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('bio')->nullable();
    $table->timestamps();
});
```

Usage:

```php
$user = User::find(1);
echo $user->profile->bio;   // forward: User -> Profile
$profile = Profile::find(5);
echo $profile->user->name;  // inverse: Profile -> User
```

**Which side is `belongsTo`?** The model whose table holds the foreign-key column. If `profiles` has `user_id`, then `Profile belongsTo User`.

### `hasOneOfMany`: the "latest/oldest one" trick

Sometimes you have many rows but want a single "current" one — e.g. a user's most recent login. Use `latestOfMany`, `oldestOfMany`, or `ofMany`:

```php
public function latestLogin(): \Illuminate\Database\Eloquent\Relations\HasOne
{
    return $this->hasOne(Login::class)->latestOfMany();
}

public function largestOrder(): \Illuminate\Database\Eloquent\Relations\HasOne
{
    // ofMany(column to aggregate, aggregate function)
    return $this->hasOne(Order::class)->ofMany('total', 'max');
}
```

---

## 4. One-to-many: `hasMany`

A `User` writes many `Post`s. Same foreign-key story (`posts.user_id`), but the parent side returns a **collection**.

```php
class User extends Model
{
    public function posts(): \Illuminate\Database\Eloquent\Relations\HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

```php
$user = User::find(1);

foreach ($user->posts as $post) {
    echo $post->title;
}

// Because the method returns a query builder, you can constrain it:
$recent = $user->posts()
    ->where('published', true)
    ->latest()
    ->take(5)
    ->get();
```

> **Property vs. method, again.** `$user->posts` runs `SELECT * FROM posts WHERE user_id = 1` once and **caches** the collection on the model. `$user->posts()` gives you a fresh query builder every time — use it when you need extra constraints.

---

## 5. Many-to-many: `belongsToMany`

A `User` has many `Role`s and a `Role` belongs to many `User`s. There's no foreign key on either table; instead a **pivot table** (`role_user`) holds pairs of IDs.

```php
class User extends Model
{
    public function roles(): \Illuminate\Database\Eloquent\Relations\BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}

class Role extends Model
{
    public function users(): \Illuminate\Database\Eloquent\Relations\BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }
}
```

The pivot migration:

```php
Schema::create('role_user', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('role_id')->constrained()->cascadeOnDelete();
    $table->boolean('is_primary')->default(false); // extra pivot column
    $table->timestamps();
    $table->primary(['user_id', 'role_id']); // composite key prevents duplicates
});
```

### Extra pivot columns: `withPivot` and `withTimestamps`

By default Eloquent only reads the two ID columns from the pivot. To read extra columns you must list them with `withPivot`. To have Eloquent maintain `created_at`/`updated_at` on the pivot, add `withTimestamps()`:

```php
public function roles(): \Illuminate\Database\Eloquent\Relations\BelongsToMany
{
    return $this->belongsToMany(Role::class)
        ->withPivot('is_primary')
        ->withTimestamps();
}
```

Access pivot data via the `pivot` accessor on each related model:

```php
foreach ($user->roles as $role) {
    echo $role->name;
    echo $role->pivot->is_primary;  // extra column
    echo $role->pivot->created_at;  // from withTimestamps()
}
```

You can rename the pivot accessor with `->as('membership')`, so you'd read `$role->membership->is_primary` instead — handy for readability.

### Filtering on the pivot

```php
$admins = $user->roles()
    ->wherePivot('is_primary', true)
    ->wherePivotIn('name', ['admin', 'owner'])
    ->get();
```

### Modifying the relationship: attach / detach / sync / toggle

These are the methods every interview touches. They operate on the **pivot table**.

```php
$user = User::find(1);

// attach: add rows. Existing rows are NOT removed; duplicates ARE created
//         unless the pivot has a unique/composite key.
$user->roles()->attach($roleId);
$user->roles()->attach([2, 3]);                     // multiple
$user->roles()->attach([2 => ['is_primary' => true]]); // with pivot data

// detach: remove specific rows, or ALL rows if called with no args
$user->roles()->detach($roleId);
$user->roles()->detach();        // removes every role from this user

// sync: make the pivot EXACTLY match the given list.
// Returns ['attached' => [...], 'detached' => [...], 'updated' => [...]]
$user->roles()->sync([1, 2, 3]);
$user->roles()->sync([1 => ['is_primary' => true], 2, 3]); // with pivot data

// syncWithoutDetaching: like sync but never removes existing rows
$user->roles()->syncWithoutDetaching([4, 5]);

// toggle: attach IDs that are missing, detach IDs that are present
$user->roles()->toggle([1, 2]);

// updateExistingPivot: change pivot columns on an already-attached row
$user->roles()->updateExistingPivot($roleId, ['is_primary' => false]);
```

```text
Output of sync([1, 2, 3]) when user previously had roles [2, 9]:
['attached' => [1, 3], 'detached' => [9], 'updated' => []]
```

> **Gotcha:** `attach` does **not** dedupe by itself. If your pivot lacks a unique constraint and you `attach(2)` twice, you get two rows. `sync`/`syncWithoutDetaching` are idempotent for the ID set and are usually what you want for form submissions (e.g. a multi-select of roles).

### A custom pivot model

When the pivot carries real behavior (extra columns, casts, accessors), back it with a model. Extend `Pivot` (or `MorphPivot` for polymorphic) and register it with `using()`:

```php
use Illuminate\Database\Eloquent\Relations\Pivot;

class Membership extends Pivot
{
    protected $casts = [
        'is_primary' => 'boolean',
        'joined_at'  => 'datetime',
    ];
}

// On the relationship:
public function roles(): \Illuminate\Database\Eloquent\Relations\BelongsToMany
{
    return $this->belongsToMany(Role::class)
        ->using(Membership::class)
        ->withPivot('is_primary', 'joined_at')
        ->as('membership');
}
```

> Pivot models do **not** have an incrementing `id` by default. If your pivot table has its own `id` primary key, set `public $incrementing = true;` on the pivot model.

---

## 6. Has-Many-Through and Has-One-Through

`hasManyThrough` reaches a **distant** relation via an **intermediate** model. Classic example: a `Country` has many `Post`s *through* `User` — countries don't store posts, but each user belongs to a country and writes posts.

Tables: `countries(id)`, `users(id, country_id)`, `posts(id, user_id)`.

```php
class Country extends Model
{
    public function posts(): \Illuminate\Database\Eloquent\Relations\HasManyThrough
    {
        // hasManyThrough(final, intermediate, fk on intermediate, fk on final, local, second local)
        return $this->hasManyThrough(Post::class, User::class);
    }
}
```

```php
$country = Country::find(1);
foreach ($country->posts as $post) {
    echo $post->title; // all posts by all users in this country
}
```

`hasOneThrough` is identical but returns a single model. Example: each `Supplier` has one `User`, and each `User` has one `AccountHistory`; the supplier can reach the history directly:

```php
public function accountHistory(): \Illuminate\Database\Eloquent\Relations\HasOneThrough
{
    return $this->hasOneThrough(AccountHistory::class, User::class);
}
```

> **Fluent string syntax (Laravel 9+).** If both legs are already defined as relationships — say `Country` has a `users()` relation and `User` has a `posts()` relation — you can build the through-relation by chaining `through()->has()`, which reuses the key conventions of those existing relations:
>
> ```php
> // String-based form
> return $this->through('users')->has('posts');
> // Dynamic form (resolves throughUsers() then posts())
> return $this->throughUsers()->hasPosts();
> ```
>
> This is often more readable than spelling out six key arguments, and it stays correct if you later rename a key on one of the underlying relations.

---

## 7. Polymorphic relationships

A **polymorphic** relationship lets one model belong to **more than one other type** of model on a single association. The textbook case: a `Comment` can be attached to a `Post` *or* a `Video`. Rather than two nullable foreign keys (`post_id`, `video_id`), you store **two columns**: a `*_id` and a `*_type` (the related model's class name).

### One-to-many polymorphic: `morphMany` / `morphTo`

```php
// comments table: id, body, commentable_id, commentable_type, timestamps
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->text('body');
    $table->morphs('commentable'); // adds commentable_id + commentable_type (+ index)
    $table->timestamps();
});
```

```php
class Comment extends Model
{
    public function commentable(): \Illuminate\Database\Eloquent\Relations\MorphTo
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments(): \Illuminate\Database\Eloquent\Relations\MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model
{
    public function comments(): \Illuminate\Database\Eloquent\Relations\MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

```php
$post  = Post::find(1);
$post->comments()->create(['body' => 'Great article!']);

$comment = Comment::find(1);
$parent  = $comment->commentable; // returns a Post OR a Video instance
```

`morphOne` is the one-to-one version (e.g. an `Image` for an `avatarable`).

### Many-to-many polymorphic: `morphToMany` / `morphedByMany`

A `Tag` can be applied to many `Post`s and many `Video`s, and each of those has many `Tag`s. You need a pivot table with polymorphic columns (`taggables`):

```php
Schema::create('taggables', function (Blueprint $table) {
    $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
    $table->morphs('taggable'); // taggable_id + taggable_type
    $table->primary(['tag_id', 'taggable_id', 'taggable_type']);
});
```

```php
class Post extends Model
{
    public function tags(): \Illuminate\Database\Eloquent\Relations\MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

class Tag extends Model
{
    public function posts(): \Illuminate\Database\Eloquent\Relations\MorphToMany
    {
        // morphedByMany: the INVERSE side of morphToMany
        return $this->morphedByMany(Post::class, 'taggable');
    }

    public function videos(): \Illuminate\Database\Eloquent\Relations\MorphToMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}
```

All the `attach`/`detach`/`sync`/`toggle` methods work here too.

### The morph map (do this in production)

By default the `*_type` column stores the fully-qualified class name, e.g. `App\Models\Post`. That couples your **database** to your **namespace** — rename or move a class and existing rows break. Fix it with a **morph map** (registered in a service provider's `boot()`):

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post'  => \App\Models\Post::class,
    'video' => \App\Models\Video::class,
]);
```

Now the `*_type` column stores `'post'` instead of the class path. There are two registration methods: `Relation::morphMap([...])` just defines the aliases, while `Relation::enforceMorphMap([...])` defines them **and throws a `ClassMorphViolationException`** the moment you try to persist a model that isn't in the map — so a forgotten alias fails loudly instead of silently writing a class-path string. Prefer `enforceMorphMap`, and add it from day one: retrofitting a map onto a table already full of class-path strings means a data migration to rewrite every `*_type` value.

### Eager-loading and querying a `morphTo`

Eager-loading a `morphTo` is special: because the parent can be several different classes, Eloquent first loads the comments, groups them by `*_type`, then runs **one query per related type**. To constrain those per-type loads, use `morphWith` inside `with`:

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$comments = Comment::with(['commentable' => function (MorphTo $morphTo) {
    $morphTo->morphWith([
        Post::class  => ['author'],          // nested-load author on Posts
        Video::class => ['transcoder'],
    ]);
}])->get();
```

To filter parents by a polymorphic relation, use `whereHasMorph` (and `whereDoesntHaveMorph`). Pass `'*'` to check every mapped type:

```php
// Comments whose commentable is a Post with a published parent
$comments = Comment::whereHasMorph(
    'commentable',
    [Post::class],
    fn (Builder $q) => $q->where('published', true)
)->get();

// Across all morph types
$comments = Comment::whereHasMorph('commentable', '*', fn (Builder $q) => $q->where('approved', true))->get();
```

---

## 8. Querying relationships: lazy vs. eager

### Lazy loading (the default, and the trap)

When you access `$user->posts` for the first time, Eloquent runs the query **on demand**. Inside a loop this becomes the **N+1 problem**:

```php
$users = User::all();          // Query 1: SELECT * FROM users
foreach ($users as $user) {
    echo $user->posts->count(); // Query 2..N+1: one SELECT per user!
}
```

With 100 users that's **101 queries**. This is the single most common performance interview question in the Laravel world.

### Eager loading with `with()`

`with()` loads the relationship up front using **one extra query** with a `WHERE IN`:

```php
$users = User::with('posts')->get();
// Query 1: SELECT * FROM users
// Query 2: SELECT * FROM posts WHERE user_id IN (1, 2, 3, ...)

foreach ($users as $user) {
    echo $user->posts->count(); // 0 extra queries
}
```

Two queries total, regardless of user count. Eloquent then **matches** posts to users in PHP by `user_id`.

### `load()` — eager loading after the fact

If you already have a model/collection and *then* decide you need a relation, use `load()`:

```php
$users = User::all();
$users->load('posts'); // one query, no N+1

// loadMissing only loads relations that aren't already loaded
$users->loadMissing('posts');
```

### Nested and multiple eager loads

```php
// Multiple relations
$users = User::with(['posts', 'profile'])->get();

// Nested: load posts, and each post's comments, and each comment's author
$users = User::with('posts.comments.author')->get();
```

### Constrained eager loading

Pass a closure to filter what gets eager-loaded — **without** triggering N+1:

```php
$users = User::with(['posts' => function ($query) {
    $query->where('published', true)->latest()->limit(3);
}])->get();
```

Modern shorthand using a typed closure and arrow function:

```php
use Illuminate\Database\Eloquent\Builder;

$users = User::with(['posts' => fn (Builder $q) => $q->where('published', true)])->get();
```

> **Per-parent `limit` (Laravel 11+).** Since Laravel 11 you can `limit()`/`take()` inside a constrained eager-load closure and get **N rows per parent**, not N rows total. Laravel rewrites it with a window function — `ROW_NUMBER() OVER (PARTITION BY posts.user_id …)` — so the `->latest()->limit(3)` example above really does give each user their three most-recent posts. On Laravel 10 and earlier this needed the `staudenmeir/eloquent-eager-limit` package; it is now built in.

### `loadCount` and counting on demand

```php
$user = User::find(1);
$user->loadCount('posts');        // adds posts_count without loading the posts
echo $user->posts_count;          // e.g. 12

$user->loadCount(['posts' => fn (Builder $q) => $q->where('published', true)]);
```

---

## 9. Detecting N+1 with `preventLazyLoading`

You can make Eloquent **throw** whenever a relationship is lazy-loaded — turning silent N+1 bugs into loud exceptions during development. Put this in `AppServiceProvider::boot()`:

```php
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());
}
```

Now `$user->posts` inside a loop where you forgot `with('posts')` throws a `Illuminate\Database\LazyLoadingViolationException`:

```text
Attempted to lazy load [posts] on model [App\Models\User] but lazy loading is disabled.
```

The `! app()->isProduction()` guard means it only fires in local/testing — production keeps lazy loading enabled so a missed eager load degrades performance rather than crashing for users. There's also `Model::preventSilentlyDiscardingAttributes()` and `Model::preventAccessingMissingAttributes()`; `Model::shouldBeStrict()` enables all three at once.

---

## 10. Querying *by* relationship existence

### `has`, `whereHas`, `orWhereHas`, `doesntHave`

Filter parents by whether a relationship exists, optionally with constraints:

```php
// Users that have at least one post
$users = User::has('posts')->get();

// Users with 3+ posts
$users = User::has('posts', '>=', 3)->get();

// Users with at least one PUBLISHED post (constrained existence check)
$users = User::whereHas('posts', fn (Builder $q) => $q->where('published', true))->get();

// OR variant
$users = User::where('vip', true)
    ->orWhereHas('posts', fn (Builder $q) => $q->where('featured', true))
    ->get();

// Users with NO posts at all
$users = User::doesntHave('posts')->get();

// Users with no PUBLISHED posts (but may have drafts)
$users = User::whereDoesntHave('posts', fn (Builder $q) => $q->where('published', true))->get();

// Nested existence (dot syntax)
$users = User::has('posts.comments')->get();
```

> `whereHas` generates a `WHERE EXISTS (SELECT ... )` subquery. It **filters parents**; it does **not** load the relation. If you want both — filter the parents *and* eager-load only the matching children — use `withWhereHas`:

```php
// Users that have published posts, AND eager-load only their published posts
$users = User::withWhereHas('posts', fn (Builder $q) => $q->where('published', true))->get();
```

This is shorthand for `with(['posts' => $constraint])->whereHas('posts', $constraint)` in one call — a genuinely useful Laravel 9+ addition.

---

## 11. Aggregating: `withCount`, `withSum`, `withExists`

These add aggregate columns to your parent models using subqueries — no N+1, no loading the children.

```php
$users = User::withCount('posts')->get();
echo $users->first()->posts_count;     // attribute is "<relation>_count"

// Multiple + constrained + aliased
$users = User::withCount([
    'posts',
    'posts as published_count' => fn (Builder $q) => $q->where('published', true),
])->get();
echo $users->first()->published_count;

// Sum / avg / min / max of a related column
$users = User::withSum('orders', 'total')->get();
echo $users->first()->orders_sum_total;   // attribute is "<relation>_sum_<column>"

$users = User::withAvg('orders', 'total')->get();   // orders_avg_total
$users = User::withMax('orders', 'total')->get();   // orders_max_total

// withExists: boolean — does at least one related row exist?
$users = User::withExists('posts')->get();
echo $users->first()->posts_exists ? 'has posts' : 'none';  // "<relation>_exists"
```

```text
Generated SQL for withCount('posts') (simplified):
SELECT users.*,
  (SELECT count(*) FROM posts WHERE posts.user_id = users.id) AS posts_count
FROM users
```

---

## 12. Writing through relationships

### `save` and `create` (one-to-many / one-to-one)

```php
$user = User::find(1);

// create: mass-assign + persist + set the foreign key. Returns the new model.
$post = $user->posts()->create([
    'title' => 'Hello',
    'body'  => '...',
]); // posts.user_id is set to 1 automatically

// save: persist an existing model instance, wiring up the foreign key
$post = new Post(['title' => 'Draft']);
$user->posts()->save($post);

// saveMany / createMany for bulk
$user->posts()->createMany([
    ['title' => 'A'],
    ['title' => 'B'],
]);
```

> `create` respects `$fillable`/`$guarded` mass-assignment rules. If a column is missing from `$fillable`, it's silently dropped — unless you've enabled `preventSilentlyDiscardingAttributes()`, which makes it throw.

> ⚠️ **Security — mass assignment.** Because `create()` (and `fill()`/`update()`) write any allowed key in the array, never pass unvalidated request input straight through: `$user->posts()->create($request->all())` lets an attacker set columns you never intended (e.g. `is_admin`, `user_id`, `published`). Always pass `$request->validated()` (from a Form Request) or an explicit whitelist, keep models guarded with a tight `$fillable`, and **never** globally disable protection with `Model::unguard()` outside of seeders.

### `associate` and `dissociate` (belongsTo)

On the **child** side, set or clear the foreign key:

```php
$post = Post::find(1);
$user = User::find(5);

$post->author()->associate($user); // sets posts.author_id = 5 (in memory)
$post->save();                     // persists

$post->author()->dissociate();     // sets posts.author_id = null (in memory)
$post->save();
```

### `push` — save the model *and* its loaded relations

```php
$user = User::with('posts')->find(1);
$user->name = 'New Name';
$user->posts->first()->title = 'Edited';

$user->push(); // saves the user AND every dirty, loaded related model
```

> `push()` only cascades to relations that are **already loaded**. It won't reach lazy relations you never accessed. (`pushQuietly()` does the same without firing model events.)

### Touching parent timestamps with `$touches`

When a child changes, you often want the parent's `updated_at` bumped (useful for cache invalidation). List the relation names in `$touches`:

```php
class Comment extends Model
{
    protected $touches = ['post'];

    public function post(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}
```

Now saving a `Comment` also updates its `Post`'s `updated_at`. You can also manually call `$post->touch()`.

---

## ⚠️ Common Mistakes & Gotchas

1. **The N+1 query bug.** Accessing a relation inside a loop without eager loading runs one query per iteration.
   *Fix:* eager load with `User::with('posts')->get()`, and enable `Model::preventLazyLoading()` in non-production so you catch it immediately.

2. **`with()` and `withCount()` are different things and you need both sometimes.** `with('posts')` loads the *posts*; `withCount('posts')` only gives you a number. Loading 10,000 posts just to call `->count()` is wasteful.
   *Fix:* use `withCount` when you only need the number; `with` when you need the records.

3. **Filtering with a closure inside `with()` does NOT filter the parents.** `User::with(['posts' => fn($q) => $q->where('published', true)])->get()` still returns *every* user — users with no published posts just get an empty `posts` collection.
   *Fix:* combine with `whereHas`, or use `withWhereHas` to filter parents and constrain the load in one shot.

4. **`attach()` creates duplicate pivot rows.** It blindly inserts; without a unique/composite key on the pivot you'll get duplicates, and even with one you'll get a constraint error.
   *Fix:* use `sync()` / `syncWithoutDetaching()` for idempotent updates (e.g. a roles multi-select form), and add a composite primary key on the pivot.

5. **Forgetting `withPivot`.** Extra pivot columns are invisible (`$role->pivot->is_primary` is `null`) unless you declare them with `->withPivot('is_primary')`.
   *Fix:* list every pivot column you read; add `->withTimestamps()` if the pivot has `created_at`/`updated_at`.

6. **Storing class names in `*_type` without a morph map.** Renaming or moving a model breaks every existing polymorphic row.
   *Fix:* call `Relation::enforceMorphMap([...])` from a service provider before you have production data.

7. **Constraining a relation method and reusing the cached property.** `$user->posts()->where(...)` runs a fresh query; `$user->posts` returns the cached, *unconstrained* collection. Mixing them confuses people who expect the cache to reflect the filter.
   *Fix:* be deliberate — method `()` for ad-hoc queries, property for the cached set.

---

## ✅ Best Practices

- **Type-hint relationship return types** (`: HasMany`, `: BelongsTo`, …). It documents intent and helps static analysis (PHPStan/Larastan) and IDEs.
- **Always use the `::class` constant**, never a string literal, for the related model.
- **Enable strict mode in development:** `Model::shouldBeStrict(! app()->isProduction())` in `AppServiceProvider::boot()` — it turns N+1, missing attributes, and silently-dropped fields into exceptions.
- **Eager load by default in controllers**; let `preventLazyLoading` keep you honest.
- **Prefer `withCount`/`withSum`/`withExists`** over loading whole relations just to aggregate.
- **Use `sync`/`syncWithoutDetaching`** for many-to-many form updates; reserve `attach`/`detach` for known single mutations.
- **Register a morph map** for every polymorphic relation, and back behavior-rich pivots with a `Pivot` model.
- **Use foreign-key constraints** in migrations (`->constrained()->cascadeOnDelete()`) so the database enforces integrity, not just your code.
- **Use `$touches`** to invalidate parent-based caches when children change.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is the N+1 problem and how do you fix it?**
A: Lazy loading a relation inside a loop runs one query per parent row (1 parent query + N child queries). Fix it by eager loading with `with()` (or `load()` after the fact), which uses a single `WHERE IN` query. Detect it by enabling `Model::preventLazyLoading()` in non-production, which throws a `LazyLoadingViolationException` on any lazy load.

**Q2. How does eager loading actually work under the hood?**
A: `with('posts')` runs the parent query first, collects the parent keys, then runs **one** child query `WHERE user_id IN (...)`. Eloquent then matches children to parents **in PHP** (an in-memory dictionary keyed by the foreign key) and hydrates each parent's relation. So it's 2 queries, not 1 big JOIN — which avoids row duplication and lets each model stay a clean object.

**Q3. Difference between `with()` and `load()`?**
A: `with()` is set on the *query* before execution (eager). `load()` is called on an already-retrieved model/collection (lazy eager loading) — same single-query efficiency, just deferred. `loadMissing()` only loads relations not already present.

**Q4. `whereHas` vs `with` vs `withWhereHas` vs `withCount`?**
A: `whereHas` filters parents by a constrained `EXISTS` subquery (doesn't load the relation). `with` loads the relation (doesn't filter parents). `withWhereHas` does both — filter parents and eager-load only matching children. `withCount` adds a count column via subquery without loading children.

**Q5. What's the foreign-key convention for `belongsTo` vs `hasMany`, and where does the FK live?**
A: For both, the foreign key is `<singular_parent>_id` and it lives on the **child** table. The difference is direction: `hasMany`/`hasOne` is the parent looking down; `belongsTo` is the child looking up. The pivot for `belongsToMany` defaults to the two model names singular, snake_case, alphabetical (`role_user`).

**Q6. `attach` vs `sync` vs `syncWithoutDetaching` vs `toggle`?**
A: `attach` adds rows (no dedupe). `sync` makes the pivot exactly match the given list (attaches missing, detaches extras, returns a changes array). `syncWithoutDetaching` attaches missing but never removes. `toggle` flips membership per ID. `updateExistingPivot` edits pivot columns on an existing row.

**Q7. How do polymorphic relationships store data, and why a morph map?**
A: They store a `*_id` and a `*_type`. `morphTo` reads `*_type` to know which model class to instantiate. A morph map aliases class names to short strings so the DB isn't coupled to your PHP namespace; `enforceMorphMap` additionally throws on unmapped models.

**Q8. When would you use `hasManyThrough`?**
A: When the parent reaches a distant relation through an intermediate model it *does* directly relate to — e.g. `Country` → `User` → `Post`. It saves you from manually joining or looping through the intermediate.

**Q9. Property access vs method call on a relationship — what's the difference?**
A: Property (`$user->posts`) executes the query once and caches the result (collection/model). Method (`$user->posts()`) returns the relationship/query-builder so you can chain constraints. The property is "lazy loaded once"; the method is "give me a fresh query."

**Q10. How do you keep a parent's `updated_at` fresh when a child changes?**
A: Add the relation name to the child's `$touches` array (e.g. `protected $touches = ['post'];`), or call `$parent->touch()` manually. Great for cache busting.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- DEFINITIONS ---
$this->hasOne(Profile::class);                       // 1:1 (FK on child)
$this->belongsTo(User::class);                       // inverse (FK on this table)
$this->hasMany(Post::class);                         // 1:many
$this->belongsToMany(Role::class)                    // many:many
    ->withPivot('is_primary')->withTimestamps()->using(Membership::class)->as('m');
$this->hasManyThrough(Post::class, User::class);     // distant 1:many
$this->hasOneThrough(History::class, User::class);   // distant 1:1
$this->morphTo();                                    // polymorphic owner
$this->morphMany(Comment::class, 'commentable');     // polymorphic 1:many
$this->morphOne(Image::class, 'imageable');          // polymorphic 1:1
$this->morphToMany(Tag::class, 'taggable');          // polymorphic m:m
$this->morphedByMany(Post::class, 'taggable');       // inverse polymorphic m:m

// --- READING ---
$user->posts;            // property: cached collection
$user->posts();          // method: query builder (chainable)
User::with('posts.comments')->get();                 // eager (nested)
$users->load('posts'); $users->loadMissing('posts'); // deferred eager
User::with(['posts' => fn ($q) => $q->where('published', true)])->get(); // constrained
$user->loadCount('posts');                           // posts_count, on demand

// --- EXISTENCE FILTERS ---
User::has('posts')->get();
User::has('posts', '>=', 3)->get();
User::whereHas('posts', fn ($q) => $q->where('published', true))->get();
User::doesntHave('posts')->get();
User::whereDoesntHave('posts', fn ($q) => $q->where('published', true))->get();
User::withWhereHas('posts', fn ($q) => $q->where('published', true))->get();
Comment::whereHasMorph('commentable', [Post::class], fn ($q) => $q->where('published', true))->get();
Comment::with(['commentable' => fn ($m) => $m->morphWith([Post::class => ['author']])])->get(); // morphTo eager

// --- AGGREGATES (subquery columns) ---
User::withCount('posts')->get();                 // posts_count
User::withSum('orders', 'total')->get();         // orders_sum_total
User::withAvg('orders', 'total')->get();         // orders_avg_total
User::withExists('posts')->get();                // posts_exists (bool)

// --- WRITING ---
$user->posts()->create([...]);                   // create child + set FK
$user->posts()->save($post); $user->posts()->saveMany([...]);
$post->author()->associate($user); $post->save();// belongsTo set
$post->author()->dissociate(); $post->save();    // belongsTo clear
$user->roles()->attach($id, ['is_primary' => true]);
$user->roles()->detach($id);                     // or detach() for all
$user->roles()->sync([1, 2, 3]);                 // exact match
$user->roles()->syncWithoutDetaching([4]);
$user->roles()->toggle([1, 2]);
$user->roles()->updateExistingPivot($id, ['is_primary' => false]);
$user->push();                                   // save model + loaded relations

// --- SAFETY (AppServiceProvider::boot) ---
Model::preventLazyLoading(! app()->isProduction());
Model::shouldBeStrict(! app()->isProduction());
Relation::enforceMorphMap(['post' => Post::class, 'video' => Video::class]);
```

---

## 🧪 Mini Exercises

1. **Blog schema.** Build `User`, `Post`, `Comment`, and `Tag` models. A user has many posts; a post has many comments; posts and comments are both taggable (many-to-many polymorphic). Write the migrations, the relationship methods (with typed return types), and a morph map.

2. **Kill the N+1.** Given a controller that lists 50 posts and prints each author's name and comment count, first reproduce the N+1 (count queries with `DB::enableQueryLog()` / Telescope), then rewrite it so the whole page runs in **3 queries or fewer**. Enable `preventLazyLoading` and confirm it no longer throws.

3. **Roles management form.** A user-edit form submits an array of selected role IDs plus an `is_primary` flag for one of them. Implement the save so the pivot ends up *exactly* matching the selection (with the correct primary flag) regardless of the previous state — using a single method call where possible.

4. **Distant relation.** Add a `Country` model and wire up `Country::posts()` via `hasManyThrough`. Then write a query that returns countries having at least 5 published posts, ordered by that count, in a single query.

5. **Latest-of-many.** Give `User` a `latestComment` relationship using `hasOne(...)->latestOfMany()` over its comments (through whatever path makes sense in your schema), and eager-load it on a user listing without triggering N+1.
