# Post Types

Pollora uses PHP 8 attributes to declare custom post types. This approach provides a clean, declarative syntax and better organization of your post types.

## Using PHP Attributes

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
- `$textDomain` (optional): The text domain for auto-generated label translations. Defaults to `'pollora'`

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

### Common Attributes

Pollora provides many attributes to configure your post types. Here are the most commonly used ones:

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

#### `Hierarchical`

Makes the post type hierarchical like pages, allowing for parent/child relationships.

```php
#[Hierarchical]
// or explicitly
#[Hierarchical(true)]
// or to make non-hierarchical
#[Hierarchical(false)]
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

#### `Labels`

Overrides specific labels with named parameters. Only the labels you provide are overridden; the rest keep their auto-generated values.

```php
#[Labels(
    addNew: 'New Project',
    notFound: 'No projects found.',
    notFoundInTrash: 'No projects found in trash.',
)]
```

> **Note:** PHP attributes only accept constant expressions, so `__()` cannot be used here. For translatable labels, use `withArgs()` or `configuring()` instead (see [Internationalization](#internationalization) below).

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

#### `Rewrite`

Sets the rewrite rules for the post type. Can be a boolean or an array of options.

```php
#[Rewrite(['slug' => 'events', 'with_front' => false])]
// or to use default rewrite rules
#[Rewrite(true)]
// or to disable rewrite rules
#[Rewrite(false)]
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
        $postType->showInRest();
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
    
    // Visibility and UI (parameterless = true, use inverse methods for false)
    $postType->public();             // or ->private()
    $postType->publiclyQueryable();  // or ->notPubliclyQueryable()
    $postType->showUi();             // or ->hideUi()
    $postType->showInMenu();         // or ->hideFromMenu()
    $postType->showInNavMenus();     // or ->hideFromNavMenus()
    $postType->showInAdminBar();     // or ->hideFromAdminBar()
    
    // Features and capabilities
    $postType->supports(['title', 'editor', 'thumbnail']);
    $postType->capabilityType('post');
    $postType->mapMetaCap();         // or ->withoutMetaCap()
    
    // Archive and hierarchical
    $postType->hasArchive(true);     // accepts bool|string
    $postType->nonHierarchical();    // or ->hierarchical()
    
    // REST API
    $postType->showInRest();         // or ->hideFromRest()
    $postType->restBase('books');
    $postType->restNamespace('wp/v2');
    
    // URL rewriting
    $postType->rewrite(['slug' => 'books', 'with_front' => false]);
    $postType->queryVar(true);       // accepts bool|string
    
    // Menu configuration
    $postType->menuPosition(5);
    $postType->menuIcon('dashicons-book');
    
    // Search and export
    $postType->includeInSearch();    // or ->excludeFromSearch()
    $postType->canExport();          // or ->cannotExport()
    $postType->keepOnUserDelete();   // or ->deleteWithUser()
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

### Internationalization

Pollora provides three approaches for translatable post type labels, depending on your needs:

#### Auto-Generated Labels (Default)

When no labels are specified, Pollora auto-generates them using `sprintf(__('Edit %s', 'pollora'), $singular)`. These patterns are extractible and included in the framework's `.pot` file. The `textDomain` parameter lets you use your own domain:

```php
#[PostType('project', textDomain: 'my-theme')]
```

#### Static Overrides with `#[Labels]`

The `#[Labels]` attribute provides partial label overrides using named parameters. However, PHP attributes only accept constant expressions — `__()` calls are not allowed in attributes.

```php
#[PostType]
#[Labels(
    addNew: 'New Project',
    notFound: 'No projects found.',
)]
class Project {}
```

#### Translatable Labels with `withArgs()`

For fully translatable labels, use `withArgs()` where `__()` calls are evaluated at runtime and extractible by `wp i18n make-pot`:

```php
#[PostType]
#[HasArchive]
class Service
{
    public function withArgs(): array
    {
        return [
            'labels' => [
                'name' => __('Services', 'my-theme'),
                'singular_name' => __('Service', 'my-theme'),
                'add_new' => __('Add New', 'my-theme'),
                'add_new_item' => __('Add New Service', 'my-theme'),
                'edit_item' => __('Edit Service', 'my-theme'),
                'all_items' => __('All Services', 'my-theme'),
            ],
        ];
    }
}
```

#### Translatable Labels with `configuring()`

The `configuring()` hook also supports `__()` for translatable labels with access to the full Entity API:

```php
#[PostType]
class Event
{
    public function configuring(\Pollora\Entity\Domain\Model\PostType $postType): void
    {
        $postType->labels([
            'name' => __('Events', 'my-theme'),
            'singular_name' => __('Event', 'my-theme'),
            'add_new_item' => __('Add New Event', 'my-theme'),
        ]);
    }
}
```

**Summary:**

| Approach | Translatable | Extractible by `make-pot` | Use case |
|----------|-------------|---------------------------|----------|
| Auto-generated | Yes (via framework `.pot`) | Yes | Default behavior, no code needed |
| `#[Labels]` | No | No | Quick static overrides |
| `withArgs()` | Yes | Yes | Full i18n support |
| `configuring()` | Yes | Yes | Dynamic + i18n support |

For a complete list of all available attributes, see [Post Type Attributes Reference](post-types-reference.md).
