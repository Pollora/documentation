# Gutenberg Blocks

Pollora provides a complete system for building custom Gutenberg blocks with Vite and JSX. Blocks can live in themes, plugins, or Laravel modules — the registration works identically across all three.

## How It Works

The `BlockRegistrar` service scans a directory for subdirectories containing `block.json`, then:

1. Creates a dedicated `{parent}.blocks` asset container with no `basePath` for direct Vite manifest resolution
2. Pre-registers each `file:./` asset (scripts and styles) via `wp_register_script` / `wp_register_style` with the Vite-compiled URLs
3. Adds `type="module"` and `crossorigin` attributes for Vite scripts
4. Calls `register_block_type()` — WordPress finds the pre-registered handles and skips its own resolution

## Quick Start

### 1. Scaffold a Block

```bash
php artisan pollora:make-block hero-banner --theme
```

This creates all the files in `resources/blocks/hero-banner/` and bootstraps the Vite infrastructure on first use (vite.config.js patching, npm dependencies, `BlocksServiceProvider`).

### 2. Build

```bash
cd themes/your-theme
npm install
npm run build   # or npm run dev for HMR
```

### 3. Done

The block appears in the Gutenberg inserter. No manual `register_block_type()` needed.

## Block Structure

Each block lives in its own directory under `resources/blocks/`:

```
resources/blocks/hero-banner/
├── block.json       # WordPress block metadata
├── index.jsx        # Entry point — registers the block
├── edit.jsx         # Editor component
├── save.jsx         # Frontend save (static) or omitted (dynamic)
├── render.php       # Server-side render (dynamic blocks only)
├── editor.css       # Editor-only styles
├── style.css        # Shared styles (editor + frontend)
└── view.js          # Frontend-only script (optional)
```

### block.json

