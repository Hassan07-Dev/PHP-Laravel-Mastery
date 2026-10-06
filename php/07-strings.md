# Strings in PHP (and Laravel)

Strings are the most-used data type in real-world backend code. Every HTTP request, JSON payload, database row, validation message, log line, email, and rendered HTML page is, at some layer, a string. Mastering strings means mastering *how text is stored, parsed, transformed, and safely emitted* — which is exactly what interviewers probe because bugs here cause security holes (XSS, injection), broken UTF-8, and subtle off-by-one errors.

This module takes you from "what is a string literal" to "why `strlen` lies about emoji and how to fix it," with runnable examples and Laravel-specific helpers.

> **What you'll learn**
> - The four ways to write string literals (single quotes, double quotes, heredoc, nowdoc) and exactly how interpolation differs between them
> - Interpolation forms for variables, array elements, and object properties — including the `{$...}` complex syntax
> - Concatenation vs interpolation, and when each is the right call
> - The essential string function families: searching, slicing, replacing, trimming, casing, padding, splitting/joining
> - Formatting numbers and money with `sprintf`, `number_format`, and `NumberFormatter`
> - Why byte-length functions (`strlen`, `substr`) are dangerous on UTF-8 and how the `mb_*` family fixes it
> - Safe HTML output (`htmlspecialchars`) and a first look at the `preg_*` regex functions
> - Laravel's `Str` helper and fluent `Stringable` API that wrap all of the above

---

## 1. What a string actually is

A PHP string is **a sequence of bytes**, not a sequence of characters. This single fact explains 90% of string gotchas. PHP does not store an "encoding" alongside the string — it just stores raw bytes and trusts you (and your functions) to interpret them correctly. ASCII text happens to be one byte per character, so `strlen("cat")` is `3` and everyone is happy. The moment you store `"café"` (where `é` is two bytes in UTF-8), `strlen` returns `5`, not `4`. We'll return to this repeatedly.

```php
<?php
echo strlen("cat");   // 3
echo PHP_EOL;
echo strlen("café");  // 5  (the é is 2 bytes in UTF-8)
```

Keep "bytes vs characters" in the back of your mind through every example below.

---

## 2. String literals: the four syntaxes

### 2.1 Single-quoted strings — literal, no interpolation

Single quotes are the **most literal** form. PHP does *not* parse variables or most escape sequences inside them. Only two escapes are recognized: `\'` (a literal single quote) and `\\` (a literal backslash).

```php
<?php
$name = 'World';
echo 'Hello $name';     // Output: Hello $name   (NOT interpolated)
echo 'Line1\nLine2';    // Output: Line1\nLine2  (\n is literal, not a newline)
echo 'It\'s fine';      // Output: It's fine     (\' is an escaped quote)
echo 'C:\\path';        // Output: C:\path
```

**Why use single quotes?** They're slightly faster (the parser never scans for `$`) and, more importantly, they signal intent: "this text is exactly what you see." Use them for fixed strings, array keys, and anything with no variables.

### 2.2 Double-quoted strings — interpolation + full escapes

Double quotes parse embedded variables and a rich set of escape sequences.

```php
<?php
$name = 'World';
echo "Hello $name";   // Output: Hello World
echo "Line1\nLine2";  // Output: two lines (\n is a real newline)
echo "Tab\tEnd";      // Output: Tab    End
```

Common escape sequences inside double quotes (and heredoc):

| Escape | Meaning |
|--------|---------|
| `\n` | newline (LF, 0x0A) |
| `\r` | carriage return (0x0D) |
| `\t` | horizontal tab |
| `\\` | literal backslash |
| `\"` | literal double quote |
| `\$` | literal dollar sign (suppresses interpolation) |
| `\u{1F600}` | Unicode codepoint (UTF-8 encoded) — PHP 7+ |
| `\x41` | byte from hex (`A`) |
| `\101` | byte from octal (`A`) |

```php
<?php
echo "Price: \$5";        // Output: Price: $5   (\$ escapes interpolation)
echo "\u{2764}";          // Output: ❤  (heavy black heart, UTF-8)
echo "\x48\x69";          // Output: Hi
```

### 2.3 Heredoc — like double quotes, for multi-line blocks

Heredoc behaves like a double-quoted string (interpolation + escapes) but is designed for large multi-line text without escaping every quote. Syntax: `<<<LABEL` ... closing `LABEL`.

```php
<?php
$user = 'Ada';
$role = 'admin';

$html = <<<HTML
<div class="card">
    <h1>Welcome, $user</h1>
    <p>Your role is {$role}.</p>
</div>
HTML;

echo $html;
/* Output:
<div class="card">
    <h1>Welcome, Ada</h1>
    <p>Your role is admin.</p>
</div>
*/
```

**PHP 7.3+ flexible heredoc:** the closing marker may be indented, and that indentation is stripped from every line. This lets heredoc align with your code:

