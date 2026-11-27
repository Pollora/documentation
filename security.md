# Security (Hornet)

[`pollora/hornet`](https://github.com/Pollora/hornet) audits a site, scans it, and tells what to fix and how. It runs on Pollora, on a classic WordPress and on Bedrock. It reads and reports: nothing is hardened, sent or changed until you ask for it.

> Version 0.9: the settings, commands, hooks and abilities can still change before 1.0.

## Install

```bash
composer require pollora/hornet
```

On Pollora nothing else is needed: Laravel discovers the service provider. The screen is in **Tools > Security**, and the audit is also one line of `php artisan pollora:doctor`.

Requirements: PHP 8.3, WordPress 6.9, MySQL 8 or MariaDB 11.

## Settings

```bash
php artisan vendor:publish --tag=hornet-config
```

The file is `config/pollora/hornet.php`. A `config/hornet.php` is **not** read (Hornet says so in Site Health). Everything that acts on the site is off until it is written there:

```php
return [
    'scanner' => ['schedule' => 'daily'],
    'hardening' => ['xmlrpc' => true, 'user_enumeration' => true],
    'headers' => ['content_type_options' => true],
    'journal' => ['enabled' => true],
];
```

## What it looks at

- **Configuration and accounts**: keys and salts, debug mode, HTTPS, file editor, versions, XML-RPC, user enumeration, open registration, application passwords, new and dormant administrators, updates and inactive plugins and themes.
- **Pollora only**: `APP_KEY`, `APP_DEBUG` in production, Debugbar forced on.
- **Scan**: core, plugin and theme files, plugins closed or left behind on WordPress.org, suspicious files, disguised JavaScript in the database, files the site gives away, security headers, and, opt-in, known vulnerabilities.

A score from 0 to 100 sums it up, with what to fix first. The [README](https://github.com/Pollora/hornet#what-it-checks) describes each check.

**Tools > Site Health > Info** also has a Hornet section: the version, the date of the last scan, what is turned on and the score. The last two are private: "Copy site info to clipboard" leaves them out.

## Closing the REST API to visitors

```php
'hardening' => [
    'restrict_rest_api' => true,
    'rest_api_allow' => ['wp/v2/posts', 'wp/v2/pages'],   // added to the defaults
],
```

Visitors who are not logged in get a `401`, unknown routes included. Logged-in users, application passwords included, keep the whole API.

**Test your front end before you turn it on**: whatever it calls without a login stops working until it is listed. That includes a payment webhook or a licence server that checks its own key without logging a user in. A headless front end usually reads `wp/v2`, which is not open by default.

Open by default: `oembed/1.0/embed`, `contact-form-7/v1/contact-forms`, `wc/store` and, with `pollora/headless`, `headless/v1`. With `wp/v2/posts` open, turn `user_enumeration` on too, or `?_embed` still gives the author of each post. The [README](https://github.com/Pollora/hornet#closing-the-rest-api-to-visitors) has the exact rules.

## Closed plugins and Composer packages

WordPress says nothing when a plugin you run is closed on WordPress.org. The scan reports it (high), and also a plugin with no release there for two years or more (low). For this it sends the **folder name** of each plugin to `api.wordpress.org`; `'disabled_checks' => ['abandoned-plugins']` stops it.

Known vulnerabilities of the project's Composer packages are a second opt-in:

```php
'vulnerabilities' => ['enabled' => true, 'composer' => true],
```

`enabled` alone covers plugins and themes only, and `composer` alone does nothing. With both, the **name and version** of each package installed from packagist.org, development packages included, are sent to [OSV](https://osv.dev) (`api.osv.dev`). Packages from anywhere else are counted, never named.

## Two-factor in the journal

With the journal on and the [Two-Factor](https://wordpress.org/plugins/two-factor/) plugin, it records a second factor passed or refused, and a method turned on or off. A code or a key is never read.

A `login.success` marked `second_factor: pending` only means the password was right: `login.second_factor_passed` says the visitor got in. A refused second factor is `login.second_factor_failed`, no longer `login.failed`: **update any alert or filter that counts `login.failed`**.

## The scan needs the scheduled tasks

Pollora sets `DISABLE_WP_CRON`, so the server runs them (see [Schedule Events](schedule-events.md)):

```
* * * * * curl -s https://example.com/cms/wp-cron.php
```

Without it, **Scan now** waits, and the screen says why.

## Command line

```bash
wp hornet status                 # the audit
wp hornet scan                   # the scan, in one go
wp hornet server-config nginx    # rules for the web server (apache, htaccess too); printed, never written
wp hornet unlock --all           # lift every login lock
```

## Limits

Translations are machine-made. Hornet has not been verified on a multisite network. Hornet is not a firewall, nor a signature-based malware scanner.
