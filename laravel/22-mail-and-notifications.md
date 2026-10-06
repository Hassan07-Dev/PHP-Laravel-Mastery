# Mail & Notifications in Laravel 12

Sending email and pushing alerts to users is one of the most common "real-world" tasks a backend engineer ships. Laravel gives you two related-but-distinct tools for this:

- **Mailables** — a class that represents *one specific email* (a welcome email, a receipt, a password reset). It is email-only.
- **Notifications** — a class that represents *a thing the user should be told*, which can be delivered over **many channels at once** (email, a row in the database, SMS, Slack, a broadcast/websocket push).

This module teaches both, when to reach for each, and the production concerns (queueing, testing, previewing) that interviewers love to probe.

---

**What you'll learn**

- How Laravel's mail config and "mailers" work (`smtp`, `mailgun`, `ses`, `log`, `array`) and why `log`/`array` exist
- Building Mailable classes: `envelope()`, `content()`, `attachments()`, `headers()`, and passing data
- Markdown mailables and reusable Blade mail components
- Previewing email in the browser without sending anything
- Sending, queueing (`ShouldQueue`), and faking mail in tests (`Mail::fake`)
- Building Notifications: `via()`, `toMail`/`toArray`/`toDatabase`, the `Notifiable` trait, channels, and on-demand notifications
- Database notifications: the table, reading them, and marking as read; plus `Notification::fake`
- A clear decision rule for **mail vs notification**

---

## 1. Why two systems? Mail vs Notification

A **Mailable** answers: "What does this email look like?" It knows about a subject, a from address, a view, and attachments. It does not know or care about SMS or websockets.

A **Notification** answers: "How should we tell this user that *event X* happened?" The same notification (e.g. `InvoicePaid`) might:

- email the customer,
- store a record in the `notifications` table so it appears in an in-app bell icon,
- ping a Slack channel for the finance team.

Rule of thumb:

> If the message is **email-only and content-heavy** (marketing email, a detailed receipt with a PDF attachment), reach for a **Mailable**.
> If the message is an **event** that should reach the user through **one or more channels** (and especially if you want an in-app inbox), reach for a **Notification**.

Crucially, notifications *use* mailables under the hood for their mail channel — so these are layers, not competitors. You'll learn mail first because notifications build on it.

---

## 2. Mail configuration & mailers

### The config file

Mail settings live in `config/mail.php`, driven by your `.env`. A **mailer** is a named transport configuration. Laravel ships with several drivers (transports): `smtp`, `mailgun`, `ses`, `postmark`, `resend`, `sendmail`, `log`, `array`, and `failover`/`roundrobin`.

```env
# .env  (keys from a fresh Laravel 12 .env.example)
MAIL_MAILER=log

MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=postmaster@example.com
MAIL_PASSWORD=secret
MAIL_SCHEME=null   # 'smtp' (STARTTLS) or 'smtps' (implicit TLS); null lets Symfony auto-negotiate

MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

> **Version note:** Laravel 9+ switched the SMTP mailer to a `scheme` option, and a fresh Laravel 11/12 `.env.example` ships `MAIL_SCHEME` (not `MAIL_ENCRYPTION`). The old `MAIL_ENCRYPTION=tls`/`ssl` keys still work for backwards compatibility, but new projects should prefer `MAIL_SCHEME`. Never commit real `MAIL_PASSWORD`/provider secrets — keep them in `.env`, which is git-ignored.

`MAIL_MAILER` chooses the *default* mailer. The drivers worth knowing:

| Driver  | What it does | When to use |
|---------|--------------|-------------|
| `smtp`  | Talks SMTP to any mail server | Generic; works with Mailgun/SES/Gmail SMTP endpoints |
| `mailgun` | Mailgun HTTP API | High volume, Mailgun account |
| `ses`   | Amazon SES API | AWS-hosted apps |
| `log`   | **Writes the email to `storage/logs/laravel.log` instead of sending** | Local dev — see the email without a mail server |
| `array` | **Keeps emails in an in-memory array; sends nothing** | Tests (this is what `Mail::fake` builds on conceptually) |
| `failover` | Tries a list of mailers in order until one works | Production resilience |

```php
// config/mail.php (excerpt)
'default' => env('MAIL_MAILER', 'log'),

