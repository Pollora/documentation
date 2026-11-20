# Quality Gates

[`pollora/harness`](https://github.com/Pollora/harness) is the set of rules, checked by machines, that every Pollora repository runs: code style, static analysis, architecture, tests, security, dependencies and the public contract. The same gates run on a developer's machine (`pollora-qa`), in a coding agent's hooks and in CI, so a change that breaks a rule is stopped before it is merged. This page describes the version 0.1.33 of the harness: how to adopt it, which rules it enforces, how to read a failure and how to take an exemption.

It is a development tool. Install it with `--dev`; nothing of it ships to the sites that run your code. It constrains the code of the framework, the packages and the plugins, never how a Pollora project is written.

- [Adopting the harness](#adopting-the-harness)
- [Observe and enforce](#observe-and-enforce)
- [What changes in v0.1.33](#what-changes-in-v0133)
- [The rules](#the-rules)
- [Reading a failure](#reading-a-failure)
- [Exemptions](#exemptions)
- [The public surface](#the-public-surface)
- [Commands](#commands)
- [See also](#see-also)

## Adopting the harness

```bash
composer require --dev pollora/harness
vendor/bin/pollora-qa init --profile=core
composer qa
```

`init` never overwrites a file. It writes `.harness.json`, adds the harness preset to `phpstan.neon`, and creates what is missing: `tests/Arch.php`, `rector.php`, `.github/workflows/harness.yml`, `.claude/settings.json`, `AGENTS.md`, and the `qa` and `qa:fast` Composer scripts.

`.harness.json` declares what the repository is:

```json
{
    "profile": "core",
    "mode": "observe"
}
```

The profile is one of `core` (standalone packages), `framework`, `integration` (packages built on the framework), `wp-plugin`, `template` and `project`. It decides which rules apply: a `core` package must stay Laravel-free (ARCH-04), only an `integration` package, a template or a project may require `pollora/framework` (ARCH-06).

CI runs the reusable workflow, pinned to a release tag. `init` writes the call for you, at the tag of the installed harness:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/php-package.yml@v0.1.33
    permissions:
      contents: read
```

`php-package.yml` runs the quality gates, a test matrix (PHP versions, lowest and stable dependencies), a gitleaks scan, the PHP API comparison and, for a core package, the **downstream** job that runs the framework's tests against the change. A template (the skeleton, a theme, a plugin or module template) cannot carry `pollora/harness` in its `require-dev`, since every project created from it would install it. It adds one workflow that calls `template.yml` instead, with `mode` and `observed` inputs:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/template.yml@v0.1.33
    with:
      mode: enforce
    permissions:
      contents: read
```

`template.yml` runs the gates that need nothing but the source: secrets in the history (SEC-08), code style, the Semgrep patterns (SEC-01, SEC-03, SEC-04, SEC-05, CONV-10) and the dependency audit of a committed `composer.lock` (SEC-09).

## Observe and enforce

`.harness.json` sets the `mode`:

| Mode | Effect |
| --- | --- |
| `observe` | Every gate runs and reports. None blocks. This is where a repository starts. |
| `enforce` | A failed gate fails the check. Mark the jobs as required in the branch protection. |

A repository that is ready for most gates but not all lists the others under `observed`. They are reported in the job summary and never block, in `enforce` mode too:

```json
{
    "profile": "core",
    "mode": "enforce",
    "observed": ["Code style", "Rector"]
}
```

The names are the ones `pollora-qa check` prints: `Code style`, `Static analysis`, `Rector`, `Tests and architecture`, `Dependency policy`, `Dependency analysis`, `Security patterns`, `Dependency audit`, `Public surface`, `Ratchet`. Shorten the list as each gate comes clean.

A gate whose tool is not installed does not run, and is not a pass: the summary marks it `no tool`, prints the command that installs the tool, and ends with "Every gate that ran passed, but not every gate ran." `pollora-qa check --strict` makes a missing tool fail the check. Semgrep is the usual case locally; CI installs it.

## What changes in v0.1.33

> **Six rules become blocking: SEC-02, SEC-06, SEC-07, QA-03, CONV-04 and CONV-06.** In `enforce` mode, every repository that raises the harness to v0.1.33 is now stopped by them.
>
> - **SEC-02, SEC-06, SEC-07** (Semgrep) move into the blocking *Security patterns* gate. The harness measured them over 45 repositories of the ecosystem before promoting them: no result, apart from two `nosemgrep` comments that remain valid.
> - **CONV-04, CONV-06** now fail the *Public surface* gate. Before, a badly named hook or command only added a line to the output.
> - **QA-03** (the downstream job) was already blocking in `enforce` mode; only the catalogue said otherwise.
> - The *Security patterns (observed)* gate is gone: one step less in the report.
>
> Also new: the `services` key (`["redis"]`) asks the CI for a Redis, and a cache key, transient or option named `pollora/…` is no longer read as a hook of the public surface.
>
> The catalogue now counts 32 enforced, 1 observed and 7 planned rules.

## The rules

`vendor/bin/pollora-qa rules` prints the catalogue and `pollora-qa rules <ID>` explains one rule. A rule is **enforced** (fails the gate), **observed** (reported only) or **planned** (not checked by this version yet). The gates are G0 (local and agent hooks), G1 (fast pull request checks), G2 (full pull request checks, blocks the merge), G3 (nightly) and G4 (release).

### Architecture (ARCH)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| ARCH-01 | Domain code imports neither Illuminate nor the Infrastructure or UI layers | Pest arch | enforced |
| ARCH-02 | Domain code calls no global WordPress function | PHPStan rule `pollora.noWordPressCallInDomain` | enforced |
| ARCH-03 | Application code depends on the Domain layer only | Pest arch | enforced |
| ARCH-04 | A core package requires neither `illuminate/*` nor `laravel/*`, unless listed in `.harness.json` | `pollora-qa deps` | enforced |
| ARCH-05 | The framework core references no third-party plugin (WooCommerce, ACF, Meta Box, WP Rocket) | Semgrep | enforced |
| ARCH-06 | Only integration packages, templates and projects depend on `pollora/framework` | `pollora-qa deps` | enforced |
| ARCH-07 | Every dependency used is declared, none declared is unused | composer-dependency-analyser | enforced |
| ARCH-08 | Attribute classes are final, readonly and declare their targets | Pest arch | enforced |
| ARCH-09 | `declare(strict_types=1)` in every file | Pest arch and Pint | enforced |

### Conventions (CONV)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| CONV-01 | Expressive API: the common case needs no configuration | Review | planned |
| CONV-02 | A WordPress registration is an attribute or a typed object, never a raw argument array | PHPStan custom rule | planned |
| CONV-03 | Laravel naming: `ServiceProvider`, `Command`, `Exception`, `Repository`, `Factory` suffixes | Pest arch | enforced |
| CONV-04 | Hooks fired by Pollora are named `pollora/<module>/<event_snake_case>` | `pollora-qa surface` | enforced |
| CONV-05 | Configuration lives under `pollora.<package>`; `env()` is only called in `config/` | Pest arch | enforced |
| CONV-06 | Commands are named `pollora:<verb>` or `pollora:<group>:<verb>`; older dashed names stay as aliases | `pollora-qa surface` | enforced |
| CONV-07 | Domain and Application code never calls `app()` or `resolve()` | Pest arch | enforced |
| CONV-08 | A class is final unless it is an extension point | Pest arch | planned |
| CONV-09 | A facade is a thin front for an injectable service; a new facade resolves a contract | Pest arch | planned |
| CONV-10 | One named exception per business case; never throw a bare `\Exception` | Semgrep | enforced |
| CONV-11 | Native types everywhere; PHPDoc only for generics, array shapes, `@internal`, `@deprecated` | PHPStan, Pest type coverage | **observed** |
| CONV-12 | Pest tests are `it()` / `test()` functions, no PHPUnit test class | Pest arch, review | planned |

### Security (SEC)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| SEC-01 | A REST route declared by attribute states its permission; a public route says so | Semgrep | enforced |
| SEC-02 | An AJAX action checks a nonce by default; guest access is explicit | Semgrep | enforced |
| SEC-03 | No `$wpdb` query built by interpolation without `prepare()` | Semgrep | enforced |
| SEC-04 | No unescaped Blade output `{!! !!}` outside a commented allowlist | Semgrep | enforced |
| SEC-05 | No `unserialize`, `eval` or `extract` | Semgrep | enforced |
| SEC-06 | A request value is validated strictly (pattern ends with `\z` or the `D` modifier); a signature, token or secret is compared with `hash_equals()`; thumbnails only accept declared sizes | Semgrep, contract test (media) | enforced |
| SEC-07 | An `admin_post_*` / `wp_ajax_*` closure calls `current_user_can()`; a menu page never uses the `read` capability | Semgrep (closures and menu pages only) | enforced |
| SEC-08 | No secret in the repository | gitleaks | enforced |
| SEC-09 | No dependency with a known vulnerability | `composer audit`, `npm audit` | enforced |

### Performance (PERF)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| PERF-01 | SQL queries per page type never exceed the reference | Golden project | planned |
| PERF-02 | Discovery reads no file when its cache is warm | Pest test in `pollora/discovery` | enforced |
| PERF-03 | No N+1 query in Colt | Pest test in `pollora/colt` | enforced |
| PERF-07 | Boot, cold discovery and thumbnail timings tracked nightly, alert above 10 % | PHPBench | planned |

PERF-02 and PERF-03 run in the own test suites of those two packages. The framework's page budgets and nightly measures are described in [Performance Gates](performance-gates.md).

### Process (QA)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| QA-01 | Ratchet: no metric gets worse (PHPStan level and baseline, coverage, mutation score) | `pollora-qa ratchet` | enforced |
| QA-02 | `.harness.json`, `phpstan-baseline.neon` and `api-surface.json` change only through a person | Agent hooks, CODEOWNERS | enforced |
| QA-03 | A change to a core package keeps the framework green | Downstream job | enforced |
| QA-04 | The public surface changes only with a label: `api-change` to add, `breaking` to remove or change | `pollora-qa surface` | enforced |
| QA-05 | No PHP signature breaks backward compatibility outside a major version | roave/backward-compatibility-check | enforced |
| QA-06 | The release archive carries only what runs: tests, CI, tool configuration and agent files are `export-ignore`d | `pollora-qa archive` | enforced |

## Reading a failure

Every failure quotes the identifier of the rule. These are the blocking rules most often met, with the message you get, its usual cause and the fix. Where the exact wording is not given, the rule's own title (above) is what the tool reports.

| Rule | Message | Usual cause | Fix |
| --- | --- | --- | --- |
| ARCH-02 | `[ARCH-02] The domain calls the WordPress function get_option().` | A `Domain` class calls a global WordPress function | Move the call behind a port (an interface) implemented in the infrastructure layer |
| ARCH-04 | `illuminate/support is a Laravel package: a core package must stay usable without Laravel.` | A `core` package requires `illuminate/*` or `laravel/*` | Depend on a PSR interface or a Pollora port, or list the package under `allow_laravel` in `.harness.json` with the issue that removes it |
| ARCH-06 | `pollora/framework is required by a core package, which sits below the framework.` | A package below the framework requires it | Move the code into an integration package, or invert the dependency through a port |
| ARCH-09 | Pint, or a Pest failure headed `ARCH-09` listing the files | A file lacks `declare(strict_types=1)` | Add it |
| SEC-01 | `This route declares no permission, and pollora/wp-rest then registers it with __return_true` | A `#[WpRestRoute]` without a permission | Pass a `Permission` (`IsLoggedIn`, `IsAdmin`, `Can`) on the class or the method; for a public route, say so |
| SEC-02 | `A hand-registered wp_ajax_* / wp_ajax_nopriv_* hook skips the nonce check that pollora/ajax performs.` | `add_action('wp_ajax_x', …)` written by hand | Declare the action with `#[Ajax('name')]`; for a guest endpoint say so (`nonce: false`, `->withoutNonce()` or `->guest()`) |
| SEC-03 | `$METHOD() receives SQL that does not come from $wpdb->prepare().` | A query built by interpolation | Use `prepare()` with placeholders (`%s`, `%d`, `%i`), or a Colt query |
| SEC-04 | `{!! !!} prints without escaping.` | Unescaped Blade output | Use `{{ }}`; for trusted HTML (already through `wp_kses_post()`) keep `{!! !!}` with a `nosemgrep` comment that says why |
| SEC-05 | `unserialize(), maybe_unserialize(), eval() and extract() run or inject whatever the input holds.` | One of those calls | Use `json_decode()` and explicit assignments |
| SEC-06 | `This pattern ends with $ and has neither the D modifier nor \z` | `/^[a-z]+$/` also accepts `"evil\n"` | End the pattern with `\z`, or add the `D` modifier |
| SEC-06 | `A signature, HMAC, token or secret is compared with an equality operator` | `$token === $given` | `hash_equals($known, $given)`, the known value first |
| SEC-07 | `This admin_post_* / wp_ajax_* handler never calls current_user_can() or user_can().` | A nonce proves the origin of a request, not the right to act | Check the capability the action needs and answer 403 otherwise |
| SEC-07 | `This administration page is registered with an empty or read capability` | `add_menu_page(…, 'read', …)` | Give the capability the page needs (`manage_options`, `edit_posts`, a custom one) |
| CONV-04 | `The hook foo_bar should read pollora/<module>/<event_snake_case>.` | A hook fired by Pollora with another name | Rename it, keeping the old name with a deprecation if it was released |
| CONV-06 | `The command make-thing should start with pollora:.` | A console command without the prefix | Rename it (the old name stays as an alias), or list it under `allow_commands` with a reason |
| CONV-10 | `Throw a named exception for this case … not a bare \Exception.` | `throw new \Exception(…)` | A named exception per case, with a message that says how to fix it |
| QA-01 | The *Ratchet* gate prints a metric as `regressed` | Coverage, PHPStan level or baseline, or mutation score got worse | Fix the cause. Never edit `.harness.json` to lower a threshold |
| QA-02 | `[QA-02] .harness.json is protected` | An agent tried to edit a threshold file | Fix the code instead; a person changes the file |
| QA-04 | `The public surface changes: label the pull request "api-change"` | An attribute, hook, config key or command was added or changed | Label the pull request, then run `pollora-qa surface --update` |
| QA-06 | `N development files would travel in the archive Composer downloads` | `.gitattributes` lacks `export-ignore` lines | `vendor/bin/pollora-qa archive --fix` |
| SEC-09 | `exception expired on 2026-12-18` (npm), or `composer audit` output | A vulnerable dependency, or an expired `npm_audit` exception | Update the dependency; see [Exemptions](#exemptions) |

SEC-08 (gitleaks) names the secret's location and hides its value: rotate the secret, then remove it from the history. QA-03 fails the *downstream* job when the framework's tests break against your change. QA-05 lists the broken PHP signatures; a pull request labelled `breaking` reports it without blocking, since that is a major version.

## Exemptions

An exemption is written down, with its reason, in a place a reviewer sees. `.harness.json` is a protected file (QA-02): a person adds an entry, an agent does not. `init` and `ratchet --update` keep your entries.

### A command that keeps its name (`allow_commands`, CONV-06)

A package whose commands are named for the tool (`new`, `nectar:mcp`) cannot take the `pollora:` prefix without a breaking change. One entry per command, or per family of names, each with a `reason`:

```json
{
    "allow_commands": [
        { "command": "new", "reason": "the installer is named for the tool, like laravel new" },
        { "prefix": "nectar:", "reason": "nectar:* is the name of the MCP server" }
    ]
}
```

An entry gives `command` (the exact name) or `prefix`, not both. The *Public surface* gate prints the command as `keeps its name (allow_commands: …)`; any other command without the prefix is still reported.

### A line of code (`nosemgrep`)

For the Semgrep rules, write the exception on the line it concerns, with the reason after `--`:

```php
// nosemgrep -- the capability is checked by the route middleware, see routes/admin.php
add_action('admin_post_export', function () { /* … */ });
```

The harness only states the form (`// nosemgrep -- reason`, as its README does). `AGENTS.md` forbids a `nosemgrep` comment, a baseline entry or an `@phpstan-ignore` without a written reason on the same line. In PHP, a PHPStan exception reads `// @phpstan-ignore <identifier> (reason)`. Where exactly the comment must sit for Semgrep to honour it (the line of the match, or the one above) is Semgrep's own rule: run `semgrep` once to check it.

### An npm advisory with no fix (`npm_audit.allow`, SEC-09)

The npm audit blocks on every high or critical advisory. When an advisory has no patched version, list it with a reason and an expiry:

```json
{
    "npm_audit": {
        "allow": [
            { "advisory": "GHSA-fc5q-39xc-vjpf", "reason": "ReDoS in picomatch <= 4.0.7, no patched version; build-time only", "expires": "2026-12-18" }
        ]
    }
}
```

- `advisory` is a GitHub identifier (`GHSA-xxxx-xxxx-xxxx`), `reason` is required, `expires` is a date `YYYY-MM-DD` **at most 90 days from today**. Anything else makes `.harness.json` invalid.
- The exception holds through the day it names, then the gate fails with `exception expired on …`.
- One entry covers the packages that only depend on the vulnerable one.
- An exception that matches no advisory any more is reported, so you can remove it.
- If `npm audit` cannot produce a report, the gate fails: a missing report is never read as a clean one.

A template reads the same key from `.harness.json` or, failing that, from `.github/harness.json`; `pollora-qa npm-audit` runs the audit alone.

### An architecture rule a repository cannot satisfy (`allow_rules`)

A rule of `tests/Arch.php` (ARCH-01, ARCH-03, ARCH-08, ARCH-09, CONV-03, CONV-05, CONV-07) can be exempted for the whole repository, with a `reason`:

```json
{
    "allow_rules": [
        { "rule": "ARCH-08", "reason": "the attribute classes are published; making them final is a breaking change" }
    ]
}
```

The tests of that rule are registered as skipped, with the reason printed; every other rule runs. The number of exemptions is a ratchet metric, `exempted_rules`.

### Mutation score (`mutation`)

`check --mutation` runs Pest's mutation testing. By default it mutates the whole repository and the score is held by the ratchet's `msi`. A repository too large for that sets a scope:

```json
"mutation": { "scope": "changed", "min": 80 }
```

| `scope` | Mutates | Held to |
| --- | --- | --- |
| `all` (default) | The whole repository | `ratchet.msi`, which only goes up |
| `changed` | The PHP source files the change touches | `min`, a threshold and not a ratchet |
| `added` | Only the files the change adds | `min` |

The base of the comparison is `HEAD^1`; `--mutation-base=REF` gives another. The score is computed on whole files, so a surviving mutant in a file you touched counts even if you did not write it. A change with nothing to mutate passes.

### Services the tests need (`services`)

```json
{ "services": ["redis"] }
```

The reusable workflow starts a Redis 7 on `127.0.0.1:6379` and installs the `redis` and `igbinary` PHP extensions in the quality and test jobs. The tests read `REDIS_HOST` and `REDIS_PORT`. `redis` is the only known service; any other name makes `.harness.json` invalid.

### The ratchet (`ratchet`)

A metric starts from where the repository is today and may only improve. `.harness.json` records the best value reached for `phpstan_level` (higher is better), `phpstan_baseline`, `exempted_rules` (lower), `coverage`, `type_coverage` and `msi` (higher). `pollora-qa ratchet --update` records improvements and never a regression; only a person lowers a threshold by hand. Line coverage is read from a Clover report (`pollora-qa check --coverage` writes `.harness/coverage.xml`); without a report nothing is compared. Record the first `msi` with `pollora-qa ratchet --update --metric=msi:<score>`, a couple of points under what you measured: the score moves between runs.

## The public surface

Projects rely on more than PHP signatures: the attributes and their parameters, the hooks Pollora fires, the configuration keys and the console commands. `pollora-qa surface --update` records them in `api-surface.json`, one line per entry:

```json
"hook:pollora/media/stored": "hook",
"config:media.disk": "key",
"command:pollora:make:block": "command"
```

Run `vendor/bin/pollora-qa surface` to compare the code with the file. In CI, a pull request that changes the surface needs a label (QA-04):

| Label | When | Release |
| --- | --- | --- |
| `api-change` | It only adds entries | Minor |
| `breaking` | It removes or changes an entry, or adds a required parameter to an attribute | Major, after a deprecation |

The workflow turns the label into `--allow=additions` (`api-change`) or `--allow=all` (`breaking`). The diff then shows exactly what a release changes for its users. `api-surface.json` is protected (QA-02): a person records it. PHP signatures are compared with the latest tag by `roave/backward-compatibility-check` (QA-05).

**What CONV-04 and CONV-06 check.** In the recorded surface, every `hook:` entry must read `pollora/<module>/<event_snake_case>` and every `command:` entry must start with `pollora:`, unless `allow_commands` lists it. A name problem fails the gate whether or not the surface changed. **What they ignore.** A string `pollora/…` that is the first argument of a function that does not name a hook is a key, not a hook: the object cache (`wp_cache_get`, `wp_cache_set`, …), transients (`get_transient`, `set_site_transient`, …) and options (`get_option`, `update_site_option`, …), with their site and multisite variants. `wp_cache_get('pollora/theme_templates', 'pollora')` is not recorded. Every other string counts, whether passed to `do_action()` and `apply_filters()` or not.

## Commands

| Command | Does |
| --- | --- |
| `pollora-qa check` | Every gate |
| `pollora-qa check --fast` | The fast gates (G0) on the files changed in the working tree |
| `pollora-qa check --strict` | Also fails when a gate could not run because a tool is missing |
| `pollora-qa check --coverage[=FILE]` | Runs the tests with coverage and compares the Clover report with the ratchet |
| `pollora-qa check --mutation` | Also runs Pest's mutation testing; `--mutation-base=REF` sets the base for `changed` |
| `pollora-qa ratchet [--update]` | Compares the metrics with `.harness.json`; `--update` records improvements |
| `pollora-qa deps` | Checks the declared dependencies against the profile (ARCH-04, ARCH-06) |
| `pollora-qa surface [--update]` | Compares the public surface with `api-surface.json`; `--update` records it |
| `pollora-qa archive [--fix]` | Checks that development files stay out of the release archive (QA-06) |
| `pollora-qa npm-audit [--config=FILE]` | The npm audit alone (SEC-09) |
| `pollora-qa rules [ID]` | Lists the rules, or explains one |
| `pollora-qa agents` | Regenerates the harness block of `AGENTS.md` |
| `pollora-qa hook <event>` | Agent hooks: `pre-edit`, `post-edit`, `stop` |

`check` writes `.harness/report.json` and, in GitHub Actions, a table in the job summary.

## See also

- [Performance Gates](performance-gates.md) — the framework's query budgets and nightly measures
- [Testing a Pollora Project](testing.md) — unit and feature tests of your own project
- [Nectar — AI Context](nectar.md) — guidelines and skills for coding agents
