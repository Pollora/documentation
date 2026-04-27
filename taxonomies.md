# Taxonomies

Pollora uses PHP 8 attributes to declare custom taxonomies. This approach provides a clean, declarative syntax and better organization of your taxonomies.

## Using PHP Attributes

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
- `$textDomain` (optional): The text domain for auto-generated label translations. Defaults to `'pollora'`

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

Defines the post types this taxonomy is associated with.

```php
#[ObjectType('post')]
// or with multiple post types
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

#### `Priority`

Sets the priority for the taxonomy registration during WordPress initialization. Lower numbers execute earlier.

```php
#[Priority(10)]  // Registers later
#[Priority(5)]   // Default priority
#[Priority(1)]   // Registers early
```

**Parameters:**
- `$priority` (int): The registration priority (default: 5)

This determines when the taxonomy is registered during WordPress initialization. Use lower priorities for foundation taxonomies that other taxonomies depend on, and higher priorities for extended taxonomies.

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

#### `Labels`

Overrides specific labels with named parameters. Only the labels you provide are overridden; the rest keep their auto-generated values.

```php
#[Labels(
    name: 'Project Categories',
    singularName: 'Project Category',
    addNewItem: 'Add New Category',
)]
```

Available parameters: `name`, `singularName`, `menuName`, `allItems`, `editItem`, `viewItem`, `updateItem`, `addNewItem`, `newItemName`, `searchItems`, `popularItems`, `separateItemsWithCommas`, `addOrRemoveItems`, `chooseFromMostUsed`, `notFound`, `parentItem`, `parentItemColon`, `noTerms`, `filterByItem`, `itemsListNavigation`, `itemsList`, `backToItems`.

> **Note:** PHP attributes only accept constant expressions, so `__()` cannot be used here. For translatable labels, use `withArgs()` or `configuring()` instead (see [Internationalization](#internationalization) below).

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

### The `configuring()` Lifecycle Hook

The `configuring()` method provides a powerful way to programmatically configure your taxonomy before registration. This method receives an Entity Taxonomy instance that gives you access to the full Entity API for dynamic configuration.

```php
#[Taxonomy('product-category', objectType: ['product'])]
#[Hierarchical]
#[ShowInRest]
class ProductCategory
{
    /**
     * Configure the taxonomy before registration.
     * This method is called automatically during discovery.
     */
    public function configuring(\Pollora\Entity\Domain\Model\Taxonomy $taxonomy): void
    {
        // Set custom labels
        $taxonomy->labels([
            'name' => 'Product Categories',
            'singular_name' => 'Product Category',
            'add_new_item' => 'Add New Category',
            'edit_item' => 'Edit Category',
            'all_items' => 'All Product Categories',
        ]);
        
        // Configure rewrite rules
        $taxonomy->rewrite([
            'slug' => 'shop/categories',
            'with_front' => false,
            'hierarchical' => true,
        ]);
        
        // REST API configuration
        $taxonomy->showInRest();
        $taxonomy->restBase('product-categories');
        
        // Configure meta box behavior
        $taxonomy->showInQuickEdit();
        $taxonomy->showAdminColumn();
    }
}
```

**Key Features:**

- **Entity API Access**: The `configuring()` method receives a full Entity Taxonomy instance, giving you access to all Entity methods like `labels()`, `rewrite()`, `showInRest()`, etc.
- **Priority Over Attributes**: Configurations made in `configuring()` take priority over attribute configurations. However, attributes still provide missing values.
- **Smart Merging**: Labels are intelligently merged - your custom labels are used as the base, with missing labels automatically filled from attribute configurations.

**Available Entity Methods:**

The Entity Taxonomy instance provides methods for all WordPress taxonomy arguments:

```php
public function configuring(\Pollora\Entity\Domain\Model\Taxonomy $taxonomy): void
{
    // Labels and descriptions
    $taxonomy->label('Custom Label');
    $taxonomy->labels(['name' => 'Categories', 'singular_name' => 'Category']);
    $taxonomy->description('A custom category taxonomy');
    
    // Visibility and UI (parameterless = true, use inverse methods for false)
    $taxonomy->public();             // or ->private()
    $taxonomy->publiclyQueryable();  // or ->notPubliclyQueryable()
    $taxonomy->showUi();             // or ->hideUi()
    $taxonomy->showInMenu();         // or ->hideFromMenu()
    $taxonomy->showInNavMenus();     // or ->hideFromNavMenus()
    
    // Hierarchy and structure
    $taxonomy->hierarchical();       // or ->nonHierarchical()
    $taxonomy->sort();               // or ->unsort()
    
    // Admin interface
    $taxonomy->showAdminColumn();    // or ->hideAdminColumn()
    $taxonomy->showInQuickEdit();    // or ->hideFromQuickEdit()
    $taxonomy->showTagcloud();       // or ->hideFromTagcloud()
    
    // REST API
    $taxonomy->showInRest();         // or ->hideFromRest()
    $taxonomy->restBase('categories');
    $taxonomy->restNamespace('wp/v2');
    $taxonomy->restControllerClass('WP_REST_Terms_Controller');
    
    // URL rewriting
    $taxonomy->rewrite([
        'slug' => 'categories',
        'with_front' => false,
        'hierarchical' => true
    ]);
    $taxonomy->queryVar(true);
    
    // Advanced features
    $taxonomy->updateCountCallback([$this, 'updateTermCount']);
    $taxonomy->metaBoxCb([$this, 'renderMetaBox']);
}
```

**Taxonomy-Specific Methods:**

```php
public function configuring(\Pollora\Entity\Domain\Model\Taxonomy $taxonomy): void
{
    // Note: objectType is set via the #[Taxonomy] attribute, not in configuring()
    // e.g. #[Taxonomy(objectType: ['post', 'page', 'product'])]
    
    // Default term
    $taxonomy->defaultTerm([
        'name' => 'Uncategorized',
        'slug' => 'uncategorized',
        'description' => 'Default category'
    ]);
    
    // Query arguments
    $taxonomy->args(['orderby' => 'name', 'order' => 'ASC']);
    
    // Meta box behavior
    $taxonomy->checkedOntop();       // or ->uncheckedOntop()
    $taxonomy->nonExclusive();       // or ->exclusive()
    
    // Callback functions
    $taxonomy->metaBoxCb([$this, 'customMetaBox']);
    $taxonomy->metaBoxSanitizeCb([$this, 'sanitizeCallback']);
    $taxonomy->updateCountCallback([$this, 'updateCount']);
}
```

**Use Cases:**

1. **Dynamic Labels**: Configure labels based on user locale or site settings
2. **Conditional Object Types**: Associate with different post types based on site configuration
3. **Complex Rewrite Rules**: Set up sophisticated URL structures
4. **Integration Setup**: Configure REST API endpoints or custom capabilities
5. **Custom Meta Boxes**: Set up specialized admin interfaces
6. **Multi-language Support**: Configure taxonomy behavior for different languages

**Advanced Example:**

```php
#[Taxonomy('event-category', objectType: ['event'])]
#[Hierarchical]
class EventCategory
{
    public function configuring(\Pollora\Entity\Domain\Model\Taxonomy $taxonomy): void
    {
        // Dynamic labels based on site language
        $locale = get_locale();
        if ($locale === 'fr_FR') {
            $taxonomy->labels([
                'name' => 'Catégories d\'événements',
                'singular_name' => 'Catégorie d\'événement',
                'add_new_item' => 'Ajouter une nouvelle catégorie',
            ]);
        } else {
            $taxonomy->labels([
                'name' => 'Event Categories',
                'singular_name' => 'Event Category',
                'add_new_item' => 'Add New Category',
            ]);
        }
        
        // Conditional object types based on site features
        $objectTypes = ['event'];
        if (class_exists('WooCommerce')) {
            $objectTypes[] = 'product';
        }
        $taxonomy->objectType($objectTypes);
        
        // Custom rewrite structure
        $taxonomy->rewrite([
            'slug' => 'events/category',
            'with_front' => false,
            'hierarchical' => true,
        ]);
        
        // Custom meta box with color picker
        $taxonomy->metaBoxCb([$this, 'colorPickerMetaBox']);
    }
    