'mailers' => [
    'smtp' => [
        'transport' => 'smtp',
        'host' => env('MAIL_HOST', '127.0.0.1'),
        'port' => env('MAIL_PORT', 587),
        // ...
    ],

    'failover' => [
        'transport' => 'failover',
        'mailers' => ['smtp', 'log'], // try smtp, fall back to log
    ],
],
```

> **Tip:** In local dev, set `MAIL_MAILER=log` and tail `storage/logs/laravel.log` to read your emails. Even better, use a tool like **Mailpit** (set `MAIL_MAILER=smtp`, `MAIL_HOST=127.0.0.1`, `MAIL_PORT=1025`) which catches all outgoing mail in a web UI.

### Mailgun/SES/Postmark/Resend extra packages

`smtp`, `sendmail`, `log`, and `array` work out of the box. The API-based drivers (`mailgun`, `postmark`, `resend`, `ses`) need their respective SDKs via Composer — Laravel wraps Symfony Mailer's transports under the hood:

```bash
composer require symfony/mailgun-mailer symfony/http-client    # Mailgun
composer require symfony/postmark-mailer symfony/http-client   # Postmark
composer require resend/resend-php                             # Resend
composer require aws/aws-sdk-php                                # SES (v2 / SESv2)
```

You also configure these drivers' credentials in **`config/services.php`** (driven by `.env`), not just `config/mail.php`. For example:

```php
// config/services.php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'), // 'api.eu.mailgun.net' for the EU region
],

'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

> Laravel's API drivers (Mailgun/Postmark/Resend) are usually **simpler and faster** than SMTP — Laravel recommends them over SMTP when your provider offers one.

---

## 3. Your first Mailable

Generate one with Artisan:

```bash
php artisan make:mail OrderShipped
```

This creates `app/Mail/OrderShipped.php`. A modern (Laravel 9+/12) Mailable has three core methods: `envelope()`, `content()`, and `attachments()`.

```php
<?php

namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable
{
    use Queueable, SerializesModels;

    // Constructor promotion: $order is automatically a public property,
    // and public properties are auto-shared with the view.
    public function __construct(public Order $order) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: "Your order #{$this->order->id} has shipped!",
        );
    }

    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
        );
    }

    /**
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [];
    }
}
```

### The Envelope: from, subject, cc, replyTo, tags

The `Envelope` object describes the "headers" of the message.

```php
use Illuminate\Mail\Mailables\Address;

public function envelope(): Envelope
{
    return new Envelope(
        from: new Address('orders@example.com', 'Acme Orders'),
        subject: "Your order #{$this->order->id} has shipped!",
        cc: ['warehouse@example.com'],
        replyTo: [new Address('support@example.com', 'Acme Support')],
        tags: ['shipment'],                 // Mailgun/SES tagging
        metadata: ['order_id' => $this->order->id],
        using: [
            // raw access to the Symfony Message before sending
            function (\Symfony\Component\Mime\Email $message) {
                $message->getHeaders()->addTextHeader('X-Custom', 'value');
            },
        ],
    );
}
```

If you omit `from`, Laravel uses `MAIL_FROM_ADDRESS`/`MAIL_FROM_NAME` from config. Set a global from in `config/mail.php` once instead of per-mailable.

### The Content: views, plain text, and passing data

```php
public function content(): Content
{
    return new Content(
        view: 'mail.orders.shipped',      // HTML view
        text: 'mail.orders.shipped-text', // optional plain-text alternative
        with: [                            // explicit data (alternative to public props)
            'orderName'  => $this->order->name,
            'orderPrice' => $this->order->price,
        ],
    );
}
```

There are **two ways to pass data** to the view:

1. **Public properties** — any `public` property on the Mailable is auto-available in the view. (This is why constructor promotion with `public` is so handy.)
2. **The `with` array** — explicit, and useful when you want to expose computed/renamed values without making them public properties. If you use `with`, only those keys (plus public props) are available.

The Blade view:

```blade
{{-- resources/views/mail/orders/shipped.blade.php --}}
<h1>Order #{{ $order->id }} shipped</h1>

<p>Hi {{ $order->customer_name }}, your order is on its way.</p>

<p>Total: ${{ number_format($order->price, 2) }}</p>
```

### Attachments

Return an array of `Attachment` objects from `attachments()`. They can come from a path, the storage disk, or raw in-memory data.

```php
use Illuminate\Mail\Mailables\Attachment;

public function attachments(): array
{
    return [
        // From a local path
        Attachment::fromPath('/path/to/invoice.pdf')
            ->as('invoice.pdf')
            ->withMime('application/pdf'),

        // From a configured storage disk
        Attachment::fromStorageDisk('s3', "invoices/{$this->order->id}.pdf"),

        // Raw data generated on the fly (e.g. a PDF in memory)
        Attachment::fromData(fn () => $this->buildPdf(), 'receipt.pdf')
            ->withMime('application/pdf'),
    ];
}
```

### Custom headers

For things like `Message-ID`, `In-Reply-To`, or custom `X-` headers, add a `headers()` method:

```php
use Illuminate\Mail\Mailables\Headers;

public function headers(): Headers
{
    return new Headers(
        messageId: 'order-'.$this->order->id.'@example.com',
        references: ['previous-message@example.com'],
        text: [
            'X-Order-Id' => (string) $this->order->id,
        ],
    );
}
```

---

## 4. Markdown mailables & components

Hand-coding responsive, email-client-safe HTML is miserable. **Markdown mailables** give you a pre-styled, mobile-friendly template using Blade mail components (buttons, panels, tables) plus inline-CSS rendering automatically.

Generate one with the `--markdown` flag:

