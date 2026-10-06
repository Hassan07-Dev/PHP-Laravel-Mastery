# Queues & Jobs in Laravel

Queues let your application defer time-consuming work — sending email, processing video, calling slow third-party APIs — so that the HTTP request that triggered it can return *immediately*. This module takes you from "what is a queue" to running production workers with Supervisor and Horizon, batching thousands of jobs, and testing it all without a real queue.

> **Target stack:** PHP 8.4 and Laravel 12. Where Laravel 10/11 or PHP 8.1–8.3 behave differently, it's called out inline.

## **What you'll learn**

- **Why** queues exist and which classes of work belong on them.
- How to configure queue **connections and drivers** (`sync`, `database`, `redis`, `sqs`, `beanstalkd`).
- How to create jobs with `make:job`, the `handle()` method, constructor injection, and `SerializesModels`.
- Every way to **dispatch** a job: `dispatch`, `dispatchSync`, `onQueue`, `delay`, `afterCommit`, and the `ShouldQueue` contract.
- How to **run and supervise workers** (`queue:work` vs `queue:listen`, key flags, Supervisor, Horizon).
- How **failures** are handled: the `failed_jobs` table, `tries`, `backoff`, `retryUntil`, `maxExceptions`, and the `failed()` hook.
- Advanced patterns: **job middleware**, **batching**, **chaining**, **unique jobs**, and **encrypted jobs**.
- How to **test** queued code with `Queue::fake()` and `Bus::fake()`.

---

## 1. Why queues? Deferring slow work

Imagine a controller that registers a user and emails them a welcome message:

```php
public function store(Request $request)
{
    $user = User::create($request->validated());

    // This call talks to an SMTP server. It can take 300ms–3s.
    Mail::to($user)->send(new WelcomeMail($user));

    return response()->json($user, 201);
}
```

The user's browser is *blocked* the entire time the mail server is responding. If the mail server is slow, your request is slow. If it throws, your whole request 500s — even though the account was created fine.

A **queue** decouples "I want this work done" from "the work is done now." You push a **job** (a serialized message describing the work) onto the queue and return instantly. A separate long-running process — the **worker** — pulls jobs off the queue and executes them in the background.

```php
public function store(Request $request)
{
    $user = User::create($request->validated());

    SendWelcomeEmail::dispatch($user); // returns in ~1ms

    return response()->json($user, 201);
}
```

**Good candidates for queueing:** sending email/SMS/push, image and video processing, generating PDFs/exports, calling slow external APIs, fanning out webhooks, warming caches, anything idempotent and non-urgent.

**Bad candidates:** work whose result the user needs *in this response* (e.g. a payment authorization the checkout page must confirm). For those, do it synchronously or use a request-scoped pattern.

The mental model: **a queue is a producer/consumer message bus.** Your web process is the producer; one or more worker processes are the consumers.

---

## 2. Queue configuration & drivers

Queue behavior is configured in `config/queue.php`, driven by environment variables. The active connection is set by `QUEUE_CONNECTION`.

```env
# .env
QUEUE_CONNECTION=database
```

> **Laravel 11/12 default:** fresh apps default to `QUEUE_CONNECTION=database`. Laravel 10 and earlier defaulted to `sync`. Always check your `.env` — `sync` means "no queue at all."

`config/queue.php` defines named **connections**, each tied to a **driver**:

```php
// config/queue.php (trimmed)
'default' => env('QUEUE_CONNECTION', 'database'),

'connections' => [
    'sync' => [
        'driver' => 'sync',
    ],

    'database' => [
        'driver' => 'database',
        'connection' => env('DB_QUEUE_CONNECTION'),
        'table' => env('DB_QUEUE_TABLE', 'jobs'),
        'queue' => env('DB_QUEUE', 'default'),
        'retry_after' => (int) env('DB_QUEUE_RETRY_AFTER', 90),
        'after_commit' => false,
    ],

    'redis' => [
        'driver' => 'redis',
        'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => (int) env('REDIS_QUEUE_RETRY_AFTER', 90),
        'block_for' => null,
        'after_commit' => false,
    ],

    'sqs' => [
        'driver' => 'sqs',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
        'queue' => env('SQS_QUEUE', 'default'),
        'suffix' => env('SQS_SUFFIX'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        'after_commit' => false,
    ],

    'beanstalkd' => [
        'driver' => 'beanstalkd',
        'host' => env('BEANSTALKD_QUEUE_HOST', 'localhost'),
        'queue' => env('BEANSTALKD_QUEUE', 'default'),
        'retry_after' => (int) env('BEANSTALKD_QUEUE_RETRY_AFTER', 90),
        'block_for' => 0,
        'after_commit' => false,
    ],
],
```

### The drivers, compared

| Driver | Backing store | Use it when | Notes |
|---|---|---|---|
| `sync` | none | local dev / tests | Runs the job **immediately, inline**. No background processing. |
| `database` | a `jobs` table in your DB | small/medium apps, simplicity | Zero extra infra; uses `SELECT ... FOR UPDATE` row locking. Doesn't scale to huge throughput. |
| `redis` | Redis lists/sorted sets | most production apps | Fast, supports **Horizon**, delayed jobs via sorted sets. Recommended default. |
| `sqs` | AWS SQS | AWS-native / serverless | Managed, durable. ~256KB payload limit; 15-min max delay; visibility timeout instead of `retry_after`. |
| `beanstalkd` | Beanstalkd server | legacy / lightweight | Battle-tested work queue; less common in new apps. |

