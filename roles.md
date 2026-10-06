# Roles & Capabilities

WordPress decides what a user may do with **capabilities** (`edit_posts`, `manage_options`…), which users get through their **roles** (`editor`, `author`…). Pollora connects them to Laravel's authorization — `$user->can()`, the `can:` middleware, `@can` in Blade — checks roles the same way (`hasRole()`, `role:`, `@role`), and lets you declare roles in code instead of storing them in the database.

> **Experimental.** Declaring roles with `#[Role]`, `#[ModifyRole]` and `#[CapabilitySet]`, and checking roles with `hasRole()`, `role:` and `@role` on role classes, are new: the API may still change before it is declared stable. Checking capabilities is stable.

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

In a WordPress REST route declared with `#[WpRestRoute]`, use the `Can` permission: `permissionCallback: new Can('edit_posts')` (see [REST API](wp-rest-api.md#checking-a-capability)).

### In Blade

| Directive | Shows its content when | Provided by |
|---|---|---|
| `@can('edit_posts') … @endcan` | the user has the capability | Laravel |
| `@cannot('manage_options') … @endcannot` | the user does not have it | Laravel |
| `@canany(['edit_posts', 'moderate_comments']) … @endcanany` | the user has at least one of them | Laravel |
| `@role('editor', EventManager::class) … @endrole` | the user has one of these roles | Pollora |
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

`@role` takes slugs, compared without regard to case, or the classes of [declared roles](#declaring-a-role) — in a view, write the class with its namespace or import it with `@use`. It replaces the directive of the same name from Sage Directives, which only took slugs.

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
| `#[Role]` | `slug`, `label`, `inherits`, `allowSensitive`, `textDomain` | Declares the role. `label` defaults to the class name. `inherits` names a role — a slug or the class of another `#[Role]` — whose capabilities are the starting point. `textDomain` translates the label ([see below](#translated-labels)) |
| `#[Grants]` | capabilities, as strings or enum cases | Adds capabilities. Repeatable |
| `#[Without]` | capabilities | Removes capabilities, typically inherited ones. Repeatable |
| `#[GrantsPostType]` | post type class or slug, `Access` level | Adds the capabilities of a post type with its own capability type. Repeatable |
| `#[GrantsTaxonomy]` | taxonomy class or slug | Adds the term capabilities of a taxonomy with its own capabilities. Repeatable |

Inheritance follows the parent as it is on each request: when a plugin adds a capability to `author`, `event_manager` gets it too.

### Translated labels

WordPress looks for role names in its own catalogue only. Give the label your theme's or plugin's text domain to translate it from your catalogue:

```php
#[Role('event_manager', label: 'Event manager', textDomain: 'my-theme')]
final class EventManager {}
```

The label is then translated wherever WordPress shows role names — users list, role dropdowns, profile screen. Add `Event manager` to your `.po` file as a plain string, without context: `__('Event manager', 'my-theme')` in a file the extraction tool scans is enough to collect it. Translation happens when the admin displays the role, so it never loads your catalogue too early.

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

## Inspecting roles

`php artisan pollora:roles:list` lists the roles WordPress has once the code is applied, with where each comes from — declared by a class, stored and modified by a `#[ModifyRole]`, or stored as it is — its number of capabilities and the number of users who carry it.

`php artisan pollora:roles:show event_manager` (or `EventManager::class`) lists the effective capabilities of a role and where each comes from, then what the code removes:

```
  event_manager Event manager ............................ App\Cms\Roles\EventManager
  inherits ............................................................. author

+------------------------+---------------------------+
| Capability             | From                      |
+------------------------+---------------------------+
| edit_events            | granted by EventManager   |
| edit_posts             | inherited from author     |
| export_attendees       | granted by EventManager   |
| ...                    |                           |
+------------------------+---------------------------+

   INFO  Removed by the code: publish_posts.
```

Both take `--json`.

`php artisan pollora:doctor`, and **Tools › Site Health** in wp-admin, check what no error ever shows:

- **Users carrying a role removed from the code.** They keep its slug and get no capability from it: the check names the role and the users. Give them another role.
- **A `default_role` naming a role that no longer exists**: every new user would get no capability.
- **Capabilities given to users one by one**, outside the roles: they live in the database, not in the code.
- **What a declaration could not apply**, such as a post type grant whose post type has no capabilities of its own (until now only logged).

## Cleaning up and migrating

These commands change the database. Each shows what it would do and changes nothing unless it is run again with `--force`.

**`php artisan pollora:roles:prune --reassign=subscriber`** cleans up after roles removed from the code: it takes the dead role off the users who still carry it, gives those left with no role the `--reassign` one, and deletes the copies of removed roles a plugin wrote back to the database. Without `--reassign`, it refuses to leave a user with no role.

```
  jane ................................... remove event_manager → subscriber
  joe ................................................ remove event_manager
  stored role "event_manager" ......... delete the copy a plugin wrote back

   WARN  Dry run: nothing changed. Run again with --force to apply.
```

**`php artisan pollora:roles:import venue_staff --inherits=author`** turns a role stored in the database — made by `add_role()` or a role editor plugin — into a `#[Role]` class, in `app/Cms/Roles` (or `--theme`, `--plugin`, `--module`). With `--inherits`, the class only holds the differences:

```php
#[Role('venue_staff', label: 'Venue staff', inherits: 'author')]
#[Grants(
    'scan_tickets',
)]
#[Without(
    'publish_posts',
)]
final class VenueStaff
{
}
```

Review it before committing: a sensitive capability gets `allowSensitive: true` and a note, and a capability stored as denied (`false`) is listed in the docblock and left out, since a role only grants or removes. Core roles are refused: change them with `#[ModifyRole]`. Once the class is deployed, the code owns the role; the stored copy is ignored.

**`php artisan pollora:roles:dump`** writes the roles as the code makes them into the `{prefix}user_roles` option, for a tool that reads the database without loading the site. The code stays the source of truth: roles are still injected on every request, and what the command writes is marked, so a role later removed from the code is still removed.

## Safety rules

Distributing rights is easy to get wrong, so discovery refuses a declaration that would grant the wrong rights — the error is logged with the class named, and that declaration is not applied:

- **Sensitive capabilities** — `manage_options`, `edit_users`, `create_users`, `delete_users`, `promote_users`, `unfiltered_html`, `unfiltered_upload`, `install_plugins`, `activate_plugins`, `edit_plugins`, `edit_themes`, `edit_files`, `update_core` — can only be granted with `allowSensitive: true` on `#[Role]` or `#[ModifyRole]`. The flag makes the choice visible in code review.
- **Core roles** (`administrator`, `editor`, `author`, `contributor`, `subscriber`) cannot be redeclared with `#[Role]`; use `#[ModifyRole]`.
- **No inheritance from a super role**, no inheritance loop, no slug declared by two classes.
- **A capability both granted and removed** by the same class is refused.
- **`#[Without]` removes a capability; it never sets it to `false`.** WordPress merges the roles of a user, and an explicit denial in one role would override or be overridden by another depending on their order.

## Checking roles

A role is named by its slug (`'editor'`) or by the class of a `#[Role]` (`EventManager::class`), which the IDE can follow and rename.

### In PHP

`Pollora\Models\User` has these methods:

```php
$user = auth()->user();

$user->roles();                                // ['author', 'event_manager']
$user->hasRole(EventManager::class);           // true
$user->hasRole('editor', 'administrator');     // true if the user has one of them

$user->assignRole(EventManager::class);        // adds the role, keeping the others
$user->removeRole('event_manager');
```

`assignRole()` and `removeRole()` go through `WP_User`, so WordPress's hooks (`add_user_role`, `remove_user_role`) and user cache follow. `assignRole()` refuses a role WordPress does not know; `removeRole()` also removes a role that no longer exists. **Neither checks the rights of the code calling them**, like `WP_User::set_role()`: where a user can trigger them, check `promote_users` first.

Your own user model gets the same methods with the `Pollora\Models\Concerns\HasRoles` trait, provided it has a `toWpUser(): WP_User` method.

### In routes

```php
use Pollora\Role\Infrastructure\Middleware\EnsureUserHasRole;

Route::middleware('role:event_manager,editor')->group(function () {
    // …
});

Route::get('/events/scan', ScanController::class)
    ->middleware(EnsureUserHasRole::using(EventManager::class));
```

A user who has none of the roles, or a guest, gets a 403 response. If your application already uses the `role` alias, for another package, Pollora keeps it: use `EnsureUserHasRole::using()`.

As in Blade, prefer `can:` with a capability: `can:export_attendees` stays right when a second role is given that capability, `role:event_manager` does not.
