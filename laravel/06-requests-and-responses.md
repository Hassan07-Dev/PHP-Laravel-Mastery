# Requests & Responses in Laravel

Every HTTP interaction in a Laravel application is a conversation: the client sends a **request**, your code reads it, and you hand back a **response**. Mastering the `Illuminate\Http\Request` and `Illuminate\Http\Response` objects is the difference between writing controllers that *feel* like magic and writing controllers you actually understand. This module takes you from `$request->name` to streamed downloads, content negotiation, and how the request object is actually built under the hood.

> **Target stack:** PHP 8.4 and Laravel 12. Where Laravel 10/11 or PHP 8.1–8.3 behave differently, it is called out inline.

---

**What you'll learn**

- How to retrieve input safely with `input()`, `query()`, dynamic properties, `all()`, `only()`, `except()`, the presence helpers `has()`, `filled()`, and the typed accessors `boolean()`, `integer()`, `float()`, `string()`, `array()`, `date()`, `enum()`, and `collect()`.
- How to merge extra data into the request with `merge()` and `mergeIfMissing()`.
- How old input flashing powers "repopulate the form after a validation error" with zero manual wiring.
- How to handle uploaded files and store them with `store()` / `storeAs()`.
- How to read request metadata: path, host, URL, method, IP, headers, bearer tokens, and how to detect JSON/AJAX clients.
- How to read and queue cookies on requests and responses.
- How to return every kind of response: strings, arrays, JSON, files, downloads, and streamed output (`stream()`, `streamDownload()`, `streamJson()`, `eventStream()` SSE), with custom status codes and headers.
- How redirects work, including `back()`, `route()`, `away()`, and chaining `with()`, `withInput()`, and `withErrors()`.
- How content negotiation (`accepts()`, `prefers()`, `wantsJson()`) and response macros let you keep controllers thin.
- How the request/response lifecycle actually works in Laravel 11/12 — where the kernel and middleware now live.

---

## Why a Request Object At All?

In raw PHP you reach for superglobals: `$_GET`, `$_POST`, `$_SERVER`, `$_FILES`, `$_COOKIE`. They work, but they are global mutable state — anyone can read or scribble on them, they are hard to test, and they make no distinction between query-string data and JSON body data. **Jargon:** a *superglobal* is a built-in PHP array available in every scope without `global`.

Laravel wraps all of that in a single object, `Illuminate\Http\Request`, which extends Symfony's `Request`. Instead of `$_POST['email']` you write `$request->input('email')`. The benefits:

- **One unified API** for query string, form body, JSON body, route parameters, files, and headers.
- **Testability** — you can construct a fake request in a test instead of mutating globals.
- **Safety** — typed accessors, default values, and presence checks instead of `isset()` gymnastics.

You get the request by type-hinting it. Laravel's service container sees the type-hint and injects the current request automatically:

```php
use Illuminate\Http\Request;

Route::post('/users', function (Request $request) {
    return $request->input('name');
});
```

This works in controllers too — type-hint `Request` as the first parameter (after any route-model-bound parameters) and the container resolves it for you. You can also grab it anywhere via the `request()` helper or the `Request` facade, though dependency injection is preferred because it is explicit and testable.

---

## Retrieving Input

### `input()` — the universal accessor

`input()` reads from **both** the query string and the request body (form-encoded or JSON), with body taking precedence. The second argument is a default returned when the key is absent.

```php
$name = $request->input('name');          // null if missing
$name = $request->input('name', 'Guest'); // 'Guest' if missing
```

It supports "dot" notation for nested arrays — handy for JSON bodies:

```php
// JSON body: {"user": {"name": "Ada"}, "tags": ["php", "laravel"]}
$request->input('user.name'); // 'Ada'
$request->input('tags.0');    // 'php'
$request->input('tags.*');    // ['php', 'laravel']  (wildcard returns all)
```

Calling `input()` with no arguments returns the entire merged input array (query + body) as an associative array. The key difference from `all()`: `all()` also folds in **uploaded files**, whereas `input()` does not. So for a request with file uploads, use `all()` (or `file()`) when you need the files too.

### `query()` — query string only

When you specifically want the URL query string and want to ignore the body, use `query()`. Same signature as `input()`.

```php
// GET /search?term=laravel&page=2
$request->query('term');        // 'laravel'
$request->query('page', 1);     // '2' (note: strings!)
$request->query();              // ['term' => 'laravel', 'page' => '2']
```

> Query and form values arrive as **strings**. `page` above is `'2'`, not `2`. Use `integer()` (below) or cast deliberately when you need a real int.

### Dynamic property access

You can read input as if it were a property: `$request->name`. Under the hood `Request::__get()` first looks for the key in the **request payload (input)**, and only if it is not present there does it fall back to the **matched route's parameters**. (This order trips people up — many assume route parameters win, but per the Laravel 12 docs it is *input first, then route parameters*.) This is concise but ambiguous — `$request->user` could also collide with the authenticated-user accessor (`$request->user()` is a method, not input).

```php
$name = $request->name; // ~ $request->input('name'); if no input 'name', falls back to a route param named 'name'
```

**Prefer `input()` in non-trivial code.** Reserve dynamic access for quick reads where there is no naming collision risk.

### `all()`, `only()`, `except()`

```php
$request->all();                       // everything: query + body + files, as one array
$request->all(['name', 'email']);      // only these keys (missing keys present as null)
$request->only(['name', 'email']);     // subset; missing keys are simply absent
$request->except(['password', '_token']); // everything EXCEPT these keys
```

