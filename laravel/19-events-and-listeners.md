# Events & Listeners in Laravel

Events and listeners are Laravel's built-in implementation of the **Observer pattern**: a way to let one part of your application announce that "something happened" without knowing or caring who is listening. This module takes you from the core idea (decoupling) all the way to queued listeners, model observers, broadcasting, and testing — everything you need to discuss events confidently in a backend interview.

**What you'll learn**

- *Why* event-driven design exists and how it decouples your code
- How to define events and listeners, and how Laravel **auto-discovers** them in modern versions (vs the old `EventServiceProvider` `$listen` array)
- Dispatching events with `event()` / `Event::dispatch()` and halting propagation
- Running listeners on a queue (`ShouldQueue`, delays, retries, failure handling, `ShouldDispatchAfterCommit`)
- Event subscribers for grouping related listeners
- Eloquent **model events** and **Observers** (`make:observer`)
- Broadcasting events to the frontend (`ShouldBroadcast`, channels, Echo, Reverb/Pusher) — an overview
- Testing events with `Event::fake()` and assertions
- The decision framework: **events vs jobs vs direct method calls**

---

## 1. The Why: Decoupling Through Events

Imagine a user registers on your site. After registration you want to:

1. Send a welcome email.
2. Notify your Slack channel.
3. Award loyalty points.
4. Add the user to your CRM.

The naive approach crams all of that into the controller:

```php
public function store(Request $request)
{
    $user = User::create($request->validated());

    Mail::to($user)->send(new WelcomeEmail($user));
    Slack::send("New user: {$user->name}");
    $user->loyalty()->create(['points' => 100]);
    $this->crm->addContact($user);

    return redirect()->route('dashboard');
}
```

This **works**, but it is tightly coupled. The controller now knows about email, Slack, loyalty, and CRM. Every new "thing to do on registration" means editing this method, re-testing it, and risking a regression. This violates the **Single Responsibility Principle** — a controller's job is to handle the HTTP request, not orchestrate four subsystems.

The **event-driven** approach inverts this. The controller fires a single, factual announcement — `UserRegistered` — and walks away:

```php
public function store(Request $request)
{
    $user = User::create($request->validated());

    UserRegistered::dispatch($user); // "I'm telling the world this happened"

    return redirect()->route('dashboard');
}
```

Each side-effect becomes a **listener** that subscribes to that event. To add behavior, you write a new listener — you never touch the controller. The controller doesn't know listeners exist. This is **decoupling**: the *producer* of the event and the *consumers* are independent.

**Jargon:**
- **Event** — a value object describing something that happened (past tense: `OrderShipped`, `UserRegistered`). It carries data, not logic.
- **Listener** — a class with a `handle()` method that reacts to a specific event.
- **Dispatch / fire** — the act of broadcasting an event so listeners run.

> **When NOT to use events:** events add indirection. If exactly one thing must happen and it's core to the request, a direct call is clearer. We cover the decision framework in section 11.

---

## 2. Defining Events and Listeners

### Generating an event

```bash
php artisan make:event UserRegistered
```

This creates `app/Events/UserRegistered.php`:

```php
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, SerializesModels;

    // Constructor property promotion (PHP 8.0+) keeps this terse.
    public function __construct(public User $user)
    {
    }
}
```

Note what an event *is*: a plain PHP object that holds data. The two traits matter:

- **`Dispatchable`** gives you the static `UserRegistered::dispatch($user)` helper.
- **`SerializesModels`** lets the event be safely serialized (important for queued listeners — see section 5). It stores only the model's primary key and re-fetches a fresh model from the database on the queue side, instead of serializing the entire object graph.

### Generating a listener

```bash
php artisan make:listener SendWelcomeEmail --event=UserRegistered
```

The `--event` flag type-hints the event in the generated `handle()` signature, which in turn drives auto-discovery (next section). Result, `app/Listeners/SendWelcomeEmail.php`:

```php
<?php

namespace App\Listeners;

use App\Events\UserRegistered;
use Illuminate\Support\Facades\Mail;
use App\Mail\WelcomeEmail;

class SendWelcomeEmail
{
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user)->send(new WelcomeEmail($event->user));
    }
}
```

The listener reaches into the event for the data it needs (`$event->user`). One event can have many listeners — create `AwardLoyaltyPoints`, `NotifySlack`, etc. the same way.

---

## 3. Registration: Auto-Discovery vs the Old `$listen` Array

This is a common source of confusion between Laravel versions, and a frequent interview question.

### Modern Laravel (8.x+, and the default in 10/11/12): auto-discovery

By default, Laravel **scans your `app/Listeners` directory** and reads the type-hint of each listener's `handle()` (or `__invoke()`) method to figure out which event it listens to. Because `SendWelcomeEmail::handle()` type-hints `UserRegistered`, Laravel automatically wires them together. **You write zero registration code.**

You can confirm what's been discovered:

```bash
php artisan event:list
```

Output (abbreviated):

```
App\Events\UserRegistered ............ App\Listeners\SendWelcomeEmail
App\Events\UserRegistered ............ App\Listeners\AwardLoyaltyPoints
```

