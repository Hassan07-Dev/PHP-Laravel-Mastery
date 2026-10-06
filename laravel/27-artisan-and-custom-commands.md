# Artisan & Custom Commands

**Artisan** is Laravel's command-line interface (CLI). It ships with every Laravel application as an executable PHP script named `artisan` in the project root. You use it to scaffold code, run migrations, inspect routes, manage caches, run background workers, and — most importantly for senior engineers — to write your own **custom commands** that automate the chores specific to your application (nightly reports, data backfills, cleanup jobs, one-off maintenance scripts).

Under the hood Artisan is a thin Laravel wrapper around the **Symfony Console** component, which is why much of its vocabulary (commands, arguments, options, input/output, exit codes) matches Symfony's. Understanding that lineage helps you reason about edge cases.

> **What you'll learn**
> - What Artisan is, how it relates to Symfony Console, and how to discover commands with `list` and `help`
> - The everyday commands you'll run constantly: `migrate`, the `make:*` family, `tinker`, `route:list`, the cache/optimize commands, `queue:work`, `schedule:run`, `db:seed`, `storage:link`, `key:generate`, and `about`
> - How to explore your app interactively with the **Tinker** REPL
> - How to author a custom command with `make:command`: the **signature** string (arguments and options in all their forms), the description, and the `handle()` method
> - Reading input (`argument`/`option`) and producing rich output (`info`/`error`/`warn`/`line`/`table`/`newLine`, progress bars)
> - Interactive prompts — both the classic helpers (`ask`/`secret`/`confirm`/`choice`/`anticipate`) and the modern **Laravel Prompts** package
> - Calling commands from other commands, scheduling commands, auto-discovery, **isolatable** commands, exit codes, and **testing** commands with `expectsQuestion` and `assertExitCode`

---

## 1. Why a CLI at all?

A web framework's job is to respond to HTTP requests, but a huge amount of real work happens *outside* a request: running database migrations during deploy, generating boilerplate, sending a nightly digest email, reindexing a search engine, pruning stale rows. Doing these through a browser would be awkward, insecure, and untestable. A CLI gives you:

- **Automation**: commands run from deploy scripts, cron, CI pipelines, and supervisors with no human in the loop.
- **Scaffolding**: `make:*` commands generate consistent, convention-correct files so you don't hand-type boilerplate.
- **Operational tooling**: cache warmers, queue workers, and schedulers are all CLI processes.
- **A REPL** (`tinker`) for poking at your app's objects live.

Because Artisan commands run inside a fully booted Laravel application, they have access to the **service container**, Eloquent models, config, queues, and everything else your HTTP code does. That is the key mental model: *a command is just another entry point into your application*.

---

## 2. Discovering commands: `list` and `help`

Running `artisan` with no arguments (or `php artisan list`) prints every registered command, grouped by namespace (the part before the colon, e.g. `make`, `route`, `db`).

```bash
php artisan
# or, identically:
php artisan list
```

```text
Laravel Framework 12.x.x

Usage:
  command [options] [arguments]

Available commands:
  about                 Display basic information about your application
  clear-compiled        Remove the compiled class file
  completion            Dump the shell completion script
  db                    Start a new database CLI session
  docs                  Access the Laravel documentation
  ...
 make
  make:command          Create a new Artisan command
  make:controller       Create a new controller class
  ...
 route
  route:list            List all registered routes
```

To see how a *specific* command works — its arguments, options, and defaults — use `help`:

```bash
php artisan help migrate
# equivalent:
php artisan migrate --help
```

```text
Description:
  Run the database migrations

Usage:
  migrate [options]

Options:
      --database[=DATABASE]  The database connection to use
      --force                Force the operation to run when in production
      --path[=PATH]          The path(s) to the migrations files to be executed (multiple values allowed)
      --pretend              Dump the SQL queries that would be run
      --seed                 Indicates if the seed task should be re-run
      --step                 Force the migrations to be run so they can be rolled back individually
  ...
```

Useful global flags available on **every** command:

| Flag | Meaning |
|------|---------|
| `-h`, `--help` | Show help for the command |
| `-q`, `--quiet` | Suppress all output |
| `-v`, `-vv`, `-vvv` | Increase verbosity (the most verbose shows full exception stack traces) |
| `-n`, `--no-interaction` | Never prompt; use defaults (vital in CI/cron) |
| `--ansi` / `--no-ansi` | Force or disable colored output |
| `-V`, `--version` | Print the framework version |
| `--env=...` | Choose the environment (rarely needed; usually set via `APP_ENV`) |

> **Tip:** In CI and cron, always pass `--no-interaction` (and often `--force`) so a command never blocks waiting for a prompt.

---

## 3. The most-used built-in commands

You will type these dozens of times a day. Group them mentally by purpose.

### 3.1 Database & data

```bash
php artisan migrate                 # Run pending migrations
php artisan migrate --pretend       # Print the SQL without executing it
php artisan migrate:rollback        # Roll back the last "batch" of migrations
php artisan migrate:fresh           # Drop ALL tables, then migrate from scratch
php artisan migrate:fresh --seed    # ...and run seeders afterward
php artisan db:seed                 # Run database seeders (DatabaseSeeder by default)
php artisan db:seed --class=UserSeeder
php artisan db                      # Open an interactive DB CLI session (mysql/psql/sqlite)
```

> `migrate:fresh` drops every table including ones not created by migrations. Never run it in production. `migrate:refresh` rolls back and re-runs (safer history) but is still destructive to data.

### 3.2 Scaffolding — the `make:*` family

