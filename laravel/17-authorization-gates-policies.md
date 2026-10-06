# Authorization in Laravel: Gates & Policies

Authentication answers **"who are you?"** Authorization answers **"are you *allowed* to do this?"** This module is entirely about the second question: how Laravel lets you express, centralize, and enforce permission rules so that a logged-in user cannot, say, edit someone else's post or delete a record they don't own.

Laravel ships with two complementary primitives for authorization:

- **Gates** — simple, closure-based checks, great for actions not tied to a specific model (e.g. "view the admin dashboard").
- **Policies** — classes that group authorization logic *around a particular Eloquent model* (e.g. everything about who can view/update/delete a `Post`).

We'll build from the simplest inline check all the way to resource policies, hooks, custom denial messages, Blade directives, route middleware, and a real-world roles/permissions strategy with `spatie/laravel-permission`.

> Targets: **PHP 8.4** and **Laravel 12**. Differences for PHP 8.1–8.3 and Laravel 10/11 are called out inline. The most important framework change to know: **Laravel 11+ removed the `app/Providers/AuthServiceProvider.php` file** and the `$policies` array. Policies are now auto-discovered by naming convention. Manual registration still works via `Gate::policy()` in any service provider's `boot()`.

---

## **What you'll learn**

- Why authorization belongs in a centralized layer, not scattered `if` statements.
- How to define and consume **Gates** with `Gate::define`, `allows`, `denies`, and `authorize`.
- The **`before`/`after` hooks** and how super-admin bypass works.
- Returning rich **`Response` objects** with custom messages and status codes.
- How to generate, auto-discover, and write **Policies**, including the `before()` method and **guest (nullable user)** support.
- Every way to *enforce* a rule: controller `$this->authorize()`, `authorizeResource`, `$user->can()`, `@can`/`@canany` in Blade, the `can` route middleware, and `Gate::authorize`.
- How authorization integrates with **Form Request `authorize()`** and how a clean **roles/permissions** model (and `spatie/laravel-permission`) fits on top.

---

## 1. Why a dedicated authorization layer?

Imagine enforcing "only the author can update a post" with inline checks:

```php
// ❌ Scattered logic — duplicated, untestable, easy to forget
public function update(Request $request, Post $post)
{
    if ($post->user_id !== $request->user()->id) {
        abort(403);
    }
    // ...
}
```

The moment a second place needs the same rule (a Blade view hiding the "Edit" button, an API endpoint, a queued job), you copy-paste the condition. When the rule changes ("admins can also edit"), you must find every copy. This is exactly the kind of cross-cutting logic that drifts out of sync and causes security holes.

**Gates and Policies solve this** by giving the rule *one home* and *one name* (an "ability" like `update`). Everywhere else — controllers, Blade, middleware, jobs — you just *ask* by name. Change the rule once; every caller stays correct.

The two tools divide along one axis:

| | **Gate** | **Policy** |
|---|---|---|
| Shape | A named closure | A class of methods |
| Best for | Actions not tied to one model (`view-dashboard`, `access-billing`) | All rules around one model type (`Post`, `Invoice`) |
| Registration | `Gate::define('name', fn)` | Auto-discovered by name, or `Gate::policy(Model::class, Policy::class)` |
| Lives in | A service provider | `app/Policies/*.php` |

Both are powered by the same underlying **`Gate` service** (`Illuminate\Auth\Access\Gate`), so the consumption API (`can`, `authorize`, `@can`, etc.) is identical regardless of whether the rule is a gate closure or a policy method.

---

## 2. Gates

### 2.1 Defining a gate

Gates are defined as closures that receive the authenticated user as the first argument and return a boolean (or a `Response`, covered later). In Laravel 11/12 there is no `AuthServiceProvider` by default, so define gates in the `boot()` method of any service provider — `App\Providers\AppServiceProvider` is the conventional spot.

```php
<?php
// app/Providers/AppServiceProvider.php

namespace App\Providers;

use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // A simple gate not tied to any model.
        Gate::define('view-admin-dashboard', function (User $user) {
            return $user->is_admin;
        });

        // A gate that also takes a model instance as an extra argument.
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id;
        });
    }
}
```

> **Laravel 10 note:** there you would put these in `App\Providers\AuthServiceProvider::boot()`. The closure signatures are identical — only the file differs.

You can also point a gate at a class method instead of a closure using the **callback array** syntax — handy when logic grows:

```php
Gate::define('update-post', [\App\Policies\PostPolicy::class, 'update']);
```

### 2.2 Consuming a gate: `allows`, `denies`, `authorize`

```php
use Illuminate\Support\Facades\Gate;

// Boolean checks — return true/false, never throw.
if (Gate::allows('update-post', $post)) {
    // current user may update
}

if (Gate::denies('update-post', $post)) {
    abort(403);
}

// Throwing check — throws AuthorizationException (→ HTTP 403) on failure.
Gate::authorize('update-post', $post);
```

Three things to notice:

1. **You never pass the user.** The gate automatically uses `Auth::user()` (the currently authenticated user). To check a *different* user, use `Gate::forUser($otherUser)->allows(...)`.
2. **`allows`/`denies` are non-throwing**; `authorize` throws `Illuminate\Auth\Access\AuthorizationException`, which Laravel's exception handler renders as a **403 Forbidden**.
3. **Extra arguments** after the ability name are forwarded to the closure. For multiple extras, pass an array: `Gate::allows('move', [$post, $targetFolder])`.

Output of `Gate::authorize('update-post', $post)` when the user is *not* the author:

```
Illuminate\Auth\Access\AuthorizationException: This action is unauthorized.
// → rendered as HTTP 403 in a web request
```

### 2.3 Checking multiple abilities: `any` / `none`

```php
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // user can do at least one
}

if (Gate::none(['update-post', 'delete-post'], $post)) {
    // user can do neither
}
```

### 2.4 `before` and `after` hooks (super-admin bypass)

The **`before`** hook runs *before* every gate/policy check. If it returns a non-null value, that value short-circuits the whole check. This is the canonical way to give super-admins blanket access.

```php
Gate::before(function (User $user, string $ability) {
    if ($user->is_super_admin) {
        return true; // grant everything, skip the individual check
    }
    // return null (implicitly) to fall through to the normal check
});
```

The **`after`** hook runs *after* all other checks. The official rule (Laravel 11/12) is precise: **a value returned by an `after` callback will NOT override the result of the authorization check unless the gate or policy returned `null`** (i.e. expressed "no opinion"). In other words, `after` can only fill in a verdict when nothing else decided — it cannot flip an explicit `true`/`false` produced by the gate/policy.

```php
Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
    if ($user->hasRole('auditor') && str_starts_with($ability, 'view-')) {
        return true; // applies only if the ability check itself returned null
    }
    // returning null here leaves the original $result untouched
});
```

Note the parameter type for `$result` is `bool|null` (it may be `null` when no gate/policy made a decision), and `$arguments` is `mixed` (the array of extra arguments passed to the check).

> **Gotcha:** Returning `false` from `before` is a hard deny that also short-circuits. If you only want to *grant* in `before`, return `true` or `null` — never `false` unless you truly mean "block everyone here."

### 2.5 Inline gates (no pre-definition)

For one-off logic you don't want to name globally, Laravel provides **`Gate::allowIf`** and **`Gate::denyIf`**. These are *throwing* checks: if the closure does not authorize the action (or no user is authenticated), Laravel throws an `AuthorizationException` (→ HTTP 403). They do **not** return a boolean.

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->id === $post->user_id);

Gate::denyIf(fn (User $user) => $user->isBanned());
```

> **Important:** `Gate::allows(...)` / `Gate::denies(...)` accept a *named* ability string, **not** a closure. There is no `Gate::allows(closure)` form — passing a closure to `allows`/`denies` is a common mistake. For inline, closure-based checks use `allowIf`/`denyIf`.
>
> **Caveat:** Inline authorization via `allowIf`/`denyIf` does **not** execute the `before` or `after` hooks. If you rely on a super-admin `Gate::before` bypass, an inline check will *not* honor it.

Use inline checks sparingly — naming abilities is what makes them reusable across controllers, Blade, and middleware.

### 2.6 Resource gates

`Gate::resource` defines a set of CRUD-style abilities at once. By default it maps **five** abilities — `viewAny`, `view`, `create`, `update`, `delete` — to methods on a backing class, each prefixed by the resource name. You can override the map with a third argument.

```php
Gate::resource('photos', \App\Policies\PhotoPolicy::class);
// Defines: photos.viewAny, photos.view, photos.create,
// photos.update, photos.delete — each delegating to the matching method.

