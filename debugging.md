# Debugging with the debug bar

`pollora/debugbar` puts WordPress and Pollora in [Laravel Debugbar](https://github.com/fruitcake/laravel-debugbar). Next to Laravel's own tabs, every page shows which route or template answered it, the queries WordPress ran through `$wpdb`, the hooks that fired, how WordPress parsed the request, and WordPress's phases on the timeline. REST and admin-ajax calls appear in the bar's request list too. The aim is to make Query Monitor unnecessary in a Pollora project.

- [Installation](#installation)
- [When it runs](#when-it-runs)
- [The tabs](#the-tabs)
- [REST and admin-ajax requests](#rest-and-admin-ajax-requests)
- [In wp-admin](#in-wp-admin)
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
| **Pollora** | What answered the request — a `Route::wp()` route and its condition, the template hierarchy with the Blade view and the conditional that picked it, a Laravel route, or WordPress alone — then the versions, discovery (where each location's classes came from and how long it took), modules, the theme, async actions registered and queued by this request, WordPress constants and drop-ins |
| **Doctor** | A **Run doctor** button that runs `pollora:doctor`'s web checks on demand and lists them, errors first. Nothing runs with the page |
| **WP Request** | The rewrite rule that matched, query vars, the queried object, the main query and its results, the conditional tags that are true, the template file and the candidates of each template hierarchy. In wp-admin, the admin page and screen. On a multisite, the site and network, every switch between sites and who made it, and a warning when the request ends still switched |
| **WP Queries** | The queries WordPress ran through `$wpdb`, with their time, full backtrace, rows, errors, duplicates, slow ones, and the main query marked, grouped by component: core, a plugin, a theme, a module, the application. Laravel's **Queries** tab keeps Eloquent's |
| **WP Hooks** | The actions that fired and how often, how many callbacks each had, and the callbacks Pollora registered, by class and method |
| **WP Hook timings** | Off by default. The slowest hook callbacks, by their own time (without the hooks they fire in turn) and in total, with their hook, priority, calls and component |
| **WP HTTP** | The calls made through `wp_remote_*`: result, time, transport, and who made them; a call a plugin answered in `pre_http_request` says so |
| **WP Cache** | Object-cache hits and misses, whether the cache is persistent, the transients set and who set them, OPcache |
| **WP Capabilities** | The `current_user_can()` checks: each distinct check once, granted or refused, and how many times it was made |
| **WP Blocks** | The blocks rendered by type, with their time and nesting; Pollora's Blade blocks marked; the block bindings that gave them values |
| **WP Assets** | Scripts, styles and script modules, printed in the header or the footer, with dependencies nobody registered; the Vite build or dev server of each theme, plugin and module |
| **WP Languages** | The locale, and the translation files WordPress looked for, found or not |
| **Timeline** | WordPress loading and the callbacks of `muplugins_loaded`, `init`, `wp_loaded`, `template_redirect`, `wp_head`… beside Laravel's measures |

WordPress queries are traced through two core filters, `log_query_custom_data` and `query`, rather than a `wpdb` class of its own: it works with Pollora's `db.php` drop-in, and with any other.

## REST and admin-ajax requests

A WordPress REST request or an admin-ajax call ends with `exit` before Laravel finishes the response, so Laravel Debugbar alone never records it. With this package, the request is stored and its id sent in the `phpdebugbar-id` header: a `fetch()` or XHR made from a page with the bar shows up in the bar's request list, with all its tabs.

A `wp_redirect()` exits too: its request is kept, and the page it leads to shows both.

Stored requests are opened through `_debugbar/open`, and the doctor through `_debugbar/pollora/doctor`; Laravel Debugbar limits both to local and private addresses by default.

## In wp-admin

WordPress prints admin pages itself, without Laravel's kernel, so Laravel Debugbar alone never shows there. With this package, the bar is printed at the bottom of admin pages and the request is stored like any other. Laravel's tabs about a request it answered (route, views, session) are left out. The REST calls the block editor makes are listed in the bar's request list. Set `DEBUGBAR_POLLORA_ADMIN=false` to keep wp-admin without the bar.

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
| `options.wp_queries.trace` | `true` | Full backtrace, error, rows and component for each query |
| `options.wp_hooks.count_filters` | `false` | Count filters too. It listens to every hook call, so it costs on every `apply_filters()` |
| `options.wp_hooks.timings` | `false` | Time every hook callback (**WP Hook timings**). Each callback is wrapped in place in `$wp_filter`, so code reading `$wp_filter` directly sees the wrapper; callbacks taking parameters by reference are not timed |
| `options.wp_hooks.timings_limit` | `200` | How many callbacks the timings tab lists |
| `admin.enabled` | `true` | Show the bar on wp-admin pages |
| `admin.hidden_collectors` | `route`, `views`, `session`, `livewire`, `inertia` | Laravel Debugbar tabs left out in wp-admin |
| `options.wp_capabilities.backtrace` | `false` | Say who made each distinct capability check |
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
| Queries, by caller and component, duplicates, errors | **WP Queries** |
| Request, conditionals, template | **WP Request**, and **Pollora** for the route or Blade view that answered |
| Hooks & actions | **WP Hooks** |
| HTTP API calls | **WP HTTP** |
| Transients, object cache | **WP Cache** |
| Capability checks | **WP Capabilities**, on by default here since checks are aggregated |
| Blocks | **WP Blocks** |
| Scripts, styles | **WP Assets** |
| Languages | **WP Languages** |
| Environment | **Pollora** (versions, constants, drop-ins) |
| PHP errors, doing it wrong | Laravel Debugbar's **Exceptions** tab; WordPress's notices go to the `wordpress` log channel ([WordPress Logging](wordpress-logging.md)) |
| Logs (`qm/debug`…) and timings (`qm/start`, `qm/stop`) | Still work, in **Messages** and the **Timeline** |
| Overview | Laravel Debugbar's time and memory |
| Redirects | Kept: the next page shows both requests |
| Admin screen | **WP Request**, in wp-admin |
| Multisite | **WP Request**, on a multisite |

Query Monitor cannot install its own `db.php` next to Pollora's, so its query panel loses callers and components in a Pollora project; **WP Queries** has them.
