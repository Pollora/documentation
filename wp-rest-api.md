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
use Pollora\WpRest\AbstractWpRestRoute;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)'
)]
class DocumentAPI extends AbstractWpRestRoute {}
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
use Pollora\WpRest\AbstractWpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use WP_REST_Request;
use Illuminate\Http\JsonResponse;

class DocumentAPI extends AbstractWpRestRoute
{
    #[Method('GET')]
    public function get(int $documentId): JsonResponse
    {
        return response()->json([
            'success' => true,
            'documentId' => $documentId,
        ]);
    }

    #[Method(['POST', 'DELETE'])]
    public function delete(WP_REST_Request $request, int $documentId): JsonResponse
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
use Pollora\WpRest\AbstractWpRestRoute;
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute\Permissions\IsAdmin;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)',
    permissionCallback: IsAdmin::class
)]
class AdminDocumentAPI extends AbstractWpRestRoute {}
```

### Method-Level Permission
Permissions can also be set for specific HTTP methods:
```php
use Pollora\WpRest\AbstractWpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use WP_REST_Response;
use Pollora\Attributes\WpRestRoute\Permissions\IsAdmin;
use Pollora\Attributes\WpRestRoute\Permissions\IsAuthor;

class AdminDocumentAPI extends AbstractWpRestRoute
{
    #[Method('GET', permissionCallback: IsAdmin::class)]
    public function get(): WP_REST_Response {}

    #[Method('DELETE', permissionCallback: IsAuthor::class)]
    public function delete(): WP_REST_Response {}
}
```

If a method has its own permission callback, it **overrides** the class-level permission.

## Custom Permission Classes

A permission class must implement `Pollora\WpRest\Permission` and define an `allow()` method that returns `true`, `false`, or a `WP_Error`.

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
use Pollora\WpRest\AbstractWpRestRoute;
use Pollora\Attributes\WpRestRoute;
use Pollora\Attributes\WpRestRoute\Method;
use Pollora\Attributes\WpRestRoute\Permissions\IsAdmin;
use WP_REST_Request;
use WP_REST_Response;

#[WpRestRoute(
    namespace: 'app/v2',
    route: 'document/(?P<documentId>\\d+)',
    permissionCallback: IsAdmin::class
)]
class DocumentAPI extends AbstractWpRestRoute
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
