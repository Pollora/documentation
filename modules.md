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
        ],
        "merge-dev": false
    }
}
```

This approach ensures coherent, centralized package management while maintaining modular flexibility.

A module's `require` is merged; its `require-dev` is not (`merge-dev: false`). A module installed with Composer is merged too, and its development tools (PHPUnit, Rector…) would otherwise become requirements of the project: the next `composer install` then removed WordPress core from `public/cms`. Development dependencies belong in the project's own `require-dev`. The skeleton sets it after v13.34.0; an older project adds it to `extra.merge-plugin`.

## Installing a Module with Composer

A module published as a Composer package installs with `composer require`, straight into `Modules/`, where nwidart finds it:

```bash
composer require pollora/meilifacets
```

The skeleton routes it there with one `installer-paths` rule, handled by `composer/installers` (already used for WordPress plugins and themes):

```json
"extra": {
    "installer-paths": {
        "public/content/mu-plugins/{$name}/": ["type:wordpress-muplugin"],
        "public/content/plugins/{$name}/": ["type:wordpress-plugin"],
        "public/content/themes/{$name}/": ["type:wordpress-theme"],
        "Modules/{$name}/": ["vendor:pollora"]
    }
}
```

- **The rule must come last.** `composer/installers` applies the first rule that matches, and a `vendor:` rule ignores the package type: placed first, it would also send Pollora's WordPress plugins (`pollora/mcp-connector`, `pollora/ai-visibility`) to `Modules/`. Last, it only catches what the rules above did not, which are the `pollora/*` modules of type `laravel-library`; Pollora's other packages (`library`, `project`) are not handled by `composer/installers` and stay in `vendor/`.
- **A module from another vendor needs its own line**, for instance `"Modules/{$name}/": ["vendor:pollora", "vendor:acme"]`. The rule does not target `type:laravel-library` alone: dozens of ordinary Laravel packages declare that type and would land in `Modules/`.
- The skeleton ships this rule after v13.34.0. A project created from v13.34.0 or earlier adds it to its `composer.json`, last, and `"merge-dev": false` to `extra.merge-plugin` (see Dependency Management).

### Publishing a module

The module's `composer.json` declares the type and the folder name:

```json
{
    "name": "pollora/meilifacets",
    "type": "laravel-library",
    "require": {
        "composer/installers": "^2.3"
    },
    "extra": {
        "installer-name": "MeiliFacets"
    }
}
```

`installer-name` sets the folder, `Modules/MeiliFacets/`, with the exact case nwidart expects; the package name itself has no naming constraint.

### Enabling it

An installed module is not enabled yet: add it to `modules_statuses.json`, or run `php artisan module:enable MeiliFacets`.

```json
{
    "MeiliFacets": true
}
```

`composer remove pollora/meilifacets` deletes `Modules/MeiliFacets/`; remove its line from `modules_statuses.json` too.

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