```php
<?php
function render(string $title): string
{
    return <<<HTML
        <h1>$title</h1>
        <p>Body</p>
        HTML;   // indentation here = indentation stripped from each line
}
echo render('Hi');
/* Output (no leading spaces, because the closer was indented 8 spaces):
<h1>Hi</h1>
<p>Body</p>
*/
```

> Gotcha: the closing marker line must contain *only* the label (plus optional indentation and a trailing `;`). A common error is trailing whitespace or text after it.

### 2.4 Nowdoc — like single quotes, for multi-line blocks

Nowdoc is to heredoc what single quotes are to double quotes: **no interpolation, no escape processing**. Syntax is identical but the opening label is wrapped in single quotes: `<<<'LABEL'`.

```php
<?php
$user = 'Ada';

$template = <<<'TPL'
Hello $user
This {$x} is literal, and \n stays as backslash-n.
TPL;

echo $template;
/* Output:
Hello $user
This {$x} is literal, and \n stays as backslash-n.
*/
```

Use nowdoc for code templates, SQL with `$` placeholders, regex patterns, or any block you want copied verbatim.

---

## 3. Interpolation in depth

Interpolation = substituting a variable's value into a double-quoted/heredoc string. There are two flavors.

### 3.1 Simple syntax

PHP greedily reads a variable name after `$`. It can also follow **one** level of array access or property access without braces:

```php
<?php
$fruit = 'apple';
$basket = ['first' => 'pear'];
$user = new stdClass();
$user->name = 'Ada';

echo "I ate an $fruit";            // I ate an apple
echo "First: $basket[first]";      // First: pear   (note: NO quotes around the key!)
echo "User: $user->name";          // User: Ada
```

> Critical gotcha: in simple syntax, array keys are written **without quotes** — `"$basket[first]"`, not `"$basket['first']"`. Adding quotes breaks parsing. This is the opposite of normal array access outside strings.

### 3.2 Complex (curly-brace) syntax — `{$...}`

When the simple syntax can't express what you need (multi-dimensional arrays, method calls, ambiguous boundaries), wrap the whole expression in `{...}` starting with `$`:

```php
<?php
$user = ['profile' => ['name' => 'Ada']];
$order = new stdClass();
$order->items = ['book', 'pen'];

echo "Name: {$user['profile']['name']}";   // Name: Ada   (quotes ARE allowed here)
echo "First item: {$order->items[0]}";     // First item: book

// Method calls work inside {$...} (the expression must start with $):
$obj = new class {
    public function greet(): string { return 'hi'; }
};
echo "Method: {$obj->greet()}";            // Method: hi

// A plain function call like {strtoupper($x)} does NOT interpolate, because the
// braces only trigger parsing when immediately followed by $. Use concatenation:
echo 'Upper: ' . strtoupper($user['profile']['name']);   // Upper: ADA
```

The `{$...}` form is the safest, most readable choice for anything non-trivial. Note the brace must immediately precede the `$` (`{$x}`, not `{ $x }`). There is also a legacy `${name}` form — avoid it; **`${var}` and `${ expr }` string interpolation are deprecated as of PHP 8.2 and removed in PHP 9.0.** Always use `{$var}` instead.

### 3.3 Disambiguating boundaries

```php
<?php
$type = 'cat';
echo "I have 3 ${type}s";   // legacy form — DEPRECATED in 8.2, REMOVED in 9.0; avoid
echo "I have 3 {$type}s";   // PREFERRED: I have 3 cats
echo "I have 3 $type" . "s"; // also fine via concatenation
```

Without braces, `"$types"` would look for a variable named `$types` (likely undefined) instead of `$type` followed by `s`.

---

## 4. Concatenation vs interpolation

The concatenation operator is `.` (and `.=` for append).

```php
<?php
$first = 'Ada';
$last  = 'Lovelace';

$full = $first . ' ' . $last;     // Ada Lovelace
$greeting = 'Hi, ';
$greeting .= $full;               // Hi, Ada Lovelace
```

**Interpolation vs concatenation — which to use?**

```php
<?php
$name = 'Ada';
$id = 42;

// Interpolation: cleaner for simple insertion
$msg1 = "User $name has id #$id";

// Concatenation: clearer when mixing function calls / expressions
$msg2 = 'User ' . strtoupper($name) . ' has id #' . ($id + 1);
```

Rules of thumb:
- Prefer **interpolation** for readability when inserting plain variables: `"Hello $name"`.
- Prefer **concatenation** when joining function results or expressions, so you don't bury logic inside `{...}`.
- Performance difference is negligible in modern PHP; choose readability.

> **Watch operator precedence in PHP 8.** Historically `.` and `+` shared precedence; **PHP 8.0 lowered `.` below `+`/`-`**, so `"sum: " . 1 + 2` now parses as `"sum: " . (1 + 2)` → `"sum: 3"` (in PHP 7 it errored / behaved differently). When in doubt, parenthesize arithmetic.

---

## 5. Accessing characters by index

Strings support array-like `[]` indexing, returning a **single byte** as a one-character string. Indexing is zero-based; negative indexes count from the end.

