# Quality Gates

[`pollora/harness`](https://github.com/Pollora/harness) is the set of rules, checked by machines, that every Pollora repository runs: code style, static analysis, architecture, tests, security, dependencies and the public contract. The same gates run on a developer's machine (`pollora-qa`), in a coding agent's hooks and in CI, so a change that breaks a rule is stopped before it is merged. This page describes version 0.1.40 of the harness: how to adopt it, which rules it enforces, how to read a failure and how to take an exemption.

It is a development tool. Install it with `--dev`; nothing of it ships to the sites that run your code. It constrains the code of the framework, the packages and the plugins, never how a Pollora project is written.

- [Adopting the harness](#adopting-the-harness)
- [Observe and enforce](#observe-and-enforce)
- [CI](#ci)
- [The rules](#the-rules)
- [Observed rules](#observed-rules)
- [Performance budgets](#performance-budgets)
- [Reading a failure](#reading-a-failure)
- [Exemptions](#exemptions)
- [Mutation testing](#mutation-testing)
- [Services, ratchet and timings](#services-ratchet-and-timings)
- [WordPress integration tests](#wordpress-integration-tests)
- [Adapter contract tests](#adapter-contract-tests)
- [Promoting a rule](#promoting-a-rule)
- [Workflow security](#workflow-security)
- [The public surface](#the-public-surface)
- [Commands](#commands)
- [See also](#see-also)

## Adopting the harness

```bash
composer require --dev pollora/harness
vendor/bin/pollora-qa init --profile=core
composer qa
```

`init` never overwrites a file. It writes `.harness.json`, adds the harness preset to `phpstan.neon`, and creates what is missing: `tests/Arch.php`, `rector.php`, `.github/workflows/harness.yml`, `.claude/settings.json`, `AGENTS.md`, and the `qa` and `qa:fast` Composer scripts. It also adds the `export-ignore` lines that keep those files out of the release archive.

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
    uses: Pollora/harness/.github/workflows/php-package.yml@v0.1.40
    permissions:
      contents: read
```

`php-package.yml` runs the quality gates, a test matrix (PHP 8.3 and 8.4, lowest and stable dependencies), a gitleaks scan, the PHP API comparison and, when `.harness.json` lists `framework` under `downstream` (core packages), the **downstream** job that runs the framework's tests against the change.

### Templates

A template (the skeleton, a theme, a plugin or module template) cannot carry `pollora/harness` in its `require-dev`, since every project created from it would install it. It adds one workflow that calls `template.yml` instead:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/template.yml@v0.1.40
    with:
      mode: enforce   # observe reports and never blocks
      observed: ''    # among secrets, style, security, audit, ratchet
    permissions:
      contents: read
```

`template.yml` runs the gates that need nothing but the source: secrets in the history (SEC-08), code style, the Semgrep patterns (SEC-01 to SEC-07 and CONV-10, over `src`, `app`, `config`, `resources` and `routes`), the audit of a committed `composer.lock` (SEC-09), an npm audit that only reports, and a ratchet job. The rules and the audit code come from the version the workflow is called at.

The ratchet job does nothing unless the template carries a `.harness.json`, at its root or in `.github/harness.json` (left out of release archives). That file holds floors, mostly `ratchet.type_coverage`, and the `npm_audit.allow` list; the workflow's `mode` input decides whether a drop blocks. `pollora-qa init --profile=template [--config=.github/harness.json]` writes it and nothing else.

## Observe and enforce

`.harness.json` sets the `mode`:

| Mode | Effect |
| --- | --- |
| `observe` | Every gate runs and reports. None blocks. This is where a repository starts. |
| `enforce` | A failed gate fails the check. Require `harness / Result` in the branch protection (see [CI](#ci)). |

A repository that is ready for most gates but not all lists the others under `observed`. They are reported in the job summary and never block, in `enforce` mode too:

```json
{
    "profile": "core",
    "mode": "enforce",
    "observed": ["Code style", "Rector"]
}
```

The names are the ones `pollora-qa check` prints: `Code style`, `Static analysis`, `Rector`, `Tests and architecture`, `Dependency policy`, `Dependency analysis`, `Security patterns`, `Dependency audit`, `Public surface`, `Distribution archive`, `Performance budgets`, `Ratchet`, and for a limited mutation run `Mutation testing (changed files)`, `Mutation testing (new files)` or `Mutation testing (changed files, relative)`. Shorten the list as each gate comes clean.

Three steps are always reported and never block, whatever the file says: `Dependency audit (npm)`, `Security patterns (observed)` (SEC-10 to SEC-13) and the whole-repository `Mutation testing` step.

A gate whose tool is not installed does not run, and is not a pass: the summary marks it `no tool`, prints the command that installs the tool, and ends with "Every gate that ran passed, but not every gate ran." `pollora-qa check --strict` makes a missing tool fail the check. Semgrep is the usual case locally; CI installs it.

## CI

### Require one check

Both reusable workflows end with a job named `Result`. With the workflow call that `init` writes (the job is called `harness`), the status check is **`harness / Result`**: require that one in the branch protection or the ruleset, rather than the individual harness jobs. Their names carry the profile, the mode or a PHP version, so requiring them blocks every merge the day one of those changes.

`Result` fails when a job it needs failed or was cancelled. A skipped job (no downstream, no WordPress integration) passes. In observe mode, a failure that is only reported does not fail it. Some things still turn it red in observe mode: a job that cannot read `.harness.json`, a quality job that breaks before or outside the gates (installation, invalid configuration), and any cancelled job, which includes a job that exceeded its time limit. A pull request labelled `breaking` reports the PHP API comparison without blocking.

Do not filter out the whole workflow call (`paths`, or an `if` on the job): the check is then never reported and a pull request that requires it stays pending.

### Which harness version runs

The rules live in the Composer package, not only in the workflow.

| Repository | What CI installs |
| --- | --- |
| No `composer.lock`, workflow called at a release tag (`@v0.1.40`) | That version, in the quality, test and WordPress integration jobs (`composer.json` is not changed) |
| `composer.lock` committed | The quality job installs the locked version and warns when it differs from the tag. The jobs that run `composer update` resolve it anew |
| Workflow called at a branch or a commit, a `dev-` constraint, or a runner that does not give the workflow its tag (GitHub Enterprise Server) | What Composer resolves, with a notice or a warning |

So without a lock, a new rule arrives with the pull request that raises the tag. If the constraint in `composer.json` starts above the tag, the install fails: raise the tag. This holds from the tag `v0.1.40`; a workflow called at an older tag installs the newest release the constraint allows.

> **Upgrading from 0.1.34?** CONV-09, PERF-01 and PERF-07 now block in `enforce` mode, a mutation score the harness cannot trust (mutants not written or never loaded, as can happen with Brain Monkey) is now refused instead of read as a result, and `harness / Result` is the check to require. The [changelog](https://github.com/Pollora/harness/blob/main/CHANGELOG.md) has the rest.

### Time limits

The quality job stops after 30 minutes, and GitHub shows a timed-out job as cancelled. A repository whose full mutation run takes longer raises the limit:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/php-package.yml@v0.1.40
    with:
      quality-timeout-minutes: 90
```

## The rules

`vendor/bin/pollora-qa rules` prints the catalogue and `pollora-qa rules <ID>` explains one rule. A rule is **enforced** (fails the gate), **observed** (reported only, never blocks) or **planned** (not checked by this version yet). The catalogue counts 35 enforced, 8 observed and 1 planned rules. The gates are G0 (local and agent hooks), G1 (fast pull request checks), G2 (full pull request checks, blocks the merge), G3 (nightly) and G4 (release).

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
| CONV-02 | A WordPress registration is an attribute or a typed object, never a raw argument array in the public API | Pest arch (files, AST), review | **observed** |
| CONV-03 | Laravel naming: `ServiceProvider`, `Command`, `Exception`, `Repository`, `Factory` suffixes | Pest arch | enforced |
| CONV-04 | Hooks fired by Pollora are named `pollora/<module>/<event_snake_case>` | `pollora-qa surface` | enforced |
| CONV-05 | Configuration lives under `pollora.<package>`; `env()` is only called in `config/` | Pest arch | enforced |
| CONV-06 | Commands are named `pollora:<verb>` or `pollora:<group>:<verb>`; older dashed names stay as aliases | `pollora-qa surface` | enforced |
| CONV-07 | Domain and Application code never calls `app()` or `resolve()` | Pest arch | enforced |
| CONV-08 | A class is final unless it is an extension point | Pest arch | **observed** |
| CONV-09 | A facade is a thin front for an injectable service; a new facade resolves a contract | Pest arch | enforced |
| CONV-10 | One named exception per business case; never throw a bare `\Exception` | Semgrep | enforced |
| CONV-11 | Native types everywhere; PHPDoc only for generics, array shapes, `@internal`, `@deprecated` | PHPStan, `type_coverage` ratchet | **observed** |
| CONV-12 | Pest tests are `it()` / `test()` functions, no PHPUnit test class | Pest arch, review | **observed** |

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
| SEC-09 | No dependency with a known vulnerability | `composer audit` (blocks), `npm audit` (reported) | enforced |
| SEC-10 | A redirect target never comes from the request unchecked | Semgrep (taint) | **observed** |
| SEC-11 | An outgoing HTTP request never targets a URL taken from the request | Semgrep (taint) | **observed** |
| SEC-12 | A file path, an include or a stream never comes from the request unchecked | Semgrep (taint) | **observed** |
| SEC-13 | What a request sent is escaped where it is printed | Semgrep (taint) | **observed** |

### Performance (PERF)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| PERF-01 | SQL queries per page type never exceed the reference | Budgets file (*Performance budgets* step) | enforced |
| PERF-02 | Discovery reads no file when its cache is warm | Pest test in `pollora/discovery` | enforced |
| PERF-03 | No N+1 query in Colt | Pest test in `pollora/colt` | enforced |
| PERF-07 | Boot, cold discovery and thumbnail timings tracked nightly, alert above 10 % | Scheduled workflow (*Performance budgets* step) | enforced |

PERF-02 and PERF-03 run in the own test suites of those two packages. PERF-01 and PERF-07 are read by the harness itself, see [Performance budgets](#performance-budgets); the framework's page budgets and nightly measures are described in [Performance Gates](performance-gates.md).

### Process (QA)

| Rule | What it requires | Tool | Status |
| --- | --- | --- | --- |
| QA-01 | Ratchet: no metric gets worse (PHPStan level and baseline, coverage, mutation score) | `pollora-qa ratchet` | enforced |
| QA-02 | `.harness.json`, `phpstan-baseline.neon` and `api-surface.json` change only through a person | Agent hooks, CODEOWNERS | enforced |
| QA-03 | A change to a core package keeps the framework green | Downstream job | enforced |
| QA-04 | The public surface changes only with a label: `api-change` to add, `breaking` to remove or change | `pollora-qa surface` | enforced |
| QA-05 | No PHP signature breaks backward compatibility outside a major version | roave/backward-compatibility-check | enforced |
| QA-06 | The release archive carries only what runs: tests, CI, tool configuration and agent files are `export-ignore`d | `pollora-qa archive` | enforced |

### Observed rules

An observed rule reports and never blocks. `tests/Arch.php` registers CONV-02, CONV-08 and CONV-12 as a test that is marked incomplete: the step stays green and prints `observed, not blocking: CONV-08 — N problem(s)` with the list; it passes when nothing is found. A rule with nothing to read (no class, no test file) prints `not evaluated: <rule> — <reason>`. The rules read the files without an autoloader.

| Rule | Reported | Not reported |
| --- | --- | --- |
| CONV-02 | A public method or global function whose array parameter flows into `register_post_type()`, `register_taxonomy()`, `register_block_type()`, `register_block_type_from_metadata()`, `register_rest_route()` (their `$args`) or `add_menu_page()` / `add_submenu_page()`; a public method (interfaces too) named `register…`, `create…`, `define…` or `add…` about a post type, taxonomy, block, route or menu, with an `$args`, `$arguments`, `$config`, `$options` or `$settings` parameter that can hold an array | The infrastructure layer (`Infrastructure`, `Adapter`), non-public methods, `@internal` code, literal arrays built inside a method, object-typed parameters. An array stored then registered elsewhere, or a registration through a wrapper, is out of reach |
| CONV-08 | A class that is not `final` | An abstract class, a builder named `…Builder`, a class that uses `Macroable`, a parent that another class of the repository extends |
| CONV-12 | A concrete class that extends a `…TestCase` and has a test method | `#[RunInSeparateProcess]` / `#[RunTestsInSeparateProcesses]`, the PHPStan rule tests, the `Fixtures` and `Stubs` directories |

CONV-09, enforced, is read the same way: a facade's only method is `getFacadeAccessor()`, which returns `Service::class` or a binding name, and a `Service::class` of the repository must be an interface.

SEC-10 to SEC-13 follow a request value inside one function, not across functions or files. They accept the usual guards, such as `wp_safe_redirect()`, `wp_safe_remote_get()`, `basename()` or `esc_html()`.

A file that does not parse is listed, never skipped silently. CONV-11 has no test of its own: the *Ratchet* step counts the declarations that carry a native type (every return, parameter and property; `mixed` is a type; constructors, closures and arrow functions are out) and prints the percentage, the share of returns, parameters and properties, and the five files that hold most of the gap. The parameters of a method that overrides an untyped WordPress method (`Walker*`, `WP_Widget`) are left out and reported, since a child cannot narrow them. A repository that recorded `type_coverage` in its `ratchet` can no longer lower it: that floor is what CONV-11 will enforce, and `ratchet --update` records it. The count reads the `psr-4`, `psr-0`, `classmap` and `files` entries of `autoload` plus `src` and `app`, minus `vendor`, `node_modules`, `tests` and Blade; it needs neither Pest's type-coverage plugin nor an autoloader.

### Performance budgets

The *Performance budgets* step (PERF-01, PERF-07) reads files and never starts a site. The framework measures its queries and timings in its browser tests; the step checks that those measures are still declared, still wired and never loosened. It blocks like the other gates; a repository that declares neither budgets nor nightly measures is `skipped`.

**PERF-01.** It applies when the repository has a budgets file (`tests/e2e/budgets.json` by default):

```json
{ "pages": { "home": { "wordpress": 24, "laravel": 0 } } }
```

Every budget is a whole number of queries (0 is a real limit) or `null`; a `null` budget is listed as measured-only. A budget that went up, became `null` or was removed since the base commit fails the step. The base is `HEAD^1`, or `check --budgets-base=REF`. When the base is not in the checkout, the file is new there or the base file is unreadable, the step says `not compared with <base>: <why>` and passes on the other checks: it never reports a comparison that did not happen. The reusable workflow checks out two commits so that `HEAD^1` exists.

**PERF-07.** It applies when the repository has the measuring spec (`tests/e2e/specs/perf-metrics.spec.ts`) or declares `performance.nightly`. It checks that the measuring and the comparing files exist, that a workflow has an `on: schedule` trigger with a `cron` and runs both, and that the `--tolerance` given to the comparison is at most 0.1 (an alert above 10 %). It proves the wiring is declared, not that last night passed. A repository that declares neither is `skipped` with the reason.

Where to look is set under `performance` in `.harness.json`, every key optional, paths relative to the repository and without `..`:

```json
{
    "performance": {
        "budgets": "tests/e2e/budgets.json",
        "nightly": {
            "measure": "tests/e2e/specs/perf-metrics.spec.ts",
            "compare": "tests/e2e/bin/perf-compare.ts",
            "workflow": ".github/workflows/ci.yml"
        }
    }
}
```

`"nightly": {}` declares the nightly measures with the default paths, so a repository that lost the files is told so instead of being skipped. Without `workflow`, every file of `.github/workflows` is read.

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
| SEC-09 | `composer audit` output, or `exception expired on 2026-12-18` (npm, reported only) | A vulnerable dependency, or an expired `npm_audit` exception | Update the dependency; see [Exemptions](#exemptions) |

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

The npm audit fails on every high or critical advisory. In a package it is reported and never blocks, and `template.yml` does not block on it either. When an advisory has no patched version, list it with a reason and an expiry:

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
- The exception holds through the day it names, then the audit fails with `exception expired on …`.
- One entry covers the packages that only depend on the vulnerable one.
- An exception that matches no advisory any more is reported, so you can remove it.
- If `npm audit` cannot produce a report, the audit fails: a missing report is never read as a clean one.

A template reads the same key from `.harness.json` or, failing that, from `.github/harness.json`; `pollora-qa npm-audit` runs the audit alone.

Both dependency audits retry when the advisory service cannot be reached (three attempts), then fail: a network incident is never read as a pass.

### An architecture rule a repository cannot satisfy (`allow_rules`)

A rule of `tests/Arch.php` (ARCH-01, ARCH-03, ARCH-08, ARCH-09, CONV-02, CONV-03, CONV-05, CONV-07, CONV-08, CONV-09, CONV-12) can be exempted for the whole repository, with a `reason`:

```json
{
    "allow_rules": [
        { "rule": "ARCH-08", "reason": "the attribute classes are published; making them final is a breaking change" }
    ]
}
```

The tests of that rule are registered as skipped, with the reason printed; every other rule runs. The number of exemptions is a ratchet metric, `exempted_rules`.

**An exemption of an observed rule is declared, not applied.** CONV-02, CONV-08 and CONV-12 may be listed before they are promoted, so that a repository says why it cannot satisfy them on the day they become blocking. While the rule is observed it keeps running and reporting; the step prints `declared, not applied while the rule is observed: CONV-08 — <reason> (allow_rules)`, and the exemption already counts in the ratchet. Once the catalogue says `enforced`, the same line skips the rule.

**Some classes only.** For CONV-02, CONV-08 and CONV-09, a `classes` list limits the exemption, so the rule still judges every other class:

```json
{
    "allow_rules": [
        {
            "rule": "CONV-08",
            "reason": "these models are extended by projects",
            "classes": ["Acme\\Models\\*", "Acme\\Database"]
        }
    ]
}
```

A pattern is a full class name or a namespace prefix ending in `\*`, nothing else. A pattern that exempts nothing is reported as a warning. The ratchet counts these under `exempted_classes`: one for each class a rule would otherwise report.

## Mutation testing

`check --mutation` runs Pest's mutation testing (`pestphp/pest-plugin-mutate`). By default it mutates the whole repository; that step only reports, and the ratchet's `msi` fails a repository that recorded a score and let it drop. A repository too large for that sets a scope:

```json
"mutation": { "scope": "changed", "min": 80 }
```

| `scope` | Mutates | Held to |
| --- | --- | --- |
| `all` (default) | The whole repository | `ratchet.msi`, which only goes up |
| `changed` | The PHP source files the change touches | `min` |
| `added` | Only the files the change adds | `min` |
| `changed-relative` | The files the change touches, on the change and on the base | Each file may not score lower than before |

A limited scope blocks in `enforce` mode. The base is `HEAD^1`; `--mutation-base=REF` gives another. The score is computed on whole files, so a surviving mutant in a file you touched counts even if you did not write it. A change with nothing to mutate passes. `--mutation-scope` sets the scope for one run and wins over the file; a limited scope set that way without `min` is held to 50 %.

The harness refuses a score it cannot trust, for instance when Pest cannot write its mutants or when the test bootstrap prevents them from loading.

### On pull requests

On a `pull_request` event, a repository that recorded `ratchet.msi` and sets no `mutation.scope` mutates only the changed files, at `mutation.min` or 50 %. Every other event mutates the whole repository, so the `msi` ratchet is measured on the main branch: a drop of the whole score shows there, not on the pull request. To keep the full run on pull requests:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/php-package.yml@v0.1.40
    with:
      mutation-on-pull-request: all   # changed (default) or all
```

### `changed-relative` and its limits

```json
"mutation": { "scope": "changed-relative", "min": 80, "tolerance": 2, "min_mutants": 5, "max_files": 3, "max_minutes": 15 }
```

The last four values are the defaults. `min` has no default: set it, because new files, and files whose base cannot be measured, are held to it and are only reported without it.

- A file fails when its score drops by more than `tolerance` points **and** more mutants survive.
- A file with fewer than `min_mutants` mutants is listed as too small to judge.
- A change that touches more than `max_files` source files fails before anything runs.
- Each file takes two passes. When the next one would exceed `max_minutes`, it is not started and the gate fails.
- It does not see a change that only edits tests, nor a drop in a file the change does not touch. Files that git ignores are absent from the base.
- In CI it runs on pull requests only, and not on a release pull request (`develop` into the default branch, from the same repository). On other events, a repository with `ratchet.msi` mutates everything instead; one without it mutates nothing.

## Services, ratchet and timings

### Services the tests need (`services`)

```json
{ "services": ["redis"] }
```

The reusable workflow starts a Redis 7 on `127.0.0.1:6379` and installs the `redis` and `igbinary` PHP extensions in the quality and test jobs. The tests read `REDIS_HOST` and `REDIS_PORT`. `redis` is the only known service; any other name makes `.harness.json` invalid.

### The ratchet (`ratchet`)

A metric starts from where the repository is today and may only improve. `.harness.json` records the best value reached for `phpstan_level` (higher is better), `phpstan_baseline`, `exempted_rules`, `exempted_classes` (lower), `coverage`, `type_coverage` and `msi` (higher). `pollora-qa ratchet --update` records improvements and never a regression; only a person lowers a threshold by hand. A value that cannot be compared makes `.harness.json` invalid: `coverage`, `type_coverage` and `msi` are numbers from 0 to 100, `phpstan_level` an integer from 0 to 10 or `"max"`, the counts whole numbers from 0. Line coverage is read from a Clover report (`pollora-qa check --coverage` writes `.harness/coverage.xml`); without a report nothing is compared. Record the first `msi` with `pollora-qa ratchet --update --metric=msi:<score>`, a couple of points under what you measured: the score moves between runs.

### Where the time goes (`slow_step_share`)

`check` times every step: the console summary, the job summary (a *Time* column and a total) and `.harness/report.json` (`total_seconds`, and the `seconds` of each step) show it. To be warned when one step takes most of the run, set the share of the total above which it is flagged:

```json
{ "slow_step_share": 0.5 }
```

A step that takes more than that share and at least 30 seconds is listed as `Slow step`, and under `slow_steps` in the JSON report. It is a warning, never a failure. Without the key nothing is flagged; a value that is not above 0 and up to 1 makes `.harness.json` invalid.

## WordPress integration tests

Tests that double WordPress are fast but cannot show what only a real one does: hook order, registrations, queries, cron, REST routes. `php-package.yml` has an optional job for that, off by default:

```yaml
jobs:
  harness:
    uses: Pollora/harness/.github/workflows/php-package.yml@v0.1.40
    with:
      wordpress-integration: tests/Integration    # the suite; empty (default) = off
      wordpress-version: latest                   # latest (default), nightly, 6.9, 6.9.1
      wordpress-plugins: action-scheduler         # optional wordpress.org slugs, comma-separated
    permissions:
      contents: read
```

The job, `WordPress integration PHP <version>`, installs WordPress against MySQL 8.0, with `WP_DEBUG` on and cron off, and gives the tests `WP_PATH`, `WP_CORE_DIR`, `WP_TESTS_DB_HOST`, `WP_TESTS_DB_NAME`, `WP_TESTS_DB_USER` and `WP_TESTS_DB_PASSWORD`. It runs the `test:integration` script of `composer.json` if there is one, otherwise Pest with `phpunit.integration.xml` if that file exists, otherwise `vendor/bin/pest <directory>`. It reports in observe mode and blocks in enforce mode.

Start from the stubs:

```bash
mkdir -p tests/Integration
cp vendor/pollora/harness/stubs/wordpress/bootstrap.php.stub tests/Integration/bootstrap.php
cp vendor/pollora/harness/stubs/wordpress/WordPressLoadedTest.php.stub tests/Integration/WordPressLoadedTest.php
cp vendor/pollora/harness/stubs/wordpress/phpunit.integration.xml.stub phpunit.integration.xml
cp vendor/pollora/harness/stubs/wordpress/unit-bootstrap.php.stub tests/bootstrap.php # only if the unit suite has no bootstrap yet
```

```json
"test:integration": "pest --test-directory=tests/Integration -c phpunit.integration.xml"
```

Three things to do:

- Exclude `tests/Integration` from the suites of `phpunit.xml`.
- Declare the doubles of WordPress in the unit bootstrap (`tests/bootstrap.php`, referenced by `phpunit.xml`), not in `tests/Pest.php` or `tests/Helpers/`. Pest loads those before the integration bootstrap, and the real WordPress then fails with `Cannot redeclare class`.
- Run `vendor/bin/pollora-qa archive --fix`, so `phpunit.integration.xml` stays out of the release archive.

## Adapter contract tests

When a port (an interface) has several adapters, a contract test is one suite written against the port and run once per adapter, so a difference between them is a failing case. The adapters are a Pest dataset, one row each, and the cases are `it()` functions. The harness ships a worked example in `stubs/contract/`:

```bash
mkdir -p tests/Contract/Datasets
cp vendor/pollora/harness/stubs/contract/Datasets/Contract.php.stub tests/Contract/Datasets/Contract.php   # the helper, as is
cp vendor/pollora/harness/stubs/contract/Datasets/Stores.php.stub tests/Contract/Datasets/Stores.php       # your adapters
cp vendor/pollora/harness/stubs/contract/StoreContractTest.php.stub tests/Contract/StoreContractTest.php   # your cases
```

Add `<testsuite name="Contract"><directory>tests/Contract</directory></testsuite>` to `phpunit.xml`; there is no workflow input. Test what a caller may rely on, not what an adapter is free to do differently. A difference found and not yet decided goes in the known deviations list (`contractCase()`): the case still runs, is reported as skipped while it fails, and fails the day it passes, so the line gets deleted.

This is a pattern, not a rule: nothing checks that a port has such a suite. It is unrelated to the *PHP API compatibility* job (QA-05) of the workflow.

## Promoting a rule

A rule is born observed. Moving it to enforced turns red every repository that still breaks it, so the move is prepared first:

1. **Measure.** `vendor/bin/pollora-qa promote-report` (`--json` for a machine; paths or `--root=<dir>` give one matrix for several repositories) prints, for each observed rule, what would block in the current repository: `would block (N problems)` with the first five, `clean`, `not applicable` (nothing to read: not a pass) or `no probe` (an observed rule it cannot measure is listed, never left out). It reads files and the recorded ratchet, runs no tool and writes nothing; for the current repository it always exits 0. CONV-02, CONV-08 and CONV-12 run the same reading as their tests and CONV-11 compares the type count with the recorded `type_coverage`. SEC-10 to SEC-13 have `no probe`: read them in the output of `check`.
2. **Declare the motivated exemptions** in `allow_rules`, with their reason (see above).
3. **Record the floors.** `ratchet --update` writes `type_coverage` and `exempted_rules`. From then on a drop fails QA-01 in `enforce` mode and only warns in `observe`.
4. **Promote** by changing the rule's status in the harness catalogue, then release a new version. Each repository moves to it by raising its tag; see [Which harness version runs](#which-harness-version-runs) for the cases where Composer brings the new version earlier.

Steps 1, 3 and 4 are the maintainers' work on the harness itself; the repository that adopts it takes steps 1 to 3 on its own code.

## Workflow security

The reusable workflows (`php-package.yml`, `template.yml`) harden what they run:

- Every third-party action is referenced by the full SHA of its tag's commit, with the tag in a comment. Semgrep is installed at a fixed version and gitleaks runs from an image pinned by digest, not `latest`.
- Least privilege: no permission at the top, `contents: read` per job, `persist-credentials: false` on every checkout, no `pull_request_target`, and no `${{ }}` of a third-party value inside a `run:`.
- Composer authenticates with the job's token, to avoid the `HTTP/2 429` of anonymous downloads; its download cache is only written by runs that are not pull requests. Every job has a time limit, and a new push cancels the pull request's older run.

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
| `pollora-qa check --mutation` | Also runs Pest's mutation testing; `--mutation-base=REF` sets the base for `changed`, `--mutation-scope=all\|changed\|added\|changed-relative` the scope for this run |
| `pollora-qa check --budgets-base=REF` | The commit the query budgets (PERF-01) are compared with (default `HEAD^1`) |
| `pollora-qa check --template` | The ratchet of a template's `.harness.json` |
| `pollora-qa composer-audit [--locked]` | The Composer audit alone (SEC-09) |
| `pollora-qa promote-report [--json] [<path>… \| --root=<dir>]` | For each observed rule, what would block here if it were enforced (read only) |
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
