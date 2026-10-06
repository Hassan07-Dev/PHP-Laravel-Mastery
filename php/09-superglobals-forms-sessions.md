# Superglobals, Forms, Sessions & Cookies

The browser and the server speak HTTP, a *stateless* protocol: each request stands alone, carrying no memory of the last one. PHP's job is to translate the raw HTTP request into convenient PHP variables (the **superglobals**), give you tools to read user input *safely*, and then re-introduce "memory" through **sessions** and **cookies** so that a logged-in user stays logged in across requests. This module covers that entire pipeline end to end, in plain PHP first (so you understand the machinery) and then in Laravel 12 (so you see the production-grade abstraction).

> Mental model: the browser sends an HTTP request → the web server (Apache/Nginx + PHP-FPM, or PHP's built-in server) parses it → PHP populates superglobals → your code reads them, validates them, and responds, optionally setting cookies/session data so the *next* request can be linked to this one.

---

**What you'll learn**

- What every PHP superglobal is, when it's populated, and the important `$_SERVER` keys you'll actually use.
- The real difference between `GET` and `POST` HTML forms — and why it's about semantics, not just "in the URL or not."
- How to read and validate input safely with `filter_input` / `filter_var`, and why "never trust input" is the single most important rule.
- How file uploads work: the `$_FILES` structure, `move_uploaded_file`, and validating size and MIME type correctly.
- The full session lifecycle: `session_start`, the `$_SESSION` array, `session_regenerate_id`, `session_destroy`, and key `php.ini` settings.
- Cookies in depth: `setcookie`, the `httponly` / `secure` / `samesite` flags, and how they defend against attacks.
- The flash-message pattern and the CSRF concept, plus `header()` redirects and reading a raw JSON request body for APIs.
- How all of this maps onto Laravel 12's `Request`, validation, `Session`, `Cookie`, and CSRF middleware.

---

## 1. Superglobals: PHP's window into the request

A **superglobal** is a built-in PHP array that is automatically available in *every* scope — inside functions, methods, and the global script alike — without needing the `global` keyword. PHP populates most of them automatically at the start of each request from the incoming HTTP data.

There are nine superglobals:

| Superglobal | Holds | Populated from |
|-------------|-------|----------------|
| `$GLOBALS` | All variables in the global scope | Your own top-level variables |
| `$_SERVER` | Server & request metadata (headers, paths, method, IP) | Web server / SAPI |
| `$_GET` | Query-string parameters | The URL after `?` |
| `$_POST` | Form body fields | Request body (form-encoded or multipart) |
| `$_REQUEST` | Merge of `$_GET`, `$_POST`, `$_COOKIE` | Configurable via `request_order` |
| `$_FILES` | Uploaded files | `multipart/form-data` request body |
| `$_COOKIE` | Cookies sent by the browser | The `Cookie:` request header |
| `$_SESSION` | Server-side session data | Session store (after `session_start()`) |
| `$_ENV` | Environment variables | The process environment |

> Jargon: **SAPI** (Server API) is the interface between PHP and whatever is running it — `php-fpm`, `apache2handler`, `cli`, or the built-in `cli-server`. Which SAPI you use affects which `$_SERVER` keys exist.

### `$GLOBALS` — and why you should avoid it

`$GLOBALS` lets you read/write top-level variables from inside a function:

```php
<?php
$counter = 0;

function bump(): void {
    $GLOBALS['counter']++; // mutate the global directly
}

bump();
echo $counter; // Output: 1
```

It exists mostly for legacy reasons. In modern code you pass dependencies explicitly (arguments, constructor injection) instead of reaching into globals — globals make code untestable and hard to reason about.

> PHP 8.1+ note: you can no longer wholesale-overwrite `$GLOBALS` (e.g. `$GLOBALS = []`). Writing to individual keys still works; replacing the whole array throws a fatal error.

### `$_SERVER` — the keys that matter

`$_SERVER` is a grab-bag of request and environment data. The keys you'll reach for constantly:

```php
<?php
$_SERVER['REQUEST_METHOD'];   // 'GET', 'POST', 'PUT', 'DELETE', ...
$_SERVER['REQUEST_URI'];      // '/products?page=2' (path + query string)
$_SERVER['QUERY_STRING'];     // 'page=2'
$_SERVER['HTTP_HOST'];        // 'example.com' (from the Host: header)
$_SERVER['SERVER_NAME'];      // server's configured name
$_SERVER['HTTPS'];            // 'on' if over TLS, unset/empty otherwise
$_SERVER['REMOTE_ADDR'];      // client IP as the server sees it
$_SERVER['HTTP_USER_AGENT'];  // browser UA string
$_SERVER['HTTP_REFERER'];     // the page that linked here (note the misspelling — it's in the spec)
$_SERVER['CONTENT_TYPE'];     // 'application/json', 'multipart/form-data; boundary=...'
$_SERVER['SCRIPT_NAME'];      // '/index.php'
$_SERVER['DOCUMENT_ROOT'];    // filesystem path of the web root
$_SERVER['PHP_SELF'];         // path of the executing script
```

Any request header `X-Foo` arrives as `$_SERVER['HTTP_X_FOO']` — uppercased, dashes become underscores, prefixed with `HTTP_`.

```php
<?php
// Read a custom header sent as "X-Api-Token: abc123"
$token = $_SERVER['HTTP_X_API_TOKEN'] ?? null;
echo $token; // Output: abc123
```

> Security warning: `HTTP_*` values come straight from the client and are **spoofable**. `REMOTE_ADDR` is reliable (set by the web server from the TCP connection); `HTTP_X_FORWARDED_FOR` is *not* trustworthy unless you control the proxy chain. Never use `HTTP_HOST` or `HTTP_REFERER` for security decisions without validation.

> Proxy gotcha: behind a load balancer or reverse proxy that terminates TLS, `$_SERVER['HTTPS']` may be unset (PHP sees a plain HTTP connection to the proxy) even though the user is on HTTPS — the real scheme arrives in `X-Forwarded-Proto`. Don't read that header by hand for security; configure trusted proxies (Laravel's `TrustProxies` middleware, or your web-server config) so the framework computes the scheme correctly. Also note `$_SERVER['HTTPS']` is only set when *on* — test it with `!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off'`, since some SAPIs use the literal string `'off'`.