```php
<?php
$s = "Laravel";
echo $s[0];    // L
echo $s[-1];   // l  (last byte; negative index since PHP 7.1)
echo $s[2];    // r

$len = strlen($s);
echo $s[$len - 1]; // l
```

You can also write to a position (it mutates the string, padding with spaces if you go past the end):

```php
<?php
$s = "cat";
$s[0] = 'b';
echo $s;       // bat
```

> UTF-8 caveat: `$s[$i]` returns one **byte**, not one character. For `"café"`, `$s[3]` and `$s[4]` are the two halves of `é` — neither is a valid standalone character. To get the *n-th character* safely, use `mb_substr($s, $i, 1)` (Section 11).

Out-of-range reads return `""` and emit a warning in PHP 8; always bounds-check.

---

## 6. Searching strings

### 6.1 The modern boolean helpers (PHP 8.0+)

Before PHP 8, everyone wrote `strpos($h, $n) !== false`, which is error-prone because position `0` is falsy. PHP 8 added three clear, intent-revealing functions:

```php
<?php
$email = 'ada@example.com';

var_dump(str_contains($email, '@'));        // bool(true)
var_dump(str_starts_with($email, 'ada'));   // bool(true)
var_dump(str_ends_with($email, '.com'));    // bool(true)
```

All three are **case-sensitive** and operate on bytes. Use them for readable existence checks.

### 6.2 `strpos` / `stripos` — find a position

`strpos` returns the zero-based byte index of the first match, or `false` if not found. `stripos` is the case-insensitive variant. `strrpos` finds the *last* occurrence.

```php
<?php
$h = 'Hello, World';

$pos = strpos($h, 'World');     // int(7)
$pos2 = strpos($h, 'xyz');      // bool(false)
$pos3 = stripos($h, 'hello');   // int(0)  (case-insensitive)

// THE classic bug: position 0 is falsy
if (strpos($h, 'Hello')) {       // WRONG: 0 is treated as false!
    echo "found";
} // (this branch does NOT run)

if (strpos($h, 'Hello') !== false) {  // CORRECT
    echo "found";                      // runs
}
```

> Interview favorite: "Why must you use `!== false` with `strpos`?" Because a match at index 0 returns integer `0`, which loose comparison treats as `false`. Prefer `str_contains` when you only need yes/no.

You can pass a start offset as the third argument: `strpos($h, ',', 5)`.

---

## 7. Slicing: `substr`

`substr($string, $start, $length)` extracts a byte-substring. `$start` and `$length` may be negative.

```php
<?php
$s = "Laravel";

echo substr($s, 0, 3);   // Lar
echo substr($s, 3);      // avel        (to end)
echo substr($s, -3);     // vel         (last 3)
echo substr($s, -3, 2);  // ve
echo substr($s, 2, -1);  // rave        (from index 2, stop 1 char before end)
```

For UTF-8 text use `mb_substr` (Section 11) so you slice by character, not byte.

---

## 8. Replacing

### 8.1 `str_replace` / `str_ireplace`

Replaces **all** occurrences. Accepts arrays for batch replacement. An optional 4th argument (by-reference) receives the replacement count.

```php
<?php
echo str_replace('cat', 'dog', 'cat cat');   // dog dog

// Array search + array replace (paired by index)
echo str_replace(['a', 'e'], ['4', '3'], 'apple');  // 4ppl3

// Array search + single replace (all map to one value)
echo str_replace(['<', '>'], '', '<b>hi</b>');      // bhi/b  (only < and > are removed, the / remains)

// Count how many replacements happened
$out = str_replace('x', 'y', 'xoxo', $count);
echo $count;   // 2
```

> Order matters with arrays. `str_replace(['A','B'], ['B','C'], 'A')` yields `'C'`, because after `A→B` the result `B` is then matched by the second rule `B→C`. Sequential, not simultaneous.

### 8.2 `substr_replace` — replace by position

```php
<?php
echo substr_replace('Hello World', 'PHP', 6);      // Hello PHP   (from index 6 to end)
echo substr_replace('Hello World', 'PHP', 6, 5);   // Hello PHP   (replace 5 chars)
echo substr_replace('1234567890', '****', 0, 6);   // ****7890    (mask first 6)
```

### 8.3 Regex replace — `preg_replace` (preview)

For pattern-based replacement use `preg_replace` (full regex coverage is a separate module):

```php
<?php
echo preg_replace('/\d+/', '#', 'a1b22c333');   // a#b#c#
```

---

## 9. Trimming and the trim family

`trim` removes characters from **both** ends; `ltrim`/`rtrim` from left/right only. By default they strip whitespace (` \t\n\r\0\x0B`). You can pass a custom character mask.

```php
<?php
echo trim("  hi  ") . "|";        // hi|
echo ltrim("xxhi", 'x') . "|";    // hi|
echo rtrim("hi;;;", ';') . "|";   // hi|
echo trim("/path/", '/') . "|";   // path|

// Ranges in the mask, e.g. strip digits 0-9
echo trim("12abc34", '0..9') . "|";  // abc|
```

