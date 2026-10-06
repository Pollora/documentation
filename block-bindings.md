# Block Bindings

A Block Binding fills an attribute of a block — the text of a paragraph, the URL of a button, the image of an image block — with a value computed on the server, instead of what the editor typed. An event page, a product sheet or a team member card can then be built from core blocks, with no custom block to write.

Pollora lets you declare a source as a PHP class, ships sources that read [typed meta](meta.md) formatted by their type, and makes your own Blade blocks bindable with one line of `block.json`.

> **Experimental.** The API may still change before it is declared stable. The editor shows a bound block with the name of its source; choosing a field from the editor and previewing its value are not available yet.

## Declaring a source

A source is a class marked `#[BlockBinding]`. Each public method marked `#[BindingField]` is a field, chosen in the block with the `field` argument:

```php
use App\Cms\PostTypes\Event;
use App\Services\BookingRepository;
use Pollora\Attributes\BlockBinding;
use Pollora\Attributes\BlockBinding\BindingField;
use Pollora\BlockBinding\Domain\Models\BindingContext;

#[BlockBinding('acme/event', label: 'Event', postTypes: 'event')]
final class EventBinding
{
    #[BindingField(label: 'Remaining seats')]
    public function remainingSeats(BindingContext $context, BookingRepository $bookings): string
    {
        $event = $context->meta(Event::class);
        $left = max(0, $event->capacity - $bookings->countFor($context->postId));

        return trans_choice('events.seats', $left, ['count' => $left]);
    }

    #[BindingField(label: 'Booking link', type: 'url')]
    public function bookingUrl(BindingContext $context): string
    {
        return route('events.book', ['event' => $context->postId]);
    }
}
```

The class is discovered in the application, themes, plugins and modules, and resolved by the container: the constructor and each field receive their dependencies, like `BookingRepository` above. `php artisan pollora:make:binding EventBinding` generates one (`--theme`, `--plugin`, `--module` to choose where).

A block binds an attribute to a field in its markup:

```html
<!-- wp:paragraph {"metadata":{"bindings":{"content":{"source":"acme/event","args":{"field":"remaining_seats"}}}}} -->
<p>Seats available</p>
<!-- /wp:paragraph -->

<!-- wp:buttons --><div class="wp-block-buttons">
<!-- wp:button {"metadata":{"bindings":{"url":{"source":"acme/event","args":{"field":"booking_url"}}}}} -->
<div class="wp-block-button"><a class="wp-block-button__link wp-element-button">Book</a></div>
<!-- /wp:button -->
</div><!-- /wp:buttons -->
```

The text saved in the block is a fallback: it shows when the field returns `null`.

### Parameters

| Attribute | Parameter | Default | Effect |
|---|---|---|---|
| `#[BlockBinding]` | `name` | required | Source name, `namespace/name` in lowercase, as WordPress requires |
| `#[BlockBinding]` | `label` | class name | Label shown in the editor |
| `#[BlockBinding]` | `usesContext` | `['postId', 'postType']` | Block context the source reads |
| `#[BlockBinding]` | `postTypes` | all | The post types the source answers for; on any other post, the block keeps its content |
| `#[BindingField]` | `name` | method name in snake_case | Value of the `field` argument: `remainingSeats` is `remaining_seats` |
| `#[BindingField]` | `label` | method name, headlined | Label of the field |
| `#[BindingField]` | `type` | `text` | `text`, `url` or `image`: a `url` or `image` value is sanitized as a URL |

A class with no field defines `__invoke(BindingContext $context)` and receives every call, whatever its arguments.

### `BindingContext`

| Member | Content |
|---|---|
| `postId`, `postType` | The post of the block context, so the right one inside a query loop |
| `termId`, `taxonomy` | The term of the block context (a terms query) |
| `arg('format', $default)` | An argument of the binding, besides `field` |
| `attribute` | The bound attribute: `content`, `url`, `alt`… |
| `block` | The `WP_Block`, for advanced cases |
| `post()` | The post, as a `Pollora\Models\Post` (the model bound to its post type, when there is one) |
| `meta(Event::class)` | The [typed meta](meta.md) of a class, on the object that carries them: the post, the term, or the post's author for `#[UserMeta]` |

### What a field returns

A field declares its return type: `string`, `int`, `float`, `bool`, a `Stringable` or `null`. A class whose field has no return type, or returns an array, is reported at discovery, like a name WordPress would refuse or a field declared twice.

Pollora escapes the value for the place it lands in, so a source cannot forget to:

| Value | Written as |
|---|---|
| Text in a paragraph, a heading, a list item | Escaped: `<for>` shows as text |
| A `url` or `image` field | Sanitized as a URL |
| An `HtmlString` | Filtered like post content (`wp_kses_post()`) |
| A boolean in a paragraph | "Yes" or "No", translated |
| An attribute of a Blade block | As returned: the template escapes it, like any attribute |

