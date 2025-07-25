# Options Management

## Overview

The Option module provides a Laravel-style interface for managing WordPress options through a clean facade pattern. It bridges WordPress's native option system with Laravel's modern architecture patterns.

## Quick Start

### Basic Usage

```php
use Pollora\Support\Facades\Option;

// Set an option
Option::set('site_theme_color', '#3498db');

// Get an option with default
$color = Option::get('site_theme_color', '#ffffff');

// Update existing option
Option::update('site_title', 'My WordPress Site');

// Check if option exists
if (Option::exists('custom_setting')) {
    // Option exists
}

// Delete an option
Option::delete('old_setting');
```

### Service Injection

For better testability and dependency management, inject the service directly:

```php
use Pollora\Option\Application\Services\OptionService;

class ThemeController
{
    public function __construct(
        private readonly OptionService $optionService
    ) {}
    
    public function updateThemeSettings(array $settings): void
    {
        foreach ($settings as $key => $value) {
            $this->optionService->set($key, $value);
        }
    }
}
```

## API Reference

### Option Facade Methods

#### `get(string $key, mixed $default = null): mixed`

Retrieves an option value. Returns the default if the option doesn't exist.

```php
// Get with string default
$siteName = Option::get('site_name', 'Default Site');

// Get with array default
$themeOptions = Option::get('theme_options', [
    'color' => '#ffffff',
    'layout' => 'sidebar'
]);

// Get with null default (explicit)
$customOption = Option::get('custom_option', null);
```

#### `set(string $key, mixed $value): bool`

Creates a new option or updates an existing one. Returns `true` on success.

```php
// Set string value
Option::set('company_name', 'Acme Corp');

// Set array value (automatically serialized)
Option::set('contact_info', [
    'email' => 'contact@example.com',
    'phone' => '+1234567890'
]);

// Set boolean value
Option::set('maintenance_mode', true);

// Set object value
Option::set('api_config', new ApiConfiguration());
```

#### `update(string $key, mixed $value): bool`

Updates an existing option. Functionally identical to `set()` but semantically clearer when updating.

```php
// Update existing setting
Option::update('posts_per_page', 15);

// Update complex configuration
$config = Option::get('email_settings', []);
$config['smtp_host'] = 'new-smtp.example.com';
Option::update('email_settings', $config);
```

#### `delete(string $key): bool`

Removes an option from the database. Returns `true` if deleted or didn't exist.

```php
// Delete single option
Option::delete('temporary_setting');

// Delete multiple options
$oldSettings = ['old_theme', 'deprecated_option', 'unused_config'];
foreach ($oldSettings as $setting) {
    Option::delete($setting);
}
```

#### `exists(string $key): bool`

Checks if an option exists, regardless of its value (including `false`, `0`, empty string).

```php
// Check existence before processing
if (Option::exists('premium_features')) {
    $features = Option::get('premium_features');
    $this->enablePremiumFeatures($features);
}

// Conditional setting
if (! Option::exists('initial_setup_complete')) {
    $this->runInitialSetup();
    Option::set('initial_setup_complete', true);
}
```

#### `forget(string $key): bool`

Alias for `delete()`. Provides Laravel-style naming consistency.

```php
// Laravel-style option removal
Option::forget('cached_data');
```

## Advanced Usage

### Type Safety and Validation

The Option module automatically handles type conversion and validation:

```php
// Automatic serialization for complex types
Option::set('user_preferences', [
    'theme' => 'dark',
    'notifications' => true,
    'language' => 'en'
]);

// Type preservation
Option::set('is_enabled', false);
$isEnabled = Option::get('is_enabled'); // Returns boolean false, not string

// Null handling
Option::set('optional_setting', null);
$value = Option::get('optional_setting'); // Returns null, not false
```

### Working with Objects

```php
use Pollora\Option\Domain\Models\Option as OptionModel;

// Create option object for advanced manipulation
$option = new OptionModel('api_credentials', [
    'key' => 'abc123',
    'secret' => 'xyz789'
], autoload: false); // Don't autoload sensitive data

// Value objects are immutable
$updatedOption = $option->withValue([
    'key' => 'new123',
    'secret' => 'new789'
]);
```

### Batch Operations

```php
// Batch update with validation
$settings = [
    'site_title' => 'New Site Title',
    'site_description' => 'A great WordPress site',
    'posts_per_page' => 10
];

foreach ($settings as $key => $value) {
    try {
        Option::update($key, $value);
    } catch (InvalidOptionException $e) {
        Log::warning("Failed to update option {$key}: {$e->getMessage()}");
    }
}
```

### Configuration Management

```php
// Theme configuration pattern
class ThemeConfigManager
{
    private const CONFIG_KEY = 'theme_config';
    
    public function getConfig(): array
    {
        return Option::get(self::CONFIG_KEY, $this->getDefaultConfig());
    }
    
    public function updateConfig(array $updates): void
    {
        $config = $this->getConfig();
        $merged = array_merge($config, $updates);
        Option::update(self::CONFIG_KEY, $merged);
    }
    
    private function getDefaultConfig(): array
    {
        return [
            'layout' => 'default',
            'sidebar' => true,
            'color_scheme' => 'light'
        ];
    }
}
```

## Integration Patterns

### Service Provider Registration

The Option module is automatically registered via `OptionServiceProvider`. For custom bindings:

```php
// In your theme's service provider
public function register(): void
{
    $this->app->singleton(CustomOptionService::class, function ($app) {
        return new CustomOptionService(
            $app->make(OptionService::class),
            $app->make('config')
        );
    });
}
```

