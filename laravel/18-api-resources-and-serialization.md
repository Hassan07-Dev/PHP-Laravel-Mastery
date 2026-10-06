# API Resources & Serialization

When you build a JSON API, the single most important question is: **how does an Eloquent model become the JSON your client receives?** Get this wrong and you leak password hashes, expose internal column names, return inconsistent shapes, and trigger N+1 query storms. Get it right and your API is predictable, secure, versionable, and a pleasure to consume.

This module takes you from the raw, automatic serialization that Eloquent gives you for free, all the way up to **API Resources** — Laravel's transformation layer that sits between your database models and your JSON responses. By the end you will know exactly what happens, byte by byte, when a model is turned into a response.

> Versions assumed: **PHP 8.4** and **Laravel 12**. Where Laravel 10/11 or PHP 8.1–8.3 behave differently, it is called out inline.

---

**What you'll learn**

- How Eloquent serializes models to arrays and JSON (`toArray`, `toJson`, `attributesToArray`) and what each step does.
- How `$hidden`, `$visible`, `makeHidden()`, `makeVisible()`, `$appends`, accessors, casts, and date formats shape the output.
- How to generate and write **API Resources** with `make:resource`, what `toArray(Request $request)` returns, and the Laravel 12 `toResource()` / `toResourceCollection()` shortcuts.
- **Resource collections** (`ResourceCollection`, `::collection()`, `$collects`, `$preserveKeys`) and how response **wrapping** (the `data` key, `withoutWrapping`, `$wrap`) works.
- **Conditional attributes**: `when`, `whenHas`, `whenNotNull`, `whenLoaded`, `whenCounted`, `whenAggregated`, `mergeWhen`, `whenPivotLoaded`, plus nesting and relationship loading.
- **Pagination**, adding **meta** with `additional()` and `with()`, and customizing the HTTP response with `withResponse`.
- When to use resources vs. returning models directly, and strategies for **API versioning**.

---

## 1. Why serialization matters (the WHY)

A "model" is a PHP object full of database columns, casts, relationships, timestamps, and internal state. JSON is a flat, language-agnostic text format. **Serialization** is the act of converting that object graph into a structure (array → JSON string) a client can parse.

Laravel gives you two completely different doors into serialization:

1. **Automatic Eloquent serialization** — return a model or collection from a controller and Laravel calls `toJson()` for you. Fast to write, but the JSON shape is coupled 1:1 to your database schema.
2. **API Resources** — an explicit transformation class where *you* decide the JSON shape. Decoupled from the schema, testable, versionable.

You must understand door #1 deeply, because API Resources are built *on top of* it. A `JsonResource` ultimately produces an array that Laravel serializes the same way.

---

## 2. Eloquent serialization fundamentals

### 2.1 `toArray()` and `toJson()`

Every Eloquent model can serialize itself:

```php
use App\Models\User;

$user = User::find(1);

$user->toArray();  // returns a PHP array
$user->toJson();   // returns a JSON string
(string) $user;    // also JSON — __toString() calls toJson()
```

Example output of `toArray()`:

```php
// Output:
[
    'id'         => 1,
    'name'       => 'Ada Lovelace',
    'email'      => 'ada@example.com',
    'created_at' => '2026-01-15T09:30:00.000000Z',
    'updated_at' => '2026-02-01T11:00:00.000000Z',
]
```

`toJson()` of the same model:

```json
{
    "id": 1,
    "name": "Ada Lovelace",
    "email": "ada@example.com",
    "created_at": "2026-01-15T09:30:00.000000Z",
    "updated_at": "2026-02-01T11:00:00.000000Z"
}
```

> Notice `password` is **not** there. The default `User` model ships with `$hidden = ['password', 'remember_token']`. More on that below.

### 2.2 The serialization pipeline (under the hood)

When you call `$model->toArray()`, Eloquent does this (in order):

1. **`attributesToArray()`** — serializes the model's own attributes (the columns):
   - Removes anything in `$hidden`, keeps only `$visible` if that array is non-empty.
   - Applies **casts** (e.g. `'is_active' => 'boolean'`, JSON casts, enum casts).
   - Converts dates (`$dates`, `created_at`, `updated_at`, and any `datetime` cast) using the model's date serialization format.
   - Runs **accessors** for any attribute listed in `$appends`.
2. **`relationsToArray()`** — serializes any **already-loaded** relationships, recursively calling their `toArray()`.
3. Merges the two and returns the combined array.

`toJson()` simply does `json_encode($this->jsonSerialize())`, and `jsonSerialize()` returns `toArray()`. So **JSON output is always derived from the array output.** If something is wrong in the array, it is wrong in the JSON.

