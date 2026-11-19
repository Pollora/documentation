# Nectar — AI Development Context

[Nectar](https://github.com/Pollora/Nectar) is an AI-powered development companion for Pollora. Built on top of [Laravel Boost](https://laravel.com/docs/boost), it provides AI guidelines, agent skills, and a dedicated MCP server that gives AI agents deep context about your Pollora application.

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [MCP Server](#mcp-server)
- [Available MCP Tools](#available-mcp-tools)
- [AI Guidelines](#ai-guidelines)
- [Agent Skills](#agent-skills)
- [Upgrade Assistance](#upgrade-assistance)
- [Configuration](#configuration)
- [Architecture](#architecture)

## Overview

When working with AI coding assistants (Claude Code, Cursor, Windsurf, etc.), the quality of generated code depends heavily on the context available. Nectar bridges this gap by:

- **Injecting Pollora-specific guidelines** into your AI agent's context via Laravel Boost
- **Providing 12 on-demand skills** for domain-specific tasks (post types, typed meta, roles, themes, hooks, blocks and Block Bindings, abilities, etc.)
- **Exposing 11 MCP tools** for live introspection of your WordPress and Pollora environment
- **Offering upgrade prompts** that guide AI agents step-by-step through Pollora version upgrades

Nectar is a development-only package — it only loads in `local` and `development` environments.

## Installation

```bash
composer require pollora/nectar --dev
```

Run Boost to install guidelines and skills:

```bash
php artisan boost:install
```

Select `pollora/nectar` when prompted, or add it to your `boost.json`:

```json
{
    "packages": ["pollora/nectar"]
}
```

Then update:

```bash
php artisan boost:update
```

## MCP Server

Nectar registers a `pollora-nectar` MCP server that gives AI agents live access to your WordPress and Pollora environment.

### Starting the Server

```bash
php artisan nectar:mcp
```

### Registering in Your Project

Add it to your `.mcp.json` for automatic discovery:

```json
{
    "mcpServers": {
        "pollora-nectar": {
            "command": "php",
            "args": ["artisan", "nectar:mcp"]
        }
    }
}
```

For DDEV projects, use the DDEV wrapper:

```json
{
    "mcpServers": {
        "pollora-nectar": {
            "command": "ddev",
            "args": ["exec", "php", "artisan", "nectar:mcp"]
        }
    }
}
```

## Available MCP Tools

### `pollora_status`

Returns the overall framework status: PHP, Laravel, Pollora, and WordPress versions, active theme, discovery cache status, and all installed Pollora packages.

### `wordpress_info`

Returns WordPress information: version, site URL, active/inactive plugins with versions, active theme, parent theme, multisite status, key constants (`WP_DEBUG`, `MULTISITE`, etc.), locale, and permalink structure.

### `post_types_info`

Lists registered WordPress post types with their full configuration.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filter` | string | Filter by post type slug (e.g., `"book"`) |
| `include_builtin` | boolean | Include built-in types (`post`, `page`, etc.). Default: `false` |

Returns: label, singular name, public/queryable status, REST support, archive, supports, taxonomies, rewrite rules, menu icon, and capability type.

### `taxonomies_info`

Lists registered WordPress taxonomies with their configuration.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filter` | string | Filter by taxonomy slug |
| `include_builtin` | boolean | Include built-in taxonomies (`category`, `post_tag`). Default: `false` |

Returns: label, singular name, associated post types, public/hierarchical status, REST support, and rewrite rules.

### `registered_hooks`

Lists hooks discovered via `#[Action]` and `#[Filter]` attributes through the Pollora discovery system.

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Filter by `"action"` or `"filter"`. Omit for both |

Returns: class name, method, hook name, and priority for each discovered hook.

### `active_theme_info`

Returns details about the active Pollora theme: directory structure, service providers, config files, Gutenberg blocks (those in `resources/views/blocks`, and separately any left in the deprecated `resources/blocks`), Blade templates, Vite/Tailwind/theme.json status, and the computed theme namespace.

### `discovered_components`

Lists all auto-discovered Pollora components from the discovery system, grouped by type.

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Filter by `"post_type"`, `"taxonomy"`, `"hook"`, `"schedule"`, or `"rest_route"`. Omit for all |

Returns a summary with counts and detailed component information including attribute arguments.

### `wordpress_routes`

Lists all registered routes including `Route::wp()` WordPress condition routes.

Returns: URI, HTTP methods, route name, middleware, WordPress condition and parameters (for `Route::wp()` routes), and a summary with total/WordPress/Laravel route counts.

### `modules_info`

Lists installed Laravel Modules (nwidart) with their name, path, and enabled status. Reports if the module system is not installed.

### `scheduled_events`

Lists the scheduled events with the driver that keeps each (WP-Cron, Action Scheduler, the Laravel queue), read from the `cron` option: no query reaches the driver. Needs framework v13.36 for the `schedule` configuration; on an older 13.x every event reads as WP-Cron's.

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Optional. `"once"` or `"recurring"` |
| `driver` | string | Optional. Only events kept by this driver (`"wp-cron"`, `"action-scheduler"`, `"laravel"`) |
| `limit` | integer | Optional. Maximum number of events, soonest first (default 100) |

Returns the configured driver and whether it is available, the protected hooks, how many events are left to migrate by driver, late events of the `laravel` driver, the failures of the last week, and each event (hook, type, recurrence, next run, driver, arguments, and where it stands: `active`, `protected`, `fallback` or `to migrate`).

### `wp_option`

Reads a WordPress option value by key (read-only).

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | string | **Required.** The option key (e.g., `"blogname"`, `"permalink_structure"`) |

Returns the option value and whether it exists.

#### Secret options are refused

Whatever this tool returns is sent by your AI agent to a model provider. Plugins store API keys, tokens, passwords, licence keys and salts in ordinary options, so `wp_option` does not return them by default.

- **Refused by name.** An option whose name matches `/(pass(word)?|secret|token|api[_-]?key|private[_-]?key|salt|auth|credential|license)/i` is refused, as are the WordPress security keys `auth_key`, `secure_auth_key`, `logged_in_key`, `nonce_key` and `recovery_keys`. The check runs **before** `get_option()` is called, so the answer does not reveal whether the option exists. The message tells the agent why and how to allow it.
- **Masked inside a value.** In an option that holds an array or an object, every sub-key whose name matches the same rule is replaced by `[masqué]`, at any depth. This applies to an allowed option as well.

The match is a substring match on the name, so it can refuse a harmless option (for instance `author_page_layout` contains `auth`). That is deliberate: refusing too much is safe, and you can allow an option explicitly.

#### Allowing a specific option

List the option's exact name under `wp_option.allow` in `config/nectar.php`:

```php
'wp_option' => [
    'allow' => [
        'author_page_layout',
    ],
],
```

Names are matched exactly and case-insensitively. A pattern such as `*` or `my_plugin_*` is never expanded: an entry allows only the option it spells out. Allowing an option does not unmask secret-looking sub-keys inside it, which stay `[masqué]`.

> **Warning: `wp_cli.allowed_commands` is reserved.** `config/nectar.php` still contains a `wp_cli.allowed_commands` list, but no Nectar tool reads or applies it. Do not rely on it to restrict anything, and do not treat its presence as a safeguard.

## AI Guidelines

Guidelines are injected automatically into your AI agent's context when Boost runs. They cover:

- **Pollora architecture** — Laravel-first routing, Blade templates, DDD structure, auto-discovery
- **PHP 8 attributes** — `#[PostType]`, `#[Taxonomy]`, `#[Action]`, `#[Filter]`, `#[Schedule]`, `#[WpRestRoute]`, `#[Ajax]`, `#[Ability]`, `#[SkipDiscovery]`
- **WordPress routing** — `Route::wp()` with template conditions and hierarchy fallback
- **Blade templating** — Sage Directives (`@posts`, `@title`, `@content`, etc.)
- **Theme development** — Convention-based structure, Vite, Tailwind CSS, Asset facade, design tokens in `@theme`, blocks with `<InnerBlocks />`
- **Available Artisan commands** — All Pollora-specific commands

Version-specific guidelines (e.g., Pollora 13.x with Laravel 13 and WordPress 7) are selected automatically from your installed framework version.

### What the Guidelines Teach an Agent

The core guidelines and the `pollora-rest-api` skill carry a few rules an agent applies to every change. Two are about security.

**AJAX actions** (`pollora/ajax` 1.2+). Every `#[Ajax]` action must say how it is protected against cross-site request forgery. The agent is taught to choose, never to leave the parameter out:

```php
#[Ajax('save_note', nonce: true)]                                   // verifies the nonce `save_note`
public function saveNote(): void {}

#[Ajax('track_visit', access: AjaxAccess::GUEST, nonce: false)]     // no nonce, on purpose
public function trackVisit(): void {}
```

- `nonce: true` verifies a nonce named after the action before the method runs (the page creates it with `wp_create_nonce('save_note')` and sends it as `_ajax_nonce`). A request without a valid nonce gets a 403 and the method does not run. `nonce: 'name'` verifies the nonce of that name.
- `nonce: false` turns the check off on purpose, for a public beacon or a webhook that signs its own requests. The agent is told not to use it for an action that changes data without another protection.
- An action that does not choose runs unchecked for now and is reported by WordPress with `WP_DEBUG`. The next major version of `pollora/ajax` verifies a nonce by default.
- An action is never opened to guests without a decision, and the agent no longer adds a `check_ajax_referer()` call in a method that declares `nonce: true`.

**REST routes.** Every `#[WpRestRoute]` declares a permission (`Everyone`, `IsLoggedIn`, `IsAdmin`); a route without one is deprecated in `pollora/wp-rest` 0.2.

**MCP tools.** The guidelines list `wp_option` with its limits and tell the agent not to try to get around a refusal: the project owner allows a safe option by its exact name in `wp_option.allow`.

## Agent Skills

Skills are activated on-demand when working on specific tasks:

| Skill | When It Activates |
|-------|-------------------|
| `pollora-post-types` | Creating custom post types with `#[PostType]` attributes |
| `pollora-taxonomies` | Creating custom taxonomies with `#[Taxonomy]` attributes |
| `pollora-theming` | Theme development (Blade, Vite, Tailwind, assets, theme.json) |
| `pollora-hooks` | Registering WordPress actions and filters with attributes |
| `pollora-blocks` | Gutenberg blocks with JSX/TSX, Blade rendering, `<InnerBlocks />` and Tailwind |
| `pollora-rest-api` | REST API endpoints with `#[WpRestRoute]`, AJAX handlers with `#[Ajax]` |
| `pollora-scheduling` | Scheduled tasks with `#[Schedule]` and the fluent API, on WP-Cron, Action Scheduler or the Laravel queue |
| `pollora-modules` | Laravel Modules (nwidart) with auto-discovery integration |
| `pollora-abilities` | WordPress Abilities API with `#[Ability]` and the `Ability` facade |
| `pollora-typed-meta` | Post, term, user and comment meta as typed properties with `#[Meta]`, `Meta::of()`, `pollora:meta:audit` |
| `pollora-roles` | Capabilities through Laravel's Gate, roles declared with `#[Role]`, `#[ModifyRole]`, `#[CapabilitySet]`, `pollora:roles:*` |
| `pollora-block-bindings` | `#[BlockBinding]` sources, `pollora/post-meta` and the other sources for core blocks, bindable Blade blocks |

Each skill provides detailed instructions, code examples, and best practices specific to its domain.

## Upgrade Assistance

Nectar includes MCP upgrade prompts that guide AI agents through Pollora version upgrades. Prompts are **automatically registered** when the current project version matches — no manual activation needed.

### Available Upgrade Prompts

| Prompt | Available When | What It Covers |
|--------|---------------|----------------|
| `upgrade-pollora-v13` | Pollora 12.x detected | Full migration guide from Pollora 12 to 13 |
| `upgrade-pollora-v13-32` | Pollora 13.0 up to, but not including, 13.32 detected | Migration to 13.32: dependencies, composer-patches 2, renamed commands, extracted packages, blocks, skeleton files, theme.json |

### Pollora 12→13 Upgrade Coverage

The `upgrade-pollora-v13` prompt covers all breaking changes and migration steps:

- **Loop facade removal** (v13.2) — migration to Sage Directives (`@title`, `@content`, `@excerpt`)
- **Config-based registration removal** (v13.4) — migration from `config/post-types.php` and `config/taxonomies.php` to `#[PostType]` and `#[Taxonomy]` attribute classes
- **`@theme` Blade directive removal** (v13.4) — conflicts with Tailwind CSS v4 `@theme` at-rule
- **Route model namespace change** (v13.4) — from `Domain\Models` to `Infrastructure\Models`
- **CSRF middleware rename** (Laravel 13) — `ValidateCsrfToken` to `PreventRequestForgery` with WordPress-specific route exclusions
- **Theme Vite configuration** — `@roots/vite-plugin` with `wordpressThemeJson` for font and theme.json resolution (without copying the built `theme.json` back over the theme's own)
- **WordPress 7.0 update**
- **Post-upgrade cleanup** — cache clearing, discovery rebuild, `wp transient delete --all`

The prompt follows a systematic 6-step process: assess → create safety net → analyze codebase → apply changes → update dependencies → clean up and verify.

### Pollora 13.x → 13.32 Upgrade Coverage

The `upgrade-pollora-v13-32` prompt covers the move from 13.0 up to 13.31 to 13.32, whose version now tracks the Laravel release it targets:

- **Dependencies** — `laravel/framework` `^13.35` (`pollora/framework` 13.35 requires it), `johnpbloch/wordpress` `^7.1`, WordPress packages from wp-packages instead of wpackagist
- **WordPress patching** — `cweagans/composer-patches` 2, `patches.lock.json`, the `patches-relock` script, and checking that WordPress's `__()` is patched
- **Renamed Artisan commands** — the colon convention (`pollora:make:theme`…), former names kept as aliases
- **Extracted packages** — the classes moved to `pollora/hook`, `pollora/option` and `pollora/ajax`
- **`Loop` and `Query` removal**, **`Translater` domain**, **blocks** in `resources/views/blocks`, registered without a service provider
- **Skeleton files** — `routes/web.php`, the cache table, the trailing-slash rule of `.htaccess`
- **Themes** — design tokens in `@theme static`, generated `theme.json`

### Version-Specific Guidelines

`resources/boost/guidelines/framework.blade.php` holds the Pollora 12.x and 13.x guidelines and gives a project the ones of its installed framework major (a branch without an alias gets the 13.x text). Boost reads every guideline file of a package outside Laravel's own, so versions live in one file rather than in `12/` and `13/` folders.

## Configuration

Publish the configuration file:

```bash
php artisan vendor:publish --tag=nectar-config
```

```php
// config/nectar.php
return [
    // Enable/disable Nectar entirely
    'enabled' => env('NECTAR_ENABLED', true),

    'mcp' => [
        'tools' => [
            // Tool class names to exclude from the MCP server
            'exclude' => [],
            // Additional tool class names to include
            'include' => [],
        ],
    ],

    // Exact names of options that wp_option may show despite a secret-looking name
    'wp_option' => [
        'allow' => [],
    ],
];
```

See [Secret options are refused](#secret-options-are-refused) for what `wp_option` hides and how to allow an option. The `wp_cli.allowed_commands` key that the published file also holds is reserved and applied by no tool.

Disable Nectar via environment variable:

```env
NECTAR_ENABLED=false
```

## Architecture

Nectar extends Laravel Boost with three layers:

```
┌────────────────────────────────────────────────────┐
│                   Laravel Boost                     │
│              (AI context framework)                 │
└───────────┬──────────────┬──────────────┬──────────┘
            │              │              │
┌───────────▼──┐  ┌────────▼───────┐  ┌──▼──────────┐
│  Guidelines  │  │    Skills      │  │  MCP Server  │
│  (Blade)     │  │  (Markdown)    │  │  (Tools +    │
├──────────────┤  ├────────────────┤  │   Prompts)   │
│ core.blade   │  │ pollora-hooks  │  ├─────────────┤
│ framework    │  │ pollora-blocks │  │ 11 tools    │
│   .blade     │  │ pollora-theme  │  │ 2 upgrade   │
│              │  │ ...6 more      │  │   prompts   │
└──────────────┘  └────────────────┘  └─────────────┘
```

### Key Design Decisions

- **Read-only tools**: All MCP tools are annotated with `#[IsReadOnly]` — they never modify your application state
- **Discovery integration**: Hooks and components are retrieved via `DiscoveryManager`, ensuring consistency with the framework's own discovery system
- **Environment-gated**: Nectar only loads in `local`/`development` environments to avoid any production overhead
- **WordPress safety**: Tools that depend on wp-admin functions (like `get_plugins()`) safely load required files before calling them

## See also

- [AI coding agents for WordPress projects](https://pollora.dev/guides/ai-coding-agents-wordpress/)
