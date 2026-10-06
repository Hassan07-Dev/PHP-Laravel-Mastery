# Task Scheduling in Laravel

Almost every real application needs to do work *on a clock*: send a nightly report, prune stale records every hour, retry failed payments every five minutes, warm a cache before the morning traffic spike. The traditional Unix tool for this is **cron** — a background daemon that runs commands at fixed times defined in a text file (the *crontab*). Cron works, but managing dozens of crontab lines across multiple servers is fragile, version-control-hostile, and impossible to test. Laravel's **task scheduler** solves this by letting you define all your scheduled work *in code*, in your repository, in a fluent, readable, testable API — while still relying on a single, boring cron entry under the hood.

This module takes you from "what is cron" to confidently designing production schedules with overlap protection, multi-server coordination, and lifecycle hooks.

**What you'll learn**

- Why Laravel's scheduler exists and how it relates to raw cron (it does *not* replace cron — it sits on top of it)
- Where to define schedules in Laravel 11/12 (`routes/console.php`, a provider, or `withSchedule()` in `bootstrap/app.php`) versus the older `app/Console/Kernel.php` style
- How to schedule closures, Artisan commands, queued jobs, and shell commands
- The full vocabulary of frequency methods, time constraints, and conditional execution (including sub-minute scheduling)
- Production-grade controls: `withoutOverlapping`, `onOneServer`, `runInBackground`, maintenance-mode behavior, and schedule groups
- Capturing and emailing task output, and wiring up lifecycle hooks, scheduler events, and external heartbeat monitoring
- The CLI tools `schedule:list`, `schedule:test`, `schedule:work`, `schedule:interrupt`, and `schedule:clear-cache`

Target stack: **PHP 8.4** and **Laravel 12**. Differences in PHP 8.1–8.3 and Laravel 10/11 are called out where they matter.

---

## 1. The "why": cron is great, raw crontabs are painful

A typical crontab entry looks like this:

```bash
# ┌── minute (0-59)
# │ ┌── hour (0-23)
# │ │ ┌── day of month (1-31)
# │ │ │ ┌── month (1-12)
# │ │ │ │ ┌── day of week (0-6, Sun=0)
# │ │ │ │ │
  0 3 * * * /usr/bin/php /var/www/app/artisan reports:generate >> /var/log/reports.log 2>&1
  */5 * * * * /usr/bin/php /var/www/app/artisan payments:retry
  0 0 * * 0 /usr/bin/php /var/www/app/artisan db:prune
```

This *works*, but it has real problems:

- **Not in version control.** The schedule lives on the server, not in your repo. A new deploy or a new server means re-creating it by hand.
- **No code review.** Changing a frequency is an SSH session, not a pull request.
- **Hard to test.** You can't easily assert "this command runs every Monday at 3am" in a test suite.
- **Cron's syntax is cryptic.** `*/5 * * * *` is fine once you know it, but `dailyAt('03:00')` is self-documenting.
- **No built-in coordination.** Cron has no notion of "don't start this if the last run is still going" or "only run on one of my five servers."

Laravel's scheduler keeps cron's reliability but moves the *definition* of work into your application code. The trick: **you register exactly one cron entry**, and Laravel figures out which of your many scheduled tasks are actually due each minute.

```bash
* * * * * cd /var/www/app && php artisan schedule:run >> /dev/null 2>&1
```

That single line runs `schedule:run` every minute. When it executes, Laravel evaluates every task you've defined, checks which ones are due *right now*, and runs only those. One crontab line, unlimited tasks, all defined in code.

> **Jargon check.** A *daemon* is a long-running background process. *cron* is the scheduling daemon; the *crontab* ("cron table") is its config file. *Artisan* is Laravel's command-line tool (`php artisan ...`).

---

## 2. Where schedules live: `routes/console.php` vs the old Kernel

This is the single most version-sensitive part of the topic, so let's be precise.

### Laravel 11 and 12 (the modern way)

Laravel 11 removed `app/Console/Kernel.php` as part of its slimmer skeleton. Scheduled tasks are now defined in **`routes/console.php`** using the `Schedule` facade:

```php
<?php
// routes/console.php

use App\Jobs\PruneOldRecords;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::command('inspire')->hourly();

Schedule::job(new PruneOldRecords)->daily();

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->daily();
```

You can also define schedules inside a service provider's `boot()` method by calling the same facade — useful if you want to keep scheduling logic close to a feature module:

```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Support\Facades\Schedule;

public function boot(): void
{
    Schedule::command('reports:generate')->dailyAt('03:00');
}
```

A third option, documented by Laravel, is to keep `routes/console.php` for command definitions only and register schedules via the `withSchedule()` callback in `bootstrap/app.php`. The closure receives the underlying `Schedule` instance:

```php
<?php
// bootstrap/app.php

use Illuminate\Console\Scheduling\Schedule;

return Application::configure(basePath: dirname(__DIR__))
    // ...routing, middleware, exceptions...
    ->withSchedule(function (Schedule $schedule) {
        $schedule->command('reports:generate')->dailyAt('03:00');
    })
    ->create();
```

> **Why `routes/console.php`?** Laravel 11 unified "things that respond to console invocation" — both Artisan closures *and* scheduled tasks — into one place. It's just a plain PHP file that runs at boot; the `Schedule` facade registers tasks with the scheduler service. Note that in `routes/console.php` and a provider you use the **`Schedule` facade** (`Illuminate\Support\Facades\Schedule`), while the `withSchedule()` callback hands you the **concrete scheduler** (`Illuminate\Console\Scheduling\Schedule`) and you call `$schedule->...` on it.

