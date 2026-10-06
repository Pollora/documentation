# Typed Meta

In WordPress, a meta is a string key and a string value: its key is repeated wherever it is read or written, and every read casts it again by hand. Pollora lets you declare a meta once, as a typed PHP property, and derives the rest: `register_meta()`, sanitization, the REST schema, and reads and writes with the PHP type.

> **Experimental.** The API may still change before it is declared stable. Projects that do not use `#[Meta]` see no change.

## Declaring meta

A meta is a public typed property marked `#[Meta]`, on the class that already carries `#[PostType]` or `#[Taxonomy]` — or on a class of its own for [users, comments, and post types or taxonomies the project does not declare](#other-objects). The property type gives the meta type, its initial value the default, and its name the key, in snake_case:

```php
use App\Enums\EventStatus;
use Carbon\CarbonImmutable;
use Pollora\Attributes\Meta;
use Pollora\Attributes\PostType;
use Pollora\Attributes\PostType\ShowInRest;
use Pollora\Attributes\PostType\Supports;

#[PostType('event')]
#[ShowInRest]
#[Supports(['title', 'editor'])]
class Event
{
    #[Meta(showInRest: true, label: 'Start')]
    public ?CarbonImmutable $startsAt = null;     // key: starts_at

    #[Meta(showInRest: true)]
    public int $capacity = 0;

    #[Meta(showInRest: true)]
    public bool $soldOut = false;

    #[Meta(showInRest: true)]
    public EventStatus $status = EventStatus::Draft;   // a backed enum

    #[Meta(key: '_event_internal_ref', capability: 'manage_options')]
    public ?string $internalRef = null;
}
```

Taxonomies work the same way: the meta are registered on the taxonomy's terms.

```php
use Pollora\Attributes\Meta;
use Pollora\Attributes\Taxonomy;

#[Taxonomy('genre')]
class Genre
{
    #[Meta(showInRest: true)]
    public ?string $color = null;
}
```

Each property needs a default value or a nullable type: it is what an absent meta reads as. The meta are registered on `init`, after post types and taxonomies, and declaring a class is all it takes — run `php artisan discovery:clear` after adding one in development.

### Other objects

Meta of objects the project does not declare go on a class of their own, marked with the object they belong to:

```php
use Pollora\Attributes\CommentMeta;
use Pollora\Attributes\Meta;
use Pollora\Attributes\PostMeta;
use Pollora\Attributes\TermMeta;
use Pollora\Attributes\UserMeta;

#[PostMeta('product')]                 // a plugin's post type (WooCommerce)
class ProductExtras
{
    #[Meta(showInRest: true)]
    public ?string $warrantyNotice = null;
}

#[PostMeta(['post', 'page'])]          // several post types share the schema
class ArticleExtras
{
    #[Meta(showInRest: true, revisions: true)]
    public ?string $subtitle = null;
}

#[TermMeta('category')]
class CategoryExtras
{
    #[Meta(showInRest: true)]
    public ?string $color = null;
}

#[UserMeta]
class MemberProfile
{
    #[Meta(showInRest: true)]
    public bool $newsletterOptIn = false;
}

#[CommentMeta]
class ReviewMeta
{
    #[Meta]
    public int $rating = 5;
}
```

| Attribute | Meta of | Parameter |
|---|---|---|
| `#[PostMeta]` | posts of the given post types | one slug or a list |
| `#[TermMeta]` | terms of the given taxonomies | one slug or a list |
| `#[UserMeta]` | users | — |
| `#[CommentMeta]` | comments, of every type | — |

Several classes may declare meta for the same objects — one `#[UserMeta]` per module, for instance — as long as their keys differ. Comment meta apply to every comment type, WooCommerce reviews included: WordPress has no per-type registration for comments.

### `#[Meta]` parameters

| Parameter | Default | Effect |
|---|---|---|
| `key` | property name in snake_case | Key stored in the database: `startsAt` is stored as `starts_at` |
| `showInRest` | `false` | Exposes the meta in the REST API, with a schema derived from its type |
| `label` | none | Label shown by the editor |
| `description` | none | Description passed to `register_meta()` |
| `sanitize` | derived from the type | A callable (`'wp_kses_post'`, `[MyClass::class, 'method']`) replacing the sanitization |
| `capability` | the right to edit the post or term | Capability required to write the meta through REST and the editor |
| `revisions` | `false` | Versions the meta with post revisions (post types only; the post type needs `revisions` support) |

### Types

| PHP type | Stored as | Sanitized with |
|---|---|---|
| `string` | as is | `sanitize_text_field()` — use `sanitize: 'wp_kses_post'` for HTML |
| `int` | `"250"` | conversion to an integer |
| `float` | `"9.5"` | conversion to a number |
| `bool` | `"1"` or `"0"` | `1`, `true`, `yes`, `on` / `0`, `false`, `no`, `off` |
| `DateTimeInterface`, Carbon | ISO 8601 in UTC: `2026-11-14T09:00:00+00:00` | parsed; a value that is not a date is emptied |
| backed enum | the case value | `tryFrom()`; an unknown value is emptied |

A nullable type (`?int`) allows the meta to be absent. A date property typed with an interface (`DateTimeInterface`, `CarbonInterface`) reads as a `CarbonImmutable`; a concrete class (`Carbon`, `DateTimeImmutable`) reads as that class.

The sanitization applies to every write, wherever it comes from: `Meta::of()`, `update_post_meta()`, the REST API, the block editor. A value it cannot read is emptied, and an empty meta reads as the property's default.

## Reading and writing

`Meta::of()` gives the meta of one object — post, term, user or comment, by its ID — with their PHP types:

```php
use App\Cms\PostTypes\Event;
use Pollora\Support\Facades\Meta;

$event = Meta::of(Event::class, $postId);

$event->capacity;                 // int
$event->soldOut;                  // bool
$event->status;                   // EventStatus
$event->startsAt?->isFuture();    // CarbonImmutable|null
$event->starts_at;                // the same meta, by its key

$event->capacity = 250;
$event->fill(['status' => EventStatus::Published, 'soldOut' => true])->save();

$event->set('startsAt', null)->save();   // null deletes a nullable meta
$event->toArray();                        // every meta, by property name

Meta::of(MemberProfile::class, $userId)->newsletterOptIn;   // bool
Meta::of(ReviewMeta::class, $commentId)->rating;           // int
```

Values are read when first accessed, through `get_metadata_raw()`, so the object cache applies; an absent meta gives the property's default. Writes are checked at once — `$event->capacity = 'many'` throws an `InvalidMetaValueException` before anything is written — and stored by `save()` through `update_metadata()`, so sanitization, meta hooks and cache invalidation apply.

WordPress's functions keep working: `get_post_meta($postId, 'capacity', true)` still returns the stored string.

### On the Eloquent models

The models of `Pollora\Models` — `Post`, `Page`, `Term`, `User`, `Comment` — read the typed meta of their object as attributes. Bind a model to its post type with `$postType`:

```php
use Pollora\Models\Post;

class Event extends Post
{
    protected $postType = 'event';
}
```

```php
$event = Post::find($id);         // an Event: models with a $postType are bound at discovery
$event->capacity;                 // int, or by its key: $event->sold_out
$event->capacity = 250;           // checked at once: 'many' throws InvalidMetaValueException
$event->save();                   // written through update_metadata()

Event::whereMeta('capacity', '>=', 100)->get();
Event::whereMeta('status', EventStatus::Published)->get();
Event::whereMeta('subtitle', null)->get();    // the meta is absent

auth()->user()->newsletterOptIn;  // a #[UserMeta] on the user model
```

Typed meta are not part of `toArray()` or `getAttributes()`: they are not columns.

`whereMeta()` compares stored values — numbers as numbers, dates in UTC — with `=`, `!=`, `<`, `<=`, `>`, `>=`. A model without the meta is not matched, even if its default would be.

A model class known to carry typed meta — a post model whose `$postType` has some, the user and comment models once a `#[UserMeta]` or `#[CommentMeta]` exists — stops eager loading the `meta` relation: a collection loads its meta in one query, through WordPress's cache. Keys no `#[Meta]` declares keep working as before (`$post->some_key` returns the raw value), and a project without typed meta sees no change.

### A value that cannot be read

If the database holds a value that does not match the declared type (`"many"` for an `int`, written before the meta was declared), reading it:

- in debug mode (`APP_DEBUG=true`), throws an `InvalidMetaValueException` naming the meta, so it shows during development;
- in production, returns the property's default and logs a warning, so a damaged value never takes a page down.

## REST API

With `showInRest: true`, the meta appears under `meta` in the REST response of the post, term, user or comment, with its type:

```json
{
  "id": 42,
  "meta": {
    "starts_at": "2026-11-14T09:00:00+00:00",
    "capacity": 250,
    "sold_out": false,
    "status": "published"
  }
}
```

The schema follows the type: dates have the `date-time` format and enums list their values, so a write with an unknown value is refused with a 400 error (`rest_not_in_enum`).

**A post type exposes its meta in REST only if it supports `custom-fields`.** Pollora adds that support to a post type declared with `#[PostType]` as soon as one of its meta has `showInRest: true`. It does not change a post type it does not declare: core posts and pages support it already; for a plugin's post type targeted by `#[PostMeta]`, check that it does.

A key starting with an underscore is protected: WordPress hides it from custom fields and REST. Exposing one requires an explicit `capability`.

## Errors at discovery

A declaration WordPress cannot register is refused at discovery, and the error is logged with the class and the property named; the other meta of the project still register:

- a union type, an untyped property, or an unsupported type (arrays and objects are not supported yet);
- an enum without backing values (`enum Mood { … }` instead of `enum Mood: string { … }`);
- a non-nullable property without a default value;
- a protected key exposed in REST without a `capability`;
- `revisions: true` on anything but posts;
- the same key declared twice, in one class or by two classes for the same objects (a post type in common, users, comments).

## Not available yet

- Arrays and structured objects, validation rules.