> **Key distinction — `retry_after` vs visibility timeout.** For `database`, `redis`, and `beanstalkd`, `retry_after` (seconds) tells the queue: "if a worker hasn't reported this job done within N seconds, assume the worker died and make the job available again." For `sqs`, AWS controls this via the queue's **visibility timeout**. **`retry_after` must be longer than your longest job's `--timeout`,** or a still-running job could be picked up a second time.

### Setting up the `database` driver

Laravel 11/12 ship the migrations by default. To (re)publish them:

```bash
php artisan make:queue-table        # creates the jobs migration (if not present)
php artisan make:queue-failed-table # creates the failed_jobs migration
php artisan migrate
```

> In Laravel 10 the command was `php artisan queue:table`. From Laravel 11 the default `database/migrations` already contains `jobs`, `job_batches`, and `failed_jobs` tables, so a plain `php artisan migrate` is usually enough.

---

## 3. Creating a job

Generate a job class:

```bash
php artisan make:job SendWelcomeEmail
```

Output:

```
INFO  Job [app/Jobs/SendWelcomeEmail.php] created successfully.
```

```php
<?php

namespace App\Jobs;

use App\Mail\WelcomeMail;
use App\Models\User;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use Queueable;

    /**
     * Constructor property promotion (PHP 8.0+) keeps job data terse.
     */
    public function __construct(public User $user)
    {
    }

    public function handle(): void
    {
        Mail::to($this->user)->send(new WelcomeMail($this->user));
    }
}
```

A few things to unpack.

### `ShouldQueue` — the interface that makes it asynchronous

`ShouldQueue` is a **marker interface** (no methods). Its presence tells Laravel: "when this job is dispatched, push it onto a queue connection instead of running it inline." Remove the interface and the same class runs **synchronously** when dispatched — useful for jobs you sometimes want sync and sometimes async.

### `Queueable` — the trait that provides the machinery

In Laravel 11/12, `make:job` uses a single `Queueable` trait (from `Illuminate\Foundation\Queue\Queueable`) that bundles the behaviors that used to be separate traits: `Dispatchable`, `InteractsWithQueue`, `Queueable` (the older queue-config trait), `Batchable`, and `SerializesModels`.

> **Laravel 10 difference:** generated jobs used `use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;` listing each trait explicitly. The behavior is the same; Laravel 11+ just consolidated them.

### `handle()` — where the work happens

`handle()` is called by the worker. **You can type-hint dependencies in `handle()` and the service container will inject them** — `handle()` is resolved through the container, so this works:

```php
public function handle(VideoEncoder $encoder): void
{
    $encoder->encode($this->video);
}
```

The constructor data (`$this->video`) is serialized into the queue payload; `handle()` dependencies are resolved fresh at execution time on the worker.

### `SerializesModels` — don't serialize the whole model

When you pass an Eloquent model to a job's constructor, the job is serialized to JSON and stored in the queue. The `SerializesModels` trait (included in `Queueable`) ensures **only the model's primary key and connection are stored**, not the entire row. When the worker runs the job, the model is **re-fetched fresh from the database**.

Why this matters:

```php
new SendWelcomeEmail($user); // stores: model class + id 42, NOT all the columns
```

- The worker always operates on **current** data, not a stale snapshot from dispatch time.
- The payload stays tiny.
- **Gotcha:** if the model is *deleted* between dispatch and execution, the re-fetch throws `ModelNotFoundException` **by default**, so the job fails and (after exhausting `tries`) lands in `failed_jobs`. To instead have the job **quietly delete itself** when a model is missing, set `public bool $deleteWhenMissingModels = true;` on the job — it defaults to `false`. See Common Mistakes.

> Passing a whole **collection** works too; it stores the IDs and re-fetches. Passing raw values (strings, ints, arrays, DTOs) just serializes them as-is.

---

## 4. Dispatching jobs

There are several ways to put a job on the queue. All of these come from the `Dispatchable` trait.

```php
use App\Jobs\SendWelcomeEmail;

// Static helper (most common)
SendWelcomeEmail::dispatch($user);

// Global helper function
dispatch(new SendWelcomeEmail($user));

// Run inline RIGHT NOW, bypassing the queue (even if it implements ShouldQueue)
SendWelcomeEmail::dispatchSync($user);

// Conditional dispatch
SendWelcomeEmail::dispatchIf($user->wantsEmail, $user);
SendWelcomeEmail::dispatchUnless($user->optedOut, $user);
```

### Targeting a connection and a queue

A **connection** is *which queue system* (e.g. `redis`). A **queue** is a *named lane* within that connection (e.g. `emails`, `default`, `high`). Workers can be told to process specific queues, letting you prioritize.

```php
SendWelcomeEmail::dispatch($user)
    ->onConnection('redis')   // choose the connection
    ->onQueue('emails');      // choose the named queue (lane)
```

You can also set these as properties/defaults on the job class:

```php
public $connection = 'redis';
public $queue = 'emails';
```

### Delaying execution