```bash
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

Now `content()` uses `markdown:` instead of `view:`:

```php
public function content(): Content
{
    return new Content(
        markdown: 'mail.orders.shipped',
        with: ['url' => route('orders.show', $this->order)],
    );
}
```

The Markdown view mixes Markdown with Blade components:

```blade
{{-- resources/views/mail/orders/shipped.blade.php --}}
<x-mail::message>
# Order Shipped

Your order **#{{ $order->id }}** is on its way.

<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>

<x-mail::panel>
Estimated delivery: {{ $order->eta->format('M j, Y') }}
</x-mail::panel>

<x-mail::table>
| Item        | Qty | Price  |
|:------------|:---:|-------:|
| Widget      |  2  | $20.00 |
| Shipping    |  -  |  $5.00 |
</x-mail::table>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

Available components: `<x-mail::message>`, `<x-mail::button>` (colors: `primary`, `success`, `error`), `<x-mail::panel>`, `<x-mail::table>`, and `<x-mail::subcopy>`.

To customize the look, publish the components and theme:

```bash
php artisan vendor:publish --tag=laravel-mail
```

This drops the components into `resources/views/vendor/mail/` and a CSS theme you can edit. You can also set a custom theme per mailable: `return (new Content(...))` won't do it — instead set `public $theme = 'invoice';` on the Mailable, which maps to `resources/views/vendor/mail/html/themes/invoice.css`.

---

## 5. Previewing mail in the browser

You do **not** need to send an email to see it. A Mailable is `Renderable` and `Responsable`, so you can return it directly from a route and Laravel renders the HTML:

```php
// routes/web.php  (wrap in an env check so it never ships to prod)
use App\Mail\OrderShipped;
use App\Models\Order;

if (app()->environment('local')) {
    Route::get('/mailable', function () {
        $order = Order::factory()->make(); // no DB write needed
        return new OrderShipped($order);
    });
}
```

Visit `/mailable` in the browser and you'll see the rendered email, attachments excluded. This is the fastest design loop for tweaking Markdown templates.

You can also render to a string in tests/tinker: `(new OrderShipped($order))->render()`.

---

## 6. Sending mail

Use the `Mail` facade. `to()` accepts an email string, an `Address`, a model with an `email` attribute, or a collection of models.

```php
use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;

// Single recipient
Mail::to($request->user())->send(new OrderShipped($order));

// Multiple recipients + cc/bcc, chained
Mail::to($order->customer)
    ->cc($admins)
    ->bcc('audit@example.com')
    ->send(new OrderShipped($order));

// Choose a non-default mailer explicitly
Mail::mailer('ses')->to($user)->send(new OrderShipped($order));
```

`send()` returns a `SentMessage` (or `null` if queued). For a quick one-off without a Mailable class, `Mail::raw()` exists:

```php
Mail::raw('Server CPU at 95%', function ($message) {
    $message->to('ops@example.com')->subject('Alert');
});
```

---

## 7. Queueing mailables (the production default)

Sending email is slow (network round-trips to the mail server). Doing it during a web request makes the user wait. The fix: **push it onto a queue** so a background worker sends it.

### Option A: `ShouldQueue` (always queued)

Implement `ShouldQueue` on the Mailable. Now **any** `Mail::to(...)->send(...)` for this class is automatically queued — you don't change call sites.

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;
    // ...
}
```

`SerializesModels` is what makes this safe: instead of serializing the entire `Order` model into the queue payload, it stores only the **primary key** and re-fetches the model fresh when the job runs. This avoids stale data and huge payloads. (Caveat: the model must still exist when the job runs.)

### Option B: queue at the call site

If a Mailable is *usually* sync but you want to queue it in one place:

```php
Mail::to($user)->queue(new OrderShipped($order));

// Delay 10 minutes
Mail::to($user)->later(now()->addMinutes(10), new OrderShipped($order));
```

### Customizing the queue/connection

```php
Mail::to($user)->send(
    (new OrderShipped($order))
        ->onConnection('redis')
        ->onQueue('emails')
);
```

> **Reminder:** queued mail only actually sends when a worker is running: `php artisan queue:work`. With `QUEUE_CONNECTION=sync` (the default in a fresh app), "queued" jobs run immediately/inline.

---

## 8. Testing mail with `Mail::fake`

In tests you don't want to hit a real mail server. `Mail::fake()` swaps the mailer for a spy that records what *would* have been sent, then lets you assert on it.

```php
use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;