> Gotcha: `trim` does **not** collapse internal whitespace. `trim(" a   b ")` is `"a   b"`. To collapse internal runs, use `preg_replace('/\s+/', ' ', $s)`.

> UTF-8 gotcha: `trim` works on bytes, so it can corrupt multibyte characters if your mask overlaps their bytes. **PHP 8.4 added `mb_trim()` / `mb_ltrim()` / `mb_rtrim()`**, which trim by character and, by default, strip the full set of Unicode whitespace (including non-breaking spaces). On PHP 8.3 and earlier, use `preg_replace('/^\s+|\s+$/u', '', $s)` or Laravel's `Str::trim`.

---

## 10. Case functions

```php
<?php
echo strtolower('HeLLo');   // hello
echo strtoupper('HeLLo');   // HELLO
echo ucfirst('hello');      // Hello   (first char only)
echo lcfirst('Hello');      // hello
echo ucwords('hello world');// Hello World (first letter of each word)
echo ucwords('a-b c', '-'); // A-B c    (custom word delimiters)
```

> These are **ASCII-only / locale-dependent** and will NOT correctly uppercase accented letters in a portable way. `strtoupper('café')` leaves `é` unchanged. For Unicode-aware casing use `mb_strtoupper('café')` → `CAFÉ`. For the first-character variants there is no `mb_ucfirst`/`mb_lcfirst` before PHP 8.4 — **both were added in PHP 8.4** (`mb_ucfirst('élan')` → `Élan`). (Section 11.)

---

## 11. Multibyte / UTF-8: the `mb_*` family

This is the most interview-relevant section. **PHP's core string functions count bytes.** UTF-8 encodes most non-ASCII characters in 2–4 bytes, so byte-based functions miscount length, slice mid-character, and corrupt text. The `mb_*` (multibyte) functions from the `mbstring` extension operate on **characters** in a given encoding (default UTF-8).

```php
<?php
$s = "café";          // é = U+00E9 = 2 bytes in UTF-8
echo strlen($s);       // 5  (bytes)
echo mb_strlen($s);    // 4  (characters)

$emoji = "Hi 👋";      // 👋 = 4 bytes (U+1F44B)
echo strlen($emoji);   // 7
echo mb_strlen($emoji);// 4
```

Multibyte equivalents you should know:

| Byte-based | Multibyte-safe | Purpose |
|------------|----------------|---------|
| `strlen` | `mb_strlen` | character count |
| `substr` | `mb_substr` | slice by character |
| `strpos` | `mb_strpos` | find char position |
| `strtolower` | `mb_strtolower` | Unicode lowercase |
| `strtoupper` | `mb_strtoupper` | Unicode uppercase |
| `ucfirst` | `mb_ucfirst` | uppercase first char (PHP 8.4+) |
| `lcfirst` | `mb_lcfirst` | lowercase first char (PHP 8.4+) |
| `str_split` | `mb_str_split` | array of characters |
| `str_pad` | `mb_str_pad` | pad by character (PHP 8.3+) |
| `trim` | `mb_trim` | trim by character (PHP 8.4+) |
| `ltrim` | `mb_ltrim` | left-trim by character (PHP 8.4+) |
| `rtrim` | `mb_rtrim` | right-trim by character (PHP 8.4+) |
| (none) | `mb_convert_encoding` | re-encode between charsets |
| (none) | `mb_detect_encoding` | best-guess encoding |
| (none) | `mb_check_encoding` | validate a string is valid UTF-8 |

```php
<?php
$s = "café au lait";

echo mb_substr($s, 0, 4);              // café   (4 characters, correct)
echo substr($s, 0, 4);                 // caf?   (4 bytes — splits the é!)

echo mb_strtoupper("café");            // CAFÉ
print_r(mb_str_split("héllo"));        // ['h','é','l','l','o']

var_dump(mb_check_encoding($s, 'UTF-8'));  // bool(true)
```

> **Uppercasing the first Unicode character.** PHP **8.4 added `mb_ucfirst()`** (and `mb_lcfirst()`), so `mb_ucfirst('élan')` → `Élan`. On PHP 8.3 and earlier there is no `mb_ucfirst`, so do it manually: `mb_strtoupper(mb_substr($s, 0, 1)) . mb_substr($s, 1)`. Plain `ucfirst('élan')` leaves the `é` untouched because it is ASCII-only.

> **Counting grapheme clusters.** Even `mb_strlen("👨‍👩‍👧")` (a family emoji built from joined codepoints) returns more than 1, because it counts codepoints, not user-perceived characters. For true "what a human sees" counting, use the `intl` extension's `grapheme_strlen`. Mentioning this in an interview signals depth.

**Set the default encoding** once at bootstrap so all `mb_*` calls assume UTF-8:

```php
<?php
mb_internal_encoding('UTF-8');
```

In Laravel this is already configured; the framework runs on UTF-8 end to end (and uses `mb_*` internally).

---

## 12. Formatting: `sprintf`, `printf`, `number_format`, padding, repeat

