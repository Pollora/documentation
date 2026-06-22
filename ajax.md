# AJAX

The `Ajax` facade simplifies the management of WordPress AJAX calls by providing a fluent, chainable API.

## Basic Usage

Use the `listen()` method to register an AJAX handler:

```php
use Pollora\Support\Facades\Ajax;

Ajax::listen('my_action', function () {
    wp_send_json_success(['message' => 'It works!']);
});
```

By default, this registers `wp_ajax_my_action` **only** — the handler is restricted to **logged-in users** for security. Unauthenticated users cannot reach it.

## Security Model

AJAX endpoints are **secure by default**: `listen()` only registers the `wp_ajax_*` hook (authenticated users). You must explicitly opt in to expose an endpoint to unauthenticated visitors.

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

> **Why?** Exposing `wp_ajax_nopriv_*` allows any unauthenticated visitor to call the endpoint. This should be a conscious decision, not an implicit default.

## Using a Controller Method

You can reference a controller instead of a closure:

```php
Ajax::listen('load_more_posts', [PostController::class, 'loadMore']);
```

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

## How It Works

When you call `Ajax::listen()`, Pollora registers WordPress action hooks:

| Method | WordPress Hook | Audience |
|---|---|---|
| Default | `wp_ajax_{action}` | Logged-in users only |
| `forAllUsers()` | `wp_ajax_{action}` + `wp_ajax_nopriv_{action}` | Everyone |
| `forLoggedUsers()` | `wp_ajax_{action}` | Logged-in users only |
| `forGuestUsers()` | `wp_ajax_nopriv_{action}` | Guests only |

The callback receives the standard WordPress AJAX context. Use `wp_send_json_success()` / `wp_send_json_error()` to send responses.