test('it emails the customer when an order ships', function () {
    Mail::fake();

    // ... action that triggers the email
    $this->post("/orders/{$order->id}/ship");

    Mail::assertSent(OrderShipped::class);

    // Assert recipient + payload
    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($order) {
        return $mail->hasTo($order->customer->email)
            && $mail->order->is($order);
    });

    Mail::assertSent(OrderShipped::class, 1);     // count
    Mail::assertNotSent(SpamMail::class);
    Mail::assertNothingSent();                    // (would fail here)
});
```

Key assertions:

- `assertSent` / `assertNotSent` / `assertNothingSent`
- `assertQueued` / `assertNotQueued` — **important:** if your Mailable implements `ShouldQueue`, use `assertQueued`, **not** `assertSent`. They are different buckets.
- Helpers inside the closure: `hasTo`, `hasCc`, `hasBcc`, `hasSubject`.

---

## 9. Notifications

Now the second system. Generate one:

```bash
php artisan make:notification InvoicePaid
```

This creates `app/Notifications/InvoicePaid.php`.

```php
<?php

namespace App\Notifications;

use App\Models\Invoice;
use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    use Queueable;

    public function __construct(public Invoice $invoice) {}

    // Which channels should this notification go out on?
    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Invoice Paid')
            ->greeting("Hi {$notifiable->name},")
            ->line("Your invoice #{$this->invoice->id} was paid.")
            ->action('View Invoice', route('invoices.show', $this->invoice))
            ->line('Thank you for your business!');
    }

    public function toArray(object $notifiable): array
    {
        return [
            'invoice_id' => $this->invoice->id,
            'amount'     => $this->invoice->amount,
        ];
    }
}
```

### `via()` — the channel router

`via()` returns the list of channels for *this* notification, and it can be **dynamic per recipient**:

```php
public function via(object $notifiable): array
{
    return $notifiable->prefers_sms ? ['vonage', 'database'] : ['mail', 'database'];
}
```

Channels and the `to*` method each one invokes:

| Channel     | Method called          | Returns | Ships in core? |
|-------------|------------------------|---------|----------------|
| `mail`      | `toMail()`             | `MailMessage` (or a Mailable) | ✅ built-in |
| `database`  | `toDatabase()` or `toArray()` | `array` | ✅ built-in |
| `broadcast` | `toBroadcast()` or `toArray()` | `BroadcastMessage`/array | ✅ built-in |
| `vonage`    | `toVonage()`           | `VonageMessage` | ❌ separate package |
| `slack`     | `toSlack()`            | `SlackMessage` | ❌ separate package |

> `toDatabase()` takes precedence over `toArray()` for the database channel; `toArray()` is the shared fallback used by `database` and `broadcast`.

> ⚠️ **`vonage` and `slack` are NOT built into the framework.** Only `mail`, `database`, and `broadcast` ship in `laravel/framework`. SMS and Slack require first-party add-on packages and channel-specific config (Vonage credentials, a Slack app/bot token), so you must install them before listing those channels in `via()`:
>
> ```bash
> composer require laravel/vonage-notification-channel guzzlehttp/guzzle   # SMS (Vonage, formerly Nexmo)
> composer require laravel/slack-notification-channel                       # Slack
> ```
>
> Listing `'vonage'` or `'slack'` in `via()` without the package installed throws an `InvalidArgumentException: Unable to locate a notification channel...` at send time.

### `toMail` returning a Mailable

`toMail` usually returns a `MailMessage` (the fluent builder), but it can return a full **Mailable** when you need attachments or custom markdown:

```php
use App\Mail\InvoicePaidMail;

public function toMail(object $notifiable): InvoicePaidMail
{
    return (new InvoicePaidMail($this->invoice))
        ->to($notifiable->email);
}
```

This is the bridge between the two systems mentioned in section 1.

### `toDatabase`, `toVonage`, `toSlack`

Each non-mail channel has its own builder method. `toDatabase()` (or `toArray()`) returns the array stored as JSON in the `notifications` table:

```php
public function toDatabase(object $notifiable): array
{
    return [
        'invoice_id'      => $this->invoice->id,
        'amount'          => $this->invoice->amount,
        'tracking_number' => $this->invoice->tracking_number,
    ];
}
```

SMS via the Vonage channel (requires `laravel/vonage-notification-channel`):

```php
use Illuminate\Notifications\Messages\VonageMessage;

public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content("Invoice #{$this->invoice->id} was paid. Thanks!");
}
```

Slack via the Slack channel (requires `laravel/slack-notification-channel`). Laravel 11/12 uses Slack's Block Kit builder:

```php
use Illuminate\Notifications\Slack\SlackMessage;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;

public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text("Invoice #{$this->invoice->id} was paid.")
        ->headerBlock('Invoice Paid')
        ->sectionBlock(function (SectionBlock $block) {
            $block->text("Amount: \${$this->invoice->amount}");
        });
}
```

---

## 10. The `Notifiable` trait & sending notifications

To *receive* notifications, a model uses the `Notifiable` trait. `App\Models\User` has it by default.

```php
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
}
```

Two ways to send:

```php
use App\Notifications\InvoicePaid;
use Illuminate\Support\Facades\Notification;

// 1. Via the notifiable (uses the trait)
$user->notify(new InvoicePaid($invoice));

