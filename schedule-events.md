# Scheduler

The Schedule attribute provides an elegant way to create and manage WordPress cron jobs. It allows you to easily schedule recurring tasks in your WordPress application using multiple approaches: predefined schedules, enum values, or custom intervals.

## Basic Usage

To schedule a recurring task, simply add the `Schedule` attribute to your method:

```php
use Pollora\Attributes\Schedule;

class Maintenance implements Attributable
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

class Maintenance implements Attributable
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

class Maintenance implements Attributable
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
class Maintenance implements Attributable
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

class Maintenance implements Attributable
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

class Maintenance implements Attributable
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

class SystemMaintenance implements Attributable
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

1. **The class must implement the `Attributable` interface:**
```php
use Pollora\Attributes\Attributable;

class YourClass implements Attributable
{
    // Your scheduled methods here
}
```

2. **After setting up your scheduled tasks, you need to process them:**
```php
use Pollora\Attributes\AttributeProcessor;

$maintenance = new SystemMaintenance();
AttributeProcessor::process($maintenance);
```

3. **Hook names are automatically generated in snake_case format if not specified:**
   - For a class `DatabaseMaintenance` with method `cleanupOldRecords`
   - The generated hook name would be `database_maintenance_cleanup_old_records`

4. **The Schedule attribute ensures that events are only registered once**, even if the processor is called multiple times.

5. **Custom schedules are automatically registered** when using:
   - `Every::MONTH` and `Every::YEAR` enum values
   - `Interval` class instances
   - Legacy array syntax with custom intervals

6. **All syntax variations support the same parameters:**
   - `hook: string` - Custom hook name
   - `args: array` - Arguments passed to the scheduled method