### `$_GET`, `$_POST`, `$_REQUEST`

- `$_GET` is parsed from the URL query string regardless of HTTP method. A `POST` request to `/search?q=php` still fills `$_GET['q']`.
- `$_POST` is populated only when the body is `application/x-www-form-urlencoded` or `multipart/form-data`. A JSON body does **not** populate `$_POST` (see §8).
- `$_REQUEST` merges them. Avoid it: you lose track of *where* a value came from, which is a security smell (a value you expected in the body could be injected via the URL).

```php
<?php
// URL: /search?q=laravel&page=3
echo $_GET['q'];       // Output: laravel
echo $_GET['page'];    // Output: 3 (always a string: "3")
var_dump($_POST);      // Output: array(0) {}  (no POST body)
```

### `$_ENV` vs `getenv()`

`$_ENV` is populated from the process environment, but **only if** `php.ini`'s `variables_order` includes `E`. Because that's not guaranteed everywhere, `getenv('NAME')` is the more portable read. In Laravel you almost never touch either directly — you use `env()` (only inside config files) and `config()` everywhere else.

```php
<?php
$debug = getenv('APP_DEBUG');     // portable
$debug = $_ENV['APP_DEBUG'] ?? null; // only if variables_order has 'E'
```

---

## 2. HTML Forms: GET vs POST

A form's `method` attribute decides how the browser packages the data.

```html
<!-- GET: data goes in the query string -->
<form action="/search" method="get">
  <input name="q" value="php">
  <button>Search</button>
</form>
<!-- Browser requests: GET /search?q=php -->

<!-- POST: data goes in the request body -->
<form action="/register" method="post">
  <input name="email">
  <input name="password" type="password">
  <button>Register</button>
</form>
<!-- Browser sends a POST body: email=...&password=... -->
```

The choice is **semantic**, defined by HTTP:

| | GET | POST |
|---|-----|------|
| Intent | *Retrieve* / read (idempotent, safe) | *Submit* / change state |
| Data location | URL query string | Request body |
| Bookmarkable / shareable | Yes | No |
| Cached / in browser history / server logs | Yes (so **never** send secrets) | No |
| Length limit | Practical URL length limit (~2–8 KB) | Effectively unbounded (server config) |
| Re-submit on refresh warning | No | Yes ("Confirm form resubmission") |

Rules of thumb: use **GET** for searches, filters, pagination — anything you'd want to bookmark or share. Use **POST** for logins, creating records, payments, deletes — anything with side effects or secrets. Passwords in a GET form would land in browser history, proxy logs, and the `Referer` header. That's a real breach.

> The "Post/Redirect/Get" (PRG) pattern fixes the refresh-resubmit problem: after a successful POST, send a `302` redirect to a GET URL so a refresh re-fetches the result page instead of re-submitting the form. See §7.

---

## 3. Reading & validating input safely — *never trust input*

**The cardinal rule of web security: all input is hostile until proven otherwise.** Every value in `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, and the `HTTP_*` keys of `$_SERVER` is attacker-controllable. Validation answers "is this the shape I expect?"; sanitization/escaping answers "is this safe in *this* context (SQL, HTML, shell)?". You need both, at the right moments.

### Step 1: don't assume keys exist

Accessing a missing key emits a warning and yields `null`. Use the null-coalescing operator `??`:

```php
<?php
$page = $_GET['page'] ?? '1';     // default if absent — no warning
```

### Step 2: validate with `filter_input` / `filter_var`

`filter_input(type, name, filter, options)` reads *and* validates a single superglobal entry. `filter_var(value, filter, options)` does the same for a value you already have. They share the same filters.

```php
<?php
// Validate an email straight from POST
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
if ($email === false || $email === null) {
    // false = present but invalid; null = key absent
    http_response_code(422);
    exit('Invalid email');
}
echo $email; // Output: a syntactically valid email, e.g. ada@example.com
```

The `INPUT_*` constants are `INPUT_GET`, `INPUT_POST`, `INPUT_COOKIE`, `INPUT_SERVER`, `INPUT_ENV`. (Note: `INPUT_REQUEST` and `INPUT_SESSION` are reserved but **not implemented** — don't use them.)

Common validation filters:

```php
<?php
filter_var('42',          FILTER_VALIDATE_INT);     // int(42)
filter_var('3.14',        FILTER_VALIDATE_FLOAT);   // float(3.14)
filter_var('yes',         FILTER_VALIDATE_BOOLEAN); // true ('1','true','on','yes' => true)
filter_var('not a number',FILTER_VALIDATE_INT);     // false
filter_var('https://x.io',FILTER_VALIDATE_URL);     // string('https://x.io')
filter_var('203.0.113.5', FILTER_VALIDATE_IP);      // string('203.0.113.5')
```

> Gotcha with `FILTER_VALIDATE_BOOLEAN`: by default it returns `false` for *both* a valid falsy value (`'0'`, `'no'`, `'off'`, `'false'`) **and** an invalid/garbage value (`'banana'`) — so you can't tell "the user said no" from "the user sent nonsense." Add the `FILTER_NULL_ON_FAILURE` flag to make garbage return `null` instead, leaving `false` to mean a genuine falsy input:
>
> ```php
> <?php
> filter_var('banana', FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE); // null
> filter_var('no',     FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE); // false
> filter_var('yes',    FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE); // true
> ```

Use `options` to bound a value — this is how you enforce ranges in one call:

```php
<?php
$page = filter_input(INPUT_GET, 'page', FILTER_VALIDATE_INT, [
    'options' => ['default' => 1, 'min_range' => 1, 'max_range' => 1000],
]);
// /products?page=2     => int(2)
// /products?page=abc   => int(1)  (unparseable -> default)
// /products?page=99999 => int(1)  (out of range -> ALSO the default, when 'default' is set)
```

> Gotcha (verified on PHP 8.x): when you supply a `'default'`, it is returned for **both** an unparseable value *and* a value that falls outside `min_range`/`max_range`. There is no way to distinguish "missing" from "out of range" once a default is set.
>
> If you need to tell those cases apart, **omit `'default'`** — then the filter returns `false` on any failure (bad parse *or* out of range), and you handle `false` yourself:
>
> ```php
> <?php
> $page = filter_input(INPUT_GET, 'page', FILTER_VALIDATE_INT, [
>     'options' => ['min_range' => 1, 'max_range' => 1000],   // no 'default'
> ]);
> // /products?page=99999 => false   (no default => failure returns false)
> // /products?page=abc   => false
> $page = ($page === false || $page === null) ? 1 : $page;     // apply default manually
> ```

### Step 3: escape on output (context matters)

Validation isn't escaping. To prevent **XSS** (Cross-Site Scripting — injecting `<script>` into a page), escape when rendering into HTML:

```php
<?php
$name = $_GET['name'] ?? '';
echo '<p>Hello ' . htmlspecialchars($name, ENT_QUOTES | ENT_HTML5, 'UTF-8') . '</p>';
// Input: <script>alert(1)</script>
// Output: <p>Hello &lt;script&gt;alert(1)&lt;/script&gt;</p>  (rendered as text, not executed)
```

> PHP 8.1+ note: the default flag for `htmlspecialchars`/`htmlentities` changed from `ENT_QUOTES | ENT_HTML401` to `ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401`, so single quotes are now escaped by default and invalid UTF-8 is replaced instead of producing an empty string. Passing the flags explicitly (as above) is still the clearest, most portable habit. Note `htmlspecialchars` escapes the five HTML special chars only — for a value inside an HTML *attribute* always wrap it in quotes, and for JS/CSS contexts you need a different escaper.

To prevent **SQL injection**, never concatenate input into SQL. Use prepared statements:

```php
<?php
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ?');
$stmt->execute([$email]);   // the driver handles quoting/escaping safely
```

> Important: the old `FILTER_SANITIZE_STRING` was **deprecated in PHP 8.1 and removed in 8.4**. Do not use it. For HTML output use `htmlspecialchars`; for stripping tags use `strip_tags` deliberately, knowing it's lossy.

### Laravel 12: validation done right

Plain PHP teaches the mechanics; Laravel gives you a declarative, reusable layer. The framework reads input, validates against rules, auto-redirects back with errors on failure, and (critically) old PHP superglobal pitfalls largely vanish.

```php
<?php
// app/Http/Controllers/RegisterController.php
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;