// 2. Via the facade — good for many recipients at once
Notification::send($users, new InvoicePaid($invoice));
```

`Notification::send()` sends synchronously per recipient unless the notification is queued; `Notification::sendNow()` always sends immediately, ignoring `ShouldQueue`.

### Routing notifications: where does mail/sms go?

For the `mail` channel, Laravel uses the notifiable's `email` attribute by default. To change *which* attribute/address a channel targets, add a `routeNotificationFor*` method:

```php
class User extends Authenticatable
{
    use Notifiable;

    public function routeNotificationForMail(Notification $notification): string
    {
        return $this->billing_email; // override default `email`
    }

    public function routeNotificationForVonage(Notification $notification): string
    {
        return $this->phone_number;  // E.164 format
    }
}
```

### On-demand notifications (no model needed)

To notify someone who isn't a stored model (e.g. a one-off email/phone), use `Notification::route()`:

```php
Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '15551234567')
    ->route('slack', '#deployments')
    ->notify(new InvoicePaid($invoice));

// Named mail recipient
Notification::route('mail', ['taylor@example.com' => 'Taylor Otwell'])
    ->notify(new InvoicePaid($invoice));
```

Inside the notification, `$notifiable` will be an `AnonymousNotifiable` instance, so don't assume `$notifiable->name` exists for on-demand sends.

---

## 11. Database notifications (the in-app inbox)

The `database` channel writes a row to a `notifications` table — perfect for a bell icon / activity feed.

### Create the table

Laravel 11+ ships the migration via Artisan:

```bash
php artisan make:notifications-table
php artisan migrate
```

The table has: `id` (a **UUID** primary key, via `$table->uuid('id')->primary()`), `type` (the notification class), `notifiable_type`/`notifiable_id` (polymorphic owner, via `$table->morphs('notifiable')`), `data` (a `text` column cast to an array, populated from `toArray`/`toDatabase`), `read_at` (nullable timestamp), and timestamps.

> If your notifiable model (e.g. `User`) uses UUID/ULID primary keys instead of auto-increment integers, edit the generated migration to replace `morphs('notifiable')` with `uuidMorphs('notifiable')` (or `ulidMorphs`) so the `notifiable_id` column type matches.

### Reading notifications

The `Notifiable` trait adds relationships:

```php
// All notifications, newest first
$user->notifications;

// Only unread
$user->unreadNotifications;

// Only read
$user->readNotifications;

foreach ($user->unreadNotifications as $notification) {
    echo $notification->data['invoice_id'];
    echo $notification->type;       // App\Notifications\InvoicePaid
    echo $notification->created_at;
}
```

The `data` JSON column is auto-cast to an array. The `type` lets your UI decide how to render each item.

### Marking as read

```php
// Mark one
$notification->markAsRead();

// Mark all unread. NOTE the subtle difference:
$user->unreadNotifications->markAsRead();        // loads the collection, then issues one UPDATE per row
$user->unreadNotifications()->update(['read_at' => now()]); // single mass-UPDATE, never loads the models (most efficient)

// Helpers
$notification->markAsUnread();
$notification->read();    // bool: read_at is set?
$notification->unread();  // bool
```

A common controller pattern:

```php
public function index(Request $request)
{
    $notifications = $request->user()
        ->notifications()
        ->latest()
        ->paginate(15);

    return view('notifications.index', compact('notifications'));
}

public function markRead(Request $request, string $id)
{
    $request->user()->notifications()->findOrFail($id)->markAsRead();

    return back();
}
```

---

## 12. Queued notifications

Like mail, notifications should usually be queued. Implement `ShouldQueue` and use `Queueable`:

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Bus\Queueable;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;
    // ...
}
```

When queued, **each channel becomes its own queued job**. So a notification with `via() = ['mail', 'database']` dispatches two jobs — if the mail server is down, the database row can still succeed independently, and only the failed channel retries.

Control delay/connection/queue:

```php
// Delay all channels
$user->notify((new InvoicePaid($invoice))->delay(now()->addMinutes(5)));

// Per-channel delay — keys are the channel names returned by via()
$user->notify((new InvoicePaid($invoice))->delay([
    'mail'   => now()->addMinutes(5),
    'vonage' => now()->addMinute(),
]));
```

For per-channel control declared *on the notification class* rather than at the call site, define methods returning a channel-keyed map:

```php
// Which queue each channel uses
public function viaQueues(): array
{
    return ['mail' => 'mail-queue', 'database' => 'default'];
}

// Which queue connection each channel uses
public function viaConnections(): array
{
    return ['mail' => 'redis', 'database' => 'sync'];
}

// Per-channel delay
public function withDelay(object $notifiable): array
{
    return ['mail' => now()->addMinutes(5)];
}
```

To force immediate (synchronous) send even when the notification implements `ShouldQueue`, use `$user->notifyNow(...)` or `Notification::sendNow($users, ...)`.

---

## 13. Testing notifications with `Notification::fake`

