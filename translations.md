# Translations

- [Why two systems](#why-two-systems)
- [The routing rule](#the-routing-rule)
  - [A string is a WordPress text domain](#a-string-is-a-wordpress-text-domain)
  - [A non-empty array is a Laravel call](#a-non-empty-array-is-a-laravel-call)
  - [No second argument is ambiguous](#no-second-argument-is-ambiguous)
- [The WordPress system (.po/.mo)](#the-wordpress-system-pomo)
  - [Text domain and Domain Path](#text-domain-and-domain-path)
  - [Extracting strings and compiling](#extracting-strings-and-compiling)
  - [Translating theme markup](#translating-theme-markup)
  - [Translating theme config](#translating-theme-config)
- [The Laravel system](#the-laravel-system)
  - [Scoped catalogues per theme and module](#scoped-catalogues-per-theme-and-module)
  - [The application's own catalogue](#the-applications-own-catalogue)
  - [Named placeholders and pluralization](#named-placeholders-and-pluralization)
- [Which locale is actually used](#which-locale-is-actually-used)
- [Choosing between the two](#choosing-between-the-two)
- [How the override works](#how-the-override-works)

## Why two systems

A Pollora project sits on top of two ecosystems that each ship their own way of translating strings, and neither can be dropped:

- **WordPress core, WooCommerce, and every third-party plugin** translate through gettext — `.po`/`.mo` catalogues keyed by a **text domain**, read with `__($text, $domain)`. This is what `Loco Translate`, `WPML`, and `Polylang` all hook into, and it's the only vocabulary a non-technical translator's tooling understands.
- **Laravel** translates through its own translator — JSON or PHP-array catalogues, read with `__($key, $replace)`, with named placeholders (`:brand`) and pluralization (`trans_choice()`) that gettext's `%s`/`sprintf()` style doesn't have.

Rather than forcing you to pick one and give up the other, Pollora overrides `__()` itself (via the [`pollora/helper-overrider`](https://github.com/Pollora/helper-overrider) package) so a single call routes to whichever system the call actually means. You keep writing `__()` everywhere — in theme Blade views, in config files, in WooCommerce filters — and it resolves correctly regardless of which side owns the string.

> **This is why the override exists.** WordPress declares `__()` unguarded in `wp-includes/l10n.php`; Laravel declares its own `__()` behind a `function_exists()` guard. Pollora patches WordPress core to free the name (renaming WordPress's version to `__wp()`) and installs a single `__()` that understands both.

## The routing rule

The **second argument** is what tells the two systems apart — there is no configuration flag to flip, and no import to choose between:

| Call | Routed to | Why |
|---|---|---|
| `__('Save changes')` | Laravel, then WordPress `default` domain | No intent expressed — the only ambiguous case. |
| `__('Save changes', 'my-plugin')` | WordPress, `my-plugin` domain | A text domain is an explicit WordPress call. |
| `__('Shipping :brand', ['brand' => $name])` | Laravel, then WordPress `default` domain — placeholders always filled | Named replacements are a Laravel idiom; WordPress has no equivalent. |

### A string is a WordPress text domain

```php
__('Add New', 'my-plugin');
```

This is exactly WordPress's own `__($text, $domain)` signature. Laravel is **never consulted** — which matters as soon as you have a Laravel translation key that happens to share the same text. A string second argument always means "look this up in the named gettext catalogue," full stop.

### A non-empty array is a Laravel call

```php
__('Shipping :brand', ['brand' => $carrier->name]);
```

WordPress's gettext strings don't support named placeholders — a `msgid` uses `%s`, substituted by `sprintf()` at the call site — so a replacement array can only mean one thing: a Laravel translation, resolved through Laravel's translator with its usual `:brand` / `:Brand` / `:BRAND` substitution rules.

If the key isn't in the Laravel catalogue for the current locale, Pollora falls back to the WordPress `default` domain **for the line itself**, and still fills in the placeholders. In practice this means `__('Shipping :brand', ['brand' => $name])` is always safe to write — a WooCommerce or core string that happens to share the key keeps its own translation, and an untranslated key still renders correctly instead of leaking `:brand` to the visitor.

### No second argument is ambiguous

```php
__('Save changes');
```

This is the one case where the caller expressed no preference. Pollora tries the Laravel catalogue first (your app's own strings usually take priority), then falls back to the WordPress `default` domain — which is where WordPress core, most themes, and any plugin that never declared its own domain keep their strings.

There is deliberately no key-prefix escape hatch to force a bare-argument call onto one side or the other. A prefix like `wordpress.` would be invisible to gettext extraction tools (`wp i18n make-pot`, Poedit) — they read the literal `__()` argument, so a translator would translate a `msgid` the runtime never actually looks up once the prefix is stripped. If a bare key would collide with a catalogue entry you don't want consulted, use the explicit forms above instead: pass the WordPress text domain, or pass a (possibly single-entry) replacement array to force Laravel.

## The WordPress system (.po/.mo)

Use this system for anything that should behave like a normal WordPress string: translatable through the same `.po` files a WordPress-savvy translator already knows how to edit, filterable through the `gettext` hook, and compatible with `WPML`/`Polylang`/`Loco Translate` without any extra glue.

### Text domain and Domain Path

A theme or plugin declares its text domain and the directory holding its `.mo` files in its header (`style.css` for themes, the plugin's main file for plugins):

```
Text Domain: my-theme
Domain Path: /languages
```

WordPress loads the matching `.mo` for the active locale on first use — you don't call `load_theme_textdomain()` yourself. What you do have to get right is the **file name**, and it is not the same for themes and plugins:

| Where the catalogue lives | Expected file name | Example |
|---|---|---|
| Inside the theme (the `Domain Path` above) | `{locale}.mo` | `themes/my-theme/languages/fr_FR.mo` |
| A plugin, or a theme catalogue installed in `WP_LANG_DIR` | `{domain}-{locale}.mo` | `my-plugin-fr_FR.mo` |

Getting this wrong fails silently: the domain simply never loads and every string renders untranslated. A theme catalogue named `my-theme-fr_FR.mo` is the common mistake — inside the theme, WordPress looks for `fr_FR.mo` and nothing else.

> **Timing matters since WordPress 6.7.** A text domain loaded before the `after_setup_theme` action triggers a `_doing_it_wrong()` notice ("Translation loading for the `…` domain was triggered too early"), and the string comes back untranslated. This is why Pollora defers some theme config files — see [Translating theme config](#translating-theme-config).

### Extracting strings and compiling

The usual gettext round-trip applies. Extract a `.pot` template from your source with WP-CLI, copy it per locale, translate, then compile:

```bash
# 1. Extract every __( ) call into a template
wp i18n make-pot themes/my-theme themes/my-theme/languages/my-theme.pot

# 2. One .po per locale (copy the .pot, then translate it in Poedit / Loco Translate)
cp themes/my-theme/languages/my-theme.pot themes/my-theme/languages/fr_FR.po
```

Note the file naming again: inside a theme, the `.po`/`.mo` pair is named after the **locale** (`fr_FR.po` → `fr_FR.mo`), not after the text domain.

Pollora then ships a pure-PHP `.po` → `.mo` compiler (`Pollora\Translation\Infrastructure\Services\GettextMoCompiler`) — no `msgfmt` system binary required, which matters in Docker images and shared hosting where it may not be installed.

`php artisan pollora:make:theme` compiles every `.po` file it finds in the generated theme's `languages/` directory as part of scaffolding. To recompile after editing a `.po` file by hand, use the same compiler directly:

```php
use Pollora\Translation\Infrastructure\Services\GettextMoCompiler;

$compiler = new GettextMoCompiler;

// One file
$compiler->compile('/path/to/languages/my-theme-fr_FR.po');

// Every .po file in a directory
$compiler->compileDirectory('/path/to/languages');
```

> The `Pollora\Translation\Domain\Contracts\TranslationCompilerInterface` contract is what the scaffolding command depends on — bind your own implementation in the container if you need `.po` compilation backed by the system `msgfmt` binary instead.

### Translating theme markup

Inside a theme, `__()` with a text domain works exactly as it does in any WordPress theme:

```blade
<button>{{ __('Add to cart', 'my-theme') }}</button>
```

The same applies to any plugin or third-party code you don't control — `__($text, 'woocommerce')`, `__($text, 'default')` for WordPress core strings, and so on all resolve the normal WordPress way, since a string second argument always takes the WordPress path.

### Translating theme config

Theme config files are plain PHP, so they can call `__()` directly — this is how the default theme labels its menus:

```php
// themes/my-theme/config/menus.php
return [
    'menu-header' => __('Menu header', 'my-theme'),
    'menu-footer' => __('Menu footer', 'my-theme'),
];
```

**This only works in the config files Pollora defers.** Config is normally loaded while the theme boots, which is well before `after_setup_theme` — too early for any text domain, per the WordPress 6.7 rule above. Pollora works around it by deferring exactly three files to the `init` action, because those are the ones whose values reach WordPress late enough to matter:

| Config file | Loaded |
|---|---|
| `menus.php`, `sidebars.php`, `templates.php` | Deferred to `init` — `__()` is safe |
| everything else (`supports.php`, `images.php`, `gutenberg.php`, `providers.php`, …) | Eagerly, at theme boot — `__()` is **not** safe |

Calling `__($text, 'my-theme')` in a non-deferred config file returns the untranslated string and may emit a `_doing_it_wrong()` notice. If you need a translated value there, resolve it later — from a hook or a service provider that runs on `init` or after — rather than at config-load time.

> Pollora also carries a `Pollora\Services\Translater` helper, used internally to translate menu and sidebar config arrays by key (`(new Translater($items, 'menus'))->translate(['*'])`). It looks each value up as `{domain}.{value}`, which lets a project translate plain config strings through a Laravel `menus`/`sidebars` group file instead of gettext. The default theme uses explicit `__($text, 'my-theme')` calls in config instead, which is the clearer of the two.

## The Laravel system

Use this system for strings that belong to your application logic rather than to WordPress markup: validation messages, notification copy, anything with a named placeholder or a plural form, and any catalogue you'd rather manage as JSON/PHP arrays in version control than as binary `.mo` files.

### Scoped catalogues per theme and module

Themes and Laravel Modules each get their own **namespaced** translation catalogue automatically, rooted at their `lang/` directory:

```
themes/my-theme/lang/
├─ fr/
│  └─ messages.php
└─ fr_FR/
    └─ messages.php

Modules/Shop/lang/
└─ fr/
    └─ checkout.php
```

```php
// themes/my-theme/lang/fr/messages.php
return [
    'welcome' => 'Bienvenue',
];
```

```php
__('my-theme::messages.welcome');   // 'Bienvenue' on a fr/fr_FR site
trans('shop::checkout.confirm');    // same mechanism, module-scoped
```

This registration happens when the theme (or module) loads — there's nothing to configure. It is a genuinely separate mechanism from the `.po`/`.mo` catalogue above: a theme can carry both a `languages/` directory for its WordPress-facing strings and a `lang/` directory for its Laravel-facing ones, and the two never collide because they're reached through entirely different call shapes (`__($text, 'my-theme')` vs. `__('my-theme::messages.welcome')`).

### The application's own catalogue

Bare keys with no namespace (`__('Save changes')`, `__('Shipping :brand', [...])`) resolve against the application's own catalogue — Laravel's default `lang/` directory at the project root:

```
lang/
├─ fr.json
└─ fr_FR.json
```

```json
{
    "Shipping :brand": "Livraison via :brand"
}
```

Pollora doesn't scaffold this directory for you; create it the way any Laravel project would.

### Named placeholders and pluralization

```php
__('Shipping :brand', ['brand' => 'Colissimo']);
// 'Shipping Colissimo', or 'Livraison via Colissimo' with the JSON entry above

trans_choice('{0} No items|{1} One item|[2,*] :count items', $count);
```

`trans_choice()` and every other Laravel translation helper work exactly as they do in a plain Laravel app — only `__()` is overridden, because it's the one call shape shared with WordPress.

## Which locale is actually used

Both sides resolve against **WordPress's site locale** (`get_locale()`), not Laravel's `config('app.locale')` / `APP_LOCALE`. That config value only matters once, during `php artisan pollora:install` (the WordPress installer bootstrap) — it has no effect on a running site's translations. Changing `APP_LOCALE` in `.env` will not change what `__()` returns; changing the site language in **Settings → General** will.

WordPress locales are regional (`fr_FR`, `pt_BR`); Laravel's JSON lookup matches the locale file exactly and has no language fallback of its own. Pollora bridges this: on a `fr_FR` site, the Laravel lookup tries `fr_FR` first and then falls back to the base language `fr` — so a project shipping only `lang/fr.json` still gets picked up on a `fr_FR` site, while an existing `lang/fr_FR.json` always wins when present.

You can also pass an explicit locale as the third argument, which is tried the same way (regional, then base language) ahead of the site's own:

```php
__('Save changes', [], 'de_DE');
```

## Choosing between the two

| Use WordPress (`.po`/`.mo`, text domain) when… | Use Laravel (JSON/array, no domain) when… |
|---|---|
| The string is markup shipped by a theme or plugin, translated by non-technical contributors with `Loco Translate` / `Poedit` | The string belongs to your application logic — validation, notifications, admin UI copy |
| You need `WPML` / `Polylang` compatibility, or the `gettext` filter | You need named placeholders (`:brand`) or pluralization (`trans_choice()`) |
| The string already exists in a catalogue you don't own (WordPress core, WooCommerce, a third-party plugin) | You'd rather keep translations in version-controlled JSON/PHP than binary `.mo` files |
| You're translating a value you don't control the format of (a plugin's own strings) | You're building the string yourself and control every placeholder |

When a call fits both — a plain string with no placeholders and no explicit domain — write it as a bare `__('Some string')` and let Pollora fall back through Laravel then WordPress; there's no need to force one side over the other unless you actually need the routing guarantee described above.

## How the override works

WordPress declares `__()` unguarded, so it can't simply be redefined. The framework carries a Composer patch (`patches/wordpress-core.patch`, applied via `cweagans/composer-patches`) that renames WordPress core's version to `__wp()`, freeing the name for `pollora/helper-overrider`'s own `__()`, loaded through Composer's `autoload.files` ahead of `laravel/framework`'s own `__()` definition.

> **Extension author note:** `pollora/helper-overrider` is a standalone package with no runtime Composer dependency on Laravel by design — `laravel/framework` declares `__()` behind the same `function_exists()` guard, and Composer emits `autoload.files` in dependency order. Anything that made the package depend on `illuminate/*` would sort its `helpers.php` after Laravel's and silently hand `__()` back to Laravel, with every WordPress catalogue call quietly falling through to whatever Laravel returns for a missing key. This is enforced by a test in the package, not by convention alone.

For the full routing logic — including how replacements survive a WordPress fallback and how group-key edge cases are handled — see [`Pollora\HelperOverrider\Translation\TranslationResolver`](https://github.com/Pollora/helper-overrider/blob/main/src/Translation/TranslationResolver.php) in the package itself.
