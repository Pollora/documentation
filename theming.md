# Creating a Theme

## Introduction

The Pollora framework offers a robust and flexible system for creating and managing WordPress themes. This guide will help you understand how to create, customize, and use themes with Pollora.

## Creating a New Theme

### Generating a New Theme

To generate a new theme, run the following command:

```bash
php artisan theme:make
```

You'll be prompted to answer several questions to configure your theme. Optionally, you can define the theme's slug right away:

```bash
php artisan theme:make {theme-name}
```

This command creates a new theme with the necessary folder structure and base files.

### Theme Structure

A typical Pollora theme has the following structure:

```plaintext
theme-name/
├─ config/
│  ├─ gutenberg.php
│  ├─ images.php
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
├─ tailwind.config.js
├─ theme.json
├─ vite.config.json
└─ vite.config.js
```

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

You can access the current template hierarchy in your views or controllers:

```php
// In a controller
use Pollora\Theme\TemplateHierarchy;

public function show()
{
    $hierarchy = TemplateHierarchy::instance()->hierarchy();
    
    return view('page', ['templateHierarchy' => $hierarchy]);
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

Plugins can extend the template hierarchy for specific content types:

```php
// In a plugin or theme service provider
$templateHierarchy = \Pollora\Theme\TemplateHierarchy::instance();

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

## TailwindCSS Integration

Each Pollora theme is pre-configured with TailwindCSS. The `package.json` file includes the necessary dependencies:

```json
{
    "devDependencies": {
        "@tailwindcss/forms": "^0.5.6",
        "@tailwindcss/typography": "^0.5.9",
        "tailwindcss": "^3.3.3",
        // ... (other dependencies)
    },
    "dependencies": {
        "@tailwindcss/aspect-ratio": "^0.4.2"
    }
}
```

To use TailwindCSS in your theme, make sure to include the necessary directives in your main CSS file.

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

Language files should be placed in the `lang/` folder of your theme. Pollora will load them automatically.

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

### Asset Directory Configuration

In your theme's `config.php` file, you can specify the directory structure for different asset types:

```php
return [
    // ... autres configurations ...
    'asset_dir' => [
        'root' => 'assets',
        'images' => 'images',
        'fonts' => 'fonts',
        'css' => 'css',
        'js' => 'js',
    ],
];
```

This configuration is used by the asset management system to locate assets correctly.

### Using Theme Assets

The framework automatically sets up a default container for the active theme. You can reference assets using Vite macros:

- `Vite::image()`: For referencing image assets
- `Vite::font()`: For referencing font assets
- `Vite::css()`: For referencing CSS assets
- `Vite::js()`: For referencing JavaScript assets

Each of these macros need asecond parameter filled the theme asset container slug (`theme`)

```php
use Illuminate\Support\Facades\Vite;

// Get the URL for an image in the theme
$logoUrl = Vite::image('logo.png', 'theme');

// Get the URL for a CSS file
$styleUrl = Vite::css('app.css', 'theme');

// Get the URL for a JavaScript file
$scriptUrl = Vite::js('app.js', 'theme');

// Get the URL for a font file
$fontUrl = Vite::font('myfont.woff2', 'theme');
```

These macros automatically use the correct asset directory based on your theme's configuration.

#### Usage in Blade Templates

You can use these macros in your Blade templates:

```php
<img src="{{ Vite::image('logo.png', 'theme') }}" alt="Logo">
<link rel="stylesheet" href="{{ Vite::css('app.css', 'theme') }}">
<script src="{{ Vite::js('app.js', 'theme') }}"></script>
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

Then, in your service provider, you can extend the template hierarchy:

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

    public function boot()
    {
        // Get template hierarchy instance
        $templateHierarchy = TemplateHierarchy::instance();
        
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
        add_action('template_redirect', function() {
            $templateHierarchy = TemplateHierarchy::instance();
            
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
public function boot()
{
    // Get template hierarchy instance
    $templateHierarchy = TemplateHierarchy::instance();
    
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
public function boot()
{
    if (config('app.debug')) {
        add_action('template_redirect', function() {
            $templateHierarchy = TemplateHierarchy::instance();
            
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