Key takeaway: **only loaded relationships are serialized.** If you didn't `with()` or `load()` a relation, it won't appear — and accessing it inside a resource will trigger a lazy query (an N+1 risk). We return to this in the resources section.

### 2.3 `$hidden` and `$visible`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    // Never serialize these:
    protected $hidden = ['password', 'remember_token'];

    // OR: only ever serialize these (whitelist). Usually you use one or the other.
    // protected $visible = ['id', 'name'];
}
```

- `$hidden` is a **blacklist**; `$visible` is a **whitelist**.
- If `$visible` is non-empty, *only* those attributes survive — `$hidden` becomes redundant.
- These affect **serialization only**. `$user->password` still works in PHP; it's just omitted from `toArray()`/`toJson()`.

### 2.4 `makeHidden()` / `makeVisible()` — per-instance overrides

Sometimes you need to flip visibility for one request without editing the model:

```php
// Reveal a normally-hidden attribute for this instance:
$user->makeVisible('email_verified_at')->toArray();

// Hide an attribute just this once:
$user->makeHidden(['created_at', 'updated_at'])->toArray();

// Works on collections too:
$users->makeHidden('email');
```

The full family (all return the model for chaining):

| Method | Effect |
|---|---|
| `makeVisible($attrs)` | Reveal normally-hidden attributes |
| `makeHidden($attrs)` | Hide normally-visible attributes |
| `mergeVisible($attrs)` | Add to the `$visible` allow-list |
| `mergeHidden($attrs)` | Add to the `$hidden` block-list |
| `setVisible($attrs)` | **Replace** the entire `$visible` array |
| `setHidden($attrs)` | **Replace** the entire `$hidden` array |

Prefer `makeHidden`/`makeVisible` (or the `merge*` variants) for incremental changes; reach for `setHidden`/`setVisible` only when you truly want to discard the model's configured arrays.

### 2.5 Accessors and `$appends`

An **accessor** is a computed attribute. Modern (Laravel 9+) syntax uses the `Attribute` cast object:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) =>
                "{$attributes['first_name']} {$attributes['last_name']}",
        );
    }
}
```

By default, accessors are **not** serialized — they only run when you read `$user->full_name`. To include one in `toArray()`/`toJson()`, add it to `$appends`:

```php
protected $appends = ['full_name'];
```

```php
$user->toArray();
// Output now includes:
// 'full_name' => 'Ada Lovelace',
```

> Note the snake_case `full_name` in `$appends` maps to the camelCase `fullName()` accessor. Laravel handles the conversion automatically.

Appended attributes still **respect `$hidden`/`$visible`** — if `full_name` is in `$hidden` it won't appear even when appended.

You can append dynamically per instance, and there are several runtime helpers (all return the model):

```php
$user->append('full_name');                 // add one appended accessor
$user->mergeAppends(['full_name', 'rank']); // add several
$user->setAppends(['full_name']);           // replace the entire $appends array
$user->withoutAppends();                    // drop all appended accessors
$users->each->append('full_name');          // works across a collection
```

### 2.6 The effect of casts

Casts transform raw DB values into PHP types **and** drive serialization:

```php
use App\Enums\AccountStatus;

protected function casts(): array   // method form — Laravel 11+
{
    return [
        'is_active'   => 'boolean',
        'options'     => 'array',          // JSON column <-> PHP array
        'price'       => 'decimal:2',
        'status'      => AccountStatus::class,  // backed enum cast
        'verified_at' => 'datetime',
        'meta'        => 'collection',     // JSON <-> Illuminate Collection
    ];
}
```

