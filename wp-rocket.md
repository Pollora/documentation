# WP Rocket

[WP Rocket](https://wp-rocket.me) writes its page cache rules into `.htaccess` and sets the `WP_CACHE` constant in `wp-config.php`. In a Pollora project the web server's rules and the constants belong to the project, not to a plugin: the package [`pollora/wp-rocket`](https://github.com/Pollora/wp-rocket) tells WP Rocket not to do either, unless you ask for it.

> This used to live in the framework. Since the framework release that moved it out, a project that runs WP Rocket must install the package. `php artisan pollora:doctor` (and Tools › Site Health) warns when WP Rocket is active and the package is missing.

## Installation

```bash
composer require pollora/wp-rocket
```

Laravel discovers the service provider: there is nothing to register.

## What it does

The package answers two WP Rocket filters, from two settings. Both are **off** by default:

| Setting | WP Rocket filter | Default | When it is `true` |
| --- | --- | --- | --- |
| `generate_htaccess` | `rocket_init_cache_dir_generate_htaccess` | `false` | WP Rocket writes its rules into `.htaccess` |
| `set_cache_constant` | `rocket_set_wp_cache_constant` | `false` | WP Rocket sets `WP_CACHE` in `wp-config.php` |

With both off, WP Rocket does not touch your files. Its page cache needs `WP_CACHE` to be `true`, so define it yourself through the [WordPress configuration](wordpress-config.md):

```php
// config/wordpress.php
'constants' => [
    'WP_CACHE' => true,
],
```

## Configuration

The settings live under the `wprocket` key of `config/wordpress.php` (publish it with `php artisan vendor:publish --tag=wordpress`, see [WordPress Configuration](wordpress-config.md)):

```php
'wprocket' => [
    'generate_htaccess' => false,  // let WP Rocket write its rules into .htaccess
    'set_cache_constant' => false, // let WP Rocket set WP_CACHE
],
```

A value you do not set stays `false`. Turn `generate_htaccess` on only when the web server reads a `.htaccess` that Pollora's document root contains (Apache); with nginx the rules WP Rocket generates are useless, and its documentation gives the nginx equivalent to put in the server configuration.

## Coming from the framework

| Was | Is |
| --- | --- |
| `Pollora\ThirdParty\WpRocket\WpRocketServiceProvider` | `Pollora\WpRocket\WpRocketServiceProvider` |

The old class name keeps working once the package is installed, and is removed in the next major release. The `wordpress.wprocket` settings are unchanged: nothing to migrate.

Without the package, WP Rocket goes back to doing what it does elsewhere: it writes to `.htaccess` and sets `WP_CACHE` on its own, with no error. That is what the Doctor check is for.
