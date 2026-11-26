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
- **Scan**: core, plugin and theme files, suspicious files, disguised JavaScript in the database, files the site gives away, security headers, and, opt-in, known vulnerabilities.

A score from 0 to 100 sums it up, with what to fix first. The [README](https://github.com/Pollora/hornet#what-it-checks) describes each check.

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
