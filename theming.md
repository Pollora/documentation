# Creating a Theme

## Introduction

The Pollora framework offers a robust and flexible system for creating and managing WordPress themes. This guide will help you understand how to create, customize, and use themes with Pollora.

## Creating a New Theme

### Generating a New Theme

To generate a new theme, run the following command:

```bash
php artisan pollora:make:theme
```

You'll be prompted to answer several questions to configure your theme, starting with the template to start from:

| Template | Repository | What it is |
|---|---|---|
| Default | `pollora/theme-default` | A Blade starter: Vite, Tailwind CSS |
| E-commerce | `pollora/theme-apiary` | A WooCommerce storefront, Blade and Alpine.js |
| Magazine | `pollora/theme-buzz` | A Full Site Editing block theme — see [Block Themes](#block-themes-full-site-editing) |
| Custom | any `owner/repo` | Your own template, from GitHub |

`--repository=pollora/theme-buzz` skips the prompt. The command downloads the template's latest tag and fills in its placeholders (name, namespace, pattern slugs, `style.css` header). Alternatively, you can pass the configuration as options:

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
- `--activate` : Activate the generated theme without asking
- `--no-activate` : Leave the active theme as it is, without asking

This command creates a new theme with the necessary folder structure and base files.

It activates the new theme only where the site needs one: a site with no usable theme — a first install — gets it without a question; a site that already has one is asked, **No** by default, so `--no-interaction` never replaces a working theme. `--activate` and `--no-activate` decide without asking. Activation goes through WordPress's `switch_theme()`, so `after_switch_theme` runs.

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

Your theme's design lives in one place: the `@theme` block of
`resources/assets/css/app.css`. At build time, the colours, font sizes, fonts
and radii declared there are written into `theme.json`, and Pollora hands that
file to WordPress. Change a value in `app.css`, run the build, and the front end
and the block editor change together.

### How It Works

The `@roots/vite-plugin` package includes a `wordpressThemeJson` plugin that:

1. Reads your source `theme.json` from the theme root (the base configuration)
2. Reads the `@theme` variables of the compiled stylesheet, in four families:

   | Variable | Written to |
   |---|---|
   | `--color-*` | `settings.color.palette` |
   | `--text-*` | `settings.typography.fontSizes` |
   | `--font-*` | `settings.typography.fontFamilies` |
   | `--radius-*` | `settings.border.radiusSizes` |

3. Writes the merged result to `public/build/theme/{slug}/assets/theme.json`

Pollora's `ThemeJsonResolver` hooks into WordPress's `wp_theme_json_data_theme`
filter (WP 6.1+) to inject this built file at runtime.

### Declaring the design in `@theme`

```css
@import "tailwindcss";

@theme static {
    --color-primary: #1f2937;
    --color-primary-hover: #111827;
    --color-surface: #f9fafb;

    --text-sm: 0.875rem;
    --text-base: 1rem;
    --text-lg: 1.125rem;

    --radius-md: 0.375rem;
    --radius-lg: 0.5rem;
}

@layer base {
    :root {
        --color-primary: var(--wp--preset--color--primary, #1f2937);
        --color-primary-hover: var(--wp--preset--color--primary-hover, #111827);
        --color-surface: var(--wp--preset--color--surface, #f9fafb);
    }
}
```

Three rules make this work:

- **`@theme static`, and a plain `@import "tailwindcss"`.** Tailwind only emits
  the variables your templates use, and the plugin only sees what Tailwind
  emits. `static` on your own `@theme` block emits every token you declare, so
  each one reaches the editor. Do not write `@import "tailwindcss" theme(static)`:
  it emits Tailwind's whole default theme, and the editor then offers some 290
  colours (`red-50` to `stone-950`) and thirteen font sizes nobody chose.
- **Concrete values in `@theme`.** The plugin copies each value as it is. A token
  written `var(--wp--preset--color--primary, #1f2937)` becomes a preset defined
  as itself, a cycle CSS discards: a block coloured "Primary" is transparent.
- **The `:root` rule points the colour utilities at the presets.** `bg-primary`
  then follows the palette, so a colour changed in the Site Editor reaches the
  whole theme, not only the blocks. The fallback keeps the utilities working
  where WordPress prints no presets.

Everything a template uses and `@theme` does not declare also lands in
`theme.json` (`text-white` brings `white`, for instance). Keep the templates on
your tokens and the editor shows your design and nothing else.

### Source theme.json

The `theme.json` at the theme root is the **base configuration**: layout
sizes, appearance tools, and anything CSS does not express.

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
            "dropCap": false,
            "defaultFontSizes": false,
            "customFontSize": false
        }
    }
}
```

When the base and `@theme` both define a slug, **the base wins**. Leave the
palette, font sizes, fonts and radii out of it unless you mean to override
them, and never copy the built file back over it: the base would then hold
every generated value, and later edits to `app.css` would never reach the
editor.

### Keeping a value out of Tailwind

You do not have to tie `theme.json` to Tailwind. Two ways, from the narrowest:

- **One slug.** Define it in the base `theme.json`: it wins over the value of
  `@theme`, and the rest of the family is still generated.
- **A whole family.** Turn its generation off, and the base `theme.json` is used
  as written:

  ```javascript
  wordpressThemeJson({
      baseThemeJsonPath: './theme.json',
      disableTailwindColors: false,
      disableTailwindFontSizes: false,
      disableTailwindFonts: true,        // fonts come from theme.json only
      disableTailwindBorderRadius: false,
  }),
  ```

The default theme turns the fonts off: its Inter font needs a `fontFace`
declaration, which only `theme.json` can hold, so the font lives there.

A family taken out of the generation is no longer linked: the preset in the
editor and the Tailwind utility can then differ, and keeping them in step is up
to you.

### Not generated

- **Spacing.** Tailwind v4 derives every spacing utility from one variable,
  `--spacing`; WordPress's spacing presets are a named scale. The plugin does
  not map one to the other. Changing `--spacing` changes the page; the editor
  keeps WordPress's default spacing presets unless `theme.json` declares
  `settings.spacing.spacingSizes`.
- **Layout.** `contentSize` and `wideSize` live in the base `theme.json`.

### Vite Configuration

```javascript
import { wordpressThemeJson } from '@roots/vite-plugin';

