# Auth

The `Auth` facade in Laravel allows you to manage user authentication in your application, with adaptations for WordPress integration. By implementing the `WordPressGuard`, you can handle authentication using the WordPress schema, while still leveraging Laravel's standard features.

## User Authentication

You can use the standard methods of the `Auth` facade to manage user authentication in your Laravel application, with adjustments for the WordPress schema.

### Checking Authentication

```php
if (Auth::check()) {
    // User is authenticated
} else {
    // User is not authenticated
}
```

The `check()` method verifies if the user is currently authenticated via WordPress. It uses the `check()` method you've implemented in the `WordPressGuard` class.

### Retrieving the Authenticated User

```php
$user = Auth::user();

if ($user) {
    // Authenticated user
    echo "Hello, " . $user->display_name;
} else {
    // No authenticated user
}
```

The `user()` method returns the currently authenticated user via WordPress. If no user is authenticated, it returns `null`. This uses the `user()` method you've set up in the `WordPressGuard` class.

### Attempting Authentication via WordPress

```php
if (Auth::attempt(['username' => $username, 'password' => $password, 'remember' => true])) {
    // User authenticated successfully
} else {
    // Authentication failed
}
```

The `attempt()` method tries to authenticate a user using the provided credentials. In this case, the credentials are verified by WordPress, and if successful, the user is automatically authenticated via Laravel. This method uses the `attempt()` method you've implemented in the `WordPressGuard` class.

### Temporary Authentication with `once()`

```php
if (Auth::once(['username' => $username, 'password' => $password])) {
    // User temporarily authenticated successfully
} else {
    // Temporary authentication failed
}
```

The `once()` method works similarly to `attempt()`, but it temporarily authenticates the user only for the duration of the current request. After the request ends, the user won't remain authenticated. This uses the `once()` method in the `WordPressGuard` class.

### Logging in Manually with `login()`

```php
$user = Auth::user();

if ($user) {
    Auth::login($user);
    // User manually logged in successfully
} else {
    // No authenticated user
}
```

The `login()` method allows you to manually authenticate a specific user. You need to pass an instance of a Laravel user (corresponding to a WordPress user) as an argument. This uses the `login()` method in the `WordPressGuard` class.

### Logging in via ID with `loginUsingId()`

```php
if (Auth::loginUsingId($userId)) {
    // User authenticated successfully via ID
} else {
    // Authentication failed via ID
}
```

The `loginUsingId()` method lets you authenticate a user using their ID. It uses the ID to find and authenticate the user, connecting them. This method uses the `loginUsingId()` method you've implemented in the `WordPressGuard` class.

### Temporary Login via ID with `onceUsingId()`

```php
if (Auth::onceUsingId($userId)) {
    // User temporarily authenticated successfully via ID
} else {
    // Temporary authentication failed via ID
}
```

The `onceUsingId()` method is similar to `loginUsingId()`, but it temporarily authenticates the user only for the duration of the current request. This uses the `onceUsingId()` method you've implemented in the `WordPressGuard` class.

## Authorization

What an authenticated user may do — `$user->can('edit_posts')`, the `can:` middleware, `@can` in Blade, roles declared in code — is covered in [Roles & Capabilities](roles.md).
