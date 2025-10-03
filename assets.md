# Assets Management

The Pollora framework provides a powerful asset management system with support for ViteJS integration and Asset Containers.

## Basic Usage
The `Asset` class simplifies adding CSS and JavaScript assets with chainable methods:
```php
$css = Asset::add('my-theme', 'css/screen.min.css')
    ->dependencies(['bootstrap'])
    ->version('1.0')
    ->media('all')
    ->toFrontend();
$js = Asset::add('theme-script', 'js/mythemescript.js')
    ->dependencies(['jquery'])
    ->version('1.0')
    ->loadInFooter()
    ->loadStrategy('defer')
    ->toFrontend()
    ->localize('myData', ['key' => 'value']);

## Asset Containers

Asset Containers allow you to group and manage assets with specific configurations:

### Root Container for Native Laravel Assets

The framework automatically registers a `root` container for native Laravel assets. This container is configured to work with Laravel's default asset structure and Vite configuration:

```php
// The root container is automatically registered and set as default
// It looks for assets in the standard Laravel locations:
// - resources/css/app.css
// - resources/js/app.js
```

To use native Laravel assets with the root container:

```php
Asset::add('root/styles', 'resources/css/app.css')
    ->container('root')
    ->toBackend()
    ->useVite();

Asset::add('root/scripts', 'resources/js/app.js')
    ->container('root')
    ->toBackend()
    ->useVite();
```

The root container configuration:
- **Hot file**: `public/hot`
- **Build directory**: `build`
- **Manifest path**: `manifest.json`
- **Base path**: Empty (assets are at the root level)

### Configuring Custom Asset Containers

You can configure additional Asset Containers in your service provider or configuration file:

```php
app(\Pollora\Asset\AssetContainerManager::class)->addContainer('my-plugin', [
    'hot_file' => public_path("my-plugin.hot"),
    'build_directory' => "build/my-plugin",
    'manifest_path' => public_path("build/my-plugin/manifest.json"),
    'base_path' => '',
    'asset_dir' => [
        'root' => 'assets',
        'images' => 'images',
        'fonts' => 'fonts',
        'css' => 'css',
        'js' => 'js',
    ]
]);
```

## ViteJS Integration

To use ViteJS for asset management:

```php
Asset::add('my-vite-asset', 'path/to/asset.js')
    ->container('theme') // Optional: specify a container
    ->useVite()
    ->toFrontend();
```

### Configuration Options

The following methods are available for configuring assets:

- `add(string $handle, string $path)`: Add a new asset.
- `container(string $containerName)`: Specify the asset container to use.
- `dependencies(array $dependencies)`: Specify asset dependencies.
- `version(string $version)`: Set the asset version.
- `media(string $media)`: Set the media type for styles.
- `loadInFooter()`: Load the script in the footer.
- `loadStrategy(string $strategy)`: Set the script load strategy (e.g., 'defer').
- `toFrontend()`, `toBackend()`, `toLoginScreen()`, `toCustomizer()`, `toEditor()`: Specify where to load the asset.
- `useVite()`: Use ViteJS for this asset.

### Localizing JavaScript

Use the `localize(string $objectName, array $data)` method to localize JavaScript:

```php
Asset::add('handle', 'path/to/asset.js')
    ->localize('myData', ['key' => 'value']);
```

### Inline Content

Add inline content for CSS or JavaScript using the `inline(string $content)` method:

```php
Asset::add('my-theme', 'css/screen.min.css')
    ->inline('.panel-main { border: 1px solid blue; }');

Asset::add('typekit', 'https://use.typekit.net/fdsjhizo.js')
    ->inline('try{Typekit.load({ async: true });}catch(e){}');
```

### Default Container

The framework automatically sets `root` as the default container for native Laravel assets. This means if you don't specify a container, the root container will be used:

```php
// These two are equivalent when root is the default container
Asset::add('app-styles', 'resources/css/app.css')
    ->useVite()
    ->toBackend();

Asset::add('app-styles', 'resources/css/app.css')
    ->container('root')
    ->useVite()
    ->toBackend();
```

For theme-specific assets, you should explicitly specify the theme container:

```php
Asset::add('theme/script', Theme::path('assets/app.js'))
    ->container('theme')
    ->toFrontend()
    ->useVite();
```

### Non-Vite Assets

You can use Asset Containers for non-Vite assets as well:

```php
Asset::add('plugin/legacy-script', 'js/legacy.js')
    ->container('plugin')
    ->toBackend();
```

This allows for consistent asset management across your entire WordPress setup, regardless of whether you're using ViteJS or traditional asset inclusion methods.

## Accessing Assets

You can easily reference assets using the `Asset` facade:

```php
use Pollora\Support\Facades\Asset;

// Reference an image
$imageUrl = Asset::url('assets/images/logo.png');

// Reference a CSS file
$cssUrl = Asset::url('assets/css/app.css');

// Reference a JavaScript file
$jsUrl = Asset::url('assets/js/app.js');

// Reference a font file
$fontUrl = Asset::url('assets/fonts/myfont.woff2');
```

### Using a Specific Asset Container

By default, the framework uses the **root container** for native Laravel assets. If you want to explicitly reference assets from a different container, use the `from()` method:

```php
use Pollora\Support\Facades\Asset;

// Reference an image from the "admin" container
$imageUrl = Asset::url('assets/images/logo.png')->from('admin');

// Reference a CSS file from the "theme" container
$cssUrl = Asset::url('assets/css/styles.css')->from('theme');

// Reference native Laravel assets from the root container
$appCss = Asset::url('resources/css/app.css')->from('root');
```

### Inline Usage in Blade Templates

You can also use the `Asset` facade directly within your Blade templates for dynamic asset referencing:

```blade
<img src="{{ Asset::url('assets/images/logo.png') }}">
<img src="{{ Asset::url('assets/images/admin-logo.png')->from('admin') }}">
<link rel="stylesheet" href="{{ Asset::url('assets/css/app.css') }}">
<script src="{{ Asset::url('assets/js/app.js') }}"></script>
```

### How It Works

The `Asset` facade leverages an internal `AssetFile` class that dynamically resolves the correct URL based on the specified asset path and container. The default container is now set to `root`, which handles native Laravel assets. 

If no container is specified, the framework falls back to the default container automatically. This means native Laravel assets can be referenced without specifying a container.

### Examples

```php
// Using the default root container for Laravel assets
$appStyles = Asset::url('resources/css/app.css');

// Using a specific container for theme assets
$themeImage = Asset::url('assets/images/logo.png')->from('theme');

// Switching to another container
$sharedCss = Asset::url('assets/css/shared-styles.css')->from('shared');
```

This updated approach provides greater flexibility and maintains clarity in your asset management workflow. It ensures that assets are always correctly referenced, regardless of their container or context.

## Important: Use of the Service Provider

It is essential to declare assets in a **Service Provider** rather than a **Hookable** class. The service provider ensures the assets are properly enqueued via the WordPress `wp_enqueue_scripts` hook.

By following these guidelines, you can ensure consistent and efficient asset management across your entire WordPress setup, leveraging the power of ViteJS and Pollora's asset management system.
