# Debugging with the debug bar

`pollora/debugbar` puts WordPress and Pollora in [Laravel Debugbar](https://github.com/fruitcake/laravel-debugbar). Next to Laravel's own tabs, every page shows which route or template answered it, the queries WordPress ran through `$wpdb`, the hooks that fired, how WordPress parsed the request, and WordPress's phases on the timeline. REST and admin-ajax calls appear in the bar's request list too. The aim is to make Query Monitor unnecessary in a Pollora project.

- [Installation](#installation)
- [When it runs](#when-it-runs)
- [The tabs](#the-tabs)
- [REST and admin-ajax requests](#rest-and-admin-ajax-requests)
- [Configuration](#configuration)
- [Adding your own data](#adding-your-own-data)
  - [From a WordPress plugin or theme](#from-a-wordpress-plugin-or-theme)
  - [From a package or module](#from-a-package-or-module)
  - [With Laravel Debugbar alone](#with-laravel-debugbar-alone)
- [Coming from Query Monitor](#coming-from-query-monitor)

## Installation

```bash
composer require --dev pollora/debugbar
```

It brings `fruitcake/laravel-debugbar` with it and needs Pollora 13.35.3 or later. Nothing else to register: the service provider is discovered.

It is a development dependency on purpose. `composer install --no-dev` leaves it out, so a production deployment never ships it.

## When it runs

Exactly when Laravel Debugbar does: `DEBUGBAR_ENABLED`, or `APP_DEBUG` when that is not set, and never in the `production` or `testing` environments. Turning the bar off turns everything here off, including `SAVEQUERIES`.

To keep Laravel's tabs and drop Pollora's, set `DEBUGBAR_POLLORA_ENABLED=false`.

## The tabs

Pollora's tab comes right after Laravel's, then the WordPress tabs, prefixed `WP`, then those added by plugins and packages.

| Tab | Shows |
| --- | --- |
| **Pollora** | What answered the request — a `Route::wp()` route and its condition, the template hierarchy with the Blade view and the conditional that picked it, a Laravel route, or WordPress alone — then the versions, discovery (where each location's classes came from and how long it took), modules, the theme and async actions |
| **WP Request** | The rewrite rule that matched, query vars, the queried object, the main query and its results, the conditional tags that are true, the template file and the candidates of each template hierarchy |
| **WP Queries** | The queries WordPress ran through `$wpdb`, with their time and caller, duplicates, slow ones, and the main query marked. Laravel's **Queries** tab keeps Eloquent's |
| **WP Hooks** | The actions that fired and how often, how many callbacks each had, and the callbacks Pollora registered, by class and method |
| **Timeline** | WordPress loading and the callbacks of `muplugins_loaded`, `init`, `wp_loaded`, `template_redirect`, `wp_head`… beside Laravel's measures |

## REST and admin-ajax requests

A WordPress REST request or an admin-ajax call ends with `exit` before Laravel finishes the response, so Laravel Debugbar alone never records it. With this package, the request is stored and its id sent in the `phpdebugbar-id` header: a `fetch()` or XHR made from a page with the bar shows up in the bar's request list, with all its tabs.

Stored requests are opened through `_debugbar/open`, which Laravel Debugbar limits to local and private addresses by default.

## Configuration

```bash
php artisan vendor:publish --tag=debugbar-pollora-config
```

`config/debugbar-pollora.php` turns each tab on or off and sets their options. Each key also reads an environment variable:

| Key | Default | Effect |
| --- | --- | --- |
| `collectors.wp_queries` | `true` | Also turns `SAVEQUERIES` on; off, WordPress keeps no queries |
| `options.wp_queries.slow_threshold` | `50` | Milliseconds from which a query is highlighted |
| `options.wp_queries.soft_limit` / `hard_limit` | `100` / `500` | Past the first, no caller is kept; past the second, queries are left out |
| `options.wp_hooks.count_filters` | `false` | Count filters too. It listens to every hook call, so it costs on every `apply_filters()` |
| `options.bridges.query_monitor` | `true` | Keep Query Monitor's `qm/*` logging actions working |

## Adding your own data

Every tab says where its data comes from: Pollora, WordPress, or the name you give. Tab names starting with `wp_` or `pollora` are reserved; prefix yours with your own name.

### From a WordPress plugin or theme

Use actions: they need no dependency on the package, and do nothing where it is not installed.

```php
add_action('pollora/debugbar/register', function ($bar): void {
    // A tab of rows
    $bar->table('acme_cart', 'Acme cart', fn (): array => acme_cart_rows(), origin: 'acme-shop');

    // A tab of name => value pairs
    $bar->variables('acme_info', 'Acme', fn (): array => ['mode' => 'test'], origin: 'acme-shop');

    // A section in an existing tab
    $bar->section('wp_request', 'Acme', fn (): array => ['Cart' => acme_cart_id()]);
});

do_action('pollora/debugbar/message', 'Cart {id} rebuilt', 'info', ['id' => $cartId]);

do_action('pollora/debugbar/start', 'acme-sync');
// …
do_action('pollora/debugbar/stop', 'acme-sync');
```

The closures run when the bar collects, at the end of the request.

### From a package or module

Extend `Pollora\Debugbar\Collector` and tag the class:

```php
use Pollora\Debugbar\Collector;
use Pollora\Debugbar\CollectorRegistrar;
use Pollora\Debugbar\Widget;

final class CartCollector extends Collector
{
    public function getName(): string { return 'acme_cart'; }
    public function title(): string { return 'Acme cart'; }
    public function origin(): string { return 'acme-shop'; }
    public function widget(): Widget { return Widget::Table; }
    public function columns(): array { return ['qty' => 'Quantity']; }

    protected function data(): array
    {
        return ['apple' => ['qty' => 3]];
    }
}

// In a service provider's register(), when the package is installed
if (class_exists(Collector::class)) {
    $this->app->tag([CartCollector::class], CollectorRegistrar::COLLECTORS_TAG);
}
```

`widget()` is `Widget::Variables` (the default), `Widget::Table` or `Widget::Queries` (php-debugbar's SQL statement shape). `icon()` takes one of the icons php-debugbar ships, such as `box`, `table`, `tags` or `bolt`. To add a section to an existing tab instead, implement `Pollora\Debugbar\Contracts\SectionProvider` and tag it `CollectorRegistrar::SECTIONS_TAG`.

### With Laravel Debugbar alone

`Debugbar::addCollector()`, `debugbar.custom_collectors`, `Debugbar::addMessage()` and `startMeasure()` work as usual. Those tabs keep Debugbar's place, among Laravel's.

## Coming from Query Monitor

The two can run side by side while you switch. What Query Monitor shows and where it is here:

| Query Monitor | Here |
| --- | --- |
| Queries, duplicates | **WP Queries** |
| Request, conditionals, template | **WP Request**, and **Pollora** for the route or Blade view that answered |
| Hooks & actions | **WP Hooks** |
| PHP errors, doing it wrong | Laravel Debugbar's **Exceptions** tab; WordPress's notices go to the `wordpress` log channel ([WordPress Logging](wordpress-logging.md)) |
| Logs (`qm/debug`…) and timings (`qm/start`, `qm/stop`) | Still work, in **Messages** and the **Timeline** |
| Overview | Laravel Debugbar's time and memory |

HTTP API calls, transients and object cache, capabilities, blocks, enqueued assets and languages come in a later release of the package.
