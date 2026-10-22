# Local Development Environments

- [Choose Your Environment](#choose-your-environment)
- [The Three Conditions](#the-three-conditions)
- [Environments](#environments)
    - [`php artisan serve`](#php-artisan-serve)
    - [Laravel Valet](#laravel-valet)
    - [Laravel Herd](#laravel-herd)
    - [MAMP](#mamp)
    - [DDEV](#ddev)
    - [Apache with PHP-FPM](#apache-with-php-fpm)
    - [nginx with PHP-FPM](#nginx-with-php-fpm)
    - [Laradock](#laradock)
    - [Update an Existing Project](#update-an-existing-project)
- [The Database](#the-database)
- [Diagnosing a Setup](#diagnosing-a-setup)
- [After the Installation](#after-the-installation)

<a name="choose-your-environment"></a>
## Choose Your Environment

Pollora runs on any server that can serve a PHP application from a document root. The table lists what was set up and run for each tool, so you can see what has been checked and what has not. It is not a ranking: pick the tool you already use.

| Environment | Document root | `APP_URL` | Known trap | Tested |
| --- | --- | --- | --- | --- |
| [`php artisan serve`](#php-artisan-serve) | `public/`, set by Laravel | `http://localhost:8000` | Nothing to configure | macOS 15, PHP 8.4 |
| [Laravel Valet](#laravel-valet) | `public/`, set by Valet | `http://my-project.test`, or `https://` after `valet secure` | A default site answers every unknown `.test` domain | macOS 15, PHP 8.4, HTTP and HTTPS (Valet 4.12) |
| [Laravel Herd](#laravel-herd) | `public/`, set by Herd | `http://my-project.test` | `wp-login.php` and `wp-admin` answer 404 without `LocalValetDriver.php` | macOS 15, PHP 8.4 (Herd 1.30), with the driver |
| [MAMP](#mamp) | Set it to `public/` yourself | `http://localhost:8888` | A document root on the project folder serves `.env`; a PHP version older than the one that ran `composer` stops the site | macOS 15, PHP 8.3.30 (MAMP 7.4) |
| [DDEV](#ddev) | `public/`, set by `ddev config --docroot=public` | `https://my-project.ddev.site` | The database host is `db`, not `localhost` | Linux, nginx-fpm and Apache-fpm. Not tested on macOS or Windows |
| [Apache with PHP-FPM](#apache-with-php-fpm) | Set it to `public/` yourself | The address of the virtual host | A document root on the project folder serves `.env` | macOS 15 (Homebrew, Apache 2.4, PHP 8.4); Linux (DDEV) |
| [nginx with PHP-FPM](#nginx-with-php-fpm) | Set it to `public/` yourself | The address of the server block | See [Server Configuration](server-configuration.md) | Linux (DDEV, nginx-fpm). Not tested on macOS |
| [Laradock](#laradock) | Not covered | Not covered | Not covered | Not tested |

<a name="the-three-conditions"></a>
## The Three Conditions

Whatever the server, three things must agree:

1. **The document root is the project's `public/` folder.** `public/` holds the front controller (`index.php`), WordPress core (`cms/`) and the content folder (`content/`). A server pointed at the project folder answers a 404 on the home page and, worse, serves `/.env` as a text file with the application key and the database password.
2. **The site is opened at the same address as `APP_URL`.** WordPress sends `/cms/wp-admin/` to the host written in `APP_URL`. Open the site on another address (`127.0.0.1` instead of `my-project.test`) and the admin redirects you to the `APP_URL` host, where you are not logged in.
3. **`APP_URL` names a scheme**, `http://` or `https://`. Pollora reads `APP_URL=my-project.test` as `http://my-project.test`, but everything else that reads the setting (Laravel's URL helpers in the console, mails, a queue worker) gets the bare host. `php artisan pollora:doctor` warns about it.

<a name="environments"></a>
## Environments

<a name="php-artisan-serve"></a>
### `php artisan serve`

Nothing to configure. In `.env`:

```dotenv
APP_URL=http://localhost:8000
```

```bash
php artisan serve
```

Open `http://localhost:8000/cms/wp-admin/`: the login page is shown. PHP's built-in server (`php -S localhost:8000 -t public`) gives the same result.

<a name="laravel-valet"></a>
### Laravel Valet

[Valet](https://laravel.com/docs/valet) names the site after the project folder, or after the name given to `valet link`.

```bash
cd my-project
valet link
```

```dotenv
APP_URL=http://my-project.test
```

To serve the site over HTTPS, run `valet secure` and write the scheme in `.env`:

```bash
valet secure
```

```dotenv
APP_URL=https://my-project.test
```

Open `http://my-project.test/cms/wp-admin/`: the login page is shown.

Valet runs the site through its own `server.php`, not `public/index.php`. Pollora handles that since 13.38.1: [update the framework](#update-an-existing-project) if the home page redirects to itself.

> **Note**  
> A default site (`valet default`, stored in `~/.config/valet/config.json`) answers for every `.test` domain Valet does not know. If the site shown is not your project, run `valet which` in the project folder to see which site Valet serves.

<a name="laravel-herd"></a>
### Laravel Herd

Add the project folder to [Herd](https://herd.laravel.com), then write the address Herd gives the site in `.env`:

```dotenv
APP_URL=http://my-project.test
```

Herd sends every `.php` URL to the front controller. WordPress's own entry points (`wp-login.php`, `wp-admin/*.php`, `wp-cron.php`, `xmlrpc.php`) never run: they answer the site's 404, and there is no way to log in. The skeleton ships a `LocalValetDriver.php` at the project root that fixes this. Valet and Herd both read a driver of that name from the project. It hands an existing `.php` file under `public/` to PHP, as a web server with a document root does, and never one under `uploads/`. Everything else goes to the front controller as before.

A project created with the 13.38.1 skeleton or later has the file. For an existing project, copy [`LocalValetDriver.php`](https://github.com/Pollora/pollora/blob/main/LocalValetDriver.php) from the skeleton to the project root.

Open `http://my-project.test/cms/wp-admin/`: the login page is shown.

<a name="mamp"></a>
### MAMP

[MAMP](https://www.mamp.info) uses Apache, and its document root is a setting you must change.

1. Preferences › Server › **Document root**: choose the `public/` folder of the project, not the project folder.
2. Preferences › PHP: choose PHP 8.3 or 8.4.
3. In `.env`:

```dotenv
APP_URL=http://localhost:8888
```

MAMP's MySQL user is `root` with the password `root`. See [The Database](#the-database).

The PHP version that serves the site must be at least the one `composer` resolved the dependencies for. A project whose lock was resolved on PHP 8.4 and that MAMP serves with its default PHP 8.3 stops with `Your Composer dependencies require a PHP version ">= 8.4.1". You are running 8.3.30.` The skeleton has resolved its lock for PHP 8.3 since 13.38.1. For an older project, tell Composer to resolve for PHP 8.3, then update the Symfony packages:

```bash
composer config platform.php 8.3.0
composer update "symfony/*" --with-all-dependencies
```

Open `http://localhost:8888/cms/wp-admin/`: the login page is shown.

> [!WARNING]
> Do not point MAMP at the project folder. The site answers a 404 and `.env` is served as a text file. If Site Health reports it (Tools › Site Health, *Document root*), fix the document root, then change `APP_KEY` and the passwords in `.env`.

<a name="ddev"></a>
### DDEV

[DDEV](https://ddev.readthedocs.io) runs the site in containers. The Pollora CLI sets the whole thing up:

```bash
pollora new example-app --ddev
```

With `--ddev`, the CLI runs `ddev config --project-type=wordpress --docroot=public --php-version=8.4 --database=mariadb:10.11 --disable-settings-management`, starts the project, installs Pollora inside the container, writes the connection settings to `.env`, and runs the WordPress installation:

```dotenv
APP_URL=https://example-app.ddev.site
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=db
DB_USERNAME=db
DB_PASSWORD=db
```

The site is at `https://example-app.ddev.site`: open `https://example-app.ddev.site/cms/wp-admin/` for the login page. Run Artisan commands inside the container, for example `ddev exec php artisan pollora:doctor`.

Inside a DDEV container (`IS_DDEV_PROJECT=true`), the questions `pollora:env:setup` asks are pre-filled from DDEV: the site URL from `DDEV_PRIMARY_URL`, the port, and the database host (`db`), name and user (`PGDATABASE` and `PGUSER`, `db` when missing). The password is always asked; type `db`, the DDEV default.

> **Note**  
> For a project that already exists, configure DDEV with the same `ddev config` options as above, then run `ddev start`.

<a name="apache-with-php-fpm"></a>
### Apache with PHP-FPM

Point the document root of the virtual host to the `public/` folder of the project, and write the address of the virtual host in `APP_URL`. The skeleton's `public/.htaccess` sends requests to the front controller. [Server Configuration](server-configuration.md) covers the document root, the directory browsing protection and HTTPS behind a proxy.

> [!WARNING]
> Do not point Apache at the project folder: `.env` is then served as a text file.

<a name="nginx-with-php-fpm"></a>
### nginx with PHP-FPM

Point the document root of the server block to the `public/` folder and write the address of the server in `APP_URL`. [Server Configuration](server-configuration.md) has the configuration; it is not repeated here.

<a name="laradock"></a>
### Laradock

The framework recognizes a Laradock environment (`LARADOCK_PHP_VERSION` is set) and pre-fills the questions of `pollora:env:setup` from its `MYSQL_*` variables. Laradock has not been run for this documentation: nothing else is claimed about it.

<a name="update-an-existing-project"></a>
### Update an Existing Project

Several fixes for local servers shipped with 13.38.1. In a project created before it:

```bash
composer update pollora/framework
```

Then, depending on your server, add the [`LocalValetDriver.php`](#laravel-herd) file (Herd and Valet) or the `platform.php` setting [shown for MAMP](#mamp).

<a name="the-database"></a>
## The Database

Pollora needs MySQL 5.7+ or MariaDB 10.3+. WordPress is only loaded when Laravel's default connection uses the `mysql` driver, so SQLite and PostgreSQL are not supported. Set the connection in `.env` with Laravel's names (`DB_CONNECTION=mysql`, `DB_HOST`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`).

An empty password is accepted: the MySQL `root` user installed with Homebrew has none, so leave the password empty when `pollora:install` asks for it. MAMP's MySQL user is `root` with the password `root`. On macOS, [DBngin](https://dbngin.com/) installs MySQL or MariaDB. DDEV provides its own database; see [DDEV](#ddev).

<a name="diagnosing-a-setup"></a>
## Diagnosing a Setup

Start with the doctor:

```bash
php artisan pollora:doctor
```

For a local setup, four of its checks matter:

| Check | What it catches | Runs in |
| --- | --- | --- |
| Environment file | `.env` names Pollora does not read (`DB_NAME`, `DB_USER`, `WP_HOME`…), MySQL settings on a sqlite connection | Console, Site Health |
| Application URL | `APP_URL` without a scheme | Console, Site Health |
| Configuration and route caches | Edits to `.env` or `routes/` ignored because the configuration or the routes are cached outside production | Console, Site Health |
| Document root | The web server serves `.env`: it exposes the project folder, not `public/` | Site Health only |

The *Document root* check needs a web request, so it appears in WordPress's Tools › Site Health, not in the console. The doctor has more checks, for the theme, the patches, the blocks and the other modules: see [Diagnosing a project](dashboard.md#diagnosing-a-project-polloradoctor).

If the site does not behave, find your symptom:

| Symptom | Cause | Fix |
| --- | --- | --- |
| Blank page, or `Your Composer dependencies require a PHP version ">= 8.4.1"` | The PHP that serves the site (MAMP, Herd) is older than the one that ran `composer` | Compare `php -v` with the PHP chosen in the server's settings; use 8.3 or 8.4, or see [MAMP](#mamp) |
| 404 on the home page, and `/.env` is readable | The document root is the project folder | Point it to `public/`, then change `APP_KEY` and the passwords |
| Redirect to another address, or logged out of the admin | The site is opened on another address than `APP_URL` | Open the site at the address written in `APP_URL` |
| The home page redirects to itself (Valet, Herd) | Framework older than 13.38.1 | `composer update pollora/framework` |
| `/cms/wp-admin/` answers 404 (Valet, Herd) | Framework older than 13.38.1 | `composer update pollora/framework` |
| `wp-login.php` and `wp-admin` answer 404, no login (Herd) | Herd sends every `.php` URL to the front controller | Copy [`LocalValetDriver.php`](#laravel-herd) to the project root |
| Artisan commands crash, or the admin answers 404, with `APP_URL=my-project.test` | `APP_URL` has no scheme | Write `http://` or `https://` in `APP_URL` |
| The site shown is not your project (Valet, Herd) | A default site answers every unknown `.test` domain | `valet which`, or check the site in Herd's settings |

If it still fails, these are what we need to see: `php -v` and the PHP version chosen in your server's settings, the output of `php artisan pollora:doctor`, and what `curl -I` answers for `/` and `/cms/wp-login.php` (status line and `Location`). When the page is blank, add the last lines of the server's error log and of `storage/logs/laravel.log`.

<a name="after-the-installation"></a>
## After the Installation

Check that the site works:

1. The home page answers `200`: `curl -I http://my-project.test/` (use your `APP_URL`).
2. `/cms/wp-admin/` shows the WordPress login page, and the admin credentials you chose during the installation log you in.
3. `php artisan pollora:doctor` ends without an error.

Then:

- [Configuration](installation.md) — environment variables and WordPress settings
- [Server Configuration](server-configuration.md) — document root, directory browsing protection and HTTPS
- [Theming](theming.md) — create your first theme
