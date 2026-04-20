# ci-composite-actions Specification

## Purpose
TBD - created by archiving change add-ci-composite-actions. Update Purpose after archive.
## Requirements
### Requirement: Repository ships eight independently invokable composite GitHub Actions

The repository SHALL provide eight composite GitHub Actions, each in its own subdirectory at the repo root with an `action.yml` file. Each sub-action MUST be invokable in a consumer workflow as `setono/sylius-plugin/<sub-action-name>@<ref>` and MUST be self-contained (it MUST NOT depend on any other sub-action in this repo).

The eight sub-actions are:
- `coding-standards`
- `dependency-analysis`
- `static-code-analysis`
- `unit-tests`
- `integration-tests`
- `mutation-tests`
- `code-coverage`
- `backwards-compatibility`

#### Scenario: Consumer invokes a single sub-action

- **WHEN** a consumer's workflow declares `uses: setono/sylius-plugin/unit-tests@v2`
- **THEN** GitHub resolves the action from `unit-tests/action.yml` at the floating `v2` tag and runs its steps in the calling job

#### Scenario: Sub-action runs without referencing other sub-actions

- **WHEN** any sub-action's `action.yml` is inspected
- **THEN** none of its steps use a `setono/sylius-plugin/<other>@<ref>` reference (each sub-action inlines its own setup)

### Requirement: Repository ships a root composite action listed on the GitHub Marketplace

The repository SHALL provide a root `action.yml` at the repository root. The root action MUST invoke all seven sub-actions sequentially via their full repo paths and pinned floating major tag (e.g., `setono/sylius-plugin/coding-standards@v2`). The root action MUST include `branding:` (icon and color) compliant with GitHub Marketplace requirements.

#### Scenario: Consumer invokes the root action

- **WHEN** a consumer's workflow declares `uses: setono/sylius-plugin@v2`
- **THEN** all seven sub-actions run in sequence in the calling job, each performing checkout + PHP setup + composer install before its own checks

#### Scenario: Root action is publishable to the Marketplace

- **WHEN** the maintainer publishes a release with the "Publish to Marketplace" option
- **THEN** GitHub accepts the release because the root `action.yml` declares a `name`, `description`, `author`, and a `branding` block with a valid Feather icon and allowed color

### Requirement: Coding standards sub-action runs composer validation, normalization, style, rector, and template linting

The `coding-standards` sub-action SHALL run, in order: `composer validate --strict`, `composer normalize --dry-run`, `vendor/bin/ecs check`, `vendor/bin/rector process --dry-run`, `bin/console lint:yaml ../../.github ../../config ../../translations` (from `tests/Application`), and `bin/console lint:twig ../../templates` (from `tests/Application`). It SHALL accept inputs `php-version` (default the lowest supported), `dependencies` (default `highest`), and `extensions` (default `intl, mbstring`).

#### Scenario: Default invocation against the standard plugin scaffold

- **WHEN** a consumer invokes `setono/sylius-plugin/coding-standards@v2` with no inputs
- **THEN** the action runs all six checks against the consumer's checked-out repository using the default PHP version

#### Scenario: Consumer overrides PHP version

- **WHEN** a consumer invokes the action with `php-version: '8.4'`
- **THEN** PHP 8.4 is installed and used for all six checks

### Requirement: Dependency analysis sub-action runs composer-dependency-analyser against production dependencies

The `dependency-analysis` sub-action SHALL unset the consumer's `require-dev` block before installing composer dependencies, then run `composer-dependency-analyser`. It SHALL accept inputs `php-version`, `dependencies`, `symfony` (Symfony version constraint passed to Flex), and `extensions`. The `composer-dependency-analyser` tool MUST be installed via the setup-php action's `tools` input.

#### Scenario: Analyser sees only production dependencies

- **WHEN** the action is invoked
- **THEN** `composer config --unset require-dev` runs before `composer install`, so the analyser cannot mistake dev-only packages for production usage