public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([
        'email'    => ['required', 'email', 'unique:users,email'],
        'age'      => ['required', 'integer', 'min:18', 'max:120'],
        'website'  => ['nullable', 'url'],
    ]);

    // $validated contains ONLY the keys you validated — a safe, whitelisted array.
    User::create($validated);

    return redirect()->route('dashboard')->with('status', 'Registered!');
}
```

For complex rules, extract a **Form Request** (`php artisan make:request StoreUserRequest`), which keeps validation + authorization out of the controller. Reading input in Laravel:

```php
<?php
$request->input('email');          // single field (works for GET, POST, JSON)
$request->query('page', 1);        // query string only, with default
$request->boolean('subscribe');    // casts '1','true','on','yes' => true
$request->integer('page');         // cast to int
$request->only(['email', 'name']); // whitelist
$request->except(['password']);    // blacklist
$request->has('coupon');           // key present?
$request->filled('coupon');        // present AND not empty
```

Blade auto-escapes by default — `{{ $name }}` runs `htmlspecialchars` for you. Use `{!! $html !!}` only for trusted HTML.

---

## 4. File uploads

When a form has `enctype="multipart/form-data"`, uploaded files land in `$_FILES` (not `$_POST`).

```html
<form action="/upload" method="post" enctype="multipart/form-data">
  <input type="file" name="avatar">
  <button>Upload</button>
</form>
```

### The `$_FILES` structure

For a single file input named `avatar`, `$_FILES['avatar']` is an array:

```php
<?php
/*
array(5) {
  ["name"]     => "photo.jpg"        // original client filename (UNTRUSTED)
  ["type"]     => "image/jpeg"       // client-supplied MIME (UNTRUSTED — spoofable)
  ["tmp_name"] => "/tmp/phpA1b2C3"   // server temp path
  ["error"]    => 0                  // UPLOAD_ERR_OK
  ["size"]     => 51234              // bytes (trust this)
  ["full_path"]=> "photo.jpg"        // PHP 8.1+: original path incl. subdirs for directory uploads
}
*/
```

The `error` codes are the first thing to check:

| Constant | Value | Meaning |
|----------|-------|---------|
| `UPLOAD_ERR_OK` | 0 | Success |
| `UPLOAD_ERR_INI_SIZE` | 1 | Exceeds `upload_max_filesize` |
| `UPLOAD_ERR_FORM_SIZE` | 2 | Exceeds form's `MAX_FILE_SIZE` hidden field |
| `UPLOAD_ERR_PARTIAL` | 3 | Only partially uploaded |
| `UPLOAD_ERR_NO_FILE` | 4 | No file submitted |
| `UPLOAD_ERR_NO_TMP_DIR` | 6 | Missing temp folder |
| `UPLOAD_ERR_CANT_WRITE` | 7 | Disk write failed |
| `UPLOAD_ERR_EXTENSION` | 8 | A PHP extension blocked it |

### Validating and storing a file safely

```php
<?php
const MAX_BYTES   = 2 * 1024 * 1024;                 // 2 MB
const ALLOWED_EXT = ['jpg' => 'image/jpeg', 'png' => 'image/png'];

