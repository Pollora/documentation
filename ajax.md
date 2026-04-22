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

This automatically registers WordPress hooks on `wp_ajax_my_action` and `wp_ajax_nopriv_my_action`.

## Targeting Users

By default, AJAX handlers are available to all users. You can restrict access:

```php
// Only logged-in users
Ajax::listen('my_action', function () {
    wp_send_json_success(['user' => wp_get_current_user()->display_name]);
})->forLoggedUsers();

// Only guest users (not logged in)
Ajax::listen('my_action', function () {
    wp_send_json_success(['message' => 'Hello guest!']);
})->forGuestUsers();
```

## Using a Controller Method

You can reference a controller instead of a closure:

```php
Ajax::listen('load_more_posts', [PostController::class, 'loadMore'])
    ->forLoggedUsers();
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

| Method | WordPress Hook |
|---|---|
| Default (both) | `wp_ajax_{action}` + `wp_ajax_nopriv_{action}` |
| `forLoggedUsers()` | `wp_ajax_{action}` only |
| `forGuestUsers()` | `wp_ajax_nopriv_{action}` only |

The callback receives the standard WordPress AJAX context. Use `wp_send_json_success()` / `wp_send_json_error()` to send responses.