### Event Integration

```php
use Pollora\Events\OptionUpdated;

// Listen for option changes
Event::listen(OptionUpdated::class, function (OptionUpdated $event) {
    if ($event->key === 'site_theme') {
        Cache::forget('compiled_css');
    }
});

// Custom option service with events
class EventfulOptionService
{
    public function __construct(
        private readonly OptionService $optionService,
        private readonly EventDispatcher $events
    ) {}
    
    public function set(string $key, mixed $value): bool
    {
        $result = $this->optionService->set($key, $value);
        
        if ($result) {
            $this->events->dispatch(new OptionUpdated($key, $value));
        }
        
        return $result;
    }
}
```

### Caching Strategy

```php
// Cached option service for performance
class CachedOptionService
{
    private const CACHE_TTL = 3600; // 1 hour
    
    public function __construct(
        private readonly OptionService $optionService,
        private readonly CacheManager $cache
    ) {}
    
    public function get(string $key, mixed $default = null): mixed
    {
        $cacheKey = "option.{$key}";
        
        return $this->cache->remember($cacheKey, self::CACHE_TTL, function () use ($key, $default) {
            return $this->optionService->get($key, $default);
        });
    }
    
    public function set(string $key, mixed $value): bool
    {
        $result = $this->optionService->set($key, $value);
        
        if ($result) {
            $this->cache->forget("option.{$key}");
        }
        
        return $result;
    }
}
```

## Error Handling

### Exception Types

```php
use Pollora\Option\Domain\Exceptions\OptionNotFoundException;
use Pollora\Option\Domain\Exceptions\InvalidOptionException;

// Handle missing required options
try {
    $apiKey = Option::get('required_api_key');
    if ($apiKey === null) {
        throw new OptionNotFoundException('API key is required');
    }
} catch (OptionNotFoundException $e) {
    Log::error('Configuration error: ' . $e->getMessage());
    return response()->json(['error' => 'Service unavailable'], 503);
}

// Handle invalid option data
try {
    Option::set('', 'invalid-empty-key');
} catch (InvalidOptionException $e) {
    // Option key cannot be empty
}
```

### Graceful Degradation

```php
// Safe option retrieval with fallbacks
class ConfigService
{
    public function getEmailConfig(): array
    {
        try {
            return Option::get('email_config', $this->getDefaultEmailConfig());
        } catch (Exception $e) {
            Log::warning('Failed to load email config', ['error' => $e->getMessage()]);
            return $this->getDefaultEmailConfig();
        }
    }
    
    private function getDefaultEmailConfig(): array
    {
        return [
            'driver' => 'smtp',
            'host' => env('MAIL_HOST', 'localhost'),
            'port' => env('MAIL_PORT', 587)
        ];
    }
}
```

## Testing

### Unit Testing

```php
use Pollora\Option\Application\Services\OptionService;
use Pollora\Option\Domain\Contracts\OptionRepositoryInterface;

class OptionServiceTest extends TestCase
{
    private OptionService $service;
    private OptionRepositoryInterface $repository;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->repository = Mockery::mock(OptionRepositoryInterface::class);
        $this->service = new OptionService($this->repository);
    }
    
    public function test_can_get_option_with_default(): void
    {
        $this->repository
            ->shouldReceive('get')
            ->with('test_key')
            ->once()
            ->andReturn(null);
            
        $result = $this->service->get('test_key', 'default_value');
        
        $this->assertEquals('default_value', $result);
    }
}
```

### Feature Testing

```php
class OptionIntegrationTest extends TestCase
{
    public function test_option_crud_operations(): void
    {
        // Create
        $result = Option::set('test_option', 'test_value');
        $this->assertTrue($result);
        
        // Read
        $value = Option::get('test_option');
        $this->assertEquals('test_value', $value);
        
        // Update
        $result = Option::update('test_option', 'updated_value');
        $this->assertTrue($result);
        
        $value = Option::get('test_option');
        $this->assertEquals('updated_value', $value);
        
        // Delete
        $result = Option::delete('test_option');
        $this->assertTrue($result);
        
        $this->assertFalse(Option::exists('test_option'));
    }
}
```

## Performance Considerations

### Best Practices

1. **Use specific keys** - Prefer targeted gets over broad queries
2. **Batch operations** - Group related option updates together
3. **Cache frequently accessed options** - Use Laravel's cache for hot options
4. **Consider autoload** - Set `autoload: false` for large or rarely accessed options
5. **Use meaningful defaults** - Always provide sensible fallback values

### WordPress Compatibility

The Option module maintains full compatibility with WordPress's native option system:

- All options are stored in `wp_options` table
- Serialization matches WordPress conventions
- Autoload behavior preserved
- Third-party plugins can access options normally

## Migration from WordPress Native

### Before (WordPress Native)
```php
// Old WordPress way
$theme_options = get_option('theme_options', []);
$theme_options['color'] = '#ff0000';
update_option('theme_options', $theme_options);

if (get_option('feature_enabled') === 'yes') {
    // Enable feature
}

delete_option('old_setting');
```

### After (Pollora Option)
```php
// New Pollora way
$themeOptions = Option::get('theme_options', []);
$themeOptions['color'] = '#ff0000';
Option::update('theme_options', $themeOptions);

if (Option::get('feature_enabled', false)) {
    // Enable feature
}

Option::delete('old_setting');
```

The Pollora Option module provides a more expressive, type-safe, and testable interface while maintaining complete WordPress compatibility.
