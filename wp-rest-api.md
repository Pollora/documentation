# WordPress REST API

Pollora provides a clean and structured way to declare REST API routes for WordPress using PHP attributes. This system enables developers to define endpoints, specify HTTP methods, and implement permission handling in an intuitive way.

## Table of Contents

- [Overview](#overview)
- [Defining Routes](#defining-routes)
- [Registering Methods](#registering-methods)
- [Permission Handling](#permission-handling)
- [Custom Permission Classes](#custom-permission-classes)
- [Example Usage](#example-usage)
- [How It Works](#how-it-works)

## Overview
Pollora's API routing system is based on PHP attributes, eliminating the need to manually register REST API routes in WordPress.

It consists of three key components:
1. **`#[WpRestRoute]`**: Defines the base REST route.
2. **`#[Method]`**: Specifies the HTTP method(s) for a given function.
3. **Permission Classes**: Handles user permissions before executing the request.

## Defining Routes
To create a new API route, use the `#[WpRestRoute]` attribute on a class.

```php
use Pollora\Attributes\WpRestRoute;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)'
)]
class DocumentAPI {}
```

### Route Structure
- **`namespace`**: Defines the base namespace for the route.
- **`route`**: Defines the endpoint pattern (supporting regex parameters).

Once defined, WordPress will recognize the API endpoint:
```http
GET wp-json/app/v2/document/18
```

## Registering Methods

Use the `#[Method]` attribute on methods inside the class to define HTTP methods:

```php
use Pollora\Attributes\WpRestRoute\Method;
use WP_REST_Request;
use WP_REST_Response;

class DocumentAPI
{
    #[Method('GET')]
    public function get(int $documentId): WP_REST_Response
    {
        return new WP_REST_Response([
            'success' => true,
            'documentId' => $documentId,
        ]);
    }

    #[Method(['POST', 'DELETE'])]
    public function delete(WP_REST_Request $request, int $documentId): WP_REST_Response
    {
        return new WP_REST_Response([
            'success' => true,
            'deleted' => $documentId,
        ]);
    }
}
```

### Supported HTTP Methods
- `GET`
- `POST`
- `PUT`
- `DELETE`
- `PATCH`

If an invalid HTTP method is provided, an exception will be thrown during route registration.

## Permission Handling

Pollora allows defining **permissions** at both the class and method level.


### Route-Level Permission

Permissions can be applied globally to all methods within a class:

```php
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute;
use Pollora\WpRest\Permissions\IsAdmin;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)',
    permissionCallback: IsAdmin::class
)]
class AdminDocumentAPI {}
```

### Method-Level Permission
Permissions can also be set for specific HTTP methods:
```php
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use WP_REST_Response;
use Pollora\WpRest\Permissions\IsAdmin;
use Pollora\WpRest\Permissions\IsLoggedIn;

class AdminDocumentAPI
{
    #[Method('GET', permissionCallback: IsLoggedIn::class)]
    public function get(): WP_REST_Response {}

    #[Method('DELETE', permissionCallback: IsAdmin::class)]
    public function delete(): WP_REST_Response {}
}
```

If a method has its own permission callback, it **overrides** the class-level permission.

### Checking a capability

`Can` allows the user when they have a WordPress capability. Unlike `IsAdmin`, it takes arguments, so pass an instance:

```php
use App\Cms\Roles\EventCap;
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use Pollora\WpRest\Permissions\Can;

#[WpRestRoute('app/v1', 'events/(?P<id>\\d+)', permissionCallback: new Can('edit_posts'))]
class EventAPI
{
    #[Method('GET', permissionCallback: new Can(EventCap::ExportAttendees))]
    public function attendees(int $id): array {}

    #[Method('PUT', permissionCallback: new Can('edit_post', parameter: 'id'))]
    public function update(int $id): array {}
}
```

| Parameter | Effect |
|---|---|
| `capability` | A capability, or a case of a [`#[CapabilitySet]`](roles.md#project-capabilities) enum |
| `parameter` | A request parameter whose value is passed with the capability, for a meta capability such as `edit_post` on the post being edited |

A guest is refused with a 401 status, a logged-in user without the capability with a 403. `permissionCallback` accepts an instance of any permission class, yours included.

## Custom Permission Classes

A permission class must implement `Pollora\Attributes\WpRestRoute\Permission` and define an `allow()` method that returns `true`, `false`, or a `WP_Error`.

### Example: Restrict to Administrators
```php
use Pollora\Attributes\WpRestRoute\Permission;
use WP_REST_Request;
use WP_Error;

class IsAdmin implements Permission
{
    public function allow(WP_REST_Request $request): bool|WP_Error
    {
        return current_user_can('manage_options') ?: new WP_Error(
            'rest_forbidden',
            __('You do not have permission to access this endpoint.'),
            ['status' => 403]
        );
    }
}
```

## Example Usage
```php
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use Pollora\WpRest\Permissions\IsAdmin;
use WP_REST_Request;
use WP_REST_Response;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)',
    permissionCallback: IsAdmin::class
)]
class DocumentAPI
{
    #[Method('GET')]
    public function get(int $documentId): WP_REST_Response
    {
        return new WP_REST_Response(['success' => true, 'documentId' => $documentId]);
    }

    #[Method(['DELETE', 'POST'])]
    public function delete(WP_REST_Request $request, int $documentId): WP_REST_Response
    {
        return new WP_REST_Response(['success' => true, 'deleted' => $documentId]);
    }
}
```

## How It Works
1. **Pollora scans attributes** and detects classes annotated with `#[WpRestRoute]`.
2. **It registers API endpoints** dynamically within WordPress.
3. **Methods with `#[Method]` are linked** to the appropriate HTTP method.
4. **Permissions are validated** before executing the request.
