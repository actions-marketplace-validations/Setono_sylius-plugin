# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`setono/sylius-plugin` is a Composer **meta-package** intended to be installed as a dev-only dependency (`composer require --dev setono/sylius-plugin`) inside Sylius plugins. Its sole purpose is to bundle the curated tooling stack Setono uses to develop Sylius plugins, so each plugin inherits the same linters, static analyzers, and test tooling without pinning them individually.

It has no runtime source code — no `src/`, no `tests/`. The `require-dev` entry on `sylius/sylius` exists only so the bundled PHPStan stubs resolve against a real Sylius codebase during local development of this package.

Everything shipped by this package is either:
- A pinned `require` entry in `composer.json` (PHPStan + its extensions, PHPUnit, Rector, Infection, Sylius Labs coding standard, composer-normalize, dependency-analyser, etc.), or
- The small PHPStan extension under `phpstan/` that is auto-loaded in downstream projects.

## The PHPStan extension

`composer.json` exposes `extra.phpstan.includes` → `phpstan/extension.neon`. When a consuming plugin has `phpstan/extension-installer`, this config is auto-registered and adds:

- `phpstan/stubs/ResourceInterface.stub` — re-declares `Sylius\Resource\Model\ResourceInterface::getId()` with the return type `int|string|null` so plugin code analyzes correctly against the real (untyped) interface.

When touching this extension: edits to `phpstan/extension.neon` use paths relative to that file (`stubs/…`), not to the repo root. Any new stub must be listed in `extension.neon` to take effect.

## Version compatibility

CI (`.github/workflows/build.yaml`) matrix-tests against PHP 8.2 / 8.3 / 8.4 and Symfony 6.4 / 7.4. Dependency version bumps must stay installable across that full matrix. The README's "use the tag that matches the Sylius version you want to use" convention means branch/tag bumps are the signal to consumers — keep that in mind when changing required versions.

## Tags serve two consumers

This repo is both a composer meta-package AND a set of GitHub composite actions (root `action.yml` plus seven sub-actions in subdirectories). Tags are dual-purpose:

- **Composer consumers** depend on `^2.0` and pull in the dev tooling.
- **Action consumers** pin `setono/sylius-plugin@v2` (or a sub-action like `setono/sylius-plugin/unit-tests@v2`).

The root action references its own sub-actions via the floating major tag: `setono/sylius-plugin/coding-standards@v2` etc. This means **every release must force-push the floating major tag** alongside the exact tag, otherwise action consumers stay pinned to the previous patch.

Note the asymmetry: exact tags are bare-numeric (`2.0.0`, `2.1.0` — composer-friendly, matches the historical convention) but the floating major tag uses the `v` prefix (`v2`) to match GitHub Actions ecosystem convention (consumers expect `@v1`, `@v2`, etc.).

Use `bin/release <version>` to cut a release — it enforces the dual-tag invariant (validates the working tree, checks `action.yml` references match the major, then tags + force-pushes both). Never run the underlying tag/push commands by hand unless the script is broken; the whole point is to keep the two consumers in sync without remembering it.

**Never edit the sub-action references in `action.yml` on a routine release.** They pin the floating major specifically so patch releases don't require touching the file. Only edit them on a major bump.

## Common commands

```shell
composer validate --strict     # what CI runs
composer normalize --dry-run   # what CI runs; drop --dry-run to actually reformat composer.json
composer normalize             # reformat composer.json in place
composer update                # refresh composer.lock after dependency changes
```

There is no test suite, no lint target, and no PHPStan run in this repo — those tools exist here only to be re-exported to consumers.

## Documenting features

Every time a feature is added to this library, it MUST be documented in `README.md`. Consumers discover what this package ships via the README, so any new PHPStan stub, bundled tool, or behavior change is not considered complete until the README reflects it.