// Customizing the ability → method map:
Gate::resource('photos', PhotoPolicy::class, [
    'list' => 'viewAny',
    'show' => 'view',
]);
```

> **Note:** `Gate::resource` is a real framework method but it is **not covered in the official Laravel docs** and is rarely used. If you're already thinking in CRUD-per-model terms, a **Policy** (next) — with `authorizeResource` on the controller — is the idiomatic, documented choice. Reach for resource gates only in unusual cases where you want gate-style registration with resource-shaped names.

---

## 3. Policies

A **Policy** is a class whose methods are *abilities* for one model. Each method receives the user and (usually) the model instance and returns `bool` or a `Response`.

### 3.1 Generating a policy

```bash
# Empty policy
php artisan make:policy PostPolicy

# Pre-filled with CRUD ability stubs (viewAny, view, create, update, delete, restore, forceDelete)
php artisan make:policy PostPolicy --model=Post
```

This creates `app/Policies/PostPolicy.php`.

### 3.2 Auto-discovery vs. manual registration

**Laravel 11/12 auto-discovers policies** by convention: for `App\Models\Post`, it looks for `App\Policies\PostPolicy`. The rule (per the official docs): take the model's class name, append the `Policy` suffix, and look in a `Policies` directory located **at or above** the directory that contains the model. Concretely, for a model in `app/Models`, Laravel checks `app/Models/Policies` first, then `app/Policies` (so `App\Models\Post` → `App\Policies\PostPolicy`).

If your naming doesn't follow the convention, register explicitly in a service provider's `boot()`:

```php
use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::policy(Post::class, PostPolicy::class);
}
```

You can also override the discovery logic globally with `Gate::guessPolicyNamesUsing(fn ($modelClass) => ...)`.

Laravel 11/12 adds a third option: the **`#[UsePolicy]` attribute** placed directly on the model, which pins the policy without touching a service provider:

```php
use App\Policies\PostPolicy;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;
use Illuminate\Database\Eloquent\Model;

#[UsePolicy(PostPolicy::class)]
class Post extends Model
{
    // ...
}
```

> **Laravel 10 note:** auto-discovery existed but the canonical place to *list* policies was the `protected $policies = [...]` array in `AuthServiceProvider`. That array is gone in 11/12; use `Gate::policy()` if you need manual mapping.

### 3.3 Anatomy of a policy

```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class PostPolicy
{
    // Anyone logged in can see the index.
    public function viewAny(User $user): bool
    {
        return true;
    }

    // Anyone can view a published post; only the author can view a draft.
    public function view(User $user, Post $post): bool
    {
        return $post->published || $user->id === $post->user_id;
    }

    public function create(User $user): bool
    {
        return $user->hasVerifiedEmail();
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    // Returning a Response lets us attach a human-readable message.
    public function delete(User $user, Post $post): Response
    {
        return $user->id === $post->user_id
            ? Response::allow()
            : Response::deny('You may only delete your own posts.');
    }
}
```

**Ability → method mapping.** When you call `$user->can('update', $post)` or `$this->authorize('update', $post)`, Laravel finds the policy for `$post`'s class and invokes the method **named exactly like the ability** (`update`). The mapping is literal string-to-method-name; there is no magic pluralization.

Standard ability names Laravel's resource controllers and `authorizeResource` expect:

| Controller action | Policy method | Model passed? |
|---|---|---|
| `index` | `viewAny` | No |
| `show` | `view` | Yes |
| `create` / `store` | `create` | No |
| `edit` / `update` | `update` | Yes |
| `destroy` | `delete` | Yes |
| (soft deletes) | `restore` / `forceDelete` | Yes |

> **`viewAny` and `create` receive no model instance** — there isn't one yet. Their methods take only `User $user`. Forgetting this and typing `view(User $user, Post $post)` for the *index* ability is a common slip.

### 3.4 The `before()` method (per-policy bypass)

Just like the global `Gate::before`, a policy can define a `before` method that runs ahead of every method *in that policy*. Return `true`/`false` to short-circuit, or `null` to fall through.

```php
public function before(User $user, string $ability): ?bool
{
    if ($user->is_admin) {
        return true; // admins bypass every check in this policy
    }

    return null; // otherwise, run the requested ability method
}
```

> **Critical gotcha:** if `before` returns `false`, it denies *everything* — including abilities you intended to allow. Always return `null` (not `false`) for the "I have no opinion, continue" case.

