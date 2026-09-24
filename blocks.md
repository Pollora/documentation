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
php artisan pollora:make:block hero-banner --theme
```

This creates all the files in `resources/views/blocks/hero-banner/` and bootstraps the Vite infrastructure on first use (vite.config.js patching, npm dependencies). No service provider is written: Pollora registers the blocks of every theme, plugin and module itself (see [Registration](#registration)).

### 2. Build

```bash
cd themes/your-theme
npm install
npm run build   # or npm run dev for HMR
```

### 3. Done

The block appears in the Gutenberg inserter. No manual `register_block_type()` needed.

## Block Structure

Each block lives in its own directory under `resources/views/blocks/`, next to the other Blade views:

```
resources/views/blocks/hero-banner/
├── block.json         # WordPress block metadata
├── render.blade.php   # Server-side render (default)
├── index.jsx          # Entry point — registers the block
├── edit.jsx           # Editor component — shows what the page will show
├── save.jsx           # Frontend save (static blocks only, see --static)
├── editor.css         # Editor-only styles
├── style.css          # Shared styles (editor + frontend)
└── view.js            # Frontend-only script (optional)
```

Blocks are **dynamic by default**: `render.blade.php` renders them on each request, so their markup is not stored in `post_content`. Changing the markup updates every existing block instead of triggering the editor's "This block contains unexpected or invalid content" error.

`view.js` keeps its WordPress meaning — the frontend script — and the server template is `render.blade.php`.

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
    "viewScript": "file:./view.js",
    "render": "file:./render.blade.php"
}
```

### index.jsx

Entry point that registers the block with WordPress:

```jsx
import { registerBlockType } from '@wordpress/blocks';
import Edit from './edit';
import metadata from './block.json';
import './editor.css';
import './style.css';

registerBlockType(metadata.name, {
    edit: Edit,
    save: () => null, // Rendered server-side by render.blade.php
});
```

A static block (`--static`) imports `save` from `./save` and passes it instead.

### edit.jsx

The editor shows what the page will show. For a dynamic block, the generated `edit.jsx` renders `render.blade.php` through `ServerSideRender`, for the block's current attributes:

```jsx
import { useBlockProps } from '@wordpress/block-editor';
import ServerSideRender from '@wordpress/server-side-render';
import metadata from './block.json';

export default function Edit({ attributes }) {
    const blockProps = useBlockProps();

    return (
        <div {...blockProps}>
            <ServerSideRender block={metadata.name} attributes={attributes} />
        </div>
    );
}
```

A static block's `edit.jsx` mirrors the markup `save.jsx` writes. A block made with `--inner-blocks` keeps the `InnerBlocks` editor, which a server render cannot provide.

## Registration

Pollora registers the blocks of every active theme, plugin and module by itself: whatever holds a `resources/views/blocks` directory (or the former `resources/blocks`) has its blocks registered on WordPress `init`. There is nothing to write — no service provider, no `register_block_type()` call.

A service provider of your own could not do this reliably. Over HTTP, WordPress is loaded, and `init` has fired, before theme and plugin providers boot; a REST request — which the editor and `ServerSideRender` rely on — is answered before they boot at all. A block registered from a provider existed in WP-CLI only.

A `BlocksServiceProvider` written by an earlier `pollora:make:block` is harmless — a block WordPress already holds is skipped — and can be deleted.

Each block's assets resolve through the asset container of the module that ships it:

| Module type | Container name |
|---|---|
| Theme | `theme` |
| Plugin | `plugin.{slug}` |
| Module | `module.{slug}` |

A Laravel module declares its own container in a provider that boots too late for `init`, so Pollora creates it when it is missing, building from `public/build/module/{slug}`. The `BlockRegistrar` automatically creates a `{container}.blocks` child container with an empty `basePath` to resolve block assets directly against the Vite manifest.

Asset entry points are resolved relative to the Vite project root — `resources/views/blocks/hero-banner/index.jsx` — which is both the manifest key and the dev server path. The root is the closest parent directory holding a `vite.config.{js,ts,mjs}`, or else the directory containing `resources/`: keep the Vite config at the root of the theme, plugin or module.

## Vite Configuration

Block entry points must be registered in `vite.config.js`. The recommended pattern auto-discovers all block assets:

```js
import { wordpressPlugin } from '@roots/vite-plugin';
import { globSync } from 'glob';

const blockEntries = globSync([
    './resources/views/blocks/*/{index,view}.{js,jsx,ts,tsx}',
    './resources/views/blocks/*/{editor,style}.css',
])
    .reduce((acc, file) => {
        acc[file.replace(/^\.\//, '').replace(/\.\w+$/, '')] = file;
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

### Full reloads

Blocks live under `resources/views`, so a `refresh` glob such as `resources/views/**` — including laravel-vite-plugin's default `refreshPaths` — would reload the whole page on every block JSX change instead of hot-replacing it. Reload on Blade templates only:

```js
refresh: [
    ...refreshPaths.filter((refreshPath) => refreshPath !== 'resources/views/**'),
    'resources/views/**/*.blade.php',
],
```

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

## `pollora:make:block` Command

### Usage

```bash
php artisan pollora:make:block <name> [options]
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
| `--static` | Create a static block saved in `post_content` (`save.jsx`, no `render.blade.php`) |
| `--dynamic` | Deprecated: blocks are dynamic by default |
| `--inner-blocks` | Add InnerBlocks support |
| `--no-view-script` | Skip frontend view script |
| `--force` | Overwrite existing block |

### Examples

```bash
# Block rendered with Blade in the active theme
php artisan pollora:make:block testimonial --theme --title="Testimonial"

# Static block saved in post content
php artisan pollora:make:block hero-banner --theme --static

# Block with InnerBlocks in a plugin
php artisan pollora:make:block accordion --plugin=my-plugin --inner-blocks

# Custom namespace and category
php artisan pollora:make:block pricing-table --theme --namespace=starter --category=design
```

### First-Run Bootstrap

When creating the first block in a theme or plugin, the command automatically:

1. Patches `vite.config.js` with block entry discovery, `wordpressPlugin()` and Blade-only full reloads
2. Adds required npm dependencies (`@roots/vite-plugin`, `@wordpress/blocks`, etc.)

A block needs a Vite build: the command refuses a theme or plugin that has no `package.json` or no `vite.config.js`, and writes nothing. A plugin made with `pollora:make:plugin --asset` has both.

In a theme or plugin whose blocks are still in `resources/blocks`, it skips the bootstrap and updates the `vite.config.js` block entries to build both locations.

## Rendering with Blade

`render.blade.php` receives the block's `$attributes` (array), `$content` (inner blocks HTML) and `$block` (`WP_Block`):

```blade
<section {!! get_block_wrapper_attributes(['class' => 'py-16']) !!}>
    <h2 class="text-3xl font-bold">{{ $attributes['heading'] ?? '' }}</h2>

    <x-button :href="esc_url_raw($attributes['buttonUrl'] ?? '#')">
        {{ $attributes['buttonText'] ?? __('Learn more', 'my-theme') }}
    </x-button>

    {!! $content !!}
</section>
```

- `{{ }}` escapes; use `{!! !!}` only for `get_block_wrapper_attributes()` and `$content`, which WordPress already escaped.
- For URLs, filter the protocol with `esc_url_raw()` and let `{{ }}` escape: `href="{{ esc_url_raw($url) }}"`. `esc_url()` already HTML-encodes, so inside `{{ }}` a `&` would come out as `&amp;#038;`.
- Blade components work as in any view. The block's `$attributes` array is restored after each `<x-…>` tag, even though components use their own `$attributes` bag.
- Tailwind classes used in the template are picked up as long as your CSS scans `resources/views`.

A render file must stay inside the block directory: if `block.json` points outside it, or to a missing file, the block renders nothing and a warning is logged.

A `render.php` file still works and is included as plain PHP.

## Migrating from `resources/blocks`

Blocks used to live in `resources/blocks`. That directory is still registered, with a deprecation notice in the log, and support ends in Pollora v15.

1. Move the blocks: `git mv resources/blocks resources/views/blocks`
2. Delete `app/Providers/BlocksServiceProvider.php` if you have one: Pollora registers both locations itself, and a block present in both is taken from `resources/views/blocks`.
3. In `vite.config.js`, glob `./resources/views/blocks/*/…` and restrict full reloads to Blade files (see [Full reloads](#full-reloads)). Running `pollora:make:block` does this for you.
4. Optionally convert a static block to Blade: add `"render": "file:./render.blade.php"` to `block.json`, move the markup of `save.jsx` into `render.blade.php`, and set `save: () => null`. Existing content keeps its saved markup until the post is edited, then renders from Blade.

## npm Dependencies

Block development requires these packages (added automatically by `pollora:make:block`):

```json
{
    "devDependencies": {
        "@roots/vite-plugin": "^2.0.0",
        "glob": "^11.0.0",
        "@wordpress/blocks": "^14.0.0",
        "@wordpress/block-editor": "^14.0.0",
        "@wordpress/components": "^29.0.0",
        "@wordpress/element": "^6.0.0",
        "@wordpress/i18n": "^5.0.0",
        "@wordpress/server-side-render": "^5.0.0"
    }
}
```

## Stubs

Block stubs can be published and customized:

```bash
php artisan vendor:publish --tag=pollora-block-stubs
```

This copies the stubs to `stubs/pollora-block/` where you can modify them for your project conventions.