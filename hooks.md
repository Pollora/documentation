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
- [Singleton access](#singleton-access)

## Overview

The "hookable" class is a concept created by the [Themosis Framework](https://framework.themosis.com). In lieu of plugins for adding functionalities to your WordPress application, "hookable" classes can be utilized.

Coding within a "hookable" class provides access to all APIs specified by WordPress and your application packages. By default, "hookable" classes are initiated right before `mu-plugins`. The class code can also be executed at a specific WordPress action or filter hook with a defined priority.

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

### Manual declaration

#### Declaring hooks in `bootstrap/hooks.php` (recommended)

In addition to `config/app.php`, hookable classes can also be registered in the `bootstrap/hooks.php` file. The hooks declared in this file are merged with those in `config/app.php`. Here's an example of how to declare a hookable class in the `bootstrap/hooks.php` file:

```php
// bootstrap/hooks.php
return [
    App\Hooks\MySecondHook::class,
];
```

#### Declaring hooks in `config/app.php`

To register a hookable class, you need to declare it in the `hooks` property of the `config/app.php` file. Here is an example of how to declare a hookable class in this file:

```php
// bootstrap/hooks.php
return [
    // other configurations

    'hooks' => [
        App\Hooks\MyHook::class,
    ],
];
```

Both files will be merged during the boot process, allowing you to register hooks in either or both locations.


## Hookable action

A particular WordPress action hook can also be designated to delay the execution of a hookable class. For instance, the code within the `register` method may need to be executed only when WordPress triggers the `init` action.

To accomplish this, you just need to define a `$hook` instance property in your class and assign it the value of one of the WordPress action hooks:

```php
<?php

namespace App\Hooks;

use Pollora\Hook\Hookable;

class MyFirstHook extends Hookable
{
    public $hook = 'init';

    /**
     * Augment WordPress.
     */
    public function register()
    {
        // Code is executed only on the "init" WordPress action.
    }
}
```

Additionally, an execution priority can be established by setting the `$priority` property:

```php
<?php

namespace App\Hooks;

use Pollora\Hook\Hookable;

class MyFirstHook extends Hookable
{
    public $hook = 'init';

    public $priority = 20;

    /**
     * Augment WordPress.
     */
    public function register()
    {
        // Code is executed only on the "init" WordPress action.
    }
}
```

### Registering hookable actions

Hookable classes must be registered in `config/app.php` or `bootstrap/hooks.php`. Once registered, they will be loaded into the application lifecycle and triggered at the appropriate time.

## Hookable filter

Filters work similarly to actions, but the `register()` method receives parameters and must return a value. This is common for modifying data like content output. For example, here's how you might define a hookable class for the `the_content` filter:

```php
<?php

namespace App\Hooks;

use Pollora\Hook\Hookable;

class ContentOverride extends Hookable
{
    public $hook = 'the_content';

    public function register(string $content): string
    {
        return str_replace('ugly', 'shiny', $content);
    }
}
```

After defining the filter, remember to register it in `config/app.php` or `bootstrap/hooks.php`.

## Attribute-based hooks

With the new attribute-based approach, you can define action and filter hooks using PHP attributes. This approach provides a more concise and expressive way to define hooks within your classes.

### Declarate with artisan

You can create a new attribute-based hook class using the `make:action` or `make:filter` Artisan commands:

```bash
# Create a new action hook class
php artisan make:action MyAction

# Create a new action hook class with a specific WordPress hook
php artisan make:action MyAction --hook=init

# Create a new action hook class with a specific hook and priority
php artisan make:action MyAction --hook=init --priority=20

# Create a new filter hook class
php artisan make:filter MyFilter

# Create a new filter hook class with a specific WordPress hook
php artisan make:filter MyFilter --hook=the_content

# Create a new filter hook class with a specific hook and priority
php artisan make:filter MyFilter --hook=the_content --priority=20
```

These commands will:
1. Create a new hook class in the `app/Hooks` directory
2. Automatically register the class in the `bootstrap/hooks.php` file

If you run the same command on an existing hook class file, it will append the new hook to the file.

Make sure that the hook class file is included in the `bootstrap/hooks.php` file. With the Artisan commands, this step is done automatically.

### Action hooks

To define an action hook, you can use the `Action` attribute from the `Pollora\Attributes\Action` namespace. Here's an example:

```php
<?php

namespace App\Hooks;

use Pollora\Attributes\Action;

class MyAction
{
    #[Action('init', priority: 20)]
    public function handleInit(): void
    {
        // Code to execute for the 'init' WordPress action
    }
}
```

In this example, the `handleInit` method will be executed when the `init` WordPress action is triggered, with a priority of `20`.

### Filter hooks

To define a filter hook, you can use the `Filter` attribute from the `Pollora\Attributes\Filter` namespace. Here's an example:

```php
<?php

namespace App\Hooks;

use Pollora\Attributes\Filter;

class MyFilter
{
    #[Filter('the_content', priority: 10)]
    public function handleTheContent(string $content): string
    {
        // Modify the content and return the updated value
        return str_replace('ugly', 'shiny', $content);
    }
}
```

In this example, the `handleTheContent` method will be executed when the `the_content` WordPress filter is triggered, with a priority of `10`. The method receives the content as a parameter and should return the modified content.


## Singleton access

For optimized access and lifecycle management, all hookable classes are loaded into the application through a singleton named `wp.hooks`. This singleton centralizes access to all registered hooks from both the `bootstrap/hooks.php` and `config/app.php` files.

You can access the hooks through the singleton as follows:

```php
$hooks = app('wp.hooks');
```

This singleton ensures that all hooks are loaded once and are available globally throughout the application. The `wp.hooks` singleton merges hooks from both `bootstrap/hooks.php` and `config/app.php`, providing a unified interface for managing hooks within your application.

By centralizing hooks in a singleton, your application benefits from a streamlined access pattern and reduced duplication of configuration logic.
