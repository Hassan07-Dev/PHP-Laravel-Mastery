# API Development & Sanctum in Laravel 12

Building HTTP APIs is the bread and butter of backend work. In this module you'll learn how Laravel turns controllers and Eloquent models into clean JSON APIs, how to authenticate clients with **Laravel Sanctum**, and how to handle the messy realities — errors, pagination, rate limits, and CORS — that separate a toy endpoint from a production-grade service.

**What you'll learn**

- How to design RESTful APIs in Laravel: resourceful routes, HTTP status codes, JSON responses, and URL versioning.
- The role of `routes/api.php` and the `api` middleware group (stateless, throttled).
- Shaping responses with **API Resources** (`JsonResource`, `ResourceCollection`) and paginating them.
- API-friendly **exception handling** in `bootstrap/app.php` (`withExceptions`) — returning JSON for 404/422/500.
- **Rate limiting** with `RateLimiter::for`, the `throttle` middleware, and the `X-RateLimit-*` headers.
- **CORS** configuration via `config/cors.php`.
- **Sanctum** end to end: API tokens (`createToken`, abilities/scopes, `tokenCan`, revoking) and SPA cookie auth (stateful domains, CSRF), plus `auth:sanctum`.
- **Sanctum vs Passport**, and writing fast API tests with `getJson`/`postJson`/`assertJson`.

---

## 1. Why APIs, and why REST?

An **API** (Application Programming Interface) is a contract that lets one program talk to another. A **web API** does this over HTTP. **REST** (REpresentational State Transfer) is a set of conventions for designing such APIs around **resources** (nouns like `users`, `posts`) that you manipulate with **HTTP verbs** (GET, POST, PUT/PATCH, DELETE).

The "why" matters: REST is popular because it's predictable. Once you know an API is RESTful, you can *guess* the URLs and methods. `GET /api/posts` lists posts; `POST /api/posts` creates one; `GET /api/posts/5` shows one; `DELETE /api/posts/5` removes it. No documentation lookup required for the basics.

### The verb → action → status mapping

| Verb     | URL                | Action  | Typical success status      |
|----------|--------------------|---------|-----------------------------|
| `GET`    | `/api/posts`       | index   | `200 OK`                    |
| `POST`   | `/api/posts`       | store   | `201 Created`               |
| `GET`    | `/api/posts/{id}`  | show    | `200 OK`                    |
| `PUT`/`PATCH` | `/api/posts/{id}` | update | `200 OK`                  |
| `DELETE` | `/api/posts/{id}`  | destroy | `204 No Content`            |

`PUT` means "replace the whole resource"; `PATCH` means "partially update." Laravel routes both to the same `update` action.

### Status codes you must know

- **2xx Success:** `200 OK`, `201 Created`, `204 No Content` (success, empty body).
- **3xx Redirect:** rare in JSON APIs.
- **4xx Client error:** `400 Bad Request`, `401 Unauthorized` (not authenticated), `403 Forbidden` (authenticated but not allowed), `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity` (validation failed), `429 Too Many Requests` (rate limited).
- **5xx Server error:** `500 Internal Server Error`, `503 Service Unavailable`.

> **Jargon check.** `401` vs `403`: 401 says "I don't know who you are — log in." 403 says "I know who you are, but you can't do this."

---

## 2. `routes/api.php` and the `api` middleware group

In Laravel 11 and 12, `routes/api.php` does **not** exist by default in a fresh app. You opt in by running:

```bash
php artisan install:api
```

This command (introduced in Laravel 11) creates `routes/api.php`, registers it in `bootstrap/app.php` via `withRouting(api: ...)`, **and installs Sanctum** (requires the `laravel/sanctum` package and publishes its `personal_access_tokens` migration). You still add the `HasApiTokens` trait to your `User` model yourself (see Section 9). In Laravel 10 and earlier, `routes/api.php` shipped by default and routing was configured in `app/Providers/RouteServiceProvider.php`, which no longer exists.

After `install:api`, your `bootstrap/app.php` looks like this:

```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',   // <-- added by install:api
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

### What's special about the `api` group?

Routes registered via the `api:` key are automatically:

1. **Prefixed with `/api`.** `Route::get('/posts', ...)` becomes `GET /api/posts`. You can change the prefix with `apiPrefix: 'api/admin'` in `withRouting()`.
2. **Stateless.** Unlike `web` routes, there's no session, no CSRF cookie middleware, and no `StartSession`. Each request stands alone — authentication is expected via a token header.
3. **Bound.** The group runs `SubstituteBindings` so route-model binding works.

> **Important (Laravel 11/12 change).** Unlike Laravel 10, the `api` group is **NOT throttled by default**. The `RateLimiter::for('api', ...)` definition lives in `AppServiceProvider::boot()`, but the `throttle:api` middleware is *not* auto-applied to the group. You opt in either per route/group (`->middleware('throttle:api')`) or globally by appending it to the `api` group in `bootstrap/app.php`:
>
> ```php
> ->withMiddleware(function (Middleware $middleware) {
>     $middleware->api(append: ['throttle:api']);   // apply the named "api" limiter to all api routes
> })
> ```

A minimal `routes/api.php`:

```php
<?php
// routes/api.php

use App\Http\Controllers\Api\PostController;
use Illuminate\Support\Facades\Route;

Route::get('/ping', fn () => response()->json(['message' => 'pong']));

