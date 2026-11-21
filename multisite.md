# Multisite: what works, what does not

Pollora does not install or manage a WordPress network. A network you create yourself runs, with the limits below. `pollora:doctor` prints them whenever it finds `MULTISITE` (check `multisite`, also in Tools > Site Health).

Measured on a sub-directory network (main site + `/site2/`), WordPress 7.1, SQLite and PHP's built-in server. Anything not listed there is not verified.

## Declare the network

`pollora:install` has no network option. Convert the site with WordPress's own tool, then declare the constants in `config/wordpress.php` (or `public/wp-config.php`):

```php
'constants' => [
    'wp_allow_multisite' => true,
    'multisite' => true,
    'subdomain_install' => false,
    'domain_current_site' => 'example.test',   // with the port if APP_URL has one
    'path_current_site' => '/',
    'site_id_current_site' => 1,
    'blog_id_current_site' => 1,
],
```

`APP_URL`, `DOMAIN_CURRENT_SITE` and `PATH_CURRENT_SITE` must agree, port included.

## What Pollora does when `MULTISITE` is declared

- `WP_HOME` and `WP_SITEURL` are not defined: WordPress ignores them in a network and reads the database. The `home` option of the main site is rewritten to `APP_URL` only when it is exactly `APP_URL` + `/cms` (the value the installer stores). Without this, `home_url()` and the canonical redirect sent visitors to `/cms/...`.
- The console presents `HTTP_HOST` with the port of `APP_URL` and a default `REQUEST_URI`. If WordPress still redirects the command (unknown host), the command exits 1 with a message naming the host and `DOMAIN_CURRENT_SITE`; it used to exit 0 with no output.
- A single site is unchanged: all of this is off unless `MULTISITE` is declared.

## What works (verified)

- Front of the main site and of a sub-site (`/`, `/site2/`), REST (`/site2/wp-json/`), title and theme per site.
- `network_site_url()` points to `/cms/wp-admin/network/`; the network admin works for a super admin.
- `pollora:doctor`, `artisan` commands, when the host matches the network.

## What does not work

- **The administration of a sub-site in a sub-directory network.** `/site2/wp-admin/` and `/site2/wp-login.php` are not files: the front controller answers (Laravel page or redirect to itself). Fix at the web server: send them to WordPress's directory **keeping the original URL**. Verified by a router that does exactly this (login, then the dashboard of "Site Two"); the Apache rule itself is not tested:

  ```apache
  # before the trailing-slash and front-controller rules of public/.htaccess
  RewriteRule ^[_0-9a-zA-Z-]+/((wp-admin|wp-includes)(/.*)?|wp-login\.php)$ cms/$1 [L]
  ```

  Without this, administer sub-sites from the network admin.
- **Laravel queries on WordPress tables read the main site's tables** whichever site is served: the installed check and, with `wordpress.api_plugins`, the plugin list of `/api/` requests.
- **The discovery cache is keyed by path, not by site.** Not verified when sites activate different themes or plugins.

## Not verified

The `--network` option of the schedule commands (read in code, not run), subdomain networks, WP-CLI with `--url`, WooCommerce and other plugins in a network, Laravel sessions and cookies across sites, plugins activated network-wide, the Apache and nginx rules above.
