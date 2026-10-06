# Practice Projects: Six Builds from Beginner to Capstone

Reading tutorials and watching videos creates the *illusion* of competence. The moment you face a blank `composer create-project` prompt, that illusion shatters. **Building** is the only way to convert passive recognition ("oh yeah, I remember `hasMany`") into active recall ("I need a one-to-many here, so I'll add `post_id` to `comments` and a `hasMany` on `Post`"). This module gives you six progressively harder project specs — each a self-contained brief you could hand to a junior dev — designed to be built during a 5-week interview prep sprint.

Each spec is deliberately **incomplete on implementation detail**. That's the point: a spec tells you *what* and *why*; you supply the *how*. That gap is exactly what an interviewer probes.

> **Target stack:** PHP 8.4 and Laravel 12. Where Laravel 10/11 or PHP 8.1–8.3 differ, it's called out inline.

---

## **What you'll learn**

- How to scope a project into a **data model**, a **route/API surface**, and a list of **Laravel concepts** it exercises — the same decomposition interviewers expect.
- Six concrete project specs covering CRUD, auth, many-to-many, REST APIs, authorization, file uploads, queues, scheduling, broadcasting, multi-tenancy, billing, and caching.
- A repeatable **how-to-approach-building** workflow (migrations first, tests as a safety net, vertical slices).
- The exact Laravel **artisan commands and code skeletons** to bootstrap each project quickly.
- Which **stretch goals** turn a "tutorial clone" into something that signals seniority.
- **What to demo in an interview** and how to talk about trade-offs you made.

---

## Why projects (and not more tutorials)

Interviewers rarely ask you to recite the definition of a service container. They ask: *"Walk me through a project you built. Why did you structure it that way? What would you change?"* A portfolio of 3–4 finished, deployed projects answers that question before it's even asked. The projects below are ordered so each one introduces **two or three genuinely new concepts** while reinforcing everything before it. Build them in order. Resist the urge to skip to the SaaS capstone — the capstone assumes you've internalised auth, policies, and queues from earlier builds.

A useful mental model for the whole module:

```
Project 1  CRUD + validation + token auth        (foundations)
Project 2  Blade UI + relationships + policies     (full-stack)
Project 3  Stateless REST + Sanctum + resources    (API design)
Project 4  Authorization + uploads + queues + cron (background work)
Project 5  Broadcasting + events + WebSockets       (real-time)
Project 6  Multi-tenancy + billing + caching + CI   (production craft)
```

---

## Project 1 — Todo / Task API (Beginner)

### Goal
A JSON API that lets an authenticated user create, read, update, and delete their own tasks. No frontend required — drive it entirely with `curl`, Postman, or `php artisan test`. The goal is to get migrations, Eloquent, validation, and **API token authentication** under your fingers.

### Features / user stories
- As a user, I can **register** and **log in** to receive an API token.
- As a user, I can **create** a task with a title, optional description, due date, and status.
- As a user, I can **list** my tasks, optionally filtered by status.
- As a user, I can **update** a task's fields and mark it complete.
- As a user, I can **delete** a task.
- A user can **only see and modify their own** tasks (no peeking at others').

### Laravel concepts exercised
Migrations, Eloquent models, `$fillable`/mass-assignment, `FormRequest` validation, route model binding, controllers (resource controllers), **Sanctum** token issuance, the `auth:sanctum` middleware, JSON responses, and a first taste of feature tests.

### Suggested data model

```
users
  id, name, email, password, timestamps

tasks
  id, user_id (FK -> users), title, description (nullable),
  status (enum: pending|in_progress|done, default pending),
  due_at (nullable datetime), timestamps
```

Relationships: `User hasMany Task`, `Task belongsTo User`.

Use a **native PHP enum** for status — it's the modern idiom and casts cleanly:

```php
<?php

enum TaskStatus: string
{
    case Pending = 'pending';
    case InProgress = 'in_progress';
    case Done = 'done';
}
```

```php
// app/Models/Task.php
class Task extends Model
{
    protected $fillable = ['title', 'description', 'status', 'due_at'];

    protected function casts(): array
    {
        return [
            'status' => TaskStatus::class,
            'due_at' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

> **Laravel 12 / 11 note:** the `casts()` *method* (introduced in Laravel 11) is preferred over the older `protected $casts = [...]` *property*. Both still work in Laravel 12; the method lets you reference enums and call functions.

### API / route list

```
POST   /api/register        -> issue token
POST   /api/login           -> issue token
POST   /api/logout          -> revoke current token   (auth:sanctum)
GET    /api/tasks           -> list current user's tasks (auth:sanctum)
POST   /api/tasks           -> create                    (auth:sanctum)
GET    /api/tasks/{task}    -> show                       (auth:sanctum)
PUT|PATCH /api/tasks/{task} -> update                     (auth:sanctum)
DELETE /api/tasks/{task}    -> delete                     (auth:sanctum)
```

> `apiResource` registers a single `update` route bound to **both** `PUT` and `PATCH` (and omits the `create`/`edit` HTML-form routes that the full `resource` macro adds).

Most of the CRUD lines collapse into one declaration:

```php
// routes/api.php
use App\Http\Controllers\TaskController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('tasks', TaskController::class);
});
```

> **Laravel 11/12 note:** `routes/api.php` is no longer present by default. Run `php artisan install:api` once — it publishes `routes/api.php`, installs Sanctum, and registers the `api` route file in `bootstrap/app.php` (via `->withRouting(api: ...)`). Laravel 12 ships *without* Sanctum until you run this command.

**Issuing a token** is the heart of this project. `createToken()` returns a `NewAccessToken`; its `plainTextToken` is the only time you'll ever see the un-hashed value (Sanctum stores a SHA-256 hash):

```php
// app/Http/Controllers/AuthController.php
public function register(Request $request): JsonResponse
{
    $data = $request->validate([
        'name'     => ['required', 'string', 'max:255'],
        'email'    => ['required', 'email', 'unique:users,email'],
        'password' => ['required', Password::defaults(), 'confirmed'],
    ]);

    $user = User::create([
        'name'     => $data['name'],
        'email'    => $data['email'],
        'password' => Hash::make($data['password']),   // NEVER store plaintext
    ]);

    return response()->json([
        'token' => $user->createToken('api')->plainTextToken,
    ], 201);
}