### Laravel 10 and earlier (the legacy way)

In Laravel 10 (and 9, 8…), you defined schedules in the `schedule()` method of `app/Console/Kernel.php`, using the injected `$schedule` instance:

```php
<?php
// app/Console/Kernel.php  (Laravel 10 and earlier)

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->command('inspire')->hourly();
        $schedule->job(new \App\Jobs\PruneOldRecords)->daily();
    }
}
```

The fluent API (`->hourly()`, `->daily()`, etc.) is **identical** between the two styles — only the entry point changed (`$schedule->...` vs `Schedule::...`). Everything in this module applies to both; examples use the Laravel 12 facade form.

> **Upgrade note.** A Laravel 10 app upgraded to 11/12 can *keep* its `app/Console/Kernel.php` — Laravel still calls `schedule()` if the file exists. You are not forced to migrate, but new apps won't have that file.

---

## 3. The four things you can schedule

Laravel can schedule four kinds of work. Pick the right one for the job.

### 3.1 Artisan commands — `command()`

The most common case. Reference a command by its signature string or by its class name (prefer the `::class` constant — it survives refactors and gives IDE support):

```php
use App\Console\Commands\SendDailyDigest;

// By signature string (arguments/options inline)
Schedule::command('emails:send Taylor --force')->dailyAt('07:00');

// By class name (recommended) with an array of arguments/options
Schedule::command(SendDailyDigest::class, ['Taylor', '--force'])->dailyAt('07:00');
```

> **Argument array gotcha.** When you pass arguments via the array form, each element is a *positional argument* (e.g. `'Taylor'`) or an *option string* (e.g. `'--force'`, `'--queue=high'`). It is **not** an associative `['--queue' => 'high']` array — that form is not how the scheduler hands arguments to the command. Bare options are written as full strings: `['--queue=high', 'Taylor']`.

If your command is a *closure command* (defined inline in `routes/console.php` with `Artisan::command(...)`), schedule it by chaining off the definition instead, passing arguments with `->schedule([...])`:

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\DB;

Artisan::command('emails:send {user} {--force}', function (string $user) {
    // ...
})->purpose('Send emails to the specified user')
  ->schedule(['Taylor', '--force'])
  ->daily();
```

### 3.2 Queued jobs — `job()`

Pushes a job onto the queue rather than running it inline. The actual work happens in your queue worker, keeping `schedule:run` fast:

```php
use App\Jobs\GenerateMonthlyInvoices;

Schedule::job(new GenerateMonthlyInvoices)->monthlyOn(1, '02:00');

// job($job, $queue, $connection): 2nd arg = queue NAME, 3rd arg = connection.
// Here: queue "invoices" on the "sqs" connection.
Schedule::job(new GenerateMonthlyInvoices, 'invoices', 'sqs')->monthly();
```

> **Why `job()` over `command()`?** A queued job returns immediately after dispatch, so `schedule:run` doesn't block. Use it for anything heavy — let the queue worker carry the load. Note: you need a running queue worker for the job to actually execute.

### 3.3 Closures / callables — `call()`

For quick, inline logic that doesn't warrant a whole command class:

```php
use App\Models\RecentVisit;

Schedule::call(function () {
    RecentVisit::where('created_at', '<', now()->subDay())->delete();
})->hourly();

// Invokable class instances also work
Schedule::call(new DeleteRecentUsers)->daily();
```

Closures run *inside* the `schedule:run` process, so keep them fast. Anything slow should be a queued job.

### 3.4 Shell commands — `exec()`

Run an arbitrary operating-system command. Great for non-PHP tooling (backups, rsync, etc.):

```php
Schedule::exec('node /home/forge/script.js')->daily();

