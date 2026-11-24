# Object Cache

WordPress caches options, posts, terms and queries for the length of one request. A *persistent* object cache keeps them in a shared store, so the next request, and the next worker, finds the work already done instead of rebuilding it from the database. The package [`pollora/object-cache`](https://github.com/Pollora/object-cache) does this with Redis, and adds a Laravel cache driver over the same store.

> **Status: early release (`0.3`).** The package has not run in production yet, and its API may change before `1.0`: pin the version. The use outside Pollora is new in 0.3 and has not run on a production site yet. It does nothing until you install its drop-in (`php artisan pollora:object-cache:enable` in a Pollora project, `wp object-cache enable` elsewhere).

The package itself does not depend on Laravel: it needs PHP and phpredis only, and also runs on a classic WordPress or Bedrock site. In a Pollora project, the framework ships it and adds the Laravel side: the settings read from `config/database.php`, the `pollora:object-cache:*` commands, the `wordpress` cache driver and the `cache:clear` hook.

WordPress loads a persistent object cache from a drop-in file, `object-cache.php`, in the content directory. The package implements the whole contract of `WP_Object_Cache`: `add`, `get`, `set`, `replace`, `delete`, `incr`, `decr`, their `*_multiple` forms, `flush`, `flush_group`, `flush_runtime`, global and non-persistent groups, and multisite. It follows WordPress's own behavior, down to a stored `false` being a hit and a number never going below zero.

## Installation

### In a Pollora project

`pollora/object-cache` comes with the framework (from v13.43.3): there is nothing to `composer require`. Install the drop-in:

```bash
php artisan pollora:object-cache:enable
```

| Command | What it does |
| --- | --- |
| `pollora:object-cache:enable` | Copies `object-cache.php` into the content directory |
| `pollora:object-cache:disable` | Removes the drop-in |
| `pollora:object-cache:status` | Shows the drop-in, phpredis, Redis and the key prefix |
| `pollora:object-cache:flush` | Empties this site's keys, nothing else |

`ext-redis` (phpredis) is needed to reach Redis, and `ext-igbinary` to use the igbinary serializer. Both are optional: without `ext-redis` the drop-in defines nothing and WordPress runs on its own in-memory cache, with no error.

Updating from a version of the package that required Laravel (0.1 and 0.2): run `pollora:object-cache:enable` again. The drop-in is version 2, and `enable` replaces an older one.

### On a classic WordPress or Bedrock site

Install the package with Composer, set its settings in `wp-config.php`, then install the drop-in with WP-CLI:

```bash
composer require pollora/object-cache
wp object-cache enable
```

```php
// wp-config.php, before wp-settings.php is loaded
define('WP_OBJECT_CACHE_CONFIG', [
    'redis' => ['host' => '127.0.0.1', 'port' => 6379, 'database' => 5],
    'prefetch' => ['enabled' => true],
]);
```

`wp object-cache` has the same four commands as above (`enable`, `disable`, `status`, `flush`). Requires PHP 8.3+ and phpredis. Without `WP_OBJECT_CACHE_CONFIG` the drop-in stays inert and WordPress keeps its own cache.

`enable` records the absolute path of the Composer autoloader in the drop-in, because WordPress loads `object-cache.php` before any plugin. The package therefore has to be installed with the site's Composer (Bedrock does, since `wp-config.php` loads the autoloader first); a package installed with `wp package install` lands in the home directory of the user who ran it, where a web request does not look.

`enable` and `disable` are idempotent. They only touch a drop-in this package wrote: another object cache's `object-cache.php` is left alone, and `enable --force` replaces it. A drop-in from an older version of the package is updated by running `enable` again.

## Configuration

The settings are one array. In a Pollora project they live under `pollora.object-cache`: the framework builds the array from that configuration and from the Redis connection of `config/database.php` (the one `connection` names), and the environment variables below fill it. On a classic WordPress or Bedrock site, the array is the `WP_OBJECT_CACHE_CONFIG` constant (the keys `connection`, `laravel.*` and the environment variables do not exist there; `redis.host`, `redis.port`, `redis.password`, `redis.url` and the like replace the connection). Every key is optional.

| Key | Environment (Pollora) | Default |
| --- | --- | --- |
| `connection` | `OBJECT_CACHE_REDIS_CONNECTION` | `default` (Pollora only) |
| `redis.database` | `OBJECT_CACHE_REDIS_DATABASE` | the connection's, `0` on WordPress |
| `redis.connect_timeout`, `redis.read_timeout` | `OBJECT_CACHE_CONNECT_TIMEOUT`, `OBJECT_CACHE_READ_TIMEOUT` | `1.0` seconds |
| `redis.persistent` | `OBJECT_CACHE_PERSISTENT` | `false` |
| `prefix` | `OBJECT_CACHE_PREFIX` | derived (see below) |
| `serializer` | `OBJECT_CACHE_SERIALIZER` | `php` (or `igbinary`) |
| `allowed_classes` | | `true`: any class (or `false`, or a list of class names) |
| `max_ttl` | `OBJECT_CACHE_MAX_TTL` | `0`, no limit |
| `global_groups`, `non_persistent_groups` | | `[]`; a name may end with `*` |
| `prefetch.enabled` | `OBJECT_CACHE_PREFETCH` | `false` |
| `prefetch.max_keys` | `OBJECT_CACHE_PREFETCH_MAX_KEYS` | `300` |
| `prefetch.ttl` | `OBJECT_CACHE_PREFETCH_TTL` | `604800` seconds |
| `prefetch.exclude_groups` | | `['userlogins']`; a name may end with `*` |
| `laravel.group` | `OBJECT_CACHE_LARAVEL_GROUP` | `laravel` (Pollora only) |
| `laravel.flush_with_cache_clear` | `OBJECT_CACHE_FLUSH_WITH_CACHE_CLEAR` | `true` (Pollora only) |

A list a project sets (`exclude_groups`, `global_groups`) replaces the default list; the keyed arrays (`redis`, `prefetch`) are merged key by key.

- **Use a database of its own.** The default connection is `default`, not `cache`: Laravel's cache store empties its whole Redis database on `cache:clear`, and the object cache would go with it. If `default` is shared with something else, point `OBJECT_CACHE_REDIS_DATABASE` at a database nothing else uses.
- **Timeouts.** Without a limit, phpredis waits for the PHP socket timeout (60 s) when Redis stops answering. The one-second default keeps a request from hanging.
- **Objects.** Plugins cache objects of their own, so `allowed_classes` rebuilds any class by default. Set a list of class names to narrow it.
- **Groups.** `global_groups` are shared by every site of a network; `non_persistent_groups` are kept out of Redis. A name may end with `*`, as in `wc_cache_*`.

## Prefetch

A WordPress page reads a hundred keys or more, one call to Redis each. With the prefetch on (`OBJECT_CACHE_PREFETCH=true`, or `'prefetch' => ['enabled' => true]` in `WP_OBJECT_CACHE_CONFIG`), the pages of one kind remember what they read, and the next one loads those keys with a single call.

- **A kind of page** is the first segment of the path (`/shop/...`, `/blog/...`, the home page), the blog of a network, and whether the visitor is signed in. The query string does not count.
- **Which requests take part**: pages of the front, fetched with GET or HEAD. Console commands, the administration, AJAX, REST, cron and form submissions neither load nor record anything. A page that is not answered with a 200 writes nothing.
- **The profile** is written when a request read keys it did not know, up to `max_keys`, and expires after `ttl` seconds without a rewrite. It lives under the site prefix, so a flush removes it too. A page that brings few new keys does not rewrite a full profile.
- **What is never loaded early**: non-persistent groups, `prefetch.exclude_groups`, and what a request wrote before its first read. A key changed or deleted since the profile was written is read from Redis as of the request, so the prefetch cannot serve an older value than a normal read would. A value that holds an object whose class is not loaded yet is left out and read normally when it is asked for.
- **Groups where a value read a moment too soon would be wrong** (sessions, logins) belong in `prefetch.exclude_groups`. In a Pollora project another package adds its own from a service provider's `register()`: `config()->push('pollora.object-cache.prefetch.exclude_groups', 'session_*')`. On a classic WordPress site with WooCommerce, add `wc_session_id` and `wc_cache_*` to `exclude_groups` yourself: nothing does it for you there.
- If Redis stops answering while the profile is loaded, the request degrades as for any read.

It is off by default: turn it on, measure the number of Redis commands per page (`redis-cli INFO commandstats`), and keep it if it falls. On a development site (WordPress 7, WooCommerce, a Blade theme), once the profiles were written, the commands per request went from about 230 to 100-130 on the home page, from 203 to 126 on the shop page, and from 168 to 52 on a product page. These numbers are an orientation: they depend on the site. A WooCommerce cart and a checkout through the Store API gave the same results with the prefetch on as without it.

## Key prefix and flush scope

Every key carries a site prefix, which tells sites apart on a shared Redis. Left empty, `prefix` is `WP_CACHE_KEY_SALT` when the project defines it. Without it, a Pollora project derives it from the site URL, the database name and the table prefix, and a classic WordPress site from `DB_NAME`, `DB_HOST` and the table prefix (the package never reads the database for it). In a Pollora project, web requests, `artisan`, WP-CLI and queue workers all compute the same prefix, because it is read from Laravel's configuration.

A flush removes the keys of that prefix and nothing else. It walks them with `SCAN` and removes them with `UNLINK`; it never runs `FLUSHDB`. A Redis shared with other sites keeps their caches.

## When Redis is down

A site never fails because of its cache. If the store cannot be reached, the request keeps serving from its request cache, stops asking Redis for the rest of the request, and the failure goes to the PHP error log once.

`add`, `replace` and `incr`/`decr` are operations of the store, not a read followed by a write, so two workers cannot race between the two steps.

## Queue workers (Pollora)

The request cache is emptied before each job, so a job never starts from what the previous one cached. A worker also gets another chance at Redis after an outage.

## The `wordpress` cache driver (Pollora)

Declare a store with the `wordpress` driver in `config/cache.php`:

```php
'stores' => [
    'wordpress' => ['driver' => 'wordpress'],
],
```

`Cache::store('wordpress')` then reads and writes the object cache, in one group (`laravel` by default, see `laravel.group`), under the same site prefix as `wp_cache_*`. A value put from Laravel is readable from WordPress:

```php
Cache::store('wordpress')->put('greeting', 'hello', 600);

wp_cache_get('greeting', 'laravel'); // 'hello'
```

- Reads always go to Redis, never to the request cache, so a counter or a flag written by another process is seen.
- `Cache::add()` is atomic.
- Tags are not supported.
- The events (`CacheHit`, `KeyWritten`…) are those of Laravel's repository, enabled like for any store.
- `Cache::flush()` on this store empties its group only, never WordPress's own groups and never the Redis database.
- If Redis stops answering, reads miss and writes report `false`. Nothing throws.

### `cache:clear`

`php artisan cache:clear` also empties this site's object cache (only its keys), unless `laravel.flush_with_cache_clear` is `false`. When the cleared store is the `wordpress` one, only its group goes. `cache:clear` still succeeds if Redis is down.

## Doctor check

`php artisan pollora:doctor` has a check, `object-cache-package`. It runs in the console only.

| Situation | Result |
| --- | --- |
| The package is installed (it is, with the framework from v13.43.3), but its drop-in is not | Warns, and suggests `php artisan pollora:object-cache:enable` |
| The package is installed, but the drop-in is another object cache's | Warns, and suggests `enable --force` |
| The package is installed and its drop-in is in place | Ok |
| The package is absent, phpredis is loaded and no persistent drop-in exists | Warns, and suggests `composer require pollora/object-cache` |
| Another object cache is in place without the package, phpredis is not loaded, or WordPress is not loaded | Skipped |
