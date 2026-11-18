# Colt: Eloquent models for WordPress

[`pollora/colt`](https://github.com/Pollora/colt) is a set of [Eloquent](https://laravel.com/docs/eloquent) models over the WordPress tables: posts, pages, custom post types, meta, taxonomies, terms, comments, menus, options and users. In a Pollora project it is already installed, and you use the `Pollora\Models\*` models (`Post`, `Page`, `Term`, `User`…), which extend Colt's.

```php
use Pollora\Models\Post;

$posts = Post::published()->latest('post_date')->take(20)->get();
```

## Collections

A query returns an Eloquent `Collection`, and every type is a subclass of it, so `map()`, `filter()`, `pluck()` and the rest keep working:

| Models | Collection |
|---|---|
| Posts (and pages, custom post types) | `PostCollection` |
| Menu items (`$menu->items`) | `MenuItemCollection` |
| Terms and comments | `MetaSiblingsCollection` |

The collection is what lets Colt load what a loop will need once for the whole set, instead of once per item.

## No more N+1 queries

Reading a relation of each item in a loop used to cost one or more queries per item. Colt now loads them once per collection, with no change to the API. Measured on 100 posts, each with its own author, image, terms and comments:

| What you do | Queries before | Queries after |
|---|---|---|
| `Post::get()->toArray()` or `toJson()` | 502 | 7 |
| Read `$post->terms` (or `main_category`, `keywords`) in a loop | 202 | 4 |
| Read `$post->image` in a loop | 302 | 5 |
| `$menuItem->instance()` and `$menuItem->parent()` in a loop over `$menu->items` | 204 | 6 |
| `$term->meta` or `$comment->meta` in a loop over a collection | 201 | 2 |

Serialising posts loads the `image`, `terms`, `main_category` and `keywords` attributes once for the whole collection. Menu items resolve their targets with one query per target class. Meta is still not loaded, nor serialised, until something reads it. A second call to `instance()` on the same menu item queries again, as before.

## Load relations you read in a loop

Colt cannot batch the lazy Eloquent relations `author`, `comments`, `taxonomies` and `thumbnail`: reading one in a loop is one query per post, as with any Eloquent model. Name them with `with()` when you read them in a loop:

```php
$posts = Post::published()
    ->with('author', 'comments', 'taxonomies')
    ->take(100)
    ->get();

foreach ($posts as $post) {
    echo $post->author->display_name, ' ', $post->comments->count();
}
```

That is 12 queries for 100 posts, whatever the number of posts.
