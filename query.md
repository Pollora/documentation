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

## Decimal meta values

A float compared in a meta query (`1.5`) is cast with `DECIMAL(30,10)`: the scale is at least 10, more when the value has more decimals (up to 30), and a list takes its widest scale. A numeric string with decimals (`'1.5'`) is no longer truncated to an integer either, so `1.5` and `1.9` no longer compare as `2`. An explicit type still wins:

```php
PostQuery::select()
    ->metaQuery(function (MetaQuery $query) {
        $query->where($query->meta('price')->greaterThan(1.5));                           // DECIMAL(30,10)
        $query->where($query->meta('weight')->ofType('DECIMAL(10,2)')->greaterThan(1.5)); // yours
    })
    ->get();
```

## `containsAll()` and child terms

On a hierarchical taxonomy WordPress includes the children of every listed term, so a native `AND` clause asked for a post that has a term and its child together. `containsAll()` therefore sends one `IN` clause per term, joined with `AND`: a post that has each term, or one of its children. With `excludeChildren()` or `onlyParent()` the single native `AND` clause is kept.

```php
PostQuery::select()
    ->taxQuery(function (TaxQuery $query) {
        $query->where($query->taxonomy('category')->containsAll([4, 7]));              // IN(4 + children) AND IN(7 + children)
        // ->containsAll([4, 7])->onlyParent() would send one AND clause
    })
    ->get();
```