public function login(Request $request): JsonResponse
{
    $data = $request->validate([
        'email'    => ['required', 'email'],
        'password' => ['required'],
    ]);

    $user = User::where('email', $data['email'])->first();

    // generic message + hash check avoids leaking which field was wrong
    if (! $user || ! Hash::check($data['password'], $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    return response()->json([
        'token' => $user->createToken('api')->plainTextToken,
    ]);
}

public function logout(Request $request): JsonResponse
{
    $request->user()->currentAccessToken()->delete();   // revoke this token

    return response()->json(status: 204);
}
```

> **Security note:** always `Hash::make()` passwords (never store or compare plaintext), and return a **generic** "credentials are incorrect" error so you don't reveal whether an email exists. The `User` model casts `password` to `hashed` by default in Laravel 11/12, so `Hash::make()` is belt-and-suspenders if you set the attribute directly — but explicit hashing here keeps the example unambiguous.

A clean store action with a `FormRequest` and scoping to the current user:

```php
// app/Http/Requests/StoreTaskRequest.php
class StoreTaskRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title'       => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string'],
            'status'      => ['sometimes', Rule::enum(TaskStatus::class)],
            'due_at'      => ['nullable', 'date', 'after_or_equal:today'],
        ];
    }
}
```

```php
// app/Http/Controllers/TaskController.php
public function store(StoreTaskRequest $request): JsonResponse
{
    $task = $request->user()->tasks()->create($request->validated());

    return response()->json($task, 201);
}
```

A sample request and the expected response:

```bash
curl -X POST http://localhost:8000/api/tasks \
  -H "Authorization: Bearer 1|abcDEF..." \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"title":"Buy milk","due_at":"2026-06-20 09:00:00"}'
```

```json
// Output: HTTP 201 Created
{
  "id": 1,
  "user_id": 1,
  "title": "Buy milk",
  "description": null,
  "status": "pending",
  "due_at": "2026-06-20T09:00:00.000000Z",
  "created_at": "2026-06-18T12:00:00.000000Z",
  "updated_at": "2026-06-18T12:00:00.000000Z"
}
```

**Ownership enforcement** is the one trap here. Route model binding will happily load *any* task by id. Scope it:

```php
public function show(Task $task): JsonResponse
{
    abort_if($task->user_id !== request()->user()->id, 403);
    return response()->json($task);
}
```

A cleaner approach (preview of Project 2): a `TaskPolicy` + `$this->authorize('view', $task)`.

### Stretch goals
- Add `due_at` filtering and sorting (`?sort=-due_at`).
- Add **soft deletes** so `DELETE` archives instead of destroying (`SoftDeletes` trait + `deleted_at`).
- Add a `priority` enum and a "tasks due today" endpoint.
- Write feature tests covering the happy path and the 403 cross-user case.

---

## Project 2 — Blog with Blade UI (Intermediate)

### Goal
A server-rendered blog with a public reader experience and an authenticated authoring experience. Posts have **categories** (one-to-many) and **tags** (many-to-many), readers can **comment**, and only an author/admin can edit. This is your first full-stack Laravel app and your first real use of **policies** and **Blade**.

### Features / user stories
- Visitors can **browse** published posts, view a single post, and read comments.
- Visitors can **filter** posts by category or tag.
- Registered users can **comment** on a post.
- Authors can **create, edit, and delete their own posts** (draft/published states).
- Admins can edit or delete **any** post and moderate comments.
- Posts support a **slug** for clean URLs.

### Laravel concepts exercised
Blade templating (`@extends`, components, `@auth`, `@can`), **all relationship types** (`hasMany`, `belongsTo`, `belongsToMany`, polymorphic optional), pivot tables, **policies + gates**, `@can` directives in Blade, validation, flash messages/sessions, factories and seeders, and the built-in **Laravel Breeze** starter for auth scaffolding.

### Suggested data model

```
users        id, name, email, password, is_admin (bool, default false), timestamps
categories   id, name, slug, timestamps
posts        id, user_id (FK), category_id (FK), title, slug (unique),
             body (text), published_at (nullable), timestamps
tags         id, name, slug, timestamps
post_tag     post_id (FK), tag_id (FK)        -- pivot, composite PK
comments     id, post_id (FK), user_id (FK), body (text), timestamps
```

Relationships:
- `Post belongsTo User` (author) and `belongsTo Category`.
- `Post hasMany Comment`; `Comment belongsTo Post` and `belongsTo User`.
- `Post belongsToMany Tag` through `post_tag`; `Tag belongsToMany Post`.

```php
class Post extends Model
{
    public function author(): BelongsTo   { return $this->belongsTo(User::class, 'user_id'); }
    public function category(): BelongsTo  { return $this->belongsTo(Category::class); }
    public function comments(): HasMany     { return $this->hasMany(Comment::class); }
    public function tags(): BelongsToMany   { return $this->belongsToMany(Tag::class); }

    public function scopePublished(Builder $query): void
    {
        $query->whereNotNull('published_at')->where('published_at', '<=', now());
    }
}
```

Attaching tags from a request (sync replaces the whole set, perfect for an edit form):

```php
$post = $request->user()->posts()->create($validated);
$post->tags()->sync($request->input('tags', []));   // array of tag IDs
```

### Authorization with a policy

```php
// app/Policies/PostPolicy.php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->is_admin || $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $this->update($user, $post);
    }
}
```

> **Laravel 11/12 note:** policies are **auto-discovered** by naming convention (`Post` -> `PostPolicy`). The old `AuthServiceProvider::$policies` array is gone. Override discovery with `Gate::policy()` only if your naming is non-standard.

In Blade, guard the edit button:

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}" class="btn">Edit</a>
@endcan
```

In the controller:

```php
public function update(UpdatePostRequest $request, Post $post)
{
    $this->authorize('update', $post);   // throws 403 if denied
    $post->update($request->validated());
    $post->tags()->sync($request->input('tags', []));

    return redirect()->route('posts.show', $post)->with('status', 'Post updated.');
}
```

### Route list

```
GET    /                       posts.index    (published, paginated)
GET    /posts/{post:slug}       posts.show
GET    /categories/{category:slug}  category filter
GET    /tags/{tag:slug}          tag filter

// auth-only
GET    /dashboard               my posts
GET    /posts/create            posts.create
POST   /posts                   posts.store
GET    /posts/{post}/edit       posts.edit
PUT|PATCH /posts/{post}         posts.update
DELETE /posts/{post}            posts.destroy
POST   /posts/{post}/comments   comments.store
```

Note `{post:slug}` — Laravel resolves route model binding by the `slug` column instead of `id`.

A Blade layout component (Laravel 12 ships Blade components and the `<x-layout>` style):

