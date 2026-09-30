# Dashboard, Status & Doctor

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
Pollora v13.34.0 (latest: v13.34.0) ✓

  PHP 8.4.12 | Laravel 13.34 | WordPress 7.1

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
        "current": "13.34.0",
        "latest": "13.34.0",
        "update_available": false,
        "development": false
    },
    "environment": {
        "php": "8.4.12",
        "laravel": "13.34.0",
        "wordpress": "7.1"
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

When running a dev branch (`dev-develop`, `13.x-dev`, etc.), the command adapts its output:

```
Pollora dev-develop (latest stable: v13.34.0)
```

No misleading "update available" warning is shown for development installations: `development` is `true` in the JSON output, and Site Health reports a development build instead of comparing it with releases.

## Diagnosing a project: `pollora:doctor`

`pollora:status` says what is there. `pollora:doctor` says whether it works — and, under each problem, the command that fixes it:

```bash
php artisan pollora:doctor
```

```plaintext
  ✓ WordPress core patch — The core is patched, and __() is Pollora's.
  ✗ Composer patches lock — patches.lock.json is older than the framework's patches: Composer applies the old ones, or none.
      johnpbloch/wordpress-core: Patch __ method in l10n to stop conflicting with Laravel
      → composer patches-relock && composer patches-repatch
  ! Theme pattern files — 1 pattern file(s) are never registered by WordPress.
      patterns/masthead.html: WordPress reads only .php files in patterns/
      → Make each one a .php file whose header is a docblock with Title and Slug (/** Title: … Slug: my-theme/… */)

  1 error(s), 1 warning(s).
```

It looks for failures that stay silent — the site renders, every command exits 0 — each one met in practice:

| Check | What it catches |
|---|---|
| WordPress core patch | the core still declares `__()` (a patch Composer skipped), or `__()` is not Pollora's |
| Composer patches lock | `patches.lock.json` missing, or older than the framework's patches |
| Environment file | `.env` names Pollora does not read (`DB_NAME`, `DB_USER`, `WP_HOME`…), MySQL settings on a sqlite connection |
| Configuration and route caches | configuration or routes cached outside production: edits to `.env` or `routes/` are ignored |
| Discovery cache | classes added since the discovery cache was written, which are not registered — named, one by one |
| Theme, plugin and module builds | no theme; a build missing, or written to another folder than Pollora reads; a hot file pointing at a Vite dev server that is stopped or whose port is not exposed |
| Symlinked directories | a theme, plugin or module linked under another name: the build and the site disagree on its folder |
| Template placeholders | `%theme_*%` / `%plugin_*%` or `.stub` files left in a theme or plugin copied instead of generated |
| Theme pattern files | `.html` files in `patterns/` (WordPress reads only `.php`), patterns without a Title or Slug |
| Theme pattern cache | pattern files missing from WordPress's cached list |
| Routes over block templates | `Route::wp()` routes answering in place of a block theme's templates |
| Blocks in the legacy folder | blocks still in `resources/blocks`, which stops loading in v15 |

The builds, directories, placeholders and blocks are checked for the active theme, every Pollora plugin and every enabled Laravel module.

`--json` prints the same for scripts. The command exits `1` when a check finds an error (warnings exit `0`), so it can gate a deploy or a CI job.

### In Site Health

The same checks appear in WordPress's **Tools › Site Health**, with a *Pollora* badge, plus one only a web request can make: every block of the theme, the Pollora plugins and the modules is registered. Site Health runs in an administrator's web request — the boot a visitor gets — which is where a block can be missing while WP-CLI sees it.

## Programmatic Access

The `SystemInfoCollector` service is registered as a singleton and can be injected into your own code to access system information programmatically:

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