# Hookable Hooks

- [Attribute-based hooks](#attribute-based-hooks)
  - [Declarate with artisan](#declarate-with-artisan-1)
  - [Action hooks](#action-hooks)
  - [Filter hooks](#filter-hooks)
- [Facades](#action-and-filter-facades)
  - [Registering an action or filter](#registering-an-action-or-filter)
  - [Using functions, closures, or instances](#using-functions-closures-or-instances)
  - [Constructor injection](#constructor-injection)
  - [Executing actions and filters](executing-actions-and-filters)
  - [Checking existence](#checking-existence)
  - [Removing actions and filters](#removing-actions-and-filters)
  - [Retrieving callbacks](#retrieving-callbacks)
- [Singleton access](#singleton-access)

## Attribute-based hooks

You can define action and filter hooks using PHP attributes. This provides a more concise way to define hooks within your classes.

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

use Pollora\Attributes\Action;
use Pollora\Hook\Contracts\Hooks;

class MyAction implements Hooks
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

use Pollora\Attributes\Filter;
use Pollora\Hook\Contracts\Hooks;

class MyFilter implements Hooks
{
    #[Filter('the_content', priority: 10)]
    public function handleTheContent(string $content): string
    {
        return str_replace('ugly', 'shiny', $content);
    }
}
```

All classes inside the `app/Hooks` folder and implementing `Hooks` are automatically resolved when instantiated by Laravel.

## Action and Filter facades

Pollora provides two powerful facades, `Action` and `Filter`, to interact with WordPress hooks in a structured and maintainable way. These facades simplify the registration, execution, and management of actions and filters, while ensuring dependency injection and class-based organization.

### Registering an action or filter

To register an action or filter, use the respective `add()` method:

```php
use Pollora\Support\Facades\Action;
use Pollora\Support\Facades\Filter;

// Register an action
Action::add('init', [App\Hooks\InitHandler::class, 'boot']);

// Register a filter explicitly specifying the method
Filter::add('the_content', [App\Hooks\ContentHandler::class, 'modify']);

// Register a filter without specifying the method
Filter::add('the_content', App\Hooks\ContentHandler::class);
```

When passing a class reference alone, Pollora will automatically call a method following Laravel's naming conventions: it will look for `theContent()` inside `ContentHandler`. If a method is explicitly provided, that method will be called instead. If the method is not found, an exception is thrown, specifying the missing method.

### Using functions, closures, or instances

Actions and filters can also be registered using functions, closures, or object instances:

```php
// Function
function custom_callback() {
    // Code...
}
Action::add('init', 'custom_callback');

// Closure
Filter::add('the_content', function($content) {
    return strtoupper($content);
});

// Object instance
$handler = new App\Hooks\ContentHandler();
Filter::add('the_content', [$handler, 'modify']);
```

### Constructor injection

Pollora automatically resolves dependencies when registering actions and filters using a class reference. Dependencies are injected via the constructor:

```php
namespace App\Hooks;

use Illuminate\Http\Request;
use App\Services\ContentProcessor;

class ContentHandler {
    protected $request;
    protected $processor;

    public function __construct(Request $request, ContentProcessor $processor) {
        $this->request = $request;
        $this->processor = $processor;
    }

    public function theContent($content) {
        return $this->processor->process($content);
    }

    public function modify($content) {
        return strtoupper($content);
    }
}
```

This class can be registered in two ways:

```php
// Calls the `theContent` method automatically
Filter::add('the_content', App\Hooks\ContentHandler::class);

// Calls the `modify` method explicitly
Filter::add('the_content', [App\Hooks\ContentHandler::class, 'modify']);
```

### Executing actions and filters

To execute an action, use `do()`, which functions like `do_action()` in WordPress:

```php
Action::do('custom_event', $arg1, $arg2);
```

To apply a filter, use `apply()`, similar to `apply_filters()`:

```php
$modified_value = Filter::apply('custom_filter', $original_value, $arg1);
```

### Checking existence

To verify if an action or filter is registered:

```php
if (Action::exists('custom_event')) {
    // Code...
}

if (Filter::exists('custom_filter')) {
    // Code...
}
```

### Removing actions and filters

Actions and filters can be removed using `remove()`:

```php
Action::remove('init', [App\Hooks\InitHandler::class, 'boot']);
Filter::remove('the_content', [App\Hooks\ContentHandler::class, 'modify']);
```

> **Note:** If an action or filter was registered with a specific priority, the same priority must be specified when removing it.

### Retrieving callbacks

To get the registered callback for an action or filter:

```php
$callback = Action::getCallbacks('init');
$filterCallback = Filter::getCallbacks('the_content');
```

## Singleton access

For optimized access and lifecycle management, all hookable classes are loaded into the application through a singleton named `wp.hooks`. This singleton centralizes access to all registered hooks.

You can access the hooks through the singleton as follows:

```php
$hooks = app('wp.hooks');
```
