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

By default, Pollora will automatically register menus defined in your theme configuration:

```php
// config/theme.php
return [
    'menus' => [
        'primary_navigation' => 'Primary Navigation',
        'footer_navigation' => 'Footer Links'
    ]
];
```

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
