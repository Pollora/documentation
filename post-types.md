# Custom Post Types

The `config/post-types.php` file allows you to create your own custom post types.

Pollora integrates the [Extended CPTs](https://github.com/johnbillion/extended-cpts) library and a dedicated post types service provider. This setup allows you to configure post types directly from a config file.

By default, there is a post type called `supplier`. You can delete or replace it with a type that better suits your needs.

All [parameters provided by Extended CPTs](https://github.com/johnbillion/extended-cpts/wiki/Registering-Post-Types) can be used. An example post type included in Pollora demonstrates how to change the names associated with `singular`, `plural`, and `slug`.

## Creating Multiple Custom Post Types

To create multiple custom post types, simply add new keys to the `post_types` array in the `config/post-types.php` file. The example below illustrates the creation of two post types: `supplier` and `book`.

```php
return [
    'supplier' => [
        'public' => true,
        'capability_type' => 'page',
        'label' => 'Supplier',
        'map_meta_cap' => true,
        'menu_position' => 5,
        'hierarchical' => false,
        'rewrite' => false,
        'query_var' => false,
        'delete_with_user' => true,
        'supports' => [
            'title',
            'revisions',
        ],
        'names' => [
            'singular' => 'Supplier',
            'plural' => 'Suppliers',
            'slug' => 'supplier',
        ],
    ],
    'book' => [
        'menu_icon' => 'dashicons-book',
        'supports' => [
            'title',
            'editor',
            'author', 'thumbnail'
        ],
        'show_in_rest' => true,
    ],
];
```

## Using the PostType Class

Instead of using the configuration file, you can also use the fluent PostType class. This approach is more readable and provides an object-oriented method to declaratively set post types.

Each method has been designed to be easy to write and understand, based on the naming conventions of WordPress parameters and the Extended CPT package. Additional methods have been introduced to enhance clarity.

Here's an example:

```php
PostType::make('form-submission', 'Form Submission', 'Form Submissions')
    ->titlePlaceholder('The custom title placeholder')
    ->private()
    ->chronological()
    ->showInAdminBar();
```

In this example, we're creating a post type with the slug `form-submission`, the singular label `Form Submission`, and the plural label `Form Submissions`. We specify a custom title placeholder, mark the post type as private, order it chronologically, and display it in the admin menu bar.

### Important: Use of the Service Provider

It is essential that the declaration of post types is not done in a **Hookable** class but rather in a **Service Provider**. The service provider is responsible for attaching the post type declaration via the WordPress `init` hook. If the declaration is made in a hookable class without using a service provider, the post type arguments might not be properly registered.