// One line registers index/store/show/update/destroy:
Route::apiResource('posts', PostController::class);
```

`apiResource` is like `resource` but **omits** the `create` and `edit` routes (those return HTML forms, which APIs don't need). Run `php artisan route:list --path=api` to verify:

```bash
php artisan route:list --path=api
```

```
GET|HEAD   api/posts ............ posts.index   › PostController@index
POST       api/posts ............ posts.store   › PostController@store
GET|HEAD   api/posts/{post} ..... posts.show    › PostController@show
PUT|PATCH  api/posts/{post} ..... posts.update  › PostController@update
DELETE     api/posts/{post} ..... posts.destroy › PostController@destroy
GET|HEAD   api/ping
```

### Returning JSON

Any controller method that returns an array, an Eloquent model, or a collection is **automatically serialized to JSON** with a `200` status and `Content-Type: application/json`. For explicit control, use the `response()` helper:

```php
return response()->json(['data' => $post], 201);

// With custom headers:
return response()->json($payload, 200, ['X-Custom' => 'value']);
```

---

## 3. API versioning via a prefix

APIs evolve, and you can't break existing clients. The simplest, most common strategy is **URL prefix versioning** — putting `v1`, `v2` in the path so old and new can coexist.

```php
<?php
// routes/api.php

use Illuminate\Support\Facades\Route;

Route::prefix('v1')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V1\PostController::class);
});