```bash
php artisan make:model Post -mfsc   # model + migration + factory + seeder + controller (combined flags)
php artisan make:controller PostController --resource
php artisan make:migration create_posts_table
php artisan make:request StorePostRequest
php artisan make:command SendNewsletter
php artisan make:job ProcessPodcast
php artisan make:event OrderShipped
php artisan make:listener SendShipmentNotification
php artisan make:middleware EnsureTokenIsValid
php artisan make:policy PostPolicy --model=Post
php artisan make:enum Status
php artisan make:test PostTest        # add --unit for a unit test
```

The `make:model` combined flags are worth memorizing: `-m` migration, `-f` factory, `-s` seeder, `-c` controller, `-r` resource controller, `--all`/`-a` for everything, `--pivot` for a pivot model.

### 3.3 The REPL

```bash
php artisan tinker                  # interactive PHP shell with your app booted
```

Covered in depth in section 4.

### 3.4 Inspecting the app

```bash
php artisan route:list                       # all routes, methods, names, middleware
php artisan route:list --path=api            # filter by URI substring
php artisan route:list --method=GET --except-vendor
php artisan about                            # environment, versions, drivers, cache state
php artisan about --only=environment         # just one section
php artisan env                              # current APP_ENV
```

`php artisan about` is your one-stop diagnostic — it shows PHP/Laravel versions, the active cache/queue/session/mail drivers, and whether config/routes/events are cached.

```text
  Environment ......................................................
  Application Name ................................... Laravel
  Laravel Version .................................... 12.x.x
  PHP Version ........................................ 8.4.x
  Composer Version ................................... 2.x.x
  Environment ........................................ local
  Debug Mode ......................................... ENABLED
  Cache ..............................................
  Config ............................................. NOT CACHED
  Routes ............................................. NOT CACHED
  Drivers ............................................
  Cache .............................................. database
  Queue .............................................. database
```

### 3.5 Performance: caching & optimizing

```bash
php artisan config:cache     # Merge all config into one cached file (fast boot)
php artisan config:clear     # Remove the config cache
php artisan route:cache      # Serialize routes for fast registration
php artisan route:clear
php artisan view:cache       # Precompile all Blade templates
php artisan event:cache      # Cache discovered event listeners
php artisan optimize         # Run config:cache + route:cache + view:cache + event:cache
php artisan optimize:clear   # Clear ALL of the above plus the application/compiled caches
```

> **Critical gotcha:** once you run `config:cache`, the framework stops reading your `.env` file at runtime — all values come from the cached config array. Any later `env()` call *outside* a config file returns `null`. Always read configuration through `config('services.x')`, never `env()` directly in app code. Run `optimize` in production deploys, but run `optimize:clear` (or just never cache config) in local dev so `.env` edits take effect.

### 3.6 Queues & scheduling

```bash
php artisan queue:work                 # Long-running worker that processes jobs
php artisan queue:work --queue=high,default --tries=3 --timeout=90
php artisan queue:listen               # Like work but reloads code each job (dev only, slower)
php artisan queue:retry all            # Re-push failed jobs
php artisan queue:failed               # List failed jobs
php artisan schedule:run               # Runs all due scheduled tasks; cron calls this every minute
php artisan schedule:work              # Foreground scheduler for local dev (no cron needed)
php artisan schedule:list              # Show the schedule and next run times
```

A worker (`queue:work`) boots the framework **once** and keeps it in memory for speed; that's why you must restart workers after deploying new code (`php artisan queue:restart`). `queue:listen` reboots per job, so it's slower but always runs fresh code — handy in development.

### 3.7 App setup & maintenance

```bash
php artisan key:generate          # Generate APP_KEY (32-byte base64) used for encryption/cookies
php artisan storage:link          # Symlink public/storage -> storage/app/public
php artisan down                  # Put the app into maintenance mode
php artisan up                    # Bring it back
php artisan down --secret="bypass-token"   # Allow a bypass URL while down
```

`key:generate` runs automatically right after `composer create-project`. If `APP_KEY` is missing you'll get "No application encryption key has been specified" on any encrypted cookie/session operation. `storage:link` makes files saved to the `public` disk reachable at `/storage/...` URLs.

---

## 4. The Tinker REPL

**Tinker** (`php artisan tinker`) is a REPL — a *Read-Eval-Print Loop* — built on the PsySH shell. It boots your full application, so you can run live PHP against your models, services, and config. It's the fastest way to answer "what does this return?" without writing a script.

```bash
php artisan tinker
```

```php
>>> User::count()
=> 42

>>> $u = User::factory()->create(['name' => 'Ada'])
=> App\Models\User {#... name: "Ada", ...}

>>> $u->name
=> "Ada"

>>> config('app.timezone')
=> "UTC"

>>> collect([1, 2, 3])->map(fn ($n) => $n * 2)->sum()
=> 12

>>> cache()->put('greeting', 'hi', 60); cache('greeting')
=> "hi"
```

Handy Tinker tricks:

- `>>> exit` or `Ctrl+D` to leave.
- `>>> doc User::find` shows the docblock; `>>> show User` dumps the class source.
- `>>> ls` lists variables/methods in scope.
- Run a one-off expression without entering the shell:

```bash
php artisan tinker --execute="echo User::count();"
```

> **Warning:** Tinker runs with your real environment by default. In production it talks to the live database — a stray `User::truncate()` is unrecoverable. Treat it like a loaded gun.

---

## 5. Writing your first custom command

Generate the skeleton:

```bash
php artisan make:command SendNewsletter
```

