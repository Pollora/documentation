<p align="center">
  <a href="https://pollora.dev">
    <img src="https://raw.githubusercontent.com/Pollora/.github/main/brand/banners/documentation.png" width="100%" alt="Pollora documentation: The Laravel framework for WordPress">
  </a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/github/license/Pollora/documentation" alt="License"></a>
</p>

This repository is the source of the documentation of **Pollora, the Laravel framework for WordPress**, published at **[pollora.dev](https://pollora.dev)**. Read it there for search, navigation and the latest version; open pull requests here to change it: the site copies these pages, so a fix made anywhere else is overwritten on the next sync.

[Read the docs on pollora.dev](https://pollora.dev) · [Framework](https://github.com/Pollora/framework) · [Skeleton](https://github.com/Pollora/pollora) · [Changelog](https://github.com/Pollora/framework/blob/main/CHANGELOG.md)

## Getting Started

- [Installation](installation.md) — Setup with Composer, DDEV, and environment configuration
- [Getting Started](getting-started.md) — Build your first Pollora application
- [Server Configuration](server-configuration.md) — Apache/Nginx setup, directory protection, reverse proxy & HTTPS
- [IDE Integration](ide.md) — Configure your editor for optimal DX
- [Environment Management](environment-management.md) — Managing WordPress constants and `.env`

## Routing & Controllers

- [Routing](routing.md) — `Route::wp()`, WordPress conditions, and hybrid Laravel+WP routing
- [Controllers](controllers.md) — Controllers with dependency injection
- [Middleware](middleware.md) — Request/response filtering and authentication

## Content Management

- [Post Types](post-types.md) — `#[PostType]` attribute, config-based registration
- [Post Type Attributes Reference](post-types-reference.md) — Every attribute available for post type registration
- [Taxonomies](taxonomies.md) — `#[Taxonomy]` attribute, custom taxonomies
- [Typed Meta](meta.md) — `#[Meta]` properties, `Meta::of()`, typed reads and writes, REST schema
- [Block Bindings](block-bindings.md) — `#[BlockBinding]` sources, typed meta in core blocks, bindable Blade blocks
- [Options](options.md) — WordPress options with Laravel's fluent API

## WordPress Integration

- [Hooks](hooks.md) — `#[Action]` / `#[Filter]` attributes, hookable classes
- [Asynchronous Actions](async-actions.md) — `->async()` and `#[Async]`: run an action after the request, through WP-Cron, Action Scheduler or a Laravel queue
- [Authentication](auth.md) — WordPress authentication guard
- [Roles & Capabilities](roles.md) — `$user->can()`, `@can`, `#[Role]`, `#[ModifyRole]`, post types with their own capabilities
- [WP REST API](wp-rest-api.md) — `#[WpRestRoute]` attribute, custom REST endpoints
- [Abilities](abilities.md) — `#[Ability]` attribute, the WordPress Abilities API for AI agents and MCP
- [WP-CLI Commands](wp-cli-commands.md) — Custom Artisan-style WP-CLI commands
- [WordPress Config](wordpress-config.md) — Managing WordPress constants via Laravel config
- [WordPress Logging](wordpress-logging.md) — WordPress error logging through Laravel
- [Translations](translations.md) — `__()` routing between Laravel's translator and WordPress's `.po`/`.mo` catalogues

## Frontend & Theming

- [Theming](theming.md) — Theme structure, Blade templates, parent/child themes
- [Assets](assets.md) — Vite integration, HMR, Tailwind CSS
- [Blocks](blocks.md) — Custom Gutenberg blocks with Vite and JSX
- [Patterns](patterns.md) — Gutenberg block patterns and categories

## Monitoring & Tooling

- [Dashboard & Status](dashboard.md) — Admin dashboard, `pollora:status` command, `--json` output
- [Nectar — AI Context](nectar.md) — AI guidelines, agent skills, and MCP tools for AI-assisted development

## Advanced Features

- [Discovery](discovery.md) — Auto-discovery system for PHP attributes
- [Modules](modules.md) — Modular architecture with nwidart/laravel-modules
- [Plugins](plugins.md) — Plugin development with modern tooling
- [Events & Listeners](events-listeners.md) — WordPress event dispatching and Laravel listeners
- [WordPress Events Reference](wordpress-events-reference.md) — Every WordPress and plugin event Pollora dispatches
- [Scheduling](schedule-events.md) — `#[Schedule]` attribute, WordPress cron management
- [AJAX](ajax.md) — Handling AJAX requests
- [Menu](menu.md) — Admin menu management

## Requirements

| Dependency | Version |
|---|---|
| PHP | 8.4+ for a new project (the framework alone accepts 8.3) |
| Laravel | 13.35 |
| WordPress | 7.1+ |

## Contributing

Contributions are welcome: see the [contributing guide](https://github.com/Pollora/.github/blob/main/CONTRIBUTING.md). Report security issues privately, as described in the [security policy](https://github.com/Pollora/.github/blob/main/SECURITY.md).

## License

The Pollora documentation is open-source software licensed under the [MIT license](LICENSE). © [RuBee group](https://rubee.group)
