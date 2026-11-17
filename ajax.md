# AJAX

Pollora provides two ways to register WordPress AJAX handlers: the **imperative `Ajax` facade** and the **declarative `#[Ajax]` attribute**. Both default to logged-in users only (security-by-default).

## Imperative API (Facade)

Use the `listen()` method to register an AJAX handler inline:

```php
use Pollora\Support\Facades\Ajax;

Ajax::listen('my_action', function () {
    wp_send_json_success(['message' => 'It works!']);
});
```

By default, this registers `wp_ajax_my_action` **only** — the handler is restricted to **logged-in users**. Unauthenticated users cannot reach it.

### Targeting Users

```php
// Default — logged-in users only (wp_ajax_*)
Ajax::listen('my_action', function () {
    wp_send_json_success(['user' => wp_get_current_user()->display_name]);
});

// Explicit — all users, logged-in AND guests (wp_ajax_* + wp_ajax_nopriv_*)
Ajax::listen('public_action', function () {
    wp_send_json_success(['message' => 'Hello everyone!']);
})->forAllUsers();

// Guest users only (wp_ajax_nopriv_*)
Ajax::listen('guest_action', function () {
    wp_send_json_success(['message' => 'Hello guest!']);
})->forGuestUsers();
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
    // Logged-in users only (default)
    #[Ajax('subscribe')]
    public function subscribe(): void
    {
        wp_send_json_success(['message' => 'Subscribed!']);
    }

    // All users (explicit opt-in)
    #[Ajax('load_more', access: AjaxAccess::ALL)]
    public function loadMore(): void
    {
        wp_send_json_success([/* ... */]);
    }

    // Guest users only, no nonce on purpose
    #[Ajax('track_visit', access: AjaxAccess::GUEST, nonce: false)]
    public function trackVisit(): void
    {
        wp_send_json_success([/* ... */]);
    }

    // Logged-in users, the nonce is verified before the method runs
    #[Ajax('save_note', nonce: true)]
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
| `nonce` | `bool\|string\|null` | `null` | `true` verifies a nonce named after the action, a string verifies the nonce of that name, `false` turns it off on purpose, `null` has not chosen yet (see [Nonces and CSRF](#nonces-and-csrf)) |

### AjaxAccess Enum

| Value | WordPress Hook | Audience |
|---|---|---|
| `AjaxAccess::LOGGED` | `wp_ajax_{action}` | Logged-in users only (default) |
| `AjaxAccess::ALL` | `wp_ajax_{action}` + `wp_ajax_nopriv_{action}` | Everyone |
| `AjaxAccess::GUEST` | `wp_ajax_nopriv_{action}` | Guests only |

## Security Model

> **Why secure-by-default?** Exposing `wp_ajax_nopriv_*` allows any unauthenticated visitor to call the endpoint. This should be a conscious decision, not an implicit default. Both the facade and attribute API require explicit opt-in to expose an endpoint publicly.

## Nonces and CSRF

An AJAX request carries the visitor's cookies, so a page of another site can make a logged-in visitor's browser send it. A nonce, a token WordPress creates for a user and an action, tells a request your own page sent from one it did not. Pollora can verify it for you before your handler runs.

### Verify the nonce

```php
// Facade: the nonce is verified before the callback runs
Ajax::listen('save_note', function () {
    wp_send_json_success(['message' => 'Saved!']);
})->requireNonce();

// Another nonce action than the AJAX action name, when a form already has one
Ajax::listen('save_note', $callback)->requireNonce('notes_form');

// No nonce on purpose: a public beacon, a webhook that signs its own requests
Ajax::listen('track_visit', $callback)->forGuestUsers()->withoutNonce();
```

The attribute says the same: `#[Ajax('save_note', nonce: true)]`, `nonce: 'notes_form'` or `nonce: false`.

A request without a valid nonce gets a `403` and the callback does not run. The response is JSON:

```json
{ "success": false, "data": { "code": "pollora_ajax_invalid_nonce", "message": "Invalid or expired security token." } }
```

The nonce is read from `_ajax_nonce`, then `_wpnonce`, in the request, as `check_ajax_referer()` does. By default the nonce action is the AJAX action name; create the nonce for it, for example with `wp_create_nonce('save_note')` (see [Frontend JavaScript](#frontend-javascript)).

### An action that does not choose

An action that never says `requireNonce()` or `withoutNonce()` (`nonce: null` on the attribute) still runs **without a check** in the 1.x versions of `pollora/ajax`, so existing code keeps working. WordPress is told, through `_doing_it_wrong()`, which action does not choose and how to fix it; the notice is visible with `WP_DEBUG` and skipped while an AJAX request is answered. **The next major version verifies a nonce by default.**

To get that behaviour now, on every action that did not choose, return `true` from the `pollora/ajax/nonce_by_default` filter:

```php
add_filter('pollora/ajax/nonce_by_default', '__return_true');
```

A guest's nonce is tied to no user, and a cached page can serve an old one: prefer another protection for an action open to guests, and say `withoutNonce()` when you do.

### Verify it yourself

The handler can still check the nonce on its own, with `check_ajax_referer()`, and should check the user's capability as well when the action changes data:

```php
Ajax::listen('my_action', function () {
    check_ajax_referer('my_action'); // ends the request with a 403 if the nonce is missing or invalid

    wp_send_json_success(['message' => 'It works!']);
})->withoutNonce(); // already checked here
```

Use an action name of your own for the nonce (`wp_create_nonce('my_action')`), not `wp_rest`, which is the REST API's.

### Other endpoints

Laravel's CSRF middleware plays no part in AJAX. `admin-ajax.php` is a file under `wp-admin/` run by PHP directly: the Laravel HTTP kernel never handles the request, so no Laravel middleware runs. What protects each kind of endpoint:

| Endpoint | Laravel CSRF (`PreventRequestForgery`) | Who verifies | How |
|---|---|---|---|
| admin-ajax (`Ajax::listen()`, `#[Ajax]`) | Not involved | Pollora, with `requireNonce()` or `nonce: true`; otherwise your handler | `check_ajax_referer('my_action')` |
| REST (`/wp-json/...`, `#[WpRestRoute]`) | Not involved | WordPress REST server | The `X-WP-Nonce` header (or `_wpnonce`), action `wp_rest`, for cookie authentication. With no nonce the request is treated as unauthenticated; the route's `permission_callback` then decides |
| `Route::wp()` | Runs, because the routes sit in the `web` group, unless `wordpress.sessions` is `false` | Laravel | `@csrf` / `X-CSRF-TOKEN` on `POST`, `PUT`, `PATCH`, `DELETE` ([Middleware](middleware.md)) |
| `/api/...` (`routes/api.php` of a theme, plugin or module) | Not applied: no session, no `web` group | Nobody by default | Add your own authentication middleware to the routes ([Routing](routing.md)) |

The skeleton's `bootstrap/app.php` also lists `wp-admin/*`, `wp-login.php`, `wp-cron.php` and `wp-json/*` in the CSRF exceptions of the `web` group.

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