In Laravel 11 and 12 there is **no `app/Providers/EventServiceProvider.php` by default** — it was removed from the default skeleton. Auto-discovery is simply the way things work.

### Manual registration (still fully supported)

You sometimes need explicit registration — e.g. a closure-based listener, or wiring an event to a listener that lives outside `app/Listeners`. In Laravel 11/12 you do this in `AppServiceProvider::boot()` using the `Event` facade:

```php
use Illuminate\Support\Facades\Event;
use App\Events\UserRegistered;
use App\Listeners\SendWelcomeEmail;

public function boot(): void
{
    // Class listener
    Event::listen(UserRegistered::class, SendWelcomeEmail::class);

    // Closure listener — great for tiny, one-off reactions
    Event::listen(function (UserRegistered $event) {
        logger("User {$event->user->id} registered");
    });
}
```

### The legacy way (Laravel 10 and earlier): the `$listen` array

Before auto-discovery became the norm, *and* if you scaffolded a project that still has an `EventServiceProvider`, you registered events explicitly:

```php
// app/Providers/EventServiceProvider.php  (Laravel 10 style)
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmail::class,
        AwardLoyaltyPoints::class,
    ],
];
```

If you still have an `EventServiceProvider`, you can **opt out** of (or into) auto-discovery:

```php
public function shouldDiscoverEvents(): bool
{
    return false; // disable scanning; rely solely on $listen
}
```

**Production tip:** auto-discovery scans the filesystem on each request unless cached. Always cache events when you deploy:

```bash
php artisan event:cache    # build the cached manifest
php artisan event:clear    # remove it (e.g. before local dev)
```

`php artisan optimize` (run during deploys) includes event caching.

---

## 4. Dispatching Events & Halting Propagation

There are several equivalent ways to fire an event:

```php
use App\Events\UserRegistered;

// 1. The Dispatchable trait's static helper (most common, most readable)
UserRegistered::dispatch($user);

// 2. The global helper function
event(new UserRegistered($user));

// 3. The Event facade
use Illuminate\Support\Facades\Event;
Event::dispatch(new UserRegistered($user));

// 4. Conditional dispatch helpers (Dispatchable trait)
UserRegistered::dispatchIf($shouldNotify, $user);
UserRegistered::dispatchUnless($user->isGuest(), $user);
```

All four do the same thing: resolve every registered listener and invoke it **synchronously, in registration order**, within the same request/process (unless a listener is queued — section 5).

### Halting propagation

A listener can **stop** subsequent listeners for the same event by returning `false` from `handle()`:

```php
public function handle(UserRegistered $event): bool
{
    if ($event->user->isBanned()) {
        return false; // listeners registered AFTER this one will NOT run
    }

    // ... normal work
    return true;
}
```

Returning any non-`false` value (including `null`/`void`) lets the chain continue. This is rarely needed; reserve it for genuine short-circuit logic.

### Return values with `dispatch()`

If you dispatch with `halt` semantics or need responses, `Event::until()` dispatches until the first non-null response:

```php
$response = Event::until(new CheckingOut($cart)); // returns first non-null listener result
```

This is an advanced pattern; most apps never need it.

---

## 5. Queued Listeners

By default listeners run **synchronously** — the user waits while the welcome email sends. For slow work (email, HTTP calls, image processing) you want it to run **in the background** on a queue. Just implement `ShouldQueue`:

```php
<?php

namespace App\Listeners;

use App\Events\UserRegistered;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public function handle(UserRegistered $event): void
    {
        // runs on a queue worker, off the request lifecycle
    }
}
```

Now dispatching the event pushes the listener onto the queue instead of running it inline. The event object is serialized — this is why `SerializesModels` matters: the `User` becomes just an ID, and a fresh `User` is loaded when the job runs.

> **Requirement:** queued listeners only actually run in the background if a queue worker is running (`php artisan queue:work`) and your `QUEUE_CONNECTION` is not `sync`. With `QUEUE_CONNECTION=sync`, "queued" listeners still execute immediately and synchronously — a classic gotcha.
>
> **Version note:** a fresh Laravel 11/12 `.env.example` actually ships with `QUEUE_CONNECTION=database` (and `DB_CONNECTION=sqlite`), so out of the box jobs are *persisted* to a queue — but they still won't run until you start a worker. The string `sync` is the *fallback default* baked into `config/queue.php` (`env('QUEUE_CONNECTION', 'sync')`), which only takes effect if the env var is missing. Older Laravel skeletons (10 and earlier) did ship `QUEUE_CONNECTION=sync`. Don't assume "sync by default" on a modern app — check the actual `.env`.

### Configuring the queue, connection, and delay

```php
class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public string $connection = 'redis';   // which queue connection
    public string $queue = 'emails';        // which queue/tube
    public int $delay = 60;                 // seconds before the job becomes available

    public function handle(UserRegistered $event): void { /* ... */ }
}
```

For dynamic values, implement methods instead of properties:

```php
use Illuminate\Support\Carbon;

public function withDelay(UserRegistered $event): int
{
    return $event->user->isVip() ? 0 : 300;
}

public function viaQueue(): string
{
    return 'emails';
}

public function viaConnection(): string
{
    return 'redis';
}
```

