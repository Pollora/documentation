# Query

[`pollora/query`](https://github.com/Pollora/Query) builds the arguments of the WordPress query classes (`WP_Query`, `WP_Term_Query`, `WP_User_Query`, `WP_Comment_Query`, `WP_Site_Query`, `WP_Network_Query`) with chained, type-hinted methods instead of nested arrays. In a Pollora project it is already installed: `PostQuery` is available as the `Pollora\Support\Facades\PostQuery` facade.

```php
use Pollora\Support\Facades\PostQuery;

$query = PostQuery::select()
    ->postType('post')
    ->postStatus('publish')
    ->tagSlugIn(['laravel', 'wordpress'])
    ->latest()
    ->take(5)
    ->get(); // WP_Query
```

The package documents every method, with a page per query class, in its [`docs/`](https://github.com/Pollora/Query/tree/main/docs) folder. This page only lists what matters when you build on it from a Pollora project.

## Tag arguments take slugs

`tagSlugAnd()` (posts that have all the tags) and `tagSlugIn()` (posts that have any of them) take **slugs**, as an `array<string>`, not term IDs. They write `tag_slug__and` and `tag_slug__in`. For IDs, use the `tagIn()` / `tagAnd()` family.

```php
PostQuery::select()->tagSlugAnd(['php', 'wordpress'])->get();
```

## Extending a query class

`metaQuery()`, `dateQuery()` and `taxQuery()` share one protected method, `mergeSubQuery()`, which merges the sub-query built by the callback into the current one and returns the merged query. It replaces `addSubQuery()`: a subclass that called or overrode `addSubQuery()` must use `mergeSubQuery()` instead.

```php
protected function mergeSubQuery(string $type, ?array $current, callable|SubQueryAbstract $callback): array
```

`$type` is `meta`, `tax` or `date`. Nothing changes for code that only uses the public builders.