Schedule::exec('pg_dump mydb > /backups/mydb.sql')->dailyAt('01:00');
```

---

## 4. Frequency methods: saying *when*

Laravel ships a rich vocabulary of frequency methods. Here are the workhorses, grouped by granularity:

```php
Schedule::command('x')->everySecond();         // Laravel 11+ only
Schedule::command('x')->everyTwoSeconds();      // also five/ten/fifteen/twenty/thirty seconds
Schedule::command('x')->everyMinute();
Schedule::command('x')->everyTwoMinutes();      // also three/four/five/ten/fifteen/thirty
Schedule::command('x')->hourly();
Schedule::command('x')->hourlyAt(15);           // at :15 past every hour
Schedule::command('x')->everyOddHour();         // optional minutes arg: everyOddHour(15)
Schedule::command('x')->everyTwoHours();        // also three/four/six hours, each takes a minutes arg
Schedule::command('x')->daily();                // midnight
Schedule::command('x')->dailyAt('13:00');
Schedule::command('x')->twiceDaily(1, 13);      // 01:00 and 13:00
Schedule::command('x')->twiceDailyAt(1, 13, 15);// 01:15 and 13:15
Schedule::command('x')->daysOfMonth([1, 10, 20]); // those days of the month
Schedule::command('x')->weekly();               // Sunday 00:00
Schedule::command('x')->weeklyOn(1, '08:00');   // Monday 08:00 (weeklyOn: 0=Sun, 1=Mon … 6=Sat)
Schedule::command('x')->monthly();              // 1st of month, 00:00
Schedule::command('x')->monthlyOn(4, '15:00');  // 4th at 15:00
Schedule::command('x')->twiceMonthly(1, 16, '13:00');
Schedule::command('x')->lastDayOfMonth('15:00');
Schedule::command('x')->quarterly();
Schedule::command('x')->quarterlyOn(4, '14:00');// 4th of each quarter at 14:00
Schedule::command('x')->yearly();
Schedule::command('x')->yearlyOn(6, 1, '17:00');// June 1st at 17:00
```

> **Day-number conventions differ.** `weeklyOn()` and the `mondays()`/`days()` family treat **Sunday as 0** and Saturday as 6 (the standard cron convention). So `weeklyOn(1, ...)` is Monday and `days([0, 3])` is Sunday and Wednesday. Do not confuse this with Carbon's `dayOfWeek` (also 0=Sunday) or ISO-8601 (1=Monday). When in doubt, use the `Schedule::SUNDAY … Schedule::SATURDAY` constants shown below.

> **Version note — sub-minute scheduling.** `everySecond()` and friends were introduced in **Laravel 11**. Under the hood, when sub-minute tasks exist, `schedule:run` keeps the process alive for the full minute and re-evaluates them. Sub-minute tasks are *not* available in Laravel 10. Because a slow sub-minute task can delay later ones, Laravel recommends sub-minute tasks dispatch a **queued job** (`job()`) or use `runInBackground()` rather than doing heavy work inline.

### Day-of-week constraints

Chain these onto a daily-or-finer schedule to limit which days it runs:

```php
Schedule::command('weekday:report')->weekdays()->dailyAt('08:00');
Schedule::command('weekend:cleanup')->weekends()->dailyAt('02:00');
Schedule::command('payroll')->mondays()->dailyAt('09:00');
// sundays(), tuesdays(), ... saturdays() all exist
Schedule::command('x')->days([1, 4])->hourly();          // Mon & Thu (0=Sun … 6=Sat)
```

The day constants (`SUNDAY`, `MONDAY`, … `SATURDAY`) live on the **concrete** `Illuminate\Console\Scheduling\Schedule` class, *not* on the `Illuminate\Support\Facades\Schedule` facade. To use them you must import the concrete class — the facade has no such constants:

```php
use Illuminate\Support\Facades;
use Illuminate\Console\Scheduling\Schedule;

Facades\Schedule::command('emails:send')
    ->hourly()
    ->days([Schedule::MONDAY, Schedule::FRIDAY]);   // constants from the concrete class
```

### `cron()` — the escape hatch

When the fluent helpers can't express what you need, drop to a raw cron expression:

```php
// Every 5 minutes — equivalent to ->everyFiveMinutes()
Schedule::command('payments:retry')->cron('*/5 * * * *');

// At 23:00 on the last weekday of the month: run on the last 4 calendar days,
// but only fire when TODAY is both the last day of the month and a weekday.
Schedule::command('x')
    ->cron('0 23 28-31 * *')
    ->when(fn () => now()->isLastOfMonth() && now()->isWeekday());
```

> Carbon's `isLastOfMonth()` checks whether *today* is the month's final day; the older `lastOfMonth()` *returns* a new date (it does not test "today"), so `now()->lastOfMonth()->isWeekday()` would always evaluate the same fixed day — a subtle bug. Prefer the boolean `isLastOfMonth()` inside `when()`.

### `timezone()` — anchoring to a clock

Frequency times are interpreted in the **app's default timezone** (`config/app.php` → `timezone`) unless you override per-task:

```php
Schedule::command('reports:generate')
    ->dailyAt('09:00')
    ->timezone('America/New_York');
```

To apply a default timezone to *all* tasks without repeating yourself, set the **`schedule_timezone`** option in `config/app.php`. (The old `scheduleTimezone()` method on the console Kernel still works in legacy Kernel-based apps, but the config option is the documented approach in Laravel 11/12.)

```php
// config/app.php
'timezone' => 'UTC',
'schedule_timezone' => 'America/Chicago',
```

> **DST gotcha.** During Daylight Saving Time transitions, a task scheduled at a time that is *skipped* (spring forward) may not run, and one at a *repeated* time (fall back) may run twice. The Laravel docs explicitly **recommend avoiding timezone scheduling when possible** for this reason — prefer UTC scheduling and avoid the ambiguous hour for critical jobs.

### Time-window and conditional constraints

```php
// Only between 7am and 10pm
Schedule::command('cache:warm')->everyFifteenMinutes()->between('7:00', '22:00');

// The inverse: skip during a window (here, the maintenance hour)
Schedule::command('sync:data')->everyTenMinutes()->unlessBetween('1:00', '2:00');

// when(): run only if the closure returns true
Schedule::command('promo:notify')->daily()->when(function () {
    return Feature::active('summer_sale');
});

// skip(): the inverse — skip if the closure returns true
Schedule::command('newsletter:send')->daily()->skip(function () {
    return now()->isWeekend();
});

// environments(): only in specific environments
Schedule::command('debug:snapshot')->hourly()->environments(['staging']);
```

> `when()` and `skip()` are evaluated *every minute* by `schedule:run`, so keep their closures cheap — no slow queries or HTTP calls inside them.

---

## 5. Production controls: overlaps, single-server, background

These three methods separate a toy schedule from a production-ready one.

### 5.1 `withoutOverlapping()` — don't stack instances

Imagine a task that usually takes 30 seconds but occasionally takes 3 minutes under load. With `everyMinute()`, cron will happily launch a *second* copy while the first is still running — leading to duplicate work, race conditions, or resource exhaustion. `withoutOverlapping()` prevents this by acquiring a lock:

```php
Schedule::command('report:generate')->everyMinute()->withoutOverlapping();

