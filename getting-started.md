# Getting Started with Pollora

Welcome to Pollora! This guide will help you get a working installation up and running. Pollora is a WordPress framework built on top of Laravel — it replaces WordPress's frontend templating with Laravel's Blade engine while keeping WordPress's full backend (admin, database, plugins).

## Requirements

- PHP 8.3 or higher
- Composer 2.x
- MySQL 5.7+ / MariaDB 10.3+ / SQLite
- Node.js and NPM (for theme asset bundling)
- [DDEV](https://ddev.readthedocs.io) (optional, for a ready-made local environment)

## Current release

Pollora's version numbers follow the Laravel release it is built on, so the
current line is **13.32**, built on Laravel 13.32.

Two packages carry that number and they are tagged independently:

| Package | Current | What it is |
|---|---|---|
| `pollora/pollora` | `v13.32.0-beta.4` | the skeleton — what `create-project` installs |
| `pollora/framework` | `v13.32.0-beta.6` | the framework the skeleton requires, at `^13.32@beta` |

So a fresh install gives you the latest skeleton **and** the latest framework
beta, whichever numbers those happen to be.

Because these are pre-releases, Composer never selects them by default: a
plain `composer create-project pollora/pollora` still installs the last stable
release, **v13.4.0**. The commands below ask for the beta explicitly.

## Installation Methods

Pollora offers two ways to create a project:

1. The Pollora CLI (recommended)
2. Composer `create-project`

Both end up running the same interactive setup, which you can also [run by hand](#running-the-setup-manually).

### 1. Pollora CLI

Install the CLI globally once:

```bash
composer global require pollora/cli
```

Make sure Composer's global `vendor/bin` directory is in your `PATH` — `composer global config bin-dir --absolute` prints it. Then create a project:

```bash
pollora new example-app
```

Or let the CLI provision a full local environment with DDEV (recommended):

```bash
pollora new example-app --ddev
```

With `--ddev`, the CLI configures DDEV (WordPress project type, PHP 8.4, MariaDB 10.11), starts it, installs the project inside the container, writes the database credentials to `.env`, and runs the WordPress installation. Your site is then available at `https://example-app.ddev.site`.

#### CLI options

| Option | Description |
|---|---|
| `--ddev` | Set up the project with DDEV |
| `--force`, `-f` | Force install even if the directory already exists |
| `--git` | Initialize a Git repository |
| `--branch=NAME` | Branch name for the new repository (default: `main`) |
| `--ver=VERSION` | Install a specific version or constraint (e.g. `13.32.0-beta.4`, `^13.32@beta`) |
| `--stable` | Install the latest stable release instead of the latest pre-release |

`pollora new` installs the latest release **including pre-releases**, so you get the current beta. Pass `--stable` to stay on the last stable release, or `--ver` to pin an exact version.

### 2. Composer create-project

```bash
composer create-project "pollora/pollora:^13.32@beta" example-app
```

The `@beta` flag is what allows Composer to pick the current pre-release. To install the last stable release instead, drop the constraint:

```bash
composer create-project pollora/pollora example-app
```

Either command will:

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
- **Theme**: the name of the theme generated from `pollora/theme-default` (defaults to `default`)

## Running the setup manually

If you prefer to run the installation steps yourself — or need to re-run them — use the following Artisan commands:

```bash
# Configure environment
php artisan pollora:env:setup

# Install WordPress
php artisan pollora:install
```

These commands guide you through the same interactive setup as the automatic installation.

### Non-Interactive Installation

For automated deployments, CI/CD pipelines, or scripted setups, you can bypass the interactive prompts by passing all required options directly:

```bash
php artisan pollora:install \
    --title="My Site" \
    --description="A Pollora-powered site" \
    --admin-user=admin \
    --admin-email=admin@example.com \
    --admin-password=secretpassword \
    --locale=en_US \
    --public=true \
    --theme=default
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
| `--theme` | Name of the theme to generate (defaults to `default`) |
| `--install` | Suppress informational output for automated runs |

Any option that is omitted will trigger its corresponding interactive prompt. This means you can mix CLI options and prompts — for example, provide the title and admin credentials via options while being prompted for language selection.

## Web-based Installation

If you prefer the traditional WordPress installation interface, you can:

1. Run the environment setup:
```bash
php artisan pollora:env:setup
```

2. Once the `.env` file is configured, visit your site's URL and follow the WordPress installation wizard.

## Post-Installation Verification

After successful installation:

1. Start the development server: `php artisan serve` (or, with DDEV, just open `https://example-app.ddev.site`)
2. Access your site at the configured URL
3. Access your WordPress admin panel at: `your-site-url/wp-admin`
4. Verify you can log in with the admin credentials you configured

For post-installation configuration (WordPress settings, environment variables, development environments), see the [Configuration](installation.md) guide.

<div class="alert alert-info" role="alert"><strong>Heads up!</strong> Pollora rids WordPress of frontend responsibilites altogether, this means theme support in WordPress is dropped completely. Don't worry though, any functions you can run in vanilla WordPress you can run in Pollora!</div>
