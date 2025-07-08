# Pollora Framework Documentation

Welcome to the Pollora Framework documentation! Pollora is a modern Laravel & WordPress integration framework that creates a "sweet blend" between the two platforms, allowing developers to use Laravel's architecture patterns while maintaining WordPress functionality and compatibility.

## Table of Contents

### Getting Started
- [Installation](installation.md) - Complete installation guide and setup instructions
- [Getting Started](getting-started.md) - Quick start guide to build your first application
- [IDE Integration](ide.md) - Configure your IDE for optimal development experience

### Core Concepts
- [Modules](modules.md) - Understanding the modular architecture and auto-discovery system
- [Discovery](discovery.md) - How Pollora automatically discovers and registers components
- [Routing](routing.md) - Modern routing system with Laravel-style controllers
- [Controllers](controllers.md) - Building controllers with dependency injection
- [Middleware](middleware.md) - Request/response middleware for authentication and filtering

### WordPress Integration
- [Hooks](hooks.md) - Attribute-based WordPress hooks system
- [Post Types](post-types.md) - Creating custom post types with attributes
- [Taxonomies](taxonomies.md) - Managing custom taxonomies
- [WP REST API](wp-rest-api.md) - Building REST endpoints with WordPress API
- [Authentication](auth.md) - WordPress authentication integration

### Frontend Development
- [Assets](assets.md) - Modern asset management with Vite and Tailwind CSS
- [Theming](theming.md) - Theme development with Blade templates and Laravel patterns
- [Patterns](patterns.md) - WordPress block patterns and categories

### Advanced Features
- [Plugins](plugins.md) - Complete plugin development guide with modern tooling
- [Events & Listeners](events-listeners.md) - Event-driven programming patterns
- [Schedule Events](schedule-events.md) - Cron jobs and scheduled tasks
- [AJAX](ajax.md) - Handling AJAX requests with modern patterns
- [Admin Pages](admin-pages.md) - Creating WordPress admin interfaces
- [Menu](menu.md) - Admin menu management and navigation

### Development Tools
- [Documentation](documentation.md) - Documentation standards and guidelines

## Framework Overview

### What is Pollora?

Pollora bridges the gap between WordPress and Laravel by providing:

- **Laravel-style Architecture**: Service providers, dependency injection, and modern PHP patterns
- **PSR-4 Autoloading**: Automatic class loading with namespace conventions
- **Attribute-driven Configuration**: PHP 8 attributes for declarative programming
- **Modern Asset Management**: Vite integration with hot reload and Tailwind CSS
- **Command-line Tools**: Artisan commands for scaffolding and management
- **Comprehensive Testing**: Built-in testing support with PHPUnit

### Architecture Highlights

#### Domain-Driven Design
The framework follows a strict DDD architecture with clear separation of concerns:
- **Application Layer**: Use cases and orchestration
- **Domain Layer**: Business logic and entities
- **Infrastructure Layer**: External concerns and adapters
- **UI Layer**: Controllers and console commands

#### Automatic Discovery
Pollora automatically discovers and registers:
- Service providers
- Views and templates
- Routes and controllers
- Translations
- Database migrations
- WordPress hooks and filters

#### Modern Development Stack
- **PHP 8.1+**: Latest PHP features and performance
- **Laravel Components**: Illuminate packages for modern development
- **Vite**: Fast build tool with hot module replacement
- **Tailwind CSS**: Utility-first CSS framework
- **Blade Templates**: Powerful templating engine

### Key Features

#### For WordPress Developers
- Familiar WordPress hooks and filters
- Seamless integration with existing WordPress sites
- Compatible with WordPress plugins and themes
- Maintains WordPress coding standards where appropriate

#### For Laravel Developers
- Service container and dependency injection
- Eloquent ORM for database operations
- Artisan commands for common tasks
- Modern PHP patterns and practices

#### For Both
- Automatic component discovery
- Attribute-based configuration
- Modern asset compilation
- Comprehensive testing tools
- Clean, maintainable code structure

## Getting Help

### Documentation Structure
Each documentation file covers specific aspects of the framework:
- **Conceptual guides** explain how things work
- **Step-by-step tutorials** guide you through common tasks
- **Reference materials** provide detailed API information
- **Best practices** share recommended approaches

### Code Examples
All documentation includes practical code examples that you can copy and adapt for your projects. Examples are tested and maintained to ensure they work with the latest version of the framework.

### Community Resources
- **GitHub Repository**: Report issues and contribute to the framework
- **Discussions**: Community Q&A and feature discussions
- **Examples**: Sample projects and implementations

## Contributing

We welcome contributions to both the framework and documentation! Please see our [Contributing Guide](../CONTRIBUTING.md) for details on:
- Code standards and conventions
- Testing requirements
- Pull request process
- Documentation guidelines

## License

The Pollora Framework is open-source software licensed under the MIT License. See the [LICENSE](../LICENSE) file for details.

---

**Ready to get started?** Begin with the [Installation Guide](installation.md) and then follow the [Getting Started](getting-started.md) tutorial to build your first Pollora application.