```php
use App\Notifications\InvoicePaid;
use Illuminate\Notifications\AnonymousNotifiable;
use Illuminate\Support\Facades\Notification;

test('it notifies the user when invoice is paid', function () {
    Notification::fake();

    $this->post("/invoices/{$invoice->id}/pay");

    // Sent to a specific notifiable, on the expected channels
    Notification::assertSentTo(
        $user,
        InvoicePaid::class,
        function (InvoicePaid $notification, array $channels) use ($invoice) {
            return in_array('mail', $channels)
                && $notification->invoice->is($invoice);
        }
    );

    Notification::assertCount(1);
    Notification::assertNotSentTo($otherUser, InvoicePaid::class);

    // On-demand assertion
    Notification::assertSentOnDemand(InvoicePaid::class);
});
```

`Notification::fake()` short-circuits delivery (no mail/db rows written), so combine it with channel assertions rather than checking the `notifications` table.

---

## ⚠️ Common Mistakes & Gotchas

1. **Using `Mail::assertSent` when the Mailable implements `ShouldQueue`.**
   A queued mailable is recorded as *queued*, not *sent*. The assertion silently fails (count 0).
   **Fix:** use `Mail::assertQueued(OrderShipped::class)`. Same idea for notifications: queued notifications still use `Notification::assertSentTo` (the fake intercepts before queueing), but for *mail* the sent/queued distinction is real.

2. **Forgetting `SerializesModels` on a queued mailable/notification — or relying on it after the model is deleted.**
   Without the trait, the full model is serialized (bloated payload, stale data). *With* it, only the primary key is stored and the model is re-fetched at run time — so if the row is deleted before the worker runs, the job throws `ModelNotFoundException`.
   **Fix:** keep `SerializesModels`; for jobs that may outlive the model, pass scalar data instead, or handle the missing-model case.

3. **Expecting queued mail to send with `QUEUE_CONNECTION=sync` or no worker running.**
   In a fresh app `QUEUE_CONNECTION=sync` runs "queued" jobs inline, which masks problems. In production with `redis`/`database`, nothing sends until `php artisan queue:work` is running.
   **Fix:** run a worker (and a process supervisor like Supervisor/Horizon in prod); set the connection deliberately per environment.

4. **`toArray()` vs `toDatabase()` confusion / changing the `data` shape later.**
   `database` uses `toDatabase()` if present, otherwise `toArray()`. Stored rows freeze whatever shape you wrote. If you later remove a key from `toArray()`, old rows still have it and your Blade may break.
   **Fix:** treat the `data` array as a versioned contract; guard access in views (`$notification->data['x'] ?? null`).

5. **Putting secrets or huge HTML in `MAIL_MAILER=log` and thinking it "sent."**
   `log` and `array` drivers never send real email. Easy to forget in staging.
   **Fix:** verify `MAIL_MAILER` per environment; use Mailpit for a realistic catch-all in dev.

6. **On-demand notifications assuming `$notifiable` is a User.**
   `Notification::route(...)` passes an `AnonymousNotifiable`, which has no `name`, `email` attribute, or DB id.
   **Fix:** guard usages, e.g. `$notifiable instanceof User ? $notifiable->name : 'there'`.

7. **Listing `'vonage'` or `'slack'` in `via()` without installing the channel package.**
   Only `mail`, `database`, and `broadcast` ship in core. SMS and Slack live in `laravel/vonage-notification-channel` and `laravel/slack-notification-channel`. Sending throws `InvalidArgumentException: Unable to locate a notification channel for [vonage]`.
   **Fix:** `composer require` the channel package and add its config before referencing the channel.

8. **`$user->unreadNotifications->markAsRead()` on a huge inbox.**
   Reading the *property* loads every unread row into memory, then issues one UPDATE per row. Fine for a handful; wasteful for thousands.
   **Fix:** for a mass "mark all read", use the *relationship method* with a single query: `$user->unreadNotifications()->update(['read_at' => now()])`.

9. **Using `Mail::raw()` with unescaped user input.**
   `Mail::raw()` sends a plain-text body; if you build it from request data, treat it like any other untrusted string. For HTML mail, never concatenate raw user input into a Blade view without `{{ }}` escaping (Blade auto-escapes; `{!! !!}` does not).
   **Fix:** keep user-controlled values inside escaped `{{ }}` output, and validate/normalize addresses before passing them to `to()`.

---

## ✅ Best Practices

- **Queue all transactional mail and notifications** (`ShouldQueue`). Never block a web request on SMTP.
- **Set a global `from` address** in `config/mail.php` rather than repeating it in every Mailable's envelope.
- **Prefer Markdown mailables** for consistent, responsive, client-safe HTML; publish and theme the components for branding.
- **Use notifications when you want multi-channel delivery or an in-app inbox**; use Mailables for email-only, attachment-heavy messages.
- **Always test with `Mail::fake()` / `Notification::fake()`** instead of hitting real providers; assert recipients and payloads, not just "something sent."
- **Keep notification `toArray()` small and stable** — store IDs, render details in the UI from the live model when possible.
- **Use `routeNotificationFor*`** to centralize "which address/phone gets this channel" on the notifiable.
- **Tag/add metadata** on the envelope for deliverability analytics with Mailgun/SES.
- **Wrap mail-preview routes in an `environment('local')` guard** so they never reach production.
- **Install the channel package before using `vonage`/`slack`** and keep provider credentials in `.env`/`config/services.php` — never hard-code secrets in the notification class.
- **Prefer API mail drivers** (Mailgun/Postmark/Resend) over raw SMTP when available — they are simpler and faster.
- **Use Mailpit (or `MAIL_MAILER=log`) in local/CI**, real providers only in staging/production; verify `MAIL_MAILER` per environment.

