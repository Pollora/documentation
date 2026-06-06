# Configuration

- [Initial Configuration](#initial-configuration)
    - [Environment Based Configuration](#environment-based-configuration)
    - [Databases & Migrations](#databases-and-migrations)
    - [Directory Configuration](#directory-configuration)
- [WordPress Configuration](#wordpress-configuration)
    - [Publishing the Configuration File](#publishing-the-configuration-file)
    - [Customizing Route Conditions](#customizing-route-conditions)
    - [WordPress Authentication Keys](#wordpress-authentication-keys)
    - [Multisite Configuration](#multisite-configuration)
    - [Database Caching](#database-caching)
    - [WordPress Constants](#wordpress-constants)
- [Environment Variables](#environment-variables)
- [Development Environments](#development-environments)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

<a name="initial-configuration"></a>
## Initial Configuration

All configuration files for Pollora are located in the `config` directory. Feel free to familiarize yourself with the options as each one is well-documented.

Out of the box, Pollora requires minimal configuration. However, it might be worthwhile to review the `config/app.php` file and its accompanying documentation to tailor settings like `timezone` and `locale` to your needs.

<a name="environment-based-configuration"></a>
### Environment Based Configuration

Configuration values in Pollora can vary depending on the environment (local vs. production). These values are usually defined in the `.env` file at your application's root.

For security reasons, never commit your `.env` file to source control. Different developers or servers might need different configurations, and exposing sensitive credentials would pose a significant risk.

> **Note**  
> For a deep dive into the `.env` file and environment configurations, peruse the full [configuration documentation](https://laravel.com/docs/13.x/configuration#environment-configuration).

<a name="databases-and-migrations"></a>
### Databases & Migrations

With your Pollora application ready, you might want to store data. By default, the application's `.env` configuration indicates that Pollora will interact with a MySQL database. If you're on macOS, installing MySQL, Postgres, or Redis is a breeze with [DBngin](https://dbngin.com/).

If you'd rather not use MySQL or Postgres, [SQLite](https://www.sqlite.org/index.html) is a lightweight alternative. To begin, create a SQLite database in the `database` directory:

```shell
touch database/database.sqlite
```

Then, adjust your `.env` file to utilize Pollora's `sqlite` database driver. Unneeded configurations can be removed.

Finally, run your application's [database migrations](https://laravel.com/docs/13.x/migrations) to establish your database tables:

```shell
php artisan migrate
```

<a name="directory-configuration"></a>
### Directory Configuration

Always serve Pollora from the root of the "web directory" set for your server. Avoid serving Pollora from a subdirectory as it could inadvertently expose sensitive files.

<a name="wordpress-configuration"></a>
## WordPress Configuration

Pollora uses a `wordpress.php` configuration file that contains several important settings:

1. **WordPress Route Conditions**: Mappings between WordPress conditional tags and route URIs
2. **WordPress Authentication Keys**: Keys and salts for WordPress security
3. **Multisite Configuration**: Settings for multisite installations
4. **Database Caching**: Options for database caching
5. **WordPress Constants**: Define WordPress behavior through constants

<a name="publishing-the-configuration-file"></a>
### Publishing the Configuration File

To customize these settings, you can publish the configuration file to your application:

```bash
php artisan vendor:publish --tag=wp-config
```

This command will copy the framework's configuration file to your application's `config/` directory, allowing you to customize it according to your needs.

<a name="customizing-route-conditions"></a>
### Customizing Route Conditions

WordPress route conditions are particularly useful for defining routes that match WordPress conditional functions. You can add your own conditions or replace existing ones:

```php
// config/wordpress.php
return [
    'conditions' => [
        // Add your custom conditions
        'is_custom_post_type' => 'custom-post-type',
        
        // Override existing conditions
        'is_page' => ['page', 'static-page'],
    ],
    // ... other configuration options
];
```

For more information on using route conditions, see the [Routing](routing.md#wordpress-route-conditions) documentation.

<a name="wordpress-authentication-keys"></a>
### WordPress Authentication Keys

The configuration file also contains WordPress authentication keys and salts, which are essential for your application's security:

```php
// config/wordpress.php
return [
    // ... other options
    
    // WordPress authentication keys and salts
    'auth_key' => env('AUTH_KEY'),
    'secure_auth_key' => env('SECURE_AUTH_KEY'),
    'logged_in_key' => env('LOGGED_IN_KEY'),
    'nonce_key' => env('NONCE_KEY'),
    'auth_salt' => env('AUTH_SALT'),
    'secure_auth_salt' => env('SECURE_AUTH_SALT'),
    'logged_in_salt' => env('LOGGED_IN_SALT'),
    'nonce_salt' => env('NONCE_SALT'),
];
```

These values are typically defined in your `.env` file during installation. If you need to generate new keys, you can use the [WordPress Secret Key Generator](https://api.wordpress.org/secret-key/1.1/salt/).

<a name="multisite-configuration"></a>
### Multisite Configuration

If you want to enable WordPress multisite functionality, you can configure the following parameters:

```php
// config/wordpress.php
return [
    // ... other options
    
    // WordPress multisite configuration
    'wp_allow_multisite' => env('WP_ALLOW_MULTISITE'),
    'multisite' => env('MULTISITE'),
    'subdomain_install' => env('SUBDOMAIN_INSTALL'),
    'domain_current_site' => env('DOMAIN_CURRENT_SITE'),
    'path_current_site' => env('PATH_CURRENT_SITE'),
    'site_id_current_site' => env('SITE_ID_CURRENT_SITE'),
    'blog_id_current_site' => env('BLOG_ID_CURRENT_SITE'),
];
```

Make sure to define these variables in your `.env` file if you enable multisite functionality.

<a name="database-caching"></a>
### Database Caching

Pollora also supports caching WordPress database queries:

```php
// config/wordpress.php
return [
    // ... other options
    
    // Database caching
    'caching' => env('DB_CACHE'),
];
```

Enable this option by setting `DB_CACHE=true` in your `.env` file to improve your application's performance.

<a name="wordpress-constants"></a>
### WordPress Constants

You can define additional WordPress constants in your configuration file. These constants control various aspects of WordPress behavior:

```php
// config/wordpress.php
return [
    // ... other options
    
    // WordPress constants
    'constants' => [
        'WP_AUTO_UPDATE_CORE' => false,
        'DISALLOW_FILE_MODS' => true,
        'DISALLOW_FILE_EDIT' => true,
        'DISABLE_WP_CRON' => true,
        'WP_POST_REVISIONS' => 5,
        // Add your custom constants here
    ],
];
```

By default, Pollora sets several constants for security and performance:

- `WP_AUTO_UPDATE_CORE`: Disables WordPress core auto-updates
- `DISALLOW_FILE_MODS`: Prevents plugin and theme installations from the admin
- `DISALLOW_FILE_EDIT`: Disables the built-in file editor
- `DISABLE_WP_CRON`: Disables the WordPress cron system (use Laravel's scheduler instead)
- `WP_POST_REVISIONS`: Limits the number of post revisions stored

You can override these defaults or add your own constants in your application's configuration file.

<a name="environment-variables"></a>
## Environment Variables

The installation process will create a `.env` file with your configuration. Key variables include:

```env
# Application settings
APP_URL=your-site-url
APP_ENV=local
APP_DEBUG=true

# Database settings
DB_CONNECTION=mysql
DB_HOST=your-database-host
DB_PORT=3306
DB_DATABASE=your-database-name
DB_USERNAME=your-database-user
DB_PASSWORD=your-database-password

# WordPress authentication keys and salts
AUTH_KEY=your-auth-key
SECURE_AUTH_KEY=your-secure-auth-key
LOGGED_IN_KEY=your-logged-in-key
NONCE_KEY=your-nonce-key
AUTH_SALT=your-auth-salt
SECURE_AUTH_SALT=your-secure-auth-salt
LOGGED_IN_SALT=your-logged-in-salt
NONCE_SALT=your-nonce-salt

# WordPress multisite configuration (if needed)
# WP_ALLOW_MULTISITE=true
# MULTISITE=true
# SUBDOMAIN_INSTALL=false
# DOMAIN_CURRENT_SITE=example.com
# PATH_CURRENT_SITE=/
# SITE_ID_CURRENT_SITE=1
# BLOG_ID_CURRENT_SITE=1

# WordPress database caching
DB_CACHE=false
```

During installation, the WordPress authentication keys and salts are automatically generated for security. You can regenerate these keys at any time using the [WordPress Secret Key Generator](https://api.wordpress.org/secret-key/1.1/salt/).

<a name="development-environments"></a>
## Development Environments

Pollora automatically detects and configures itself for common development environments:

### DDEV
When using DDEV, the system will automatically:
- Detect DDEV configuration
- Use appropriate database settings
- Set the correct site URL

### Laradock
With Laradock, the system will:
- Use Laradock's database configuration
- Configure appropriate host settings

<a name="troubleshooting"></a>
## Troubleshooting

### Database Connection Issues
If you encounter database connection issues:
1. Verify your database credentials
2. Ensure your database server is running
3. Check if the database exists and is accessible
4. Run `php artisan pollora:env-setup` to reconfigure database settings

### Installation Failed

If the WordPress installation fails:

1. Check the error message
2. Verify database permissions
3. Ensure all required PHP extensions are installed
4. Run `php artisan pollora:install` to retry the installation

<a name="next-steps"></a>
## Next Steps

With your Pollora project configured, here are the recommended next steps:

- [Routing](routing.md) — Learn WordPress routing with `Route::wp()`
- [Theming](theming.md) — Create your first theme with Blade templates
- [Post Types](post-types.md) — Register custom post types with PHP attributes

For Laravel-specific concepts, see the [Laravel documentation](https://laravel.com/docs/13.x).
