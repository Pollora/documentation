# Routing

The framework implements a hybrid routing system that combines Laravel's powerful routing capabilities with WordPress's template hierarchy. This system follows a clear priority order:

1. **Laravel Routes First**: All routes defined in your `routes/web.php` file take precedence. This gives you full control over specific URLs and allows you to leverage Laravel's routing features when needed.

2. **WordPress Template Hierarchy as Fallback**: If no Laravel route matches the current URL, the system falls back to WordPress's template hierarchy system, automatically mapping URLs to corresponding Blade views.

This "Laravel-first, WordPress-fallback" approach provides the best of both worlds:
- Full control and flexibility of Laravel routing when you need it
- Automatic handling of standard WordPress content through the template hierarchy
- No need to define routes for every WordPress page or post

```
Incoming Request
       │
       ▼
┌──────────────┐
│ WordPress    │    Yes    ┌─────────────────┐
│ Admin Area?  ├──────────►│ Handle via      │
└──────┬───────┘           │ Admin Route     │
       │ No                └─────────────────┘
       ▼
┌──────────────┐
│ Special WP   │    Yes    ┌─────────────────┐
│ Request?     ├──────────►│ Handle via      │
└──────┬───────┘           │ Special Route   │
       │ No                └─────────────────┘
       ▼
┌──────────────┐
│ Match Laravel│    Yes    ┌─────────────────┐
│    Route?    ├──────────►│ Execute Laravel │
└──────┬───────┘           │     Route       │
       │                   └─────────────────┘
       │ No
       ▼
┌──────────────┐
│    Match     │    Yes    ┌─────────────────┐
│   WordPress  ├──────────►│ Render Template │
│  Template?   │           └─────────────────┘
└──────┬───────┘
       │ No
       ▼
┌──────────────┐
│  Fallback to │
│ Frontend     │
│ Controller   │
└──────────────┘
```

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

WordPress routes use conditions that can be specified in two ways:

1. Using predefined aliases (shorter syntax)
2. Using actual WordPress conditional function names (more explicit)

The framework provides a default set of aliases in the `config/wordpress.php` file. These aliases map WordPress conditional tags to route URIs:

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

The framework provides a default set of aliases in its `config/wordpress.php` file. You can override or extend these aliases in your application's `config/wordpress.php` file.

#### Publishing the WordPress Configuration File

To customize WordPress route conditions, you can publish the framework's configuration file to your application:

```bash
php artisan vendor:publish --tag=wp-config
```

This command will copy the framework's configuration file to your application's `config/` directory, allowing you to customize it according to your needs.

Each alias maps a WordPress conditional function (like `is_page()`) to a route URI or array of URIs. For example, `'is_page' => 'page'` means that when you use `Route::wp('page', ...)` or `Route::wp('is_page', ...)`, it will check if the current request matches the `is_page()` WordPress condition.

### Using WordPress Routes

To define a WordPress-specific route, you can use either the `wp` or `wpMatch` method. You have two ways to specify the WordPress condition:

1. Using the predefined aliases (shorter syntax)
2. Using the actual WordPress conditional function names (more explicit)

Here are examples of both approaches:

```php
// Using aliases (shorter syntax)
Route::wp('single', function () {
    return view('post');
});

// Using actual WordPress function names (more explicit)
Route::wp('is_single', function () {
    return view('post');
});

// Using wpMatch with aliases
Route::wpMatch('GET', 'page', 'landing-page', function() {
    return view('pages.landing');
});

// Using wpMatch with actual WordPress function names
Route::wpMatch('GET', 'is_page', 'landing-page', function() {
    return view('pages.landing');
});

// Using wpMatch with multiple HTTP verbs and aliases
Route::wpMatch(['GET', 'POST'], 'page', ['contact', 'request-a-quote'], [FormController::class, 'index']);

// Using wpMatch with multiple HTTP verbs and actual WordPress function names
Route::wpMatch(['GET', 'POST'], 'is_page', ['contact', 'request-a-quote'], [FormController::class, 'index']);
```

Both methods accept a variable number of arguments:
- For `wp`:
  - First argument is the WordPress condition (either an alias or the actual function name)
  - Last argument is the callback function or controller action
  - Any arguments in between are passed as parameters to the WordPress condition function

- For `wpMatch`:
  - First argument is the HTTP method(s) ('GET', 'POST', etc. or an array of methods)
  - Second argument is the WordPress condition (either an alias or the actual function name)
  - Last argument is the callback function or controller action
  - Any arguments in between are passed as parameters to the WordPress condition function

