# Creating a Theme

## Introduction

The Pollora framework offers a robust and flexible system for creating and managing WordPress themes. This guide will help you understand how to create, customize, and use themes with Pollora.

## Creating a New Theme

### Generating a New Theme

To generate a new theme, run the following command:

```bash
php artisan pollora:make:theme
```

You'll be prompted to answer several questions to configure your theme. Alternatively, you can pass the configuration as options:

```bash
php artisan pollora:make:theme {theme-name} \
  --theme-author="Author Name" \
  --theme-author-uri="https://author.com" \
  --theme-uri="https://theme.com" \
  --theme-description="Theme description" \
  --theme-version="1.0.0"
```

### Command Options

- `--theme-author` : Theme author name
- `--theme-author-uri` : Theme author URI
- `--theme-uri` : Theme URI
- `--theme-description` : Theme description
- `--theme-version` : Theme version
- `--repository` : GitHub repository to download (owner/repo format)
- `--repo-version` : Specific version/tag to download
- `--force` : Force create theme with same name

This command creates a new theme with the necessary folder structure and base files.

### Theme Structure

A typical Pollora theme has the following structure:

```plaintext
theme-name/
├─ config/
│  ├─ gutenberg.php
│  ├─ images.php
│  ├─ login.php
│  ├─ menus.php
│  ├─ providers.php
│  ├─ sidebars.php
│  ├─ supports.php
│  ├─ templates.php
├─ assets/
│  └─ css/
│      └─ app.css
│  └─ fonts/
│  └─ images/
│  └─ js/
│      `└─ `bootstrap.js
│  └─ app.js
├─ views/
│  ├─ example.blade.php
│  ├─ layouts/
│  ├─ parts/
│  ├─ patterns/
│  ├─ home.blade.php
│  ├─ page.blade.php
│  └─ post.blade.php
├─ favicon.png
├─ index.php
├─ package.json
├─ style.css
├─ theme.json
└─ vite.config.js
```

### Theme Registration (functions.php)

The theme's `functions.php` file registers the theme with the Pollora framework using the `pollora_register()` helper:

```php
<?php

declare(strict_types=1);

use Pollora\Modules\Domain\Enums\ModuleType;

pollora_register(ModuleType::Theme);
```

That's it — three lines. The `pollora_register()` helper:
- Auto-detects the active theme name and path from WordPress
- Resolves the correct registrar (`ThemeRegistrarInterface`)
- Triggers automatic discovery of all PHP attributes, service providers, post types, hooks, etc.

For themes, no additional arguments are needed — the framework reads the theme name and directory from WordPress automatically.

## Theme.json and Vite Build Integration

Pollora provides automatic integration between the Vite build process and WordPress's `theme.json` system. Tailwind CSS variables are extracted at build time and injected into WordPress at runtime — the block editor receives your full design token palette without manual synchronization.

### How It Works

The `@roots/vite-plugin` package includes a `wordpressThemeJson` plugin that:

1. Reads your source `theme.json` from the theme root (base configuration)
2. Extracts CSS variables from the compiled Tailwind CSS (colors, font families, font sizes, border radius)
3. Generates an enriched `theme.json` in the build output at `public/build/theme/{slug}/assets/theme.json`

Pollora's `ThemeJsonResolver` hooks into WordPress's `wp_theme_json_data_theme` filter (WP 6.1+) to inject this built data at runtime. WordPress receives the full Tailwind-enriched theme settings dynamically.

### Source theme.json

Your source `theme.json` at the theme root serves as the **base configuration** — layout sizes, appearance tools, and any settings not derived from CSS:

```json
{
    "$schema": "https://schemas.wp.org/wp/6.0/theme.json",
    "version": 2,
    "settings": {
        "appearanceTools": true,
        "layout": {
            "contentSize": "1024px",
            "wideSize": "1275px"
        },
        "typography": {
            "dropCap": false
        }
    }
}
```

### Vite Configuration

In your `vite.config.js`, include the `wordpressThemeJson` plugin:

```javascript
import { wordpressThemeJson } from '@roots/vite-plugin';

export default defineConfig({
    plugins: [
        tailwindcss(),
        laravel(getThemeConfig()),
        wordpressThemeJson({
            baseThemeJsonPath: './theme.json',
        }),
    ],
});
```

When you run `npm run build`, the plugin merges your base `theme.json` with all Tailwind CSS variables extracted from the compiled stylesheet. The result is written to the build directory.

### What Gets Injected

The built `theme.json` includes:

- **Colors** — All Tailwind color palette shades (e.g., Red 50–950, Blue 50–950)
- **Font sizes** — Tailwind's typography scale (`xs` through `9xl`)
- **Font families** — `sans`, `serif`, `mono` from Tailwind defaults or your custom fonts
- **Border radius** — Tailwind's radius scale (`xs` through `4xl`)

These values are available in the WordPress block editor (Gutenberg) — users can select Tailwind colors, font sizes, and border radii directly from the editor UI.

### Architecture

The resolution follows the hexagonal architecture pattern used throughout the framework:

| Layer | Class | Role |
|-------|-------|------|
| Domain | `ThemeJsonResolverInterface` | Contract for resolving built theme.json data |
| Infrastructure | `ThemeJsonResolver` | Reads from `public/build/theme/{slug}/assets/theme.json` with in-memory cache |
| Provider | `ThemeServiceProvider` | Registers the service and hooks into `wp_theme_json_data_theme` |

The resolver is registered as a singleton and uses in-memory caching — the filesystem is read at most once per request per theme.

## Template Hierarchy

Pollora extends WordPress's template hierarchy system to provide a more flexible and powerful theming experience. This system determines which template file should be used for the current request.

### Understanding Template Hierarchy

The template hierarchy is a list of possible template files arranged from most specific to most generic. The system searches through this list until it finds a matching template file.

### Enhanced Template System

Pollora's template system offers several advantages over WordPress's standard template hierarchy:

1. **Blade Integration**: Templates can be written using Laravel's Blade templating engine, offering features like layouts, components, and directives.

2. **Dynamic Registration**: Plugins can dynamically register custom template types through the `TemplateHierarchy` class.

3. **Block Theme Support**: The system automatically checks for block theme templates (.html files) when appropriate.

4. **Performance Optimization**: Templates can be cached for improved performance.

### Accessing the Template Hierarchy

You can access the template hierarchy in your views or controllers through dependency injection:

```php
// In a controller
use Pollora\Theme\TemplateHierarchy;

class PageController extends Controller
{
    private TemplateHierarchy $templateHierarchy;

    public function __construct(TemplateHierarchy $templateHierarchy)
    {
        $this->templateHierarchy = $templateHierarchy;
    }

    public function show()
    {
        $hierarchy = $this->templateHierarchy->hierarchy();
        
        return view('page', ['templateHierarchy' => $hierarchy]);
    }
}
```

```blade
{{-- In a Blade view --}}
<div class="debug-info">
    <h3>Template Hierarchy</h3>
    <ul>
        @foreach($templateHierarchy as $template)
            <li>{{ $template }}</li>
        @endforeach
    </ul>
</div>
```

### Extending the Template Hierarchy

Plugins can extend the template hierarchy for specific content types. You can inject the `TemplateHierarchy` class into your service providers or use the container to resolve it:

```php
// In a plugin or theme service provider
use Pollora\Theme\TemplateHierarchy;

class ThemeServiceProvider extends ServiceProvider
{
    public function boot(TemplateHierarchy $templateHierarchy)
    {
        // Register a custom template handler for product pages on sale
        $templateHierarchy->registerTemplateHandler('product_on_sale', function($queriedObject) {
            if (!$queriedObject || !function_exists('wc_get_product')) {
                return [];
            }
            
            $product = wc_get_product($queriedObject->ID);
            if (!$product || !$product->is_on_sale()) {
                return [];
            }
            
            return [
                "product-on-sale-{$product->get_slug()}.blade.php",
                'product-on-sale.blade.php',
            ];
        });

        // Add the corresponding condition
        add_filter('pollora/template_hierarchy/conditions', function($conditions) {
            $conditions['is_product_on_sale'] = 'product_on_sale';
            return $conditions;
        });
    }
}
```