This creates `app/Console/Commands/SendNewsletter.php`:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class SendNewsletter extends Command
{
    /**
     * The name and signature of the console command.
     */
    protected $signature = 'app:send-newsletter';

    /**
     * The console command description.
     */
    protected $description = 'Command description';

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        //
    }
}
```

Two properties and one method matter:

- **`$signature`** — the command's name plus its arguments and options, written in a compact DSL (domain-specific language) string. This is what you type after `php artisan`.
- **`$description`** — the one-line summary shown in `php artisan list` and `help`.
- **`handle()`** — the body that runs. Anything you `return` here becomes the **exit code** (more in section 11). Returning nothing/`void` is treated as exit code `0` (success).

> Laravel 12 generates `app:`-prefixed names by default to avoid clashing with framework commands. You can name it anything you like — `newsletter:send`, `reports:nightly`, etc.

---

## 6. The signature string in depth

The signature is the heart of a command. Its first token is the **name**; everything after defines **arguments** (positional, in `{}`) and **options** (flags, in `{--...}`).

### 6.1 Arguments

Arguments are positional values typed after the command name.

```php
// Required argument
protected $signature = 'mail:send {user}';

// Optional argument (note the trailing ?)
protected $signature = 'mail:send {user?}';

// Optional argument with a default value
protected $signature = 'mail:send {user=guest}';

// Array / variadic argument (one or more values) — note the * 
protected $signature = 'mail:send {user*}';

// Optional array argument
protected $signature = 'mail:send {user?*}';

// With an inline description (shown in help) after a colon
protected $signature = 'mail:send {user : The ID of the user to email}';
```

Usage examples for `{user*}`:

```bash
php artisan mail:send 1 2 3      # $this->argument('user') === ['1', '2', '3']
```

### 6.2 Options

Options are named flags, written with a leading `--`. There are two kinds.

**Switch options** (boolean — present or absent, no value):

```php
protected $signature = 'mail:send {user} {--queue}';
```

```bash
php artisan mail:send 1 --queue   # $this->option('queue') === true
php artisan mail:send 1           # $this->option('queue') === false
```

**Value options** (require a value — note the `=`):

```php
protected $signature = 'mail:send {user} {--queue=}';            // value, default null
protected $signature = 'mail:send {user} {--queue=default}';     // value, default "default"
protected $signature = 'mail:send {user} {--id=*}';              // accepts multiple values (array)
```

```bash
php artisan mail:send 1 --queue=high
php artisan mail:send 1 --id=1 --id=2     # $this->option('id') === ['1', '2']
```

**Shortcuts** — a single-letter alias declared before a `|`:

```php
protected $signature = 'mail:send {user} {--Q|queue}';        // switch shortcut
protected $signature = 'mail:send {user} {--Q|queue=}';       // value-option shortcut
```

```bash
php artisan mail:send 1 -Q          # switch: same as --queue
php artisan mail:send 1 -Qhigh      # value shortcut: same as --queue=high
```

> **Gotcha:** when passing a value to a *shortcut*, you attach the value directly with no `=` and no space — `-Qhigh`, not `-Q=high` or `-Q high`. The long form still uses `--queue=high`.

**Option descriptions** use the same `: description` syntax:

```php
protected $signature = 'mail:send
    {user : The user ID}
    {--Q|queue : Whether the mail should be queued}';
```

Splitting a long signature across multiple lines (as above) is perfectly valid and recommended for readability.

> **Note:** The signature DSL itself is unchanged across Laravel 10, 11, and 12 — what you learn here is stable. If you prefer not to use the string DSL, you can override the Symfony-level `getArguments()` / `getOptions()` methods on the command to define `InputArgument`/`InputOption` objects by hand, but the `$signature` string is by far the common idiom and the one shown throughout the docs. (Do not confuse those with the `arguments()` / `options()` *reader* methods covered in section 7, which return the values the user supplied.)

---

## 7. Reading input

Inside `handle()`, read declared input with `argument()` and `option()`:

```php
public function handle(): int
{
    $userId   = $this->argument('user');     // single value (string) or array for {user*}
    $allArgs  = $this->arguments();           // ['command' => ..., 'user' => ...]

    $queue    = $this->option('queue');       // bool for switches, string/array for value options
    $allOpts  = $this->options();

    // Did the user actually pass this VALUE option on the CLI?
    // NOTE: hasOption() does NOT answer this — it only reports whether the
    // option is DEFINED in the signature (so for 'queue' it is always true).
    // Instead, compare the resolved value against the "not passed" sentinel:
    if (! is_null($this->option('queue'))) {   // for {--queue=} (default null)
        // the user passed --queue=<something>
    }

    // For a boolean switch like {--force}, just read it directly:
    if ($this->option('force')) {
        // ...
    }

    return Command::SUCCESS;
}
```

Everything from the CLI arrives as **strings** (or arrays of strings). Cast deliberately: `(int) $this->argument('user')`.

> **Gotcha:** `$this->hasOption('queue')` / `$this->hasArgument('user')` check whether the option/argument is **declared in the signature**, *not* whether the user supplied it. They almost always return `true` for your own declared inputs, so don't use them to detect presence. To know whether a value option was actually passed, give it a default of `null` (`{--queue=}`) and check `is_null()`; for switches, the value is simply `false` when absent.

---

## 8. Producing output

The base `Command` class gives you styled output helpers. Each renders in a distinct color.

```php
$this->line('Plain text, no color.');
$this->info('Green — success / informational.');
$this->comment('Yellow-ish — secondary note.');
$this->question('Highlighted question style.');
$this->warn('Yellow — a warning.');
$this->error('Red background — something failed.');
$this->alert('Boxed banner heading.');   // big bordered title
$this->newLine();      // blank line
$this->newLine(3);     // three blank lines
```

The optional second argument of `line()` is a **named style** (`'info'`, `'comment'`, `'question'`, `'error'`), so `$this->line('Saved', 'info')` is equivalent to `$this->info('Saved')`. For arbitrary colors, embed a Symfony style tag directly in the string:

```php
$this->line('<fg=cyan;options=bold>Custom styled</>');
```

### 8.1 Tables

`table()` renders an aligned ASCII table — perfect for dumping query results.

```php
$this->table(
    ['ID', 'Name', 'Email'],                          // headers
    User::query()->limit(3)->get(['id', 'name', 'email'])->toArray()  // rows
);
```

```text
+----+-------+------------------+
| ID | Name  | Email            |
+----+-------+------------------+
| 1  | Ada   | ada@example.com  |
| 2  | Alan  | alan@example.com |
| 3  | Grace | grace@example.com|
+----+-------+------------------+
```

### 8.2 Progress bars

For long loops, show progress. The simplest API is `withProgressBar()`:

```php
$users = User::all();

