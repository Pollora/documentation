# Taxonomies

Just like the implementation of post types, Pollora leverages the [Extended CPTs](https://github.com/johnbillion/extended-cpts) library, coupled with a post types service provider, to enable taxonomy configuration from a config file.

By default, the `customer_type` taxonomy is defined and linked to the `supplier` post type. You can remove this taxonomy or replace it with one that better suits your needs.

Here is an example configuration :

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

All [parameters available in Extended CPTs](https://github.com/johnbillion/extended-cpts/wiki/Registering-taxonomies) are supported.

Rather than relying on the configuration file to define taxonomies, consider using the Taxonomy class which offers a streamlined, object-oriented approach. This method presents a clearer way of defining taxonomies in a fluent style.

The methods integrated within this class were crafted for simplicity in both writing and comprehension. They resonate with the conventions set by WordPress for taxonomy parameters and are influenced by the Extended CPT package. Additional methods are in place to provide further clarity regarding their purpose.

Here's a demonstration:

```php
Taxonomy::make('product_cat', 'Product Category', 'Product Categories')
    ->showInQuickEdit()
    ->showTagcloud()
    ->sort()
    ->showInAdminBar();
```

In this example, we're establishing a custom taxonomy with the slug 'product_cat', a singular label of 'Product Category', and a plural label of 'Product Categories'. Further configurations indicate that it should appear in quick edits, be part of the tag cloud, have sorting capabilities, and be visible in the admin menu bar. A neat way to express taxonomy configurations, right? :)

### Important: Use of the Service Provider

It is essential that the declaration of taxonomies is not done in a **Hookable** class but rather in a **Service Provider**. The service provider is responsible for attaching the taxonomy declaration via the WordPress `init` hook. If the declaration is made in a hookable class without using a service provider, the taxonomy arguments might not be properly registered.
