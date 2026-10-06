# Validation in Laravel

Validation is the gatekeeper of your application. Every byte that enters from the outside world — a registration form, a JSON API payload, a query string, an uploaded file — is *untrusted* until you prove otherwise. This module teaches you how Laravel turns that messy, hostile input into clean, typed, business-rule-compliant data, and how to do it idiomatically in Laravel 12 on PHP 8.4.

> **Versions targeted:** PHP 8.4 (with notes on 8.1–8.3) and Laravel 12 (with notes where Laravel 10/11 behave differently). Validation is one of the most stable parts of the framework, so almost everything here works unchanged back to Laravel 9; version-specific notes are called out inline.

---

## **What you'll learn**

- *Why* validation matters (security, data integrity, UX) and where it belongs in the request lifecycle.
- The three ways to validate: the `validate()` helper on requests, **Form Request** classes, and the manual `Validator` factory.
- How validation errors flow back to Blade views via the **error bag**, `@error`, and `old()`.
- The most common rules — `required`, `nullable`, `sometimes`, `email`, `unique`, `exists`, `confirmed`, `min/max`, `in`, `regex`, `image`, `mimes` — and how to validate **arrays and nested data** with dot and `*` notation.
- **Conditional** validation (`sometimes`, `required_if`, the `sometimes()` method) and **custom rules** (closures, `Rule` objects, invokable rule classes).
- How Laravel produces the **422 JSON error format** for APIs, and the crucial difference between `validated()` and `all()`.

---

## 1. Why validate at all?

Three independent reasons, each sufficient on its own:

1. **Security.** Untrusted input is the root of most web vulnerabilities — SQL injection, mass assignment, stored XSS, path traversal. Validation is your first line of defense. Even with Eloquent's parameter binding protecting you from SQLi, you still need to reject a `role=admin` field that a user smuggled into a registration form.
2. **Data integrity.** Your database has constraints (a `users.email` column should be unique and look like an email). Catching violations *before* the query runs gives users a friendly message instead of a 500 error from a duplicate-key exception.
3. **User experience.** When a form is wrong, users expect to see *which* fields are wrong, *why*, and to keep what they already typed. Laravel's validation system gives you all three nearly for free.

**The golden rule:** *Never trust the client.* JavaScript validation is a UX nicety; it can be bypassed with a single `curl` command. Server-side validation is the real enforcement.

### Where validation sits in the lifecycle

A request flows: **HTTP request → middleware → route → controller (or Form Request) → business logic → response.** Validation runs at the *boundary* — as early as possible, before any business logic touches the data. If validation fails, Laravel short-circuits: it never reaches your controller body.

---

## 2. The `validate()` method — your everyday tool

The fastest way to validate is the `validate()` method available on every incoming `Illuminate\Http\Request`. You call it inside a controller:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Post;

class PostController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([
            'title'   => ['required', 'string', 'max:255'],
            'body'    => ['required', 'string'],
            'publish' => ['boolean'],
        ]);

        // Execution only reaches here if ALL rules pass.
        // $validated contains ONLY the keys that had rules.
        $post = Post::create($validated);

        return redirect()->route('posts.show', $post);
    }
}
```

**What happens on failure?** `validate()` throws an `Illuminate\Validation\ValidationException`. Laravel's exception handler catches it and does one of two things based on the request's `Accept` header / whether it `expectsJson()`:

- **Web request (HTML):** redirect *back* to the previous URL (HTTP 302), with the errors and the old input flashed to the session.
- **API request (JSON):** return a **422 Unprocessable Entity** response with a JSON body of errors (covered in §11).

This automatic branching is why the *same* validation code works for both web forms and APIs.

### Rule syntax: array vs. pipe string

Two equivalent syntaxes:

```php
// Pipe-delimited string (concise, but fragile)
'title' => 'required|string|max:255',

// Array (preferred — required when a rule value contains a "|", e.g. regex)
'title' => ['required', 'string', 'max:255'],
```

> **Prefer the array syntax.** A regex rule like `regex:/^[a-z|0-9]+$/` contains a pipe that breaks the string parser. Arrays also allow you to pass **rule objects** (`Rule::unique(...)`), which strings cannot. The array form is the modern idiom.

---

## 3. Displaying errors in Blade

When a web request fails validation and redirects back, Laravel flashes two things to the session:

- An `$errors` variable — an instance of `Illuminate\Support\MessageBag` — **automatically available in every Blade view** (injected by the `Illuminate\View\Middleware\ShareErrorsFromSession` middleware that lives in the `web` middleware group).
- The **old input**, retrievable with the `old()` helper.

### The `@error` directive (the modern way)

```blade
<form method="POST" action="/posts">
    @csrf

    <label for="title">Title</label>
    <input id="title"
           name="title"
           type="text"
           value="{{ old('title') }}"
           class="@error('title') is-invalid @enderror">

    @error('title')
        <span class="text-red-600">{{ $message }}</span>
    @enderror

    <button type="submit">Save</button>
