# Taxonomies

Pollora offers two different ways to create custom taxonomies:

1. **Using PHP Attributes (Recommended)** - A modern, declarative approach using PHP 8 attributes
2. **Using the Configuration File** - Define taxonomies in the `config/taxonomies.php` file

## 1. Using PHP Attributes (Recommended)

The most elegant way to define custom taxonomies in Pollora is by using PHP 8 attributes. This approach provides a clean, declarative syntax and better organization of your taxonomies.

### Creating a Taxonomy with Attributes

You can create a taxonomy class anywhere in your application. The simplest approach is to create a class with the `#[Taxonomy]` attribute:

```php
<?php

namespace App\Cms\Taxonomies;

use Pollora\Attributes\Taxonomy;
use Pollora\Attributes\Taxonomy\ShowTagcloud;
use Pollora\Attributes\Taxonomy\ShowInQuickEdit;
use Pollora\Attributes\Taxonomy\ShowAdminColumn;
use Pollora\Attributes\Taxonomy\Hierarchical;
use Pollora\Attributes\Taxonomy\ShowInRest;

/**
 * EventType taxonomy.
 *
 * This class defines the EventType custom taxonomy using PHP attributes
 * for WordPress registration.
 */
#[Taxonomy(objectType: ['event', 'post'])]
#[ShowTagcloud]
#[ShowInQuickEdit]
#[ShowAdminColumn]
#[Hierarchical]
#[ShowInRest]
class EventType
{
    #[MetaBoxCb]
    public function customMetaBox($post, $box): void
    {
        // Custom meta box display logic
        echo '<div class="custom-meta-box">';
        // ... your custom meta box HTML ...
        echo '</div>';
    }

    #[MetaBoxSanitizeCb]
    public function sanitizeMetaBoxData($termId, $taxonomy): void
    {
        // Sanitize the taxonomy data
        // ...
    }

    #[UpdateCountCallback]
    public function updateTermCount($terms, $taxonomy): void
    {
        // Custom term count update logic
        // ...
    }
}
```

### The `#[Taxonomy]` Attribute

The `#[Taxonomy]` attribute is the foundation for defining custom taxonomies. It accepts optional parameters:

```php
#[Taxonomy]                                          // Auto-generated slug, singular, and plural; defaults to 'post'
#[Taxonomy('product-type')]                          // Custom slug, auto-generated names; defaults to 'post'
#[Taxonomy('event-category', singular: 'Event Category', plural: 'Event Categories')]  // Custom names; defaults to 'post'
#[Taxonomy(objectType: ['post', 'page'])]           // Auto-generated names; custom object types
#[Taxonomy('genre', singular: 'Genre', plural: 'Genres', objectType: 'book')]  // All custom
```

**Parameters:**
- `$slug` (optional): The taxonomy slug. If not provided, automatically generated from class name using kebab-case
- `$singular` (optional): The singular name. If not provided, automatically generated from class name
- `$plural` (optional): The plural name. If not provided, automatically pluralized from singular name
- `$objectType` (optional): The post types this taxonomy applies to. Can be a string or array. Defaults to `['post']`

### Auto-Generation Examples

When you don't specify parameters, Pollora automatically generates them from your class name:

```php
#[Taxonomy]
class Category {}
// Generates: slug="category", singular="Category", plural="Categories", objectType=['post']

#[Taxonomy]
class EventType {}
// Generates: slug="event-type", singular="Event Type", plural="Event Types", objectType=['post']

#[Taxonomy]
class ProductCategory {}
// Generates: slug="product-category", singular="Product Category", plural="Product Categories", objectType=['post']

#[Taxonomy(objectType: ['book', 'review'])]
class BookGenre {}
// Generates: slug="book-genre", singular="Book Genre", plural="Book Genres", objectType=['book', 'review']
```

All labels (menu names, descriptions, etc.) are automatically generated based on the singular and plural names, providing a complete WordPress taxonomy configuration with minimal code.