// Custom lock expiry in minutes (default 1440 = 24h)
Schedule::command('long:task')->everyMinute()->withoutOverlapping(10);
```

If a task crashes without releasing its lock, the lock auto-expires after the TTL (default 24 hours, or your custom value) so the task isn't stuck forever.

> **Under the hood.** `withoutOverlapping` uses Laravel's **atomic cache lock** (your application's cache store). The `database`, `redis`, `memcached`, and `dynamodb` drivers support atomic locks and persist across the separate `schedule:run` processes that cron spawns each minute. The `array` driver is in-memory and per-process, so its lock vanishes the instant `schedule:run` exits — useless for overlap protection. The `file` driver is also not a reliable cross-process lock store. Use `redis` or `database` in production. The lock key is derived from a hash of the task (its command/closure signature, plus any `name()` you set), so two *different* tasks don't block each other.
>
> If a lock ever gets stuck (e.g. a server died mid-task), clear it with `php artisan schedule:clear-cache`.

### 5.2 `onOneServer()` — coordinate across a fleet

If you deploy the same codebase to multiple app servers, each one has the cron entry, so each one runs `schedule:run` — meaning a `daily()` email would be sent *N* times. `onOneServer()` ensures only the first server to grab the lock for that minute actually runs the task:

```php
Schedule::command('reports:email')->daily()->onOneServer();
```

Requirements and notes:

- Requires a **shared, lock-capable cache** across all servers: `redis`, `database`, `memcached`, or `dynamodb`. The `file`/`array` drivers are per-server and won't coordinate.
- Each scheduled task you mark `onOneServer()` must be **uniquely named**. Laravel derives a name automatically for commands, but anonymous closures need an explicit name:

```php
Schedule::call(fn () => Stats::compute())
    ->daily()
    ->onOneServer()
    ->name('compute-daily-stats');   // required for closures on one server
```

You can override which cache store the scheduler uses for its single-server locks with `Schedule::useCache('database')` — handy when your default cache is per-server (e.g. `array` in tests) but you want locks in a shared store:

```php
// routes/console.php (or a provider boot)
Schedule::useCache('redis');
```

> **`withoutOverlapping` vs `onOneServer`.** They solve different problems. `withoutOverlapping` prevents a task overlapping *itself across time* (on one machine). `onOneServer` prevents the *same minute's run* happening on *multiple machines*. Heavy production schedules often use both.

### 5.3 `runInBackground()` — don't block the queue of tasks

`schedule:run` executes tasks **sequentially** by default. A long-running task delays everything defined after it. `runInBackground()` forks the task into a separate OS process so the scheduler can move on:

```php
Schedule::command('slow:analytics')->everyMinute()->runInBackground();
```

Only use it with `command()` and `exec()` tasks (it has no effect on `call()` closures). Combine it with `withoutOverlapping()` for the safest heavy-task pattern:

```php
Schedule::command('analytics:build')
    ->hourly()
    ->withoutOverlapping()
    ->runInBackground();
```

### 5.4 Maintenance mode — `evenInMaintenanceMode()`

When your app is in maintenance mode (`php artisan down`), **scheduled tasks do not run** — Laravel does not want background work interfering with whatever you are deploying or fixing. If a specific task *must* run even during maintenance, opt it in:

```php
Schedule::command('heartbeat:ping')->everyMinute()->evenInMaintenanceMode();
```

### 5.5 Schedule groups — share config across tasks

When several tasks share the same settings, define them once with `group()` instead of repeating chains on every task. Call the shared configuration methods first, then `group()` with a closure that defines the tasks:

```php
Schedule::daily()
    ->onOneServer()
    ->timezone('America/New_York')
    ->group(function () {
        Schedule::command('emails:send --force');
        Schedule::command('emails:prune');
    });
```

Both commands inherit `daily()`, `onOneServer()`, and the timezone — DRYer and less error-prone than copy-pasting the chain.

---

## 6. Capturing task output

By default, output from scheduled commands goes nowhere. You can capture it:

```php
// Write stdout to a file (overwrites each run)
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo('/var/log/report.log');

// Append instead of overwrite
Schedule::command('report:generate')
    ->daily()
    ->appendOutputTo('/var/log/report.log');

// Email the output after the task runs (capture to a file FIRST)
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo('/var/log/report.log')
    ->emailOutputTo('ops@example.com');

// Only email when the task FAILS (non-zero exit code) — no sendOutputTo needed
Schedule::command('report:generate')
    ->daily()
    ->emailOutputOnFailure('ops@example.com');
```

`emailOutputTo()` builds on the captured output, so you should pair it with `sendOutputTo()` (as above), and it requires a configured mail mailer. `emailOutputOnFailure()` does not need a prior `sendOutputTo()` and is the practical choice for most ops setups — silence on success, alert on failure.

> **Gotcha.** All four output methods — `sendOutputTo`, `appendOutputTo`, `emailOutputTo`, and `emailOutputOnFailure` — are **exclusive to `command()` and `exec()`** tasks. They cannot capture output from `job()` or `call()` closures, because those don't produce a captured process stream. For a queued job, handle notification inside the job (e.g. its `failed()` method).

---

## 7. Lifecycle hooks and heartbeat monitoring

Hooks let you run code around a task and ping external monitors. Order of evaluation: `before` → task → (`onSuccess` or `onFailure`) → `after`.

```php
use Illuminate\Support\Stringable;