For example:
```php
// Route for GET requests to a specific page using alias
Route::wpMatch('GET', 'page', 'landing-page', function() {
    return view('pages.landing');
});

// Route for GET requests to a specific page using actual function name
Route::wpMatch('GET', 'is_page', 'landing-page', function() {
    return view('pages.landing');
});

// Route for POST requests to a form page using alias
Route::wpMatch('POST', 'page', 'contact', [FormController::class, 'submit']);

// Route for POST requests to a form page using actual function name
Route::wpMatch('POST', 'is_page', 'contact', [FormController::class, 'submit']);

// Route accepting both GET and POST for a product using alias
Route::wpMatch(['GET', 'POST'], 'singular', 'product', 123, function() {
    return view('product.show');
});

// Route accepting both GET and POST for a product using actual function name
Route::wpMatch(['GET', 'POST'], 'is_singular', 'product', 123, function() {
    return view('product.show');
});

// Route for all HTTP verbs using wp shortcut with alias
Route::wp('tax', 'product_cat', 'electronics', function() {
    return view('taxonomy.product-category');
});

// Route for all HTTP verbs using wp shortcut with actual function name
Route::wp('is_tax', 'product_cat', 'electronics', function() {
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

### Automatic Template Routing

The framework implements a cascading routing system that works as follows:

1. **WordPress Admin Requests**: If the request is for the WordPress admin area, it is handled by a dedicated admin route.

2. **Special WordPress Requests**: If the request is for a special WordPress feature (robots.txt, favicon, feeds, trackbacks), it is handled by a dedicated special route.

3. **Laravel Custom Routes**: The framework checks if a matching route exists in your `routes/web.php` file. These routes have high priority and will be executed if they match the requested URL.

4. **WordPress Template Hierarchy**: If no custom route matches, the `FrontendController` takes over and:
   - Determines the appropriate template hierarchy based on WordPress conditional tags
   - Sequentially searches for matching Blade views in your views directory
   - Returns the first view found in the hierarchy order

5. **Fallback Handling**: If no view is found in the template hierarchy, a 404 error is returned.

For example, for an "About" page, the process would be as follows:

```php
// 1. First checks if a custom route exists in routes/web.php
Route::wp('page', 'about-us', function() {
    return view('pages.about');
});

// 2. If no route matches, FrontendController checks the template hierarchy:
// - page-about.blade.php
// - page.blade.php
// - singular.blade.php
// - index.blade.php

// 3. If no view is found, returns a 404
```

This approach offers several benefits:
- Allows fine-grained customization via Laravel routes when needed
- Maintains compatibility with WordPress template hierarchy
- Provides automatic fallback to more generic templates
- Reduces boilerplate code by automatically handling standard cases
- Ensures that all requests are handled appropriately, even special WordPress requests

⚠️ **Important**: Routes defined in `routes/web.php` always take precedence over the automatic template system. Use this advantage to:
- Add custom logic to specific pages
- Inject specific data into your views
- Handle special cases requiring processing before display

### Special WordPress Requests

The framework automatically handles special WordPress request types, including:

- **robots.txt**: Requests for `/robots.txt` are handled by WordPress's built-in `do_robots` action
- **favicon.ico**: Requests for `/favicon.ico` are handled by WordPress's built-in `do_favicon` action
- **Feeds**: RSS and Atom feed requests are handled by WordPress's built-in `do_feed` function
- **Trackbacks**: Trackback requests are handled by WordPress's built-in trackback handler

You can override these default handlers by defining your own routes for these special requests:

```php
// Override the default robots.txt handler
Route::wp('is_robots', function() {
    return response("User-agent: *\nDisallow: /wp-admin/\nAllow: /wp-admin/admin-ajax.php", 200)
        ->header('Content-Type', 'text/plain');
});

// Override the default feed handler
Route::wp('is_feed', function() {
    return response()->view('custom-feed', ['posts' => Post::latest()->take(10)->get()])
        ->header('Content-Type', 'application/rss+xml');
});
```

### Fallback Routes

When no route matches the current request, the framework automatically creates a fallback route that delegates to the `FrontendController`. This controller:

1. Determines the appropriate template based on WordPress's template hierarchy
2. Renders the corresponding Blade view
3. Returns a 404 response if no appropriate view is found

This ensures that all requests are handled appropriately, even if no explicit route is defined.