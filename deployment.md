# Deploying a Pollora Project to Production

A Pollora project is a Laravel application, so Laravel's [deployment guide](https://laravel.com/docs/13.x/deployment) applies. This page covers what is specific to Pollora: what to build, what discovery needs after a release, what runs WP-Cron and the queue, and what to check before you open the site to visitors.

- [Release sequence](#release-sequence)
- [What to build, and where](#what-to-build-and-where)
- [What to cache](#what-to-cache)
- [Cron and queue](#cron-and-queue)
- [Go-live checklist](#go-live-checklist)

The page assumes a project that is already installed and a web server whose document root is `public/` ([Server Configuration](server-configuration.md)). Production values for `.env` are in [Environment Management](environment-management.md).

## Release sequence

Run these from the project root, in this order, on every release:

1. **Put the code in place** (`git pull`, or an artifact built in CI). `.env`, `storage/` and `public/content/uploads` belong to the server, not to the release.
2. **Install PHP dependencies.**

   ```bash
   composer install --no-dev --no-interaction
   ```

   Composer applies the framework's patches to WordPress core during the install, from the committed `patches.lock.json`. The skeleton's `post-autoload-dump` script runs `pollora:env:setup`; with no terminal to ask in, it prints a warning when the database is not configured and exits 0, so a deploy script does not hang or fail on it.
3. **Build the assets**, unless CI already did and shipped them. See [What to build, and where](#what-to-build-and-where).
4. **Run the migrations.**

   ```bash
   php artisan migrate --force
   ```

   The skeleton ships migrations for sessions, cache, jobs and failed jobs, and its `.env.example` uses the `database` driver for the first three.
5. **Refresh the discovery cache.**

   ```bash
   php artisan discovery:clear
   php artisan discovery:run
   ```

   Outside debug mode, discovery stores its scans in Laravel's cache (the `database` store in the skeleton), so the entries of the previous release survive a deploy and hide every class you added. `discovery:clear` drops them; `discovery:run` scans again, which fills the cache before the first visitor does. See [Auto-Discovery](discovery.md#caching-system). `discovery:run` prints `Cache Status: Enabled` when it writes a cache. `Disabled` means `APP_DEBUG` is on.
6. **Cache the configuration** (optional; see [What to cache](#what-to-cache)).

   ```bash
   php artisan config:cache
   ```

7. **Restart the queue workers**, so they load the new code. This is standard Laravel: [Queue Workers and Deployment](https://laravel.com/docs/13.x/queues#queue-workers-and-deployment).

   ```bash
   php artisan queue:restart
   ```

8. **Gate the release on the doctor.**

   ```bash
   php artisan pollora:doctor
   ```

   It exits `1` when a check finds an error and `0` otherwise, warnings included, so a deploy script can stop on it. See the [go-live checklist](#go-live-checklist).

## What to build, and where

The skeleton's `.gitignore` excludes `/public/build`, so **no build output, and no `manifest.json`, is committed**. Every release has to produce it, either in CI (and ship `public/build`) or on the server.

| What | Build | Output |
| --- | --- | --- |
| Project assets | `npm install && npm run build` at the project root | `public/build/manifest.json` |
| The active theme | `npm install && npm run build` in the theme's folder | `public/build/theme/<slug>/`, including the `theme.json` WordPress reads ([Theming](theming.md#how-it-works)) |
| Each Pollora plugin with a `vite.config.js` | the same, in the plugin's folder ([Plugin Development](plugins.md)) | `public/build/plugin/<slug>/` |
| Each enabled module with a `vite.config.js` | the same, in the module's folder ([Modules](modules.md)) | `public/build/module/<kebab-name>/` |

Things to know:

- The skeleton's `.gitignore` also excludes `/themes/*`. A project that versions its theme has to remove that line.
- If you build in CI, ship `public/build` with the release; there is no step on the server that regenerates it.
- Never deploy a hot file (`public/hot`, `public/<name>.hot`). While one exists, the assets are requested from the Vite dev server, and `pollora:doctor` reports an error when nothing answers there.
- A missing build does not answer 500: the build's assets resolve to no URL, an admin notice names the manifest and the command to run, and the doctor's *Theme, plugin and module builds* check reports an error with the folder to build.

See [Assets & Vite](assets.md) for the containers and the manifest.

## What to cache

| Cache | Status |
| --- | --- |
| Discovery | Used outside debug mode, filled by `discovery:run`. Required step in the sequence above. |
| `config:cache` | Optional. In production the doctor treats Laravel's caches as expected and does not warn when they are missing; in any other environment it warns when they are on. |
| `route:cache` | Not recommended here: how it behaves with `Route::wp()` routes has not been verified. |

In the `production` environment (`APP_ENV=production`), the doctor's *Configuration and route caches* check reports "Production: caches are expected". In any other environment, a cached configuration or route file is a warning, because edits to `.env` and `routes/` are then ignored. A staging site with `APP_ENV=staging` that caches therefore gets a warning, which does not fail the doctor.

Laravel's own rule applies to [`config:cache`](https://laravel.com/docs/13.x/configuration#configuration-caching): once the configuration is cached, `.env` is no longer loaded, so `env()` must only be called from `config/` files.

When a module is switched on or off, a cached configuration or route file still holds the previous state: run `php artisan optimize:clear`, then cache again ([Modules](modules.md#where-the-state-lives)). `pollora:doctor` warns about it.

## Cron and queue

Pollora sets `DISABLE_WP_CRON`, so a page load never runs WP-Cron. Which of the two things below your site needs depends on the drivers it uses.

| You use | What must run |
| --- | --- |
| WP-Cron events: your `#[Schedule]` events, WordPress and plugin events, `->async()` with the `wp-cron` or `action-scheduler` driver | A system cron that requests `wp-cron.php` every minute |
| `->async()` with the `queue` driver | A queue worker on the connection the actions use (`HOOKS_ASYNC_CONNECTION`, see [Asynchronous Actions](async-actions.md#something-must-run-the-queue)) |
| The `laravel` [scheduling driver](schedule-events.md#the-laravel-driver) | A queue worker (it replaces the system cron for those events) |

### The system cron

Either line, in the crontab of the user that owns the project:

```bash
* * * * * curl -s https://example.com/cms/wp-cron.php > /dev/null
* * * * * cd /path/to/project && wp cron event run --due-now
```

Both come from [Asynchronous Actions](async-actions.md#something-must-run-the-queue), where the drivers are explained.

### The queue worker

Keep `php artisan queue:work` running under a process manager. Supervisor is the one Laravel documents: this is adapted from the [example in Laravel's documentation](https://laravel.com/docs/13.x/queues#supervisor-configuration) (path, user, log file and `numprocs` are ours to change; add the connection to the command if it is not the default one).

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/project/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/path/to/project/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start "laravel-worker:*"
```

`stopwaitsecs` must be longer than your longest job. Pollora's own messages also name systemd as an alternative to Supervisor; any process manager that restarts `queue:work` does the job.

### Checking that it runs

- `pollora:doctor` reports queued asynchronous actions that nothing picks up: on a **`database`** queue connection, jobs that have waited 15 minutes. It cannot read other queue drivers (Redis, SQS), so use the queue tools of your driver there. The check needs WordPress loaded: if the console reports it as skipped, read it in **Tools › Site Health**.
- With the `laravel` scheduling driver, the doctor reports events more than ten minutes late, which is what a stopped worker looks like.
- WP-Cron events overdue for an hour mean nothing requests `wp-cron.php`: the doctor says so and prints the cron line to add.

## Go-live checklist

1. **Document root is `public/`.** `curl -I https://example.com/.env` must not answer `200`. The doctor's *Document root* check does the same from **Tools › Site Health** in wp-admin, since it needs a web request. See [Server Configuration](server-configuration.md#document-root).
2. **Environment.** `APP_ENV=production`, `APP_DEBUG=false` (debug mode turns the discovery cache off and sets `WP_DEBUG`), a generated `APP_KEY`, and `APP_URL` written with its scheme, `https://`. See [Environment Management](environment-management.md).
3. **HTTPS behind a proxy** trusts the proxy: [Reverse Proxy & HTTPS](server-configuration.md#reverse-proxy-and-https).
4. **The database is MySQL or MariaDB**, and the migrations have run.
5. **Assets are built** and no hot file is present.
6. **Discovery is warm** (`discovery:run` printed `Cache Status: Enabled`).
7. **The system cron is installed**, and a **queue worker runs** if the project uses one.
8. **`php artisan pollora:doctor` ends without an error.**

The last item is the gate. In a deploy script:

```bash
php artisan pollora:doctor || exit 1
```

Warnings exit `0`. To fail on them as well, read the JSON output, whose `errors` and `warnings` fields hold the counts:

```bash
php artisan pollora:doctor --json
```

The checks are listed in [Dashboard, Status & Doctor](dashboard.md#diagnosing-a-project-polloradoctor).