Route::prefix('v2')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V2\PostController::class);
});
```

This yields `GET /api/v1/posts` and `GET /api/v2/posts`. Mirror the structure in your code: `app/Http/Controllers/Api/V1/` and `App\Http\Resources\V1\`. Header-based versioning (`Accept: application/vnd.myapp.v2+json`) exists too but is harder to test and debug, so prefix versioning is the pragmatic default for interviews and most teams.

---

## 4. API Resources: shaping the JSON

Returning a raw Eloquent model leaks your database schema — column names, internal flags, timestamps — straight to clients. **API Resources** are a transformation layer between your models and the JSON you emit. They let you rename fields, hide secrets, format values, and embed relationships consistently.

Generate one:

```bash
php artisan make:resource PostResource
php artisan make:resource PostCollection   # optional, for collection-level metadata
```

```php
<?php
// app/Http/Resources/PostResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id'           => $this->id,
            'title'        => $this->title,
            'body'         => $this->body,
            'is_published' => (bool) $this->published_at,        // rename + cast
            'published_at' => $this->published_at?->toIso8601String(),
            // Only include the author block if it was eager-loaded:
            'author'       => UserResource::make($this->whenLoaded('user')),
            'comment_count' => $this->whenCounted('comments'),
            'links'        => [
                'self' => route('posts.show', $this->id),
            ],
        ];
    }
}
```

`whenLoaded('user')` is the key idiom: the `author` key only appears if you eager-loaded the relation (`Post::with('user')`), preventing the **N+1 query problem** and keeping payloads lean. `whenCounted('comments')` works similarly with `withCount`.

Use it in the controller:

```php
<?php
// app/Http/Controllers/Api/PostController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\PostResource;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index(): \Illuminate\Http\Resources\Json\AnonymousResourceCollection
    {
        $posts = Post::with('user')->latest()->paginate(15);

        return PostResource::collection($posts);
    }

    public function show(Post $post): PostResource
    {
        return PostResource::make($post->load('user'));   // route-model binding
    }

    public function store(Request $request): PostResource
    {
        $data = $request->validate([
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required', 'string'],
        ]);

        $post = $request->user()->posts()->create($data);

        return PostResource::make($post)
            ->response()
            ->setStatusCode(201);   // 201 Created
    }
}
```

A single resource is wrapped in a `data` key by default:

```json
{
  "data": {
    "id": 1,
    "title": "Hello",
    "body": "World",
    "is_published": true,
    "published_at": "2026-06-18T10:00:00+00:00",
    "links": { "self": "http://localhost/api/posts/1" }
  }
}
```

> **Disabling the wrapper.** Call `JsonResource::withoutWrapping();` in a service provider's `boot()` if you don't want the `data` envelope. Be consistent — frontends depend on the shape.

---

## 5. Pagination in APIs

Never return an unbounded list — a table with a million rows will time out and exhaust memory. Eloquent's `paginate()` slices results and, when handed to a Resource collection, **automatically adds pagination metadata**.

```php
$posts = Post::latest()->paginate(perPage: 15);   // ?page=2 controlled by query string
return PostResource::collection($posts);
```

Output (note the auto-generated `links` and `meta`):

```json
{
  "data": [ /* 15 PostResource items */ ],
  "links": {
    "first": "http://localhost/api/posts?page=1",
    "last":  "http://localhost/api/posts?page=9",
    "prev":  null,
    "next":  "http://localhost/api/posts?page=2"
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 9,
    "path": "http://localhost/api/posts",
    "per_page": 15,
    "to": 15,
    "total": 130
  }
}
```

Three pagination flavors:

- **`paginate()`** — runs a `COUNT(*)` so you get `total` and `last_page`. Most common.
- **`simplePaginate()`** — no count query (faster); only `prev`/`next` links, no `total`.
- **`cursorPaginate()`** — cursor-based, O(1) regardless of page depth; ideal for infinite scroll and very large datasets. Returns opaque cursors instead of page numbers.

```php
return PostResource::collection(Post::latest('id')->cursorPaginate(15));
// meta contains "next_cursor"/"prev_cursor" base64 strings, no "total"
```

Let clients tune page size safely:

```php
$perPage = min((int) $request->integer('per_page', 15), 100);   // cap to 100
return PostResource::collection(Post::paginate($perPage));
```

---

## 6. Exception handling for APIs

Web apps render error *pages*; APIs must return error *JSON*. In Laravel 11/12 there is **no `app/Exceptions/Handler.php`** — all exception customization lives in `bootstrap/app.php` inside `withExceptions()`.

The good news: Laravel already does the right thing for JSON. When a request **expects JSON** — i.e. `$request->expectsJson()` is `true`, which happens when it sends `Accept: application/json` (or `X-Requested-With: XMLHttpRequest`) — the framework renders exceptions as JSON automatically:

- `ValidationException` → `422` with an `errors` object.
- `ModelNotFoundException` / `NotFoundHttpException` (e.g., route-model binding miss) → `404`.
- `AuthenticationException` → `401`.
- `AuthorizationException` / `AccessDeniedHttpException` → `403`.
- `ThrottleRequestsException` → `429`.
- Anything else → `500` (message hidden unless `APP_DEBUG=true`).

A `422` response from `$request->validate()` looks like:

```json
{
  "message": "The title field is required. (and 1 more error)",
  "errors": {
    "title": ["The title field is required."],
    "body":  ["The body field is required."]
  }
}
```

> **Gotcha.** The auto-JSON behavior depends on the request **expecting JSON via headers**, not on the URL prefix. Tools like `getJson()`/`postJson()` set `Accept: application/json` for you. But a browser **navigating** to an `/api/...` URL sends `Accept: text/html`, so by default it would receive an **HTML** error page (or a redirect on validation). If you want every `api/*` request to error in JSON regardless of headers, register `shouldRenderJsonWhen(fn ($req, $e) => $req->is('api/*'))` (shown below).

### Customizing error JSON

Use `withExceptions` to override the shape, log selectively, or remap exception types:

```php
<?php
// bootstrap/app.php (excerpt)

use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

->withExceptions(function (Exceptions $exceptions) {
    // Force JSON for everything under /api, even without an Accept header:
    $exceptions->shouldRenderJsonWhen(function (Request $request, \Throwable $e) {
        return $request->is('api/*') || $request->expectsJson();
    });

    // Custom payload for 404s on API routes:
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'Resource not found.',
                'status'  => 404,
            ], 404);
        }
    });

    // Don't report (log) a noisy custom exception:
    $exceptions->dontReport(\App\Exceptions\PaymentDeclinedException::class);
})
```

You can also implement the `Illuminate\Contracts\Support\Responsable` interface on a custom exception, or give it a `render(Request $request)` method that returns a response — Laravel calls it automatically.

### A reusable custom exception

```php
<?php
// app/Exceptions/ApiException.php
namespace App\Exceptions;

use Exception;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class ApiException extends Exception
{
    public function __construct(string $message, private int $status = 400)
    {
        parent::__construct($message);
    }

    public function render(Request $request): JsonResponse
    {
        return response()->json([
            'message' => $this->getMessage(),
            'status'  => $this->status,
        ], $this->status);
    }
}

// throw new ApiException('Insufficient balance', 409);
```

---

## 7. Rate limiting

**Rate limiting** caps how many requests a client can make in a window. It protects you from abuse, runaway scripts, and brute-force attacks. In Laravel it's powered by named **rate limiters** registered with `RateLimiter::for` and applied via the `throttle` middleware.

The default `api` limiter is defined in `app/Providers/AppServiceProvider.php` (Laravel 12) — older versions used `RouteServiceProvider`:

```php
<?php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        RateLimiter::for('api', function (Request $request) {
            // 60/min per authenticated user, else per IP:
            return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
        });

        // A stricter limiter for an expensive endpoint:
        RateLimiter::for('uploads', function (Request $request) {
            return $request->user()?->isPremium()
                ? Limit::none()                                   // unlimited
                : Limit::perMinute(10)->by($request->user()?->id ?: $request->ip())
                    ->response(function (Request $request, array $headers) {
                        return response()->json(
                            ['message' => 'Upload limit reached. Try later.'],
                            429, $headers
                        );
                    });
        });
    }
}
```

Apply named limiters with the `throttle:<name>` middleware:

```php
Route::middleware('throttle:uploads')->post('/upload', UploadController::class);

// Inline limits (no named limiter needed): 100 requests per 1 minute
Route::middleware('throttle:100,1')->get('/search', SearchController::class);
```

Every throttled response carries these headers so clients can self-pace:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 57
```

When the limit is exceeded, the response is `429 Too Many Requests` with:

```
Retry-After: 53
X-RateLimit-Reset: 1718705000
```

Return multiple limits from one limiter (e.g., per-minute AND per-day) by returning an array of `Limit` objects.

---

## 8. CORS

**CORS** (Cross-Origin Resource Sharing) is a browser security mechanism. By default a browser refuses to let JavaScript on `https://app.example.com` read a response from `https://api.example.com` — a *different origin* — unless the API explicitly opts in via response headers. CORS only affects **browsers**; server-to-server and mobile clients are unaffected.

Laravel handles CORS through `config/cors.php` (publish it with `php artisan config:publish cors` if it's missing — it's not present by default in Laravel 11/12). The `HandleCors` middleware is registered globally out of the box.

```php
<?php
// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],   // which paths CORS applies to

    'allowed_methods' => ['*'],                     // GET, POST, ...

    'allowed_origins' => ['https://app.example.com'],   // NOT '*' if using credentials

    'allowed_origins_patterns' => [],

    'allowed_headers' => ['*'],

    'exposed_headers' => ['X-RateLimit-Remaining'], // headers JS is allowed to read

    'max_age' => 0,                                 // how long preflight is cached (sec)

    'supports_credentials' => true,                 // REQUIRED for Sanctum SPA cookies
];
```

> **Critical pairing.** If `supports_credentials` is `true`, `allowed_origins` **cannot** be `['*']` — the browser rejects a wildcard origin alongside credentials. List exact origins. For SPA cookie auth (next section), you need `supports_credentials => true` and `sanctum/csrf-cookie` in `paths`.

---

## 9. Sanctum: the two authentication modes

**Laravel Sanctum** is the official lightweight auth package for APIs. It does **two distinct things**, and conflating them is the #1 source of confusion:

1. **API token authentication** — for mobile apps, CLIs, and third-party server clients. You issue a personal access token (a random string); the client sends it as a `Bearer` token. Stateless.
2. **SPA (Single-Page App) cookie authentication** — for a JavaScript frontend (Vue/React) served from a domain you control. Uses normal Laravel session cookies + CSRF protection, *no tokens to manage*.

Install. The one-liner does everything:

```bash
php artisan install:api    # adds laravel/sanctum, publishes its migration, runs install:api wiring
php artisan migrate        # creates the personal_access_tokens table (install:api offers to run it)
```

If you ever need to publish Sanctum's config or migration manually (e.g., to customize them), use tag-based publishing — the modern equivalent of the old `--provider` flag:

```bash
php artisan vendor:publish --tag=sanctum-config       # config/sanctum.php
php artisan vendor:publish --tag=sanctum-migrations    # the personal_access_tokens migration
```

Add the trait to your model:

```php
<?php
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
    // ...
}
```

### 9a. API token authentication

Issue a token on login. `createToken` returns a `NewAccessToken` whose `plainTextToken` is the **only time** you'll ever see the raw token — only a SHA-256 hash is stored.

```php
<?php
// app/Http/Controllers/Api/AuthController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email'    => ['required', 'email'],
            'password' => ['required'],
        ]);

        $user = User::where('email', $credentials['email'])->first();

        // Hash::check runs even when $user is null-guarded; do NOT leak which part
        // (email vs password) was wrong — a single generic message prevents account enumeration.
        if (! $user || ! Hash::check($credentials['password'], $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);   // -> 422
        }

        // Second arg = abilities (scopes). Third arg = optional expiry.
        $token = $user->createToken(
            name: $request->string('device_name')->value() ?: 'api',
            abilities: ['posts:read', 'posts:create'],
            expiresAt: now()->addDays(30),
        );

        return response()->json([
            'token' => $token->plainTextToken,   // e.g. "3|aZ7...": id|secret — shown ONCE
            'user'  => $user->only('id', 'name', 'email'),
        ]);
    }
}
```

> **Security: throttle login.** Token issuance is the prime target for credential-stuffing and brute force. Always rate-limit the login route, e.g. `Route::post('/login', ...)->middleware('throttle:5,1');` (5 attempts/min), or define a `login` limiter keyed by email+IP. Returning a single generic "credentials are incorrect" message (as above) also prevents account enumeration.

The client then sends it on every request:

```bash
curl https://api.example.com/api/user \
  -H "Authorization: Bearer 3|aZ7xKp9..." \
  -H "Accept: application/json"
```

Guard routes with the `auth:sanctum` guard:

```php
<?php
// routes/api.php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::post('/login', [\App\Http\Controllers\Api\AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn (Request $request) => $request->user());
    Route::apiResource('posts', \App\Http\Controllers\Api\PostController::class);
    Route::post('/logout', function (Request $request) {
        $request->user()->currentAccessToken()->delete();   // revoke this token
        return response()->noContent();   // 204
    });
});
```

**Abilities / scopes** let one token do less than the full account. Check them with `tokenCan`:

```php
public function store(Request $request)
{
    if (! $request->user()->tokenCan('posts:create')) {
        abort(403, 'This token cannot create posts.');
    }
    // ...
}
```

You can also enforce abilities as middleware. **Important (Laravel 11/12):** Sanctum's ability middleware are **not aliased automatically** — you must register the aliases yourself in `bootstrap/app.php`:

```php
<?php
// bootstrap/app.php (excerpt)
use Laravel\Sanctum\Http\Middleware\CheckAbilities;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'abilities' => CheckAbilities::class,      // ALL listed abilities required
        'ability'   => CheckForAnyAbility::class,  // ANY ONE of the listed abilities
    ]);
})
```

Then use them on routes:

```php
// ALL listed abilities required (plural "abilities"):
Route::post('/posts', ...)->middleware(['auth:sanctum', 'abilities:posts:read,posts:create']);

// ANY ONE of the listed abilities is enough (singular "ability"):
Route::get('/posts', ...)->middleware(['auth:sanctum', 'ability:posts:read,posts:create']);
```

> **Watch the naming:** `abilities` (plural) = *all required*; `ability` (singular) = *any one*. It is the opposite of what intuition suggests, so memorize it.

> A token created with the wildcard `['*']` ability passes every `tokenCan` check.

**Revoking tokens:**

```php
$user->tokens()->delete();                    // log out everywhere
$user->tokens()->where('id', $id)->delete();  // revoke one specific token
$user->currentAccessToken()->delete();        // log out the current device
```

### 9b. SPA cookie-based authentication

For a first-party JavaScript SPA, tokens are overkill and storing them in `localStorage` is an XSS risk. Sanctum's SPA mode uses **secure, HTTP-only session cookies** plus CSRF protection instead.

The flow:

1. The SPA first calls `GET /sanctum/csrf-cookie`. Laravel sets an `XSRF-TOKEN` cookie.
2. The SPA logs in via your normal `POST /login` (web guard, session-based). A session cookie is set.
3. Subsequent XHR requests automatically send the cookies. Sanctum's middleware sees the request came from a **stateful domain** and authenticates via the session — no `Authorization` header needed.

> **Constraint.** The SPA and the API must share the same top-level domain (they may differ by subdomain). Cross-domain SPA cookie auth is not supported — use token auth for that.

**Step 1 — enable stateful API middleware (Laravel 11/12).** This is the step everyone forgets. In `bootstrap/app.php`, call `statefulApi()` so requests from your stateful domains are authenticated by session cookie before falling back to a token:

```php
<?php
// bootstrap/app.php (excerpt)
->withMiddleware(function (Middleware $middleware) {
    $middleware->statefulApi();   // enables Sanctum's EnsureFrontendRequestsAreStateful on the api group
})
```

**Step 2 — configure stateful domains and session cookie:**

```env
# .env
SANCTUM_STATEFUL_DOMAINS=localhost:3000,app.example.com   # include the port if there is one
SESSION_DOMAIN=.example.com    # shared parent so cookie is sent to api.example.com
SESSION_SECURE_COOKIE=true     # in production (HTTPS)
```

```php
// config/sanctum.php — defaults pull from the env var above
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
    '%s%s',
    'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
    Sanctum::currentApplicationUrlWithPort(),
))),
```

Frontend (axios automatically reads the `XSRF-TOKEN` cookie and echoes it as the `X-XSRF-TOKEN` header):

```javascript
import axios from 'axios';
axios.defaults.withCredentials = true;     // send & accept cookies
axios.defaults.withXSRFToken = true;       // required in newer axios

await axios.get('/sanctum/csrf-cookie');           // step 1
await axios.post('/login', { email, password });   // step 2 (session created)
const { data } = await axios.get('/api/user');     // step 3 (authenticated by cookie)
```

The **same** `auth:sanctum` middleware protects routes for *both* modes. Sanctum first checks for a stateful session cookie (SPA), and if absent, falls back to the `Authorization: Bearer` token. This is the elegant part: you write one guard and it serves your web SPA, mobile app, and partners.

> **Why CSRF here but not for tokens?** Cookie auth is vulnerable to CSRF (a malicious site can make the browser send your cookies). Bearer tokens aren't auto-sent by the browser, so token-mode APIs don't need CSRF protection — which is why `api.php` routes are CSRF-exempt.

### 9c. How Sanctum works under the hood

A personal access token's plaintext form is `{id}|{40-char-secret}`. On each request, Sanctum's `TransientToken`/guard logic:

1. Splits on `|` to get the token model `id` and the secret.
2. Looks up `personal_access_tokens` by `id`.
3. Hashes the incoming secret with SHA-256 and compares it (constant-time) to the stored `token` column.
4. Checks `expires_at` and `abilities`, updates `last_used_at`, then resolves the related `tokenable` model (your `User`) and sets it as the authenticated user.

Because only the hash is stored, a database leak doesn't expose usable tokens.

---

## 10. Sanctum vs Passport — which and when

| | **Sanctum** | **Passport** |
|---|---|---|
| Protocol | Simple opaque tokens / session cookies | Full **OAuth2** server (+ JWT access tokens) |
| Setup | One trait, one table | Heavy: clients, scopes, grant types, keys |
| Best for | First-party SPAs, mobile apps, simple API tokens | Third-party developers consuming your API (authorization code, client-credentials, etc.) |
| Token format | Random string (DB-hashed) | Signed JWT |
| Complexity | Low | High |

**Rule of thumb:** Reach for **Sanctum** unless you specifically need OAuth2 — i.e., you're building a platform where *external* developers register apps and users grant them scoped access ("Log in with MyApp," third-party integrations). For your own SPA/mobile clients, Sanctum is the lighter, recommended default. (Note: Passport now also offers a built-in `passport:install` and has slimmed down, but the decision rule still holds.)

---

## 11. Testing APIs

Laravel's HTTP test helpers send real requests through the framework (no live server needed) and set `Accept: application/json` for you. Use `getJson`, `postJson`, `putJson`, `patchJson`, `deleteJson`, then fluent assertions.

> **Default runner: Pest.** Since Laravel 11, fresh apps ship with **Pest** as the default test runner (PHPUnit is still under the hood). Both styles work — the class-based PHPUnit example below maps 1:1 to Pest's `test('...', fn () => ...)` functions. A Pest version is shown right after.

PHPUnit (class-based) style:

```php
<?php
// tests/Feature/PostApiTest.php
namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class PostApiTest extends TestCase
{
    use RefreshDatabase;

    public function test_index_returns_paginated_posts(): void
    {
        Post::factory()->count(3)->create();

        $this->getJson('/api/posts')
            ->assertOk()                                  // 200
            ->assertJsonCount(3, 'data')
            ->assertJsonStructure([
                'data' => [['id', 'title', 'body']],
                'meta' => ['current_page', 'total'],
            ]);
    }

    public function test_store_requires_authentication(): void
    {
        $this->postJson('/api/posts', ['title' => 'x', 'body' => 'y'])
            ->assertUnauthorized();                       // 401
    }

    public function test_validation_errors_return_422(): void
    {
        Sanctum::actingAs(User::factory()->create(), ['posts:create']);

        $this->postJson('/api/posts', [])                 // missing fields
            ->assertStatus(422)
            ->assertJsonValidationErrors(['title', 'body']);
    }

    public function test_authenticated_user_can_create_post(): void
    {
        $user = User::factory()->create();
        Sanctum::actingAs($user, ['posts:create']);       // fake a token w/ abilities

        $this->postJson('/api/posts', [
            'title' => 'Hello',
            'body'  => 'World',
        ])
            ->assertCreated()                             // 201
            ->assertJson(['data' => ['title' => 'Hello']])  // partial match
            ->assertJsonPath('data.title', 'Hello');     // exact path match
    }
}
```

The same suite written in **Pest** (the default in Laravel 11/12) — note `uses()` to attach traits and `$this` inside the closures:

```php
<?php
// tests/Feature/PostApiTest.php  (Pest)

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Sanctum\Sanctum;

uses(RefreshDatabase::class);

test('index returns paginated posts', function () {
    Post::factory()->count(3)->create();

    $this->getJson('/api/posts')
        ->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure(['data' => [['id', 'title', 'body']], 'meta' => ['current_page', 'total']]);
});

test('store requires authentication', function () {
    $this->postJson('/api/posts', ['title' => 'x', 'body' => 'y'])
        ->assertUnauthorized();   // 401
});

test('authenticated user can create a post', function () {
    Sanctum::actingAs(User::factory()->create(), ['posts:create']);

    $this->postJson('/api/posts', ['title' => 'Hello', 'body' => 'World'])
        ->assertCreated()
        ->assertJsonPath('data.title', 'Hello');
});
```

Key assertions:

- `assertOk()` / `assertCreated()` / `assertNoContent()` / `assertStatus(429)` — status codes.
- `assertJson([...])` — the response **contains** this subset.
- `assertExactJson([...])` — the response equals this exactly.
- `assertJsonPath('data.0.id', 1)` — pinpoint a nested value.
- `assertJsonCount(3, 'data')` — array length.
- `assertJsonValidationErrors(['title'])` — 422 errors for given fields.
- `Sanctum::actingAs($user, $abilities)` — bypass real token issuance in tests.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `Accept: application/json`, so errors come back as HTML.** Auto-JSON depends on `$request->expectsJson()` (the `Accept` header), **not** the `/api` URL prefix. A client without the header may receive an HTML error page or a `302` redirect on validation failure. **Fix:** Always send `Accept: application/json` from clients; in tests use `getJson`/`postJson`; in `withExceptions` use `shouldRenderJsonWhen(fn ($req, $e) => $req->is('api/*'))` to force JSON for all `api/*` paths.

2. **`supports_credentials => true` with `allowed_origins => ['*']`.** Browsers silently reject this combination, and your SPA login mysteriously fails. **Fix:** When using cookies/credentials (Sanctum SPA), list exact origins in `allowed_origins`, never a wildcard.

3. **Storing API tokens in `localStorage` for a first-party SPA.** This exposes them to XSS theft. **Fix:** Use Sanctum's SPA cookie mode (HTTP-only cookies + CSRF) for first-party SPAs; reserve bearer tokens for mobile/CLI/third-party.

4. **Returning models directly and leaking columns (or causing N+1).** `return $post;` ships `password`-adjacent fields, internal flags, and triggers a query per related model in loops. **Fix:** Use API Resources with `whenLoaded()`, and eager-load relations (`Post::with('user')`) before passing to the resource.

5. **Expecting `routes/api.php` to exist in Laravel 11/12.** It doesn't by default, and neither does `app/Exceptions/Handler.php`. **Fix:** Run `php artisan install:api`; configure exceptions in `bootstrap/app.php → withExceptions`.

6. **SPA cookie auth fails because the SPA domain isn't stateful — or `statefulApi()` was never enabled.** If `SANCTUM_STATEFUL_DOMAINS` doesn't include your frontend's host:port, OR you forgot `$middleware->statefulApi()` in `bootstrap/app.php`, Sanctum treats the request as a token request and you get 401. **Fix:** Call `statefulApi()`, add the exact `host:port` to `SANCTUM_STATEFUL_DOMAINS`, set a shared `SESSION_DOMAIN`, and call `/sanctum/csrf-cookie` first.

7. **Using `204 No Content` but still returning a body.** Some clients choke when a 204 carries content. **Fix:** Use `response()->noContent()` for deletes; reserve bodies for `200`/`201`.

8. **Assuming the `api` group is rate-limited out of the box (Laravel 11/12).** Unlike Laravel 10, `throttle:api` is NOT auto-applied to the `api` group — only the `RateLimiter::for('api', ...)` *definition* exists. An unprotected API silently accepts unlimited traffic. **Fix:** Append `throttle:api` to the group (`$middleware->api(append: ['throttle:api'])`) or attach `->middleware('throttle:api')` per route.

9. **Using `ability`/`abilities` middleware without registering the aliases.** In Laravel 11/12 Sanctum does NOT auto-alias them; the route just errors with "Target class [ability] does not exist." **Fix:** Register `'ability' => CheckForAnyAbility::class` and `'abilities' => CheckAbilities::class` in `bootstrap/app.php`. And remember: `abilities` = all required, `ability` = any one.

10. **Confusing `401` and `419`.** Token/credential failures return `401`; an expired *session* or a missing/mismatched CSRF token in SPA mode returns `419 Page Expired`. **Fix:** In the SPA, on `419` re-fetch `/sanctum/csrf-cookie`; on `401` send the user back to login.

---

## ✅ Best Practices

- **Always wrap output in API Resources** — never `return $model`. Decouples DB schema from API contract.
- **Use proper status codes**: `201` on create, `204` on delete, `422` for validation, `404` for missing, `403` vs `401` correctly.
- **Version from day one** (`/api/v1`) even if you only have one version. Cheap insurance.
- **Paginate every list endpoint** and cap `per_page`. Prefer `cursorPaginate` for huge or fast-moving datasets.
- **Validate in Form Requests** (`php artisan make:request`) to keep controllers thin and reuse rules.
- **Scope tokens with abilities** — issue the *least privilege* token a client needs; check with `tokenCan`/`ability` middleware.
- **Set token expiry** (`expiresAt`) and provide a logout/revoke endpoint.
- **Rate-limit by user when authenticated, by IP otherwise**, and expose `Retry-After` so clients can back off. In Laravel 11/12 you must *explicitly* enable throttling on the `api` group — it is no longer automatic.
- **Throttle authentication endpoints separately** (login, password reset, token issuance) with a strict limiter to blunt brute-force and credential-stuffing.
- **Keep `api.php` stateless** — no session reliance for token clients; use cookie mode only for first-party SPAs.
- **Return consistent error envelopes** (`message` + `errors`) so frontends parse one shape.
- **Test the contract**, not the implementation: assert status codes, JSON structure, and validation errors.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `Route::resource` and `Route::apiResource`?**
A. `apiResource` registers only the 5 JSON-relevant actions (index, store, show, update, destroy) and **omits** `create` and `edit`, which exist to serve HTML forms. APIs have no forms, so those routes are pointless.

**Q2. 401 vs 403 — when do you return each?**
A. `401 Unauthorized` means *not authenticated* (no/invalid credentials) — the client should log in. `403 Forbidden` means *authenticated but not permitted* — logging in again won't help. Also remember `422` for validation, distinct from `400`.

**Q3. How does Sanctum authenticate a request under the hood?**
A. A token is `{id}|{secret}`. Sanctum splits on `|`, looks up `personal_access_tokens` by `id`, SHA-256-hashes the incoming secret and compares (constant-time) against the stored hash, verifies `expires_at` and `abilities`, updates `last_used_at`, then loads the polymorphic `tokenable` (the User) as the authenticated user. Only the hash is stored, so a DB leak can't be replayed.

**Q4. Sanctum vs Passport?**
A. Sanctum = simple opaque tokens + SPA session cookies; minimal setup; best for first-party SPAs, mobile, and basic API keys. Passport = a full OAuth2 server with JWTs and grant types; use it when *third-party* developers need scoped, delegated access to your API. Default to Sanctum unless you genuinely need OAuth2.

**Q5. How does Sanctum's SPA mode avoid storing tokens, and why is CSRF involved?**
A. It rides on Laravel's normal session cookies (HTTP-only, so JS can't read them, mitigating XSS). Because cookies are auto-sent by browsers, the API is now vulnerable to CSRF, so Sanctum requires the `XSRF-TOKEN` cookie to be echoed back as the `X-XSRF-TOKEN` header. Setup needs three things in Laravel 11/12: `$middleware->statefulApi()` in `bootstrap/app.php`, the SPA's exact `host:port` in `SANCTUM_STATEFUL_DOMAINS`, and a shared `SESSION_DOMAIN`. The SPA and API must share a top-level domain.

**Q6. Where do you customize API exception responses in Laravel 11/12?**
A. In `bootstrap/app.php` inside `withExceptions()` — there's no `Handler.php` anymore. Use `render()` callbacks per exception type and `shouldRenderJsonWhen()` to force JSON. Laravel already returns JSON automatically for requests expecting it (422 for validation, 404 for binding misses, etc.).

**Q7. How does rate limiting work and what headers does it send?**
A. Named limiters are defined with `RateLimiter::for('name', fn ($request) => Limit::perMinute(60)->by(...))` and applied via `throttle:name`. Responses carry `X-RateLimit-Limit` and `X-RateLimit-Remaining`; on exceed you get `429` plus `Retry-After` and `X-RateLimit-Reset`. Limiting `by($user->id ?: $ip)` gives fair per-client buckets.

**Q8. What's the N+1 problem in an API context and how do Resources help?**
A. Iterating a collection and lazily accessing a relation per item fires one query per row. Resources expose `whenLoaded('relation')`, which only includes the relation if you eager-loaded it (`with('relation')`), nudging you to load relations once up front and keeping payloads conditional.

**Q9. When would you choose `cursorPaginate` over `paginate`?**
A. For very large tables or infinite-scroll feeds. `paginate` runs a `COUNT(*)` and uses `OFFSET`, which gets slower the deeper you page; `cursorPaginate` uses a keyset (`WHERE id > ?`) so it stays O(1) and is stable when rows are inserted, at the cost of no random page access and no `total`.

**Q10. What does CORS protect against and who enforces it?**
A. CORS is enforced by the **browser**, not the server — it prevents JavaScript on one origin from reading responses from another origin unless the server opts in via `Access-Control-*` headers. It does nothing for server-to-server or native mobile clients. Configure it in `config/cors.php`; remember credentials require explicit (non-wildcard) origins.

**Q11. What changed for APIs in Laravel 11/12 vs Laravel 10?**
A. (1) `routes/api.php` is opt-in via `php artisan install:api`; (2) there's no `app/Http/Kernel.php`, no `app/Console/Kernel.php`, and no `app/Exceptions/Handler.php` — middleware, scheduling (`routes/console.php`), and exceptions are configured fluently in `bootstrap/app.php`; (3) providers are listed in `bootstrap/providers.php`; (4) the `api` group is **no longer throttled by default** — you opt in with `throttle:api`; (5) the default `RateLimiter::for('api', ...)` now lives in `AppServiceProvider`, not `RouteServiceProvider` (which is gone); (6) Pest is the default test runner; (7) Sanctum's `ability`/`abilities` middleware and SPA `statefulApi()` must be wired up explicitly in `bootstrap/app.php`.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Scaffolding
php artisan install:api                 # routes/api.php + Sanctum
php artisan make:controller Api/PostController --api
php artisan make:resource PostResource
php artisan make:request StorePostRequest
php artisan route:list --path=api
php artisan migrate                     # personal_access_tokens table
```

```php
// Routes
Route::apiResource('posts', PostController::class);          // 5 actions
Route::prefix('v1')->group(fn () => /* ... */);             // versioning
Route::middleware('auth:sanctum')->group(fn () => /* ... */);
Route::middleware('throttle:60,1')->get('/x', ...);         // inline limit
Route::post('/x', ...)->middleware('ability:posts:create'); // scope check

