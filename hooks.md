# Hookable Hooks

- [Overview](#overview)
- [Lifecycle hooks](#lifecycle-hooks)
- [Hookable declaration](#hookable-declaration)
- [Basic hookable class](#basic-hookable-class)
- [Hookable action](#hookable-action)
- [Hookable filter](#hookable-filter)
- [Singleton access](#singleton-access)

## Overview

The "hookable" class is a concept created by the [Themosis Framework](https://framework.themosis.com). In lieu of plugins for adding functionalities to your WordPress application, "hookable" classes can be utilized.

Coding within a "hookable" class provides access to all APIs specified by WordPress and your application packages. By default, "hookable" classes are initiated right before `mu-plugins`. The class code can also be executed at a specific WordPress action or filter hook with a defined priority.

## Lifecycle hooks

Hookable classes are not automatically enlisted in the application lifecycle. Whenever a new hookable class is created, it needs to be registered in the `config/app.php` file within the `hooks` property or in the `bootstrap/hooks.php` file. Both locations are merged into the application lifecycle during boot, ensuring that all declared hooks are properly loaded.

## Hookable declaration


### Declaring hooks in `bootstrap/hooks.php` (recommended)

In addition to `config/app.php`, hookable classes can also be registered in the `bootstrap/hooks.php` file. The hooks declared in this file are merged with those in `config/app.php`. Here's an example of how to declare a hookable class in the `bootstrap/hooks.php` file:

```php
// bootstrap/hooks.php
return [
    App\Hooks\MySecondHook::class,
];
```

### Declaring hooks in `config/app.php`

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

## Basic hookable class

Here is an example of a hookable class:

```php
<?php

namespace App\Hooks;

use Pollora\Hook\Hookable;

class MyHook extends Hookable
{
    /**
     * Augment WordPress.
     */
    public function register()
    {
        // Implement your code...
    }
}
```

Once you have created the class, you must register it by adding it either to the `hooks` property in `config/app.php` or in the `bootstrap/hooks.php` file.

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

## Singleton access

For optimized access and lifecycle management, all hookable classes are loaded into the application through a singleton named `wp.hooks`. This singleton centralizes access to all registered hooks from both the `bootstrap/hooks.php` and `config/app.php` files.

You can access the hooks through the singleton as follows:

```php
$hooks = app('wp.hooks');
```

This singleton ensures that all hooks are loaded once and are available globally throughout the application. The `wp.hooks` singleton merges hooks from both `bootstrap/hooks.php` and `config/app.php`, providing a unified interface for managing hooks within your application.

By centralizing hooks in a singleton, your application benefits from a streamlined access pattern and reduced duplication of configuration logic.
