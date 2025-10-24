# WordPress Logging

The Pollora framework provides a WordPress error logging system. This system captures WordPress errors, warnings, and deprecated function usage, logging them through Laravel's logging infrastructure while preventing them from breaking your application.

## Overview

The WordPress logging module intercepts and logs various WordPress error types:
- **Doing it wrong**: When WordPress functions are used incorrectly
- **Deprecated functions**: When deprecated WordPress functions are called
- **Deprecated arguments**: When functions are called with deprecated parameters

All errors are logged to a dedicated `wordpress` log channel while preventing PHP errors from being displayed to users.

## Architecture

The logging system follows a strict DDD architecture with clear separation of concerns:

### Domain Layer (`src/Logging/Domain/`)

#### Contracts
- **`WordPressErrorLoggerInterface`**: Defines the contract for logging WordPress errors
- **`WordPressHookRegistrarInterface`**: Defines the contract for registering WordPress hooks

#### Models
- **`WordPressError`**: Domain entity representing a WordPress error with contextual information
- **`WordPressErrorType`**: Enum defining the three types of WordPress errors and their log levels

#### Services
- **`WordPressErrorHandler`**: Pure domain service handling error processing logic

### Application Layer (`src/Logging/Application/`)

#### Services
- **`WordPressErrorLoggingService`**: Orchestrates error handling, builds context, and coordinates between domain and infrastructure layers

### Infrastructure Layer (`src/Logging/Infrastructure/`)

#### Adapters
- **`LaravelWordPressErrorLogger`**: Adapts domain logging interface to Laravel's logging system

#### Services
- **`WordPressHookRegistrar`**: Registers WordPress hooks and filters using the container pattern

#### Providers
- **`LoggingServiceProvider`**: Configures dependency injection and initializes the logging system

### Support Layer (`src/Support/`)

#### Facades
- **`WordPressError`**: Laravel facade providing convenient static methods for logging WordPress errors

## Usage

The logging system is automatically registered through the main `PolloraServiceProvider`. No manual configuration is required for basic usage.

### Laravel Facade (Recommended)

The simplest way to log WordPress errors in Laravel is using the `WordPressError` facade:

```php
use Pollora\Support\Facades\WordPressError;

// Log a "doing it wrong" error
WordPressError::doingItWrong(
    'wp_enqueue_script',
    'Scripts should be enqueued in wp_enqueue_scripts action',
    '6.0.0'
);

// Log a deprecated function usage
WordPressError::deprecatedFunction(
    'mysql_query',
    'wpdb::prepare()',
    '3.9.0'
);

// Log a deprecated argument usage
WordPressError::deprecatedArgument(
    'get_posts',
    'The "numberposts" parameter is deprecated. Use "posts_per_page" instead.',
    '4.4.0'
);
```

### Error Types and Log Levels

```php
// Domain model showing error types and their log levels
enum WordPressErrorType: string
{
    case DOING_IT_WRONG = 'doing_it_wrong';      // Logs as 'warning'
    case DEPRECATED_FUNCTION = 'deprecated_function'; // Logs as 'info'
    case DEPRECATED_ARGUMENT = 'deprecated_argument'; // Logs as 'info'
}
```

### Creating WordPress Errors

The `WordPressError` model provides static factory methods:

```php
// For "doing it wrong" errors
$error = WordPressError::doingItWrong(
    function: 'wp_enqueue_script',
    message: 'Scripts should be enqueued in wp_enqueue_scripts action',
    version: '6.0.0',
    context: ['url' => 'https://example.com']
);

// For deprecated functions
$error = WordPressError::deprecatedFunction(
    function: 'mysql_query',
    replacement: 'wpdb::prepare()',
    version: '3.9.0',
    context: ['file' => 'plugins/old-plugin/plugin.php']
);

// For deprecated arguments
$error = WordPressError::deprecatedArgument(
    function: 'get_posts',
    message: 'The "numberposts" parameter is deprecated. Use "posts_per_page" instead.',
    version: '4.4.0',
    context: ['caller' => 'my_theme_function()']
);
```

## Configuration

### Environment Variables

