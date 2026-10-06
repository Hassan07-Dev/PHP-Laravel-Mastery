# File Storage in Laravel 12

Laravel's file storage system gives you **one consistent API** for reading and writing files no matter where they actually live — your server's local disk, a shared `public/` folder, Amazon S3, DigitalOcean Spaces, an FTP server, or an in-memory fake during tests. You write `Storage::put(...)` once and swap the destination by changing a single config value. This module teaches you that abstraction from the ground up.

## What you'll learn

- Why Laravel wraps filesystems behind an abstraction (Flysystem) and what a **disk** is.
- How to configure `local`, `public`, and `s3` disks in `config/filesystems.php`.
- The full `Storage` facade API: `put`, `get`, `exists`, `delete`, `copy`, `move`, `size`, `lastModified`, `append`/`prepend`, `makeDirectory`, and listing helpers.
- The `public` disk, the `storage:link` symlink, and how `url()` generates public URLs.
- Storing uploaded files (`store`, `storeAs`, `storePublicly`) and generating safe filenames.
- Downloading and **streaming** files, including large files, without exhausting memory.
- File **visibility** (public/private) and **temporary signed URLs** on S3.
- Testing file code with `Storage::fake()` and `UploadedFile::fake()`, plus mime/size validation and an image-processing note.

---

## 1. Why an abstraction at all? (The "why" before the "how")

Imagine you write code that saves user avatars by calling PHP's built-in `file_put_contents('/var/www/uploads/avatar.png', $bytes)`. It works on your laptop. Then:

- Production runs on **multiple servers** behind a load balancer — a file written on server A doesn't exist on server B.
- Your boss says "move uploads to **S3**" — now you must rewrite every `file_put_contents`, `fopen`, `unlink` call across the codebase.
- Your **tests** actually touch the disk, leaving junk files and running slowly.

Laravel solves this with a **filesystem abstraction**: a single, stable API (`Storage::put`, `Storage::get`, …) that delegates to a configurable backend. Change the backend in config; your application code never changes.

### What is Flysystem?

Under the hood, Laravel's `Storage` does not talk to disks directly. It wraps **[Flysystem](https://flysystem.thephpleague.com/)**, a popular PHP package by The PHP League that provides driver implementations (adapters) for local disks, S3, FTP, SFTP, and more. Laravel adds a friendly facade, config integration, URL generation, uploaded-file helpers, and testing fakes on top.

> **Jargon:** An **adapter** (or **driver**) is the concrete code that knows how to talk to one specific storage backend (e.g. the AWS S3 adapter). A **disk** is *your* named, configured instance of an adapter.

```
Your code  ──►  Storage facade  ──►  Flysystem  ──►  Adapter  ──►  Real backend
Storage::put()                       (League)       (local/s3)     (disk, S3 bucket…)
```

---

## 2. Disks and `config/filesystems.php`

A **disk** is a named storage location plus its driver and options. They're defined in `config/filesystems.php`. Here is a representative Laravel 12 config:

```php
<?php
// config/filesystems.php

return [

    'default' => env('FILESYSTEM_DISK', 'local'),

    'disks' => [

        'local' => [
            'driver' => 'local',
            'root'   => storage_path('app/private'), // Laravel 11+ default
            'serve'  => true,
            'throw'  => false,                       // throw exceptions on failure?
            'report' => false,
        ],

        'public' => [
            'driver'     => 'local',
            'root'       => storage_path('app/public'),
            'url'        => env('APP_URL') . '/storage',
            'visibility' => 'public',
            'throw'      => false,
        ],

        's3' => [
            'driver'                  => 's3',
            'key'                     => env('AWS_ACCESS_KEY_ID'),
            'secret'                  => env('AWS_SECRET_ACCESS_KEY'),
            'region'                  => env('AWS_DEFAULT_REGION'),
            'bucket'                  => env('AWS_BUCKET'),
            'url'                     => env('AWS_URL'),
            'endpoint'                => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
            'throw'                   => false,
        ],

    ],

    // Where storage:link creates symlinks (target => link)
    'links' => [
        public_path('storage') => storage_path('app/public'),
    ],

];
```

Key points:

- **`default`** — the disk used when you call `Storage::put(...)` without naming one. Driven by `FILESYSTEM_DISK` in `.env`.
- **`local`** driver — files on the server's own disk, rooted at `root`.
- **`public`** disk — also `local`, but rooted at `storage/app/public` and meant to be web-accessible via a symlink (Section 5).
- **`s3`** driver — Amazon S3 (or any S3-compatible service like MinIO/DigitalOcean Spaces via `endpoint`).
- **`throw`** — if `true`, failed operations raise an exception instead of returning `false`. Default is `false` (silent). Set `true` in production so failures aren't swallowed.

> **Laravel 10 vs 11/12 difference:** In Laravel 10 the `local` disk rooted at `storage/app`, and there was no `storage/app/private` convention. Laravel 11 introduced the `app/private` root for the `local` disk and `app/public` for the `public` disk. If you migrate an older app, double-check your `root` paths.

The matching `.env`:

```env
FILESYSTEM_DISK=local

AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=my-app-bucket
AWS_USE_PATH_STYLE_ENDPOINT=false
```

### Selecting a disk in code

```php
use Illuminate\Support\Facades\Storage;

// Uses the default disk
Storage::put('notes.txt', 'hello');

// Explicitly pick a disk
Storage::disk('s3')->put('notes.txt', 'hello');
Storage::disk('public')->put('avatars/1.png', $bytes);
```

You can also define a disk on the fly without touching config (useful in packages or one-off tasks):

```php
$disk = Storage::build([
    'driver' => 'local',
    'root'   => storage_path('app/exports'),
]);

$disk->put('report.csv', $csv);
```

---

## 3. The `Storage` facade: core read/write API

All methods below operate on the default disk unless you prefix `disk('name')`.

### Writing

```php
use Illuminate\Support\Facades\Storage;

// Write a string (creates parent directories automatically)
Storage::put('docs/welcome.txt', 'Welcome aboard!');

// Append / prepend lines (great for logs)
Storage::append('logs/app.log', 'Line added at the end');
Storage::prepend('logs/app.log', 'Line added at the top');

// Make a directory
Storage::makeDirectory('reports/2026');
```

`put` returns `true` on success, `false` on failure (unless `throw` is enabled).

### Reading

```php
$contents = Storage::get('docs/welcome.txt');
// Output: "Welcome aboard!"  (string), or null if missing

$json = Storage::json('config/settings.json'); // decodes JSON to array, Laravel 9+
// Output: ['theme' => 'dark', ...]
```

### Existence checks

```php
Storage::exists('docs/welcome.txt');  // true
Storage::missing('nope.txt');         // true  (inverse of exists)
Storage::fileExists('docs/welcome.txt');     // true — file specifically
Storage::directoryExists('reports/2026');    // true — directory specifically
```

### Copy, move, delete

```php
Storage::copy('docs/welcome.txt', 'backups/welcome.txt');
Storage::move('docs/welcome.txt', 'archive/welcome.txt'); // rename or relocate

Storage::delete('archive/welcome.txt');
Storage::delete(['a.txt', 'b.txt']);          // delete several at once
Storage::deleteDirectory('reports/2026');     // recursive
```

### Metadata

```php
Storage::size('archive/welcome.txt');         // int bytes, e.g. 15
Storage::lastModified('archive/welcome.txt');  // UNIX timestamp, e.g. 1750000000
Storage::mimeType('avatars/1.png');            // "image/png"
Storage::path('archive/welcome.txt');          // absolute path (local driver only)
Storage::checksum('a.txt', ['checksum_algo' => 'sha256']); // hash, Laravel 10+
```

> `Storage::path()` returns a real filesystem path and only makes sense for the `local`/`public` drivers. On `s3` there is no local path, so don't rely on it.

### Listing files and directories

```php
// Files in a directory (not recursive)
Storage::files('reports');
// Output: ['reports/jan.csv', 'reports/feb.csv']

// All files, recursively
Storage::allFiles('reports');
// Output: ['reports/jan.csv', 'reports/2026/q1.csv', ...]

// Sub-directories (not recursive)
Storage::directories('reports');
// Output: ['reports/2026']

// All directories, recursively
Storage::allDirectories('reports');
```

---

## 4. Storing uploaded files