- **`boolean`** → DB `1` becomes JSON `true`, not `"1"` or `1`.
- **`array` / `collection`** → a JSON DB column is decoded to a PHP array/Collection, then re-encoded in the output as a nested object/array.
- **`decimal:2`** → serializes as a **string** `"19.99"` (preserves precision; floats can't).
- **Backed enum cast** → serializes the enum's **`->value`**, e.g. `'active'`, not the enum object. (Requires PHP 8.1+ enums.)

> Laravel 11+ prefers the `protected function casts(): array` **method**. The older `protected $casts = [...]` **property** still works in Laravel 12 and is fine for static maps. The method form lets you use logic and is the modern idiom.

### 2.7 Date serialization format

By default Laravel serializes dates as **ISO-8601 with microseconds in UTC**:

```
2026-01-15T09:30:00.000000Z
```

This comes from `Model::serializeDate()`. Override it globally or per-model:

```php
use DateTimeInterface;

class User extends Model
{
    protected function serializeDate(DateTimeInterface $date): string
    {
        return $date->format('Y-m-d H:i:s');   // e.g. "2026-01-15 09:30:00"
    }
}
```

You can also set the serialized format **per attribute** via the cast, which overrides `serializeDate()` for just that column:

```php
protected function casts(): array
{
    return [
        'birthday'  => 'date:Y-m-d',          // "1815-12-10"
        'joined_at' => 'datetime:Y-m-d H:00', // "2026-01-15 09:00"
    ];
}
```

> Important distinction: changing `$dateFormat` affects how dates are **stored/parsed in the DB**, while `serializeDate()` (and the `date:`/`datetime:` cast formats) affect only how they appear in **JSON output**. Don't confuse the two.

---

## 3. API Resources: the transformation layer

### 3.1 Why not just return the model?

Returning a model directly couples your **public API contract** to your **private database schema**. Problems:

- Rename a column → every API client breaks.
- Add a sensitive column → it leaks unless you remember to hide it.
- You can't easily reshape (`first_name`+`last_name` → `name`), add links, or version the output.
- Conditional inclusion (admin-only fields, lazy-loaded relations) becomes messy.

**API Resources** give you an explicit, single-responsibility class that maps a model to its JSON representation. The model stays internal; the resource is the contract.

### 3.2 Generating a resource

```bash
php artisan make:resource UserResource
# -> app/Http/Resources/UserResource.php  (extends JsonResource)

# A dedicated collection class (two equivalent ways):
php artisan make:resource User --collection   # explicit flag -> UserCollection
php artisan make:resource UserCollection       # name ending in "Collection" is auto-detected
# -> app/Http/Resources/UserCollection.php  (extends ResourceCollection)
```

> Either the `--collection` flag **or** a class name ending in `Collection` tells Artisan to extend `ResourceCollection` instead of `JsonResource`.

### 3.3 `toArray(Request $request)`

The heart of a resource is `toArray()`. `$this` proxies to the underlying model (via `__get`), so `$this->id` reads the model's `id`.

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'joined_at'  => $this->created_at,   // serialized by Eloquent's date rules
        ];
    }
}
```

Use it in a controller:

```php
use App\Http\Resources\UserResource;
use App\Models\User;

public function show(User $user): UserResource
{
    return new UserResource($user);
}
```

Output (note the `data` wrapper — explained in §4):

```json
{
    "data": {
        "id": 1,
        "name": "Ada Lovelace",
        "email": "ada@example.com",
        "joined_at": "2026-01-15T09:30:00.000000Z"
    }
}
```

> A single `JsonResource` is `Responsable`, so returning it from a controller produces a proper `JsonResponse` (HTTP 200, `Content-Type: application/json`). You don't need `response()->json(...)`.

### 3.4 `toResource()` — the modern shortcut

Recent Laravel (11.x onward, with `#[UseResource]` and `toResourceCollection` rounded out in 12) added a `toResource()` method on models so you don't have to name the resource class at the call site. It discovers the matching resource by convention (`User` → `App\Http\Resources\UserResource`, searching the `Http\Resources` namespace closest to the model):

```php
public function show(User $user): UserResource
{
    return $user->toResource();              // discovers UserResource automatically
    // return $user->toResource(AdminUserResource::class);  // or be explicit
}
```

If your model's resource doesn't follow the naming convention, point to it once with the `#[UseResource]` attribute on the model:

```php
use App\Http\Resources\CustomUserResource;
use Illuminate\Database\Eloquent\Attributes\UseResource;

#[UseResource(CustomUserResource::class)]
class User extends Model { /* ... */ }
```

> `toResource()` is sugar for `new UserResource($user)` — identical output. Use whichever reads better; explicit `new UserResource(...)` is clearest when the resource isn't conventionally named.

---

## 4. Resource collections & response wrapping

### 4.1 `::collection()`

To transform a list, call the static `collection()` method:

```php
public function index(): \Illuminate\Http\Resources\Json\AnonymousResourceCollection
{
    return UserResource::collection(User::all());
}
```

Output:

```json
{
    "data": [
        { "id": 1, "name": "Ada Lovelace", "email": "ada@example.com", "joined_at": "..." },
        { "id": 2, "name": "Alan Turing",  "email": "alan@example.com", "joined_at": "..." }
    ]
}
```

`::collection()` returns an `AnonymousResourceCollection` that maps each item through `UserResource`.

Laravel 12 also offers the convention-based `toResourceCollection()` on Eloquent collections and paginators, mirroring `toResource()`:

```php
return User::all()->toResourceCollection();        // discovers UserCollection / UserResource
return User::paginate(15)->toResourceCollection(); // works on paginators too
```

It looks for a `UserCollection` first, then falls back to wrapping each model in `UserResource`. Like `#[UseResource]`, you can override discovery on the model with the `#[UseResourceCollection(CustomCollection::class)]` attribute.

### 4.2 Dedicated `ResourceCollection`

When you need collection-level metadata or custom logic, make a real collection class:

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    // Optionally pin the per-item resource (otherwise inferred):
    public $collects = UserResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,   // each item already mapped via UserResource
            'meta' => [
                'total_users' => $this->collection->count(),
                'generated_at' => now()->toIso8601String(),
            ],
        ];
    }
}
```

```php
return new UserCollection(User::all());
```

> Naming convention: if you have `UserResource`, a collection named `UserCollection` is auto-discovered by `UserResource::collection()` when it exists. Otherwise an anonymous collection is used.

> Preserving keys: when a collection is returned from a route, Laravel resets its keys to a sequential `0,1,2,...` (so it serializes as a JSON array). To keep the original keys (e.g. after `->keyBy('id')`, producing a JSON object), set `public $preserveKeys = true;` on the **item** resource class.

### 4.3 The `data` key and `withoutWrapping`

By default, **the top-level resource is wrapped in a `data` key**. This is deliberate: it leaves room to add sibling keys like `meta` and `links` without ambiguity, and it's the JSON:API convention.

To remove the wrapper globally, call `withoutWrapping()` — usually in a service provider's `boot()`:

```php
namespace App\Providers;

use Illuminate\Http\Resources\Json\JsonResource;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        JsonResource::withoutWrapping();
    }
}
```

Now `new UserResource($user)` returns the object **without** the `data` envelope.

You can also change the wrapper key per resource:

```php
class UserResource extends JsonResource
{
    public static $wrap = 'user';   // wraps in "user" instead of "data"
}
```

> Gotcha: wrapping is only applied at the **outermost** level. Nested resources are never double-wrapped. And `withoutWrapping()` is ignored when the resource also contains pagination links/meta (those need a wrapper).

---

## 5. Conditional attributes

Resources let you include keys **only when conditions hold**. The returned array is post-processed to drop "missing value" markers, so the keys vanish entirely (not set to `null`).

### 5.1 `when()`

```php
public function toArray(Request $request): array
{
    return [
        'id'    => $this->id,
        'name'  => $this->name,

        // Only present for admins:
        'email' => $this->when($request->user()?->isAdmin(), $this->email),

        // Lazy value via closure (only evaluated if condition is true):
        'secret_score' => $this->when(
            $request->user()?->isAdmin(),
            fn () => $this->calculateExpensiveScore(),
        ),
    ];
}
```

If the condition is false, the `email` key is **omitted** from the JSON — not rendered as `null`.

Two close relatives are driven by the attribute itself rather than an external condition:

```php
// Include only if the attribute is actually present on the model
// (e.g. you selected a subset of columns):
'name' => $this->whenHas('name'),

// Include only if the attribute is not null:
'avatar' => $this->whenNotNull($this->avatar),
```

### 5.2 `mergeWhen()`

Merge multiple keys conditionally:

```php
return [
    'id'   => $this->id,
    'name' => $this->name,
    $this->mergeWhen($request->user()?->isAdmin(), [
        'internal_notes' => $this->internal_notes,
        'risk_flag'      => $this->risk_flag,
    ]),
];
```

> Gotcha: `mergeWhen` (and `merge`) only work correctly with **string keys**. Don't use them inside a numerically-indexed array, or keys will be re-sequenced.

### 5.3 `whenLoaded()` — the relationship guard

`whenLoaded()` includes a relationship **only if it's already loaded**, preventing accidental lazy queries (N+1):

```php
use App\Http\Resources\PostResource;

return [
    'id'    => $this->id,
    'name'  => $this->name,

    // Included ONLY if posts were eager-loaded; no query triggered otherwise:
    'posts' => PostResource::collection($this->whenLoaded('posts')),

    // A single relation:
    'profile' => new ProfileResource($this->whenLoaded('profile')),
];
```

Controller side:

```php
return UserResource::collection(
    User::with('posts')->get()   // eager-load to make 'posts' appear
);
```

If you forget `with('posts')`, the `posts` key is simply omitted — no error, no extra query. This is the single most important habit for performant resource APIs.

Laravel 11/12 also provide count and aggregate guards:

```php
// Count: only present if you ran ->withCount('posts') or ->loadCount('posts').
'posts_count' => $this->whenCounted('posts'),

