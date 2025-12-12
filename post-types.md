# Post Types

Pollora offers two different ways to create custom post types:

1. **Using PHP Attributes (Recommended)** - A modern, declarative approach using PHP 8 attributes
2. **Using the Configuration File** - Define post types in the `config/post-types.php` file

## 1. Using PHP Attributes (Recommended)

The most elegant way to define custom post types in Pollora is by using PHP 8 attributes. This approach provides a clean, declarative syntax and better organization of your post types.

### Creating a Post Type with Attributes

You can create a post type class anywhere in your application. The simplest approach is to create a class with the `#[PostType]` attribute:

```php
<?php

namespace App\Cms\PostTypes;

use Pollora\Attributes\PostType;
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
#[PostType]
#[PubliclyQueryable]
#[HasArchive]
#[Supports(['title', 'editor', 'thumbnail'])]
#[MenuIcon('dashicons-calendar')]
class Event
{
}
```

### The `#[PostType]` Attribute

The `#[PostType]` attribute is the foundation for defining custom post types. It accepts optional parameters:

```php
#[PostType]                                          // Auto-generated slug, singular, and plural
#[PostType('book')]                                  // Custom slug, auto-generated names
#[PostType('product', singular: 'Product', plural: 'Products')]  // All custom
```

**Parameters:**
- `$slug` (optional): The post type slug. If not provided, automatically generated from class name using kebab-case
- `$singular` (optional): The singular name. If not provided, automatically generated from class name
- `$plural` (optional): The plural name. If not provided, automatically pluralized from singular name

### Auto-Generation Examples

When you don't specify parameters, Pollora automatically generates them from your class name:

```php
#[PostType]
class Event {}
// Generates: slug="event", singular="Event", plural="Events"

#[PostType]
class EventCategory {}
// Generates: slug="event-category", singular="Event Category", plural="Event Categories"

#[PostType]
class BookReview {}
// Generates: slug="book-review", singular="Book Review", plural="Book Reviews"
```