### 12.1 `sprintf` / `printf`

`sprintf` returns a formatted string; `printf` prints it and returns the length. Format specifiers start with `%`.

```php
<?php
printf("Hello %s, you are #%d\n", 'Ada', 7);   // Hello Ada, you are #7
echo sprintf("%05d", 42);        // 00042   (pad to width 5 with zeros)
echo sprintf("%.2f", 3.14159);   // 3.14    (2 decimals)
echo sprintf("%+d", 5);          // +5      (force sign)
echo sprintf("%x", 255);         // ff      (hex)
echo sprintf("%'*10s", 'hi');    // ********hi  (pad with * to width 10)
echo sprintf("%-10s|", 'hi');    // hi        |  (left-justify)
echo sprintf("%1\$s %1\$s", 'ho'); // ho ho   (argument numbering: reuse arg 1)
```

Common specifiers: `%s` string, `%d` integer, `%f` float, `%.2f` fixed decimals, `%x`/`%X` hex, `%b` binary, `%%` a literal percent.

### 12.2 `number_format` — thousands separators

```php
<?php
echo number_format(1234567.891);              // 1,234,568  (rounded, 0 decimals)
echo number_format(1234567.891, 2);           // 1,234,567.89
echo number_format(1234567.891, 2, '.', ' '); // 1 234 567.89 (custom separators)
echo number_format(1234.5, 2, ',', '.');      // 1.234,50   (German style)
```

Signature: `number_format(float $num, int $decimals = 0, string $decimalSep = '.', string $thousandsSep = ',')`.

### 12.3 Locale-aware money — `NumberFormatter` (intl)

For real currency formatting (correct symbol, placement, and grouping per locale), use the `intl` extension's `NumberFormatter` rather than hand-rolling with `number_format`:

```php
<?php
$fmt = new NumberFormatter('en_US', NumberFormatter::CURRENCY);
echo $fmt->formatCurrency(1234.5, 'USD');   // $1,234.50

$de = new NumberFormatter('de_DE', NumberFormatter::CURRENCY);
echo $de->formatCurrency(1234.5, 'EUR');    // 1.234,50 €
```

> Money rule: never store money as a float. Store integer cents (or use a decimal/BCMath type) and format only at the display layer. Floats cause rounding errors (`0.1 + 0.2 !== 0.3`).

### 12.4 `str_pad` and `str_repeat`

```php
<?php
echo str_pad('7', 3, '0', STR_PAD_LEFT);     // 007
echo str_pad('hi', 6, '.', STR_PAD_RIGHT);   // hi....
echo str_pad('hi', 6, '.', STR_PAD_BOTH);    // ..hi..
echo str_repeat('=', 10);                    // ==========
echo str_repeat('ab', 3);                    // ababab
```

> UTF-8 note: `str_pad` pads to a target **byte** width, so padding a string containing multibyte characters can produce the wrong visible width. **PHP 8.3 added `mb_str_pad()`**, which pads by character count — use it for UTF-8 text.

---

## 13. Splitting and joining

### 13.1 `explode` / `implode`

`explode` splits a string into an array by a delimiter; `implode` (alias `join`) joins an array into a string with a glue.

```php
<?php
$csv = "ada,grace,linus";
$parts = explode(',', $csv);
print_r($parts);                    // ['ada','grace','linus']

echo implode(' | ', $parts);        // ada | grace | linus

// Limit the number of pieces
print_r(explode(',', 'a,b,c,d', 2));   // ['a', 'b,c,d']

// Negative limit drops trailing pieces
print_r(explode(',', 'a,b,c,d', -1));  // ['a','b','c']
```

> Gotcha: `explode('', $s)` throws a `ValueError` — the delimiter cannot be empty. To split into individual characters, use `str_split` (bytes) or `mb_str_split` (characters).

### 13.2 `str_split` / `mb_str_split`

```php
<?php
print_r(str_split('abcdef', 2));   // ['ab','cd','ef']
print_r(mb_str_split('héllo'));    // ['h','é','l','l','o']  (UTF-8 safe)
```

---

## 14. Text & HTML helpers

### 14.1 `wordwrap`

Wraps a string to a given line width on word boundaries.

```php
<?php
echo wordwrap("The quick brown fox", 10, "\n", true);
/* Output:
The quick
brown fox
*/
```

### 14.2 `nl2br`

Inserts `<br />` before newlines (useful when rendering user text as HTML).

```php
<?php
echo nl2br("Line1\nLine2");   // Line1<br />\nLine2
```

### 14.3 `htmlspecialchars` and `htmlentities` — XSS defense

When you output user data into HTML, you **must** encode HTML metacharacters or you open an XSS hole. `htmlspecialchars` converts `& < > " '` into entities.

```php
<?php
$evil = '<script>alert(1)</script>';
echo htmlspecialchars($evil, ENT_QUOTES, 'UTF-8');
// Output: &lt;script&gt;alert(1)&lt;/script&gt;
```

