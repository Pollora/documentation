# Post Types

Pollora uses PHP 8 attributes to declare custom post types, providing a clean, declarative syntax with automatic discovery and registration.

## Creating a Post Type

Create a class with the `#[PostType]` attribute anywhere in your application:

```php
<?php

namespace App\Cms\PostTypes;

use Pollora\Attributes\PostType;
use Pollora\Attributes\PostType\PubliclyQueryable;
use Pollora\Attributes\PostType\HasArchive;
use Pollora\Attributes\PostType\Supports;
use Pollora\Attributes\PostType\MenuIcon;

#[PostType]
#[PubliclyQueryable]
#[HasArchive]
#[Supports(['title', 'editor', 'thumbnail'])]
#[MenuIcon('dashicons-calendar')]
class Event
{
}
```

You can also generate a post type class using Artisan:

```bash
php artisan pollora:make:post-type Event
```

### The `#[PostType]` Attribute

```php
#[PostType]                                          // Auto-generated slug, singular, and plural
#[PostType('book')]                                  // Custom slug, auto-generated names
#[PostType('product', singular: 'Product', plural: 'Products')]  // All custom
```

**Parameters:**
- `$slug` (optional): Post type slug. Auto-generated from class name in kebab-case if omitted
- `$singular` (optional): Singular name. Auto-generated from class name if omitted
- `$plural` (optional): Plural name. Auto-pluralized from singular if omitted
- `$textDomain` (optional): Text domain for label translations. Defaults to `'pollora'`

### Auto-Generation

Pollora generates slug, singular, and plural names from your class name:

```php
#[PostType] class Event {}           // slug="event", singular="Event", plural="Events"
#[PostType] class EventCategory {}   // slug="event-category", singular="Event Category", plural="Event Categories"
#[PostType] class BookReview {}      // slug="book-review", singular="Book Review", plural="Book Reviews"
```

All labels (menu names, descriptions, etc.) are automatically generated from the singular and plural names.

## Common Attributes

### Visibility & Structure

| Attribute | Purpose | Usage |
|-----------|---------|-------|
| `PubliclyQueryable` | Front-end queryable | `#[PubliclyQueryable]` or `#[PubliclyQueryable(false)]` |
| `PublicPostType` | Visible in admin + front end | `#[PublicPostType]` or `#[PublicPostType(false)]` |
| `HasArchive` | Enable archive page | `#[HasArchive]` or `#[HasArchive('custom-slug')]` |
| `Hierarchical` | Parent/child like pages | `#[Hierarchical]` or `#[Hierarchical(false)]` |
| `Rewrite` | URL rewrite rules | `#[Rewrite(['slug' => 'events', 'with_front' => false])]` |
| `ShowInRest` | REST API + Gutenberg | `#[ShowInRest]` or `#[ShowInRest(false)]` |

### Features & Content

| Attribute | Purpose | Usage |
|-----------|---------|-------|
| `Supports` | Supported features | `#[Supports(['title', 'editor', 'thumbnail'])]` |
| `Taxonomies` | Associated taxonomies | `#[Taxonomies(['category', 'post_tag'])]` |
| `Template` | Default block template | `#[Template([['core/heading', ['level' => 2]]])]` |
| `BlockEditor` | Gutenberg on/off | `#[BlockEditor]` or `#[BlockEditor(false)]` |

### Admin UI

| Attribute | Purpose | Usage |
|-----------|---------|-------|
| `ShowUI` | Admin management UI | `#[ShowUI]` or `#[ShowUI(false)]` |
| `ShowInMenu` | Admin menu placement | `#[ShowInMenu]`, `#[ShowInMenu(false)]`, or `#[ShowInMenu('tools.php')]` |
| `MenuIcon` | Admin menu icon | `#[MenuIcon('dashicons-calendar')]` |
| `Labels` | Override specific labels | `#[Labels(addNew: 'New Project', notFound: 'None found.')]` |

> **Note:** `#[Labels]` only accepts constant expressions -- `__()` is not allowed. For translatable labels, use `withArgs()` or `configuring()` (see [Internationalization](#internationalization)).

Support options for `#[Supports]`: `title`, `editor`, `author`, `thumbnail`, `excerpt`, `comments`, `revisions`, `custom-fields`, `page-attributes`.

Post types with `#[PostType]` are automatically discovered and registered -- no manual registration or configuration needed.

## Advanced Usage

### `withArgs()` Method

Provide additional WordPress post type arguments merged with attribute-generated ones:

```php
#[PostType('book')]
#[PubliclyQueryable]
#[HasArchive]
class Book
{
    public function withArgs(): array
    {
        return [
            'description' => 'A custom book post type for our library',
            'show_in_rest' => true,
        ];
    }
}
```

### The `configuring()` Lifecycle Hook

Programmatically configure the post type before registration using the Entity API:

```php
#[PostType('book')]
#[PubliclyQueryable]
#[HasArchive]
class Book
{
    public function configuring(\Pollora\Entity\Domain\Model\PostType $postType): void
    {
        $postType->labels([
            'name' => 'Library Books',
            'singular_name' => 'Library Book',
            'add_new_item' => 'Add New Book to Library',
        ]);
        
        $postType->supports(['title', 'editor', 'thumbnail', 'custom-fields']);
        $postType->capabilityType('book');
        $postType->showInRest();
        $postType->restBase('library-books');
        $postType->rewrite(['slug' => 'library/books', 'with_front' => false]);
    }
}
```

**Key features:**
- Receives a `\Pollora\Entity\Domain\Model\PostType` instance with the full Entity API
- Configurations in `configuring()` take priority over attributes; attributes fill remaining gaps
- Labels are smart-merged: your custom labels override, missing labels auto-generate

**Available Entity methods:**

```php
public function configuring(\Pollora\Entity\Domain\Model\PostType $postType): void
{
    // Labels and descriptions
    $postType->label('Custom Label');
    $postType->labels(['name' => 'Books', 'singular_name' => 'Book']);
    $postType->description('A custom book post type');
    
    // Visibility (parameterless = true, use inverse methods for false)
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

## Internationalization

Pollora provides three approaches for translatable post type labels:

### Auto-Generated Labels (Default)

Pollora auto-generates labels using `sprintf(__('Edit %s', 'pollora'), $singular)`. The `textDomain` parameter lets you use your own domain:

```php
#[PostType('project', textDomain: 'my-theme')]
```

### Static Overrides with `#[Labels]`

Partial label overrides using named parameters (no `__()` -- constant expressions only):

```php
#[PostType]
#[Labels(addNew: 'New Project', notFound: 'No projects found.')]
class Project {}
```

### Translatable Labels with `withArgs()`

For fully translatable labels with `__()` calls evaluated at runtime:

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
                'add_new_item' => __('Add New Service', 'my-theme'),
            ],
        ];
    }
}
```

### Translatable Labels with `configuring()`

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

| Approach | Translatable | Extractible by `make-pot` | Use case |
|----------|-------------|---------------------------|----------|
| Auto-generated | Yes (via framework `.pot`) | Yes | Default behavior, no code needed |
| `#[Labels]` | No | No | Quick static overrides |
| `withArgs()` | Yes | Yes | Full i18n support |
| `configuring()` | Yes | Yes | Dynamic + i18n support |

For a complete list of all available attributes, see [Post Type Attributes Reference](post-types-reference.md).