### Retries, timeouts, and backoff

```php
class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public int $tries = 3;          // attempts before marking failed
    public int $timeout = 30;       // seconds before the attempt is killed
    public int $maxExceptions = 2;  // unhandled exceptions allowed before failing

    // Exponential-ish backoff: 10s, then 60s, then 120s between retries
    public function backoff(): array
    {
        return [10, 60, 120];
    }

    // Stop retrying after this moment, regardless of $tries
    public function retryUntil(): \DateTime
    {
        return now()->addMinutes(10);
    }
}
```

### Handling failures

When a queued listener exhausts its retries, Laravel calls the listener's `failed()` method (if defined) with the event and the exception:

```php
use Throwable;

public function failed(UserRegistered $event, Throwable $exception): void
{
    Log::error("Welcome email failed for user {$event->user->id}", [
        'error' => $exception->getMessage(),
    ]);
}
```

The failed job is also recorded in the `failed_jobs` table, so you can inspect and retry:

```bash
php artisan queue:failed        # list failures
php artisan queue:retry all     # re-push them all
```

You can also manually release a job back onto the queue from inside `handle()` (this is what `InteractsWithQueue` provides):

```php
public function handle(UserRegistered $event): void
{
    if (! $this->thirdPartyApiAvailable()) {
        $this->release(30); // try again in 30 seconds; counts toward $tries
        return;
    }
}
```

### Conditional queueing

Implement `shouldQueue()` to decide at runtime whether a listener queues at all:

```php
public function shouldQueue(UserRegistered $event): bool
{
    return $event->user->wantsEmails();
}
```

### `ShouldDispatchAfterCommit` — the database-transaction trap

This is one of the most important real-world gotchas. Consider:

```php
DB::transaction(function () use ($data) {
    $user = User::create($data);
    UserRegistered::dispatch($user); // queued listener
});
```

A queued listener may be picked up by a worker **before the transaction commits**. The worker then tries to load `User::find($id)` — and finds nothing, because the row isn't committed yet. Result: a `ModelNotFoundException` or silently missing data.

The fix is to delay dispatching the queued job until **after** the transaction commits. There are two levers:

**On the event** — implement `ShouldDispatchAfterCommit`:

```php
use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;

class UserRegistered implements ShouldDispatchAfterCommit
{
    use Dispatchable, SerializesModels;

    public function __construct(public User $user) {}
}
```

**On the listener** — in modern Laravel (11/12) the idiomatic way is to implement the `ShouldQueueAfterCommit` interface (it pairs naturally with `ShouldQueue`):

```php
use Illuminate\Contracts\Queue\ShouldQueueAfterCommit;
use Illuminate\Queue\InteractsWithQueue;

class SendWelcomeEmail implements ShouldQueueAfterCommit
{
    use InteractsWithQueue;
    // the listener is now dispatched only after the open transaction commits
}
```

> The older `public bool $afterCommit = true;` property still works (it lives on the underlying queued job via `InteractsWithQueue`), but `ShouldQueueAfterCommit` is what the current docs recommend.

You can also configure it globally per queue connection in `config/queue.php` with `'after_commit' => true` — then *every* queued job/listener on that connection defers to after-commit by default, and you opt individual ones out with `ShouldQueue` + `public bool $afterCommit = false;`. Mention these levers in interviews — it shows production maturity.

> **Two distinct interfaces, don't mix them up:** `ShouldDispatchAfterCommit` (namespace `Illuminate\Contracts\Events`) goes on the **event** and defers when the *event itself* is dispatched. `ShouldQueueAfterCommit` (namespace `Illuminate\Contracts\Queue`) goes on the **queued listener/job** and defers when that *job* is pushed.

---

## 6. Event Subscribers

When one class needs to listen to **several related events**, a *subscriber* groups those listeners together instead of scattering them across many files. A subscriber is a single class with a `subscribe()` method.

```bash
php artisan make:listener UserEventSubscriber
```

```php
<?php

namespace App\Listeners;

use Illuminate\Events\Dispatcher;
use App\Events\UserRegistered;
use App\Events\UserLoggedOut;

class UserEventSubscriber
{
    public function handleUserRegistered(UserRegistered $event): void
    {
        // ...
    }

    public function handleUserLogout(UserLoggedOut $event): void
    {
        // ...
    }

    // Map events to handler methods on THIS class.
    public function subscribe(Dispatcher $events): array
    {
        return [
            UserRegistered::class => 'handleUserRegistered',
            UserLoggedOut::class  => 'handleUserLogout',
        ];
    }
}
```

Returning an array from `subscribe()` (Laravel 8+) is the modern idiom. Alternatively, you can call `$events->listen(...)` directly inside `subscribe()`.

Subscribers are **not** auto-discovered. Register them in `AppServiceProvider::boot()` (Laravel 11/12) or in the `$subscribe` array of `EventServiceProvider` (Laravel 10):

```php
// AppServiceProvider::boot()
use Illuminate\Support\Facades\Event;
use App\Listeners\UserEventSubscriber;

Event::subscribe(UserEventSubscriber::class);
```

