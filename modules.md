# Module Management in Pollora

The Pollora framework utilizes the concept of modules to organize coherent sets of functionalities, thus enhancing maintainability, scalability, and clear separation of responsibilities. This modular management relies on the [Laravel Modules](https://laravelmodules.com/) package developed by Nwidart.

## Philosophy: Module vs WordPress Plugin

In Pollora, a module organizes functionalities specific to a particular project, which are usually hard to reuse elsewhere. This contrasts with a WordPress plugin, which is generally designed to be generic and reusable across multiple projects. Thus, modules provide an optimal solution to encapsulate specific business logic while fully leveraging the Laravel ecosystem.

All functionalities available in the main application directory (`app/`), such as managing post types, taxonomies, WordPress hooks, etc., are fully accessible within modules.

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

## Best Practices

- Keep each module focused on a single responsibility (Single Responsibility Principle).
- Use modules to clearly separate business contexts (Domain-Driven Design).
- Prefer using module-specific namespaces to avoid conflicts.
- Individually test modules using unit and integration tests within the `tests` directory.

## Learn More

To further explore the advanced features of the Laravel Modules package, refer to:
- [Laravel Modules Official Documentation](https://laravelmodules.com/docs)
- [Artisan Commands Specific to Laravel Modules](https://laravelmodules.com/docs/advanced/artisan-commands)
