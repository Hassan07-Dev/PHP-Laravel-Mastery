# Controllers in Laravel 12

Controllers are the **C** in Laravel's loose interpretation of MVC. They are the classes that receive an HTTP request (routed to them from your route files) and decide what should happen next: load some data, validate input, talk to the business layer, and finally produce a response (HTML, JSON, a redirect, etc.). Instead of stuffing all that logic into closures inside `routes/web.php`, you give it a home: a dedicated, testable, injectable class.

This module takes you from `php artisan make:controller` all the way to the "thin controller" architecture that senior engineers expect to see in production code.

> **MVC** = Model–View–Controller, a pattern that separates *data* (Model), *presentation* (View), and *request-handling/coordination* (Controller). A **controller** in Laravel is a PHP class, conventionally stored in `app/Http/Controllers`.

---

## **What you'll learn**

- How to generate controllers with `make:controller` and the flags that matter (`--resource`, `--api`, `--invokable`, `--model`, `--requests`).
- The difference between basic, single-action (`__invoke`), resource, and API resource controllers — and which to reach for.
- The seven RESTful resource methods and exactly what HTTP verb + URI each maps to.
- How the **service container** auto-resolves dependencies via constructor *and* method injection, including the `Request`.
- How **route model binding** turns a URL segment into a fully hydrated Eloquent model right in your method signature.
- The Laravel 11+ way to attach middleware (`HasMiddleware` + the static `middleware()` method) versus the legacy constructor approach.
- How to return views, JSON, and redirects, and the canonical form-handling (validate → act → redirect) pattern.
- The **thin-controller principle**: why business logic belongs in services/actions, not in the controller.

---

## 1. Why controllers exist (the WHY before the HOW)

You *can* put logic directly in a route:

```php
// routes/web.php
use App\Models\Post;

Route::get('/posts', function () {
    return view('posts.index', ['posts' => Post::latest()->paginate(15)]);
});
```

This is fine for a one-off. But as an app grows, route-file closures become a problem:

1. **They can't be cached.** `php artisan route:cache` (a production speed-up) silently *fails* if any route uses a `Closure`, because closures cannot be serialized.
2. **They're hard to test in isolation.** You can't easily new-up a closure and call it.
3. **They scatter related logic.** Everything about "posts" should live together, not be smeared across a 500-line route file.

A controller solves all three. It's a regular class, so it's cacheable, unit-testable, and groupable. You point a route at a `[Controller::class, 'method']` pair, and Laravel instantiates the class (through the container) and calls the method when the route matches.

```php
// routes/web.php
use App\Http\Controllers\PostController;

Route::get('/posts', [PostController::class, 'index']);
```

> The `::class` constant resolves to the fully-qualified class name string (`"App\Http\Controllers\PostController"`). Prefer it over hard-coded strings — your IDE can refactor it, and typos become compile-time-ish errors.

---

## 2. Generating controllers with `make:controller`

The Artisan generator scaffolds the file in `app/Http/Controllers`.

```bash
# A bare controller (empty class)
php artisan make:controller PostController
```

```php
<?php

namespace App\Http\Controllers;

// Laravel 11+ note: the abstract base `Controller` lives at
// App\Http\Controllers\Controller and no longer extends a
// framework class by default. It's just a place for shared traits.
class PostController extends Controller
{
    //
}
```

### Useful flags

