# Environment Management

A Pollora project has one source of truth for its environment: Laravel's `.env` file. `APP_ENV` and `APP_DEBUG` there decide the WordPress environment type, `WP_DEBUG`, and how the framework caches. This page lists what to set in `local`, `staging` and `production`, and why.

- [What to set per environment](#what-to-set-per-environment)
- [What each setting changes](#what-each-setting-changes)
- [A production `.env`](#a-production-env)
- [Reading the environment in code](#reading-the-environment-in-code)
- [Checking a deployment](#checking-a-deployment)

## What to set per environment

The `local` column is the skeleton's `.env.example`. The `staging` and `production` columns are what to set: staging should behave like production, so it differs only by `APP_ENV`.

| Variable | local | staging | production |
| --- | --- | --- | --- |
| `APP_ENV` | `local` | `staging` | `production` |
| `APP_DEBUG` | `true` | `false` | `false` |
| `APP_URL` | `http://localhost` | the staging URL | the public URL |
| `QUEUE_CONNECTION` | `database` | a connection a worker runs | a connection a worker runs |
| `HOOKS_ASYNC_DRIVER` | `auto` (add `sync` to run handlers at once) | `auto` | `auto` |
| `CACHE_STORE` | `database` | your choice | your choice |

Where each value comes from:

- `APP_ENV=local`, `APP_DEBUG=true`, `QUEUE_CONNECTION=database` and `CACHE_STORE=database` are the skeleton's `.env.example`.
- `APP_ENV=staging` and `APP_ENV=production` are the values the framework recognises when it derives the WordPress environment type (`src/WordPress/Bootstrap.php`, see [WordPress environment type](wordpress-config.md#wordpress-environment-type)). Any other name falls back to `production` on the WordPress side.
- `APP_DEBUG=false` follows from what debug mode does, described below. The framework has no environment where debug is turned on or off by the environment name: it reads `APP_DEBUG` only.
- `QUEUE_CONNECTION` and `CACHE_STORE` for staging and production are choices, not framework requirements. The `sync` and `null` queue connections are the ones to avoid, see [the queue](#the-queue).

## What each setting changes

### `APP_ENV`

Laravel's `app.env` gives WordPress its environment type, unless the process already has a `WP_ENVIRONMENT_TYPE`:

| `app.env` | WordPress environment type |
| --- | --- |
| `local` | `local` |
| `dev`, `development`, `testing` | `development` |
| `staging` | `staging` |
| `prod`, `production`, any other or missing value | `production` |

Plugins and themes read it with `wp_get_environment_type()`. The mapping and its overrides are described in [WordPress Configuration](wordpress-config.md#wordpress-environment-type); it does not depend on `APP_DEBUG`.

`APP_ENV=production` also matters to `pollora:doctor`: in production it treats Laravel's configuration and route caches (`php artisan optimize`) as expected and does not warn when they are missing, and in any other environment it warns when they are on.

### `APP_DEBUG`

`app.debug` is read once, when WordPress is bootstrapped, and drives:

- **`WP_DEBUG`, `WP_DEBUG_DISPLAY` and `JETPACK_DEV_DEBUG`**: all three take its value (`src/WordPress/Bootstrap.php`). With `APP_DEBUG=true` on a public site, PHP notices and warnings are printed in the pages.
- **The discovery cache**: with `APP_DEBUG=true` the persistent cache is off, so a class you add is found on the next request, and a slow location is logged; otherwise scans are stored in Laravel's cache until `php artisan discovery:clear` (`LaravelDiscoveryCacheManagerFactory`). See [Discovery](discovery.md). In production, run `discovery:clear` on each deployment.
- **The template marker**, an HTML comment naming the template that answered, printed only when `WP_DEBUG` is on. See [Debugging](debugging.md#the-template-marker).
- **Async actions**: an argument that cannot be stored throws an exception in debug mode; in production the action runs synchronously and the incident goes to the Laravel log ([Asynchronous Actions](async-actions.md)).

Laravel's own debug page and the error-view diagnostics follow `APP_DEBUG` as well.

Pollora does not set `WP_DEBUG_LOG`. To write WordPress's debug log, add it under `constants` in `config/wordpress.php`.

### WP-Cron

Pollora always defines `DISABLE_WP_CRON` as `true`, in every environment: page views never trigger WordPress's pseudo-cron. In production something else has to request it, or scheduled events and WP-Cron or Action Scheduler async actions never run:

```bash
* * * * * curl -s https://example.com/cms/wp-cron.php > /dev/null
# or, with WP-CLI
* * * * * cd /path/to/project && wp cron event run --due-now
```

See [Something must run the queue](async-actions.md#something-must-run-the-queue). Locally, no cron means scheduled events wait for a manual `wp cron event run --due-now`.

### The queue

`QUEUE_CONNECTION` does not decide whether Pollora's async actions run in the request. With the default `HOOKS_ASYNC_DRIVER=auto`, the Laravel queue is used only when `HOOKS_ASYNC_CONNECTION` names a connection, and a `sync` or `null` connection is always skipped; then Action Scheduler or WP-Cron take over (`config/hooks.php`, `QueueDriver`). Only `HOOKS_ASYNC_DRIVER=sync` runs every handler inside the request.

So:

- To run async actions on a queue in production, set `HOOKS_ASYNC_CONNECTION` to a connection that is not `sync`, and keep `php artisan queue:work` running under Supervisor or systemd.
- Laravel jobs of your own, and the `laravel` [scheduling driver](schedule-events.md#the-laravel-driver), need a worker on `QUEUE_CONNECTION` too. With `sync`, they run inside the request.
- In development, `HOOKS_ASYNC_DRIVER=sync` makes handlers run at once, without a worker and without touching the code.

### Cache

`CACHE_STORE` is the store Laravel's cache uses, and so the one that holds the discovery scans outside debug mode. The skeleton ships `database`. Pollora does not pick a store for you.

## A production `.env`

```bash
APP_NAME=Pollora
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

DB_CONNECTION=mysql
# DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD

QUEUE_CONNECTION=database
HOOKS_ASYNC_CONNECTION=database

CACHE_STORE=database
SESSION_DRIVER=database
```

`APP_KEY` and the WordPress keys and salts (`AUTH_KEY`, `SECURE_AUTH_KEY`, `LOGGED_IN_KEY`, `NONCE_KEY` and their `_SALT` counterparts, read by `config/wordpress.php`) must be set as well, with values of your own for each environment. Leave `HOOKS_ASYNC_CONNECTION` out if no worker runs.

## Reading the environment in code

Use what Laravel and WordPress provide:

```php
// Laravel
app()->environment('production');
app()->environment('local', 'staging');
config('app.debug');

// WordPress: the value mapped from app.env, see the table above
wp_get_environment_type();
```

The framework keeps a few internal classes for debug and console detection (`DebugDetectorInterface`, `DebugService`, `ConsoleDetector`). They are not a public API. Do not depend on them.

## Checking a deployment

```bash
php artisan pollora:doctor
```

It reports, among others, configuration and route caches that are on outside production, a discovery cache that no longer matches the code, and async actions that nothing runs (overdue WP-Cron events, jobs waiting for a worker).

## See also

- [WordPress Configuration](wordpress-config.md): the `app.env` to WordPress environment mapping and `wordpress.constants`
- [Discovery](discovery.md): the discovery cache
- [Asynchronous Actions](async-actions.md): drivers and what must run them
- [Debugging](debugging.md): the debug bar and the template marker
