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

## Images

Both WordPress image editors, GD and Imagick, work on the library as they are. During a request, a file that was written or read (up to 16 MiB) is kept in a local copy, so the original of an uploaded image is not read back from the disk for its size, its EXIF data and each of its sizes: an upload of an image with nine sizes makes ten writes and a handful of existence checks.

Set `async_thumbnails` to `true` to make only `thumbnail` and `medium` during the upload and generate the other sizes afterwards, in an asynchronous action that runs on the queue, Action Scheduler or WP-Cron (see [asynchronous actions](async-actions.md)). It is off by default, because the block editor sometimes waits for a size right after the upload; until the job ran, WordPress serves the nearest size it has.

The personal data export keeps working on any disk: WordPress builds its archive with `ZipArchive`, which cannot write to a stream wrapper, so the archive is built locally, copied to the disk, and removed from it when it expires.

## Image sizes

Declare the sizes your design uses with the `ImageSize` facade, instead of leaving WordPress to make every registered size for every image:

```php
use Pollora\Support\Facades\ImageSize;

ImageSize::add('card')->cover(600, 400)->webp(75);
ImageSize::add('hero')->scale(width: 1920)->webp(70)->widths([640, 1024, 1440]);
ImageSize::add('thumb')->cover(150, 150)->eager();
```

Declare them where the application boots, for instance in a service provider. A size is then used by its name, like any WordPress size: `wp_get_attachment_image($id, 'card')`, `the_post_thumbnail('hero')`, or `wp_get_attachment_image_src()`.

| Option | Role | Default |
|---|---|---|
| `cover(w, h)`, `scale(width:, height:)`, `contain(w, h)` | How the image fits: crop to the box, resize keeping the ratio, fit inside the box | none: one is required |
| `webp(q)`, `avif(q)`, `original()` | The format of the file and its quality | WebP, quality 80 |
| `widths([...])` | Extra widths for a `srcset`, each a bounded size of its own | none |
| `eager()` | Made when the image is uploaded | no: made when first asked for |
| `upscale()` | Allow enlarging an original smaller than the size | off |

`scale()` and `contain()` never enlarge an original that is smaller than the size, and `cover()` then keeps the ratio of the box at the size the original allows. A GIF is never resized, which would stop its animation: its sizes are the original.

**The URL of a size is worked out, never read.** It comes from the metadata of the original (its file, width and height), whether the file of the size exists or not: `photo-card-600x400-{signature}.webp`. The signature is an HMAC of the original and of the parameters of the size, made with the application key and cut to 12 characters: changing a size changes its name, so no cache ever holds a stale file. The `width`, `height` and `srcset` WordPress writes come from the same computation, with no call to the disk.