</form>
```

- `@error('title') ... @enderror` runs its body only if the `title` field has an error. Inside, `$message` is the first error message for that field.
- `value="{{ old('title') }}"` repopulates the field with what the user previously typed, so they don't lose their work. The second argument is a default: `old('publish', $post->publish)`.

### Working with the error bag directly

```blade
{{-- Is there any error at all? --}}
@if ($errors->any())
    <div class="alert">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

{{-- First message for a specific field --}}
{{ $errors->first('email') }}

{{-- ALL messages for a field (a field can fail multiple rules) --}}
@foreach ($errors->get('email') as $message)
    <p>{{ $message }}</p>
@endforeach

{{-- Does a field have an error? --}}
@if ($errors->has('email')) ... @endif
```

### Named error bags

If a page has multiple forms (e.g. a "login" and a "register" form), give each its own bag so their errors don't collide:

```php
return redirect('dashboard')->withErrors($validator, 'login');
```

```blade
{{ $errors->login->first('email') }}
```

Behind the scenes `$errors` is actually a `ViewErrorBag` that holds one or more named `MessageBag` instances; `$errors->first()` reads from the `default` bag.

---

## 4. Form Request classes — the idiomatic choice

For anything beyond a trivial form, move validation out of the controller into a dedicated **Form Request** class. This keeps controllers thin and makes validation reusable and testable.

```bash
php artisan make:request StorePostRequest
# => app/Http/Requests/StorePostRequest.php
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    /**
     * Authorize the request (replaces a policy check / gate inline).
     * Return false -> 403 Forbidden, request never validated.
     */
    public function authorize(): bool
    {
        return $this->user()?->can('create', \App\Models\Post::class) ?? false;
    }

    /**
     * The validation rules.
     *
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'title'      => ['required', 'string', 'max:255'],
            'body'       => ['required', 'string'],
            'slug'       => ['required', 'string', Rule::unique('posts', 'slug')],
            'tags'       => ['array'],
            'tags.*'     => ['string', 'max:50'],
            'publish_at' => ['nullable', 'date', 'after:now'],
        ];
    }
}
```

Then **type-hint** the Form Request in your controller. Laravel's service container resolves it, and validation runs *automatically* before the method body executes:

```php
public function store(StorePostRequest $request)
{
    // Already authorized and validated by the time we get here.
    $post = Post::create($request->validated());

    return redirect()->route('posts.show', $post);
}
```

### The full toolbox of a Form Request

```php
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'email' => ['required', 'email'],
            'age'   => ['required', 'integer', 'min:18'],
        ];
    }

    /**
     * Custom error messages. Keyed by "field.rule".
     */
    public function messages(): array
    {
        return [
            'email.required' => 'We need your email to contact you.',
            'age.min'        => 'You must be at least :min years old.',
        ];
    }

    /**
     * Human-friendly attribute names used inside messages.
     * Turns "The age field is required" -> "The applicant age field is required".
     */
    public function attributes(): array
    {
        return [
            'age' => 'applicant age',
        ];
    }

    /**
     * Normalize / clean input BEFORE rules run.
     * Great for trimming, casting, defaulting.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower(trim((string) $this->input('email'))),
            'slug'  => $this->slug ?: \Illuminate\Support\Str::slug($this->title),
        ]);
    }

    /**
     * Runs AFTER validation passes. Use it to mutate the request payload
     * (e.g. cast/replace fields) via $this->merge()/$this->replace().
     * Note: it does not change which keys validated() returns — only the
     * keys that had rules are returned, but their values reflect changes
     * made here if you merge/replace them.
     */
    protected function passedValidation(): void
    {
        $this->replace(['email' => strtolower($this->email)]);
    }
}
```

### `after` hooks — cross-field validation in one place

When a rule depends on multiple fields or external state, the cleanest Laravel 12 approach is the `after()` method on the Form Request (introduced in Laravel 11). It returns an array of callables (or invokable objects) that receive the `Validator`:

```php
use Illuminate\Validation\Validator;

public function after(): array
{
    return [
        function (Validator $validator) {
            if ($this->starts_at && $this->ends_at && $this->ends_at <= $this->starts_at) {
                $validator->errors()->add(
                    'ends_at',
                    'The end time must be after the start time.'
                );
            }
        },
    ];
}
```

> **Laravel 10 and earlier** did not have the `after()` method on Form Requests. There you override `withValidator(Validator $validator)` and call `$validator->after(function ($v) { ... })`. Both still work in 12; `after()` is just the tidier modern form.

```php
// Laravel 10 style (still valid in 12)
public function withValidator(\Illuminate\Validation\Validator $validator): void
{
    $validator->after(function ($validator) {
        if (/* condition */) {
            $validator->errors()->add('field', 'message');
        }
    });
}
```

---

## 5. The `Validator` factory — manual validation

Sometimes you need full control: validating data that didn't come from an HTTP request (a console command, a queued job, a CSV row), or deciding for yourself what to do on failure. Use the `Validator` facade:

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'email' => ['required', 'email'],
    'votes' => ['required', 'integer', 'min:0'],
]);

if ($validator->fails()) {
    // $validator->errors() is a MessageBag
    return back()->withErrors($validator)->withInput();
}

// Throws ValidationException if it fails (same behavior as $request->validate())
$validated = $validator->validate();

// Or just grab the clean, rule-covered subset:
$clean = $validator->validated();
```