// Aggregates: signature is whenAggregated(relation, column, function).
// Populated by ->withSum('posts', 'words'), ->withAvg(...), etc.
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```

Note `whenCounted` reads the `{relation}_count` attribute; the JSON key you choose (`posts_count` above) is independent of it.

### 5.4 `whenPivotLoaded()` — many-to-many pivot data

For `belongsToMany` relations carrying pivot columns:

```php
// In the related model's resource (e.g. RoleResource):
return [
    'id'   => $this->id,
    'name' => $this->name,
    'assigned_at' => $this->whenPivotLoaded('role_user', fn () =>
        $this->pivot->created_at
    ),
];
```

The first argument is the **pivot table name**. If you use a [custom intermediate table model](https://laravel.com/docs/12.x/eloquent-relationships#defining-custom-intermediate-table-models), pass an instance of it instead of the table name:

```php
'assigned_at' => $this->whenPivotLoaded(new Membership, fn () => $this->pivot->created_at),
```

If you named the pivot accessor (`->as('membership')`), use `whenPivotLoadedAs('membership', 'role_user', fn () => $this->membership->created_at)` — the first argument is the accessor name, the second the pivot table.

### 5.5 Nesting resources

Resources compose naturally:

```php
class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'     => $this->id,
            'title'  => $this->title,
            'author' => new UserResource($this->whenLoaded('author')),
            'tags'   => TagResource::collection($this->whenLoaded('tags')),
        ];
    }
}
```

Nested resources are **not** wrapped in their own `data` key — only the outermost level is.

---

## 6. Pagination with resources

Pass a paginator straight into a resource collection and Laravel automatically adds `links` and `meta`:

```php
public function index(): \Illuminate\Http\Resources\Json\AnonymousResourceCollection
{
    return UserResource::collection(
        User::with('posts')->paginate(15)
    );
}
```

Output:

```json
{
    "data": [ /* 15 UserResource objects */ ],
    "links": {
        "first": "https://api.test/users?page=1",
        "last":  "https://api.test/users?page=8",
        "prev":  null,
        "next":  "https://api.test/users?page=2"
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 8,
        "path": "https://api.test/users",
        "per_page": 15,
        "to": 15,
        "total": 120
    }
}
```

- This works whether or not wrapping is disabled — pagination forces the `data` envelope.
- For infinite-scroll without a total count, use `cursorPaginate(15)` (efficient, no `COUNT(*)`) or `simplePaginate(15)` (prev/next only). Both serialize cleanly through resources. Note their `meta`/`links` differ: `cursorPaginate` has no `total`/`last_page` (only cursor-based `prev`/`next`), and `simplePaginate` omits `total`/`last_page` too.

### 6.1 Customizing the pagination `meta`/`links`

To reshape the auto-generated pagination block, define a `paginationInformation()` method on the resource. It receives the request, the raw `$paginated` array, and the `$default` array (containing the `links` and `meta` keys):

```php
public function paginationInformation($request, array $paginated, array $default): array
{
    $default['links']['custom'] = 'https://example.com';

    return $default;
}
```

---

## 7. Adding meta, links, and `with()`

### 7.1 Per-response meta with `additional()`

Attach top-level data at the call site without touching the resource class:

```php
return UserResource::collection($users)->additional([
    'meta' => [
        'request_id' => $request->header('X-Request-Id'),
        'version'    => 'v1',
    ],
]);
```

### 7.2 Class-level meta with `with()`

Define data that should *always* accompany the resource:

```php
class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return ['id' => $this->id, 'name' => $this->name];
    }

    public function with(Request $request): array
    {
        return [
            'meta' => [
                'api'       => 'users',
                'docs_url'  => 'https://docs.api.test/users',
            ],
        ];
    }
}
```

Output:

```json
{
    "data": { "id": 1, "name": "Ada Lovelace" },
    "meta": { "api": "users", "docs_url": "https://docs.api.test/users" }
}
```

> `with()` data is only added when the resource is the **outermost** (top-level) resource of the response, never for nested resources.

---

## 8. Customizing the HTTP response with `withResponse`

`toArray()` controls the **body**. To control **status code, headers, or cookies**, override `withResponse()`:

```php
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return ['id' => $this->id, 'name' => $this->name];
    }

    public function withResponse(Request $request, JsonResponse $response): void
    {
        $response->header('X-Resource-Type', 'user');
        $response->header('Cache-Control', 'private, max-age=60');
    }
}
```

To return a non-200 status (e.g. 201 on create), it's usually clearer to do it at the controller:

```php
return (new UserResource($user))
    ->response()
    ->setStatusCode(201);
