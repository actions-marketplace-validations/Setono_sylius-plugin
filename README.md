# Sylius plugin

A **development-only dependency pack** for [Sylius](https://sylius.com) plugin authors.

Installing this single package pulls in the curated tooling stack used to build Sylius plugins — static analysis (PHPStan with Sylius-aware stubs), tests (PHPUnit, Prophecy, Infection), refactoring (Rector), coding standard, and more — so individual plugins don't each have to pin and upgrade those tools themselves.

Install it as a dev dependency in your Sylius plugin, and pin the tag that matches the Sylius version you are targeting:

```shell
composer require --dev setono/sylius-plugin
```

## GitHub Actions

This repository also ships a suite of composite GitHub Actions that implement the Setono Sylius plugin CI pipeline. Each check is its own sub-action, addressable as `setono/sylius-plugin/<name>@<ref>`, so consumers can run each in its own job with its own matrix. There is also a root action `setono/sylius-plugin@<ref>` listed on the GitHub Marketplace that runs all checks sequentially in one job — handy for trying it out, slow for real CI.

Nine actions ship in this repository:

| Action | Purpose |
|---|---|
| `setono/sylius-plugin@<ref>` | Root action. Runs all eight checks sequentially in one job |
| `setono/sylius-plugin/coding-standards@<ref>` | composer validate, normalize, check-style, rector dry-run, yaml/twig lint |
| `setono/sylius-plugin/dependency-analysis@<ref>` | composer-dependency-analyser against production deps |
| `setono/sylius-plugin/static-code-analysis@<ref>` | `vendor/bin/phpstan analyse`, with `sylius/sylius` removed first |
| `setono/sylius-plugin/unit-tests@<ref>` | `vendor/bin/phpunit` |
| `setono/sylius-plugin/functional-tests@<ref>` | MySQL + Doctrine schema validation against `tests/Application` |
| `setono/sylius-plugin/mutation-tests@<ref>` | Infection, with optional Stryker Dashboard reporting |
| `setono/sylius-plugin/code-coverage@<ref>` | PHPUnit with pcov, upload to Codecov |
| `setono/sylius-plugin/backwards-compatibility@<ref>` | Roave backward-compatibility-check against the PR base ref |

Pin the floating major (`@v2`) for automatic patch updates, or pin a specific tag (`@2.1.0`) for full reproducibility. Note the asymmetry: exact tags are bare-numeric (composer convention), the floating major uses the `v` prefix (action ecosystem convention).

### Scaffold assumptions

The actions assume your plugin follows the standard Setono Sylius plugin scaffold. Specifically:

- A test application at `tests/Application/` with `bin/console` available
- Config files at the repo root: `ecs.php` (Easy Coding Standard), `phpstan.neon` / `phpstan.neon.dist` (PHPStan), and `phpunit.xml` / `phpunit.xml.dist` (PHPUnit)
- For functional tests: `tests/Application/` runs in a Symfony `test` env and uses Doctrine with a MySQL-backed connection
- For mutation tests: Infection is configured (typically via `infection.json5`)
- For code coverage: PHPUnit is configured to write a clover report to `.build/logs/clover.xml`

If your plugin doesn't match this scaffold, pick sub-actions selectively or fork.

### Sub-actions

#### `coding-standards`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version. Use the lowest supported version — higher versions can hide syntax errors |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `extensions` | `intl, mbstring` | PHP extensions to install |

```yaml
jobs:
    coding-standards:
        runs-on: "ubuntu-latest"
        steps:
            -   uses: "setono/sylius-plugin/coding-standards@v2"
```

#### `dependency-analysis`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `symfony` | `""` | Symfony version constraint for Flex (e.g. `~7.4.0`). Empty disables the constraint |
| `extensions` | `intl, mbstring` | PHP extensions to install |

The action unsets `require-dev` before installing so the analyser only sees production dependencies.

```yaml
jobs:
    dependency-analysis:
        runs-on: "ubuntu-latest"
        strategy:
            fail-fast: false
            matrix:
                php-version: ["8.2", "8.3", "8.4"]
                dependencies: ["lowest", "highest"]
                symfony: ["~6.4.0", "~7.4.0"]
        steps:
            -   uses: "setono/sylius-plugin/dependency-analysis@v2"
                with:
                    php-version: "${{ matrix.php-version }}"
                    dependencies: "${{ matrix.dependencies }}"
                    symfony: "${{ matrix.symfony }}"
```

#### `static-code-analysis`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `symfony` | `""` | Symfony version constraint for Flex |
| `extensions` | `intl, mbstring` | PHP extensions to install |

The action removes `sylius/sylius` from `composer.json` before install, so analyser output isn't polluted by errors in Sylius's own source.

```yaml
jobs:
    static-code-analysis:
        runs-on: "ubuntu-latest"
        strategy:
            fail-fast: false
            matrix:
                php-version: ["8.2", "8.3", "8.4"]
                dependencies: ["lowest", "highest"]
                symfony: ["~6.4.0", "~7.4.0"]
        steps:
            -   uses: "setono/sylius-plugin/static-code-analysis@v2"
                with:
                    php-version: "${{ matrix.php-version }}"
                    dependencies: "${{ matrix.dependencies }}"
                    symfony: "${{ matrix.symfony }}"
```

#### `unit-tests`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `symfony` | `""` | Symfony version constraint for Flex |
| `extensions` | `intl, mbstring` | PHP extensions to install |
| `testsuite` | `""` | PHPUnit testsuite name to run. Empty runs all suites |

```yaml
jobs:
    unit-tests:
        runs-on: "ubuntu-latest"
        strategy:
            fail-fast: false
            matrix:
                php-version: ["8.2", "8.3", "8.4"]
                dependencies: ["lowest", "highest"]
                symfony: ["~6.4.0", "~7.4.0"]
        steps:
            -   uses: "setono/sylius-plugin/unit-tests@v2"
                with:
                    php-version: "${{ matrix.php-version }}"
                    dependencies: "${{ matrix.dependencies }}"
                    symfony: "${{ matrix.symfony }}"
```

#### `functional-tests`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `symfony` | `""` | Symfony version constraint for Flex |
| `extensions` | `intl, mbstring` | PHP extensions to install |
| `database-url` | `mysql://root:root@127.0.0.1/sylius?serverVersion=8.0` | Symfony `DATABASE_URL` for the test application |
| `node-version` | `20` | Node.js version for the `yarn install` / `yarn build` steps |
| `testsuite` | `""` | PHPUnit testsuite name to run after the setup. Empty skips the phpunit step entirely |

Starts MySQL, installs PHP and Node, builds assets (`yarn install` + `yarn build`), then runs `lint:container`, `doctrine:database:create`, `doctrine:schema:create`, `doctrine:schema:validate -vvv`, and `sylius:fixtures:load -n` from `tests/Application`. If `testsuite` is set, finishes by running `vendor/bin/phpunit --testsuite=<value>`.

```yaml
jobs:
    functional-tests:
        runs-on: "ubuntu-latest"
        strategy:
            fail-fast: false
            matrix:
                php-version: ["8.2", "8.3", "8.4"]
                dependencies: ["lowest", "highest"]
                symfony: ["~6.4.0", "~7.4.0"]
        steps:
            -   uses: "setono/sylius-plugin/functional-tests@v2"
                with:
                    php-version: "${{ matrix.php-version }}"
                    dependencies: "${{ matrix.dependencies }}"
                    symfony: "${{ matrix.symfony }}"
```

#### `mutation-tests`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.3` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `extensions` | `intl, mbstring` | PHP extensions to install |
| `stryker-dashboard-api-key` | `""` | Stryker Dashboard API key. Pass via secrets. Empty skips dashboard reporting (Infection still runs) |

```yaml
jobs:
    mutation-tests:
        runs-on: "ubuntu-latest"
        steps:
            -   uses: "setono/sylius-plugin/mutation-tests@v2"
                with:
                    stryker-dashboard-api-key: "${{ secrets.STRYKER_DASHBOARD_API_KEY }}"
```

#### `code-coverage`

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.3` | PHP version to install |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `extensions` | `intl, mbstring` | PHP extensions to install |
| `codecov-token` | _(required)_ | Codecov upload token. Pass via secrets |
| `testsuite` | `""` | PHPUnit testsuite name to run. Empty runs all suites |

```yaml
jobs:
    code-coverage:
        runs-on: "ubuntu-latest"
        steps:
            -   uses: "setono/sylius-plugin/code-coverage@v2"
                with:
                    codecov-token: "${{ secrets.CODECOV_TOKEN }}"
```

#### `backwards-compatibility`

Wraps [Roave's `backward-compatibility-check`](https://github.com/Roave/BackwardCompatibilityCheck). Compares the PR's diff against its base ref and fails on any public-API break. Inline annotations show up directly on the changed lines via `--format=github-actions`.

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to install |
| `from` | `origin/${{ github.event.pull_request.base.ref }}` | Git ref to compare against. The default only resolves on `pull_request` triggers — pass an explicit ref for other triggers |

The root action invokes this sub-action only on `pull_request` triggers (gated via `if:`), so it's safe to consume the root from any workflow. When invoking this sub-action standalone, scope the workflow to `on: pull_request` (or pass an explicit `from` ref).

```yaml
name: "Backwards compatibility"

on:
    pull_request: ~

jobs:
    backwards-compatibility:
        runs-on: "ubuntu-latest"
        steps:
            -   uses: "setono/sylius-plugin/backwards-compatibility@v2"
```

### Root action

The root action runs all seven sub-actions sequentially in a single job. **It is roughly five times slower than running the same checks as parallel jobs using the sub-actions**, because each sub-action repeats checkout + PHP setup + composer install. The root exists for the GitHub Marketplace listing and as a quick way to try the suite; for real CI, use the sub-actions in parallel jobs.

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.3` | PHP version for most checks |
| `php-version-lowest` | `8.2` | PHP version for `coding-standards` (use the lowest supported version) |
| `dependencies` | `highest` | Composer dependency versions: `lowest` or `highest` |
| `symfony` | `~7.4.0` | Symfony version constraint for Flex |
| `extensions` | `intl, mbstring` | PHP extensions to install |
| `database-url` | `mysql://root:root@127.0.0.1/sylius?serverVersion=8.0` | `DATABASE_URL` for functional tests |
| `codecov-token` | `""` | Codecov upload token. Empty skips the code-coverage step entirely |
| `stryker-dashboard-api-key` | `""` | Stryker Dashboard API key. Empty skips dashboard reporting (Infection still runs) |

```yaml
jobs:
    ci:
        runs-on: "ubuntu-latest"
        steps:
            -   uses: "setono/sylius-plugin@v2"
                with:
                    codecov-token: "${{ secrets.CODECOV_TOKEN }}"
                    stryker-dashboard-api-key: "${{ secrets.STRYKER_DASHBOARD_API_KEY }}"
```

### Releasing

Exact tags are bare-numeric (`2.0.0`, `2.1.0`) for composer compatibility. The floating major tag uses the `v` prefix (`v2`) to match GitHub Actions ecosystem convention. Both must be pushed atomically on every release so composer and action consumers stay in sync.

Run the release script:

```shell
bin/release 2.x.y
```

The script validates the working tree, confirms `action.yml` references match the major being released, fetches origin, prompts for confirmation, then tags `2.x.y` and force-pushes the floating `v2` tag.

After the script completes, open the GitHub release UI for `2.x.y`, tick "Publish to Marketplace", and pick a primary and secondary category.

The root action's sub-action references (`setono/sylius-plugin/<name>@v2`) pin the floating major, so patch releases don't require editing `action.yml`. Only force-push the `v2` tag — never edit the root action's references on each release.

Equivalent manual commands (what `bin/release` runs):

```shell
git tag -a 2.x.y -m "Release 2.x.y" && git push origin 2.x.y
git tag -fa v2 -m "Update floating v2 tag to 2.x.y" && git push --force origin v2
```
