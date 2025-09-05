# Environment Management

The Pollora framework provides a comprehensive environment management system that works seamlessly in both WordPress and Laravel contexts. This system follows hexagonal architecture principles to ensure clean separation of concerns and easy testability.

## Overview

The environment management system consists of:

- **Environment Detection**: Automatic detection of the current environment (development, production, staging, etc.)
- **Context-Aware Implementation**: Different detection strategies for WordPress and Laravel environments
- **Service Layer**: High-level service for environment-based decision making
- **Performance Optimization**: Environment-based caching strategies

## Architecture

### Domain Layer

**EnvironmentDetectorInterface** (`src/Application/Domain/Contracts/EnvironmentDetectorInterface.php`)

The core contract defining environment detection capabilities:

```php
interface EnvironmentDetectorInterface
{
    public function getEnvironment(): string;
    public function isProduction(): bool;
    public function isDevelopment(): bool;
    public function isStaging(): bool;
    public function isEnvironment(string $environment): bool;
}
```

### Infrastructure Layer

**WordPressEnvironmentDetector** (`src/Application/Infrastructure/Services/WordPressEnvironmentDetector.php`)

Uses WordPress's native `wp_get_environment_type()` function:

```php
class WordPressEnvironmentDetector implements EnvironmentDetectorInterface
{
    public function getEnvironment(): string
    {
        return wp_get_environment_type() ?: 'production';
    }
    
    public function isDevelopment(): bool
    {
        return $this->getEnvironment() === 'development';
    }
}
```

**LaravelEnvironmentDetector** (`src/Application/Infrastructure/Services/LaravelEnvironmentDetector.php`)

Uses Laravel's application instance for environment detection:

```php
class LaravelEnvironmentDetector implements EnvironmentDetectorInterface
{
    public function __construct(private readonly Application $app) {}
    
    public function getEnvironment(): string
    {
        return $this->app->environment() ?? 'production';
    }
    
    public function isDevelopment(): bool
    {
        return $this->app->environment('local', 'development');
    }
}
```

### Application Layer

**ApplicationEnvironmentService** (`src/Application/Application/Services/ApplicationEnvironmentService.php`)

High-level service providing environment-based utilities:

```php
class ApplicationEnvironmentService
{
    public function __construct(
        private readonly EnvironmentDetectorInterface $detector
    ) {}

    public function shouldEnableDebug(): bool
    {
        return $this->isDevelopment() || $this->isStaging();
    }

    public function shouldEnableCache(): bool
    {
        return $this->isProduction() || $this->isStaging();
    }

    public function getConfigPrefix(): string
    {
        return match ($this->getEnvironment()) {
            'development' => 'dev',
            'staging' => 'stage',
            'production' => 'prod',
            default => 'default',
        };
    }
}
```

## Usage

### Basic Environment Detection

```php
use Pollora\Application\Application\Services\ApplicationEnvironmentService;

class MyController
{
    public function __construct(
        private readonly ApplicationEnvironmentService $environment
    ) {}

    public function index()
    {
        if ($this->environment->isProduction()) {
            // Production-specific logic
            $this->enableCaching();
        }

        if ($this->environment->isDevelopment()) {
            // Development-specific logic
            $this->enableDebugMode();
        }
    }
}
```

### Service Container Resolution

The environment service is automatically registered and can be resolved in multiple ways:

```php
// Via dependency injection
public function __construct(ApplicationEnvironmentService $env) {}

// Via service locator
$environment = app(ApplicationEnvironmentService::class);

// Via alias
$environment = app('pollora.environment');
```

### Environment-Specific Configuration

```php
class ConfigurationService
{
    public function __construct(
        private readonly ApplicationEnvironmentService $environment
    ) {}

    public function getCacheSettings(): array
    {
        $prefix = $this->environment->getConfigPrefix();
        
        return [
            'enabled' => $this->environment->shouldEnableCache(),
            'ttl' => $this->environment->isProduction() ? 3600 : 60,
            'prefix' => $prefix . '_cache_',
        ];
    }
}
```

## Laravel Environment Configuration

Laravel environments are configured through the standard `.env` file:

```bash
APP_ENV=local
APP_DEBUG=true

# Other environment-specific settings
CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync
```

## Performance Integration

The environment system is integrated with performance-critical components like the [Discovery Engine](discovery.md):

```php
class DiscoveryEngine
{
    private function getCacheDriver(): DiscoverCacheDriver
    {
        // Disable cache in development for faster iteration
        if ($this->environment->isDevelopment() || 
            $this->environment->isEnvironment('local')) {
            return new NullDiscoverCacheDriver();
        }

        // Enable cache in production for performance
        return new LaravelDiscoverCacheDriver();
    }
}
```

## Advanced Usage

### Custom Environment Types

```php
// Check for custom environment
if ($environment->isEnvironment('testing')) {
    $this->setupTestDatabase();
}

// Environment-specific feature flags
$features = [
    'debug_toolbar' => $environment->isDevelopment(),
    'query_logging' => !$environment->isProduction(),
    'cache_warmup' => $environment->isProduction(),
];
```

### Environment-Based Service Registration

```php
class CustomServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $environment = $this->app->make(ApplicationEnvironmentService::class);
        
        if ($environment->isDevelopment()) {
            $this->app->register(DevelopmentServiceProvider::class);
        }
        
        if ($environment->isProduction()) {
            $this->app->register(ProductionOptimizationProvider::class);
        }
    }
}
```