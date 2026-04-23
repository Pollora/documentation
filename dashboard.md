# Dashboard & Status

Pollora includes a built-in admin dashboard and an Artisan CLI command that give you a complete overview of your framework installation, discovered entities, and system health.

## Admin Dashboard

The dashboard is accessible in the WordPress admin under **Tools > Pollora**. It displays a branded overview of your project with the following cards:

### Information displayed

| Card | Details |
|------|---------|
| **Environment** | PHP, Laravel, and WordPress versions |
| **WordPress Config** | WP_DEBUG status, multisite, permalink structure |
| **Post Types** | Discovered post types with labels, slugs, and class names |
| **Taxonomies** | Discovered taxonomies with labels, slugs, and class names |
| **Hooks** | Total count of discovered actions and filters |
| **REST API Routes** | Classes discovered via `#[WpRestRoute]` |
| **WP-CLI Commands** | Classes discovered via `#[WpCli]` |
| **Scheduled Tasks** | Methods discovered via `#[Schedule]` |
| **Auto-discovered Providers** | Service providers found by the discovery engine |
| **Modules** | Laravel modules status (enabled/disabled) via nwidart/laravel-modules |
| **Discovery Cache** | Cache driver and enabled status |
| **Discovery Performance** | Cache hits/misses, classes processed, instance pool size |
| **Active Theme** | Theme name, version, and template directory |

### Update notification

When a newer stable version of Pollora is available on Packagist, a notification badge appears on the **Tools > Pollora** menu item, similar to WordPress's Site Health counter. Dev versions (`dev-develop`, `dev-main`, etc.) are excluded from this check to avoid false positives.

### Access control

The dashboard page requires the `manage_options` capability (administrators only).

## CLI Command

The `pollora:status` Artisan command provides the same information in the terminal:

```bash
php artisan pollora:status
```

Example output:

```
Pollora v13.4.0 (latest: v13.4.0) ✓

  PHP 8.3.12 | Laravel 13.5 | WordPress 6.9

  WP_DEBUG: off | Multisite: no | Permalinks: /%postname%/

  Post Types: 2 registered (via discovery)
    · Projects [project] — App\Cms\PostTypes\Project
    · Services [service] — App\Cms\PostTypes\Service
  Taxonomies: 1 registered
    · Project Categories [project-category] — App\Cms\Taxonomies\ProjectCategory
  Hooks: 4 registered (2 actions, 2 filters)
  REST API routes: 1 registered
    · App\Cms\Rest\ProjectController
  WP-CLI commands: 1 registered
    · App\Cms\Cli\SeedCommand
  Scheduled tasks: 2 registered
    · App\Cms\Schedule\CacheCleanup::cleanExpiredTransients()
    · App\Cms\Schedule\CacheCleanup::cleanRevisions()
  Auto-discovered providers: 1
    · App\Providers\AppServiceProvider

  Modules: 0 total (0 enabled, 0 disabled)

  Discovery cache: enabled (LaravelDiscoverCacheDriver)
  Discovery stats: 3 cache hits, 0 misses, 24 classes

  Theme: Starter Theme v1.0.0 (pollora-starter)
```

### JSON output

Use the `--json` flag for machine-readable output, useful for AI agents, CI pipelines, or monitoring tools:

```bash
php artisan pollora:status --json
```

This outputs the complete system information as a JSON object:

```json
{
    "framework": {
        "current": "13.4.0",
        "latest": "13.4.0",
        "update_available": false
    },
    "environment": {
        "php": "8.3.12",
        "laravel": "13.5.0",
        "wordpress": "6.9"
    },
    "wordpress": {
        "debug": false,
        "multisite": false,
        "permalink_structure": "/%postname%/"
    },
    "discovery": {
        "post_types": { "count": 2, "items": ["..."] },
        "taxonomies": { "count": 1, "items": ["..."] },
        "hooks": { "count": 4, "actions": 2, "filters": 2 },
        "rest_routes": { "count": 1, "items": ["..."] },
        "wp_cli_commands": { "count": 1, "items": ["..."] },
        "schedules": { "count": 2, "items": ["..."] },
        "service_providers": { "count": 1, "items": ["..."] }
    },
    "performance": { "..." },
    "cache": { "driver": "LaravelDiscoverCacheDriver", "enabled": true },
    "modules": { "count": 0, "enabled": 0, "disabled": 0, "items": [] },
    "theme": { "name": "Starter Theme", "version": "1.0.0", "template": "pollora-starter" }
}
```

### Dev version detection

When running a dev branch (`dev-develop`, `dev-main`, etc.), the command adapts its output:

```
Pollora dev-develop (latest stable: v13.4.0)
```

No misleading "update available" warning is shown for development installations.

## Architecture

The dashboard module follows the same DDD structure as other framework components:

```
src/Dashboard/
├── Domain/Services/
│   └── SystemInfoCollector.php    # Collects all system data
├── Infrastructure/Providers/
│   └── DashboardServiceProvider.php  # Registers page, command, badge
└── UI/
    ├── Console/
    │   └── StatusCommand.php      # pollora:status Artisan command
    └── Http/
        └── DashboardController.php   # Admin page renderer
```

The `SystemInfoCollector` service is registered as a singleton and can be injected into your own code if needed:

```php
use Pollora\Dashboard\Domain\Services\SystemInfoCollector;

class MyController
{
    public function __construct(
        private readonly SystemInfoCollector $collector
    ) {}

    public function health(): array
    {
        return $this->collector->collect();
    }
}
```