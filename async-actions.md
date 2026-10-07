# Asynchronous Actions

An action becomes asynchronous by adding `->async()` to its registration, or `#[Async]` next to `#[Action]`. When the hook fires, the handler does not run inside the request: it is queued, then run a moment later by WP-Cron, Action Scheduler or a Laravel queue worker. A slow call to a CRM, an ERP or an email API no longer slows down saving a post or paying for an order, and its failure no longer makes that operation fail.

- [Four rules first](#four-rules-first)
- [Making an action asynchronous](#making-an-action-asynchronous)
  - [With the facade](#with-the-facade)
  - [With attributes](#with-attributes)
  - [Generate with Artisan](#generate-with-artisan)
- [Options](#options)
- [What the handler receives](#what-the-handler-receives)
  - [Hook arguments](#hook-arguments)
  - [The context](#the-context)
  - [Capturing values at trigger time](#capturing-values-at-trigger-time)
  - [Skipping the queue](#skipping-the-queue)
- [Drivers](#drivers)
  - [The default driver](#the-default-driver)
  - [Something must run the queue](#something-must-run-the-queue)
  - [Configuration](#configuration)
- [Retries and failures](#retries-and-failures)
- [Closures](#closures)
- [Testing](#testing)
- [Inspecting](#inspecting)
- [Without the framework](#without-the-framework)

## Four rules first

The handler is written exactly like a synchronous one, but its guarantees change:

1. **At least once, not exactly once.** A retry after a failure, or a hook fired twice, can run the handler twice. It must be safe to replay: check before you send, upsert rather than insert.
2. **No guaranteed order.** Two asynchronous actions on the same hook may run in any order.
3. **A variable delay.** A few seconds with a queue worker; with WP-Cron, until something requests `wp-cron.php` (see [Something must run the queue](#something-must-run-the-queue)).
4. **Prefer a class to a closure.** A class runs in the version currently deployed. A closure runs as it was when it was queued.

An asynchronous handler cannot change the current request either: it cannot redirect, print, or alter what the rest of the request reads. Only actions can be asynchronous — a filter returns a value to its caller.

## Making an action asynchronous

### With the facade

```php
use Pollora\Support\Facades\Action;

Action::add('save_post_event', [SyncEventToCrm::class, 'handle'], 10, 2)->async();

// Options chain after async(), in the style of Laravel jobs
Action::add('woocommerce_order_status_completed', [OrderExporter::class, 'export'])
    ->async()
    ->delay(60)
    ->unique()
    ->tries(3);
```

`async()` applies to every hook of the `add()` call just before it. `except` keeps some of them synchronous:

```php
Action::add(['save_post_event', 'save_post_page'], [Indexer::class, 'index'])
    ->async(except: 'save_post_page');
```

A hook in `except` that the registration does not have is an error, to catch typos.

### With attributes

`#[Async]` goes next to `#[Action]`: `#[Action]` keeps the hook and the priority, `#[Async]` carries the options. Removing the line makes the action synchronous again.

```php
use Pollora\Attributes\Action;
use Pollora\Attributes\Async;

class SyncEventToCrm
{
    #[Action('save_post_event', priority: 20)]
    #[Async(delay: 60, tries: 3, unique: true)]
    public function handle(int $postId): void
    {
        // Runs after the request, through the default driver
    }
}
```

- **Several hooks.** `#[Action]` is repeatable; one `#[Async]` applies to all of them, except those in `except`: `#[Async(except: ['save_post_page'])]`.
- **On a class.** `#[Async]` on the class makes every `#[Action]` method asynchronous. A method's own `#[Async]` replaces the class options for that method.

A declaration that cannot be honoured — a hook in `except` the method does not declare, a `capture` or `when` method that is missing or not public, invalid attempts, backoff or lock — is logged and the action **runs synchronously**: the work still happens. So is `#[Async]` on a `#[Filter]` or on a method without `#[Action]`. `pollora:doctor` lists them. Run `php artisan discovery:clear` after adding attributes during development.

### Generate with Artisan

```bash
php artisan pollora:make:action SyncEventToCrm --hook=save_post_event --async
```

The generated method carries `#[Async]`. Run the command again on the same class with another `--hook` to add a method to it.

## Options

Every option exists as a chained method after `async()` and as a named parameter of `#[Async]`.

| Option | Default | Role |
|---|---|---|
| `delay` | `0` | Minimum delay before execution, in seconds or as a `DateInterval` |
| `via` | the default driver | Driver for this action: `queue`, `action-scheduler`, `wp-cron`, `sync` |
| `onQueue` | `hooks.async.queue.queue` | Queue name. The group with Action Scheduler; ignored by WP-Cron |
| `unique` | off | Merges identical triggers (same hook, handler and arguments) until the first one starts. `unique()` locks for a day; `unique(for: 600)` or `#[Async(unique: 600)]` sets the duration in seconds |
| `tries` | `hooks.async.tries` (1) | Attempts when the handler throws |
| `backoff` | `hooks.async.backoff` ([10, 60, 300]) | Seconds before each retry; the last value repeats |
| `asUser` | `hooks.async.as_user` (false) | Runs the handler as the user who fired the hook |
| `capture` | none | Records values at trigger time, see [below](#capturing-values-at-trigger-time) |
| `when` | none | Decides at trigger time whether to queue at all |
| `keepMissing` | off | Runs the handler with `null` in place of an object deleted in the meantime |
| `except` | none | Hooks of the registration that stay synchronous |

With the attribute, `capture` and `when` name a public method of the same class.

## What the handler receives

### Hook arguments

The handler receives the hook arguments, as it would synchronously. They are stored when the hook fires; objects are stored as a reference and reloaded at execution, like Laravel's `SerializesModels`.

| Argument | Stored as | At execution |
|---|---|---|
| Scalar, array of scalars | Its value | Identical |
| `WP_Post`, `WP_Term`, `WP_User`, `WP_Comment` | Type and ID | Reloaded, in its state at that moment |
| Eloquent model | Class and key | Reloaded |
| Enum, date | Value, ISO 8601 date | Rebuilt |
| `JsonSerializable` object | Array | Array |
| Any other object | Refused | — |

An argument that cannot travel throws an exception in debug mode. In production the action runs synchronously and the incident goes to the Laravel log. When a referenced object was deleted before execution, the handler is skipped, unless `keepMissing` is set.

Other parameters typed with a class are resolved from the container, as for a synchronous action:

```php
public function handle(int $postId, CrmClient $crm): void
```

### The context

A parameter typed `AsyncContext`, anywhere in the signature, receives the context of the trigger:

```php
use Pollora\Hook\Async\AsyncContext;

public function handle(int $postId, AsyncContext $context): void
{
    $context->userId;        // user who fired the hook (0 without one)
    $context->blogId;        // site, on a multisite
    $context->locale;
    $context->hook;
    $context->dispatchedAt;  // DateTimeImmutable
    $context->attempt;       // from 1
    $context->get('status'); // a value recorded by capture()
}
```

The site and the locale are restored before the handler runs. The user is not, unless `asUser` is set: without a user, a capability check fails, which is the safe behaviour.

### Capturing values at trigger time

Between the trigger and the execution, things change: there is no current user or request data any more, and the post may have been edited. `capture()` runs at trigger time, in the original request, with the hook arguments, and returns the values to keep:

```php
Action::add('save_post_event', [SyncEventToCrm::class, 'handle'], 10, 3)
    ->async()
    ->capture(fn (int $postId, WP_Post $post, bool $update): array => [
        'status' => $post->post_status, // the state at save time
        'source' => $_POST['acme_source'] ?? 'admin',
    ]);
```

With the attribute, `capture` names a method of the class. A class passed to `Action::add()` that has a public `capture()` method uses it without being told.

```php
#[Action('transition_post_status', priority: 10)]
#[Async(capture: 'captureEditor')]
public function notify(string $new, string $old, WP_Post $post, AsyncContext $context): void
{
    Mail::to($context->get('editorEmail'))->send(new StatusChanged($post, $old, $new));
}

public function captureEditor(string $new, string $old, WP_Post $post): array
{
    return ['editorEmail' => wp_get_current_user()->user_email];
}
```

Captured values follow the same rules as arguments, and they stay in the database, in clear, until execution: capture an ID rather than personal data, never a secret.

### Skipping the queue

`when()` receives the hook arguments and returns a boolean. On `false`, nothing is queued — the place to leave out revisions and autosaves, and the first way to keep a busy hook from flooding the queue (`unique` is the second):

```php
Action::add('save_post', [Indexer::class, 'index'])
    ->async()
    ->when(fn (int $postId): bool => ! wp_is_post_revision($postId) && ! wp_is_post_autosave($postId));
```

## Drivers

| Driver | Queues with | Runs | Needs |
|---|---|---|---|
| `queue` | A Laravel `RunAsyncAction` job | In a queue worker | A worker on the connection |
| `action-scheduler` | `as_enqueue_async_action()` | In Action Scheduler's runner; history in Tools › Scheduled Actions | A plugin bundling it (WooCommerce…), and WP-Cron running |
| `wp-cron` | `wp_schedule_single_event()` | On the next WP-Cron run | WP-Cron running |
| `sync` | Nothing | At once, in the request | — |

Action Scheduler is never required: the driver is used when a plugin that bundles it is active.

### The default driver

The default is `auto`, resolved each time a hook fires: the Laravel queue first **only if `HOOKS_ASYNC_CONNECTION` names a connection**, then Action Scheduler when it is available, then WP-Cron.

The queue is opt-in because the skeleton ships `QUEUE_CONNECTION=database`: with the queue first by default, every asynchronous action of a fresh project would wait for a worker nobody started. A `sync` or `null` connection is always skipped.

`via()` forces a driver for one action. When it is not available as the hook fires, an exception is thrown in debug mode; in production the default driver takes over and the incident is logged.

### Something must run the queue

Nothing tells you when queued work never runs. Check that the driver you use has something behind it:

- **WP-Cron and Action Scheduler.** Pollora sets `DISABLE_WP_CRON`, so page loads never run WP-Cron. Add a system cron that requests it every minute:

  ```bash
  * * * * * curl -s https://example.com/cms/wp-cron.php > /dev/null
  # or, with WP-CLI
  * * * * * cd /path/to/project && wp cron event run --due-now
  ```

  With `wordpress.use_laravel_scheduler`, WP-Cron events are Laravel jobs: a queue worker runs them instead.
- **The queue.** Keep a worker running on the connection, under Supervisor or systemd:

  ```bash
  php artisan queue:work database
  ```

`pollora:doctor` warns when WP-Cron events are an hour late or queued jobs have waited fifteen minutes.

### Configuration

```bash
php artisan vendor:publish --tag=pollora-hooks
```

publishes `config/hooks.php`:

```php
return [
    'async' => [
        // auto, queue, action-scheduler, wp-cron, sync
        'default' => env('HOOKS_ASYNC_DRIVER', 'auto'),

        'tries' => 1,
        'backoff' => [10, 60, 300],
        'as_user' => false,

        'queue' => [
            // The connection a worker runs. null: the default connection, and auto leaves the queue out
            'connection' => env('HOOKS_ASYNC_CONNECTION'),
            'queue' => env('HOOKS_ASYNC_QUEUE', 'default'),
        ],
    ],
];
```

`HOOKS_ASYNC_DRIVER=sync` in a developer's `.env` runs every handler at once, without touching the code.

## Retries and failures

When the handler throws, it is queued again after the `backoff` delay, until `tries` is reached. With the queue, each retry is a new job and the last failure lands in the failed jobs table (`php artisan queue:failed`). Every final failure goes to the Laravel log and fires the `pollora/async/failed` action with the payload and the exception:

```php
#[Action('pollora/async/failed')]
public function alert(AsyncPayload $payload, Throwable $exception): void
```

A handler whose class or method no longer exists fails with an explicit message and is not retried. A handler attached to `save_post` that calls `wp_update_post()` does not queue itself again while it runs.

## Closures

A closure can be asynchronous. It is serialized with `laravel/serializable-closure` and signed with the application key; its signature is checked before it runs. Prefer a class anyway: a closure runs as it was when queued, and changing `APP_KEY` makes the closures still in the queue fail. An anonymous class cannot be queued.

## Testing

`Async::fake()` records what would be queued, without running it:

```php
use Pollora\Hook\Async\Async;
use Pollora\Hook\Async\AsyncPayload;

Async::fake();

wp_update_post(['ID' => $event->ID, 'post_title' => 'New title']);

Async::assertDispatched(SyncEventToCrm::class, fn (AsyncPayload $payload, int $delay): bool =>
    $payload->captured['status'] === 'publish' && $delay === 60
);
Async::assertDispatchedTimes(SyncEventToCrm::class, 1);
Async::assertNotDispatched(Indexer::class);
```

`Async::runDispatched()` then runs what was recorded. Without `fake()`, `HOOKS_ASYNC_DRIVER=sync` runs everything at once, which suits end-to-end tests.

## Inspecting

```bash
php artisan pollora:async:list
php artisan pollora:async:list --json
```

lists every asynchronous action (hook, priority, handler, driver, delay, attempts, lock, queue, user), which drivers are available, and the default driver with where it comes from.

`pollora:doctor` and Tools › Site Health (check *Asynchronous actions*) flag what fails silently: an ignored `#[Async]`, an unavailable driver, a `HOOKS_ASYNC_CONNECTION` naming a `sync` or undefined connection, overdue WP-Cron events, jobs waiting for a worker, and payloads or locks the daily `pollora/async/recover` task left behind.

## Without the framework

The mechanism lives in [`pollora/hook`](https://github.com/Pollora/hook), which a plain WordPress plugin or theme can use on its own, with the same API:

```php
use Pollora\Hook\Action;

Action::add('save_post', [SyncToCrm::class, 'handle'])->async();
```

There, `auto` picks Action Scheduler then WP-Cron, set the driver with the `POLLORA_ASYNC_DRIVER` constant in `wp-config.php` or the `pollora/hook/async_driver` filter, and closures need `laravel/serializable-closure` installed. The `queue` driver, `#[Async]` and `config/hooks.php` come with the framework.

## See also

- [Hooks](hooks.md) — `#[Action]`, `#[Filter]` and the facades
- [Scheduling](schedule-events.md) — recurring tasks with `#[Schedule]`
- [Dashboard & Status](dashboard.md) — `pollora:doctor`