All labels (menu names, descriptions, etc.) are automatically generated based on the singular and plural names, providing a complete WordPress post type configuration with minimal code.

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
#[PostType]
class Event
{
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

Defines an admin column for a post type. This attribute is applied to methods that generate the content for the column. It supports all the features provided by the [Extended CPTs library](https://github.com/johnbillion/extended-cpts/wiki/Admin-columns).

```php
#[PostType]
class Event
{
    #[AdminCol('title', 'Event Title')]
    public function formatTitle($postId): void
    {
        echo get_the_title($postId);
    }
    
    #[AdminCol('featured_image', 'Featured Image', width: 80)]
    public function displayImage($postId): void
    {
        echo get_the_post_thumbnail($postId, [50, 50]);
    }
    
    #[AdminCol('event_date', 'Event Date', sortable: true, dateFormat: 'd/m/Y')]
    public function displayEventDate($postId): void
    {
        $date = get_post_meta($postId, 'event_date', true);
        echo date('d/m/Y', strtotime($date));
    }
    
    #[AdminCol('price', 'Price', sortable: 'meta_value_num', metaKey: 'event_price')]
    public function displayPrice($postId): void
    {
        $price = get_post_meta($postId, 'event_price', true);
        echo '$' . number_format($price, 2);
    }
}
```

**Parameters:**

- `$key` (string): The column key
- `$title` (string): The column title
- `$sortable` (bool|string, optional): Enable sorting. Use `true` for basic sorting, or specify a field like `'meta_value_num'` for custom sorting
- `$titleIcon` (string|null, optional): Dashicon for column header (e.g., `'dashicons-calendar'`)
- `$dateFormat` (string|null, optional): Date format for date-based fields (e.g., `'d/m/Y'`)
- `$link` (string|null, optional): Link behavior: `'view'`, `'edit'`, `'list'`, or `'none'`
- `$cap` (string|null, optional): User capability required to view column
- `$postCap` (string|null, optional): Capability required to view column content
- `$default` (string|null, optional): Set as default sorting column (`'ASC'` or `'DESC'`)
- `$metaKey` (string|null, optional): For meta field columns
- `$taxonomy` (string|null, optional): For taxonomy columns
- `$featuredImage` (string|null, optional): Featured image size (e.g., `'thumbnail'`)
- `$postField` (string|null, optional): Standard post table field
- `$width` (int|null, optional): Column width in pixels

**Extended CPT Features:**

You can also use specialized column types without custom methods:

```php
#[PostType]
class Product
{
    // Meta field column with automatic sorting
    #[AdminCol('price', 'Price', sortable: 'meta_value_num', metaKey: 'product_price')]
    public function dummyPrice() {} // Method name doesn't matter for meta columns
    
    // Taxonomy column
    #[AdminCol('categories', 'Categories', taxonomy: 'product_category')]
    public function dummyCategories() {}
    
    // Featured image column
    #[AdminCol('image', 'Image', featuredImage: 'thumbnail', width: 80)]
    public function dummyImage() {}
    
    // Post field column (built-in WordPress fields)
    #[AdminCol('author', 'Author', postField: 'post_author', link: 'edit')]
    public function dummyAuthor() {}
}
```

**Advanced Examples:**

```php
#[PostType]
class Event
{
    // Column with custom icon and capability restriction
    #[AdminCol(
        'vip_status',
        'VIP Event',
        titleIcon: 'dashicons-star-filled',
        cap: 'manage_options',
        sortable: true
    )]
    public function displayVipStatus($postId): void
    {
        $isVip = get_post_meta($postId, 'is_vip_event', true);
        echo $isVip ? '★ VIP' : 'Regular';
    }
    
    // Date column with custom format and default sorting
    #[AdminCol(
        'start_date',
        'Start Date',
        dateFormat: 'M j, Y',
        default: 'ASC',
        sortable: true
    )]
    public function displayStartDate($postId): void
    {
        $date = get_post_meta($postId, 'event_start_date', true);
        echo date('M j, Y', strtotime($date));
    }
}
```

#### `Priority`

Sets the priority for the post type registration during WordPress initialization. Lower numbers execute earlier.

```php
#[Priority(10)]  // Registers later
#[Priority(5)]   // Default priority
#[Priority(1)]   // Registers early
```

**Parameters:**
- `$priority` (int): The registration priority (default: 5)

This determines when the post type is registered during WordPress initialization. Use lower priorities for foundation types that other types depend on, and higher priorities for extended types.

#### `Chronological`

Sets the post type to be displayed in chronological order (newest first) in admin lists and queries.

```php
#[Chronological]
// or explicitly
#[Chronological(true)]
```

This is a convenience attribute that sets the default ordering to date, in descending order.

### Automatic Registration

Post types defined with the `#[PostType]` attribute are automatically discovered and registered with WordPress. You don't need to manually register them or add them to a configuration file.

### Advanced Usage

For more complex scenarios, you can add additional methods to your post type classes:

```php
#[PostType('book')]
#[PubliclyQueryable]
#[HasArchive]
class Book
{
    /**
     * Add custom arguments to the post type registration.
     * This method will be called automatically if it exists.
     */
    public function withArgs(): array
    {
        return [
            'description' => 'A custom book post type for our library',
            'show_in_rest' => true,
        ];
    }
    
    /**
     * Custom method for additional functionality.
     */
    public function getBookDetails(): array
    {
        // Your custom logic here
        return [];
    }
}
```

The `withArgs()` method, if present, allows you to provide additional WordPress post type arguments that will be merged with those generated from attributes.

### The `configuring()` Lifecycle Hook

The `configuring()` method provides a powerful way to programmatically configure your post type before registration. This method receives an Entity PostType instance that gives you access to the full Entity API for dynamic configuration.

```php
#[PostType('book')]
#[PubliclyQueryable]
#[HasArchive]
class Book
{
    /**
     * Configure the post type before registration.
     * This method is called automatically during discovery.
     */
    public function configuring(\Pollora\Entity\Domain\Model\PostType $postType): void
    {
        // Set custom labels
        $postType->labels([
            'name' => 'Library Books',
            'singular_name' => 'Library Book',
            'add_new_item' => 'Add New Book to Library',
            'edit_item' => 'Edit Library Book',
        ]);
        
        // Configure supports
        $postType->supports(['title', 'editor', 'thumbnail', 'custom-fields']);
        
        // Set custom capabilities
        $postType->capabilityType('book');
        
        // Configure REST API
        $postType->showInRest(true);
        $postType->restBase('library-books');
        
        // Set custom rewrite rules
        $postType->rewrite([
            'slug' => 'library/books',
            'with_front' => false,
        ]);
    }
}
```

**Key Features:**

- **Entity API Access**: The `configuring()` method receives a full Entity PostType instance, giving you access to all Entity methods like `labels()`, `supports()`, `capabilityType()`, etc.
- **Priority Over Attributes**: Configurations made in `configuring()` take priority over attribute configurations. However, attributes still provide missing values.
- **Smart Merging**: Labels are intelligently merged - your custom labels are used as the base, with missing labels automatically filled from attribute configurations.

**Available Entity Methods:**

The Entity PostType instance provides methods for all WordPress post type arguments:

```php
public function configuring(\Pollora\Entity\Domain\Model\PostType $postType): void
{
    // Labels and descriptions
    $postType->label('Custom Label');
    $postType->labels(['name' => 'Books', 'singular_name' => 'Book']);
    $postType->description('A custom book post type');
    
    // Visibility and UI
    $postType->public(true);
    $postType->publiclyQueryable(true);
    $postType->showUi(true);
    $postType->showInMenu(true);
    $postType->showInNavMenus(true);
    $postType->showInAdminBar(true);
    
    // Features and capabilities
    $postType->supports(['title', 'editor', 'thumbnail']);
    $postType->capabilityType('post');
    $postType->mapMetaCap(true);
    
    // Archive and hierarchical
    $postType->hasArchive(true);
    $postType->hierarchical(false);
    
    // REST API
    $postType->showInRest(true);
    $postType->restBase('books');
    $postType->restNamespace('wp/v2');
    
    // URL rewriting
    $postType->rewrite(['slug' => 'books', 'with_front' => false]);
    $postType->queryVar(true);
    
    // Menu configuration
    $postType->menuPosition(5);
    $postType->menuIcon('dashicons-book');
    
    // Search and export
    $postType->excludeFromSearch(false);
    $postType->canExport(true);
    $postType->deleteWithUser(false);
}
```

**Use Cases:**

1. **Dynamic Labels**: Configure labels based on user locale or site settings
2. **Conditional Features**: Enable/disable features based on site configuration
3. **Complex Rewrite Rules**: Set up sophisticated URL structures
4. **Integration Setup**: Configure REST API endpoints or custom capabilities
5. **Theme-Specific Configuration**: Adjust post type behavior based on active theme

**Best Practices:**

- Use attributes for static configuration and `configuring()` for dynamic behavior
- Leverage the smart merging - define base labels in `configuring()` and let attributes fill gaps
- Keep the method focused on configuration logic, avoid business logic
- Use type hints for better IDE support: `\Pollora\Entity\Domain\Model\PostType $postType`

## 2. Using the Configuration File

For projects that prefer configuration over attributes, you can define post types in the `config/post-types.php` file. Please refer to the configuration documentation for details on this approach. 