```php
// Laravel 10 EventServiceProvider
protected $subscribe = [
    UserEventSubscriber::class,
];
```

---

## 7. Eloquent Model Events & Observers

Eloquent fires events automatically throughout a model's lifecycle. You can hook into them without ever calling `dispatch()` yourself. This is a distinct, built-in event system layered on top of the generic one.

### The model event lifecycle

| Event        | Fires...                                                        | Can cancel? |
|--------------|-----------------------------------------------------------------|-------------|
| `retrieved`  | after a model is loaded from the DB                             | no          |
| `creating`   | before an INSERT (new record)                                   | yes         |
| `created`    | after an INSERT                                                 | no          |
| `updating`   | before an UPDATE (existing record)                             | yes         |
| `updated`    | after an UPDATE                                                 | no          |
| `saving`     | before any save (both insert and update)                       | yes         |
| `saved`      | after any save                                                 | no          |
| `deleting`   | before a DELETE                                                | yes         |
| `deleted`    | after a DELETE                                                  | no          |
| `restoring`  | before a soft-deleted model is restored                        | yes         |
| `restored`   | after restore                                                   | no          |
| `trashed`    | after a soft delete (Laravel 10+)                              | no          |
| `forceDeleting` / `forceDeleted` | around a permanent delete of a soft-deletable model | `forceDeleting` only |
| `replicating`| when a model is being cloned via `$model->replicate()`         | no          |

The full Laravel 12 list is: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `trashed`, `forceDeleting`, `forceDeleted`, `restoring`, `restored`, and `replicating`.

The `-ing` events run **before** the DB write and can be aborted by returning `false`. The `-ed` events run after.

```php
User::creating(function (User $user) {
    if (empty($user->uuid)) {
        $user->uuid = (string) Str::uuid(); // mutate before insert
    }
});
```

> **Critical gotcha:** model events are **only fired when you go through an Eloquent model instance**. Bulk operations that bypass model hydration — `User::query()->update([...])`, `User::where(...)->delete()`, raw `DB::table()` queries, and mass `insert()` — do **not** fire model events. If you need events, iterate models or use `each()`.

### Observers — the clean way to organize model hooks

Closures in a service provider get messy fast. An **Observer** is a dedicated class whose method names mirror the model events.

```bash
php artisan make:observer UserObserver --model=User
```

```php
<?php

namespace App\Observers;

use App\Models\User;

class UserObserver
{
    public function creating(User $user): void
    {
        $user->uuid ??= (string) \Illuminate\Support\Str::uuid();
    }

    public function created(User $user): void
    {
        // Dispatch your own domain event from here if you like
        UserRegistered::dispatch($user);
    }

    public function updated(User $user): void
    {
        if ($user->wasChanged('email')) {
            // react to email change
        }
    }

    public function deleted(User $user): void
    {
        $user->profile()->delete();
    }
}
```

### Registering an Observer

In **Laravel 11/12**, the idiomatic way is the `#[ObservedBy]` attribute on the model:

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    // ...
}
```

Or register manually in `AppServiceProvider::boot()` (works in all modern versions):

```php
use App\Models\User;
use App\Observers\UserObserver;

public function boot(): void
{
    User::observe(UserObserver::class);
}
```

> `wasChanged()` (after save) vs `isDirty()` (before save): inside `updating` use `$model->isDirty('column')` to see pending changes; inside `updated` use `$model->wasChanged('column')`. `getOriginal('column')` gives you the pre-save value in both.

### Custom event mapping with `$dispatchesEvents`

You can map specific model events to your own event classes, getting a real event object you can listen to like any other:

```php
class Order extends Model
{
    protected $dispatchesEvents = [
        'created' => \App\Events\OrderCreated::class,
        'deleted' => \App\Events\OrderDeleted::class,
    ];
}
```

Now `OrderCreated` flows through the normal event/listener pipeline.

---

## 8. Broadcasting Events (Overview)

**Broadcasting** pushes a server-side event over WebSockets to the **browser** in real time — think live notifications, chat, presence indicators. The same event object you dispatch on the server can be broadcast to clients.

### Making an event broadcastable

Implement `ShouldBroadcast` (queues the broadcast) or `ShouldBroadcastNow` (sends immediately, synchronously):

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Queue\SerializesModels;
use Illuminate\Foundation\Events\Dispatchable;

class OrderShipped implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Order $order) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('orders.' . $this->order->user_id),
        ];
    }

    // Optional: custom event name on the client side
    public function broadcastAs(): string
    {
        return 'order.shipped';
    }

    // Optional: control exactly what payload the client receives
    public function broadcastWith(): array
    {
        return ['id' => $this->order->id, 'status' => $this->order->status];
    }
}
```

### Channel types

- **Public channel** (`Channel`) — anyone can subscribe. No auth.
- **Private channel** (`PrivateChannel`) — requires authorization. Subscribers are checked against a callback in `routes/channels.php`.
- **Presence channel** (`PresenceChannel`) — a private channel that also tracks *who* is currently subscribed (great for "users online").