When a user uploads a file, Laravel hands you an `Illuminate\Http\UploadedFile` object (a subclass of Symfony's `UploadedFile`). It carries convenient `store*` methods that move the temp upload into your configured disk.

### A complete controller example

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Storage;

class AvatarController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        // Always validate uploads (Section 11)
        $validated = $request->validate([
            'avatar' => ['required', 'image', 'mimes:jpg,png,webp', 'max:2048'], // KB
        ]);

        // store() picks an auto-generated unique name; arg 1 is the directory
        $path = $request->file('avatar')->store('avatars');
        // $path === "avatars/8f2c…hashed.png"

        // Persist the returned path on the model — NOT the full URL
        $request->user()->update(['avatar_path' => $path]);

        return back()->with('status', 'Avatar uploaded!');
    }
}
```

`store('avatars')` generates a unique filename via `hashName()` (a 40-character **random** string — not a hash of the file's contents — plus an extension derived from the file's MIME type) and returns the **relative path** you should save in your database. Storing the path (not the URL) keeps you portable across disks and domains.

### Choosing a disk and a custom name

```php
$file = $request->file('avatar');

// 1) store() on a specific disk (2nd arg is the disk name OR an options array)
$path = $file->store('avatars', 's3');

// 2) storeAs() — you control the filename
$path = $file->storeAs('avatars', 'user-'.$request->user()->id.'.png');
$path = $file->storeAs('avatars', 'user-1.png', 's3'); // + disk

// 3) storePublicly() / storePubliclyAs() — force public visibility (matters on S3)
$path = $file->storePublicly('avatars', 's3');
$path = $file->storePubliclyAs('avatars', 'user-1.png', 's3');

// Calling on the Storage facade instead (equivalent), Laravel 9+:
$path = Storage::disk('s3')->putFile('avatars', $file);
$path = Storage::disk('s3')->putFileAs('avatars', $file, 'user-1.png');
```

### Generating filenames yourself

Auto-hashed names avoid collisions and hide the original name. When you need a deterministic or human-friendly name, build it safely:

```php
use Illuminate\Support\Str;

$file = $request->file('document');

// Preserve the extension but randomize the stem
$name = Str::uuid() . '.' . $file->getClientOriginalExtension();
$path = $file->storeAs('docs', $name);

// Or slugify the original name (never trust the raw client name as a path!)
$original = pathinfo($file->getClientOriginalName(), PATHINFO_FILENAME);
$name = Str::slug($original) . '-' . now()->timestamp . '.' . $file->extension();
$path = $file->storeAs('docs', $name);
```

> **Security:** `getClientOriginalName()` and `getClientOriginalExtension()` come from the *client* and can contain `../`, null bytes, or a fake extension. Use `$file->extension()` (which guesses from MIME) when you can, and always sanitize before using any client-provided string as a filename or path.

---

## 5. The `public` disk, `storage:link`, and `url()`

Files stored on the `public` disk live in `storage/app/public`, which is **not** inside the web-accessible `public/` directory. So how do browsers reach them?

You create a **symbolic link** from `public/storage` to `storage/app/public`:

```bash
php artisan storage:link
# Output: INFO  The [public/storage] link has been connected to [storage/app/public].
```

Now a file at `storage/app/public/avatars/1.png` is served at `http://your-app.test/storage/avatars/1.png`.

```php
// Generate the public URL from a stored path
$url = Storage::disk('public')->url('avatars/1.png');
// Output: "http://your-app.test/storage/avatars/1.png"

// or the global helper
$url = asset('storage/avatars/1.png');
```

In Blade:

```blade
<img src="{{ Storage::disk('public')->url($user->avatar_path) }}" alt="Avatar">

{{-- equivalently --}}
<img src="{{ Storage::url($user->avatar_path) }}" alt="Avatar"> {{-- if 'public' is default --}}
```

> **Gotcha:** `url()` works only on disks that have a `url` configured (the `public` disk and `s3` do; the bare `local` disk does **not**). Calling `url()` on a disk with no `url` throws a `RuntimeException`.

