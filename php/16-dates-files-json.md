# Dates, Files & JSON in PHP

This module covers three pillars of everyday backend work: working with **dates and times** correctly (the source of an astonishing number of production bugs), reading and writing **files** safely, and encoding/decoding **JSON** — the lingua franca of APIs. We focus on PHP 8.4 idioms and note where Laravel 12 (and its Carbon library) sits on top.

> **Why these three together?** They are the most common "I/O boundary" operations in a real app: a request comes in with a JSON body containing date strings, you parse and validate the dates, maybe log something to a file or read a CSV upload, then encode a JSON response. Getting each step right — especially time zones and untrusted input — separates a junior from a senior.

---

## What you'll learn

- Why `DateTimeImmutable` is the correct default and how `DateTime`'s mutability causes bugs
- `DateInterval`, `DateTimeZone`, `diff()`, `format()` tokens, `createFromFormat()`, `strtotime()`, and how to compare dates safely
- How Carbon (Laravel's date library) builds on these primitives
- The PHP filesystem API: low-level `fopen`/`fread`/`fwrite`, high-level `file_get_contents`/`file_put_contents`, line-by-line reads, file locking, and CSV parsing
- Path and existence helpers: `pathinfo`, `dirname`, `basename`, `realpath`, `is_file`, `glob`, `scandir`, plus `mkdir`/`copy`/`rename`/`unlink`
- A working mental model of **streams** (the abstraction behind every file function)
- `json_encode`/`json_decode` with their important flags, error handling with `JSON_THROW_ON_ERROR`, and the `JsonSerializable` interface
- Why `unserialize()` on untrusted input is a critical security hole — and what to use instead

---

## Part 1 — Dates & Times

### The mental model: a moment, a zone, and a format

Three concepts get confused constantly. Keep them separate:

- A **moment** is an instant in time, stored internally as a Unix timestamp (seconds since 1970-01-01 00:00:00 UTC). It has no time zone of its own.
- A **time zone** (`DateTimeZone`) is the lens through which we view that moment as a wall-clock reading (e.g. `2026-06-18 14:00` in `Europe/London`).
- A **format** is how we render that wall-clock reading into a string (`"18/06/2026"`, `"June 18th"`, etc.).

PHP's modern date objects bundle a moment plus a zone. The format is applied at display time.

### `DateTime` vs `DateTimeImmutable` — prefer immutable

Both classes implement the `DateTimeInterface`. The crucial difference: methods like `modify()`, `add()`, and `sub()` **mutate** a `DateTime` object in place, but **return a new object** from `DateTimeImmutable`, leaving the original untouched.

```php
<?php

// Mutable — the classic trap
$start = new DateTime('2026-06-18 09:00');
$end   = $start->modify('+1 hour'); // mutates $start AND returns it
echo $start->format('H:i'), "\n";    // 10:00  <-- $start changed!
echo $end->format('H:i'), "\n";      // 10:00  <-- same object
var_dump($start === $end);           // bool(true) — same instance
```

Output:

```
10:00
10:00
bool(true)
```

The same logic with `DateTimeImmutable` does what you actually expect:

```php
<?php

$start = new DateTimeImmutable('2026-06-18 09:00');
$end   = $start->modify('+1 hour'); // returns a NEW object
echo $start->format('H:i'), "\n";    // 09:00  <-- unchanged
echo $end->format('H:i'), "\n";      // 10:00
var_dump($start === $end);           // bool(false)
```

Output:

```
09:00
10:00
bool(false)
```

**Why immutable is the default choice:** dates are passed around to helper functions, stored in arrays, and captured in closures. With mutable objects, a function that calls `->modify()` on an argument silently changes the caller's value — a "spooky action at a distance" bug. Immutable objects are safe to share. Carbon offers `CarbonImmutable` for the same reason. **Rule of thumb: reach for `DateTimeImmutable` unless you have a specific reason not to.**

> Note: `==` compares the underlying moment (two objects representing the same instant are equal even if different instances), while `===` requires the same instance.

### Constructing dates

```php
<?php

$now   = new DateTimeImmutable();                 // current time, default TZ
$epoch = new DateTimeImmutable('@1750000000');    // from a Unix timestamp (@ prefix)
$rel   = new DateTimeImmutable('+2 weeks');       // relative
$fixed = new DateTimeImmutable('2026-12-25 18:30:00');

echo $epoch->format('Y-m-d H:i:s'), "\n"; // 2025-06-15 15:06:40  (note: '@' forces UTC)
```

> Gotcha: when you construct from `'@timestamp'`, the time zone is forced to **UTC** and any zone argument is ignored. Call `->setTimezone(...)` afterward if you need local wall-clock time.

### `DateTimeZone` and why it matters

A timestamp is unambiguous; a wall-clock reading is not. The same moment is `17:00` in London and `12:00` in New York. Always be explicit about zones when crossing the I/O boundary.

```php
<?php

$utc    = new DateTimeImmutable('2026-06-18 17:00:00', new DateTimeZone('UTC'));
$london = $utc->setTimezone(new DateTimeZone('Europe/London'));
$ny     = $utc->setTimezone(new DateTimeZone('America/New_York'));

echo $utc->format('H:i T'),    "\n"; // 17:00 UTC
echo $london->format('H:i T'), "\n"; // 18:00 BST  (summer = +1)
echo $ny->format('H:i T'),     "\n"; // 13:00 EDT
```

Output:

```
17:00 UTC
18:00 BST
13:00 EDT
```

`setTimezone()` does **not** change the moment — only how it is displayed. London shows 18:00 because of daylight saving time (BST). This is exactly why you **store UTC in the database** and convert to the user's zone only for display.

You can set the process-wide default with `date_default_timezone_set('UTC')` or the `date.timezone` INI setting, but explicit per-object zones are more robust.

### `format()` tokens you must know

`format()` turns a date object into a string. The most common tokens:

| Token | Meaning | Example |
|-------|---------|---------|
| `Y` | 4-digit year | `2026` |
| `y` | 2-digit year | `26` |
| `m` | Month, zero-padded | `06` |
| `n` | Month, no padding | `6` |
| `M` | Short month name | `Jun` |
| `F` | Full month name | `June` |
| `d` | Day, zero-padded | `18` |
| `j` | Day, no padding | `18` |
| `D` | Short day name | `Thu` |
| `l` | Full day name (lowercase L) | `Thursday` |
| `N` | ISO weekday (1=Mon..7=Sun) | `4` |
| `H` | Hour 24h, padded | `14` |
| `G` | Hour 24h, no pad | `14` |
| `h` | Hour 12h, padded | `02` |
| `i` | Minutes | `05` |
| `s` | Seconds | `09` |
| `A` / `a` | AM/PM | `PM` / `pm` |
| `T` | Time zone abbreviation | `UTC` |
| `e` | Time zone identifier | `Europe/London` |
| `P` | UTC offset with colon | `+01:00` |
| `U` | Unix timestamp | `1781791509` |
| `c` | ISO 8601 (`Y-m-d\TH:i:sP`) | `2026-06-18T14:05:09+00:00` |
| `r` | RFC 2822 | `Thu, 18 Jun 2026 14:05:09 +0000` |

```php
<?php

$d = new DateTimeImmutable('2026-06-18 14:05:09', new DateTimeZone('UTC'));
echo $d->format('l, F j, Y \a\t H:i'), "\n"; // Thursday, June 18, 2026 at 14:05
echo $d->format('c'), "\n";                   // 2026-06-18T14:05:09+00:00
```

> To print a **literal letter** that is also a token, escape it with a backslash: `\a\t` prints "at". For full ISO 8601 prefer the constant `DateTimeInterface::ATOM` (identical to `'c'`).

### `createFromFormat` — parsing strings strictly

`new DateTimeImmutable($string)` is forgiving and guesses the format. When you receive a string in a **known** format (e.g. a UK date `18/06/2026`), parse it explicitly so an ambiguous string can't be silently misread.

```php
<?php

// Without createFromFormat, "01/02/2026" is ambiguous (Jan 2 vs Feb 1).
$d = DateTimeImmutable::createFromFormat('d/m/Y', '01/02/2026');
echo $d->format('Y-m-d'), "\n"; // 2026-02-01

// Detect a malformed input:
$bad = DateTimeImmutable::createFromFormat('Y-m-d', '2026-13-99');
var_dump($bad); // bool(false) on hard failure...
print_r(DateTimeImmutable::getLastErrors());
```

> Important gotcha: `createFromFormat` may **return an object** even when the input is wrong, because it "rolls over" out-of-range values unless your format is exact. Always check `DateTimeImmutable::getLastErrors()` (returns `false` in PHP 8.2+ when there are no warnings/errors). A robust validator:

```php
<?php

function parseStrict(string $format, string $input): ?DateTimeImmutable
{
    $d = DateTimeImmutable::createFromFormat($format, $input);
    $errors = DateTimeImmutable::getLastErrors();
    // getLastErrors() returns false (PHP 8.2+) when clean
    if ($d === false || ($errors && ($errors['warning_count'] || $errors['error_count']))) {
        return null;
    }
    // Round-trip check to reject rolled-over values like "2026-02-30"
    return $d->format($format) === $input ? $d : null;
}

var_dump(parseStrict('Y-m-d', '2026-02-30')); // NULL  (Feb 30 rolled to Mar 2)
var_dump(parseStrict('Y-m-d', '2026-02-28') instanceof DateTimeImmutable); // true
```

### `strtotime()`, `time()`, and `date()` — the procedural toolkit

These predate the object API but appear everywhere in legacy code and quick scripts.

- `time()` — current Unix timestamp (an `int`).
- `date(string $format, ?int $timestamp = null)` — format a timestamp using the **default** time zone.
- `strtotime(string $time, ?int $baseTimestamp = null)` — parse an English textual datetime into a timestamp; returns `false` on failure.

```php
<?php

$ts = time();                                  // e.g. 1781791200 (assume default TZ = UTC)
echo date('Y-m-d H:i:s', $ts), "\n";           // 2026-06-18 14:00:00
echo date('Y-m-d', strtotime('next monday')), "\n"; // 2026-06-22
echo date('Y-m-d', strtotime('+3 days', strtotime('2026-06-18'))), "\n"; // 2026-06-21

var_dump(strtotime('not a date'));             // bool(false)
```

> Prefer the object API for anything non-trivial. `strtotime()` is locale/format-fragile and silently returns `false`; an unguarded `false` then flows into `date()` as `0`, printing `1970-01-01`. That "January 1970" bug is a classic.

### `DateInterval` and date arithmetic

A `DateInterval` represents a **duration** (e.g. "2 days, 3 hours"), not a point in time. Construct it from an ISO 8601 duration string (`P` = period, `T` separates the time part).

```php
<?php

$start = new DateTimeImmutable('2026-06-18 09:00', new DateTimeZone('UTC'));

$twoDays   = new DateInterval('P2D');      // 2 days
$ninetyMin = new DateInterval('PT90M');    // 90 minutes
$complex   = new DateInterval('P1Y2M10DT2H30M'); // 1y 2m 10d 2h 30m

echo $start->add($twoDays)->format('Y-m-d H:i'), "\n";   // 2026-06-20 09:00
echo $start->sub($ninetyMin)->format('Y-m-d H:i'), "\n"; // 2026-06-18 07:30
```

ISO duration cheat: `P3W` = 3 weeks, `P1DT12H` = 1 day 12 hours, `PT45S` = 45 seconds.

> DST-aware tip: adding `P1D` adds a **calendar day** (respecting DST, so the wall-clock hour is preserved across a DST switch), whereas adding `PT24H` adds exactly 24 hours. They can differ by an hour on the days the clocks change.

### `diff()` — the difference between two moments

`DateTimeInterface::diff()` returns a `DateInterval` describing the gap, with an `invert` flag (1 if the argument is earlier) and a `days` property holding the **total** day count.

```php
<?php

$a = new DateTimeImmutable('2026-06-18 09:00');
$b = new DateTimeImmutable('2026-09-01 12:30');

$diff = $a->diff($b);
echo $diff->days, " total days\n";                 // 75 total days
echo $diff->format('%m months, %d days, %h hours\n'); // 2 months, 14 days, 3 hours
echo $diff->invert, "\n";                          // 0 (b is after a)
```

> The `%d` in `DateInterval::format()` is the day component **within** the period (after months are counted), while the `->days` property is the absolute total. Don't confuse them. Also: `DateInterval::format()` tokens use `%` and are **not** the same as `DateTime::format()` tokens.

### Comparing dates

Comparison operators work directly on `DateTimeInterface` objects and compare the underlying moment:

```php
<?php

$a = new DateTimeImmutable('2026-06-18 09:00', new DateTimeZone('UTC'));
$b = new DateTimeImmutable('2026-06-18 10:00', new DateTimeZone('America/New_York'));
// b is 10:00 EDT = 14:00 UTC, so b is later
var_dump($a < $b);  // bool(true)
var_dump($a == $b); // bool(false)

// To sort an array of dates:
$dates = [
    new DateTimeImmutable('2026-03-01'),
    new DateTimeImmutable('2026-01-15'),
    new DateTimeImmutable('2026-02-20'),
];
usort($dates, fn($x, $y) => $x <=> $y); // spaceship works on DateTime
echo implode(', ', array_map(fn($d) => $d->format('Y-m-d'), $dates)), "\n";
// 2026-01-15, 2026-02-20, 2026-03-01
```

Comparisons correctly account for time zones because they compare the absolute moment, not the wall-clock string.

### A Carbon mention (Laravel)

Laravel ships with **Carbon**, a fluent wrapper around `DateTimeImmutable`/`DateTime`. Eloquent casts date columns to `Illuminate\Support\Carbon` (a Carbon subclass) automatically. Carbon adds expressive helpers and human-friendly diffs.

```php
<?php

use Carbon\Carbon;
use Carbon\CarbonImmutable;

$now = Carbon::now();                       // mutable by default
$later = $now->copy()->addDays(3)->setTime(9, 0); // copy() to avoid mutation

echo Carbon::parse('2026-06-18')->diffForHumans(); // "in X / X ago" style, e.g. "2 days ago"
echo Carbon::parse('2026-06-18')->isWeekend();     // false (Thursday)

// Prefer immutable in app code to dodge the same mutation traps as DateTime:
$ci = CarbonImmutable::now()->addWeek();    // original untouched
```

```php
// In a Laravel model, casting makes the attribute a Carbon instance:
protected function casts(): array
{
    return ['published_at' => 'immutable_datetime']; // Carbon 'datetime' or 'immutable_datetime'
}
```

> Laravel 11/12 favor immutable casts (`immutable_datetime`) for new code. Carbon `diffForHumans()` and methods like `startOfDay()`, `isPast()`, `between()` are interview-favorite conveniences — but underneath it's still the PHP date engine described above.

---

## Part 2 — The Filesystem

### Two API levels: convenience vs control

PHP gives you two layers:

1. **Whole-file convenience functions** — `file_get_contents`, `file_put_contents`, `file()`. Read or write an entire file in one call. Simple, ideal for small files and config.
2. **Stream handle functions** — `fopen`, `fread`, `fwrite`, `fgets`, `fgetcsv`, `fclose`. You hold an open *resource* and read/write incrementally. Essential for **large files** (you don't load gigabytes into memory) and for **locking**.

### Convenience functions

```php
<?php

// Write (overwrites). Returns bytes written or false.
$bytes = file_put_contents('/tmp/note.txt', "line 1\nline 2\n");

// Append instead of overwrite:
file_put_contents('/tmp/note.txt', "line 3\n", FILE_APPEND | LOCK_EX);

// Read the whole file into a string:
$content = file_get_contents('/tmp/note.txt');

// Read into an array of lines (one per element):
$lines = file('/tmp/note.txt', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
print_r($lines);
```

Output:

```
Array
(
    [0] => line 1
    [1] => line 2
    [2] => line 3
)
```

> `LOCK_EX` passed to `file_put_contents` acquires an exclusive lock for that write — use it when multiple processes might write the same file. `file_get_contents` also fetches URLs when `allow_url_fopen` is on, but for HTTP prefer cURL/Guzzle for control over timeouts and headers.

### Low-level handles and file modes

`fopen($path, $mode)` returns a resource. The **mode** controls read/write, truncation, position, and whether the file is created.

| Mode | Read | Write | Pointer start | Truncate? | Create if missing? | Fails if exists? |
|------|------|-------|---------------|-----------|--------------------|------------------|
| `r`  | yes  | no    | beginning     | no        | no                 | no |
| `r+` | yes  | yes   | beginning     | no        | no                 | no |
| `w`  | no   | yes   | beginning     | **yes**   | yes                | no |
| `w+` | yes  | yes   | beginning     | **yes**   | yes                | no |
| `a`  | no   | yes   | **end**       | no        | yes                | no |
| `a+` | yes  | yes   | end (write)   | no        | yes                | no |
| `x`  | no   | yes   | beginning     | no        | yes                | **yes** |
| `x+` | yes  | yes   | beginning     | no        | yes                | **yes** |

Append a `b` for binary safety (`'rb'`, `'wb'`) — recommended on all platforms; harmless on Unix, essential on Windows where text mode mangles `\n`.

```php
<?php

$fh = fopen('/tmp/data.bin', 'wb');
if ($fh === false) {
    throw new RuntimeException('cannot open /tmp/data.bin'); // prefer an exception over `or die()`
}
fwrite($fh, "header\n");
fwrite($fh, pack('N', 42)); // write a 32-bit big-endian int
fclose($fh);

// Read a fixed chunk:
$fh = fopen('/tmp/data.bin', 'rb');
$header = fread($fh, 7);     // read up to 7 bytes -> "header\n"
fclose($fh);
echo $header;
```

### Reading line by line with `fgets`/`feof`

For a multi-gigabyte log you must stream, not slurp. Loop until end-of-file.

```php
<?php

$fh = fopen('/var/log/app.log', 'rb');
if ($fh === false) {
    throw new RuntimeException('Could not open log');
}
try {
    while (!feof($fh)) {
        $line = fgets($fh);        // reads one line (incl. trailing \n), or false at EOF
        if ($line === false) {
            break;                 // guard: feof() is only true AFTER a failed read
        }
        // process $line ...
    }
} finally {
    fclose($fh);                   // always close, even on exception
}
```

> Classic `feof` gotcha: `feof()` returns `true` only **after** a read attempt has already hit EOF, so always check the return of `fgets()`/`fread()` too. Relying on `feof()` alone can cause an extra empty iteration or, on a stuck stream, an infinite loop.

### File locking with `flock`

Locks coordinate concurrent access. They are **advisory** — they only work if every process cooperates by calling `flock`.

```php
<?php

$fh = fopen('/tmp/counter.txt', 'c+'); // 'c+' = read/write, create, DON'T truncate
if (flock($fh, LOCK_EX)) {             // exclusive (write) lock; blocks until acquired
    $current = (int) stream_get_contents($fh);
    $current++;
    rewind($fh);          // move pointer to start
    ftruncate($fh, 0);    // clear old contents
    fwrite($fh, (string) $current);
    fflush($fh);          // flush PHP buffers before releasing
    flock($fh, LOCK_UN);  // release
}
fclose($fh);              // closing also releases the lock
```

`LOCK_SH` is a shared (read) lock; multiple readers can hold it simultaneously but it blocks writers. Add `LOCK_NB` (`LOCK_EX | LOCK_NB`) for a non-blocking attempt that returns `false` immediately if the lock is unavailable.

### Reading CSV with `fgetcsv`

`fgetcsv` parses one CSV record per call, handling quoted fields and embedded commas/newlines correctly — far safer than `explode(',', $line)`.

```php
<?php
// people.csv:
// name,age,city
// "Smith, John",42,"New York"
// Jane,29,London

$fh = fopen('/tmp/people.csv', 'rb');
$header = fgetcsv($fh, escape: "");  // ['name','age','city'] — escape:"" is 8.4-correct
$rows = [];
while (($row = fgetcsv($fh, escape: "")) !== false) {
    $rows[] = array_combine($header, $row); // map columns by header
}
fclose($fh);

print_r($rows[0]);
```

Output:

```
Array
(
    [name] => Smith, John
    [age] => 42
    [city] => New York
)
```

> PHP 8.4 note: the `$escape` parameter of `fgetcsv`/`fputcsv`/`str_getcsv` now defaults to `""` (empty) and passing the old default is **deprecated**, because the legacy backslash-escaping diverged from the RFC 4180 CSV standard and corrupted data. For 8.4-correct, standards-compliant parsing pass `escape: ""` explicitly: `fgetcsv($fh, escape: "")`. Write with `fputcsv($fh, $fields, escape: "")`.

### Existence, type, and metadata checks

```php
<?php

file_exists('/tmp/x');   // true for files, dirs, symlinks
is_file('/tmp/x');       // true only if a regular file
is_dir('/tmp/logs');     // true only if a directory
is_readable('/tmp/x');   // permission check
is_writable('/tmp/x');

filesize('/tmp/x');               // bytes
filemtime('/tmp/x');              // last-modified Unix timestamp
echo date('Y-m-d', filemtime('/tmp/x'));
```

> Stat results are **cached** within a request. After you change a file's size/mtime in the same script and need fresh data, call `clearstatcache()`.

### Listing directories: `scandir` vs `glob`

```php
<?php

// scandir: every entry, including '.' and '..'
$all = scandir('/tmp/logs');
// ['.', '..', 'app.log', 'error.log']

// glob: shell-style pattern matching, returns full-ish paths matching the pattern
$logs = glob('/tmp/logs/*.log');
// ['/tmp/logs/app.log', '/tmp/logs/error.log']

$jsonOrYaml = glob('/tmp/conf/*.{json,yaml}', GLOB_BRACE);
```

`glob` is concise for patterns; `scandir` gives you everything (filter `.`/`..` yourself). For deep recursion use `RecursiveDirectoryIterator` + `RecursiveIteratorIterator`.

### Creating, copying, moving, deleting

```php
<?php

mkdir('/tmp/a/b/c', 0755, recursive: true); // recursive: create parents too
copy('/tmp/note.txt', '/tmp/note.bak');
rename('/tmp/note.bak', '/tmp/note.old');    // rename = move (works across names/dirs)
unlink('/tmp/note.old');                     // delete a FILE
rmdir('/tmp/a/b/c');                         // delete an EMPTY directory only
```

> `rmdir` fails on a non-empty directory. To delete a tree, recurse and `unlink` files first, or use Laravel's `Storage::deleteDirectory()` / `File::deleteDirectory()`.

### Path functions

These are **pure string** helpers — they don't touch the filesystem (except `realpath`).

```php
<?php

$p = '/var/www/app/index.php';

echo basename($p), "\n";            // index.php
echo basename($p, '.php'), "\n";    // index
echo dirname($p), "\n";             // /var/www/app
echo dirname($p, 2), "\n";          // /var/www  (go up 2 levels)

print_r(pathinfo($p));
// ['dirname' => '/var/www/app', 'basename' => 'index.php',
//  'extension' => 'php', 'filename' => 'index']

echo realpath('/var/www/app/../app/index.php'), "\n"; // /var/www/app/index.php (resolves .., symlinks; false if not found)
```

> `realpath` returns `false` if the path doesn't exist — handy as an existence + canonicalization check. Use the `DIRECTORY_SEPARATOR` constant or forward slashes (PHP accepts `/` on Windows too) for portable code.

### Streams in one breath

Every file function above actually operates on a **stream** — a uniform read/write abstraction. The `fopen` path can be a **wrapper**: `file://` (default), `php://`, `http://`, `data://`, `phar://`, etc.

```php
<?php

$stdout = fopen('php://stdout', 'wb');     // write to STDOUT
fwrite($stdout, "hello\n");

$mem = fopen('php://memory', 'rb+');       // in-RAM scratch file
fwrite($mem, 'cached');
rewind($mem);
echo stream_get_contents($mem);            // cached

// php://temp behaves like memory but spills to a temp file past ~2MB
```

`php://input` reads the raw request body (useful for non-form payloads), and a **stream context** lets you set options such as HTTP headers and timeouts on `file_get_contents`:

```php
<?php

$ctx = stream_context_create([
    'http' => [
        'method'  => 'GET',
        'header'  => "Accept: application/json\r\n",
        'timeout' => 5,
    ],
]);
$body = file_get_contents('https://example.com/api', false, $ctx);
// For real HTTP work prefer cURL / Guzzle / Laravel's Http facade — they give
// far better control over retries, TLS, and error handling.
```

This abstraction is why the same `fread`/`fwrite` code works on files, sockets, and compressed archives.

---

## Part 3 — JSON

### `json_encode` — PHP value → JSON string

```php
<?php

$data = [
    'name'   => 'José',
    'url'    => 'https://example.com/path',
    'tags'   => ['php', 'json'],
    'active' => true,
    'score'  => null,
];

echo json_encode($data), "\n";
// {"name":"José","url":"https:\/\/example.com\/path","tags":["php","json"],"active":true,"score":null}
```

Note two defaults that surprise people: non-ASCII is `\u`-escaped and forward slashes are escaped (`\/`). Flags fix both:

```php
<?php

$flags = JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES;
echo json_encode($data, $flags), "\n";
```

Output:

```json
{
    "name": "José",
    "url": "https://example.com/path",
    "tags": [
        "php",
        "json"
    ],
    "active": true,
    "score": null
}
```

Useful flags:

- `JSON_PRETTY_PRINT` — human-readable indentation (logs, files; skip it for wire payloads to save bytes).
- `JSON_UNESCAPED_UNICODE` — emit real UTF-8 instead of `\uXXXX`.
- `JSON_UNESCAPED_SLASHES` — leave `/` alone (cleaner URLs).
- `JSON_THROW_ON_ERROR` — throw `JsonException` instead of returning `false` (strongly recommended).
- `JSON_PRESERVE_ZERO_FRACTION` — keep `1.0` as `1.0`, not `1`.
- `JSON_NUMERIC_CHECK` — convert numeric **strings** to numbers (careful: turns `"0123"` into `123`).

> Array vs object output: a PHP array with **sequential integer keys from 0** encodes as a JSON **array** `[...]`; any other array (string keys or gaps) encodes as a JSON **object** `{...}`. An empty array `[]` encodes as `[]` — if you need `{}`, cast to object `(object)[]` or use `new stdClass`.

### `json_decode` — JSON string → PHP value

```php
<?php

$json = '{"name":"José","tags":["php","json"],"meta":{"id":7}}';

// Second arg = associative? false (default) -> stdClass objects
$obj = json_decode($json);
echo $obj->meta->id, "\n";       // 7

// true -> nested associative arrays
$arr = json_decode($json, associative: true);
echo $arr['meta']['id'], "\n";   // 7
```

Signature: `json_decode(string $json, ?bool $associative = null, int $depth = 512, int $flags = 0)`.

- **`$depth`** caps nesting to prevent stack exhaustion from deeply nested (possibly malicious) input. Exceeding it is an error.
- Big-integer handling: numbers larger than PHP's int become floats (losing precision) unless you pass `JSON_BIGINT_AS_STRING`.

```php
<?php
$big = json_decode('{"id": 12345678901234567890}', flags: JSON_BIGINT_AS_STRING);
var_dump($big->id); // string(20) "12345678901234567890"  (precision preserved)
```

### Error handling: `json_last_error` vs `JSON_THROW_ON_ERROR`

The old way: `json_decode` returns `null` on failure — but `null` is also a valid decoding of the literal `null`. So you must check `json_last_error()`:

```php
<?php

$result = json_decode('{bad json}');
if (json_last_error() !== JSON_ERROR_NONE) {
    echo 'Decode failed: ', json_last_error_msg(), "\n";
    // Decode failed: Syntax error
}
```

The modern way (PHP 7.3+, idiomatic in 8.x): pass `JSON_THROW_ON_ERROR` and catch `JsonException`. No ambiguous return values.

```php
<?php

try {
    $data = json_decode('{bad json}', true, flags: JSON_THROW_ON_ERROR);
} catch (JsonException $e) {
    echo 'Invalid JSON: ', $e->getMessage(), "\n"; // Invalid JSON: Syntax error
}

// Encoding can also fail (e.g., invalid UTF-8 or a resource):
$out = json_encode(['x' => "\xB1"], JSON_THROW_ON_ERROR); // throws JsonException
```

**Always use `JSON_THROW_ON_ERROR` for new code** — it converts silent failures into catchable exceptions.

### `JsonSerializable` — control how your objects encode

By default `json_encode` only serializes an object's **public** properties. Implement `JsonSerializable::jsonSerialize()` to define exactly what shape an object produces — without exposing internals.

```php
<?php

final class Money implements JsonSerializable
{
    public function __construct(
        private int $cents,
        private string $currency,
    ) {}

    public function jsonSerialize(): array
    {
        return [
            'amount'   => number_format($this->cents / 100, 2, '.', ''),
            'currency' => $this->currency,
        ];
    }
}

echo json_encode(['price' => new Money(1599, 'USD')], JSON_PRETTY_PRINT);
```

Output:

```json
{
    "price": {
        "amount": "15.99",
        "currency": "USD"
    }
}
```

> In Laravel, Eloquent models and API Resources already implement this pattern; `toArray()`/`toJson()` and `JsonResource` give you fine-grained control over API output (hiding columns, renaming keys, adding computed fields).

### `serialize`/`unserialize` and the critical security risk

`serialize()` produces a PHP-specific string that preserves types, including **objects** (class name + properties). `unserialize()` reconstructs them.

```php
<?php

$s = serialize(['a' => 1, 'b' => [2, 3]]);
echo $s, "\n";
// a:2:{s:1:"a";i:1;s:1:"b";a:2:{i:0;i:2;i:1;i:3;}}

$back = unserialize($s);
print_r($back); // Array ( [a] => 1 [b] => Array ( [0] => 2 [1] => 3 ) )
```

**The danger: never `unserialize()` untrusted input.** Because the payload names a class and supplies its properties, an attacker can craft a string that instantiates arbitrary objects. When those objects are destroyed or used, their magic methods (`__wakeup`, `__destruct`, `__toString`) run with attacker-controlled state — a **PHP Object Injection** attack that can chain into remote code execution or file deletion ("POP gadget chains"). The same class of risk does **not** apply to `json_decode`, which only ever produces arrays, `stdClass`, scalars, and `null` — never arbitrary instantiated classes.

Mitigations if you must `unserialize`:

```php
<?php

// Restrict which classes may be instantiated (others become __PHP_Incomplete_Class):
$data = unserialize($input, ['allowed_classes' => false]);          // arrays/scalars only
$data = unserialize($input, ['allowed_classes' => [Money::class]]); // a strict allowlist
```

**Best rule: for any data crossing a trust boundary (HTTP, cookies, queues, cache from untrusted sources), use JSON, not `serialize`.** Reserve `serialize` for internal, trusted round-trips where you control both ends — and even then prefer JSON or a typed DTO.

---

## ⚠️ Common Mistakes & Gotchas

1. **Mutating a `DateTime` by accident.** `$d->modify('+1 day')` changes `$d` in place and *also* returns it, so `$other = $d->modify(...)` aliases the same object. **Fix:** use `DateTimeImmutable` (or `CarbonImmutable`), or `clone`/`->copy()` before mutating.

2. **Trusting `createFromFormat` / `strtotime` blindly.** `createFromFormat('Y-m-d', '2026-02-30')` returns a *valid* object for March 2 (silent rollover), and `strtotime('garbage')` returns `false` which becomes timestamp `0` → "1970-01-01". **Fix:** check `DateTimeImmutable::getLastErrors()` plus a format round-trip; guard `strtotime`'s `false` before using it.

3. **Storing local time instead of UTC.** Saving wall-clock strings without a zone makes DST transitions and multi-region users ambiguous. **Fix:** store UTC (or timestamps) in the DB; convert to the user's `DateTimeZone` only for display.

4. **Looping on `feof()` alone.** `while (!feof($fh)) { $line = fgets($fh); use($line); }` processes one bogus iteration at EOF (or loops forever on a stalled stream) because `feof()` only flips *after* a failed read. **Fix:** check the read result: `while (($line = fgets($fh)) !== false)`.

5. **Forgetting `fclose`, locks, or buffer flushes.** Open handles leak descriptors; unflushed writes (or a held lock) corrupt concurrent access. **Fix:** wrap in `try/finally` and `fclose`; use `LOCK_EX` + `fflush` for shared files; `c+` mode (not `w+`) when you want to keep existing content while locking.

6. **`json_decode` returning `null` mistaken for valid `null`.** A parse error and a literal `null` both return `null`. **Fix:** pass `JSON_THROW_ON_ERROR` and catch `JsonException` (or check `json_last_error()`).

7. **Calling `unserialize()` on user input.** Opens the door to PHP Object Injection / RCE. **Fix:** use JSON for untrusted data; if unavoidable, pass `['allowed_classes' => false]` or a strict allowlist.

8. **Expecting an empty PHP array to become `{}`.** `json_encode([])` is `[]`, not `{}`. **Fix:** cast to `(object)[]` when the consumer expects a JSON object.

---

## ✅ Best Practices

- **Default to immutability** for dates (`DateTimeImmutable`, `CarbonImmutable`). Mutate only with a clear, local reason.
- **Be explicit about time zones** at every boundary; store UTC, display local. Set `date.timezone = UTC` as the app default.
- **Parse strictly**: known formats → `createFromFormat` + error/round-trip validation; never feed raw user strings to `new DateTimeImmutable` and hope.
- **Stream large files** with `fopen`/`fgets`/`fgetcsv`; reserve `file_get_contents`/`file()` for small, known-size files.
- **Always `try/finally` your handles** and close them; use `flock` (+`LOCK_EX`/`fflush`) for any file multiple processes touch.
- **Validate paths** before filesystem ops (`is_file`, `realpath`); never interpolate raw user input into a path (directory traversal). Constrain to a base directory.
- **Always encode/decode JSON with `JSON_THROW_ON_ERROR`**; choose array-vs-object decoding deliberately; add `JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES` for clean, compact output.
- **Implement `JsonSerializable`** (or Laravel API Resources) to define an object's public JSON shape instead of leaking properties.
- **Never `unserialize` untrusted data.** Prefer JSON; otherwise allowlist classes.
- In Laravel, prefer the **`Storage` facade** (filesystem abstraction over local/S3) and Carbon casts over raw PHP calls where the framework already wraps them.

---

## 🎯 Interview Tips & Likely Questions

1. **Q: Why prefer `DateTimeImmutable` over `DateTime`?**
   A: `DateTime`'s `modify/add/sub` mutate the object in place and return it, so shared references get changed unexpectedly (action at a distance). Immutable variants return a new object, making dates safe to pass around. It mirrors the value-object semantics dates should have.

2. **Q: How do you store and display dates across time zones correctly?**
   A: Store the moment in UTC (or a Unix timestamp) in the database. Keep zone info separate. At display time, `setTimezone(new DateTimeZone($userZone))` and `format()`. Never store ambiguous local wall-clock strings; DST makes them irreversible.

3. **Q: `createFromFormat` returned an object for an invalid date — why, and how do you validate?**
   A: It rolls over out-of-range components (Feb 30 → Mar 2) unless the format matches exactly. Validate by checking `DateTimeImmutable::getLastErrors()` for warnings/errors *and* round-tripping (`$d->format($fmt) === $input`).

4. **Q: Difference between `diff()->days` and the `%d` token?**
   A: `->days` is the absolute total number of days between two moments. `%d` in `DateInterval::format()` is the day remainder after years and months are extracted. They answer different questions.

5. **Q: When do you use `fopen`/`fgets` instead of `file_get_contents`?**
   A: For large files or streaming — to avoid loading everything into memory — and when you need locking (`flock`) or incremental processing like CSV. `file_get_contents` is fine for small config-sized files.

6. **Q: What's the `feof()` pitfall?**
   A: `feof()` only becomes true *after* a read hits EOF, so looping on it alone yields a stray empty iteration or, on a stuck stream, an infinite loop. Always check the return value of `fgets`/`fread`.

7. **Q: How does `json_decode` signal errors, and what's the modern approach?**
   A: Historically it returns `null`, indistinguishable from a literal `null`, so you check `json_last_error()`. Modern code passes `JSON_THROW_ON_ERROR` and catches `JsonException` for unambiguous, exception-based handling.

8. **Q: How does `json_encode` decide between a JSON array and object? (under the hood)**
   A: It inspects the PHP array's keys. Sequential integer keys starting at 0 (a "list") emit `[...]`; any string key or gap emits `{...}`. Internally PHP checks if the array is "packed"/list-like. Empty arrays emit `[]`; cast to object for `{}`. Objects use public props unless `JsonSerializable::jsonSerialize()` overrides the shape.

9. **Q: Why is `unserialize()` on untrusted input dangerous, while `json_decode` isn't? (under the hood)**
   A: `serialize` encodes the class name and properties, so `unserialize` *instantiates* those classes and may invoke magic methods (`__wakeup`, `__destruct`, `__toString`) with attacker-controlled state — PHP Object Injection / POP gadget chains leading to RCE or file deletion. `json_decode` can only produce arrays, `stdClass`, scalars, and `null` — never an arbitrary instantiated class — so there is no gadget surface. Mitigate `unserialize` with `allowed_classes`, but prefer JSON across trust boundaries.

10. **Q: What does the `b` flag in `fopen('file','wb')` do, and why use it?**
    A: It selects binary mode. On Windows, text mode translates `\n`↔`\r\n` and can corrupt binary data; `b` disables that. On Unix it's a no-op. Using `b` everywhere makes code portable and binary-safe.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ----- Dates -----
$d = new DateTimeImmutable('2026-06-18 14:00', new DateTimeZone('UTC'));
$d->modify('+3 days');                         // new object (immutable)
$d->add(new DateInterval('P1M'));              // +1 month
$d->sub(new DateInterval('PT90M'));            // -90 minutes
$d->setTimezone(new DateTimeZone('America/New_York'));
$d->format('Y-m-d H:i:s P');                   // 2026-06-18 14:00:00 +00:00
DateTimeImmutable::createFromFormat('d/m/Y', '18/06/2026');
$a->diff($b);                                  // DateInterval (->days, %m %d %h)
$a < $b; $a == $b; $a <=> $b;                  // moment comparison
time(); date('Y-m-d', $ts); strtotime('next monday'); // procedural

// ----- Files (convenience) -----
file_get_contents($path);                      // whole file -> string
file_put_contents($path, $data, FILE_APPEND|LOCK_EX);
file($path, FILE_IGNORE_NEW_LINES|FILE_SKIP_EMPTY_LINES); // -> array of lines

// ----- Files (handles) -----
$fh = fopen($path, 'rb');                      // modes: r r+ w w+ a a+ x x+ (+b)
while (($line = fgets($fh)) !== false) { /* ... */ }
$row = fgetcsv($fh, escape: "");               // 8.4: pass escape:""
flock($fh, LOCK_EX); /* ... */ fflush($fh); flock($fh, LOCK_UN);
fwrite($fh, $s); fread($fh, $bytes); rewind($fh); ftruncate($fh, 0);
fclose($fh);

// ----- Filesystem ops -----
file_exists($p); is_file($p); is_dir($p); is_writable($p);
mkdir($p, 0755, recursive: true); rmdir($emptyDir);
copy($a,$b); rename($a,$b); unlink($file);
scandir($dir); glob('/path/*.log');
basename($p); dirname($p, 2); pathinfo($p); realpath($p);

// ----- JSON -----
json_encode($v, JSON_PRETTY_PRINT|JSON_UNESCAPED_UNICODE|JSON_UNESCAPED_SLASHES|JSON_THROW_ON_ERROR);
json_decode($json, associative: true, depth: 512, flags: JSON_THROW_ON_ERROR);
json_last_error(); json_last_error_msg();      // legacy error path
class X implements JsonSerializable { public function jsonSerialize(): array {/*...*/} }

// ----- serialize (TRUSTED ONLY) -----
$s = serialize($v);
unserialize($s, ['allowed_classes' => false]); // never trust user input
```

---

## 🧪 Mini Exercises

1. **Strict date parser.** Write a function `toIso(string $ukDate): ?string` that accepts a UK date `DD/MM/YYYY`, rejects impossible dates like `31/02/2026` (return `null`), and otherwise returns the ISO `YYYY-MM-DD` string. Use `createFromFormat` plus a round-trip and `getLastErrors` check.

2. **Business-day countdown.** Given two `DateTimeImmutable` dates, compute the number of **weekdays** (Mon–Fri) strictly between them. Iterate with a `DateInterval('P1D')` loop and skip Saturdays/Sundays (`format('N') >= 6`).

3. **Locked log appender.** Write `appendLog(string $path, string $message): void` that opens the file in append mode, acquires an exclusive lock, writes a line prefixed with a UTC ISO 8601 timestamp, flushes, and releases the lock — using `try/finally` so the handle always closes.

4. **CSV → JSON converter.** Read a CSV with a header row using `fgetcsv` (8.4-correct `escape: ""`), turn each row into an associative array keyed by the header, and write the whole set to a `.json` file with pretty-printed, unescaped-unicode output and `JSON_THROW_ON_ERROR`.

5. **Safe transport round-trip.** Build a `Coordinate` value object (`lat`, `lng`) implementing `JsonSerializable`. Encode an array of coordinates to JSON, then decode it back into `Coordinate` instances. Explain in a comment why you used JSON here instead of `serialize`/`unserialize`.
