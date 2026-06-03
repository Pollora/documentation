# Nectar — AI Development Context

Nectar is an AI-powered development companion for Pollora. Built on top of [Laravel Boost](https://laravel.com/docs/boost), it provides AI guidelines, agent skills, and a dedicated MCP server that gives AI agents deep context about your Pollora application.

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [MCP Server](#mcp-server)
- [Available MCP Tools](#available-mcp-tools)
- [AI Guidelines](#ai-guidelines)
- [Agent Skills](#agent-skills)
- [Configuration](#configuration)
- [Architecture](#architecture)

## Overview

When working with AI coding assistants (Claude Code, Cursor, Windsurf, etc.), the quality of generated code depends heavily on the context available. Nectar bridges this gap by:

- **Injecting Pollora-specific guidelines** into your AI agent's context via Laravel Boost
- **Providing 8 on-demand skills** for domain-specific tasks (post types, themes, hooks, etc.)
- **Exposing 10 MCP tools** for live introspection of your WordPress and Pollora environment

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

Returns details about the active Pollora theme: directory structure, service providers, config files, registered Gutenberg blocks, Blade templates, Vite/Tailwind/theme.json status, and the computed theme namespace.

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

### `wp_option`

Reads a WordPress option value by key (read-only).

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | string | **Required.** The option key (e.g., `"blogname"`, `"permalink_structure"`) |

Returns the option value and whether it exists.

## AI Guidelines

Guidelines are injected automatically into your AI agent's context when Boost runs. They cover:

- **Pollora architecture** — Laravel-first routing, Blade templates, DDD structure, auto-discovery
- **PHP 8 attributes** — `#[PostType]`, `#[Taxonomy]`, `#[Action]`, `#[Filter]`, `#[Schedule]`, `#[WpRestRoute]`
- **WordPress routing** — `Route::wp()` with template conditions and hierarchy fallback
- **Blade templating** — Sage Directives (`@posts`, `@title`, `@content`, etc.)
- **Theme development** — Convention-based structure, Vite, Tailwind CSS, Asset facade
- **Available Artisan commands** — All Pollora-specific commands

Version-specific guidelines (e.g., Pollora 13.x with Laravel 13 and WordPress 6.9) are loaded automatically based on your installed framework version.

## Agent Skills

Skills are activated on-demand when working on specific tasks:

| Skill | When It Activates |
|-------|-------------------|
| `pollora-post-types` | Creating custom post types with `#[PostType]` attributes |
| `pollora-taxonomies` | Creating custom taxonomies with `#[Taxonomy]` attributes |
| `pollora-theming` | Theme development (Blade, Vite, Tailwind, assets, theme.json) |
| `pollora-hooks` | Registering WordPress actions and filters with attributes |
| `pollora-blocks` | Gutenberg block development with JSX/TSX and Tailwind |
| `pollora-rest-api` | REST API endpoints with `#[WpRestRoute]` |
| `pollora-scheduling` | Scheduled tasks with `#[Schedule]` and WordPress cron |
| `pollora-modules` | Laravel Modules (nwidart) with auto-discovery integration |

Each skill provides detailed instructions, code examples, and best practices specific to its domain.

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
];
```

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
│  (Blade)     │  │  (Markdown)    │  │  (Tools)     │
├──────────────┤  ├────────────────┤  ├─────────────┤
│ core.blade   │  │ pollora-hooks  │  │ 10 tools    │
│ 13/core.blade│  │ pollora-blocks │  │ read-only   │
│              │  │ pollora-theme  │  │ live data   │
│              │  │ ...6 more      │  │             │
└──────────────┘  └────────────────┘  └─────────────┘
```

### Key Design Decisions

- **Read-only tools**: All MCP tools are annotated with `#[IsReadOnly]` — they never modify your application state
- **Discovery integration**: Hooks and components are retrieved via `DiscoveryManager`, ensuring consistency with the framework's own discovery system
- **Environment-gated**: Nectar only loads in `local`/`development` environments to avoid any production overhead
- **WordPress safety**: Tools that depend on wp-admin functions (like `get_plugins()`) safely load required files before calling them