function handleUpload(array $file, string $destDir): string
{
    // 1. Check the error code first.
    if (($file['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
        throw new RuntimeException('Upload failed, code ' . $file['error']);
    }

    // 2. Enforce size on the server (never trust the client's MAX_FILE_SIZE).
    if ($file['size'] > MAX_BYTES) {
        throw new RuntimeException('File too large.');
    }

    // 3. Confirm it was actually an HTTP upload, not a forged tmp_name path.
    if (!is_uploaded_file($file['tmp_name'])) {
        throw new RuntimeException('Possible attack: not an uploaded file.');
    }

    // 4. Determine the REAL MIME type from contents, ignoring the client header.
    $finfo = new finfo(FILEINFO_MIME_TYPE);
    $mime  = $finfo->file($file['tmp_name']);
    $ext   = array_search($mime, ALLOWED_EXT, true);
    if ($ext === false) {
        throw new RuntimeException("Disallowed type: {$mime}");
    }

    // 5. Generate a safe, random filename — NEVER reuse the client's name.
    $safeName = bin2hex(random_bytes(16)) . '.' . $ext;
    $target   = rtrim($destDir, '/') . '/' . $safeName;

    // 6. Move it out of the temp dir into permanent storage.
    if (!move_uploaded_file($file['tmp_name'], $target)) {
        throw new RuntimeException('Could not move uploaded file.');
    }

    return $safeName;
}

// Usage
try {
    $stored = handleUpload($_FILES['avatar'], __DIR__ . '/uploads');
    echo "Saved as {$stored}"; // Output: Saved as 9f86d081...e3b0c4.jpg
} catch (RuntimeException $e) {
    http_response_code(422);
    echo $e->getMessage();
}
```

Why each step matters:
- **`is_uploaded_file()`** ensures `tmp_name` is genuinely a PHP-managed upload, blocking an attacker who guesses a server path to leak a file.
- **`finfo` (magic-byte) detection** ignores the spoofable `type` field. A `.php` shell renamed to `.jpg` would be caught.
- **Random filenames + storing outside the web root** (or a directory with execution disabled) prevents an uploaded `.php` from being requested and executed.
- Server-side size checks are mandatory because client `MAX_FILE_SIZE` is trivially bypassed.

### Multiple files

`<input type="file" name="docs[]" multiple>` produces a "transposed" structure — each key is an array:

```php
<?php
// $_FILES['docs']['name'][0], ['name'][1], ['tmp_name'][0], etc.
foreach ($_FILES['docs']['tmp_name'] as $i => $tmp) {
    if ($_FILES['docs']['error'][$i] === UPLOAD_ERR_OK) {
        // process file $i
    }
}
```

### Laravel 12 file uploads

Laravel normalizes all of this into an `UploadedFile` object and provides validated, secure storage:

```php
<?php
$request->validate([
    'avatar' => ['required', 'image', 'mimes:jpg,png', 'max:2048'], // max is in KILOBYTES
]);

// store() generates a random name and saves to the 'public' disk; returns the path.
$path = $request->file('avatar')->store('avatars', 'public');
// e.g. "avatars/3kP9...e1.jpg"

// Or keep the original (sanitized) name:
$path = $request->file('avatar')->storeAs('avatars', 'logo.png', 'public');
```

- `image` validates the file is a real image (jpg, png, gif, webp, etc.) by inspecting it, not by trusting the extension.
- `mimes:jpg,png` validates against the **guessed extension** — Laravel reads the file's true MIME type (magic bytes) and checks that the extension Symfony derives from it is in your list. (`mimes:jpg` accepts `image/jpeg`.) If you want to match the literal MIME string instead, use `mimetypes:image/jpeg,image/png`.
- `max:2048` means 2048 **kilobytes** (about 2 MB) — Laravel's size rules are always in KB for files.

Either way Laravel inspects the actual bytes, so a `.php` shell renamed to `.jpg` is rejected — the same magic-byte principle as the plain-PHP `finfo` approach above.

---

## 5. Sessions

HTTP is stateless, so the server needs a way to recognize "this is the same visitor as a moment ago." A **session** solves it: the server stores per-user data on the *server side* (file, database, Redis…) and gives the browser a single opaque **session ID** in a cookie. On each request the browser sends that ID back, and PHP loads the matching data into `$_SESSION`.

```
Request 1: browser sends no cookie → PHP creates session "abc123" → sets Cookie: PHPSESSID=abc123
Request 2: browser sends Cookie: PHPSESSID=abc123 → PHP loads $_SESSION for abc123
```

### Starting a session

```php
<?php
session_start();              // MUST be called before any output (it sends headers)
$_SESSION['user_id'] = 42;    // write
$count = $_SESSION['views'] ?? 0;
$_SESSION['views'] = $count + 1;
echo $_SESSION['views'];      // Output: 1, then 2, then 3... on each reload
```

`session_start()` must run **before any bytes are sent** to the browser, because it emits the `Set-Cookie` header. A stray space before `<?php`, an `echo`, or BOM in the file will trigger: `Warning: session_start(): Cannot send session cookie - headers already sent`.

You can pass runtime options:

```php
<?php
session_start([
    'cookie_lifetime' => 0,        // 0 = until browser closes
    'cookie_httponly' => true,     // JS can't read the session cookie (anti-XSS theft)
    'cookie_secure'   => true,     // only sent over HTTPS
    'cookie_samesite' => 'Lax',    // CSRF mitigation
    'use_strict_mode' => true,     // reject client-supplied unknown session IDs
]);
```

### Lifecycle: login, regenerate, logout

**Session fixation** is an attack where an attacker plants a known session ID in the victim's browser, then waits for them to log in under that ID. Defense: regenerate the ID at every privilege change (especially right after login).

```php
<?php
// --- LOGIN ---
session_start();
if (password_verify($inputPassword, $user->password_hash)) {
    session_regenerate_id(true);     // new ID; true = delete the old session file
    $_SESSION['user_id'] = $user->id;
    $_SESSION['logged_in_at'] = time();
}

// --- LOGOUT (the correct, complete teardown) ---
session_start();
$_SESSION = [];                                   // 1. clear in-memory data
if (ini_get('session.use_cookies')) {             // 2. expire the browser cookie
    $p = session_get_cookie_params();
    setcookie(session_name(), '', [
        'expires'  => time() - 42000,
        'path'     => $p['path'],
        'domain'   => $p['domain'],
        'secure'   => $p['secure'],
        'httponly' => $p['httponly'],
        'samesite' => $p['samesite'],
    ]);
}
session_destroy();                                // 3. delete server-side storage
```

> Why all three steps? `$_SESSION = []` clears data for the current request, `setcookie(...)` tells the browser to drop the cookie, and `session_destroy()` removes the server-side store. Skipping any one leaves a reusable artifact.

`session_unset()` clears `$_SESSION` but keeps the session alive; `session_destroy()` kills the server-side data but does *not* clear `$_SESSION` in the current request nor remove the cookie — hence the manual cookie expiry above.

### Key `php.ini` session settings

| Directive | Purpose / good value |
|-----------|----------------------|
| `session.save_handler` | `files` (default), `redis`, `memcached`, custom |
| `session.save_path` | Where file sessions live |
| `session.gc_maxlifetime` | Seconds of inactivity before GC eligibility (e.g. `1440`) |
| `session.cookie_lifetime` | `0` = session cookie (dies with browser) |
| `session.cookie_httponly` | `1` — keep it on |
| `session.cookie_secure` | `1` in production (HTTPS) |
| `session.cookie_samesite` | `Lax` or `Strict` |
| `session.use_strict_mode` | `1` — rejects uninitialized IDs (anti-fixation) |
| `session.use_only_cookies` | `1` — never accept the ID from the URL |

> Under the hood: with the default `files` handler, each session is a file like `sess_abc123` in `save_path`. Garbage collection isn't a timer — it runs *probabilistically* on `session_start()`, governed by `gc_probability`/`gc_divisor` (e.g. 1/100 = a 1% chance per request). On many distros a cron job handles cleanup instead, so `gc_probability` is set to 0.

### Laravel 12 sessions

Laravel abstracts the store behind the `session` service and the `Session` facade; `config/session.php` chooses the driver (`file`, `cookie`, `database`, `redis`, …). Laravel handles ID regeneration on login for you.

```php
<?php
session(['cart_count' => 3]);          // write via helper
$count = session('cart_count', 0);     // read with default

// Facade / Request API
$request->session()->put('key', 'val');
$request->session()->get('key');
$request->session()->forget('key');
$request->session()->flush();          // clear all
$request->session()->regenerate();     // e.g. after login
$request->session()->invalidate();     // flush + regenerate (logout)
$request->session()->pull('key');      // get then forget
```

---

## 6. Cookies

A **cookie** is a small key/value string the server asks the browser to store and send back on subsequent requests to the same domain. Sessions use *one* cookie (the ID); but cookies are also used directly for preferences, "remember me" tokens, and analytics. Unlike sessions, cookie *values* live on the client, so they're visible and tamperable — never store secrets or trust them unsigned.

### Setting a cookie

`setcookie()` sends a `Set-Cookie` header, so — like `session_start()` — it must run **before any output**.

```php
<?php
// Modern array-options signature (PHP 7.3+):
setcookie('theme', 'dark', [
    'expires'  => time() + 60 * 60 * 24 * 30, // 30 days; 0 = session cookie
    'path'     => '/',                         // valid site-wide
    'domain'   => '',                          // current host only
    'secure'   => true,                        // HTTPS only
    'httponly' => true,                        // not readable by JavaScript
    'samesite' => 'Lax',                       // 'Strict' | 'Lax' | 'None'
]);
```

The cookie is **not** available in `$_COOKIE` on the request that set it — only from the *next* request:

```php
<?php
// Next request:
$theme = $_COOKIE['theme'] ?? 'light';
echo $theme; // Output: dark
```

### The security flags explained

- **`httponly`**: the cookie is invisible to `document.cookie` in JavaScript. This blocks an XSS payload from stealing a session cookie. Always set it for auth/session cookies.
- **`secure`**: the browser only sends the cookie over HTTPS, preventing interception on plaintext connections.
- **`samesite`**: controls whether the cookie rides along on cross-site requests — the core CSRF defense at the cookie layer:
  - `Strict` — never sent on cross-site navigation (most secure; can break "click a link from email into your logged-in app").
  - `Lax` (modern browser default) — sent on top-level GET navigations only; not on cross-site POST/iframe/fetch.
  - `None` — always sent, **but requires `secure`** or browsers reject it.

### Deleting a cookie

There's no "delete" call — you re-set it with a past expiry and the *same* path/domain:

```php
<?php
setcookie('theme', '', ['expires' => time() - 3600, 'path' => '/']);
```

> Gotcha: a cookie set with `path => '/app'` is a *different* cookie from one set with `path => '/'`. To delete, the path and domain must match exactly, or the browser keeps the original.

### Laravel 12 cookies

Laravel **encrypts and signs** all cookies by default (via `EncryptCookies` middleware), so they're tamper-evident out of the box.

```php
<?php
use Illuminate\Support\Facades\Cookie;

// Queue a cookie to be attached to the outgoing response (minutes, not seconds):
return response('OK')->cookie('theme', 'dark', 60 * 24 * 30);

// Or via the facade queue:
Cookie::queue('theme', 'dark', 60 * 24 * 30);

// Read:
$theme = $request->cookie('theme', 'light');

// Forget:
return response('Bye')->withoutCookie('theme');
```

To store a value the JS layer must read (e.g. an SPA CSRF token), add it to the `except` list in the encrypt-cookies config so it isn't encrypted.

---

## 7. Headers, redirects, and the flash-message pattern

### `header()` and redirects

`header()` sends a raw HTTP response header — again, before any output. A redirect is a `Location` header plus a 3xx status:

```php
<?php
// After a successful POST (Post/Redirect/Get):
header('Location: /dashboard', true, 302); // 302 Found (temporary)
exit;                                       // ALWAYS stop — header() does not halt the script
```

Status codes you should know: `301` permanent redirect, `302`/`303` temporary (`303 See Other` is the technically-correct PRG code that forces a GET), `307`/`308` redirects that preserve the method. Use `http_response_code(404)` to set the status without a redirect.

> Gotcha: forgetting `exit;` after a redirect is a classic bug — the rest of the script keeps running (and can leak data or perform writes) even though the browser will navigate away.

### The flash-message pattern

A **flash message** is data that survives exactly *one* redirect, then auto-deletes — perfect for "Your profile was saved." after a PRG redirect. In plain PHP you build it on top of the session:

```php
<?php
session_start();

// --- After the POST, before redirecting ---
$_SESSION['flash'] = ['type' => 'success', 'text' => 'Profile saved!'];
header('Location: /profile', true, 303);
exit;

// --- On the destination page ---
session_start();
if (!empty($_SESSION['flash'])) {
    $flash = $_SESSION['flash'];
    unset($_SESSION['flash']);   // consume it so it shows only once
    echo "<div class='{$flash['type']}'>"
       . htmlspecialchars($flash['text'], ENT_QUOTES) . "</div>";
}
// Output (first load after redirect): <div class='success'>Profile saved!</div>
// Output (on refresh): (nothing — it was consumed)
```

Laravel has this built in. `->with()` flashes to the session; `@session` / `$errors` read it in Blade:

```php
<?php
return redirect()->route('profile')->with('status', 'Profile saved!');
```

```blade
{{-- resources/views/profile.blade.php --}}
@if (session('status'))
    <div class="alert alert-success">{{ session('status') }}</div>
@endif

{{-- Validation errors are auto-flashed on a failed validate() --}}
@error('email')
    <span class="text-red-600">{{ $message }}</span>
@enderror
```

Under the hood, Laravel stores flashed keys in a special `_flash.new` list in the session; after the next request renders, the session middleware "ages" them into `_flash.old` and then deletes them — implementing the show-once behavior.

---

## 8. Reading the raw request body (JSON APIs)

Form-encoded bodies fill `$_POST`. A JSON API client sends `Content-Type: application/json` with a JSON body — and PHP does **not** populate `$_POST` for that. You read the raw body from the `php://input` stream and decode it yourself:

```php
<?php
// Incoming: POST /api/users  Content-Type: application/json
// Body: {"name":"Ada","age":36}

$raw = file_get_contents('php://input');
$data = json_decode($raw, true, flags: JSON_THROW_ON_ERROR); // assoc array; throws on malformed JSON

echo $data['name']; // Output: Ada
echo $data['age'];  // Output: 36
```

Notes:
- Use `JSON_THROW_ON_ERROR` (PHP 7.3+) so bad JSON raises a `JsonException` instead of silently returning `null`. Wrap the decode in `try/catch (\JsonException $e)` and return a `400`.
- `php://input` is **not** available for `multipart/form-data` (PHP consumes that into `$_POST`/`$_FILES`), but works for JSON, XML, and other raw bodies. Since PHP 5.6 the stream is re-readable for most SAPIs, so reading it more than once is generally fine; capture it into a variable once and reuse that to be safe.
- Always guard with a check on `$_SERVER['CONTENT_TYPE']` if your endpoint accepts multiple formats.
- For large payloads, note that `file_get_contents('php://input')` buffers the entire body into memory; stream it with `fopen('php://input', 'r')` if you expect very large uploads.

```php
<?php
$contentType = $_SERVER['CONTENT_TYPE'] ?? '';
if (str_contains($contentType, 'application/json')) {
    $data = json_decode(file_get_contents('php://input'), true, flags: JSON_THROW_ON_ERROR);
} else {
    $data = $_POST;
}
```

Laravel does all of this transparently — `$request->input('name')`, `$request->json('name')`, and `$request->all()` work identically whether the client sent form data or JSON. That's a big reason to prefer the framework's request object over touching superglobals directly.

---

## 9. CSRF, briefly (see the Security module for depth)

**CSRF** (Cross-Site Request Forgery) tricks a logged-in user's browser into submitting a state-changing request to your site without their intent — exploiting the fact that the browser auto-attaches cookies (including the session cookie) to *any* request to your domain. Example: a malicious page silently POSTs to `https://yourbank.com/transfer` while the victim is logged in.

The standard defense is the **synchronizer token**: the server embeds an unpredictable, per-session token in every form; the attacker's page can't read it (same-origin policy), so a forged request lacks it and is rejected.

```html
<!-- Plain PHP: generate once, store in session, embed in the form, verify on submit -->
<form method="post" action="/transfer">
  <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION['csrf_token']) ?>">
  <!-- ...fields... -->
</form>
```

```php
<?php
// Generate (e.g. when rendering the form)
$_SESSION['csrf_token'] ??= bin2hex(random_bytes(32));

// Verify on POST — hash_equals prevents timing attacks
if (!hash_equals($_SESSION['csrf_token'] ?? '', $_POST['csrf_token'] ?? '')) {
    http_response_code(419);
    exit('CSRF token mismatch');
}
```

Laravel automates this: the `VerifyCsrfToken` middleware checks every non-GET request, and you just drop `@csrf` into a Blade form. Combine tokens with `SameSite=Lax/Strict` cookies for defense in depth.

```blade
<form method="POST" action="/transfer">
    @csrf
    {{-- expands to <input type="hidden" name="_token" value="..."> --}}
</form>
```

> Laravel 11/12 note: middleware registration moved to `bootstrap/app.php` (the old `app/Http/Kernel.php` is gone). To exempt URIs from CSRF you use `$middleware->validateCsrfTokens(except: [...])` in `bootstrap/app.php`.

---

## ⚠️ Common Mistakes & Gotchas

1. **"Headers already sent" when calling `session_start()`, `setcookie()`, or `header()`.**
   These send HTTP headers, so any prior output — even a blank line or BOM before `<?php`, or a stray `echo` — breaks them.
   *Fix:* send all headers/cookies/redirects **before** any output. Remove trailing whitespace after closing `?>` (or omit `?>` entirely in pure-PHP files). The error message tells you the file and line where output started.

2. **Trusting the client-supplied file type or filename on uploads.**
   `$_FILES[...]['type']` and `['name']` come from the browser and are forgeable; a `shell.php` can claim `image/jpeg`.
   *Fix:* verify the real type with `finfo`/`FILEINFO_MIME_TYPE`, call `is_uploaded_file()`, generate a random server-side filename, and store outside the web root (or where execution is disabled). Always enforce size server-side.

3. **Forgetting `exit;` after a redirect.**
   `header('Location: ...')` does not stop execution; the script keeps running and may perform writes or leak content.
   *Fix:* always `exit;` (or `return` in a framework controller) immediately after issuing a redirect.

4. **Not regenerating the session ID at login (session fixation), or an incomplete logout.**
   Reusing the pre-login ID lets an attacker who planted it ride the authenticated session; a logout that only does `$_SESSION = []` leaves the cookie and server file alive.
   *Fix:* `session_regenerate_id(true)` right after authentication; on logout clear `$_SESSION`, expire the cookie with `setcookie()`, and call `session_destroy()`.

5. **Expecting `$_POST` to be filled for a JSON request body.**
   JSON (`application/json`) does not populate `$_POST`; you'll see an empty array and think the data vanished.
   *Fix:* read `file_get_contents('php://input')` and `json_decode(..., true, flags: JSON_THROW_ON_ERROR)`, or in Laravel just use `$request->input()`.

6. **Using `$_REQUEST` or `FILTER_SANITIZE_STRING`.**
   `$_REQUEST` blurs the source of data (URL vs body vs cookie), enabling parameter injection; `FILTER_SANITIZE_STRING` was removed in PHP 8.4.
   *Fix:* read the specific superglobal you mean (`$_GET`/`$_POST`), and escape on output with `htmlspecialchars` instead of sanitizing input destructively.

7. **`setcookie()` value appearing immediately in `$_COOKIE`.**
   It won't — `$_COOKIE` reflects what the browser *sent*, so a cookie you just set shows up only on the *next* request.
   *Fix:* if you need the value within the same request, set both `$_COOKIE['x']` manually and call `setcookie()`.

8. **Assuming an out-of-range `filter_var` returns `false` even when a `'default'` is set.**
   With a `'default'` option, both an unparseable value *and* an out-of-range value return the default — you cannot distinguish "missing" from "too big," and you may silently accept a clamped value you didn't intend.
   *Fix:* omit `'default'` if you need to detect failures (the filter then returns `false` on any failure), and apply your fallback manually after checking for `false`/`null`.

9. **Setting `SameSite=None` without `Secure`.**
   Modern browsers reject (or drop) a `SameSite=None` cookie that isn't also marked `Secure`, so a cross-site cookie you "set" never comes back.
   *Fix:* always pair `'samesite' => 'None'` with `'secure' => true` (and serve over HTTPS).

10. **Reading the JSON body twice / decoding without error handling.**
    `json_decode` without `JSON_THROW_ON_ERROR` returns `null` on malformed input, which is indistinguishable from a legitimate JSON `null` — bugs hide here.
    *Fix:* capture `php://input` into a variable once, decode with `JSON_THROW_ON_ERROR`, and `try/catch (\JsonException $e)` to return a `400`.

---

## ✅ Best Practices

- **Never trust input.** Validate shape/type/range on the way in; escape for the correct context (HTML, SQL, shell, header) on the way out. Use prepared statements for SQL and `htmlspecialchars` for HTML.
- **Prefer the framework's request object** (`$request->input()`, validation rules, Form Requests) over raw superglobals — it normalizes form vs JSON and gives you a validated whitelist.
- **Use the right HTTP method:** GET for safe/idempotent reads, POST/PUT/PATCH/DELETE for state changes. Apply Post/Redirect/Get to avoid double submits.
- **Lock down cookies:** set `httponly`, `secure`, and `samesite` on every cookie, especially auth/session cookies. In Laravel, keep cookie encryption on.
- **Harden sessions:** `use_strict_mode=1`, `use_only_cookies=1`, regenerate the ID on privilege changes, and pick a scalable store (Redis/database) for multi-server apps.
- **Validate uploads defensively:** check the error code, enforce size, detect MIME from contents, randomize names, and store outside the document root.
- **Keep secrets out of URLs.** No passwords/tokens in GET parameters — they leak to logs, history, and `Referer`.
- **Use CSRF protection on every state-changing form** (`@csrf` in Laravel) and pair it with `SameSite` cookies.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between GET and POST, and when do you choose each?**
GET appends data to the URL query string; it's meant for safe, idempotent reads, is cacheable/bookmarkable, and shows up in history and logs. POST puts data in the request body; it's for state changes and secrets, isn't cached, and warns on refresh. Choose GET for search/filter/pagination, POST for logins, creates, payments, and deletes. The distinction is semantic (defined by HTTP), not merely cosmetic.

**Q2. How do PHP sessions work under the hood?**
On `session_start()`, PHP looks for a session ID in the `PHPSESSID` cookie. If absent (and `use_strict_mode` is on), it generates a new one and sends a `Set-Cookie`. It then loads server-side data (by default a file `sess_<id>` in `save_path`) into `$_SESSION`. At script end, `$_SESSION` is serialized back to the store. Garbage collection of stale files is probabilistic, controlled by `gc_probability/gc_divisor` and `gc_maxlifetime` (or handled by cron). The browser only ever holds the opaque ID — the data stays on the server.

**Q3. Why and when do you call `session_regenerate_id()`?**
To prevent session fixation. If an attacker can set or learn a victim's session ID before login, they can reuse it afterward. Regenerating the ID (with `true` to delete the old store) at every privilege boundary — most importantly right after successful authentication — issues a fresh ID the attacker doesn't know, severing the fixed session.

**Q4. How do you validate a file upload securely?**
Check `['error'] === UPLOAD_ERR_OK`; enforce the size limit server-side (don't rely on the client `MAX_FILE_SIZE`); confirm `is_uploaded_file($tmp_name)`; detect the true MIME via `finfo` rather than trusting `['type']`; generate a random filename; and store outside the web root (or disable execution there). Then `move_uploaded_file()` it into place.

**Q5. Explain `httponly`, `secure`, and `samesite` on cookies.**
`httponly` hides the cookie from JavaScript (`document.cookie`), mitigating cookie theft via XSS. `secure` restricts the cookie to HTTPS, preventing plaintext interception. `samesite` controls cross-site sending: `Strict` never sends cross-site, `Lax` (default) sends only on top-level GET navigations, `None` always sends but mandates `secure`. Together they harden against XSS-based theft and CSRF.

**Q6. What is CSRF and how do synchronizer tokens stop it?**
CSRF abuses the browser auto-attaching cookies to forge a state-changing request from a malicious site while the user is authenticated. A synchronizer token is a per-session unpredictable value embedded in each form; the attacker's cross-origin page can't read it (same-origin policy), so its forged request omits the token and the server rejects it. Use `hash_equals()` to compare tokens to avoid timing attacks, and pair with `SameSite` cookies.

**Q7. Why doesn't `$_POST` contain my JSON payload, and how do you read it?**
PHP only auto-parses `application/x-www-form-urlencoded` and `multipart/form-data` into `$_POST`. For `application/json` you read the raw stream: `json_decode(file_get_contents('php://input'), true, flags: JSON_THROW_ON_ERROR)`. Laravel's `$request->input()` handles both transparently.

**Q8. What causes "Cannot send headers, headers already sent"?**
Any output before a header-sending call (`session_start`, `setcookie`, `header`). Common culprits: whitespace/BOM before `<?php`, content after `?>`, debug `echo`s, or warnings printed earlier. Fix by ensuring no output precedes header calls — and omit the closing `?>` in pure-PHP files so a trailing newline can't leak.

**Q9. What's the difference between `session_unset()`, `session_destroy()`, and clearing the cookie?**
`session_unset()` empties `$_SESSION` but keeps the session/store. `session_destroy()` deletes the server-side data but leaves `$_SESSION` (this request) and the browser cookie intact. A complete logout does all three: clear `$_SESSION`, expire the cookie with `setcookie()`, then `session_destroy()`.

**Q10. Why prefer `filter_var`/framework validation over manual `if` checks?**
Hand-rolled checks are easy to get subtly wrong (e.g. validating emails or URLs by regex). `filter_var` uses vetted, battle-tested validators with range/options support, and frameworks add reusable rule sets, automatic error redirects, and a validated whitelist output that prevents mass-assignment of unexpected fields.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Superglobals ---
$_GET['x'] / $_POST['x'] / $_COOKIE['x'] / $_SESSION['x'] / $_FILES['x']
$_SERVER['REQUEST_METHOD' | 'REQUEST_URI' | 'HTTP_HOST' | 'REMOTE_ADDR' | 'CONTENT_TYPE']
$_SERVER['HTTP_X_CUSTOM']  // request header "X-Custom"
getenv('NAME')             // portable env read

// --- Safe input ---
$v = $_GET['k'] ?? 'default';
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
$n = filter_input(INPUT_GET, 'page', FILTER_VALIDATE_INT,
        ['options' => ['default' => 1, 'min_range' => 1]]);
echo htmlspecialchars($s, ENT_QUOTES | ENT_HTML5, 'UTF-8'); // escape for HTML
$pdo->prepare('... WHERE id = ?')->execute([$id]);           // SQL safety

// --- File upload ---
is_uploaded_file($f['tmp_name']);
(new finfo(FILEINFO_MIME_TYPE))->file($f['tmp_name']);       // real MIME
move_uploaded_file($f['tmp_name'], $target);

// --- Sessions ---
session_start(['cookie_httponly'=>true,'cookie_secure'=>true,'cookie_samesite'=>'Lax']);
session_regenerate_id(true);   // after login
session_destroy();             // + clear $_SESSION + expire cookie on logout

// --- Cookies ---
setcookie('k','v',['expires'=>time()+3600,'path'=>'/','secure'=>true,
                   'httponly'=>true,'samesite'=>'Lax']);
setcookie('k','',['expires'=>time()-3600,'path'=>'/']); // delete (match path/domain)

// --- Redirect / status ---
header('Location: /x', true, 303); exit;
http_response_code(404);

// --- Raw JSON body ---
$data = json_decode(file_get_contents('php://input'), true, flags: JSON_THROW_ON_ERROR);
```

```php
// --- Laravel 12 ---
$request->input('email'); $request->query('page',1); $request->boolean('flag');
$request->validate(['email'=>['required','email']]);          // auto-redirects on fail
$request->file('avatar')->store('avatars','public');
session(['k'=>'v']); session('k','default'); $request->session()->regenerate();
return response('ok')->cookie('k','v',60);                     // minutes
return redirect('/x')->with('status','Saved!');               // flash message
// Blade: {{ $x }} (escaped) | @csrf | @error('field') {{ $message }} @enderror
```

---

## 🧪 Mini Exercises

1. **Safe search endpoint.** Write a plain-PHP script that accepts `?q=...&page=N` via GET, validates `page` as an integer ≥ 1 (default 1) using `filter_input`, escapes `q` for HTML output, and prints `Searching for "<q>" on page <N>`. Verify what happens with `?page=abc` and `?page=-5`.

2. **Upload guard.** Build an upload handler for a single image that rejects anything over 1 MB, rejects any file whose *real* MIME (via `finfo`) isn't `image/jpeg` or `image/png`, stores it under a random filename, and returns the stored path. Test it by renaming a `.txt` to `.jpg` and confirming it's rejected.

3. **Login/logout flow.** Implement `login()` and `logout()` functions over native sessions: `login()` regenerates the session ID and stores `user_id`; `logout()` performs the complete three-step teardown (clear `$_SESSION`, expire the cookie, `session_destroy()`). Add a `view_count` that increments per request to prove the session persists between requests but resets after logout.

4. **Flash messages.** Implement a `flash($key, $msg)` setter and a `consumeFlash($key)` getter on top of `$_SESSION` such that a message set before a `303` redirect displays exactly once on the destination page and is gone on refresh.

5. **JSON API echo.** Write an endpoint that accepts a POST with `Content-Type: application/json` body `{"name": "...", "items": [...]}`, reads it via `php://input` with `JSON_THROW_ON_ERROR`, validates that `name` is a non-empty string and `items` is an array, and responds with `Content-Type: application/json` echoing back `{"received": <count>, "name": "..."}`. Handle malformed JSON with a `400` status.