`Validator::make($data, $rules, $messages = [], $attributes = [])` accepts the same four arguments as the Form Request methods. Useful inspection methods:

| Method | Returns |
|---|---|
| `$validator->fails()` | `bool` — true if any rule failed |
| `$validator->passes()` | `bool` — inverse of `fails()` |
| `$validator->errors()` | `MessageBag` of all errors |
| `$validator->validated()` | array of validated data |
| `$validator->safe()` | a `ValidatedInput` object (supports `->only()`, `->except()`, `->merge()`) |
| `$validator->validate()` | validated data, or **throws** on failure |

---

## 6. The common rules, by category

Below is the working vocabulary. Rules are evaluated left-to-right per field.

### Presence & nullability

```php
'name'      => ['required', 'string'],
'nickname'  => ['nullable', 'string'],        // null/empty allowed; rules skipped if empty
'avatar'    => ['sometimes', 'image'],         // only validated IF the key is present
'terms'     => ['accepted'],                   // must be "yes", "on", 1, true
'filled'    => ['filled'],                      // if present, must not be empty
```

- **`required`** — must be present *and* non-empty (`null`, `''`, empty array, no uploaded file all fail).
- **`nullable`** — explicitly allows `null`. **Critical gotcha:** without `nullable`, a field sent as `null` will *fail* rules like `date` or `integer`, because by default Laravel does not skip validation for `null` values on those rules.
- **`sometimes`** — *"only run the rest of the rules if this field is present in the input."* Lets you support partial updates (PATCH) where the client sends only some fields.
- **`filled`** — *if* the field is present it must not be empty (but it may be absent).

### Types

```php
'age'    => ['integer'],
'price'  => ['numeric', 'decimal:2'],   // decimal:2 => exactly 2 decimal places
'active' => ['boolean'],                // accepts true/false/1/0/"1"/"0"
'name'   => ['string'],
'tags'   => ['array'],
'meta'   => ['json'],
```

### Strings & format

```php
'email'    => ['required', 'email:rfc,dns'],   // dns = check the domain has MX records
'website'  => ['url', 'active_url'],
'username' => ['alpha_dash'],                   // letters, numbers, dashes, underscores
'code'     => ['regex:/^[A-Z]{3}-\d{4}$/'],     // e.g. ABC-1234
'phone'    => ['regex:/^\+?[1-9]\d{7,14}$/'],
'bio'      => ['string', 'max:500'],
'uuid'     => ['uuid'],
'ulid'     => ['ulid'],
```