`only()` and `except()` are your friends when mass-assigning. **Never** pass `$request->all()` straight into `Model::create()` — that is how mass-assignment vulnerabilities happen. Use `only()` or, better, `$request->validated()` from a Form Request.

### Presence and emptiness helpers

These read better than `isset()` and `empty()` and handle the array/dot cases for you.

```php
$request->has('name');            // key is PRESENT (even if value is "" or null)
$request->has(['name', 'email']); // true only if ALL keys present
$request->hasAny(['a', 'b']);     // true if ANY present
$request->missing('nickname');    // inverse of has()

$request->filled('name');         // present AND not empty ("", null, [] count as empty)
$request->isNotFilled('name');    // inverse
$request->anyFilled(['a', 'b']);  // any of them filled
```

> **Gotcha — `has()` vs `filled()` with the default middleware:** Laravel 11/12 apps ship with the `TrimStrings` and `ConvertEmptyStringsToNull` middleware in the global stack. That means a submitted-but-blank text field arrives as `null`, not `""`. So `has('name')` is still `true` (the key *is* present), but `filled('name')` is `false`. If you need empty-string semantics, you can opt those fields out of normalization in `bootstrap/app.php` via `$middleware->convertEmptyStringsToNull(except: [...])`.

A common pattern — run a callback only when a key exists or is filled:

```php
$request->whenHas('name', function (string $name) {
    // runs only if 'name' is present
});

$request->whenFilled('name', function (string $name) {
    // runs only if 'name' is present and non-empty
});

// Each accepts an optional SECOND closure that runs when the condition is NOT met:
$request->whenFilled('name', fn (string $n) => /* filled */ null, fn () => /* not filled */ null);
$request->whenMissing('nickname', fn () => /* missing */ null, fn () => /* present */ null);
```

### Typed retrieval helpers

Because raw input is stringy, Laravel provides typed accessors that cast for you. These are the ones interviewers love.

```php
$request->boolean('subscribe');  // true for: 1, "1", true, "true", "on", "yes"; false otherwise
$request->integer('page', 1);    // (int) cast, default 1 if missing
$request->float('amount');       // (float) cast
$request->string('name');        // returns a Stringable (Illuminate\Support\Stringable), NOT a plain string
$request->array('versions');     // always casts to an array; [] if missing
$request->date('published_at');  // a Carbon instance, or null if the key is MISSING
$request->date('published_at', '!Y-m-d', 'UTC'); // with format + timezone
$request->enum('status', Status::class);                 // a backed-enum case, or null if missing/invalid
$request->enum('status', Status::class, Status::Pending); // with a default for missing/invalid (Laravel 11+)
$request->enums('roles', Role::class);                   // array of enum cases (Laravel 11+)
$request->collect();             // ALL input as an Illuminate Collection
$request->collect('tags');       // a single key as a Collection
```

> **`date()` and invalid formats:** if the key is *missing*, `date()` returns `null`. But if the value is *present yet does not match the expected format*, it throws an `InvalidArgumentException`. Validate the field (e.g. with the `date` rule) before calling `date()`, or be prepared to catch that exception.

`boolean()` is the classic gotcha-fixer: an HTML checkbox sends `"on"` or nothing, and `"0"`/`"false"` strings are truthy in PHP. `$request->boolean('subscribe')` normalizes all of that.

```php
// Checkbox checked  -> "on"  -> boolean('subscribe') === true
// Checkbox unchecked-> absent -> boolean('subscribe') === false
// JSON  {"flag":"0"} -> boolean('flag') === false   (a plain (bool)"0" would be false too,
//                                                     but (bool)"false" === true — boolean() fixes that)
```

`string()` returns a `Stringable`, so you can chain string methods. Cast to a plain string with `(string)` or `->toString()` when needed:

```php
$slug = $request->string('title')->slug()->value(); // e.g. "Hello World" -> "hello-world"
```

> **PHP 8.4 note:** none of these accessors depend on a specific PHP minor version, but PHP 8.1+ is required for the backed-enum casting used by `enum()`/`enums()`. On PHP 8.4 you also get cleaner enum syntax overall, but the Laravel API is identical across 8.1–8.4. (Laravel 12 itself requires PHP 8.2 or newer.)

### Merging additional input

Sometimes you need to inject or default values *into* the request before it reaches validation or a controller (common in middleware). Use `merge()` to add/overwrite keys and `mergeIfMissing()` to add only keys that are not already present:

```php
$request->merge(['status' => 'pending']);        // overwrites 'status' if it already exists
$request->mergeIfMissing(['votes' => 0]);          // adds 'votes' only if absent
```

This mutates the request's input bag, so subsequent `input()`/`validated()` calls see the merged data.

---

## Old Input & Flashing

**The problem:** a user fills a long form, submits, validation fails, and you redirect back. Without help, every field is now empty and the user is furious.

**The solution:** *flashing*. Laravel can stash the current input into the session for the **next** request only, then you re-read it with `old()`.

```php
$request->flash();                 // flash ALL input to the session
$request->flashOnly(['name']);     // flash a subset
$request->flashExcept(['password']); // flash everything except sensitive fields
```

You rarely call these manually, because redirect helpers do it for you:

```php
return redirect('/form')->withInput();                 // flashes all input
return redirect('/form')->withInput($request->except('password'));
return back()->withInput();                            // most common
```

Then in Blade, repopulate fields with the `old()` helper. The second argument is the fallback for the very first page load:

```blade
<input type="text" name="name" value="{{ old('name') }}">
<input type="email" name="email" value="{{ old('email', $user->email) }}">
```

**Validation does this automatically.** When validation fails (via `$request->validate()` or a Form Request), Laravel redirects back *and* flashes input for you — you only need `old()` in the view. That is why you almost never write `flash()` by hand.

---

## Files

Uploaded files are not in `input()` — they are accessed via `file()` (or dynamic property access), and arrive as `Illuminate\Http\UploadedFile` instances (an extension of Symfony's `UploadedFile`, itself wrapping PHP's `$_FILES`).

```php
$file = $request->file('avatar');   // UploadedFile|null
$file = $request->avatar;           // dynamic access also works

if ($request->hasFile('avatar') && $request->file('avatar')->isValid()) {
    // safe to proceed
}
```

Useful methods on `UploadedFile`:

```php
$file->getClientOriginalName();      // "vacation.jpg" (user-supplied — do NOT trust as a path)
$file->getClientOriginalExtension(); // "jpg"   (client-supplied — also untrusted)
$file->extension();                  // guessed from the file's CONTENTS/MIME type (safer)
$file->getMimeType();                // "image/jpeg"
$file->getSize();                    // bytes
$file->path();                       // temp path on disk (Laravel's UploadedFile::path(); getRealPath() also works)
```

> `UploadedFile` extends PHP's `SplFileInfo` (via Symfony's `UploadedFile`), so all the Symfony/`SplFileInfo` methods are available too. Prefer `extension()` over `getClientOriginalExtension()` when deciding how to handle a file, since the client-supplied extension can lie.

### Storing files

The framework integrates with the filesystem (Flysystem) so you do not handle temp paths yourself.

```php
// store() picks a random unique name, returns the path relative to the disk root
$path = $request->file('avatar')->store('avatars');
// e.g. "avatars/9xQk2v...hashed.jpg"

// choose a disk (configured in config/filesystems.php)
$path = $request->file('avatar')->store('avatars', 's3');

// storeAs() lets you set the filename explicitly
$path = $request->file('avatar')->storeAs('avatars', 'user-42.jpg');
$path = $request->file('avatar')->storeAs('avatars', 'user-42.jpg', 's3');

// storePublicly() forces public visibility
$path = $request->file('avatar')->storePublicly('avatars', 's3');

// hashName() is the same random-name generator store() uses, if you need it yourself
$name = $request->file('avatar')->hashName(); // "9xQk2v...hashed.jpg"
```

> **Disk root (Laravel 11/12):** the default `local` disk now roots at `storage/app/private` (private by default), and the `public` disk roots at `storage/app/public` (served via the `public/storage` symlink created by `php artisan storage:link`). So `store('avatars')` on the default disk lands in `storage/app/private/avatars`, not the web-accessible directory — pass the `public` disk when you want a publicly reachable file.

You can also store directly through a `Storage` disk, which is handy when the source is not an `UploadedFile`:

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('s3')->put('avatars/foo.jpg', file_get_contents($file->getRealPath()));
// or, memory-efficient streaming straight from an UploadedFile:
$path = Storage::disk('s3')->putFile('avatars', $request->file('avatar'));
```

> **Security:** never build a destination path from `getClientOriginalName()` without sanitizing — that opens path-traversal (`../../etc/...`). `store()`/`storeAs()` keep you inside the disk root, and `store()`'s hashed name avoids collisions and leaking user file names.

---

## Request Metadata

The request knows everything about *how* it arrived.

```php
$request->path();          // "users/42"      (no leading slash, no query string)
$request->url();           // "https://app.test/users/42" (no query string)
$request->fullUrl();       // "https://app.test/users/42?ref=email" (with query string)
$request->fullUrlWithQuery(['page' => 2]); // append/override query params
$request->fullUrlWithoutQuery(['ref']);    // strip given query params (Laravel 10+)

$request->host();              // "app.test"        (host only)
$request->httpHost();          // "app.test:8080"   (host + non-standard port)
$request->schemeAndHttpHost(); // "https://app.test" (scheme + host)

$request->method();        // "POST"
$request->isMethod('post'); // true (case-insensitive)

$request->ip();            // "203.0.113.7"  (client IP; honors trusted proxies)
$request->ips();           // array of IPs from the forwarded chain

