# WP CLI Commands

The Pollora framework provides a declarative way to create WordPress CLI commands using PHP attributes. Commands are automatically discovered and registered with intelligent slug auto-generation.

## Table of Contents

- [Quick Start](#quick-start)
- [Single Commands](#single-commands)
- [Command Suites (Subcommands)](#command-suites-subcommands)
- [Automatic Slug Generation](#automatic-slug-generation)
- [WP-CLI Attributes](#wp-cli-attributes)
- [Command Generator](#command-generator)
- [Examples](#examples)
- [Best Practices](#best-practices)

## Quick Start

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

#[WpCli]
/**
 * A simple hello world command
 */
class HelloWorldCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        WP_CLI::success('Hello, World!');
    }
}
```

```bash
wp hello-world
```

That's it! The command slug is auto-generated from the class name.

## Single Commands

For commands without subcommands, use the `__invoke()` method:

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

#[WpCli]
/**
 * Create a new user
 *
 * ## OPTIONS
 *
 * <username>
 * : The username for the new user
 *
 * [--role=<role>]
 * : The user role
 * ---
 * default: subscriber
 * ---
 *
 * ## EXAMPLES
 *
 *     wp user-create john --role=editor
 */
class UserCreateCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        $username = $arguments[0] ?? null;
        $role = $options['role'] ?? 'subscriber';
        
        if (!$username) {
            WP_CLI::error('Username is required.');
            return;
        }
        
        WP_CLI::success("User '{$username}' created with role '{$role}'");
    }
}
```

## Command Suites (Subcommands)

WP-CLI automatically registers all **public methods** as subcommands, under the method name as written: `listUsers()` answers to `listUsers`. For a kebab-case name, put `#[Command]` on the method: with no argument it derives `list-users` from the method name (see [Custom Subcommand Names](#custom-subcommand-names-with-command)).

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

#[WpCli]
/**
 * User management commands
 */
class UserManagementCommand
{
    /**
     * Create a new user
     */
    public function create(array $arguments, array $options): void
    {
        WP_CLI::success('User created');
    }
    
    /**
     * Delete a user
     */
    public function delete(array $arguments, array $options): void
    {
        WP_CLI::success('User deleted');
    }
    
    /**
     * List all users
     */
    public function listUsers(array $arguments, array $options): void
    {
        WP_CLI::success('Listing users...');
    }
}
```

```bash
wp user-management create
wp user-management delete
wp user-management listUsers
```

### Custom Subcommand Names with #[Command]

Use `#[Command]` **only when you need a custom slug** different from the method name:

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\Command;
use WP_CLI;

#[WpCli]
/**
 * User management commands
 */
class UserManagementCommand
{
    #[Command('new')]  // Custom slug instead of "createUser"
    /**
     * Create a new user
     */
    public function createUser(array $arguments, array $options): void
    {
        WP_CLI::success('User created');
    }
    
    #[Command('rm')]  // Custom slug instead of "deleteUser"
    /**
     * Delete a user
     */
    public function deleteUser(array $arguments, array $options): void
    {
        WP_CLI::success('User deleted');
    }
    
    // No #[Command] needed - the slug is the method name, "list"
    public function list(array $arguments, array $options): void
    {
        WP_CLI::success('Listing users...');
    }
}
```

```bash
wp user-management new      # Uses custom slug
wp user-management rm       # Uses custom slug
wp user-management list     # Uses the method name
```

## Automatic Slug Generation

### Class Name → Command Slug

The "Command" suffix is automatically removed:

| Class Name | Generated Slug |
|------------|----------------|
| `UserManagementCommand` | `user-management` |
| `DatabaseCleanupCommand` | `database-cleanup` |
| `CacheHelper` | `cache-helper` |

### Method Name → Subcommand Slug

Without `#[Command]`, WP-CLI uses the method name as written. Only a bare `#[Command]` (no argument) derives a kebab-case slug from it.

| Method Name | Without `#[Command]` | With bare `#[Command]` |
|-------------|----------------------|------------------------|
| `createUser()` | `createUser` | `create-user` |
| `deleteById()` | `deleteById` | `delete-by-id` |
| `list()` | `list` | `list` |

### Custom Slugs

Override auto-generation when needed:

```php
#[WpCli('cache')]  // Custom command slug
class CacheManagementCommand
{
    #[Command('flush')]  // Custom subcommand slug
    public function clearAllCaches(): void { }
    
    public function warmup(): void { }  // Method name: "warmup"
}
```

```bash
wp cache flush    # Custom slug
wp cache warmup   # Method name
```

## WP-CLI Attributes

### #[Synopsis]

Define command arguments and options:

```php
use Pollora\Attributes\WpCli\Synopsis;

#[WpCli]
#[Synopsis('<name> [--greeting=<greeting>] [--uppercase]')]
class GreetCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        $name = $arguments[0];
        $greeting = $options['greeting'] ?? 'Hello';
        WP_CLI::success("{$greeting}, {$name}!");
    }
}
```

### #[When]

Control when the command is available:

```php
use Pollora\Attributes\WpCli\When;

#[WpCli]
#[When('after_wp_load')]  // Execute after WordPress loads (most common)
class MyCommand { }

#[WpCli]
#[When('before_wp_load')]  // Execute before WordPress loads
class EarlyCommand { }
```

### #[BeforeInvoke] / #[AfterInvoke]

Execute callbacks before/after the command:

```php
use Pollora\Attributes\WpCli\BeforeInvoke;
use Pollora\Attributes\WpCli\AfterInvoke;

#[WpCli]
#[BeforeInvoke([self::class, 'setup'])]
#[AfterInvoke([self::class, 'cleanup'])]
class MyCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        WP_CLI::success('Command executed');
    }
    
    public static function setup(): void
    {
        WP_CLI::debug('Setting up...');
    }
    
    public static function cleanup(): void
    {
        WP_CLI::debug('Cleaning up...');
    }
}
```

### #[IsDeferred]

Control registration timing:

```php
use Pollora\Attributes\WpCli\IsDeferred;

#[WpCli]
#[IsDeferred(false)]  // Register immediately (default is true)
class MyCommand { }
```

### Complete Example

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\Synopsis;
use Pollora\Attributes\WpCli\When;
use Pollora\Attributes\WpCli\BeforeInvoke;
use WP_CLI;

#[WpCli('greet')]
#[When('after_wp_load')]
#[Synopsis('<name> [--greeting=<greeting>] [--format=<format>]')]
#[BeforeInvoke([self::class, 'validate'])]
/**
 * Greet someone with style
 *
 * ## EXAMPLES
 *
 *     wp greet John
 *     wp greet John --greeting="Bonjour"
 */
class GreetCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        $name = $arguments[0];
        $greeting = $options['greeting'] ?? 'Hello';
        
        WP_CLI::success("{$greeting}, {$name}!");
    }
    
    public static function validate(): void
    {
        WP_CLI::debug('Validating input...');
    }
}
```

## Command Generator

Generate command classes with Artisan:

```bash
# Basic command
php artisan pollora:make:wp-cli TestCommand --description="Test command"

# In a theme
php artisan pollora:make:wp-cli ThemeCommand --theme=mytheme --description="Theme utilities"

# In a plugin
php artisan pollora:make:wp-cli PluginCommand --plugin=myplugin --description="Plugin tools"

# In a module
php artisan pollora:make:wp-cli ModuleCommand --module=Portfolio --description="Module tools"
```

The command is also available as `pollora:make-wp-cli`.

### What it generates

`php artisan pollora:make:wp-cli TestCommand --description="Test command"` writes `app/Cms/Commands/TestCommand.php`:

```php
<?php

declare(strict_types=1);

namespace App\Cms\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

/**
 * Test command
 */
#[WpCli]
class TestCommand
{
    /**
     * Execute the WP CLI command.
     *
     * @param array $arguments Positional arguments passed to the command
     * @param array $options Named options passed to the command
     * @return void
     */
    public function __invoke(array $arguments, array $options): void
    {
        // Add your command logic here
        WP_CLI::success('TestCommand executed successfully!');
    }
}
```

The class is a single command (`wp test`: the `Command` suffix is dropped, the rest is kebab-cased). Add `#[Command]` methods yourself for subcommands. The folder is `Cms/Commands` under the `app/` directory of the target, and the namespace follows it:

| Target | File | Namespace |
|--------|------|-----------|
| Application | `app/Cms/Commands/` | `App\Cms\Commands` |
| `--theme=mytheme` | `themes/mytheme/app/Cms/Commands/` | `Theme\Mytheme\Cms\Commands` |
| `--plugin=myplugin` | `{WP_PLUGIN_DIR}/myplugin/app/Cms/Commands/` | `Plugin\Myplugin\Cms\Commands` |
| `--module=Portfolio` | `Modules/Portfolio/app/Cms/Commands/` | The module's PSR-4 prefix for `app/`, else `Modules\Portfolio\Cms\Commands` |

### Options

| Option | Description |
|--------|-------------|
| `--description, -d` | Command description, written in the class docblock (default: `A custom WP CLI command`) |
| `--force, -f` | Overwrite existing files |
| `--theme` | Generate in specific theme |
| `--plugin` | Generate in specific plugin |
| `--module` | Generate in specific module |
| `--path` | Custom generation path |

## Examples

### Database Cleanup

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

#[WpCli]
/**
 * Clean up database
 */
class DatabaseCleanupCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        $dryRun = isset($options['dry-run']);
        
        if ($dryRun) {
            WP_CLI::log('DRY RUN MODE');
        }
        
        $spamCount = get_comments(['status' => 'spam', 'count' => true]);
        
        if (!$dryRun && $spamCount > 0) {
            global $wpdb;
            $wpdb->delete($wpdb->comments, ['comment_approved' => 'spam']);
        }
        
        WP_CLI::success("Cleaned {$spamCount} spam comments");
    }
}
```

### Cache Management Suite

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\Command;
use WP_CLI;

#[WpCli('cache')]
/**
 * Cache management commands
 */
class CacheCommand
{
    #[Command('flush')]
    /**
     * Flush all caches
     */
    public function flushAll(array $arguments, array $options): void
    {
        wp_cache_flush();
        WP_CLI::success('All caches flushed');
    }
    
    public function warmup(array $arguments, array $options): void
    {
        get_posts(['numberposts' => 10]);
        WP_CLI::success('Cache warmed up');
    }
}
```

```bash
wp cache flush
wp cache warmup
```

## Best Practices

### Error Handling

```php
public function __invoke(array $arguments, array $options): void
{
    try {
        $this->doSomething();
        WP_CLI::success('Done!');
    } catch (\Exception $e) {
        WP_CLI::error($e->getMessage());
    }
}
```

### Input Validation

```php
public function __invoke(array $arguments, array $options): void
{
    if (empty($arguments[0])) {
        WP_CLI::error('First argument is required.');
        return;
    }
}
```

### Progress Bars

```php
public function __invoke(array $arguments, array $options): void
{
    $items = $this->getItems();
    $progress = \WP_CLI\Utils\make_progress_bar('Processing', count($items));
    
    foreach ($items as $item) {
        $this->process($item);
        $progress->tick();
    }
    
    $progress->finish();
}
```

### Dry Run Support

```php
public function __invoke(array $arguments, array $options): void
{
    $dryRun = isset($options['dry-run']);
    
    foreach ($items as $item) {
        if ($dryRun) {
            WP_CLI::log("Would delete: {$item->title}");
        } else {
            $this->delete($item);
        }
    }
}
```

### Service Injection

The command class is built with its constructor dependencies, and that instance runs the command and every subcommand. This needs `pollora/wp-cli` 0.1.1 or later (Pollora 13.36): before it, WP-CLI built the class itself, without arguments.

```php
class MyCommand
{
    public function __construct(
        private MyService $service
    ) {}
    
    public function __invoke(array $arguments, array $options): void
    {
        $this->service->doSomething();
    }
}
```

## Discovery Locations

Commands are automatically discovered in:

- `app/Cms/Commands/`
- `themes/{theme}/app/Cms/Commands/`
- `plugins/{plugin}/app/Cms/Commands/`
- `Modules/{Module}/app/Cms/Commands/`

`Cms/Commands` is where the generator writes, not a requirement: Pollora scans the whole `app/` directory of the application, of each theme and plugin (`src/` when there is no `app/`) and of each module, and registers every class that carries `#[WpCli]`. A command generated with `--module` is therefore discovered.