```php
use Illuminate\Support\Carbon;

// Don't run for 10 minutes
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(10));

// Or a number of seconds
SendWelcomeEmail::dispatch($user)->delay(600);
```

> **SQS caveat:** the maximum delay SQS supports is **15 minutes**. Longer delays silently won't work as expected on `sqs`.

### `afterCommit` — the database-transaction trap

If you dispatch a job *inside* a database transaction, a fast worker can pick up the job **before the transaction commits**, so the job's `SerializesModels` re-fetch finds *nothing*. Example bug:

```php
DB::transaction(function () use ($data) {
    $user = User::create($data);
    SendWelcomeEmail::dispatch($user); // worker may run before COMMIT!
});
```

Fix it per-dispatch:

```php
SendWelcomeEmail::dispatch($user)->afterCommit();
```

Or globally per connection in `config/queue.php`:

```php
'redis' => [
    // ...
    'after_commit' => true,
],
```

With `after_commit` enabled globally you can opt a single dispatch back out with `->beforeCommit()`.

### Dispatching after the response is sent

For trivial work where you don't want a queue at all, `dispatch(...)->afterResponse()` runs the job in the same PHP process *after* the HTTP response is sent to the browser:

```php
SendWelcomeEmail::dispatchAfterResponse($user);
```

This is not a real queue (no retries, no separate worker) — just a way to defer to after the response.

---

## 5. Running workers

A **worker** is a long-lived PHP process that pulls jobs and runs them. Two commands exist.

### `queue:work` (use this)

```bash
php artisan queue:work
```

`queue:work` boots the framework **once** and keeps it in memory, processing job after job. It's fast. The catch: because code is held in memory, **deploying new code requires restarting the worker** (see `queue:restart` below).

### `queue:listen` (rarely)

```bash
php artisan queue:listen
```

`queue:listen` re-boots the framework **for every job**. It's much slower but always uses the latest code without a restart. Useful only in development when you don't want to restart after each edit.

### Important `queue:work` options

```bash
php artisan queue:work redis \
    --queue=high,default,low \   # process queues in priority order (high first)
    --tries=3 \                  # max attempts before marking failed
    --timeout=60 \               # kill the job after 60s (SIGALRM)
    --backoff=10 \               # wait 10s before retrying a failed attempt
    --max-jobs=1000 \            # exit after 1000 jobs (let supervisor restart -> fresh memory)
    --max-time=3600 \            # exit after 1 hour
    --memory=256 \               # restart if memory exceeds 256MB
    --sleep=3 \                  # if no jobs, sleep 3s before polling again
    --rest=0                      # seconds to rest between jobs
```

Notes:

- **`--queue=high,default,low`** processes queues left-to-right by priority: it fully drains `high` before touching `default`. This is how you implement priority lanes.
- **`--tries`** sets the max attempts. `--tries=1` (the default if neither the command nor the job sets it) means a single attempt with no retries. **`--tries=0` means retry *indefinitely*** — handy when used together with `$maxExceptions` or `retryUntil()` to bound the retries some other way.
- **`--timeout`** uses the `pcntl` extension to send an alarm signal. **`--timeout` must be shorter than the connection's `retry_after`.** If a job exceeds the timeout, the worker is killed (and Supervisor restarts it); the job's `failed()` method runs only if attempts are exhausted.
- **`--backoff`** delays the *retry*, not the first attempt. Can be a comma list for exponential backoff: `--backoff=10,30,60`.
- **`--max-jobs` / `--max-time` / `--memory`** are anti-memory-leak guards: the worker exits cleanly and Supervisor relaunches it with fresh memory.

### Restarting workers after deploy

Because `queue:work` caches code in memory, after every deploy run:

```bash
php artisan queue:restart
```

This signals all running workers to gracefully die after their current job; Supervisor then relaunches them with the new code.

### Supervisor in production

You never run `queue:work` by hand in production — the process would die when your SSH session closes, and nothing would restart it after a crash, OOM, or `--max-jobs` exit. **Supervisor** is a process monitor that keeps N workers alive.

```ini
; /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/app/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

`numprocs=8` runs 8 concurrent workers. `stopwaitsecs` should exceed your longest job so Supervisor doesn't kill a job mid-flight on shutdown.

### Horizon (for Redis)

**Laravel Horizon** is a first-party dashboard and process manager built specifically for **Redis** queues. It replaces hand-written Supervisor configs with a clean config file, and adds a beautiful real-time dashboard (throughput, runtime, failures, retries), tagging, and metrics.

```bash
composer require laravel/horizon
php artisan horizon:install
```

```php
// config/horizon.php (excerpt)
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['high', 'default', 'low'],
            'balance' => 'auto',            // auto (default) | simple | false
            'autoScalingStrategy' => 'time', // 'time' (clear-time) or 'size' (job count)
            'minProcesses' => 1,
            'maxProcesses' => 20,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
            'tries' => 3,
            'timeout' => 60,
        ],
    ],
],
```

> **Heads-up on priority under `auto`.** With `balance => 'auto'`, the *order* of queues in the `queue` array does **not** create strict priority — Horizon allocates workers by load (per `autoScalingStrategy`). If you need a hard "drain `high` before `default`", either use `balance => false` (strict order, still scales process count) or split the queues into separate supervisors. Plain `queue:work --queue=high,default,low` is the only mode that gives true left-to-right draining.

Run it (under Supervisor, with a *single* `php artisan horizon` process — Horizon spawns the workers itself):

```bash
php artisan horizon
```

```ini
; Supervisor entry for Horizon
[program:horizon]
command=php /var/www/app/artisan horizon
autostart=true
autorestart=true
stopwaitsecs=3600
```

After deploy: `php artisan horizon:terminate` (Horizon's equivalent of `queue:restart`). `balance => 'auto'` dynamically shifts worker processes between queues based on load — its headline feature.

---

## 6. Handling failures

A job **fails** when an uncaught exception escapes `handle()` and all attempts are exhausted, or when it times out repeatedly. Failed jobs land in the `failed_jobs` table.

### Controlling attempts

Three independent knobs, settable as properties on the job or via worker flags:

```php
class ProcessPayment implements ShouldQueue
{
    use Queueable;