Standard WordPress [block metadata](https://developer.wordpress.org/block-editor/reference-guides/block-api/block-metadata/). Asset fields use `file:./` references — Pollora resolves them through Vite:

```json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "my-theme/hero-banner",
    "title": "Hero Banner",
    "category": "design",
    "icon": "cover-image",
    "textdomain": "my-theme",
    "editorScript": "file:./index.jsx",
    "editorStyle": "file:./editor.css",
    "style": "file:./style.css",
    "viewScript": "file:./view.js"
}
```

### index.jsx

Entry point that registers the block with WordPress:

```jsx
import { registerBlockType } from '@wordpress/blocks';
import Edit from './edit';
import save from './save';
import metadata from './block.json';
import './editor.css';
import './style.css';

registerBlockType(metadata.name, {
    edit: Edit,
    save,
});
```

For dynamic blocks (server-rendered), replace `save` with `() => null`:

```jsx
registerBlockType(metadata.name, {
    edit: Edit,
    save: () => null,
});
```

## BlocksServiceProvider

Each theme, plugin, or module that contains blocks needs a `BlocksServiceProvider` to register them. The `pollora:make-block` command creates this automatically on first use.

```php
<?php

namespace Theme\MyTheme\Providers;

use Illuminate\Support\ServiceProvider;
use Pollora\Block\Infrastructure\Services\BlockRegistrar;

class BlocksServiceProvider extends ServiceProvider
{
    public function boot(BlockRegistrar $registrar): void
    {
        $registrar->registerDirectory(
            directory: dirname(__DIR__, 2) . '/resources/blocks',
            containerName: 'theme',
        );
    }
}
```

The `containerName` matches the asset container for your module type:

| Module type | Container name |
|---|---|
| Theme | `theme` |
| Plugin | `plugin.{slug}` |
| Module | `module.{slug}` |

The `BlockRegistrar` automatically creates a `{container}.blocks` child container with an empty `basePath` to resolve block assets directly against the Vite manifest.

## Vite Configuration

Block entry points must be registered in `vite.config.js`. The recommended pattern auto-discovers all block assets:

```js
import { wordpressPlugin } from '@roots/vite-plugin';
import { globSync } from 'glob';

const blockEntries = globSync('./resources/blocks/*/{index,view}.{js,jsx,ts,tsx}')
    .concat(globSync('./resources/blocks/*/{editor,style}.css'))
    .reduce((acc, file) => {
        const slug = path.basename(path.dirname(file));
        const name = path.basename(file, path.extname(file));
        acc[`blocks/${slug}/${name}`] = file;
        return acc;
    }, {});
const hasBlocks = Object.keys(blockEntries).length > 0;
```

Then in the Laravel Vite config:

```js
input: ["./resources/assets/app.js", ...Object.values(blockEntries)],
```

And in `plugins`:

```js
plugins: [
    laravel(getThemeConfig()),
    ...(hasBlocks ? [wordpressPlugin()] : []),
],
```

The `@roots/vite-plugin` provides the `wordpressPlugin()` which generates `editor.deps.json` with WordPress script dependencies.

## Tailwind CSS in Blocks

Pollora uses **Tailwind CSS v4** with automatic source detection. No `tailwind.config.js` is needed. Tailwind utilities work in blocks via two approaches:

### Utility Classes in JSX

Use Tailwind classes directly in your block JSX — they work on both frontend and editor:

```jsx
// edit.jsx
export default function Edit() {
    const blockProps = useBlockProps();
    return (
        <div {...blockProps}>
            <h2 className="text-3xl font-bold text-white mb-4">
                {__('Hello World', 'my-theme')}
            </h2>
            <p className="text-lg text-indigo-100/80 max-w-2xl mx-auto">
                {__('A description', 'my-theme')}
            </p>
        </div>
    );
}
```

### `@apply` in Block CSS

Use `@apply` in `style.css` and `editor.css` for component-level styles:

```css
/* style.css — loaded on frontend AND in editor */
@import "tailwindcss" source(".");

.wp-block-my-theme-hero {
    @apply relative py-24 px-8 rounded-xl overflow-hidden;
    background: linear-gradient(135deg,
        theme(--color-indigo-950) 0%,
        theme(--color-violet-900) 100%
    );
}
```

```css
/* editor.css — loaded only in editor */
@reference "tailwindcss";

.wp-block-my-theme-hero {
    @apply border-2 border-dashed border-black/15 min-h-[300px];
}
```

### Key Directives

| Directive | Use in | Purpose |
|---|---|---|
| `@import "tailwindcss" source(".")` | `style.css` | Full Tailwind import, scoped to the block's directory. Generates utility classes from JSX files. |
| `@reference "tailwindcss"` | `editor.css` | Access to `@apply` and `theme()` without generating utilities. |
| `theme(--color-*)` | Any CSS | Access Tailwind theme values as CSS functions. |

### Why `source(".")`?

The `source(".")` parameter tells Tailwind to scan **only the block's directory** for utility classes, not the entire project. This keeps the generated CSS small while ensuring all classes used in `edit.jsx`, `save.jsx`, and `index.jsx` are available in the editor iframe.

Without `source(".")`, the block's CSS would include utilities from the entire project — much larger than necessary.

## `pollora:make-block` Command

### Usage

```bash
php artisan pollora:make-block <name> [options]
```

### Arguments

| Argument | Description |
|---|---|
| `name` | Block slug in kebab-case (e.g. `hero-banner`) |

### Options

| Option | Description |
|---|---|
| `--theme[=NAME]` | Create in theme (default: active theme) |
| `--plugin=NAME` | Create in plugin |
| `--namespace=NS` | Block namespace (before the `/`) |
| `--title=TITLE` | Block title in the inserter |
| `--category=CAT` | Gutenberg category (default: `widgets`) |
| `--icon=ICON` | Dashicon name (default: `block-default`) |
| `--dynamic` | Create a dynamic block with `render.php` |
| `--inner-blocks` | Add InnerBlocks support |
| `--no-view-script` | Skip frontend view script |
| `--force` | Overwrite existing block |

### Examples

```bash
# Simple static block in the active theme
php artisan pollora:make-block hero-banner --theme

# Dynamic block with server-side rendering
php artisan pollora:make-block testimonial --theme --dynamic --title="Testimonial"

# Block with InnerBlocks in a plugin
php artisan pollora:make-block accordion --plugin=my-plugin --inner-blocks

# Custom namespace and category
php artisan pollora:make-block pricing-table --theme --namespace=starter --category=design
```

### First-Run Bootstrap

When creating the first block in a theme or plugin, the command automatically:

1. Creates `app/Providers/BlocksServiceProvider.php`
2. Patches `vite.config.js` with block entry discovery and `wordpressPlugin()`
3. Adds required npm dependencies (`@roots/vite-plugin`, `@wordpress/blocks`, etc.)

## Dynamic Blocks

Dynamic blocks render on the server using `render.php`. Add `"render": "file:./render.php"` to `block.json`:

```json
{
    "name": "my-theme/testimonial",
    "render": "file:./render.php",
    "editorScript": "file:./index.jsx"
}
```

The render template receives `$attributes`, `$content`, and `$block`:

```php
<?php
$heading = $attributes['heading'] ?? '';
?>
<div <?php echo get_block_wrapper_attributes(); ?>>
    <h2><?php echo esc_html($heading); ?></h2>
</div>
```

In `index.jsx`, set `save: () => null` since rendering is handled server-side.

## npm Dependencies

Block development requires these packages (added automatically by `pollora:make-block`):

```json
{
    "devDependencies": {
        "@roots/vite-plugin": "^2.0.0",
        "glob": "^11.0.0",
        "@wordpress/blocks": "^14.0.0",
        "@wordpress/block-editor": "^14.0.0",
        "@wordpress/components": "^29.0.0",
        "@wordpress/element": "^6.0.0",
        "@wordpress/i18n": "^5.0.0"
    }
}
```

## Stubs

Block stubs can be published and customized:

```bash
php artisan vendor:publish --tag=pollora-block-stubs
```

This copies the stubs to `stubs/pollora-block/` where you can modify them for your project conventions.