```blade
{{-- resources/views/posts/show.blade.php --}}
<x-app-layout>
    <article class="prose">
        <h1>{{ $post->title }}</h1>
        <p class="meta">By {{ $post->author->name }} in
            <a href="{{ route('categories.show', $post->category) }}">{{ $post->category->name }}</a>
        </p>
        {!! nl2br(e($post->body)) !!}
    </article>

    <section>
        <h2>Comments ({{ $post->comments->count() }})</h2>
        @foreach ($post->comments as $comment)
            <div class="comment">
                <strong>{{ $comment->user->name }}</strong>
                <p>{{ $comment->body }}</p>
            </div>
        @endforeach

        @auth
            <form method="POST" action="{{ route('comments.store', $post) }}">
                @csrf
                <textarea name="body" required></textarea>
                <button>Post comment</button>
            </form>
        @else
            <p><a href="{{ route('login') }}">Log in</a> to comment.</p>
        @endauth
    </section>
</x-app-layout>
```

> Always escape user content with `{{ }}` (auto-escaped) rather than `{!! !!}` (raw). Use raw only for trusted HTML you control. Here `nl2br(e($post->body))` escapes first, then converts newlines.

### Stretch goals
- Add full-text search across post title/body.
- Add **eager loading** everywhere to kill N+1 queries (`Post::with(['author','category','tags'])`). Verify with Laravel Debugbar or `DB::listen`.
- Add a polymorphic `likes` relationship so both posts and comments can be liked.
- Add image upload for a post cover (preview of Project 4).
- Add markdown rendering for the body.

---

## Project 3 — Store REST API (Intermediate)

### Goal
A **stateless JSON API** for an e-commerce backend: browse products, manage a cart, and place orders. No Blade — this is pure API design. The headline concepts are **Sanctum** token auth, **API Resources** for shaping output, **pagination**, and **search/filter** query parameters.

### Features / user stories
- Anyone can browse and search products with pagination and filters (category, price range, in-stock).
- Authenticated users have a **cart**; they can add, update quantity, and remove items.
- Users can **checkout** the cart into an **order**, which snapshots prices and decrements stock.
- Users can list their past orders and view a single order.
- Responses are consistently shaped via API Resources (no leaking of internal columns).

### Laravel concepts exercised
Sanctum, **API Resource & Resource Collections**, conditional resource attributes, pagination (`paginate()` + resource meta), query scopes for filtering, `when()`-based conditional queries, database transactions, and decimal/money handling.

### Suggested data model

```
products    id, name, slug, description, price_cents (int), stock (int),
            category_id (FK), is_active (bool), timestamps
categories  id, name, slug, timestamps
carts       id, user_id (FK), timestamps
cart_items  id, cart_id (FK), product_id (FK), quantity (int), timestamps
orders      id, user_id (FK), status (enum), total_cents (int), timestamps
order_items id, order_id (FK), product_id (FK), quantity,
            unit_price_cents (int)   -- price snapshot at purchase time
```

> **Money tip:** store money as **integer cents** (`price_cents`), never as a float. Floats lose precision (`0.1 + 0.2 !== 0.3`). Format to dollars only at the presentation edge.

### API Resources — why and how
A model returned directly leaks every column and every future column you add. An **API Resource** is a transformer: a stable contract between your DB and your JSON.

```php
// app/Http/Resources/ProductResource.php
class ProductResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'        => $this->id,
            'name'      => $this->name,
            'slug'      => $this->slug,
            'price'     => $this->price_cents / 100,   // present as dollars
            'in_stock'  => $this->stock > 0,
            'category'  => CategoryResource::make($this->whenLoaded('category')),
            // only included when caller is admin
            'stock'     => $this->when($request->user()?->is_admin, $this->stock),
        ];
    }
}
```

`whenLoaded('category')` only includes the category if it was eager-loaded, preventing N+1. `when(...)` conditionally hides fields.

> **Resource auth gotcha:** `$request->user()?->is_admin` relies on the request being authenticated. On a **public** product endpoint with no Sanctum middleware, `$request->user()` is `null`, so the `stock` field is correctly hidden — good. But never put an authorization *decision* (like "can this user act on the record") inside a Resource; Resources shape output, **policies** decide access. Keep the two concerns separate.

### Filtering, searching, pagination
Build the query incrementally with `when()` so each filter is optional:

```php
public function index(Request $request)
{
    $products = Product::query()
        ->where('is_active', true)
        ->when($request->filled('search'), fn ($q) =>
            $q->where('name', 'like', '%'.$request->string('search').'%'))
        ->when($request->filled('category'), fn ($q) =>
            $q->whereRelation('category', 'slug', $request->string('category')))
        ->when($request->filled('min_price'), fn ($q) =>
            $q->where('price_cents', '>=', $request->integer('min_price') * 100))
        ->when($request->boolean('in_stock'), fn ($q) =>
            $q->where('stock', '>', 0))
        ->with('category')
        ->paginate($request->integer('per_page', 15));

    return ProductResource::collection($products);
}
```

`ProductResource::collection($paginator)` automatically attaches pagination `meta` and `links`:

```json
// GET /api/products?search=shirt&in_stock=1&per_page=2
// Output: HTTP 200
{
  "data": [
    { "id": 4, "name": "Blue Shirt", "slug": "blue-shirt", "price": 29.99, "in_stock": true },
    { "id": 7, "name": "Red Shirt",  "slug": "red-shirt",  "price": 24.5,  "in_stock": true }
  ],
  "links": { "first": "...?page=1", "last": "...?page=9", "prev": null, "next": "...?page=2" },
  "meta": { "current_page": 1, "from": 1, "last_page": 9, "per_page": 2, "to": 2, "total": 18 }
}
```

> **`string()` / `integer()` / `boolean()` note:** these typed request accessors (`$request->string()`, `->integer()`, `->boolean()`) return `Stringable`/`int`/`bool` and are safer than raw `$request->input()`. Available since Laravel 9/10 and the recommended idiom in 12.

> **`LIKE` gotcha:** Eloquent bindings protect you from SQL *injection* here, but a user-supplied `%` or `_` is still interpreted as a wildcard, and a leading `%...%` can't use a normal B-tree index (it forces a full scan). For real catalog search, escape user wildcards or — better — reach for a full-text index (`whereFullText`) or a search engine (Laravel **Scout** + Meilisearch/Typesense). The `like` filter is fine to start, but name this trade-off in an interview.

### Checkout inside a transaction
Checkout touches multiple tables and must be atomic — if stock can't be decremented, nothing should commit:

```php
public function checkout(Request $request)
{
    $cart = $request->user()->cart()->with('items.product')->firstOrFail();

    $order = DB::transaction(function () use ($request, $cart) {
        $order = $request->user()->orders()->create([
            'status'      => OrderStatus::Pending,
            'total_cents' => 0,
        ]);

        $total = 0;
        foreach ($cart->items as $item) {
            $product = $item->product;

            // lockForUpdate prevents overselling under concurrency
            $fresh = Product::whereKey($product->id)->lockForUpdate()->first();
            abort_if($fresh->stock < $item->quantity, 422, "Out of stock: {$fresh->name}");
            $fresh->decrement('stock', $item->quantity);

            $order->items()->create([
                'product_id'       => $product->id,
                'quantity'         => $item->quantity,
                'unit_price_cents' => $product->price_cents,
            ]);
            $total += $product->price_cents * $item->quantity;
        }

        $order->update(['total_cents' => $total]);
        $cart->items()->delete();
        return $order;
    });

    return OrderResource::make($order->load('items.product'))
        ->response()->setStatusCode(201);
}
```

### Route list

```
GET    /api/products              list + search + filter + paginate
GET    /api/products/{product}    show
GET    /api/cart                  view cart            (auth:sanctum)
POST   /api/cart/items            add item             (auth:sanctum)
PATCH  /api/cart/items/{item}     update quantity      (auth:sanctum)
DELETE /api/cart/items/{item}     remove               (auth:sanctum)
POST   /api/checkout              cart -> order        (auth:sanctum)
GET    /api/orders                list my orders       (auth:sanctum)
GET    /api/orders/{order}        show my order        (auth:sanctum)
```

### Stretch goals
- Add **rate limiting** to the public product endpoints (`throttle:60,1`).
- Add cursor pagination (`cursorPaginate()`) for large catalogs — faster than offset pagination.
- Add a coupon/discount system applied at checkout.
- Version the API (`/api/v1/...`) and document it with an OpenAPI spec or Scribe.

---

## Project 4 — Job Board / Multi-Role App (Advanced)

### Goal
A job board where **employers** post jobs and **candidates** apply by uploading a resume. This is the project where **background work** becomes essential: emailing employers on a new application, processing/scanning uploads, and a **nightly scheduled job** that closes expired listings. Headline concepts: role-based authorization, **file uploads to storage**, **notifications**, **queues**, and the **scheduler**.

### Features / user stories
- Users register as either **employer** or **candidate** (a role).
- Employers can post, edit, and close job listings.
- Candidates can browse jobs and **apply**, uploading a PDF resume.
- On a new application, the employer receives a **notification** (mail + database).
- Resume processing (virus scan stub, thumbnail, or text extraction) runs on a **queue**.
- A **scheduled task** runs nightly to close listings past their deadline and purge orphaned resume files.

### Laravel concepts exercised
Multiple guards/roles, **policies driven by role**, `Storage` facade and the filesystem (local/S3), **validated file uploads**, **queued jobs** (`ShouldQueue`), **notifications** (mail + database channels), **events & listeners**, the **task scheduler** (`routes/console.php`), and queue workers.

### Suggested data model

```
users         id, name, email, password, role (enum: employer|candidate), timestamps
companies     id, user_id (FK -> employer), name, website, timestamps
jobs          id, company_id (FK), title, description, location,
              salary_cents, closes_at (datetime), status (open|closed), timestamps
applications  id, job_id (FK), candidate_id (FK -> users), resume_path,
              cover_letter (text), status (enum), timestamps
notifications id, type, notifiable_type, notifiable_id, data (json), read_at  -- Laravel's table
```

> The **database** notification channel needs a `notifications` table. Generate the migration and run it: `php artisan notifications:table && php artisan migrate` (`notifications:table` is an alias of `make:notifications-table`).

> **Naming caution:** an Eloquent model called `Job` reads ambiguously next to queue **jobs** (`App\Jobs\*`, the `jobs` queue table). It works, but consider naming the domain model `Listing` or `JobPosting` to avoid confusing yourself and reviewers. The examples below keep `Job` to match the spec.

### File upload, validated and stored

```php
// app/Http/Requests/StoreApplicationRequest.php
public function rules(): array
{
    return [
        'cover_letter' => ['nullable', 'string', 'max:5000'],
        'resume'       => ['required', 'file', 'mimes:pdf', 'max:5120'], // 5 MB
    ];
}
```

```php
public function store(StoreApplicationRequest $request, Job $job)
{
    $this->authorize('apply', $job);   // candidate-only, job must be open

    // store on the 'private' disk; returns a path like applications/ab12.pdf
    $path = $request->file('resume')->store('applications', 'private');

    $application = $job->applications()->create([
        'candidate_id' => $request->user()->id,
        'cover_letter' => $request->input('cover_letter'),
        'resume_path'  => $path,
    ]);

    // dispatch heavy work to a queue; respond immediately
    ProcessResume::dispatch($application);

    // notify the employer (queued automatically if notification ShouldQueue)
    $job->company->owner->notify(new NewApplicationReceived($application));

    return response()->json($application, 201);
}
```

> **Never** store uploads under `public/` if they're private (resumes!). Use a non-public disk and serve them through a controller that runs a policy check, e.g. `return Storage::disk('private')->download($app->resume_path)`.

### A queued job

```php
// app/Jobs/ProcessResume.php
class ProcessResume implements ShouldQueue
{
    use Queueable;

    public function __construct(public Application $application) {}

    public function handle(): void
    {
        // e.g. extract text, generate a preview, run an AV scan stub
        $text = $this->extractText($this->application->resume_path);
        $this->application->update(['parsed_text' => $text]);
    }
}
```

`ProcessResume::dispatch($application)` pushes onto the queue; a worker (`php artisan queue:work`) executes `handle()` out-of-band. Choose a driver in `.env`:

```env
QUEUE_CONNECTION=database
# alternatives: redis (production), sync (runs inline — handy for local tests)
```

> **`sync` gotcha:** the default in a fresh app may be `sync`, which runs jobs *immediately and synchronously* — your "async" code isn't async at all. Set `QUEUE_CONNECTION=database` (and run the worker) to actually observe queueing.

### A notification on two channels

```php
class NewApplicationReceived extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(public Application $application) {}

    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('New application for '.$this->application->job->title)
            ->line($this->application->candidate->name.' has applied.')
            ->action('Review', url('/jobs/'.$this->application->job_id.'/applications'));
    }

    public function toArray(object $notifiable): array
    {
        return [
            'application_id' => $this->application->id,
            'job_title'      => $this->application->job->title,
        ];
    }
}
```

### Scheduled cleanup
In Laravel 11/12 the scheduler lives in `routes/console.php` (the old `app/Console/Kernel.php@schedule` was removed):

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;
use App\Models\Job;