### Requirement: Static code analysis sub-action removes sylius/sylius before running PHPStan

The `static-code-analysis` sub-action SHALL run `composer remove --dev --no-install --no-update --no-plugins --no-scripts sylius/sylius` before installing composer dependencies, then run `vendor/bin/phpstan analyse`. It SHALL accept inputs `php-version`, `dependencies`, `symfony`, and `extensions`.

The Sylius removal step exists because Sylius's source code triggers analyser errors that pollute plugin-level analysis output.

#### Scenario: Analyser runs without Sylius source in the autoloader

- **WHEN** the action is invoked
- **THEN** `sylius/sylius` is removed from `composer.json` before `composer install`, and `vendor/bin/phpstan analyse` runs against the remaining dependencies plus the consumer's plugin code

### Requirement: Unit tests sub-action runs PHPUnit

The `unit-tests` sub-action SHALL install composer dependencies, then run `vendor/bin/phpunit`. It SHALL accept inputs `php-version`, `dependencies`, `symfony`, and `extensions`.

#### Scenario: Action runs PHPUnit against installed dependencies

- **WHEN** the action is invoked
- **THEN** `vendor/bin/phpunit` runs using the consumer's `phpunit.xml` (or `phpunit.xml.dist`) configuration

### Requirement: Integration tests sub-action validates Doctrine schema against a live MySQL instance

The `integration-tests` sub-action SHALL start the runner's pre-installed MySQL service, install composer dependencies, then run, in order: `bin/console lint:container`, `bin/console doctrine:database:create`, `bin/console doctrine:schema:create`, `bin/console doctrine:schema:validate -vvv` — each from `tests/Application`. It SHALL accept inputs `php-version`, `dependencies`, `symfony`, `extensions`, and `database-url` (default `mysql://root:root@127.0.0.1/sylius?serverVersion=8.0`). The `database-url` input MUST be exposed as `DATABASE_URL` to each Doctrine command. `APP_ENV` MUST be set to `test` for each Doctrine command.

#### Scenario: Schema validation runs against a real database

- **WHEN** the action is invoked with default `database-url`
- **THEN** MySQL is started, the test database is created with the schema, and `doctrine:schema:validate -vvv` reports any missing or extra SQL statements

#### Scenario: Consumer overrides database connection

- **WHEN** the action is invoked with a custom `database-url` input
- **THEN** that URL is used for all four Doctrine commands

### Requirement: Mutation tests sub-action runs infection with optional Stryker Dashboard reporting

The `mutation-tests` sub-action SHALL install composer dependencies and run `infection`. It SHALL accept inputs `php-version` (default `8.3`), `dependencies` (default `highest`), `extensions`, and `stryker-dashboard-api-key` (default empty string). The `infection` tool MUST be installed via setup-php's `tools` input. PHP coverage driver MUST be `pcov`. The `stryker-dashboard-api-key` input MUST be exposed as the `STRYKER_DASHBOARD_API_KEY` environment variable to the `infection` step.

#### Scenario: Action runs without dashboard reporting

- **WHEN** the action is invoked without `stryker-dashboard-api-key`
- **THEN** `infection` runs with an empty `STRYKER_DASHBOARD_API_KEY`, and no dashboard upload occurs

#### Scenario: Action runs with dashboard reporting

- **WHEN** the action is invoked with a `stryker-dashboard-api-key` input
- **THEN** `infection` runs and pushes results to the Stryker Dashboard using the provided key

### Requirement: Code coverage sub-action runs phpunit with pcov and uploads to Codecov

The `code-coverage` sub-action SHALL install composer dependencies, run `vendor/bin/phpunit --coverage-clover=.build/logs/clover.xml`, then upload the resulting clover file via `codecov/codecov-action@v5`. It SHALL accept inputs `php-version` (default `8.3`), `dependencies` (default `highest`), `extensions`, and `codecov-token` (required). PHP coverage driver MUST be `pcov`. The `codecov-token` input MUST be passed to the codecov action's `token` parameter.