$request->is('admin/*');   // does the path match this pattern?
$request->routeIs('users.*'); // does the matched route NAME match?
```

> **Trusted proxies:** behind a load balancer, the real client IP is in `X-Forwarded-For`. Laravel only trusts that header from proxies you whitelist. In Laravel 11/12 this is configured fluently in `bootstrap/app.php`:
>
> ```php
> ->withMiddleware(function (Middleware $middleware): void {
>     $middleware->trustProxies(at: ['10.0.0.0/8']);   // or at: '*' for cloud LBs
> })
> ```
>
> In Laravel 10 and earlier this lived in `App\Http\Middleware\TrustProxies` (a file that no longer exists in a fresh Laravel 11/12 app). Misconfigure it and `ip()` returns your load balancer's IP instead of the client. The Laravel 12 docs also caution that IP addresses are user-controllable and should be treated as untrusted, informational data.

### Headers and bearer tokens

```php
$request->header('Accept');                 // "application/json" (or null)
$request->header('X-Custom', 'fallback');   // with default
$request->hasHeader('Authorization');       // bool
$request->bearerToken();                     // the token from "Authorization: Bearer <token>"
```

`bearerToken()` parses the `Authorization: Bearer xyz` header and returns just `xyz` — exactly what Sanctum/Passport token guards consume.

> **Gotcha:** when there is no `Authorization` header, `bearerToken()` returns an **empty string `''`, not `null`** (per the Laravel 12 docs). So guard with `if ($token = $request->bearerToken())` rather than `!== null`.

### Detecting the client's intent

```php
$request->expectsJson();   // true if XHR/AJAX OR Accept header prefers JSON
$request->wantsJson();     // true ONLY if Accept header explicitly prefers JSON
$request->acceptsJson();   // true if the client accepts application/json at all
$request->acceptsHtml();   // true if the client accepts text/html
$request->accepts(['text/html', 'application/json']); // true if ANY of these is accepted
$request->getAcceptableContentTypes();                // raw array of accepted types, best-first
$request->ajax();          // true if X-Requested-With: XMLHttpRequest
$request->pjax();          // true if a PJAX request
$request->prefers(['application/json', 'text/html']); // the single content type the client prefers most (or null)
```

The practical difference: `wantsJson()` checks only the `Accept` header; `expectsJson()` is broader (also true for AJAX requests even without a JSON Accept header). Laravel's exception handler uses `expectsJson()` to decide whether to render an HTML error page or a JSON error body — which is why an API client gets `{"message": "..."}` and a browser gets a styled error page.

---

## Cookies on the Request

Read incoming cookies via `cookie()`:

```php
$value = $request->cookie('preferred_theme');          // null if absent
$value = $request->cookie('preferred_theme', 'light'); // with default
```

> By default Laravel **encrypts** cookies through the `EncryptCookies` middleware. So `$request->cookie('x')` transparently decrypts; a cookie set outside Laravel (or excluded from encryption) will read as its raw value. If you need an unencrypted cookie, add its name to the `EncryptCookies` exception list (configured in `bootstrap/app.php` via `encryptCookies(except: [...])` in Laravel 11/12).

Sending cookies back is covered under **Queued Response Cookies** below.

---

## Responses

A controller does not have to manually build a `Response`. Laravel inspects whatever you return and converts it.

### Strings and arrays (auto-conversion)

```php
Route::get('/hi', fn () => 'Hello');             // text/html, 200
Route::get('/data', fn () => ['name' => 'Ada']); // auto-JSON! Content-Type: application/json
Route::get('/list', fn () => collect([1, 2, 3])); // collections auto-JSON too
```

Returning an array or a `Jsonable`/`Arrayable` (including Eloquent models and collections) produces a JSON response automatically. This is the idiomatic way to write API endpoints.

### The `response()` helper

When you need control over status or headers, use the `response()` helper, which returns an `Illuminate\Http\Response` you can fluently configure.

```php
return response('Created', 201)
    ->header('X-Custom', 'value')
    ->withHeaders([                 // set several at once
        'X-Version' => '1.0',
        'X-Env'     => app()->environment(),
    ]);
```

Use HTTP status constants from Symfony for readability instead of magic numbers:

```php
use Symfony\Component\HttpFoundation\Response as HttpResponse;

return response('No', HttpResponse::HTTP_FORBIDDEN); // 403
```

### `response()->json()`

For explicit JSON with status codes and headers:

```php
return response()->json([
    'message' => 'Created',
    'id'      => $user->id,
], 201);

// JSONP (rare): wrap in a callback
return response()->json(['ok' => true])->withCallback($request->input('callback'));
```

`json()` sets `Content-Type: application/json` and JSON-encodes Laravel-aware structures (models, collections, enums) for you. For full API output shaping, reach for **API Resources** (`JsonResource`) — outside this module's scope, but they wrap `json()` internally.

### `noContent()` and the empty 204

```php
return response()->noContent();      // 204 No Content, empty body — ideal for DELETE
```

### File responses: `download()` and `file()`

```php
// Force a download with a Content-Disposition: attachment header
return response()->download($pathOnLocalDisk);
return response()->download($pathOnLocalDisk, 'invoice.pdf'); // custom filename
return response()->download($pathOnLocalDisk, 'invoice.pdf', [
    'X-Header' => 'value',
]);
// delete the file after sending (e.g. a temp export)
return response()->download($tmpPath)->deleteFileAfterSend();

// Display inline in the browser (Content-Disposition: inline) — e.g. show a PDF/image
return response()->file($pathToPdf);

// Serve a file that lives on a Storage disk
return Storage::disk('s3')->download('exports/report.csv'); // forced download (documented)
return Storage::disk('local')->response('docs/manual.pdf'); // inline from a disk
```

`response()->download()` and `response()->file()` expect an **absolute path on the server's filesystem** (a string), not an `UploadedFile`. For files stored on a configured disk, use the disk's own `download()` (documented) or `response()` methods shown above. (Note: Symfony's HttpFoundation, which backs these, requires an ASCII filename for the download name.)

### Streamed responses

When the payload is large or generated on the fly (think a 2 GB CSV export), you do not want to build it all in memory. Stream it.

```php
// Stream arbitrary chunks
return response()->stream(function () {
    foreach (largeDataset() as $row) {
        echo implode(',', $row) . "\n";
        ob_flush();
        flush();
    }
}, 200, [
    'Content-Type'      => 'text/csv',
    'X-Accel-Buffering' => 'no', // disable Nginx response buffering so chunks arrive immediately
]);