> **Deployment note:** The symlink is not committed to git (it's inside `public/`). Run `php artisan storage:link` as part of every deploy, or your `/storage/...` URLs will 404. On Windows, you may need to run the terminal as Administrator to create symlinks.

---

## 6. Downloading and streaming

### Force a download

```php
use Illuminate\Support\Facades\Storage;

// Browser downloads with the given filename and headers
return Storage::download('reports/jan.csv', 'january-report.csv');

// Add custom headers
return Storage::download('reports/jan.csv', 'january-report.csv', [
    'Content-Type' => 'text/csv',
]);
```

`Storage::download()` returns a `StreamedResponse` with `Content-Disposition: attachment`, so the file streams to the client rather than loading entirely into memory.

For local files specifically, the response helper also works:

```php
// Only for files with a real local path
return response()->download(Storage::path('reports/jan.csv'), 'january-report.csv');

// Inline (display in browser instead of downloading)
return response()->file(Storage::path('invoices/123.pdf'));
```

### Stream a remote/large file through your app

```php
// Streams from any disk (including S3) to the client
return Storage::response('videos/intro.mp4'); // inline
return Storage::download('videos/intro.mp4'); // attachment
```

### Reading and writing streams (large files, low memory)

Loading a 2 GB file with `Storage::get()` would try to hold it all in memory. Use **streams** instead — a stream is a resource you read/write in chunks.

```php
// Read as a stream (PHP resource handle)
$stream = Storage::readStream('videos/intro.mp4');
// ... fread($stream, 8192) in a loop ...
fclose($stream);

// Write from a stream — copy a large file from local to S3 without buffering it all
$source = Storage::disk('local')->readStream('big.zip');
Storage::disk('s3')->writeStream('archives/big.zip', $source);
if (is_resource($source)) {
    fclose($source);
}

// putFile() with a large uploaded file also streams under the hood
Storage::disk('s3')->putFile('uploads', $request->file('big'));
```

> **Why streaming matters:** PHP processes have a `memory_limit`. `Storage::get()` returns the entire file as a string in memory; for big files that triggers an out-of-memory fatal error. `readStream`/`writeStream` move data in small chunks, so memory use stays flat regardless of file size.

---

## 7. Visibility and temporary signed URLs

### Visibility (public vs private)

**Visibility** is Flysystem's portable abstraction over file permissions. On the `local` driver it maps to Unix file modes; on `s3` it maps to object ACLs (`public-read` vs `private`).

```php
// Set visibility when writing
Storage::put('reports/secret.pdf', $bytes, 'private');
Storage::put('banners/promo.png', $bytes, 'public');

// Read / change later
Storage::getVisibility('banners/promo.png');           // "public"
Storage::setVisibility('reports/secret.pdf', 'private');
```

- `public` files are world-readable (and on S3 reachable at a stable public URL).
- `private` files are not directly reachable; you serve them through your app (auth-checked) or via a **temporary signed URL**.

### Temporary signed URLs (S3)

A **temporary URL** is a time-limited, cryptographically signed link that grants access to a *private* object without making it public. Perfect for "download your invoice" links that expire.

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::disk('s3')->temporaryUrl(
    'invoices/2026/123.pdf',
    now()->addMinutes(10)            // expires in 10 minutes
);
// Output: "https://my-bucket.s3.amazonaws.com/invoices/2026/123.pdf?X-Amz-Expires=600&X-Amz-Signature=..."

// Override response headers in the signed URL (force a download name, set content type)
$url = Storage::disk('s3')->temporaryUrl(
    'invoices/2026/123.pdf',
    now()->addMinutes(10),
    [
        'ResponseContentType'        => 'application/pdf',
        'ResponseContentDisposition' => 'attachment; filename="invoice-123.pdf"',
    ]
);
```

For **uploads** straight to S3 from the browser (bypassing your server), use a temporary upload URL. `temporaryUploadUrl()` returns an associative array you destructure into the `url` and the `headers` to send with the upload request:

```php
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

['url' => $url, 'headers' => $headers] = Storage::disk('s3')
    ->temporaryUploadUrl('uploads/'.Str::uuid().'.jpg', now()->addMinutes(5));
// The frontend then PUTs the file to $url with $headers.
```

> **Local driver note:** `temporaryUrl()` originated as an S3 feature, but **since Laravel 11 the `local` driver supports temporary (signed) URLs out of the box** — no config change needed on a fresh Laravel 11/12 app. The `'serve' => true` option only matters for apps created *before* that feature shipped (it must be enabled to turn on the route that serves local temporary URLs); the current skeleton already sets `'serve' => true` on the `local` disk. `temporaryUploadUrl()` is likewise supported by both the `s3` and `local` drivers. Calling either method on a disk whose driver genuinely doesn't support it (e.g. a custom adapter) throws a `RuntimeException` unless you register a generator with `Storage::disk(...)->buildTemporaryUrlsUsing(...)`.

---

## 8. An image-handling note: `intervention/image`

Laravel's `Storage` moves bytes around; it does **not** resize, crop, or re-encode images. For that, the de-facto package is **Intervention Image** (v3).

```bash
composer require intervention/image
```

```php
use Intervention\Image\ImageManager;
use Intervention\Image\Drivers\Gd\Driver; // or Imagick\Driver
use Illuminate\Support\Facades\Storage;

public function store(Request $request)
{
    $request->validate(['photo' => ['required', 'image', 'max:5120']]);

    $manager = new ImageManager(new Driver());

    // Read the uploaded file, resize to a 600px-wide thumbnail, re-encode as WebP
    $image = $manager->read($request->file('photo')->getRealPath());
    $image->scaleDown(width: 600);
    $encoded = $image->toWebp(quality: 80); // EncodedImage (binary)

    // Persist the processed bytes via Storage
    $path = 'photos/' . \Illuminate\Support\Str::uuid() . '.webp';
    Storage::disk('public')->put($path, (string) $encoded);

    return ['path' => $path, 'url' => Storage::disk('public')->url($path)];
}
```

The key idea: **process in memory with Intervention, then hand the resulting bytes to `Storage::put`.** The two tools are complementary — Intervention transforms, Storage persists.

> Intervention v3 requires the `gd` or `imagick` PHP extension. v3's API (`ImageManager`, `read()`, `scaleDown()`, `toWebp()`) differs from the older v2 (`Image::make(...)`). Confirm the version before copying snippets.

---

## 9. Validating uploads (mime / size tie-in)

Never trust an upload. Validation rules pair naturally with file storage:

```php
$request->validate([
    // 'image' = must be jpg, jpeg, png, bmp, gif, or webp (verified by content).
    // NOTE: SVG is NOT allowed by default (XSS risk) — see the warning below.
    'avatar' => ['required', 'image', 'max:2048'], // max in KILOBYTES

    // Restrict by extension AND verified MIME type
    'doc'    => ['required', 'file', 'mimes:pdf,docx', 'max:10240'],

    // Match by raw MIME type
    'video'  => ['required', 'mimetypes:video/mp4,video/quicktime', 'max:51200'],

    // Image dimensions (ratio is width/height)
    'banner' => ['required', 'image', 'dimensions:min_width=1200,ratio=16/9'],
]);
```

- `max:2048` means **2048 KB ≈ 2 MB**. Units are kilobytes, a classic gotcha.
- `mimes:` and `mimetypes:` both read the file's **contents** to guess the MIME type, so neither trusts the (spoofable) client extension alone. The difference: `mimes:` maps a list of *extensions* to their expected MIME types, while `mimetypes:` matches against a list of raw MIME strings (and supports wildcards like `image/*`). Note that `mimes:` does **not** check that the guessed type agrees with the name the user gave the file — a valid PNG named `photo.txt` still passes `mimes:png`.
- `image` is a convenience rule for common web image types.

> **⚠️ Security — SVG and the `image` rule:** Since Laravel 11, the `image` rule **rejects SVG files by default** because SVGs are XML and can carry embedded `<script>` (stored XSS). If you genuinely must accept SVGs, opt in explicitly with `image:allow_svg`, and serve them with a restrictive `Content-Security-Policy` / `Content-Disposition: attachment`, or sanitize them server-side. Never blindly allow `mimes:svg` for user uploads that are later served inline.

> Also enforce a server-side limit: PHP's `upload_max_filesize` and `post_max_size` (in `php.ini`) cap what ever reaches Laravel. If a 50 MB upload is rejected before validation runs, check those directives.

---

## 10. Testing: `Storage::fake()` and `UploadedFile::fake()`

This is where the abstraction pays off massively. `Storage::fake('disk')` swaps the named disk for a **temporary, in-memory-style local disk** for the duration of the test, so assertions are fast and leave no junk behind.

```php
<?php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

it('stores an uploaded avatar on the public disk', function () {
    Storage::fake('public');

    // Generate a fake image without needing a real file on disk
    $file = UploadedFile::fake()->image('avatar.jpg', width: 200, height: 200);

    $response = $this->actingAs($user = User::factory()->create())
        ->post('/avatar', ['avatar' => $file]);

    $response->assertSessionHas('status');

    // Assert the file landed where we expect
    $path = 'avatars/' . $file->hashName();
    Storage::disk('public')->assertExists($path);

    // Other assertions:
    Storage::disk('public')->assertMissing('avatars/nope.png');
    Storage::disk('public')->assertCount('avatars', 1);
    Storage::disk('public')->assertDirectoryEmpty('temp');
});
```

`UploadedFile::fake()` helpers:

```php
// A fake JPEG of given dimensions (real image bytes via GD)
UploadedFile::fake()->image('photo.jpg', 640, 480);

// A fake file of an arbitrary type and size (in kilobytes)
UploadedFile::fake()->create('report.pdf', 1024, 'application/pdf'); // 1 MB pdf
```

> **PHPUnit vs Pest:** The example above uses Pest's `it(...)`. In classic PHPUnit it's `public function test_stores_avatar(): void { ... }` inside a `TestCase`. The `Storage::fake` / `assertExists` calls are identical either way.

> **Gotcha:** `Storage::fake('public')` only fakes the `public` disk. If your code writes to `s3`, you must `Storage::fake('s3')`. Faking the wrong disk silently writes to the real one (or fails). Fake every disk your code under test touches.

---

## 11. End-to-end mini project: secure invoice download

Tying it together — store invoices privately on S3, then serve them via short-lived signed URLs only to the owner.

```php
// Storing (e.g. in a job after generating the PDF)
$path = "invoices/{$user->id}/" . Str::uuid() . '.pdf';
Storage::disk('s3')->put($path, $pdfBytes, 'private');
$invoice->update(['storage_path' => $path]);

// Controller: hand the user a 5-minute signed URL — only if they own it
public function download(Invoice $invoice)
{
    $this->authorize('view', $invoice); // policy check

    return redirect(
        Storage::disk('s3')->temporaryUrl($invoice->storage_path, now()->addMinutes(5))
    );
}
```

The file is never public; the URL self-destructs after 5 minutes; access is gated by a policy. Storage, visibility, signed URLs, and authorization working together.

---

## ⚠️ Common Mistakes & Gotchas

1. **Storing the full URL in the database instead of the path.**
   If you save `https://app.test/storage/avatars/1.png` and later move to S3 or change domains, every record is broken.
   **Fix:** Store only the relative path returned by `store()`/`storeAs()` (e.g. `avatars/1.png`). Generate the URL at read time with `Storage::url($path)` or `temporaryUrl()`.

2. **Forgetting `php artisan storage:link` (404s on every uploaded image).**
   Files saved to the `public` disk are in `storage/app/public`, unreachable by the browser until the symlink exists. It's also missing on fresh deploys because it lives in `.gitignore`d `public/`.
   **Fix:** Run `php artisan storage:link` after setup and in your deploy script.

3. **Confusing `max:2048` units in validation.**
   `max:2048` is **2048 kilobytes (~2 MB)**, not bytes and not megabytes. People expect bytes and accidentally allow tiny or huge files.
   **Fix:** Remember the unit is KB. For 10 MB use `max:10240`. Also raise `upload_max_filesize`/`post_max_size` in `php.ini` to match.

4. **Calling `url()` or `path()` on a disk that doesn't support it.**
   `Storage::url()` on the bare `local` disk (no `url` configured) throws `RuntimeException`; `Storage::path()` on `s3` is meaningless since there's no local path.
   **Fix:** Use the `public` disk (or set a `url`) for browser-served files; use `temporaryUrl()` for private S3 objects; never assume a local path exists for remote disks.

5. **Trusting `getClientOriginalName()` as a path.**
   The client controls this string; it can contain `../`, slashes, or a misleading extension, enabling path traversal or content spoofing.
   **Fix:** Generate names server-side (`Str::uuid()`, `hashName()`), or `Str::slug()` the stem; derive the extension with `$file->extension()` (content-based) and always validate MIME.

6. **Loading huge files with `Storage::get()` and hitting `memory_limit`.**
   `get()` reads the whole file into a PHP string.
   **Fix:** Use `Storage::readStream()` / `writeStream()` for large files and `Storage::download()`/`response()` which stream to the client.

7. **Forgetting to fake the right disk in tests, leaving real files behind.**
   `Storage::fake('public')` doesn't fake `s3`.
   **Fix:** Fake every disk the code under test writes to.

8. **Assuming the `image` rule accepts SVG (and serving uploaded SVGs inline).**
   Since Laravel 11 the `image` rule **rejects SVG by default**, so an SVG upload will surprisingly fail validation. Worse, if you "fix" that by switching to `mimes:svg`, you can now store attacker-controlled XML that executes JavaScript when served inline (stored XSS).
   **Fix:** Keep the default (reject SVG). Only opt in with `image:allow_svg` when you must, and then serve such files with `Content-Disposition: attachment` (or a strict CSP) and/or sanitize the markup.

---

## ✅ Best Practices

- **Persist paths, build URLs lazily.** The database holds `avatars/1.png`; the view calls `Storage::url(...)`.
- **Default to `private` visibility.** Make files public deliberately, not by accident. Serve sensitive files through authorization + `temporaryUrl()`.
- **Set `'throw' => true`** on production disks so silent failures surface as exceptions you can log.
- **Stream large files** (`readStream`/`writeStream`, `download`, `response`) instead of buffering them in memory.
- **Validate every upload** with `image`/`mimes`/`mimetypes` + `max`, and align PHP's `upload_max_filesize`/`post_max_size`.
- **Keep SVG uploads off by default.** The `image` rule already rejects SVG; only allow it (`image:allow_svg`) when required, and then serve as an attachment or sanitize to avoid stored XSS.
- **Generate filenames server-side** (UUID/hash) to prevent collisions and path-traversal; never echo a user-controlled name into a path.
- **Use `Storage::fake()` + `UploadedFile::fake()`** in tests — never touch the real filesystem.
- **Keep environment-specific destinations in config/env** (`FILESYSTEM_DISK`) so the same code runs locally on `local` and in prod on `s3`.
- **Run `storage:link` in deploys** and treat the symlink as part of provisioning.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is the Laravel filesystem abstraction, and what is it built on?**
A unified API (the `Storage` facade) for file operations that delegates to **Flysystem** (The PHP League). Flysystem provides adapters for local, S3, FTP, SFTP, etc. You configure named *disks* in `config/filesystems.php` and swap backends without changing application code.

**Q2. How does `Storage` actually resolve and talk to a disk under the hood?**
The `Storage` facade resolves the `FilesystemManager` from the container. `Storage::disk('s3')` asks the manager to build (and cache) a `Filesystem` instance for that disk, reading its config and instantiating the matching Flysystem adapter (e.g. `AwsS3V3Adapter`). Calls like `put()` flow through Laravel's `FilesystemAdapter` wrapper into Flysystem, which invokes the adapter's backend-specific code. The default disk comes from `filesystems.default` / `FILESYSTEM_DISK`.

**Q3. Difference between the `local` disk and the `public` disk?**
Both use the `local` driver. The `local` disk roots at `storage/app/private` and is **not** web-accessible — for private files. The `public` disk roots at `storage/app/public`, defaults to `public` visibility, has a configured `url`, and is exposed to the web via the `public/storage` symlink created by `storage:link`.

**Q4. Why `storage:link`, and why does it break after deploys?**
Web servers serve from `public/`, but uploads live in `storage/`. The symlink bridges them so `public/storage/...` resolves to `storage/app/public/...`. It lives inside `public/` which is git-ignored, so it isn't checked out on deploy — you must re-run `php artisan storage:link`.

**Q5. How do you serve a private file securely?**
Keep it `private`. Either stream it through a controller after an authorization/policy check (`Storage::download()`), or hand the client a **temporary signed URL** (`Storage::disk('s3')->temporaryUrl($path, now()->addMinutes(5))`) that expires and requires no public ACL.

**Q6. `store()` vs `storeAs()` vs `storePublicly()`?**
`store($dir, $disk)` saves with an auto-generated hashed name and returns the path. `storeAs($dir, $name, $disk)` lets you set the filename. `storePublicly()`/`storePubliclyAs()` are the same but force `public` visibility (relevant on S3 where objects are private by default).

**Q7. How do you handle very large files without exhausting memory?**
Use streams: `Storage::readStream()` / `writeStream()` move data in chunks, and `Storage::download()` / `Storage::response()` stream the file to the client. Avoid `Storage::get()`, which loads the entire file into a string and can blow the `memory_limit`.

**Q8. How do you unit-test code that writes files?**
`Storage::fake('disk')` replaces the disk with a temporary sandbox; `UploadedFile::fake()->image(...)`/`->create(...)` builds fake uploads. Then assert with `assertExists`, `assertMissing`, `assertCount`. No real files are created, tests are fast and isolated.

**Q9. What does file "visibility" mean, and how does it map across drivers?**
A portable `public`/`private` concept. On `local` it maps to Unix permissions (file mode); on `s3` it maps to object ACLs (`public-read` vs `private`). Set it via the 3rd arg to `put()` or `setVisibility()`.

**Q10. Why store the path, not the URL, in the DB?**
Portability and security. The same path works whether the disk is `local`, `public`, or `s3`, across domains and environments; and you can choose at read time to return a public URL or a short-lived signed URL.

---

## 📋 Quick Reference / Cheat Sheet

```php
use Illuminate\Support\Facades\Storage;

// --- Disks ---
Storage::disk('s3')->put(...);              // pick a disk
Storage::build(['driver' => 'local', ...]); // on-the-fly disk

// --- Write / read ---
Storage::put($path, $contents, 'private');  // visibility arg optional
Storage::get($path);                        // string | null
Storage::json($path);                       // decoded array
Storage::append($path, $line);
Storage::prepend($path, $line);

// --- Existence / metadata ---
Storage::exists($path);  Storage::missing($path);
Storage::fileExists($path);  Storage::directoryExists($dir);
Storage::size($path);                       // bytes (int)
Storage::lastModified($path);               // unix ts
Storage::mimeType($path);
Storage::path($path);                       // local only

// --- Copy / move / delete ---
Storage::copy($from, $to);  Storage::move($from, $to);
Storage::delete($path);  Storage::delete([$a, $b]);
Storage::makeDirectory($dir);  Storage::deleteDirectory($dir);

// --- Listing ---
Storage::files($dir);  Storage::allFiles($dir);
Storage::directories($dir);  Storage::allDirectories($dir);

// --- Uploads (UploadedFile) ---
$request->file('x')->store('dir', 'disk');
$request->file('x')->storeAs('dir', 'name.ext', 'disk');
$request->file('x')->storePublicly('dir', 'disk');
$file->hashName();  $file->extension();  $file->getClientOriginalName();

// --- URLs / download / stream ---
Storage::url($path);                        // public disks / s3
Storage::disk('s3')->temporaryUrl($path, now()->addMinutes(10)); // also: local (L11+)
['url' => $u, 'headers' => $h] = Storage::disk('s3')->temporaryUploadUrl($path, now()->addMinutes(5));
Storage::download($path, 'name.ext', $headers);
Storage::response($path);                   // inline stream
Storage::readStream($path);  Storage::writeStream($path, $resource);

// --- Visibility ---
Storage::getVisibility($path);  Storage::setVisibility($path, 'private');

// --- Testing ---
Storage::fake('public');
UploadedFile::fake()->image('a.jpg', 200, 200);
UploadedFile::fake()->create('a.pdf', 1024, 'application/pdf');
Storage::disk('public')->assertExists($path);
Storage::disk('public')->assertMissing($path);
Storage::disk('public')->assertCount($dir, 1);
```

```bash
php artisan storage:link   # symlink public/storage -> storage/app/public
```

```env
FILESYSTEM_DISK=local
AWS_BUCKET=my-bucket
AWS_DEFAULT_REGION=us-east-1
```

---

## 🧪 Mini Exercises

1. **Avatar pipeline.** Build an endpoint that validates an uploaded image (`image`, max 2 MB), stores it on the `public` disk under `avatars/` with a UUID filename, saves the path on the `User` model, and returns the public URL. Then write a test using `Storage::fake('public')` and `UploadedFile::fake()->image(...)` that asserts the file exists and the user's `avatar_path` is set.

2. **Private S3 documents.** Configure an `s3` disk, store an uploaded PDF privately, and expose a controller route that returns a 10-minute `temporaryUrl()` — but only after an authorization policy check. Verify (by reasoning) that an unauthorized user cannot obtain the URL.

3. **Large-file copy.** Write an Artisan command that copies a multi-gigabyte file from the `local` disk to `s3` using `readStream`/`writeStream`, ensuring memory usage stays flat. Close the stream handle when done.

4. **Cleanup job.** Write a scheduled task that lists `allFiles('temp')`, deletes any file whose `lastModified()` is older than 24 hours, and removes now-empty directories. Cover it with a test using `Storage::fake()`.

5. **Thumbnail generator.** Using `intervention/image`, accept an upload, generate a 300px-wide WebP thumbnail in memory, store both the original and the thumbnail on the `public` disk, and return both URLs. Validate dimensions with the `dimensions` rule.
