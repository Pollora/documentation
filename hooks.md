# Hookable Hooks

- [Overview](#overview)
- [Lifecycle hooks](#lifecycle-hooks)
- [Hookable declaration](#hookable-declaration)
  - [Declarate with artisan](#declarate-with-artisan)
  - [Manual declaration](#manual-declaration)
- [Attribute-based hooks](#attribute-based-hooks)
  - [Declarate with artisan](#declarate-with-artisan-1)
  - [Action hooks](#action-hooks)
  - [Filter hooks](#filter-hooks)
- [Facades](#facades)
  - [Action Facade](#action-facade)
  - [Filter Facade](#filter-facade)
- [Singleton access](#singleton-access)

## Overview

The "hookable" class is a concept created by the [Themosis Framework](https://framework.themosis.com). Instead of using plugins to add functionalities to your WordPress application, "hookable" classes can be utilized.

Using a "hookable" class provides access to all APIs specified by WordPress and your application packages. By default, "hookable" classes are initiated right before `mu-plugins`. The class code can also be executed at a specific WordPress action or filter hook with a defined priority.

## Lifecycle hooks

Hookable classes are not automatically enlisted in the application lifecycle. Whenever a new hookable class is created, it needs to be registered in the `config/app.php` file within the `hooks` property or in the `bootstrap/hooks.php` file. Both locations are merged into the application lifecycle during boot, ensuring that all declared hooks are properly loaded.

## Hookable declaration

### Declarate with artisan

You can create a new hookable class using the `make:hook` artisan command:

```bash
# Create a basic hookable class
php artisan make:hook MyHook

# Create a hookable class with a specific WordPress hook
php artisan make:hook MyHook --hook=init

# Create a hookable class with a specific hook and priority
php artisan make:hook MyHook --hook=init --priority=20

# Create a hookable class in a subdirectory
php artisan make:hook Admin/MyHook
```

The command will:
1. Create a new hookable class in the `app/Hooks` directory
2. Automatically register it in the `bootstrap/hooks.php` file

Here is an example of a generated hookable class:

```php
<?php

namespace App\Hooks;

use Pollora\Hook\Hookable;

class MyHook extends Hookable
{
    public $hook = 'init';
    
    public int $priority = 10;

    /**
     * Extend WordPress.
     */
    public function register()
    {
        // Code is executed on the "init" WordPress action only.
    }
}
```

Once generated, you can start implementing your hook logic in the `register()` method.

### Manual declaration

#### Declaring hooks in `bootstrap/hooks.php` (recommended)

In addition to `config/app.php`, hookable classes can also be registered in the `bootstrap/hooks.php` file. The hooks declared in this file are merged with those in `config/app.php`. Here's an example:

```php
// bootstrap/hooks.php
return [
    App\Hooks\MySecondHook::class,
];
```

#### Declaring hooks in `config/app.php`

To register a hookable class, declare it in the `hooks` property of the `config/app.php` file:

```php
// config/app.php
return [
    'hooks' => [
        App\Hooks\MyHook::class,
    ],
];
```

Both files will be merged during the boot process.

## Attribute-based hooks

With the new attribute-based approach, you can define action and filter hooks using PHP attributes. This provides a more concise way to define hooks within your classes.

### Declarate with artisan

You can create a new attribute-based hook class using the `make:action` or `make:filter` Artisan commands:

```bash
# Create a new action hook class
php artisan make:action MyAction

# Create a new filter hook class
php artisan make:filter MyFilter
```

### Action hooks

To define an action hook, use the `Action` attribute from the `Pollora\Attributes\Action` namespace:

```php
<?php

namespace App\Hooks;

use Pollora\Attributes\Attributable;
use Pollora\Attributes\Action;

class MyAction implements Attributable
{
    #[Action('init', priority: 20)]
    public function handleInit(): void
    {
        // Code to execute for the 'init' WordPress action
    }
}
```

Multiple action hooks can be declared for the same method:

```php
#[Action('init', priority: 10)]
#[Action('wp_loaded', priority: 15)]
public function setup(): void
{
    // Runs on multiple hooks
}
```

### Filter hooks

To define a filter hook, use the `Filter` attribute from the `Pollora\Attributes\Filter` namespace:

```php
<?php

namespace App\Hooks;

use Pollora\Attributes\Attributable;
use Pollora\Attributes\Filter;

class MyFilter implements Attributable
{
    #[Filter('the_content', priority: 10)]
    public function handleTheContent(string $content): string
    {
        return str_replace('ugly', 'shiny', $content);
    }
}
```

### Automatic Processing with Laravel

All hookable classes implementing `Attributable` are automatically resolved when instantiated by Laravel. This is done in the service provider:

```php
$this->app->resolving(Attributable::class, function ($object, $app) {
    \Pollora\Hook\AttributeProcessor::process($object);
});
```

The `AttributeProcessor` dynamically finds and applies the correct registrars (`ActionRegistrar`, `FilterRegistrar`, etc.) based on the attributes present in a class.


## Singleton access

For optimized access and lifecycle management, all hookable classes are loaded into the application through a singleton named `wp.hooks`. This singleton centralizes access to all registered hooks from both the `bootstrap/hooks.php` and `config/app.php` files.

You can access the hooks through the singleton as follows:

```php
$hooks = app('wp.hooks');
```

This singleton ensures that all hooks are loaded once and are available globally throughout the application. The `wp.hooks` singleton merges hooks from both `bootstrap/hooks.php` and `config/app.php`, providing a unified interface for managing hooks within your application.

By centralizing hooks in a singleton, your application benefits from a streamlined access pattern and reduced duplication of configuration logic.
