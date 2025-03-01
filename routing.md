# Routing

Laravel's routing system offers a robust way to define URLs and their corresponding actions. By integrating WordPress-like route declarations, along with advanced concepts like route groups, middleware, and route naming, you can create a highly customizable and organized routing structure.

## Laravel Routing Basics

In Laravel, routes are defined in the `routes/web.php` file. A route associates a URL with a controller action or a closure that returns a view. For example:

```php
// routes/web.php

use App\Http\Controllers\HomeController;

Route::get('/', [HomeController::class, 'index']);
```

Here, when the URL `/home` is accessed, the `index` method of the `HomeController` is invoked.

### Route Groups and Middleware

You can group routes to apply shared attributes, such as middleware, prefixes, or namespaces. Middleware can help perform actions before or after a request reaches the controller. For example:

```php
// Group routes with middleware and prefix
Route::middleware('auth')->prefix('admin')->group(function () {
    Route::get('dashboard', 'AdminController@dashboard');
    Route::get('settings', 'AdminController@settings');
});
```

### Route Alias (Naming)

Route naming, or aliasing, allows you to refer to routes by a name instead of their URL. This is particularly useful when generating URLs or redirecting. Here's how you can name a route:

```php
// Named route for the home page
Route::get('/', [HomeController::class, 'index'])->name('home');
```

You can find more about Laravel's routes in [the official documentation](https://laravel.com/docs/routing).

## WordPress-Style Route Declarations

Pollora Framework provides two ways to define routes: standard Laravel routes and WordPress-specific routes. The WordPress-specific routes are designed to integrate with WordPress's conditional tags system.

### WordPress Routes vs Standard Routes

There are two main methods to define routes in your application:

1. **Standard Laravel Routes** (`Route::get()`, `Route::post()`, `Route::any()`, etc.):
   - These routes follow Laravel's standard routing behavior
   - They do not interact with WordPress's conditional tags system
   - They do not automatically receive WordPress-specific middleware

2. **WordPress-Specific Routes** (`Route::wordpress()`):
   - These routes integrate with WordPress's conditional tags system
   - They automatically receive WordPress-specific middleware:
     - `WordPressBindings`: Adds WordPress objects (post, query) to the route
     - `WordPressHeaders`: Manages HTTP headers for WordPress responses
     - `WordPressBodyClass`: Handles body classes for WordPress templates
   - They are processed through WordPress's conditional logic

### WordPress Route Conditions

WordPress routes use conditions defined in the `config/wordpress.php` file. These conditions map WordPress conditional tags to route URIs:

```php
return [
    // ...
    'conditions' => [
        // Define WordPress-like route conditions and aliases
        'is_404' => '404',
        'is_archive' => 'archive',
        'is_attachment' => 'attachment',
        'is_author' => 'author',
        'is_category' => ['category', 'cat'],
        'is_date' => 'date',
        'is_day' => 'day',
        'is_front_page' => ['/', 'front'],
        'is_home' => ['home', 'blog'],
        'is_month' => 'month',
        'is_page' => 'page',
        'is_paged' => 'paged',
        'is_page_template' => 'template',
        'is_post_type_archive' => ['post-type-archive', 'postTypeArchive'],
        'is_search' => 'search',
        'is_single' => 'single',
        'is_singular' => 'singular',
        'is_sticky' => 'sticky',
        'is_subpage' => ['subpage', 'subpageof'],
        'is_tag' => 'tag',
        'is_tax' => 'tax',
        'is_time' => 'time',
        'is_year' => 'year',

        // WooCommerce
        'is_shop' => 'shop',
        'is_product' => 'product',
        'is_cart' => 'cart',
        'is_checkout' => 'checkout',
        'is_account_page' => 'account',
        'is_product_category' => 'product_category',
        'is_product_tag' => 'product_tag',
        'is_wc_endpoint_url' => 'wc_endpoint',
    ],
];
```

### Using WordPress Routes

To define a WordPress-specific route, use the `wordpress` method:

```php
// Dynamic route for single posts
Route::wordpress('single', function () {
    return view('post');
});

// Route for a specific page
Route::wordpress('page', ['landing-page', function() {
    return view('pages.landing');
}]);

// Route with multiple page slugs
Route::wordpress('page', [['contact', 'request-a-quote'], [FormController::class, 'index']]);

// Custom template route
Route::wordpress('template', ['contact', function () {
    return view('page');
}]);
```

#### Important Differences from Standard Routes

When using `Route::wordpress()` instead of `Route::any()` or other standard methods:

1. The route will be processed through WordPress's conditional tags system
2. WordPress-specific middleware will be automatically applied
3. WordPress objects like `$post` and `$wp_query` will be available in the route parameters
4. The route will only match if the WordPress condition is met

#### Template Routes

When working with WordPress custom templates (defined in `themes/[theme-name]/config/templates.php`), you can create specific routes to handle these templates. For example:

```php
// This will match any page using the "contact" template
Route::wordpress('template', ['contact', function () {
    return view('page');
}]);
```

⚠️ **Important**: Template routes must be declared **before** the generic page route (`Route::wordpress('page')`), otherwise they won't be taken into account.

### Extending Route Conditions

You can extend your WordPress-style routes with your own conditions in the `config/wordpress.php` file, providing greater customization in route behaviors.