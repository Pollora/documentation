# Post Types

Pollora offers three different ways to create custom post types:

1. **Using PHP Attributes (Recommended)** - A modern, declarative approach using PHP 8 attributes
2. **Using the Configuration File** - Define post types in the `config/post-types.php` file
3. **Using the PostType Class** - A fluent, object-oriented approach

## 1. Using PHP Attributes (Recommended)

The most elegant way to define custom post types in Pollora is by using PHP 8 attributes. This approach provides a clean, declarative syntax and better organization of your post types.

### Creating a Post Type with Attributes

You can generate a new post type class using the Artisan command:

```bash
php artisan pollora:make-posttype Event
```

This will create a new class in `app/Cms/PostTypes/Event.php` with the following structure:

```php
<?php

namespace App\Cms\PostTypes;

use Pollora\PostType\AbstractPostType;
use Pollora\Attributes\PostType\PubliclyQueryable;
use Pollora\Attributes\PostType\HasArchive;
use Pollora\Attributes\PostType\Supports;
use Pollora\Attributes\PostType\MenuIcon;

/**
 * Event post type.
 *
 * This class defines the Event custom post type using PHP attributes
 * for WordPress registration.
 */
#[PubliclyQueryable]
#[HasArchive]
#[Supports(['title', 'editor', 'thumbnail'])]
#[MenuIcon('dashicons-admin-post')]
class Event extends AbstractPostType
{
}
```

> **Note:** You don't need to define the `$slug` property. If not explicitly set, the slug will be automatically generated from the class name using Laravel's `Str::kebab()` method. For example, the `Event` class will have a slug of `event`.

> **Note:** You also don't need to define the `getName()` and `getPluralName()` methods. If not explicitly overridden, these methods will automatically generate human-readable names from the class name. For example, the `Event` class will have a singular name of "Event" and a plural name of "Events". The `EventCategory` class will have a singular name of "Event Category" and a plural name of "Event Categories".

> **Note:** You don't need to define the `getLabels()` method. If not explicitly overridden, a default set of labels will be generated based on the singular and plural names. You can still override this method if you need custom labels.

### Available Attributes

Pollora provides several attributes to configure your post types:

#### `PubliclyQueryable`

Makes the post type publicly queryable on the front end.

```php
#[PubliclyQueryable]
```

By default, it's set to `true`. You can set it to `false` to make the post type not publicly queryable.

#### `PublicPostType`

Sets the post type as public, making it visible in the admin UI and on the front end.

```php
#[PublicPostType]
// or explicitly
#[PublicPostType(true)]
// or to make private
#[PublicPostType(false)]
```

#### `HasArchive`

Enables an archive page for the post type.

```php
#[HasArchive]
// or with a custom archive slug
#[HasArchive('custom-archive-slug')]
```

#### `Supports`

Defines which features the post type supports.

```php
#[Supports(['title', 'editor', 'thumbnail'])]
```

Common support options include:
- `title` - Post title
- `editor` - Content editor
- `author` - Author selection
- `thumbnail` - Featured image
- `excerpt` - Post excerpt
- `comments` - Comments
- `revisions` - Revisions
- `custom-fields` - Custom fields
- `page-attributes` - Page attributes (template, parent, menu order)

#### `MenuIcon`

Sets the dashicon to use for the post type in the admin menu.

```php
#[MenuIcon('dashicons-calendar')]
```