```

`->response()` converts the resource into a mutable `JsonResponse` you can configure before returning.

---

## 9. Resources vs. returning models directly

| Concern | Return model directly | API Resource |
|---|---|---|
| Speed to write | Fastest | Small upfront cost |
| Schema coupling | Tight (column renames break clients) | Decoupled |
| Hiding fields | `$hidden` only | Full control + conditional |
| Conditional/admin fields | Awkward | `when`, `mergeWhen` |
| N+1 safety on relations | Manual | `whenLoaded` guards it |
| Versioning | Hard | Natural (per-version resource) |
| Meta / links / pagination shape | Manual | Built-in |

**Rule of thumb:** internal/throwaway endpoints or prototypes → returning models is fine. **Any public, long-lived, or client-facing API → always use resources.** The decoupling pays for itself the first time you rename a column.

---

## 10. API versioning strategies

Resources make versioning clean because each version gets its own transformation class.

**URI versioning** (most common, explicit):

```php
// routes/api.php
Route::prefix('v1')->group(fn () => require __DIR__.'/api_v1.php');
Route::prefix('v2')->group(fn () => require __DIR__.'/api_v2.php');
```

```
App\Http\Resources\V1\UserResource   // legacy shape
App\Http\Resources\V2\UserResource   // new shape, e.g. split name into first/last
```

The same `User` model feeds both; only the resource differs. You can deprecate `V1` later without touching the database.

**Header versioning** (e.g. `Accept: application/vnd.api.v2+json`) is also possible — branch to the right resource in the controller or via middleware. URI versioning is friendlier to debug and cache.

---

## ⚠️ Common Mistakes & Gotchas

1. **Triggering N+1 by not using `whenLoaded()`.**
   Writing `'posts' => PostResource::collection($this->posts)` lazily loads `posts` for every parent row.
   **Fix:** use `'posts' => PostResource::collection($this->whenLoaded('posts'))` and eager-load with `->with('posts')` in the controller. The relation appears only when loaded; no stray queries.

2. **Expecting conditional keys to render as `null`.**
   `when(false, $value)` does **not** produce `"key": null` — it removes the key entirely.
   **Fix:** if you genuinely want `null`, return `null` directly: `'x' => $cond ? $value : null`. Use `when()` only when you want the key *absent*.

3. **Forgetting that nested resources aren't wrapped, but top-level ones are.**
   People disable wrapping expecting it to vanish everywhere, then are surprised pagination still wraps in `data`.
   **Fix:** understand wrapping is outermost-only; pagination always forces a `data` envelope regardless of `withoutWrapping()`.

4. **Leaking sensitive attributes by returning the model directly.**
   `return $user;` serializes everything not in `$hidden`. Add a sensitive column to the DB later and it leaks silently.
   **Fix:** use a resource that whitelists fields explicitly, or at minimum keep `$hidden` rigorously up to date and prefer `$visible` whitelists for sensitive models.

5. **`mergeWhen()` with numeric keys.**
   Merging into a numerically-indexed array re-sequences keys and corrupts output.
   **Fix:** only use `mergeWhen`/`merge` with associative (string-keyed) arrays.

6. **Assuming accessors appear in JSON automatically.**
   An accessor returns a value when you read `$model->thing`, but it's invisible to `toArray()` unless appended.
   **Fix:** add it to `$appends`, or call `->append('thing')` on the instance/collection.

7. **Decimal precision via float casts.**
   Casting money to `'float'` introduces binary rounding errors in JSON (`19.989999...`).
   **Fix:** use `'decimal:2'` (serializes as a precise string `"19.99"`) or store/return integer cents.

---

## ✅ Best Practices

- **Always use resources for public APIs.** Treat the resource as your API contract, the model as internal state.
- **Whitelist, don't blacklist.** Explicitly list fields in `toArray()`; never spread the whole model.
- **Guard every relationship with `whenLoaded()`** and eager-load deliberately in the controller. This is your primary N+1 defense.
- **Keep one resource per model per version.** Put versioned resources in `Resources\V1`, `Resources\V2` namespaces.
- **Return `decimal:N` for money** (string output) or integer minor units; never floats.
- **Use `cursorPaginate()` for large/feed-style lists** to avoid expensive `COUNT(*)`.
- **Put always-present metadata in `with()`, per-call metadata in `additional()`.**
- **Set status codes via `->response()->setStatusCode()`** (e.g. 201 on store) rather than returning the bare resource.
- **Decide wrapping once, globally,** in `AppServiceProvider::boot()` and stick with it for consistency.
- **Test resources in isolation** — they're pure functions of (model, request), trivial to assert against.
- **Use `whenAggregated`/`whenCounted` for stats** rather than computing sums/counts in `toArray()` — keeps the work in SQL and guards against N+1.
- **`toResource()` is fine for brevity,** but keep it convention-driven; if you find yourself passing the class explicitly everywhere, prefer plain `new XResource(...)` for readability.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between returning an Eloquent model and an API Resource from a controller?**
A. A model auto-serializes via `toJson()`, coupling output 1:1 to the schema and honoring only `$hidden`/`$visible`/`$appends`. A resource is an explicit transformation class where you choose the shape, add conditional/computed fields, guard relationships, and version the contract independently of the database.

**Q2. (Under the hood) Walk me through what happens when `$model->toJson()` is called.**
A. `toJson()` calls `json_encode($this->jsonSerialize())`. `jsonSerialize()` returns `toArray()`. `toArray()` runs `attributesToArray()` (applies `$hidden`/`$visible`, casts, date serialization via `serializeDate()`, and `$appends` accessors) then `relationsToArray()` (serializes only **loaded** relations recursively), and merges. So JSON is always derived from the array form.

**Q3. How does `whenLoaded()` prevent N+1 queries?**
A. It checks `relationLoaded('name')` and returns a `MissingValue` if not loaded, which is stripped from the output. Because it never touches the relation's getter when absent, it can't trigger a lazy query. The relation only appears when you eager-loaded it.

**Q4. Why is the response wrapped in `data` by default, and how do you remove it?**
A. The `data` envelope reserves namespace for sibling keys (`meta`, `links`) and follows JSON:API convention. Remove it with `JsonResource::withoutWrapping()` in a provider, or change the key via `public static $wrap`. Note pagination always re-introduces the wrapper.

**Q5. What's the difference between `with()` and `additional()`?**
A. `with()` is defined on the resource class and always adds top-level data (when it's the outermost resource). `additional()` is called at the controller/call site for per-response data. Both only apply at the top level.

**Q6. How do `$hidden`/`$visible` differ from a resource's field selection?**
A. `$hidden`/`$visible` filter attributes globally at the model layer for *all* serialization. A resource selects fields explicitly per endpoint/version and can include computed or conditional fields the model can't express. They can be combined, but resources give finer, request-aware control.

**Q7. How does Laravel serialize dates, enums, and decimals, and how do you change them?**
A. Dates go through `serializeDate()` (default ISO-8601 UTC with microseconds); override the method to change format. Backed enums (via enum cast) serialize as their `->value`. `decimal:N` serializes as a precise string; floats risk rounding artifacts.

**Q8. How do you return a 201 with a resource on create?**
A. `return (new UserResource($user))->response()->setStatusCode(201);` — `->response()` yields a mutable `JsonResponse`. Headers/cookies can also be set via the resource's `withResponse(Request, JsonResponse)`.

**Q9. How do you include pivot data in a many-to-many resource?**
A. Use `whenPivotLoaded('pivot_table', fn () => $this->pivot->column)`, or `whenPivotLoadedAs('alias', 'pivot_table', ...)` if you set a custom pivot accessor with `->as()`.

**Q10. How would you version an API with resources?**
A. Namespace resources per version (`Resources\V1`, `Resources\V2`), route via URI prefixes (`/v1`, `/v2`) or an `Accept` header, and map the same models through the appropriate version's resource. The DB stays unchanged; only the transformation differs.

**Q11. (Laravel 12) What do `toResource()` and `toResourceCollection()` do?**
A. They're convenience methods on models, Eloquent collections, and paginators that locate the matching resource by naming convention (`User` → `UserResource`, `UserCollection`) and wrap the instance for you — so `$user->toResource()` equals `new UserResource($user)`. Override discovery with the `#[UseResource]` / `#[UseResourceCollection]` attributes on the model when the names don't follow convention.