$this->withProgressBar($users, function (User $user) {
    $this->sendMailTo($user);
});

$this->newLine(2);
$this->info('All done!');
```

For finer control, drive the bar manually:

```php
$bar = $this->output->createProgressBar(count($users));
$bar->start();

foreach ($users as $user) {
    $this->sendMailTo($user);
    $bar->advance();
}

$bar->finish();
$this->newLine();
```

```text
 12/12 [============================] 100%
```

---

## 9. Interactive prompts

Commands can ask the user questions when run interactively. (When `--no-interaction` is set, prompts fall back to their defaults.)

### 9.1 Classic helpers

```php
$name     = $this->ask('What is your name?');
$name     = $this->ask('What is your name?', 'Anonymous');   // with default
$password = $this->secret('Enter the deploy token');         // hidden input
$wants    = $this->confirm('Do you wish to continue?');      // returns bool, default false
$wants    = $this->confirm('Continue?', true);               // default true

// Single choice from a list
$role = $this->choice(
    'Which role?',
    ['Admin', 'Editor', 'Viewer'],   // choices
    0,                               // default index (returns the value, e.g. "Admin")
);

// Multiple choice: 4th arg = max attempts, 5th arg = allowMultipleSelections (true)
$roles = $this->choice('Pick roles', ['Admin', 'Editor', 'Viewer'], null, null, true);

// Autocomplete with suggestions (user may still type anything)
$name = $this->anticipate('Name?', ['Ada', 'Alan', 'Grace']);
```

### 9.2 Laravel Prompts (the modern way)

Since Laravel 10.17, the framework bundles the **Laravel Prompts** package — a set of beautiful, validated, accessible prompt functions you import as plain functions. They are the recommended approach in Laravel 11/12. They auto-degrade to the classic helpers on unsupported terminals (e.g. Windows without WSL) and respect `--no-interaction`.

```php
use function Laravel\Prompts\text;
use function Laravel\Prompts\password;
use function Laravel\Prompts\confirm;
use function Laravel\Prompts\select;
use function Laravel\Prompts\multiselect;
use function Laravel\Prompts\search;
use function Laravel\Prompts\spin;

public function handle(): int
{
    $name = text(
        label: 'What is your name?',
        placeholder: 'E.g. Ada Lovelace',
        required: true,
        validate: fn (string $v) => strlen($v) < 3 ? 'At least 3 characters.' : null,
    );

    $token = password(label: 'Deploy token', required: true);

    $confirmed = confirm(label: 'Deploy to production?', default: false);

    $role = select(
        label: 'Role',
        options: ['admin' => 'Admin', 'editor' => 'Editor'],
        default: 'editor',
    );

    $perms = multiselect(
        label: 'Permissions',
        options: ['read', 'write', 'delete'],
    );

    // A spinner while a slow task runs:
    $result = spin(
        message: 'Fetching data...',
        callback: fn () => $this->slowApiCall(),
    );

    return Command::SUCCESS;
}
```

Laravel Prompts shines because of `validate:` (re-prompts on bad input) and `required:` (no empty answers) — things the classic helpers don't give you for free.

### 9.3 Prompting for missing arguments

Instead of erroring when a required argument is omitted, your command can prompt for it automatically by implementing the `PromptsForMissingInput` interface — no extra code needed for the basic case:

```php
use Illuminate\Console\Command;
use Illuminate\Contracts\Console\PromptsForMissingInput;

class SendEmails extends Command implements PromptsForMissingInput
{
    protected $signature = 'mail:send {user}';
    // ...
}
```

Now `php artisan mail:send` (no argument) asks for `user` rather than failing. To customize the wording, return questions keyed by argument name:

```php
/**
 * @return array<string, string>
 */
protected function promptForMissingArgumentsUsing(): array
{
    return [
        'user' => 'Which user ID should receive the mail?',
    ];
}
```

This respects `--no-interaction`: in automation a missing required argument still produces the normal error instead of hanging.

---

## 10. Calling other commands & scheduling

### 10.1 Calling commands

From inside a command (or anywhere in the app) you can invoke another command:

```php
use Illuminate\Support\Facades\Artisan;

