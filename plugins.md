# Plugin Development with Pollora

Pollora provides a comprehensive plugin development system that bridges WordPress plugin development with Laravel's modern architecture patterns. This documentation covers everything you need to know about creating, managing, and deploying plugins using the Pollora framework.

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Plugin Architecture](#plugin-architecture)
- [Creating a Plugin](#creating-a-plugin)
- [Plugin Structure](#plugin-structure)
- [Service Providers](#service-providers)
- [Attribute-Based Hooks](#attribute-based-hooks)
- [Autoloading](#autoloading)
- [Plugin Management](#plugin-management)
- [Configuration](#configuration)
- [Assets & Vite Integration](#assets--vite-integration)
- [Views](#views)
- [Frontend Development](#frontend-development)
- [Translations](#translations)
- [Testing](#testing)
- [Deployment](#deployment)
- [Best Practices](#best-practices)

## Overview

The Pollora plugin system extends WordPress plugin development by providing:

- **Laravel-style Architecture**: Service providers, dependency injection, and modern PHP patterns
- **PSR-4 Autoloading**: Automatic class loading with fixed namespace conventions `Plugin\{PluginName}\`
- **Attribute-driven Configuration**: Use PHP 8 attributes for declarative hook registration
- **Command-line Tools**: Artisan commands for plugin scaffolding and management
- **Modern Asset Management**: Vite integration with hot reload and Tailwind CSS support
- **Automatic Module Discovery**: Views, routes, translations automatically handled
- **Comprehensive Testing**: Built-in testing support with PHPUnit

## Quick Start

### Creating Your First Plugin

Generate a new plugin using the Artisan command:

```bash
# Create a plugin with modern asset management (recommended for frontend-heavy plugins)
php artisan pollora:make-plugin my-awesome-plugin \
    --plugin-author="John Doe" \
    --plugin-author-uri="https://johndoe.com" \
    --plugin-uri="https://github.com/johndoe/my-awesome-plugin" \
    --plugin-description="An awesome plugin built with Pollora" \
    --plugin-version="1.0.0" \
    --asset=true

# Or create a minimal plugin without assets (good for backend-focused plugins)
php artisan pollora:make-plugin my-awesome-plugin \
    --plugin-author="John Doe" \
    --plugin-author-uri="https://johndoe.com" \
    --plugin-uri="https://github.com/johndoe/my-awesome-plugin" \
    --plugin-description="An awesome plugin built with Pollora" \
    --plugin-version="1.0.0" \
    --asset=false
```

This creates a complete plugin structure with:
- Main plugin file with WordPress headers
- Service provider for dependency injection
- Configuration files
- View templates with Blade support
- Asset directories with Tailwind CSS (if `--asset=true` is used)
- JavaScript and CSS files with modern tooling (if `--asset=true` is used)
- Package.json with development dependencies (if `--asset=true` is used)
- Vite configuration for asset compilation (if `--asset=true` is used)

### Plugin Registration

Plugins are automatically discovered and registered when they contain a valid main plugin file. The registration is handled by the Pollora PluginRegistrar service:

```php
<?php
/**
 * Plugin Name: My Awesome Plugin
 * Plugin URI: https://github.com/johndoe/my-awesome-plugin
 * Description: An awesome plugin built with Pollora
 * Version: 1.0.0
 * Author: John Doe
 * Author URI: https://johndoe.com
 * Text Domain: my-awesome-plugin
 * Domain Path: /languages
 * Requires at least: 5.0
 * Tested up to: 6.3
 * Requires PHP: 8.1
 */

declare(strict_types=1);

// Prevent direct access
if (!defined('ABSPATH')) {
    exit;
}

// Define plugin constants
define('MY_AWESOME_PLUGIN_VERSION', '1.0.0');
define('MY_AWESOME_PLUGIN_PLUGIN_FILE', __FILE__);
define('MY_AWESOME_PLUGIN_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('MY_AWESOME_PLUGIN_PLUGIN_URL', plugin_dir_url(__FILE__));

// Register with Pollora framework
if (class_exists('Pollora\\Plugin\\Application\\Services\\PluginRegistrar')) {
    $registrar = app('Pollora\\Plugin\\Application\\Services\\PluginRegistrar');
    $registrar->register('my-awesome-plugin', __DIR__);
}

/**
 * Initialize the plugin.
 */
function my_awesome_plugin_init(): void
{
    // Load plugin textdomain for translations
    load_plugin_textdomain(
        'my-awesome-plugin',
        false,
        dirname(plugin_basename(__FILE__)) . '/languages'
    );

    // Initialize plugin functionality
    if (class_exists('Plugin\\MyAwesomePlugin\\MyAwesomePluginPlugin')) {
        $plugin = new Plugin\MyAwesomePlugin\MyAwesomePluginPlugin();
        
        // Register activation/deactivation hooks through the plugin class
        register_activation_hook(__FILE__, [$plugin, 'activate']);
        register_deactivation_hook(__FILE__, [$plugin, 'deactivate']);
        register_uninstall_hook(__FILE__, [Plugin\MyAwesomePlugin\MyAwesomePluginPlugin::class, 'uninstall']);
    }
}
add_action('plugins_loaded', 'my_awesome_plugin_init');
```

## Plugin Architecture

### Domain-Driven Design

Pollora plugins follow a strict DDD architecture with automatic module discovery:

```
public/content/plugins/my-awesome-plugin/
├── app/                          # PSR-4 autoloaded application code
│   ├── Providers/               # Service providers (auto-discovered)
│   │   ├── AssetServiceProvider.php
│   │   └── PluginServiceProvider.php
│   └── MyAwesomePluginPlugin.php # Main plugin class with attributes
├── resources/                   # Laravel-style resources
│   ├── assets/                 # Vite-managed assets
│   │   ├── app.js             # Main JavaScript file
│   │   ├── admin.js           # Admin JavaScript
│   │   └── app.css            # Main CSS with Tailwind
│   └── views/                  # Blade templates (auto-discovered)
│       └── welcome.blade.php
├── config/                      # Configuration files
│   └── plugin.php              # Plugin configuration
├── languages/                   # Translation files (auto-discovered)
├── routes/                      # Route definitions (auto-discovered)
│   ├── web.php
│   ├── api.php
│   └── admin.php
├── database/                    # Database migrations (auto-discovered)
├── tests/                       # Plugin tests
├── vite.config.js              # Vite configuration
├── tailwind.config.js          # Tailwind CSS configuration
├── postcss.config.mjs          # PostCSS configuration
├── package.json                # NPM dependencies
└── my-awesome-plugin.php        # Main plugin file
```

### Namespace Conventions

Plugins use the fixed namespace convention: `Plugin\{PluginName}\`

For a plugin named "my-awesome-plugin", the namespace would be:
```php
namespace Plugin\MyAwesomePlugin;
```

## Creating a Plugin

### Using the Make Command

The `pollora:make-plugin` command provides several options:

```bash
# Basic plugin creation
php artisan pollora:make-plugin my-plugin \
    --plugin-author="Author" \
    --plugin-author-uri="https://author.com" \
    --plugin-uri="https://plugin.com" \
    --plugin-description="Description" \
    --plugin-version="1.0.0"

# Create plugin with assets (JS/CSS with ViteJS compilation)
php artisan pollora:make-plugin my-plugin \
    --plugin-author="Author" \
    --plugin-author-uri="https://author.com" \
    --plugin-uri="https://plugin.com" \
    --plugin-description="Description" \
    --plugin-version="1.0.0" \
    --asset=true

# Create plugin without assets (minimal structure)
php artisan pollora:make-plugin my-plugin \
    --plugin-author="Author" \
    --plugin-author-uri="https://author.com" \
    --plugin-uri="https://plugin.com" \
    --plugin-description="Description" \
    --plugin-version="1.0.0" \
    --asset=false

# Create from GitHub repository
php artisan pollora:make-plugin my-plugin \
    --plugin-author="Author" \
    --plugin-author-uri="https://author.com" \
    --plugin-uri="https://plugin.com" \
    --plugin-description="Description" \
    --plugin-version="1.0.0" \
    --repository=owner/repo

# Force overwrite existing plugin
php artisan pollora:make-plugin my-plugin \
    --plugin-author="Author" \
    --plugin-author-uri="https://author.com" \
    --plugin-uri="https://plugin.com" \
    --plugin-description="Description" \
    --plugin-version="1.0.0" \
    --force
```

### Command Options

- `--plugin-author` : Plugin author name
- `--plugin-author-uri` : Plugin author URI
- `--plugin-uri` : Plugin URI
- `--plugin-description` : Plugin description
- `--plugin-version` : Plugin version
- `--repository` : GitHub repository to download (owner/repo format)
- `--repo-version` : Specific version/tag to download
- `--asset=true/false` : Include asset files (JS/CSS) with ViteJS compilation (default: false)
- `--force` : Force create plugin with same name

### Asset Management

When creating a plugin with the `--asset=true` option, the following files are included:
- `vite.config.js` - Vite configuration for asset compilation
- `tailwind.config.js` - Tailwind CSS configuration
- `postcss.config.mjs` - PostCSS configuration
- `app/Providers/AssetServiceProvider.php` - Asset service provider
- `resources/assets/` - Directory containing CSS and JS files

After plugin creation with assets, install and build them:

```bash
cd public/content/plugins/my-plugin
npm install
npm run dev    # For development with hot reload
npm run build  # For production build
```

> **Note**: If you create a plugin with `--asset=false` or without the `--asset` option, these files will be excluded and no npm commands will be run during plugin creation.

## Plugin Structure

### Plugin Types

Pollora supports two types of plugin structures depending on your needs:

#### With Assets (`--asset=true`)
For plugins that require modern frontend development with JavaScript, CSS, and build tools:

```
public/content/plugins/my-awesome-plugin/
├── app/                          # PSR-4 autoloaded application code
│   ├── Providers/               # Service providers (auto-discovered)
│   │   ├── AssetServiceProvider.php  # Asset management
│   │   └── PluginServiceProvider.php # Plugin services
│   └── MyAwesomePluginPlugin.php # Main plugin class
├── resources/                   # Laravel-style resources
│   ├── assets/                 # Vite-managed assets
│   │   ├── app.js             # Main JavaScript file
│   │   ├── admin.js           # Admin JavaScript
│   │   └── app.css            # Main CSS with Tailwind
│   └── views/                  # Blade templates
├── vite.config.js              # Vite configuration
├── tailwind.config.js          # Tailwind CSS configuration
├── postcss.config.mjs          # PostCSS configuration
├── package.json                # NPM dependencies
└── my-awesome-plugin.php        # Main plugin file
```

#### Without Assets (`--asset=false` or default)
For plugins that focus on backend functionality without complex frontend requirements:

```
public/content/plugins/my-awesome-plugin/
├── app/                          # PSR-4 autoloaded application code
│   ├── Providers/               # Service providers (auto-discovered)
│   │   └── PluginServiceProvider.php # Plugin services only
│   └── MyAwesomePluginPlugin.php # Main plugin class
├── resources/                   # Laravel-style resources
│   └── views/                  # Blade templates only
├── config/                      # Configuration files
│   └── plugin.php              # Plugin configuration
└── my-awesome-plugin.php        # Main plugin file
```

### Main Plugin Class with Attributes

The main plugin class uses PHP 8 attributes for clean hook registration:

```php
<?php

declare(strict_types=1);

namespace Plugin\MyAwesomePlugin;

use Pollora\Attributes\Action;
use Pollora\Hook\Domain\Contracts\Hooks;

/**
 * Main plugin class for My Awesome Plugin.
 *
 * Uses Pollora's attribute-based hook system for cleaner code organization.
 */
class MyAwesomePluginPlugin implements Hooks
{
    protected string $version = '1.0.0';
    protected string $slug = 'my-awesome-plugin';

    public function __construct()
    {
        // Plugin initialization happens through attribute-based hooks
        // Service providers, routes, migrations, etc. are automatically handled by the Modules system
    }

    /**
     * Plugin activation callback.
     */
    public function activate(): void
    {
        // Plugin activation logic
        flush_rewrite_rules();
    }

    /**
     * Plugin deactivation callback.
     */
    public function deactivate(): void
    {
        // Plugin deactivation logic
        flush_rewrite_rules();
    }

    /**
     * Plugin uninstall callback.
     */
    public static function uninstall(): void
    {
        // Plugin uninstall logic
        // - Remove database tables
        // - Delete plugin options
        // - Clean up all plugin data
    }

    /**
     * WordPress init hook callback.
     */
    #[Action('init', priority: 10)]
    public function onInit(): void
    {
        // Initialize plugin functionality
        load_plugin_textdomain(
            $this->slug,
            false,
            dirname(plugin_basename(MY_AWESOME_PLUGIN_PLUGIN_FILE)) . '/languages'
        );
    }

    /**
     * Example admin initialization.
     */
    #[Action('admin_init', priority: 10)]
    public function onAdminInit(): void
    {
        // Initialize admin-specific functionality
    }

    /**
     * Example admin menu setup.
     */
    #[Action('admin_menu', priority: 10)]
    public function addAdminMenu(): void
    {
        // Add admin menu items
        // add_options_page(
        //     'My Awesome Plugin Settings',
        //     'My Plugin',
        //     'manage_options',
        //     $this->slug,
        //     [$this, 'renderSettingsPage']
        // );
    }
}
```

## Service Providers

### Plugin Service Provider

Service providers are automatically discovered and registered by the Modules system:

```php
<?php

declare(strict_types=1);

namespace Plugin\MyAwesomePlugin\Providers;

use Illuminate\Support\ServiceProvider;

/**
 * Plugin service provider for My Awesome Plugin.
 *
 * This service provider is automatically discovered and registered by the Pollora Modules system.
 * Views, routes, translations, migrations, etc. are automatically handled by the framework.
 * Use this provider only for custom service registrations specific to your plugin.
 */
class PluginServiceProvider extends ServiceProvider
{
    /**
     * Register plugin services.
     */
    public function register(): void
    {
        // Register your custom plugin services here
        // Example:
        // $this->app->singleton(CustomService::class, function ($app) {
        //     return new CustomService();
        // });
    }

    /**
     * Bootstrap plugin services.
     */
    public function boot(): void
    {
        // Add your custom bootstrap logic here
        // Note: Views, routes, translations, assets are automatically handled by the Modules system
        
        // Example of custom boot logic:
        // $this->registerCustomValidationRules();
        // $this->extendExistingServices();
    }
}
```

### Asset Service Provider

Assets are managed through a dedicated service provider with Vite integration:

```php
<?php

declare(strict_types=1);

namespace Plugin\MyAwesomePlugin\Providers;

use Illuminate\Support\ServiceProvider;
use Pollora\Support\Facades\Asset;

/**
 * Asset service provider for My Awesome Plugin.
 *
 * Handles asset registration and enqueuing for the plugin.
 * Uses Vite for modern asset management with hot reload support.
 */
class AssetServiceProvider extends ServiceProvider
{
    protected array $config;

    public function register(): void
    {
        // Load plugin configuration
        $configPath = __DIR__ . '/../../config/plugin.php';
        $this->config = file_exists($configPath) ? require $configPath : [];
    }

    public function boot(): void
    {
        $this->registerPluginAssets();
    }

    protected function registerPluginAssets(): void
    {
        $containerName = $this->getContainerName();
        $slug = $this->config['slug'] ?? 'my-awesome-plugin';

        // Admin assets
        Asset::add("{$slug}/admin", 'admin.js')
            ->container($containerName)
            ->toBackend()
            ->useVite();

        // Plugin styles
        Asset::add("{$slug}/styles", 'app.css')
            ->container($containerName)
            ->toFrontend()
            ->useVite();
    }

    protected function getContainerName(): string
    {
        $slug = $this->config['slug'] ?? 'my-awesome-plugin';
        return "plugin.{$slug}";
    }
}
```

## Attribute-Based Hooks

### Using Action Attributes

Instead of manual `add_action()` calls, use PHP 8 attributes:

```php
use Pollora\Attributes\Action;
use Pollora\Hook\Domain\Contracts\Hooks;

class MyPluginClass implements Hooks
{
    #[Action('init', priority: 10)]
    public function onInit(): void
    {
        // Code to execute for the 'init' WordPress action
    }

    #[Action('wp_enqueue_scripts', priority: 10)]
    public function enqueueScripts(): void
    {
        // Enqueue scripts and styles
    }

    // Multiple action hooks on the same method
    #[Action('wp_loaded', priority: 20)]
    #[Action('template_redirect', priority: 10)]
    public function onWordPressLoaded(): void
    {
        // Code that runs on multiple hooks
    }
}
```

### Using Filter Attributes

```php
use Pollora\Attributes\Filter;

class MyPluginClass implements Hooks
{
    #[Filter('the_content', priority: 10)]
    public function modifyContent(string $content): string
    {
        return str_replace('old', 'new', $content);
    }
}
```

## Autoloading

### PSR-4 Autoloading

Plugins use PSR-4 autoloading with the namespace `Plugin\{PluginName}\`:

```php
// File: app/Services/ExampleService.php
namespace Plugin\MyAwesomePlugin\Services;

class ExampleService
{
    public function doSomething(): string
    {
        return 'Hello from plugin service!';
    }
}
```

### Source Directories

The autoloader looks for classes in these directories (in order):
1. `app/` (preferred)
2. `src/` (fallback)

## Plugin Management

### Listing Plugins

```bash
# List all plugins
php artisan pollora:plugin:list

# List only active plugins
php artisan pollora:plugin:list --active

# List with detailed information
php artisan pollora:plugin:list --detailed

# Output as JSON
php artisan pollora:plugin:list --format=json
```

### Plugin Status

```bash
# Show all plugin status
php artisan pollora:plugin:status

# Show specific plugin details
php artisan pollora:plugin:status my-awesome-plugin

# Show only active plugins
php artisan pollora:plugin:status --active
```

### Programmatic Access

```php
use Pollora\Plugin\Application\Services\PluginManager;

$pluginManager = app(PluginManager::class);

// Get all plugins
$plugins = $pluginManager->getAllPlugins();

// Find specific plugin
$plugin = $pluginManager->findPlugin('my-awesome-plugin');

// Check if plugin is active
$isActive = $pluginManager->isPluginActive('my-awesome-plugin');

// Get plugin information
$info = $pluginManager->getPluginInfo('my-awesome-plugin');
```

## Configuration

### Plugin Configuration

The plugin configuration is automatically loaded by the Modules system:

```php
// config/plugin.php
return [
    'name' => 'My Awesome Plugin',
    'version' => '1.0.0',
    'slug' => 'my-awesome-plugin',
    
    // Vite asset container configuration
    'assets' => [
        'version' => '1.0.0',
        'assets_path' => 'resources/assets',
        
        'asset_container' => [
            'hot_file' => public_path('my-awesome-plugin.hot'),
            'build_directory' => 'build/plugins/my-awesome-plugin',
            'manifest_path' => 'manifest.json',
            'base_path' => 'resources/assets/',
            'module_path' => __DIR__ . '/../',
            'module_type' => 'plugin',
        ],
    ],
    
    'features' => [
        'admin_menu' => true,
        'settings_page' => true,
        'frontend_scripts' => true,
        'ajax_handlers' => true,
    ],
    
    'admin_menu' => [
        'page_title' => 'My Plugin Settings',
        'menu_title' => 'My Plugin',
        'capability' => 'manage_options',
        'menu_slug' => 'my-plugin-settings',
    ],
];
```

Access configuration in your plugin:

```php
$config = config('plugin.plugin');
$features = config('plugin.plugin.features');
```

## Assets & Vite Integration

### Modern Asset Management

Pollora plugins support modern asset management with Vite, including hot reload and Tailwind CSS:

### Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from "vite";
import laravel, { refreshPaths } from 'laravel-vite-plugin';
import path from 'path';
import tailwindcss from '@tailwindcss/vite';

const pluginName = path.basename(__dirname);
const port = 5174; // Different port for plugins

const getPluginConfig = () => ({
    base: "/build/plugins/" + pluginName,
    input: ["./resources/assets/app.js"],
    publicDirectory: "../../public",
    hotFile: path.join("../../public", `${pluginName}.hot`),
    buildDirectory: path.join("build", "plugins", pluginName),
    refresh: [
        ...refreshPaths,
        'public/content/plugins/'+pluginName+'/resources/views/**',
        'public/content/plugins/'+pluginName+'/app/**/*.php',
    ],
});

export default defineConfig({
    base: "/build/plugins/" + pluginName,
    build: {
        emptyOutDir: false,
    },
    plugins: [
        tailwindcss(),
        laravel(getPluginConfig()),
    ],
    server: {
        port,
        // Additional server configuration for Docker/DDEV
    }
});
```

### Package.json

```json
{
    "name": "my-awesome-plugin",
    "version": "1.0.0",
    "private": true,
    "type": "module",
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview"
    },
    "devDependencies": {
        "@tailwindcss/postcss": "^4.0.0",
        "@tailwindcss/vite": "^4.0.0",
        "laravel-vite-plugin": "^1.0.0",
        "postcss": "^8.4.49",
        "tailwindcss": "^4.0.0",
        "vite": "^6.0.0"
    }
}
```

### Asset Container

Assets are automatically registered with a plugin-specific container:

```php
// Automatic asset registration via AssetServiceProvider
Asset::add('my-plugin/app', 'app.js')
    ->container('plugin.my-awesome-plugin')
    ->toFrontend()
    ->useVite();

Asset::add('my-plugin/styles', 'app.css')
    ->container('plugin.my-awesome-plugin')
    ->toFrontend()
    ->useVite();
```

### Development Workflow

```bash
# Install dependencies
npm install

# Start development server with hot reload
npm run dev

# Build for production
npm run build
```

## Views

### Automatic View Registration

Plugin views are automatically discovered and registered by the Modules system. Views are located in `resources/views/` and are accessible via the plugin namespace:

```php
// Render a plugin view
return view('my-awesome-plugin::welcome', ['data' => $someData]);

// In a controller or plugin method
public function showPage()
{
    $data = [
        'title' => 'My Plugin Page',
        'content' => 'Hello from plugin!'
    ];
    
    return view('my-awesome-plugin::admin.settings', $data);
}
```

### Blade Templates

Create Blade templates in the `resources/views/` directory:

```blade
{{-- resources/views/welcome.blade.php --}}
@extends('layouts.app')

@section('content')
<div class="my-awesome-plugin-container">
    <div class="my-awesome-plugin-card">
        <h1>Welcome to {{ $title }}</h1>
        <p>{{ $content }}</p>
    </div>
</div>
@endsection
```

### View Paths

The following view paths are automatically registered:
- `{plugin}/resources/views` (primary)
- `{plugin}/views` (fallback)

### View Namespaces

Views are automatically namespaced with the plugin slug:
- `my-awesome-plugin::template-name`
- `my-awesome-plugin::admin.settings`
- `my-awesome-plugin::components.button`

## Frontend Development

### JavaScript Structure

```javascript
// resources/assets/app.js
import './app.css';

/**
 * Main JavaScript file for My Awesome Plugin.
 */
class MyAwesomePluginPlugin {
    constructor() {
        this.init();
    }

    init() {
        if (document.readyState === 'loading') {
            document.addEventListener('DOMContentLoaded', () => this.onDOMReady());
        } else {
            this.onDOMReady();
        }
    }

    onDOMReady() {
        console.log('My Awesome Plugin loaded');
        this.setupPlugin();
    }

    setupPlugin() {
        // Add your plugin-specific initialization code here
    }

    /**
     * Utility method for AJAX requests
     */
    async ajaxRequest(action, data = {}) {
        const formData = data instanceof FormData ? data : new FormData();
        
        if (!(data instanceof FormData)) {
            Object.entries(data).forEach(([key, value]) => {
                formData.append(key, value);
            });
        }

        formData.append('action', action);
        formData.append('nonce', window.my_awesome_plugin_ajax?.nonce || '');

        try {
            const response = await fetch(window.my_awesome_plugin_ajax?.ajax_url || '/wp-admin/admin-ajax.php', {
                method: 'POST',
                body: formData
            });

            return await response.json();
        } catch (error) {
            console.error('AJAX request failed:', error);
            throw error;
        }
    }
}

// Initialize the plugin
new MyAwesomePluginPlugin();
```

### CSS with Tailwind

```css
/* resources/assets/app.css */
@import "tailwindcss";

/* Plugin-specific styles */
.my-awesome-plugin-container {
    max-width: 56rem;
    margin: 0 auto;
    padding: 1rem;
}

.my-awesome-plugin-card {
    background-color: white;
    border-radius: 0.5rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
    padding: 1.5rem;
    border: 1px solid #e5e7eb;
}
```

### Tailwind Configuration

```javascript
// tailwind.config.js
export default {
    content: [
        "./resources/**/*.blade.php",
        "./resources/assets/**/*.{js,css}",
        "./app/**/*.php",
        "./resources/views/**/*.blade.php"
    ],
    theme: {
        extend: {
            // Plugin-specific theme extensions
        },
    },
    plugins: [
        // Add Tailwind plugins here if needed
    ],
}
```

## Translations

### Automatic Loading

Plugin translations are automatically loaded by the Modules system from the `languages/` directory.

### Text Domain

Use your plugin slug as the text domain:

```php
// In PHP
__('Hello World', 'my-awesome-plugin');
_e('Hello World', 'my-awesome-plugin');
sprintf(__('Hello %s', 'my-awesome-plugin'), $name);

// In Blade templates
{{ __('Hello World', 'my-awesome-plugin') }}
```

### Language Files

Create `.pot`, `.po`, and `.mo` files in the `languages/` directory following WordPress standards.

## Testing

### Unit Tests

Create tests in the `tests/` directory:

```php
<?php

namespace Tests\Plugin\MyAwesomePlugin;

use PHPUnit\Framework\TestCase;
use Plugin\MyAwesomePlugin\Services\ExampleService;

class ExampleServiceTest extends TestCase
{
    public function test_it_does_something(): void
    {
        $service = new ExampleService();
        $result = $service->doSomething();
        
        $this->assertEquals('Hello from plugin service!', $result);
    }
}
```

### Running Tests

```bash
# Run all framework tests
composer test

# Run plugin-specific tests
vendor/bin/phpunit tests/Plugin/MyAwesomePlugin/
```

## Deployment

### Plugin Packaging

Create a deployment script or use the built-in tools to package your plugin:

```bash
# Build assets for production
npm run build

# Create plugin zip file
zip -r my-awesome-plugin.zip my-awesome-plugin/ \
    -x "*.git*" "node_modules/*" "tests/*" "*.dev.*" "vite.config.js" "tailwind.config.js"
```

### WordPress Plugin Directory

For submission to the WordPress plugin directory:

1. Ensure all requirements are met
2. Include proper plugin headers
3. Follow WordPress coding standards
4. Include proper documentation
5. Test thoroughly

## Best Practices

### Code Organization

- Use service providers for dependency injection
- Keep controllers thin, services fat
- Use PHP 8 attributes for declarative hook registration
- Follow PSR-4 autoloading standards
- Implement proper error handling
- Let the Modules system handle discovery automatically

### Modern Development

- Use Vite for asset compilation and hot reload
- Leverage Tailwind CSS for styling
- Write modern JavaScript with ES6+ features
- Use Blade templates for views
- Follow Laravel conventions where applicable

### Security

- Always sanitize input data
- Use WordPress nonces for form submissions
- Validate user capabilities
- Escape output data
- Use prepared SQL statements

### Performance

- Load assets only when needed via conditional asset registration
- Use WordPress transients for caching
- Optimize database queries
- Implement lazy loading where possible
- Leverage Vite's code splitting features

### WordPress Integration

- Follow WordPress coding standards
- Use WordPress hooks and filters appropriately
- Respect WordPress conventions
- Ensure compatibility with popular plugins
- Test with different themes

### Module System Integration

- Let the Modules system handle automatic discovery
- Don't duplicate functionality (views, routes, translations, etc.)
- Use the mutualized asset and view management
- Follow the attribute-based hook pattern
- Keep service providers minimal and focused

This comprehensive guide reflects the modern plugin development approach with Pollora, emphasizing automatic discovery, modern tooling, and clean architecture patterns.