| Flag | What it generates |
|------|-------------------|
| `--resource` / `-r` | A controller with the seven RESTful stubs (`index`, `create`, `store`, `show`, `edit`, `update`, `destroy`). |
| `--api` | Same as `--resource` but **without** `create` and `edit` (those return HTML forms, which an API doesn't need). |
| `--invokable` / `-i` | A single-action controller with one `__invoke` method. |
| `--model=Post` / `-m Post` | Type-hints the `Post` model in the relevant methods (enables route model binding). |
| `--requests` / `-R` | Generates `StorePostRequest` / `UpdatePostRequest` Form Request classes and wires them in. |
| `--parent=Author` / `-p` | Scaffolds a *nested* resource controller (shallow/nested bindings). |
| `--singleton` / `-s` | A singleton resource controller (no `index`; one implicit resource). |
| `--test` / `--pest` | Also create a matching test file. |

```bash
# The combo you'll use most for a CRUD admin feature:
php artisan make:controller PostController --resource --model=Post --requests

# An API-only resource controller bound to a model:
php artisan make:controller Api/PostController --api --model=Post
```

> Slashes create sub-namespaces/sub-folders. `Api/PostController` lands at `app/Http/Controllers/Api/PostController.php` with namespace `App\Http\Controllers\Api`.

---

## 3. Basic controllers & methods

A "basic" controller is just a class with public methods, each wired to a route.

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\View\View;

class UserController extends Controller
{
    public function show(string $id): View
    {
        return view('users.profile', [
            'user' => User::findOrFail($id),
        ]);
    }
}
```

```php
// routes/web.php
use App\Http\Controllers\UserController;

Route::get('/users/{id}', [UserController::class, 'show']);
```

Visiting `/users/42` calls `show('42')`. The `{id}` route parameter is passed positionally to the method. (We'll soon replace this manual `findOrFail` with route model binding.)

> **Return type hints** (`: View`) are optional but recommended in modern PHP. They document intent and let static analysers (PHPStan/Larastan) catch mistakes. Common Laravel return types: `Illuminate\View\View`, `Illuminate\Http\JsonResponse`, `Illuminate\Http\RedirectResponse`, `Illuminate\Http\Response`.

---

## 4. Single-action controllers (`__invoke`)

When a controller has exactly **one** job, giving it a single `__invoke` method is cleaner than inventing a method name like `handle` or `index`.

`__invoke` is a PHP **magic method**: any object that defines it becomes "callable" like a function (`$obj()`). Laravel detects it, so you route to the *class itself*, not a `[class, method]` pair.

```bash
php artisan make:controller ProvisionServer --invokable
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\Server;
use Illuminate\Http\RedirectResponse;

class ProvisionServer extends Controller
{
    public function __invoke(Server $server): RedirectResponse
    {
        $server->provision();

        return redirect()
            ->route('servers.show', $server)
            ->with('status', 'Server provisioning started.');
    }
}
```

```php
// routes/web.php — note: no method name in the array
use App\Http\Controllers\ProvisionServer;

Route::post('/servers/{server}/provision', ProvisionServer::class)
    ->name('servers.provision');
```

**Why use them?** They keep classes small and single-responsibility, they read beautifully (`Route::post(..., LoginController::class)`), and they pair naturally with the "action class" architecture discussed at the end.

---

## 5. Resource controllers — the seven RESTful methods

Most web CRUD (Create, Read, Update, Delete) features follow the same shape. **Resource controllers** standardize that shape so you don't reinvent route names and verbs each time.

```bash
php artisan make:controller PhotoController --resource --model=Photo
```

```php
// routes/web.php
use App\Http\Controllers\PhotoController;

Route::resource('photos', PhotoController::class);
```

That **one line** registers seven routes. Here is the complete mapping — memorize this table; it's a frequent interview question:

| Verb | URI | Controller method | Route name | Purpose |
|------|-----|-------------------|------------|---------|
| GET | `/photos` | `index` | `photos.index` | List all photos |
| GET | `/photos/create` | `create` | `photos.create` | Show "new photo" form |
| POST | `/photos` | `store` | `photos.store` | Persist the new photo |
| GET | `/photos/{photo}` | `show` | `photos.show` | Show one photo |
| GET | `/photos/{photo}/edit` | `edit` | `photos.edit` | Show "edit photo" form |
| PUT/PATCH | `/photos/{photo}` | `update` | `photos.update` | Persist edits |
| DELETE | `/photos/{photo}` | `destroy` | `photos.destroy` | Delete the photo |

Verify it yourself:

```bash
php artisan route:list --name=photos
```

```text
# Output (abridged; exact spacing/sort order varies by version):
  GET|HEAD   photos ................... photos.index › PhotoController@index
  POST       photos ................... photos.store › PhotoController@store
  GET|HEAD   photos/create ........... photos.create › PhotoController@create
  GET|HEAD   photos/{photo} ......... photos.show › PhotoController@show
  PUT|PATCH  photos/{photo} ......... photos.update › PhotoController@update
  DELETE     photos/{photo} ......... photos.destroy › PhotoController@destroy
  GET|HEAD   photos/{photo}/edit .... photos.edit › PhotoController@edit
```

> **Why two GET forms (`create`/`edit`)?** Browsers can only natively send GET and POST. The `create`/`edit` methods return HTML *forms*; the form then POSTs (or spoofs PUT/PATCH/DELETE). APIs skip `create`/`edit` because there's no HTML form — hence `--api`.

### A worked resource controller

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePhotoRequest;
use App\Http\Requests\UpdatePhotoRequest;
use App\Models\Photo;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class PhotoController extends Controller
{
    public function index(): View
    {
        return view('photos.index', ['photos' => Photo::latest()->paginate(20)]);
    }

    public function create(): View
    {
        return view('photos.create');
    }

    public function store(StorePhotoRequest $request): RedirectResponse
    {
        // $request->validated() returns ONLY the rules-approved fields.
        $photo = Photo::create($request->validated());

        return redirect()->route('photos.show', $photo)
            ->with('status', 'Photo uploaded!');
    }

    public function show(Photo $photo): View          // route model binding
    {
        return view('photos.show', compact('photo'));
    }

    public function edit(Photo $photo): View
    {
        return view('photos.edit', compact('photo'));
    }

    public function update(UpdatePhotoRequest $request, Photo $photo): RedirectResponse
    {
        $photo->update($request->validated());

        return redirect()->route('photos.show', $photo)
            ->with('status', 'Photo updated!');
    }

    public function destroy(Photo $photo): RedirectResponse
    {
        $photo->delete();

        return redirect()->route('photos.index')
            ->with('status', 'Photo deleted!');
    }
}
```

### Trimming and renaming resource routes

```php
// Only generate a subset:
Route::resource('photos', PhotoController::class)->only(['index', 'show']);

// Generate all except some:
Route::resource('photos', PhotoController::class)->except(['destroy']);

// Rename the generated route names:
Route::resource('photos', PhotoController::class)->names([
    'index' => 'photos.gallery',
]);

// Customize the bound parameter name (default is {photo}):
Route::resource('photos', PhotoController::class)->parameters([
    'photos' => 'image',     // → /photos/{image}
]);

// Register many at once:
Route::resources([
    'photos'  => PhotoController::class,
    'posts'   => PostController::class,
]);
```

### Nested resources

```php
// /authors/{author}/photos/{photo}
Route::resource('authors.photos', AuthorPhotoController::class);

// "Shallow" nesting: index/create/store are nested, but show/edit/update/destroy
// are flat (/photos/{photo}) since the ID is already unique.
Route::resource('authors.photos', AuthorPhotoController::class)->shallow();
```

### Singleton resources (one-and-only-one instances)

Some resources only ever have a single instance per context — a user's `profile`, an image's `thumbnail`. Register these with `Route::singleton`, which omits `index`/`create`/`store` and the `{id}` segment:

```php
use App\Http\Controllers\ProfileController;

Route::singleton('profile', ProfileController::class);
// → GET /profile (show), GET /profile/edit (edit), PUT/PATCH /profile (update)

// Generate the controller stub for it:
// php artisan make:controller ProfileController --singleton   (or -s)

// Allow creation/deletion on a singleton (adds create/store and a destroy route):
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();

// API singleton (drops the HTML create/edit routes):
Route::apiSingleton('profile', ProfileController::class);
```

### Soft-deletable resources (Laravel 12)

`Route::softDeletableResources()` is shorthand for registering several resources that should all allow binding to soft-deleted models (equivalent to calling `->withTrashed()` on each):

```php
Route::softDeletableResources([
    'photos' => PhotoController::class,
    'posts'  => PostController::class,
]);
```

---

## 6. API resource controllers

For JSON APIs, use `--api`. The generated controller omits `create` and `edit`:

```bash
php artisan make:controller Api/PhotoController --api --model=Photo
```

```php
// routes/api.php  (in Laravel 11+, run `php artisan install:api` to enable this file)
use App\Http\Controllers\Api\PhotoController;

Route::apiResource('photos', PhotoController::class);
```

`apiResource` registers only **five** routes (`index`, `store`, `show`, `update`, `destroy`). An API controller typically returns JSON via **API Resources** (`Illuminate\Http\Resources\Json\JsonResource`) for shape control:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePhotoRequest;
use App\Http\Resources\PhotoResource;
use App\Models\Photo;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Response;

class PhotoController extends Controller
{
    public function index(): AnonymousResourceCollection
    {
        return PhotoResource::collection(Photo::paginate(20));
    }

    public function store(StorePhotoRequest $request): JsonResponse
    {
        $photo = Photo::create($request->validated());

        // 201 Created is the correct status for a successful POST.
        return (new PhotoResource($photo))
            ->response()
            ->setStatusCode(201);
    }

    public function show(Photo $photo): PhotoResource
    {
        return new PhotoResource($photo);
    }

    public function destroy(Photo $photo): Response
    {
        $photo->delete();

        // noContent() returns a truly empty body with a 204 status.
        // Avoid response()->json(status: 204) — that emits a "{}" body,
        // which contradicts the "No Content" semantics of a 204.
        return response()->noContent();   // 204 No Content
    }
}
```

```bash
# There is no `php artisan apiResource` command — you register apiResource()
# in routes/api.php (shown above). Inspect the result with:
php artisan route:list --path=api/photos
```

```text
# Output:
  GET|HEAD   api/photos .......... photos.index
  POST       api/photos .......... photos.store
  GET|HEAD   api/photos/{photo} .. photos.show
  PUT|PATCH  api/photos/{photo} .. photos.update
  DELETE     api/photos/{photo} .. photos.destroy
```

> **API resources stub:** `php artisan make:resource PhotoResource` creates the transformer. It is *not* the same as an "API resource controller" — the controller routes the request; the resource shapes the JSON.

---

## 7. Dependency injection resolved by the container

Laravel's **service container** is the engine that builds your controllers. When a route resolves to a controller, the container instantiates the class and *automatically supplies* anything you type-hint — both in the **constructor** and in the **method**. This is **dependency injection (DI)**: you declare what you need, the container hands it to you.

> **Service container** = Laravel's "IoC container," a registry that knows how to build objects and their dependencies. **IoC** = Inversion of Control: you don't `new` things up yourself; you let the framework do it.

### Constructor injection (with PHP 8 constructor promotion)

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;
use Illuminate\View\View;

class UserController extends Controller
{
    // Constructor property promotion (PHP 8.0+): declaring `private` here
    // also creates and assigns $this->users in one line.
    public function __construct(
        private readonly UserRepository $users,
    ) {}

    public function index(): View
    {
        return view('users.index', ['users' => $this->users->all()]);
    }
}
```

You never write `new UserController(new UserRepository(...))`. The container sees the type-hint `UserRepository`, builds it (recursively building *its* dependencies), and injects it. `readonly` (PHP 8.1+) makes the property immutable after construction — a good default for injected dependencies.

### Method injection

The container also inspects each controller *method's* signature and injects matches. This is how `Request` gets in:

```php
use Illuminate\Http\Request;

public function update(Request $request, UserService $service, string $id)
{
    // $request and $service are injected by the container;
    // $id comes from the route URI {id}.
}
```

**Ordering rule:** type-hinted dependencies and route parameters can appear in any order, but matching is by *type* for dependencies and by *name* for route parameters. Put the `Request` first by convention.

---

## 8. Injecting the `Request`

The `Illuminate\Http\Request` object represents the incoming HTTP request: query string, form body, headers, cookies, files, the authenticated user, and more.

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;

class SearchController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        // Reading input — all of these are common in interviews:
        $term   = $request->input('q', 'default');   // body or query, with default
        $page   = $request->query('page', 1);        // query string only
        $all    = $request->all();                    // everything
        $only   = $request->only(['q', 'page']);      // whitelist
        $has    = $request->has('q');                 // bool
        $filled = $request->filled('q');              // present AND not empty
        $file   = $request->file('avatar');           // UploadedFile|null
        $user   = $request->user();                   // authenticated user
        $isJson = $request->expectsJson();            // content negotiation

        // Inline validation (throws ValidationException → auto redirect/JSON 422):
        $validated = $request->validate([
            'q'    => ['required', 'string', 'max:100'],
            'page' => ['integer', 'min:1'],
        ]);

        return redirect()->route('search.results', $validated);
    }
}
```

For anything non-trivial, prefer a **Form Request** (a dedicated validation class) over inline `$request->validate()`:

```bash
php artisan make:request StorePhotoRequest
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePhotoRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', \App\Models\Photo::class);
    }

    public function rules(): array
    {
        return [
            'title'   => ['required', 'string', 'max:255'],
            'caption' => ['nullable', 'string'],
            'image'   => ['required', 'image', 'max:5120'], // 5 MB
        ];
    }
}
```

Type-hinting `StorePhotoRequest $request` in a controller method triggers validation *and* authorization automatically — if either fails, your method body never runs. This is a huge part of keeping controllers thin.

---

## 9. Route model binding into controller methods

**Route model binding** automatically converts a URI segment into an Eloquent model instance. Instead of receiving a string `$id` and calling `findOrFail`, you type-hint the model and Laravel does the lookup — returning a 404 automatically if not found.

### Implicit binding

The route parameter name must match the method parameter variable name.

```php
// routes/web.php — parameter is {photo}
Route::get('/photos/{photo}', [PhotoController::class, 'show']);
```

```php
// Method variable is $photo — names match → Laravel runs Photo::findOrFail(...)
public function show(Photo $photo): View
{
    return view('photos.show', compact('photo'));
}
```

`/photos/7` → Laravel fetches the `Photo` with primary key 7. Not found → automatic `404`.

### Customizing the key column

By default binding uses the primary key. To bind by `slug` instead:

```php
// Option A: inline in the route
Route::get('/photos/{photo:slug}', [PhotoController::class, 'show']);
```

```php
// Option B: globally for the model — override getRouteKeyName()
class Photo extends Model
{
    public function getRouteKeyName(): string
    {
        return 'slug';
    }
}
```

### Scoped & soft-deleted bindings

```php
// Scope nested child to its parent (only return the comment that belongs to this post).
// On a standalone route use ->scopeBindings(); the child query is constrained to the
// parent's relationship (here: $post->comments()).
Route::get('/posts/{post}/comments/{comment}', [CommentController::class, 'show'])
    ->scopeBindings();