> **Subtle behavior (Laravel docs):** a policy's `before` method is **not** called if the policy class doesn't contain a method whose name matches the ability being checked. So a `before` that grants admins everything will only fire for abilities that have a corresponding policy method — it won't magically authorize an ability the policy never declared. (The global `Gate::before` hook does still run for undefined abilities.)

(The official docs type-hint this method as `bool|null`, which is identical to the `?bool` shown above.)

### 3.5 Guest users (nullable user)

By default, if there is no authenticated user, the gate/policy check **returns `false` automatically** without even invoking your closure/method. If you want to allow *guests* to pass certain checks, type-hint the user parameter as **nullable** (`?User $user`). Laravel sees the nullable hint and runs the method even for unauthenticated requests.

```php
public function view(?User $user, Post $post): bool
{
    // Guests may view published posts; only the author sees drafts.
    if ($post->published) {
        return true;
    }

    return $user?->id === $post->user_id; // $user is null for guests
}
```

The `?User` hint plus the null-safe operator (`$user?->id`) is the modern, crash-safe way to handle this.

### 3.6 Policy responses with messages and status

`Illuminate\Auth\Access\Response` lets a denial carry a message, an error code, and even a custom HTTP status.

```php
use Illuminate\Auth\Access\Response;

public function update(User $user, Post $post): Response
{
    if ($user->id !== $post->user_id) {
        return Response::deny('Only the author may edit this post.', 'POST_NOT_OWNED');
    }

    if ($post->locked) {
        // denyWithStatus returns 404 instead of 403 — useful to "hide" existence.
        return Response::denyWithStatus(404);
        // Shorthand for the common 404 case:
        // return Response::denyAsNotFound();
    }

    return Response::allow();
}
```

Because hiding a resource behind a `404` is such a common pattern, Laravel offers **`Response::denyAsNotFound()`** as a convenience shorthand for `Response::denyWithStatus(404)`.

When such a denial reaches `$this->authorize(...)` or `Gate::authorize(...)`, the `AuthorizationException` carries the message, and the response is rendered with that message and status. You can read the message back via `$response->message()`.

To inspect a response without throwing:

```php
$response = Gate::inspect('update', $post);

if ($response->allowed()) {
    // ...
} else {
    echo $response->message(); // "Only the author may edit this post."
}
```

---

## 4. Enforcing authorization

You've defined the rules; now *consume* them. Every method below routes through the same Gate, so a gate ability and a policy ability are interchangeable from the caller's perspective.

### 4.1 In controllers: `$this->authorize()`

In Laravel 11/12, the `App\Http\Controllers\Controller` base class **no longer includes the `AuthorizesRequests` trait by default**. To use `$this->authorize()`, add the trait yourself:

```php
<?php
// app/Http/Controllers/Controller.php

namespace App\Http\Controllers;

use Illuminate\Foundation\Auth\Access\AuthorizesRequests;

abstract class Controller
{
    use AuthorizesRequests;
}
```

Then:

```php
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post); // throws 403 if denied

    $post->update($request->validated());

    return redirect()->route('posts.show', $post);
}
```

For abilities without a model (e.g. `create`), pass the **class name** so Laravel can locate the right policy:

```php
$this->authorize('create', Post::class);
```

> **Laravel 10 note:** the base controller *did* include `AuthorizesRequests` automatically, so `$this->authorize()` worked out of the box. In 11/12 you opt in.

### 4.2 `authorizeResource` — one call wires a whole controller

For a standard resource controller, `authorizeResource` maps each controller method to its policy ability automatically (using the table in §3.3).

```php
class PostController extends Controller
{
    public function __construct()
    {
        // Model class, route parameter name.
        $this->authorizeResource(Post::class, 'post');
    }

    public function index() { /* viewAny checked automatically */ }
    public function show(Post $post) { /* view checked automatically */ }
    public function update(Request $request, Post $post) { /* update checked */ }
    // ...
}
```

The route parameter (`'post'`) must match the resource route's parameter, and your route definition must be a resource route: `Route::resource('posts', PostController::class)`.

> **Tip (from the Laravel docs):** generate the controller with `php artisan make:controller PostController --model=Post --resource` so each method already has the right type-hints (`Post $post`) and signatures that `authorizeResource` expects. Under the hood, `authorizeResource` attaches the `can` middleware to each action; the action→ability mapping is: `index→viewAny`, `create→create`, `store→create`, `show→view`, `edit→update`, `update→update`, `destroy→delete`.

### 4.3 On the user model: `can` / `cannot`

