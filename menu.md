# Menu Manager

Pollora let you enhance your WordPress menus with an elegant API for managing classes and attributes. This library provides granular control over each menu element, perfectly suited for modern frameworks like TailwindCSS and JavaScript libraries like AlpineJS.

## Features

- Rule-based configuration with depth and position targeting
- Flexible attributes and classes management
- Built-in translation support for menu labels

## Usage Guide

### Configuration Structure

The library uses a rule-based approach to configure three types of elements:

- Links (`link_config`)
- List items (`item_config`)
- Submenus (`submenu_config`)

Each rule can include:
- `depth`: Menu depth level (required)
- `eq`: Specific item index (optional)
- `class`: CSS classes to apply
- `attrs`: Additional HTML attributes

### Basic Usage

```php
wp_nav_menu([
    'theme_location' => 'main_menu',
    'link_config' => [
        [
            'depth' => 0,
            'class' => [
                'text-gray-700',
                'hover:text-blue-500',
                'sm:text-lg'
            ]
        ],
        [
            'depth' => 1,
            'class' => 'text-gray-600'
        ]
    ]
]);
```

### Advanced Examples

#### Targeting Specific Items

```php
wp_nav_menu([
    'theme_location' => 'main_menu',
    'link_config' => [
        [
            'depth' => 0,
            'class' => 'text-xl',
            'attrs' => ['data-tracking' => 'primary-link']
        ],
        [
            'depth' => 0,
            'eq' => 2, // Target the third item specifically
            'class' => [
                'text-blue-500',
                'hover:text-blue-700'
            ]
        ]
    ]
]);
```

#### Interactive Menu with AlpineJS

```php
wp_nav_menu([
    'container' => 'nav',
    'item_config' => [
        [
            'depth' => 0,
            'class' => 'relative',
            'attrs' => ['x-data' => '{open: false}']
        ]
    ],
    'link_config' => [
        [
            'depth' => 0,
            'class' => [
                'flex items-center justify-between',
                'hover:text-blue-500'
            ],
            'attrs' => ['@click' => 'open = !open']
        ]
    ],
    'submenu_config' => [
        [
            'depth' => 0,
            'class' => [
                'pl-4 mt-2',
                'transition-all'
            ],
            'attrs' => [
                'x-show' => 'open',
                'x-transition' => ''
            ]
        ]
    ]
]);
```

### Menu Registration

Pollora registers the menu locations declared in your theme's `config/menus.php`. The file returns a map of location identifier to label, with no wrapping key:

```php
// themes/my-theme/config/menus.php
return [
    'main_menu' => __('Main Menu', 'my-theme'),
    'footer_navigation' => __('Footer Links', 'my-theme'),
];
```

Pollora hands the map to WordPress's `register_nav_menus()` on `after_setup_theme` (priority 20). The identifier is what you pass as `theme_location` to `wp_nav_menu()`. A `config/theme.php` file is not a menu configuration: nothing reads a `menus` key from it.

#### Translating menu labels

The labels shown in the admin are translated in two ways, and both can be used:

- Call `__()` in the file, as above. This is what the default theme does. `menus.php` is one of the theme config files Pollora loads late enough for a text domain to be available, see [Translating theme config](translations.md#translating-theme-config).
- Write plain strings and translate them from a Laravel `menus` group file, `lang/{locale}/menus.php` in your project. Before registering the locations, Pollora looks each label up as `menus.{label}` and keeps the label unchanged when no entry matches:

```php
// config/menus.php
return [
    'main_menu' => 'Main Menu',
];

// lang/{locale}/menus.php
return [
    'Main Menu' => 'Menu principal',
];
```

Both happen when the locations are registered, on `after_setup_theme`. Locations are not registered earlier, so `has_nav_menu()` and `get_registered_nav_menus()` return nothing at the top of `functions.php`.

### Widget Support

Apply styles to menu widgets using the `widget_nav_menu_args` filter:

```php
add_filter('widget_nav_menu_args', function($args, $nav_menu, $widget_args) {
    if ($widget_args['id'] === 'footer-menu') {
        $args['link_config'] = [
            [
                'depth' => 0,
                'class' => [
                    'text-sm',
                    'text-gray-500',
                    'hover:text-gray-700'
                ]
            ]
        ];
    }
    return $args;
}, 10, 3);
```

## Troubleshooting

### Common Issues

- Verify that `depth` is set correctly for each rule
- Check that `eq` values match your menu structure (0-based indexing)
- Validate attribute array syntax
- Ensure your theme configuration is properly loaded

### Debug Tips

1. Check the rendered HTML to see if classes are being applied
2. Use browser dev tools to inspect the menu structure
3. Enable WordPress debug mode to catch any potential errors
4. Verify menu registration in WordPress admin panel