> **PHP 8.4 note:** Laravel's `email` rule uses the `egulias/email-validator` package, not PHP's native `filter_var`, so behavior is consistent across PHP versions. The `email:rfc,dns,strict` options control how strict the check is. The available validators are: `rfc` (default), `strict` (fail on RFC *warnings*), `dns` (MX record check), `spoof` (homograph/spoofing check), `filter` and `filter_unicode` (PHP's `filter_var`).
>
> **Laravel 12 fluent form:** there is now a `Rule::email()` builder that reads better than the colon string:
> ```php
> use Illuminate\Validation\Rule;
>
> 'email' => ['required', Rule::email()->rfcCompliant(strict: false)->validateMxRecord()->preventSpoofing()],
> ```

### Size constraints — `min`, `max`, `between`, `size`

The meaning of these **depends on the field's type**, which is why ordering matters:

```php
'name'   => ['string', 'min:2', 'max:50'],   // string LENGTH
'age'    => ['integer', 'min:18', 'max:120'], // numeric VALUE
'tags'   => ['array', 'min:1', 'max:5'],      // array COUNT
'avatar' => ['file', 'max:2048'],             // file KILOBYTES (so 2 MB)
```

> **Gotcha:** `'votes' => ['max:10']` with no type rule treats a numeric string as a number, but treats a non-numeric string by length. Always declare the type first (`'integer'`, `'string'`, `'array'`) so `min`/`max` are unambiguous.

### Enumerations — `in`, `not_in`, and `Rule::in`

```php
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Enum;
use App\Enums\Status;

'visibility' => ['required', Rule::in(['public', 'private', 'unlisted'])],
'role'       => ['required', 'not_in:admin,superadmin'],

// Validate against a PHP 8.1+ backed enum (the modern idiom):
'status'     => ['required', Rule::enum(Status::class)],
// equivalent older form:
'status'     => ['required', new Enum(Status::class)],
```

Where `Status` is:

```php
<?php

namespace App\Enums;

enum Status: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';
}
```

> **Laravel 11+** added `Rule::enum(...)->only([...])` and `->except([...])` to restrict which enum cases are accepted — handy when a public form should not allow an internal status.

### Database rules — `unique` and `exists`

These run a query, so they require a DB connection.

```php
use Illuminate\Validation\Rule;

// exists: the value must be a row in another table
'category_id' => ['required', Rule::exists('categories', 'id')],

// unique on create:
'email' => ['required', 'email', Rule::unique('users', 'email')],

// unique on UPDATE — ignore the current row, else the user's own email "collides":
'email' => [
    'required',
    'email',
    Rule::unique('users', 'email')->ignore($this->user()->id),
],

// add WHERE clauses (e.g. soft-deletes or scoping to a tenant):
'slug' => [
    Rule::unique('posts', 'slug')
        ->where(fn ($q) => $q->where('team_id', $this->team_id))
        ->whereNull('deleted_at'),
],
```

> **Security gotcha with `ignore`:** never pass user-supplied input to `->ignore()` directly when it could be a raw SQL value; pass the model or its ID (`->ignore($user->id, 'id')`). Using `->ignore($user)` (a model) is safest.

### Confirmation — `confirmed`

```php
'password' => ['required', 'min:8', 'confirmed'],
```

`confirmed` expects a *second* field named `<field>_confirmation` (here, `password_confirmation`) to be present and equal. No matching field = failure. In **Laravel 11+** you can customize the expected field: `'confirmed:repeat_password'`.

### Dates

```php
'starts_at' => ['required', 'date'],
'ends_at'   => ['required', 'date', 'after:starts_at'],     // cross-field!
'born_on'   => ['date', 'before:today', 'date_format:Y-m-d'],
'expires'   => ['date', 'after_or_equal:tomorrow'],
```

`after`/`before` accept another field name *or* any `strtotime`-parseable string (`today`, `+1 week`).

### Files, images, mimes

```php
'avatar'   => ['required', 'image', 'mimes:jpg,jpeg,png,webp', 'max:2048', 'dimensions:max_width=2000'],
'document' => ['required', 'file', 'mimetypes:application/pdf', 'max:10240'],
```

- **`image`** — the file must be jpg, jpeg, png, bmp, gif, or webp. **As of Laravel 11, `svg` is NOT allowed by default** (SVG can carry XSS payloads). If you genuinely need SVGs, opt in explicitly with `image:allow_svg`. (In Laravel 10 and earlier, `svg` *was* part of the default `image` set — a frequent footgun.)
- **`mimes:...`** — checks by *extension-derived* MIME guess.
- **`mimetypes:...`** — checks the actual MIME of the file contents (stronger; harder to spoof).
- **`max:2048`** — size in **kilobytes**.
- **`dimensions:...`** — `min_width`, `max_width`, `ratio=3/2`, etc.

> **Laravel 11/12 modern idiom:** use the fluent `File` rule object instead of string rules for files:
> ```php
> use Illuminate\Validation\Rules\File;
> use Illuminate\Validation\Rule;
>
> // File::image() takes an $allowSvg flag (default false) and chains size/type helpers.
> // It does NOT have a ->dimensions() method — apply dimensions as a SEPARATE rule
> // in the array via Rule::dimensions() (or the string 'dimensions:...').
> 'avatar' => [
>     'required',
>     File::image()->max(2 * 1024),                 // max() is in kilobytes; also accepts '2mb'
>     Rule::dimensions()->maxWidth(2000)->maxHeight(2000),
> ],
> 'doc'    => ['required', File::types(['pdf'])->max('10mb')],
> ```
> The `max()`/`min()`/`size()` helpers accept either an integer count of **kilobytes** or a human string like `'500kb'`, `'2mb'`, `'1gb'`.

---

## 7. Arrays and nested data — dot and `*` notation

Real payloads are rarely flat. Laravel uses **dot notation** for nested keys and the **`*` wildcard** to apply a rule to every element of an array.

Given this JSON body:

```json
{
  "user": { "name": "Ada", "email": "ada@example.com" },
  "tags": ["php", "laravel"],
  "items": [
    { "sku": "A1", "qty": 2 },
    { "sku": "B2", "qty": 5 }
  ]
}
```

You validate it like this:

```php
$request->validate([
    // Nested object via dot notation
    'user'         => ['required', 'array'],
    'user.name'    => ['required', 'string', 'max:100'],
    'user.email'   => ['required', 'email'],

    // Flat array of scalars
    'tags'         => ['required', 'array', 'min:1'],
    'tags.*'       => ['string', 'distinct', 'max:30'],

    // Array of objects via *
    'items'        => ['required', 'array'],
    'items.*.sku'  => ['required', 'string'],
    'items.*.qty'  => ['required', 'integer', 'min:1'],
]);
```

- **`tags.*`** applies its rules to *each* element of `tags`.
- **`items.*.qty`** applies to the `qty` key of *every* object in `items`.
- **`distinct`** ensures no duplicate values within the array (great for tag lists).

### Per-element messages with the wildcard

```php
public function messages(): array
{
    return [
        'items.*.qty.min' => 'Each item must have a quantity of at least 1.',
        // Reference a specific index in the default English message via :index / :position
    ];
}
```

In rule logic and messages you can use `:index` (0-based) and `:position` (1-based) placeholders to point at the offending element.

### `array` rule with allowed keys (Laravel 9+)

```php
// Reject unexpected keys inside "user" — only name/email permitted:
'user' => ['array:name,email'],
```

---

## 8. Conditional validation

### `required_*` family

```php
'payment_method' => ['required', Rule::in(['card', 'paypal'])],
'card_number'    => ['required_if:payment_method,card', 'digits:16'],
'paypal_email'   => ['required_if:payment_method,paypal', 'email'],
'reason'         => ['required_unless:status,approved', 'string'],
'shipping'       => ['required_with:billing'],          // required if billing present
'coupon'         => ['required_without:gift_card'],
'b'              => ['required_with_all:a,c'],
```

### `exclude_*` — drop a field conditionally

```php
// If "has_company" is false, "company_name" is excluded entirely
// (won't appear in validated(), won't be validated):
'company_name' => ['exclude_if:has_company,false', 'required', 'string'],
'tax_id'       => ['exclude_unless:country,US', 'required'],
```

This is the clean way to make sure irrelevant fields never leak into `validated()`.

### The `sometimes()` method — complex conditions

For logic that's too rich for the string rules, add rules conditionally on the `Validator`:

```php
$validator = Validator::make($data, [
    'email' => ['required', 'email'],
]);

// Apply the second arg's rules to 'reason' ONLY when the closure returns true.
$validator->sometimes('reason', ['required', 'max:500'], function ($input) {
    return (int) $input->games >= 100;
});

// Conditionally validate every element of an array (Laravel 9+):
$validator->sometimes('items.*.discount', ['required', 'numeric'], function ($input, $item) {
    return $item->on_sale ?? false;
});
```

---

## 9. Custom rules

When the built-in rules can't express your business logic, you have three escalating options.

### Option A — inline closure (one-off)

```php
'token' => [
    'required',
    function (string $attribute, mixed $value, \Closure $fail) {
        if (! str_starts_with($value, 'tok_')) {
            $fail("The {$attribute} must start with 'tok_'.");
        }
    },
],
```

The closure receives the attribute name, the value, and a `$fail` callback. Call `$fail(...)` (optionally chaining `->translate()`) to register an error; do nothing to pass.

### Option B — invokable rule object (reusable, the modern default)

```bash
php artisan make:rule Uppercase
# Laravel 12 generates a class implementing ValidationRule
```

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (strtoupper((string) $value) !== $value) {
            $fail('The :attribute must be uppercase.');
        }
    }
}
```

Use it:

```php
use App\Rules\Uppercase;

