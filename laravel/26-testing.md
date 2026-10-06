# Testing in Laravel (PHPUnit & Pest)

Testing is the discipline of writing *code that checks your code*. In a Laravel
interview, "do you write tests?" is almost guaranteed, and the strongest
candidates can speak fluently about **feature vs unit tests**, the
**RefreshDatabase** trait, **fakes/mocks**, and **factories**. This module takes
you from "what is a test" all the way to parallel execution and coverage.

> Target stack: **PHP 8.4** (notes for 8.1–8.3 where relevant) and
> **Laravel 12** (notes for Laravel 10/11 where behavior differs).

---

## **What you'll learn**

- *Why* automated tests exist and the cost they save you.
- The difference between **PHPUnit** and **Pest**, and how to read/write both.
- **Unit vs feature** tests and when to reach for each.
- Scaffolding tests with `make:test` and the database-isolation traits
  (`RefreshDatabase`, `DatabaseTransactions`, `DatabaseMigrations`).
- Writing **HTTP tests** (`get`/`post`/`getJson`/`actingAs`/`withoutMiddleware`)
  and the assertion vocabulary (`assertOk`, `assertJsonStructure`,
  `assertDatabaseHas`, `assertSoftDeleted`, …).
- **Fakes and mocks** (`Mail::fake`, `Queue::fake`, `Http::fake`, `mock()`,
  `spy()`, partial mocks) and *why* you fake instead of hitting the real thing.
- **Factories**, **datasets/data providers**, the **Arrange-Act-Assert** rhythm,
  a nod to **TDD**, and how to run tests fast (`--filter`, `--parallel`,
  coverage).

---

## 1. Why test at all?

A test is a small program that exercises part of your application and *fails
loudly* when behavior changes unexpectedly. The "why" boils down to four things:

1. **Confidence to change code.** Without tests, every refactor is a gamble. A
   test suite is a safety net that screams when you break something.
2. **Living documentation.** A well-named test like
   `test_guest_cannot_view_dashboard` tells the next developer exactly what the
   system is supposed to do — and it can't go stale, because CI runs it.
3. **Faster debugging.** A failing test pinpoints the broken behavior; you don't
   click through the UI 30 times to reproduce a bug.
4. **Design pressure.** Code that's hard to test is usually badly coupled. The
   pain of testing nudges you toward smaller, injectable units.

The classic counter-argument is "tests slow me down." They slow down the *first*
write and dramatically speed up everything after — especially when a 5-week
sprint turns into a year-long product.

---

## 2. PHPUnit vs Pest

There are two test runners you'll see in the Laravel world.

**PHPUnit** is the foundational, class-based testing framework for PHP. It's been
the default for over a decade. Tests are methods on a class that extends
`TestCase`.

**Pest** is a newer testing *framework built on top of PHPUnit*. It keeps all of
PHPUnit's engine but gives you a concise, function-based syntax inspired by
JavaScript's Jest. Because Pest runs on PHPUnit, every Laravel assertion works
identically — only the *outer* syntax changes.

> **Laravel 11+ defaults to Pest.** When you run `laravel new`, the installer
> asks which suite you want and ships Pest by default. Laravel 10 and earlier
> scaffolded PHPUnit. Either way, both are fully supported and you can switch.

### The same test in both syntaxes

PHPUnit (class-based):

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;

