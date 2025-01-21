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

Your implementation allows WordPress-style route declarations. You can define route conditions and their corresponding aliases using the 'conditions' array in the `config/app.php` file:

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

### WordPress-Like Route Usage

You can integrate WordPress-like route declarations using your defined conditions and aliases. Here are a few examples:

```php
// Dynamic route for single posts
Route::any('post', function () {
    return view('post');
});

// Route for a landing page
Route::any('page', ['landing-page', function() {
    return view('pages.landing');
}]);

// Route with multiple page slugs
Route::any('page', [['contact', 'request-a-quote'], [FormController::class, 'index']]);

// Custom template route
Route::any('template', ['contact', function () {
    return view('page');
}]);
```

#### Template Routes

When working with WordPress custom templates (defined in `themes/[theme-name]/config/templates.php`), you can create specific routes to handle these templates. For example, the route above will intercept all pages using the "contact" template.

⚠️ **Important**: Template routes must be declared **before** the generic page route (`Route::any('page')`), otherwise they won't be taken into account.

### Extending Route Conditions

You can extend your WordPress-style routes with your own conditions in the `config/app.php` file, providing greater customization in route behaviors.
