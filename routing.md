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

Pollora Framework provides WordPress-specific routes that integrate with WordPress's conditional tags system.

### WordPress Routes vs Standard Routes

There are two main methods to define routes in your application:

1. **Standard Laravel Routes** (`Route::get()`, `Route::post()`, `Route::any()`, etc.):
   - These routes follow Laravel's standard routing behavior
   - They do not interact with WordPress's conditional tags system
   - They do not automatically receive WordPress-specific middleware

2. **WordPress-Specific Routes** (`Route::wp()` and `Route::wpMatch()`):
   - These routes integrate with WordPress's conditional tags system
   - They automatically receive WordPress-specific middleware:
     - `WordPressBindings`: Adds WordPress objects (post, query) to the route
     - `WordPressHeaders`: Manages HTTP headers for WordPress responses
     - `WordPressBodyClass`: Handles body classes for WordPress templates
   - They are processed through WordPress's conditional logic
   - `Route::wp()` accepts all HTTP verbs
   - `Route::wpMatch()` allows specifying specific HTTP verbs

### WordPress Route Conditions

WordPress routes use conditions defined in the `config/wordpress.php` file. These conditions map WordPress conditional tags to route URIs:

```php
return [
    'conditions' => [
        // Core WordPress conditions
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

        // WooCommerce conditions
        'is_shop' => 'shop',
        'is_product' => 'product',
        'is_cart' => 'cart',
        'is_checkout' => 'checkout',
        'is_account_page' => 'account',
        'is_product_category' => 'product_category',
        'is_product_tag' => 'product_tag',
        'is_wc_endpoint_url' => 'wc_endpoint',
    ],
    // ... other WordPress configuration options
];
```

The framework provides a default set of conditions in its `config/wordpress.php` file. You can override or extend these conditions in your application's `config/wordpress.php` file.

#### Publishing the WordPress Configuration File

To customize WordPress route conditions, you can publish the framework's configuration file to your application:

```bash
php artisan vendor:publish --tag=wp-config
```

This command will copy the framework's configuration file to your application's `config/` directory, allowing you to customize it according to your needs.

Each condition maps a WordPress conditional function (like `is_page()`) to a route URI or array of URIs. For example, `'is_page' => 'page'` means that when you use `Route::wordpress('page', ...)`, it will check if the current request matches the `is_page()` WordPress condition.

### Using WordPress Routes

To define a WordPress-specific route, you can use either the `wp` or `wpMatch` method:

```php
// Using wp (accepts all HTTP verbs)
Route::wp('single', function () {
    return view('post');
});

// Using wpMatch with specific HTTP verbs
Route::wpMatch('GET', 'page', 'landing-page', function() {
    return view('pages.landing');
});

// Using wpMatch with multiple HTTP verbs
Route::wpMatch(['GET', 'POST'], 'page', ['contact', 'request-a-quote'], [FormController::class, 'index']);

// Custom template route
Route::wp('template', 'contact', function () {
    return view('page');
});
```

The `wp` method is a convenient shortcut that accepts all HTTP verbs. If you need to restrict a route to specific HTTP methods, use `wpMatch` instead.

Both methods accept a variable number of arguments:
- For `wp`:
  - First argument is the WordPress condition (e.g., 'single', 'page')
  - Last argument is the callback function or controller action
  - Any arguments in between are passed as parameters to the WordPress condition function

- For `wpMatch`:
  - First argument is the HTTP method(s) ('GET', 'POST', etc. or an array of methods)
  - Second argument is the WordPress condition
  - Last argument is the callback function or controller action
  - Any arguments in between are passed as parameters to the WordPress condition function

For example:
```php
// Route for GET requests to a specific page
Route::wpMatch('GET', 'page', 'landing-page', function() {
    return view('pages.landing');
});

// Route for POST requests to a form page
Route::wpMatch('POST', 'page', 'contact', [FormController::class, 'submit']);

// Route accepting both GET and POST for a product
Route::wpMatch(['GET', 'POST'], 'singular', 'product', 123, function() {
    return view('product.show');
});

// Route for all HTTP verbs using wp shortcut
Route::wp('tax', 'product_cat', 'electronics', function() {
    return view('taxonomy.product-category');
});
```

#### Important Differences from Standard Routes

When using `Route::wp()` or `Route::wpMatch()` instead of `Route::any()` or other standard methods:

1. The route will be processed through WordPress's conditional tags system
2. WordPress-specific middleware will be automatically applied
3. WordPress objects like `$post` and `$wp_query` will be available in the route parameters
4. The route will only match if the WordPress condition is met

#### Template Routes

When working with WordPress custom templates (defined in `themes/[theme-name]/config/templates.php`), you can create specific routes to handle these templates. For example:

```php
// This will match any page using the "contact" template
Route::wp('template', 'contact', function () {
    return view('page');
});

// This will match any page using the "landing" template and pass data to the view
Route::wp('template', 'landing', function () {
    return view('page.landing', [
        'features' => Feature::all(),
    ]);
});
```

⚠️ **Important**: Template routes must be declared **before** the generic page route (`Route::wp('page')`), otherwise they won't be taken into account.

### Extending Route Conditions

You can extend the WordPress route conditions by adding your own entries to the `conditions` array in your application's `config/wordpress.php` file:

```php
// config/wordpress.php
return [
    'conditions' => [
        // Add your custom conditions
        'is_custom_post_type' => 'custom-post-type',
        'is_special_page' => ['special-page', 'special'],
        
        // Override existing conditions
        'is_page' => ['page', 'static-page'],
    ],
];
```

You can also create your own WordPress conditional functions and use them in your routes:

```php
// functions.php or a custom plugin file
function is_special_page() {
    // Your custom logic to determine if this is a special page
    return is_page() && has_term('special', 'page_category');
}

// routes/web.php
Route::wp('special-page', function() {
    return view('pages.special');
});
```

This approach allows you to create highly customized routing logic while maintaining the clean syntax of WordPress-style routes.