class HomePageTest extends TestCase
{
    public function test_the_home_page_returns_a_successful_response(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

Pest (function-based) — the *exact same behavior*:

```php
<?php

// tests/Feature/HomePageTest.php

it('returns a successful response from the home page', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

Notes on Pest:

- `it('...')` and `test('...')` are interchangeable; `it` reads nicely
  ("it returns…").
- `$this` inside the closure is bound to the underlying `TestCase`, so every
  Laravel helper (`$this->get`, `$this->actingAs`, etc.) is available.
- Pest also offers an **expectation API**: `expect($value)->toBe(200)`,
  `->toBeTrue()`, `->toContain('x')`, `->toThrow(...)`.

```php
expect(2 + 2)->toBe(4);
expect([1, 2, 3])->toContain(2)->toHaveCount(3);
expect(fn () => throw new RuntimeException)->toThrow(RuntimeException::class);
```

**Which should you learn?** Learn to *read* both — interviewers and legacy
codebases use PHPUnit, new projects use Pest. The Laravel-specific assertions are
identical, which is 90% of what you'll write.

---

## 3. Test types: unit vs feature

Laravel ships two directories under `tests/`:

| Type        | Directory          | Boots Laravel? | Speed   | Tests…                                  |
|-------------|--------------------|----------------|---------|-----------------------------------------|
| **Unit**    | `tests/Unit`       | No (by default)| Fastest | A single class/method in isolation      |
| **Feature** | `tests/Feature`    | Yes            | Slower  | A whole slice: route → controller → DB  |

- A **unit test** verifies one small piece of logic with no framework
  bootstrapping — e.g. a pure method on a service or a value object. By default
  a Unit test extends `PHPUnit\Framework\TestCase`, so the container,
  config, and DB are **not** available.
- A **feature test** boots the full Laravel application and exercises behavior
  end-to-end through HTTP or the container. This is where most of your value
  lives in a typical web app, because it tests the *integration* of routes,
  middleware, validation, controllers, and the database.

> Rule of thumb: prefer feature tests for "does this endpoint behave?" and unit
> tests for "is this algorithm correct?" Don't over-unit-test glue code.

Example unit test (no DB, pure logic):

```php
<?php

namespace Tests\Unit;

use App\Support\Money;
use PHPUnit\Framework\TestCase; // note: NOT Tests\TestCase

class MoneyTest extends TestCase
{
    public function test_it_formats_cents_to_dollars(): void
    {
        $this->assertSame('$12.34', Money::fromCents(1234)->format());
    }
}
```

If a "unit" test needs the database or container, it isn't really a unit test —
make it extend `Tests\TestCase` (or move it to `tests/Feature`).

---

## 4. Scaffolding tests with `make:test`

Generate tests with Artisan:

```bash
# Feature test (default) -> tests/Feature/OrderTest.php
php artisan make:test OrderTest

# Unit test -> tests/Unit/MoneyTest.php
php artisan make:test MoneyTest --unit

# Force a Pest-style stub even if the project default is PHPUnit
php artisan make:test OrderTest --pest
```

`--unit` places the file in `tests/Unit` and (for PHPUnit) extends the bare
PHPUnit `TestCase`. By default the generated stub follows whichever runner your
project uses (Pest in a fresh Laravel 11/12 app, PHPUnit otherwise); `--pest`
forces a Pest stub.

> **There is no `--phpunit` flag** on `make:test`. To get a PHPUnit-style class,
> simply run `make:test` without `--pest` in a project whose default is PHPUnit.
> The stub used is also controlled by the `tests` block in `composer.json` and
> can be customized via stub publishing.

---

## 5. The TestCase and the configuration files

### `tests/TestCase.php`

Every feature test extends `Tests\TestCase`, which extends
`Illuminate\Foundation\Testing\TestCase`. This base class **creates a fresh
application instance** before each test and tears it down afterward, giving you a
clean container per test.

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    // Add shared setUp() logic here, e.g. seeding roles for every test.
}
```

You can hook into the lifecycle with `setUp()` / `tearDown()` (always call the
parent first):

```php
protected function setUp(): void
{
    parent::setUp(); // MUST be first — boots the app

    $this->seed(RoleSeeder::class);
}
```

### `phpunit.xml` and the test environment

The `phpunit.xml` at the project root drives both PHPUnit and Pest. The key part
is the environment block that forces a *test* configuration:

```xml
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="DB_CONNECTION" value="sqlite"/>
    <env name="DB_DATABASE" value=":memory:"/>
    <env name="CACHE_STORE" value="array"/>
    <env name="MAIL_MAILER" value="array"/>
    <env name="QUEUE_CONNECTION" value="sync"/>
    <env name="SESSION_DRIVER" value="array"/>
</php>
```

- `DB_CONNECTION=sqlite` + `DB_DATABASE=:memory:` gives you an **in-memory SQLite
  database** — created in RAM, blazing fast, and destroyed when the process ends.
  This is the most common test DB setup.
- The other `array`/`sync` drivers keep tests deterministic and side-effect-free.

> **Laravel 11/12 note:** newer skeletons use `CACHE_STORE` (older used
> `CACHE_DRIVER`) and `php artisan test` automatically uses the `testing`
> environment. You can keep a separate `.env.testing` file too — it overrides
> `phpunit.xml` values when present.

> **In-memory SQLite caveats:** (1) Some MySQL/Postgres-only features (certain
> JSON path functions, fulltext indexes, specific column types, vendor-specific
> SQL) behave differently or aren't supported, so a green SQLite run can hide a
> production bug. (2) Parallel runs work, but Laravel's `--parallel` flow is
> designed around *named* per-process databases (`your_db_test_1`,
> `your_db_test_2`, …); a true `:memory:` DB only exists within a single
> connection, so a file-based SQLite DB or a real MySQL/Postgres is the more
> natural fit for `--parallel`. For high-fidelity tests, point at a real
> MySQL/Postgres test database that matches production.

---

## 6. Database traits: isolating each test

If test A creates a user and test B counts users, test B must not see A's data.
Laravel gives you several traits that reset DB state between tests. Use **exactly
one** per test class. `RefreshDatabase` is the right default for the vast majority
of suites.

### `RefreshDatabase` (the default choice)

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderTest extends TestCase
{
    use RefreshDatabase;
}
```

How it works under the hood: on the **first** test in a run, `RefreshDatabase`
**migrates** the database once — but only if the schema isn't already up to date
(it tracks migration state, so back-to-back `php artisan test` runs skip the
migration step). Then, for every test, it wraps the test in a **database
transaction** and rolls it back at the end. So migrations run at most once (fast)
and each test is isolated by a transaction rollback (also fast).

> The official docs put it precisely: *"The `RefreshDatabase` trait does not
> migrate your database if your schema is up to date. Instead, it will only
> execute the test within a database transaction."* A practical consequence: if a
> test that does **not** use the trait inserts rows, those rows can survive into
> later tests — only the transaction-wrapped tests get rolled back.

> Exception: for an **in-memory SQLite** DB the migration runs per test, because
> the `:memory:` database lives only for the life of its connection and is rebuilt
> each time. It's still fast because SQLite in RAM is cheap.

### `DatabaseTransactions`

```php
use Illuminate\Foundation\Testing\DatabaseTransactions;
```

Wraps each test in a transaction and rolls it back — but assumes your schema is
**already migrated**. Use it when you maintain the test DB schema yourself and
don't want migrations to run.

### `DatabaseMigrations` (a.k.a. migrate fresh per test)

```php
use Illuminate\Foundation\Testing\DatabaseMigrations;
```

Runs `migrate:fresh` *before every test*. This is the slowest option but the most
hermetic — useful when a test mutates schema or you don't trust transaction
isolation (e.g. code that issues its own `COMMIT`).

### `DatabaseTruncation` (migrate once, truncate between tests)

```php
use Illuminate\Foundation\Testing\DatabaseTruncation;
```

A middle ground introduced in recent Laravel versions: it migrates on the first
test (if needed), then **truncates** the tables between tests instead of using a
transaction rollback. Slower than `RefreshDatabase` but it survives code that
commits its own transactions — without the per-test full re-migration cost of
`DatabaseMigrations`.

**Pest equivalent:** apply the trait once, often in `tests/Pest.php`, for the
whole directory:

```php
// tests/Pest.php

// Classic form (Pest 2/3): bind the base TestCase + trait to a directory
uses(Tests\TestCase::class, Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature');

// Newer Laravel 12 idiom shown in the docs — pest()->use() inside a file
// or applied per-suite. Both work; pick one style and stay consistent.
pest()->use(Illuminate\Foundation\Testing\RefreshDatabase::class);
```

| Trait                  | Migrates       | Isolation            | Survives self-COMMITs? | Speed   |
|------------------------|----------------|----------------------|------------------------|---------|
| `RefreshDatabase`      | once (or per test on `:memory:`) | transaction rollback | No | Fast |
| `DatabaseTransactions` | never          | transaction rollback | No  | Fastest (no migration) |
| `DatabaseTruncation`   | once           | truncate tables      | Yes | Medium  |
| `DatabaseMigrations`   | per test       | fresh schema         | Yes | Slowest |

---

## 7. Factories in tests

A **factory** is a class that produces model instances with fake but realistic
data. Factories make tests readable: you say "given a user with these traits"
instead of hand-writing every column.

```php
use App\Models\User;
use App\Models\Post;

// One persisted user
$user = User::factory()->create();

// 3 users, not persisted (in-memory only)
$users = User::factory()->count(3)->make();

// Override attributes
$admin = User::factory()->create(['is_admin' => true]);

// States and relationships
$user = User::factory()
    ->has(Post::factory()->count(2))
    ->create();

// Belongs-to relationship
$post = Post::factory()->for(User::factory())->create();
```

`create()` writes to the (test) database and returns a saved model; `make()`
builds an unsaved instance — handy for unit-testing logic without touching the
DB. Combine factories with `RefreshDatabase` and the fake data evaporates after
each test.

---

## 8. HTTP (feature) tests

This is the bread and butter of Laravel testing: send a fake HTTP request through
the full kernel and assert on the response.

### Verbs and JSON variants

```php
$this->get('/posts');
$this->post('/posts', ['title' => 'Hi']);
$this->put('/posts/1', ['title' => 'Edit']);
$this->patch('/posts/1', ['title' => 'Edit']);
$this->delete('/posts/1');

// JSON variants set the Accept/Content-Type headers and parse JSON responses
$this->getJson('/api/posts');
$this->postJson('/api/posts', ['title' => 'Hi']);
$this->putJson('/api/posts/1', ['title' => 'Edit']);
$this->deleteJson('/api/posts/1');
```

### Headers, auth, and bypassing middleware

```php
use App\Models\User;

$user = User::factory()->create();

// Act as a user (sets the authenticated user for the request)
$this->actingAs($user)->get('/dashboard')->assertOk();

// Authenticate via a specific guard / Sanctum token
$this->actingAs($user, 'sanctum')->getJson('/api/me');

// Custom headers (auth bearer, locale, etc.)
$this->withHeaders([
    'Authorization' => 'Bearer ' . $token,
    'X-Requested-With' => 'XMLHttpRequest',
])->getJson('/api/orders');

// Skip a specific middleware — useful to test a controller without throttle/auth.
// In Laravel 11/12 there is no app/Http/Middleware/VerifyCsrfToken anymore; the
// framework's CSRF middleware is Illuminate\Foundation\Http\Middleware\ValidateCsrfToken.
$this->withoutMiddleware(\Illuminate\Routing\Middleware\ThrottleRequests::class)
    ->post('/contact', ['msg' => 'hi']);

// Disable ALL middleware (use sparingly — you lose real-world fidelity)
$this->withoutMiddleware();

// Conversely, opt INTO middleware that's normally off in tests:
$this->withMiddleware();
```

> **CSRF is already off during tests.** Laravel disables CSRF verification for
> the testing environment, so you usually don't need to skip it manually — that's
> why your `post()` calls work without a `_token`. Reach for `withoutMiddleware`
> to bypass auth, throttling, or a custom guard instead.

> **Security note:** `withoutMiddleware()` (especially the no-argument form that
> drops *all* middleware) trades real-world fidelity for convenience. A test that
> passes only because auth/authorization middleware was disabled gives false
> confidence. Prefer testing *through* the middleware (e.g. `actingAs()` for auth)
> and reserve `withoutMiddleware` for narrowly isolating one layer.

### A complete feature test

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class CreatePostTest extends TestCase
{
    use RefreshDatabase;

    public function test_authenticated_user_can_create_a_post(): void
    {
        // Arrange
        $user = User::factory()->create();

        // Act
        $response = $this->actingAs($user)->postJson('/api/posts', [
            'title' => 'My first post',
            'body'  => 'Hello world',
        ]);

        // Assert
        $response->assertCreated() // 201
            ->assertJson(['title' => 'My first post']);

        $this->assertDatabaseHas('posts', [
            'title'   => 'My first post',
            'user_id' => $user->id,
        ]);
    }

    public function test_guest_cannot_create_a_post(): void
    {
        $this->postJson('/api/posts', ['title' => 'x'])
            ->assertUnauthorized(); // 401
    }
}
```

---

## 9. The assertion vocabulary

### Response assertions

```php
$response->assertStatus(200);
$response->assertOk();           // 200
$response->assertCreated();      // 201
$response->assertNoContent();    // 204
$response->assertNotFound();     // 404
$response->assertForbidden();    // 403
$response->assertUnauthorized(); // 401
$response->assertRedirect('/login');
$response->assertSessionHasErrors(['email']);
$response->assertSessionHas('status');
```

### HTML content assertions

```php
$response->assertSee('Welcome back');
$response->assertSeeText('Plain text only'); // strips HTML tags first
$response->assertDontSee('Admin panel');
$response->assertViewIs('dashboard');
$response->assertViewHas('user');
```

### JSON assertions

```php
// Subset match: the given array must EXIST WITHIN the JSON response. Extra
// keys in the response are ignored — the test passes as long as this fragment
// is present (matching nested structure, not just top-level keys).
$response->assertJson(['data' => ['id' => 1, 'name' => 'Ada']]);

// Require an EXACT match of the entire payload (no extra keys allowed):
$response->assertExactJson(['data' => ['id' => 1, 'name' => 'Ada']]);

// Fluent JSON: assert attribute-by-attribute and forbid stray keys with etc()
use Illuminate\Testing\Fluent\AssertableJson;
$response->assertJson(fn (AssertableJson $json) =>
    $json->where('id', 1)
         ->where('name', 'Ada')
         ->missing('password')
         ->etc()
);

// Only the SHAPE matters, not the values (great for dynamic data)
$response->assertJsonStructure([
    'data' => [
        '*' => ['id', 'name', 'email', 'created_at'],
    ],
    'meta' => ['total', 'per_page'],
]);

// At least one element/object contains this fragment
$response->assertJsonFragment(['name' => 'Ada']);
$response->assertJsonMissing(['name' => 'Deleted User']);
$response->assertJsonCount(3, 'data'); // data array has 3 items
$response->assertJsonPath('data.0.email', 'ada@example.com');
```

> **`assertJson` vs `assertJsonFragment` vs `assertJsonStructure`:**
> `assertJson` converts the response to an array and asserts the given array
> *exists within* the response, matching the **nested structure** you provide
> (so `['data' => ['id' => 1]]` looks for `id` *inside* `data`, not at the root).
> `assertJsonFragment` is looser: it finds the given key/values *anywhere* in the
> payload regardless of nesting path. `assertJsonStructure` ignores values and
> checks keys only (use `*` for "every element of this array"). Use
> `assertExactJson` when no extra keys are allowed, and `assertJsonPath` to pin a
> single value at a precise dot-path.

### Database assertions

```php
$this->assertDatabaseHas('users', ['email' => 'ada@example.com']);
$this->assertDatabaseMissing('users', ['email' => 'gone@example.com']);
$this->assertDatabaseCount('orders', 5);

// Soft deletes (model uses the SoftDeletes trait)
$this->assertSoftDeleted($post);                 // pass a model
$this->assertSoftDeleted('posts', ['id' => 1]);  // or table + attrs
$this->assertNotSoftDeleted($post);

// Model existence helpers (Laravel 9+)
$this->assertModelExists($user);
$this->assertModelMissing($deletedUser);
```

---

## 10. Mocking and fakes — and *why*

Tests should be **fast, deterministic, and free of real side effects**. You don't
want a test to actually send an email, charge a credit card, or hit a third-party
API. The fix is to swap the real dependency for a test double.

- A **fake** replaces a whole Laravel subsystem with an in-memory recorder you
  can assert against (`Mail::fake()`, `Queue::fake()`).
- A **mock** is a hand-built stand-in (via Mockery) for a single object whose
  method calls you script and verify.
- A **spy** records calls and lets you assert on them *after* the fact.

### Built-in Laravel fakes

```php
use Illuminate\Support\Facades\{Mail, Queue, Event, Storage, Bus, Notification};
use Illuminate\Support\Facades\Http;

// --- Mail ---
Mail::fake();
// ... run code that should send mail ...
Mail::assertSent(OrderShipped::class);
Mail::assertSent(OrderShipped::class, fn ($mail) => $mail->order->id === 1);
Mail::assertNothingSent();
Mail::assertQueued(OrderShipped::class); // for ShouldQueue mailables

// --- Queue ---
Queue::fake();
ProcessPodcast::dispatch($podcast);
Queue::assertPushed(ProcessPodcast::class);
Queue::assertPushedOn('high', ProcessPodcast::class);
Queue::assertNothingPushed();

// --- Bus (jobs/batches) ---
Bus::fake();
Bus::assertDispatched(ImportCsv::class);
Bus::assertBatched(fn ($batch) => $batch->jobs->count() === 3);

// --- Events ---
Event::fake();
event(new OrderPlaced($order));
Event::assertDispatched(OrderPlaced::class);
Event::assertNotDispatched(OrderRefunded::class);
// Fake only specific events; let the rest fire normally:
Event::fake([OrderPlaced::class]);

// --- Notifications ---
Notification::fake();
Notification::assertSentTo($user, InvoicePaid::class);

// --- Storage ---
Storage::fake('avatars'); // swaps the disk for an in-memory fake
// ... upload code ...
Storage::disk('avatars')->assertExists('photo.jpg');
Storage::disk('avatars')->assertMissing('old.jpg');

// --- Http client (outgoing requests) ---
Http::fake([
    'github.com/*' => Http::response(['login' => 'octocat'], 200),
    'api.stripe.com/*' => Http::response([], 500),
]);
// ... code that calls Http::get('https://github.com/...') ...
Http::assertSent(fn ($request) => $request->url() === 'https://github.com/users/1');
```

### Time control

```php
use Illuminate\Support\Carbon;

$this->travelTo(Carbon::parse('2026-01-01 12:00:00'));
// ... assertions that depend on "now" ...
$this->travel(5)->days(); // jump forward
$this->travelBack();      // return to real time

// Freeze "now" so it can't drift mid-test (great for timestamp assertions):
$this->freezeTime();                    // freeze at the current instant
$this->freezeTime(function (Carbon $t) {
    // ... time stays frozen for the duration of this closure ...
});
```

### Mockery: `mock()` and `spy()`

When you depend on a *custom* service, mock the contract and bind it into the
container so the code-under-test resolves your double.

```php
use App\Services\PaymentGateway;

public function test_it_charges_the_customer(): void
{
    // Expectation-style mock: charge() MUST be called once with 1000, returns true
    $this->mock(PaymentGateway::class, function ($mock) {
        $mock->shouldReceive('charge')
             ->once()
             ->with(1000)
             ->andReturn(true);
    });

    $this->postJson('/api/checkout', ['amount' => 1000])->assertOk();
}
```

A **spy** is looser — let the call happen, then assert afterward:

```php
$gateway = $this->spy(PaymentGateway::class);

$this->postJson('/api/checkout', ['amount' => 1000]);

$gateway->shouldHaveReceived('charge')->once()->with(1000);
```

A **partial mock** keeps real methods except the ones you override — useful when
you only want to stub one expensive call on an otherwise real object:

```php
$service = $this->partialMock(ReportService::class, function ($mock) {
    $mock->shouldReceive('fetchFromApi')->andReturn(['rows' => []]);
    // every other method runs its real implementation
});
```

> **`mock()` vs `spy()`:** a mock sets expectations *up front* and fails if they
> aren't met; a spy records everything and you assert *after the act*. Mocks suit
> "this must be called exactly so"; spies suit "let it run, then check".

---

## 11. Testing validation, auth, and policies

### Validation

```php
public function test_title_is_required(): void
{
    $user = User::factory()->create();

    $response = $this->actingAs($user)->postJson('/api/posts', [
        'title' => '', // invalid
        'body'  => 'x',
    ]);

    $response->assertStatus(422) // Unprocessable Entity
        ->assertJsonValidationErrors(['title']);
}
```

For web (non-JSON) forms, validation failures redirect back with session errors:

```php
$this->post('/posts', ['title' => ''])
    ->assertRedirect()
    ->assertSessionHasErrors(['title']);
```

`assertValid` / `assertInvalid` are the **generic** validation assertions that
work regardless of whether errors come back as JSON *or* are flashed to the
session — handy when you don't want to special-case the two transports:

```php
// Field(s) failed validation (JSON errors OR session-flashed)
$response->assertInvalid(['title']);
// With an expected message:
$response->assertInvalid(['title' => 'The title field is required.']);

// No validation errors at all
$response->assertValid();
// These specific fields passed:
$response->assertValid(['body']);
```

### Auth state

```php
$this->assertGuest();                 // nobody is logged in
$this->actingAs($user);
$this->assertAuthenticated();
$this->assertAuthenticatedAs($user);
```

### Policies / authorization

```php
public function test_user_cannot_update_others_post(): void
{
    $owner   = User::factory()->create();
    $other   = User::factory()->create();
    $post    = Post::factory()->for($owner)->create();

    $this->actingAs($other)
        ->putJson("/api/posts/{$post->id}", ['title' => 'Hacked'])
        ->assertForbidden(); // 403 from the PostPolicy
}
```

You can also test a policy directly via the gate:

```php
$this->assertTrue($owner->can('update', $post));
$this->assertFalse($other->can('update', $post));
```

---

## 12. Datasets / data providers

Run the *same test body* against many inputs. This keeps coverage high without
copy-pasting test methods.

### PHPUnit data providers

In PHPUnit 10+/11 the modern way is the `#[DataProvider]` attribute (the old
`@dataProvider` docblock still works but is deprecated):

```php
use PHPUnit\Framework\Attributes\DataProvider;

class EmailValidationTest extends TestCase
{
    #[DataProvider('invalidEmails')]
    public function test_it_rejects_invalid_emails(string $email): void
    {
        $this->postJson('/api/subscribe', ['email' => $email])
            ->assertJsonValidationErrors(['email']);
    }

    public static function invalidEmails(): array
    {
        return [
            'missing @'      => ['plainaddress'],
            'missing domain' => ['user@'],
            'spaces'         => ['a b@x.com'],
        ];
    }
}
```

> The provider method **must be `static`** in PHPUnit 10+. The string array keys
> become labels in the test output, e.g. `it rejects invalid emails with data
> set "missing @"`.

### Pest datasets

```php
it('rejects invalid emails', function (string $email) {
    $this->postJson('/api/subscribe', ['email' => $email])
        ->assertJsonValidationErrors(['email']);
})->with([
    'missing @'      => 'plainaddress',
    'missing domain' => 'user@',
    'spaces'         => 'a b@x.com',
]);
```

You can also register reusable named datasets in `tests/Pest.php` via
`dataset('emails', [...])` and reference them with `->with('emails')`.

---

## 13. Arrange-Act-Assert and a TDD nod

**Arrange-Act-Assert (AAA)** is the rhythm every good test follows:

1. **Arrange** — set up the world (factories, fakes, auth).
2. **Act** — perform the one action under test (a request, a method call).
3. **Assert** — verify the outcome (status, DB rows, fakes recorded calls).

Keep exactly *one* "Act" per test. If you're acting twice, you probably have two
tests.

**TDD (Test-Driven Development)** flips the order — you write the failing test
*first*, then the minimum code to make it pass, then refactor. The loop is
**Red → Green → Refactor**:

```
1. Red:      write a test for behavior that doesn't exist yet → it fails.
2. Green:    write the simplest code that makes it pass.
3. Refactor: clean up with the test as your safety net.
```

You don't have to do strict TDD, but interviewers love hearing that you can
articulate the loop and that tests *drive design* toward small, injectable units.

---

## 14. Running tests, filtering, parallel, and coverage

```bash
# Run the whole suite (preferred — pretty output, uses the testing env)
php artisan test

# Run with the raw PHPUnit binary
./vendor/bin/phpunit

# Run Pest directly
./vendor/bin/pest

# Only one file
php artisan test tests/Feature/CreatePostTest.php

# Filter by name (matches test method / Pest description, regex-ish substring)
php artisan test --filter=guest_cannot_create

# Stop on the first failure (fast feedback loop)
php artisan test --stop-on-failure

# List your 10 slowest tests (find what to optimize)
php artisan test --profile

# Run only a group/suite
php artisan test --testsuite=Feature

# Run tests in parallel across CPU cores (uses brianium/paratest)
php artisan test --parallel
php artisan test --parallel --processes=4

# Recreate per-process test databases when running in parallel
php artisan test --parallel --recreate-databases
```

### Parallel testing notes

`--parallel` spins up one process per core and **one test database per process**
(e.g. `your_db_test_1`, `your_db_test_2`). This is why pure in-memory SQLite is a
poor fit for parallel runs — point at a file-based or real DB. If you need
per-process setup, hook `ParallelTesting::setUpProcess(...)` in a service
provider.

### Coverage

Coverage measures which lines your tests actually execute. It requires **Xdebug**
or **PCOV** installed:

```bash
# Text coverage summary in the terminal
php artisan test --coverage

# Fail the build if coverage drops below a threshold (great for CI)
php artisan test --coverage --min=80

# Generate an HTML report with the raw binary
./vendor/bin/phpunit --coverage-html coverage/
```

Output (abridged — `php artisan test --coverage` prints a per-file table then a
total):

```
  ...........................................                    43 / 43 (100%)

  Tests:    43 passed (118 assertions)
  Duration: 1.84s

  ...
  App/Models/User .................................................. 100.0 %
  App/Http/Controllers/PostController ............................... 78.4 %
  ...
  Total: 81.3 %
```

> Coverage is a *floor, not a ceiling*. 100% coverage of trivial getters proves
> little; thoughtful tests of branching business logic prove a lot. Treat
> `--min` as a regression guard, not a goal in itself.

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting `RefreshDatabase` (or using two DB traits).**
   *Symptom:* tests pass alone but fail when run together because leftover rows
   leak between tests, or `assertDatabaseCount` is off by N.
   *Fix:* add `use RefreshDatabase;` to the test class (or register it once in
   `tests/Pest.php`). Use exactly **one** database trait per test.

2. **A "unit" test that needs the database extends the wrong base class.**
   *Symptom:* `Error: A facade root has not been set` or null container, because
   `tests/Unit` extends bare `PHPUnit\Framework\TestCase` which doesn't boot
   Laravel.
   *Fix:* either make the test extend `Tests\TestCase`, or move it to
   `tests/Feature`. True unit tests shouldn't touch the DB at all.

3. **`actingAs($user)` placed *after* the request.**
   *Symptom:* the request runs as a guest and you get 401/302.
   *Fix:* chain it *before* the verb: `$this->actingAs($user)->get(...)`. The
   call returns `$this`, so the order matters.

4. **Asserting on a real side effect instead of faking it.**
   *Symptom:* tests are slow/flaky, emails actually send, or the suite fails when
   a third-party API is down.
   *Fix:* call `Mail::fake()` / `Http::fake()` / `Queue::fake()` **before** the
   Act step, then assert with `Mail::assertSent(...)`. Faking after the action
   records nothing.

5. **`assertJson` when you meant `assertJsonFragment` (or vice versa).**
   *Symptom:* false negatives because `assertJson` matches the **nested
   structure** you give it, so a value that lives at a different path (or in a
   list whose order you didn't account for) won't be found.
   *Fix:* use `assertJsonFragment` to find values *anywhere* regardless of path,
   `assertJsonPath` for a precise location, `assertExactJson` for a full match,
   and `assertJsonStructure` when only the shape matters.

6. **Forgetting that PHPUnit 10+ data providers must be `static`.**
   *Symptom:* `Data Provider method must be static` warning/error after
   upgrading.
   *Fix:* mark the provider `public static function ...` and use the
   `#[DataProvider('name')]` attribute instead of the `@dataProvider` docblock.

7. **Relying on in-memory SQLite for MySQL-specific behavior.**
   *Symptom:* tests green locally but a JSON query / fulltext index / enum column
   behaves differently in production.
   *Fix:* run integration-critical tests against a real MySQL/Postgres test DB,
   especially before relying on DB-vendor features.

---

## ✅ Best Practices

- **One behavior per test, and name it like a sentence.**
  `test_guest_is_redirected_to_login` beats `testIndex`.
- **Follow Arrange-Act-Assert** with a single Act. If you assert before acting
  to "check setup," split the test.
- **Prefer feature tests for endpoints, unit tests for algorithms.** Don't unit
  test framework glue you don't own.
- **Fake the boundaries** (mail, queue, HTTP, storage, events) so tests are fast
  and deterministic — never hit real third parties.
- **Use factories, not hand-written inserts.** Override only the columns the test
  cares about; leave the rest to the factory.
- **Reset state with a database trait**, and keep tests independent — they must
  pass in any order and in parallel.
- **Make tests fast.** A slow suite doesn't get run. Use `:memory:` or a small
  test DB, `--parallel`, and `--stop-on-failure` while developing.
- **Test behavior, not implementation.** Assert on outcomes (status, DB rows,
  recorded fakes), not on private internals — so refactors don't break tests.
- **Wire coverage with `--coverage --min=N` in CI** as a regression guard, not a
  vanity metric.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What's the difference between a unit test and a feature test in Laravel?**
A unit test exercises a single class/method in isolation and, by default,
*doesn't boot the framework* (it extends `PHPUnit\Framework\TestCase`). A feature
test boots the full app and exercises a slice end-to-end — route, middleware,
controller, validation, DB — usually via HTTP helpers. Most web-app value lives
in feature tests.

**Q2. PHPUnit vs Pest — what's the relationship?**
Pest is a thin, function-based layer *built on top of PHPUnit*. It uses the same
engine, so all Laravel assertions are identical; only the outer syntax differs
(`it('...', fn () => ...)` and `expect()` vs `class … extends TestCase`).
Laravel 11+ defaults to Pest; Laravel 10 and earlier scaffolded PHPUnit.

**Q3. How does `RefreshDatabase` actually work under the hood?**
On a persistent DB it migrates **once** for the whole run, then wraps **each
test in a transaction and rolls it back** afterward — fast isolation without
re-migrating. On an in-memory SQLite DB it migrates per test because the DB only
exists for the life of the connection. Contrast with `DatabaseMigrations`, which
runs `migrate:fresh` before *every* test (slow but hermetic), and
`DatabaseTransactions`, which only rolls back and assumes the schema already
exists.

**Q4. Why and how do you fake mail/queue/HTTP in a test?**
To keep tests fast, deterministic, and side-effect-free — you don't want to send
real email or hit a live API. Call `Mail::fake()` / `Queue::fake()` /
`Http::fake([...])` **before** the action, then assert with `Mail::assertSent`,
`Queue::assertPushed`, `Http::assertSent`. The fake swaps the real subsystem for
an in-memory recorder.

**Q5. `mock()` vs `spy()` vs a partial mock?**
`mock()` sets expectations up front and fails if they're not met exactly;
`spy()` lets the call happen and you assert afterward
(`shouldHaveReceived(...)`); a partial mock runs real methods except the few you
stub. All three bind a Mockery double into the container so the
code-under-test resolves it.

**Q6. How do you test that a request returns the right JSON?**
Use `assertJson` to assert a given array exists *within* the response (matching
the nested structure you pass), `assertExactJson` for a full exact match,
`assertJsonFragment` to find values *anywhere* regardless of path,
`assertJsonStructure` to check keys/shape (with `*` for list elements),
`assertJsonPath('data.0.id', 1)` for an exact location, and
`assertJsonCount(n, 'data')` for list sizes. For attribute-by-attribute checks
that also forbid stray keys, pass a closure: `assertJson(fn (AssertableJson
$json) => $json->where(...)->etc())`.

**Q7. How do you test validation failures?**
For APIs, assert a `422` and `assertJsonValidationErrors(['field'])`. For web
forms, assert a redirect and `assertSessionHasErrors(['field'])`.

**Q8. What is AAA and how does it relate to TDD?**
Arrange-Act-Assert is the structure of a single test. TDD is the *workflow*:
write a failing test first (Red), make it pass minimally (Green), then refactor
safely (Refactor). TDD produces tests that drive better, more decoupled design.

**Q9. How do you speed up a slow suite?**
In-memory SQLite or a lean test DB, `RefreshDatabase` (migrate once), fake all
external boundaries, run `--parallel`, and use `--filter`/`--stop-on-failure`
during development. Coverage needs Xdebug/PCOV and is separate from speed.

**Q10. Your test passes alone but fails in the suite — why?**
Almost always **shared state leaking** between tests: a missing DB-isolation
trait, a static/singleton holding data, time not frozen, or order dependence.
The fix is a database trait, faking time with `travelTo`, and ensuring each test
arranges its own world.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Scaffolding
php artisan make:test OrderTest            # feature
php artisan make:test MoneyTest --unit     # unit
php artisan make:test FooTest --pest       # pest syntax

# Running
php artisan test                           # full suite
php artisan test --filter=create_post      # by name
php artisan test tests/Feature/X.php       # one file
php artisan test --stop-on-failure
php artisan test --parallel --processes=4
php artisan test --coverage --min=80
```

```php
// HTTP verbs
$this->get('/x');  $this->getJson('/api/x');
$this->post('/x', $data);  $this->postJson('/api/x', $data);
$this->put(...); $this->patch(...); $this->delete(...);

// Request modifiers
->actingAs($user)            // ->actingAs($user, 'sanctum')
->withHeaders([...])
->withoutMiddleware()        // or ::class to drop a specific one

// Response assertions
->assertOk()                 // 200
->assertCreated()            // 201
->assertNoContent()          // 204
->assertNotFound()           // 404
->assertForbidden()          // 403
->assertUnauthorized()       // 401
->assertRedirect('/login')
->assertSee('text')  ->assertSeeText('text')  ->assertDontSee('x')
->assertViewIs('name')  ->assertViewHas('key')
->assertSessionHasErrors(['email'])
->assertJsonValidationErrors(['email'])   // 422 APIs
->assertInvalid(['email'])  ->assertValid()   // generic: JSON or session

// JSON
->assertJson([...])            // array exists WITHIN response (nested match)
->assertExactJson([...])       // full exact match, no extra keys
->assertJsonFragment([...])    // values anywhere, any path
->assertJsonMissing([...])
->assertJsonStructure([...])   // keys only; '*' = each element
->assertJsonCount(3, 'data')
->assertJsonPath('data.0.id', 1)

// Database
$this->assertDatabaseHas('t', [...]);
$this->assertDatabaseMissing('t', [...]);
$this->assertDatabaseCount('t', 5);
$this->assertSoftDeleted($model);  $this->assertNotSoftDeleted($model);
$this->assertModelExists($m);  $this->assertModelMissing($m);

// Auth
$this->assertGuest();
$this->assertAuthenticated();  $this->assertAuthenticatedAs($user);

// Fakes (call BEFORE the act)
Mail::fake();          Mail::assertSent(X::class);
Queue::fake();         Queue::assertPushed(X::class);
Bus::fake();           Bus::assertDispatched(X::class);
Event::fake();         Event::assertDispatched(X::class);
Notification::fake();  Notification::assertSentTo($u, X::class);
Storage::fake('d');    Storage::disk('d')->assertExists('f.jpg');
Http::fake([...]);     Http::assertSent(fn ($r) => ...);

// Mockery
$this->mock(Svc::class, fn ($m) => $m->shouldReceive('go')->once());
$this->spy(Svc::class);            // assert later: shouldHaveReceived(...)
$this->partialMock(Svc::class, fn ($m) => $m->shouldReceive('x'));

// Factories
User::factory()->create();         // persisted
User::factory()->make();           // unsaved
User::factory()->count(3)->create();
User::factory()->has(Post::factory()->count(2))->create();
Post::factory()->for(User::factory())->create();

// Time
$this->travelTo(now()->addDay());  $this->travel(5)->days();  $this->travelBack();
$this->freezeTime();   // freeze "now" so timestamps don't drift mid-test
```

```php
// Database traits — pick exactly ONE
use Illuminate\Foundation\Testing\RefreshDatabase;      // migrate once + tx rollback (default)
use Illuminate\Foundation\Testing\DatabaseTransactions; // tx rollback only, no migrate
use Illuminate\Foundation\Testing\DatabaseTruncation;   // migrate once + truncate (survives commits)
use Illuminate\Foundation\Testing\DatabaseMigrations;   // migrate:fresh per test (slowest)
```

---

## 🧪 Mini Exercises

1. **First feature test.** Scaffold `php artisan make:test ProductIndexTest`, add
   `RefreshDatabase`, create 3 products with a factory, hit `GET /api/products`,
   and assert a `200` plus `assertJsonCount(3, 'data')`.

2. **Validation + auth.** Write two tests for `POST /api/products`: one where a
   guest gets `401`, and one where an authenticated user submitting an empty
   `name` gets `422` with `assertJsonValidationErrors(['name'])`.

3. **Fake a boundary.** A controller dispatches `SendWelcomeEmail` to a user on
   registration. Use `Mail::fake()` (or `Queue::fake()` if it's queued) to assert
   the mailable was sent to the right address — without sending real mail.

4. **Mock a service.** Bind a fake `PaymentGateway` with `$this->mock(...)` so
   `charge()` must be called once with the correct amount during
   `POST /api/checkout`. Then rewrite it using `$this->spy(...)` and
   `shouldHaveReceived`.

5. **Datasets.** Convert three near-identical "invalid email" tests into a single
   parameterized test — once with a PHPUnit `#[DataProvider]` and once with a Pest
   `->with([...])` dataset. Confirm the data-set labels appear in the test output.