'sku' => ['required', new Uppercase()],
```

> **Version note:** The `ValidationRule` interface with the single `validate()` method is the form used in Laravel 10 (≥10.x), 11, and 12. **Laravel 9 and earlier** used the older `Illuminate\Contracts\Validation\Rule` interface, which required separate `passes()` and `message()` methods. The `make:rule --invokable` flag was a transitional generation and **no longer exists in Laravel 12** — the only flags now are `--force` and `--implicit`. On Laravel 12 just run `php artisan make:rule X` and you'll get the `ValidationRule` form. Use `make:rule X --implicit` for an *implicit* rule (one that still runs even when the field is absent/empty, e.g. a `required`-style custom rule).

#### Rules that need other fields or DI

```php
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Contracts\Validation\DataAwareRule;
use Illuminate\Contracts\Validation\ValidatorAwareRule;

class MatchesConfirmation implements ValidationRule, DataAwareRule
{
    protected array $data = [];

    public function setData(array $data): static
    {
        $this->data = $data;
        return $this;
    }

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if ($value !== ($this->data['confirm_'.$attribute] ?? null)) {
            $fail('The :attribute does not match its confirmation.');
        }
    }
}
```

Implement `DataAwareRule` to get all input via `setData()`, or `ValidatorAwareRule` to get the `Validator` instance.

### Option C — `Rule` builder objects

For DB-style and enum rules you've already seen `Rule::unique`, `Rule::exists`, `Rule::in`, `Rule::enum`. There's also `Rule::when()` for conditional rule sets inline:

```php
'discount' => Rule::when($order->isVip(), ['required', 'numeric'], ['prohibited']),
```

---

## 10. `bail` and stopping on first failure

By default Laravel runs *all* rules for a field and reports every failure. Sometimes you want to stop a field at its first error (e.g. don't run an expensive `unique` DB query if the value isn't even a valid email):

```php
'email' => ['bail', 'required', 'email', Rule::unique('users')],
```

With `bail`, if `email` fails the format check, the `unique` query never runs. `bail` only affects the field it's attached to.

To stop on the **first failure across the entire request**, set `$stopOnFirstFailure` in a Form Request, or call `->stopOnFirstFailure()` on a manual validator:

```php
// In a Form Request:
protected $stopOnFirstFailure = true;