// Convenience for downloads — accepts (callback, filename, headers)
return response()->streamDownload(function () {
    echo generateHugePdf();
}, 'report.pdf');
```

> **Generator shortcut (Laravel 11/12):** if the closure you pass to `stream()` *returns a Generator* (uses `yield`), Laravel automatically flushes the output buffer between yields and disables Nginx buffering for you — no manual `ob_flush()/flush()` needed:
>
> ```php
> return response()->stream(function (): Generator {
>     foreach (largeDataset() as $row) {
>         yield implode(',', $row) . "\n";
>     }
> }, 200, ['Content-Type' => 'text/csv']);
> ```

#### `streamJson()` — incremental JSON

For large datasets sent as a single JSON document but produced lazily (e.g. from a database cursor), use `streamJson()`. It serializes progressively so you never build the whole array in memory:

```php
use App\Models\User;

return response()->streamJson([
    'users' => User::cursor(), // lazily streamed, not loaded all at once
]);
```

#### `eventStream()` — Server-Sent Events (SSE)

Laravel 11/12 adds `eventStream()` for SSE (`Content-Type: text/event-stream`), useful for streaming LLM tokens or live updates. Yield values (or `StreamedEvent` instances to name the event):

```php
use Illuminate\Http\StreamedEvent;

return response()->eventStream(function () {
    foreach (tokenGenerator() as $token) {
        yield $token;                                  // a default "message" event
        // or: yield new StreamedEvent(event: 'update', data: $token);
    }
});
```

By default `eventStream()` sends a final `</stream>` message when the generator finishes, which the browser's `EventSource` (or Laravel's `useEventStream` JS hook) can watch for to close the connection.

---

## Redirects

A redirect is just a response with a `Location` header and a 3xx status. The `redirect()` helper (or the `to_route()`/`back()` helpers) builds one.

```php
return redirect('/dashboard');                 // to a URL
return redirect()->to('/dashboard', 301);      // with explicit status
return redirect()->route('users.show', $user); // to a named route (passes a model -> its key)
return redirect()->route('users.show', ['user' => 42]);
return to_route('users.show', $user);          // shorthand for redirect()->route(...) (Laravel 9+)
return redirect()->back();                     // previous URL (uses session)
return back();                                 // shorthand
return redirect()->away('https://external.example.com'); // EXTERNAL url, no URL-generator validation
return redirect()->action([UserController::class, 'index']); // to a controller action
return redirect()->intended('/dashboard');     // to where the user was headed before login
```

> **Why `away()` exists:** `redirect()->to()` runs the target through Laravel's URL generator (good for internal paths). For an arbitrary external URL you must use `away()`. **Never** pass unvalidated user input to a redirect target — that is an open-redirect vulnerability. Validate against an allowlist first.

### Chaining flash data onto a redirect

This is where redirects shine in form workflows:

```php
return redirect()->route('profile.edit')
    ->with('status', 'Profile updated!')   // flash a one-request session value
    ->withInput()                          // flash old input (re-fill the form)
    ->withErrors(['email' => 'Taken'])     // flash a MessageBag (used by $errors in Blade)
    ->withCookie(cookie('seen_tour', true, 60)); // queue a cookie
```

Read the flashed value back in the next request:

```blade
@if (session('status'))
    <div class="alert">{{ session('status') }}</div>
@endif

@error('email')
    <span class="error">{{ $message }}</span>
@enderror
```

`withErrors()` accepts an array, a `MessageBag`, or a `Validator` instance; Laravel shares the errors as the `$errors` variable in every view via the `ShareErrorsFromSession` middleware (part of the `web` middleware group). That is why `$errors` is *always* available in Blade even when there are none.

---

## Queued (Response) Cookies

You attach cookies to outgoing responses two ways. Either build the cookie and chain `withCookie()`:

```php
$response = response('Welcome')->withCookie(
    cookie('name', 'value', minutes: 60)   // name, value, lifetime-in-minutes
);
```

…or *queue* a cookie, which attaches it to whatever response is eventually returned — useful in middleware or when you do not hold the response object:

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('theme', 'dark', 60);        // attached to the outgoing response automatically
Cookie::queue(Cookie::make('theme', 'dark', 60));
```

To delete a cookie:

```php
return response('Bye')->withoutCookie('theme'); // when you hold the response
Cookie::expire('theme');                         // when you don't (Laravel 12's documented helper)
Cookie::queue(Cookie::forget('theme'));          // older equivalent; still works
```

> The `cookie()` helper and `Cookie` facade produce *encrypted, HttpOnly* cookies by default (HttpOnly means JavaScript cannot read them — a security win). Cookies set this way are signed/encrypted unless excluded from the `EncryptCookies` middleware.

---

## Content Negotiation

Content negotiation means returning a *different representation* of the same resource based on what the client asked for (via the `Accept` header). You drive it with the request's preference methods:

```php
public function show(Request $request, Report $report)
{
    if ($request->wantsJson()) {
        return response()->json($report);
    }

    if ($request->prefers(['html', 'json']) === 'html') {
        return view('reports.show', compact('report'));
    }

    return response()->download($report->pdfPath());
}
```

`prefers()` inspects the `Accept` header and returns the *best match* from the content types you offer, or `null` if none match. This keeps a single URL serving HTML to browsers and JSON to API clients.

---

## Response Macros (Mention)

A **macro** is a way to add a custom method to a class at runtime via the `Macroable` trait. `ResponseFactory` is macroable, so you can teach `response()` a reusable helper. Register macros in a service provider's `boot()` method:

```php
use Illuminate\Support\Facades\Response;

// In App\Providers\AppServiceProvider::boot()
Response::macro('caps', function (string $value) {
    return Response::make(strtoupper($value));
});

// Anywhere:
return response()->caps('hello'); // "HELLO"
```

Macros are great for a standardized API envelope (`response()->success($data)`, `response()->fail($message, 422)`), keeping controllers terse and consistent.

---

## How It Works Under the Hood

When a request hits `public/index.php`, Laravel calls `Request::capture()`, which builds an `Illuminate\Http\Request` from PHP's superglobals (`$_GET`, `$_POST`, `$_SERVER`, `$_FILES`, `$_COOKIE`) by extending Symfony's `Request::createFromGlobals()`. That single object is bound into the service container as a singleton, so every `Request` type-hint, the `request()` helper, and the `Request` facade all resolve to the **same instance**.

The request flows through the **HTTP kernel** and the middleware pipeline (each middleware receives `$request` and a `$next` closure). **Laravel 11/12 note:** the kernel still exists — it is `Illuminate\Foundation\Http\Kernel` inside the framework — but the old user-facing `app/Http/Kernel.php` is **gone**. You no longer edit a kernel file to register middleware or configure groups; instead you do it fluently in `bootstrap/app.php` via `->withMiddleware(...)`. (Likewise there is no `app/Console/Kernel.php` — scheduled tasks live in `routes/console.php`, exception handling is configured with `->withExceptions(...)`, and providers are listed in `bootstrap/providers.php`.)

Your route closure or controller runs, and whatever it returns is passed to `Router::prepareResponse()`, which:

1. If it is already a `Symfony\Response`, leaves it alone.
2. If it is a `Responsable` (implements `toResponse()`), calls that.
3. If it is an array / `Arrayable` / `Jsonable` / `JsonSerializable`, wraps it in a `JsonResponse`.
4. Otherwise wraps it in a plain `Response`.

Then the response travels *back out* through the middleware (which is why middleware can modify the response, e.g. `AddQueuedCookiesToResponse` attaches queued cookies, and `EncryptCookies` encrypts them). Finally `Response::send()` emits the status line, headers, and body. Knowing this pipeline explains *why* queued cookies and shared `$errors` "just appear" — middleware on the outbound trip put them there.

---

## ⚠️ Common Mistakes & Gotchas

1. **Mass-assigning `$request->all()`.**
   `User::create($request->all())` lets an attacker set any column (e.g. `is_admin`). **Fix:** use `$request->only([...])`, or validate first and pass `$request->validated()` from a Form Request. Also keep `$fillable`/`$guarded` correct on the model as a second line of defense.

2. **Treating query/form input as already typed.**
   `$request->input('page') + 1` works by coercion, but `if ($request->input('active'))` is `true` even for the string `"false"` or `"0"`... wait — `"0"` is falsy, but `"false"` is **truthy** in PHP. **Fix:** use `$request->boolean('active')` and `$request->integer('page')` instead of trusting raw values.

3. **Expecting `old()` to repopulate without flashing input.**
   If you redirect with `redirect('/form')` (no `->withInput()`), `old('name')` returns the default. **Fix:** chain `->withInput()` on manual redirects. (Validation failures flash automatically, so this bites mostly on hand-rolled redirects.)

4. **Using `$request->all()` to read uploaded files for `store()`.**
   Files are not plain input — `$request->input('avatar')` is `null` for a file field. **Fix:** use `$request->file('avatar')` / `$request->hasFile('avatar')`, and always check `->isValid()` before storing.

5. **Building file paths from `getClientOriginalName()`.**
   The original name is attacker-controlled and can contain `../` or collide with other users' files. **Fix:** use `store()` (hashed name) or sanitize and namespace with `storeAs("users/{$id}", $safeName)`.

6. **Confusing `wantsJson()` and `expectsJson()`.**
   An AJAX request without an explicit JSON `Accept` header returns `false` from `wantsJson()` but `true` from `expectsJson()`. **Fix:** use `expectsJson()` when you want "this is probably an API/AJAX call," and `wantsJson()` when you strictly mean "the client asked for JSON."

7. **Open redirects via `away()` or `intended()` with user input.**
   `redirect()->away($request->input('next'))` lets an attacker phish via your domain. **Fix:** validate the target against an allowlist of internal routes/hosts before redirecting.

8. **Wrong client IP behind a proxy.**
   `$request->ip()` returns the load balancer's IP unless trusted proxies are configured. **Fix:** configure `trustProxies()` in `bootstrap/app.php` so `X-Forwarded-For` is honored from your proxies only.

9. **Hunting for `app/Http/Kernel.php` to register middleware.**
   In Laravel 11/12 there is no `app/Http/Kernel.php` or `app/Console/Kernel.php`. **Fix:** register/append/prepend middleware and groups in `bootstrap/app.php` via `->withMiddleware(...)`; configure exceptions via `->withExceptions(...)`; put scheduled tasks in `routes/console.php`; list providers in `bootstrap/providers.php`. The classes like `EncryptCookies` and `ShareErrorsFromSession` still exist — only the *registration surface* moved.

10. **Assuming a blank field is `""` so `filled()` is `true`.**
    With the default `ConvertEmptyStringsToNull` middleware, an empty submitted field becomes `null`. `has('name')` is still `true` but `filled('name')` is `false`. **Fix:** use `filled()`/`whenFilled()` when you mean "has a real value," and opt fields out of normalization in `bootstrap/app.php` only if you genuinely need empty strings.

