# Testing a Pollora Project

A Pollora project is tested with PHPUnit, through `composer test`. Plain classes take an ordinary unit test; code that needs WordPress takes a feature test that loads the site first; and the asynchronous actions, the scheduler and the options have helpers so a test does not queue, schedule or store anything for real.

- [The runner and `composer test`](#the-runner-and-composer-test)
- [What the test environment changes](#what-the-test-environment-changes)
- [A first unit test](#a-first-unit-test)
- [A first feature test that reaches WordPress](#a-first-feature-test-that-reaches-wordpress)
  - [Why the test loads WordPress itself](#why-the-test-loads-wordpress-itself)
  - [Why it runs in its own process](#why-it-runs-in-its-own-process)
- [Fakes and helpers](#fakes-and-helpers)
- [In a CI job](#in-a-ci-job)

## The runner and `composer test`

The skeleton requires `phpunit/phpunit` in `require-dev`, with Mockery and Faker. `pestphp/pest` is not installed: `tests/Pest.php` is a template file that does nothing until you install Pest yourself.

`composer test` clears the configuration cache, then runs Laravel's test command:

```json
"test": [
    "@php artisan config:clear --ansi",
    "@php artisan test"
]
```

`phpunit.xml` defines two suites, `Unit` (`tests/Unit`) and `Feature` (`tests/Feature`), so you can run one of them, or one file, through the same command:

```bash
composer test
php artisan test --testsuite=Unit
php artisan test tests/Feature/FrontPageTest.php
php artisan test --filter=FrontPageTest
```

A fresh skeleton ships tests of its own. `tests/Unit` checks that the WordPress core patch and the Composer patches are in place. `tests/Feature/Installation` checks the installation: the `.env` file, the database connection, the WordPress tables and options, and, over HTTP, that `APP_URL` answers. Those need an installed site and a web server serving `APP_URL`.

Two other scripts exist for the skeleton's own maintenance: `composer test:integration` (a suite of HTTP checks against a running site, whose URL comes from `POLLORA_TEST_URL`) and `composer test:install`. They are not part of `composer test`.

## What the test environment changes

`phpunit.xml` sets, for the run:

| Setting | Value |
|---|---|
| `APP_ENV` | `testing` |
| `CACHE_STORE` | `array` |
| `QUEUE_CONNECTION` | `sync` |
| `SESSION_DRIVER` | `array` |
| `MAIL_MAILER` | `array` |

Pollora maps the `testing` environment to WordPress's `development` environment type, so `wp_get_environment_type()` returns `development` during a test, and `DISABLE_WP_CRON` is on.

Two things to know:

- PHPUnit's `<env>` does not replace a variable that is already set. If your shell exports `APP_ENV=local`, the tests run as `local`.
- `phpunit.xml` does not choose a database: its `DB_CONNECTION` and `DB_DATABASE` lines are commented out. The tests use the database of your `.env`, which is the one holding your WordPress site. Point `.env` at a separate database for the tests, or use a throwaway one in CI, before a test writes anything.

## A first unit test

A class that does not call WordPress or Laravel needs only PHPUnit's `TestCase`. This is what the skeleton's `tests/Unit/ExampleTest.php` does:

```php
// app/Support/Vat.php
namespace App\Support;

final class Vat
{
    public static function including(int $cents, float $rate): int
    {
        return (int) round($cents * (1 + $rate));
    }
}
```

```php
// tests/Unit/VatTest.php
namespace Tests\Unit;

use App\Support\Vat;
use PHPUnit\Framework\TestCase;

class VatTest extends TestCase
{
    public function test_it_adds_the_rate(): void
    {
        $this->assertSame(1200, Vat::including(1000, 0.2));
    }
}
```

When the class takes the options repository, mock `OptionRepositoryInterface` rather than WordPress: see [Options, Unit Testing](options.md#unit-testing).

## A first feature test that reaches WordPress

`Tests\TestCase` extends Laravel's `TestCase`, so a feature test boots the application, and Pollora loads WordPress with it. What it does not do is process a request. A test that asks WordPress what it would do for a URL (which page is it, is it a 404) has to set the URL and ask WordPress to parse it.

Add a base class once:

```php
// tests/WordPressTestCase.php
namespace Tests;

abstract class WordPressTestCase extends TestCase
{
    protected string $uri = '/';

    protected function setUp(): void
    {
        $_SERVER['REQUEST_URI'] = $this->uri;
        $_SERVER['SERVER_NAME'] ??= 'localhost';
        $_SERVER['HTTP_HOST'] ??= 'localhost';

        defined('WP_USE_THEMES') || define('WP_USE_THEMES', true);

        parent::setUp();

        wp();
    }
}
```

Then a test declares the URL it visits and runs in its own process:

```php
// tests/Feature/UnknownUrlTest.php
namespace Tests\Feature;

use PHPUnit\Framework\Attributes\PreserveGlobalState;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;
use Tests\WordPressTestCase;

#[RunTestsInSeparateProcesses]
#[PreserveGlobalState(false)]
final class UnknownUrlTest extends WordPressTestCase
{
    protected string $uri = '/no-such-page/';

    public function test_an_unknown_url_is_a_404(): void
    {
        $this->assertTrue(is_404());
    }
}
```

With `protected string $uri = '/'` (the default), `is_front_page()` is true. WordPress's conditional tags (`is_page()`, `is_singular()`, `is_post_type_archive()`…) answer for the URL, and `get_queried_object()` returns what WordPress found. The site must be installed (`php artisan pollora:install`) and the database must be reachable.

### Why the test loads WordPress itself

Under `php artisan test` the application runs in the console. In a console run, the framework does not switch the theme layer on: it defines `WP_USE_THEMES` as `false` (`src/WordPress/Bootstrap.php` in `pollora/framework`, `defineWordPressConstants()`). Nothing parses the URL either. In a test without the base class above, `WP_USE_THEMES` was `false` and both `is_front_page()` and `is_404()` returned `false` whatever the URL, because WordPress had not run its query.

The base class does what a web request does. It sets `REQUEST_URI` and the server variables, defines `WP_USE_THEMES` as `true` if nothing has defined it yet, before the application boots, and calls `wp()`, the WordPress function that parses the request and builds the main query.

### Why it runs in its own process

`WP_USE_THEMES` is a PHP constant, and a constant is defined once per process. The first test to define it, or to let the framework define it, decides the value for every test after it. The same goes for the other WordPress constants and globals and the files WordPress has loaded. `#[RunTestsInSeparateProcesses]` starts a new PHP process for each test of the class, with `#[PreserveGlobalState(false)]` so the child does not inherit the parent's globals. The framework's own test of the WordPress constants, `tests/Unit/WordPress/BootstrapEnvironmentTest.php`, uses these two attributes for the same reason.

Two details:

- Put the attributes on the test class itself. Placed on an abstract base class, they were not applied: the tests of a class extending it shared one process.
- Instead of attributes, `php artisan test --process-isolation` runs every test of the suite in its own process. It is slower.

The tests in the next section call neither `wp()` nor `define()`, and ran in the suite's process.

## Fakes and helpers

| What | Helper | Documented in |
|---|---|---|
| Asynchronous actions | `Async::fake()` records what would be queued; `assertDispatched()`, `assertDispatchedTimes()`, `assertNotDispatched()`, `assertNothingDispatched()` | [Asynchronous Actions, Testing](async-actions.md#testing) |
| Scheduler | `Schedule::fake()` records events and attaches no handler; `assertScheduled()`, `assertScheduledTimes()`, `assertNotScheduled()`, `assertNothingScheduled()`; `Schedule::reset()` brings the real driver back | [Scheduling, Testing](schedule-events.md#testing) |
| Options | `Pollora\Option\Domain\Contract\OptionRepositoryInterface` (`get`, `store`, `update`, `delete`, `exists`), to mock or to replace | [Options, Unit Testing](options.md#unit-testing) |

The two fakes work in a feature test with the skeleton's `Tests\TestCase`, with no `wp()` call. This test queues an asynchronous action and checks it without running it, then runs it:

```php
use Pollora\Hook\Async\Async;
use Pollora\Hook\Async\AsyncPayload;
use Pollora\Support\Facades\Action;
use Tests\TestCase;

class SyncToCrmTest extends TestCase
{
    public function test_the_handler_is_queued_then_run(): void
    {
        $fake = Async::fake();
        Action::add('my_event', [SyncToCrm::class, 'handle'])->async()->delay(60);

        do_action('my_event', 7);

        Async::assertDispatched(SyncToCrm::class, fn (AsyncPayload $payload, int $delay): bool => $delay === 60);
        Async::assertDispatchedTimes(SyncToCrm::class, 1);

        $fake->runDispatched();
    }
}
```

`runDispatched()` is a method of the object `Async::fake()` returns, not of the `Async` class.

For the scheduler:

```php
use Pollora\Schedule\Event;
use Pollora\Schedule\Schedule;

public function test_a_digest_is_scheduled(): void
{
    Schedule::fake();

    Schedule::once('send_digest')->in(minutes: 10)->with(['user' => 42])->call(SendDigest::class);

    Schedule::assertScheduled('send_digest', fn (Event $event) => $event->args === ['user' => 42]);
    Schedule::assertNotScheduled('send_survey');

    Schedule::reset();
}
```

Call `Schedule::reset()` at the end of a test that faked the scheduler: the fake stays in place until it is reset.

## In a CI job

The job needs what the tests need: PHP 8.3 or later, the Composer dependencies, a `.env`, a database, and an installed WordPress. The skeleton's own workflow (`.github/workflows/tests.yml`) does this, in order:

```bash
composer install --no-interaction --no-progress
php artisan key:generate --no-interaction
php artisan pollora:install --install --title="…" --description="…" \
  --admin-user=admin --admin-email=admin@example.com --admin-password="…" \
  --locale=en_US --public=false --no-interaction
composer test
```

with a `.env` copied from `.env.example` and its `DB_*` lines pointed at the job's database. `pollora:install` creates the WordPress tables that the shipped `Feature/Installation` tests read.

`APP_ENV=testing` comes from `phpunit.xml`, as above.

Add the doctor after the tests:

```bash
php artisan pollora:doctor
```

`pollora:doctor` checks the project for failures that leave the site rendering: a missing Composer patches lock, a stale discovery cache, a theme without its Vite build, an ignored `#[Async]`, and the others listed in [Diagnosing a project](dashboard.md#diagnosing-a-project-polloradoctor). It exits `0` when no check finds an error (warnings do not change that) and `1` when one does, so the job fails on its own. `--json` prints the report for a script, and exits with the same code. Run it in the job that builds the project, after the theme's `npm run build`, since it also looks for the builds. The checks that need a web request, such as the document root, run in Tools › Site Health, not in the console.

## See also

- [Asynchronous Actions](async-actions.md#testing) — `Async::fake()`
- [Scheduling](schedule-events.md#testing) — `Schedule::fake()`
- [Options](options.md#unit-testing) — `OptionRepositoryInterface`
- [Dashboard & Status](dashboard.md#diagnosing-a-project-polloradoctor) — `pollora:doctor`
- [Local Development Environments](local-environments.md) — a database and a server for the project
