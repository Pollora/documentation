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

### WordPressBodyClass

Adds route-based CSS classes to the WordPress `body_class` output, allowing you to style pages based on the matched route condition.

### WordPressShutdown

Ensures WordPress shutdown hooks (`shutdown` action, output buffer flushing) are properly executed after the Laravel response is sent.

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