    public int $tries = 5;          // max total attempts
    public int $maxExceptions = 2;  // fail after 2 *unhandled exceptions* even if tries remain
    public int $timeout = 120;      // seconds before the worker kills this job

    // Exponential-ish backoff between retries (seconds)
    public function backoff(): array
    {
        return [10, 30, 60];        // 4th+ retry reuses the last value (60s)
    }
}
```

- **`$tries`** — hard cap on attempts. `--tries` on the command is the default; the property overrides it.
- **`$maxExceptions`** — lets a job retry many times (e.g. for a rate-limited resource) but bail early if it's *really* broken. Distinct from `$tries`: a job released back manually (via `$this->release()`) consumes a try but not a "max exception."
- **`backoff()`** (method) or `public int|array $backoff` (property) — delay between retries.

### `retryUntil` — time-based expiry

Instead of a fixed number of tries, expire by wall-clock time:

```php
public function retryUntil(): \DateTime
{
    return now()->addMinutes(10); // keep retrying until 10 min from now
}
```

The deadline is evaluated **per attempt** relative to "now," so returning `now()->addMinutes(10)` does **not** anchor to first-dispatch time — to get a fixed wall-clock expiry, compute it once (e.g. from a constructor timestamp) rather than recomputing `now()` each call.

> **Precedence:** if both `retryUntil()` and `$tries` are defined, `retryUntil()` wins — the job keeps retrying until the timestamp passes, regardless of the attempt count. If only `retryUntil()` is set, `$tries` is effectively unbounded until the deadline.

### The `failed()` hook

When a job permanently fails, Laravel calls its `failed()` method (if defined), passing the exception. Use it to alert, clean up, or compensate:

```php
public function failed(?\Throwable $exception): void
{
    Log::error('Payment job failed', [
        'user_id' => $this->user->id,
        'error'   => $exception?->getMessage(),
    ]);

    $this->user->notify(new PaymentFailedNotification());
}
```

> `failed()` runs **once**, after the final attempt fails. Don't put core logic here — it's for cleanup/alerting.

### Inspecting and replaying failed jobs

```bash
php artisan queue:failed                  # list failed jobs with their UUIDs
php artisan queue:retry 6b1d...e9          # retry one job by its UUID (re-queues it)
php artisan queue:retry all               # retry every failed job
php artisan queue:retry --queue=emails    # retry all failures from a given queue
php artisan queue:forget 6b1d...e9        # delete one failed job record by UUID
php artisan queue:flush                   # delete ALL failed job records (does not retry)
php artisan queue:flush --hours=24        # only flush failures older than 24h
php artisan queue:prune-failed --hours=48 # prune failed jobs older than 48h
```

> The `id` column shown by `queue:failed` is the failed job's **UUID**, and that UUID is what `queue:retry` / `queue:forget` expect (e.g. `queue:retry 6b1d-...-e9`). Retrying a job pushes it back onto its original queue as a fresh attempt and removes it from `failed_jobs`.

Output of `queue:failed`:

```
+--------------------------------------+------------+---------+----------------------------+---------------------+
| ID                                   | Connection | Queue   | Class                      | Failed At           |
+--------------------------------------+------------+---------+----------------------------+---------------------+
| 6b1d...e9                            | redis      | default | App\Jobs\ProcessPayment    | 2026-06-18 09:14:02 |
+--------------------------------------+------------+---------+----------------------------+---------------------+
```

You can hook a global handler in `app/Providers/AppServiceProvider.php` (Laravel 11/12) using the `Queue::failing` event:

```php
use Illuminate\Support\Facades\Queue;
use Illuminate\Queue\Events\JobFailed;

Queue::failing(function (JobFailed $event) {
    // $event->connectionName, $event->job, $event->exception
});
```

> **Laravel 11/12 structure:** there is no `app/Console/Kernel.php` or `app/Http/Kernel.php` anymore. Queue event hooks like `Queue::failing` go in a service provider's `boot()` (e.g. `AppServiceProvider`), providers are registered in `bootstrap/providers.php`, and any scheduled maintenance commands live in `routes/console.php`.

### Pruning failed jobs on a schedule

Failed-job records accumulate forever unless you prune them. In Laravel 11/12, scheduling lives in `routes/console.php` (not a console Kernel):

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('queue:prune-failed --hours=48')->daily();
```

### Manually releasing or failing a job