// Manual:
Validator::make($data, $rules)->stopOnFirstFailure()->validate();
```

---

## 11. APIs: the 422 JSON error format

When the incoming request **expects JSON** (`Accept: application/json`, or it's an XHR/`/api` route), a `ValidationException` is rendered as **HTTP 422 Unprocessable Entity** with this exact shape:

```json
{
  "message": "The email field is required. (and 1 more error)",
  "errors": {
    "email": [
      "The email field is required."
    ],
    "password": [
      "The password field must be at least 8 characters."
    ]
  }
}
```

- **`message`** — a human-readable summary (the first error, plus a count).
- **`errors`** — an object keyed by field name; each value is an **array** of messages (a field can fail multiple rules).

Front-end frameworks (Inertia, Vue, React with Axios) are built to consume exactly this structure. Your job is simply to validate — Laravel produces the 422 automatically.

> **How "expects JSON" is decided:** Laravel checks `$request->expectsJson()`, which is true if the request is AJAX (`X-Requested-With: XMLHttpRequest`) or its `Accept` header prefers JSON over HTML. Routes in `routes/api.php` are stateless and almost always trigger the JSON branch.

### Customizing the response

Override `failedValidation()` in a Form Request to throw your own response:

```php
use Illuminate\Http\Exceptions\HttpResponseException;
use Illuminate\Contracts\Validation\Validator;

protected function failedValidation(Validator $validator): void
{
    throw new HttpResponseException(
        response()->json([
            'ok'     => false,
            'errors' => $validator->errors(),
        ], 422)
    );
}
```

> **Laravel 11/12 note:** global exception customization moved to `bootstrap/app.php` via `->withExceptions(function ($exceptions) { ... })`, replacing the old `app/Exceptions/Handler.php`. You can use `$exceptions->render(...)` there to reshape all `ValidationException`s app-wide.

---

## 12. `validated()` vs `all()` — the interview favorite

This distinction prevents mass-assignment bugs:

| | `$request->all()` / `$request->input()` | `$request->validated()` |
|---|---|---|
| Returns | **every** field the client sent | **only** fields that had validation rules |
| Trust level | untrusted, includes extras | trusted, rule-covered subset |
| Use with `Model::create()` | **dangerous** — can mass-assign unexpected fields | **safe** |

```php
// Client posts: { "title": "Hi", "body": "...", "is_admin": true }
// rules() covers only title + body.

