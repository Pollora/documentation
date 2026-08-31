# Abilities

WordPress 6.9 introduced the **Abilities API**: a registry where plugins, themes and core declare what they can do in a machine-readable form — inputs, outputs, permissions, behaviour — so that AI agents and automation tools can discover and invoke site functionality without a bespoke integration for each one.

Pollora wraps it in the [`pollora/abilities`](https://github.com/Pollora/abilities) package, with two ways to declare an ability: the **imperative `Ability` facade** and the **declarative `#[Ability]` attribute**. Permission checks default to *refusing*, so an ability that forgets to declare one is inert rather than open.

> **Requires WordPress 6.9 or later.** On an older install the declarations are accepted and simply never published — there is nowhere to put them.

## Abilities and MCP

An ability is not an MCP tool, though it is what an MCP tool is usually made of. The [MCP Adapter](https://github.com/WordPress/mcp-adapter) plugin publishes registered abilities over the Model Context Protocol; the core abilities REST controllers expose them at `/wp-json/wp-abilities/v1/abilities`. Neither is Pollora's concern: you declare the ability once, and whatever consumes abilities picks it up.

That is also why the package is named for abilities rather than for MCP — it implements no part of that protocol.

## Categories

Every ability is filed under a category, and the category has to exist first. Declare it once, typically in a service provider:

```php
use Pollora\Support\Facades\Ability;

Ability::category('acme-content', 'Editorial', 'Posts and pages.');
```

Category slugs are **global to the install** and WordPress core already claims several of them, so prefix yours with something the project owns. A collision is not benign: the second registration is refused and every ability pointing at the category fails with it.

The label falls back to a title-cased slug, and the description to one derived from the label. Both fallbacks exist because WordPress rejects a category with a blank description by returning `null` rather than raising — an empty one would vanish without a word.

## Imperative API (Facade)

```php
use Pollora\Abilities\Domain\Model\Input;
use Pollora\Abilities\Domain\Schema\SchemaBuilder;
use Pollora\Support\Facades\Ability;

Ability::define('acme/get-posts')
    ->description('Returns the most recent posts, newest first.')
    ->category('acme-content')
    ->input(fn (SchemaBuilder $schema) => $schema
        ->integer('limit', 'How many posts to return.', default: 10, minimum: 1, maximum: 100))
    ->can(fn (Input $input): bool => current_user_can('edit_posts'))
    ->using(fn (Input $input): array => array_map(
        static fn (WP_Post $post): array => ['id' => $post->ID, 'title' => $post->post_title],
        get_posts(['numberposts' => $input->integer('limit', 10)]),
    ));
```

Nothing is registered until a body is supplied through `using()` or `handledBy()`, so an abandoned chain declares nothing rather than something broken.

Ability names are `namespace/slug`, both parts lowercase alphanumerics separated by single dashes. A bare slug registers nothing and WordPress reports it through `_doing_it_wrong()`, where it is easy to miss — so Pollora refuses it at declaration time instead, along with an empty label, an empty description, or an input schema that is not an object.

### When declarations are published

Declaration and registration are two phases. You declare wherever it is natural to write it — a service provider, a discovered class, a theme bootstrap — which is almost always before WordPress has initialised its abilities registry. Pollora queues the declaration and flushes it on `wp_abilities_api_categories_init` and then `wp_abilities_api_init`, the only moments WordPress accepts them.

You never hook those yourself; the framework does. It does mean an ability declared *after* something has already touched the registry is too late for that request.

## Declarative API (Attribute)

For anything past a couple of lines, write a class. Everything the ability needs sits in one place with a typed signature, which makes it straightforward to unit-test without registering anything:

```php
use Pollora\Abilities\Domain\Contracts\AbilityHandler;
use Pollora\Abilities\Domain\Model\Behaviour;
use Pollora\Abilities\Domain\Model\Input;
use Pollora\Abilities\Domain\Schema\SchemaBuilder;
use Pollora\Attributes\Ability;

#[Ability(
    name: 'acme/create-post',
    description: 'Creates a post from a title and a status.',
    category: 'acme-content',
    behaviour: Behaviour::Creates,
)]
final class CreatePost implements AbilityHandler
{
    public function schema(SchemaBuilder $schema): void
    {
        $schema->string('title', 'Title of the post to create.', required: true);
        $schema->enum('status', 'Publication status.', ['draft', 'publish'], default: 'draft');
    }

    public function authorize(Input $input): mixed
    {
        return current_user_can('edit_posts')
            ?: new WP_Error('forbidden', 'You cannot create posts.', ['status' => 403]);
    }

    public function handle(Input $input): mixed
    {
        return ['id' => wp_insert_post([
            'post_title'  => $input->string('title'),
            'post_status' => $input->string('status', 'draft'),
        ])];
    }
}
```

The class is discovered anywhere the discovery system scans — `app/`, a theme, a module — and instantiated through the service container, so constructor injection works as expected.

The attribute carries what the ability *is*; the handler carries what it *does*. If the named category has not been declared, Pollora declares it for you rather than letting the ability disappear; an explicit `Ability::category()` still wins, whichever ran first.

### Attribute Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | *(required)* | Fully-qualified ability name, `namespace/slug` |
| `description` | `string` | *(required)* | What the ability does — this is the tool description |
| `category` | `string` | *(required)* | Slug of the category the ability is filed under |
| `label` | `string` | *(title-cased slug)* | Short human-readable title |
| `behaviour` | `Behaviour` | `Behaviour::Reads` | What the ability does to the site |

A class carrying `#[Ability]` without implementing `AbilityHandler` is logged as an error rather than skipped: the attribute is an explicit statement of intent, so failing to honour it is worth surfacing.

## Behaviour

Every ability declares what it does to the site. WordPress publishes this under `meta.annotations`, and consumers turn it into the `readOnlyHint`, `destructiveHint` and `idempotentHint` tool annotations a client uses to decide how much ceremony an invocation deserves — a read may run unattended, a delete should be confirmed with the user first.

| Facade | Attribute | `readonly` | `destructive` | `idempotent` | Means |
|---|---|---|---|---|---|
| `->reads()` *(default)* | `Behaviour::Reads` | ✓ | | ✓ | Changes nothing |
| `->creates()` | `Behaviour::Creates` | | | | Adds something on every call — two calls, two records |
| `->updates()` | `Behaviour::Updates` | | ✓ | ✓ | Overwrites part of a record; the previous value is gone |
| `->deletes()` | `Behaviour::Deletes` | | ✓ | ✓ | Removes a record |

Getting these wrong is worse than omitting them, which is why they are declared as one of four shapes rather than three loose booleans.

> **They are advisory.** WordPress does not enforce them. The permission callback is what protects the site.

## Permissions

The permission check receives the **same input** as the body, which is what allows per-object checks — `edit_post` on the identifier being edited, rather than a blanket `edit_posts`:

```php
Ability::define('acme/update-post')
    ->description('Updates the title of an existing post.')
    ->category('acme-content')
    ->updates()
    ->input(fn (SchemaBuilder $schema) => $schema
        ->integer('id', 'Identifier of the post to update.', required: true)
        ->string('title', 'The new title.', required: true))
    ->can(fn (Input $input): bool => current_user_can('edit_post', $input->id('id')))
    ->using(fn (Input $input): array => …);
```

Return a `WP_Error` instead of `false` to explain the refusal. A client that knows *why* it was refused can act on it, where a bare denial leaves the model guessing.

Omitting `can()` entirely means the ability refuses everything. That is deliberate: a forgotten check should make an ability useless, not public.

## Input

`Input` is a defensive reader over what the ability was handed. WordPress validates against the declared schema before the body runs, but it does not guarantee a shape — an ability whose every property is optional can legitimately be invoked with `null`.

```php
$input->string('title');                              // '' when absent
$input->integer('limit', default: 10, max: 100);      // coerces "12", clamps 100000 → 100
$input->float('score', min: 0.0, max: 1.0);
$input->id('post_id');                                // 0 when absent or invalid, never negative
$input->boolean('draft');                             // accepts true, "true", "1", "yes", "on"
$input->stringList('tags');                           // trims, drops blanks and non-scalars
$input->idList('post_ids');
$input->map('terms');                                 // free-form associative array
$input->all();                                        // everything, for the rare case
```

Accessors coerce rather than throw. A model that sends `"12"` where an integer was asked for should get a working call, not an error it cannot act on.

`has()` and `filled()` are distinct on purpose:

| Method | `''` | `[]` | `0` | absent | `null` |
|---|---|---|---|---|---|
| `has()` | ✓ | ✓ | ✓ | | |
| `filled()` | | | ✓ | | |

Use `filled()` by default — callers routinely send empty strings for properties they mean to leave alone, and treating those as present produces empty search terms and cleared taxonomies. Use `has()` where "explicitly zero" is a different instruction from "not mentioned", such as a menu order or a parent identifier.

## Schema

The input schema is the only documentation a language model gets about your ability, so descriptions are not decoration — they *are* the interface. Every `SchemaBuilder` method takes one, and there is no overload that omits it.

```php
$schema
    ->string('title', 'Title of the post.', required: true)
    ->string('url', 'Source URL.', format: 'uri')
    ->enum('status', 'Publication status.', ['draft', 'publish'], default: 'draft')
    ->integer('limit', 'How many to return.', default: 10, minimum: 1, maximum: 100)
    ->number('score', 'Relevance threshold.', minimum: 0.0, maximum: 1.0)
    ->boolean('sticky', 'Whether to pin the post.')
    ->list('tags', 'Tag slugs to attach.')
    ->map('terms', 'Taxonomy slug to term slugs.', ['type' => 'array'])
    ->object('author', 'The post author.', fn (SchemaBuilder $author) => $author
        ->integer('id', 'User identifier.', required: true))
    ->raw('id', ['oneOf' => [['type' => 'string'], ['type' => 'integer']]]);
```

Two habits worth keeping:

- **Prefer `enum()` over a free string** wherever the accepted values are known. A model that can see the options picks one; a model given a free string invents a plausible value that fails downstream.
- **Set `maximum` on anything that sizes a query**, so a model cannot ask for every row in the table.

WordPress validates input against this schema before your body runs, and returns a `WP_Error` naming the offending property when it does not match — you do not have to check for missing required values yourself.

### Describing the output

`output()` is optional and takes the same builder. Declare it where the returned shape is stable: it lets a client validate what it got instead of trusting it.

```php
Ability::define('acme/get-post')
    ->description('Returns a single post by identifier.')
    ->category('acme-content')
    ->output(fn (SchemaBuilder $schema) => $schema
        ->integer('id', 'The post identifier.')
        ->string('title', 'The post title.'))
    // …
```

## Testing an ability

Registered abilities are exposed through the core REST controllers, so you can exercise one without an MCP client:

```bash
wp eval '$a = wp_get_ability("acme/get-posts"); var_dump($a->execute(["limit" => 3]));'
```

```
GET  /wp-json/wp-abilities/v1/abilities
GET  /wp-json/wp-abilities/v1/abilities/acme/get-posts
POST /wp-json/wp-abilities/v1/abilities/acme/get-posts/run
```

A handler class needs no WordPress at all to unit-test — call `schema()`, `authorize()` and `handle()` directly:

```php
it('creates a draft post', function (): void {
    $result = (new CreatePost)->handle(Input::wrap(['title' => 'Hello']));

    expect($result['id'])->toBeInt();
});
```

## Package API

The framework wires the package up; you normally only touch the facade and the attribute. The pieces underneath, for the rare case that needs them:

| Class | Purpose |
|---|---|
| `Pollora\Abilities\Factory\AbilityFactory` | Bound as `wp.abilities`, what the facade resolves |
| `Pollora\Abilities\Application\Service\RegisterAbilityService` | Holds the declaration queues; `registered()` reports what went live |
| `Pollora\Abilities\Port\Out\AbilityRegistrarPort` | Swap to publish declarations somewhere other than WordPress |
| `Pollora\Abilities\Domain\Model\Ability` | The immutable declaration itself |

`RegisterAbilityService::registered()` is worth knowing about: it returns the exact list of ability names published this request, which is what a downstream consumer — an MCP server declaration, a settings screen — needs without rediscovering it.

## Related

- [Discovery](discovery.md) — how `#[Ability]` classes are found
- [WP REST API](wp-rest-api.md) — `#[WpRestRoute]` for hand-written endpoints
- [Nectar — AI Context](nectar.md) — AI guidelines and agent skills for development