// Allow binding to a soft-deleted model:
Route::get('/photos/{photo}', [PhotoController::class, 'show'])
    ->withTrashed();
```

> **Resource-route equivalent:** on `Route::resource(...)` you scope (and optionally pick the key column) with `->scoped(['comment' => 'slug'])`, not `->scopeBindings()`. Using a custom-keyed nested parameter (e.g. `{comment:slug}`) auto-enables scoping. `->scopeBindings()` is the standalone-route helper; `->scoped([...])` is the resource-route helper.

### Enums in route bindings (PHP 8.1+ / Laravel 9+)

You can type-hint a **backed enum** as a route parameter; Laravel rejects values that aren't valid enum cases with a 404.

```php
enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
// /categories/fruits → "fruits"   ;   /categories/cars → 404
```

> **Under the hood:** implicit binding is performed by the `SubstituteBindings` middleware (in the route's middleware stack). It reads the method's reflection, sees a parameter type-hinting an `Illuminate\Database\Eloquent\Model` subclass, and resolves it before your controller runs.

---

## 10. Controller middleware

**Middleware** are layers that wrap a request/response — auth checks, throttling, logging. You often want middleware applied only to certain controller methods.

### The Laravel 11+ way: `HasMiddleware`

Laravel 11 introduced the `Illuminate\Routing\Controllers\HasMiddleware` interface and a **static `middleware()`** method. This *replaced* calling `$this->middleware(...)` in the constructor. In Laravel 11/12 the base `App\Http\Controllers\Controller` no longer extends `Illuminate\Routing\Controller`, so the `middleware()` instance method that used to live there is **gone** — calling `$this->middleware('auth')` in a default app now throws `Error: Call to undefined method`. This is a hard breaking change, not a soft deprecation.

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;

class PhotoController extends Controller implements HasMiddleware
{
    public static function middleware(): array
    {
        return [
            'auth',                                              // applies to all methods
            new Middleware('verified', only: ['store']),        // only store()
            new Middleware('throttle:6,1', except: ['index']),  // all but index()
        ];
    }

    // ... methods ...
}
```

