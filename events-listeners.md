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

### WordPress Events
- [Core WordPress Events](#core-wordpress-events)
  - [Post Events](#post-events)
  - [Media Events](#media-events)
  - [Taxonomy Events](#taxonomy-events)
  - [User Events](#user-events)
  - [Menu Events](#menu-events)
  - [Widget Events](#widget-events)
  - [Option Events](#option-events)
  - [Comment Events](#comment-events)
  - [Blog Events](#blog-events)
  - [Installer Events](#installer-events)

### Plugin Events
- [WooCommerce Events](#woocommerce-events)
- [Yoast SEO Events](#yoast-seo-events)
- [Two Factor Events](#two-factor-events)
- [User Switching Events](#user-switching-events)

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
   - Consider making properties readonly in PHP 8.1+

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

## Core WordPress Events

### Post Events

#### PostCreated

Fired when a new post is created.

```php
use Pollora\Events\WordPress\Post\PostCreated;

class PostCreatedListener
{
    public function handle(PostCreated $event): void
    {
        $post = $event->post; // WP_Post instance
        // Your logic here
    }
}
```

#### PostUpdated

Fired when a post is updated.

```php
use Pollora\Events\WordPress\Post\PostUpdated;

class PostUpdatedListener
{
    public function handle(PostUpdated $event): void
    {
        $post = $event->post; // WP_Post instance
        $oldStatus = $event->oldStatus; // Previous status
        $newStatus = $event->newStatus; // New status
        // Your logic here
    }
}
```

#### PostPublished

Fired when a post is published.

```php
use Pollora\Events\WordPress\Post\PostPublished;

class PostPublishedListener
{
    public function handle(PostPublished $event): void
    {
        $post = $event->post; // WP_Post instance
        // Your logic here
    }
}
```

#### PostTrashed

Fired when a post is moved to trash.

```php
use Pollora\Events\WordPress\Post\PostTrashed;

class PostTrashedListener
{
    public function handle(PostTrashed $event): void
    {
        $post = $event->post; // WP_Post instance
        // Your logic here
    }
}
```

#### PostRestored

Fired when a post is restored from trash.

```php
use Pollora\Events\WordPress\Post\PostRestored;

class PostRestoredListener
{
    public function handle(PostRestored $event): void
    {
        $post = $event->post; // WP_Post instance
        // Your logic here
    }
}
```

#### PostDeleted

Fired when a post is permanently deleted.

```php
use Pollora\Events\WordPress\Post\PostDeleted;

class PostDeletedListener
{
    public function handle(PostDeleted $event): void
    {
        $post = $event->post; // WP_Post instance
        // Your logic here
    }
}
```

### Media Events

#### MediaCreated

Fired when a new media file is uploaded or attached.

```php
use Pollora\Events\WordPress\Media\MediaCreated;

class MediaCreatedListener
{
    public function handle(MediaCreated $event): void
    {
        $attachment = $event->attachment; // WP_Post instance of the attachment
        // Your logic here
    }
}
```

#### MediaUpdated

Fired when a media file's metadata is updated.

```php
use Pollora\Events\WordPress\Media\MediaUpdated;

class MediaUpdatedListener
{
    public function handle(MediaUpdated $event): void
    {
        $attachment = $event->attachment; // WP_Post instance of the attachment
        // Your logic here
    }
}
```

#### MediaDeleted

Fired when a media file is permanently deleted.

```php
use Pollora\Events\WordPress\Media\MediaDeleted;

class MediaDeletedListener
{
    public function handle(MediaDeleted $event): void
    {
        $attachment = $event->attachment; // WP_Post instance of the attachment
        // Your logic here
    }
}
```

#### MediaEdited

Fired when an image is edited using the WordPress image editor.

```php
use Pollora\Events\WordPress\Media\MediaEdited;

class MediaEditedListener
{
    public function handle(MediaEdited $event): void
    {
        $attachment = $event->attachment; // WP_Post instance of the attachment
        $filename = $event->filename; // Edited image filename
        // Your logic here
    }
}
```

### Taxonomy Events

The following events are available for WordPress taxonomies:

#### TermCreated

Fired when a new taxonomy term is created.

```php
use Pollora\Events\WordPress\Taxonomy\TermCreated;

class TermCreatedListener
{
    public function handle(TermCreated $event): void
    {
        $term = $event->term; // WP_Term instance
        // Your logic here
    }
}
```

#### TermUpdated

Fired when a taxonomy term is updated.

```php
use Pollora\Events\WordPress\Taxonomy\TermUpdated;

class TermUpdatedListener
{
    public function handle(TermUpdated $event): void
    {
        $term = $event->term; // WP_Term instance
        $previousTerm = $event->previousTerm; // Previous WP_Term state
        // Your logic here
    }
}
```

#### TermDeleted

Fired when a taxonomy term is deleted.

```php
use Pollora\Events\WordPress\Taxonomy\TermDeleted;

class TermDeletedListener
{
    public function handle(TermDeleted $event): void
    {
        $term = $event->term; // WP_Term instance
        // Your logic here
    }
}
```

### User Events

The following events are available for WordPress users:

#### UserCreated

Fired when a new user account is created.

```php
use Pollora\Events\WordPress\User\UserCreated;

class UserCreatedListener
{
    public function handle(UserCreated $event): void
    {
        $user = $event->user; // WP_User instance
        $creator = $event->creator; // WP_User instance or null if self-registered
        // Your logic here
    }
}
```

#### UserUpdated

Fired when a user's profile is updated.

```php
use Pollora\Events\WordPress\User\UserUpdated;

class UserUpdatedListener
{
    public function handle(UserUpdated $event): void
    {
        $user = $event->user; // WP_User instance
        // Your logic here
    }
}
```

#### UserDeleted

Fired when a user account is deleted.

```php
use Pollora\Events\WordPress\User\UserDeleted;

class UserDeletedListener
{
    public function handle(UserDeleted $event): void
    {
        $user = $event->user; // WP_User instance
        $roles = $event->roles; // Array of role names the user had
        // Your logic here
    }
}
```

#### UserRoleChanged

Fired when a user's role is changed.

```php
use Pollora\Events\WordPress\User\UserRoleChanged;

class UserRoleChangedListener
{
    public function handle(UserRoleChanged $event): void
    {
        $user = $event->user; // WP_User instance
        $oldRoles = $event->oldRoles; // Previous roles
        $newRole = $event->newRole; // New role or null if removed
        // Your logic here
    }
}
```

#### UserPasswordReset

Fired when a user resets their password.

```php
use Pollora\Events\WordPress\User\UserPasswordReset;

class UserPasswordResetListener
{
    public function handle(UserPasswordReset $event): void
    {
        $user = $event->user; // WP_User instance
        // Your logic here
    }
}
```

#### UserPasswordRequested

Fired when a user requests a password reset.

```php
use Pollora\Events\WordPress\User\UserPasswordRequested;

class UserPasswordRequestedListener
{
    public function handle(UserPasswordRequested $event): void
    {
        $user = $event->user; // WP_User instance
        // Your logic here
    }
}
```

#### UserLoggedIn

Fired when a user logs in.

```php
use Pollora\Events\WordPress\User\UserLoggedIn;

class UserLoggedInListener
{
    public function handle(UserLoggedIn $event): void
    {
        $user = $event->user; // WP_User instance
        // Your logic here
    }
}
```

#### UserLoggedOut

Fired when a user logs out.

```php
use Pollora\Events\WordPress\User\UserLoggedOut;

class UserLoggedOutListener
{
    public function handle(UserLoggedOut $event): void
    {
        $user = $event->user; // WP_User instance
        // Your logic here
    }
}
```

### Menu Events

The following events are available for WordPress navigation menus:

#### MenuCreated

Fired when a new navigation menu is created.

```php
use Pollora\Events\WordPress\Menu\MenuCreated;

class MenuCreatedListener
{
    public function handle(MenuCreated $event): void
    {
        $menu = $event->getMenu(); // WP_Term instance
        // Your logic here
    }
}
```

#### MenuUpdated

Fired when an existing navigation menu is updated.

```php
use Pollora\Events\WordPress\Menu\MenuUpdated;

class MenuUpdatedListener
{
    public function handle(MenuUpdated $event): void
    {
        $menu = $event->getMenu(); // WP_Term instance
        // Your logic here
    }
}
```

#### MenuDeleted

Fired when a navigation menu is deleted.

```php
use Pollora\Events\WordPress\Menu\MenuDeleted;

class MenuDeletedListener
{
    public function handle(MenuDeleted $event): void
    {
        $menu = $event->getMenu(); // WP_Term instance
        // Your logic here
    }
}
```

#### MenuLocationChanged

Fired when a menu is assigned to or removed from a theme location.

```php
use Pollora\Events\WordPress\Menu\MenuLocationChanged;

class MenuLocationChangedListener
{
    public function handle(MenuLocationChanged $event): void
    {
        $menu = $event->getMenu(); // WP_Term instance
        $location = $event->getLocation(); // Theme location identifier
        $isAssigned = $event->isAssigned(); // true if assigned, false if removed
        // Your logic here
    }
}
```

### Widget Events

The following events are available for WordPress widgets:

#### WidgetAdded

Fired when a new widget is added to a sidebar.

```php
use Pollora\Events\WordPress\Widget\WidgetAdded;

class WidgetAddedListener
{
    public function handle(WidgetAdded $event): void
    {
        $widgetId = $event->widgetId;      // Widget ID
        $widgetName = $event->widgetName;  // Widget type name (e.g., "Archives", "Categories")
        $widgetTitle = $event->widgetTitle; // Widget instance title
        $sidebarId = $event->sidebarId;    // Sidebar ID where the widget was added
        // Your logic here
    }
}
```

#### WidgetRemoved

Fired when a widget is removed from a sidebar.

```php
use Pollora\Events\WordPress\Widget\WidgetRemoved;

class WidgetRemovedListener
{
    public function handle(WidgetRemoved $event): void
    {
        $widgetId = $event->widgetId;      // Widget ID
        $widgetName = $event->widgetName;  // Widget type name
        $widgetTitle = $event->widgetTitle; // Widget instance title
        $sidebarId = $event->sidebarId;    // Sidebar ID where the widget was removed from
        // Your logic here
    }
}
```

#### WidgetMoved

Fired when a widget is moved from one sidebar to another.

```php
use Pollora\Events\WordPress\Widget\WidgetMoved;

class WidgetMovedListener
{
    public function handle(WidgetMoved $event): void
    {
        $widgetId = $event->widgetId;        // Widget ID
        $oldSidebarId = $event->oldSidebarId; // Source sidebar ID
        $newSidebarId = $event->newSidebarId; // Destination sidebar ID
        $widgetName = $event->widgetName;    // Widget type name
        $widgetTitle = $event->widgetTitle;   // Widget instance title
        // Your logic here
    }
}
```

#### WidgetUpdated

Fired when a widget's settings are updated.

```php
use Pollora\Events\WordPress\Widget\WidgetUpdated;

class WidgetUpdatedListener
{
    public function handle(WidgetUpdated $event): void
    {
        $widgetId = $event->widgetId;        // Widget ID
        $oldInstance = $event->oldInstance;   // Previous widget settings
        $newInstance = $event->newInstance;   // Updated widget settings
        $widgetName = $event->widgetName;    // Widget type name
        $widgetTitle = $event->widgetTitle;   // Widget instance title
        $sidebarId = $event->sidebarId;      // Sidebar ID containing the widget
        // Your logic here
    }
}
```

#### WidgetReordered

Fired when widgets within a sidebar are reordered.

```php
use Pollora\Events\WordPress\Widget\WidgetReordered;

class WidgetReorderedListener
{
    public function handle(WidgetReordered $event): void
    {
        $sidebarId = $event->sidebarId;  // Sidebar ID where reordering occurred
        $oldOrder = $event->oldOrder;     // Previous order of widget IDs
        $newOrder = $event->newOrder;     // New order of widget IDs
        // Your logic here
    }
}
```

### OptionUpdated

Fired when a WordPress option is updated.

```php
use Pollora\Events\WordPress\Option\OptionUpdated;

class OptionUpdatedListener
{
    public function handle(OptionUpdated $event): void
    {
        $optionName = $event->optionName; // string
        $oldValue = $event->oldValue;     // mixed
        $newValue = $event->newValue;     // mixed
        // Your logic here
    }
}
```

Note: Some options are ignored by default to prevent unnecessary event firing:
- `cron` and `doing_cron`
- Transients (`_transient_` and `_site_transient_`)
- Theme modifications (`theme_mods_`)

Also, events are not fired when the new value is identical to the old value.

### Comment Events

The following events are available for WordPress comments:

#### CommentCreated

Fired when a new comment is created.

```php
use Pollora\Events\WordPress\Comment\CommentCreated;

class CommentCreatedListener
{
    public function handle(CommentCreated $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

#### CommentUpdated

Fired when a comment is edited.

```php
use Pollora\Events\WordPress\Comment\CommentUpdated;

class CommentUpdatedListener
{
    public function handle(CommentUpdated $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

#### CommentStatusChanged

Fired when a comment's status changes (e.g., from pending to approved).

```php
use Pollora\Events\WordPress\Comment\CommentStatusChanged;

class CommentStatusChangedListener
{
    public function handle(CommentStatusChanged $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        $oldStatus = $event->oldStatus;
        $newStatus = $event->newStatus;
        // Your logic here
    }
}
```

#### CommentTrashed

Fired when a comment is moved to trash.

```php
use Pollora\Events\WordPress\Comment\CommentTrashed;

class CommentTrashedListener
{
    public function handle(CommentTrashed $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

#### CommentRestored

Fired when a comment is restored from trash.

```php
use Pollora\Events\WordPress\Comment\CommentRestored;

class CommentRestoredListener
{
    public function handle(CommentRestored $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

#### CommentSpammed

Fired when a comment is marked as spam.

```php
use Pollora\Events\WordPress\Comment\CommentSpammed;

class CommentSpammedListener
{
    public function handle(CommentSpammed $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

#### CommentDeleted

Fired when a comment is permanently deleted.

```php
use Pollora\Events\WordPress\Comment\CommentDeleted;

class CommentDeletedListener
{
    public function handle(CommentDeleted $event): void
    {
        $comment = $event->comment; // WP_Comment instance
        // Your logic here
    }
}
```

### Blog Events

The following events are available for WordPress multisite blog management:

#### BlogCreated

Fired when a new blog is created in the network.

```php
use Pollora\Events\WordPress\Blog\BlogCreated;

class BlogCreatedListener
{
    public function handle(BlogCreated $event): void
    {
        $site = $event->site; // WP_Site instance
        $args = $event->args; // Initialization arguments
        // Your logic here
    }
}
```

#### BlogDeleted

Fired when a blog is permanently deleted from the network.

```php
use Pollora\Events\WordPress\Blog\BlogDeleted;

class BlogDeletedListener
{
    public function handle(BlogDeleted $event): void
    {
        $site = $event->site; // WP_Site instance (last known state)
        // Your logic here
    }
}
```

#### BlogArchived

Fired when a blog is archived.

```php
use Pollora\Events\WordPress\Blog\BlogArchived;

class BlogArchivedListener
{
    public function handle(BlogArchived $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogUnarchived

Fired when a blog is restored from archive.

```php
use Pollora\Events\WordPress\Blog\BlogUnarchived;

class BlogUnarchivedListener
{
    public function handle(BlogUnarchived $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogMarkedAsSpam

Fired when a blog is marked as spam.

```php
use Pollora\Events\WordPress\Blog\BlogMarkedAsSpam;

class BlogMarkedAsSpamListener
{
    public function handle(BlogMarkedAsSpam $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogMarkedAsNotSpam

Fired when a blog is marked as not spam.

```php
use Pollora\Events\WordPress\Blog\BlogMarkedAsNotSpam;

class BlogMarkedAsNotSpamListener
{
    public function handle(BlogMarkedAsNotSpam $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogMarkedAsMature

Fired when a blog is marked as mature.

```php
use Pollora\Events\WordPress\Blog\BlogMarkedAsMature;

class BlogMarkedAsMatureListener
{
    public function handle(BlogMarkedAsMature $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogMarkedAsNotMature

Fired when a blog is marked as not mature.

```php
use Pollora\Events\WordPress\Blog\BlogMarkedAsNotMature;

class BlogMarkedAsNotMatureListener
{
    public function handle(BlogMarkedAsNotMature $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogTrashed

Fired when a blog is moved to trash.

```php
use Pollora\Events\WordPress\Blog\BlogTrashed;

class BlogTrashedListener
{
    public function handle(BlogTrashed $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogRestored

Fired when a blog is restored from trash.

```php
use Pollora\Events\WordPress\Blog\BlogRestored;

class BlogRestoredListener
{
    public function handle(BlogRestored $event): void
    {
        $site = $event->site; // WP_Site instance
        // Your logic here
    }
}
```

#### BlogVisibilityUpdated

Fired when a blog's visibility is updated.

```php
use Pollora\Events\WordPress\Blog\BlogVisibilityUpdated;

class BlogVisibilityUpdatedListener
{
    public function handle(BlogVisibilityUpdated $event): void
    {
        $site = $event->site; // WP_Site instance
        $visibility = $event->visibility; // New visibility value
        // Your logic here
    }
}
```

### Installer Events

Events related to WordPress installation, updates, and management of plugins and themes.

#### PluginInstalled

Fired when a new plugin is installed.

```php
use Pollora\Events\WordPress\Installer\Plugin\PluginInstalled;

class PluginInstalledListener
{
    public function handle(PluginInstalled $event): void
    {
        $name = $event->name;        // Plugin name
        $version = $event->version;   // Plugin version
        $slug = $event->slug;        // Plugin slug
        // Your logic here
    }
}
```

#### PluginActivated

Fired when a plugin is activated.

```php
use Pollora\Events\WordPress\Installer\Plugin\PluginActivated;

class PluginActivatedListener
{
    public function handle(PluginActivated $event): void
    {
        $name = $event->name;              // Plugin name
        $networkWide = $event->networkWide; // Whether activated network-wide
        // Your logic here
    }
}
```

#### PluginDeactivated

Fired when a plugin is deactivated.

```php
use Pollora\Events\WordPress\Installer\Plugin\PluginDeactivated;

class PluginDeactivatedListener
{
    public function handle(PluginDeactivated $event): void
    {
        $name = $event->name;              // Plugin name
        $networkWide = $event->networkWide; // Whether deactivated network-wide
        // Your logic here
    }
}
```

#### PluginUpdated

Fired when a plugin is updated.

```php
use Pollora\Events\WordPress\Installer\Plugin\PluginUpdated;

class PluginUpdatedListener
{
    public function handle(PluginUpdated $event): void
    {
        $name = $event->name;           // Plugin name
        $version = $event->version;      // New version
        $oldVersion = $event->oldVersion; // Previous version
        // Your logic here
    }
}
```

#### ThemeInstalled

Fired when a new theme is installed.

```php
use Pollora\Events\WordPress\Installer\Theme\ThemeInstalled;

class ThemeInstalledListener
{
    public function handle(ThemeInstalled $event): void
    {
        $name = $event->name;        // Theme name
        $version = $event->version;   // Theme version
        $slug = $event->slug;        // Theme slug
        // Your logic here
    }
}
```

#### ThemeActivated

Fired when a theme is activated.

```php
use Pollora\Events\WordPress\Installer\Theme\ThemeActivated;

class ThemeActivatedListener
{
    public function handle(ThemeActivated $event): void
    {
        $name = $event->name; // Theme name
        // Your logic here
    }
}
```

#### ThemeDeleted

Fired when a theme is deleted.

```php
use Pollora\Events\WordPress\Installer\Theme\ThemeDeleted;

class ThemeDeletedListener
{
    public function handle(ThemeDeleted $event): void
    {
        $name = $event->name; // Theme name
        // Your logic here
    }
}
```

#### ThemeUpdated

Fired when a theme is updated.

```php
use Pollora\Events\WordPress\Installer\Theme\ThemeUpdated;

class ThemeUpdatedListener
{
    public function handle(ThemeUpdated $event): void
    {
        $name = $event->name;           // Theme name
        $version = $event->version;      // New version
        $oldVersion = $event->oldVersion; // Previous version
        // Your logic here
    }
}
```

#### WordPressUpdated

Fired when WordPress core is updated.

```php
use Pollora\Events\WordPress\Installer\WordPressUpdated;

class WordPressUpdatedListener
{
    public function handle(WordPressUpdated $event): void
    {
        $newVersion = $event->newVersion;   // New WordPress version
        $oldVersion = $event->oldVersion;   // Previous WordPress version
        $autoUpdated = $event->autoUpdated; // Whether it was an automatic update
        // Your logic here
    }
}
```

### WooCommerce Events

The following events are available for WooCommerce actions:

#### OrderStatusChanged

Fired when a WooCommerce order status is changed.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\OrderStatusChanged;

class OrderStatusChangedListener
{
    public function handle(OrderStatusChanged $event): void
    {
        $order = $event->order; // WC_Order instance
        $oldStatus = $event->oldStatus;
        $newStatus = $event->newStatus;
        // Your logic here
    }
}
```

#### AttributeCreated

Fired when a new product attribute is created.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\AttributeCreated;

class AttributeCreatedListener
{
    public function handle(AttributeCreated $event): void
    {
        $attributeId = $event->attributeId;
        $attribute = $event->attribute; // Array containing attribute data
        // Your logic here
    }
}
```

#### AttributeUpdated

Fired when a product attribute is updated.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\AttributeUpdated;

class AttributeUpdatedListener
{
    public function handle(AttributeUpdated $event): void
    {
        $attributeId = $event->attributeId;
        $attribute = $event->attribute; // Array containing attribute data
        // Your logic here
    }
}
```

#### AttributeDeleted

Fired when a product attribute is deleted.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\AttributeDeleted;

class AttributeDeletedListener
{
    public function handle(AttributeDeleted $event): void
    {
        $attributeId = $event->attributeId;
        $attributeName = $event->attributeName;
        // Your logic here
    }
}
```

#### TaxRateCreated

Fired when a new tax rate is created.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\TaxRateCreated;

class TaxRateCreatedListener
{
    public function handle(TaxRateCreated $event): void
    {
        $taxRateId = $event->taxRateId;
        $taxRate = $event->taxRate; // Array containing tax rate data
        // Your logic here
    }
}
```

#### TaxRateUpdated

Fired when a tax rate is updated.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\TaxRateUpdated;

class TaxRateUpdatedListener
{
    public function handle(TaxRateUpdated $event): void
    {
        $taxRateId = $event->taxRateId;
        $taxRate = $event->taxRate; // Array containing tax rate data
        // Your logic here
    }
}
```

#### TaxRateDeleted

Fired when a tax rate is deleted.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\TaxRateDeleted;

class TaxRateDeletedListener
{
    public function handle(TaxRateDeleted $event): void
    {
        $taxRateId = $event->taxRateId;
        // Your logic here
    }
}
```

#### SettingUpdated

Fired when a WooCommerce setting is updated.

```php
use Pollora\Events\WordPress\Plugins\WooCommerce\SettingUpdated;

class SettingUpdatedListener
{
    public function handle(SettingUpdated $event): void
    {
        $optionName = $event->optionName;
        $oldValue = $event->oldValue;
        $newValue = $event->newValue;
        // Your logic here
    }
}
```

### Gravity Forms Events

Events related to Gravity Forms plugin operations.

#### Forms

##### FormCreated

Fired when a new form is created.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Form\FormCreated;

class FormCreatedListener
{
    public function handle(FormCreated $event): void
    {
        $form = $event->form; // Form data array
        // Your logic here
    }
}
```

##### FormUpdated

Fired when a form is updated.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Form\FormUpdated;

class FormUpdatedListener
{
    public function handle(FormUpdated $event): void
    {
        $form = $event->form; // Form data array
        // Your logic here
    }
}
```

##### FormDeleted

Fired when a form is permanently deleted.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Form\FormDeleted;

class FormDeletedListener
{
    public function handle(FormDeleted $event): void
    {
        $form = $event->form; // Form data array
        // Your logic here
    }
}
```

##### FormTrashed

Fired when a form is moved to trash.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Form\FormTrashed;

class FormTrashedListener
{
    public function handle(FormTrashed $event): void
    {
        $form = $event->form; // Form data array
        // Your logic here
    }
}
```

##### FormRestored

Fired when a form is restored from trash.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Form\FormRestored;

class FormRestoredListener
{
    public function handle(FormRestored $event): void
    {
        $form = $event->form; // Form data array
        // Your logic here
    }
}
```

#### Confirmations

##### ConfirmationCreated

Fired when a new form confirmation is created.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Confirmation\ConfirmationCreated;

class ConfirmationCreatedListener
{
    public function handle(ConfirmationCreated $event): void
    {
        $confirmation = $event->confirmation; // Confirmation data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

##### ConfirmationUpdated

Fired when a form confirmation is updated.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Confirmation\ConfirmationUpdated;

class ConfirmationUpdatedListener
{
    public function handle(ConfirmationUpdated $event): void
    {
        $confirmation = $event->confirmation; // Confirmation data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

##### ConfirmationDeleted

Fired when a form confirmation is deleted.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Confirmation\ConfirmationDeleted;

class ConfirmationDeletedListener
{
    public function handle(ConfirmationDeleted $event): void
    {
        $confirmation = $event->confirmation; // Confirmation data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

#### Notifications

##### NotificationCreated

Fired when a new form notification is created.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Notification\NotificationCreated;

class NotificationCreatedListener
{
    public function handle(NotificationCreated $event): void
    {
        $notification = $event->notification; // Notification data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

##### NotificationUpdated

Fired when a form notification is updated.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Notification\NotificationUpdated;

class NotificationUpdatedListener
{
    public function handle(NotificationUpdated $event): void
    {
        $notification = $event->notification; // Notification data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

##### NotificationDeleted

Fired when a form notification is deleted.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Notification\NotificationDeleted;

class NotificationDeletedListener
{
    public function handle(NotificationDeleted $event): void
    {
        $notification = $event->notification; // Notification data array
        $form = $event->form; // Associated form data array
        // Your logic here
    }
}
```

#### Entries

##### EntryDeleted

Fired when a form entry is deleted.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Entry\EntryDeleted;

class EntryDeletedListener
{
    public function handle(EntryDeleted $event): void
    {
        $entry = $event->entry; // Entry data array
        // Your logic here
    }
}
```

##### EntryNoteAdded

Fired when a note is added to a form entry.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Entry\EntryNoteAdded;

class EntryNoteAddedListener
{
    public function handle(EntryNoteAdded $event): void
    {
        $entry = $event->entry; // Entry data array
        $noteId = $event->noteId; // Note ID
        $userId = $event->userId; // User ID who added the note
        $userName = $event->userName; // Username who added the note
        $note = $event->note; // Note content
        $noteType = $event->noteType; // Note type
        // Your logic here
    }
}
```

##### EntryStatusUpdated

Fired when a form entry's status is updated.

```php
use Pollora\Events\WordPress\Plugins\GravityForms\Entry\EntryStatusUpdated;

class EntryStatusUpdatedListener
{
    public function handle(EntryStatusUpdated $event): void
    {
        $entry = $event->entry; // Entry data array
        $status = $event->status; // New status
        $previousStatus = $event->previousStatus; // Previous status
        // Your logic here
    }
}
```

### Yoast SEO Events

Events related to Yoast SEO plugin operations.

#### Meta Events

##### MetaAdded

Fired when a new Yoast SEO meta field is added to a post.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\MetaAdded;

class MetaAddedListener
{
    public function handle(MetaAdded $event): void
    {
        $objectId = $event->objectId;   // ID of the post
        $metaKey = $event->metaKey;     // Meta key (starts with '_yoast_wpseo_')
        $metaValue = $event->metaValue; // Meta value
        // Your logic here
    }
}
```

##### MetaUpdated

Fired when an existing Yoast SEO meta field is updated.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\MetaUpdated;

class MetaUpdatedListener
{
    public function handle(MetaUpdated $event): void
    {
        $objectId = $event->objectId;   // ID of the post
        $metaKey = $event->metaKey;     // Meta key (starts with '_yoast_wpseo_')
        $metaValue = $event->metaValue; // Meta value
        // Your logic here
    }
}
```

##### MetaDeleted

Fired when a Yoast SEO meta field is removed from a post.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\MetaDeleted;

class MetaDeletedListener
{
    public function handle(MetaDeleted $event): void
    {
        $objectId = $event->objectId;   // ID of the post
        $metaKey = $event->metaKey;     // Meta key (starts with '_yoast_wpseo_')
        $metaValue = $event->metaValue; // Meta value
        // Your logic here
    }
}
```

#### Settings Events

##### SettingsImported

Fired when settings are imported from another SEO plugin into Yoast SEO.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\SettingsImported;

class SettingsImportedListener
{
    public function handle(SettingsImported $event): void
    {
        $source = $event->source;           // Name of the source plugin
        $deleteOldData = $event->deleteOldData; // Whether old data was deleted
        // Your logic here
    }
}
```

##### SettingsExported

Fired when Yoast SEO settings are exported to a file.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\SettingsExported;

class SettingsExportedListener
{
    public function handle(SettingsExported $event): void
    {
        $includeTaxonomyMeta = $event->includeTaxonomyMeta; // Whether taxonomy meta was included
        // Your logic here
    }
}
```

#### File Events

##### FileUpdated

Fired when robots.txt or .htaccess files are created or updated through Yoast SEO.

```php
use Pollora\Events\WordPress\Plugins\YoastSeo\FileUpdated;

class FileUpdatedListener
{
    public function handle(FileUpdated $event): void
    {
        $action = $event->action; // 'create_robots', 'update_robots', or 'update_htaccess'
        // Your logic here
    }
}
```

### Two Factor Events

Events related to the Two Factor plugin operations.

#### TwoFactorAuthenticated

Fired when a user successfully authenticates using Two Factor authentication.

```php
use Pollora\Events\WordPress\Plugins\TwoFactor\TwoFactorAuthenticated;

class TwoFactorAuthenticatedListener
{
    public function handle(TwoFactorAuthenticated $event): void
    {
        $user = $event->user; // WP_User object
        $provider = $event->provider; // String identifying the authentication method used
        // Your logic here
    }
}
```
#### TwoFactorAuthenticationFailed

Fired when a Two Factor authentication attempt fails.

```php
use Pollora\Events\WordPress\Plugins\TwoFactor\TwoFactorAuthenticationFailed;

class TwoFactorAuthenticationFailedListener
{
    public function handle(TwoFactorAuthenticationFailed $event): void
    {
        $user = $event->user; // WP_User object
        $errorCode = $event->errorCode; // String error code
        $errorMessage = $event->errorMessage; // String error message
        // Your logic here
    }
}
```

#### TwoFactorConfigurationChanged

Fired when a user's Two Factor configuration is modified.

```php
use Pollora\Events\WordPress\Plugins\TwoFactor\TwoFactorConfigurationChanged;

class TwoFactorConfigurationChangedListener
{
    public function handle(TwoFactorConfigurationChanged $event): void
    {
        $user = $event->user; // WP_User object
        $action = $event->action; // String: 'enabled', 'disabled', or 'updated'
        $provider = $event->provider; // String: provider name or configuration type
        $oldValue = $event->oldValue; // ?array: previous configuration value
        $newValue = $event->newValue; // ?array: new configuration value
        // Your logic here
    }
}
```

### User Switching Events

Events fired by the User Switching plugin actions.

#### UserSwitchedTo

Fired when a user switches to another user account.

```php
use Pollora\Events\WordPress\Plugins\UserSwitching\UserSwitchedTo;

class UserSwitchedToListener
{
    public function handle(UserSwitchedTo $event): void
    {
        $user = $event->user;       // WP_User being switched to
        $oldUser = $event->oldUser; // WP_User switching from
        // Your logic here
    }
}
```

#### UserSwitchedBack

Fired when a user switches back to their original account.

```php
use Pollora\Events\WordPress\Plugins\UserSwitching\UserSwitchedBack;

class UserSwitchedBackListener
{
    public function handle(UserSwitchedBack $event): void
    {
        $user = $event->user;       // WP_User switching back to
        $oldUser = $event->oldUser; // WP_User switching from (null if switching back after being switched off)
        // Your logic here
    }
}
```

#### UserSwitchedOff

Fired when a user switches off (logs out of the switched account).

```php
use Pollora\Events\WordPress\Plugins\UserSwitching\UserSwitchedOff;

class UserSwitchedOffListener
{
    public function handle(UserSwitchedOff $event): void
    {
        $user = $event->user; // WP_User who switched off
        // Your logic here
    }
}
```