export default defineConfig({
    plugins: [
        tailwindcss(),
        laravel(getThemeConfig()),
        wordpressThemeJson({
            baseThemeJsonPath: './theme.json',
            // Names shown in the editor; the slug is used when none is given
            fontSizeLabels: { sm: 'Small', base: 'Medium', lg: 'Large' },
            borderRadiusLabels: { md: 'Medium', lg: 'Large' },
            fontLabels: { sans: 'Sans Serif' },
        }),
    ],
});
```

When you run `npm run build`, the plugin merges your base `theme.json` with the
`@theme` values of the compiled stylesheet and writes the result to the build
directory. Other options (`outputPath`, `cssFile`, `*.theme.js` partials) are
described in the [plugin's README](https://github.com/roots/vite-plugin).

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
#1f2937)`, which a `theme.json` built from a non-concrete `@theme` value holds —
resolves to its declared fallback, because the login screen carries none of the stylesheets
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

## Block Themes (Full Site Editing)

A Pollora theme can be a WordPress **block theme**: its templates are `templates/*.html`, edited in the Site Editor, instead of Blade views. Start from the Magazine template:

```bash
php artisan pollora:make:theme my-journal --repository=pollora/theme-buzz
```

### How it renders

Nothing to configure. When no Blade view answers a request, Pollora's fallback lets WordPress resolve the block template itself, and answers with the right status: a block theme's `404.html` answers HTTP 404, with `error404` on `<body>`. Blade views still win where they exist, and a `Route::wp()` or Laravel route answers before any template — so don't declare routes for pages the block templates render.

### Where files go

One rule: **the theme root holds what WordPress reads itself; `resources/views/` holds Blade.**

```plaintext
my-journal/
├─ templates/                  # Block templates: index, single, page, archive, search, 404…
├─ parts/                      # Template parts: header, footer
├─ patterns/                   # Static patterns, registered by WordPress
│  └─ masthead.php
├─ resources/
│  ├─ assets/                  # CSS, JS, fonts — built by Vite
│  └─ views/patterns/          # Patterns that need Laravel, registered by Pollora
│     └─ colophon.blade.php
├─ theme.json                  # The design system; the build adds the @theme colours
└─ style.css
```

- `templates/` and `parts/` must be at the theme root: WordPress has no setting to move them.
- This is the layout the Site Editor exports, and the one block themes such as Ollie use: a template exported from the editor drops into place.

### Patterns

A static pattern is a native WordPress pattern, `patterns/*.php`: block markup under a header docblock.

```php
<?php
/**
 * Title: Masthead
 * Slug: my-journal/masthead
 * Categories: my-journal/patterns
 * Block Types: core/template-part/header
 * Inserter: false
 */
?>
<!-- wp:group {"tagName":"header"} -->
<header class="wp-block-group"><!-- wp:site-title /--></header>
<!-- /wp:group -->
```

WordPress reads **only `.php` files** in `patterns/`: an `.html` file there is silently ignored. The `.php` extension also lets a pattern translate a string or print a theme URL when it needs to.

A pattern that needs Laravel — configuration, a helper, a computed value — is a Blade view in `resources/views/patterns`, registered by Pollora (see [Patterns](patterns.md)).

Templates stay thin and point at patterns: `<!-- wp:pattern {"slug":"my-journal/masthead"} /-->`.

WordPress caches the list of a theme's `patterns/` files, so a new file appears once the cache is cleared — `wp eval 'wp_get_theme()->delete_pattern_cache();'` — or at once with `WP_DEVELOPMENT_MODE=theme`.

### Assets and debugging

Assets work as in any Pollora theme (`Asset::add(...)->useVite()`): a Vite entry is enqueued as a script module, printed after WordPress's import map, so the core blocks' own modules — the navigation block's, for one — keep working.

With `WP_DEBUG` on, the template marker of a block theme always reads `template="template-canvas"`: WordPress renders every block template through `wp-includes/template-canvas.php`. Tell templates apart by the `<body>` classes (`single-post`, `search-results`, `error404`…).

## See also

- [Blade templates in WordPress](https://pollora.dev/guides/blade-templates-wordpress/)