The `Middleware` value object takes `only:` / `except:` to scope which methods it wraps. Returning a plain string (like `'auth'`) applies it everywhere.

You can also use a **closure** as inline middleware:

```php
use Closure;
use Illuminate\Http\Request;

public static function middleware(): array
{
    return [
        function (Request $request, Closure $next) {
            // do something before...
            return $next($request);
        },
    ];
}
```

### The older (pre-11) constructor approach — REMOVED in Laravel 11

In Laravel 10 and earlier you registered controller middleware in the constructor:

```php
// ⚠️ Laravel 10 and EARLIER ONLY. This does NOT work in Laravel 11/12 —
// the base controller no longer provides a middleware() method, so this
// throws "Call to undefined method ...::middleware()". Use HasMiddleware instead.
public function __construct()
{
    $this->middleware('auth');
    $this->middleware('subscribed')->only('store');
    $this->middleware('throttle:6,1')->except('index');
}
```

If you upgrade an older project to Laravel 11/12, every such constructor must be migrated to the static `middleware()` method shown above (or the middleware moved to the route definition).

> **Heads-up:** the Laravel docs note that a controller implementing `HasMiddleware` should **not** also extend the old `Illuminate\Routing\Controller`. In a fresh Laravel 11/12 app this is a non-issue because the generated `App\Http\Controllers\Controller` is a plain abstract class, but it matters when porting legacy code.

