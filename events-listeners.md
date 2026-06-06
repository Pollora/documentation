# Events & Listeners

This document describes all the events available in the Pollora framework and how to listen to them.

## Table of Contents

### Framework Basics
- [Introduction](#introduction)
- [Event & Listener Implementation](#event--listener-implementation)
  - [Defining Events](#defining-events)
  - [Defining Listeners](#defining-listeners)
  - [Event Discovery](#event-discovery)
  - [Event Subscribers](#event-subscribers)
  - [Advanced Queue Configuration](#advanced-queue-configuration)
  - [Dispatching Events](#dispatching-events)

### WordPress Integration
- [WordPress Hooks to Laravel Events](#wordpress-hooks-to-laravel-events)
- [Event Naming Convention](#event-naming-convention)
- [WordPress Data in Events](#wordpress-data-in-events)

### Development Guide
- [Best Practices](#best-practices)
  - [Event Design](#event-design)
  - [Listener Design](#listener-design)
- [Debugging Events](#debugging-events)
  - [Event Listening](#event-listening)
  - [Event Discovery](#event-discovery)
  - [Queue Monitoring](#queue-monitoring)
  - [Testing Events](#testing-events)
- [Performance Considerations](#performance-considerations)
  - [Event Optimization](#event-optimization)
  - [Queue Optimization](#queue-optimization)
  - [Memory Management](#memory-management)

## Introduction

Events provide a simple observer pattern implementation, allowing you to subscribe and listen for various events that occur in your WordPress application. Event classes are typically stored in the `app/Events` directory, while their listeners are stored in `app/Listeners`.

### What are Events?

Events serve as a great way to decouple various aspects of your application, since a single event can have multiple listeners that do not depend on each other. For example, you might want to send a Slack notification and queue a task when a post is published. Instead of coupling these actions to your post publishing code, you can raise a `PostPublished` event which your listeners can handle.

### Laravel Events vs WordPress Hooks

While WordPress provides its own hooks system (actions and filters), Laravel's event system offers several advantages:

1. **Type Safety**: Events are proper PHP classes, providing better IDE support and type hinting
2. **Dependency Injection**: Listeners can use Laravel's service container for dependencies
3. **Queueing**: Listeners can be queued for better performance
4. **Testing**: Laravel provides tools for testing events and listeners
5. **Organization**: Events and listeners are organized in a more structured way

The Pollora framework bridges these two worlds by automatically converting some useful WordPress hooks into Laravel events, giving you the best of both ecosystems.

## Event & Listener Implementation

### Defining Events

Events are typically simple classes that hold the data related to the event. For example:

```php
namespace App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PostPublished
{
    use Dispatchable, SerializesModels;

    public $post;

    public function __construct($post)
    {
        $this->post = $post;
    }
}
```

### Defining Listeners

Listeners are classes that define the handling logic for events:

```php
namespace App\Listeners;

class SendPostPublishedNotification
{
    /**
     * Create the event listener.
     */
    public function __construct()
    {
        // Constructor injection is supported
    }

    /**
     * Handle the event.
     */
    public function handle(PostPublished $event)
    {
        // Access the post using $event->post
        // Handle the event...
    }
}
```

### Event Discovery

By default, Laravel will automatically discover and register event listeners by scanning your application's `Listeners` directory. You can disable this in your `EventServiceProvider`:

```php
/**
 * Determine if events and listeners should be automatically discovered.
 */
public function shouldDiscoverEvents(): bool
{
    return false;
}
```

### Event Subscribers

Event subscribers are classes that may subscribe to multiple events from within the class itself. Subscribers should define a `subscribe` method that will be passed an event dispatcher instance:

```php
namespace App\Listeners;

use Illuminate\Events\Dispatcher;

class WordPressEventSubscriber
{
    public function handlePostCreated($event) {}
    public function handlePostUpdated($event) {}
    public function handleUserCreated($event) {}

    public function subscribe(Dispatcher $events): array
    {
        return [
            'Pollora\Events\WordPress\Post\PostCreated' => 'handlePostCreated',
            'Pollora\Events\WordPress\Post\PostUpdated' => 'handlePostUpdated',
            'Pollora\Events\WordPress\User\UserCreated' => 'handleUserCreated',
        ];
    }
}
```

### Advanced Queue Configuration

When using queued listeners, you can customize the queue connection and queue name:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class SendPostPublishedNotification implements ShouldQueue
{
    public $connection = 'redis';
    public $queue = 'listeners';
    public $delay = 60; // seconds
    public $tries = 3;
    public $backoff = [2, 5, 10]; // retry delays in seconds
    
    public function handle($event)
    {
        // Handle the event...
    }

    public function failed($event, $exception)
    {
        // Handle failed job...
    }
}
```

### Dispatching Events

You can dispatch events using several methods:

```php
// Using the Event facade
use Illuminate\Support\Facades\Event;
Event::dispatch(new PostPublished($post));

// Using the event helper
event(new PostPublished($post));

// Using the Dispatchable trait
PostPublished::dispatch($post);
```

## WordPress Integration

### WordPress Hooks to Laravel Events

The Pollora framework automatically converts some WordPress hooks into Laravel events, allowing you to use Laravel's powerful event system with WordPress. Here's how it works:

1. WordPress actions and filters are intercepted by the framework
2. They are converted into Laravel events
3. Your Laravel event listeners are triggered
4. The event data is passed back to WordPress if needed

### Event Naming Convention

WordPress events in Pollora follow a consistent naming convention:

- Post events: `Pollora\Events\WordPress\Post\Post{Action}`
- User events: `Pollora\Events\WordPress\User\User{Action}`
- Media events: `Pollora\Events\WordPress\Media\Media{Action}`
- And so on...

### WordPress Data in Events

All WordPress events include the relevant WordPress objects (WP_Post, WP_User, etc.) as properties of the event class. This allows you to access WordPress data in a Laravel-friendly way:

```php
use Pollora\Events\WordPress\Post\PostCreated;

class PostCreatedListener
{
    public function handle(PostCreated $event)
    {
        $post = $event->post; // WP_Post instance
        $title = $post->post_title;
        $content = $post->post_content;
        // ...
    }
}
```

## Best Practices

### Event Design

1. **Keep Events Simple**
   - Events should primarily hold data
   - Avoid including business logic in event classes
   - Use meaningful names that describe what happened

2. **Event Properties**
   - Make properties public for easy access
   - Use type hints for better IDE support
   - Use `readonly` properties for immutable event data

```php
class PostPublished
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly WP_Post $post,
        public readonly string $oldStatus,
        public readonly string $newStatus
    ) {}
}
```

### Listener Design

1. **Single Responsibility**
   - Each listener should do one thing well
   - Split complex listeners into multiple simpler ones
   - Use dependency injection for services

2. **Error Handling**
   - Always catch and handle exceptions
   - Log errors appropriately
   - Consider what should happen if the listener fails

```php
class SendPostNotification
{
    public function __construct(
        private NotificationService $notifications,
        private LoggerInterface $logger
    ) {}

    public function handle(PostPublished $event)
    {
        try {
            $this->notifications->send($event->post);
        } catch (Exception $e) {
            $this->logger->error('Failed to send notification', [
                'post_id' => $event->post->ID,
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

3. **Queue Configuration**
   - Use queues for time-consuming operations
   - Set appropriate timeouts and retry settings
   - Handle failed jobs gracefully

```php
class ProcessPostImages implements ShouldQueue
{
    public $tries = 3;
    public $backoff = [30, 60, 120];
    public $timeout = 120;

    public function handle(PostPublished $event)
    {
        // Process images...
    }

    public function failed(PostPublished $event, Throwable $e)
    {
        Log::error('Image processing failed', [
            'post_id' => $event->post->ID,
            'error' => $e->getMessage()
        ]);
    }
}
```

## Debugging Events

### Event Listening

You can listen to all events fired in your application using the `Event::listen()` method with a wildcard:

```php
Event::listen('*', function ($event, $data) {
    Log::info('Event fired:', [
        'event' => $event,
        'data' => $data
    ]);
});
```

### Event Discovery

To see which events and listeners are registered in your application, use the artisan command:

```bash
php artisan event:list
```

### Queue Monitoring

For queued listeners, you can monitor the queue using Horizon or the built-in queue commands:

```bash
# Monitor failed jobs
php artisan queue:failed

# Retry failed jobs
php artisan queue:retry all

# Clear failed jobs
php artisan queue:clear
```

### Testing Events

Laravel provides several methods to test events:

```php
use Illuminate\Support\Facades\Event;

class PostTest extends TestCase
{
    public function test_post_created_event_is_dispatched()
    {
        Event::fake();

        // Perform the action that should fire the event
        $post = Post::create(['title' => 'Test Post']);

        // Assert the event was dispatched
        Event::assertDispatched(PostCreated::class, function ($event) use ($post) {
            return $event->post->ID === $post->ID;
        });
    }

    public function test_listener_is_called()
    {
        Event::fake([PostCreated::class]);

        // Create a post
        $post = Post::create(['title' => 'Test Post']);

        // Assert the listener was called
        Event::assertListening(
            PostCreated::class,
            SendPostNotification::class
        );
    }
}
```

## Performance Considerations

### Event Optimization

1. **Selective Event Registration**
   - Only register listeners for events you need
   - Use event discovery with caution in production
   - Consider disabling unused WordPress hooks

```php
// In your service provider
public function register()
{
    // Disable specific WordPress hooks
    add_filter('some_unused_filter', '__return_false');
}
```

2. **Efficient Event Handling**
   - Keep event handlers lightweight
   - Move heavy processing to queued jobs
   - Use caching when appropriate

```php
class ProcessPostImages implements ShouldQueue
{
    public function handle(PostPublished $event)
    {
        // Check cache first
        if (Cache::has("processed_images.{$event->post->ID}")) {
            return;
        }

        // Process images
        $this->processImages($event->post);

        // Cache the result
        Cache::put("processed_images.{$event->post->ID}", true, now()->addDay());
    }
}
```

### Queue Optimization

1. **Queue Configuration**
   - Choose the right queue driver for your needs
   - Set appropriate worker configurations
   - Monitor queue performance

```php
// config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),
    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
        ],
    ],
];
```

2. **Horizon Configuration**
   If you're using Laravel Horizon for queue management:

```php
// config/horizon.php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['default'],
            'balance' => 'simple',
            'processes' => 10,
            'tries' => 3,
        ],
    ],
],
```

### Memory Management

1. **Garbage Collection**
   - Clear event listeners when no longer needed
   - Use `forget()` to remove specific listeners
   - Consider cleanup in long-running processes

```php
// Remove specific listener
Event::forget('Pollora\Events\WordPress\Post\PostCreated');

// Remove all listeners
Event::flush();
```

2. **Memory Leaks Prevention**
   - Avoid storing large objects in event properties
   - Use references when possible
   - Clean up temporary files and resources

```php
class ImageProcessor
{
    private $tempFiles = [];

    public function __destruct()
    {
        // Clean up temporary files
        foreach ($this->tempFiles as $file) {
            if (file_exists($file)) {
                unlink($file);
            }
        }
    }
}
```

---

For a complete catalog of available WordPress events, see [WordPress Events Reference](wordpress-events-reference.md).