`null` keeps what the block holds. A field that throws is logged and treated as `null`, so a failing source never takes a page down; with `APP_DEBUG` on, the exception is thrown.

## Bindable Blade blocks

A block rendered on the server becomes bindable by listing its attributes under `pollora.bindings` in its `block.json`:

```json
{
    "name": "acme/event-card",
    "attributes": {
        "title": { "type": "string" },
        "ctaUrl": { "type": "string" }
    },
    "pollora": { "bindings": ["title", "ctaUrl"] },
    "render": "file:./render.blade.php"
}
```

WordPress replaces the bound attributes before the block renders, so the view does not change:

```blade
<div {!! get_block_wrapper_attributes() !!}>
    <h3>{{ $attributes['title'] }}</h3>
    <a href="{{ $attributes['ctaUrl'] }}">{{ __('Book', 'acme') }}</a>
</div>
```

Only a block with a `render` can be bound: WordPress cannot rewrite the saved HTML of a block that is not one of its own. A block listing bindings without `render`, or an attribute it does not declare, is reported in the log.

## Typed meta in core blocks

Pollora ships four sources. Those reading meta format the value by its declared type, which `core/post-meta` does not: a date reads as a date, not `2026-11-14T09:00:00+00:00`.

| Source | Reads | Arguments |
|---|---|---|
| `pollora/post-meta` | A typed meta of the post of the block | `key`, `format`, `decimals`, `true`, `false`, `size`, `fallback` |
| `pollora/term-meta` | A typed meta of the term of the block, or of the queried term on a term archive | same |
| `pollora/author-meta` | A typed meta of the author of the post | same |
| `pollora/option` | A site option | `name`, `fallback` |

```html
<!-- wp:heading {"metadata":{"bindings":{"content":{"source":"pollora/post-meta","args":{"key":"starts_at","format":"l j F Y"}}}}} -->
<h2 class="wp-block-heading">Date to come</h2>
<!-- /wp:heading -->
```

The title of a post, its link or date, and the name of a term are served by WordPress's own `core/post-data` and `core/term-data`.

### Formatting

| Meta type | Shown as | Argument |
|---|---|---|
| `string` | As stored, escaped | none |
| `int`, `float` | A number with the site's separators | `decimals` |
| `bool` | "Yes" or "No", translated | `true`, `false` for other words |
| Date | The site's date format, in its language | `format`, a PHP date format |
| Enum | Its `label()` when the enum defines one, else its value | none |
| Array | Its items, as a list: "rock, jazz, and blues" | as for one item |
| Attachment ID | Depends on the bound attribute ([see below](#attachments)) | `size` |

`format: raw` gives the value as stored, and `fallback` a text for an empty meta. An object meta is not shown.

### Attachments

A meta marked `#[Meta(media: true)]` holds an attachment ID. Bound to an image, it gives what each attribute needs:

```php
#[Meta(showInRest: true, media: true)]
public ?int $coverImageId = null;
```

```html
<!-- wp:image {"metadata":{"bindings":{
    "url":{"source":"pollora/post-meta","args":{"key":"cover_image_id","size":"large"}},
    "alt":{"source":"pollora/post-meta","args":{"key":"cover_image_id"}}}}} -->
<figure class="wp-block-image"><img alt=""/></figure>
<!-- /wp:image -->
```

| Attribute | Value |
|---|---|
| `url` (and any other) | The image URL at `size` (`full` by default) |
| `alt` | The alternative text |
| `title` | The attachment title |
| `caption` | The caption |
| `id` | The ID |

## Security

Anyone who can edit a post can bind any source to it, so each source only shows what may be public, and the checks live in one place that no source can skip:

- **The post or term must be visible.** Nothing is shown of a post the visitor cannot read (unpublished without `read_post`, waiting for its password) or of a term of a taxonomy that is not publicly queryable — the checks of `core/post-meta` and `core/term-data`.
- **Post and term meta**: only a meta declared with `showInRest: true`, under a key that does not start with `_`, as `core/post-meta` does.
- **User meta**: `showInRest` is not enough, a user meta may be personal. `pollora/author-meta` only reads a meta marked `#[Meta(public: true)]`.
- **Options**: `pollora/option` only reads the options listed in `config/block-bindings.php`, none by default:

```php
// config/block-bindings.php
return [
    'options' => ['blogdescription'],
];
```

- **No source reads the logged-in visitor**: the value would leak through the page cache.

## Performance

A field is called once per request for the same post, attribute and arguments, and a source class is only made the first time one of its fields is used. Typed meta are read through WordPress's meta cache, already primed for the posts of a query loop. In a loop of 20 posts, four bound blocks make 80 calls: a field should not run an uncached query.
