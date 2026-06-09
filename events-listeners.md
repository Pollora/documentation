# Events & Listeners

## Introduction

Pollora bridges WordPress hooks and Laravel's event system. You can use standard Laravel events while also listening to WordPress actions and filters as typed Laravel events.

For standard Laravel event/listener patterns (defining events, defining listeners, event discovery, subscribers, queuing, dispatching), see the [Laravel Events documentation](https://laravel.com/docs/events).

## WordPress Integration

### WordPress Hooks as Laravel Events

Pollora automatically converts WordPress hooks into Laravel events:

1. WordPress actions and filters are intercepted by the framework
2. They are converted into typed Laravel event classes
3. Your Laravel event listeners handle them with full dependency injection and queue support

### Event Naming Convention

WordPress events follow a consistent namespace pattern:

- `Pollora\Events\WordPress\Post\Post{Action}` (e.g., `PostCreated`, `PostPublished`)
- `Pollora\Events\WordPress\User\User{Action}` (e.g., `UserCreated`, `UserLoggedIn`)
- `Pollora\Events\WordPress\Media\Media{Action}`
- `Pollora\Events\WordPress\Comment\Comment{Action}`
- `Pollora\Events\WordPress\Menu\Menu{Action}`
- `Pollora\Events\WordPress\Widget\Widget{Action}`
- `Pollora\Events\WordPress\Option\OptionUpdated`
- `Pollora\Events\WordPress\Blog\Blog{Action}` (multisite)
- `Pollora\Events\WordPress\Installer\Plugin\Plugin{Action}`
- `Pollora\Events\WordPress\Installer\Theme\Theme{Action}`
- Plugin events: `Pollora\Events\WordPress\Plugins\{PluginName}\{Event}`

### Listening to WordPress Events

All WordPress events include the relevant WordPress objects as typed properties:

```php
use Pollora\Events\WordPress\Post\PostPublished;

class SendPostNotification
{
    public function handle(PostPublished $event): void
    {
        $post = $event->post; // WP_Post instance
        $title = $post->post_title;
        // ...
    }
}
```

### Pollora-Specific Tips

- **Use queued listeners** for heavy processing (image optimization, notifications) to avoid slowing down WordPress admin operations.
- **WordPress events fire frequently** -- only register listeners you actually need.
- Use `php artisan event:list` to see all registered events and listeners.

---

For a complete catalog of all available WordPress events and their properties, see [WordPress Events Reference](wordpress-events-reference.md).
