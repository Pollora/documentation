# Middleware

Middleware sits between a request and a response, allowing you to filter and manipulate HTTP requests. Pollora uses standard Laravel middleware and adds WordPress-specific middleware for seamless integration.

For general Laravel middleware concepts, see the [official Laravel documentation](https://laravel.com/docs/13.x/middleware).

## Pollora Middleware

Pollora includes four WordPress-specific middleware that are automatically applied to WordPress routes:

### WordPressBindings

Injects WordPress objects (`WP_Post`, `WP_Query`, `WP_User`, etc.) into route closures and controller methods based on type hints.

```php
// The $post parameter is automatically resolved from the current WordPress context
Route::wp('single', function (WP_Post $post) {
    return view('post', compact('post'));
});
```

### WordPressHeaders

Manages HTTP headers for WordPress responses:
- Adds `X-Powered-By: Pollora` header
- Controls cache headers
- Removes unnecessary WordPress headers for unauthenticated requests

### Body classes on Laravel routes

Not a middleware: a listener on Laravel's `RouteMatched` event, so it also reaches routes that carry none of the middleware above.

WordPress resolves every request against its own rewrite rules first, so a URL only a Laravel route knows (`Route::get('/dashboard/{tab}')`) comes out of it as a 404. On such a route, Pollora clears that verdict — `is_404()` is false, `<body>` loses `error404` and the title is no longer "Page not found" — and adds the route's URI segments as body classes: `/dashboard/settings` gives `dashboard tab-settings`.

A route WordPress answers — `Route::wp()` and the template-hierarchy fallback — is left as WordPress resolved it, so a real 404 keeps its `error404` class.

### WordPressShutdown

Ensures WordPress shutdown hooks (`shutdown` action, output buffer flushing) are properly executed after the Laravel response is sent.

### WordPressTemplateEnhancement

Gives a Blade page what WordPress gives a template it includes: its **template enhancement output buffer**. WordPress 7 relies on it for classic themes — block styles load on demand, are printed at `wp_footer` once the page's blocks are known, then are moved back into the `<head>` with `global-styles` through the `wp_template_enhancement_output_buffer` filter. Without it, those styles end up at the bottom of the page, after the content they style.

The middleware fires `wp_template_enhancement_output_buffer_started` before the view renders, then runs the filter and the `wp_finalized_template_enhancement_output_buffer` action on the HTML of the response. Any plugin built on that filter sees Pollora's pages too. JSON, redirects and streamed responses are left alone, and a site that opted out through `wp_should_output_buffer_template_for_enhancement` keeps its responses as they are. Since v13.34.1.

A plain Laravel route does not get it. If its view prints `wp_head()` and `wp_footer()`, add the middleware yourself:

```php
use Pollora\Route\Infrastructure\Middleware\WordPressTemplateEnhancement;

Route::get('/dashboard', DashboardController::class)
    ->middleware(WordPressTemplateEnhancement::class);
```

## Creating Custom Middleware

Use Artisan to generate a new middleware:

```bash
php artisan make:middleware CheckWordPressCapability
```

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckWordPressCapability
{
    public function handle(Request $request, Closure $next, string $capability = 'edit_posts')
    {
        if (! current_user_can($capability)) {
            abort(403);
        }

        return $next($request);
    }
}
```

## Applying Middleware to WordPress Routes

```php
Route::wp('page', fn () => view('page'))
    ->middleware('auth');

Route::wp('single', fn () => view('post'))
    ->middleware(CheckWordPressCapability::class . ':publish_posts');
```

## Middleware Groups

You can group WordPress routes with shared middleware:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::wp('page', fn () => view('page'));
    Route::wp('single', fn () => view('post'));
});
```