    public function colorPickerMetaBox($post, $box): void
    {
        // Custom meta box with category color selection
        echo '<div class="category-color-picker">';
        // ... color picker implementation ...
        echo '</div>';
    }
}
```

**Best Practices:**

- Use attributes for static configuration and `configuring()` for dynamic behavior
- Leverage the smart merging - define base labels in `configuring()` and let attributes fill gaps
- Keep the method focused on configuration logic, avoid business logic
- Use type hints for better IDE support: `\Pollora\Entity\Domain\Model\Taxonomy $taxonomy`
- Consider caching for expensive dynamic configurations

### Internationalization

Pollora provides three approaches for translatable taxonomy labels, depending on your needs:

#### Auto-Generated Labels (Default)

When no labels are specified, Pollora auto-generates them using `sprintf(__('Edit %s', 'pollora'), $singular)`. These patterns are extractible and included in the framework's `.pot` file. The `textDomain` parameter lets you use your own domain:

```php
#[Taxonomy('genre', objectType: 'book', textDomain: 'my-theme')]
```

#### Static Overrides with `#[Labels]`

The `#[Labels]` attribute provides partial label overrides using named parameters. However, PHP attributes only accept constant expressions — `__()` calls are not allowed in attributes.

```php
#[Taxonomy(objectType: 'project')]
#[Labels(
    name: 'Project Categories',
    singularName: 'Project Category',
)]
class ProjectCategory {}
```

#### Translatable Labels with `withArgs()`

For fully translatable labels, use `withArgs()` where `__()` calls are evaluated at runtime and extractible by `wp i18n make-pot`:

```php
#[Taxonomy(objectType: 'project')]
#[Hierarchical(true)]
class ProjectCategory
{
    public function withArgs(): array
    {
        return [
            'labels' => [
                'name' => __('Project Categories', 'my-theme'),
                'singular_name' => __('Project Category', 'my-theme'),
                'all_items' => __('All Categories', 'my-theme'),
                'edit_item' => __('Edit Category', 'my-theme'),
                'add_new_item' => __('Add New Category', 'my-theme'),
            ],
        ];
    }
}
```

#### Translatable Labels with `configuring()`

The `configuring()` hook also supports `__()` for translatable labels with access to the full Entity API:

```php
#[Taxonomy(objectType: 'event')]
class EventCategory
{
    public function configuring(\Pollora\Entity\Domain\Model\Taxonomy $taxonomy): void
    {
        $taxonomy->labels([
            'name' => __('Event Categories', 'my-theme'),
            'singular_name' => __('Event Category', 'my-theme'),
            'add_new_item' => __('Add New Category', 'my-theme'),
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