Configure the WordPress logging channel through environment variables:

```env
# Log level for WordPress errors (debug, info, notice, warning, error, critical, alert, emergency)
WORDPRESS_LOG_LEVEL=debug

# Number of days to retain WordPress logs
WORDPRESS_LOG_DAYS=7
```

### Custom Configuration

Override the default logging configuration in your `config/logging.php`:

```php
'channels' => [
    'wordpress' => [
        'driver' => 'single',
        'path' => storage_path('logs/wordpress.log'),
        'level' => env('WORDPRESS_LOG_LEVEL', 'debug'),
        'days' => env('WORDPRESS_LOG_DAYS', 7),
        'replace_placeholders' => true,
    ],
],
```

### Advanced Configuration

For more complex setups, you can configure multiple channels or use different drivers:

```php
'channels' => [
    'wordpress' => [
        'driver' => 'stack',
        'channels' => ['wordpress-file', 'wordpress-slack'],
    ],
    
    'wordpress-file' => [
        'driver' => 'daily',
        'path' => storage_path('logs/wordpress.log'),
        'level' => 'info',
        'days' => 14,
    ],
    
    'wordpress-slack' => [
        'driver' => 'slack',
        'url' => env('SLACK_WEBHOOK_URL'),
        'username' => 'WordPress Logger',
        'emoji' => ':warning:',
        'level' => 'warning',
    ],
],
```

## Context Information

The logging system automatically includes contextual information:

### Standard Context
- **type**: Error type (`doing_it_wrong`, `deprecated_function`, `deprecated_argument`)
- **function**: Name of the WordPress function
- **version**: WordPress version where the issue was introduced
- **message**: Error description (HTML tags stripped)
- **url**: Current request URL
- **method**: HTTP method
- **ip**: Client IP address

### Development Context
In local environments, additional debugging information is included:
- **backtrace**: Clean stack trace (limited to 10 frames, excluding internal logging calls)

### Example Log Entry

```json
{
    "message": "WordPress: wp_enqueue_script called incorrectly",
    "context": {
        "type": "doing_it_wrong",
        "function": "wp_enqueue_script",
        "version": "6.0.0",
        "message": "Scripts should be enqueued in wp_enqueue_scripts action",
        "url": "https://example.com/admin",
        "method": "GET",
        "ip": "192.168.1.100",
        "backtrace": [
            "#0 MyTheme\\Services\\AssetService::enqueueScripts() in /themes/mytheme/app/Services/AssetService.php:45",
            "#1 MyTheme\\Providers\\ThemeServiceProvider::boot() in /themes/mytheme/app/Providers/ThemeServiceProvider.php:23"
        ]
    },
    "level": "warning",
    "level_name": "WARNING",
    "channel": "wordpress",
    "datetime": "2024-01-15T10:30:45.123456+00:00"
}
```

## Extending the System

### Custom Error Logger

Implement your own logger by creating a class that implements `WordPressErrorLoggerInterface`:

```php
use Pollora\Logging\Domain\Contracts\WordPressErrorLoggerInterface;
use Pollora\Logging\Domain\Models\WordPressError;

class CustomWordPressErrorLogger implements WordPressErrorLoggerInterface
{
    public function logError(WordPressError $error): void
    {
        // Custom logging implementation
        $this->sendToExternalService($error);
    }
}
```

Then bind it in your service provider:

```php
$this->app->bind(
    WordPressErrorLoggerInterface::class,
    CustomWordPressErrorLogger::class
);
```

### Custom Hook Registration

Implement custom hook registration by implementing `WordPressHookRegistrarInterface`:

```php
use Pollora\Logging\Domain\Contracts\WordPressHookRegistrarInterface;

class CustomWordPressHookRegistrar implements WordPressHookRegistrarInterface
{
    public function registerErrorHandlers(): void
    {
        // Custom hook registration logic
        add_action('doing_it_wrong_run', [$this, 'handleError'], 10, 3);
        // ... register other hooks
    }
}
```

## Testing

The DDD architecture makes the logging system highly testable:

### Testing Domain Logic