```php
$user = $request->user();

if ($user->can('update', $post)) { /* ... */ }
if ($user->cannot('delete', $post)) { abort(403); }

// Class name for model-less abilities:
if ($user->can('create', Post::class)) { /* ... */ }

// canAny returns true if AT LEAST ONE of the abilities passes.
$user->canAny(['update', 'delete'], $post);
```

These come from the `Illuminate\Foundation\Auth\Access\Authorizable` trait, which `App\Models\User` gets via `Illuminate\Foundation\Auth\User` (the `Authenticatable` base class). They're non-throwing booleans — perfect for conditionals and view logic.

### 4.4 In Blade: `@can`, `@cannot`, `@canany`

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@elsecan('view', $post)
    <a href="{{ route('posts.show', $post) }}">View</a>
@endcan

@cannot('delete', $post)
    <p>You cannot delete this post.</p>
@endcannot

{{-- Model-less ability --}}
@can('create', App\Models\Post::class)
    <a href="{{ route('posts.create') }}">New Post</a>
@endcan

{{-- Passes if ANY ability is allowed --}}
@canany(['update', 'delete'], $post)
    <div class="admin-actions">…</div>
@endcanany
```

These directives are sugar over `$user->can(...)`. They render the inner block only when authorized, which is how you keep UI in lockstep with backend rules. **Never rely on hiding a button alone for security** — always pair it with a server-side `authorize` on the action.

### 4.5 Route middleware: the `can` middleware

You can gate an entire route without touching the controller:

```php
use App\Models\Post;

// Model-bound ability — Laravel resolves {post} via route-model binding
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('can:update,post');

// Model-less ability — pass the class name as a string
Route::post('/posts', [PostController::class, 'store'])
    ->middleware('can:create,' . Post::class);
```

In Laravel 11/12 there's also a fluent helper, `->can()`, that reads better:

```php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->can('update', 'post');

Route::post('/posts', [PostController::class, 'store'])
    ->can('create', Post::class);
```

If the check fails, the middleware throws `AuthorizationException` → **403** before your controller ever runs.

### 4.6 `Gate::authorize` (facade, anywhere)

Outside controllers (jobs, commands, services) you don't have `$this->authorize()`. Use the facade:

```php
use Illuminate\Support\Facades\Gate;

Gate::authorize('update', $post); // throws on failure
```

### 4.7 Returning 403 explicitly

When you want a manual deny without a gate, `abort(403)` or the `Response` helpers work:

```php
abort_if($post->locked, 403, 'This post is locked.');
abort_unless($request->user()->is_admin, 403);
```

Any thrown `AuthorizationException` (with or without a message) is converted to a 403 response by the framework's exception handler — JSON for API requests (`{"message": "..."}`), an HTML error page for web.

---

## 5. Combining with a Form Request `authorize()`

Form Requests (`php artisan make:request StorePostRequest`) have an `authorize()` method that runs **before validation**. Return `false` (or a denial `Response`) to abort with 403. This is the natural place to put the authorization for a *write* endpoint because it keeps the rule next to the validation rules.

```php
<?php
// app/Http/Requests/UpdatePostRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Gate;

class UpdatePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Delegate to the policy so the rule has one home.
        return Gate::allows('update', $this->route('post'));
        // Equivalent: $this->user()->can('update', $this->route('post'));
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required', 'string'],
        ];
    }
}
```

> **Best practice:** don't *reimplement* the rule inside `authorize()`. **Delegate** to the policy/gate (`Gate::allows`, `$this->user()->can`). That preserves the single-source-of-truth principle. If `authorize()` returns `false`, validation never runs and the user gets a 403.

> **Default:** if you **omit** the `authorize()` method entirely, the base `FormRequest` treats the request as authorized (the parent method returns `true`). In modern Laravel (10/11/12), `php artisan make:request` generates a stub whose `authorize()` already returns `true`; much older versions generated `return false;`. Either way, be explicit — leaving an accidental `return false;` will silently 403 every request to that endpoint.

---

## 6. Roles & permissions, and `spatie/laravel-permission`

Gates and policies answer *"can this user do X to this thing?"* They don't, by themselves, model **roles** (admin, editor) or fine-grained **permissions** (`posts.publish`). You typically layer roles/permissions *underneath* policies: the policy method asks the user about a permission, and the permission system answers.

A hand-rolled minimal version:

```php
// In a policy
public function publish(User $user, Post $post): bool
{
    return $user->hasPermission('posts.publish');
}
```

For anything beyond toy scale, use the de-facto standard package **`spatie/laravel-permission`**. It stores roles and permissions in the database, adds traits to your `User`, and integrates cleanly with Laravel's Gate (so `@can`, `$user->can`, and policies all work).

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

```php
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles; // adds roles(), permissions(), hasRole(), hasPermissionTo(), ...
}
```

```php
// Seeding roles/permissions
use Spatie\Permission\Models\{Role, Permission};