### Available Attributes

Pollora provides several attributes to configure your taxonomies:

#### `ObjectType`

Définit les types de posts auxquels cette taxonomie est associée.

```php
#[ObjectType('post')]
// ou avec plusieurs types de posts
#[ObjectType(['post', 'page', 'product'])]
```

#### `ShowTagcloud`

Determines whether to list the taxonomy in the tag cloud widget controls.

```php
#[ShowTagcloud]
// or explicitly
#[ShowTagcloud(true)]
// or to hide from tag cloud
#[ShowTagcloud(false)]
```

#### `ShowInQuickEdit`

Determines whether to show the taxonomy in the quick/bulk edit panel.

```php
#[ShowInQuickEdit]
// or explicitly
#[ShowInQuickEdit(true)]
// or to hide from quick edit
#[ShowInQuickEdit(false)]
```

#### `ShowAdminColumn`

Determines whether to display a column for the taxonomy on its post type listing screens.

```php
#[ShowAdminColumn]
// or explicitly
#[ShowAdminColumn(true)]
// or to hide admin column
#[ShowAdminColumn(false)]
```

#### `MetaBoxCb`

Defines a callback function for the meta box display. This attribute is applied to methods that will be used as callbacks.

```php
#[MetaBoxCb]
```

#### `MetaBoxSanitizeCb`

Defines a callback function for sanitizing taxonomy data saved from a meta box. This attribute is applied to methods that will be used as callbacks.

```php
#[MetaBoxSanitizeCb]
```

#### `UpdateCountCallback`

Defines a callback function that will be called when the count is updated. This attribute is applied to methods that will be used as callbacks.

```php
#[UpdateCountCallback]
```

#### `DefaultTerm`

Sets the default term name for this taxonomy.

```php
#[DefaultTerm('Uncategorized')]
// or with additional options
#[DefaultTerm(['name' => 'Uncategorized', 'slug' => 'uncategorized', 'description' => 'Default term'])]
```

#### `Sort`

Determines whether terms in this taxonomy should be sorted in the order they are provided to `wp_set_object_terms()`.

```php
#[Sort]
// or explicitly
#[Sort(true)]
// or to disable sorting
#[Sort(false)]
```

#### `Args`

Sets an array of arguments to automatically use inside `wp_get_object_terms()` for this taxonomy.

```php
#[Args(['orderby' => 'name', 'order' => 'ASC'])]
```

#### `CheckedOntop`

Determines whether checked terms should appear on top.

```php
#[CheckedOntop]
// or explicitly
#[CheckedOntop(true)]
// or to disable
#[CheckedOntop(false)]
```

#### `Exclusive`

Makes the taxonomy exclusive, meaning a post can only have one term from this taxonomy.

```php
#[Exclusive]
// or explicitly
#[Exclusive(true)]
// or to disable
#[Exclusive(false)]
```

#### `AllowHierarchy`

Allows hierarchy in the taxonomy's rewrite rules.

```php
#[AllowHierarchy]
// or explicitly
#[AllowHierarchy(true)]
// or to disable
#[AllowHierarchy(false)]
```

In addition to these taxonomy-specific attributes, you can also use common attributes:

#### `PublicTaxonomy`

Sets the taxonomy as public, making it visible in the admin UI and on the front end.

```php
#[PublicTaxonomy]
// or explicitly
#[PublicTaxonomy(true)]
// or to make private
#[PublicTaxonomy(false)]
```

#### `Label`

Sets the singular label for the taxonomy.

```php
#[Label('Event Type')]
```

#### `Description`

Sets the description for the taxonomy.

```php
#[Description('Event types for the website')]
```

#### `ShowUI`

Controls whether to generate a default UI for managing this taxonomy in the admin.

```php
#[ShowUI]
// or explicitly
#[ShowUI(true)]
// or to hide UI
#[ShowUI(false)]
```

