# Media Storage

Pollora can keep the WordPress media library on any Laravel disk — local, S3, R2, Scaleway, MinIO — with two settings: the disk and the public URL. WordPress goes on using its own file functions, so Gutenberg, ACF, Meta Box, WooCommerce and imports follow without any adaptation.

The storage is the package [`pollora/media`](https://github.com/Pollora/media), which the framework requires. The framework keeps the Laravel side: the connection to your disks, `config/media.php`, the Artisan commands and the `pollora:doctor` check.

## Principles

- **Two settings, no more**: the disk (`media.disk`) and the public URL (`media.url`). The rest is derived.
- **A media URL is built, never asked**: the public URL and the relative path of the file, such as `https://media.client.fr/2026/10/photo.jpg`. No call to the disk happens while a public page renders.
- **The core is the integration point**: the `upload_dir` filter and a `pollora://` stream wrapper. Nothing is specific to a plugin.
- **Off by default**: without `media.disk`, WordPress keeps its own folder, `public/content/uploads`, and nothing changes.

## Setup

Declare the disk like any Laravel disk, in `config/filesystems.php`:

```php
'disks' => [
    'media' => [
        'driver' => 's3',
        'key' => env('MEDIA_KEY'),
        'secret' => env('MEDIA_SECRET'),
        'region' => env('MEDIA_REGION', 'eu-west-3'),
        'bucket' => env('MEDIA_BUCKET'),
        'endpoint' => env('MEDIA_ENDPOINT'),          // R2, Scaleway, MinIO…
        'use_path_style_endpoint' => env('MEDIA_PATH_STYLE', false),
        'url' => env('MEDIA_URL'),
    ],
],
```

Install the S3 driver of Flysystem when you use S3 or a compatible service (a local disk needs nothing):

```bash
composer require league/flysystem-aws-s3-v3
```

Then point the module at it, in `.env`:

```env
MEDIA_DISK=media
MEDIA_URL=https://media.client.fr
```

`MEDIA_URL` is the address the library is served from, a domain or a CDN in front of the bucket. Left empty, it is the `url` of the disk. Publish `config/media.php` to change the other options:

```bash
php artisan vendor:publish --tag=media-config
```

| Key | Default | Meaning |
|---|---|---|
| `disk` | `null` | The disk of `config/filesystems.php` that holds the library. `null`: the module is off |
| `url` | `null` | Public URL of the library. `null`: the `url` of the disk |
| `legacy_urls` | `[]` | Old base URLs of the library, replaced by `url` in the content and in rendered blocks during a transition |
| `cache_control` | `public, max-age=31536000, immutable` | `Cache-Control` set when a file is written |
| `private.expiry` | `+6 hours` | Life of a signed URL |
| `async_thumbnails` | `false` | Generate the image sizes after the upload |

## What follows without any code

WordPress reads the upload directory through `wp_upload_dir()`, which the module points at the wrapper and the public URL. Everything built on it follows: the media library, `wp_get_attachment_url()`, image sizes and `srcset`, `Pollora\Models\Attachment`, and the fields of ACF and Meta Box, which store an attachment ID. The date folders (`2026/10`) and, in a network, the folder of each site are kept.

A plugin that writes under `WP_CONTENT_DIR . '/uploads'` by hand ignores `upload_dir`, and its files stay local. A plugin that uses `ZipArchive` or a C library on a path fails on a stream wrapper. WooCommerce writes its logs under `uploads/wc-logs`: keep them on a local path.

## Changing the storage

The URL of a media stays valid when you change the disk, as long as `media.url` does not move: point a domain at the new storage and nothing in the content changes. Otherwise, WordPress stores relative paths in the database, so only the absolute URLs written in the content break.

To keep old URLs working during a transition, list them in `legacy_urls`; they are replaced by `media.url` with a plain string replacement, which is undone by removing the setting.

To move the files, copy them first:

```bash
php artisan pollora:media:migrate --from=uploads --to=media             # dry run: what would be copied
php artisan pollora:media:migrate --from=uploads --to=media --execute   # copy the files
```

Without `--from`, the source is the folder WordPress uses without the module. A file already on the target with the same size is skipped, so the command can run again after an interruption, and the source is never changed. Add `--replace-urls` (and `--old-url=` when it is not `APP_URL/content/uploads`) to replace the old base URL in the database: post content and excerpts, post, term and user meta, and options, serialized values and the JSON of blocks included. The `guid` is left alone. It only writes with `--execute`.

## Commands

| Command | Role |
|---|---|
| `pollora:media:status` | The disk, the public URL, and a test write |
| `pollora:media:migrate` | Copy the library from a disk to another; dry run unless `--execute` |

`pollora:doctor` also checks that the disk takes a file, and warns of files that are still only in the local uploads folder.

## Hooks

Extending the module goes through WordPress hooks, which work without Laravel too.

| Hook | Role |
|---|---|
| `pollora/media/attachment_files` (filter) | Files a plugin keeps next to a media (a WebP twin, a preview): a list of paths, deleted with the media |
| `pollora/media/filesystem` (filter) | Without Laravel: provides a `League\Flysystem\FilesystemOperator` for a service the constants cannot describe |

## Without Laravel: a classic theme, a plugin or Bedrock

`pollora/media` also runs in a WordPress project that is not a Pollora application. Set the library with constants in `wp-config.php`, then register it from a must-use plugin:

```php
// wp-config.php
define('POLLORA_MEDIA_DRIVER', 's3');
define('POLLORA_MEDIA_BUCKET', 'client-media');
define('POLLORA_MEDIA_REGION', 'eu-west-3');
define('POLLORA_MEDIA_URL', 'https://media.client.fr');
```

```php
// mu-plugins/media.php
\Pollora\Media\Media::register();
```

The constants are `POLLORA_MEDIA_URL`, `_LEGACY_URLS`, `_CACHE_CONTROL`, `_PRIVATE_EXPIRY`, `_ASYNC_THUMBNAILS`, `_DRIVER` (`local` or `s3`), `_ROOT` (the local folder) and, for S3 and compatible services, `_BUCKET`, `_REGION`, `_KEY`, `_SECRET`, `_ENDPOINT` and `_PATH_STYLE`. The package's [README](https://github.com/Pollora/media#readme) lists them all, with `wp pollora media status` and `wp pollora media migrate`, the WP-CLI twins of the Artisan commands.