> **Interview-worthy distinction:** middleware can also be attached at the *route* level (`Route::get(...)->middleware('auth')`), in *route groups*, or — for resource controllers specifically — via `->middlewareFor('show', 'auth')` and `->withoutMiddlewareFor(...)` on the route registration (Laravel 12). Controller-level middleware (the static `middleware()` method) is just a convenience for method-scoped rules co-located with the controller, and it's preferred over the old constructor approach because it doesn't require instantiating the controller to read its middleware.

---

## 11. Returning responses: views, JSON, redirects

A controller method's return value is turned into an HTTP response. Laravel is smart about types.

### Views (HTML)

```php
return view('photos.show', ['photo' => $photo]);
return view('photos.show', compact('photo'));        // same thing
return view('photos.index')->with('photos', $photos);
```

### JSON

```php
return response()->json(['ok' => true]);                  // 200
return response()->json(['error' => 'Nope'], 403);       // explicit status
return response()->json($photo, 201);

// Returning an array/Arrayable/JsonResource directly → auto-JSON:
return ['ok' => true];                                    // becomes JSON
return Photo::all();                                      // Eloquent collection → JSON
return new PhotoResource($photo);                         // API resource → JSON
```

### Redirects

```php
return redirect('/home');                                 // to a path
return redirect()->route('photos.show', $photo);          // to a named route
return redirect()->route('photos.show', ['photo' => 7]);  // explicit params
return redirect()->back()->withInput();                   // back + repopulate form
return redirect()->route('login')->with('status', 'Bye'); // flash a session message
return to_route('photos.index');                          // helper shorthand (Laravel 9+)
```

