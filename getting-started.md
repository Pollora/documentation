# Getting Started with Pollora

Welcome to Pollora! This guide will help you get a working installation up and running. Pollora is a WordPress framework built on top of Laravel — it replaces WordPress's frontend templating with Laravel's Blade engine while keeping WordPress's full backend (admin, database, plugins).

## Requirements

- PHP 8.3 or higher
- Composer 2.x
- MySQL 5.7+ / MariaDB 10.3+ / SQLite
- Node.js and NPM (for theme asset bundling)

## Installation Methods

Pollora offers three ways to install your application:

1. Automatic installation via Composer (recommended)
2. Manual installation via Artisan commands
3. Web-based installation after environment setup

### 1. Automatic Installation

The recommended way to install Pollora is using Composer:

```bash
composer create-project pollora/pollora example-app
```

This command will:
1. Create a new Pollora project
2. Install all dependencies
3. Automatically launch the LaunchPad setup process

During the setup, you'll be prompted for:

#### Environment Configuration (pollora:env:setup)

- **Site URL**: Your site's URL (e.g., https://example.com)
- **Database Configuration**:
    - Host (default: localhost)
    - Port (default: 3306)
    - Database name
    - Username
    - Password

The system will test the database connection. If it fails, you'll have the option to retry with different credentials.

#### WordPress Installation (pollora:install)

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
php artisan pollora:env:setup

# Install WordPress
php artisan pollora:install
```

These commands will guide you through the same interactive setup process as the automatic installation.

#### Non-Interactive Installation

For automated deployments, CI/CD pipelines, or scripted setups, you can bypass the interactive prompts by passing all required options directly:

```bash
php artisan pollora:install \
    --title="My Site" \
    --description="A Pollora-powered site" \
    --admin-user=admin \
    --admin-email=admin@example.com \
    --admin-password=secretpassword \
    --locale=en_US \
    --public=true
```

Available options:

| Option | Description |
|---|---|
| `--title` | Site title |
| `--description` | Site description |
| `--admin-user` | Admin username |
| `--admin-email` | Admin email address |
| `--admin-password` | Admin password (min. 8 characters) |
| `--locale` | Site locale (e.g. `en_US`, `fr_FR`) |
| `--public` | Allow search engine indexing (`true` or `false`) |
| `--install` | Suppress informational output for automated runs |

Any option that is omitted will trigger its corresponding interactive prompt. This means you can mix CLI options and prompts — for example, provide the title and admin credentials via options while being prompted for language selection.

### 3. Web-based Installation

If you prefer the traditional WordPress installation interface, you can:

1. Run the environment setup:
```bash
php artisan pollora:env:setup
```

2. Once the `.env` file is configured, visit your site's URL and follow the WordPress installation wizard.

## Post-Installation Verification

After successful installation:

1. Start the development server: `php artisan serve`
2. Access your site at the configured URL
3. Access your WordPress admin panel at: `your-site-url/wp-admin`
4. Verify you can log in with the admin credentials you configured

For post-installation configuration (WordPress settings, environment variables, development environments), see the [Configuration](installation.md) guide.

<div class="alert alert-info" role="alert"><strong>Heads up!</strong> Pollora rids WordPress of frontend responsibilites altogether, this means theme support in WordPress is dropped completely. Don't worry though, any functions you can run in vanilla WordPress you can run in Pollora!</div>