Schedule::command('backup:run')
    ->daily()
    ->before(function () {
        Log::info('Backup starting');
    })
    ->after(function () {
        Log::info('Backup finished (success or failure)');
    })
    ->onSuccess(function (Stringable $output) {
        // $output contains the command's captured output
        Log::info('Backup succeeded', ['output' => (string) $output]);
    })
    ->onFailure(function (Stringable $output) {
        Notification::route('slack', config('alerts.slack'))
            ->notify(new BackupFailed((string) $output));
    });
```

> The `$output` argument is available in the `after`, `onSuccess`, and `onFailure` hooks (type-hint `Illuminate\Support\Stringable`); it is **not** passed to `before` (the task hasn't produced output yet). It is only populated for `command()`/`exec()` tasks, where Laravel can capture the process output.

### Heartbeat / dead-man's-switch monitoring

External uptime services like **Envoyer**, **Oh Dear**, **Healthchecks.io**, or **Cronitor** work on a "ping me when you run; alert me if you *don't*" model. Laravel has dedicated ping hooks that fire an HTTP request:

```php
Schedule::command('reports:nightly')
    ->daily()
    ->pingBefore('https://hc-ping.com/your-uuid/start')   // ping when starting
    ->thenPing('https://hc-ping.com/your-uuid');          // ping when finished

// Success/failure pings
Schedule::command('x')
    ->daily()
    ->pingOnSuccess('https://example.com/ok')
    ->pingOnFailure('https://example.com/fail');

// Conditional variants: ping only when $condition is true
Schedule::command('x')
    ->daily()
    ->pingBeforeIf($condition, 'https://example.com/start')
    ->thenPingIf($condition, 'https://example.com/done')
    ->pingOnSuccessIf($condition, 'https://example.com/ok')
    ->pingOnFailureIf($condition, 'https://example.com/fail');
```

If `schedule:run` never executes (server down, cron broken), the monitor never gets pinged and alerts you. This is the standard way to detect a *silently dead* scheduler — something internal logging can never catch.

> **Requirement.** The ping hooks need the **Guzzle HTTP client** (`composer require guzzlehttp/guzzle`), which is included in a default Laravel install.

---

## 8. Running the scheduler: the one cron entry

In **production**, add exactly one crontab line on each server (`crontab -e`):

```bash
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

- `* * * * *` = every minute.
- `cd /path-to-your-project &&` ensures the working directory is correct.
- `>> /dev/null 2>&1` discards cron's own output (your tasks handle their own output via the methods in §6).

That's the *only* cron entry you ever need, no matter how many tasks you define. On platforms like **Laravel Forge**, this entry is created automatically. In **Docker/Kubernetes**, you typically run a dedicated container executing `php artisan schedule:work` (see below) or a sidecar cron.

> **Sail / local Docker.** `./vendor/bin/sail` users can run `sail artisan schedule:work` in a terminal during development.

### `schedule:work` — the dev-friendly foreground runner

You do *not* want to edit your local crontab for development. Instead, run:

```bash
php artisan schedule:work
```

This is a long-running foreground process that invokes `schedule:run` every minute (and handles sub-minute tasks) until you stop it with Ctrl+C. Perfect for local dev and for container-based deployments where a persistent process is easier than a crontab.

### `schedule:interrupt` — stop an in-flight run after deploys

When **sub-minute tasks** are defined, `schedule:run` stays alive for the whole minute (re-checking second-level tasks). That means a `schedule:run` started just before a deploy keeps running your *old* code until the minute ends. Add `schedule:interrupt` to the end of your deploy script to gracefully stop any in-progress run so the next minute picks up the new code:

```bash
php artisan schedule:interrupt
```

---

## 9. Inspecting and testing schedules

### `schedule:list` — see everything at a glance

```bash
php artisan schedule:list
```

```text
Output:
  0 7 * * *  php artisan digest:send ........... Next Due: 9 hours from now
  */5 * * * *  php artisan payments:retry ....... Next Due: 2 minutes from now
  0 0 * * 0  App\Jobs\PruneOldRecords .......... Next Due: 3 days from now
```

It prints every registered task, its cron expression, its target, and when it next runs — the fastest way to sanity-check your schedule. Add `--timezone=America/New_York` to view "next due" in a specific zone.

### `schedule:test` — run a single task on demand

Rather than waiting for a task to be "due," run it immediately:

```bash
php artisan schedule:test
# → presents an interactive list of tasks to pick from

php artisan schedule:test --name="reports:nightly"
# → runs that specific task now, regardless of its schedule
```

This executes the task *as the scheduler would* (including its constraints being shown), which is invaluable for debugging "why didn't my task run."

### Testing in PHPUnit / Pest

You can assert scheduling behavior in tests using time travel and standard fakes:

```php
use Illuminate\Support\Facades\Queue;

test('monthly invoice job is dispatched on the 1st', function () {
    Queue::fake();

    // Travel to the 1st of next month at 02:00
    $this->travelTo(now()->addMonthNoOverflow()->startOfMonth()->setTime(2, 0));

    $this->artisan('schedule:run');

    Queue::assertPushed(\App\Jobs\GenerateMonthlyInvoices::class);
});
```

