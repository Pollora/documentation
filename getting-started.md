# Getting Started

Welcome! This guide will help you get started with a working install of Pollora.

First things first, it is recommended that you get yourself familiar with [Laravel](https://laravel.com/docs/10.x/). Pollora is built on top of Laravel to provide a WordPress administration panel on the backend, Laravel knowledge is assumed throughout the documentation.

## Installing Pollora

The recommended (and only supported) way of installing Pollora is by creating a new composer project:

```bash
$ composer create-project pollora/pollora
```

This will pull down a skeleton install of Pollora, you will then need to configure your environment variables. First copy `.env.example` to `.env` and fill out your database details.

Then, generate your application keys [here](https://roots.io/salts.html) and paste them in your environment file, alternatively you can use [wp-cli-dotenv-command](https://github.com/aaemnnosttv/wp-cli-dotenv-command) to automatically do this for you:

```bash
$ wp package install aaemnnosttv/wp-cli-dotenv-command
$ wp dotenv salts regenerate
```

Then generate an application key using `php artisan key:generate`.

Now, when visit your site, you will automatically be prompted to install the WordPress database.

Congratulations, you're ready to use Pollora now!

<div class="alert alert-info" role="alert"><strong>Heads up!</strong> Pollora rids WordPress of frontend responsibilites altogether, this means theme support in WordPress is dropped completely. Don't worry though, any functions you can run in vanilla WordPress you can run in Pollora!</div>
