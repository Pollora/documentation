# WordPress Configuration

This document outlines all the configuration options available in the `config/wordpress.php` configuration file of the Pollora framework.

## Overview

The WordPress configuration file (`wordpress.php`) contains various settings that control how the Pollora framework interacts with WordPress. These settings include route conditions, mail handling, plugin-specific conditions, and WordPress constants.

## Publishing the Configuration

By default, Pollora uses the configuration file located in the framework package. To customize the configuration, you can publish it to your application's config directory using the following Artisan command:

```bash
php artisan vendor:publish --tag=wordpress
```

This will create a copy of the configuration file at `config/wordpress.php` in your application. Any changes you make to this file will override the default configuration provided by the framework.

If you don't publish the configuration file, the framework will use the default configuration located at `vendor/pollora/framework/config/wordpress.php`.

## Configuration Options

### Route Conditions

The `conditions` array maps WordPress conditional functions to their route aliases. These conditions are used by the `Route::wordpress()` and `Route::wp()` methods to define routes based on WordPress conditions.

```php
'conditions' => [
    // Error and special pages
    'is_404' => ['404', 'not_found'],
    'is_search' => 'search',
    'is_paged' => 'paged',
    
    // Homepage and blog index
    'is_front_page' => ['/', 'front'],
    'is_home' => ['home', 'blog'],
    
    // And many more...
],
```

Each key is a WordPress conditional function (like `is_404`, `is_search`, etc.), and the value is either a string or an array of strings representing the route aliases.

See [Routing](routing.md) for more information about routing in Pollora.

#### Available Conditions

- **Error and Special Pages**:
  - `is_404`: Matches 404 (not found) pages
  - `is_search`: Matches search result pages
  - `is_paged`: Matches paginated pages

- **Homepage and Blog Index**:
  - `is_front_page`: Matches the site's front page
  - `is_home`: Matches the blog posts index

- **Specific Template**:
  - `is_page_template`: Matches pages using specific templates

- **Custom Post Type Hierarchy**:
  - `is_singular`: Matches any single post of any post type
  - `is_single`: Matches single posts
  - `is_attachment`: Matches attachment pages
  - `is_post_type_archive`: Matches post type archive pages
  - `is_archive`: Matches any archive page

- **Taxonomies**:
  - `is_category`: Matches category archive pages
  - `is_tag`: Matches tag archive pages
  - `is_tax`: Matches custom taxonomy archive pages

- **Time Hierarchy**:
  - `is_date`: Matches date archive pages
  - `is_year`: Matches year archive pages
  - `is_month`: Matches month archive pages
  - `is_day`: Matches day archive pages
  - `is_time`: Matches time archive pages

- **Other Conditions**:
  - `is_author`: Matches author archive pages
  - `is_sticky`: Matches sticky posts
  - `is_subpage`: Matches subpages

#### Plugin Conditions

The `plugin_conditions` array contains plugin-specific conditions for routing. Currently, it includes conditions for WooCommerce.

```php
'plugin_conditions' => [
    'woocommerce' => [
        // WooCommerce conditions
        'is_shop' => 'shop',
        'is_product' => 'product',
        // And more...
    ],
],
```

##### WooCommerce Conditions

- `is_shop`: Matches the WooCommerce shop page
- `is_product`: Matches single product pages
- `is_cart`: Matches the cart page
- `is_checkout`: Matches the checkout page
- `is_account_page`: Matches the account page
- `is_product_category`: Matches product category pages
- `is_product_tag`: Matches product tag pages
- `is_wc_endpoint_url`: Matches WooCommerce endpoint URLs


### Mail Handling

The `enable_mail_handling` option controls whether the Pollora framework should handle WordPress mail functionality.

```php
'enable_mail_handling' => true,
```

- When set to `true` (default), Pollora overrides the `wp_mail` function to use Laravel's mail system.
- When set to `false`, WordPress will use its native mail handling.

### WordPress Constants

The `constants` array defines WordPress constants that will be automatically defined by the framework. These constants are processed in the `Bootstrap::defineWordPressConstants()` method, where each key-value pair is transformed into a PHP constant.

```php
'constants' => [
    // WordPress authentication keys and salts
    'auth_key' => env('AUTH_KEY'),
    'secure_auth_key' => env('SECURE_AUTH_KEY'),
    'logged_in_key' => env('LOGGED_IN_KEY'),
    'nonce_key' => env('NONCE_KEY'),
    'auth_salt' => env('AUTH_SALT'),
    'secure_auth_salt' => env('SECURE_AUTH_SALT'),
    'logged_in_salt' => env('LOGGED_IN_SALT'),
    'nonce_salt' => env('NONCE_SALT'),

    // WordPress multisite configuration
    'wp_allow_multisite' => env('WP_ALLOW_MULTISITE'),
    'multisite' => env('MULTISITE'),
    'subdomain_install' => env('SUBDOMAIN_INSTALL'),
    'domain_current_site' => env('DOMAIN_CURRENT_SITE'),
    'path_current_site' => env('PATH_CURRENT_SITE'),
    'site_id_current_site' => env('SITE_ID_CURRENT_SITE'),
    'blog_id_current_site' => env('BLOG_ID_CURRENT_SITE'),
    
    // Additional constants can be added here
    // 'wp_debug' => true,
],
```

#### How Constants Work

All entries in the `constants` array are automatically transformed into WordPress constants by the framework. The process works as follows:

1. Each key in the array is converted to uppercase
2. The key-value pair is queued as a constant
3. Constants are applied during the WordPress bootstrap process

This means you can add any WordPress constant to this array, not just the ones shown above. For example, you could add:

```php
'constants' => [
    // ... existing constants
    'wp_debug' => true,
    'disallow_file_edit' => true,
    'autosave_interval' => 160,
    // ... any other WordPress constant
],
```

#### Common WordPress Constants

Here are some common WordPress constants you might want to add:

- `wp_debug`: Enable WordPress debug mode
- `disallow_file_edit`: Disable the file editor in the WordPress admin
- `disallow_file_mods`: Disable file modifications in WordPress
- `wp_post_revisions`: Control the number of post revisions to keep
- `autosave_interval`: Set the autosave interval in seconds

## Usage Examples

### Controlling Mail Handling

To disable Pollora's mail handling and use WordPress native mail:

```php
// In your wordpress.php config file
'enable_mail_handling' => false,
```

### Adding Custom WordPress Constants

To add custom WordPress constants:

```php
// In your wordpress.php config file
'constants' => [
    // ... existing constants
    'wp_memory_limit' => '256M',
    'wp_max_memory_limit' => '512M',
],
```
