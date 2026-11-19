# Performance Gates

The Pollora framework checks its own cost in its browser tests: how many SQL queries each type of page runs, and how long the boot, a cold discovery and a thumbnail take. This page describes the diagnostic header those checks read, the query budgets and the nightly measures. They belong to the framework's repository (`tests/e2e`); the header is the part a project can use too.

- [The diagnostic header](#the-diagnostic-header)
- [Query budgets per page type](#query-budgets-per-page-type)
- [Nightly measures](#nightly-measures)
- [See also](#see-also)

## The diagnostic header

A request that sends `X-Pollora-Diagnostics: 1` gets a report of what it cost, as long as `WP_DEBUG` is on:

```bash
curl -s -H 'X-Pollora-Diagnostics: 1' https://example.test/ | tail -3
curl -sI -H 'X-Pollora-Diagnostics: 1' https://example.test/wp-json/ | grep -i x-pollora-diagnostics
```

The report holds four values:

| Key | Meaning |
| --- | --- |
| `wordpress_queries` | Queries WordPress ran through `$wpdb` (`$wpdb->num_queries`) |
| `laravel_queries` | Queries Laravel ran, counted on its `QueryExecuted` event |
| `boot_ms` | Milliseconds from the start of the request to `wp_loaded` |
| `elapsed_ms` | Milliseconds from the start of the request to the report |

Where it appears depends on the response:

- **A REST response** carries it in the `X-Pollora-Diagnostics` response header, as JSON.
- **An HTML page** carries it in a trailing comment after the document: `<!-- pollora:diagnostics {...} -->`. Redirects, JSON, XML (feeds, sitemaps) and plain text never get one, and neither do wp-admin screens and admin-ajax calls.

Nothing is reported unless **both** conditions hold: `WP_DEBUG` is true and the request sends the header. A production site, where `WP_DEBUG` is false, never reports anything, whatever a visitor sends. A request without the header costs nothing: no hook is registered for it. The report does not depend on the [debug bar](debugging.md).

## Query budgets per page type

`tests/e2e/specs/query-budget.spec.ts` asks seven types of page for their own cost: home, post, page, archive, 404, search and REST. It fails when one runs more queries than `tests/e2e/budgets.json` allows, counted separately in WordPress and in Laravel.

- The margin is zero: counts are deterministic. Fewer queries than the budget passes.
- `0` is a real limit: a page allowed no Laravel query fails on the first one. `null` means no budget yet: the page is measured and reported, never failed.
- Raising a budget is a deliberate edit of `budgets.json`, reviewed like code. Lowering one keeps a gain.

The budgets are those of the CI site, a fresh install plus the fixtures of the suite. Another site has other counts. The pages measured are `/?p=1`, `/?page_id=2` and `/?cat=1`, the content WordPress creates on install.

To write the measured counts into `budgets.json`:

```bash
npm run budgets:update       # sets E2E_UPDATE_BUDGETS=1, measures, then writes the file
```

Run it against a site like the CI one, or take the first measure from a green CI run: every run writes `test-results/budgets.measured.json` and the CI uploads it as the `e2e-budgets-measured-*` artifact. Commit the result.

At the time of writing, `budgets.json` on the framework's `develop` branch still holds `null` everywhere (measure only). A pull request, #651, makes them strict: home 24, post 20, page 19, archive 26, 404 15, search 22, REST 19, and 0 Laravel queries on every page. Read the file for the current values.

## Nightly measures

`tests/e2e/specs/perf-metrics.spec.ts` runs only when `E2E_PERF=1`, which the nightly job sets. It writes `perf-metrics.json` with three values, each the median of several runs:

| Metric | Measures |
| --- | --- |
| `boot_ms` | A warm boot, up to `wp_loaded` |
| `cold_boot_ms` | The first request after `php artisan discovery:clear`, so with discovery rebuilt from scratch |
| `thumbnail_ms` | Generating every size of one 2400x1600 JPEG |

The job compares them with the last night that passed (`bin/perf-compare.ts`) and fails when one is more than 10 % slower. The reference is restored through `actions/cache`. A failing night does not replace the reference, so a regression cannot become the new normal. To accept a slowdown on purpose, delete the `perf07-reference-*` cache entries.

To run it by hand:

```bash
E2E_PERF=1 npx playwright test perf-metrics
node bin/perf-compare.ts current.json reference.json --tolerance=0.1
```

## See also

- [Debugging](debugging.md) — the debug bar, for a closer look at the queries a page runs
- [Testing a Pollora Project](testing.md) — unit and feature tests of your own project
