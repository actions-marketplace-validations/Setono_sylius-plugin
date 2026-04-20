## Context

`setono/sylius-plugin` is currently a composer dev-dependency meta-package: no PHP source, just pinned tooling and a small PHPStan extension under `phpstan/`. Each Setono Sylius plugin separately maintains a near-identical GitHub Actions workflow. Centralizing the workflow needs a distribution channel that supports versioning, discoverability, and per-job invocation. Two GitHub-native options exist:

- **Reusable workflows** (`on: workflow_call`): support multi-job orchestration and matrices natively, but cannot be listed on the Marketplace.
- **Composite actions**: Marketplace-listable, but cannot define their own job matrices (matrices are job-level constructs in the caller's workflow).

The constraint that drives most of this design: Marketplace listing requires a composite (or JS/Docker) action, and composite actions ship single-job step bundles, not multi-job workflows.

The repo's existing tag convention is "use the tag matching the Sylius version you want" — tags like `2.0.0`, `2.1.0`. Composer consumers depend on `^2.0`. Action consumers will pin the same tags or the floating major.

## Goals / Non-Goals

**Goals:**

- Each CI check (coding standards, dependency analysis, static code analysis, unit tests, integration tests, mutation tests, code coverage) is invokable as its own composite action so the consumer can put each in its own job with its own matrix.
- A root composite action exists primarily to satisfy Marketplace's "one action per repo at the root" listing model and to provide a one-line "run everything" entry point.
- Versioning is reused from the existing tag scheme; consumers pin `@v2` (floating major, with `v` prefix per GitHub Actions convention) or `@2.1.0` (exact, bare-numeric per composer convention).
- Each sub-action is self-contained: cloning the repo at any tag yields a working set of actions with no missing internal references.
- Marketplace listing is publishable on first release without bootstrapping more than once.

**Non-Goals:**

- Generality beyond the Setono Sylius plugin scaffold. Hardcoded paths (`tests/Application/`), hardcoded composer scripts (`check-style`, `analyse`, `phpunit`), and Sylius-specific behavior (removing `sylius/sylius` before static analysis) stay hardcoded.
- A self-test workflow that exercises every action against a fixture plugin. The repo has no plugin code; correctness is validated by a downstream Setono plugin consuming a release-candidate tag.
- Opt-out toggles in the root action. The root runs all seven; consumers needing selective runs use sub-actions directly.
- Splitting actions into a separate repo (e.g., `setono/sylius-plugin-actions`). Shared tagging with the composer package is acceptable cost.

## Decisions

### Decision 1: Composite actions, not a reusable workflow

**Why**: Marketplace doesn't accept reusable workflows. Marketplace presence is the entire point of going through this conversion.

**Alternatives considered**:
- *Reusable workflow only* — keeps multi-job orchestration trivial, but no Marketplace listing.
- *JavaScript or Docker action* — overkill; this is shell orchestration, no runtime needed.

**Trade-off accepted**: Composite actions can't define their own matrices, so each consumer writes the matrix in their workflow. This is fine — it's actually more flexible than baking a matrix into the action.

### Decision 2: Seven sub-actions plus a root, all in `setono/sylius-plugin`

**Why**: Each sub-action handles one logical CI check and can be invoked in its own job for parallel execution. The root exists to (a) satisfy the Marketplace listing requirement (one `action.yml` at the repo root), and (b) give curious users a one-line invocation, even though it's slow.

**Alternatives considered**:
- *Single composite action that runs everything* — Marketplace-able but no per-check parallelism, no per-check matrix.
- *Separate repo (`setono/sylius-plugin-actions`)* — cleaner separation but two repos to maintain, two tag namespaces, two READMEs. The composer package and the action are both "Setono Sylius plugin tooling"; co-locating is acceptable.

**Trade-off accepted**: Tags serve dual purpose (composer release + action release). Bumping the composer dep matrix forces an action retag with no action-side changes (and vice versa). Acceptable for the simplicity gain.

### Decision 3: Inline setup in every sub-action — no shared `setup/` action

**Why**: A shared `setup/` sub-action would be referenced via `setono/sylius-plugin/setup@v2`, which only works after the floating `v2` tag exists. This creates a chicken-and-egg problem: changes to `setup/` are untestable in the same release that introduces them, because the sub-actions referencing it pull in the *previously released* version. Inlining the ~10 lines of checkout + setup-php + composer-install in each of seven actions costs ~70 lines of duplication but eliminates the bootstrap problem entirely and makes each sub-action self-verifying.

**Alternatives considered**:
- *Shared `setup/` sub-action* — DRYer (~60 fewer lines), but introduces tag-bootstrap fragility and an inconsistency: only 4 of 7 sub-actions could use it (the other 3 have pre-install customization), so the abstraction covers ~half the cases.

**Trade-off accepted**: Duplication. When the setup recipe changes (e.g., a new `actions/checkout` major), seven files need editing instead of one. This is rare and grep-able.

### Decision 4: Root invokes all seven sub-actions sequentially

**Why**: The root's primary job is to exist for the Marketplace listing. Nobody is expected to actually use it in serious CI — they'll use sub-actions. Running all seven sequentially is simple and faithful to the listing's name ("run all PHP CI checks").

**Alternatives considered**:
- *Root runs only `coding-standards`* — fast, but misleading for a listing called "PHP CI".
- *Root has `enable-*` toggles* — adds inputs and `if:` complexity for a code path nobody uses.

**Trade-off accepted**: ~5× slower than the equivalent parallel-jobs setup using sub-actions. Documented in README so anyone who does try the root understands.

### Decision 5: Reuse existing tags + maintain a floating `v2` major tag

**Why**: Action ecosystem convention is to pin floating majors (`@v1`, `@v2`). Consumers expect this. Maintaining `v2` alongside `2.x.y` requires one extra command per release (`git tag -fa v2 && git push --force origin v2`) — trivial.

The asymmetry is deliberate: exact tags stay bare-numeric (`2.1.0`) because the existing composer-consumer tag scheme uses bare numerics, but the floating major adopts the `v` prefix because that's what action consumers expect to type. Both conventions are honored; neither is forced onto the wrong audience.

Self-references inside the root action (`setono/sylius-plugin/coding-standards@v2` etc.) point at the floating major so a patch release doesn't require editing the root. Without the floating tag, the root would need a find/replace of all seven references on every patch.

**Alternatives considered**:
- *Exact-tag self-references* — every release requires bumping seven references in the root before retagging. Error-prone.
- *Branch self-references (`@2.x`)* — points at "whatever's latest on the branch right now", which destabilizes consumers between releases.
- *Bare-numeric floating major (`@2`)* — internally consistent with composer tags, but breaks the action ecosystem convention; consumers would have to remember an unusual ref. Rejected after surfacing the inconsistency mid-implementation.
- *Separate `v1` action versioning* — decouples action semver from composer semver but doubles the release surface.

**Trade-off accepted**: Force-pushing the major tag rewrites a ref. Standard practice; no real downside for action consumers, who pin to it specifically because it floats.

### Decision 6: No self-test workflow

**Why**: This repo has no plugin source, no `tests/Application/`, no `phpunit.xml(.dist)` — nothing for the actions to chew on. A meaningful self-test would require building a fixture plugin, which is significant new code for one purpose.

**Alternatives considered**:
- *Fixture plugin under `tests/fixture/`* — real correctness signal, but ongoing maintenance.
- *Smoke test that just verifies each `action.yml` parses* — catches typos but not behavior.

**Trade-off accepted**: First release is validated by wiring a release-candidate tag (e.g., `2.x.y-rc1`) into one real Setono plugin's CI, observing it, then promoting to `2.x.y`. Subsequent changes follow the same pattern. The risk: a bug in a sub-action that the chosen downstream plugin doesn't exercise (e.g., the plugin has no Infection config) ships unnoticed.

### Decision 7: Marketplace-required metadata

The root action.yml must include:
- `name`: globally unique on Marketplace (proposed: "Setono Sylius Plugin CI")
- `description`: one-line summary
- `author`: "Setono"
- `branding`: an icon from the Feather icons subset GitHub allows + an allowed color (proposed: `icon: check-square`, `color: blue`)

Sub-actions don't need `branding` — they're not separately listed.

LICENSE file already exists (MIT) — Marketplace requirement satisfied.

## Risks / Trade-offs

- **Risk: Dual-purpose tags couple two release cadences** → Mitigation: Accept the coupling. A composer-only change can still cut a tag; a tag with no action-relevant change still propagates the floating `2` correctly. Documented in CLAUDE.md so future maintainers know.

- **Risk: First release bootstrap (sub-action self-references point at `@v2` before `v2` exists)** → Mitigation: Documented bootstrap procedure: during initial development, references temporarily point at `@main`; final flip to `@v2` happens in the same commit that gets tagged. After the first release, no further bootstrap is needed.

- **Risk: Hardcoded Sylius scaffold assumptions break for non-conforming consumers** → Mitigation: README explicitly states the scaffold assumptions. Non-conforming consumers should pick sub-actions selectively or fork. Not a goal to support arbitrary PHP projects.

- **Risk: No self-test means bugs ship to consumers** → Mitigation: Release-candidate tags wired into one downstream plugin before promotion. Acceptable for this scope; revisit if release frequency grows.

- **Risk: Root action's slowness damages Marketplace first impression** → Mitigation: README leads with "use the sub-actions directly" examples; root is documented as "quickest way to try it out." User explicitly accepted this trade-off.

- **Risk: Action self-references hardcode `setono/sylius-plugin` — a fork can't substitute its own org** → Mitigation: Acknowledged. Forks would need to find/replace the org name. Not common enough to design around.

- **Risk: `shivammathur/setup-php@v2`, `ramsey/composer-install@v4`, `codecov/codecov-action@v5`, `actions/checkout@v6` are pinned to floating majors and could break unexpectedly** → Mitigation: Standard ecosystem practice. If breakage occurs, pin to a specific minor/SHA in a follow-up patch release.