Post::create($request->all());        // BAD: tries to set is_admin too
Post::create($request->validated());  // GOOD: only title + body
```

Refine further with `safe()`:

```php
$request->safe()->only(['title', 'body']);
$request->safe()->except(['captcha']);
$request->safe()->merge(['author_id' => $request->user()->id]);
```

> Even with Eloquent's `$fillable`/`$guarded` protecting you, `validated()` is the defense-in-depth habit interviewers look for.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `nullable` on optional typed fields.**
   `'published_at' => ['date']` will **fail** when the client sends `null` or an empty string, throwing a confusing "is not a valid date" error.
   **Fix:** `'published_at' => ['nullable', 'date']`. Use `nullable` whenever empty/`null` is a legitimate value for a typed rule.

2. **Using `Model::create($request->all())`.**
   Passing *all* input lets an attacker set columns you never intended (mass assignment), and pulls in `_token`, `password_confirmation`, etc.
   **Fix:** use `$request->validated()` (only rule-covered keys) and rely on `$fillable` as a second layer.

3. **`unique` rule fails on the user's own record during updates.**
   `Rule::unique('users', 'email')` will report a collision with the row being edited.
   **Fix:** ignore the current record: `Rule::unique('users', 'email')->ignore($user->id)` — and prefer passing the model/ID, never raw user input, to avoid SQL injection via `ignore`.

4. **Pipe-string rules breaking on regex / passing model objects.**
   `'code' => 'required|regex:/a|b/'` — the parser splits on the `|` *inside* the regex, producing garbage rules. Strings also can't hold `Rule::unique(...)` objects.
   **Fix:** always use the array syntax for anything non-trivial: `['required', 'regex:/a|b/']`.

5. **Expecting `$errors` to exist outside the `web` middleware group.**
   The `$errors` Blade variable is shared by `ShareErrorsFromSession`, which lives in the `web` group. API routes (no session) don't get it — and shouldn't, since they return JSON.
   **Fix:** for web forms ensure the route is in the `web` group; for APIs consume the 422 JSON instead.

6. **`max:2048` on a file thinking it's bytes.**
   File `max`/`min` are in **kilobytes**, not bytes. `max:2048` is a 2 MB limit, not 2 KB.
   **Fix:** remember the unit. String rules take a plain integer of KB (`'file', 'max:2048'`) — you cannot put a PHP expression like `2 * 1024` inside a *string* rule. The fluent `File::image()->max(...)` builder accepts an integer of KB **or** a human string such as `'2mb'` / `'500kb'` (L11+).

   **Also (L11+):** `svg` is no longer part of the default `image` rule — `'image'` now rejects SVGs, and you must opt in with `image:allow_svg` (or `File::image(allowSvg: true)`).

7. **Putting cross-field checks in `rules()` with closures that reference siblings incorrectly.**
   A closure rule on one field can't easily see another field's value cleanly.
   **Fix:** use `after()` (L11+) / `withValidator()` for cross-field logic, or `required_if`/`gt:other_field` style rules.

---

## ✅ Best Practices

- **Validate at the boundary.** Use Form Requests so the controller body assumes clean data. One Form Request per action (`StorePostRequest`, `UpdatePostRequest`).
- **Always consume `validated()`**, never `all()`, when persisting. Combine with `$fillable`.
- **Prefer array rule syntax** and **rule objects** (`Rule::unique`, `Rule::enum`, `File::image`) over pipe strings.
- **Use `prepareForValidation()`** to normalize input (trim, lowercase emails, derive slugs) so your rules operate on clean data.
- **Reach for `sometimes` / `exclude_*`** for PATCH-style partial updates instead of writing branching logic in the controller.
- **Make custom rules invokable classes** (`ValidationRule`) once you reuse logic, not copy-pasted closures.
- **Keep messages in `lang/`** for i18n; only override per-field messages in `messages()` when you need context the default can't give.
- **Set `bail`** on fields where later rules are expensive (DB lookups) and pointless if earlier ones fail.
- **Test Form Requests in isolation** — they're plain classes; you can instantiate, set data, and assert on `rules()` / authorization.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between `$request->all()` and `$request->validated()`?**
`all()` returns *every* field the client submitted, including ones you never validated (a vector for mass assignment). `validated()` returns *only* the keys that appeared in your rules — the trusted subset. Always persist with `validated()`.

**Q2. Where do validation errors come from in a Blade view, and how is `$errors` always available?**
On a failed web request, Laravel redirects back and flashes the errors to the session. The `ShareErrorsFromSession` middleware (in the `web` group) pulls them out and shares a `ViewErrorBag` as `$errors` with *every* view, so it's never undefined — it's just empty when there are no errors.

**Q3. How does the same `validate()` call serve both web forms and APIs?**
It throws a `ValidationException`. The exception handler inspects `$request->expectsJson()`: if true (AJAX or `Accept: application/json`), it renders a 422 JSON body; otherwise it issues a 302 redirect back with errors and old input flashed to the session.

**Q4. `nullable` vs `sometimes` vs `required` — explain each.**
`required`: must be present and non-empty. `nullable`: may be `null`/empty and the *other* rules are skipped when it is. `sometimes`: the field's rules run *only if the key is present* in the input (ideal for partial/PATCH updates). They solve different problems and are often combined: `['sometimes', 'nullable', 'string']`.

**Q5. How do you make a `unique` rule work correctly on an update form? (under the hood)**
`unique` issues a `SELECT ... WHERE column = ? LIMIT 1` against the table. On update, the row being edited matches itself, so you chain `->ignore($model->id)`, which adds `AND id <> ?` to the query. You can also add `->where(...)` for scoping. Pass the model or its ID — never raw user input — to avoid injecting into the ignore clause.

**Q6. What does `bail` do, and where does it apply?**
`bail` stops running further rules **for that single field** after the first failure. It's per-field, used to skip expensive checks (like a DB `unique` query) when a cheaper earlier rule (like `email` format) already failed. To stop the *whole* validator on the first failure, use `stopOnFirstFailure()` / `$stopOnFirstFailure = true`.

**Q7. How do you write a reusable custom rule in Laravel 12?**
Generate `php artisan make:rule X`, which implements `Illuminate\Contracts\Validation\ValidationRule` with a single `validate(string $attribute, mixed $value, Closure $fail)` method. Call `$fail('...')` to register an error. Implement `DataAwareRule`/`ValidatorAwareRule` if you need the full input or the validator. (Pre-L10 used the two-method `Rule` interface with `passes()`/`message()`.)

**Q8. Show how you'd validate an array of line items where each must have a positive quantity.**
```php
'items'        => ['required', 'array', 'min:1'],
'items.*.sku'  => ['required', 'string'],
'items.*.qty'  => ['required', 'integer', 'min:1'],
```
The `*` wildcard applies the rules to every element; `:index`/`:position` placeholders let messages reference the offending row.

**Q9. What is the exact JSON shape of a 422 validation response, and why does it matter?**
A top-level `message` (human summary) and an `errors` object mapping each field to an **array** of messages. It matters because client libraries (Axios/Inertia/Vue) are built to parse exactly this structure to render field-level errors — so you should not invent your own shape unless you also update the client.

**Q10. Where would you normalize input before rules run, and where would you do cross-field checks?**
Normalize (trim/lowercase/derive) in `prepareForValidation()` via `$this->merge([...])` — it runs *before* rules. Cross-field checks go in `after()` (L11+) or `withValidator()` (all versions), where you add errors with `$validator->errors()->add(...)`.

---

## 📋 Quick Reference / Cheat Sheet

```php
// --- Three ways to validate ---
$request->validate([...]);                       // controller helper (throws on fail)
StorePostRequest $request;                        // Form Request (auto-validates)
Validator::make($data, $rules)->validate();       // manual factory

