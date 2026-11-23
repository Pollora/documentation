# Object Cache

WordPress caches options, posts, terms and queries for the length of one request. A *persistent* object cache keeps them in a shared store, so the next request, and the next worker, finds the work already done instead of rebuilding it from the database. The package [`pollora/object-cache`](https://github.com/Pollora/object-cache) does this with Redis, and adds a Laravel cache driver over the same store.

> **Status: in development, not released yet.** The package has no tag, and its API may change before the first one. Installing it changes nothing until you run `php artisan pollora:object-cache:enable`.

WordPress loads a persistent object cache from a drop-in file, `object-cache.php`, in the content directory. The package implements the whole contract of `WP_Object_Cache`: `add`, `get`, `set`, `replace`, `delete`, `incr`, `decr`, their `*_multiple` forms, `flush`, `flush_group`, `flush_runtime`, global and non-persistent groups, and multisite. It follows WordPress's own behavior, down to a stored `false` being a hit and a number never going below zero.

## Installation

```bash
composer require pollora/object-cache
```

Requires PHP 8.3+. `ext-redis` (phpredis) is needed to reach Redis, and `ext-igbinary` to use the igbinary serializer. Both are optional: without `ext-redis`, or without the package, the drop-in defines nothing and WordPress runs on its own in-memory cache, with no error.

Laravel discovers the service provider. Then copy the drop-in into place:

```bash
php artisan pollora:object-cache:enable
```

| Command | What it does |
| --- | --- |
| `pollora:object-cache:enable` | Copies `object-cache.php` into the content directory |
| `pollora:object-cache:disable` | Removes the drop-in |
| `pollora:object-cache:status` | Shows the drop-in, phpredis, Redis and the key prefix |
| `pollora:object-cache:flush` | Empties this site's keys, nothing else |

`enable` and `disable` are idempotent. They only touch a drop-in this package wrote: another object cache's `object-cache.php` is left alone, and `enable --force` replaces it. A drop-in from an older version of the package is updated by running `enable` again.

## Configuration

Publish the file with `php artisan vendor:publish --tag=object-cache-config`, or set the environment variables. The settings live under `pollora.object-cache`; the connection itself (host, port, credentials) is read from Laravel's `database.redis`.

| Key | Environment | Default |
| --- | --- | --- |
| `connection` | `OBJECT_CACHE_REDIS_CONNECTION` | `default` |
| `redis.database` | `OBJECT_CACHE_REDIS_DATABASE` | the connection's |
| `redis.connect_timeout`, `redis.read_timeout` | `OBJECT_CACHE_CONNECT_TIMEOUT`, `OBJECT_CACHE_READ_TIMEOUT` | `1.0` seconds |
| `redis.persistent` | `OBJECT_CACHE_PERSISTENT` | `false` |
| `prefix` | `OBJECT_CACHE_PREFIX` | derived (see below) |
| `serializer` | `OBJECT_CACHE_SERIALIZER` | `php` (or `igbinary`) |
| `allowed_classes` | | `true`: any class (or `false`, or a list of class names) |
| `max_ttl` | `OBJECT_CACHE_MAX_TTL` | `0`, no limit |
| `global_groups`, `non_persistent_groups` | | `[]`; a name may end with `*` |
| `laravel.group` | `OBJECT_CACHE_LARAVEL_GROUP` | `laravel` |
| `laravel.flush_with_cache_clear` | `OBJECT_CACHE_FLUSH_WITH_CACHE_CLEAR` | `true` |

- **Use a database of its own.** The default connection is `default`, not `cache`: Laravel's cache store empties its whole Redis database on `cache:clear`, and the object cache would go with it. If `default` is shared with something else, point `OBJECT_CACHE_REDIS_DATABASE` at a database nothing else uses.
- **Timeouts.** Without a limit, phpredis waits for the PHP socket timeout (60 s) when Redis stops answering. The one-second default keeps a request from hanging.
- **Objects.** Plugins cache objects of their own, so `allowed_classes` rebuilds any class by default. Set a list of class names to narrow it.
- **Groups.** `global_groups` are shared by every site of a network; `non_persistent_groups` are kept out of Redis. A name may end with `*`, as in `wc_cache_*`.

## Key prefix and flush scope

Every key carries a site prefix, which tells sites apart on a shared Redis. Left empty, `prefix` is derived from the site URL, the database name and the table prefix, and `WP_CACHE_KEY_SALT` is honored when the project defines it. Web requests, `artisan`, WP-CLI and queue workers all compute the same prefix, because it is read from Laravel's configuration.

A flush removes the keys of that prefix and nothing else. It walks them with `SCAN` and removes them with `UNLINK`; it never runs `FLUSHDB`. A Redis shared with other sites keeps their caches.

## When Redis is down

A site never fails because of its cache. If the store cannot be reached, the request keeps serving from its request cache, stops asking Redis for the rest of the request, and the failure goes to the PHP error log once.

`add`, `replace` and `incr`/`decr` are operations of the store, not a read followed by a write, so two workers cannot race between the two steps.

## Queue workers

The request cache is emptied before each job, so a job never starts from what the previous one cached. A worker also gets another chance at Redis after an outage.

## The `wordpress` cache driver

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
| phpredis is loaded and no persistent drop-in exists | Warns, and suggests `composer require pollora/object-cache && php artisan pollora:object-cache:enable` |
| The package is installed, but its drop-in is not | Warns |
| The package is installed, but the drop-in is another object cache's | Warns |
| Anything else | Skipped |