public function handle(): int
{
    // Call and capture exit code; arguments/options passed as an array:
    $exit = $this->call('db:seed', ['--class' => 'UserSeeder', '--force' => true]);

    // Same, but suppress the called command's output:
    $this->callSilently('cache:clear');

    // From outside a command (e.g. a controller or job):
    Artisan::call('mail:send', ['user' => 1, '--queue' => 'high']);

    return Command::SUCCESS;
}
```

> Note that switch options are passed as `'--force' => true`, and value options as `'--class' => 'UserSeeder'`. Positional arguments use their declared name as the key.

### 10.2 Scheduling a command

Laravel's scheduler lets you define recurring tasks **in code** instead of editing crontabs. In Laravel 11/12 you schedule in `routes/console.php` (or a dedicated `bootstrap/app.php` callback); in Laravel 10 it lived in `app/Console/Kernel.php`'s `schedule()` method.

```php
// routes/console.php  (Laravel 11/12)
use Illuminate\Support\Facades\Schedule;

Schedule::command('app:send-newsletter')
    ->dailyAt('07:00')
    ->timezone('America/New_York')
    ->withoutOverlapping()       // skip if the previous run is still going
    ->onOneServer()              // run on only one server in a multi-server farm
    ->runInBackground()
    ->emailOutputOnFailure('ops@example.com');

Schedule::command('queue:prune-batches')->daily();
Schedule::command('telescope:prune')->daily();
```

```php
// app/Console/Kernel.php  (Laravel 10)
protected function schedule(Schedule $schedule): void
{
    $schedule->command('app:send-newsletter')->dailyAt('07:00');
}
```

You still need exactly **one** real cron entry on the server that ticks every minute:

```bash
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1
```

That single line calls `schedule:run` each minute; Laravel then decides which defined tasks are due. Frequency helpers include `->everyMinute()`, `->everyFiveMinutes()`, `->hourly()`, `->daily()`, `->weeklyOn(1, '8:00')`, `->monthly()`, and raw `->cron('* * * * *')`.

---

## 11. Auto-discovery, isolatable commands, and exit codes

### 11.1 Auto-discovery

You don't manually register commands in modern Laravel. Any class extending `Illuminate\Console\Command` placed in `app/Console/Commands` is **auto-discovered** and registered. The discovery is wired in `app/Console/Kernel.php` (Laravel 10) via `$this->load(__DIR__.'/Commands')`, or handled automatically by the framework's default kernel in Laravel 11/12. To register a command living elsewhere, add it to the `withCommands()` call in `bootstrap/app.php` (L11/12) or the `$commands`/`commands()` list (L10).

You can also define a tiny command inline as a closure in `routes/console.php` using `Artisan::command()`. The closure is bound to the command instance (so `$this->info()` etc. work) and receives the declared arguments/options as parameters; you may also type-hint container dependencies:

```php
// routes/console.php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Foundation\Inspiring;

Artisan::command('inspire', function () {
    $this->comment(Inspiring::quote());
})->purpose('Display an inspiring quote');

// With input and an injected dependency:
Artisan::command('mail:send {user} {--queue=}', function (\App\Support\DripEmailer $drip, string $user) {
    $this->info("Sending to {$user} on queue: " . ($this->option('queue') ?? 'sync'));
})->purpose('Send a marketing email');
```

### 11.2 Isolatable commands

When a scheduled command could be triggered more than once at the same instant (e.g. two servers, or a slow run overlapping the next tick), you risk running it concurrently. Implement the `Isolatable` interface and Laravel automatically adds an `--isolated` option (you do *not* declare it in the signature). When invoked with `--isolated`, Laravel acquires an atomic cache lock so only one instance runs at a time.

```php
use Illuminate\Console\Command;
use Illuminate\Contracts\Console\Isolatable;

class SendNewsletter extends Command implements Isolatable
{
    protected $signature = 'app:send-newsletter';
    // ...
}
```

Then run with `--isolated`:

```bash
php artisan app:send-newsletter --isolated
# Optionally specify the exit code returned when the lock is already held (blocked invocation):
php artisan app:send-newsletter --isolated=12
```

> **Subtle but important:** by default, when another instance already holds the lock the blocked invocation does **not** run but still exits with a **successful** (0) status code — so cron won't treat the skip as an error. Pass a value (`--isolated=12`) to return a specific non-zero code instead.

Isolation requires an atomic-lock-capable default cache driver: `memcached`, `redis`, `dynamodb`, `database`, `file`, or `array` (and in a multi-server setup all servers must talk to the same central cache). By default the lock key is the command name; override it with an `isolatableId()` method to fold arguments into the key, and adjust expiry with `isolationLockExpiresAt()`. The lock auto-releases when the command finishes (or after one hour if the process is killed mid-run).

```php
public function isolatableId(): string
{
    return $this->argument('user');   // one lock per user, not one global lock
}
```

### 11.3 Exit codes

A command's exit code communicates success/failure to the shell (`$?`), CI, and supervisors. The convention: **0 = success, non-zero = failure**.

```php
public function handle(): int
{
    if ($this->somethingFailed()) {
        $this->error('Boom.');
        return Command::FAILURE;   // === 1
    }

    if ($this->wasMisused()) {
        return Command::INVALID;   // === 2
    }

    return Command::SUCCESS;       // === 0
}
```

Use the `Command::SUCCESS` / `Command::FAILURE` / `Command::INVALID` constants rather than magic numbers. If `handle()` returns `void`/`null`, Laravel treats it as success (`0`). You can also abort with a thrown exception (non-zero) or call `$this->fail('message')` — available since Laravel 10.17 — which immediately terminates the command and returns exit code `1` (it does this by throwing a `Symfony\Component\Console\Exception\RuntimeException` carrying your message).

```bash
php artisan app:send-newsletter
echo $?      # prints 0 on success, 1 on FAILURE
```

---

## 12. Dependency injection in commands

Because commands resolve through the service container, you can type-hint dependencies in either the constructor or `handle()` and they're auto-injected:

```php
use App\Services\NewsletterService;

