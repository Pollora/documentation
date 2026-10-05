# Roles & Capabilities

WordPress decides what a user may do with **capabilities** (`edit_posts`, `manage_options`…), which users get through their **roles** (`editor`, `author`…). Pollora connects them to Laravel's authorization — `$user->can()`, the `can:` middleware, `@can` in Blade — and lets you declare roles in code instead of storing them in the database.

> **Experimental.** Declaring roles with `#[Role]`, `#[ModifyRole]` and `#[CapabilitySet]` is new: the API may still change before it is declared stable. Checking capabilities is stable.

## Checking capabilities

### In PHP

The authenticated user is a `Pollora\Models\User`, and Laravel's authorization answers with WordPress capabilities:

```php
use Illuminate\Support\Facades\Gate;

$user = auth()->user();

$user->can('edit_posts');            // WordPress capability
$user->can('edit_post', $post);      // meta capability, resolved by map_meta_cap()
$user->cannot('manage_options');

Gate::allows('edit_post', $post);
Gate::authorize('manage_options');   // throws an AuthorizationException (403)
```

Eloquent models passed as arguments (`$post` above, a `Pollora\Models\Post`) are converted to their ID, which is what WordPress expects. A guest is denied.

Abilities you define yourself with `Gate::define()`, and policies, keep priority: WordPress is only asked when they do not decide. Capabilities declared in an enum ([see below](#project-capabilities)) can be passed as enum cases: `Gate::allows(EventCap::ExportAttendees)`.

WordPress's own functions keep working: `current_user_can('edit_posts')` and `user_can($userId, 'edit_posts')` give the same answers.

### In routes

```php
Route::get('/reports', ReportController::class)->middleware('can:manage_options');
```

A user without the capability gets a 403 response.

### In Blade

| Directive | Shows its content when | Provided by |
|---|---|---|
| `@can('edit_posts') … @endcan` | the user has the capability | Laravel |
| `@cannot('manage_options') … @endcannot` | the user does not have it | Laravel |
| `@canany(['edit_posts', 'moderate_comments']) … @endcanany` | the user has at least one of them | Laravel |
| `@role('editor', 'author') … @endrole` | the user has one of these roles | Sage Directives |
| `@user … @enduser` | a user is logged in | Sage Directives |
| `@guest … @endguest` | nobody is logged in | Sage Directives |

`@can` accepts arguments and alternatives, like any Laravel ability:

```blade
@can('edit_post', $post)
    <a href="{{ get_edit_post_link($post->ID) }}">Edit</a>
@endcan

@can('manage_options')
    <a href="{{ admin_url() }}">Administration</a>
@elsecan('export_attendees')
    <a href="{{ route('attendees.export') }}">Export attendees</a>
@else
    <p>Nothing to manage here.</p>
@endcan
```

**Check a capability rather than a role.** `@can('export_attendees')` stays right when you later give that capability to a second role; `@role('event_manager')` becomes wrong. Keep `@role` for content that is about the role itself, such as a welcome message.

## Post types with their own capabilities

By default a post type shares the capabilities of posts: anyone who can edit posts can edit its entries. To control it separately, give it its own capability type:

```php
use Pollora\Attributes\PostType;
use Pollora\Attributes\PostType\CapabilityType;
use Pollora\Attributes\PostType\MapMetaCap;

#[PostType('event')]
#[CapabilityType('event')]
#[MapMetaCap]
class Event {}
```

WordPress then expects capabilities named after it: `edit_events`, `publish_events`, `edit_others_events`… **No role has them**, administrators included — the post type would disappear from the admin. Pollora gives them to the **super roles** automatically (see [Super roles](#super-roles)), and you grant them to other roles with [`#[GrantsPostType]`](#post-types-and-taxonomies).

`#[MapMetaCap]` lets WordPress turn checks on a single entry (`edit_post` on post 42) into these capabilities; keep it with `#[CapabilityType]`. The names follow `get_post_type_capabilities()`; `#[Capabilities([...])]` renames some of them, and Pollora uses the renamed ones. See the [post type attributes reference](post-types-reference.md).

Taxonomies work the same way: one with its own `#[Capabilities]` (`manage_terms`, `edit_terms`, `delete_terms`, `assign_terms`) stops sharing `manage_categories` with categories.

## Declaring a role

A role is a class carrying `#[Role]`, with attributes that grant or remove capabilities. Generate one with:

```bash
php artisan pollora:make:role EventManager
```

```php
use App\Cms\PostTypes\Event;
use App\Cms\Roles\EventCap;
use Pollora\Attributes\Role;
use Pollora\Attributes\Role\Grants;
use Pollora\Attributes\Role\GrantsPostType;
use Pollora\Attributes\Role\Without;
use Pollora\Role\Domain\Enums\Access;

#[Role('event_manager', label: 'Event manager', inherits: 'author')]
#[GrantsPostType(Event::class, Access::Editor)]
#[Grants(EventCap::ExportAttendees, 'moderate_comments')]
#[Without('publish_posts')]
final class EventManager {}
```

This role starts from the capabilities of an author, manages every event, can export attendees and moderate comments, and cannot publish posts. Like other attribute classes, it is discovered automatically in `app/`, modules, plugins and themes.

| Attribute | Parameters | Effect |
|---|---|---|
| `#[Role]` | `slug`, `label`, `inherits`, `allowSensitive` | Declares the role. `label` defaults to the class name. `inherits` names a role — a slug or the class of another `#[Role]` — whose capabilities are the starting point |
| `#[Grants]` | capabilities, as strings or enum cases | Adds capabilities. Repeatable |
| `#[Without]` | capabilities | Removes capabilities, typically inherited ones. Repeatable |
| `#[GrantsPostType]` | post type class or slug, `Access` level | Adds the capabilities of a post type with its own capability type. Repeatable |
| `#[GrantsTaxonomy]` | taxonomy class or slug | Adds the term capabilities of a taxonomy with its own capabilities. Repeatable |

Inheritance follows the parent as it is on each request: when a plugin adds a capability to `author`, `event_manager` gets it too.

### Post types and taxonomies

`#[GrantsPostType]` takes an access level, named after the core role that has it on posts. Each level includes the previous ones. For the `event` capability type:

| Level | The user can | Capabilities added |
|---|---|---|
| `Access::Contributor` | create and edit their own drafts, without publishing | `edit_events`, `delete_events` |
| `Access::Author` | publish and manage their own entries | + `publish_events`, `edit_published_events`, `delete_published_events` |
| `Access::Editor` | manage everyone's entries, private ones included | + `edit_others_events`, `delete_others_events`, `read_private_events`, `edit_private_events`, `delete_private_events` |

Pass the post type's class when you can: Pollora checks it at once and refuses a post type that has no `#[CapabilityType]`, with the attribute to add. A slug is resolved later, which also covers a post type declared in a theme or a plugin; one that cannot be found is skipped and logged.

## Modifying an existing role

A role the project does not own — `editor`, WooCommerce's `shop_manager`, a plugin's role — is adjusted with `#[ModifyRole]`, without redefining it:

```php
use Pollora\Attributes\ModifyRole;
use Pollora\Attributes\Role\Grants;
use Pollora\Attributes\Role\GrantsPostType;
use Pollora\Attributes\Role\Without;
use Pollora\Role\Domain\Enums\Access;

#[ModifyRole('editor')]
#[GrantsPostType(Event::class, Access::Editor)]
#[Without('edit_theme_options')]
final class EditorAdjustments {}
```

Several classes may modify the same role, for instance one per module. If one grants a capability another removes, discovery refuses the second and names both.

## Project capabilities

Capabilities your own code checks are best declared in a backed enum marked `#[CapabilitySet]`:

```php
use Pollora\Attributes\CapabilitySet;

#[CapabilitySet(label: 'Events')]
enum EventCap: string
{
    case ExportAttendees = 'export_attendees';
    case ScanTickets = 'scan_tickets';
    case RefundTickets = 'refund_tickets';
}
```

You grant them as enum cases (`#[Grants(EventCap::ScanTickets)]`), check them the same way (`Gate::allows(EventCap::RefundTickets)`), and the super roles receive all of them — including a capability no role is granted yet, which only administrators then have.

## Super roles

The roles listed in `roles.super_roles` — `administrator` by default — receive every capability the project declares: those of post types with `#[CapabilityType]`, of taxonomies with their own capabilities, and of `#[CapabilitySet]` enums. To change the list, create `config/roles.php` in your project:

```php
return [
    'super_roles' => ['administrator', 'shop_manager'],
];
```

A `#[Role]` cannot inherit from a super role.

## How roles reach WordPress

WordPress stores roles in the database (the `{prefix}user_roles` option), and `add_role()` writes them once, then does nothing on later calls. Pollora does not write declared roles there: it puts them into WordPress's role registry **in memory, on every request**, when WordPress builds it (the `wp_roles_init` action).

- **The code is the only source of truth.** Change a class and deploy: the role follows, on every environment, with nothing to migrate. Revert the commit and the previous rights are back.
- **Removed from the code means gone.** A plugin that calls `add_cap()` makes WordPress write all in-memory roles back to the database, declared ones included. Pollora marks what it adds, and undoes it on the next request if the declaration is no longer there: the role disappears, a capability granted by a `#[ModifyRole]` is removed, and one it removed is restored. Users who had a removed role keep its slug but get no capability from it.
- **Role editor plugins** cannot change a declared role: the code wins on every request.
- **Multisite:** roles are injected again on each site switch.
- **Plugins and themes:** their roles are discovered while WordPress loads them, after it has built its roles; Pollora injects them into the existing registry and recomputes the current user's capabilities, so they apply to the request that is running.

A tool that reads the `user_roles` option directly, without loading the site, does not see declared roles.

## Safety rules

Distributing rights is easy to get wrong, so discovery refuses a declaration that would grant the wrong rights — the error is logged with the class named, and that declaration is not applied:

- **Sensitive capabilities** — `manage_options`, `edit_users`, `create_users`, `delete_users`, `promote_users`, `unfiltered_html`, `unfiltered_upload`, `install_plugins`, `activate_plugins`, `edit_plugins`, `edit_themes`, `edit_files`, `update_core` — can only be granted with `allowSensitive: true` on `#[Role]` or `#[ModifyRole]`. The flag makes the choice visible in code review.
- **Core roles** (`administrator`, `editor`, `author`, `contributor`, `subscriber`) cannot be redeclared with `#[Role]`; use `#[ModifyRole]`.
- **No inheritance from a super role**, no inheritance loop, no slug declared by two classes.
- **A capability both granted and removed** by the same class is refused.
- **`#[Without]` removes a capability; it never sets it to `false`.** WordPress merges the roles of a user, and an explicit denial in one role would override or be overridden by another depending on their order.

## Not available yet

- Checking roles in PHP (`hasRole()`), a `role:` route middleware and a REST permission for capabilities are planned. Until then, check capabilities with `can()` and the `can:` middleware, and roles in Blade with `@role`.
- Role labels are shown as declared; they are not translated yet.