Inside `handle()`, `InteractsWithQueue` (part of `Queueable`) gives you:

```php
public function handle(): void
{
    if ($this->apiIsRateLimited()) {
        $this->release(30); // put back on the queue, retry in 30s (consumes a try)
        return;
    }

    if ($this->dataIsInvalid()) {
        $this->fail(new \RuntimeException('Bad data')); // fail immediately
        return;
    }

    echo $this->attempts(); // current attempt number
}
```

---

## 7. Job middleware

**Job middleware** wraps `handle()` execution the same way HTTP middleware wraps a request. Define a `middleware()` method returning an array of middleware objects. Laravel ships two very useful ones.

### `WithoutOverlapping` — prevent concurrent runs

Ensures only one job with a given key runs at a time (backed by an atomic cache lock). Perfect for per-resource serialization, e.g. don't process two jobs for the same user account simultaneously:

```php
use Illuminate\Queue\Middleware\WithoutOverlapping;

public function middleware(): array
{
    return [
        (new WithoutOverlapping($this->user->id))
            ->releaseAfter(60)   // if locked, release & retry in 60s
            ->expireAfter(180),  // auto-expire the lock after 180s (crash safety)
    ];
}
```

> Without `expireAfter`, a worker that dies while holding the lock leaves it stuck until cache eviction. Always set an expiry. Use `->dontRelease()` to silently drop overlapping jobs instead of retrying.

### `RateLimited` — throttle by a named limiter

Define a limiter (typically in `AppServiceProvider::boot()`), then attach it:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

// In AppServiceProvider::boot()
RateLimiter::for('external-api', function (object $job) {
    return Limit::perMinute(30); // 30 jobs/min across all workers
});
```

```php
use Illuminate\Queue\Middleware\RateLimited;

public function middleware(): array
{
    return [new RateLimited('external-api')];
}
```

When the limit is hit, the job is released back onto the queue and retried later. There's also `RateLimitedWithRedis` for a Redis-backed atomic limiter (recommended at scale), and `ThrottlesExceptions` to back off when a downstream service is throwing.

---

## 8. Job batching

A **batch** is a group of jobs you dispatch together and track as a unit, with callbacks for completion. Great for "process 10,000 rows" jobs where you want a progress bar and a "do X when all done" hook.

Requires the `job_batches` table (shipped by default in Laravel 11/12; otherwise `php artisan make:queue-batches-table && php artisan migrate`).

The batched jobs must use the `Batchable` trait (included in `Queueable`).

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;
use Throwable;

$batch = Bus::batch([
    new ImportCsvChunk($chunk1),
    new ImportCsvChunk($chunk2),
    new ImportCsvChunk($chunk3),
])->name('User import')
  ->onQueue('imports')
  ->then(function (Batch $batch) {
      // ALL jobs completed successfully
  })->catch(function (Batch $batch, Throwable $e) {
      // First failure detected
  })->finally(function (Batch $batch) {
      // Batch finished executing (success or fail)
  })->dispatch();

return $batch->id; // store this to query progress later
```

Inside a batched job, inspect or add to the batch:

```php
public function handle(): void
{
    if ($this->batch()->cancelled()) {
        return; // stop early if the batch was cancelled
    }

    // ... do work ...

    // Dynamically add more jobs to the running batch:
    $this->batch()->add([new ImportCsvChunk($extraChunk)]);
}
```

Query progress anywhere:

```php
$batch = Bus::findBatch($batchId);

$batch->totalJobs;       // e.g. 3
$batch->pendingJobs;     // remaining
$batch->processedJobs(); // total - pending
$batch->failedJobs;      // count
$batch->progress();      // integer 0–100
$batch->finished();      // bool
$batch->cancel();        // cancel the batch
```

> **`allowFailures()`:** by default, one failed job cancels the batch (no more jobs run). Call `->allowFailures()` on the batch builder to let the rest continue and still fire `then()` if the others succeed.

> **Chains inside batches:** to batch chains, pass an array of arrays — `Bus::batch([ [new A, new B], [new C, new D] ])`. Each **inner array is a chain** that runs sequentially (A then B), while the chains themselves run in parallel. The batch's completion callbacks fire only once every chain has finished.

---

## 9. Job chaining

A **chain** runs jobs **sequentially**, each only after the previous one succeeds. If any link fails, the rest of the chain is abandoned.

```php
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ProcessPodcast($podcast),
    new OptimizeAudio($podcast),
    new ReleasePodcast($podcast),
])->catch(function (Throwable $e) {
    // any job in the chain failed
})->onQueue('media')
  ->dispatch();
```

Or the fluent form from any chainable job:

```php
ProcessPodcast::withChain([
    new OptimizeAudio($podcast),
    new ReleasePodcast($podcast),
])->dispatch($podcast);
```

A job can also append to its own chain at runtime:

```php
public function handle(): void
{
    // ...
    $this->prependToChain(new ExtraStep());  // runs next
    $this->appendToChain(new FinalStep());   // runs at the end
}
```

> **Chain vs Batch:** chain = sequential, order matters, stop on failure. Batch = parallel, order doesn't matter, aggregate completion callback. Use a chain when step B depends on step A's side effects.

---

## 10. Unique jobs

