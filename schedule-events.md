# Scheduler

Pollora schedules tasks for WordPress as code you can read: a `#[Schedule]` attribute on a method, or a [fluent API](#fluent-api) when the event depends on a condition or a computed value. One engine, the [`pollora/schedule`](https://github.com/Pollora/schedule) package, stands behind both, and a [driver](#drivers) decides where events are kept and who runs them: WP-Cron (the default), Action Scheduler, or your Laravel queue.

## Basic Usage

To schedule a recurring task, simply add the `Schedule` attribute to your method:

```php
use Pollora\Attributes\Schedule;

class Maintenance
{
    #[Schedule('daily')]
    public function cleanupDatabase(): void
    {
        // This method will run once per day
        // The hook name will be: maintenance_cleanup_database
    }
}
```

## Using the Every Enum (Recommended)

The framework provides an `Every` enum for type-safe scheduling with predefined intervals:

```php
use Pollora\Attributes\Schedule;
use Pollora\Schedule\Every;

class Maintenance
{
    #[Schedule(Every::HOUR)]
    public function pingSlack(): void
    {
        // Runs every hour
    }

    #[Schedule(Every::DAY)]
    public function dailyBackup(): void
    {
        // Runs once per day
    }

    #[Schedule(Every::WEEK)]
    public function weeklyReport(): void
    {
        // Runs once per week
    }

    #[Schedule(Every::MONTH)]
    public function monthlyCleanup(): void
    {
        // Runs once per month (custom schedule automatically registered)
    }

    #[Schedule(Every::YEAR)]
    public function yearlyArchive(): void
    {
        // Runs once per year (custom schedule automatically registered)
    }
}
```

### Available Every Options

- `Every::HOUR` - Runs once every hour (uses WordPress `hourly`)
- `Every::TWICE_DAILY` - Runs twice per day (uses WordPress `twicedaily`)
- `Every::DAY` - Runs once per day (uses WordPress `daily`)
- `Every::WEEK` - Runs once per week (uses WordPress `weekly`)
- `Every::MONTH` - Runs once per month (automatically creates custom schedule)
- `Every::YEAR` - Runs once per year (automatically creates custom schedule)

## Built-in Schedule Intervals (Legacy)

WordPress provides several default scheduling intervals that you can still use with strings:

- `hourly` - Runs once every hour
- `twicedaily` - Runs twice per day
- `daily` - Runs once per day
- `weekly` - Runs once per week

## Custom Intervals with Interval Class

For precise timing control, use the `Interval` class which provides a fluent API:

```php
use Pollora\Attributes\Schedule;
use Pollora\Schedule\Interval;

class Maintenance
{
    #[Schedule(new Interval(hours: 3, minutes: 30))]
    public function checkFeeds(): void
    {
        // Runs every 3 hours and 30 minutes
    }

    #[Schedule(new Interval(
        days: 1,
        hours: 12,
        minutes: 15,
        display: 'Daily plus 12h15m'
    ))]
    public function complexTask(): void
    {
        // Runs every 1 day, 12 hours, and 15 minutes
    }

    #[Schedule(new Interval(weeks: 2, display: 'Bi-weekly'))]
    public function biweeklyTask(): void
    {
        // Runs every 2 weeks
    }
}
```

### Interval Constructor Parameters

- `seconds: int = 0` - Additional seconds
- `minutes: int = 0` - Additional minutes  
- `hours: int = 0` - Additional hours
- `days: int = 0` - Additional days
- `weeks: int = 0` - Additional weeks
- `display: string = 'Custom schedule'` - Human-readable description

## Custom Scheduling (Legacy Array Syntax)

For backward compatibility, you can still create custom schedules using arrays:

```php
class Maintenance
{
    #[Schedule(
        ['interval' => 3600 * 6, 'display' => '6 hours']
    )]
    public function performCheck(): void
    {
        // Runs every 6 hours
    }
}
```

The custom schedule array requires:
- `interval`: Time in seconds between executions
- `display`: Human-readable name for the schedule

## Custom Hook Names

By default, the hook name is generated from the class and method names, but you can specify a custom hook name:

```php
use Pollora\Schedule\Every;
use Pollora\Schedule\Interval;

class Maintenance
{
    // With string schedule
    #[Schedule('hourly', hook: 'custom_maintenance_hook')]
    public function performTask(): void
    {
        // Uses 'custom_maintenance_hook' instead of 'maintenance_perform_task'
    }

    // With Every enum
    #[Schedule(Every::DAY, hook: 'daily_cleanup')]
    public function cleanup(): void
    {
        // Uses 'daily_cleanup' hook name
    }

    // With Interval class
    #[Schedule(new Interval(hours: 6), hook: 'six_hourly_check')]
    public function checkSystem(): void
    {
        // Uses 'six_hourly_check' hook name
    }
}
```

## Schedule with Arguments

You can pass additional arguments to your scheduled function with any schedule type:

```php
use Pollora\Schedule\Every;
use Pollora\Schedule\Interval;

class Maintenance
{
    // With string schedule
    #[Schedule(
        'daily',
        hook: 'cleanup_task',
        args: ['type' => 'full', 'force' => true]
    )]
    public function cleanup(array $args): void
    {
        $type = $args['type']; // 'full'
        $force = $args['force']; // true
        
        // Perform cleanup based on arguments
    }

    // With Every enum and arguments
    #[Schedule(Every::DAY, hook: 'custom_hook', args: ['type' => 'full'])]
    public function updateIndex(array $args): void
    {
        // Process with arguments
    }

    // With Interval class and arguments
    #[Schedule(
        new Interval(hours: 2, minutes: 30),
        hook: 'data_sync',
        args: ['source' => 'api', 'batch_size' => 100]
    )]
    public function syncData(array $args): void
    {
        // Sync data with specified parameters
    }
}
```

## All-in-One Example

Here's a comprehensive example showing various ways to use the Schedule attribute:

```php
use Pollora\Attributes\Schedule;
use Pollora\Schedule\Every;
use Pollora\Schedule\Interval;

class SystemMaintenance
{
    // Using Every enum (recommended)
    #[Schedule(Every::DAY)]
    public function dailyCleanup(): void
    {
        // Daily cleanup using enum
    }

    // Custom interval using Interval class
    #[Schedule(new Interval(hours: 4, display: '4 hours'))]
    public function systemHealthCheck(): void
    {
        // Runs every 4 hours
    }

    // Monthly schedule with custom hook (auto-registered)
    #[Schedule(Every::MONTH, hook: 'monthly_reports')]
    public function generateReports(): void
    {
        // Monthly report generation
    }

    // Complex interval with arguments
    #[Schedule(
        new Interval(hours: 2, minutes: 30),
        hook: 'data_sync',
        args: ['type' => 'incremental', 'source' => 'api']
    )]
    public function syncData(array $args): void
    {
        // Runs every 2.5 hours with arguments
    }

    // Legacy string syntax (still supported)
    #[Schedule('twicedaily')]
    public function legacyTask(): void
    {
        // Backward compatibility
    }

    // Legacy array syntax (still supported)
    #[Schedule(
        ['interval' => 3600 * 6, 'display' => '6 hours'],
        hook: 'legacy_check'
    )]
    public function legacyCustomTask(): void
    {
        // Backward compatibility with arrays
    }
}
```

## Schedule Type Summary

| Type | Syntax | Use Case | Custom Registration |
|------|--------|----------|-------------------|
| **Every Enum** | `Every::DAY` | Type-safe, predefined intervals | Auto for MONTH/YEAR |
| **Interval Class** | `new Interval(hours: 2)` | Precise custom timing | Always |
| **String** | `'daily'` | WordPress built-ins | Never |
| **Array** | `['interval' => 3600]` | Legacy custom intervals | Always |

## Important Notes

1. **Hook names are automatically generated in snake_case format if not specified:**
   - For a class `DatabaseMaintenance` with method `cleanupOldRecords`
   - The generated hook name would be `database_maintenance_cleanup_old_records`

2. **The Schedule attribute ensures that events are only registered once** through automatic discovery.

3. **Custom schedules are automatically registered** when using:
   - `Every::MONTH` and `Every::YEAR` enum values
   - `Interval` class instances
   - Legacy array syntax with custom intervals

4. **All syntax variations support the same parameters:**
   - `hook: string` - Custom hook name
   - `args: array` - Arguments passed to the scheduled method
## Fluent API

When the schedule is not known at class-definition time, declare the event in code. Declare it on every request: the handler is attached each time, and the event is scheduled only when none with the same hook and arguments exists.

```php
use Pollora\Schedule\Interval;
use Pollora\Schedule\Schedule;

Schedule::event('reports_compile')->daily()->at('02:00')->call([Reports::class, 'compile']);
Schedule::event('cache_warm')->every(new Interval(minutes: 15))->call(WarmCache::class);

Schedule::once('send_digest')->in(minutes: 10)->with(['user' => 42])->call(SendDigest::class);
Schedule::once('close_poll')->at($poll->closesAt)->call([Polls::class, 'close']);

Schedule::cancel('send_digest', ['user' => 42]);
Schedule::next('reports_compile');   // DateTimeImmutable, or null
Schedule::events();                  // everything that is scheduled, whoever scheduled it
```

`at('02:00')` follows the clock and the timezone of the site. `call()` takes an invokable class, `[Class::class, 'method']`, a closure, or the name of a WordPress hook to fire; classes are built through Laravel's container, so their dependencies are injected.

### Testing

```php
Schedule::fake();

$checkout->complete($order);

Schedule::assertScheduled('send_receipt', fn (Event $event) => $event->args === ['order' => 12]);
Schedule::assertNotScheduled('send_survey');
```

Nothing is scheduled and no handler is attached while faking; `Schedule::reset()` brings the real driver back.

## Drivers

The driver is set in `config/schedule.php` (publish it with `php artisan vendor:publish --tag=pollora-schedule`) or with `POLLORA_SCHEDULE_DRIVER`. It decides where **every** event goes: `#[Schedule]`, `Schedule::event()`, and the `wp_schedule_event()` calls of WordPress, plugins and themes.

| Driver | Where events live | Who runs them | Needs |
|---|---|---|---|
| `wp-cron` (default) | The `cron` option | WP-Cron, requested by a system cron | Nothing |
| `action-scheduler` | Action Scheduler, in a group of its own (`pollora/schedule`) | Action Scheduler's queue runner, itself run by WP-Cron | A plugin that bundles it (WooCommerce does) |
| `laravel` | A delayed job per event in the application's queue | A queue worker | A queue connection that accepts delays, and `php artisan queue:work` |

```php
// config/schedule.php
'driver' => env('POLLORA_SCHEDULE_DRIVER', 'wp-cron'),

// hooks that stay on WP-Cron whatever the driver
'protected_hooks' => [],
```

A driver that is not available (Action Scheduler without a plugin that bundles it, a `sync` queue) leaves every event on WP-Cron, and `pollora:doctor` says so. Action Scheduler's own queue runner always stays on WP-Cron: handed to Action Scheduler, it would never run.

### The `cron` option indexes every event

Whatever the driver, the `cron` option lists every scheduled event, with a `driver` key on the entries another driver keeps. WordPress reads it without a query, so `wp_next_scheduled()`, Site Health, WP Crontrol and Query Monitor see the events, and WP-Cron never runs one it does not own. A plugin that rewrites the whole array still works: removing or moving an entry reaches the driver that keeps it.

### The `laravel` driver

```env
POLLORA_SCHEDULE_DRIVER=laravel
```

```bash
php artisan queue:work
```

Each event becomes a job delayed until it is due, in the application's queue (`database`, Redis, Horizon, SQS). There is no table of Pollora's own and no `schedule:run`: a worker is all that runs the events. Without one, none of them fires, and `pollora:doctor` reports events more than ten minutes late.

```php
// config/schedule.php
'laravel' => [
    'connection' => env('POLLORA_SCHEDULE_QUEUE_CONNECTION'), // null: the default connection
    'queue' => env('POLLORA_SCHEDULE_QUEUE'),                 // null: the connection's default queue
    'max_delay' => env('POLLORA_SCHEDULE_MAX_DELAY'),         // null: 900 s on SQS, no limit elsewhere
    'late_after' => (int) env('POLLORA_SCHEDULE_LATE_AFTER', 600),
],
```

How it behaves:

- **The `cron` option is the record.** Cancelling an event removes its entry; the job still wakes, finds no entry (or one with another identifier or time), and stops. Nothing reads the queue.
- **Recurring events** queue their next occurrence once they ran, or failed for good: never in the past, so a worker that was down does not replay every missed run.
- **Retries.** A job is tried three times (30 s, then 5 min apart). After that the failure lands in `failed_jobs` as usual, a recurring event goes on, and a single event leaves the schedule. `pollora:doctor` reports failures of the last week.
- **Delays longer than the connection allows** (15 minutes on SQS) are chained: the job wakes, sees it is early, and waits again.
- **Lost jobs.** A flushed Redis or a purged queue loses jobs. As a request ends, at most every five minutes, Pollora checks the index, and an event more than `late_after` seconds past its time is queued again **with the same identifier**, so it never runs twice. Run the check by hand with:

  ```bash
  php artisan pollora:schedule:recover            # --dry-run lists, --network covers every site
  ```

- **Multisite.** The queue is shared by the network, with a single worker. Each job runs in the site that scheduled the event, whose `cron` option is its index.

`wp cron event run --due-now` is WP-CLI's manual command: it runs every due event, whoever keeps it, and takes it off the schedule; the job that wakes later stops.

## Listing events

```bash
php artisan pollora:schedule:list                 # --type=once|recurring, --driver=laravel, --site=2, --network, --json
wp cron event list --fields=hook,type,driver      # WP-CLI, with type and driver columns and the same filters
```

Both read the `cron` option only: no query reaches Action Scheduler or the queue. A driver column reads `wp-cron (protected)` for a hook that stays on WP-Cron on purpose, `(fallback)` when the driver is not available, and `(to migrate)` for an event a former driver still keeps.

## Changing driver

Changing `POLLORA_SCHEDULE_DRIVER` sends **new** events to the new driver. Events already scheduled stay where they are, and keep running there, until you move them:

```bash
php artisan pollora:schedule:migrate --dry-run                       # what would move
php artisan pollora:schedule:migrate                                 # from WP-Cron to the driver of the site
php artisan pollora:schedule:migrate --from=laravel --to=wp-cron     # and back
```

An event is created in the target before it leaves the source, so a crash duplicates it instead of losing it, and running the command again does not create it twice. `pollora:doctor`, Site Health and an administration notice remind you while events are left to move.

## Checks

`pollora:doctor` and Tools › Site Health (check *Scheduled events*) report a driver that does not exist or is not available, events a former driver still keeps, late events when the `laravel` driver has no worker, and a scheduled event that failed for good.

## Upgrading from `use_laravel_scheduler`

The option `wordpress.use_laravel_scheduler` and the bridge behind it are removed, without a deprecation period: `Pollora\Schedule\Scheduler`, `SchedulerInterface`, `Events\*`, `Jobs\JobDispatcher`, the `pre_schedule_event` and `pre_option_cron` filters it installed, and the `wp_events` table. Setting the option no longer does anything.

To keep WP-Cron events on Laravel's queue:

1. Set `POLLORA_SCHEDULE_DRIVER=laravel`.
2. Run a worker (`php artisan queue:work`) in place of `schedule:run`.
3. Let `#[Schedule]` and your own code schedule their events again, then move what WP-Cron still holds with `php artisan pollora:schedule:migrate`.
4. Drop the old table: `DROP TABLE <prefix>wp_events`, with the prefix of your database connection. Events that sat in it are not moved.

## Outside Pollora

The engine runs without Laravel, in a plugin or a Bedrock site: see [`pollora/schedule`](https://github.com/Pollora/schedule).