### Other response helpers

```php
return response('Hello', 200)->header('Content-Type', 'text/plain');
return response()->noContent();                           // 204
return response()->download($pathToFile);                 // force download

// streamDownload takes a CLOSURE that echoes the content (not an arrow fn —
// `echo` is a statement, so `fn () => echo ...` is a parse error):
return response()->streamDownload(function () {
    echo $csv;
}, 'export.csv');

abort(404);                                                // throw an HTTP exception
abort_if($photo->locked, 403, 'Locked');                  // conditional abort
```

---

## 12. The form-handling pattern (validate → act → redirect)

The canonical web-form flow uses the **POST/Redirect/GET (PRG)** pattern to prevent duplicate submissions when a user refreshes.

```blade
{{-- resources/views/photos/create.blade.php --}}
<form method="POST" action="{{ route('photos.store') }}" enctype="multipart/form-data">
    @csrf  {{-- REQUIRED: injects the CSRF token hidden field --}}

    <input type="text" name="title" value="{{ old('title') }}">
    @error('title') <span class="error">{{ $message }}</span> @enderror

    <input type="file" name="image">
    <button type="submit">Upload</button>
</form>
```

To send PUT/PATCH/DELETE from an HTML form (which only knows GET/POST), spoof the method:

```blade
<form method="POST" action="{{ route('photos.update', $photo) }}">
    @csrf
    @method('PUT')   {{-- injects a hidden _method=PUT field --}}
    {{-- ... --}}
</form>
```

```php
public function store(StorePhotoRequest $request): RedirectResponse
{
    // 1. VALIDATE — handled by the Form Request before we even get here.
    $data = $request->validated();

    // 2. ACT — persist (ideally delegated to a service; see next section).
    $photo = Photo::create($data);

    // 3. REDIRECT — PRG pattern + flash a success message.
    return redirect()
        ->route('photos.show', $photo)
        ->with('status', 'Photo created!');
}
```

> **Why redirect instead of returning a view?** If you returned a view after a POST, hitting refresh re-POSTs the form (the browser warns "resubmit form?"), creating duplicates. Redirecting to a GET route makes refresh safe.

---

## 13. The thin-controller principle

A controller's job is **HTTP coordination**, not business logic. A "fat" controller mixes validation, database queries, external API calls, emails, and formatting into one bloated method — untestable and unreusable. The fix: **push logic down** into services or single-purpose action classes, and keep the controller a thin orchestrator.

### Before — fat controller (anti-pattern)

```php
public function store(Request $request)
{
    $data = $request->validate([/* ... */]);

    $photo = new Photo($data);
    $photo->user_id = $request->user()->id;
    $photo->thumbnail = Image::make($request->file('image'))->fit(200, 200)->encode();
    $photo->save();

    Mail::to($request->user())->send(new PhotoUploaded($photo));
    Cache::forget('homepage_photos');
    Log::info('Photo uploaded', ['id' => $photo->id]);

    return redirect()->route('photos.show', $photo);
}
```

### After — thin controller + action class

```php
// app/Actions/UploadPhoto.php
namespace App\Actions;

use App\Models\Photo;
use App\Models\User;
use Illuminate\Http\UploadedFile;

class UploadPhoto
{
    public function handle(User $user, UploadedFile $image, array $attributes): Photo
    {
        // ... all the thumbnailing, saving, mailing, cache-busting lives here ...
        return $photo;
    }
}
```

```php
// app/Http/Controllers/PhotoController.php
use App\Actions\UploadPhoto;

public function store(StorePhotoRequest $request, UploadPhoto $action): RedirectResponse
{
    $photo = $action->handle(
        user: $request->user(),
        image: $request->file('image'),
        attributes: $request->safe()->except('image'),
    );

    return redirect()->route('photos.show', $photo)
        ->with('status', 'Photo created!');
}
```

Now `UploadPhoto` is unit-testable without HTTP, reusable from a queued job or an Artisan command, and the controller reads like a table of contents. Note `$action` is injected by the container — no manual wiring.

> A good rule of thumb: a controller method should be **~5–15 lines**, mostly delegation. If you see loops, conditionals on business rules, or query building, it probably belongs in a service/action/model scope.

---

## ⚠️ Common Mistakes & Gotchas

1. **Route model binding parameter name mismatch.**
   `Route::get('/photos/{photo}', ...)` but the method signature is `show(Photo $image)`. The names `{photo}` and `$image` differ, so binding silently fails and `$image` arrives empty/unresolved.
   **Fix:** make the route segment name match the variable name (`{photo}` ↔ `$photo`), or alias with `{photo:slug}` and keep the var name `$photo`.