// --- Getting clean data ---
$request->validated();                            // only rule-covered keys
$request->safe()->only([...]);                    // subset
$request->safe()->merge([...]);                   // add derived values
$validator->validated();                          // from manual validator

// --- Presence ---
required  nullable  sometimes  filled  present  prohibited  prohibited_if

// --- Types ---
string  integer  numeric  boolean  array  json  decimal:2

// --- Strings ---
email:rfc,dns  url  alpha  alpha_num  alpha_dash  uuid  ulid  regex:/.../

// --- Size (meaning depends on type) ---
min:n  max:n  between:a,b  size:n  digits:n  digits_between:a,b

// --- Sets / enums ---
in:a,b,c  not_in:x,y  Rule::in([...])  Rule::enum(Status::class)

// --- Database ---
Rule::exists('table','col')
Rule::unique('table','col')->ignore($id)->where(fn($q) => ...)

// --- Confirmation / matching ---
confirmed                                          // expects <field>_confirmation
confirmed:other_field                              // custom confirmation field (L11+)
same:other  different:other  gt:other  gte:other  lt:other  lte:other

// --- Dates ---
date  date_format:Y-m-d  after:starts_at  before:today  after_or_equal:tomorrow

// --- Files ---
file  image  image:allow_svg  mimes:jpg,png  mimetypes:image/png  max:2048(KB)  dimensions:max_width=2000
File::image()->max('2mb')                          // fluent (L11+); File has NO ->dimensions()
Rule::dimensions()->maxWidth(2000)->ratio(3/2)     // dimensions are a SEPARATE rule

// --- Conditional ---
required_if:other,val  required_unless  required_with  required_without
exclude_if:other,val   exclude_unless    Rule::when($cond, $then, $else)
$validator->sometimes('f', $rules, fn($input) => ...);

// --- Control ---
bail                                               // stop this field on first fail
$stopOnFirstFailure = true;                        // stop whole request on first fail

// --- Nested ---
'user.email'  'tags.*'  'items.*.qty'  'array:name,email'
```

```blade
{{-- Blade error display --}}
@error('field') <span>{{ $message }}</span> @enderror
{{ old('field') }}
@if ($errors->any()) ... @endif
{{ $errors->first('field') }}
@foreach ($errors->get('field') as $m) {{ $m }} @endforeach
```

```bash
# Artisan
php artisan make:request StorePostRequest
php artisan make:rule Uppercase            # implements ValidationRule (validate() method)
php artisan make:rule ActiveUser --implicit  # implicit rule (runs even when field is empty/absent)
```

---

## 🧪 Mini Exercises

1. **Registration form.** Build a `RegisterUserRequest` that validates `name` (required, 2–50 chars), `email` (required, valid, unique on `users`), `password` (required, min 8, confirmed), and `terms` (must be accepted). In `prepareForValidation()`, lowercase and trim the email. Display every error in a Blade form and repopulate `name`/`email` with `old()`.

2. **Conditional checkout.** Validate a payment form where `payment_method` is one of `card`/`paypal`. When it's `card`, require `card_number` (exactly 16 digits) and `cvc` (3–4 digits); when it's `paypal`, require a valid `paypal_email`. Use the `required_if` rules — then rewrite the same logic using `$validator->sometimes(...)`.

3. **Nested order payload.** Given a JSON body with `customer.name`, `customer.email`, and an `items` array of `{ sku, qty }`, write rules that enforce: at least one item, every `sku` is a string and `distinct`, and every `qty` is an integer ≥ 1. Add a custom per-element message for the `qty` minimum.

4. **Custom rule.** Create an invokable `ValidationRule` class `StrongPassword` that fails unless the value has at least one uppercase letter, one digit, and one symbol. Use it in a Form Request and return the proper 422 JSON shape for an API client.

5. **Update vs create.** Create both `StoreProductRequest` and `UpdateProductRequest`. The store request requires `sku` to be unique; the update request must allow the product to keep its own `sku` (use `->ignore()`), and should use `sometimes` so a PATCH that omits a field doesn't wipe it. Confirm that persisting with `validated()` ignores an injected `is_featured` field that has no rule.
