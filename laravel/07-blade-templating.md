# Blade Templating

Blade is Laravel's built-in templating engine. It lets you write HTML mixed with concise, expressive directives instead of raw `<?php ?>` tags, while compiling down to plain PHP for zero runtime overhead. This module takes you from echoing a variable to building reusable, attribute-aware components used across an entire application.

> **Templating engine** = a tool that turns *templates* (files mixing static markup with placeholders/logic) into final output (usually HTML) by injecting data. Blade is the engine; `.blade.php` files are the templates.

---

## **What you'll learn**

- What Blade is, *why* it exists, and how it **compiles to cached PHP** behind the scenes.
- Echoing data: auto-escaped `{{ }}` vs. unescaped `{!! !!}` (and the XSS trap), plus escaping Blade itself with `@`.
- Control-flow directives: `@if`, `@unless`, `@isset`, `@empty`, `@switch`, all loop forms, and the magic `$loop` variable.
- Code reuse: includes (`@include` family), traditional layouts (`@extends`/`@section`/`@yield`), and **stacks**.
- **Components** — the modern reuse primitive: class-based & anonymous, props, slots, named slots, and the attributes bag.
- Form & auth helpers: `@csrf`, `@method`, `@error`, `@auth`/`@guest`, `@can`, and conditional `@class`/`@style`.
- Passing data to views, **sharing globally**, and **view composers**.
- When Blade is the right tool vs. a front-end framework like Vue/React/Livewire.

---

## 1. What Blade Is and Why It Exists

Before templating engines, PHP views looked like this — a tangle of `<?php ?>` islands inside HTML:

```php
<!-- The "bad old days" -->
<ul>
<?php foreach ($users as $user): ?>
    <li><?php echo htmlspecialchars($user->name, ENT_QUOTES); ?></li>
<?php endforeach; ?>
</ul>
```

Three problems: it's **verbose**, it's **easy to forget escaping** (a security hole), and the logic visually drowns the markup. Blade fixes all three:

```blade
<ul>
    @foreach ($users as $user)
        <li>{{ $user->name }}</li>
    @endforeach
</ul>
```

Cleaner, and `{{ }}` escapes output **automatically**. The key insight that trips up beginners:

> Blade adds **no runtime cost**. Directives are not interpreted on every request. Blade *compiles* each template to a plain PHP file the first time it's needed, caches it, and afterward serves the compiled PHP directly. Blade is a build step disguised as a templating language.

### Where views live

By convention, Blade templates live in `resources/views/` and end in `.blade.php`. You render one from a controller or route with the `view()` helper:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;

Route::get('/welcome', function () {
    return view('welcome', ['name' => 'Ada']);
});
```

`view('welcome')` maps to `resources/views/welcome.blade.php`. Dots are directory separators: `view('admin.users.index')` → `resources/views/admin/users/index.blade.php`.

---

## 2. How Blade Works Under the Hood (Compilation & Caching)

This is the single most important mental model, and a common interview question.

When a `.blade.php` file is rendered, Laravel's `BladeCompiler` runs the raw template through a series of regex/token transformations that rewrite each directive into PHP. The result is written to a **compiled cache** file (named after a hash of the view's path — Laravel 12 uses `hash('xxh128', ...)`; older versions used `sha1`/`md5`) under:

```
storage/framework/views/
```

On subsequent requests, Blade checks whether the source `.blade.php` file's modification time is newer than the compiled file. If not, it simply `include`s the cached PHP — **no recompilation, no Blade parsing**.

A template like:

```blade
<p>Hello, {{ $name }}!</p>
@if ($admin)
    <span>Admin</span>
@endif
```

compiles to roughly:

```php
<p>Hello, <?php echo e($name); ?>!</p>
<?php if ($admin): ?>
    <span>Admin</span>
