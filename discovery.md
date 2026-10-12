# Discovery System

The Pollora Discovery system provides automatic component discovery in your application, WordPress themes, and Laravel modules. It uses a modern architecture inspired by Tempest Framework while leveraging Spatie's structure discovery capabilities to identify and process different types of components according to your needs.

The engine lives in its own package, [`pollora/discovery`](https://github.com/Pollora/discovery), which the framework installs and wires: in a Pollora project there is nothing to install. The package also runs without Laravel, in a WordPress plugin or any PHP project; see [Outside Laravel](#outside-laravel).

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Built-in Discovery Classes](#built-in-discovery-classes)
- [Skipping Discovery](#skipping-discovery)
- [API Usage](#api-usage)
- [Creating Custom Discovery Classes](#creating-custom-discovery-classes)
- [Discovery Engine](#discovery-engine)
- [Caching System](#caching-system)
- [Outside Laravel](#outside-laravel)
- [Console Commands](#console-commands)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)
- [Migration from Old Discoverer](#migration-from-old-discoverer)
- [Troubleshooting](#troubleshooting)

## Overview

The Discovery system automatically scans your codebase to find and register components like:

- **Custom Post Types** with `#[PostType]` attributes
- **Custom Taxonomies** with `#[Taxonomy]` attributes
- **Scheduled Tasks** with `#[Schedule]` attributes
- **WordPress Hooks** with `#[Action]` and `#[Filter]` attributes
- **REST API Routes** with `#[WpRestRoute]` attributes
- **Custom Components** through extensible discovery classes

### Key Features

- **Modern API**: Inspired by Tempest Framework's discovery system
- **Performance Optimized**: Built on Spatie's structure discoverer with caching
- **Domain-Driven Design**: Clean architecture with proper separation of concerns
- **Extensible**: Easy to create custom discovery classes for any component type
- **Type Safe**: Full PHPDoc coverage and strict type declarations
- **Laravel Integration**: Seamless integration with Laravel's service container
- **Standalone**: the engine (`pollora/discovery`) runs without Laravel's application

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    DiscoveryManager                         │
│                   (High-level API)                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    DiscoveryEngine                          │
│                 (Orchestration Layer)                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
┌───▼─────────┐  ┌─────────▼──────┐  ┌───────────▼──────────┐
│ Discovery   │  │   Discovery    │  │   Discovery Cache    │
│ Locations   │  │   Classes      │  │   (Spatie driver)    │
└─────────────┘  └────────────────┘  └──────────────────────┘
```

### Discovery Flow

1. **Registration**: Discovery classes are registered with unique identifiers
2. **Location Setup**: Discovery locations (namespaces + paths) are configured
3. **Discovery Phase**: Engine scans all locations using registered discoveries
4. **Application Phase**: Discovered items are applied/registered with the framework
5. **Caching**: Results are cached for performance in production

## Built-in Discovery Classes

### PostType Discovery
**Location**: `Pollora\PostType\Infrastructure\Services\PostTypeDiscovery`
**Discovers**: Classes with `#[PostType]` attributes

```php
#[PostType('product')]
#[PublicPostType]
class Product
{
    // Post type implementation
}
```

### Taxonomy Discovery
**Location**: `Pollora\Taxonomy\Infrastructure\Services\TaxonomyDiscovery`
**Discovers**: Classes with `#[Taxonomy]` attributes

```php
#[Taxonomy('product-category')]
#[Hierarchical]
class ProductCategory
{
    // Taxonomy implementation
}
```

### Schedule Discovery
**Location**: `Pollora\Schedule\Infrastructure\Services\ScheduleDiscovery`
**Discovers**: Methods with `#[Schedule]` attributes

```php
class TaskManager
{
    #[Schedule('daily')]
    public function cleanupTempFiles(): void
    {
        // Scheduled task implementation
    }
}
```

### Hook Discovery
**Location**: `Pollora\Hook\Infrastructure\Services\HookDiscovery`
**Discovers**: Methods with `#[Action]` or `#[Filter]` attributes

```php
class UserHooks
{
    #[Action('user_register')]
    public function onUserRegister(int $userId): void
    {
        // Action handler
    }

    #[Filter('the_content')]
    public function filterContent(string $content): string
    {
        // Filter handler
        return $content;
    }
}
```

### WP REST Discovery
**Location**: `Pollora\WpRest\Infrastructure\Services\WpRestDiscovery`
**Discovers**: Methods with `#[WpRestRoute]` attributes

```php
class ApiController
{
    #[WpRestRoute('v1', '/products', ['GET'])]
    public function getProducts(): array
    {
        // REST endpoint implementation
        return [];
    }
}
```

## Skipping Discovery

### Excluding a Class Entirely

Use the `#[SkipDiscovery]` attribute to prevent a class from being processed by the discovery engine. No reflection will be loaded and no attributes will be scanned — the class is completely invisible to all discoveries.

```php
use Pollora\Attributes\SkipDiscovery;

#[SkipDiscovery]
class InternalHelper
{
    // This class will never be discovered, regardless of its attributes
}
```

This is useful for:
- Base classes or abstract helpers that should not be auto-registered
- Test fixtures or development-only classes
- Classes with attributes that are processed by external systems

### Selective Exclusion with `except`

Sometimes you want a class to be ignored by most discoveries but still processed by specific ones. Use the `except` parameter to list the discovery classes that should still see the class:

```php
use Pollora\Attributes\SkipDiscovery;
use Pollora\Hook\Infrastructure\Services\HookDiscovery;

#[SkipDiscovery(except: [HookDiscovery::class])]
class SpecializedHookHandler
{
    #[Action('init')]
    public function onInit(): void
    {
        // This hook WILL be discovered (HookDiscovery is in the except list)
    }
}
```

In this example, the class is skipped by `PostTypeDiscovery`, `ServiceProviderDiscovery`, `ScheduleDiscovery`, etc. — but `HookDiscovery` will still process it and register the `#[Action]` attribute.

You can list multiple discovery classes:

```php
#[SkipDiscovery(except: [HookDiscovery::class, ScheduleDiscovery::class])]
class MyClass
{
    // Only hooks and schedules will be discovered
}
```

### Config-Level Exclusions

For cases where you cannot modify the source code (third-party packages, vendor classes), use the `config/discovery.php` configuration file to exclude classes or paths globally.

Publish the config file:

```bash
php artisan vendor:publish --tag=pollora-config
```

Then configure the exclusions:

```php
// config/discovery.php
return [
    /*
     * Fully qualified class names to exclude from discovery.
     */
    'skip_classes' => [
        App\Legacy\OldController::class,
        SomeVendor\Package\InternalHelper::class,
    ],

    /*
     * Path patterns to exclude from discovery scanning.
     * Matched with str_contains() — use directory names or partial paths.
     */
    'skip_paths' => [
        '/Fixtures/',
        '/Tests/',
        '/stubs/',
    ],
];
```

Config-level exclusions are checked before any reflection is loaded, just like `#[SkipDiscovery]`.

### How It Works

The `#[SkipDiscovery]` attribute is detected during the Spatie token-parsing phase — before any reflection is loaded. This means:

- **Zero overhead** on classes without the attribute
- **No reflection cost** for fully skipped classes (`#[SkipDiscovery]` without `except`)
- Reflection is only loaded when `except` is used, to read the parameter values

Config-level exclusions (`skip_classes`, `skip_paths`) are also checked at the same early stage — no reflection is triggered for excluded classes.

## API Usage

### Using the Discovery Manager

```php
use Pollora\Discovery\Application\Services\DiscoveryManager;

// Get the discovery manager from container
$discoveryManager = app(DiscoveryManager::class);

// Add discovery locations
$discoveryManager->addLocation('MyTheme\\', get_stylesheet_directory() . '/app');

// Run discovery and apply all results
$discoveryManager->run();

// Or run discovery phases separately
$discoveryManager->discover();  // Discovery phase only
$discoveryManager->apply();     // Application phase only
```

### Getting Discovery Information

```php
// Check if a discovery is registered
if ($discoveryManager->hasDiscovery('post_types')) {
    $items = $discoveryManager->getDiscoveredItems('post_types');
    echo "Found " . count($items) . " post types\n";
}

// Get all registered discoveries
$discoveries = $discoveryManager->getDiscoveries();

// Get all discovery locations
$locations = $discoveryManager->getLocations();
```

### Cache Management

```php
// Clear discovery caches
$discoveryManager->clearCache();
```

## Creating Custom Discovery Classes

> **Extension author API:** The `Pollora\Discovery\Domain\Contracts\*` and `Pollora\Discovery\Domain\Services\*` namespaces are the **stable public contracts** for building custom discoveries. These types are intentionally exposed to extension authors and will follow semantic versioning guarantees.

### 1. Basic Discovery Class

```php
<?php

declare(strict_types=1);

namespace MyTheme\Discovery;

use MyTheme\Attributes\CustomComponent;
use Pollora\Discovery\Domain\Contracts\DiscoveryInterface;
use Pollora\Discovery\Domain\Contracts\DiscoveryLocationInterface;
use Pollora\Discovery\Domain\Services\IsDiscovery;
use Spatie\StructureDiscoverer\Data\DiscoveredStructure;

/**
 * Custom Component Discovery
 *
 * Discovers classes with the CustomComponent attribute
 */
final class CustomComponentDiscovery implements DiscoveryInterface
{
    use IsDiscovery;

    public function __construct(
        private readonly MyComponentService $componentService
    ) {}

    public function discover(DiscoveryLocationInterface $location, DiscoveredStructure $structure): void
    {
        // Only process classes
        if (!$structure instanceof \Spatie\StructureDiscoverer\Data\DiscoveredClass) {
            return;
        }

        // Check for our custom attribute
        $customAttribute = null;
        foreach ($structure->attributes as $attribute) {
            if ($attribute->name === CustomComponent::class) {
                $customAttribute = $attribute;
                break;
            }
        }

        if ($customAttribute === null || $structure->isAbstract) {
            return;
        }

        // Collect the discovered class
        $this->getItems()->add($location, [
            'class' => $structure->name,
            'attribute' => $customAttribute,
            'structure' => $structure,
        ]);
    }

    public function apply(): void
    {
        foreach ($this->getItems() as $discoveredItem) {
            [
                'class' => $className,
                'attribute' => $attribute,
                'structure' => $structure
            ] = $discoveredItem;

            try {
                // Register the component through your service
                $this->componentService->registerComponent($className);
            } catch (\Throwable $e) {
                error_log("Failed to register component {$className}: " . $e->getMessage());
            }
        }
    }

    public function getIdentifier(): string
    {
        return 'custom_components';
    }
}
```

### 2. Path-Aware Discovery Class

`DiscoversPathInterface` declares a `discoverPath()` method for discoveries that read files rather than classes. The engine does not call it yet: a discovery that needs files reads them from its location in `discover()` or `apply()`.

### 3. Registering Custom Discovery

Discovery classes are **automatically registered** by the framework. You only need to bind your discovery as a singleton in a service provider — the `DiscoveryRegistrar` will detect it and add it to the engine.

```php
<?php

namespace MyTheme\Providers;

use Illuminate\Support\ServiceProvider;
use MyTheme\Discovery\CustomComponentDiscovery;

class ThemeServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Just register as a singleton — auto-registration handles the rest
        $this->app->singleton(CustomComponentDiscovery::class);
    }
}
```

That's it. The `DiscoveryRegistrar` scans the container for all singletons whose class name ends with `Discovery` and that implement `DiscoveryInterface`. It registers them with the engine using the identifier returned by `getIdentifier()`.

> **How it works:** During the engine's `discover()` phase, the registrar iterates over container bindings, resolves any that look like discovery classes, and calls `addDiscovery()` automatically. No manual wiring needed.

## Discovery Engine

### Engine Configuration

```php
use Pollora\Discovery\Domain\Contracts\DiscoveryEngineInterface;
use Pollora\Discovery\Domain\Models\DiscoveryLocation;

/** @var DiscoveryEngineInterface $engine */
$engine = app(DiscoveryEngineInterface::class);

// Add discovery locations
$engine->addLocation(new DiscoveryLocation('App\\', app_path()));
$engine->addLocation(new DiscoveryLocation('MyTheme\\', get_stylesheet_directory() . '/app'));

// Discoveries are auto-registered from container singletons.
// Manual registration is no longer needed.

// Run discovery
$engine->run(); // Discovery + Apply
// or
$engine->discover(); // Discovery only
$engine->apply();    // Apply only
```

### Retrieving Discoveries

```php
// Get specific discovery
$postTypeDiscovery = $engine->getDiscovery('post_types');
$discoveredItems = $postTypeDiscovery->getItems()->all();

// Get all discoveries
$allDiscoveries = $engine->getDiscoveries();

// Get all locations
$locations = $engine->getLocations();
```

## Caching System

### What is cached

Discovery scans each location once per process and keeps the result in memory. Outside debug mode, scans are also stored in Laravel's cache, through Spatie's `LaravelDiscoverCacheDriver` under the `pollora` prefix, one entry per location. In debug mode (`APP_DEBUG=true`) the persistent cache is off and every request scans the disk; a location whose scan takes more than 250 ms is logged, as it usually holds `node_modules`, `vendor` or a build output.

To use another store, or another Spatie driver, set `structure-discoverer.cache` in `config/structure-discoverer.php`:

```php
return [
    'cache' => [
        'driver' => \Spatie\StructureDiscoverer\Cache\LaravelDiscoverCacheDriver::class,
        'store' => 'redis',
    ],
];
```

### Clearing the cache

A class added after the cache was filled is not discovered until the cache is cleared:

```bash
php artisan discovery:clear
```

`php artisan pollora:doctor` names the classes on disk that the cache lacks.

> `DiscoveryCacheInterface` and `LaravelDiscoveryCache` are deprecated and removed in 15.0: nothing reads that cache.

## Outside Laravel

`pollora/discovery` runs without Laravel's application: in a WordPress plugin, a Bedrock site or any PHP project.

```bash
composer require pollora/discovery
```

Every dependency of the engine is optional. Without them, it builds classes by reflection, keeps no persistent cache and runs only the discoveries added to it:

```php
use Pollora\Discovery\Domain\Models\DiscoveryLocation;
use Pollora\Discovery\Infrastructure\Cache\Psr16DiscoverCacheDriver;
use Pollora\Discovery\Infrastructure\Services\DiscoveryCacheManager;
use Pollora\Discovery\Infrastructure\Services\DiscoveryEngine;

$engine = new DiscoveryEngine(
    // Optional: cache scans in any PSR-16 cache (transients, object cache)
    cacheManager: new DiscoveryCacheManager(new Psr16DiscoverCacheDriver($psr16Cache)),
);

$engine
    ->addLocation(new DiscoveryLocation('MyPlugin\\', __DIR__.'/src'))
    ->addDiscovery('shortcodes', new ShortcodeDiscovery)
    ->run();
```

Two contracts let an environment plug in: `InstanceResolverInterface` builds the classes discovery needs (`ContainerInstanceResolver` wraps any PSR-11 container), and `DiscoveryRegistrarInterface` adds the environment's discoveries before each run, as the framework's `DiscoveryRegistrar` does with the container. See the [package README](https://github.com/Pollora/discovery#readme).

## Console Commands

### Run Discovery

```bash
# Run all discoveries
php artisan discovery:run

# Run specific discovery
php artisan discovery:run --discovery=post_types

# Clear cache before running
php artisan discovery:run --clear-cache

# Run with verbose output
php artisan discovery:run -v
```

### Clear Discovery Cache

```bash
# Clear all discovery caches
php artisan discovery:clear
```

## Configuration

### Service Provider Configuration

```php
// In a service provider — just register discoveries as singletons
public function register(): void
{
    $this->app->singleton(MyComponentDiscovery::class);
    $this->app->singleton(MyServiceDiscovery::class);
}

// The DiscoveryRegistrar auto-detects them. No boot() wiring needed.
```

## Usage Examples

### Example 1: Theme Component Discovery

```php
<?php

namespace MyTheme\Discovery;

use MyTheme\Attributes\Component;
use Pollora\Discovery\Domain\Contracts\DiscoveryInterface;
use Pollora\Discovery\Domain\Contracts\DiscoveryLocationInterface;
use Pollora\Discovery\Domain\Services\IsDiscovery;
use Spatie\StructureDiscoverer\Data\DiscoveredStructure;

final class ComponentDiscovery implements DiscoveryInterface
{
    use IsDiscovery;

    public function discover(DiscoveryLocationInterface $location, DiscoveredStructure $structure): void
    {
        if (!$structure instanceof \Spatie\StructureDiscoverer\Data\DiscoveredClass) {
            return;
        }

        foreach ($structure->attributes as $attribute) {
            if ($attribute->name === Component::class) {
                $this->getItems()->add($location, [
                    'class' => $structure->name,
                    'attribute' => $attribute,
                ]);
                break;
            }
        }
    }

    public function apply(): void
    {
        foreach ($this->getItems() as $item) {
            $this->registerComponent($item['class'], $item['attribute']);
        }
    }

    public function getIdentifier(): string
    {
        return 'theme_components';
    }

    private function registerComponent(string $className, $attribute): void
    {
        // Component registration logic
        add_action('wp_enqueue_scripts', function () use ($className) {
            $component = app($className);
            $component->register();
        });
    }
}
```

### Example 2: Plugin Service Discovery

```php
<?php

namespace MyPlugin\Discovery;

use MyPlugin\Contracts\PluginService;
use Pollora\Discovery\Domain\Contracts\DiscoveryInterface;
use Pollora\Discovery\Domain\Contracts\DiscoveryLocationInterface;
use Pollora\Discovery\Domain\Services\IsDiscovery;
use Spatie\StructureDiscoverer\Data\DiscoveredStructure;

final class ServiceDiscovery implements DiscoveryInterface
{
    use IsDiscovery;

    public function discover(DiscoveryLocationInterface $location, DiscoveredStructure $structure): void
    {
        if (!$structure instanceof \Spatie\StructureDiscoverer\Data\DiscoveredClass) {
            return;
        }

        // Check if class implements PluginService
        if (in_array(PluginService::class, $structure->implements)) {
            $this->getItems()->add($location, ['class' => $structure->name]);
        }
    }

    public function apply(): void
    {
        foreach ($this->getItems() as $item) {
            $serviceClass = $item['class'];
            
            // Register as singleton in container
            app()->singleton($serviceClass);
            
            // Auto-bind to interface if it exists
            $interfaces = class_implements($serviceClass);
            foreach ($interfaces as $interface) {
                if ($interface !== PluginService::class) {
                    app()->bind($interface, $serviceClass);
                }
            }
        }
    }

    public function getIdentifier(): string
    {
        return 'plugin_services';
    }
}
```

## Migration from Old Discoverer

### Key Changes

1. **Namespace**: `Pollora\Discoverer` → `Pollora\Discovery`
2. **API**: `PolloraDiscover::scout()` → `DiscoveryManager::run()`
3. **Architecture**: Scout-based → Discovery-based
4. **Placement**: Central package → Module-specific discoveries

### Migration Steps

1. **Update Service Provider Registration**:
```php
// Old
$this->app->register(DiscovererServiceProvider::class);

// New
$this->app->register(DiscoveryServiceProvider::class);
```

2. **Convert Scout Classes to Discovery Classes**:
```php
// Old Scout
class MyScout extends AbstractPolloraScout
{
    protected function criteria(Discover $discover): Discover
    {
        return $discover->classes()->implementing(MyInterface::class);
    }
}

// New Discovery
class MyDiscovery implements DiscoveryInterface
{
    use IsDiscovery;

    public function discover(DiscoveryLocationInterface $location, DiscoveredStructure $structure): void
    {
        if ($structure instanceof DiscoveredClass) {
            if (in_array(MyInterface::class, $structure->implements)) {
                $this->getItems()->add($location, ['class' => $structure->name]);
            }
        }
    }

    public function apply(): void
    {
        foreach ($this->getItems() as $item) {
            // Register discovered class
        }
    }

    public function getIdentifier(): string
    {
        return 'my_discovery';
    }
}
```

3. **Update Usage**:
```php
// Old
PolloraDiscover::register('my_scout', MyScout::class);
$classes = PolloraDiscover::scout('my_scout');

// New — just register as singleton, auto-discovered
$this->app->singleton(MyDiscovery::class);
```

## Troubleshooting

### Common Issues

#### 1. Discovery Not Found

```
DiscoveryNotFoundException: Discovery not found with identifier: my_discovery
```

**Solution**: Ensure the discovery is registered before use:

```php
/** @var DiscoveryManager $manager */
$manager = app(DiscoveryManager::class);

if (!$manager->hasDiscovery('my_discovery')) {
    $manager->addDiscovery('my_discovery', MyDiscovery::class);
}
```

#### 2. Components Not Discovered

**Possible causes**:
- Discovery locations not configured
- Discovery class not properly implementing interface
- Attribute/interface not found

**Debug**:

```php
// Enable verbose console output
php artisan discovery:run -v

// Check discovery locations
$manager = app(DiscoveryManager::class);
$locations = $manager->getLocations();
dump($locations);

// Check discovered items
$items = $manager->getDiscoveredItems('my_discovery');
dump($items);
```

#### 3. Cache Issues

**Solution**: Clear discovery cache:

```bash
# Via console
php artisan discovery:clear

# Via code
app(DiscoveryManager::class)->clearCache();
```

#### 4. Performance Issues

**Solution**: Optimize discovery locations:

```php
// Be specific about discovery paths
$manager->addLocation('App\\Services\\', app_path('Services'));
$manager->addLocation('App\\Models\\', app_path('Models'));

// Instead of scanning entire app directory
// $manager->addLocation('App\\', app_path());
```

### Debug Mode

Enable debug logging during development:

```php
try {
    $manager = app(DiscoveryManager::class);
    $manager->run();
    
    $discoveries = $manager->getDiscoveries();
    foreach ($discoveries as $identifier => $discovery) {
        $count = count($manager->getDiscoveredItems($identifier));
        error_log("Discovery '{$identifier}' found {$count} items");
    }
} catch (\Throwable $e) {
    error_log('Discovery error: ' . $e->getMessage());
    error_log('Stack trace: ' . $e->getTraceAsString());
}
```

### Performance Monitoring

```php
$start = microtime(true);

$manager = app(DiscoveryManager::class);
$manager->run();

$duration = (microtime(true) - $start) * 1000;
error_log("Discovery completed in {$duration}ms");
```

## See also

- [WordPress hooks with PHP 8 attributes](https://pollora.dev/guides/wordpress-hooks-php-attributes/)
- [Custom post types and taxonomies with PHP attributes](https://pollora.dev/guides/custom-post-types-php-attributes/)