**Q12. Difference between `whenLoaded`, `whenCounted`, and `whenAggregated`?**
A. All three drop their key when the underlying data isn't present, avoiding stray queries. `whenLoaded('posts')` includes a relation only if eager-loaded; `whenCounted('posts')` includes the `{relation}_count` value only if `withCount`/`loadCount` ran; `whenAggregated('posts', 'words', 'sum')` includes an aggregate only if `withSum`/`withAvg`/`withMin`/`withMax` populated it.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Eloquent serialization ---
$model->toArray();                  // array (loaded relations included)
$model->toJson();                   // JSON string  (toJson(JSON_PRETTY_PRINT) too)
$model->attributesToArray();        // attributes only, no relations
$model->makeHidden(['col']);        // hide for this instance
$model->makeVisible(['col']);       // reveal for this instance
$model->setHidden([...]);           // replace whole $hidden array
$model->append('accessor_attr');    // add accessor to output
$model->setAppends([...]);          // replace whole $appends array
protected $hidden   = ['password']; // blacklist
protected $visible  = ['id','name'];// whitelist
protected $appends  = ['full_name'];// accessors to serialize
protected function casts(): array { return ['active' => 'boolean']; } // L11+
protected function serializeDate(DateTimeInterface $d): string { return $d->format('Y-m-d'); }
```

```bash
php artisan make:resource UserResource         # JsonResource
php artisan make:resource User --collection     # ResourceCollection (UserCollection)
php artisan make:resource UserCollection        # ResourceCollection (name auto-detected)
```

```php
// --- Using resources ---
return new UserResource($user);                 // single, wrapped in "data"
return $user->toResource();                     // L12 shortcut (auto-discovers resource)
return UserResource::collection($users);        // many
return $users->toResourceCollection();          // L12 shortcut for collections
return UserResource::collection(User::paginate(15)); // + links + meta
return (new UserResource($user))->response()->setStatusCode(201);
return UserResource::collection($users)->additional(['meta' => [...]]);

