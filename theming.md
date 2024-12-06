# Creating a Theme

## Introduction

The Pollora framework offers a robust and flexible system for creating and managing WordPress themes. This guide will help you understand how to create, customize, and use themes with Pollora.

## Creating a New Theme

## Generating a New Theme

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

## Conclusion

The Pollora framework offers a powerful and flexible theme system, integrating modern tools like Vite and TailwindCSS. By following these guidelines, you can create robust and maintainable WordPress themes.

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
