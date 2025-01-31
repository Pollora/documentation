# Scheduler

The Schedule attribute provides an elegant way to create and manage WordPress cron jobs. It allows you to easily schedule recurring tasks in your WordPress application.

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

## Built-in Schedule Intervals

WordPress provides several default scheduling intervals that you can use:

- `hourly` - Runs once every hour
- `twicedaily` - Runs twice per day
- `daily` - Runs once per day
- `weekly` - Runs once per week

## Custom Scheduling

For more specific timing needs, you can create a custom schedule by providing an array with interval details:

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
class Maintenance implements Attributable
{
    #[Schedule('hourly', 'custom_maintenance_hook')]
    public function performTask(): void
    {
        // Uses 'custom_maintenance_hook' instead of 'maintenance_perform_task'
    }
}
```

## Schedule with Arguments

You can pass additional arguments to your scheduled function:

```php
class Maintenance implements Attributable
{
    #[Schedule(
        'daily',
        'cleanup_task',
        args: ['type' => 'full', 'force' => true]
    )]
    public function cleanup(array $args): void
    {
        $type = $args['type']; // 'full'
        $force = $args['force']; // true
        
        // Perform cleanup based on arguments
    }
}
```

## All-in-One Example

Here's a comprehensive example showing various ways to use the Schedule attribute:

```php
use Pollora\Attributes\Schedule;

class SystemMaintenance implements Attributable
{
    // Basic daily schedule
    #[Schedule('daily')]
    public function standardCleanup(): void
    {
        // Daily cleanup task
    }

    // Custom interval with specific hook name
    #[Schedule(
        ['interval' => 3600 * 4, 'display' => '4 hours'],
        'system_health_check'
    )]
    public function checkHealth(): void
    {
        // Runs every 4 hours
    }

    // Schedule with arguments
    #[Schedule(
        'twicedaily',
        'backup_task',
        args: ['type' => 'incremental']
    )]
    public function performBackup(array $args): void
    {
        // Runs twice per day with specified arguments
    }
}
```

## Important Notes

1. The class must implement the `Attributable` interface:
```php
use Pollora\Attributes\Attributable;

class YourClass implements Attributable
{
    // Your scheduled methods here
}
```

2. After setting up your scheduled tasks, you need to process them:
```php
use Pollora\Attributes\AttributeProcessor;

$maintenance = new SystemMaintenance();
AttributeProcessor::process($maintenance);
```

3. Hook names are automatically generated in snake_case format if not specified:
   - For a class `DatabaseMaintenance` with method `cleanupOldRecords`
   - The generated hook name would be `database_maintenance_cleanup_old_records`

4. The Schedule attribute ensures that events are only registered once, even if the processor is called multiple times.