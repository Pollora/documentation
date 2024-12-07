# Administration Pages

The `AdminPage` facade provides an easy way to create and manage WordPress administration pages using Laravel's fluent syntax.

## Basic Usage

The `AdminPage` facade allows you to add administration pages and subpages to WordPress:

```php
use Pollora\Support\Facades\AdminPage;

// Add a main page
AdminPage::page(
    'My Plugin',           // Page title
    'My Plugin',           // Menu title
    'manage_options',      // Required capability
    'my-plugin',          // Slug
    function () {         // Action/Callback
        return view('admin.index');
    },
    'dashicons-admin-generic', // Icon (optional)
    20                        // Position (optional)
);

// Add a subpage
AdminPage::subpage(
    'my-plugin',          // Parent slug
    'Settings',           // Page title
    'Settings',           // Menu title
    'manage_options',     // Required capability
    'my-plugin-settings', // Slug
    function () {         // Action/Callback
        return view('admin.settings');
    }
);
```

## Laravel Integration

Callbacks can return Laravel views or use dependency injection:

```php
AdminPage::page(
    'Analytics',
    'Analytics',
    'manage_options',
    'analytics',
    function (AnalyticsService $analytics) {
        $data = $analytics->getReports();
        return view('admin.analytics', compact('data'));
    }
);
```

## Method Chaining

The `page()` and `subpage()` methods return the factory instance, allowing for method chaining:

```php
AdminPage::page(/* ... */)
    ->subpage(/* ... */)
    ->subpage(/* ... */);
```

## Best Practices

### Code Location

It's recommended to declare your admin pages in a service provider:

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Pollora\Support\Facades\AdminPage;

class AdminServiceProvider extends ServiceProvider
{
    public function boot()
    {
        AdminPage::page(/* ... */);
    }
}
```

### View Organization

Organize your administration views in a dedicated folder, for example `resources/views/admin/`:

```
resources/
└── views/
    └── admin/
        ├── index.blade.php
        ├── settings.blade.php
        └── partials/
            └── header.blade.php
```

## Parameter Reference

### page() Method

- `$pageTitle`: Text to be displayed in the page title tags
- `$menuTitle`: Text to be used for the menu
- `$capability`: WordPress capability required to access the page
- `$slug`: Unique identifier for the page
- `$action`: Callback or action to execute to display content
- `$iconUrl`: URL or dashicon class for menu icon (optional)
- `$position`: Position in the menu (optional)

### subpage() Method

- `$parent`: Parent page slug
- `$pageTitle`: Text to be displayed in the page title tags
- `$menuTitle`: Text to be used for the menu
- `$capability`: WordPress capability required to access the page
- `$slug`: Unique identifier for the subpage
- `$action`: Callback or action to execute to display content