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

### Configuring Asset Containers

You can configure Asset Containers in your service provider or configuration file:

```php
app('asset.container')->addContainer('my-plugin', [
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

### Accessing Assets

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

#### Using a Specific Asset Container

By default, the framework uses the **active theme's asset container**. If you want to explicitly reference assets from a different container, use the `from()` method:

```php
use Pollora\Support\Facades\Asset;

// Reference an image from the "admin" container
$imageUrl = Asset::url('assets/images/logo.png')->from('admin');

// Reference a CSS file from the "shared" container
$cssUrl = Asset::url('assets/css/styles.css')->from('shared');
```

#### Inline Usage in Blade Templates

You can also use the `Asset` facade directly within your Blade templates for dynamic asset referencing:

```blade
<img src="{{ Asset::url('assets/images/logo.png') }}">
<img src="{{ Asset::url('assets/images/admin-logo.png')->from('admin') }}">
<link rel="stylesheet" href="{{ Asset::url('assets/css/app.css') }}">
<script src="{{ Asset::url('assets/js/app.js') }}"></script>
```

#### How It Works

The `Asset` facade leverages an internal `AssetFile` class that dynamically resolves the correct URL based on the specified asset path and container. The default container is set to `theme`, which corresponds to the assets of the active theme. 

If no container is specified, the framework falls back to the default container automatically.

#### Examples

```php
// Using the default "theme" container
$imageUrl = Asset::url('assets/images/theme-logo.png');

// Switching to another container
$sharedCss = Asset::url('assets/css/shared-styles.css')->from('shared');
```

This updated approach provides greater flexibility and maintains clarity in your asset management workflow. It ensures that assets are always correctly referenced, regardless of their container or context.

## ViteJS Integration

To use ViteJS for asset management:

```php
Asset::add('my-vite-asset', 'path/to/asset.js')
    ->container('theme') // Optional: specify a container
    ->useVite()
    ->toFrontend();
```

## Configuration Options

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

## Localizing JavaScript

Use the `localize(string $objectName, array $data)` method to localize JavaScript:

```php
Asset::add('handle', 'path/to/asset.js')
    ->localize('myData', ['key' => 'value']);
```

## Inline Content

Add inline content for CSS or JavaScript using the `inline(string $content)` method:

```php
Asset::add('my-theme', 'css/screen.min.css')
    ->inline('.panel-main { border: 1px solid blue; }');

Asset::add('typekit', 'https://use.typekit.net/fdsjhizo.js')
    ->inline('try{Typekit.load({ async: true });}catch(e){}');
```

## Default Container

A default container is automatically set for the theme. If you don't specify a container, the default will be used:

```php
Asset::add('default/script', Theme::path('assets/app.js'))
    ->toFrontend()
    ->useVite();
```

This will use the default theme container configuration.

## Non-Vite Assets

You can use Asset Containers for non-Vite assets as well:

```php
Asset::add('plugin/legacy-script', 'js/legacy.js')
    ->container('plugin')
    ->toBackend();
```

This allows for consistent asset management across your entire WordPress setup, regardless of whether you're using ViteJS or traditional asset inclusion methods.

## Important: Use of the Service Provider

It is essential to declare assets in a **Service Provider** rather than a **Hookable** class. The service provider ensures the assets are properly enqueued via the WordPress `wp_enqueue_scripts` hook.

By following these guidelines, you can ensure consistent and efficient asset management across your entire WordPress setup, leveraging the power of ViteJS and Pollora's asset management system.