$editor = Role::create(['name' => 'editor']);
$publish = Permission::create(['name' => 'posts.publish']);
$editor->givePermissionTo($publish);

$user->assignRole('editor');

// Now this works everywhere because the package registers permissions as gates:
$user->can('posts.publish');   // true
```

```blade
@role('editor')
    <button>Publish</button>
@endrole

@haspermission('posts.publish')
    <button>Publish</button>
@endhaspermission
```

The package auto-registers each permission name as a Gate ability, which is why `$user->can('posts.publish')` and `@can('posts.publish')` "just work" alongside your own policies. A common, clean architecture: **roles/permissions in spatie → consumed inside your own policy methods → enforced via `authorize`/`@can`.** Policies stay the single source of truth for "what does this action require," and spatie answers "does this user have it."

---

## ⚠️ Common Mistakes & Gotchas

1. **Returning `false` from `before()` to mean "no opinion."**
   A `before` hook (global or per-policy) that returns `false` **denies everything** and short-circuits the real check. Always return `null` for "fall through to the normal logic"; only return `false`/`true` when you truly intend a blanket deny/grant.
   *Fix:* `return $user->is_admin ? true : null;`

2. **Type-hinting a non-nullable `User` and being surprised guests are auto-denied.**
   If the user param is `User $user` (not nullable), Laravel returns `false` for unauthenticated requests *without running your method*. So your "anyone can view published posts" logic never executes for guests.
   *Fix:* use `?User $user` and `$user?->id` when guests should be considered.

3. **Forgetting that `$this->authorize()` requires the `AuthorizesRequests` trait in Laravel 11/12.**
   The base controller no longer includes it. Calling `$this->authorize()` throws "Call to undefined method."
   *Fix:* add `use AuthorizesRequests;` to `app/Http/Controllers/Controller.php`.

4. **Passing an instance where a class name is required (or vice-versa).**
   For model-less abilities (`create`, `viewAny`) you must pass the **class name**: `$this->authorize('create', Post::class)`. Passing `$post` there makes Laravel call `create(User, Post)` which doesn't match the no-model signature.
   *Fix:* pass `Post::class` for `create`/`viewAny`; pass the instance for `view`/`update`/`delete`.

5. **Relying on `@can` to "secure" an action.**
   Hiding a button in Blade is UX, not security — anyone can `curl` the endpoint. The route/controller is still wide open if you don't also `authorize`.
   *Fix:* always enforce on the server (`$this->authorize`, `can` middleware, or Form Request `authorize`), and use `@can` only to tidy the UI.

6. **Reimplementing the rule inside `FormRequest::authorize()`.**
   Duplicating `$post->user_id === $this->user()->id` there reintroduces the drift problem policies were meant to fix.
   *Fix:* delegate — `return $this->user()->can('update', $this->route('post'));`.

7. **Expecting auto-discovery to find a misnamed policy.**
   `App\Models\Post` maps to `App\Policies\PostPolicy`. If you name it `PostsPolicy` or put it elsewhere, discovery silently fails and checks default to denied/undefined behavior.
   *Fix:* follow the convention, or register with `Gate::policy(Post::class, PostsPolicy::class)`. (You can also pin a policy with the `#[UsePolicy(PostPolicy::class)]` attribute on the model.)