#### `ShowInNavMenus`

Controls whether this taxonomy is available for selection in navigation menus.

```php
#[ShowInNavMenus]
// or explicitly
#[ShowInNavMenus(true)]
// or to hide from nav menus
#[ShowInNavMenus(false)]
```

#### `QueryVar`

Sets the query_var key for this taxonomy.

```php
#[QueryVar('event_type')]
// or to use the taxonomy name as query_var
#[QueryVar(true)]
// or to disable query_var
#[QueryVar(false)]
```

#### `Rewrite`

Sets the rewrite rules for the taxonomy.

```php
#[Rewrite(['slug' => 'event-types', 'with_front' => false])]
// or to use default rewrite rules
#[Rewrite(true)]
// or to disable rewrite rules
#[Rewrite(false)]
```

#### `RestNamespace`

Sets the REST API namespace for the taxonomy.

```php
#[RestNamespace('my-app/v1')]
```

#### `RestControllerClass`

Specifies the controller class that should be used for handling REST API requests for this taxonomy.

```php
#[RestControllerClass('WP_REST_Terms_Controller')]
// or a custom controller
#[RestControllerClass('MyApp\\REST\\EventTypeController')]
```

#### `RestBase`

Sets the base URL segment that will be used in REST API endpoints for this taxonomy.

```php
#[RestBase('event-types')]
```

#### `ShowInRest`

Makes the taxonomy available via the REST API.

```php
#[ShowInRest]
// or explicitly
#[ShowInRest(true)]
// or to hide from REST API
#[ShowInRest(false)]
```

#### `Hierarchical`

Makes the taxonomy hierarchical like categories, allowing for parent/child relationships.

```php
#[Hierarchical]
// or explicitly
#[Hierarchical(true)]
// or to make non-hierarchical
#[Hierarchical(false)]
```

### Automatic Registration

Taxonomies defined with the `#[Taxonomy]` attribute are automatically discovered and registered with WordPress. You don't need to manually register them or add them to a configuration file.

### Advanced Usage

For more complex scenarios, you can add additional methods to your taxonomy classes:

```php
#[Taxonomy('book-genre', objectType: 'book')]
#[Hierarchical]
#[ShowInRest]
class BookGenre
{
    /**
     * Add custom arguments to the taxonomy registration.
     * This method will be called automatically if it exists.
     */
    public function withArgs(): array
    {
        return [
            'description' => 'A custom genre taxonomy for our book library',
            'show_in_nav_menus' => true,
        ];
    }
    
    /**
     * Custom method for additional functionality.
     */
    public function getGenreDetails(): array
    {
        // Your custom logic here
        return [];
    }
}
```

The `withArgs()` method, if present, allows you to provide additional WordPress taxonomy arguments that will be merged with those generated from attributes.

## 2. Using the Configuration File

The `config/taxonomies.php` file allows you to create your own custom taxonomies.

Pollora integrates the [Extended CPTs](https://github.com/johnbillion/extended-cpts) library and a dedicated taxonomies service provider. This setup allows you to configure taxonomies directly from a config file.

By default, there is a taxonomy called `customer_type`. You can delete or replace it with a type that better suits your needs.

All [parameters provided by Extended CPTs](https://github.com/johnbillion/extended-cpts/wiki/Registering-taxonomies) can be used. An example taxonomy included in Pollora demonstrates how to change the names associated with `singular`, `plural`, and `slug`.

Here's an example configuration:

```php
return [
    'customer_type' => [
        'labels' => [
            'singular' => 'Customer Type',
            'plural' => 'Customer Types',
        ],
        'links' => ['supplier'],
        'meta_box' => 'radio',
        'names' => [
            'singular' => 'Customer Type',
            'plural' => 'Customer Types',
        ],
    ],
];
```

For projects that prefer configuration over attributes, you can define taxonomies in the `config/taxonomies.php` file. Please refer to the configuration documentation for details on this approach. 