`ShouldBeUnique` prevents **duplicate** jobs from being queued at the same time — e.g. don't queue two "rebuild report for account 42" jobs simultaneously.

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;

class RebuildReport implements ShouldQueue, ShouldBeUnique
{
    use Queueable;

    public function __construct(public Account $account) {}

    // Unique key (defaults to the job class name if omitted)
    public function uniqueId(): string
    {
        return (string) $this->account->id;
    }

    // How long the uniqueness lock lasts (seconds) — safety net if a job hangs
    public int $uniqueFor = 3600;
}
```

How it works: when dispatched, Laravel acquires a cache lock keyed by `class + uniqueId`. While that lock is held, further dispatches of the same key are **silently ignored**. By default the lock is held until **the job completes processing or fails all of its retry attempts** (or until `uniqueFor` expires, whichever comes first).

> **Requires a cache driver that supports atomic locks** — currently `memcached`, `redis`, `dynamodb`, `database`, `file`, and `array`. (Older Laravel versions did not support locks on `file`; modern versions do.) Redis is the usual production choice. You can point uniqueness at a specific lock store by implementing a `uniqueVia()` method that returns a cache repository.

Variants — these differ in **when the lock is released**:

- **`ShouldBeUnique`** (default) — lock held until the job **finishes processing or exhausts all retries**. A duplicate cannot be queued while the first is still running.
- **`ShouldBeUniqueUntilProcessing`** — lock released the moment the job **begins processing**, so a duplicate may be queued while the first job is mid-flight. Use this when you only want to dedupe the *waiting* portion of the queue, not in-flight work.

> **Gotcha:** people often assume the lock drops at processing-start (that is `ShouldBeUniqueUntilProcessing`'s behavior, not the default). If you need true "no second copy until this one is fully done," use plain `ShouldBeUnique` and make sure `uniqueFor` is comfortably longer than the job's worst-case runtime so the safety-net expiry never releases the lock prematurely.

---

## 11. Encrypted jobs

If a job's payload contains sensitive data (PII, tokens), implement `ShouldBeEncrypted`. Laravel encrypts the serialized payload before storing it on the queue, using your app's `APP_KEY`.

```php
use Illuminate\Contracts\Queue\ShouldBeEncrypted;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendSensitiveDocument implements ShouldQueue, ShouldBeEncrypted
{
    use Queueable;

    public function __construct(
        public string $ssn,
        public string $email,
    ) {}

    public function handle(): void
    {
        // payload was encrypted at rest in the queue store
    }
}
```

This matters most for `database`, `redis`, and `sqs` where the payload sits in storage that ops/DBAs might inspect. There's no API change to dispatch — just add the interface.

---

## 12. Testing queues without a real queue

In tests you don't want jobs to actually run. **Fake** the queue and assert what *would* have been dispatched.

### `Queue::fake()` — intercept low-level pushes

```php
use App\Jobs\SendWelcomeEmail;
use Illuminate\Support\Facades\Queue;

public function test_registration_queues_welcome_email(): void
{
    Queue::fake();

    $this->post('/register', ['email' => 'a@b.com', 'password' => 'secret123']);

    Queue::assertPushed(SendWelcomeEmail::class);

    // With a closure for richer assertions
    Queue::assertPushed(function (SendWelcomeEmail $job) {
        return $job->user->email === 'a@b.com';
    });

    Queue::assertPushedOn('emails', SendWelcomeEmail::class); // on a specific queue
    Queue::assertPushed(SendWelcomeEmail::class, 1);          // exactly once
    Queue::assertNotPushed(SomeOtherJob::class);
    Queue::assertNothingPushed();
}
```

Fake only some jobs, let the rest dispatch for real:

```php
Queue::fake([SendWelcomeEmail::class]); // only this one is faked
Queue::fake()->except([CriticalJob::class]);
```

### `Bus::fake()` — for chains and batches

`Queue::fake()` doesn't capture chains/batches well; use `Bus::fake()` for those:

```php
use Illuminate\Support\Facades\Bus;

Bus::fake();

// ... action that dispatches ...