The sizes WordPress registers itself (`add_image_size()`, a theme's `config/images.php`, WooCommerce, the core sizes) are left as they are, with the names WordPress gives them: a theme keeps its output until it declares sizes of its own. A declared size of the same name replaces the WordPress one.

### Sizes made on demand

A size is made at upload only if it is declared `eager()`. The others are made the first time their URL is asked for: the web server finds no file, sends the request to PHP, and the module makes the size at the exact path of its URL and answers with it (`200`, `Cache-Control: public, max-age=31536000, immutable`, never a redirect). The next request finds the file, and the web server or the CDN serves it without PHP. The URL never changes, so nothing has to be regenerated when a size is added or changed.

- **Local disk.** Nginx (`try_files $uri /index.php`) and Apache (the `!-f` rule to `index.php`) already send a missing file to PHP: nothing to set up.
- **S3 or compatible service behind a CDN.** The origin of the CDN for the library is the site: one rule at the CDN, and PHP sees each file only the first time.
- **S3 without a CDN.** There is no fallback: declare the sizes you use `eager()` and run `pollora:media:thumbs:warm`. The URL stays the same.

What the URL says is not trusted. Before anything is made, without asking the disk: the name is one of a declared size, the signature is the one the site would have made, the extension is an image's. Then, against the original: its dimensions are those the name gives (nobody can name a size of their own), it is not private, it is not a GIF. A request that fails a check gets a `404` and costs nothing.

Making a size costs CPU, so it is limited per server in `config/media.php`:

```php
'thumbnails' => [
    'max_pixels' => 50_000_000,   // an original above it gets no size, and stays downloadable
    'max_bytes' => 30 * 1024 * 1024,
    'concurrency' => 4,           // sizes made at the same time on a server
    'wait' => 10,                 // seconds a request waits for a place
],
```

Ten simultaneous requests for the same missing size make one. A request that cannot get a place within `wait` seconds is handed a size that already exists (the smallest that is at least as wide, the original at worst) with `Cache-Control: no-store`: the visitor sees an image, the CDN does not keep the stand-in, and the real size is made on a later request. `pollora:doctor` reports a `.htaccess` in the uploads folder that contains `RewriteEngine On`, which makes Apache ignore the rules above it and cuts the fallback; a security plugin usually adds it.

An image uploaded before a size was declared has no file for it yet. Make them before a heavy page goes live, and remove the files of a size you changed or dropped:

```bash
php artisan pollora:media:thumbs:warm --size=card --since=2026-01-01
php artisan pollora:media:thumbs:prune             # a dry run
php artisan pollora:media:thumbs:prune --execute
```

`prune` only considers files named the way the module names its sizes, next to an original that is still there: what WordPress or a plugin made is left alone. `pollora:doctor` reports a declaration that cannot make a size, and a format the image editor of the server cannot write (GD and Imagick write WebP; AVIF depends on the server).

### Sizes made by a service

For a very large library, an image service can make the sizes instead of the site. The declaration of the sizes does not change; only the shape of the URL does. Choose the driver in `config/media.php`:

```php
'thumbnails' => [
    'driver' => env('MEDIA_THUMBNAILS_DRIVER', 'pollora'),   // pollora | bunny | imgproxy
    'bunny' => ['pull_zone' => env('BUNNY_PULL_ZONE'), 'api_key' => env('BUNNY_API_KEY')],
    'imgproxy' => [
        'url' => env('IMGPROXY_URL'),
        'key' => env('IMGPROXY_KEY'),             // hexadecimal, as on the server
        'salt' => env('IMGPROXY_SALT'),
        'signature_size' => (int) env('IMGPROXY_SIGNATURE_SIZE', 32),
        'source' => env('IMGPROXY_SOURCE', 'url'), // url | s3
    ],
],
```

| Driver | URL of a size | Defined at the service by |
|---|---|---|
| `pollora` (default) | `photo-card-600x400-{signature}.webp`, a file of the library | the module |
| `bunny` | `photo.jpg?class=card&v=3f9a1c2b`, an [image class](https://bunny.net/docs/optimizer/image-classes) of Bunny Optimizer | `pollora:media:thumbs:export --driver=bunny` |
| `imgproxy` | `{imgproxy}/{signature}/card/plain/{source}@webp?v=3f9a1c2b`, signed, presets-only | `pollora:media:thumbs:export --driver=imgproxy` |

`v` is a short tag that changes when the size does. The service ignores it, but a CDN keys its cache on it: without it, a class or a preset you changed would keep being served as it was under the same URL (checked on a real Bunny pull zone). With a service, no file is made (neither at upload nor on demand), the fallback route is not registered and `pollora:media:thumbs:warm` has nothing to do. A size keeps a single URL, built from the metadata of the original, and each width of `widths()` becomes a class or a preset of its own (`hero-640w`). **A private media never goes to the service**: its sizes are not asked of it, WordPress serves the original, signed, as for any private media.

**Bunny.** `media.url` must be the pull zone that serves the library, with the Dynamic Image API enabled in Optimizer → Settings. Bunny accepts only letters and digits in the name of a class, so the class of `hero-640w` is `hero640w` and that of `my_size` is `mysize`; two sizes whose classes would be the same cannot both be served, and the second is refused (the export and `pollora:doctor` say so). Forcing classes ("Force Transformation Classes") is up to you: when it is on, a valid class is served, an unknown class or a raw transformation gets a `403`, and the original, asked with no parameter, is still served. Bunny has no cover mode, so a `cover()` size is made with `aspect_ratio` and a width; the dimensions served are the ones the module advertises, for an original that is big enough as for one that is too small, give or take one pixel: when the ratio of the original does not divide exactly, Bunny rounds the height and then takes the width back to the integer below (1199 pixels for a size of 1200). imgproxy serves exactly the advertised dimensions. Bunny encodes AVIF up to 4 megapixels and falls back to another format above that.

**imgproxy** (4.0.x). The URL is signed with HMAC-SHA256 and names a preset, so a visitor cannot ask for any other transformation. Set `IMGPROXY_KEY`, `IMGPROXY_SALT` and `IMGPROXY_SIGNATURE_SIZE` to the same values on the server. With `source` set to `url`, imgproxy fetches the original at `media.url`; with `s3`, it reads the object of the bucket of the media disk (`IMGPROXY_USE_S3` on the server). imgproxy refuses a source on the loopback address by default (a `media.url` of `localhost`): for local development only, start it with `IMGPROXY_ALLOW_LOOPBACK_SOURCE_ADDRESSES=true`; the export warns about it. A size cannot be named `default`, which imgproxy applies to every image.

Declare the classes or presets from your sizes, and run it again when a size changes:

```bash
php artisan pollora:media:thumbs:export --driver=bunny            # a dry run: what would be created or updated
php artisan pollora:media:thumbs:export --driver=bunny --execute  # write the classes to the pull zone
php artisan pollora:media:thumbs:export --driver=imgproxy         # the configuration to give imgproxy
```

For Bunny, the classes are read, merged by name and written back: the ones you made are kept, nothing is deleted, and the setting that forces classes is never changed. The merge matters: Bunny replaces the whole list of classes with the one it is sent. The access key is sent in the `AccessKey` header and nowhere else. imgproxy has no API for presets: the command prints `IMGPROXY_PRESETS`, `IMGPROXY_ONLY_PRESETS=true`, `IMGPROXY_MAX_SRC_RESOLUTION` (in megapixels) and `IMGPROXY_ALLOWED_SOURCES`, never the key or the salt. A driver that is badly set up does not take the site down: the default driver is kept, and `pollora:doctor` says why.

## Private media

A media is private when its post meta `_pollora_media_private` is `1`, or when the `pollora/media/is_private` filter says so. A private media is served through a signed URL that expires after `media.private.expiry` (`+6 hours` by default): its URL, the URL of each size and every source of its `srcset`. A public media keeps its plain URL, and the disk is never asked to tell the two apart: the flag is read with the attachment.

Nothing in the admin marks a media private. Do it from the command line, from code, or by rule:

```bash
php artisan pollora:media:visibility 42 --private   # every file of the media changes on the disk
php artisan pollora:media:visibility 42 --public
php artisan pollora:media:files 42                  # the original, the sizes and the declared files
```

```php
// By rule: a media created private is private on the disk from the start
add_filter('pollora/media/is_private', fn (bool $private, int $id) => get_post_meta($id, 'invoice', true) !== '', 10, 2);
```

The signature is made by the Laravel disk (`temporaryUrl()`): an S3 pre-signed URL for S3 and compatible services. A disk that cannot sign, such as a plain local disk, serves the plain URL of a private media instead of a broken one, and `pollora:doctor` says so: a private media needs an object store, or a signature of your own through the `pollora/media/signed_url` filter.

The module's actions reach Laravel as events, for an audit or to purge a CDN: `Pollora\Media\Domain\Events\MediaStored`, `MediaDeleted` and `MediaVisibilityChanged`.

## Commands

| Command | Role |
|---|---|
| `pollora:media:status` | The disk, the public URL, and a test write |
| `pollora:media:migrate` | Copy the library from a disk to another; dry run unless `--execute` |
| `pollora:media:thumbs:warm` | Make the missing files of the declared sizes for the images already uploaded (`--size=`, `--since=`) |
| `pollora:media:thumbs:prune` | Delete the files of a size that was changed or removed; dry run unless `--execute` |
| `pollora:media:thumbs:export` | Export the declared sizes as Bunny image classes or imgproxy presets (`--driver=`, `--execute` for Bunny) |
| `pollora:media:files {id}` | The files of a media: original, sizes, declared files |
| `pollora:media:visibility {id}` | Make a media private (`--private`) or public (`--public`) |

`pollora:doctor` also checks that the disk takes a file, and warns of files that are still only in the local uploads folder.

## Hooks

Extending the module goes through WordPress hooks, which work without Laravel too.

| Hook | Role |
|---|---|
| `pollora/media/attachment_files` (filter) | Files a plugin keeps next to a media (a WebP twin, a preview): a list of paths, deleted with the media |
| `pollora/media/is_private` (filter) | `(bool $private, int $attachmentId)`: the visibility of a media |
| `pollora/media/signed_url` (filter) | `(string $url, int $attachmentId)`: the final signed URL, for a CDN with its own signature |
| `pollora/media/stored`, `pollora/media/deleted`, `pollora/media/visibility_changed` (actions) | A file is on the disk; a media and its files were deleted; a media changed visibility |
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

The constants are `POLLORA_MEDIA_URL`, `_LEGACY_URLS`, `_CACHE_CONTROL`, `_PRIVATE_EXPIRY`, `_ASYNC_THUMBNAILS`, `_THUMBNAILS_DRIVER` (`pollora`, `bunny` or `imgproxy`), `_BUNNY_PULL_ZONE`, `_BUNNY_API_KEY`, `_IMGPROXY_URL`, `_IMGPROXY_KEY`, `_IMGPROXY_SALT`, `_IMGPROXY_SIGNATURE_SIZE`, `_IMGPROXY_SOURCE`, `_DRIVER` (`local` or `s3`), `_ROOT` (the local folder) and, for S3 and compatible services, `_BUCKET`, `_REGION`, `_KEY`, `_SECRET`, `_ENDPOINT` and `_PATH_STYLE`. The package's [README](https://github.com/Pollora/media#readme) lists them all, with `wp pollora media status` and `wp pollora media migrate`, the WP-CLI twins of the Artisan commands.
