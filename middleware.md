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

## Sessions and the `web` group

Pollora adds no session of its own. Whether a request starts a Laravel session depends on which route answers it:

- **The template hierarchy** (the catch-all `{any}` route that resolves `page.blade.php`, `single.blade.php`, and so on) only gets the four Pollora middleware above. It has no session, no cookie and no CSRF check.
- **`Route::wp()` routes declared in `routes/web.php`** inherit Laravel's `web` group. Every request, anonymous visitors and crawlers included, starts a session: with the `database` driver that is an `INSERT` into the `sessions` table, plus two `Set-Cookie` headers (`<app>_session` and `XSRF-TOKEN`).

On a site where visitors authenticate through WordPress and the pages are served through a cache such as Varnish, that session is rarely useful. The cookies make responses look personalised to a proxy, and a cached page can carry one visitor's CSRF token to the others.

### Turning the session off for `Route::wp()` routes

Set `sessions` to `false` in `config/wordpress.php` (see [WordPress Configuration](wordpress-config.md#sessions)):

```php
'sessions' => false,
```

`Route::wp()` routes then skip `StartSession`, `ShareErrorsFromSession` and the CSRF middleware (`PreventRequestForgery`, also known as `ValidateCsrfToken`). Routes you declare with `Route::get()`, `Route::post()` and the like keep the full `web` group.

Without a session:

- `csrf_token()` returns an empty string, so a `<meta name="csrf-token">` tag has nothing to show.
- `old()`, `session()`, flash data and `$errors` have nothing to read from.
- A `POST` handled by a `Route::wp()` route is no longer checked for a CSRF token. If a form needs Laravel validation or CSRF protection, declare it with `Route::post()` instead, or leave `sessions` at `true`.

`Auth::once()` (see [Authentication](auth.md)) never relied on the session, and WordPress keeps its own login cookies, so WordPress authentication is not affected.

To remove the session for the whole application instead, remove the three middleware from the `web` group in `bootstrap/app.php`. Do this only if you have no Laravel-native form or authenticated route: it also removes CSRF protection from them.