Authorization for private/presence channels lives in `routes/channels.php`:

```php
use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('orders.{userId}', function ($user, $userId) {
    return (int) $user->id === (int) $userId; // true => allowed to listen
});
```

### Drivers: Reverb vs Pusher vs Ably

- **Reverb** — Laravel's own first-party WebSocket server (the default recommendation in Laravel 11/12). Self-hosted, no per-message fees. Install with `php artisan install:broadcasting` which also scaffolds Echo.
- **Pusher Channels** — hosted SaaS; zero server management, paid.
- **Ably** — another hosted option.
- **`log` / `null`** — for local dev/testing.

Set the driver in `.env`:

```env
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=local
REVERB_APP_KEY=local-key
REVERB_APP_SECRET=local-secret
REVERB_HOST=localhost
REVERB_PORT=8080
```

> In Laravel 10 the variable was `BROADCAST_DRIVER`; Laravel 11/12 renamed it to `BROADCAST_CONNECTION`. Worth knowing if you maintain older apps.

Run the Reverb server with:

```bash
php artisan reverb:start
```

### Listening on the client with Laravel Echo

```js
// resources/js/echo.js (this is the shape install:broadcasting scaffolds for Reverb in L12)
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

> The official React/Vue starter kits instead scaffold `configureEcho({ broadcaster: 'reverb' })` from `@laravel/echo-react` / `@laravel/echo-vue`, reading the same `VITE_REVERB_*` variables. The raw `new Echo({...})` form above is the framework-agnostic equivalent.

```js
// Subscribe and react
window.Echo.private(`orders.${userId}`)
    .listen('.order.shipped', (e) => {       // leading dot uses broadcastAs() name
        console.log('Order shipped!', e.id);
    });

// Presence channel
window.Echo.join('room.42')
    .here((users) => console.log('Currently here:', users))
    .joining((user) => console.log(user.name, 'joined'))
    .leaving((user) => console.log(user.name, 'left'));
```

> **`toOthers()`:** if the user who triggered the event shouldn't receive their own broadcast (they already updated the UI optimistically), use `broadcast(new OrderShipped($order))->toOthers();`. This requires `InteractsWithSockets`.

---

## 9. Testing Events

Laravel makes event testing painless with `Event::fake()`, which swaps the real dispatcher for one that **records** dispatches instead of running listeners. This isolates the code under test from side-effects.

> **Test runner note:** Laravel 11/12 starter kits scaffold **Pest** as the default test runner (PHPUnit is still fully supported and runs underneath). The same `Event::fake()` / `Event::assertDispatched()` helpers work identically in either. The PHPUnit-class form is shown first; a Pest equivalent follows.

**PHPUnit style:**

```php
use Illuminate\Support\Facades\Event;
use App\Events\UserRegistered;

public function test_registration_dispatches_event(): void
{
    Event::fake();

    $this->post('/register', [
        'name' => 'Ada',
        'email' => 'ada@example.com',
        'password' => 'secret-pass',
        'password_confirmation' => 'secret-pass',
    ]);

    Event::assertDispatched(UserRegistered::class);

    // Assert with a closure on the payload
    Event::assertDispatched(UserRegistered::class, function ($event) {
        return $event->user->email === 'ada@example.com';
    });

    // Other assertions
    Event::assertDispatchedTimes(UserRegistered::class, 1);
    Event::assertDispatchedOnce(UserRegistered::class);            // L11/12 shorthand for Times(…, 1)
    Event::assertNotDispatched(\App\Events\UserBanned::class);
    Event::assertNothingDispatched();
}
```

**Pest style** (the default in modern Laravel starter kits) — same helpers, less boilerplate:

```php
use Illuminate\Support\Facades\Event;
use App\Events\UserRegistered;

it('dispatches UserRegistered on registration', function () {
    Event::fake();

    $this->post('/register', [
        'name' => 'Ada',
        'email' => 'ada@example.com',
        'password' => 'secret-pass',
        'password_confirmation' => 'secret-pass',
    ]);

    Event::assertDispatched(UserRegistered::class, fn ($e) => $e->user->email === 'ada@example.com');
});
```

> **Gotcha:** `Event::fake()` prevents listeners from running. If your test relies on a listener's side-effect (e.g. a row being created), either don't fake that event, or fake selectively:

```php
// Fake only these events; let everything else dispatch normally
Event::fake([UserRegistered::class]);

// OR fake all EXCEPT these
Event::fakeExcept([ImportantEvent::class]);
```

### Verifying a specific listener is attached

```php
Event::assertListening(UserRegistered::class, SendWelcomeEmail::class);
```

### Testing model observers

Because `Event::fake()` also fakes model events, faking can disable your observers. Often it's cleaner to **not** fake and assert on the actual outcome (e.g. `assertDatabaseHas`), or to fake the queue (`Queue::fake()`) when the listener is queued and you just want to assert it was *queued*:

```php
use Illuminate\Support\Facades\Queue;