Schedule::call(function () {
    Job::where('status', 'open')
       ->where('closes_at', '<', now())
       ->update(['status' => 'closed']);
})->dailyAt('02:00')->name('close-expired-jobs')->withoutOverlapping();
```

One cron entry runs the whole scheduler in production:

```bash
* * * * * cd /path/to/app && php artisan schedule:run >> /dev/null 2>&1
```

> Locally you don't need cron — run `php artisan schedule:work` (a long-running process that triggers due tasks every minute).

### Route list (abridged)

```
POST   /api/jobs                       employer creates a job
PATCH  /api/jobs/{job}                  employer updates
POST   /api/jobs/{job}/close            employer closes
GET    /api/jobs                        public listing
POST   /api/jobs/{job}/applications     candidate applies (file upload)
GET    /api/jobs/{job}/applications     employer views applicants (policy)
GET    /api/applications/{app}/resume   secure resume download (policy)
GET    /api/notifications               current user's notifications
POST   /api/notifications/{id}/read     mark read
```

### Stretch goals
- Move uploads to **S3** (just change the disk; no app code changes — that's the point of the filesystem abstraction).
- Add **failed job** handling (`failed_jobs` table, `queue:retry`, a `failed()` method).
- Add job **batching** to process many resumes with a completion callback.
- Add **Laravel Horizon** if using Redis, for a queue dashboard.

---

## Project 5 — Real-Time Chat / Notifications (Advanced)

### Goal
Add live updates with **broadcasting**: a chat room where messages appear instantly for all participants, or live notification toasts. Concepts: **events**, **broadcasting** over WebSockets via **Laravel Reverb** (the first-party WebSocket server) and **Laravel Echo** on the client, **private/presence channels**, and queues to offload broadcast work.

### Why broadcasting
HTTP is request/response: the server can't *push* to a browser. Polling (`setInterval`-fetch) is wasteful and laggy. **WebSockets** keep a persistent connection so the server pushes events the instant they happen. Laravel's broadcasting layer lets you fire a normal PHP **event** and have it transparently shipped to subscribed browsers.

> **Reverb note:** Laravel **Reverb** is Laravel's official, self-hosted WebSocket server, introduced in Laravel 11 and the **recommended first-party** broadcaster in 12. Broadcasting is *not* enabled out of the box — run `php artisan install:broadcasting`, which **prompts** you for a broadcaster; choose Reverb and it runs `reverb:install`, publishes `config/broadcasting.php` and `config/reverb.php`, creates `routes/channels.php`, installs the `laravel-echo` + `pusher-js` npm packages, and writes the Echo scaffolding into `resources/js/echo.js`. Pusher (hosted) and Ably remain alternatives the same prompt offers.

### Suggested data model

```
rooms          id, name, timestamps
room_user      room_id (FK), user_id (FK)            -- pivot, who's in a room
messages       id, room_id (FK), user_id (FK), body, timestamps
```

### The flow end to end

1. User POSTs a message.
2. Controller persists it and fires an event.
3. The event implements `ShouldBroadcast` and is queued.
4. Reverb pushes it to everyone subscribed to that room's private channel.
5. Echo (browser) receives it and appends to the DOM.

```php
// app/Events/MessageSent.php
class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Message $message) {}

    public function broadcastOn(): PrivateChannel
    {
        return new PrivateChannel('room.'.$this->message->room_id);
    }

    public function broadcastWith(): array
    {
        return [
            'id'   => $this->message->id,
            'body' => $this->message->body,
            'user' => $this->message->user->only('id', 'name'),
            'at'   => $this->message->created_at->toIso8601String(),
        ];
    }
}
```

```php
// controller
public function store(Request $request, Room $room)
{
    $message = $room->messages()->create([
        'user_id' => $request->user()->id,
        'body'    => $request->validate(['body' => 'required|string|max:2000'])['body'],
    ]);

    broadcast(new MessageSent($message))->toOthers(); // don't echo back to sender

    return response()->json($message, 201);
}
```

> `->toOthers()` excludes the originating socket so the sender doesn't get a duplicate (they already added it optimistically). It requires the `X-Socket-ID` header that Echo sends automatically.

**Channel authorization** — only members of a room may subscribe:

```php
// routes/channels.php
Broadcast::channel('room.{roomId}', function (User $user, int $roomId) {
    return $user->rooms()->whereKey($roomId)->exists();
});
```

**Client (Echo + Reverb):**

```js
// resources/js/app.js imports the Echo config that install:broadcasting wrote
// to resources/js/echo.js; once bootstrapped, window.Echo is available:
window.Echo.private(`room.${roomId}`)
    .listen('MessageSent', (e) => {
        appendMessage(e); // e.body, e.user.name, e.at
    });