Bus::assertDispatched(ProcessPodcast::class);
Bus::assertChained([
    ProcessPodcast::class,
    OptimizeAudio::class,
    ReleasePodcast::class,
]);
Bus::assertBatched(function (\Illuminate\Bus\PendingBatch $batch) {
    return $batch->name === 'User import' && $batch->jobs->count() === 3;
});
Bus::assertDispatchedTimes(ProcessPodcast::class, 1);
Bus::assertNothingDispatched();
```

> **Testing tip:** to actually *run* a queued job in a test (to assert its side effects), use the `sync` connection (`QUEUE_CONNECTION=sync` in `phpunit.xml`/`.env.testing`) so dispatch executes inline, **and don't fake**. Faking and running are mutually exclusive.

---

## ⚠️ Common Mistakes & Gotchas

1. **`QUEUE_CONNECTION=sync` in production.** With `sync`, "queued" jobs run inline in the web request — zero deferral, and a slow job blocks the response. **Fix:** set `QUEUE_CONNECTION=redis` (or `database`/`sqs`) and run real workers under Supervisor.

2. **Dispatching inside a transaction without `afterCommit`.** A worker grabs the job before `COMMIT`, the `SerializesModels` re-fetch finds nothing, and you get `ModelNotFoundException` (or stale reads). **Fix:** `->afterCommit()` on the dispatch, or set `'after_commit' => true` on the connection.

3. **Forgetting to restart workers after deploy.** `queue:work` caches code in memory, so old code keeps running with new files on disk — leading to baffling "I fixed that bug but it's still happening" reports. **Fix:** run `php artisan queue:restart` (or `horizon:terminate`) in your deploy script.

4. **`--timeout` >= `retry_after`.** If the worker timeout is longer than (or equal to) the connection's `retry_after`, the queue can hand the *same job* to a second worker while the first is still running it — double execution. **Fix:** always keep `--timeout` comfortably below `retry_after` (e.g. timeout 60, retry_after 90).

5. **Passing huge/unserializable data to the constructor.** Closures, PDO connections, or open file handles can't be serialized and throw on dispatch; passing massive arrays bloats the payload. **Fix:** pass IDs/scalars (rely on `SerializesModels` for Eloquent), and re-fetch or rebuild heavy objects inside `handle()`.

6. **Assuming a deleted model still exists.** Because `SerializesModels` re-fetches by ID, if the record was deleted between dispatch and run, the re-fetch throws `ModelNotFoundException` and the job **fails** (`deleteWhenMissingModels` defaults to `false`). **Fix:** if a missing model should be a no-op, set `public bool $deleteWhenMissingModels = true;` so the job is quietly deleted instead of failing — or pass scalars instead of the model and handle the lookup yourself.

7. **Unique jobs with a lock-incapable cache driver.** `ShouldBeUnique` needs a cache store that supports atomic locks (`redis`, `memcached`, `dynamodb`, `database`, `file`, `array`). If `CACHE_STORE` points at a driver without lock support, the uniqueness check is a no-op and duplicates slip through. **Fix:** use one of the supported stores — Redis is the common production choice — or point uniqueness at a specific store via `uniqueVia()`.

---

## ✅ Best Practices

- **Keep jobs small, focused, and idempotent.** A job may run more than once (retries, double-delivery). Design `handle()` so running it twice is harmless (check "already processed?" before acting).
- **Pass IDs, not big objects.** Lean payloads serialize fast and avoid stale data; re-fetch inside `handle()`.
- **Always set `tries`, `timeout`, and `backoff`** (on the job or worker) — never rely on "unlimited retries" by accident.
- **Use named queues for prioritization** (`high`, `default`, `low`) and start workers with `--queue=high,default,low`.
- **Run `queue:restart`/`horizon:terminate` on every deploy.**
- **Use Horizon for Redis** in production for visibility and auto-balancing; use Supervisor with `numprocs` otherwise.
- **Add `afterCommit()`** (or enable it globally) whenever dispatching from within transactions.
- **Monitor `failed_jobs`** and alert on the `failed()` hook / `Queue::failing` event; prune old failures with `queue:prune-failed`.
- **Use `WithoutOverlapping`** for per-resource serialization and `RateLimited` for downstream API limits instead of `sleep()` hacks.
- **Fake the bus in tests** (`Queue::fake`/`Bus::fake`) so suites are fast and deterministic.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Why use a queue instead of doing the work in the request?**
A: To decouple slow/unreliable/non-urgent work from the request lifecycle. The user gets a fast response; the work runs in background workers with retries and isolation, so a slow mail server or flaky API doesn't degrade the user-facing endpoint.

**Q2. What does the `ShouldQueue` interface actually do?**
A: It's a marker interface with no methods. The dispatcher checks `instanceof ShouldQueue`; if present, it pushes the job to the configured queue connection instead of executing it inline. Remove it and the same class runs synchronously when dispatched.

**Q3. How does `SerializesModels` work, and why is it important? (under the hood)**
A: When a job is serialized for the queue, the trait hooks into `__serialize`/`__sleep` and replaces any Eloquent model with a small `ModelIdentifier` (class, primary key, connection, loaded relations). On `__wakeup`/`__unserialize`, it re-queries the database for the current row. This keeps payloads tiny and ensures the worker sees fresh data — at the cost of an extra query and, if the record was deleted, a `ModelNotFoundException` by default (set `$deleteWhenMissingModels = true` to silently delete the job instead).

**Q4. Explain `queue:work` vs `queue:listen`. Which in production?**
A: `queue:work` boots the framework once and keeps it resident — fast, but needs `queue:restart` after deploys because code is cached in memory. `queue:listen` re-bootstraps per job — slower, but always uses fresh code. Production uses `queue:work` (under Supervisor/Horizon) for performance.

**Q5. What's the relationship between `--timeout`, `retry_after`, and double execution? (under the hood)**
A: `retry_after` (or SQS visibility timeout) is the queue's "lease" duration — if a job isn't acknowledged within it, the queue assumes the worker died and re-releases the job. `--timeout` is how long the worker lets a single job run before killing it (via `pcntl` SIGALRM). If `--timeout >= retry_after`, the queue can re-issue a job that's still legitimately running, causing it to run twice. Keep timeout < retry_after.

**Q6. How do `tries`, `maxExceptions`, and `retryUntil` differ?**
A: `tries` caps **total attempts** (every attempt counts, including manual `release()`s). `maxExceptions` caps how many of those attempts may end in an **unhandled exception** — a job that `release()`s itself does *not* burn a `maxExceptions`, so it can retry many times against a rate-limited resource yet still bail fast if it actually throws. `retryUntil` replaces a count with a **deadline** — keep retrying until a timestamp. If both `retryUntil` and `tries` are defined, `retryUntil` takes precedence.

**Q7. When would you choose chaining vs batching?**
A: Chaining for **sequential, dependent** steps where order matters and a failure should halt the rest (A then B then C). Batching for **parallel, independent** jobs where you want aggregate completion callbacks and progress (process 10k rows, then "notify when all done").

**Q8. How do you prevent two jobs operating on the same resource concurrently?**
A: The `WithoutOverlapping` job middleware keyed by the resource ID. It acquires an atomic cache lock; overlapping jobs are released (retried later) or dropped. Always set `expireAfter()` so a crashed worker doesn't leave a stuck lock.

**Q9. How do you make a job's payload encrypted at rest?**
A: Implement `ShouldBeEncrypted`. Laravel encrypts the serialized payload with the app's `APP_KEY` before pushing and decrypts on pop. Useful when the queue store (DB/Redis/SQS) holds PII.

**Q10. How do you test queued behavior?**
A: `Queue::fake()` to assert jobs were pushed (`assertPushed`, `assertPushedOn`, `assertNotPushed`) without running them; `Bus::fake()` for chains/batches (`assertChained`, `assertBatched`). To assert side effects, use the `sync` connection so jobs run inline, and don't fake.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Create
php artisan make:job ProcessPodcast
php artisan make:queue-table          # jobs table (L11+: usually already present)
php artisan make:queue-failed-table   # failed_jobs table
php artisan make:queue-batches-table  # job_batches table

# Run workers
php artisan queue:work redis --queue=high,default --tries=3 --timeout=60 --backoff=10 --max-jobs=1000 --max-time=3600 --memory=256
php artisan queue:listen              # dev only (reloads code per job)
php artisan queue:restart             # graceful restart after deploy

# Horizon (Redis)
php artisan horizon
php artisan horizon:terminate         # graceful restart after deploy

# Failed jobs (the {uuid} is the value shown in queue:failed's ID column)
php artisan queue:failed
php artisan queue:retry {uuid|all}
php artisan queue:forget {uuid}
php artisan queue:flush               # delete all failed records
php artisan queue:prune-failed --hours=48
```

