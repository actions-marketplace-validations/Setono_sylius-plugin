## 1. Sub-actions

- [x] 1.1 Create `coding-standards/action.yml` with inline setup, six checks (composer validate, normalize, check-style, rector dry-run, yaml lint, twig lint), and inputs `php-version` (default `8.2`), `dependencies` (default `highest`), `extensions` (default `intl, mbstring`)
- [x] 1.2 Create `dependency-analysis/action.yml` with inline setup including `composer-dependency-analyser` in tools, an `unset require-dev` step before install, and inputs `php-version`, `dependencies`, `symfony`, `extensions`
- [x] 1.3 Create `static-code-analysis/action.yml` with inline setup, a `composer remove sylius/sylius` step before install, a `composer analyse` step, and inputs `php-version`, `dependencies`, `symfony`, `extensions`
- [x] 1.4 Create `unit-tests/action.yml` with inline setup, a `composer phpunit` step, and inputs `php-version`, `dependencies`, `symfony`, `extensions`
- [x] 1.5 Create `integration-tests/action.yml` with `sudo /etc/init.d/mysql start` first, inline setup, four Doctrine commands from `tests/Application` with `APP_ENV=test` and `DATABASE_URL` per step, and inputs `php-version`, `dependencies`, `symfony`, `extensions`, `database-url` (default `mysql://root:root@127.0.0.1/sylius?serverVersion=8.0`)
- [x] 1.6 Create `mutation-tests/action.yml` with inline setup using `coverage: pcov` and `tools: infection`, an `infection` step that exposes `STRYKER_DASHBOARD_API_KEY`, and inputs `php-version` (default `8.3`), `dependencies` (default `highest`), `extensions`, `stryker-dashboard-api-key` (default empty)
- [x] 1.7 Create `code-coverage/action.yml` with inline setup using `coverage: pcov`, a phpunit step writing `.build/logs/clover.xml`, a `codecov/codecov-action@v5` upload step, and inputs `php-version` (default `8.3`), `dependencies` (default `highest`), `extensions`, `codecov-token` (required)
- [x] 1.8 Confirm every `run:` step in every sub-action has explicit `shell: bash`
- [x] 1.9 Confirm no sub-action declares a top-level `env:` block

## 2. Root action

- [x] 2.1 Create root `action.yml` with `name: 'Setono Sylius Plugin CI'`, `author: 'Setono'`, a one-line `description`, and `branding: { icon: check-square, color: blue }`
- [x] 2.2 Add inputs to the root: `php-version` (default `8.3`), `php-version-lowest` (default `8.2`), `dependencies` (default `highest`), `symfony` (default `~7.4.0`), `extensions` (default `intl, mbstring`), `database-url` (default `mysql://root:root@127.0.0.1/sylius?serverVersion=8.0`), `codecov-token` (default empty), `stryker-dashboard-api-key` (default empty)
- [x] 2.3 Add seven sequential steps to the root, each invoking a sub-action via the temporary `setono/sylius-plugin/<name>@main` reference (will be flipped to `@v2` before tagging — see task 4.1), passing the appropriate inputs per sub-action
- [x] 2.4 Wrap the `code-coverage` step in `if: inputs.codecov-token != ''` so the root doesn't fail when no Codecov token is provided

## 3. Documentation

- [x] 3.1 Add a top-level "GitHub Actions" section to `README.md` listing all eight actions (root + seven sub-actions) with their reference paths
- [x] 3.2 For each sub-action, document its inputs in a table (name, default, description) and include at least one consumer usage example
- [x] 3.3 Document the root action's inputs and include a single-job "run everything" usage example, with a clear note that it is ~5× slower than sub-actions in parallel jobs
- [x] 3.4 Document the consumer-side scaffold assumptions (presence of `tests/Application/`, the `check-style`/`analyse`/`phpunit` composer scripts, MySQL-backed integration tests, etc.)
- [x] 3.5 Document the matrix-driven invocation pattern (consumer's workflow declares `strategy.matrix` and passes matrix values into a sub-action input)

## 4. First release bootstrap

- [x] 4.1 Find/replace every `setono/sylius-plugin/<sub-action>@main` reference inside the root action.yml to `setono/sylius-plugin/<sub-action>@v2`
- [ ] 4.2 Tag the release `2.x.y` (next release in the existing tag sequence) and push the tag
- [ ] 4.3 Force-push the floating major tag: `git tag -fa v2 -m "Update floating v2 tag" && git push --force origin v2`
- [ ] 4.4 In GitHub's release UI for `2.x.y`, tick "Publish to Marketplace", pick a primary and secondary category
- [ ] 4.5 After the release is live, manually verify the Marketplace listing page loads and shows the root action's branding/description correctly

## 5. Downstream verification

- [ ] 5.1 Wire the new release into one real Setono Sylius plugin's CI workflow (replace its existing workflow with calls to the new sub-actions in a job-per-check matrix layout)
- [ ] 5.2 Push to that plugin's repo and observe CI runs to green; investigate and fix any sub-action bug uncovered (cut a `2.x.(y+1)` patch as needed and re-force-push the `v2` tag)

## 6. Maintenance documentation

- [x] 6.1 Add a "Releasing" section to `README.md` (or `CONTRIBUTING.md` if preferred) documenting: tag the release, force-push the floating major, publish to Marketplace
- [x] 6.2 Update `CLAUDE.md` to note the dual-purpose tag scheme (composer + actions) and the floating major tag invariant, so future maintainers don't accidentally break it
