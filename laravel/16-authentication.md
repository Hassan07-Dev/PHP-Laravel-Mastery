# Authentication in Laravel 12 (PHP 8.4)

Authentication is the process of answering one question: **"Who are you?"** Everything else in a web app — dashboards, profiles, billing, admin panels — depends on first establishing the identity of the person (or machine) making the request. Laravel ships with a complete, batteries-included authentication system, but it is deliberately layered so you can use as little or as much of it as you need. This module takes you from the underlying concepts all the way to production-grade patterns.

> **Target stack:** PHP 8.4, Laravel 12. Where Laravel 10/11 or PHP 8.1–8.3 differ, it's called out inline.

## **What you'll learn**

- The difference between **authentication** and **authorization**, and why conflating them causes security bugs.
- How Laravel's auth config works: **guards**, **providers**, and how a request becomes an authenticated user.
- What the official **starter kits** (Breeze, Jetstream, and the new Laravel 12 kits) scaffold for you.
- How to do **manual authentication** with `Auth::attempt`, `login`, `logout`, `once`, and remember-me.
- How to **protect routes** with the `auth` and `guest` middleware, and confirm passwords.
- How **password hashing** works (`Hash` facade, bcrypt vs argon2), and automatic **rehashing**.
- The built-in **password reset** and **email verification** flows, plus **login throttling**.
- Advanced topics: **multiple guards**, **auth events**, **logout of other devices**, and when to reach for **Sanctum**.

---

## 1. Authentication vs. Authorization

These two words are constantly mixed up, even by experienced developers, so let's nail them down first.

- **Authentication (AuthN)** = *Who are you?* Proving identity (e.g. email + password, an API token, an OAuth login).
- **Authorization (AuthZ)** = *Are you allowed to do this?* Deciding what an already-identified user may access (e.g. "only admins can delete posts").

A useful mnemonic: **authe**N**tication** → who (**N**ame); **autho**R**ization** → **R**ights.

In Laravel these are separate subsystems:

| Concern | Subsystem | Key APIs |
| --- | --- | --- |
| Authentication | The `Auth` system | `Auth::attempt()`, guards, providers, `auth` middleware |
| Authorization | Gates & Policies | `Gate::allows()`, `$user->can()`, `can:` middleware, Policies |

This module covers **authentication only**. Authorization (Gates/Policies) is a separate topic. The common bug from conflating them: developers put a logged-in user behind `auth` middleware and assume that's enough, forgetting that *any* logged-in user can now reach an admin-only action. Authentication is the front door; authorization decides which rooms you can enter.

---

## 2. The Mental Model: Guards and Providers

Before any code, understand the two abstractions that the entire system rests on. Everything in `config/auth.php` is built from these.

- **A Provider** answers: *"Where do user records come from, and how do I fetch one?"* The default provider is `eloquent`, backed by your `App\Models\User` model. There's also a `database` provider that queries a table directly via the query builder (no Eloquent model).

- **A Guard** answers: *"How is the user authenticated for this request, and how do I persist them?"* The two built-in guard *drivers* are:
  - **`session`** — the classic stateful web guard. After login, the user's ID is stored in the session (a server-side store keyed by a cookie). Used for browser-based apps. This is the default `web` guard.
  - **`token`** — a simple stateless API token guard (legacy). For real APIs you'll use **Sanctum** or **Passport** instead, which register their own guards.

A guard *uses* a provider. Think of it as: the **guard** is the security gate's protocol (cookie/session vs. bearer token); the **provider** is the staff directory it looks people up in.

```
Request ──▶ Guard (session) ──reads──▶ Provider (eloquent) ──▶ App\Models\User
            "how do I know who"          "where do users live"
```

### 2.1 The auth config file

```php
// config/auth.php
return [
    'defaults' => [
        'guard' => 'web',
        'passwords' => 'users',
    ],

    'guards' => [
        'web' => [
            'driver'   => 'session',
            'provider' => 'users',
        ],
        // You can add more guards here, e.g. 'admin', 'api'
    ],

    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model'  => env('AUTH_MODEL', App\Models\User::class),
        ],

        // Alternative: query a table directly without a model
        // 'users' => [
        //     'driver' => 'database',
        //     'table'  => 'users',
        // ],
    ],

    'passwords' => [
        'users' => [
            'provider' => 'users',
            'table'    => env('AUTH_PASSWORD_RESET_TOKEN_TABLE', 'password_reset_tokens'),
            'expire'   => 60,   // minutes a reset link is valid
            'throttle' => 60,   // seconds before another reset email can be requested
        ],
    ],

    'password_timeout' => 10800, // seconds before "password confirmation" expires (3 hours)
];
```