8. **Passing a closure to `Gate::allows()` / `Gate::denies()`.**
   `allows`/`denies`/`authorize` take an ability **name** (a string), not a closure. For closure-based one-off checks use `Gate::allowIf(...)` / `Gate::denyIf(...)` — and note those *throw* on failure and **skip** the `before`/`after` hooks (so a super-admin `Gate::before` bypass won't apply to them).
   *Fix:* `Gate::allowIf(fn (User $u) => $u->id === $post->user_id);`

9. **Assuming a policy's `before()` runs for an ability the policy doesn't define.**
   A policy's `before` method is *not* invoked unless the policy contains a method matching the ability name. An admin-bypass `before` therefore won't authorize abilities that have no corresponding policy method.
   *Fix:* define the ability method, or use the global `Gate::before` (which runs even for undefined abilities).

---

## ✅ Best Practices

- **One rule, one home.** Define each ability once (gate or policy) and *ask by name* everywhere else.
- **Policies for models, gates for everything else.** If the rule centers on a model type, make a policy; if it's a standalone capability (`access-telescope`), a gate is cleaner.
- **Prefer named abilities over inline gates** so they're reusable and testable.
- **Use `before()` for super-admin bypass**, and always return `null` (not `false`) for fall-through.
- **Return `Response::deny('message')`** for user-facing denials so the 403 carries a helpful, secure message (avoid leaking sensitive detail; use `denyWithStatus(404)` to hide existence).
- **Enforce on the server, decorate with Blade.** `@can` is for UI only; back it with `authorize`/middleware.
- **Delegate Form Request `authorize()` to the policy** instead of duplicating logic.
- **Layer roles/permissions beneath policies.** Policies express *what an action requires*; the permission system answers *whether the user has it*. Reach for `spatie/laravel-permission` rather than hand-rolling at scale.
- **Test your policies** directly: `$this->assertTrue($user->can('update', $post))` in feature tests, plus assert 403s on endpoints.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between authentication and authorization, and which do Gates/Policies handle?**
Authentication verifies *identity* (login, sessions, tokens). Authorization verifies *permission* — what an authenticated user may do. Gates and Policies are purely authorization; they assume a user (possibly null) and decide allow/deny.

**Q2. When would you choose a Gate over a Policy?**
A Policy when the rule is about a specific model type and you have a CRUD-ish set of abilities (it groups them in one class with auto-discovery). A Gate for standalone actions not tied to a model — e.g. `view-admin-dashboard`, `access-billing` — or one-off inline checks.

**Q3. How do Policies get registered in Laravel 11/12 without `AuthServiceProvider`?**
By **auto-discovery**: `App\Models\Post` → `App\Policies\PostPolicy`. The framework derives the policy name from the model name and a sibling `Policies` directory. For non-conventional names, call `Gate::policy(Model::class, Policy::class)` in any provider's `boot()`. You can override discovery with `Gate::guessPolicyNamesUsing()`.

**Q4. How does authorization work under the hood — what happens when I call `$user->can('update', $post)`?**
`can()` (from the `Authorizable` trait) forwards to the `Gate` instance bound in the container. The Gate's `raw()` resolution: (1) runs all `before` callbacks — a non-null result wins immediately; (2) otherwise resolves the *callback* for the ability — if the first arg is a model with a registered/discovered policy, it instantiates the policy (resolved through the container, so policies can have constructor dependencies) and calls the method matching the ability name; if it's a plain gate, it calls the closure; (3) runs `after` callbacks, which may override a non-definitive result. The raw result is normalized to a `Response` (`allow`/`deny`), and `can()` returns its boolean. `authorize()` does the same but throws `AuthorizationException` (→ 403) on a denied `Response`.

**Q5. How do you support guest (unauthenticated) users in a policy?**
Type-hint the user as nullable: `?User $user`. Without the nullable hint, Laravel auto-returns `false` for guests and never calls the method. With it, the method runs and you handle `null` (e.g. `$user?->id === $post->user_id`).

**Q6. What does the `before()` method do, and what's the danger?**
It runs before every ability in that policy (or globally for `Gate::before`). Returning `true`/`false` short-circuits; returning `null` falls through. The danger: returning `false` for the "no opinion" case denies *everything*, including allowed abilities. Return `null` to continue.

**Q7. How does `authorizeResource` know which policy method to call?**
It maps resource controller actions to ability names by convention — `index→viewAny`, `show→view`, `store→create`, `update→update`, `destroy→delete` — and calls `$this->authorize()` for each in the right place, using the route-model-bound instance (or the class name for model-less abilities).

**Q8. How do you return a custom message or a 404 instead of 403 from a denial?**
Return an `Illuminate\Auth\Access\Response`: `Response::deny('message', 'code')` carries a message; `Response::denyWithStatus(404)` changes the HTTP status (useful to hide a resource's existence). The message surfaces through `AuthorizationException` and `Gate::inspect(...)->message()`.

**Q9. Where does `FormRequest::authorize()` fit, and how should it interact with policies?**
It runs before validation; returning `false` aborts with 403. Best practice is to *delegate* to the policy (`$this->user()->can('update', $this->route('post'))`) rather than duplicate the rule, keeping a single source of truth.

**Q10. How does `spatie/laravel-permission` integrate with Laravel's Gate?**
It registers every permission name as a Gate ability, so `$user->can('posts.publish')`, `@can('posts.publish')`, and even policy methods that check `hasPermissionTo()` all resolve through the same Gate. Roles/permissions live in the DB; policies typically consume them.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ---- Define ----
Gate::define('ability', fn (User $u, $model = null) => bool|Response);
Gate::resource('photos', PhotoPolicy::class);
Gate::policy(Post::class, PostPolicy::class);     // manual policy registration
Gate::before(fn (User $u, string $a) => true|null);   // super-admin bypass
Gate::after(fn (User $u, string $a, ?bool $r, $args) => ?bool);

// ---- Check (boolean, no throw) ----
Gate::allows('ability', $model);
Gate::denies('ability', $model);
Gate::any(['a','b'], $model);  Gate::none(['a','b'], $model);
Gate::forUser($other)->allows('ability', $model);
$user->can('ability', $model);     $user->cannot('ability', $model);
$user->canAny(['a','b'], $model);
Gate::inspect('ability', $model)->allowed(); // ->message()

// ---- Check (throws AuthorizationException → 403) ----
Gate::authorize('ability', $model);
$this->authorize('ability', $model);            // needs AuthorizesRequests trait
$this->authorize('create', Post::class);        // model-less
$this->authorizeResource(Post::class, 'post');  // in controller __construct

// ---- Inline (closure-based, THROWS on deny; skips before/after hooks) ----
Gate::allowIf(fn (User $u) => $u->id === $model->user_id);
Gate::denyIf(fn (User $u) => $u->isBanned());
// NOTE: Gate::allows()/denies() take an ability NAME, never a closure.
```

```php
// ---- Policy / gate responses ----
return Response::allow();
return Response::deny('Not allowed', 'CODE');
return Response::denyWithStatus(404);          // custom status
return Response::denyAsNotFound();             // shorthand for denyWithStatus(404)
```

```blade
@can('update', $post) … @elsecan('view', $post) … @endcan
@cannot('delete', $post) … @endcannot
@canany(['update','delete'], $post) … @endcanany
@can('create', App\Models\Post::class) … @endcan
```

```php
// ---- Route middleware ----
->middleware('can:update,post');                 // model-bound
->middleware('can:create,' . Post::class);       // model-less
->can('update', 'post');                          // fluent (L11/12)
```

```bash
# ---- Artisan ----
php artisan make:policy PostPolicy
php artisan make:policy PostPolicy --model=Post
php artisan make:request UpdatePostRequest
```

**Ability ↔ method ↔ controller action:**
`index→viewAny` · `show→view` · `store→create` · `update→update` · `destroy→delete` · (`restore`, `forceDelete` for soft deletes). `viewAny`/`create` take **no model**.

---

## 🧪 Mini Exercises

1. **Gate + hook.** Define a gate `manage-settings` that allows only users where `is_admin` is true. Add a global `Gate::before` so any user with `is_super_admin` bypasses *all* gates. Verify (in tinker or a test) that a non-admin is denied, an admin is allowed, and a super-admin is allowed even for unrelated abilities.

2. **Full policy.** Create a `Comment` model and `CommentPolicy` with `viewAny`, `view`, `create`, `update`, and `delete`. The author may update/delete their own comment; anyone logged in may create; published comments are viewable by guests. Use a nullable user where appropriate and a `before()` that lets admins do anything (returning `null` for non-admins).

3. **Enforce everywhere.** Wire a `CommentController` with `authorizeResource`, hide the Edit/Delete links in Blade with `@can`, and add a `can:` route middleware to the destroy route. Confirm a non-owner gets a 403 from the endpoint even when the UI links are hidden.

4. **Custom denial.** Change `CommentPolicy::delete` to return `Response::deny('You can only delete your own comments.', 'NOT_OWNER')`. Hit the endpoint as a non-owner and confirm the 403 body contains your message (test both a web and a JSON request).

5. **Roles layer.** Install `spatie/laravel-permission`, create roles `editor` and `admin` and a permission `comments.moderate`. Give `editor` that permission, assign it to a user, and update `CommentPolicy::delete` so users with `comments.moderate` may delete *any* comment. Confirm `$user->can('comments.moderate')` and the policy both reflect the assignment.
