# Server Configuration

- [Overview](#overview)
- [Document Root](#document-root)
- [Directory Browsing Protection](#directory-browsing-protection)
    - [The Problem](#the-problem)
    - [Apache Configuration](#apache-configuration)
    - [Nginx Configuration](#nginx-configuration)
- [Reverse Proxy & HTTPS](#reverse-proxy-and-https)
    - [How Pollora Handles HTTPS](#how-pollora-handles-https)
    - [TrustProxies Middleware](#trustproxies-middleware)

<a name="overview"></a>
## Overview

Pollora uses a Bedrock-style directory layout where WordPress core, plugins, themes, and uploads all live under the `public/` document root. This differs from a standard WordPress installation and requires specific web server configuration to prevent exposing internal directories.

<a name="document-root"></a>
## Document Root

Your web server's document root must point to the `public/` directory:

```
your-project/
├── app/
├── bootstrap/
├── config/
├── public/          ← document root
│   ├── index.php    ← Laravel front controller
│   ├── wp-config.php
│   ├── cms/         ← WordPress core
│   └── content/     ← wp-content (plugins, themes, uploads)
├── resources/
└── vendor/
```

<a name="directory-browsing-protection"></a>
## Directory Browsing Protection

<a name="the-problem"></a>
### The Problem

Because `content/` (WordPress's `wp-content`) is under the public document root, directories like `/content/plugins/`, `/content/themes/`, and `/content/uploads/` are web-accessible. By default, some of these directories contain `index.php` files with only a comment (`// Silence is golden.`), which causes the web server to return a **blank 200 response** — confirming the directory exists and leaking structural information.

The fix is to ensure your web server **never resolves `DirectoryIndex`** for these paths, and instead routes all non-file requests through the Laravel front controller.

<a name="apache-configuration"></a>
### Apache Configuration

The default `.htaccess` shipped with Pollora includes `Options -Indexes` to disable directory listing. To also prevent blank 200 responses from `index.php` files inside directories, add the following rule **before** the "Send Requests To Front Controller" block:

```apache
<IfModule mod_rewrite.c>
    Options -MultiViews -Indexes

    RewriteEngine On

    # Block direct directory browsing.
    # All directory requests go through the framework (returns 404),
    # except wp-admin which needs DirectoryIndex for its own index.php.
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteCond %{REQUEST_URI} !^/cms/wp-admin
    RewriteCond %{REQUEST_URI} !^/$
    RewriteRule ^ index.php [L]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

This ensures that:
- **Files** (CSS, JS, images, fonts) are served directly by Apache
- **Directories** are routed through the framework, which returns 404
- **`/cms/wp-admin`** is excluded so WordPress admin works normally
- **`/`** (root) is excluded so the front controller's `index.php` is resolved

<a name="nginx-configuration"></a>
### Nginx Configuration

For Nginx, the key is to remove the `$uri/` directive from `try_files`, which prevents Nginx from resolving `DirectoryIndex` for arbitrary directories:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/your-project/public;
    index index.php;

    # Serve static files directly, everything else goes to the framework.
    # Intentionally omit $uri/ to prevent DirectoryIndex resolution
    # for paths like /content/plugins/ or /content/themes/.
    location / {
        try_files $uri /index.php?$query_string;
    }

    # wp-admin needs DirectoryIndex resolution for its own index.php.
    location ^~ /cms/wp-admin {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP handling
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

> **Note**  
> The critical difference from a standard Laravel Nginx config is `try_files $uri /index.php?$query_string;` (without `$uri/`). The `$uri/` directive tells Nginx to look for an `index.php` inside the requested directory — which is exactly what causes the blank 200 responses.

<a name="reverse-proxy-and-https"></a>
## Reverse Proxy & HTTPS

<a name="how-pollora-handles-https"></a>
### How Pollora Handles HTTPS

When deployed behind a reverse proxy (Clever Cloud, Heroku, AWS ALB, Cloudflare, etc.), the backend server often receives requests over HTTP on an internal port (e.g., `127.0.0.1:8080`), while the public-facing URL uses HTTPS.

Pollora derives the HTTPS state from the `APP_URL` environment variable — not from request headers. When `APP_URL` starts with `https://`, Pollora automatically:

1. Forces Laravel's URL generator to use HTTPS (`URL::forceScheme('https')`)
2. Sets `$_SERVER['HTTPS'] = 'on'` so WordPress canonical redirects work correctly
3. Computes `WP_HOME`, `WP_SITEURL`, and `WP_CONTENT_URL` from `APP_URL`

This means you **do not need** to manually configure `X-Forwarded-Proto` header mapping. Just ensure `APP_URL` is set correctly in your `.env`:

```env
APP_URL=https://your-domain.com
```

<a name="trustproxies-middleware"></a>
### TrustProxies Middleware

While Pollora handles HTTPS and WordPress URLs via `APP_URL`, you may still want to configure Laravel's `TrustProxies` middleware for other features that rely on the original client request (client IP, rate limiting, etc.).

In `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(
        at: '*',  // or specific proxy IPs
        headers: Request::HEADER_X_FORWARDED_FOR |
                 Request::HEADER_X_FORWARDED_HOST |
                 Request::HEADER_X_FORWARDED_PORT |
                 Request::HEADER_X_FORWARDED_PROTO |
                 Request::HEADER_X_FORWARDED_AWS_ELB,
    );
})
```

> **Warning**  
> Using `at: '*'` trusts all proxies. In production, restrict this to your reverse proxy's IP range when possible.