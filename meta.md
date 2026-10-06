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
| `rules` | none | Laravel validation rules, checked on writes from PHP and REST ([see below](#validation)) |
| `items` | the `@var list<…>` docblock | On an `array`, the item type: `'string'`, `'int'`, `'float'`, `'bool'` or a class |
| `single` | `true` | On an `array`, `false` stores one row per item instead of one serialized array |
| `control` | derived from the type | The input a UI driver builds ([see below](#input-fields)) |
| `group` | none | The group of fields a UI driver puts the meta in |
| `hints` | `[]` | Options passed as they are to UI drivers, by driver |
| `media` | `false` | On an `int`, the value is an attachment ID: a [block binding](block-bindings.md#attachments) gives its URL, alternative text or caption |
| `public` | `false` | The value may be shown to anyone: required for a user meta to be read by the [`pollora/author-meta`](block-bindings.md#typed-meta-in-core-blocks) binding |

### Types

| PHP type | Stored as | Sanitized with |
|---|---|---|
| `string` | as is | `sanitize_text_field()` — use `sanitize: 'wp_kses_post'` for HTML |
| `int` | `"250"` | conversion to an integer |
| `float` | `"9.5"` | conversion to a number |
| `bool` | `"1"` or `"0"` | `1`, `true`, `yes`, `on` / `0`, `false`, `no`, `off` |
| `DateTimeInterface`, Carbon | ISO 8601 in UTC: `2026-11-14T09:00:00+00:00` | parsed; a value that is not a date is emptied |
| backed enum | the case value | `tryFrom()`; an unknown value is emptied |
| `array` | one serialized array, or one row per item with `single: false` | each item, by its type ([see below](#arrays)) |
| a class with public typed properties | a serialized array, by property | each property, by its type ([see below](#objects)) |

A nullable type (`?int`) allows the meta to be absent. A date property typed with an interface (`DateTimeInterface`, `CarbonInterface`) reads as a `CarbonImmutable`; a concrete class (`Carbon`, `DateTimeImmutable`) reads as that class.

The sanitization applies to every write, wherever it comes from: `Meta::of()`, `update_post_meta()`, the REST API, the block editor. A value it cannot read is emptied, and an empty meta reads as the property's default.

### Arrays

An `array` property says what it holds, with `items:` or a docblock:

```php
/** @var list<string> */
#[Meta(showInRest: true, single: false)]
public array $speakers = [];          // one row per speaker

#[Meta(showInRest: true, items: 'int')]
public array $roomIds = [];           // one serialized array

#[Meta(items: Schedule::class)]
public array $sessions = [];          // a list of objects
```

- **`single: false`** stores one row per item, as WordPress does for a meta added several times: `get_post_meta($id, 'speakers')` returns the list, and `whereMeta('speakers', 'Ada')` finds the posts where one of the speakers is Ada. Items can be strings, numbers, booleans, dates or enums.
- **Single** (the default) stores the whole array in one row, serialized by WordPress. It cannot be filtered with `whereMeta()`.

A docblock class must be fully qualified (`list<\App\Cms\Schedule>`); otherwise use `items: Schedule::class`. An array of arrays is refused: use a class.

### Objects

A class with public typed properties groups several values in one meta:

```php
final class Schedule
{
    public ?CarbonImmutable $startsAt = null;
    public int $durationMinutes = 60;
    public EventStatus $status = EventStatus::Draft;

    public function __construct(public bool $public = true) {}
}

#[Meta(showInRest: true)]
public Schedule $schedule;
```

It is stored as an array of its properties, by key in snake_case — never as a serialized PHP object — and read back as an instance. An absent meta reads as an instance with the class's defaults; declare `?Schedule $schedule = null` to read it as null instead. Properties can be strings, numbers, booleans, dates and enums, each with a default or nullable; an array or an object inside is refused.

In REST, arrays publish their `items` schema and objects their `properties`: WordPress refuses an item or a property of the wrong type, or an unknown property, with a 400 error. `rules` apply to the whole value: `rules: ['max:3']` limits an array to three items.

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

### Validation

`rules` takes Laravel validation rules:

```php
#[Meta(showInRest: true, label: 'Capacity', rules: ['min:0', 'max:5000'])]
public int $capacity = 0;

#[Meta(rules: ['email'])]
public ?string $contact = null;
```

The type rule is implied — `integer` for an `int`, `numeric` for a `float`, `date` for a date — so `max:5000` compares numbers, not lengths. An enum is checked by its backing value (`rules: ['in:published,archived']`). Null passes on a nullable meta: it deletes the value.

- **From PHP**, `Meta::of()->set()` and a model attribute (`$event->capacity = 6000`) throw a `MetaValidationException` at once, with the message of the rule; nothing is written.
- **Through REST**, a write to a post, term, user or comment that breaks a rule is refused with a 400 error before anything is stored, the message under `data.params["meta.capacity"]`, as WordPress reports an invalid parameter.
- **WordPress's own functions** (`update_post_meta()`) cannot refuse a value: they only apply the sanitization.

Messages come from Laravel's translations; the attribute is named by `label`, or by the key.

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

## Input fields

Pollora does not draw admin fields: field plugins (ACF, Meta Box…) or an editor panel do. To avoid declaring each field twice, `#[Meta]` describes the field in neutral terms, and a **UI driver** — a separate package — turns the declarations into fields.

```php
use Pollora\Meta\Domain\Enums\Control;

#[Meta(label: 'Job title', group: 'Profile')]
public ?string $jobTitle = null;                       // Control::Text, from the type

#[Meta(control: Control::Color, hints: ['acf' => ['wrapper' => ['width' => 50]]])]
public ?string $accent = null;
```

| Type | Control derived |
|---|---|
| `string` | `Text`, or `RichText` with `sanitize: 'wp_kses_post'` |
| `int`, `float` | `Number` |
| `bool` | `Toggle` |
| date | `DateTime` |
| enum | `Select` |
| array, object | none: the driver decides |

The other controls are `Textarea`, `Date`, `Media`, `Url` and `Email`. `hints` is the only place for an option specific to a plugin: Pollora passes it as it is.

Pick the driver in `config/meta.php`:

```php
return [
    'ui' => 'acf',      // null by default: no field is generated
];
```

On `init`, once the meta are registered, the driver receives each schema with the meta it supports; a meta it cannot handle gets no field. Values keep going through WordPress's meta API, so sanitization, `rules` and `capability` apply to what the fields save.

### Writing a driver

A driver implements `Pollora\Meta\Domain\Contracts\MetaUiDriver` and its package registers it:

```php
use Pollora\Meta\Domain\Contracts\MetaUiDriver;
use Pollora\Meta\Domain\Models\MetaDefinition;
use Pollora\Meta\Domain\Models\MetaSchema;

final class AcfDriver implements MetaUiDriver
{
    public function supports(MetaDefinition $definition): bool
    {
        return $definition->control !== null;   // what the plugin can store in the schema's form
    }

    public function register(MetaSchema $schema): void
    {
        // $schema->objectType, $schema->subtypes, and for each $schema->definitions:
        // key, label, description, control, group, hints, rules, default…
    }
}

// In the package's service provider
Meta::extend('acf', AcfDriver::class);
```

A driver writes values in the form the schema stores them (`supports()` says false otherwise) and lets WordPress's meta API save them. `Pollora\Meta\Testing\MetaUiDriverConformance::check($driver)` returns what breaks the contract, from any test framework: `expect(MetaUiDriverConformance::check(new AcfDriver))->toBe([])`.

Tooling can read the compiled schemas with `Meta::schemas()` and `Meta::schemaFor('post', 'event')`, or listen to the `Pollora\Meta\Domain\Events\MetaSchemasRegistered` event.

## Checking typed meta

`php artisan pollora:meta:list` lists every typed meta by class: the object it belongs to, then each meta with its key, its type and its options (`rest`, `media`, `public`, `rules`…). `--json` gives the same as JSON.

`php artisan pollora:doctor`, and **Tools › Site Health** in wp-admin, check that every declaration is registered and reachable:

- a class discovery refused (see [below](#errors-at-discovery)): its meta are never registered;
- a post type or taxonomy named by `#[PostMeta]` or `#[TermMeta]` that does not exist — a typo registers meta nothing uses;
- meta marked `showInRest` on a post type or taxonomy that is not in REST: WordPress leaves them out of every response;
- stored values that cannot be read as their type, on a sample of each key.

`php artisan pollora:meta:audit` reads every stored value (up to `--limit`, 10,000 per key by default):

```
   ERROR  1 meta have stored values that read as the default.

  Event::$capacity post "event" .............................. 2 / 340 object(s)
    The stored value of the meta "capacity" cannot be read as integer: 'a lot'. — objects 118, 214

   WARN  1 key(s) stored on your post types and taxonomies are declared by no #[Meta]: left by a renamed property, or written by a plugin.

  seats post "event" ........................................... 340 object(s)
```

A value that cannot be read as its type reads as the property default (and is logged): the page shows the default, with no error. Writes through `Meta::of()`, the models and `update_post_meta()` are sanitized to the type, so such a value comes from elsewhere — a SQL import, a plugin writing the table, a property whose type changed. The command exits with 1 when it finds one, so it can run in CI against a copy of production. The keys nothing declares are listed for information: on the post types and taxonomies the project declares, keys not starting with `_`.

## Errors at discovery

A declaration WordPress cannot register is refused at discovery, and the error is logged with the class and the property named; the other meta of the project still register:

- a union type, an untyped property, or an unsupported type;
- an array without item type, an array of arrays, `single: false` on something else than an array of scalars, dates or enums;
- an object property that is an array or an object, or that has neither default nor nullable type;
- an enum without backing values (`enum Mood { … }` instead of `enum Mood: string { … }`);
- a non-nullable property without a default value;
- a protected key exposed in REST without a `capability`;
- `revisions: true` on anything but posts;
- the same key declared twice, in one class or by two classes for the same objects (a post type in common, users, comments).