```php
// Dispatching
Job::dispatch($arg);
Job::dispatchSync($arg);                       // inline now
Job::dispatchIf($cond, $arg);
Job::dispatch($arg)->onConnection('redis')->onQueue('emails');
Job::dispatch($arg)->delay(now()->addMinutes(5));
Job::dispatch($arg)->afterCommit();
Job::dispatchAfterResponse($arg);

// Chain / Batch
Bus::chain([new A, new B])->dispatch();
Bus::batch([new A, new B])->then(fn($b)=>...)->catch(fn($b,$e)=>...)->finally(fn($b)=>...)->dispatch();

// Inside handle()
$this->release(30);  $this->fail($e);  $this->attempts();  $this->batch();

// Job knobs (properties / methods)
public int $tries = 5;
public int $maxExceptions = 2;
public int $timeout = 120;
public function backoff(): array { return [10, 30, 60]; }
public function retryUntil(): \DateTime { return now()->addMinutes(10); }
public function middleware(): array { return [new WithoutOverlapping($this->id)]; }
public function failed(?\Throwable $e): void { /* alert/cleanup */ }
```

```php
// Marker interfaces
implements ShouldQueue          // run async
implements ShouldBeUnique       // no duplicate in queue
implements ShouldBeEncrypted    // encrypt payload at rest

// Testing
Queue::fake();  Queue::assertPushed(Job::class);  Queue::assertPushedOn('q', Job::class);
Bus::fake();    Bus::assertChained([A::class,B::class]);  Bus::assertBatched(fn($b)=>...);
```

---

## 🧪 Mini Exercises

1. **Defer the welcome email.** Convert a synchronous `Mail::to($user)->send(...)` in a registration controller into a queued `SendWelcomeEmail` job that implements `ShouldQueue`. Configure the `database` driver, run a worker, and confirm the email is sent in the background. Then add `->afterCommit()` and explain (in a comment) the bug it prevents.

2. **Make a job resilient.** Add `$tries = 4`, a `backoff()` of `[5, 15, 30]`, a `$timeout`, and a `failed()` method that logs the exception. Dispatch a job that deliberately throws, observe the retries, and verify the row that lands in `failed_jobs`. Retry it with `queue:retry`.

3. **Serialize a heavy import with batching.** Split a 50,000-row CSV into chunks and dispatch them with `Bus::batch()`. Add `then()`/`catch()`/`finally()` callbacks, store the batch id, and build a tiny endpoint that returns `$batch->progress()`.

4. **Prevent overlap and throttle.** Add `WithoutOverlapping($this->account->id)` middleware to a per-account job, and a `RateLimited('billing-api')` limiter capped at 10/min. Dispatch many jobs for the same account and confirm they don't run concurrently.

5. **Test it.** Write tests using `Queue::fake()` to assert your registration endpoint pushes `SendWelcomeEmail` exactly once on the `emails` queue, and a separate test using `Bus::fake()` + `assertChained` for a 3-job chain.
