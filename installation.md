# Installation

- [Meet Pollora](#meet-pollora)
    - [Why Pollora?](#why-pollora)
- [Your First Pollora Project](#your-first-pollora-project)
- [Initial Configuration](#initial-configuration)
    - [Environment Based Configuration](#environment-based-configuration)
    - [Databases & Migrations](#databases-and-migrations)
    - [Directory Configuration](#directory-configuration)
- [Next Steps](#next-steps)

<a name="meet-pollora"></a>
## Meet Pollora

Pollora is an innovative bridge between Laravel and WordPress, offering an expressive and elegant syntax. It provides a structure and foundation for integrating WordPress into Laravel, allowing you to focus on crafting something astonishing while we handle the intricacies.

With Pollora, you get the best of both worlds - Laravel's development ease and WordPress's content management prowess. Whether you're a novice in web development or a seasoned expert, Pollora is designed to adapt and evolve with your needs. We're excited to witness the masterpieces you'll craft.

<a name="why-pollora"></a>
### Why Pollora?

When it comes to integrating WordPress into a modern web application framework like Laravel, there are options. But we firmly believe Pollora stands out as the optimal choice for a seamless, powerful, and efficient integration.

<a name="your-first-pollora-project"></a>

## Your First Pollora Project

Ensure your local machine has PHP and [Composer](https://getcomposer.org) set up before initiating your first Pollora project (Please refer to the [Laravel Documentation about Docker](https://laravel.com/docs/#laravel-and-docker)).  
For macOS developers, quickly set up PHP and Composer with [Laravel Herd](https://herd.laravel.com). We also suggest [installing Node and NPM](https://nodejs.org).

Once you're set, create a new Pollora project using the Composer `create-project` command:

```shell
composer create-project pollora/pollora example-app
```

Once your project is set up, utilize the Pollora Artisan CLI `serve` command to kickstart Pollora's local development server:

```shell
cd example-app

php artisan serve
```

<a name="initial-configuration"></a>
## Initial Configuration

All configuration files for Pollora are located in the `config` directory. Feel free to familiarize yourself with the options as each one is well-documented.

Out of the box, Pollora requires minimal configuration. However, it might be worthwhile to review the `config/app.php` file and its accompanying documentation to tailor settings like `timezone` and `locale` to your needs.

<a name="environment-based-configuration"></a>
### Environment Based Configuration

Configuration values in Pollora can vary depending on the environment (local vs. production). These values are usually defined in the `.env` file at your application's root.

For security reasons, never commit your `.env` file to source control. Different developers or servers might need different configurations, and exposing sensitive credentials would pose a significant risk.

> **Note**  
> For a deep dive into the `.env` file and environment configurations, peruse the full [configuration documentation](https://laravel.com/docs/13.x/configuration#environment-configuration).

<a name="databases-and-migrations"></a>
### Databases & Migrations

With your Pollora application ready, you might want to store data. By default, the application's `.env` configuration indicates that Pollora will interact with a MySQL database. If you're on macOS, installing MySQL, Postgres, or Redis is a breeze with [DBngin](https://dbngin.com/).

If you'd rather not use MySQL or Postgres, [SQLite](https://www.sqlite.org/index.html) is a lightweight alternative. To begin, create a SQLite database in the `database` directory:

```shell
touch database/database.sqlite
```

Then, adjust your `.env` file to utilize Pollora's `sqlite` database driver. Unneeded configurations can be removed.

Finally, run your application's [database migrations](https://laravel.com/docs/13.x/migrations) to establish your database tables:

```shell
php artisan migrate
```

<a name="directory-configuration"></a>
### Directory Configuration

Always serve Pollora from the root of the "web directory" set for your server. Avoid serving Pollora from a subdirectory as it could inadvertently expose sensitive files.

<a name="next-steps"></a>
## Next Steps

With your Pollora project set up, here are the recommended next steps:

- [Getting Started with Pollora](getting-started.md) — Build your first application
- [Routing](routing.md) — Learn WordPress routing with `Route::wp()`
- [Theming](theming.md) — Create your first theme with Blade templates
- [Post Types](post-types.md) — Register custom post types with PHP attributes

For Laravel-specific concepts, see the [Laravel documentation](https://laravel.com/docs/13.x).