## Vite Configuration

The `vite.config.js` file is automatically generated and configured for your theme. It includes:

- Automatic theme name detection
- Configuration for development and production
- Integration with the Laravel Vite plugin

```javascript
import { defineConfig } from "vite";
import laravel from "laravel-vite-plugin";
import path from 'path';

const isDevelopment = !!process.env.DDEV_PRIMARY_URL;
const port = 5173;
const publicDirectory = path.resolve(__dirname, "../../public");
const themeName = path.basename(__dirname);

// ... (detailed configuration)

export default defineConfig({
    plugins: [
        laravel(getThemeConfig()),
        // ... (other plugins)
    ],
    ...getDevServerConfig()
});
```

## Tailwind CSS Integration

Pollora themes use **Tailwind CSS v4** with the `@tailwindcss/vite` plugin. No `tailwind.config.js` or `postcss.config.mjs` is needed — Tailwind v4 auto-detects source files from the Vite module graph.

### Setup

The `package.json` includes the necessary dependencies:

```json
{
    "devDependencies": {
        "@tailwindcss/vite": "^4.2.3",
        "tailwindcss": "^4.2.3",
        "vite": "^8.0.9"
    }
}
```

The Vite plugin handles everything — add it in `vite.config.js`:

```js
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        tailwindcss(),
        // ...
    ],
});
```

### CSS Entry Point

Your main CSS file uses a single import:

```css
/* resources/assets/app.css */
@import "tailwindcss";
```

Tailwind v4 automatically scans all project files (HTML, PHP, JSX, Blade templates) for utility classes and generates only the CSS needed.

### Tailwind in Gutenberg Blocks

Block CSS files (`style.css`, `editor.css`) support Tailwind via two directives:

- **`@import "tailwindcss" source(".")`** in `style.css` — imports Tailwind scoped to the block's directory. This generates utility classes found in the block's JSX files, ensuring they work both on the frontend and in the block editor.
- **`@reference "tailwindcss"`** in `editor.css` — gives access to `@apply` without generating utilities (editor-only styles typically use `@apply`, not utility classes in JSX).

```css
/* resources/views/blocks/hero/style.css */
@import "tailwindcss" source(".");

.wp-block-my-theme-hero {
    @apply relative py-24 px-8 overflow-hidden;
}
```

> See [blocks.md](blocks.md) for detailed examples of Tailwind usage in Gutenberg blocks.

## Theme Management

Pollora's `ThemeManager` offers several useful methods for managing themes:

- `Theme::load($themeName)`: Loads a specific theme
- `Theme::getAvailableThemes()`: Retrieves the list of available themes
- `Theme::active()`: Returns the name of the active theme
- `Theme::parent()`: Returns the name of the parent theme (if it's a child theme)
- `Theme::path($path)`: Generates the full path to a file in the active theme
- `Theme::asset($path, $assetType = '')`: Retrieves the URL for a theme asset

### Using Theme Assets

The framework provides an intuitive way to manage and reference theme assets through the `Asset` facade. The `Asset` facade ensures you can easily retrieve URLs for assets in the active theme's container.

#### Accessing Theme Assets

You can reference theme assets with the following examples:

```php
use Pollora\Support\Facades\Asset;

// Get the URL for an image in the theme
$logoUrl = (string)Asset::url('assets/images/logo.png'); // the string cast is necessary to return the url

// Get the URL for a CSS file in the theme
$styleUrl = (string)Asset::url('assets/css/app.css');

// Get the URL for a JavaScript file in the theme
$scriptUrl = (string)Asset::url('assets/js/app.js');
```

#### WordPress's own theme URL functions

`get_theme_file_uri()` works too, and resolves through the same build:

```php
get_theme_file_uri('resources/assets/app.js');
// https://example.test/build/theme/my-theme/assets/app-DcI6_eae.js

get_theme_file_uri('fonts/Inter-Regular.woff2');   // relative to the container root
// https://example.test/build/theme/my-theme/assets/Inter-Regular-B0QUfDW0.woff2
```

Both spellings are accepted: the path from the theme's root, and the path
relative to the asset container's root (`resources/assets/` by default).

A file the build does not know about is handed back with WordPress's own
answer, unchanged. That answer is **not fetchable**: a theme's own directory
is not web-served on a Pollora project — only the build output under
`/build/theme/{slug}` is. If you need a URL for a file, make it part of the
Vite build.

The same applies to `get_stylesheet_directory_uri()` and
`get_template_directory_uri()`: they answer a URL, but not one that serves
your theme's files. Use `Asset::url()` or `get_theme_file_uri()`.

#### Explicitly Specifying the Theme Container

Although the framework defaults to the active theme's container, you can explicitly specify it using the `from('theme')` method for clarity:

```php
use Pollora\Support\Facades\Asset;

// Explicitly specify the "theme" container
$logoUrl = (string)Asset::url('assets/images/logo.png')->from('theme'); // the string cast is necessary to return the url
$styleUrl = (string)Asset::url('assets/css/app.css')->from('theme');
$scriptUrl = (string)Asset::url('assets/js/app.js')->from('theme');
```

#### Blade Integration for Theme Assets

You can reference theme assets directly in your Blade templates, with or without specifying the container explicitly:

```blade
<img src="{{ Asset::url('assets/images/logo.png') }}">
<link rel="stylesheet" href="{{ Asset::url('assets/css/app.css') }}">
<script src="{{ Asset::url('assets/js/app.js') }}"></script>
```

Or, explicitly specify the theme container:

```blade
<img src="{{ Asset::url('assets/images/logo.png')->from('theme') }}">
<link rel="stylesheet" href="{{ Asset::url('assets/css/app.css')->from('theme') }}">
<script src="{{ Asset::url('assets/js/app.js')->from('theme') }}">
```

## Localization

A theme can translate through WordPress's own `.po`/`.mo` catalogues (a `languages/` directory, compiled automatically by `pollora:make:theme`) and through a Laravel-namespaced catalogue (a `lang/` directory, registered automatically as `{theme-name}::group.key`) at the same time — a single `__()` call routes to the right one. See the dedicated [Translations](translations.md) guide for how the routing works and when to use which.

## Login screen

WordPress loads your theme on `wp-login.php` — `functions.php` runs, the theme
is active — but it emits none of your design there. Measured on a site running
a theme with 304 colours in its `theme.json`: zero occurrences of
`wp--preset--color` in the HTML of the login screen. So every WordPress site,
whatever it looks like, has always signed people in through the same grey form.

Pollora prints your theme's design on that screen instead. It is **opt-in**: a
theme with no `config/login.php` gets WordPress's screen unchanged, byte for
byte. Add the file and the screen is yours.

```php
<?php
// themes/your-theme/config/login.php

return [
    'enabled' => true,

    'logo' => [
        // A path inside the theme, read from disk and inlined. An absolute
        // URL, a site-root path and an attachment id all work too.
        'source' => 'resources/assets/images/logo.svg',
        'width' => 220,
        // 'height' is derived from the file's own viewBox or dimensions.
        'url' => home_url('/'),
        'text' => get_bloginfo('name', 'display'),
    ],

    'powered_by' => true,
];
```

Colours, radii and typography are **not** in this file. They are already in
`theme.json`, which is where your theme keeps its design, and Pollora reads
them from there — restyling the theme restyles its login screen, with nothing
to keep in sync.

Everything the login screen covers is one screen as far as `login_head` is
concerned, so sign-in, lost password, password reset, registration and the
confirm-admin-email prompt are all styled together.

### Design roles

A palette is a vocabulary, not a set of roles: one theme calls a shade
`primary-hover`, another calls it `primary-vivid`, and a Tailwind-built
`theme.json` buries both under three hundred primitives named `red-500` and
the like. So the login screen states the roles it needs, and each role names
the preset slugs it will accept, most specific first:

| Role | Preset slugs accepted, in order |
|---|---|
| `background` | `background`, `surface-alt`, `surface`, `base` |
| `surface` | `surface`, `background`, `base` |
| `surface-alt` | `surface-alt`, `surface`, `background` |
| `foreground` | `foreground`, `contrast`, `ink` |
| `muted` | `muted`, `subtle`, `gray-500`, `grey-500` |
| `primary` | `primary`, `brand`, `accent` |
| `primary-hover` | `primary-hover`, `primary-vivid`, `primary` |
| `accent` | `accent`, `secondary`, `primary` |
| `outline` | `outline`, `border`, `gray-200`, `grey-200` |
| `danger` | `error`, `danger`, `red-600` |
| `success` | `success`, `green-600` |
| `radius` | `lg`, `md`, `xl` (from `settings.border.radiusSizes`) |
| `radius-sm` | `md`, `sm`, `xs` |
| `font` | `body`, `sans`, `inter-var`, `base` (from `settings.typography.fontFamilies`) |
| `heading-font` | `display`, `heading`, `body`, `sans` |

A theme that speaks this vocabulary configures nothing. One that names things
its own way points a role at its own slugs, or writes a value in:

```php
'tokens' => [
    'primary' => ['brand-600', 'brand-500'],  // slugs to try, in order
    'accent' => 'oklch(70% .2 30)',           // a value no preset holds
],
```

Every role resolves to something: a theme with no `theme.json` at all still
gets a coherent screen rather than half a stylesheet over WordPress's defaults.

A preset whose value points at a CSS variable — `var(--wp--preset--color--x,
#1f2937)`, which Tailwind-built theme.json files generate — resolves to its
declared fallback, because the login screen carries none of the stylesheets
those variables come from. With no fallback to read, the role falls through to
the next slug it accepts.

### The logo

A logo file inside your theme has no URL on a Pollora site: only the Vite build
output is web-served. Pollora therefore reads the file and inlines it, which
costs one file read on a screen served rarely and cannot 404. SVG, PNG, JPEG,
GIF, WebP and AVIF are inlined, up to 96 KB.

Four kinds of source are understood:

| `source` | What happens |
|---|---|
| `'resources/assets/images/logo.svg'` | read from the theme directory and inlined |
| `'https://…'`, `'//…'`, `'/content/uploads/…'` | used as the address it is |
| `42` (an attachment id) | resolved through the media library |
| omitted | the site's `custom_logo`, if it set one; otherwise WordPress's |

Give `width` or `height` and the other is derived from the file's own
proportions, so a wide wordmark is not squeezed into WordPress's 84×84 box.

### Taking over from a module or a plugin

Four filters cover the screen without touching the theme:

| Filter | Value |
|---|---|
| `pollora/login/palette` | the resolved roles, as `role => CSS value` |
| `pollora/login/logo` | a `ResolvedLogo`, or `null` |
| `pollora/login/styles` | the complete stylesheet, before it is printed |
| `pollora/login/credit` | the footer mention's HTML; return `null` to drop it |

Returning an empty string from `pollora/login/styles` prints nothing at all.

## Theme Development

Commands needs to be run inside the theme folder.

1. Install dependencies:
   ```bash
   yarn # or 'npm install'
   ```

2. For development, use:
   ```bash
   yarn dev # or 'npm run dev'
   ```

3. For production, build the assets with:
   ```bash
   yarn build # or 'npm run build'
   ```

## Best Practices

- Use the provided folder structure to organize your code
- Take advantage of TailwindCSS for rapid and consistent CSS development
- Use the `ThemeManager` methods for efficient theme management
- Consider parent theme compatibility when developing
- Leverage the template hierarchy to create structured and maintainable views

## Asset Management

For a comprehensive guide on asset management (registering scripts, styles, containers, and Vite integration), see the dedicated [Assets documentation](assets.md).

### Quick Reference

Use the `Asset` facade to register and manage theme assets:

```php
use Pollora\Support\Facades\Asset;

// Register a script
Asset::add('theme-app', 'assets/js/app.js')
    ->toFrontend();

// Register a style
Asset::add('theme-style', 'assets/css/app.css')
    ->toFrontend();

// Get an asset URL
$logoUrl = Asset::url('assets/images/logo.png');
```

These macros ensure that your asset references are consistent with your theme's configuration and take advantage of Vite's asset handling capabilities.

## Theme Service Providers

To register custom functionality for your theme, you can create service providers in the `config/providers.php` file:

```php
<?php
// config/providers.php

return [
    // Your custom service providers
    App\Providers\ThemeServiceProvider::class,
    App\Providers\EventServiceProvider::class,
];
```

Then, in your service provider, you can extend the template hierarchy using dependency injection:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Pollora\Theme\TemplateHierarchy;

class ThemeServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Register any theme-specific services
    }

    public function boot(TemplateHierarchy $templateHierarchy)
    {
        // Register template handlers
        $templateHierarchy->registerTemplateHandler('featured_post', function($post) {
            if (!$post || !has_term('featured', 'post_tag', $post)) {
                return [];
            }
            
            return [
                'featured-post.blade.php',
                'post-featured.blade.php',
                'single-featured.blade.php',
            ];
        });
        
        // Add the corresponding condition
        add_filter('pollora/template_hierarchy/conditions', function($conditions) {
            $conditions['is_featured_post'] = 'featured_post';
            return $conditions;
        });
        
        // Define the conditional function
        if (!function_exists('is_featured_post')) {
            function is_featured_post() {
                return is_single() && has_term('featured', 'post_tag');
            }
        }
        
        // Use the template hierarchy to share data with views
        add_action('template_redirect', function() use ($templateHierarchy) {
            // Share the template hierarchy with all views
            view()->share('templateHierarchy', $templateHierarchy->hierarchy());
        });
    }
}
```

## Advanced Template Hierarchy Usage

### Caching Template Hierarchy

For performance optimization, you can cache the template hierarchy:

```php
// In a service provider
public function boot(TemplateHierarchy $templateHierarchy)
{
    // Cache the hierarchy for performance (set to true)
    add_action('template_redirect', function() use ($templateHierarchy) {
        $shouldCache = config('wordpress.template_caching', false);
        $templateHierarchy->finalizeHierarchy($shouldCache);
    }, 999);
}
```

### Modifying Template Priority

You can change the order in which templates are checked:

```php
add_filter('pollora/template_hierarchy/order', function($hierarchyOrder) {
    // Example: Give higher priority to is_author condition
    if (($key = array_search('is_author', $hierarchyOrder)) !== false) {
        unset($hierarchyOrder[$key]);
        array_unshift($hierarchyOrder, 'is_author');
    }
    return $hierarchyOrder;
});
```

### Debugging Template Hierarchy

During development, you might want to see which templates are being checked:

```php
// In a development-only service provider
public function boot(TemplateHierarchy $templateHierarchy)
{
    if (config('app.debug')) {
        add_action('template_redirect', function() use ($templateHierarchy) {
            // Force refresh the hierarchy to ensure it's up to date
            $hierarchy = $templateHierarchy->hierarchy(true);
            
            // Add debug information to footer
            add_action('wp_footer', function() use ($hierarchy) {
                echo '<div class="debug-hierarchy" style="background:#f1f1f1;padding:15px;margin-top:30px;border-top:1px solid #ddd;">';
                echo '<h3>Template Hierarchy</h3>';
                echo '<ol>';
                foreach ($hierarchy as $template) {
                    echo '<li>' . esc_html($template) . '</li>';
                }
                echo '</ol>';
                echo '</div>';
            }, 999);
        });
    }
}
```

## Using Block Theme Templates

Pollora supports WordPress block theme templates. When using a block theme, the template hierarchy automatically includes HTML templates from the block theme structure:

```php
// For a regular PHP template like 'page.php'
// These templates will be checked:
// - page.blade.php (Blade variant)
// - page.php (PHP variant)
// - wp-templates/page.html (Block theme variant)
```