> **Laravel 11/12 note:** Several values (`AUTH_MODEL`, `AUTH_PASSWORD_RESET_TOKEN_TABLE`, etc.) are now driven from `.env` defaults so the config stays slim. In Laravel 10 the `model` line was hard-coded to `App\Models\User::class`. The behavior is identical; only the config style changed.

### 2.2 What the User model needs

For the `eloquent` provider to work, your model must extend `Authenticatable` (and usually implement a couple of contracts):

```php
// app/Models/User.php
namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail; // optional: enables email verification
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements MustVerifyEmail
{
    use Notifiable;

    protected $fillable = ['name', 'email', 'password'];

    protected $hidden = ['password', 'remember_token'];

    // Laravel 11/12: casts() method instead of $casts property
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed', // auto-hashes on assignment; more below
        ];
    }
}
```

The `'password' => 'hashed'` cast is the modern way to ensure passwords are hashed whenever you set them — added in **Laravel 10.10**. With it, `$user->password = 'plain'` stores a hash (using your configured driver, bcrypt by default) automatically, so you never accidentally save a plaintext password. The cast is idempotent: it skips re-hashing a value that already looks like a valid hash, which is why you must assign **plaintext**, not a pre-hashed string (see Gotcha #4).

---

## 3. Starter Kits: Don't Build the Boring Parts by Hand

You *can* build login screens from scratch — and Section 5 shows how — but for a real project you'll usually scaffold them. Laravel offers official starter kits.

### 3.1 The Laravel 12 starter kits (the new default)

Laravel 12 introduced **first-party application starter kits** generated directly at project creation time. When you run `laravel new`, you're prompted to choose a stack:

```bash
laravel new myapp
# Prompts: choose React, Vue, or Livewire starter kit (or "none")
```

These kits scaffold full authentication (login, registration, password reset, email verification, password confirmation), a dashboard, and a settings/profile area, using **WorkOS AuthKit** or the built-in auth depending on your choice. They replace the older recommendation of "reach for Breeze."

### 3.2 Laravel Breeze (lightweight, classic)

Breeze is the minimal, readable scaffold. Great for learning because the controllers are plain and easy to read.

```bash
composer require laravel/breeze --dev
php artisan breeze:install   # prompts for Blade / React / Vue / API
php artisan migrate
npm install && npm run dev
```

Breeze scaffolds:

- Routes & controllers for **register, login, logout, password reset, email verification, password confirmation**.
- Blade (or Inertia React/Vue) views for all of the above.
- A `routes/auth.php` file wiring it together.
- Profile management (update name/email, change password, delete account).

### 3.3 Laravel Jetstream (full-featured)

Jetstream is the heavyweight option built on **Livewire** or **Inertia**, adding: two-factor authentication (2FA), browser session management ("logout other devices"), API tokens via Sanctum, and optional team management.

```bash
composer require laravel/jetstream
php artisan jetstream:install livewire   # or: inertia
php artisan migrate
```

**Rule of thumb:** Use a Laravel 12 starter kit or Breeze for most apps. Reach for Jetstream only when you need 2FA, teams, or session management out of the box. For pure APIs, scaffold with `breeze:install api` (issues Sanctum tokens) or install Sanctum directly.

---

## 4. The Hashing Layer (Understand This Before Logging Anyone In)

You must **never** store passwords in plaintext. Laravel hashes them with a one-way, salted, deliberately-slow algorithm so that even if your database leaks, the passwords are not directly usable.

### 4.1 The Hash facade

```php
use Illuminate\Support\Facades\Hash;

$hash = Hash::make('secret-password');
// Output: a 60-char bcrypt string, e.g.
// $2y$12$EUS.kJh0V5oR1mQ3eImiTXuWVxfM37uY4JANjQ8s9q1yQ3eImiTXu
// Format: $2y$ (algo) $12$ (cost) then a 22-char salt + 31-char digest.
// It's different every call due to the random salt — never compare hashes directly.

Hash::check('secret-password', $hash);  // true
Hash::check('wrong', $hash);            // false
```

Two critical properties:

1. **Salted:** `Hash::make('x')` returns a *different* string each call (a random salt is embedded). So you can never compare two hashes for equality — you must use `Hash::check()`.
2. **Slow on purpose:** the "cost" / work factor makes brute-forcing expensive. Higher cost = slower = more secure.

### 4.2 Choosing the algorithm: bcrypt vs argon2

```php
// config/hashing.php  (publish with: php artisan config:publish hashing)
'driver' => env('HASH_DRIVER', 'bcrypt'), // default. Also: 'argon' (Argon2i) or 'argon2id'

'bcrypt' => [
    'rounds'           => env('BCRYPT_ROUNDS', 12),
    'verify'           => env('HASH_VERIFY', true),     // reject hashes from a different algo
    'limit'            => null,
],

'argon' => [
    'memory'           => env('ARGON_MEMORY', 65536),
    'threads'          => env('ARGON_THREADS', 1),
    'time'             => env('ARGON_TIME', 4),
    'verify'           => env('HASH_VERIFY', true),
],

'rehash_on_login' => true, // automatically upgrade hashes on login (more in §4.3)
```

- **bcrypt** — the default, battle-tested, requires only PHP. Cost `rounds` (default 12) controls slowness.
- **argon2id** — memory-hard, the modern OWASP first choice if PHP is compiled with libsodium/argon support. More resistant to GPU attacks.

> **Version note:** the default bcrypt rounds were bumped to **12** (from 10) back in **Laravel 10.25** — so Laravel 11 and 12 both ship 12 as the default. You can force a specific driver per call: `Hash::driver('argon2id')->make($pw)`.

### 4.3 Automatic rehashing (the under-the-hood detail interviewers love)

When `Auth::attempt()` succeeds, Laravel checks whether the stored hash's work factor still matches your current config via `Hash::needsRehash()`. If you bumped `rounds` from 10 to 12, the *next time that user logs in* their password is transparently re-hashed with the stronger settings and re-saved. You get a gradual, zero-downtime security upgrade for free. You can disable it with `'rehash_on_login' => false` in `config/hashing.php`.

```php
if (Hash::needsRehash($hash)) {
    $hash = Hash::make($plainPassword); // store the upgraded hash
}
```

---

## 5. Manual Authentication

Even when using a starter kit, you must understand what it's doing. The `Auth` facade (and the `auth()` helper) is the entry point.

### 5.1 Auth::attempt() — validate credentials and log in

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Http\Request;

public function store(Request $request)
{
    $credentials = $request->validate([
        'email'    => ['required', 'email'],
        'password' => ['required'],
    ]);

    // Second arg = "remember me" boolean
    if (Auth::attempt($credentials, $request->boolean('remember'))) {
        $request->session()->regenerate(); // prevent session fixation
        return redirect()->intended('/dashboard');
    }

    return back()->withErrors([
        'email' => 'The provided credentials do not match our records.',
    ])->onlyInput('email');
}
```

How `attempt()` works under the hood:

1. The guard asks the **provider** to `retrieveByCredentials($credentials)` — every key *except* `password` becomes a `WHERE` clause. So `['email' => ..., 'password' => ...]` fetches the user by email.
2. It then calls `validateCredentials()`, which runs `Hash::check($plain, $user->password)`.
3. On success, the user is stored in the session and `true` is returned.

> Always call `$request->session()->regenerate()` after login to mint a new session ID — this defeats **session fixation** attacks. Starter kits do this for you.

> `redirect()->intended()` sends the user to the URL they originally tried to reach before being bounced to login (Laravel stores it as `url.intended` in the session).

You can add extra `WHERE` conditions just by including more keys:

```php
// Only log in active users
Auth::attempt(['email' => $email, 'password' => $pw, 'active' => 1]);

// Or use a closure for complex conditions (Laravel 9+). The closure
// receives the Eloquent query builder for the user model.
use Illuminate\Database\Eloquent\Builder;

Auth::attempt(['email' => $email, 'password' => $pw, fn (Builder $query) =>
    $query->where('active', true)->whereNull('banned_at')
]);
```

For inspection that needs the *resolved* user object (not just a query), use
`attemptWhen()` — its closure receives the candidate user and returns a bool
*before* the user is actually logged in:

```php
if (Auth::attemptWhen(
    ['email' => $email, 'password' => $pw],
    fn ($user) => $user->isNotBanned()
)) {
    // authenticated
}
```

### 5.2 Logging in a known user directly

When you already have the user (e.g. after registration, or impersonation), skip credential checking:

```php
Auth::login($user);              // log them in
Auth::login($user, remember: true); // with remember-me (named arg)
Auth::loginUsingId(1);           // by primary key
Auth::loginUsingId(1, remember: true);
```

### 5.3 once() — stateless single-request auth

`Auth::once()` authenticates the user for the **current request only** — no session, no cookie. Useful for stateless API endpoints or one-off checks.

```php
if (Auth::once(['email' => $email, 'password' => $pw])) {
    // authenticated for this request; nothing persisted
}
```

### 5.4 Logging out

```php
Auth::logout();

// For session guard, fully invalidate to be safe:
$request->session()->invalidate();
$request->session()->regenerateToken(); // new CSRF token
```

### 5.5 Remember-me, explained

If you pass `true` as the second arg to `attempt`/`login`, Laravel:

1. Generates a long random token and stores it in the `remember_token` column on the user (your `users` table needs this nullable string column — it's in the default migration).
2. Sets a long-lived cookie containing that token.

On a later visit where the session has expired, the `viaRemember()` path kicks in and re-authenticates the user from the cookie. You can check whether the *current* user was authenticated this way:

```php
if (Auth::viaRemember()) {
    // logged in via the remember cookie, not a fresh login
    // good practice: require password re-confirmation for sensitive actions
}
```

---

## 6. Retrieving the Authenticated User

There are several equivalent ways to reach the current user. Pick based on context.

```php
use Illuminate\Support\Facades\Auth;

Auth::user();        // App\Models\User|null
Auth::id();          // primary key, or null
Auth::check();       // bool: is someone logged in?
Auth::guest();       // bool: is NO one logged in?

// The helper (same thing)
auth()->user();
auth()->id();

// From the request (great for controllers/middleware)
public function show(Request $request) {
    $user = $request->user();   // App\Models\User|null
}
```

In Blade views:

```blade
@auth
    <p>Welcome, {{ auth()->user()->name }}!</p>
@endauth

@guest
    <a href="/login">Log in</a>
@endguest
```

> **Gotcha:** `Auth::user()` returns `null` when no one is logged in, so `Auth::user()->name` throws *"Attempt to read property on null"*. Guard with `@auth`, `Auth::check()`, or null-safe `Auth::user()?->name`.

To target a non-default guard, pass its name: `Auth::guard('admin')->user()`.

---

## 7. Protecting Routes with Middleware

Middleware is the gatekeeper that runs *before* your controller. Two are central to auth.

### 7.1 The `auth` middleware (require login)

```php
use App\Http\Controllers\DashboardController;

Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware('auth');

// Group many routes:
Route::middleware('auth')->group(function () {
    Route::get('/profile', /* ... */);
    Route::get('/settings', /* ... */);
});
```

If an unauthenticated user hits an `auth` route, they're redirected to the `login` named route (for web) or get a `401 Unauthenticated` JSON response (for `Accept: application/json` requests).

> **Laravel 11/12 note:** There is no `app/Http/Kernel.php` anymore. The redirect target is configured in `bootstrap/app.php`:
> ```php
> ->withMiddleware(function (Middleware $middleware) {
>     $middleware->redirectGuestsTo(fn () => route('login'));
> })
> ```

### 7.2 The `guest` middleware (only-for-logged-out)

The inverse: keep already-logged-in users *out* of login/register pages, redirecting them to the dashboard.

```php
Route::get('/login', [LoginController::class, 'create'])->middleware('guest');
```

Configure where guests-only redirects send authenticated users:

```php
// bootstrap/app.php
$middleware->redirectUsersTo('/dashboard'); // for the "guest" middleware
```

### 7.3 Specifying a guard on the middleware

```php
Route::get('/admin', /* ... */)->middleware('auth:admin'); // use the "admin" guard
```

---

## 8. Password Confirmation (the `password.confirm` middleware)

For sensitive actions (changing email, deleting account, viewing recovery codes) you want to re-verify the password *even though* the user is already logged in. The `password.confirm` middleware handles this.

```php
Route::get('/settings/danger', [DangerController::class, 'index'])
    ->middleware(['auth', 'password.confirm']);
```

When the user hits the route, if they haven't confirmed their password recently (within `auth.password_timeout`, default **3 hours**), they're redirected to a confirmation screen. After confirming, Laravel stamps `auth.password_confirmed_at` in the session.

```php
// Confirmation handler (Breeze scaffolds this)
public function store(Request $request)
{
    if (! Hash::check($request->password, $request->user()->password)) {
        return back()->withErrors(['password' => 'Incorrect password.']);
    }
    $request->session()->put('auth.password_confirmed_at', time());
    return redirect()->intended();
}
```

---

## 9. Login Throttling (Brute-Force Protection)

Without rate limiting, an attacker can try thousands of passwords. Laravel's `RateLimiter` plus the `throttle` middleware stops this.

### 9.1 Route-level throttle

```php
Route::post('/login', [LoginController::class, 'store'])
    ->middleware('throttle:5,1'); // 5 attempts per 1 minute, then HTTP 429
```

### 9.2 The robust pattern: per-email + per-IP limiting

Starter kits use a `LoginRequest` form request with explicit `RateLimiter` calls so the limit is keyed to the *email + IP combo*, not just the IP (so one attacker can't lock out everyone, and switching IPs doesn't reset the counter for a target email):

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

public function authenticate(): void
{
    $key = Str::transliterate(Str::lower($this->input('email')).'|'.$this->ip());

    if (RateLimiter::tooManyAttempts($key, maxAttempts: 5)) {
        $seconds = RateLimiter::availableIn($key);
        throw ValidationException::withMessages([
            'email' => "Too many attempts. Try again in {$seconds} seconds.",
        ]);
    }

    if (! Auth::attempt($this->only('email', 'password'), $this->boolean('remember'))) {
        RateLimiter::hit($key); // count the failure
        throw ValidationException::withMessages([
            'email' => 'These credentials do not match our records.',
        ]);
    }

    RateLimiter::clear($key); // reset on success
}
```

You can also define a named limiter centrally in `App\Providers\AppServiceProvider::boot()`:

```php
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', fn (Request $r) =>
    Limit::perMinute(5)->by($r->input('email').$r->ip())
);
// then: ->middleware('throttle:login')
```

---

## 10. Password Reset Flow

The "forgot password" flow has four moving parts. Laravel provides them all via the `Password` facade and the `password_reset_tokens` table (created by the default migration).

**The flow:**

1. User submits their email → app sends a reset link containing a signed token.
2. The token (hashed) is stored in `password_reset_tokens`.
3. User clicks the link → reset form with the token + email pre-filled.
4. User submits a new password → token verified, password updated, token deleted.

```php
use Illuminate\Support\Facades\Password;
use Illuminate\Support\Facades\Hash;
use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Support\Str;

// Step 1: send the link
public function sendLink(Request $request)
{
    $request->validate(['email' => 'required|email']);

    $status = Password::sendResetLink($request->only('email'));

    return $status === Password::ResetLinkSent
        ? back()->with('status', __($status))
        : back()->withErrors(['email' => __($status)]);
}

// Step 2: handle the new password
public function reset(Request $request)
{
    $request->validate([
        'token'    => 'required',
        'email'    => 'required|email',
        'password' => ['required', 'confirmed', Password::defaults()],
    ]);

    $status = Password::reset(
        $request->only('email', 'password', 'password_confirmation', 'token'),
        function ($user, $password) {
            $user->forceFill([
                'password'       => Hash::make($password),
                'remember_token' => Str::random(60),
            ])->save();

            event(new PasswordReset($user));
        }
    );

    return $status === Password::PasswordReset
        ? redirect()->route('login')->with('status', __($status))
        : back()->withErrors(['email' => __($status)]);
}
```

`Password::defaults()` references the password validation rules defined in a service provider (length, mixed case, uncompromised against haveibeenpwned, etc.):

```php
use Illuminate\Validation\Rules\Password as PasswordRule;

PasswordRule::defaults(fn () =>
    PasswordRule::min(8)->mixedCase()->numbers()->uncompromised()
);
```

---

## 11. Email Verification

To require users to confirm ownership of their email address:

1. Implement `MustVerifyEmail` on the `User` model (shown in §2.2). This triggers a verification email on registration via the `Registered` event listener.
2. Protect routes with the `verified` middleware.

```php
Route::get('/dashboard', /* ... */)
    ->middleware(['auth', 'verified']); // must be logged in AND verified
```

Laravel ships three routes for the flow (Breeze scaffolds them): the notice page, the signed verification link handler, and a "resend" endpoint:

```php
use Illuminate\Foundation\Auth\EmailVerificationRequest;

Route::get('/verify-email/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();          // marks email_verified_at and fires Verified event
    return redirect('/dashboard');
})->middleware(['auth', 'signed'])->name('verification.verify');

Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();
    return back()->with('status', 'verification-link-sent');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');
```

The link is **signed** (`signed` middleware) so it can't be tampered with, and the `{hash}` is `sha1` of the user's email — so changing the email invalidates pending links.

---

## 12. Multiple Guards

A common requirement: separate `admins` table from `users`, each with its own login. Add a second guard + provider:

```php
// config/auth.php
'guards' => [
    'web'   => ['driver' => 'session', 'provider' => 'users'],
    'admin' => ['driver' => 'session', 'provider' => 'admins'],
],
'providers' => [
    'users'  => ['driver' => 'eloquent', 'model' => App\Models\User::class],
    'admins' => ['driver' => 'eloquent', 'model' => App\Models\Admin::class],
],
```

Then target the guard explicitly everywhere:

```php
Auth::guard('admin')->attempt($credentials);
Auth::guard('admin')->user();
Route::middleware('auth:admin')->group(/* ... */);
```

> A user logged into `web` and `admin` are tracked **separately** in the session — logging out of one does not log out of the other. Each guard has its own session key (`login_web_…`, `login_admin_…`).

---

## 13. Auth Events

Laravel fires events throughout the auth lifecycle. Listen to them for audit logs, security alerts, or analytics. They live in `Illuminate\Auth\Events`:

| Event | Fired when |
| --- | --- |
| `Registered` | A new user registers (triggers email verification) |
| `Attempting` | `attempt()` is called |
| `Validated` | Credentials passed but before login |
| `Authenticated` | User resolved on a request |
| `Login` | A user successfully logs in |
| `Failed` | A login attempt failed |
| `Logout` | A user logs out |
| `Lockout` | Login throttle limit hit (Breeze's `LoginRequest` fires `event(new Lockout($request))`; the old `ThrottlesLogins` trait still exists but isn't used by the L11/12 starter kits) |
| `PasswordReset` | A password was reset |
| `Verified` | An email was verified |
| `CurrentDeviceLogout` / `OtherDeviceLogout` | Session logout events |

```php
// Listening (Laravel 11/12 auto-discovers listeners; or register manually)
use Illuminate\Auth\Events\Failed;
use Illuminate\Support\Facades\Event;

Event::listen(function (Failed $event) {
    logger()->warning('Failed login', ['email' => $event->credentials['email'] ?? null]);
});
```

---

## 14. Logout of Other Devices

Let a user invalidate every *other* session (e.g. "Sign out everywhere" after a password change) while staying logged in on the current device. This requires the `AuthenticateSession` middleware on your routes:

```php
// routes/web.php
Route::middleware(['web', 'auth', 'auth.session'])->group(function () {
    // ...
});
```

```php
use Illuminate\Support\Facades\Auth;

// Re-prompts for the password, then invalidates all OTHER sessions
Auth::logoutOtherDevices($request->input('password'));
```

> For this to work the session driver must persist (database, redis, etc.), and the `auth.session` middleware (`AuthenticateSession`) must be active so Laravel can compare the per-session password hash.

---

## 15. APIs: A Pointer to Sanctum

The session guard is for browsers. For **APIs and SPAs/mobile apps**, reach for **Laravel Sanctum**. It offers two modes:

- **API tokens** — issue revocable bearer tokens (`$user->createToken('name')->plainTextToken`), then authenticate with `Authorization: Bearer <token>` and the `auth:sanctum` middleware.
- **SPA authentication** — cookie/session-based for first-party single-page apps on the same domain (CSRF-protected).

```bash
php artisan install:api   # Laravel 11/12: installs Sanctum + api routes
```

```php
// Issue a token
$token = $user->createToken('mobile-app')->plainTextToken;

// Protect an API route
Route::get('/user', fn (Request $r) => $r->user())->middleware('auth:sanctum');
```

Use **Passport** instead only if you need full OAuth2 server features (third-party clients, authorization codes). For most apps, Sanctum is the answer. Sanctum gets its own dedicated module — this is just the pointer.

---

## ⚠️ Common Mistakes & Gotchas

1. **Calling `Auth::user()->name` when no one is logged in.**
   `Auth::user()` returns `null`, so this throws *"Attempt to read property 'name' on null."*
   **Fix:** Guard with `@auth` / `Auth::check()`, or use null-safe chaining: `Auth::user()?->name`. In API routes always rely on the `auth` middleware so a guaranteed user exists.

2. **Forgetting `$request->session()->regenerate()` after login.**
   Leaves the app vulnerable to **session fixation** (an attacker sets a known session ID before login and reuses it after).
   **Fix:** Always `regenerate()` on login and `invalidate()` + `regenerateToken()` on logout. Starter kits do this for you.

3. **Including `password` as a literal WHERE clause in `attempt()`.**
   `attempt()` treats *every* key except `password` as a query column; the `password` key is special-cased and checked with `Hash::check()`. People sometimes try `User::where('password', $hash)` manually — which fails because each hash is salted differently and is never equal.
   **Fix:** Never query by password. Use `Auth::attempt()` or `Hash::check()`.

4. **Double-hashing passwords.**
   With the `'password' => 'hashed'` cast (Laravel 10+), assigning `$user->password = Hash::make($pw)` hashes an already-hashed value. Logins then fail mysteriously.
   **Fix:** With the cast active, assign the **plaintext**: `$user->password = $request->password;`. Only call `Hash::make()` when the cast is *not* present (e.g. inside `Password::reset`'s `forceFill`, which bypasses casts).

5. **Throttling only by IP.**
   `throttle:5,1` keyed purely on IP lets one attacker behind many IPs hammer a single account, and can also lock out shared-NAT users.
   **Fix:** Key the limiter by `email + IP` as the starter kits do (§9.2).

6. **Putting `auth` middleware and assuming authorization is handled.**
   Any logged-in user passes `auth`. That does **not** mean they should see `/admin`.
   **Fix:** Add authorization (Gates/Policies, `can:` middleware) on top of authentication.

7. **Expecting `redirect()->intended()` to work without the redirect happening through the framework.**
   `intended()` reads `url.intended` from the session, which is only set when the `auth` middleware bounced the user. Calling it after a fresh visit to `/login` just falls back to the default.
   **Fix:** Provide a sensible default: `redirect()->intended('/dashboard')`.

---

## ✅ Best Practices

- **Always use HTTPS** in production — session cookies and credentials must never travel in plaintext. Set `SESSION_SECURE_COOKIE=true`.
- **Regenerate the session on login**, invalidate on logout.
- **Hash with at least bcrypt rounds 12** (the Laravel 12 default) or prefer **argon2id** where available.
- **Let automatic rehash-on-login** upgrade old hashes; bump `BCRYPT_ROUNDS` over time.
- **Throttle login, password-reset, and verification-resend** endpoints; key by email+IP.
- **Use the `'password' => 'hashed'` cast** so you can never accidentally store plaintext.
- **Require email verification** for any app where the email is meaningful; use the `verified` middleware.
- **Use `password.confirm`** for destructive/sensitive actions.
- **Validate password strength** centrally with `Password::defaults()` including `->uncompromised()`.
- **Don't roll your own crypto** — use the `Hash` facade and built-in flows; they're audited.
- **For APIs, use Sanctum**, not the legacy `token` guard.
- **Keep `remember_token` and `password` in `$hidden`** so they never leak in JSON responses.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between authentication and authorization in Laravel?**
A. Authentication ("who are you?") is handled by the `Auth` system — guards, providers, `Auth::attempt`, the `auth` middleware. Authorization ("are you allowed?") is handled by Gates and Policies — `Gate::allows`, `$user->can`, the `can:` middleware. They're separate; passing `auth` does not imply authorization.

**Q2. Explain guards vs. providers.**
A. A **provider** defines *where* user records come from and how to fetch them (`eloquent` model-backed, or `database` query-builder). A **guard** defines *how* a user is authenticated and persisted for a request (`session` for stateful web, `token`/Sanctum for stateless APIs). A guard uses a provider.

**Q3. How does `Auth::attempt()` work under the hood?**
A. It (1) calls the provider's `retrieveByCredentials()`, building a `WHERE` from every credential key except `password`; (2) calls `validateCredentials()`, which runs `Hash::check()` on the plaintext vs. the stored hash; (3) on success, logs the user into the session and returns `true`. It also fires `Attempting`, then `Validated`/`Login` or `Failed` events, and triggers rehash-on-login if needed.

**Q4. Why can't you compare two password hashes for equality?**
A. Bcrypt/argon embed a **random salt**, so `Hash::make('x')` differs every time. You must use `Hash::check($plain, $hash)`, which extracts the salt from the stored hash and re-derives. This also means you can never look a user up by their password column.

**Q5. What is rehash-on-login and why does it matter?**
A. After a successful login, Laravel calls `Hash::needsRehash()` to see if the stored hash uses outdated parameters (e.g. you raised `BCRYPT_ROUNDS`). If so it re-hashes with current settings and saves — upgrading security gradually without forcing password resets. Controlled by `hashing.rehash_on_login`.

**Q6. How does "remember me" work?**
A. On login with `remember = true`, Laravel stores a random token in the user's `remember_token` column and sets a long-lived cookie holding it. When the session expires, the guard re-authenticates from that cookie (`viaRemember()` returns true). Best practice: re-confirm the password for sensitive actions when `viaRemember()` is true.

**Q7. How do you implement two separate logins (users and admins)?**
A. Define a second guard and provider in `config/auth.php` (e.g. `admin` guard + `admins` provider pointing at an `Admin` model), then use `Auth::guard('admin')` and `auth:admin` middleware. Each guard has an independent session key, so they're authenticated separately.

**Q8. How do you protect login from brute-force?**
A. Rate limit with the `throttle` middleware or the `RateLimiter` facade, keyed by **email + IP** (not IP alone), clearing on success and incrementing on failure (`RateLimiter::hit/clear/tooManyAttempts`). Returns HTTP 429 / a validation error when exceeded.

**Q9. What changed about auth config/middleware in Laravel 11/12?**
A. No more `app/Http/Kernel.php` — middleware, guest redirects (`redirectGuestsTo`), and middleware aliases are configured in `bootstrap/app.php`. Config values like `AUTH_MODEL` moved to `.env` defaults. The `'password' => 'hashed'` cast (Laravel 10+) is standard, and Laravel 12 ships new first-party starter kits and bcrypt rounds default of 12.

**Q10. When would you use Sanctum vs. Passport vs. the session guard?**
A. Session guard for server-rendered browser apps. **Sanctum** for SPAs (cookie mode) and simple API tokens (bearer mode) — the default for most APIs. **Passport** only when you need a full OAuth2 server (third-party clients, authorization-code grants).

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Logging in ---
Auth::attempt(['email' => $e, 'password' => $p], $remember); // validate + login
Auth::attempt(['email' => $e, 'password' => $p, fn ($q) => $q->where('active', 1)]); // extra conditions
Auth::attemptWhen($credentials, fn ($user) => $user->isNotBanned()); // inspect user first
Auth::login($user, remember: true);     // log in a known user
Auth::loginUsingId(1);                   // by primary key
Auth::once($credentials);                // single request, no session

// --- Reading the user ---
Auth::user();  Auth::id();  Auth::check();  Auth::guest();
auth()->user();  $request->user();
Auth::viaRemember();                     // logged in via remember cookie?
Auth::guard('admin')->user();            // non-default guard

// --- Logging out ---
Auth::logout();
$request->session()->invalidate();
$request->session()->regenerateToken();
Auth::logoutOtherDevices($password);     // needs auth.session middleware

// --- Hashing ---
Hash::make($plain);                      // create hash
Hash::check($plain, $hash);              // verify
Hash::needsRehash($hash);                // upgrade check
Hash::driver('argon2id')->make($plain);

// --- Password reset ---
Password::sendResetLink(['email' => $e]);
Password::reset($data, fn ($user, $pw) => /* save */);

// --- Session security on login ---
$request->session()->regenerate();
return redirect()->intended('/dashboard');
```

```text
Route middleware:
  auth              → must be logged in
  auth:admin        → must be logged in via the "admin" guard
  guest             → must be logged OUT
  verified          → email must be verified
  password.confirm  → recent password confirmation required
  throttle:5,1      → max 5 requests / minute
  auth:sanctum      → API token / SPA auth (Sanctum)
```

```blade
{{-- Blade directives --}}
@auth ... @endauth
@guest ... @endguest
@auth('admin') ... @endauth
```

```env
# Relevant .env keys
BCRYPT_ROUNDS=12
SESSION_DRIVER=database
SESSION_SECURE_COOKIE=true
AUTH_MODEL=App\Models\User
AUTH_PASSWORD_RESET_TOKEN_TABLE=password_reset_tokens
```

---

## 🧪 Mini Exercises

1. **Manual login controller.** Without any starter kit, build a `LoginController` with `create`/`store`/`destroy` methods that validate credentials, call `Auth::attempt()` with remember-me, regenerate the session, redirect via `intended()`, and fully invalidate on logout. Add `email + IP` rate limiting that resets on success.

2. **Two-guard admin area.** Add an `Admin` model + `admins` table, register an `admin` guard and provider, and create `/admin/login` plus an `/admin/dashboard` protected by `auth:admin`. Verify that logging out of `web` does not log you out of `admin`.

3. **Rehash demonstration.** Set `BCRYPT_ROUNDS=10`, register a user, then change it to `13`. Write a small route that dumps `Hash::needsRehash($user->password)` before and after the user logs in again, and confirm the stored hash changes.

4. **Full reset + verification flow.** Wire up `Password::sendResetLink` / `Password::reset` end to end (using `php artisan tinker` or Mailpit to read emails), enforce `Password::defaults()->uncompromised()`, and require `verified` on the dashboard. Confirm an unverified user is bounced to the verification notice.

5. **Sensitive action with password confirmation.** Add a "Delete account" route behind `['auth', 'password.confirm']`. Confirm that after 3 hours the middleware re-prompts, and that `viaRemember()` users are always asked to re-confirm.