2. **Forgetting `@csrf` / getting a 419 Page Expired.**
   Every non-GET HTML form needs `@csrf`. Without it the `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` middleware (part of the `web` group) rejects the request with HTTP **419**.
   **Fix:** add `@csrf` inside every `<form method="POST|PUT|PATCH|DELETE">`. (CSRF = Cross-Site Request Forgery protection.)
   > **Naming note:** This middleware was called `VerifyCsrfToken` in Laravel 10 and earlier; it was renamed to `ValidateCsrfToken` in Laravel 11. To exempt routes (e.g. third-party webhooks), use `$middleware->validateCsrfTokens(except: [...])` in `bootstrap/app.php` — there is no `app/Http/Middleware/VerifyCsrfToken.php` to edit in Laravel 11/12.

3. **HTML forms can't send PUT/PATCH/DELETE.**
   Setting `method="PUT"` on a `<form>` doesn't work — browsers send it as GET.
   **Fix:** use `method="POST"` plus `@method('PUT')` to spoof the verb.

4. **Returning a view after a POST (no redirect).**
   Causes duplicate submissions on refresh and breaks the PRG pattern; `old()` and validation error repopulation won't behave as users expect.
   **Fix:** validate, act, then `redirect()` (optionally `->withInput()` on failure, though Form Requests handle that automatically).

5. **Closures in route files block `route:cache`.**
   `php artisan route:cache` aborts with *"Unable to prepare route for serialization. Another route uses a closure."*
   **Fix:** move closures into controllers (or invokable controllers) before caching routes in production.

6. **Using `make:controller --api` but registering with `Route::resource`.**
   `Route::resource` expects `create`/`edit` methods that an `--api` controller doesn't have, producing 404s or "method not found"-style confusion on those URIs.
   **Fix:** pair `--api` controllers with `Route::apiResource`.

7. **Putting business logic that mutates state inside `index`/`show` (GET) methods.**
   GET requests must be *safe* and *idempotent*; side effects there break caching, prefetching, and REST semantics.
   **Fix:** mutations go in `store`/`update`/`destroy` (or POST/PUT/DELETE routes).

---

## ✅ Best Practices

- **Keep controllers thin.** Delegate to Form Requests (validation/authorization), services/actions (business logic), and API Resources (response shaping).
- **Use Form Requests** instead of inline `$request->validate()` for anything beyond a couple of rules.
- **Prefer route model binding** over manual `findOrFail`. Let the framework give you the 404.
- **Type-hint everything**: parameters and return types. It documents the method and powers static analysis.
- **One resource controller per resource**; use `apiResource` for APIs and `resource` for web CRUD.
- **Name your routes** (`->name(...)` or via `Route::resource` defaults) and reference them with `route()`/`to_route()` so URLs aren't hard-coded.
- **Use `--invokable`/action classes** for one-off operations that don't fit CRUD.
- **Attach method-scoped middleware** via the Laravel 11+ static `middleware()` method.
- **Return the right status codes** in APIs (201 on create, 204 on delete, 422 on validation failure).
- **Never trust `$request->all()` for mass assignment** — use `validated()` / `safe()` and guard your models with `$fillable`.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What are the seven RESTful methods of a resource controller and what does each map to?**
A: `index` (GET /res), `create` (GET /res/create), `store` (POST /res), `show` (GET /res/{id}), `edit` (GET /res/{id}/edit), `update` (PUT/PATCH /res/{id}), `destroy` (DELETE /res/{id}). `create` and `edit` return HTML forms.

**Q2. What's the difference between `Route::resource` and `Route::apiResource`?**
A: `apiResource` omits `create` and `edit` (the form-displaying GET routes), registering five routes instead of seven, because APIs have no HTML forms.

**Q3. How does dependency injection into controllers actually work under the hood?**
A: Laravel resolves the controller through the **service container**. It uses **PHP Reflection** to read the constructor and method signatures, then for each type-hinted parameter it recursively resolves an instance from the container's bindings (or auto-resolves concrete classes). Route parameters are matched by name. The `Illuminate\Http\Request` is registered in the container as the current request instance (a single shared instance for the request lifecycle), so type-hinting it hands you that same object. Implicit model binding specifically runs in the `SubstituteBindings` middleware before the controller is called.

**Q4. What is a single-action controller and when would you use one?**
A: A controller with only an `__invoke` method, routed to via the class name alone (`Route::post('/x', DoThing::class)`). Use it when a controller has exactly one responsibility — it keeps classes small and pairs well with action-based architecture.

**Q5. How do you bind a route parameter to a model by something other than the ID?**
A: Use `{model:column}` in the route (e.g. `{post:slug}`) or override `getRouteKeyName()` on the model to return the column name globally.