// --- Conditional helpers (inside toArray) ---
$this->when($cond, $value);                     // include key only if $cond
$this->when($cond, fn () => expensive());       // lazy value
$this->whenHas('column');                       // only if attribute present on model
$this->whenNotNull($this->column);              // only if not null
$this->mergeWhen($cond, ['a' => 1, 'b' => 2]);  // merge several (string keys!)
$this->whenLoaded('relation');                  // only if eager-loaded
$this->whenCounted('relation');                 // only if withCount()/loadCount()
$this->whenAggregated('posts', 'words', 'sum'); // only if withSum() etc.
$this->whenPivotLoaded('pivot_table', fn () => $this->pivot->col);
new ChildResource($this->whenLoaded('child'));  // nested single
ChildResource::collection($this->whenLoaded('children')); // nested many

// --- Wrapping ---
JsonResource::withoutWrapping();   // in AppServiceProvider::boot()
public static $wrap = 'user';      // custom wrapper key

// --- Class-level meta / response ---
public function with(Request $r): array { return ['meta' => [...]]; }
public function withResponse(Request $r, JsonResponse $resp): void { $resp->header(...); }
```

---

## 🧪 Mini Exercises

1. **Schema decoupling.** Create a `ProductResource` that exposes `id`, a combined `display_name` (from `name` + `sku`), a `price` formatted as a `decimal:2` string, and `in_stock` as a boolean. Never expose the raw `cost` column. Verify `cost` is absent from the JSON.

2. **N+1 hunt.** Build a `PostResource` that nests an `AuthorResource` and a collection of `CommentResource`, both guarded by `whenLoaded()`. Write two controller actions — one that eager-loads `author` and `comments`, one that doesn't — and use the query log to confirm the second triggers zero relation queries (because the keys are omitted).

3. **Admin-only fields.** Extend `UserResource` so that `email` and an `internal_notes` block appear only when `$request->user()` is an admin. Use `when()` for the single field and `mergeWhen()` for the block. Confirm the keys disappear entirely (not `null`) for non-admins.

4. **Pagination + meta.** Return a paginated `UserResource::collection(User::cursorPaginate(10))` and attach an `additional(['meta' => ['version' => 'v1']])`. Inspect how cursor pagination's `meta`/`links` differ from offset pagination.

5. **Versioned output.** Create `Resources\V1\UserResource` (single `name` field) and `Resources\V2\UserResource` (`first_name` + `last_name`), route them under `/v1/users` and `/v2/users`, and confirm both read the same model while emitting different shapes.

6. **Laravel 12 shortcuts.** Replace `new UserResource($user)` with `$user->toResource()` and `UserResource::collection(User::all())` with `User::all()->toResourceCollection()`. Then rename `UserResource` to something off-convention and use the `#[UseResource]` attribute to make `toResource()` find it again.

7. **Aggregates without N+1.** Add `posts_count` via `whenCounted('posts')` and `comments_avg_rating` via `whenAggregated('comments', 'rating', 'avg')` to a `UserResource`. Load them with `User::withCount('posts')->withAvg('comments', 'rating')->get()` and confirm via the query log that no extra per-row queries fire and that the keys vanish when you omit the `with*` calls.