// Responses
return response()->json($data, 201);
return response()->noContent();                              // 204
return PostResource::make($post);                           // single
return PostResource::collection(Post::paginate(15));        // list + meta

// Sanctum tokens
$t = $user->createToken('name', ['posts:read'], now()->addDays(30));
$t->plainTextToken;                       // shown ONCE
$request->user()->tokenCan('posts:read'); // ability check
$request->user()->currentAccessToken()->delete();  // revoke current
$user->tokens()->delete();                // revoke all

// Pagination variants
->paginate(15)        // total + last_page (COUNT query)
->simplePaginate(15)  // prev/next only (no count)
->cursorPaginate(15)  // keyset, O(1), next_cursor

// bootstrap/app.php wiring (Laravel 11/12)
->withMiddleware(function (Middleware $middleware) {
    $middleware->api(append: ['throttle:api']);   // opt in to API rate limiting (NOT automatic)
    $middleware->statefulApi();                   // enable Sanctum SPA cookie auth
    $middleware->alias([                          // Sanctum ability middleware (not auto-registered)
        'abilities' => \Laravel\Sanctum\Http\Middleware\CheckAbilities::class,     // ALL required
        'ability'   => \Laravel\Sanctum\Http\Middleware\CheckForAnyAbility::class, // ANY one
    ]);
})