Queue::fake();
event(new UserRegistered($user));
Queue::assertPushed(SendWelcomeEmail::class); // queued listener became a job
```

---

## 10. Halting, Wildcards, and Other Niceties

### Wildcard listeners

Listen to many events at once with a `*` pattern. The handler receives the event *name* and an array of data:

```php
Event::listen('App\\Events\\Order*', function (string $eventName, array $data) {
    Log::info("Order event fired: {$eventName}");
});
```

### Dispatching from a model

Combine model events and custom events for clean domain logic — fire your own event from inside an observer's `created()` method (shown in section 7). This keeps controllers thin while still giving you rich, testable events.

---

## 11. Events vs Jobs vs Direct Calls — The Decision Framework

This is the single most likely conceptual interview question. Here's the mental model:

| Use a **direct call** when... | Use an **event/listener** when... | Use a **job** when... |
|-------------------------------|-----------------------------------|-----------------------|
| Exactly one thing must happen and it's core to the operation | "Something happened" and *zero-to-many* independent reactions should fire | You have one specific unit of background work to run |
| You want the simplest, most traceable code | You want decoupling and to add behavior without editing the producer | The work is heavy/slow and must run async on a queue |
| The result is needed immediately | Reactions are conceptually side-effects | You need scheduling, batching, chaining, or rate-limiting |

Key clarifications interviewers want to hear:

- **Events describe the past; jobs describe future work.** `OrderShipped` (event) vs `SendShipmentEmail` (job).
- An **event can have many listeners; a job is a single task.** Events are one-to-many fan-out; jobs are one-to-one.
- **They compose.** A common pattern: dispatch an event synchronously, and have a *queued listener* push the slow work onto the queue. The listener is effectively the bridge from "something happened" to "do this job in the background."
- If you find yourself with a single listener that always runs and does heavy work, ask whether a plain job (dispatched directly) would be simpler — you may not need the event layer at all.

---

## ⚠️ Common Mistakes & Gotchas

1. **"Queued" listeners don't run in the background — either `QUEUE_CONNECTION=sync`, or no worker is running.**
   With `QUEUE_CONNECTION=sync`, jobs execute immediately in-process, so a `ShouldQueue` listener *looks* async but isn't. (Note: a fresh Laravel 11/12 `.env.example` ships `QUEUE_CONNECTION=database`, not `sync` — that's only the `config/queue.php` fallback — so the more common modern symptom is "jobs are sitting in the `jobs` table but nothing processes them because I never started a worker.")
   **Fix:** ensure a real connection (`database`, `redis`) **and** run `php artisan queue:work`. `database` is the zero-infra default; if the migration is missing, generate it with `php artisan make:queue-table` and migrate.

2. **Listeners not firing because the event manifest is stale or auto-discovery is off.**
   If you cached events (`event:cache`) and then added a new listener, the new wiring won't be picked up. Or you disabled discovery and forgot to register manually.
   **Fix:** run `php artisan event:clear` in dev, re-cache on deploy, and verify with `php artisan event:list`.

3. **Queued listener dispatched inside a DB transaction fails to find the model.**
   The worker grabs the job before the transaction commits, so `Model::find($id)` returns `null`.
   **Fix:** implement `ShouldDispatchAfterCommit` on the **event**, implement `ShouldQueueAfterCommit` on the **queued listener** (or the older `public bool $afterCommit = true;`), or enable `'after_commit' => true` on the queue connection.

4. **Model events silently skipped on bulk/raw operations.**
   `User::where('active', false)->delete()`, `Model::query()->update([...])`, and `DB::table('users')->insert(...)` do **not** fire `deleting`/`updating`/`creating` because no model is hydrated.
   **Fix:** iterate (`User::where(...)->each(fn ($u) => $u->delete())`) or use `cursor()`/`chunk()` when you need the events. Be aware of the performance trade-off.

5. **`Event::fake()` accidentally disables behavior you're trying to test.**
   Faking stops *all* listeners (and model events) from running, so `assertDatabaseHas` may fail because the listener that creates the row never ran.
   **Fix:** fake selectively with `Event::fake([SpecificEvent::class])` / `fakeExcept(...)`, or assert that the event was dispatched rather than asserting on the downstream side-effect.

6. **Putting heavy logic in the event class instead of the listener.**
   Events are data carriers. Business logic, DB writes, and emails belong in listeners.
   **Fix:** keep events as immutable value objects (promoted constructor properties); push behavior into listeners/observers.

7. **Forgetting `SerializesModels` on a queued event, then serializing a giant object graph (or stale data).**
   Without it, the entire model (and possibly loaded relations) is serialized; the worker may act on stale attributes.
   **Fix:** keep the `SerializesModels` trait so only the primary key is stored and a fresh model is re-fetched.

---

## ✅ Best Practices

- **Name events in the past tense** (`OrderShipped`, `UserRegistered`) and **listeners as actions** (`SendShipmentNotification`). The pairing reads like English.
- **Keep events thin and immutable.** Promoted, `public readonly` constructor properties communicate intent and prevent mutation:
  ```php
  public function __construct(public readonly Order $order) {}
  ```
- **Queue anything slow** (email, HTTP, image work) by implementing `ShouldQueue`; keep request-critical reactions synchronous.
- **Always cache events on deploy** (`php artisan event:cache` / `optimize`) and never commit a stale cache to dev.
- **Use Observers for model lifecycle logic** and the `#[ObservedBy]` attribute (L11/12) for discoverability; reserve closure hooks for trivial one-liners.
- **Default queued listeners to after-commit** when they touch models created in a transaction — implement `ShouldQueueAfterCommit` on the listener (or `ShouldDispatchAfterCommit` on the event), or set `'after_commit' => true` on the connection.
- **Define `failed()` and sensible `$tries`/`backoff()`** on queued listeners so failures are observable and recoverable.
- **Don't over-eventify.** If only one consumer will ever exist and it must run inline, a direct call is clearer. Reach for events when decoupling or fan-out genuinely pays off.
- **Use `broadcast(...)->toOthers()`** to avoid echoing an action back to the user who triggered it.
- **Test the contract, not the implementation:** assert the event was dispatched (`Event::assertDispatched`) in the producer's test, and test the listener separately.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What problem do events and listeners solve?**
A: Decoupling. The code that knows "something happened" (the producer) doesn't need to know who reacts to it. You add or remove reactions (listeners) without touching the producer, satisfying the Single Responsibility and Open/Closed principles. It's Laravel's Observer pattern.

