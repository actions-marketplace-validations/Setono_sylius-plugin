## Why

Setono maintains many Sylius plugins, each with its own copy of a near-identical GitHub Actions workflow (coding standards, dependency analysis, static analysis, unit tests, integration tests, mutation tests, code coverage). Drift between plugins makes CI changes expensive (touch every repo) and inconsistencies are hard to spot. Shipping the workflow as a versioned, GitHub Marketplace-listed action lets every Setono plugin (and any third party with the same plugin scaffold) consume one tested CI implementation pinned by tag.

GitHub Marketplace does not accept reusable workflows — only actions. Composite actions are the right shape for this content because the work is shell orchestration (no JS/Docker runtime needed).

## What Changes

- Add seven composite GitHub Actions to the repository, each in its own subdirectory with an `action.yml`:
  - `coding-standards/` — composer validate/normalize, check-style, rector dry-run, yaml/twig lint
  - `dependency-analysis/` — composer-dependency-analyser against production deps
  - `static-code-analysis/` — `vendor/bin/phpstan analyse`, with `sylius/sylius` removed first
  - `unit-tests/` — `vendor/bin/phpunit`
  - `integration-tests/` — MySQL + doctrine schema validation against tests/Application
  - `mutation-tests/` — infection with optional Stryker Dashboard reporting
  - `code-coverage/` — phpunit with pcov, upload to Codecov
- Add a root `action.yml` that invokes all seven sub-actions sequentially. This is the Marketplace listing's entry point. It is intentionally slow (~5× a parallel matrix) and exists primarily for Marketplace presence; consumers who care about speed use the sub-actions directly with their own job matrices.
- The root action carries `branding:` (icon + color) required for Marketplace listing.
- Each sub-action inlines its own setup (checkout + setup-php + composer-install) — no shared `setup/` sub-action — to avoid the bootstrap-tag problem and keep each sub-action self-contained.
- Self-references between actions use the floating major tag (e.g., `setono/sylius-plugin/coding-standards@v2`), maintained by force-pushing the major tag on each release. The floating major uses the `v` prefix per GitHub Actions ecosystem convention; exact tags stay bare-numeric per the existing composer convention.
- Update `README.md` to document each sub-action, its inputs, and consumer usage examples (per the project rule that new features must be documented in the README).

Versioning: action releases reuse the existing tag scheme (`2.0.0`, `2.1.0`, …) plus a force-pushed floating `v2` major tag (the `v` prefix matches GitHub Actions convention while keeping exact tags bare-numeric for composer).

No `BREAKING` changes: this is purely additive. The repo's existing composer meta-package behavior, `phpstan/extension.neon`, and `.github/workflows/build.yaml` are untouched.

## Capabilities

### New Capabilities

- `ci-composite-actions`: Seven composite GitHub Actions plus a root action that orchestrates them, distributed via this repo's existing tags, listing the root on the GitHub Marketplace.

### Modified Capabilities

(none — no existing specs)

## Impact

- **New files**: `action.yml` (root), `coding-standards/action.yml`, `dependency-analysis/action.yml`, `static-code-analysis/action.yml`, `unit-tests/action.yml`, `integration-tests/action.yml`, `mutation-tests/action.yml`, `code-coverage/action.yml`.
- **Modified files**: `README.md` — add a section documenting the actions and their usage.
- **Unaffected**: `composer.json`, `phpstan/`, `.github/workflows/build.yaml`. The composer meta-package is unchanged; downstream composer consumers see no behavior difference.
- **Release process**: adds a step — after tagging `2.x.y`, also force-push the floating `v2` tag.
- **First release bootstrap**: sub-action self-references in the root action point at `@v2` which won't exist until the first tag is cut. During local iteration the references temporarily point at `@main` (or the dev branch), then are flipped to `@v2` immediately before tagging.
- **External dependencies (action runtime)**: `actions/checkout@v6`, `shivammathur/setup-php@v2`, `ramsey/composer-install@v4`, `codecov/codecov-action@v5`. Pinned to floating majors per action ecosystem convention.
- **Consumer assumptions**: actions assume the standard Setono Sylius plugin scaffold (`tests/Application/`, composer scripts `check-style` / `analyse` / `phpunit`, MySQL-backed integration tests, etc.). This is documented in the README; non-conforming consumers should use the sub-actions selectively or fork.
- **No self-test workflow**: the repo has no Sylius plugin code to chew on, so action correctness is validated by a real Setono plugin consuming the actions before the final tag is cut. A release-candidate tag (e.g., `2.x.y-rc1`) is wired into one downstream plugin to verify before promoting.