---

## ✅ Best Practices

- **Validate first, then read.** Prefer a Form Request and use `$validated = $request->validated()` so you only ever touch known-good, typed data.
- **Use `only()`/`validated()` for mass assignment** — never `all()`.
- **Use typed accessors** (`boolean()`, `integer()`, `date()`, `enum()`) instead of manual casts.
- **Return arrays / Eloquent models / API Resources** from API controllers and let Laravel JSON-encode them; reach for `response()->json()` only when you need a custom status or headers.
- **Use named routes in redirects** (`redirect()->route('users.show', $user)`) so URLs stay refactor-safe.
- **Use HTTP status constants** (`Response::HTTP_UNPROCESSABLE_ENTITY`) or at least name them via comments — avoid bare magic numbers.
- **Stream large payloads** with `streamDownload()`/`stream()` (or `streamJson()` for cursor-backed JSON, `eventStream()` for SSE) instead of building gigabytes in memory. Prefer the Generator (`yield`) form in Laravel 11/12 so flushing and Nginx buffering are handled for you.
- **Keep secrets out of flashed input** — `flashExcept(['password'])` or `withInput($request->except('password'))`.
- **Standardize API output with a response macro** or API Resources so every endpoint has a consistent envelope.
- **Centralize content negotiation** rather than sprinkling `wantsJson()` checks; the exception handler already does this for errors.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is the difference between `input()` and `query()`?**
`input()` reads from both the query string and the request body (JSON or form), with the body winning on conflicts. `query()` reads only the URL query string. Use `query()` when you specifically want to ignore body data.

**Q2. How does dynamic property access (`$request->name`) actually resolve?**
`Request::__get()` first looks for the key in the request **input** (query + body); only if it is absent there does it fall back to the matched **route's parameters**. (A common interview trap is to state the reverse — route params first — but the Laravel 12 docs are explicit that input wins.) Because the lookup is implicit and can collide with accessors like `user()`, prefer `input()` in real code.

**Q3. Why use `$request->boolean('x')` instead of `(bool) $request->input('x')`?**
HTML and JSON send booleans as strings. `(bool) "false"` is `true` and `(bool) "0"` is `false`, which is inconsistent. `boolean()` normalizes a known set of truthy values (`1, "1", "true", "on", "yes"`) to `true` and everything else to `false`.

**Q4. How does old input repopulate a form after validation fails?**
On a failed `validate()`/Form Request, Laravel redirects back and flashes the input into the session for one request. In Blade, `old('field')` reads that flashed value. The `$errors` `MessageBag` is also flashed and shared into every view by the `ShareErrorsFromSession` middleware.

**Q5. What is the difference between `wantsJson()`, `expectsJson()`, and `acceptsJson()`?**
`acceptsJson()` is true if the client *accepts* JSON at all. `wantsJson()` is true only if the client *prefers* JSON via its `Accept` header. `expectsJson()` is the broadest — true for explicit JSON preference **or** AJAX (`X-Requested-With`). Laravel's exception handler uses `expectsJson()` to decide HTML vs JSON error output.

**Q6. (Under the hood) Walk me through how a returned value becomes an HTTP response.**
The router passes your return value to `prepareResponse()`. If it is already a Symfony response, it is used as-is. A `Responsable` is converted via `toResponse()`. Arrays/`Arrayable`/`Jsonable`/`JsonSerializable` become a `JsonResponse`; everything else becomes a plain `Response`. The response then travels back out through middleware (attaching queued cookies, encrypting them, etc.) before `send()` emits headers and body.

**Q7. How do you stream a huge export without exhausting memory?**
Use `response()->streamDownload($callback, $filename)` or `response()->stream($callback, $status, $headers)`. You `echo`/`yield` chunks inside the callback and flush, so the response body is produced incrementally instead of held in memory.

**Q8. How do cookies get encrypted, and how do you opt out?**
Outgoing cookies pass through the `EncryptCookies` middleware which encrypts them; incoming ones are decrypted before you read them with `$request->cookie()`. To skip encryption for a specific cookie (e.g. one read by JavaScript or a third party), add it to the `EncryptCookies` exception list (`encryptCookies(except: [...])` in `bootstrap/app.php` for Laravel 11/12, or the `$except` property in the middleware class for Laravel 10).

**Q9. Why is returning an Eloquent model or array from a controller enough to get JSON?**
Because the router detects `Arrayable`/`Jsonable`/`JsonSerializable` (models and collections implement these) and wraps the value in a `JsonResponse`, setting the `application/json` content type automatically. No `response()->json()` needed.

**Q10. What is a response macro and when would you use one?**
A macro adds a method to `ResponseFactory` at runtime via the `Macroable` trait, registered in a provider's `boot()`. Use it to standardize repeated response shapes (a consistent API success/error envelope) so controllers stay thin and output stays uniform.

**Q11. (Laravel 11/12) Where do middleware and the HTTP kernel live now?**
There is no longer an `app/Http/Kernel.php`. The kernel still exists internally as `Illuminate\Foundation\Http\Kernel`, but you configure the middleware pipeline fluently in `bootstrap/app.php` with `->withMiddleware(function (Middleware $middleware) { ... })` — using methods like `append`, `prepend`, `appendToGroup`, `trustProxies`, and `encryptCookies(except: ...)`. Similarly, exception rendering/reporting moved to `->withExceptions(...)`, the console kernel is gone (schedule in `routes/console.php`), and providers are registered in `bootstrap/providers.php`. The middleware *classes* (`EncryptCookies`, `ShareErrorsFromSession`, `AddQueuedCookiesToResponse`, etc.) are unchanged — only where you wire them up changed.