```php
use Pollora\Logging\Domain\Models\WordPressError;
use Pollora\Logging\Domain\Models\WordPressErrorType;

class WordPressErrorTest extends TestCase
{
    public function test_doing_it_wrong_error_creation(): void
    {
        $error = WordPressError::doingItWrong(
            'wp_enqueue_script',
            'Invalid usage',
            '6.0.0'
        );
        
        $this->assertEquals(WordPressErrorType::DOING_IT_WRONG, $error->type);
        $this->assertEquals('warning', $error->getLogLevel());
        $this->assertEquals('WordPress: wp_enqueue_script called incorrectly', $error->getLogMessage());
    }
}
```

### Testing Application Services

```php
use Pollora\Logging\Application\Services\WordPressErrorLoggingService;
use Pollora\Logging\Domain\Services\WordPressErrorHandler;

class WordPressErrorLoggingServiceTest extends TestCase
{
    public function test_handles_doing_it_wrong(): void
    {
        $mockHandler = $this->mock(WordPressErrorHandler::class);
        $mockHandler->shouldReceive('handleDoingItWrong')
            ->once()
            ->with('wp_enqueue_script', 'Invalid usage', '6.0.0', Mockery::type('array'));
            
        $service = new WordPressErrorLoggingService(
            $mockHandler,
            app(),
            request()
        );
        
        $service->handleDoingItWrong('wp_enqueue_script', 'Invalid usage', '6.0.0');
    }
}
```

### Testing with Facade

```php
use Pollora\Support\Facades\WordPressError;
use Pollora\Logging\Application\Services\WordPressErrorLoggingService;

class WordPressErrorFacadeTest extends TestCase
{
    public function test_facade_logs_doing_it_wrong(): void
    {
        $mockService = $this->mock(WordPressErrorLoggingService::class);
        $mockService->shouldReceive('handleDoingItWrong')
            ->once()
            ->with('wp_enqueue_script', 'Invalid usage', '6.0.0');
            
        $this->app->instance(WordPressErrorLoggingService::class, $mockService);
        
        WordPressError::doingItWrong('wp_enqueue_script', 'Invalid usage', '6.0.0');
    }
}
```

## Performance Considerations

### Lazy Hook Registration
The system uses Laravel's container to resolve dependencies only when WordPress hooks are triggered, avoiding circular dependencies and reducing memory usage.

### Backtrace Limitations
- Backtraces are only generated in local environments
- Limited to 10 stack frames to prevent memory issues
- Arguments are excluded from backtraces for performance

### Log Rotation
Configure appropriate log rotation to prevent disk space issues:

```php
'wordpress' => [
    'driver' => 'daily',
    'path' => storage_path('logs/wordpress.log'),
    'level' => 'info',
    'days' => 7, // Keep logs for 7 days
],
```

## Troubleshooting

### Debugging

Enable debug mode to see backtraces in logs:

```env
APP_ENV=local
WORDPRESS_LOG_LEVEL=debug
```

### Log File Locations

Default log file locations:
- **Single channel**: `storage/logs/wordpress.log`
- **Daily rotation**: `storage/logs/wordpress-YYYY-MM-DD.log`
- **Laravel default**: `storage/logs/laravel.log` (if fallback occurs)

## Best Practices

1. **Use appropriate log levels**: Set `WORDPRESS_LOG_LEVEL=info` in production to avoid debug noise
2. **Monitor log size**: Implement log rotation in production environments
3. **Custom context**: Add relevant context when manually creating WordPress errors
4. **Error handling**: The system prevents WordPress errors from breaking your application, but fix underlying issues
5. **Performance**: Avoid logging in tight loops; the system is designed for occasional WordPress errors, not high-frequency logging

## Integration with Monitoring

The WordPress logging system integrates seamlessly with Laravel's logging infrastructure, making it compatible with:

- **Laravel Telescope**: View WordPress errors in the logs section
- **Flare**: Automatic error reporting and grouping
- **Bugsnag/Sentry**: External error monitoring services
- **ELK Stack**: Elasticsearch, Logstash, and Kibana for log analysis
- **Custom dashboards**: Parse structured log data for monitoring dashboards
