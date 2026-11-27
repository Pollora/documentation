# AJAX

Pollora provides two ways to register WordPress AJAX handlers: the **imperative `Ajax` facade** and the **declarative `#[Ajax]` attribute**. Both default to logged-in users only and verify a [nonce](#nonces-and-csrf) before your handler runs (security-by-default, `pollora/ajax` 2.0), and both can require a [capability](#capabilities-who-may-do-it).

> **In a Pollora project, this is not in effect yet.** The framework still requires `pollora/ajax: ^1.3`, which runs an action that chose nothing **without a nonce check**. Until the framework accepts 2.0, declare `nonce: true` (or `requireNonce()`) on every action that needs the protection: it works in both versions. What this page says about the default applies to `pollora/ajax` 2.0 installed on its own.

> **Coming from `pollora/ajax` 1.x?** An action that chose nothing ran without a nonce check. It is now refused with a `403` unless the request carries one: see [Upgrading from 1.x](#upgrading-from-1x).

## Imperative API (Facade)

Use the `listen()` method to register an AJAX handler inline:

```php
use Pollora\Support\Facades\Ajax;

Ajax::listen('my_action', function () {
    wp_send_json_success(['message' => 'It works!']);
});
```

The request must carry a nonce for `my_action`, see [Nonces and CSRF](#nonces-and-csrf). An action that **changes data** should also require a capability, see [Capabilities](#capabilities-who-may-do-it).

By default, this registers `wp_ajax_my_action` **only** — the handler is restricted to **logged-in users**. Unauthenticated users cannot reach it.

### Targeting Users

```php
// Default — logged-in users only (wp_ajax_*), nonce verified
Ajax::listen('my_action', function () {
    wp_send_json_success(['user' => wp_get_current_user()->display_name]);
});

// Explicit — all users, logged-in AND guests (wp_ajax_* + wp_ajax_nopriv_*)
Ajax::listen('public_action', function () {
    wp_send_json_success(['message' => 'Hello everyone!']);
})->forAllUsers()->withoutNonce(); // public read: no nonce, on purpose

// Guest users only (wp_ajax_nopriv_*)
Ajax::listen('guest_action', function () {
    wp_send_json_success(['message' => 'Hello guest!']);
})->forGuestUsers()->withoutNonce();
```

### Using a Controller Method

```php
Ajax::listen('load_more_posts', [PostController::class, 'loadMore']);
```

## Declarative API (Attribute)

Place the `#[Ajax]` attribute on a public method to auto-register it as an AJAX handler via the discovery system — no manual registration needed.

```php
use Pollora\Attributes\Ajax;
use Pollora\Ajax\Domain\Model\AjaxAccess;

class NewsletterHandler
{
    // Logged-in users only and nonce verified: both are the defaults
    #[Ajax('subscribe')]
    public function subscribe(): void
    {
        wp_send_json_success(['message' => 'Subscribed!']);
    }

    // All users (explicit opt-in); it only reads, so there is nothing to protect
    #[Ajax('load_more', access: AjaxAccess::ALL, nonce: false)]
    public function loadMore(): void
    {
        wp_send_json_success([/* ... */]);
    }

    // Guest users only, no nonce on purpose: a beacon that answers and stores nothing
    #[Ajax('ping', access: AjaxAccess::GUEST, nonce: false)]
    public function ping(): void
    {
        wp_send_json_success(['ok' => true]);
    }

    // Changes data: the nonce is verified, then the user must be able to edit posts
    #[Ajax('save_note', capability: 'edit_posts')]
    public function saveNote(): void
    {
        wp_send_json_success([/* ... */]);
    }
}
```

The class is automatically discovered and instantiated via the service container, so constructor injection works as expected.

### Attribute Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `action` | `string` | *(required)* | The WordPress AJAX action name |
| `access` | `AjaxAccess` | `AjaxAccess::LOGGED` | Audience targeting |
| `nonce` | `bool\|string\|null` | `null` | `null` and `true` verify a nonce named after the action, a string verifies the nonce of that name, `false` turns it off on purpose (see [Nonces and CSRF](#nonces-and-csrf)) |
| `capability` | `string\|null` | `null` | The WordPress capability a logged-in user must have, checked with `current_user_can()` after the nonce; `null` requires none (see [Capabilities](#capabilities-who-may-do-it)) |

### AjaxAccess Enum

| Value | WordPress Hook | Audience |
|---|---|---|
| `AjaxAccess::LOGGED` | `wp_ajax_{action}` | Logged-in users only (default) |
| `AjaxAccess::ALL` | `wp_ajax_{action}` + `wp_ajax_nopriv_{action}` | Everyone |
| `AjaxAccess::GUEST` | `wp_ajax_nopriv_{action}` | Guests only |

## Security Model

An AJAX action has three questions to answer, and Pollora asks them in this order before your handler runs:

1. **Who may reach it?** The audience: logged-in users by default, `forAllUsers()` or `forGuestUsers()` on purpose.
2. **Does the request come from your page?** The [nonce](#nonces-and-csrf).
3. **May this user do it?** The [capability](#capabilities-who-may-do-it).

A nonce proves where a request comes from, **not that the user may act**: any logged-in user, a subscriber included, can obtain a valid nonce. For any action that changes data, use a capability **and** a nonce.

> **Why secure-by-default?** Exposing `wp_ajax_nopriv_*` allows any unauthenticated visitor to call the endpoint. This should be a conscious decision, not an implicit default. Both the facade and attribute API require explicit opt-in to expose an endpoint publicly.

## Nonces and CSRF

An AJAX request carries the visitor's cookies, so a page of another site can make a logged-in visitor's browser send it. A nonce, a token WordPress creates for a user and an action, tells a request your own page sent from one it did not. Pollora verifies it before your handler runs, for every audience, unless the action says it takes none.

### Verify the nonce

```php
// Nothing to declare: the nonce is verified before the callback runs
Ajax::listen('save_note', function () {
    wp_send_json_success(['message' => 'Saved!']);
});

// Another nonce action than the AJAX action name, when a form already has one
Ajax::listen('save_note', $callback)->requireNonce('notes_form');

// No nonce on purpose: a public beacon, a webhook that signs its own requests
Ajax::listen('ping', $callback)->forGuestUsers()->withoutNonce();
```

The attribute says the same: `#[Ajax('save_note')]`, `nonce: 'notes_form'` or `nonce: false`.

A request without a valid nonce gets a `403` and the callback does not run. The response is JSON:

```json
{ "success": false, "data": { "code": "pollora_ajax_invalid_nonce", "message": "Invalid or expired security token." } }
```

When `WP_DEBUG` is on (`APP_DEBUG` in a Pollora project), `data` also has a `hint` field for the developer: the nonce that was expected, the request fields and, for an action that did not choose, that the check is the default one. It is not sent otherwise.

The nonce is read from `_ajax_nonce`, then `_wpnonce`, in the request, as `check_ajax_referer()` does. By default the nonce action is the AJAX action name; create the nonce for it with `wp_create_nonce('save_note')` (the `Ajax` facade has no method for this) (see [Frontend JavaScript](#frontend-javascript)).

### An action that does not choose

An action that says neither `requireNonce()` nor `withoutNonce()` (`nonce: null`, the default of the attribute) verifies a nonce named after the action. There is nothing to add for the common case.

### Actions open to guests

Guests are verified too, but choose for them (`forGuestUsers()`, `forAllUsers()`):

- A guest's nonce is tied to no user: any visitor can read it from your page, so it protects much less than a logged-in user's nonce.
- A page served from a cache keeps its nonce after it expired (WordPress nonces live 12 to 24 hours), and the guest gets a `403`.

For a public or cached endpoint, say `withoutNonce()`. **On `forAllUsers()` this removes the check for logged-in users too**: keep the nonce, or split the action in two, when the handler does something on behalf of the logged-in user. An action open to guests that chooses nothing is reported through `_doing_it_wrong()` (visible with `WP_DEBUG`, skipped while an AJAX request is answered).

### Upgrading from 1.x

In `pollora/ajax` 1.x, an action that chose nothing ran **without a check**. From 2.0 it refuses a request without a valid nonce. For each such action:

1. Your own page sends the request: create the nonce with `wp_create_nonce('action_name')` and send it in `_ajax_nonce` (or `_wpnonce`).
2. No nonce makes sense (public read, cached page, beacon, signed webhook): declare `withoutNonce()` or `nonce: false`. Both can be done before upgrading.
3. A form already has a nonce of another name: `requireNonce('that_name')` or `nonce: 'that_name'`.

While migrating, the `pollora/ajax/nonce_by_default` filter turns the default off for the actions that did not choose. Each one is then reported through `_doing_it_wrong()`:

```php
add_filter('pollora/ajax/nonce_by_default', '__return_false');
```

Add it before the actions are registered. It receives the action name as its second argument, only a strict `false` turns the check off, and it never overrides `requireNonce()` or `withoutNonce()`. It reopens those actions to cross-site request forgery: use it to migrate, not as a setting. If your code already hooks this filter and returns `false` for some actions, those stay unchecked in 2.0.

### Verify it yourself

The handler can still check the nonce on its own, with `check_ajax_referer()`:

```php
Ajax::listen('my_action', function () {
    check_ajax_referer('my_action'); // ends the request with a 403 if the nonce is missing or invalid

    wp_send_json_success(['message' => 'It works!']);
})->withoutNonce(); // already checked here
```

Use an action name of your own for the nonce (`wp_create_nonce('my_action')`), not `wp_rest`, which is the REST API's.

Checking the capability by hand is no longer needed: [`requireCapability()`](#capabilities-who-may-do-it) does it before your handler, and in the same order.

### Other endpoints

Laravel's CSRF middleware plays no part in AJAX. `admin-ajax.php` is a file under `wp-admin/` run by PHP directly: the Laravel HTTP kernel never handles the request, so no Laravel middleware runs. What protects each kind of endpoint:

| Endpoint | Laravel CSRF (`PreventRequestForgery`) | Who verifies | How |
|---|---|---|---|
| admin-ajax (`Ajax::listen()`, `#[Ajax]`) | Not involved | Pollora, by default (unless `withoutNonce()` / `nonce: false`), plus `requireCapability()` / `capability:` for the user's right; otherwise your handler | `check_ajax_referer('my_action')` |
| REST (`/wp-json/...`, `#[WpRestRoute]`) | Not involved | WordPress REST server | The `X-WP-Nonce` header (or `_wpnonce`), action `wp_rest`, for cookie authentication. With no nonce the request is treated as unauthenticated; the route's `permission_callback` then decides |
| `Route::wp()` | Runs, because the routes sit in the `web` group, unless `wordpress.sessions` is `false` | Laravel | `@csrf` / `X-CSRF-TOKEN` on `POST`, `PUT`, `PATCH`, `DELETE` ([Middleware](middleware.md)) |
| `/api/...` (`routes/api.php` of a theme, plugin or module) | Not applied: no session, no `web` group | Nobody by default | Add your own authentication middleware to the routes ([Routing](routing.md)) |

The skeleton's `bootstrap/app.php` also lists `wp-admin/*`, `wp-login.php`, `wp-cron.php` and `wp-json/*` in the CSRF exceptions of the `web` group.

## Capabilities: who may do it

A valid nonce tells you the request came from your page, not that the user may do what it asks. Without a capability check, any logged-in user who has a nonce can run the action. `requireCapability()` says who may:

```php
// Nonce first (by default), then the capability, then the callback
Ajax::listen('save_note', function () {
    wp_send_json_success(['message' => 'Saved!']);
})->requireCapability('edit_posts');
```

The attribute says the same: `#[Ajax('save_note', capability: 'edit_posts')]`.

This needs `pollora/ajax` 1.3 and `pollora/attributes` 0.1.4, which the framework requires.

The capability is a WordPress one (`edit_posts`, `manage_options`, or one of your own) read with `current_user_can()`. The checks run in this order, and the first one that fails ends the request:

1. the nonce (`403`, `pollora_ajax_invalid_nonce`);
2. the capability (`403`, `pollora_ajax_forbidden`);
3. your handler.

A logged-in user who lacks the capability receives:

```json
{ "success": false, "data": { "code": "pollora_ajax_forbidden", "message": "You are not allowed to do this." } }
```

A request from a guest to an action that requires a capability receives a `401`:

```json
{ "success": false, "data": { "code": "pollora_ajax_unauthenticated", "message": "You must be logged in to do this." } }
```

The capability is optional and there is no default one: which one makes sense depends on what the action does (`read` would let every account through, `edit_posts` would shut out a customer saving their own data).

### A capability and guests do not go together

A guest is not logged in, so it has no capability. If an action open to guests (`forGuestUsers()` or `forAllUsers()`) requires one, the guest side is contradictory: the request is refused with a `401` and the handler **never runs**, and WordPress is told through `_doing_it_wrong()` (visible with `WP_DEBUG`, skipped while an AJAX request is answered). Restrict the action to logged-in users, or drop the capability.

### An empty capability fails closed

An `#[Ajax]` attribute with a blank capability (`capability: ''`) is a mistake, and Pollora does not guess what you meant: the action is **not registered**, and the error is logged. It is never registered open to everyone. The facade throws an exception for the same mistake.

### A nonce and a capability

An action that requires a capability still verifies its nonce: the two checks are independent. Keep both on every action that changes data: the nonce keeps another site from driving the user's browser, the capability keeps a user who should not be there out.

## Frontend JavaScript

Send AJAX requests from JavaScript using the `ajaxurl` global provided by WordPress:

```javascript
jQuery.post(ajaxurl, {
    action: 'my_action',
    _ajax_nonce: myApp.nonce,
    data: 'some data'
}, function (response) {
    if (response.success) {
        console.log(response.data);
    }
});
```

Localize your script with a nonce created for the same action the handler checks:

```php
use Pollora\Support\Facades\Asset;

Asset::add('my-script', 'assets/js/app.js')
    ->toFrontend()
    ->localize('myApp', [
        'nonce' => wp_create_nonce('my_action'),
    ]);
```
