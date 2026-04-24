<p align="center">
  <a href="https://github.com/Pollora/framework">
    <img src="https://raw.githubusercontent.com/Pollora/framework/main/resources/images/pollora-logo.svg" width="400" alt="Pollora">
  </a>
</p>

<h3 align="center">Documentation</h3>

<p align="center">
  The Laravel & WordPress integration framework.
  <br>
  <a href="https://github.com/Pollora/framework"><strong>Framework</strong></a> &middot;
  <a href="https://github.com/Pollora/pollora"><strong>Skeleton</strong></a> &middot;
  <a href="https://github.com/Pollora/framework/blob/main/CHANGELOG.md"><strong>Changelog</strong></a>
</p>

---

## Getting Started

- [Installation](installation.md) — Setup with Composer, DDEV, and environment configuration
- [Getting Started](getting-started.md) — Build your first Pollora application
- [IDE Integration](ide.md) — Configure your editor for optimal DX
- [Environment Management](environment-management.md) — Managing WordPress constants and `.env`

## Routing & Controllers

- [Routing](routing.md) — `Route::wp()`, WordPress conditions, and hybrid Laravel+WP routing
- [Controllers](controllers.md) — Controllers with dependency injection
- [Middleware](middleware.md) — Request/response filtering and authentication

## Content Management

- [Post Types](post-types.md) — `#[PostType]` attribute, config-based registration
- [Taxonomies](taxonomies.md) — `#[Taxonomy]` attribute, custom taxonomies
- [Options](options.md) — WordPress options with Laravel's fluent API

## WordPress Integration

- [Hooks](hooks.md) — `#[Action]` / `#[Filter]` attributes, hookable classes
- [Authentication](auth.md) — WordPress authentication guard
- [WP REST API](wp-rest-api.md) — `#[WpRestRoute]` attribute, custom REST endpoints
- [WP-CLI Commands](wp-cli-commands.md) — Custom Artisan-style WP-CLI commands
- [WordPress Config](wordpress-config.md) — Managing WordPress constants via Laravel config
- [WordPress Logging](wordpress-logging.md) — WordPress error logging through Laravel

## Frontend & Theming

- [Theming](theming.md) — Theme structure, Blade templates, parent/child themes
- [Assets](assets.md) — Vite integration, HMR, Tailwind CSS
- [Blocks](blocks.md) — Custom Gutenberg blocks with Vite and JSX
- [Patterns](patterns.md) — Gutenberg block patterns and categories

## Monitoring & Tooling

- [Dashboard & Status](dashboard.md) — Admin dashboard, `pollora:status` command, `--json` output

## Advanced Features

- [Discovery](discovery.md) — Auto-discovery system for PHP attributes
- [Modules](modules.md) — Modular architecture with nwidart/laravel-modules
- [Plugins](plugins.md) — Plugin development with modern tooling
- [Events & Listeners](events-listeners.md) — WordPress event dispatching and Laravel listeners
- [Scheduling](schedule-events.md) — `#[Schedule]` attribute, WordPress cron management
- [AJAX](ajax.md) — Handling AJAX requests
- [Admin Pages](admin-pages.md) — WordPress admin interfaces
- [Menu](menu.md) — Admin menu management

## Requirements

| Dependency | Version |
|---|---|
| PHP | ^8.3 |
| Laravel | 13.x |
| WordPress | 6.9+ |

## Contributing

We welcome contributions to both the framework and documentation. See the [Contributing Guide](https://github.com/Pollora/framework/blob/main/CONTRIBUTING.md) for details.

## License

Pollora is open-source software licensed under the [GPL-2.0-or-later](https://github.com/Pollora/framework/blob/main/LICENSE) license.