**Q2. How does Laravel know which listener handles which event without me registering anything?**
A: **Auto-discovery.** Laravel scans `app/Listeners`, reflects on each class's `handle()`/`__invoke()` method, and reads the type-hinted event parameter to build the event-to-listener map. In Laravel 11/12 there's no `EventServiceProvider` by default — discovery is the norm. The older approach was the `$listen` array in `EventServiceProvider`. In production you cache this map with `php artisan event:cache`.

**Q3. (Under the hood) What actually happens when I call `event(new OrderShipped($order))`?**
A: The helper resolves the `events` singleton (the `Illuminate\Events\Dispatcher`) from the container and calls `dispatch()`. The dispatcher looks up all listeners registered for that class (and its interfaces/parents, plus wildcard patterns), then iterates them in registration order. For each listener it builds a callable: a synchronous listener is invoked immediately; a listener implementing `ShouldQueue` is wrapped by `CallQueuedListener` and pushed onto the queue instead. If any listener returns `false`, iteration halts. `Event::until()` instead returns the first non-null response.

**Q4. Synchronous vs queued listeners — when and how?**
A: Synchronous listeners run inline within the request; queued ones implement `ShouldQueue` and are pushed to a queue, requiring a running worker. Queue anything slow so it doesn't block the response. You can configure `$connection`, `$queue`, `$delay`, `$tries`, `backoff()`, and a `failed()` handler. The event must be serializable (`SerializesModels`), since the job is reconstructed on the worker.

**Q5. You dispatch a queued listener inside a database transaction and it can't find the record. Why, and how do you fix it?**
A: The worker can pick up the job before the transaction commits, so the row doesn't exist yet. Fix with `ShouldDispatchAfterCommit` on the event, `ShouldQueueAfterCommit` (the modern interface; or the older `$afterCommit = true` property) on the queued listener, or `'after_commit' => true` on the queue connection — all defer the actual dispatch until commit. Bonus: name the two interfaces' different namespaces (`Illuminate\Contracts\Events\ShouldDispatchAfterCommit` vs `Illuminate\Contracts\Queue\ShouldQueueAfterCommit`).

**Q6. What's the difference between an event and a job?**
A: An event is a past-tense announcement that can trigger many listeners (one-to-many fan-out); a job is a single unit of (usually background) work (one-to-one). They compose: a queued listener bridges an event to background work. Use a job directly when there's one specific async task; use an event when multiple independent reactions should fire.

**Q7. Which Eloquent operations do NOT fire model events?**
A: Anything that bypasses model hydration: mass `update()`/`delete()` via the query builder, `DB::table()` raw queries, and bulk `insert()`. Events only fire when you operate on an actual model instance (`save`, `delete`, `create`, etc.). To get events on bulk work, iterate the models.

**Q8. Observers vs the `$dispatchesEvents` array vs closures — when do you use each?**
A: Use an **Observer** to group all of a model's lifecycle hooks in one class (register via `#[ObservedBy]` or `Model::observe`). Use `$dispatchesEvents` when you want a real event object flowing through the normal listener pipeline (so other parts of the app can listen). Use closure hooks (`User::creating(fn...)`) only for trivial one-liners.

**Q9. Give an overview of broadcasting. Public vs private vs presence channels?**
A: Broadcasting pushes server events to the browser over WebSockets via `ShouldBroadcast`. **Public** channels need no auth; **private** channels authorize subscribers via callbacks in `routes/channels.php`; **presence** channels are private channels that also expose who's currently subscribed. The client listens with Laravel Echo. Drivers: Reverb (first-party, self-hosted, default in L11/12), Pusher/Ably (hosted SaaS).