---

## 🎯 Interview Tips & Likely Questions

**Q1. When would you choose a Notification over a Mailable?**
A Mailable represents a single email. A Notification represents an event delivered over one or more channels (mail, database, SMS, Slack, broadcast) chosen at send time per recipient via `via()`. Use Notifications for multi-channel delivery or an in-app inbox; use Mailables for email-only, attachment-heavy content. Notifications can even return a Mailable from `toMail`.

**Q2. How does `SerializesModels` work, and why does it matter for queued mail?**
When a job is queued, its payload is serialized to the queue store. `SerializesModels` overrides serialization so Eloquent models are stored as just their connection + primary key, then re-resolved (`findOrFail`) when the job runs. This keeps payloads tiny and data fresh — but means the model must still exist at run time, or you get `ModelNotFoundException`.

**Q3. What's the difference between `Mail::assertSent` and `Mail::assertQueued`?**
`assertSent` checks mailables sent synchronously; `assertQueued` checks mailables pushed to the queue. A Mailable implementing `ShouldQueue` goes into the *queued* bucket, so testing it with `assertSent` fails. Pick the assertion that matches whether the mailable queues.

**Q4. How does the `database` notification channel work under the hood?**
The channel serializes `toDatabase()` (or `toArray()`) to JSON and inserts a row into the `notifications` table — a polymorphic table keyed by `notifiable_type`/`notifiable_id` with a `read_at` column. The `Notifiable` trait exposes `notifications`, `unreadNotifications`, and `readNotifications` relationships. "Marking as read" just sets `read_at = now()`.

**Q5. What happens when a queued notification has multiple channels?**
Each channel is dispatched as a separate queued job. They run and retry independently, so a failure in the mail channel doesn't block the database channel. This is a key reliability advantage over manually sending mail in a controller.

**Q6. How do you send a notification to someone who isn't a model in your DB?**
On-demand notifications: `Notification::route('mail', 'a@b.com')->route('vonage', '155...')->notify(new X)`. Inside the notification, `$notifiable` is an `AnonymousNotifiable`, so don't rely on model attributes.