class SendNewsletter extends Command
{
    protected $signature = 'app:send-newsletter {--dry-run}';
    protected $description = 'Send the weekly newsletter to subscribers';

    // Constructor property promotion (PHP 8+) injects the service:
    public function __construct(private NewsletterService $newsletter)
    {
        parent::__construct();   // REQUIRED — sets up the signature parsing
    }

    public function handle(): int
    {
        if ($this->option('dry-run')) {
            $this->warn('Dry run — nothing will be sent.');
        }

        $count = $this->newsletter->send(dryRun: (bool) $this->option('dry-run'));
        $this->info("Queued {$count} newsletters.");

        return Command::SUCCESS;
    }
}
```

> **Gotcha:** if you write a constructor, you **must** call `parent::__construct()`. The parent constructor parses `$signature` into Symfony's input definition; skip it and you'll get errors like "The command defined in ... cannot have an empty name" or null arguments.

---

## 13. Testing commands

Commands are testable like any other code. The `$this->artisan()` test helper runs a command and returns a fluent assertion object.

> **Note:** A fresh Laravel 11/12 app ships with **Pest** as the default test runner, and `php artisan make:test` generates a Pest test unless you pass `--phpunit`. Both styles call the same `$this->artisan(...)` helper and assertions; the only difference is the surrounding test structure (`test('...', function () { ... })` for Pest vs a `test_*` method in a class for PHPUnit). Both are shown below.

**Pest (the Laravel 11/12 default):**

```php
<?php

use App\Models\User;
use Illuminate\Console\Command;
use function Pest\Laravel\artisan;

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('it sends and reports the count', function () {
    User::factory()->count(3)->create();

    artisan('app:send-newsletter')
        ->expectsOutput('Queued 3 newsletters.')
        ->doesntExpectOutput('Dry run — nothing will be sent.')
        ->assertExitCode(Command::SUCCESS);          // or ->assertSuccessful()
});

test('it handles interactive prompts', function () {
    artisan('app:setup')
        ->expectsQuestion('What is your name?', 'Ada')   // answer an ask() / Prompts text()
        ->expectsQuestion('Which role?', 'Admin')        // answer a choice() / select()
        ->expectsConfirmation('Do you wish to continue?', 'yes')
        ->expectsOutput('Welcome, Ada!')
        ->assertSuccessful();
});
```

**PHPUnit (still fully supported):**

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Tests\TestCase;
use Illuminate\Console\Command;
use Illuminate\Foundation\Testing\RefreshDatabase;

class SendNewsletterTest extends TestCase
{
    use RefreshDatabase;

    public function test_it_sends_and_reports_count(): void
    {
        User::factory()->count(3)->create();

        $this->artisan('app:send-newsletter')
            ->expectsOutput('Queued 3 newsletters.')
            ->doesntExpectOutput('Dry run — nothing will be sent.')
            ->assertExitCode(Command::SUCCESS);  // or ->assertSuccessful()
    }

    public function test_failure_path(): void
    {
        $this->artisan('app:send-newsletter')
            ->assertFailed();                    // non-zero exit code
    }
}
```

Key assertion methods:

| Method | Purpose |
|--------|---------|
| `expectsQuestion($q, $answer)` | Provide the answer to a prompt (`ask`/`choice`/`anticipate`/Prompts `text`/`select`/`multiselect`) |
| `expectsConfirmation($q, 'yes'\|'no')` | Answer a `confirm()` (answer is the string `'yes'` or `'no'`) |
| `expectsSearch($q, search:, answers:, answer:)` | Answer a Prompts `search()`/`multisearch()` |
| `expectsOutput($text)` | Assert a line of output appeared |
| `doesntExpectOutput($text)` | Assert it did NOT appear (no arg = assert *no* output at all) |
| `expectsOutputToContain($substr)` | Partial (substring) match |
| `doesntExpectOutputToContain($substr)` | Assert a substring did NOT appear |
| `expectsTable($headers, $rows)` | Assert a rendered table |
| `assertExitCode($n)` | Assert the exact exit code |
| `assertNotExitCode($n)` | Assert it did NOT exit with this code |
| `assertSuccessful()` | Exit code 0 |
| `assertFailed()` | Non-zero exit code |

You can also pass arguments/options to `artisan()`:

```php
$this->artisan('app:send-newsletter', ['--dry-run' => true])
    ->expectsOutput('Dry run — nothing will be sent.')
    ->assertExitCode(0);
```

---

## ⚠️ Common Mistakes & Gotchas

1. **Calling `env()` after `config:cache`.**
   Once config is cached, `.env` is no longer read at runtime, so `env('SOME_KEY')` returns `null` everywhere except inside `config/*.php` files (which were read at cache time). **Fix:** never call `env()` in app/command code — pull values from `config('...')`. Put `env()` calls only in config files.

2. **Forgetting `parent::__construct()` in a command's constructor.**
   If you add a constructor for dependency injection and don't call the parent, the signature never gets parsed. **Fix:** always call `parent::__construct();` first thing. (Better yet, type-hint dependencies in `handle()` instead of the constructor to sidestep it.)

