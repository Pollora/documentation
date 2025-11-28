# WP CLI Commands

The Pollora framework provides a powerful and declarative way to create WordPress CLI commands using PHP attributes. This system allows you to easily define WP CLI commands that are automatically discovered and registered, with intelligent auto-generation of command slugs.

## Table of Contents

- [Quick Start](#quick-start)
- [Creating Commands](#creating-commands)
- [Automatic Slug Generation](#automatic-slug-generation)
- [Command Structure](#command-structure)
- [Attribute Configuration](#attribute-configuration)
- [Command Generator](#command-generator)
- [Discovery System](#discovery-system)
- [Examples](#examples)
- [Best Practices](#best-practices)

## Quick Start

### 1. Create a Command Class

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

// Slug auto-generated from class name: HelloWorldCommand → hello-world
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

### 2. Use Your Command

```bash
wp hello-world
```

That's it! The command is automatically discovered and registered with WP CLI using an auto-generated slug.

## Creating Commands

### Command Types

WP CLI commands in Pollora can be created in two ways:

1. **Single Command** - Use `#[WpCli]` attribute with `__invoke()` method
2. **Command Suite** - Use `#[WpCli]` attribute with multiple `#[Command]` methods

#### Single Command

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

#[WpCli]
/**
 * Description of my command
 *
 * ## EXAMPLES
 *
 *     wp my-command --option=value
 */
class MyCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        // Your command logic here
        WP_CLI::log('Command executed successfully!');
    }
}
```

#### Command Suite with Subcommands

WP-CLI automatically registers all **public methods** as subcommands. The `#[Command]` attribute is **optional** and mainly useful for:
- Custom subcommand names (different from method names)
- Adding attributes to private/protected methods
- Better organization and explicit declaration

**Option 1: Public methods (automatic registration)**
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
     *
     * ## OPTIONS
     *
     * <username>
     * : The username for the new user
     *
     * <email>
     * : The email address for the new user
     *
     * ## EXAMPLES
     *
     *     wp user-management create-user john john@example.com
     */
    public function createUser(array $arguments, array $options): void
    {
        $username = $arguments[0] ?? null;
        $email = $arguments[1] ?? null;
        
        if (!$username || !$email) {
            WP_CLI::error('Username and email are required.');
            return;
        }
        
        $userId = wp_create_user($username, wp_generate_password(), $email);
        WP_CLI::success("User '{$username}' created with ID {$userId}");
    }
    
    /**
     * Delete a user
     *
     * ## OPTIONS
     *
     * <user_id>
     * : The ID of the user to delete
     */
    public function delete(array $arguments, array $options): void
    {
        $userId = $arguments[0] ?? null;
        
        if (!$userId) {
            WP_CLI::error('User ID is required.');
            return;
        }
        
        wp_delete_user($userId);
        WP_CLI::success("User with ID {$userId} deleted.");
    }
    
    /**
     * List users
     */
    public function listUsers(array $arguments, array $options): void
    {
        $users = get_users(['number' => 10]);
        
        foreach ($users as $user) {
            WP_CLI::log("ID: {$user->ID}, Username: {$user->user_login}, Email: {$user->user_email}");
        }
    }
}
```

**Option 2: Private methods with #[Command] (explicit control)**
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
    #[Command('create')]
    /**
     * Create a new user
     */
    private function createUser(array $arguments, array $options): void
    {
        // Only this method is registered, not auto-discovered
    }
    
    #[Command('delete')]
    /**
     * Delete a user
     */
    private function deleteUser(array $arguments, array $options): void
    {
        // Explicit registration required
    }
    
    // This method is NOT registered (private without #[Command])
    private function helperMethod(): void
    {
        // Internal helper, not exposed as subcommand
    }
}
```

**Usage (all options produce similar results):**
```bash
# Option 1: Auto-generated names from method names
wp user-management create-user john john@example.com
wp user-management delete 123
wp user-management list-users

# Option 2 : Custom command names
wp user-management create john john@example.com
wp user-management remove 123
wp user-management list
```

### Working with Arguments and Options

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

// Auto-generated slug: user-create-command → user-create
#[WpCli]
class UserCreateCommand {
    public function __invoke(array $arguments, array $options): void
    {
        // Access positional arguments
        $username = $arguments[0] ?? null;
        $email = $arguments[1] ?? null;
        
        // Access named options
        $role = $options['role'] ?? 'subscriber';
        $sendEmail = isset($options['send-email']);
        
        if (!$username || !$email) {
            WP_CLI::error('Username and email are required.');
            return;
        }
        
        $userId = wp_create_user($username, wp_generate_password(), $email);
        
        if (is_wp_error($userId)) {
            WP_CLI::error($userId->get_error_message());
            return;
        }
        
        // Set user role
        $user = new WP_User($userId);
        $user->set_role($role);
        
        WP_CLI::success("User '{$username}' created with ID {$userId}");
    }
}
```

**Usage:**
```bash
wp user-create john john@example.com --role=editor --send-email
```

## Automatic Slug Generation

Pollora automatically generates command and subcommand slugs from class and method names, converting camelCase and PascalCase to kebab-case.

### Class Name to Command Slug

- `UserManagementCommand` → `user-management` (removes "Command" suffix)
- `TestAutoSlugCommand` → `test-auto-slug`  
- `CacheHelper` → `cache-helper`
- `AdminToolsCommand` → `admin-tools`

### Method Name to Subcommand Slug

- `createUser()` → `create-user`
- `deleteUserById()` → `delete-user-by-id`
- `listActiveUsers()` → `list-active-users`
- `run()` → `run`

### Custom Slugs

You can override auto-generation by providing custom slugs:

```php
// Custom command slug
#[WpCli('users')]
/**
 * User management commands
 */
class UserManagementCommand
{
    // Custom subcommand slug
    #[Command('new')]
    /**
     * Create a new user
     */
    private function createUser(): void { /* wp users new */ }
    
    // Auto-generated subcommand slug: delete-user
    #[Command]
    /**
     * Delete a user
     */
    private function deleteUser(): void { /* wp users delete-user */ }
}
```

### Private Methods for Clean Command Structure

**Important**: When creating command suites with subcommands, declare your `#[Command]` methods as `private` to ensure only the explicitly defined subcommands are exposed:

```php
#[WpCli('hello')]
/**
 * Hello command suite
 */
class HelloWorldCommand
{
    // These methods are private, so they won't be auto-exposed by WP CLI
    // Only explicitly registered through #[Command] attributes
    
    #[Command('world')]
    private function showWorld(array $arguments, array $options): void
    {
        WP_CLI::success('Hello, World!');
    }

    #[Command]
    private function user(array $arguments, array $options): void
    {
        $name = $arguments[0] ?? 'there';
        WP_CLI::success("Hello, {$name}!");
    }
}
```

**Why use private methods?**
- `wp help hello` shows the main command description from PHPDoc
- Only the subcommands you explicitly define with `#[Command]` are registered
- No unwanted method exposure (like `show-world` from method name `showWorld`)
- Clean command structure with proper aliases (`world` instead of `show-world`)

**Usage:**
```bash
wp help hello          # Shows: "Hello command suite"
wp hello world         # Executes showWorld() method  
wp hello user john     # Executes user() method
```

### PHPDoc Documentation

**Important**: Command descriptions and detailed documentation are automatically extracted from PHPDoc comments. You no longer need to pass descriptions as parameters to the attributes.

```php
#[WpCli('hello')]
/**
 * Hello command suite
 *
 * This is the main description for the hello command.
 * It can span multiple lines and include rich formatting.
 *
 * ## EXAMPLES
 *
 *     wp hello world
 *     wp hello user John
 */
class HelloWorldCommand
{
    #[Command('world')]
    /**
     * Prints a greeting to the world
     *
     * This subcommand prints a friendly greeting to the world.
     *
     * ## OPTIONS
     *
     * [--greeting=<greeting>]
     * : The greeting to use
     * ---
     * default: Hello
     * options:
     *   - Hello
     *   - Hi
     *   - Hey
     * ---
     *
     * [--uppercase]
     * : Make the greeting uppercase
     *
     * ## EXAMPLES
     *
     *     wp hello world
     *     wp hello world --greeting=Hi
     *     wp hello world --greeting=Hey --uppercase
     *
     * @when after_wp_load
     */
    private function showWorld(array $arguments, array $options): void
    {
        $greeting = $options['greeting'] ?? 'Hello';
        if (isset($options['uppercase'])) {
            $greeting = strtoupper($greeting);
        }
        WP_CLI::success("{$greeting}, World!");
    }
}
```

**PHPDoc Features:**
- **Main Description**: First paragraph becomes the short description
- **Extended Description**: Additional paragraphs for detailed help
- **`## OPTIONS`**: Define command arguments and options with defaults and choices
- **`## EXAMPLES`**: Usage examples shown in help
- **`@when`**: Control when the command is available (e.g., `after_wp_load`)
- **Rich Formatting**: Support for Markdown-like syntax

## Command Structure

### The WpCli and Command Attributes

#### Main Command Attribute

```php
#[WpCli(?string $commandName = null)]
```

- **`commandName`** (optional): Custom slug. If null, generated from class name
- **Description**: Extracted from PHPDoc comment

**Examples:**
```php
#[WpCli]                    // Auto: user-management (from UserManagementCommand)
#[WpCli('cache')]          // Custom: cache
```

#### Subcommand Attribute

```php
#[Command(?string $commandName = null)]
```

- **`commandName`** (optional): Custom slug. If null, generated from method name
- **Description**: Extracted from method PHPDoc comment

**Examples:**
```php
#[Command]                         // Auto: create-user (from createUser method)
#[Command('remove')]                   // Custom: remove
```

## WP CLI Attribute Configuration

The Pollora framework supports all standard WP-CLI command attributes through dedicated PHP attributes. These attributes provide fine-grained control over command behavior and registration, mapping directly to WP-CLI's `add_command()` parameters.

### Available Attributes

#### #[Synopsis]
Controls the command synopsis (arguments and options definition).

```php
use Pollora\Attributes\WpCli\Synopsis;

#[Synopsis('<name> [--greeting=<greeting>] [--format=<format>]')]
class GreetCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        $name = $arguments[0];
        $greeting = $options['greeting'] ?? 'Hello';
        $format = $options['format'] ?? 'simple';
        
        // Command logic...
    }
}
```

**Supports:**
- String synopsis: `'<arg> [--option=<value>]'`
- Array synopsis: For complex argument definitions

#### #[When]
Controls when the command is available during WP-CLI execution lifecycle.

```php
use Pollora\Attributes\WpCli\When;

#[When('after_wp_load')]        // Execute after WordPress loads
#[When('before_wp_load')]       // Execute before WordPress loads  
#[When('after_wp_config_load')] // Execute after wp-config.php loads
class MyCommand
{
    // Command implementation...
}
```

**Common hooks:**
- `before_wp_load`: Before WordPress bootstrap
- `after_wp_config_load`: After wp-config.php is loaded
- `after_wp_load`: After full WordPress load (most common)

#### #[BeforeInvoke] / #[AfterInvoke]
Define callbacks executed before/after the command runs.

```php
use Pollora\Attributes\WpCli\BeforeInvoke;
use Pollora\Attributes\WpCli\AfterInvoke;

#[BeforeInvoke([self::class, 'beforeCommand'])]
#[AfterInvoke([self::class, 'afterCommand'])]
class MyCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        // Main command logic
    }
    
    public static function beforeCommand(): void
    {
        WP_CLI::debug('🚀 Before command execution');
        // Setup logic, validation, etc.
    }
    
    public static function afterCommand(): void
    {
        WP_CLI::debug('✅ After command execution');
        // Cleanup, logging, etc.
    }
}
```

**Callback formats:**
- Static method: `[ClassName::class, 'methodName']`
- Instance method: `[$instance, 'methodName']` 
- Function name: `'function_name'`
- Closure: `fn() => ...` (not recommended for attributes)

#### #[IsDeferred]
Controls whether command registration should be deferred.

```php
use Pollora\Attributes\WpCli\IsDeferred;

#[IsDeferred(true)]   // Defer registration (default)
#[IsDeferred(false)]  // Register immediately
class MyCommand
{
    // Command implementation...
}
```

**When to use:**
- `true` (default): For commands that depend on WordPress being loaded
- `false`: For commands that need early registration or don't depend on WordPress

### Complete Example with All Attributes

```php
<?php

declare(strict_types=1);

namespace App\Hooks;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\AfterInvoke;
use Pollora\Attributes\WpCli\BeforeInvoke;
use Pollora\Attributes\WpCli\IsDeferred;
use Pollora\Attributes\WpCli\Synopsis;
use Pollora\Attributes\WpCli\When;
use WP_CLI;

#[WpCli('test')]
#[When('after_wp_load')]
#[Synopsis('<name> [--greeting=<greeting>] [--format=<format>]')]
#[BeforeInvoke([self::class, 'beforeCommand'])]
#[AfterInvoke([self::class, 'afterCommand'])]
#[IsDeferred(false)]
/**
 * Test command with all WP CLI attributes
 * 
 * This demonstrates all available WP CLI attributes:
 * - Synopsis: Custom argument and option definition
 * - When: Execute after WordPress loads
 * - BeforeInvoke/AfterInvoke: Callbacks before and after execution
 * - IsDeferred: Control command registration timing
 */
class TestCommand
{
    /**
     * Say hello to someone with customizable greeting and output format
     *
     * ## ARGUMENTS
     *
     * <name>
     * : The name of the person to greet
     *
     * ## OPTIONS
     *
     * [--greeting=<greeting>]
     * : The greeting to use
     * ---
     * default: Hello
     * ---
     * 
     * [--format=<format>]
     * : Output format
     * ---
     * default: simple
     * options:
     *   - simple
     *   - json
     *   - yaml
     * ---
     *
     * ## EXAMPLES
     *
     *     # Simple greeting
     *     wp test John
     *
     *     # Custom greeting
     *     wp test John --greeting="Bonjour"
     *
     *     # JSON output
     *     wp test John --format=json
     */
    public function __invoke(array $arguments, array $options): void
    {
        $name = $arguments[0];
        $greeting = $options['greeting'] ?? 'Hello';
        $format = $options['format'] ?? 'simple';

        $message = "{$greeting}, {$name}!";
        
        switch ($format) {
            case 'json':
                WP_CLI::print_value([
                    'message' => $message, 
                    'name' => $name, 
                    'greeting' => $greeting
                ], 'json');
                break;
            case 'yaml':
                WP_CLI::print_value([
                    'message' => $message, 
                    'name' => $name, 
                    'greeting' => $greeting
                ], 'yaml');
                break;
            default:
                WP_CLI::success($message);
        }
    }

    /**
     * Callback executed before command
     */
    public static function beforeCommand(): void
    {
        WP_CLI::debug('🚀 Before command execution');
        // Add setup logic, validation, environment checks, etc.
    }

    /**
     * Callback executed after command
     */
    public static function afterCommand(): void
    {
        WP_CLI::debug('✅ After command execution');
        // Add cleanup, logging, notifications, etc.
    }
}
```

**Usage:**
```bash
# Test basic functionality
wp test John

# Test custom synopsis option
wp test John --greeting="Bonjour"

# Test format options
wp test John --format=json
wp test John --format=yaml

# See debug callbacks
wp test John --debug
```

### Method-Level Attributes

For command suites with subcommands, attributes can also be applied to individual methods:

```php
#[WpCli('user')]
/**
 * User management commands
 */
class UserCommand
{
    #[Command('create')]
    #[Synopsis('<username> <email> [--role=<role>]')]
    #[BeforeInvoke([self::class, 'validateUserData'])]
    #[AfterInvoke([self::class, 'logUserCreation'])]
    /**
     * Create a new user
     */
    private function createUser(array $arguments, array $options): void
    {
        // Subcommand implementation
    }
    
    #[Command('delete')]
    #[Synopsis('<user_id> [--reassign=<user_id>]')]
    #[BeforeInvoke([self::class, 'confirmDeletion'])]
    /**
     * Delete a user
     */
    private function deleteUser(array $arguments, array $options): void
    {
        // Subcommand implementation
    }
    
    public static function validateUserData(): void { /* ... */ }
    public static function logUserCreation(): void { /* ... */ }
    public static function confirmDeletion(): void { /* ... */ }
}
```

**Important:** WP-CLI doesn't support having both a main `__invoke` method and subcommands in the same class. Choose one approach:

- **Single Command**: Use `__invoke()` method only
- **Command Suite**: Use `#[Command]` methods only (no `__invoke`)

### Attribute Inheritance

Method-level attributes override class-level attributes:

```php
#[WpCli('base')]
#[When('before_wp_load')]           // Class-level: before_wp_load
#[BeforeInvoke([self::class, 'classCallback'])]
/**
 * Base command
 */
class MyCommand
{
    #[Command('sub')]
    #[When('after_wp_load')]        // Method override: after_wp_load
    #[BeforeInvoke([self::class, 'methodCallback'])]  // Method override
    /**
     * Sub command
     */
    private function subCommand(): void
    {
        // This method uses:
        // - When: after_wp_load (method override)
        // - BeforeInvoke: methodCallback (method override)
    }
}
```

### WP-CLI Argument Reference

These attributes map directly to WP-CLI's `add_command()` arguments:

| Attribute | WP-CLI Argument | Type | Description |
|-----------|-----------------|------|-------------|
| `#[Synopsis]` | `synopsis` | `string\|array` | Command arguments and options |
| `#[When]` | `when` | `string` | WP-CLI hook to execute on |
| `#[BeforeInvoke]` | `before_invoke` | `callable` | Callback before execution |
| `#[AfterInvoke]` | `after_invoke` | `callable` | Callback after execution |
| `#[IsDeferred]` | `is_deferred` | `bool` | Whether registration is deferred |

**Note:** `shortdesc` and `longdesc` are automatically extracted from PHPDoc comments.

### Debugging Attributes

To verify your attributes are working correctly:

1. **Synopsis Override**: Check `wp help <command>` shows your custom synopsis
2. **Callbacks**: Use `--debug` flag to see BeforeInvoke/AfterInvoke messages
3. **When Hook**: Verify command availability based on hook timing
4. **Deferred Registration**: Check command appears at the right lifecycle phase

## Command Generator

Generate new command classes using Artisan:

### Basic Usage

```bash
# Generate simple command (auto slug: test-command)
php artisan pollora:make-wp-cli TestCommand --description="Test command"

# Generate command with subcommands  
php artisan pollora:make-wp-cli UserCommand --subcommands --description="User management"

# Generate in specific locations
php artisan pollora:make-wp-cli ThemeCommand --theme=mytheme --description="Theme utilities"
php artisan pollora:make-wp-cli PluginCommand --plugin=myplugin --description="Plugin tools"
```

### Generator Options

- `--description, -d`: Command description (required)
- `--subcommands, -s`: Generate with subcommands template
- `--force, -f`: Overwrite existing files
- `--theme`: Generate in specific theme
- `--plugin`: Generate in specific plugin
- `--module`: Generate in specific module
- `--path`: Custom generation path

### Generated Examples

**Simple Command:**
```php
#[WpCli]
/**
 * Test command description
 */
class TestCommand
{
    public function __invoke(array $arguments, array $options): void
    {
        WP_CLI::success('TestCommand executed successfully!');
    }
}
```

**Subcommands Template:**
```php
#[WpCli]
/**
 * User management commands
 */
class UserCommand
{
    #[Command]
    /**
     * Create a new item
     */
    public function create(array $arguments, array $options): void { ... }
    
    #[Command]
    /**
     * List all items
     */  
    public function list(array $arguments, array $options): void { ... }
    
    #[Command]
    /**
     * Delete an item
     */
    public function delete(array $arguments, array $options): void { ... }
}
```

## Discovery System

The Pollora framework automatically discovers WP CLI commands through its discovery system:

### How It Works

1. **Discovery Phase**: The `WpCliDiscovery` service scans for classes with the `#[WpCli]` attribute
2. **Slug Generation**: Automatically generates command and subcommand slugs if not provided
3. **Validation Phase**: Ensures classes are instantiable
4. **Registration Phase**: Registers commands with WP CLI using `WP_CLI::add_command()`

### Discovery Locations

Commands are discovered in:
- **Application**: `app/Cms/Commands/`
- **Themes**: `themes/{theme-name}/app/Cms/Commands/`
- **Plugins**: `plugins/{plugin-name}/app/Cms/Commands/`
- **Modules**: Custom module paths

### Manual Registration

If you need manual control, you can register commands programmatically:

```php
use Pollora\WpCli\Application\Services\WpCliService;

$wpCliService = app(WpCliService::class);
$wpCliService->register(
    'manual-command',
    ManualCommand::class,
    'Manually registered command',
    10 // Priority
);
```

## Examples

### 1. Database Cleanup Command

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

// Auto-generated slug: database-cleanup
#[WpCli]
/**
 * Clean up database by removing spam and trash
 */
class DatabaseCleanupCommand {
    public function __invoke(array $arguments, array $options): void
    {
        $dryRun = isset($options['dry-run']);
        
        if ($dryRun) {
            WP_CLI::log('DRY RUN MODE - No changes will be made');
        }
        
        // Clean up spam comments
        $spamComments = get_comments(['status' => 'spam', 'count' => true]);
        if ($spamComments > 0) {
            if (!$dryRun) {
                $this->deleteSpamComments();
            }
            WP_CLI::log("Cleaned up {$spamComments} spam comments");
        }
        
        // Clean up trash posts
        $trashPosts = get_posts(['post_status' => 'trash', 'numberposts' => -1]);
        $trashCount = count($trashPosts);
        if ($trashCount > 0) {
            if (!$dryRun) {
                $this->deleteTrashPosts($trashPosts);
            }
            WP_CLI::log("Cleaned up {$trashCount} trash posts");
        }
        
        WP_CLI::success('Database cleanup completed!');
    }
    
    private function deleteSpamComments(): void
    {
        global $wpdb;
        $wpdb->delete($wpdb->comments, ['comment_approved' => 'spam']);
    }
    
    private function deleteTrashPosts(array $posts): void
    {
        foreach ($posts as $post) {
            wp_delete_post($post->ID, true);
        }
    }
}
```

**Usage:**
```bash
wp database-cleanup --dry-run  # Preview changes
wp database-cleanup            # Execute cleanup
```

### 2. Content Import Command

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use WP_CLI;

// Auto-generated slug: content-import
#[WpCli]
/**
 * Import content from JSON file
 */
class ContentImportCommand {
    public function __invoke(array $arguments, array $options): void
    {
        $file = $arguments[0] ?? null;
        
        if (!$file || !file_exists($file)) {
            WP_CLI::error('Please provide a valid JSON file path.');
            return;
        }
        
        $content = json_decode(file_get_contents($file), true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            WP_CLI::error('Invalid JSON file.');
            return;
        }
        
        $progress = WP_CLI\Progress::start(count($content));
        
        foreach ($content as $item) {
            $this->importItem($item);
            $progress->tick();
        }
        
        $progress->finish();
        WP_CLI::success('Content import completed!');
    }
    
    private function importItem(array $item): void
    {
        $postData = [
            'post_title'   => $item['title'],
            'post_content' => $item['content'],
            'post_status'  => 'publish',
            'post_type'    => $item['type'] ?? 'post',
        ];
        
        wp_insert_post($postData);
    }
}
```

**Usage:**
```bash
wp content-import content.json
```

### 3. Cache Management Command Suite

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\Command;
use WP_CLI;

// Auto-generated slug: cache-management
#[WpCli]
/**
 * Cache management commands
 */
class CacheManagementCommand {
    
    // Auto-generated slug: clear-cache → wp cache-management clear-cache
    #[Command]
    /**
     * Clear various types of cache
     */
    public function clearCache(array $arguments, array $options): void
    {
        $type = $arguments[0] ?? 'all';
        
        match($type) {
            'object' => $this->clearObjectCache(),
            'transients' => $this->clearTransients(),
            'opcache' => $this->clearOpCache(),
            'all' => $this->clearAllCaches(),
            default => WP_CLI::error("Unknown cache type: {$type}")
        };
    }
    
    // Auto-generated slug: flush-object-cache → wp cache-management flush-object-cache
    #[Command]
    /**
     * Flush object cache only
     */
    public function flushObjectCache(array $arguments, array $options): void
    {
        $this->clearObjectCache();
    }
    
    // Auto-generated slug: warmup → wp cache-management warmup
    #[Command]
    /**
     * Warm up cache with common queries
     */
    public function warmup(array $arguments, array $options): void
    {
        // Warm up common queries
        get_posts(['numberposts' => 10]);
        get_pages(['number' => 10]);
        
        WP_CLI::success('Cache warmed up successfully.');
    }
    
    private function clearObjectCache(): void
    {
        wp_cache_flush();
        WP_CLI::success('Object cache cleared.');
    }
    
    private function clearTransients(): void
    {
        global $wpdb;
        $wpdb->query("DELETE FROM {$wpdb->options} WHERE option_name LIKE '_transient_%'");
        WP_CLI::success('Transients cleared.');
    }
    
    private function clearOpCache(): void
    {
        if (function_exists('opcache_reset')) {
            opcache_reset();
            WP_CLI::success('OPCache cleared.');
        } else {
            WP_CLI::warning('OPCache not available.');
        }
    }
    
    private function clearAllCaches(): void
    {
        $this->clearObjectCache();
        $this->clearTransients();
        $this->clearOpCache();
        WP_CLI::success('All caches cleared.');
    }
}
```

**Usage:**
```bash
wp cache-management clear-cache              # Clear all caches
wp cache-management clear-cache object       # Clear object cache only
wp cache-management clear-cache transients   # Clear transients only
wp cache-management clear-cache opcache      # Clear OPCache only
wp cache-management flush-object-cache       # Flush object cache
wp cache-management warmup                   # Warm up cache
```

### 4. Mixed Slug Generation Example

```php
<?php

namespace App\Commands;

use Pollora\Attributes\WpCli;
use Pollora\Attributes\WpCli\Command;
use WP_CLI;

// Custom command slug
#[WpCli('dev')]
/**
 * Development utilities
 */
class DevelopmentUtilitiesCommand {
    
    // Auto-generated: clear-logs → wp dev clear-logs
    #[Command]
    /**
     * Clear application logs
     */
    public function clearLogs(): void { ... }
    
    // Custom slug → wp dev reset
    #[Command('reset')]
    /**
     * Reset development environment
     */
    public function resetEnvironment(): void { ... }
    
    // Auto-generated: generate-test-data → wp dev generate-test-data
    #[Command]
    /**
     * Generate test data
     */
    public function generateTestData(): void { ... }
}
```

## Best Practices

### 1. Command Naming

- **Class names**: Use descriptive names ending with "Command" (auto-removed from slug)
- **Method names**: Use camelCase - automatically converted to kebab-case
- **Descriptions**: Write clear, action-oriented descriptions
- **Avoid conflicts**: Check existing WP CLI commands to avoid naming conflicts

```php
// Good naming examples
class UserManagementCommand    // → user-management
{
    public function createUser()    // → create-user
    public function deleteById()    // → delete-by-id
}
```

### 2. When to Use #[Command] Attribute

The `#[Command]` attribute is **optional** for subcommands. Use it when you need:

**✅ When to use `#[Command]`:**
- Custom subcommand names different from method names
- Adding WP-CLI attributes (Synopsis, BeforeInvoke, etc.) to private/protected methods
- Explicit control over which methods are exposed as subcommands
- Better code organization with descriptive method names

**❌ When NOT to use `#[Command]`:**
- Public methods with good auto-generated names (WP-CLI handles them automatically)
- Simple command structures where method names already match desired command names

```php
// ✅ Good: Use #[Command] for custom naming
class UserCommand {
    #[Command('create')]        // Short name instead of 'show-users'
    public function showUsers() { ... }
    
    #[Command('remove')]        // Better name than 'delete'
    public function delete() { ... }
}

// ✅ Also Good: Let WP-CLI auto-register public methods
class UserCommand {
    public function create() { ... }     // Auto: wp user create
    public function delete() { ... }     // Auto: wp user delete  
    public function listUsers() { ... }  // Auto: wp user list-users
}

// ✅ Best: Private methods with explicit control
class UserCommand {
    #[Command('create')]
    private function createUser() { ... }    // Only this is registered
    
    private function helperMethod() { ... }  // Not exposed (no #[Command])
}
```

### 3. Slug Generation Guidelines

- **Let auto-generation work**: Use descriptive class and method names
- **Use custom slugs sparingly**: Only when auto-generation doesn't fit
- **Keep slugs short**: Avoid overly long command names
- **Be consistent**: Use similar patterns across related commands

```php
// Leverage auto-generation
#[WpCli]           // ✅ Auto: user-management (from UserManagementCommand)
class UserManagementCommand { ... }

// Use custom slug when needed  
#[WpCli('dev')]           // ✅ Custom: dev (shorter)
class DevelopmentToolsCommand { ... }
```

### 3. Error Handling

```php
public function __invoke(array $arguments, array $options): void
{
    try {
        // Command logic
        $this->doSomething();
        WP_CLI::success('Command completed successfully!');
    } catch (\Exception $e) {
        WP_CLI::error('Command failed: ' . $e->getMessage());
    }
}
```

### 4. Input Validation

```php
public function __invoke(array $arguments, array $options): void
{
    // Validate required arguments
    if (empty($arguments[0])) {
        WP_CLI::error('First argument is required.');
        return;
    }
    
    // Validate options
    if (isset($options['limit']) && !is_numeric($options['limit'])) {
        WP_CLI::error('Limit must be a number.');
        return;
    }
}
```

### 5. Progress Indication

For long-running commands, use progress bars:

```php
use WP_CLI\Progress;

public function __invoke(array $arguments, array $options): void
{
    $items = $this->getItems();
    $progress = Progress::start(count($items));
    
    foreach ($items as $item) {
        $this->processItem($item);
        $progress->tick();
    }
    
    $progress->finish();
}
```

### 6. Dry Run Support

For destructive operations, support dry-run mode:

```php
public function __invoke(array $arguments, array $options): void
{
    $dryRun = isset($options['dry-run']);
    
    if ($dryRun) {
        WP_CLI::log('DRY RUN MODE - No changes will be made');
    }
    
    $itemsToDelete = $this->getItemsToDelete();
    
    foreach ($itemsToDelete as $item) {
        if ($dryRun) {
            WP_CLI::log("Would delete: {$item->title}");
        } else {
            $this->deleteItem($item);
            WP_CLI::log("Deleted: {$item->title}");
        }
    }
}
```

### 7. Documentation

Always provide clear descriptions in the PHPDoc comment:

```php
#[WpCli]
/**
 * Performs complex data migration with validation and rollback support
 */
class DataMigrationCommand { ... }
```

### 8. Service Injection

Leverage Laravel's service container for dependencies:

```php
class MyCommand {
    public function __construct(
        private MyService $myService,
        private AnotherService $anotherService
    ) {}
    
    public function __invoke(array $arguments, array $options): void
    {
        $result = $this->myService->doSomething();
        // Use the injected services
    }
}
```

The framework will automatically resolve dependencies through the container.

## Integration with Laravel Services

Your commands can leverage all of Laravel's features:

```php
<?php

namespace App\Commands;

use Illuminate\Support\Facades\Mail;
use Pollora\Attributes\WpCli;
use WP_CLI;

// Auto-generated slug: newsletter-sender
#[WpCli]
/**
 * Send newsletter to subscribers
 */
class NewsletterSenderCommand {
    public function __invoke(array $arguments, array $options): void
    {
        $users = get_users(['meta_key' => 'newsletter_subscriber']);
        
        foreach ($users as $user) {
            Mail::to($user->user_email)
                ->send(new \App\Mail\Newsletter());
        }
        
        WP_CLI::success('Newsletter sent to ' . count($users) . ' subscribers!');
    }
}
```

This powerful combination allows you to create sophisticated CLI commands that leverage both WordPress functionality and Laravel's rich ecosystem, with intelligent slug generation making command creation faster and more intuitive.
