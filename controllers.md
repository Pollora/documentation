# Controllers

Controllers handle the logic behind user requests in your Pollora application. They work the same way as standard [Laravel controllers](https://laravel.com/docs/13.x/controllers) with additional WordPress integration.

## WordPress Controllers

Pollora includes a `FrontendController` that handles the WordPress template hierarchy fallback. For custom logic, create your own controllers and use them with WordPress routes.

### Using Controllers with WordPress Routes

```php
use App\Http\Controllers\BlogController;

Route::wp('home', [BlogController::class, 'index']);
Route::wp('single', [BlogController::class, 'show']);
Route::wp('page', [BlogController::class, 'show']);
```

### Accessing WordPress Objects

WordPress objects are automatically injected via type hints thanks to the `WordPressBindings` middleware:

```php
namespace App\Http\Controllers;

class BlogController extends Controller
{
    public function show(\WP_Post $post, \WP_Query $query)
    {
        return view('post', [
            'post' => $post,
            'query' => $query,
        ]);
    }
}
```

You can also access WordPress data through helper functions and Sage Directives in your Blade views without passing them from the controller.

### Invokable Controllers

For simple routes, use single-action controllers:

```php
namespace App\Http\Controllers;

class HomeController extends Controller
{
    public function __invoke()
    {
        return view('home');
    }
}
```

```php
Route::wp('front', HomeController::class);
```

## Creating Controllers

```bash
php artisan make:controller BlogController
```

## Dependency Injection

Laravel's service container injects dependencies into controller constructors and methods:

```php
namespace App\Http\Controllers;

use App\Services\PostService;

class BlogController extends Controller
{
    public function __construct(
        private readonly PostService $postService
    ) {}

    public function index()
    {
        $posts = $this->postService->getLatest();

        return view('blog.index', compact('posts'));
    }
}
```

For general controller concepts (resource controllers, middleware, form requests), see the [Laravel documentation](https://laravel.com/docs/13.x/controllers).