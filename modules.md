# Module Management in Pollora

The Pollora framework utilizes the concept of modules to organize coherent sets of functionalities, thus enhancing maintainability, scalability, and clear separation of responsibilities. This modular management relies on the [Laravel Modules](https://laravelmodules.com/) package developed by Nwidart.

## Philosophy: Module vs WordPress Plugin

In Pollora, a module organizes functionalities specific to a particular project, which are usually hard to reuse elsewhere. This contrasts with a WordPress plugin, which is generally designed to be generic and reusable across multiple projects. Thus, modules provide an optimal solution to encapsulate specific business logic while fully leveraging the Laravel ecosystem.

All functionalities available in the main application directory (`app/`), such as managing post types, taxonomies, WordPress hooks, etc., are fully accessible within modules. Additionally, Pollora provides **automatic discovery** of structures within modules, eliminating the need for manual registration of service providers, post types, taxonomies, and other Laravel/WordPress components.

## What is a Module?

A module in Pollora groups autonomous functionalities that can be enabled or disabled on demand. Each module has its own file structure, allowing clear management of associated resources, routes, views, configurations, migrations, models, and tests.

## Creating a New Module

To create a module named `Portfolio`, use the following artisan command:

```shell
php artisan module:make Portfolio
```

This automatically generates a basic structure in the `Modules/Portfolio` directory with the following architecture:

```
Modules
└── Portfolio
    ├── app
    │   ├── Http
    │   │   └── Controllers
    │   │       └── PortfolioController.php
    │   ├── Models
    │   └── Providers
    │       ├── PortfolioServiceProvider.php
    │       └── RouteServiceProvider.php
    ├── config
    │   └── config.php
    ├── database
    │   ├── factories
    │   ├── migrations
    │   └── seeders
    │       └── PortfolioDatabaseSeeder.php
    ├── resources
    │   ├── assets
    │   │   ├── js
    │   │   │   └── app.js
    │   │   └── sass
    │   │       └── app.scss
    │   └── views
    │       ├── layouts
    │       │   └── master.blade.php
    │       └── index.blade.php
    ├── routes
    │   ├── api.php
    │   └── web.php
    ├── tests
    │   ├── Feature
    │   └── Unit
    ├── composer.json
    ├── module.json
    ├── package.json
    └── vite.config.js
```

Each file serves a specific role:
- `Providers`: Configure module-specific services and routes.
- `Controllers`: Handle HTTP logic.
- `Models`: Eloquent models.
- `Views`: Blade views specific to the module.
- `Routes`: Define web and API routes for the module.
- `composer.json`: Define module-specific dependencies (merged via [wikimedia/composer-merge-plugin](https://github.com/wikimedia/composer-merge-plugin)).

## Dependency Management

Each module can have its own dependencies defined in `composer.json`, but actual installation is managed centrally at the root of the main application by merging dependency files:

```json
"extra": {
    "merge-plugin": {
        "include": [
            "Modules/*/composer.json"
        ]
    }
}
```

This approach ensures coherent, centralized package management while maintaining modular flexibility.

## Enabling and Disabling Modules

Modules can be activated or deactivated at any time, allowing dynamic management of available features:

```shell
# Enable a module
php artisan module:enable Portfolio

# Disable a module
php artisan module:disable Portfolio
```

Managing module states (enabled/disabled) is useful for:
- Gradually deploying features.
- Simplifying debugging by isolating feature sets.
- Reducing memory footprint or attack surface by temporarily disabling features.

## Automatic Discovery System

Pollora includes a powerful **automatic discovery system** that automatically detects and registers various components within your modules without requiring manual configuration.

### What Gets Discovered Automatically

The discovery system automatically finds and registers:

- **Service Providers**: Classes extending `Illuminate\Support\ServiceProvider`
- **Post Types**: Classes with `#[PostType]` attributes
- **Taxonomies**: Classes with `#[Taxonomy]` attributes  
- **WordPress Hooks**: Methods with `#[Action]` and `#[Filter]` attributes
- **REST API Routes**: Classes and methods with `#[WpRestRoute]` attributes
- **Scheduled Tasks**: Classes with `#[Schedule]` attributes

### How Discovery Works

When a module is registered, Pollora automatically:

1. **Scans** the module directory for PHP classes
2. **Discovers** classes and methods with relevant attributes or inheritance
3. **Registers** found components with WordPress and Laravel
4. **Applies** the discovered configurations

### Discovery in Action

For example, if you create a service provider in your module:

```php
// Modules/Portfolio/app/Providers/PortfolioServiceProvider.php
namespace Modules\Portfolio\Providers;

use Illuminate\Support\ServiceProvider;

class PortfolioServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Your service registrations
    }
}
```

This service provider will be **automatically discovered and registered** - no manual configuration needed!

Similarly, for WordPress post types:

```php
// Modules/Portfolio/app/Models/Project.php
namespace Modules\Portfolio\Models;

use Pollora\Attributes\PostType;

#[PostType(
    name: 'project',
    public: true,
    supports: ['title', 'editor', 'thumbnail']
)]
class Project
{
    // Your model logic
}
```

The post type will be automatically registered with WordPress.

### Manual Discovery Control

You can also trigger discovery manually using helper functions:

```php
// Discover all structures in a module
pollora_discover_module('/path/to/module');

// Discover structures in any path
pollora_discover_all_in_path('/custom/path');
```

Or use the discovery service directly:

```php
use Pollora\Modules\Domain\Contracts\ModuleDiscoveryOrchestratorInterface;

$discovery = app(ModuleDiscoveryOrchestratorInterface::class);
$discovery->discover('/path/to/module');
```

## Best Practices

- Keep each module focused on a single responsibility (Single Responsibility Principle).
- Use modules to clearly separate business contexts (Domain-Driven Design).
- Prefer using module-specific namespaces to avoid conflicts.
- Individually test modules using unit and integration tests within the `tests` directory.
- **Leverage automatic discovery**: Use PHP 8 attributes instead of manual registrations for cleaner, more maintainable code.
- **Organize by feature**: Group related service providers, models, and controllers within logical subdirectories.
- **Follow naming conventions**: Use descriptive class names that clearly indicate their purpose and functionality.

## Learn More

To further explore the advanced features of the Laravel Modules package, refer to:
- [Laravel Modules Official Documentation](https://laravelmodules.com/docs)
- [Artisan Commands Specific to Laravel Modules](https://laravelmodules.com/docs/advanced/artisan-commands)
