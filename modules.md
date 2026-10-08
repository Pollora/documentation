# Module Management in Pollora

The Pollora framework utilizes the concept of modules to organize coherent sets of functionalities, thus enhancing maintainability, scalability, and clear separation of responsibilities. This modular management relies on the [Laravel Modules](https://laravelmodules.com/) package developed by Nwidart.

## Philosophy: Module vs WordPress Plugin

In Pollora, a module organizes functionalities specific to a particular project, which are usually hard to reuse elsewhere. This contrasts with a WordPress plugin, which is generally designed to be generic and reusable across multiple projects. Thus, modules provide an optimal solution to encapsulate specific business logic while fully leveraging the Laravel ecosystem.

All functionalities available in the main application directory (`app/`), such as managing post types, taxonomies, WordPress hooks, etc., are fully accessible within modules. Additionally, Pollora provides **automatic discovery** of structures within modules, eliminating the need for manual registration of service providers, post types, taxonomies, and other Laravel/WordPress components.

## What is a Module?

A module in Pollora groups autonomous functionalities that can be enabled or disabled on demand. Each module has its own file structure, allowing clear management of associated resources, routes, views, configurations, migrations, models, and tests.

## Creating a Module

```shell
php artisan pollora:make:module Portfolio
```

The command downloads the [module-default](https://github.com/Pollora/module-default) template, fills in its name, enables the module, runs `composer dump-autoload` (the module's `composer.json` is merged into the project's) and builds its assets with npm. The default module holds what a Pollora module uses, and nothing it would have to delete:

```
Modules/Portfolio/
├── app/
│   └── Cms/Hooks/PortfolioHooks.php  # an #[Action] example, discovered without a provider
├── resources/
│   ├── assets/app.js, app.css         # Tailwind CSS v4, without its preflight
│   └── views/blocks/                  # Gutenberg blocks, registered by Pollora
├── composer.json                      # PSR-4 Modules\Portfolio\ → app/
├── module.json                        # "providers": [] — discovery does the registering
├── package.json
└── vite.config.js                     # @pollora/vite-config, type "module"
```

There is no service provider, controller, route file, configuration, seeder or test folder by default: classes declared with attributes in `app/` are discovered. Each Laravel layer is one flag away:

| Flag | Adds |
| --- | --- |
| `--provider` | `app/Providers/PortfolioServiceProvider.php`, listed in `module.json` |
| `--routes` | `routes/web.php` and the `RouteServiceProvider` that loads it (implies `--provider`) |
| `--api` | `routes/api.php`, under `/api` (implies `--routes`) |
| `--config` | `config/config.php`, read as `config('portfolio.*')` (implies `--provider`) |
| `--database` | `database/migrations`, `seeders` and `factories`, with their PSR-4 entries |
| `--tests` | `tests/Feature`, `tests/Unit` and a Pest test |
| `--full` | every layer above |
| `--no-assets` | no `package.json`, `vite.config.js` or `resources/assets`, for a PHP-only module |

Other options: `--description`, `--author`, `--no-enable`, `--no-npm`, `--repository=owner/repo` and `--repo-version=tag` for another template, `--offline` for the copy bundled with the framework (also used when GitHub cannot be reached), `--force` to replace an existing module.

A layer added later uses nwidart's own generators: `php artisan module:make-provider PortfolioServiceProvider Portfolio`, `module:make-migration`, and so on.

### `module:make`

nwidart's `php artisan module:make Portfolio` writes the same lean module, offline, from the copy bundled with the framework. A project whose `config/modules.php` sets nwidart's `paths` or `stubs` (its own published configuration) keeps nwidart's stock module instead.

### Generating into a module

Pollora's generators take `--module`, and write where nwidart found the module, under the namespace its `composer.json` maps onto `app/` — a module may use `Module\Portfolio\` rather than `Modules\Portfolio\`:

```shell
php artisan pollora:make:post-type Project --module=Portfolio
php artisan pollora:make:block project-card --module=Portfolio
```

A block lands in `resources/views/blocks/project-card`, named `portfolio/project-card`.

## Frontend

A module builds like a theme or a plugin, with the same Vite, Tailwind CSS v4 and block tooling, through [`@pollora/vite-config`](https://github.com/Pollora/vite-config):

```js
// Modules/Portfolio/vite.config.js
import { defineConfig } from 'vite';
import pollora from '@pollora/vite-config';

export default defineConfig({
    plugins: [pollora({ type: 'module', name: 'portfolio' })],
});
```

| | |
| --- | --- |
| Build | `public/build/module/<kebab-name>`, with its `manifest.json` |
| Hot file | `public/<kebab-name>.hot` |
| Dev server | DDEV-aware, port 5175 (`VITE_PORT` to change it) |
| Entries | `resources/assets/app.js`, and every block in `resources/views/blocks` |

```shell
cd Modules/Portfolio
npm install
npm run dev      # hot reload
npm run build
```

Every enabled module with a `vite.config.js` gets the `module.<kebab-name>` asset container:

```php
use Pollora\Support\Facades\Asset;

Asset::add('portfolio/app', 'app.js')
    ->container('module.portfolio')
    ->toFrontend()
    ->useVite();
```

### A module made by an older `module:make`

nwidart's stock `vite.config.js` builds into `public/build-<lower>`, where Pollora never looks; `pollora:doctor` says so. Move the module onto the template's build:

```shell
php artisan pollora:module:frontend Portfolio
```

It writes `package.json`, `vite.config.js` and `resources/assets/app.{js,css}`, keeping a `.bak` of each file it replaces (`--no-backup` to skip them). Entries other than `app.js` go in the `input` option of `pollora()`. Nothing runs at upgrade: a project's modules are its own code.

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

A module's `require` is merged; its `require-dev` is not (`merge-dev: false`). A module installed with Composer is merged too, and its development tools (PHPUnit, Rector…) would otherwise become requirements of the project: the next `composer install` then removed WordPress core from `public/cms`. Development dependencies belong in the project's own `require-dev`. The skeleton sets it since v13.34.1; an older project adds it to `extra.merge-plugin`.

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
- The skeleton ships this rule since v13.34.1. A project created from v13.34.0 or earlier adds it to its `composer.json`, last, and `"merge-dev": false` to `extra.merge-plugin` (see Dependency Management).

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

An installed module is not enabled yet: `php artisan module:enable MeiliFacets`, or Plugins › Modules (below).

`composer remove pollora/meilifacets` deletes `Modules/MeiliFacets/`; remove its state too (`pollora:doctor` lists states left for modules no longer on disk).

### Updates

A module installed with Composer carries its package's version, matched by install path; a local module has none and is never checked. Pollora reads the latest release from where the project's `composer.json` gets the package: a `composer` repository (Private Packagist, Satis), a GitHub `vcs` repository (`MODULES_GITHUB_TOKEN` for a private one), or Packagist. It checks once a day from WP-Cron and caches the answer for 12 hours — never during a front-end request:

```shell
php artisan pollora:module:outdated          # checks now
php artisan pollora:module:outdated --json
```

An update shows in Plugins › Modules ("1.3.0 · 1.4.0 available", with the `composer update` command), in Site Health ("Pollora modules are up to date") and in `pollora:status`. Code still arrives through Composer: there is no one-click update. A `dev-*` version is never reported outdated.

## Enabling and Disabling Modules

```shell
php artisan module:enable Portfolio
php artisan module:disable Portfolio
```

Module providers register during Laravel's register phase, before anything can switch them: a change applies from the next request.

### Plugins › Modules

The Plugins screen has a **Modules (n)** view next to All, Active and Must-Use: every module, its description and path, its state, its version and where its state is stored, with Enable / Disable on each row and as bulk actions. Switching needs the `activate_plugins` capability (`modules.admin.capability`). WordPress's own plugin rows are untouched: a module has no plugin file for WordPress to load.

- A **locked** module has no switch, and the row says why (see below).
- `MODULES_ADMIN_TOGGLE=false` turns every switch off; they are on by default, production included.
- With the JSON file, the view warns that the next deployment resets a change, and each switch asks first. When the file cannot be written (a read-only release directory), switches are off and the view says how to change states.

Tools › Pollora and `pollora:status` also show where module states are stored.

### Where the state lives

Whether a module is enabled comes from a **connector**:

| Connector | Reads and writes | Survives a deployment | For |
| --- | --- | --- | --- |
| `json` (default) | `modules_statuses.json`, nwidart's file | Only if committed | Local work, simple deploys |
| `database` | the `pollora_modules` WordPress option (JSON, not autoloaded) | Yes | Sites switched from the admin |
| `config` | `connectors.config.states`, or `MODULES_ENABLED` / `MODULES_DISABLED` | Yes, it ships with the code | Immutable deploys, containers |

nwidart asks which modules are enabled while it registers, before any provider of the application and before WordPress loads. The connector is therefore chosen in a published `config/modules.php`, never from a provider:

```shell
php artisan vendor:publish --tag=pollora-modules
```

```php
// config/modules.php — use Pollora\Modules\Infrastructure\Activation\ModuleConnectors;
'activator' => 'pollora',
'connector' => env('MODULES_CONNECTOR', 'json'),
'connectors' => [
    'json' => ['path' => base_path('modules_statuses.json')],
    'database' => ['option' => 'pollora_modules', 'fallback' => 'json'],
    'config' => [
        'states' => [],
        'enabled' => ModuleConnectors::names(env('MODULES_ENABLED', '')),
        'disabled' => ModuleConnectors::names(env('MODULES_DISABLED', '')),
    ],
],
'locked' => [
    'enabled' => ModuleConnectors::names(env('MODULES_LOCKED_ENABLED', '')),
    'disabled' => ModuleConnectors::names(env('MODULES_LOCKED_DISABLED', '')),
],
```

Without this file, nwidart's own activator keeps reading `modules_statuses.json`, and `MODULES_CONNECTOR` or `MODULES_LOCKED_*` are ignored (`pollora:doctor` warns).

The `database` connector reads the option with Laravel's connection, which shares WordPress's database and table prefix, and writes it with `update_option()` once WordPress is loaded. While the options table cannot be read (a fresh install), it reads its `fallback` and `pollora:doctor` says so. Switch connector after copying the current states into the new one:

```shell
php artisan pollora:module:connector                    # where the states live now
php artisan pollora:module:connector database --import  # copy them, then set MODULES_CONNECTOR=database
```

**Locked modules.** `locked.enabled` and `locked.disabled` force a state over any connector; switching a locked module from the console throws `ModuleLockedException`, and the admin shows no switch for it.

**Your own connector** implements `Pollora\Modules\Domain\Contracts\ModuleStateConnector` (`all`, `set`, `forget`, `writable`, `persistent`, `label`) and is declared by class, or registered in `bootstrap/app.php`:

```php
'connectors' => [
    'redis' => ['class' => App\Modules\RedisStateConnector::class],
],
```

```php
// bootstrap/app.php, before ->create()
use Pollora\Modules\Infrastructure\Activation\ModuleConnectors;

ModuleConnectors::extend('redis', fn ($app, array $config) => new RedisStateConnector($app['redis']));
```

A switch clears nwidart's provider manifest and the discovery cache, and fires `Pollora\Modules\Domain\Events\ModuleEnabled` or `ModuleDisabled` (module, source `admin` or `console`, WordPress user). A configuration or route cache written before the switch still holds the old state: `php artisan optimize:clear` (`pollora:doctor` warns). Multisite sites share one state for the whole network.

## Automatic Discovery System

Pollora discovers the classes of every enabled module in its `app/` directory, with no registration:

- **Post types** and **taxonomies**: classes with `#[PostType]` and `#[Taxonomy]`
- **WordPress hooks**: methods with `#[Action]` and `#[Filter]`
- **REST routes**: `#[WpRestRoute]`
- **Scheduled tasks**: `#[Schedule]`
- **Gutenberg blocks**: folders in `resources/views/blocks`

```php
// Modules/Portfolio/app/Cms/PostTypes/Project.php
namespace Modules\Portfolio\Cms\PostTypes;

use Pollora\Attributes\PostType;
use Pollora\Attributes\PostType\Supports;

#[PostType('project')]
#[Supports(['title', 'editor', 'thumbnail'])]
class Project {}
```

After adding such a class with the discovery cache on, run `php artisan discovery:clear`.

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
- Test a module in its own `tests` directory (`pollora:make:module --tests`).
- **Leverage automatic discovery**: Use PHP 8 attributes instead of manual registrations for cleaner, more maintainable code.
- **Organize by feature**: Group related service providers, models, and controllers within logical subdirectories.
- **Follow naming conventions**: Use descriptive class names that clearly indicate their purpose and functionality.

## Checks

`pollora:doctor` (and Tools › Site Health) checks modules too: an unbuilt module or one built where Pollora does not look, a stock nwidart Vite config, a connector reading its fallback, a state for a module no longer on disk, a state file that cannot be written while the admin is the way to switch, caches older than the last switch, and `MODULES_*` settings ignored without `config/modules.php`.

## Learn More

To further explore the advanced features of the Laravel Modules package, refer to:
- [Laravel Modules Official Documentation](https://laravelmodules.com/docs)
- [Artisan Commands Specific to Laravel Modules](https://laravelmodules.com/docs/advanced/artisan-commands)