**Q10. How do you test events?**
A: `Event::fake()` swaps in a recording dispatcher, then assert with `Event::assertDispatched(Event::class, $closure)`, `assertDispatchedTimes`, `assertNotDispatched`, `assertListening`. Be aware faking stops listeners and model events from running — fake selectively (`Event::fake([...])` / `fakeExcept(...)`) when you still need real behavior. For queued listeners, `Queue::fake()` + `Queue::assertPushed(...)` verifies it was queued.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeEmail --event=UserRegistered
php artisan make:listener UserSubscriber                 # subscriber (no --event)
php artisan make:observer UserObserver --model=User

# Inspect / cache
php artisan event:list          # show event => listener map
php artisan event:cache         # build cached manifest (deploy)
php artisan event:clear         # remove cached manifest (dev)

# Queues (for queued listeners)
php artisan queue:work          # process queued listeners/jobs
php artisan queue:failed        # list failures
php artisan queue:retry all

# Broadcasting
php artisan install:broadcasting  # scaffolds Echo + driver (Reverb prompt)
php artisan reverb:start          # run first-party WebSocket server
```

```php
// Dispatch
UserRegistered::dispatch($user);                 // Dispatchable trait
event(new UserRegistered($user));                // global helper
Event::dispatch(new UserRegistered($user));      // facade
UserRegistered::dispatchIf($cond, $user);
$resp = Event::until(new CheckingOut($cart));    // first non-null response

// Manual registration (AppServiceProvider::boot, L11/12)
Event::listen(UserRegistered::class, SendWelcomeEmail::class);
Event::listen(fn (UserRegistered $e) => logger($e->user->id));
Event::subscribe(UserEventSubscriber::class);

// Listener interfaces / props
implements ShouldQueue
$connection, $queue, $delay, $tries, $timeout, $maxExceptions, $afterCommit
backoff(): array  retryUntil(): DateTime  failed($event, $e)  shouldQueue($event)

// Event interfaces
implements ShouldDispatchAfterCommit     // ON THE EVENT: defer dispatch until commit (Illuminate\Contracts\Events)
implements ShouldBroadcast / ShouldBroadcastNow

// Queued-listener after-commit interface (different namespace!)
implements ShouldQueueAfterCommit        // ON THE LISTENER/JOB: defer push until commit (Illuminate\Contracts\Queue)

// Halt propagation
return false;   // from a listener's handle()

// Model events (full L12 list)
retrieved creating created updating updated saving saved deleting deleted
restoring restored trashed forceDeleting forceDeleted replicating
$model->isDirty('col')  $model->wasChanged('col')  $model->getOriginal('col')

// Observer registration (L11/12)
#[ObservedBy([UserObserver::class])]   // attribute on model
User::observe(UserObserver::class);    // in a service provider

// Testing
Event::fake();  Event::fake([X::class]);  Event::fakeExcept([Y::class]);
Event::assertDispatched(X::class, fn ($e) => ...);
Event::assertDispatchedTimes(X::class, 1);
Event::assertDispatchedOnce(X::class);
Event::assertNotDispatched(Y::class);
Event::assertNothingDispatched();
Event::assertListening(X::class, L::class);
Queue::fake();  Queue::assertPushed(SendWelcomeEmail::class);
```

```env
# Fresh L11/12 .env.example ships these:
QUEUE_CONNECTION=database     # already async-capable (sync is only config/queue.php's FALLBACK)
BROADCAST_CONNECTION=log      # default; set to 'reverb' to broadcast over WebSockets
DB_CONNECTION=sqlite          # for reference — fresh default is sqlite

# Note: BROADCAST_CONNECTION replaced BROADCAST_DRIVER in L11/12
```

---

## 🧪 Mini Exercises

1. **Fan-out without touching the controller.** Create a `UserRegistered` event dispatched from your registration flow, then add two auto-discovered listeners: `SendWelcomeEmail` (queued) and `AwardSignupBonus` (synchronous). Verify both appear in `php artisan event:list`. Confirm that adding a third listener requires no change to the controller.

2. **Survive a transaction.** Wrap user creation + event dispatch in a `DB::transaction(...)`. Make the queued listener load the user by ID inside `handle()`. Reproduce the "model not found" failure, then fix it with `ShouldDispatchAfterCommit`. Document what changed in the timing of the dispatch.

3. **Observer-driven invariants.** Build a `UserObserver` that (a) assigns a UUID in `creating`, (b) logs the old and new email in `updated` only when the email actually changed (use `wasChanged`/`getOriginal`), and (c) deletes the user's profile in `deleted`. Register it with `#[ObservedBy]`. Then prove that a bulk `User::where(...)->delete()` does NOT trigger your observer, and rewrite it so it does.

4. **Resilient queued listener.** Add `$tries = 3`, a `backoff()` of `[5, 15, 60]`, and a `failed()` method to `SendWelcomeEmail`. Simulate a failing mail driver and confirm the job lands in `failed_jobs`, then retry it with `queue:retry`.

5. **Real-time order status.** Make an `OrderShipped` event implement `ShouldBroadcast` on a `PrivateChannel('orders.{userId}')`, authorize it in `routes/channels.php`, and write the Echo client code to log a toast when it arrives. Use `broadcast(...)->toOthers()` so the shipping admin who triggered it doesn't get their own notification. (Driver: Reverb in local dev.)
