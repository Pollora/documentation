# Discoverer System

The Pollora Discoverer system enables automatic class discovery in your application, WordPress themes, and Laravel modules. It uses a "scout"-based architecture to identify and process different types of classes according to your needs.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Built-in Scouts](#built-in-scouts)
- [Simple API](#simple-api)
- [Creating a Custom Scout](#creating-a-custom-scout)
- [Registering a Scout from a Plugin](#registering-a-scout-from-a-plugin)
- [Path Management](#path-management)
- [Caching and Performance](#caching-and-performance)
- [Usage Examples](#usage-examples)
- [Troubleshooting](#troubleshooting)

## Overview

The Discoverer system replaces the old discovery system with a simplified approach based on a central registry. It allows you to:

- **Automatically discover** classes in the application, themes, and modules
- **Process different types** of classes with specialized scouts
- **Easily extend** the system with custom scouts
- **Optimize performance** with caching and parallel processing
- **Easily integrate** from plugins or modules

## Architecture

### Main Components

```
┌──────────────────────────────────────────────────────────┐
│                    PolloraDiscover (API)                 │
│                      Simplified facade                   │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                       DiscoveryService                   │
│                     Coordination service                 │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                         ScoutRegistry                    │
│                        Scout registry                    │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                    AbstractPolloraScout                  │
│                   Base class for scouts                  │
└──────────────────────────┬───────────────────────────────┘
                           │
      ┌────────────────────┼─────────────────────┐
      │                    │                     │
┌─────▼─────────┐  ┌───────▼──────┐  ┌───────────▼─────────┐
│ Attributables │  │ Hook Classes │  │ PostType & Taxonomy │
│ Classes Scout │  │     Scout    │  │         Scout       │
└───────────────┘  └──────────────┘  └─────────────────────┘
```

### Discovery Flow

1. **Registration**: Scouts are registered with a unique key
2. **Execution**: A scout is called to discover classes
3. **Processing**: Discovered classes are processed by appropriate services

## Built-in Scouts

The framework provides several ready-to-use scouts:

### `attributable` - Attributable Classes
Discovers all classes implementing the `Attributable` interface.

```php
// Usage
$attributableClasses = PolloraDiscover::scout('attributable');

// Found classes
foreach ($attributableClasses as $className) {
    $instance = app($className);
    // Process PHP 8 attributes
}
```

### `hooks` - WordPress Actions and Filters
Discovers classes containing methods with `#[Action]` or `#[Filter]` attributes.

```php
$hookClasses = PolloraDiscover::scout('hooks');
```

### `post_types` - Custom Post Types
Discovers classes with the `#[PostType]` attribute.

```php
$postTypeClasses = PolloraDiscover::scout('post_types');
```

### `taxonomies` - Custom Taxonomies
Discovers classes with the `#[Taxonomy]` attribute.

```php
$taxonomyClasses = PolloraDiscover::scout('taxonomies');
```

### `wp_rest_routes` - WordPress REST API Routes
Discovers classes extending `AbstractWpRestRoute`.

```php
$restRouteClasses = PolloraDiscover::scout('wp_rest_routes');
```

## Simple API

The `PolloraDiscover` API provides a simple interface to interact with the system:

### Register a Scout

```php
use Pollora\Discoverer\Framework\API\PolloraDiscover;

PolloraDiscover::register('my_scout', MyCustomScout::class);
```

### Execute Discovery

```php
$discoveredClasses = PolloraDiscover::scout('my_scout');

// The result is a Laravel Collection
foreach ($discoveredClasses as $className) {
    // Process each discovered class
    echo "Found class: {$className}\n";
}
```

### Check Scout Existence

```php
if (PolloraDiscover::has('my_scout')) {
    $classes = PolloraDiscover::scout('my_scout');
}
```

### List All Scouts

```php
$registeredScouts = PolloraDiscover::registered();
// ['attributable', 'hooks', 'post_types', 'taxonomies', 'wp_rest_routes', 'my_scout']
```

## Creating a Custom Scout

### 1. Create the Scout Class

```php
<?php

namespace MyPlugin\Discovery\Scouts;

use MyPlugin\Contracts\MyInterface;
use Pollora\Discoverer\Infrastructure\Services\AbstractPolloraScout;
use Spatie\StructureDiscoverer\Discover;

/**
 * Scout for discovering classes implementing MyInterface.
 */
final class MyInterfaceScout extends AbstractPolloraScout
{
    /**
     * Define discovery criteria.
     */
    protected function criteria(Discover $discover): Discover
    {
        return $discover
            ->classes()
            ->implementing(MyInterface::class);
    }

    /**
     * Optional: Customize search directories.
     */
    protected function getDefaultDirectories(): array
    {
        $paths = parent::getDefaultDirectories();
        
        // Add specific paths
        $paths[] = '/custom/path/to/scan';
        
        return array_unique(array_filter($paths));
    }
}
```

### 2. Register the Scout

In a service provider:

```php
<?php

namespace MyPlugin\Providers;

use Illuminate\Support\ServiceProvider;
use MyPlugin\Discovery\Scouts\MyInterfaceScout;
use Pollora\Discoverer\Framework\API\PolloraDiscover;

class MyPluginServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Register the custom scout
        PolloraDiscover::register('my_interface', MyInterfaceScout::class);
        
        // Use the scout to discover and process classes
        $this->processMyInterface();
    }
    
    private function processMyInterface(): void
    {
        try {
            $classes = PolloraDiscover::scout('my_interface');
            
            foreach ($classes as $className) {
                $instance = $this->app->make($className);
                // Process each instance
                $this->registerMyInterface($instance);
            }
        } catch (\Throwable $e) {
            // Graceful error handling
            error_log("Discovery error: " . $e->getMessage());
        }
    }
}
```

### 3. Discovery Criteria Examples

#### Discover by PHP 8 Attribute

```php
protected function criteria(Discover $discover): Discover
{
    return $discover
        ->classes()
        ->withAttribute(MyAttribute::class);
}
```

#### Discover by Class Name

```php
protected function criteria(Discover $discover): Discover
{
    return $discover
        ->classes()
        ->suffixed('Handler'); // Classes ending with "Handler"
}
```

#### Discover by Namespace

```php
protected function criteria(Discover $discover): Discover
{
    return $discover
        ->classes()
        ->inNamespace('MyPlugin\\Services');
}
```

#### Combined Criteria

```php
protected function criteria(Discover $discover): Discover
{
    return $discover
        ->classes()
        ->implementing(MyInterface::class)
        ->withAttribute(MyAttribute::class)
        ->inNamespace('MyPlugin\\Handlers');
}
```

## Registering a Scout from a Plugin

### Method 1: Service Provider (Recommended)

```php
<?php

namespace MyPlugin\Providers;

use Illuminate\Support\ServiceProvider;
use MyPlugin\Discovery\Scouts\CustomScout;
use Pollora\Discoverer\Framework\API\PolloraDiscover;

class DiscoveryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Register the scout in the container
        $this->app->singleton(CustomScout::class, function ($app) {
            return new CustomScout($app);
        });
    }

    public function boot(): void
    {
        // Register the scout with the discovery system
        PolloraDiscover::register('my_custom_scout', CustomScout::class);
        
        // Process discovered classes
        $this->handleDiscoveredClasses();
    }
    
    private function handleDiscoveredClasses(): void
    {
        add_action('init', function () {
            try {
                $classes = PolloraDiscover::scout('my_custom_scout');
                
                foreach ($classes as $className) {
                    // Specific processing
                    $this->processClass($className);
                }
            } catch (\Exception $e) {
                if (function_exists('error_log')) {
                    error_log("Discovery error: " . $e->getMessage());
                }
            }
        });
    }
}
```

### Method 2: WordPress Hook

```php
// In your plugin's main file
add_action('init', function () {
    if (class_exists('Pollora\Discoverer\Framework\API\PolloraDiscover')) {
        PolloraDiscover::register('my_scout', MyScout::class);
        
        // Immediate usage
        $classes = PolloraDiscover::scout('my_scout');
        foreach ($classes as $class) {
            // Processing
        }
    }
});
```

### Method 3: Laravel Module

```php
// In your module's service provider
public function boot(): void
{
    PolloraDiscover::register('module_scout', ModuleScout::class);
}
```

## Path Management

### Default Paths

`AbstractPolloraScout` automatically scans:

- **Laravel Application**: `app_path()`
- **Laravel Modules**: Enabled module directories
- **WordPress Themes**: Active theme and parent theme (lazy loading)
- **WordPress Plugins**: `WP_PLUGIN_DIR` (optional depending on scout)

### Customize Paths

```php
class CustomPathScout extends AbstractPolloraScout
{
    protected function getDefaultDirectories(): array
    {
        return [
            '/custom/path/1',
            '/custom/path/2',
            get_stylesheet_directory() . '/custom',
            ...parent::getDefaultDirectories(), // Include default paths
        ];
    }
}
```

### Construction with Custom Paths

```php
// In a service provider
$this->app->singleton(CustomScout::class, function ($app) {
    return new CustomScout($app, [
        '/specific/directory',
        '/another/path'
    ]);
});
```

## Caching and Performance

### Automatic Caching

The system automatically uses caching via Spatie Structure Discoverer:

- **Cache enabled**: In production mode (`WP_DEBUG = false`)
- **Cache disabled**: In development mode for automatic recompilation
- **Invalidation**: Automatic based on directory hash

### Performance

- **Parallel processing**: 100 processes by default
- **Smart filtering**: Only existing and readable directories are scanned
- **Lazy loading**: WordPress paths are loaded only when necessary

### Cache Configuration

```php
class OptimizedScout extends AbstractPolloraScout
{
    protected function definition(): Discover
    {
        $discover = parent::definition();
        
        // Customize parallelism level
        return $discover->parallel(50);
    }
}
```

## Usage Examples

### Example 1: Scout for Event Handlers

```php
<?php

namespace MyApp\Discovery;

use MyApp\Contracts\EventHandler;
use Pollora\Discoverer\Infrastructure\Services\AbstractPolloraScout;
use Spatie\StructureDiscoverer\Discover;

final class EventHandlerScout extends AbstractPolloraScout
{
    protected function criteria(Discover $discover): Discover
    {
        return $discover
            ->classes()
            ->implementing(EventHandler::class)
            ->inNamespace('MyApp\\EventHandlers');
    }
}

// Registration and usage
PolloraDiscover::register('event_handlers', EventHandlerScout::class);

$handlers = PolloraDiscover::scout('event_handlers');
foreach ($handlers as $handlerClass) {
    $handler = app($handlerClass);
    $eventBus->register($handler);
}
```

### Example 2: Scout for Artisan Commands

```php
<?php

namespace MyPlugin\Discovery;

use Illuminate\Console\Command;
use Pollora\Discoverer\Infrastructure\Services\AbstractPolloraScout;
use Spatie\StructureDiscoverer\Discover;

final class CommandScout extends AbstractPolloraScout
{
    protected function criteria(Discover $discover): Discover
    {
        return $discover
            ->classes()
            ->extending(Command::class)
            ->suffixed('Command');
    }

    protected function getDefaultDirectories(): array
    {
        return [
            base_path('app/Console/Commands'),
            ...parent::getDefaultDirectories(),
        ];
    }
}

// Service Provider
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        PolloraDiscover::register('console_commands', CommandScout::class);
        
        $commands = PolloraDiscover::scout('console_commands');
        $this->commands($commands->toArray());
    }
}
```

### Example 3: Scout for Services with Attributes

```php
<?php

namespace MyApp\Discovery;

use MyApp\Attributes\AutoWire;
use Pollora\Discoverer\Infrastructure\Services\AbstractPolloraScout;
use Spatie\StructureDiscoverer\Discover;

final class AutoWireScout extends AbstractPolloraScout
{
    protected function criteria(Discover $discover): Discover
    {
        return $discover
            ->classes()
            ->withAttribute(AutoWire::class);
    }
}

// Service Provider
public function register(): void
{
    PolloraDiscover::register('auto_wire', AutoWireScout::class);
}

public function boot(): void
{
    $services = PolloraDiscover::scout('auto_wire');
    
    foreach ($services as $serviceClass) {
        $reflection = new ReflectionClass($serviceClass);
        $attribute = $reflection->getAttributes(AutoWire::class)[0] ?? null;
        
        if ($attribute) {
            $config = $attribute->newInstance();
            $this->app->singleton($config->interface, $serviceClass);
        }
    }
}
```

## Troubleshooting

### Common Issues

#### 1. Scout Not Found

```
InvalidArgumentException: Scout 'my_scout' not found
```

**Solution**: Verify the scout is registered before use.

```php
// Check before usage
if (!PolloraDiscover::has('my_scout')) {
    PolloraDiscover::register('my_scout', MyScout::class);
}
```

#### 2. Classes Not Discovered

**Possible causes**:
- Incorrect search paths
- Too restrictive discovery criteria
- Autoloader not configured

**Debug**:

```php
class DebugScout extends AbstractPolloraScout
{
    protected function getValidDirectories(): array
    {
        $directories = parent::getValidDirectories();
        error_log('Scanned directories: ' . json_encode($directories));
        return $directories;
    }
    
    protected function criteria(Discover $discover): Discover
    {
        $result = $discover->classes()->implementing(MyInterface::class);
        error_log('Applied criteria: implementing MyInterface');
        return $result;
    }
}
```

#### 3. Performance Issues

**Solution**: Limit scanned directories

```php
protected function getDefaultDirectories(): array
{
    // Be specific about paths
    return [
        app_path('Services'),
        app_path('Handlers'),
        // Avoid scanning the entire project
    ];
}
```

#### 4. Cache Issues

**Solution**: Force recompilation in debug mode

```php
// In config or .env
WP_DEBUG=true  // Disables cache
```

### Logging and Debug

```php
// Enable logging for development
try {
    $classes = PolloraDiscover::scout('my_scout');
    error_log(sprintf('Scout %s found %d classes', 'my_scout', $classes->count()));
} catch (\Exception $e) {
    error_log('Discovery error: ' . $e->getMessage());
    error_log('Stack trace: ' . $e->getTraceAsString());
}
```