<?php endif; ?>
```

Notice two things:

1. `{{ $name }}` became `<?php echo e($name); ?>`. The `e()` helper is `htmlspecialchars(..., ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8', double_encode: true)` — **that's** the auto-escaping.
2. `@if` became a real `if`. There is no Blade interpreter at request time.

You can pre-compile all views during deployment so the first request isn't slowed by compilation:

```bash
php artisan view:cache    # compile every view ahead of time
php artisan view:clear    # delete all compiled views (e.g., after a bad cache)
```

> **Laravel 10/11/12 note:** behavior here is stable across versions. The compiled-view location and `view:cache`/`view:clear` commands are the same. In Laravel 12 the default skeleton still ships these.

---

## 3. Echoing Data

### 3.1 Auto-escaped: `{{ }}`

The default and the one you'll use 95% of the time. It escapes HTML, protecting against **XSS** (cross-site scripting — when attacker-controlled text containing `<script>` is rendered as live markup).

```blade
{{ $user->name }}
{{-- If $user->name is '<b>Ada</b>', output is: &lt;b&gt;Ada&lt;/b&gt; --}}
```

You can put any PHP expression inside:

```blade
{{ $count > 0 ? 'In stock' : 'Sold out' }}
{{ strtoupper($name) }}
{{ $price + $tax }}
```

**The `??` "or default" idiom** — Blade does NOT silently swallow undefined variables, so use null coalescing:

```blade
{{ $name ?? 'Guest' }}
{{-- Output if $name is undefined/null: Guest --}}
```

> Historically Blade had a `{{ $name or 'Guest' }}` syntax. It was **removed**; use PHP's `??` operator instead.

### 3.2 Unescaped: `{!! !!}` — handle with extreme care

Renders the value **raw**, with no escaping. Necessary when the value is *trusted* HTML you generated yourself (e.g., Markdown you converted server-side).

```blade
{!! $post->renderedHtml !!}
```

> **XSS WARNING:** Never put user-supplied content inside `{!! !!}`. If a user can set `$comment` to `<script>steal()</script>`, then `{!! $comment !!}` executes it in every visitor's browser. Use `{{ }}` for anything a user can influence.

### 3.3 Escaping Blade itself: `@`

Sometimes you want to output literal `{{ ... }}` — e.g., when serving a front-end framework template (Vue/Angular also use `{{ }}`). Prefix with `@` to tell Blade "leave this alone":

```blade
@{{ name }}
{{-- Output sent to browser: {{ name }} --}}  (Blade does not evaluate it)
```

For larger blocks, use the `@verbatim` directive so you don't `@`-escape every line:

```blade
@verbatim
    <div id="app">
        Hello, {{ vueName }}.  {{-- left untouched for Vue --}}
    </div>
@endverbatim
```

### 3.4 JSON for JavaScript

To safely pass PHP data into a `<script>` tag, use the `@json` directive (it calls `json_encode` with safe flags) or the `Js::from()` fluent helper:

```blade
<script>
    const app = @json($settings);
    const user = {{ Illuminate\Support\Js::from($user) }};
</script>
```

```php
// $settings = ['theme' => 'dark', 'beta' => true];
// Output: const app = {"theme":"dark","beta":true};
```

`@json` accepts `json_encode` flags as a second argument: `@json($data, JSON_PRETTY_PRINT)`.

---

## 4. Control-Flow Directives

### 4.1 Conditionals: `@if` / `@elseif` / `@else`

```blade
@if ($orders->count() === 1)
    You have one order.
@elseif ($orders->count() > 1)
    You have multiple orders.
@else
    You have no orders.
@endif
```

### 4.2 `@unless` — the inverse of `@if`

Reads naturally when you mean "if NOT":

```blade
@unless (Auth::check())
    <a href="/login">Please log in</a>
@endunless
{{-- Equivalent to: @if (! Auth::check()) ... @endif --}}
```

### 4.3 `@isset` and `@empty`

- `@isset($var)` → true if the variable is set and not null.
- `@empty($var)` → true if the variable is "empty" (`null`, `''`, `0`, `'0'`, `[]`, `false`).

```blade
@isset($record)
    {{-- $record is defined and not null --}}
    {{ $record->title }}
@endisset

@empty($comments)
    No comments yet.
@endempty
```

### 4.4 `@switch`

```blade
@switch($role)
    @case('admin')
        Full access.
        @break

    @case('editor')
        Can edit posts.
        @break

    @default
        Read-only.
@endswitch
```

> Don't forget `@break`. Like native PHP `switch`, cases fall through without it.

### 4.5 Authentication & authorization shortcuts

```blade
@auth
    Welcome back, {{ Auth::user()->name }}!
@endauth

@guest
    <a href="/register">Sign up</a>
@endguest
```

You can target a specific **guard** (an auth driver, e.g. `admin`):

```blade
@auth('admin')
    Admin tools
@endauth
```

Authorization via **gates/policies** (rules deciding if a user *may* do something):

```blade
@can('update', $post)
    <a href="/posts/{{ $post->id }}/edit">Edit</a>
@elsecan('view', $post)
    <a href="/posts/{{ $post->id }}">View</a>
@endcan

@cannot('delete', $post)
    <span>You can't delete this.</span>
@endcannot
```

There are also `@canany(['update', 'delete'], $post)` for "any of these abilities", plus the matching `@elsecannot` and `@elsecanany` chain directives, and environment helpers `@production` and `@env('staging')` (the latter also accepts an array: `@env(['staging', 'production'])`). For an action with no model instance, pass the class name: `@can('create', App\Models\Post::class)`.

---

## 5. Loops and the `$loop` Variable

Blade mirrors PHP's loop constructs:

```blade
@for ($i = 0; $i < 5; $i++)
    Item {{ $i }}
@endfor

@foreach ($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@while ($queue->isNotEmpty())
    Processing {{ $queue->pop() }}
@endwhile
```

### 5.1 `@forelse` — foreach with an empty fallback

A Blade convenience with no native PHP equivalent. It loops, but if the collection is empty it runs the `@empty` block instead — saving a separate `@if (count(...))` check:

```blade
@forelse ($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>No posts found.</p>
@endforelse
```

### 5.2 The `$loop` variable

Inside any `@foreach`/`@forelse`, Blade injects a `$loop` object with metadata — no manual counters needed:

```blade
@foreach ($users as $user)
    @if ($loop->first)
        <p>The first user is special.</p>
    @endif

    {{ $loop->iteration }}. {{ $user->name }}
    @if ($loop->last) — that's everyone! @endif
@endforeach
```

| Property | Meaning |
|---|---|
| `$loop->index` | 0-based index |
| `$loop->iteration` | 1-based index |
| `$loop->remaining` | Iterations left |
| `$loop->count` | Total items |
| `$loop->first` / `$loop->last` | Bool, first/last iteration |
| `$loop->even` / `$loop->odd` | Bool, alternating rows |
| `$loop->depth` | Nesting level (1 = outermost) |
| `$loop->parent` | The `$loop` of the enclosing loop |

```blade
@foreach ($categories as $category)
    @foreach ($category->products as $product)
        {{-- Parent loop's position --}}
        Category #{{ $loop->parent->iteration }}, product {{ $loop->iteration }}
    @endforeach
@endforeach
```

You can also skip/break with directives:

```blade
@foreach ($users as $user)
    @continue($user->isBanned())   {{-- skip this iteration --}}
    @break($loop->iteration > 10)  {{-- stop after 10 --}}
    <li>{{ $user->name }}</li>
@endforeach
```

### 5.3 Raw PHP: `@php`

For the rare case you need a statement that has no directive. Use sparingly — heavy logic belongs in the controller/view-model, not the template.

```blade
@php
    $total = $items->sum('price');
@endphp

<p>Total: {{ $total }}</p>
```

---

## 6. Including Partials

A **partial** is a small reusable view fragment (e.g., a card, a nav). Pull it in with `@include`:

```blade
{{-- resources/views/partials/alert.blade.php --}}
<div class="alert alert-{{ $type }}">{{ $message }}</div>
```

```blade
@include('partials.alert', ['type' => 'error', 'message' => 'Oops!'])
```

Included partials inherit the parent's variables *plus* any you pass explicitly. Related forms:

```blade
@includeIf('partials.maybe', ['x' => 1])         {{-- include only if the view exists --}}
@includeWhen($user->isAdmin(), 'partials.admin') {{-- include if condition true --}}
@includeUnless($user->isAdmin(), 'partials.upsell')
@includeFirst(['custom.alert', 'partials.alert'], $data) {{-- first that exists --}}
```

### `@each` — render one partial per item

A compact loop-and-include. Syntax: `@each(view, $array, $itemVarName, $emptyView)`:

```blade
@each('partials.user-row', $users, 'user', 'partials.no-users')
{{-- For each $users item, render user-row.blade.php with $user set. --}}
{{-- If $users is empty, render no-users.blade.php instead. --}}
```

> **Performance note:** `@each` re-includes a file per item with isolated scope. For complex/large lists, an inline `@foreach` (which compiles to one tight loop) is often faster. Use `@each` for readability on simple rows.

---

## 7. Layouts: Template Inheritance

The classic way to avoid repeating `<html>`, `<head>`, nav, and footer on every page. You define a **master layout** with "holes," then child views fill them.

### 7.1 `@yield` and `@section`/`@show`

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', 'My App')</title>  {{-- 2nd arg = default --}}
</head>
<body>
    @section('sidebar')
        <nav>Default sidebar</nav>
    @show   {{-- @show = define AND immediately yield this section --}}

    <main>
        @yield('content')
    </main>

    @include('partials.footer')
</body>
</html>
```

A child view **extends** it and provides sections:

```blade
{{-- resources/views/dashboard.blade.php --}}
@extends('layouts.app')

@section('title', 'Dashboard')   {{-- short form: inline value --}}

@section('sidebar')
    @parent   {{-- inject the layout's original sidebar content here --}}
    <a href="/stats">Stats</a>
@endsection

@section('content')
    <h1>Welcome to your dashboard</h1>
@endsection
```

Key directives:

- `@yield('name', $default)` — outputs the named section's content (placeholder, in the layout).
- `@section`/`@endsection` — define content (in the child).
- `@section`/`@show` — define content **and** echo it immediately (in the layout itself, so it has a default).
- `@parent` — within a child section, render the layout's version of that section too.

> **`@endsection` vs `@stop` vs `@show`:** `@endsection` and `@stop` are equivalent — they end a section definition. `@show` ends a section *and* yields it immediately. Use `@show` only in the parent layout where the section both defines a default and is the render point.

### 7.2 Stacks: `@push` / `@stack` / `@prepend`

Stacks let child views and partials *contribute* to a named location in the layout — perfect for page-specific CSS/JS that must land in `<head>` or before `</body>`.

```blade
{{-- layout --}}
<head>
    @stack('styles')
</head>
<body>
    ...
    @stack('scripts')
</body>
```

```blade
{{-- any child view or partial --}}
@push('scripts')
    <script src="/js/charts.js"></script>
@endpush

@push('styles')
    <link rel="stylesheet" href="/css/charts.css">
@endpush
```

Multiple `@push`es to the same stack **accumulate in order**. Use `@prepend` to add to the *front* (e.g., a dependency that must load first), and `@pushOnce` to avoid duplicate pushes when a partial is included many times:

```blade
@prepend('scripts')
    <script src="/js/jquery.js"></script>  {{-- must come before charts.js --}}
@endprepend

@pushOnce('scripts')
    <script src="/js/once.js"></script>   {{-- added at most once per request --}}
@endPushOnce
```

> **Casing gotcha:** the closing directive is `@endPushOnce` (capital P, capital O) — `@endpushonce` will not compile. The same applies to `@prependOnce`/`@endPrependOnce`.

The more general `@once` directive runs a block exactly once per render cycle — handy inside a component rendered in a loop to push its JS only the first time:

```blade
@once
    @push('scripts')
        <script src="/js/component.js"></script>
    @endpush
@endonce
```

---

## 8. Components — The Modern Way to Reuse

Components are the recommended reuse mechanism in modern Laravel (Laravel 7+). They behave like custom HTML tags, accept **props** (typed inputs) and **slots** (content you nest inside), and come in two flavors.

### 8.1 Anonymous components (view-only)

Just a Blade file under `resources/views/components/`. No PHP class needed.

```blade
{{-- resources/views/components/alert.blade.php --}}
<div class="alert alert-{{ $type }}">
    {{ $slot }}
</div>
```

Used via the `x-` prefix (the file name becomes the tag name):

```blade
<x-alert type="error">
    Something went wrong!
</x-alert>
```

Output:

```html
<div class="alert alert-error">
    Something went wrong!
</div>
```

`{{ $slot }}` is the **default slot** — whatever you nest between the open/close tags. Subdirectories use dots: `resources/views/components/forms/input.blade.php` → `<x-forms.input />`.

#### Declaring props with `@props`

Without `@props`, every attribute becomes part of the *attribute bag* (see §8.4). Use `@props` to declare which attributes are real inputs (props) and give defaults:

```blade
{{-- components/alert.blade.php --}}
@props([
    'type' => 'info',         {{-- default value --}}
    'dismissible' => false,
])

<div class="alert alert-{{ $type }}" {{ $attributes }}>
    {{ $slot }}
    @if ($dismissible)
        <button>×</button>
    @endif
</div>
```

```blade
<x-alert type="warning" dismissible class="mt-4">
    Heads up.
</x-alert>
```

Here `type` and `dismissible` are consumed as props; `class="mt-4"` is unknown, so it lands in `$attributes` and is merged onto the `<div>`.

### 8.2 Class-based components (with logic)

When a component needs computed values, dependencies, or methods, back it with a PHP class. Generate one:

```bash
php artisan make:component Alert
```

This creates two files:

```
app/View/Components/Alert.php                 # the class
resources/views/components/alert.blade.php    # its view
```

```php
<?php
// app/View/Components/Alert.php
namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\Contracts\View\View;

class Alert extends Component
{
    // Constructor promotion: public props are auto-available in the view.
    public function __construct(
        public string $type = 'info',
        public string $message = '',
    ) {}

    // Optional helper usable in the view as {{ $iconClass() }} or via $this
    public function iconClass(): string
    {
        return match ($this->type) {
            'error'   => 'icon-x',
            'success' => 'icon-check',
            default   => 'icon-info',
        };
    }

    public function render(): View
    {
        return view('components.alert');
    }
}
```

```blade
{{-- resources/views/components/alert.blade.php --}}
<div class="alert alert-{{ $type }}">
    <i class="{{ $iconClass() }}"></i>
    {{ $message ?: $slot }}
</div>
```

```blade
<x-alert type="success" message="Saved!" />
```

> **PHP 8.4 / version notes:** Constructor property promotion (shown above) works in PHP 8.0+. PHP 8.1+ enables `enum` props (e.g., `public AlertType $type`). Public constructor properties are automatically passed to the view; only public methods/properties are exposed. **Pass enum or object props using `:` binding** (next section), since `type="..."` is always a string.

### 8.3 Passing data: literal vs. bound attributes

- `type="error"` passes the **literal string** `"error"`.
- `:type="$variable"` (note the leading colon) evaluates the value as **PHP** — use it for variables, expressions, arrays, enums, booleans.

```blade
<x-alert :type="$level" :dismissible="true" :tags="['a', 'b']" />
<x-alert type="error" />                       {{-- string "error" --}}
<x-profile :user="$user" :is-admin="$user->isAdmin()" />
```

Attribute names in `kebab-case` map to `camelCase` props: `:is-admin` → `$isAdmin`.

Short attribute syntax (when the prop name equals the variable name):

```blade
<x-profile :$user />   {{-- shorthand for :user="$user" --}}
```

### 8.4 The attributes bag

`$attributes` collects every attribute you *didn't* declare as a prop, so components stay flexible (callers can add `class`, `id`, `data-*`, etc.). The killer feature is `merge`, which **combines** caller classes with component defaults rather than overwriting:

```blade
{{-- components/button.blade.php --}}
@props(['type' => 'button'])

<button type="{{ $type }}" {{ $attributes->merge(['class' => 'btn']) }}>
    {{ $slot }}
</button>
```

```blade
<x-button class="btn-primary" id="save">Save</x-button>
```

Output:

```html
<button type="button" class="btn btn-primary" id="save">Save</button>
```

Note `class` got **merged** (`btn btn-primary`), while `id` was simply appended. Useful attribute-bag methods:

```blade
{{ $attributes->merge(['class' => 'btn']) }}        {{-- merge, class is concatenated --}}
{{ $attributes->class(['btn', 'btn-lg' => $large]) }} {{-- conditional class merge --}}
{{ $attributes->except('data-foo') }}               {{-- all except listed --}}
{{ $attributes->only('class') }}                    {{-- only listed --}}
@if ($attributes->has('disabled')) ... @endif
{{ $attributes->get('id', 'default-id') }}
```

#### Inheriting parent data with `@aware`

A prop passed to a *parent* component is **not** automatically visible to nested child components. When a child needs a value that was only given to its parent, declare it with `@aware`:

```blade
{{-- components/menu.blade.php (parent), called as <x-menu color="purple"> --}}
@props(['color' => 'gray'])
<ul {{ $attributes->merge(['class' => 'bg-'.$color]) }}>{{ $slot }}</ul>

{{-- components/menu/item.blade.php (child) --}}
@aware(['color' => 'gray'])   {{-- pulls $color from the parent component --}}
<li class="text-{{ $color }}">{{ $slot }}</li>
```

> `@aware` only reads attributes **explicitly passed** to the parent in the template; it cannot see a parent prop's default value.

### 8.5 Slots and named slots

The default `{{ $slot }}` holds nested content. **Named slots** let a component have multiple insertion points:

```blade
{{-- components/card.blade.php --}}
<div class="card">
    <div class="card-header">{{ $title }}</div>
    <div class="card-body">{{ $slot }}</div>
    <div class="card-footer">{{ $footer ?? 'Default footer' }}</div>
</div>
```

```blade
<x-card>
    <x-slot:title>
        Monthly Report
    </x-slot>

    The body goes in the default slot.

    <x-slot:footer>
        <button>Download</button>
    </x-slot>
</x-card>
```

> **Syntax note:** modern Laravel (10/11/12) uses `<x-slot:name>` to open and a plain `</x-slot>` to close (the closing tag does **not** repeat the name — `</x-slot:title>` is wrong). The older `<x-slot name="...">…</x-slot>` form still works but the colon form is preferred. Slots also carry their **own** attribute bag: `<x-slot:title class="text-lg">` then `{{ $title->attributes->merge([...]) }}` inside the component.

### 8.6 Inline & dynamic components

Tiny class-based component whose template lives in `render()`:

```php
public function render(): \Closure|string
{
    return <<<'blade'
        <span class="badge">{{ $slot }}</span>
    blade;
}
```

Render a component whose name is decided at runtime with `<x-dynamic-component>`:

```blade
<x-dynamic-component :component="$componentName" :type="$type" />
```

---

## 9. Forms, Validation & Conditional Attributes

### 9.1 `@csrf` and `@method`

Every state-changing HTML form needs a **CSRF token** (a per-session secret that proves the request came from your own page, blocking cross-site request forgery). `@csrf` emits the hidden field. HTML forms only support `GET`/`POST`, so `@method` spoofs `PUT`/`PATCH`/`DELETE` for Laravel's router:

```blade
<form method="POST" action="/posts/{{ $post->id }}">
    @csrf
    @method('PUT')

    <input name="title" value="{{ old('title', $post->title) }}">
    <button>Update</button>
</form>
```

```html
<!-- @csrf renders: -->
<input type="hidden" name="_token" value="aBcD...random...">
<!-- @method('PUT') renders: -->
<input type="hidden" name="_method" value="PUT">
```

`old('title', $post->title)` repopulates the field with the previous submission on a validation failure (falling back to the model value) — essential for good UX.

### 9.2 `@error` — show validation messages

`@error('field')` runs its block only if that field has a validation error, exposing `$message`:

```blade
<input name="email" value="{{ old('email') }}"
       class="@error('email') is-invalid @enderror">

@error('email')
    <span class="text-red-500">{{ $message }}</span>
@enderror
```

You can scope to a specific error bag (for multiple forms on a page): `@error('email', 'login')`.

### 9.3 Conditional `@class` and `@style`

These compile an array into a `class`/`style` string, including keys only when their value is truthy. Cleaner than ternary soup:

```blade
<div @class([
    'card',
    'card-active' => $isActive,
    'card-error'  => $hasError,
    'text-muted'  => ! $isActive,
])>...</div>
```

```html
<!-- If $isActive=true, $hasError=false: -->
<div class="card card-active">...</div>
```

```blade
<span @style([
    'color: red'              => $isUrgent,
    'font-weight: bold; font-size: 1.2rem',  {{-- always applied (no condition) --}}
])>...</span>
```

There are matching `@checked`, `@selected`, `@disabled`, `@readonly`, and `@required` directives for form controls:

```blade
<input type="checkbox" @checked(old('subscribed', $user->subscribed))>
<option value="us" @selected($country === 'us')>USA</option>
<button @disabled($form->isProcessing())>Submit</button>
```

---

## 10. Comments

Blade comments are stripped at compile time — they **never reach the browser** (unlike HTML `<!-- -->` comments):

```blade
{{-- This is a Blade comment; invisible in the HTML source. --}}
<!-- This HTML comment IS sent to the browser. -->
```

Use Blade comments for notes that shouldn't leak (TODOs, internal logic explanations).

---

## 11. Passing Data to Views

### 11.1 From a controller — three styles

```php
// 1. Array as second argument
return view('profile', ['user' => $user, 'tab' => 'settings']);

// 2. with() chaining (each ->with('key', $value))
return view('profile')->with('user', $user)->with('tab', 'settings');

// 3. compact() — pulls variables of matching names into an array
$user = User::find(1);
$tab  = 'settings';
return view('profile', compact('user', 'tab'));
```

All three make `$user` and `$tab` available in the template.

### 11.2 Sharing data with ALL views: `View::share`

To expose a variable to **every** view (e.g., site settings), share it from a service provider's `boot()` method:

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\View;

public function boot(): void
{
    View::share('appName', config('app.name'));
}
```

Now `{{ $appName }}` works in any template. Use sparingly — it's a global.

### 11.3 View composers — data for *specific* views

A **view composer** is a callback (or class) that runs every time a particular view (or set of views) renders, attaching data automatically. This keeps controllers from repeatedly fetching the same sidebar/nav data.

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\View;
use App\Models\Category;

public function boot(): void
{
    // Closure composer for one view
    View::composer('partials.sidebar', function ($view) {
        $view->with('categories', Category::all());
    });

    // Class-based composer for multiple views (wildcards allowed)
    View::composer(['dashboard', 'admin.*'], \App\View\Composers\StatsComposer::class);

    // Same data for EVERY view: View::composer('*', ...)
}
```

A class composer:

```php
<?php
// app/View/Composers/StatsComposer.php
namespace App\View\Composers;

use Illuminate\View\View;
use App\Services\StatsService;

class StatsComposer
{
    public function __construct(private StatsService $stats) {}  // DI works here

    public function compose(View $view): void
    {
        $view->with('unreadCount', $this->stats->unreadFor(auth()->user()));
    }
}
```

> **Composer vs. share:** `share` = one value for all views, set once. `composer` = data attached just-in-time only when the matched view actually renders (and can run DB queries lazily, with dependency injection). Reach for composers when the data is view-specific and possibly expensive.

---

## 12. Blade vs. Front-End Frameworks

A question that *will* come up: "Why use Blade when React/Vue exist?"

- **Blade is server-side.** It runs on the server, produces final HTML, and ships zero JS by default. Great SEO, fast first paint, simple mental model, no build toolchain required for the markup itself. Best for **content-heavy, mostly-static, or form-driven** apps.
- **Vue/React are client-side (SPA).** They render in the browser and excel at **highly interactive, stateful UIs** (drag-and-drop, real-time dashboards) but cost JS payload, complexity, and SEO/SSR headaches.
- **Livewire** sits in the middle: you write Blade + PHP, and it transparently syncs state over AJAX, giving SPA-like interactivity without writing much JS. In Laravel 12, the **Livewire + Volt** starter kit is a first-class option.
- **Inertia.js** lets you use Vue/React components as your "views" while keeping Laravel routing/controllers — no separate API needed.

Rule of thumb: **start with Blade.** Add Livewire for interactive islands; reach for a full SPA only when the UI genuinely demands client-side state. They also **compose**: Blade can render the page shell and mount a Vue/React component for one interactive widget.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `{!! !!}` on user input → XSS hole.**
   `{!! $comment !!}` will execute any `<script>` an attacker stored. **Fix:** default to `{{ }}`; reserve `{!! !!}` for HTML you generated and trust (e.g., server-rendered Markdown), and sanitize first if there's any doubt.

2. **Forgetting `@csrf` (or `@method`) in forms.**
   Omitting `@csrf` on a `POST` form yields a **419 Page Expired** error. Forgetting `@method('PUT')` means your `PUT` route never matches and you get a 405. **Fix:** every non-GET HTML form needs `@csrf`; add `@method(...)` for PUT/PATCH/DELETE.

3. **Heavy logic / queries inside templates.**
   Running `User::where(...)->get()` or big `@php` blocks in a view scatters business logic and triggers N+1 queries. **Fix:** prepare data in the controller, a view-model, or a view composer; keep templates about *presentation*.

4. **Editing a view but seeing stale output in production.**
   If you ran `php artisan view:cache`, Blade serves compiled files. A code deploy that doesn't recompile shows old markup. **Fix:** run `php artisan view:clear` (or re-run `view:cache`) on deploy. In local dev this is automatic via mtime checks.

5. **Treating `@empty($var)` like `@forelse`'s `@empty`.**
   `@empty($var)...@endempty` is a standalone conditional; the `@empty` *inside* `@forelse` takes **no argument**. Mixing them up is a parse error. **Fix:** remember `@forelse ... @empty ... @endforelse` has a bare `@empty`.

6. **Overwriting classes instead of merging in components.**
   Hard-coding `class="btn"` in a component ignores a caller's `class="btn-lg"`. **Fix:** use `{{ $attributes->merge(['class' => 'btn']) }}` so caller classes combine with defaults.

7. **Passing a non-string prop with the literal syntax.**
   `<x-alert :level="3" />` passes integer `3`; `<x-alert level="3" />` passes the **string** `"3"`. For enums, arrays, booleans, and variables, always use the `:` prefix.

---

## ✅ Best Practices

- **Default to `{{ }}`.** Only reach for `{!! !!}` with trusted, generated HTML.
- **Keep logic out of views.** Controllers, view-models, composers, and component classes hold the logic; Blade renders it.
- **Prefer components over `@include` for reusable UI.** Components have explicit props, slots, and attribute handling — far more maintainable than passing loose arrays into partials.
- **Use `@props` with sensible defaults** so components are self-documenting and forgiving.
- **Use `$attributes->merge()`** so components remain customizable by callers.
- **Use stacks (`@push`/`@stack`)** for page-specific assets instead of dumping all CSS/JS in the layout.
- **Use `@class`/`@checked`/`@selected`/`@disabled`** instead of inline ternaries — they're more readable and less error-prone.
- **Run `php artisan view:cache` on deploy** for production performance; `view:clear` when troubleshooting.
- **Name views and components consistently** (kebab-case files, descriptive directories).
- **Use view composers** for data that recurs across many views (nav counts, categories) instead of repeating it in every controller.

---

## 🎯 Interview Tips & Likely Questions

**Q1. How does Blade work under the hood? (the classic "how does it work" question)**
Blade is a *compiling* engine, not an interpreter. On first render it parses the `.blade.php` source, rewrites every directive into plain PHP (e.g., `{{ $x }}` → `<?php echo e($x); ?>`), and writes the result to `storage/framework/views/`. Later requests compare file modification times; if the source is unchanged it just `include`s the cached PHP. So directives carry **no runtime overhead**, and `php artisan view:cache` can precompile everything at deploy time.

**Q2. What's the difference between `{{ }}` and `{!! !!}`?**
`{{ }}` auto-escapes via `e()` (`htmlspecialchars` with `ENT_QUOTES`), preventing XSS. `{!! !!}` outputs raw, unescaped HTML — only safe for trusted content you produced yourself. The default and safe choice is `{{ }}`.

**Q3. How do you output a literal `{{ }}` (e.g., for a Vue template)?**
Prefix it: `@{{ name }}` renders the literal `{{ name }}`. For multi-line blocks use `@verbatim ... @endverbatim`.

**Q4. Layouts (`@extends`/`@yield`) vs. components (`x-`) — when do you use each?**
`@extends`/`@yield` express *page inheritance* — a child page fills holes in one master layout. Components are *encapsulated, reusable UI units* with props, slots, and an attribute bag, composable anywhere. Modern Laravel favors components (even for layouts, via a layout component); inheritance is still fine for a simple single master template.

**Q5. What is the attributes bag and why does `merge` matter?**
`$attributes` holds every attribute a caller passed that wasn't declared a prop, letting components accept arbitrary `class`, `id`, `data-*`, etc. `merge(['class' => '...'])` concatenates the component's base classes with the caller's classes instead of one clobbering the other, so components stay both opinionated and customizable.

**Q6. What does the `$loop` variable give you?**
An object inside `@foreach`/`@forelse` with `index`, `iteration`, `first`, `last`, `count`, `remaining`, `even`, `odd`, `depth`, and `parent` (the enclosing loop's `$loop`). It removes manual counters and `if ($i === 0)` checks.

**Q7. `View::share` vs. view composers?**
`share` registers one value for *every* view, evaluated once at boot. A composer is a callback bound to specific view name(s) (wildcards allowed) that runs *only when those views render*, supports dependency injection, and can lazily compute/query data. Use composers for view-specific, possibly expensive data.

**Q8. Why might you get a 419 error submitting a form?**
A missing or invalid CSRF token. HTML forms need `@csrf`; AJAX needs the token in a header. 419 = "Page Expired" = CSRF mismatch.

**Q9. Anonymous vs. class-based components?**
Anonymous = a single Blade file under `resources/views/components/`, no PHP class — great for presentational components. Class-based = a PHP class (via `make:component`) plus a view, for components needing computed values, dependency injection, or methods. Public constructor properties are auto-exposed to the view.

**Q10. How do you safely pass PHP data to JavaScript in Blade?**
Use `@json($data)` or `{{ Js::from($data) }}`, which `json_encode` with safe flags and proper escaping — never string-concatenate values into a `<script>` (that's an XSS/injection risk).

---

## 📋 Quick Reference / Cheat Sheet

```blade
{{-- ECHO --}}
{{ $x }}              {{-- escaped (safe) --}}
{!! $html !!}         {{-- raw (XSS risk) --}}
@{{ x }}              {{-- literal braces --}}
{{ $x ?? 'default' }} {{-- null coalesce --}}
@json($data)          {{-- PHP -> JS --}}

{{-- CONDITIONALS --}}
@if(...) @elseif(...) @else @endif
@unless(...) @endunless
@isset($v) @endisset      @empty($v) @endempty
@switch($v) @case('a') @break @default @endswitch
@auth @endauth   @guest @endguest   @auth('admin') @endauth
@can('update',$m) @elsecan(...) @else @endcan
@cannot('delete',$m) @elsecannot(...) @endcannot
@canany(['a','b'],$m) @elsecanany(...) @endcanany
@production @endproduction   @env('staging') @endenv  {{-- @env(['staging','production']) ok --}}

{{-- LOOPS --}}
@for(...) @endfor
@foreach($a as $x) @endforeach
@forelse($a as $x) @empty ... @endforelse   {{-- bare @empty --}}
@while(...) @endwhile
@continue($cond)  @break($cond)
$loop->index|iteration|first|last|count|remaining|even|odd|depth|parent

{{-- RAW PHP & COMMENTS --}}
@php $t = 1; @endphp
{{-- compile-time comment (not sent to browser) --}}

{{-- INCLUDES --}}
@include('view', [...])     @includeIf(...)    @includeWhen($c,'view')
@includeUnless($c,'view')   @includeFirst([...])
@each('view',$items,'item','empty-view')

{{-- LAYOUTS --}}
@extends('layouts.app')
@section('content') ... @endsection   {{-- or @stop --}}
@section('title','Inline value')
@yield('content', 'default')
@section('sidebar') ... @show   {{-- define + yield in layout --}}
@parent   {{-- inject parent's section content --}}

{{-- STACKS --}}
@stack('scripts')
@push('scripts') ... @endpush     @prepend(...) @endprepend
@pushOnce('scripts') ... @endPushOnce   {{-- note the capital P/O --}}
@once ... @endonce               {{-- run a block once per render --}}

{{-- COMPONENTS --}}
<x-alert type="error" :level="$n" class="mt-2">body</x-alert>
<x-forms.input />          {{-- subdirectory --}}
<x-dynamic-component :component="$name" />
@props(['type' => 'info'])              {{-- in component file --}}
@aware(['color' => 'gray'])             {{-- pull parent's attribute in child --}}
{{ $slot }}                             {{-- default slot --}}
<x-slot:title>...</x-slot>              {{-- named slot (close with plain </x-slot>) --}}
{{ $attributes->merge(['class'=>'btn']) }}
{{ $attributes->class(['btn','on'=>$x]) }}
:$user   {{-- shorthand for :user="$user" --}}

{{-- FORMS & ATTRS --}}
@csrf   @method('PUT')
@error('email') {{ $message }} @enderror
@class(['card','active'=>$on])   @style([...])
@checked($b)  @selected($b)  @disabled($b)  @readonly($b)  @required($b)
{{ old('field', $default) }}
```

```bash
# Components & caching
php artisan make:component Alert            # class + view
php artisan make:component Alert --view     # anonymous (view only)
php artisan view:cache                      # precompile all views
php artisan view:clear                      # delete compiled views
```

```php
// Passing / sharing data
return view('profile', ['user' => $user]);
return view('profile', compact('user'));
return view('profile')->with('user', $user);
View::share('appName', config('app.name'));                 // global
View::composer('partials.sidebar', fn($v) => $v->with(...)); // view-specific
```

---

## 🧪 Mini Exercises

1. **Escaping & XSS.** Create a view that receives `$bio` (user-supplied). Render it safely, then add a separate `$renderedMarkdown` field you display unescaped. Explain in a comment why each uses the syntax it does, and prove the difference by mentally tracing what happens if `$bio = '<img src=x onerror=alert(1)>'`.

2. **Layout + stacks.** Build `layouts/app.blade.php` with a `@yield('content')`, a `@stack('scripts')` before `</body>`, and a default sidebar section using `@show`. Create a `reports` child view that fills `content`, extends the sidebar with `@parent`, and `@push`es a page-specific `<script>`.

3. **Reusable component.** Make a class-based `<x-alert>` component with an enum-typed `type` prop (`info`/`success`/`error`), a `dismissible` boolean, a default slot for the message, and a merged `$attributes` bag so callers can add classes. Render it three ways from a parent view.

4. **Loop metadata.** Given `$invoices` (a collection), render a table that zebra-stripes rows using `$loop->even`, shows a 1-based number with `$loop->iteration`, marks the last row with a "Total" footer using `$loop->last`, and falls back to "No invoices" via `@forelse`.

5. **View composer.** Register a composer that attaches `$notificationCount` to every view matching `dashboard.*`, pulling the count from a service via constructor injection. Then display it in a `dashboard.header` partial.
