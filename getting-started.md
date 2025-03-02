# Getting Started with Pollora

Welcome to Pollora! This guide will help you get started with a working installation of Pollora, a powerful WordPress framework built on top of Laravel.

## Requirements

- PHP 8.2 or higher
- Composer
- MySQL 5.7 or higher

## Installation Methods

Pollora offers three ways to install your application:

1. Automatic installation via Composer (recommended)
2. Manual installation via Artisan commands
3. Web-based installation after environment setup

### 1. Automatic Installation

The recommended way to install Pollora is using Composer:

```bash
composer create-project pollora/pollora
```

This command will:
1. Create a new Pollora project
2. Install all dependencies
3. Automatically launch the LaunchPad setup process

During the setup, you'll be prompted for:

#### Environment Configuration (wp:env-setup)

- **Site URL**: Your site's URL (e.g., https://example.com)
- **Database Configuration**:
    - Host (default: localhost)
    - Port (default: 3306)
    - Database name
    - Username
    - Password

The system will test the database connection. If it fails, you'll have the option to retry with different credentials.

#### WordPress Installation (wp:install)

- **Site Information**:
    - Site title
    - Site description
    - Language selection (searchable list of available languages)
- **Admin Account**:
    - Username
    - Email
    - Password (minimum 8 characters)
- **Search Engine Visibility**:
    - Option to allow or prevent search engine indexing

### 2. Manual Installation via Artisan

If you prefer to run the installation steps manually, you can use the following Artisan commands:

```bash
# Configure environment
php artisan wp:env-setup

# Install WordPress
php artisan wp:install
```

These commands will guide you through the same interactive setup process as the automatic installation.

### 3. Web-based Installation

If you prefer the traditional WordPress installation interface, you can:

1. Run the environment setup:
```bash
php artisan wp:env-setup
```

2. Once the `.env` file is configured, visit your site's URL and follow the WordPress installation wizard.

## Post-Installation

After successful installation:

1. Access your WordPress admin panel at: `your-site-url/wp-admin`
2. Start customizing your site
3. Install themes and plugins as needed

Congratulations, you're ready to use Pollora now!

## WordPress Configuration

Pollora uses a `wordpress.php` configuration file that contains several important settings:

1. **WordPress Route Conditions**: Mappings between WordPress conditional tags and route URIs
2. **WordPress Authentication Keys**: Keys and salts for WordPress security
3. **Multisite Configuration**: Settings for multisite installations
4. **Database Caching**: Options for database caching
5. **WordPress Constants**: Define WordPress behavior through constants

### Publishing the Configuration File

To customize these settings, you can publish the configuration file to your application:

```bash
php artisan vendor:publish --tag=wp-config
```

This command will copy the framework's configuration file to your application's `config/` directory, allowing you to customize it according to your needs.

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

For more information on using route conditions, see the [Routing](/routing.md#wordpress-route-conditions) documentation.

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

## Troubleshooting

### Database Connection Issues
If you encounter database connection issues:
1. Verify your database credentials
2. Ensure your database server is running
3. Check if the database exists and is accessible
4. Run `php artisan wp:env-setup` to reconfigure database settings

### Installation Failed

If the WordPress installation fails:

1. Check the error message
2. Verify database permissions
3. Ensure all required PHP extensions are installed
4. Run `php artisan wp:install` to retry the installation

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

<div class="alert alert-info" role="alert"><strong>Heads up!</strong> Pollora rids WordPress of frontend responsibilites altogether, this means theme support in WordPress is dropped completely. Don't worry though, any functions you can run in vanilla WordPress you can run in Pollora!</div>
