# Patterns

Patterns are reusable block templates that can be inserted into the Gutenberg editor. Pollora provides flexible pattern organization through a customizable folder structure.

## Configuration

Pattern categories are defined in your theme's configuration file:

```php
// themes/default/config/gutenberg.php

return [
    'patterns' => [
        'theme/patterns' => [
            'label' => 'Theme',
        ],
    ],
];
```

## Pattern Organization

### Basic Structure

Patterns are stored in your theme's `views/patterns` directory:

```
themes/default/
└── views/
    └── patterns/
        ├── example.blade.php
        └── banners/
            └── hero.blade.php
```

### Subdirectory Organization

You can organize your patterns into subdirectories for better maintenance:

```
themes/default/views/patterns/
├── banners/
│   ├── hero.blade.php
│   └── cta.blade.php
├── headers/
│   ├── simple.blade.php
│   └── complex.blade.php
└── footers/
    └── default.blade.php
```

This structure allows you to:
- Group patterns by type or function
- Maintain organized codebase
- Facilitate navigation and maintenance

## Creating Patterns

Each pattern requires a header with metadata:

```php
{{--
  Title: Hero Banner
  Slug: theme/hero-banner
  Categories: theme/patterns
--}}
```

### Complete Example

```php
{{--
  Title: Example Pattern
  Slug: theme/example
  Categories: theme/patterns
--}}

<!-- wp:group {"className":"pattern-example"} -->
<div class="wp-block-group pattern-example">
    <!-- wp:heading {"textAlign":"center","textColor":"tertiary"} -->
    <h2 class="has-text-align-center has-tertiary-color has-text-color">
        {{ get_bloginfo('title') }}
    </h2>
    <!-- /wp:heading -->

    <!-- wp:paragraph {"align":"center","textColor":"primary"} -->
    <p class="has-text-align-center has-primary-color has-text-color">
        {{ get_bloginfo('description') }}
    </p>
    <!-- /wp:paragraph -->

    <!-- wp:buttons {"layout":{"type":"flex","justifyContent":"center"}} -->
    <div class="wp-block-buttons">
        <!-- wp:button {"backgroundColor":"tertiary","style":{"border":{"radius":"100px"}}} -->
        <div class="wp-block-button">
            <a class="wp-block-button__link has-tertiary-background-color has-background">
                Read more...
            </a>
        </div>
        <!-- /wp:button -->
    </div>
    <!-- /wp:buttons -->
</div>
<!-- /wp:group -->
```

## Automatic Loading

Patterns are automatically loaded from their directory, regardless of their depth in the folder structure. Simply:

1. Create the pattern file in the desired folder
2. Ensure the header contains required metadata
3. The pattern will be available in the Gutenberg editor

## Best Practices

- Use descriptive folder names
- Maintain consistent structure
- Document your patterns with comments
- Use explicit slugs in metadata
- Group similar patterns in the same directory 