**Q7. How can you preview an email without sending it?**
Return the Mailable from a route (it's `Responsable`/`Renderable`) and Laravel renders the HTML in the browser; or call `->render()` to get the HTML string. Use `MAIL_MAILER=log` or Mailpit to inspect actual sends.

**Q8. What are the `log` and `array` mailers for?**
`log` writes emails to the log file instead of sending (local dev). `array` keeps them in memory and sends nothing (testing). They let you exercise the mail path without a real SMTP server.

**Q9. How do `toMail`, `toArray`, `toDatabase`, `toBroadcast` relate to `via()`?**
`via()` lists channels; each channel invokes its matching `to*` method to build that channel's payload. `database` prefers `toDatabase()` then falls back to `toArray()`; `broadcast` prefers `toBroadcast()` then `toArray()`. So `toArray()` often serves as the shared default.

**Q10. How would you customize which queue or connection mail/notifications use?**
For mailables: `->onConnection()`/`->onQueue()` at the call site, or properties on the class. For notifications: `->delay()`, `viaQueues()`, `viaConnections()`, or the `$connection`/`$queue` properties from the `Queueable` trait.

**Q11. Which notification channels ship with the framework, and which need a package?**
Core ships `mail`, `database`, and `broadcast`. SMS (`vonage`) and `slack` are first-party but separate packages (`laravel/vonage-notification-channel`, `laravel/slack-notification-channel`) you must `composer require` and configure before adding them to `via()` — otherwise sending throws `InvalidArgumentException: Unable to locate a notification channel`.

**Q12. What's the difference between `notify()`/`Notification::send()` and `notifyNow()`/`Notification::sendNow()`?**
`notify()`/`send()` respect `ShouldQueue` — if the notification queues, it goes to the queue; otherwise it sends synchronously. `notifyNow()`/`sendNow()` always send immediately, bypassing the queue even when `ShouldQueue` is implemented. Useful in commands/jobs where you are already off the request thread.

---

## 📋 Quick Reference / Cheat Sheet

```bash
# Generate
php artisan make:mail OrderShipped
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
php artisan make:notification InvoicePaid
php artisan make:notifications-table   # then: php artisan migrate
php artisan vendor:publish --tag=laravel-mail   # customize mail components/theme

# Add-on channels (NOT in core) — install before listing them in via()
composer require laravel/vonage-notification-channel guzzlehttp/guzzle   # SMS
composer require laravel/slack-notification-channel                       # Slack

# API mail drivers need their transport SDK
composer require symfony/mailgun-mailer symfony/http-client   # Mailgun
composer require aws/aws-sdk-php                                # SES

# Run the worker so queued mail/notifications actually send
php artisan queue:work
```

```php
// --- MAIL ---
Mail::to($user)->cc($a)->bcc($b)->send(new OrderShipped($order));
Mail::to($user)->queue(new OrderShipped($order));            // queue now
Mail::to($user)->later(now()->addMinutes(10), new OrderShipped($order));
Mail::mailer('ses')->to($user)->send(new OrderShipped($order));
Mail::raw('Body', fn ($m) => $m->to('x@y.com')->subject('Hi'));

// Mailable methods
envelope(): Envelope     // from, subject, cc, replyTo, tags, metadata
content(): Content        // view | text | markdown | with
attachments(): array      // Attachment::fromPath/fromStorageDisk/fromData
headers(): Headers        // messageId, references, custom text headers
implements ShouldQueue    // auto-queue every send

// Mail tests
Mail::fake();
Mail::assertSent(OrderShipped::class, fn ($m) => $m->hasTo('x@y.com'));
Mail::assertQueued(OrderShipped::class);   // for ShouldQueue mailables
Mail::assertNothingSent();
```

```php
// --- NOTIFICATIONS ---
$user->notify(new InvoicePaid($invoice));
$user->notifyNow(new InvoicePaid($invoice));         // ignore ShouldQueue
Notification::send($users, new InvoicePaid($invoice));
Notification::route('mail', 'a@b.com')->notify(new InvoicePaid($invoice));

// Notification methods
via($notifiable): array        // ['mail','database','broadcast','vonage','slack']
                               //   vonage + slack require add-on packages
toMail($n): MailMessage|Mailable
toArray($n): array             // db + broadcast fallback
toDatabase($n): array          // db-specific override
toBroadcast($n): BroadcastMessage
toVonage($n): VonageMessage    // add-on pkg
toSlack($n): SlackMessage      // add-on pkg

// Per-channel queue control (on the class)
viaQueues(): array        // ['mail' => 'queue-name']
viaConnections(): array   // ['mail' => 'redis']
withDelay($n): array      // ['mail' => now()->addMinutes(5)]

// MailMessage builder
(new MailMessage)->subject(..)->greeting(..)->line(..)
                 ->action('Text', $url)->line(..)->markdown('view', $data);

// Database inbox
$user->notifications; $user->unreadNotifications; $user->readNotifications;
$notification->markAsRead(); $user->unreadNotifications->markAsRead();
$notification->data; $notification->read_at; $notification->type;

// Routing override on the notifiable
public function routeNotificationForMail($n) { return $this->billing_email; }

// Notification tests
Notification::fake();
Notification::assertSentTo($user, InvoicePaid::class, fn ($n, $ch) => in_array('mail', $ch));
Notification::assertCount(1);
Notification::assertSentOnDemand(InvoicePaid::class);
```

| Mail driver | Sends? | Use |
|---|---|---|
| smtp/mailgun/ses/postmark/resend | yes | production |
| sendmail | yes | server-local |
| log | no (logs) | local dev |
| array | no (memory) | tests |
| failover/roundrobin | yes | resilience/balancing |

---

## 🧪 Mini Exercises

1. **Markdown receipt.** Create a `ReceiptMail` markdown mailable for an `Order` that includes an `<x-mail::table>` of line items, a `success`-colored button linking to the order, and a PDF attachment built from in-memory data with `Attachment::fromData`. Add a local-only route to preview it in the browser.

2. **Multi-channel notification.** Build an `OrderShipped` notification with `via()` returning `['mail', 'database']`, where `toMail` returns a `MailMessage` and `toDatabase` stores `order_id` and `tracking_number`. Make it implement `ShouldQueue`. Then write a feature test using `Notification::fake()` asserting it was sent to the customer on both channels.

3. **In-app inbox.** Add a controller with `index` (paginated notifications), `markRead` (mark one by id), and `markAllRead` (mark all unread). Render unread ones bold in a Blade view using `$notification->read()`.

4. **Dynamic channels.** First `composer require laravel/vonage-notification-channel guzzlehttp/guzzle` (the SMS channel is not in core). Then modify a notification's `via()` so it sends SMS (`vonage`) instead of email when `$notifiable->prefers_sms` is true, add a `toVonage()` method, and add a `routeNotificationForVonage` returning the user's E.164 phone number. Write a test for both branches with `Notification::fake()` and `assertSentTo(..., fn ($n, $ch) => in_array('vonage', $ch))`.

5. **Failover + preview.** Configure a `failover` mailer that tries `smtp` then `log`. Then verify your `ReceiptMail` renders correctly by returning it from a route and by calling `->render()` in `php artisan tinker`.