```

> **Event name note:** `.listen('MessageSent', ...)` matches the event's class name. If you implement `broadcastAs()` to broadcast under a custom name, prefix the listener with a dot — e.g. `.listen('.message.sent', ...)` — so Echo doesn't apply its default namespace.

```env
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=local
REVERB_APP_KEY=local-key
REVERB_APP_SECRET=local-secret
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http
```

Run three processes locally:

```bash
php artisan reverb:start      # the WebSocket server
php artisan queue:work        # so ShouldBroadcast events actually broadcast
php artisan serve             # the app
```

> **Queue gotcha:** `ShouldBroadcast` events are *queued* by default. If `QUEUE_CONNECTION` isn't running a worker (and isn't `sync`), your messages persist to the DB but never appear live. Either run `queue:work` or implement `ShouldBroadcastNow` to broadcast synchronously.

### Presence channels (stretch)
A **presence channel** is a private channel that also tracks *who is currently subscribed* — perfect for "3 people online" or typing indicators. Authorize by returning user data:

```php
Broadcast::channel('room.{roomId}', function (User $user, int $roomId) {
    if ($user->rooms()->whereKey($roomId)->exists()) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```

```js
window.Echo.join(`room.${roomId}`)
    .here(users => renderOnlineList(users))
    .joining(user => addOnline(user))
    .leaving(user => removeOnline(user))
    .listenForWhisper('typing', e => showTyping(e.name));
```

### Route list

```
GET    /api/rooms                  rooms I'm in
POST   /api/rooms/{room}/messages  send a message (broadcasts)
GET    /api/rooms/{room}/messages  message history (paginated)
```

### Stretch goals
- Typing indicators via **client events** (`whisper`).
- Read receipts using a separate event.
- Deploy Reverb behind a reverse proxy with TLS (`wss://`).
- Replace chat with a generic **live notification** feed reusing Project 4's notifications via the `broadcast` channel.

---

## Project 6 — SaaS Capstone: Teams, Roles, Billing, Caching (Capstone)

### Goal
A multi-tenant SaaS skeleton: users belong to **teams**, teams have members with **roles/permissions**, a **billing stub** gates premium features, hot paths are **cached**, and the whole thing is covered by **tests** with a documented **deployment checklist**. This is the capstone — it integrates everything and is the project you lead with in interviews.

### Features / user stories
- A user signs up and gets a personal team; they can create more teams.
- A team owner can **invite** members and assign **roles** (owner, admin, member).
- Permissions gate actions (only admins+ can invite; only owners can delete the team).
- The team has a **plan** (free/pro); pro-only features are gated by a billing check.
- A **billing stub** simulates subscribe/cancel (or wire up Laravel Cashier + Stripe test mode).
- Expensive dashboard stats are **cached** and invalidated on write.
- The codebase has feature + unit tests and a CI pipeline.

### Laravel concepts exercised
Multi-tenancy / team scoping, **roles & permissions** (hand-rolled or `spatie/laravel-permission`), **gates with team context**, **Cashier** (or a stub), the **cache** (`Cache::remember`, tags), **rate limiting per plan**, **Pest/PHPUnit** feature tests, **CI** (GitHub Actions), and a deployment checklist.

### Suggested data model

```
users           id, name, email, password, current_team_id (FK), timestamps
teams           id, owner_id (FK), name, plan (enum: free|pro), timestamps
team_user       team_id (FK), user_id (FK), role (enum: owner|admin|member)  -- pivot w/ extra col
invitations     id, team_id (FK), email, role, token, accepted_at, timestamps
subscriptions   id, team_id (FK), plan, status, current_period_end           -- or Cashier's tables
projects        id, team_id (FK), name, ...                                  -- tenant-scoped data
```

The pivot carries an extra `role` column — access it via `withPivot`:

```php
class Team extends Model
{
    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
                    ->withPivot('role')
                    ->withTimestamps();
    }
}
```

### Tenant scoping with a global scope
Every tenant-scoped query should be automatically filtered to the current team. A **global scope** + a trait is the clean way:

```php
class BelongsToTeamScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if ($teamId = auth()->user()?->current_team_id) {
            $builder->where($model->getTable().'.team_id', $teamId);
        }
    }
}
```

```php
class Project extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope(new BelongsToTeamScope);
        static::creating(fn ($p) => $p->team_id ??= auth()->user()->current_team_id);
    }
}
```

> **Tenancy gotcha:** global scopes can be silently bypassed in jobs/commands where there's no authenticated user. For multi-tenant SaaS at scale, consider a dedicated package (`stancl/tenancy`) or explicit scoping in background jobs.

### Roles & permissions via gates

```php
// a Gate defined in a service provider or AppServiceProvider::boot()
Gate::define('invite-members', function (User $user, Team $team) {
    $role = $team->members()->find($user->id)?->pivot->role;
    return in_array($role, ['owner', 'admin'], true);
});
```

```php
if (Gate::denies('invite-members', $team)) {
    abort(403);
}
```

For anything beyond a handful of permissions, reach for **`spatie/laravel-permission`** — it gives you `$user->assignRole()`, `$user->can('edit posts')`, and DB-backed roles. Mention this package by name in interviews; it's the de-facto standard.

### Caching expensive reads

```php
public function dashboardStats(Team $team): array
{
    return Cache::remember("team:{$team->id}:stats", now()->addMinutes(10), function () use ($team) {
        return [
            'projects'   => $team->projects()->count(),
            'members'    => $team->members()->count(),
            'open_tasks' => $team->projects()->withCount('openTasks')->get()->sum('open_tasks_count'),
        ];
    });
}
```

Invalidate on write so stale data doesn't linger:

```php
// in the Project model's saved/deleted observer
Cache::forget("team:{$project->team_id}:stats");
```

> **Cache invalidation gotcha:** time-based expiry (TTL) alone shows stale numbers for up to 10 minutes. Pair TTL with explicit `forget()` on writes. If your driver supports tags (Redis/Memcached), `Cache::tags("team:{$id}")->flush()` invalidates a whole group at once. The `file`/`database` cache drivers do **not** support tags.

### Billing stub vs Cashier
A **stub** is fine for a portfolio piece — a `plan` column plus a gate:

```php
Gate::define('use-pro-feature', fn (User $u) => $u->currentTeam->plan === 'pro');
```

To show real-world depth, wire **Laravel Cashier (Stripe)** in **test mode**:

```bash
composer require laravel/cashier
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

```php
$team->newSubscription('default', 'price_pro_monthly')
     ->create($paymentMethodId);   // pass a PaymentMethod id; test value: pm_card_visa
```

> **Cashier note:** modern Cashier uses Stripe **PaymentMethod** ids (`pm_...`), not the legacy card *tokens* (`tok_visa`). In tests use `pm_card_visa`. The billable model (here `Team`) needs the `Laravel\Cashier\Billable` trait. Cashier also requires a `STRIPE_KEY`/`STRIPE_SECRET` in `.env` and a webhook endpoint (`cashier:webhook`) to stay in sync with Stripe.

### Testing — the part that signals seniority

```php
// tests/Feature/InvitationTest.php (Pest)
it('lets an admin invite a member but blocks a regular member', function () {
    $team   = Team::factory()->create();
    $admin  = User::factory()->create();
    $member = User::factory()->create();
    $team->members()->attach($admin->id,  ['role' => 'admin']);
    $team->members()->attach($member->id, ['role' => 'member']);

    actingAs($admin)
        ->post("/teams/{$team->id}/invitations", ['email' => 'new@x.test', 'role' => 'member'])
        ->assertCreated();

    actingAs($member)
        ->post("/teams/{$team->id}/invitations", ['email' => 'no@x.test', 'role' => 'member'])
        ->assertForbidden();
});
```

```bash
php artisan test
# Output:
#   PASS  Tests\Feature\InvitationTest
#   ✓ it lets an admin invite a member but blocks a regular member
#   Tests: 1 passed
```

This test hits the database (`Team::factory()`, `actingAs()->post(...)`), so it needs `RefreshDatabase` to roll back between tests. In Pest you wire that once, globally, instead of per-file:

```php
// tests/Pest.php
use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->use(RefreshDatabase::class)->in('Feature');
```

> **Laravel 12 note:** new apps ship with **Pest** as the default test runner (PHPUnit still underneath). The `it()`/`test()` functions and helpers like `actingAs()` are Pest syntax. PHPUnit class-based tests work identically — there you'd add `use RefreshDatabase;` as a trait on the test class instead.

### Deployment checklist

```
[ ] APP_ENV=production, APP_DEBUG=false, strong APP_KEY set
[ ] composer install --no-dev --optimize-autoloader
[ ] php artisan optimize   (caches config + routes + events + views in one go)
[ ] php artisan migrate --force
[ ] queue worker supervised (Supervisor/systemd) + queue:restart on deploy
[ ] scheduler cron installed (* * * * * php artisan schedule:run)
[ ] Reverb (if used) running and proxied over wss://
[ ] HTTPS enforced; trusted proxies configured behind a load balancer
[ ] storage:link created; private disks NOT web-accessible
[ ] DB backups + a tested restore
[ ] error tracking (Sentry/Flare) + log channel to stderr/centralised logs
[ ] CI: tests + static analysis (PHPStan/Larastan) green before deploy
```

### Stretch goals
- Add a CI workflow (GitHub Actions) running `php artisan test` + Larastan + Pint.
- Add team-scoped **rate limits** that differ by plan.
- Add a feature-flag system gating betas per team.
- Containerise with Docker / Laravel Sail and write a `docker-compose.yml`.

---

## How to approach building any of these

A repeatable workflow beats heroics. For every project:

1. **Model the data first.** Sketch tables and relationships on paper before writing code. The schema is the spine; getting it right makes everything else fall out naturally.
2. **Migrations and factories before logic.** Create migrations, then factories + a seeder so you always have realistic data: `php artisan make:model Post -mfsc` scaffolds model, migration, factory, seeder, and controller in one shot.
3. **Build vertical slices, not horizontal layers.** Ship one complete feature end-to-end (route -> controller -> model -> response -> test) before starting the next. A working "create a task" beats half-built CRUD across ten entities.
4. **Validate at the edge.** Use `FormRequest`s from day one; never trust input.
5. **Write a test as you finish each slice.** Even one happy-path feature test per endpoint catches regressions and doubles as documentation of intent.
6. **Refactor toward Laravel idioms** once it works: extract policies, API Resources, scopes, and service/action classes. Don't pre-abstract.
7. **Commit small and often** with meaningful messages — interviewers may read your git history.
8. **Write a README** with setup steps, an architecture paragraph, and the trade-offs you knowingly made. This is the single highest-leverage artifact for interviews.

Bootstrap commands you'll reuse constantly:

```bash
laravel new myapp            # or: composer create-project laravel/laravel myapp
php artisan install:api      # Sanctum + routes/api.php  (Laravel 11/12)
php artisan install:broadcasting   # Reverb + Echo        (Laravel 11/12)
php artisan make:model Product -mfsc
php artisan make:request StoreProductRequest
php artisan make:policy ProductPolicy --model=Product
php artisan make:resource ProductResource
php artisan make:job ProcessResume
php artisan make:notification NewApplicationReceived
php artisan make:event MessageSent
php artisan make:test ProductApiTest --pest
```

---

## What to demo in an interview

You won't have time to walk through every file. Lead with the **story and the decisions**:

- **30-second pitch:** "This is a store API with Sanctum auth, API Resources, filtered pagination, and an atomic checkout that prevents overselling with row locks."
- **Show the data model** and explain one interesting relationship choice (e.g. why money is stored as integer cents, why order items snapshot the price).
- **Show one hard part working live:** real-time messages appearing in two browser windows; a queued job processing; a 403 from a policy when you act as the wrong user.
- **Talk trade-offs:** "I used a hand-rolled role check here; in production I'd reach for `spatie/laravel-permission`." This signals you know the difference between a learning project and production.
- **Show a test passing** and explain what it protects.
- **Name what you'd do next** — the stretch goals are your "what would you improve?" answer ready to go.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting to scope queries to the current user/tenant.** Route model binding loads *any* record by id. A user can `GET /api/tasks/999` and read someone else's task unless you add a policy or `where('user_id', auth()->id())`. **Fix:** use policies (`$this->authorize(...)`), owned relationships (`$request->user()->tasks()`), or global scopes.

2. **Returning Eloquent models directly from API endpoints.** It leaks every column (including ones you add later), serialises timestamps inconsistently, and exposes internals. **Fix:** wrap responses in **API Resources** so JSON shape is an explicit, stable contract.

3. **Assuming queued/broadcast code runs without a worker.** With `QUEUE_CONNECTION` set to a real driver, `dispatch()` and `ShouldBroadcast` events sit in the queue forever if no `php artisan queue:work` is running — so emails never send and live messages never appear. **Fix:** run a worker, or use `sync`/`ShouldBroadcastNow` to confirm logic locally, then switch to a real driver + worker.

4. **Storing money as floats.** `0.1 + 0.2 !== 0.3` in floating point; totals drift by cents. **Fix:** store integer **cents** (`price_cents`), do arithmetic in integers, and format to currency only at display time.

5. **N+1 queries in list views.** Looping over posts and accessing `$post->author->name` fires one query per post. **Fix:** eager load (`Post::with('author')`), and use `whenLoaded()` in resources so you never accidentally lazy-load.

6. **Putting private uploads in `public/`.** A resume in `storage/app/public` (linked to `public/`) is downloadable by anyone with the URL. **Fix:** use a non-public disk and serve files through a controller that runs a policy check.

7. **Caching without invalidation.** `Cache::remember` with a long TTL shows stale data after writes. **Fix:** `Cache::forget()` (or tag flush) in model observers on save/delete; remember `file`/`database` drivers don't support cache tags.

---

## ✅ Best Practices

- **Schema first, code second.** Migrations and factories before controllers.
- **Validate every input** with `FormRequest` classes; keep controllers thin.
- **Use enums** for fixed value sets (status, role, plan) and cast them on the model.
- **Push slow work to queues** (mail, file processing, third-party calls) so requests stay fast.
- **Authorize with policies/gates**, not inline `if` checks scattered through controllers.
- **Shape API output with Resources**; never return raw models.
- **Eager load relationships** and watch for N+1 with Debugbar/Telescope.
- **Write at least one feature test per endpoint** — happy path plus one failure case.
- **Store secrets in `.env`**, commit a `.env.example`, never commit real credentials.
- **Write a README** documenting setup, architecture, and conscious trade-offs.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Walk me through how you structured one of your projects.**
A: Lead with the data model and one or two key decisions (relationships, money as cents, where authorization lives), then the request lifecycle for one endpoint: route -> middleware -> FormRequest validation -> controller -> Eloquent -> API Resource -> JSON. Mention tests as your safety net.

**Q2. How do you make sure a user can only access their own data?**
A: Three layers — owned relationships (`$user->tasks()->...`), **policies** invoked via `authorize()`/`@can`, and, for multi-tenant apps, **global scopes** that auto-filter by team. Route model binding alone is *not* an authorization boundary.

**Q3. When would you use a queue, and what happens under the hood?**
A: Use a queue for anything slow or failure-prone that shouldn't block the response (email, file processing, broadcasting, external APIs). Under the hood, `dispatch()` **serialises** the job (using `SerializesModels`, which stores model IDs and re-fetches them on execution) into a backend (database/Redis). A separate worker process polls, deserialises, runs `handle()`, and removes the job on success or moves it to `failed_jobs` after exhausting retries.

**Q4. How does Laravel's broadcasting work under the hood?**
A: You fire a PHP **event** implementing `ShouldBroadcast`. Laravel queues it; a worker hands the payload (from `broadcastWith()`) to the configured broadcaster (Reverb/Pusher), which pushes it over an open **WebSocket** to subscribed clients. The browser's **Echo** client authenticates to private/presence channels via a signed request to `/broadcasting/auth` (your `routes/channels.php` callbacks decide who may subscribe), then listens for events by name.

**Q5. What's the difference between a Gate and a Policy?**
A: Both authorize actions. A **Gate** is a standalone closure for one-off or non-model checks (`Gate::define('view-admin', ...)`). A **Policy** is a class grouping all authorization for one model (`PostPolicy::update/delete/...`), auto-discovered by naming convention and invoked via `authorize()`/`@can`. Use policies for model CRUD, gates for everything else.

**Q6. How do you prevent overselling in a checkout under concurrent requests?**
A: Wrap checkout in a **database transaction** and use **pessimistic locking** (`lockForUpdate()`) when reading the product row, so concurrent checkouts serialise on that row. Re-check stock after locking and decrement atomically; if insufficient, abort and the transaction rolls back. Alternatively, an optimistic `where('stock', '>=', $qty)->decrement(...)` and check affected rows.

**Q7. How do API Resources help, and how do they differ from just `toArray()` on a model?**
A: Resources decouple the JSON contract from the DB schema. They let you rename/transform fields, conditionally include data (`when`, `whenLoaded`), nest related resources, and attach pagination meta automatically with `::collection()`. The model's `toArray()` dumps raw attributes and is coupled to columns.

**Q8. How would you handle file uploads securely?**
A: Validate type and size in a FormRequest (`mimes:pdf`, `max:5120`), store on a **non-public disk** (`->store('dir','private')`), persist only the path, and serve downloads through a controller that runs a policy check. Move to S3 in production by changing the disk config — no app-code change, thanks to the filesystem abstraction.

**Q9. What changed in routing/console/scheduling between Laravel 10 and 11/12?**
A: Laravel 11 slimmed the skeleton: no `app/Http/Kernel.php` or `app/Console/Kernel.php`, middleware and exception handling are configured fluently in `bootstrap/app.php` (`->withMiddleware(...)` / `->withExceptions(...)`), service providers are registered in `bootstrap/providers.php`, `routes/api.php` is opt-in via `install:api`, the scheduler lives in `routes/console.php`, and policies are auto-discovered. Laravel 12 keeps this structure, defaults the test runner to **Pest**, and makes **Reverb** the recommended broadcaster (opt-in via `install:broadcasting`, not enabled out of the box).

**Q10. How do you keep cached data fresh?**
A: Combine a sensible TTL with **explicit invalidation** on writes — call `Cache::forget(key)` (or `Cache::tags(...)->flush()` on Redis/Memcached) in model observers. Never rely on TTL alone if correctness matters; never use tags on the `file`/`database` drivers (unsupported).

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Scaffolding
laravel new app
php artisan install:api               # Sanctum + routes/api.php
php artisan install:broadcasting      # Reverb + Echo
php artisan make:model Post -mfsc     # model + migration + factory + seeder + controller
php artisan make:request StorePostRequest
php artisan make:policy PostPolicy --model=Post
php artisan make:resource PostResource
php artisan make:job ProcessThing
php artisan make:notification ThingHappened
php artisan make:event ThingBroadcast

# Running the moving parts (each in its own terminal)
php artisan serve
php artisan queue:work
php artisan schedule:work
php artisan reverb:start

# Database
php artisan migrate            # add --force in production
php artisan migrate:fresh --seed
php artisan db:seed

# Tests & quality
php artisan test
./vendor/bin/pint             # code style
./vendor/bin/phpstan analyse  # static analysis (with Larastan)

# Production caches
# NOTE: artisan takes ONE command per call — you cannot chain
#   `config:cache route:cache ...` on a single line. Run them separately,
#   or just use `optimize`, which does all of them.
php artisan config:cache      # cache config
php artisan route:cache       # cache routes
php artisan view:cache        # precompile Blade views
php artisan event:cache       # cache event/listener discovery
php artisan optimize          # all of the above (config + routes + events + views)
php artisan optimize:clear    # undo every cache above before debugging
```

| Need | Tool | Idiom |
|------|------|-------|
| Token API auth | Sanctum | `auth:sanctum` middleware, `$user->createToken()` |
| Per-model authorization | Policy | `$this->authorize('update', $post)` / `@can` |
| One-off authorization | Gate | `Gate::define(...)`, `Gate::allows(...)` |
| Shape JSON output | API Resource | `PostResource::collection($paginator)` |
| Optional query filters | `when()` | `->when($cond, fn($q)=>...)` |
| Slow work off the request | Queued Job | `Job::dispatch(...)` + `queue:work` |
| Live push to browser | Broadcasting | `ShouldBroadcast` + Reverb + Echo |
| Recurring task | Scheduler | `Schedule::call(...)->daily()` in `routes/console.php` |
| Private uploads | Storage disk | `->store('dir', 'private')` |
| Expensive reads | Cache | `Cache::remember(key, ttl, fn)` + `forget` on write |
| Atomic multi-table write | Transaction | `DB::transaction(fn()=>...)` + `lockForUpdate()` |
| Fixed value sets | Enum | `enum Status: string {}` + cast |

---

## 🧪 Mini Exercises

1. **Scope-or-leak audit.** Take your Todo API (Project 1) and write a feature test that logs in as user B and attempts `GET /api/tasks/{id}` for a task owned by user A. Make it assert `403`. If it currently returns `200`, fix the controller/policy until the test passes.

2. **Kill an N+1.** In the Blog (Project 2), render the index page that lists posts with their author and tag names. Install Laravel Debugbar (or use `DB::listen`) to count queries, then add eager loading so the page issues a constant number of queries regardless of post count. Note the before/after counts.

3. **Atomic checkout under load.** In the Store API (Project 3), write a test that sets a product's stock to 1, then fires two checkout attempts for that product. Assert exactly one succeeds (201) and the other fails (422), and that final stock is 0 — never negative.

4. **Prove the queue.** In the Job Board (Project 4), set `QUEUE_CONNECTION=database`, submit an application, and confirm the `ProcessResume` job lands in the `jobs` table *before* a worker runs, then disappears after `queue:work` processes it. Use `Queue::fake()` in a test to assert the job was dispatched without running it.

5. **Cache + invalidate.** In the SaaS capstone (Project 6), cache a team's dashboard stats with a 10-minute TTL, then create a new project and assert (in a test) that the cached stat updates immediately — proving your `Cache::forget()` invalidation fires on write.
