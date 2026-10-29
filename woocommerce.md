# WooCommerce

Pollora can write a WooCommerce shop in Blade, route its pages with `Route::wp()` and listen to its actions as Laravel events. It is the package [`pollora/woocommerce`](https://github.com/Pollora/woocommerce): the framework used to ship this integration, and does not any more.

> A project that runs WooCommerce must install the package. Without it the shop has no Blade templates, no `Route::wp()` conditions (`shop`, `product`, `cart`…), no Coming Soon page and no WooCommerce events, and nothing says so. `php artisan pollora:doctor` (and Tools › Site Health) warns when WooCommerce is active and the package is missing.

## Installation

```bash
composer require pollora/woocommerce
```

Requires PHP 8.3+, Pollora 13.39+ and WooCommerce. Laravel discovers the service provider; nothing runs unless WooCommerce is active. `pollora:make:theme` offers to install it with the `apiary` storefront theme (see [Theming](theming.md)).

## Blade templates

WooCommerce looks for a template in the theme, then in its own folder. The package adds the Blade variant of each candidate to the search, so `resources/views/woocommerce/single-product.blade.php` answers where `woocommerce/single-product.php` would, and a child theme's override folder is honoured. Template parts (`wc_get_template_part()`) and `wc_get_template()` resolve the same way.

Add a folder to the search with the `pollora/woocommerce/template_paths` filter.

## Route conditions

`Route::wp()` knows WooCommerce's conditionals (see [Routing](routing.md)):

```php
Route::wp('shop', [ShopController::class, 'index']);
Route::wp('product', [ProductController::class, 'show']);
```

| Alias | Condition |
| --- | --- |
| `shop` | `is_shop` |
| `product` | `is_product` |
| `cart` | `is_cart` |
| `checkout` | `is_checkout` |
| `account` | `is_account_page` |
| `product_category` | `is_product_category` |
| `product_tag` | `is_product_tag` |
| `wc_endpoint` | `is_wc_endpoint_url` |

The entries you set in `wordpress.plugin_conditions.woocommerce` (see [WordPress Configuration](wordpress-config.md)) win over these defaults.

## Coming Soon page

WooCommerce renders its Coming Soon page itself and exits, which skips Blade. With a view `woocommerce.coming-soon` in your theme (or `woocommerce.coming-soon-store-only` when only the store pages are hidden), the package renders that one instead.

## Events

WooCommerce's actions arrive as Laravel events, in `Pollora\WooCommerce\Events` (see [Events & Listeners](events-listeners.md)):

| Event | Comes from |
| --- | --- |
| `OrderStatusChanged` | `woocommerce_order_status_changed` |
| `AttributeCreated`, `AttributeUpdated`, `AttributeDeleted` | `woocommerce_attribute_*` |
| `TaxRateCreated`, `TaxRateUpdated`, `TaxRateDeleted` | `woocommerce_tax_rate_*` |
| `SettingUpdated` | WooCommerce options (`updated_option`) |

```php
use Illuminate\Support\Facades\Event;
use Pollora\WooCommerce\Events\OrderStatusChanged;

Event::listen(OrderStatusChanged::class, function (OrderStatusChanged $event): void {
    $event->order;      // WC_Order
    $event->newStatus;
});
```

## Coming from the framework

| Was | Is |
| --- | --- |
| `Pollora\ThirdParty\WooCommerce\…` | `Pollora\WooCommerce\…` |
| `Pollora\Events\WordPress\Plugins\WooCommerce\…` | `Pollora\WooCommerce\Events\…` |

The old names keep working and are removed in the next major release. Nothing else changes in a project's code: install the package, and the shop, its routes and its events are back.