> Combine `$this->travelTo()` (Laravel's time-travel helper, built on Carbon) with `Queue::fake()`, `Mail::fake()`, or `Bus::fake()` to assert that a task fires at the right moment without real side effects. The default test runner in fresh Laravel 11/12 apps is **Pest** (the `test('...', function () { ... })` syntax above); the PHPUnit class style works identically — `$this->travelTo(...)` and `$this->artisan(...)` are available in both because Pest binds `$this` to the underlying `TestCase`.

### Scheduler events

Laravel dispatches events throughout a scheduled run, so you can centralize logging/metrics with an event listener instead of per-task hooks:

| Event | Fires when |
| --- | --- |
| `Illuminate\Console\Events\ScheduledTaskStarting` | A task is about to run |
| `Illuminate\Console\Events\ScheduledTaskFinished` | A task finished |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` | A `runInBackground()` task finished |
| `Illuminate\Console\Events\ScheduledTaskSkipped` | A task was skipped by a constraint (`when`/`skip`/overlap/etc.) |
| `Illuminate\Console\Events\ScheduledTaskFailed` | A task threw / exited non-zero |

---

## ⚠️ Common Mistakes & Gotchas

1. **Forgetting the single cron entry entirely.** You define beautiful schedules in `routes/console.php`, deploy, and… nothing runs. The scheduler does *nothing* on its own — it needs `* * * * * php artisan schedule:run` in the server's crontab (or a running `schedule:work`). **Fix:** add the cron entry on every production server, or run `schedule:work` in a persistent container.

2. **Running `schedule:run` more or less than every minute.** Laravel's frequency logic *assumes* it is invoked exactly once per minute. If your crontab says `*/5 * * * *`, then `everyMinute()` tasks fire only every 5 minutes and `dailyAt('03:02')` may be skipped entirely (the minute it's due is never evaluated). **Fix:** always schedule `schedule:run` with `* * * * *`.

3. **Duplicate runs across multiple servers.** Deploy to 3 servers, each with the cron entry, and a `daily()` email gets sent 3 times. **Fix:** add `->onOneServer()` (and use a shared redis/database cache). Do *not* try to solve this by only adding cron to one server — that creates a single point of failure.

4. **`withoutOverlapping()` lock never releasing.** If a task is force-killed (OOM, `kill -9`) it can't release its lock, blocking future runs. **Fix:** rely on the default TTL or set an explicit one — `withoutOverlapping(30)` — and ensure your cache driver supports atomic locks (use `redis`/`database`, not `file`).

5. **Timezone confusion.** A task with `dailyAt('09:00')` runs at 9am in `config('app.timezone')` (often `UTC`), not the user's local time. Teams routinely ship reports "an hour early/late" because of this. **Fix:** set `->timezone('America/New_York')` explicitly on time-sensitive tasks, and be aware of DST skips/repeats — prefer UTC for critical jobs.

6. **Heavy closures or `when()` checks blocking the scheduler.** Every `call()` closure and every `when()`/`skip()` constraint runs *inside* `schedule:run` every minute. A slow query there stalls all subsequent tasks. **Fix:** keep constraints cheap; push heavy work into `job()` and add `runInBackground()` to long `command()` tasks.

7. **Expecting `emailOutputTo()` to work on jobs/closures.** Output capture only works for `command()` and `exec()`. **Fix:** for queued jobs, handle notifications inside the job itself (e.g. in its `failed()` method).

8. **Scheduled tasks silently stop during maintenance mode.** After `php artisan down`, the scheduler skips *every* task. A monitoring/heartbeat job that "stopped pinging" right after a deploy is often just maintenance mode. **Fix:** mark the few tasks that must keep running with `->evenInMaintenanceMode()`, and bring the app back up with `php artisan up`.

9. **Reaching for `Schedule::MONDAY` on the facade.** The day constants live on the *concrete* `Illuminate\Console\Scheduling\Schedule` class, not the `Illuminate\Support\Facades\Schedule` facade — using the facade alias raises an "undefined constant" error. **Fix:** import the concrete class for the constants (or just use the integers, `0`=Sunday … `6`=Saturday).

10. **Confusing `weeklyOn`/`days` numbering.** These use the cron convention where **Sunday is 0**. Passing `1` expecting "Sunday" (ISO/Carbon week-start confusion) silently runs on the wrong day. **Fix:** use the `Schedule::SUNDAY…SATURDAY` constants, and remember `weeklyOn(1, ...)` is Monday.

---

## ✅ Best Practices

- **Keep `schedule:run` the only crontab line.** Define everything in code so it's version-controlled and reviewable.
- **Prefer `job()` for anything heavy.** Let the queue carry the load; keep `schedule:run` fast.
- **Reference commands by `::class`,** not magic strings, for refactor safety and IDE support.
- **Always add `withoutOverlapping()` to frequent tasks** that might run long (anything `everyMinute`/`everyFiveMinutes` doing real work).
- **Use `onOneServer()` in any multi-server deployment** for tasks that must run exactly once, backed by a shared redis/database cache.
- **Add a heartbeat monitor** (`pingBefore`/`thenPing` to Healthchecks.io, Oh Dear, etc.) so you learn when the scheduler dies silently.
- **Set explicit timezones** on user-facing time-sensitive tasks; default everything else to UTC.
- **Name your closures** (`->name('...')`) when using `onOneServer`/`withoutOverlapping` so locks are stable and `schedule:list` is readable.
- **Keep `when()`/`skip()` closures and `call()` closures cheap** — no slow I/O inside the per-minute hot path.
- **Use `emailOutputOnFailure()` (not `emailOutputTo`)** to get signal without inbox noise.
- **Make sub-minute tasks dispatch jobs or run in the background** — never do heavy inline work in an `everySecond()`/`everyTenSeconds()` task, or you stall the rest of the minute.
- **Run `php artisan schedule:interrupt` at the end of your deploy script** if you use sub-minute tasks, so an in-flight `schedule:run` stops running stale code.
- **Group tasks that share config** with `->group(...)` instead of repeating `onOneServer()`/`timezone()` on each.

---

## 🎯 Interview Tips & Likely Questions

**Q1. Does Laravel's scheduler replace cron?**
A. No. It sits *on top of* cron. You still register one cron entry — `* * * * * php artisan schedule:run` — and Laravel decides, each minute, which of your in-code tasks are due. The benefit is that all task definitions live in version-controlled PHP instead of scattered crontab lines.

**Q2. How does `schedule:run` actually work under the hood?**
A. Cron invokes `schedule:run` every minute. The command loads all tasks registered via the `Schedule` facade (`routes/console.php`) and/or the Kernel `schedule()` method, then for each task evaluates its cron expression against the current time and runs the ones that are due. Tasks run sequentially unless `runInBackground()` forks them. If sub-minute tasks (`everySecond`, etc., Laravel 11+) exist, the process stays alive for the full minute and re-checks them. Locks (`withoutOverlapping`/`onOneServer`) are atomic cache locks.

**Q3. Where do you define schedules in Laravel 12 vs Laravel 10?**
A. Laravel 11/12: in `routes/console.php` (or a provider's `boot()`) using the `Schedule` facade — `app/Console/Kernel.php` was removed from the default skeleton. Laravel 10 and earlier: in the `schedule(Schedule $schedule)` method of `app/Console/Kernel.php`. The fluent frequency API is identical across both.

**Q4. What's the difference between `withoutOverlapping()` and `onOneServer()`?**
A. `withoutOverlapping()` stops a task from overlapping *itself in time* on one machine (the new run is skipped while the previous is still running). `onOneServer()` stops the *same scheduled run* from executing on *multiple machines* in a fleet. Both use atomic cache locks and need a lock-capable shared cache (redis/database) — `onOneServer` *requires* the cache to be shared across servers.

**Q5. Why might a task silently fail to run, and how do you debug it?**
A. Common causes: no cron entry / `schedule:run` not running every minute; a `when()`/`skip()`/`environments()` constraint filtering it out; wrong timezone; an unreleased `withoutOverlapping` lock. Debug with `php artisan schedule:list` (confirm it's registered and see next-due) and `php artisan schedule:test --name=...` (force-run it now). A heartbeat monitor catches the "scheduler is entirely dead" case.

**Q6. How do you run something every 5 minutes? Multiple ways?**
A. `->everyFiveMinutes()`, or the raw `->cron('*/5 * * * *')`. The fluent helper is preferred for readability; `cron()` is the escape hatch for expressions the helpers can't represent.

**Q7. When would you use `job()` instead of `command()`?**
A. When the work is heavy or slow. `job()` dispatches to the queue and returns immediately, so `schedule:run` stays fast and isn't blocked. `command()` runs inline (unless `runInBackground()`). The trade-off: `job()` needs a running queue worker.

**Q8. How do you make output go somewhere useful?**
A. `sendOutputTo()`/`appendOutputTo()` for files, `emailOutputTo()` for every run (pair it with `sendOutputTo()`, since it emails the captured file and needs a configured mailer), `emailOutputOnFailure()` for failures only. All four only work for `command()`/`exec()` tasks. For jobs/closures, handle notification inside the code itself.

**Q9. What is a heartbeat / dead-man's-switch and how does Laravel support it?**
A. An external service expects a periodic ping; if the ping stops, it alerts you. This detects a *dead scheduler*, which internal logs can't. Laravel provides `pingBefore()`, `thenPing()`, `pingOnSuccess()`, `pingOnFailure()`, plus the conditional `...If($condition, $url)` variants. Pings use the bundled Guzzle HTTP client.

**Q10. What changed about scheduling in Laravel 11?**
A. Two big things: schedules moved from `app/Console/Kernel.php` to `routes/console.php` via the `Schedule` facade (Kernel removed from the skeleton; you can alternatively use `withSchedule()` in `bootstrap/app.php`), and **sub-minute scheduling** (`everySecond()` … `everyThirtySeconds()`) was introduced (along with `schedule:interrupt` to stop in-flight sub-minute runs).

**Q11. Do scheduled tasks run while the app is in maintenance mode?**
A. No — `php artisan down` pauses the whole schedule. Opt a specific task back in with `->evenInMaintenanceMode()` (e.g. a heartbeat that must keep firing).

**Q12. How do you avoid repeating `onOneServer()`/`timezone()` on many tasks?**
A. Use a schedule group: chain the shared config, then `->group(function () { ... })` and define the tasks inside. Every task in the closure inherits the outer settings.

---

## 📋 Quick Reference / Cheat Sheet

```php
// ---- Define (Laravel 11/12: routes/console.php) ----
use Illuminate\Support\Facades\Schedule;

Schedule::command(MyCommand::class)->daily();      // Artisan command
Schedule::job(new MyJob)->hourly();                // Queued job
Schedule::call(fn () => doThing())->everyMinute(); // Closure
Schedule::exec('rsync ...')->daily();              // Shell command

// ---- Frequencies (a selection); day numbering 0=Sun … 6=Sat ----
->everySecond() ->everyTenSeconds() ->everyMinute() ->everyFiveMinutes()
->hourly() ->hourlyAt(15) ->everyOddHour() ->everyTwoHours()
->daily() ->dailyAt('13:00') ->twiceDaily(1, 13) ->twiceDailyAt(1, 13, 15)
->daysOfMonth([1, 15])
->weekly() ->weeklyOn(1, '08:00')          // weeklyOn: 1 = Monday
->monthly() ->monthlyOn(4, '15:00') ->twiceMonthly(1, 16, '13:00') ->lastDayOfMonth('15:00')
->quarterly() ->quarterlyOn(4, '14:00') ->yearly() ->yearlyOn(6, 1, '17:00')
->cron('*/5 * * * *')

// ---- Day / time constraints ----
->weekdays() ->weekends() ->mondays() ->days([0, 3])   // 0=Sun, 3=Wed
->timezone('America/New_York')             // or config app.schedule_timezone
->between('7:00', '22:00') ->unlessBetween('1:00', '2:00')
->when(fn () => $cond) ->skip(fn () => $cond)
->environments(['production', 'staging'])
->evenInMaintenanceMode()

// ---- Production controls ----
->withoutOverlapping()        // ->withoutOverlapping(10) custom TTL minutes (default 1440)
->onOneServer()               // multi-server; needs shared redis/db/memcached/dynamodb cache
->runInBackground()           // fork; command()/exec() only
->name('stable-lock-name')    // required for closures w/ onOneServer/withoutOverlapping

// ---- Output (command()/exec() only) ----
->sendOutputTo($path) ->appendOutputTo($path)
->sendOutputTo($path)->emailOutputTo($addr)   // emailOutputTo pairs with sendOutputTo
->emailOutputOnFailure($addr)

// ---- Hooks & monitoring ----
->before(fn () => ...) ->after(fn (Stringable $o) => ...)
->onSuccess(fn (Stringable $o) => ...) ->onFailure(fn (Stringable $o) => ...)
->pingBefore($url) ->thenPing($url)
->pingOnSuccess($url) ->pingOnFailure($url)
->pingBeforeIf($cond, $url) ->thenPingIf($cond, $url)

// ---- Grouping shared config ----
Schedule::daily()->onOneServer()->group(function () {
    Schedule::command('a'); Schedule::command('b');
});
```

```bash
# ---- The one cron entry (production) ----
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1

# ---- CLI tools ----
php artisan schedule:run                  # run due tasks once (cron calls this)
php artisan schedule:work                 # long-running runner (local/containers)
php artisan schedule:list                 # show all tasks + next-due (--timezone=...)
php artisan schedule:test                 # interactively run one task now
php artisan schedule:test --name="x:y"    # run a named task now
php artisan schedule:interrupt            # stop an in-flight run (deploy + sub-minute)
php artisan schedule:clear-cache          # clear stuck overlap/one-server locks
```

---

## 🧪 Mini Exercises

1. **Nightly digest with safety rails.** Create an Artisan command `digest:send` and schedule it to run every day at 07:00 in `America/New_York`, but only on weekdays. Add `withoutOverlapping()` and email the output to your ops address only on failure. Verify with `schedule:list` and `schedule:test --name=digest:send`.

2. **Multi-server-safe cache warm.** Schedule a closure that warms a cache key every fifteen minutes, but only between 06:00 and 23:00, only on one server in a fleet, and named `warm-cache`. Explain (in a comment) which cache driver(s) make `onOneServer()` actually work and why `file` does not.

3. **Heavy job, off the hot path.** You have a `RebuildSearchIndex` job that takes ~4 minutes. Schedule it hourly such that `schedule:run` is never blocked by it, it never overlaps a previous run, and a Slack notification fires if it fails. Decide whether `runInBackground()` is relevant here and justify your choice.

4. **Heartbeat wiring.** Take any daily command and wire it to Healthchecks.io: ping a `/start` URL before it runs and the base URL when it finishes, plus a failure ping. Describe what happens (from the monitor's perspective) if the server's cron daemon stops entirely.

5. **Version archaeology.** Given a Laravel 10 app that defines schedules in `app/Console/Kernel.php`, write the *equivalent* `routes/console.php` for Laravel 12 for these tasks: `inspire` hourly, `PruneOldRecords` job daily at 02:00 on one server, and a closure clearing `recent_visits` every five minutes without overlapping. Note any line that requires an explicit `->name()`.

6. **Maintenance-safe heartbeat.** You run `php artisan down` during a deploy. Schedule a `heartbeat:ping` command every minute that keeps firing *even during maintenance*, and explain why the other scheduled tasks correctly pause. Which single method makes the difference?

7. **DRY with a group.** You have three reporting commands that all need to run `daily()`, `onOneServer()`, in `America/New_York`. Rewrite them using a schedule group so the shared config appears exactly once. Then add a fourth command that needs the same config *plus* `withoutOverlapping()` — decide whether it belongs in the group or stands alone, and justify it.

8. **Sub-minute, done right.** Schedule work that must happen every ten seconds. Show the *wrong* way (heavy logic inline in a `call()` closure) and the *right* way (dispatch a queued job or use `runInBackground()`), and explain in a comment why the inline version can starve later sub-minute ticks. Mention which command you would add to your deploy script and why.