- **PHP 8.1 changed the default flags** to `ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401`, so single quotes are now encoded by default and invalid UTF-8 is replaced rather than producing an empty string. Pre-8.1, the default `ENT_COMPAT` left single quotes unescaped — a real attribute-injection risk. Still, pass `ENT_QUOTES` explicitly for clarity.
- `htmlentities` encodes *all* characters that have HTML entity equivalents (e.g. `é` → `&eacute;`). With proper UTF-8 output you rarely need it — `htmlspecialchars` is enough and keeps text readable.

> In Blade, `{{ $var }}` already runs `htmlspecialchars` (via `e()`). Use `{!! $var !!}` only for trusted HTML — it bypasses escaping.

### 14.4 `strip_tags`

Removes HTML/PHP tags. Useful for generating plain-text excerpts — but it is **not** a security sanitizer (it won't neutralize malicious attributes or scripts in allowed tags). For sanitizing rich user HTML, use a real library like HTMLPurifier.

```php
<?php
echo strip_tags('<b>Hi</b> <a href="#">there</a>');     // Hi there
echo strip_tags('<b>Hi</b> <i>there</i>', '<b>');        // <b>Hi</b> there (allow <b>)
```

---

## 15. A first look at `preg_*` (regex)

Regular expressions (regex) are patterns for matching text. PHP's `preg_*` functions use PCRE (Perl-Compatible Regular Expressions). A pattern is a string wrapped in delimiters (commonly `/.../`), optionally followed by flags like `i` (case-insensitive) or `u` (UTF-8 mode).

```php
<?php
// Test for a match
var_dump(preg_match('/^\d{3}-\d{4}$/', '555-1234'));   // int(1)  (matched)

// Capture groups
preg_match('/(\w+)@(\w+)/', 'ada@example', $m);
print_r($m);   // [0 => 'ada@example', 1 => 'ada', 2 => 'example']

// Find all matches
preg_match_all('/\d+/', 'a1b22c333', $m);
print_r($m[0]);   // ['1','22','333']

// Replace by pattern
echo preg_replace('/\s+/', ' ', "too   many    spaces");  // too many spaces

// Split by pattern
print_r(preg_split('/[\s,]+/', "a, b,  c"));   // ['a','b','c']
```

> Always add the `u` flag (`/.../u`) when matching UTF-8 so `.`, `\w`, etc. operate on codepoints, not bytes. Regex gets its own deep-dive module; this is just the map.

---

## 16. Laravel string helpers

Laravel provides two ergonomic layers over the functions above: the static `Str` facade and the fluent `Str::of()` / `str()` `Stringable` API. They are UTF-8 aware and chainable.

```php
<?php
use Illuminate\Support\Str;

Str::contains('Hello World', 'World');     // true
Str::startsWith('foo.bar', 'foo');         // true
Str::limit('A very long sentence', 10);    // "A very lon..."
Str::slug('Hello World! 2024');            // "hello-world-2024"
Str::camel('foo_bar');                     // "fooBar"
Str::snake('fooBar');                      // "foo_bar"
Str::studly('foo_bar');                    // "FooBar"
Str::title('hello world');                 // "Hello World"
Str::plural('child');                      // "children"
Str::singular('cars');                     // "car"
Str::mask('1234567890', '*', 0, 6);        // "******7890"
Str::ascii('café');                        // "cafe"
Str::uuid();                               // a UUID v4 instance
Str::random(16);                           // 16 secure-random chars
```

The fluent API (a `Stringable` object) lets you chain transformations and reads like a pipeline:

```php
<?php
use Illuminate\Support\Str;

$result = Str::of('  Hello World  ')
    ->trim()
    ->lower()
    ->replace(' ', '-')
    ->finish('!');     // ensure it ends with "!"

echo $result;   // "hello-world!"

// The str() helper is shorthand for Str::of()
echo str('laravel rocks')->headline();   // "Laravel Rocks"
```

Notable Laravel 11/12 additions worth name-dropping: `Str::isUrl()`, `Str::isJson()`, `Str::isUuid()`, `Str::wordWrap()`, `Str::take()`, `Str::chopStart()` / `Str::chopEnd()`, and the `Stringable` `->toBase64()` / `->fromBase64()` and `->hash('sha256')` methods. Laravel 12 continues this fluent API; check `Illuminate\Support\Str` for the current surface.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `strpos` truthiness.**
   ```php
   if (strpos($s, $needle)) { ... }   // BUG: a match at index 0 is falsy
   ```
   **Fix:** compare explicitly: `strpos($s, $needle) !== false`, or use `str_contains($s, $needle)` when you only need a boolean.

2. **Counting/slicing UTF-8 with byte functions.**
   ```php
   strlen("café");        // 5, not 4
   substr("café", 0, 4);  // "caf" + half of é → corrupt
   ```
   **Fix:** use `mb_strlen` / `mb_substr` (and set `mb_internal_encoding('UTF-8')`).

3. **Forgetting to escape output → XSS.** Echoing raw user input into HTML, or using Blade `{!! $userInput !!}`.
   **Fix:** use `htmlspecialchars($s, ENT_QUOTES, 'UTF-8')` in plain PHP, and `{{ $var }}` (auto-escaped) in Blade. Reserve `{!! !!}` for trusted markup.

4. **Quoting array keys in simple interpolation.**
   ```php
   echo "$arr['key']";   // Parse error / unexpected behavior
   ```
   **Fix:** drop the quotes in simple syntax (`"$arr[key]"`) or, better, use complex syntax `"{$arr['key']}"`.

5. **Expecting `trim` to collapse inner whitespace.** It only strips the ends.
   **Fix:** `preg_replace('/\s+/', ' ', trim($s))` to normalize internal runs.

6. **Storing money as floats / formatting with `number_format` and assuming locale.** Floats round badly and `number_format` ignores locale.
   **Fix:** store integer cents; format with `NumberFormatter::CURRENCY` for locale-correct output.

7. **Empty delimiter to `explode`.** `explode('', $s)` throws `ValueError` in PHP 8.
   **Fix:** use `str_split` / `mb_str_split` to split into characters.

8. **Relying on the pre-8.0 `.` precedence.** `echo "n=" . $a + $b;` parses differently in PHP 8.
   **Fix:** parenthesize: `"n=" . ($a + $b)`.

---

## ✅ Best Practices

- Default to **single quotes** for literal strings with no variables; use double quotes / heredoc only when you interpolate.
- Use the **PHP 8 boolean helpers** (`str_contains`, `str_starts_with`, `str_ends_with`) for readable existence checks instead of `strpos(...) !== false`.
- Treat all text as **UTF-8** end to end: call `mb_*` functions, set `mb_internal_encoding('UTF-8')`, declare `<meta charset="utf-8">`, and use `utf8mb4` in MySQL. On PHP 8.3+/8.4+ prefer the newer `mb_str_pad`, `mb_trim`/`mb_ltrim`/`mb_rtrim`, and `mb_ucfirst`/`mb_lcfirst` over their byte-based counterparts for user-facing text.
- **Escape at the boundary**: HTML-escape with `htmlspecialchars`/Blade on output, never trust `strip_tags` as a security control.
- Prefer **`sprintf`/`number_format`/`NumberFormatter`** over manual concatenation for formatted output; it's clearer and locale-aware.
- Use **`{$...}` complex interpolation** for anything beyond a bare variable — it's unambiguous and survives refactors.
- In Laravel, reach for **`Str`/`Stringable`** helpers; they're UTF-8 safe, chainable, and well-tested.
- Validate/normalize encoding on input with **`mb_check_encoding`** before processing untrusted text.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Difference between single and double quotes?**
A. Single quotes are literal — no variable interpolation, and only `\\` and `\'` are recognized as escapes. Double quotes interpolate variables and process the full escape set (`\n`, `\t`, `\u{...}`, etc.). Single quotes are marginally faster and signal "verbatim."

**Q2. What's the difference between heredoc and nowdoc?**
A. Heredoc (`<<<LABEL`) behaves like a double-quoted string (interpolation + escapes); nowdoc (`<<<'LABEL'`) behaves like a single-quoted string (no interpolation, no escapes). Both support multi-line blocks; since PHP 7.3 the closing marker may be indented and that indentation is stripped.

**Q3. Why can't you write `if (strpos($s, $n))`?**
A. Because `strpos` returns the integer index of the first match, and a match at index 0 is `0`, which is loosely falsy. You must use `!== false`, or use `str_contains` for a boolean.

**Q4. `strlen` vs `mb_strlen` — and how does it work under the hood?**
A. A PHP string is a byte array with no attached encoding. `strlen` returns the raw byte count. UTF-8 encodes a codepoint in 1–4 bytes (ASCII = 1, `é` = 2, most emoji = 4), so `strlen("café")` is 5. `mb_strlen` decodes the bytes according to the given encoding (default UTF-8) and counts *characters* (codepoints). Note even `mb_strlen` counts codepoints, not grapheme clusters — combined emoji like 👨‍👩‍👧 count as several; for human-perceived length use `grapheme_strlen` from intl.

**Q5. How do you prevent XSS when echoing user input into HTML?**
A. Encode HTML metacharacters with `htmlspecialchars($s, ENT_QUOTES, 'UTF-8')` (PHP 8.1+ already defaults to `ENT_QUOTES | ENT_SUBSTITUTE`). In Blade, `{{ $x }}` auto-escapes via `e()`; `{!! $x !!}` does not, so only use it for trusted HTML. `strip_tags` is not a security control.

**Q6. Concatenation vs interpolation — performance and style?**
A. Performance is effectively equal in modern PHP. Choose interpolation for inserting plain variables (`"Hi $name"`), concatenation when joining expressions or function calls so logic isn't buried inside `{...}`. Note PHP 8.0 lowered `.` precedence below `+`/`-`.

**Q7. How would you format a price for different locales?**
A. Don't hand-roll. Store money as integer cents (avoid floats). For display, use `NumberFormatter` from intl: `(new NumberFormatter('de_DE', NumberFormatter::CURRENCY))->formatCurrency($amount, 'EUR')`. `number_format` is fine for simple, locale-fixed grouping.

**Q8. Why might `substr($utf8, 0, 10)` produce garbage?**
A. It slices by bytes, so it can cut through the middle of a multibyte character, leaving an invalid byte sequence. Use `mb_substr` to slice by character.

**Q9. What does the `u` flag do in a regex, and why does it matter for strings?**
A. It puts PCRE in UTF-8 mode so the pattern and subject are treated as sequences of UTF-8 codepoints rather than bytes — `.`, `\w`, and character classes then match whole characters. Omitting it on UTF-8 input can match or split mid-character.

**Q10. How does Laravel's `Str` differ from native functions?**
A. `Str` methods are UTF-8 aware wrappers (often delegating to `mb_*`/PCRE), add high-level helpers (`slug`, `camel`, `plural`, `mask`), and have a fluent `Stringable` form via `Str::of()` / `str()` for chaining. They make intent clearer and avoid the byte-vs-character footguns.

---

## 📋 Quick Reference / Cheat Sheet

```php
// Literals
'literal $x'             // no interpolation
"interp $x and {$y->z}"  // interpolation + escapes
<<<TXT ... TXT;          // heredoc (like double quotes)
<<<'TXT' ... TXT;        // nowdoc  (like single quotes)

// Length & access (byte vs char)
strlen($s);  mb_strlen($s);
$s[0];  $s[-1];                 // single byte by index
mb_substr($s, 0, 1);            // first character (UTF-8 safe)

// Search (PHP 8+ booleans)
str_contains($h, $n);
str_starts_with($h, $n);  str_ends_with($h, $n);
strpos($h, $n) !== false;       // position (use !== false)
stripos($h, $n);                // case-insensitive

// Slice / replace
substr($s, $start, $len);  mb_substr($s, $start, $len);
str_replace($search, $replace, $subject, $count);
substr_replace($s, $repl, $start, $len);
preg_replace('/pat/u', $repl, $s);

// Trim / case
trim($s, $mask);  ltrim();  rtrim();
mb_trim($s);  mb_ltrim($s);  mb_rtrim($s);  // Unicode-safe (PHP 8.4+)
strtolower();  strtoupper();  ucfirst();  ucwords();
mb_strtolower();  mb_strtoupper();          // Unicode-safe
mb_ucfirst();  mb_lcfirst();                // Unicode-safe (PHP 8.4+)

// Format
sprintf("%05.2f", $n);  printf(...);
number_format($n, 2, '.', ',');
(new NumberFormatter('en_US', NumberFormatter::CURRENCY))->formatCurrency($n, 'USD');
str_pad($s, $len, $pad, STR_PAD_LEFT);  str_repeat($s, $n);
mb_str_pad($s, $len, $pad, STR_PAD_LEFT);   // Unicode-safe (PHP 8.3+)

// Split / join
explode(',', $s, $limit);  implode($glue, $arr);
str_split($s, $n);  mb_str_split($s);

// Text / HTML
wordwrap($s, $w, "\n", true);  nl2br($s);
htmlspecialchars($s, ENT_QUOTES, 'UTF-8');  strip_tags($s, '<b>');

// Laravel
use Illuminate\Support\Str;
Str::slug($s);  Str::limit($s, 50);  Str::camel($s);  Str::mask($s,'*',0,6);
Str::of($s)->trim()->lower()->replace(' ', '-')->finish('!');
```

---

## 🧪 Mini Exercises

1. **Safe slug maker (no Laravel).** Write `slugify(string $s): string` that lowercases (UTF-8 safe), replaces any run of non-alphanumeric characters with a single hyphen, and trims leading/trailing hyphens. `slugify("  Héllo,  World! 2024 ")` should return `hello-world-2024` (hint: combine `mb_strtolower`, `preg_replace` with the `u` flag, and `trim`).

2. **Masked card number.** Given `"4111111111111234"`, output `"************1234"` using `substr` and `str_repeat` (or `str_pad`). Then redo it with `Str::mask`. Verify both give the same result.

3. **UTF-8 length report.** Write a function that, for a given string, prints its byte length, its `mb_strlen` character length, and whether it's valid UTF-8. Test it with `"café"`, `"Hi 👋"`, and a deliberately invalid byte sequence (`"\xFF\xFE"`).

4. **Currency table.** Given an array of cent amounts `[199, 1050, 1234567]`, print each formatted as USD using `NumberFormatter`, right-aligned in a 15-character column (combine `str_pad` with the formatter output).

5. **Template renderer.** Implement `render(string $template, array $vars): string` that replaces `{{ key }}` placeholders (allowing optional surrounding spaces) with values from `$vars`, leaving unknown placeholders untouched. Use `preg_replace_callback` with a `/u` pattern. Example: `render('Hi {{ name }}', ['name' => 'Ada'])` → `Hi Ada`.
