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
- The administration of a sub-site (`/site2/wp-login.php`, then `/site2/wp-admin/`, its assets, `options-general.php`) once the web server rules below are on. Measured on Apache 2.4 + PHP-FPM, WordPress 7.1, a real two-site network.

## What does not work

- **Laravel queries on WordPress tables read the main site's tables** whichever site is served: the installed check and, with `wordpress.api_plugins`, the plugin list of `/api/` requests.
- **The discovery cache is keyed by path, not by site.** Not verified when sites activate different themes or plugins.

## Administer a sub-site (sub-directory network)

On a sub-directory network the URLs of a sub-site are `/site2/wp-admin/`, `/site2/wp-login.php`, `/site2/wp-includes/...`. WordPress lives in `cms/`, so those paths are not files: the front controller answers. Without a rule, `/site2/wp-login.php` redirects to itself and `/site2/wp-admin/css/...` is a 404. A sub-domain network does not need this.

The fix belongs to the web server: send those paths to `cms/` **keeping the original URL** (`REQUEST_URI`, which WordPress reads to find the site). Only what WordPress asks of a sub-site's own URL is sent: `wp-admin/`, `wp-includes/` (admin and login assets), `wp-login.php`, `wp-signup.php`, `wp-activate.php`, `wp-comments-post.php` (comment form), `wp-cron.php` (cron spawn) and `xmlrpc.php`. `wp-json` and `wp-content` need nothing (the REST API goes through the front controller; `content_url()` has no site prefix).

### Apache (and DDEV with `webserver_type: apache-fpm`)

The skeleton's `public/.htaccess` carries the block, **commented out**: on a single site a child page can be called `wp-admin` (`/about/wp-admin`; WordPress does not reserve the slug), so the rule is not inert there. On a sub-directory network, remove the leading `# ` of its four lines (a skeleton created before this release: copy them from the skeleton's `.htaccess`):

```apache
RewriteRule ^(?!cms/)([_0-9a-zA-Z-]+)/wp-admin$ /$1/wp-admin/ [R=301,L]
RewriteCond %{DOCUMENT_ROOT}/cms/$1 -f [OR]
RewriteCond %{DOCUMENT_ROOT}/cms/$1 -d
RewriteRule ^(?!cms/)[_0-9a-zA-Z-]+/((?:wp-admin|wp-includes)(?:/.*)?|(?:wp-login|wp-signup|wp-activate|wp-comments-post|wp-cron|xmlrpc)\.php)$ cms/$1 [L]
```

They stay before the trailing-slash and front-controller rules. `cms/` is excluded so that `/cms/wp-admin/` is not rewritten twice; a path whose target does not exist in `cms/` still goes to the front controller (its 404). Because the first segment is any `[_0-9a-zA-Z-]+`, `/anything/wp-admin/` also reaches WordPress (the login of the main site) once on.

Verified on Apache 2.4.67 + PHP-FPM 8.4 with a real two-site network (`multi.test`, site `/site2/`): before, `/site2/wp-login.php` 302 to itself, assets 404; after, login page titled "Site Two", login then dashboard "Dashboard < Site Two", `options-general.php` shows "Site Two", `/site2/wp-admin` 301 to `/site2/wp-admin/`, assets 200, `/cms/wp-login.php` and `/cms/wp-admin/` unchanged, a missing file stays a 404 of the site, and with the block commented out every answer is identical to before.

### nginx (not tested: nginx was not available)

In the `server` block, before `location /`:

```nginx
rewrite ^/(?!cms/)([_0-9a-zA-Z-]+)/wp-admin$ /$1/wp-admin/ permanent;
rewrite ^/(?!cms/)[_0-9a-zA-Z-]+/((wp-admin|wp-includes)(/.*)?|(wp-login|wp-signup|wp-activate|wp-comments-post|wp-cron|xmlrpc)\.php)$ /cms/$1 last;
```

`rewrite ... last` restarts the location search on `/cms/...`: the usual `location /` (`try_files $uri $uri/ /index.php?$query_string`) serves assets and the directory index, and the `location ~ \.php$` block runs the PHP file. `$request_uri` is not changed by `rewrite`, so with `fastcgi_param REQUEST_URI $request_uri;` (in `fastcgi_params`) WordPress sees `/site2/wp-admin/`. With DDEV (`nginx-fpm`), put the two lines in a file of `.ddev/nginx/` (snippets included in the server block) and `ddev restart`. Not verified: the nginx syntax, the `.ddev/nginx` placement.

## Not verified

The `--network` option of the schedule commands (read in code, not run), subdomain networks, WP-CLI with `--url`, WooCommerce and other plugins in a network, Laravel sessions and cookies across sites, the nginx rules above, Apache with `mod_php` or CGI (only PHP-FPM was run), sites with a path that contains a dot or a capital (`[_0-9a-zA-Z-]` as WordPress's own network rules).