3. **Treating CLI input as typed.**
   `$this->argument('count')` is the string `"5"`, not the integer `5`; a switch option is `false` when absent, not `null`. Comparing `=== 5` silently fails. **Fix:** cast explicitly — `(int) $this->argument('count')`, `(bool) $this->option('queue')`.

4. **A command hangs forever in cron/CI because it's waiting on a prompt.**
   Interactive prompts block. In automation there's no TTY to answer them. **Fix:** run with `--no-interaction` (and supply defaults), or guard prompts behind `if ($this->input->isInteractive())`.

5. **Not restarting queue workers after deploying code.**
   `queue:work` holds the framework in memory, so old code keeps running until the worker restarts. **Fix:** run `php artisan queue:restart` on deploy (it signals workers to exit gracefully after the current job, and your supervisor relaunches them).

6. **Running `migrate:fresh` / `migrate` in production without `--force`, or worse, running `migrate:fresh` at all.**
   Migration commands refuse to run in `production` without `--force` (a safety net), and `migrate:fresh` drops every table. **Fix:** use plain `migrate --force` for forward-only deploys; never `fresh`/`refresh` in prod.

7. **Long-running commands and memory leaks.**
   A command iterating millions of rows with `Model::all()` or `->get()` loads everything into memory. **Fix:** use `->chunk()`, `->chunkById()`, `->lazy()`, or `cursor()` to stream rows.

---

## ✅ Best Practices

- **Name commands by namespace**: `reports:nightly`, `users:prune`, `cache:warm`. Group related commands under a shared prefix so `list` reads well.
- **Always write a clear `$description`** and per-argument/option descriptions — your future self reads `php artisan help` more than the code.
- **Return explicit exit codes** with `Command::SUCCESS`/`FAILURE`/`INVALID` so CI and supervisors react correctly.
- **Make destructive commands confirmable** and add a `--force`/`--dry-run` pair. Use `$this->confirm()` (or `if ($this->input->isInteractive())`) before irreversible actions.
- **Keep `handle()` thin** — delegate real work to an injected service class. The command should parse input, call the service, and report results. This keeps logic unit-testable independent of the CLI.
- **Stream large datasets** with `chunkById`/`lazy` and show a progress bar so operators can see life.
- **Make scheduled commands `withoutOverlapping()` and/or `Isolatable`** to prevent concurrent runs.
- **Use Laravel Prompts** (`text`, `select`, `confirm`, …) for new interactive commands — you get validation, defaults, and graceful fallback for free.
- **Test every command** with `->expectsOutput()`, `->expectsQuestion()`, and `->assertExitCode()` — they're as testable as controllers.
- **Run `php artisan about`** when debugging environment/driver issues; it answers "is config cached?" instantly.

---

## 🎯 Interview Tips & Likely Questions

**Q1. What is Artisan and what is it built on?**
A: Artisan is Laravel's CLI, exposed via the `artisan` script in the project root. It's a wrapper around the **Symfony Console** component, which is why concepts like commands, arguments/options, input/output, and exit codes mirror Symfony. Each command runs inside a fully booted Laravel app, with access to the container, Eloquent, config, and queues.

**Q2. How do you create a custom command and what are the required pieces?**
A: `php artisan make:command Name` scaffolds a class extending `Illuminate\Console\Command`. The three essentials are `$signature` (name + arguments + options DSL), `$description`, and the `handle()` method. The class is auto-discovered from `app/Console/Commands`, so no manual registration is needed.

**Q3. Explain the signature DSL: required vs optional arguments, and the option forms.**
A: Arguments go in `{}`: `{user}` required, `{user?}` optional, `{user=guest}` optional-with-default, `{user*}` variadic array. Options go in `{--...}`: `{--queue}` is a boolean switch (true if present), `{--queue=}` requires a value (default null), `{--queue=default}` sets a default, `{--id=*}` accepts multiple values, and `{--Q|queue}` adds a `-Q` shortcut. Inline help comes after a colon: `{user : The user ID}`.