**Q6. How has controller middleware changed in Laravel 11/12?**
A: Laravel 11 added the `HasMiddleware` interface with a static `middleware()` method returning an array of strings or `Middleware` value objects (with `only:`/`except:`). The old constructor-based `$this->middleware('auth')->only(...)` was **removed** — the base controller no longer extends `Illuminate\Routing\Controller`, so that instance method is gone and calling it now errors. The static approach also avoids instantiating the controller just to read its middleware. Laravel 12 additionally added `->middlewareFor()` / `->withoutMiddlewareFor()` for per-method middleware directly on resource route registrations.

**Q7. Why should controllers be "thin," and where does the logic go?**
A: Controllers should only handle HTTP coordination so they stay testable and the logic stays reusable. Business logic goes into service classes, single-purpose action classes, model methods/scopes, jobs, and events; validation/authorization go into Form Requests; response shaping into API Resources.

**Q8. What happens if route model binding can't find the model?**
A: It throws a `ModelNotFoundException`, which Laravel's handler converts to a `404 Not Found` response automatically.

**Q9. How do you send a PUT or DELETE request from an HTML form?**
A: HTML forms only support GET/POST, so you use `method="POST"` with Blade's `@method('PUT')` / `@method('DELETE')`, which adds a hidden `_method` field that Laravel reads to spoof the verb.

**Q10. Why can't you use route caching with closures, and how do controllers fix it?**
A: `route:cache` serializes the route definitions, and PHP closures aren't serializable. Controllers are referenced by class name + method string, which *is* serializable, so they cache fine.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate controllers
php artisan make:controller PostController                  # basic
php artisan make:controller PostController -i               # invokable (__invoke)
php artisan make:controller PostController -r               # resource (7 methods)
php artisan make:controller PostController --api            # api resource (5 methods)
php artisan make:controller PostController -m Post -R       # bind model + form requests
php artisan make:controller Admin/PostController            # sub-namespace
php artisan make:request StorePostRequest                   # form request
php artisan make:resource PostResource                      # API resource (JSON shape)
php artisan route:list                                      # inspect routes
```

```php
// Routing
Route::get('/p/{post}', [PostController::class, 'show']);   // basic
Route::post('/provision', ProvisionServer::class);          // invokable
Route::resource('posts', PostController::class);            // web CRUD (7)
Route::apiResource('posts', PostController::class);         // API CRUD (5)
Route::resource('posts', PostController::class)->only(['index','show']);
Route::resource('a.b', BController::class)->shallow();      // nested shallow
Route::get('/p/{post:slug}', [PostController::class, 'show']); // bind by column
```

```php
// Request reading
$request->input('k', $default);  $request->query('k');  $request->all();
$request->only([...]);  $request->has('k');  $request->filled('k');
$request->validated();  $request->safe()->except('x');  $request->user();
$request->file('f');  $request->expectsJson();

// Responses
return view('v', $data);                       // HTML
return response()->json($data, 201);           // JSON + status
return redirect()->route('name', $param);      // redirect
return to_route('name')->with('status', '..'); // helper + flash
return response()->noContent();                // 204
abort(404); abort_if($cond, 403);
```

```php
// Laravel 11+ controller middleware
public static function middleware(): array {
    return [
        'auth',
        new Middleware('verified', only: ['store', 'update']),
        new Middleware('throttle:6,1', except: ['index']),
    ];
}
```

| Method | Verb | URI |
|--------|------|-----|
| index | GET | /res |
| create | GET | /res/create |
| store | POST | /res |
| show | GET | /res/{id} |
| edit | GET | /res/{id}/edit |
| update | PUT/PATCH | /res/{id} |
| destroy | DELETE | /res/{id} |

---

## 🧪 Mini Exercises

1. **Scaffold a full CRUD feature.** Generate a `--resource --model=Article --requests` controller, register it with `Route::resource`, and run `php artisan route:list --name=articles`. Verify all seven routes and their names appear.

2. **Convert to thin.** Take a `store` method that currently validates inline, creates a model, sends a notification, and busts a cache. Extract the post-validation work into an `App\Actions\CreateArticle` class injected into the method, leaving the controller under 10 lines.

3. **Invokable + binding.** Create an invokable controller `PublishArticle` that accepts an `Article` via route model binding (bound by `slug`), flips its `published_at` timestamp, and redirects back with a flash message. Wire it to `POST /articles/{article:slug}/publish`.

4. **API resource with correct status codes.** Build an `Api/CommentController --api`, register it with `apiResource`, return a `CommentResource` collection from `index`, a `201` on `store`, and a `204` from `destroy`. Confirm with `php artisan route:list --path=api/comments`.

5. **Method-scoped middleware.** Implement `HasMiddleware` on a controller so that `auth` applies to all methods, `throttle:3,1` applies only to `store`, and `index`/`show` are excluded from a `verified` middleware. Prove the scoping by reading `php artisan route:list`.