// Exceptions (bootstrap/app.php)
$exceptions->shouldRenderJsonWhen(fn ($req, $e) => $req->is('api/*'));
$exceptions->render(fn (NotFoundHttpException $e, $req) => /* JSON 404 */);

// Tests
$this->getJson('/api/posts')->assertOk()->assertJsonCount(3, 'data');
$this->postJson('/api/posts', [])->assertStatus(422)
     ->assertJsonValidationErrors(['title']);
Sanctum::actingAs($user, ['posts:create']);
```

**Status code crib:** 200 OK · 201 Created · 204 No Content · 401 Unauthenticated · 403 Forbidden · 404 Not Found · 422 Validation · 429 Too Many Requests · 500 Server Error.

**Rate-limit headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`, `X-RateLimit-Reset`.

---

## 🧪 Mini Exercises

1. **Versioned, resourceful CRUD.** Scaffold `GET/POST/GET/PUT/DELETE /api/v1/articles` using `apiResource` inside a `v1` prefix and a `PostController` clone. Return an `ArticleResource`, paginate the index with a client-capped `per_page` (max 50), and respond `201` on create and `204` on delete.

2. **Scoped tokens.** Add a `POST /api/login` that issues a token with abilities `['articles:read']` for normal users and `['articles:read','articles:write']` for admins. First register the `ability` alias (`CheckForAnyAbility`) in `bootstrap/app.php`, then guard the write routes with `ability:articles:write` and prove a read-only token gets `403`.

3. **Custom error envelope.** In `bootstrap/app.php`, make every `api/*` 404 and 422 return the shape `{ "error": { "code": <int>, "message": <string>, "fields": {...} } }`. Write a test asserting both shapes with `assertJsonPath`.

4. **Rate-limit an endpoint.** Define a `search` limiter at 5 requests/minute keyed by user-or-IP. Apply it to `GET /api/search`. Write a test that fires 6 requests and asserts the 6th returns `429` with a `Retry-After` header.

5. **SPA auth simulation.** Enable `$middleware->statefulApi()` in `bootstrap/app.php`, configure `SANCTUM_STATEFUL_DOMAINS` for `localhost:3000`, then write a feature test that hits `/sanctum/csrf-cookie`, logs in via the session guard, and reads `/api/user` successfully — without ever sending a bearer token.

6. **Enable API throttling (Laravel 11/12).** Confirm the `api` group is *not* throttled by default (hit a route 70 times and observe no `429`), then append `throttle:api` to the group in `bootstrap/app.php` and prove the 61st request now returns `429` with `X-RateLimit-*` and `Retry-After` headers.