**Q12. How do you stream a response without manually flushing the buffer?**
In Laravel 11/12, return a `Generator` (a closure that `yield`s) from `response()->stream(...)`; the framework flushes between yields and disables Nginx buffering automatically. For SSE use `response()->eventStream(...)`, and for large JSON built from a cursor use `response()->streamJson(...)`. Manual `ob_flush(); flush();` is only needed when your callback `echo`s instead of yielding.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Reading input ---
$request->input('k', $default);     // body + query (body wins), dot-notation OK
$request->query('k', $default);     // query string only
$request->all();  $request->only([...]);  $request->except([...]);
$request->has('k');  $request->hasAny([...]);  $request->missing('k');
$request->filled('k');  $request->whenFilled('k', fn ($v) => ...);

// --- Typed input ---
$request->boolean('k');  $request->integer('k', 0);  $request->float('k');
$request->string('k');   // Stringable
$request->date('k');     // Carbon
$request->enum('k', Status::class);   $request->enums('k', Role::class);

// --- Old input / flashing ---
$request->flash();  $request->flashExcept(['password']);
return back()->withInput();          // then old('field') in Blade

// --- Files ---
$request->file('f');  $request->hasFile('f');  $file->isValid();
$file->store('dir');  $file->storeAs('dir', 'name.ext', 'disk');

// --- Metadata ---
$request->path();  $request->url();  $request->fullUrl();
$request->host();  $request->schemeAndHttpHost();
$request->method();  $request->isMethod('post');  $request->ip();
$request->header('H', $default);  $request->bearerToken(); // "" when no header, not null
$request->expectsJson();  $request->wantsJson();  $request->ajax();
$request->accepts(['application/json']);  $request->prefers(['application/json','text/html']);
$request->is('admin/*');  $request->routeIs('users.*');

// --- Cookies ---
$request->cookie('k', $default);                 // read (auto-decrypted)
response('x')->withCookie(cookie('k','v',60));   // set
Cookie::queue('k','v',60);                       // queue
response('x')->withoutCookie('k');               // delete

// --- Responses ---
return ['ok' => true];                           // auto-JSON
return response('Body', 201)->header('X','Y');
return response()->json($data, 201);
return response()->noContent();                  // 204
return response()->download($path, 'name.pdf')->deleteFileAfterSend();
return response()->file($path);                  // inline
return response()->streamDownload(fn () => ..., 'big.csv');
return response()->streamJson(['users' => User::cursor()]); // incremental JSON
return response()->eventStream(function () { yield $token; }); // SSE (text/event-stream)

// --- Redirects ---
return redirect('/path');
return redirect()->route('users.show', $user);
return to_route('users.show', $user);
return back()->withInput()->withErrors($validator);
return redirect()->away($externalUrl);           // external (validate first!)
return redirect()->intended('/dashboard');
```

---

## 🧪 Mini Exercises

1. **Type-safe filters.** Write a `GET /products` controller action that reads `q` (search term, default empty string via `string()`), `page` (default 1 via `integer()`), and `in_stock` (a checkbox via `boolean()`), and returns the three values as a JSON object. Verify that `?in_stock=on` yields `true` and `?in_stock=false` yields `false`.

2. **Sticky form.** Build a two-field form (`name`, `email`), validate that both are required and `email` is a valid email, and on failure redirect back so the fields repopulate via `old()` and the errors show via `@error`. Do it once with a manual redirect (`back()->withInput()`) and once relying on `$request->validate()` doing the flashing for you — note the difference.

3. **Avatar upload.** Accept an `avatar` image upload, reject it unless `hasFile()` and `isValid()` pass, store it on the `public` disk under `avatars/` with a hashed name, and return the public URL as JSON. Bonus: return `422` with a JSON error when no valid file is present.

4. **CSV streamer.** Create a route that streams 50,000 rows of fake data as a downloadable `report.csv` using `response()->streamDownload()`, flushing as you go. Confirm (by reasoning about the code) that the full file is never held in memory at once.

5. **Content negotiation + macro.** Register a `response()->apiSuccess($data, $status = 200)` macro that returns `{"data": ..., "meta": {"ok": true}}`. Then write an action that returns this macro for JSON clients (`wantsJson()`) and a Blade view for browsers.

6. **Generator stream.** Rewrite exercise 4 so the `stream()` callback *returns a Generator* (`yield`s each row) instead of `echo`-ing and manually calling `ob_flush()/flush()`. Note that Laravel now handles the flushing and Nginx-buffering for you, and add the `X-Accel-Buffering: no` header explicitly to confirm you understand what it does.

7. **SSE ticker.** Build a `GET /clock` route that uses `response()->eventStream()` to `yield` the current time once per second for 10 seconds, then consume it from the browser with a raw `EventSource`, closing the connection when the final `</stream>` message arrives. (Bonus: name the event with `StreamedEvent(event: 'tick', ...)`.)

8. **Locate the wiring.** In a fresh Laravel 12 app, find where you would (a) trust all proxies, (b) exclude a `analytics_id` cookie from encryption, and (c) prepend a custom middleware to the `web` group. Confirm all three happen in `bootstrap/app.php` and that no `app/Http/Kernel.php` exists.