**Q4. How does Laravel know about your command without you registering it? (Under the hood)**
A: **Auto-discovery.** The console kernel scans `app/Console/Commands` (via `$this->load()` in L10, or the framework's default kernel/`withCommands()` in L11/12), instantiates each class extending `Command`, and registers it with the Symfony `Application`. At runtime, Symfony parses the `$signature` into an `InputDefinition`, matches the CLI args against it, binds them, then calls your `handle()` through the container (which is why DI works). The return value of `handle()` becomes the process exit code.

**Q5. What's the difference between `queue:work` and `queue:listen`, and why must you restart workers after deploys?**
A: `queue:work` boots the framework once and keeps it resident, processing many jobs in one long-lived process — fast, but it holds *old* code in memory after a deploy, so you must `php artisan queue:restart`. `queue:listen` reboots the framework for each job — always fresh code but slower; it's a development convenience.

**Q6. What happens when you run `config:cache`, and what's the classic bug it introduces?**
A: It merges every `config/*.php` file into a single cached PHP array (`bootstrap/cache/config.php`) so the framework skips reading individual config files and the `.env` on boot. The classic bug: any `env()` call *outside* config files now returns `null` because `.env` is no longer loaded at runtime. Fix by only using `config()` in app code.

**Q7. How do you prevent a scheduled command from running twice simultaneously?**
A: Two layers. `->withoutOverlapping()` in the schedule definition skips a run if the previous one is still active (single server). Implementing the `Isolatable` interface (run with `--isolated`) acquires an atomic cache lock so only one instance runs across multiple servers. Combine `->onOneServer()` for multi-server scheduling too.

**Q8. How do you signal success or failure from a command, and how do you test it?**
A: Return an exit code from `handle()` — `Command::SUCCESS` (0), `FAILURE` (1), or `INVALID` (2); void is treated as 0. In tests, use `$this->artisan('name')->assertExitCode(0)` / `->assertSuccessful()` / `->assertFailed()`. For interactive commands, drive prompts with `->expectsQuestion()` and `->expectsConfirmation()`, and assert output with `->expectsOutput()`.

**Q9. How do you call one command from another, and pass it arguments?**
A: `$this->call('db:seed', ['--class' => 'UserSeeder', '--force' => true])` returns the exit code; `$this->callSilently(...)` suppresses output. Outside a command, use the `Artisan::call(...)` facade. Switch options are passed as `'--force' => true`; value options and arguments use their name as the array key.

**Q10. Classic prompt helpers vs Laravel Prompts — when and why?**
A: The classic helpers (`ask`, `secret`, `confirm`, `choice`, `anticipate`) are simple and always available. **Laravel Prompts** (bundled since 10.17) provides validated, accessible, nicely styled prompts (`text`, `password`, `select`, `multiselect`, `search`, `spin`) with `required:` and `validate:` support, and they gracefully fall back on unsupported terminals. Prefer Prompts for new code.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Discovery
php artisan list                       # all commands
php artisan help <cmd>                  # detailed help
php artisan <cmd> --help

# Everyday
php artisan migrate [--force] [--pretend] [--seed]
php artisan migrate:fresh --seed        # DEV ONLY (drops tables)
php artisan db:seed [--class=Seeder]
php artisan tinker [--execute="..."]
php artisan route:list [--path=api] [--method=GET]
php artisan about [--only=environment]
php artisan key:generate
php artisan storage:link

# Performance (production deploy)
php artisan optimize                    # cache config+route+view+event
php artisan optimize:clear              # clear everything
php artisan config:cache | config:clear
php artisan route:cache  | route:clear

# Queues & schedule
php artisan queue:work --tries=3 --timeout=90
php artisan queue:restart               # after deploy
php artisan schedule:run                # cron calls this every minute
php artisan schedule:work               # foreground (dev)

# Scaffolding
php artisan make:command Name
php artisan make:model Post -mfsc
php artisan make:controller PostController --resource
```

```php
// Signature DSL
'mail:send {user}'                 // required arg
'mail:send {user?}'                // optional arg
'mail:send {user=guest}'           // optional + default
'mail:send {user*}'                // variadic (array)
'mail:send {--queue}'              // boolean switch
'mail:send {--queue=}'             // value option
'mail:send {--queue=default}'      // value + default
'mail:send {--id=*}'               // multi-value option
'mail:send {--Q|queue}'            // switch with -Q shortcut
'mail:send {--Q|queue=}'           // value option, called as -Qhigh
'mail:send {user : description}'   // inline help

// Reading input
$this->argument('user');  $this->arguments();
$this->option('queue');   $this->options();

// Output
$this->info(); $this->error(); $this->warn();
$this->line(); $this->comment(); $this->newLine();
$this->table($headers, $rows);
$this->withProgressBar($items, fn ($i) => ...);

// Prompts (classic)
$this->ask(); $this->secret(); $this->confirm();
$this->choice(); $this->anticipate();

// Prompts (modern) — use function Laravel\Prompts\{text,select,confirm,...}
text(label: '...', required: true, validate: fn ($v) => ...);

// Calling commands
$this->call('db:seed', ['--class' => 'UserSeeder']);
$this->callSilently('cache:clear');
Artisan::call('mail:send', ['user' => 1]);

// Exit codes
return Command::SUCCESS;   // 0
return Command::FAILURE;   // 1
return Command::INVALID;   // 2

// Schedule (routes/console.php, L11/12)
Schedule::command('app:send-newsletter')
    ->dailyAt('07:00')->withoutOverlapping()->onOneServer();
```

```php
// Testing
$this->artisan('app:send-newsletter', ['--dry-run' => true])
    ->expectsQuestion('What is your name?', 'Ada')
    ->expectsConfirmation('Continue?', 'yes')
    ->expectsOutput('Queued 3 newsletters.')
    ->assertExitCode(0);   // or ->assertSuccessful() / ->assertFailed()
```

---

## 🧪 Mini Exercises

1. **Build `users:prune`.** Create a command `php artisan make:command PruneUsers` with signature `users:prune {--days=30} {--dry-run}`. It should find users who haven't logged in for `--days` days and delete them — unless `--dry-run` is set, in which case it only prints how many *would* be deleted. Use `chunkById()` to stream, show a progress bar, and return the correct exit code.

2. **Add interactive safety.** Extend the command above so that, when run interactively without `--dry-run`, it asks `confirm('Permanently delete N users?')` first. When run with `--no-interaction`, it should proceed only if `--force` is also passed; otherwise abort with `Command::INVALID`.

3. **Schedule it.** Register `users:prune` in `routes/console.php` to run every Sunday at 02:00 in your app's timezone, `withoutOverlapping()`, and email output to an ops address on failure. Add the single required system crontab line as a comment.

4. **Make it isolatable.** Implement the `Isolatable` interface on the command and explain (in a comment) which cache drivers support the lock and what `--isolated=12` does.

5. **Test it.** Write a feature test using `RefreshDatabase` that seeds a mix of recently-active and stale users, runs `php artisan users:prune --dry-run`, asserts the output reports the right count, asserts no users were actually deleted, and `assertExitCode(0)`. Add a second test that answers the confirmation prompt with `expectsConfirmation(..., 'no')` and asserts nothing was deleted.
```