#### Scenario: Coverage is collected and uploaded

- **WHEN** the action is invoked with a valid `codecov-token`
- **THEN** clover coverage is generated and uploaded to Codecov successfully

### Requirement: Backwards compatibility sub-action runs Roave's backward-compatibility-check against the PR base ref

The `backwards-compatibility` sub-action SHALL check out the consumer's repo with `fetch-depth: 0`, install PHP, install `roave/backward-compatibility-check` via `composer global require`, then run `~/.composer/vendor/bin/roave-backward-compatibility-check --from=<from> --format=github-actions`. It SHALL accept inputs `php-version` (default `8.2`) and `from` (default `origin/${{ github.event.pull_request.base.ref }}`). The action does not install custom PHP extensions because Roave's tool inspects source files and needs nothing beyond a standard PHP build.

The sub-action itself does not gate on event type. The root action MUST gate its invocation of `backwards-compatibility` with `if: github.event_name == 'pull_request'`, so the root remains safe to consume from any workflow. Standalone consumers SHALL scope their workflow to `on: pull_request` or pass an explicit `from` ref.

#### Scenario: PR-triggered invocation against the base ref

- **WHEN** the action is invoked from a `pull_request`-triggered workflow with no `from` input
- **THEN** the BC check runs against `origin/<base-ref>` and any public-API regression is reported as a GitHub Actions inline annotation on the offending source line

#### Scenario: Consumer overrides the comparison ref

- **WHEN** the action is invoked with an explicit `from` input (e.g., `from: 'origin/main'`)
- **THEN** the BC check runs against the provided ref instead of the PR base ref, allowing the action to work outside `pull_request` triggers

#### Scenario: Root action skips backwards-compatibility on non-PR triggers

- **WHEN** the root action is invoked from a non-`pull_request` trigger
- **THEN** the root's `if: github.event_name == 'pull_request'` guard skips the backwards-compatibility step entirely, so non-PR runs of the root succeed regardless of the BC sub-action's PR requirement

### Requirement: All actions follow GitHub composite-action constraints

Every `run:` step in every action SHALL declare `shell: bash`. No action SHALL declare a top-level `env:` block (composite actions don't support it). Environment variables that need to span multiple steps SHALL be set per-step via `env:` on each `run:` step that needs them.

#### Scenario: Composite action validates against GitHub's schema

- **WHEN** GitHub Actions parses any `action.yml` in the repo
- **THEN** parsing succeeds because every `run:` step declares `shell: bash` and no top-level `env:` is present

### Requirement: Versioning reuses existing tag scheme with a force-pushed floating major

Action releases SHALL reuse the repository's existing tag scheme (e.g., `2.0.0`, `2.1.0`). On every release, the maintainer SHALL also force-push a floating major tag (e.g., `v2`) pointing at the same commit. The floating major MUST use the `v` prefix to match GitHub Actions ecosystem convention, while exact tags stay bare-numeric to match the existing composer convention. Self-references between the root action and sub-actions SHALL pin the floating major tag, not an exact tag.

#### Scenario: Patch release does not require editing self-references

- **WHEN** the maintainer cuts release `2.1.1` and force-pushes the `v2` tag
- **THEN** the root action's references like `setono/sylius-plugin/coding-standards@v2` automatically resolve to `2.1.1` without any file edits

### Requirement: README documents every sub-action and its inputs

`README.md` SHALL contain a section documenting each of the eight sub-actions and the root action: action reference path, all inputs (with defaults and descriptions), and at least one consumer usage example. Per the project's existing rule, no new feature is considered complete until the README reflects it.

#### Scenario: Consumer reads README to learn how to invoke the actions

- **WHEN** a Sylius plugin author opens `README.md`
- **THEN** they find documented invocation paths, input contracts, and matrix-driven usage examples for each sub-action

