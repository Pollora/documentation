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

## Environment Variables

The installation process will create a `.env` file with your configuration. Key variables include:

```env
APP_URL=your-site-url
DB_CONNECTION=mysql
DB_HOST=your-database-host
DB_PORT=3306
DB_DATABASE=your-database-name
DB_USERNAME=your-database-user
DB_PASSWORD=your-database-password
```

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