You can use any [WordPress Dashicon](https://developer.wordpress.org/resource/dashicons/) or a URL to a custom icon.

#### `ExcludeFromSearch`

Excludes the post type from search results.

```php
#[ExcludeFromSearch]
// or explicitly
#[ExcludeFromSearch(true)]
// or to include in search
#[ExcludeFromSearch(false)]
```

#### `Hierarchical`

Makes the post type hierarchical like pages, allowing for parent/child relationships.

```php
#[Hierarchical]
// or explicitly
#[Hierarchical(true)]
// or to make non-hierarchical
#[Hierarchical(false)]
```

#### `ShowInAdminBar`

Shows the post type in the WordPress admin bar.

```php
#[ShowInAdminBar]
// or explicitly
#[ShowInAdminBar(true)]
// or to hide from admin bar
#[ShowInAdminBar(false)]
```

#### `MenuPosition`

Sets the position in the admin menu where the post type should appear.

```php
#[MenuPosition(5)]
```

Common positions:
- 5 - below Posts
- 10 - below Media
- 15 - below Links
- 20 - below Pages
- 25 - below Comments
- 60 - below first separator
- 65 - below Plugins
- 70 - below Users
- 75 - below Tools
- 80 - below Settings
- 100 - below second separator

#### `CapabilityType`

Sets the capability type for the post type, which is used as a base to build the capabilities that users need to edit, delete, and read posts of this type.

```php
#[CapabilityType('post')]
// or
#[CapabilityType('page')]
// or custom
#[CapabilityType('product')]
```

#### `MapMetaCap`

Enables WordPress to map meta capabilities to primitive capabilities. This is usually used together with `CapabilityType`.

```php
#[MapMetaCap]
// or explicitly
#[MapMetaCap(true)]
// or to disable
#[MapMetaCap(false)]
```

#### `CanExport`

Allows the post type to be exported using WordPress's built-in export tools.

```php
#[CanExport]
// or explicitly
#[CanExport(true)]
// or to disable export
#[CanExport(false)]
```

#### `DeleteWithUser`

When enabled, posts of this type belonging to a user will be moved to trash when the user is deleted.

```php
#[DeleteWithUser]
// or explicitly
#[DeleteWithUser(true)]
// or to keep posts when user is deleted
#[DeleteWithUser(false)]
```

#### `ShowInRest`

Makes the post type available via the REST API. This is needed for the Gutenberg editor to work with the post type.

```php
#[ShowInRest]
// or explicitly
#[ShowInRest(true)]
// or to hide from REST API
#[ShowInRest(false)]
```

#### `RestBase`

Defines the base URL segment that will be used in REST API endpoints for this post type. If not specified, the post type slug will be used.

```php
#[RestBase('events')]
```

#### `RestNamespace`

Defines the REST API namespace for the post type. If not specified, the default WordPress namespace (`wp/v2`) will be used.

```php
#[RestNamespace('my-app/v1')]
```

#### `RestControllerClass`

Specifies the controller class that should be used for handling REST API requests for this post type.

```php
#[RestControllerClass('WP_REST_Posts_Controller')]
// or a custom controller
#[RestControllerClass('MyApp\\REST\\EventController')]
```

#### `BlockEditor`

Enables or disables the Gutenberg block editor for this post type. When disabled, the classic editor will be used instead.

```php
#[BlockEditor]
// or explicitly
#[BlockEditor(true)]
// or to use classic editor
#[BlockEditor(false)]
```

#### `DashboardActivity`

Shows recent activity for this post type in the WordPress dashboard "Activity" widget.

```php
#[DashboardActivity]
// or explicitly
#[DashboardActivity(true)]
// or to hide from dashboard activity
#[DashboardActivity(false)]
```

#### `QuickEdit`

Enables or disables the quick edit functionality for this post type in the WordPress admin list table.

```php
#[QuickEdit]
// or explicitly
#[QuickEdit(true)]
// or to disable quick edit
#[QuickEdit(false)]
```

#### `ShowInFeed`

Includes posts of this type in the site's main RSS feed.

```php
#[ShowInFeed]
// or explicitly
#[ShowInFeed(true)]
// or to exclude from feed
#[ShowInFeed(false)]
```

#### `TitlePlaceholder`

Customizes the placeholder text in the title field when creating a new post.

```php
#[TitlePlaceholder('Enter event title here')]
```

#### `ShowUI`

Controls whether to generate a default UI for managing this post type in the admin.

```php
#[ShowUI]
// or explicitly
#[ShowUI(true)]
// or to hide UI
#[ShowUI(false)]
```

#### `ShowInMenu`

Controls where to show the post type in the admin menu. True places it as a top-level menu, false hides it, and a string places it as a submenu of that menu.

```php
#[ShowInMenu]
// or explicitly
#[ShowInMenu(true)]
// or to hide from menu
#[ShowInMenu(false)]
// or as a submenu of another menu
#[ShowInMenu('tools.php')]
```

#### `ShowInNavMenus`

Controls whether this post type is available for selection in navigation menus.

```php
#[ShowInNavMenus]
// or explicitly
#[ShowInNavMenus(true)]
// or to hide from nav menus
#[ShowInNavMenus(false)]
```

#### `QueryVar`

Sets the query_var key for this post type. If false, no query var is registered.

```php
#[QueryVar('event')]
// or to use the post type name as query var
#[QueryVar(true)]
// or to disable query var
#[QueryVar(false)]
```

#### `Rewrite`

Sets the rewrite rules for the post type. Can be a boolean or an array of options.

```php
#[Rewrite(['slug' => 'events', 'with_front' => false])]
// or to use default rewrite rules
#[Rewrite(true)]
// or to disable rewrite rules
#[Rewrite(false)]
```

#### `Capabilities`

Provides an array of custom capabilities for this post type.

```php
#[Capabilities([
    'edit_post' => 'edit_event',
    'read_post' => 'read_event',
    'delete_post' => 'delete_event',
    'edit_posts' => 'edit_events',
    'edit_others_posts' => 'edit_others_events'
])]
```

#### `Label`

Sets the singular label for the post type. This is used in the admin UI.

```php
#[Label('Event')]
```

#### `Description`

Sets the description for the post type. This is shown in the admin UI.

```php
#[Description('Custom events for the website')]
```

#### `RegisterMetaBoxCb`

Defines a callback function that will be called when setting up meta boxes for the edit form. This attribute is applied to methods that will be used as callbacks.

```php
class Event extends AbstractPostType
{
    // ... other methods ...
    
    #[RegisterMetaBoxCb]
    public function registerEventMetaBoxes($post): void
    {
        add_meta_box(
            'event_details',
            'Event Details',
            [$this, 'renderEventDetailsMetaBox'],
            null,
            'normal',
            'default'
        );
    }
    
    public function renderEventDetailsMetaBox($post): void
    {
        // Render the meta box content
        echo '<div class="event-details-meta-box">';
        // ... your meta box HTML ...
        echo '</div>';
    }
}
```

#### `Taxonomies`

Connects the post type with specific taxonomies.

```php
#[Taxonomies(['category', 'post_tag', 'event_type'])]
```

#### `Template`

Defines a default block template for the post type in the block editor.

```php
#[Template([
    ['core/heading', ['level' => 2, 'placeholder' => 'Event Title']],
    ['core/paragraph', ['placeholder' => 'Event description...']]
])]
```

#### `TemplateLock`

Controls whether the template can be modified in the block editor.

```php
#[TemplateLock('all')] // Locks the template completely
// or
#[TemplateLock('insert')] // Prevents adding or removing blocks, but allows moving
// or
#[TemplateLock(false)] // No lock
```

#### `Archive`

Configures the archive query for the post type. This is a feature provided by the Extended CPTs library.

```php
#[Archive(['nopaging' => true, 'orderby' => 'title'])]
```

#### `AdminFilters`

Adds filterable columns to the admin list table. This is a feature provided by the Extended CPTs library.

```php
#[AdminFilters(['date', 'author', 'taxonomy' => 'event_type'])]
```

#### `FeaturedImage`

Customizes the featured image label for the post type.

```php
#[FeaturedImage('Event Cover Image')]
```

#### `SiteFilters`

Adds filterable columns to the front-end queries. This is a feature provided by the Extended CPTs library.

```php
#[SiteFilters(['date', 'author', 'taxonomy' => 'event_type'])]
```

#### `SiteSortables`

Adds sortable columns for front-end queries. This is a feature provided by the Extended CPTs library.

```php
#[SiteSortables(['title' => 'Title', 'date' => 'Date', 'author' => 'Author'])]
```

#### `DashboardGlance`

Shows the post type in the "At a Glance" widget on the WordPress dashboard.

```php
#[DashboardGlance]
// or explicitly
#[DashboardGlance(true)]
// or to hide from At a Glance widget
#[DashboardGlance(false)]
```

#### `AdminCols`

Configures the columns displayed in the admin list table for this post type.

```php
#[AdminCols([
    'title' => [
        'title'    => 'Event Title',
        'function' => function($post_id) {
            echo get_the_title($post_id);
        },
    ],
    'date' => [
        'title'    => 'Published',
        'default'  => 'ASC',
    ],
    'featured_image' => [
        'title'    => 'Image',
        'function' => function($post_id) {
            echo get_the_post_thumbnail($post_id, [50, 50]);
        },
        'width'    => 80,
    ]
])]
```

#### `AdminCol`

Defines an admin column for a post type. This attribute is applied to methods that generate the content for the column.

```php
class Event extends AbstractPostType
{
    // ... other methods ...
    
    #[AdminCol('title', 'Event Title')]
    public function formatTitle($postId): void
    {
        echo get_the_title($postId);
    }
    
    #[AdminCol('featured_image', 'Featured Image')]
    public function displayImage($postId): void
    {
        echo get_the_post_thumbnail($postId, [50, 50]);
    }
}
```

The first parameter is the column key, the second is the column title.

#### `Chronological`

Sets the post type to be displayed in chronological order (newest first) in admin lists and queries.

```php
#[Chronological]
// or explicitly
#[Chronological(true)]
```

This is a convenience attribute that sets the default ordering to date, in descending order.

### Automatic Registration

Post types defined with attributes are automatically discovered and registered with WordPress. You don't need to manually register them or add them to a configuration file. 
