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
use Pollora\Ajax\AjaxAccess;

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

    // Guest users only
    #[Ajax('track_visit', access: AjaxAccess::GUEST)]
    public function trackVisit(): void
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

### AjaxAccess Enum

| Value | WordPress Hook | Audience |
|---|---|---|
| `AjaxAccess::LOGGED` | `wp_ajax_{action}` | Logged-in users only (default) |
| `AjaxAccess::ALL` | `wp_ajax_{action}` + `wp_ajax_nopriv_{action}` | Everyone |
| `AjaxAccess::GUEST` | `wp_ajax_nopriv_{action}` | Guests only |

## Security Model

> **Why secure-by-default?** Exposing `wp_ajax_nopriv_*` allows any unauthenticated visitor to call the endpoint. This should be a conscious decision, not an implicit default. Both the facade and attribute API require explicit opt-in to expose an endpoint publicly.

## Frontend JavaScript

Send AJAX requests from JavaScript using the `ajaxurl` global provided by WordPress:

```javascript
jQuery.post(ajaxurl, {
    action: 'my_action',
    _wpnonce: myApp.nonce,
    data: 'some data'
}, function (response) {
    if (response.success) {
        console.log(response.data);
    }
});
```

Make sure to localize your script with the nonce:

```php
use Pollora\Support\Facades\Asset;

Asset::add('my-script', 'assets/js/app.js')
    ->toFrontend()
    ->localize('myApp', [
        'nonce' => wp_create_nonce('wp_rest'),
    ]);
```
