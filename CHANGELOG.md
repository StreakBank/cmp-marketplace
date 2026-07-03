# Changelog

## 2.11.0 — 2026-07-03

### Fixed — client-error pattern (generic correctness)
- **`cmp-scaffold`** — generated ViewModels/Screens now emit a one-shot `Channel<UiMessage>(BUFFERED).receiveAsFlow()` message stream + a `userMessageFor(throwable, default)` mapper seam, replacing `errorEvents: MutableSharedFlow<String>` (which replays on config change and mixes success/error/empty strings) and raw `throwable.message` piped to the UI (which leaks technical text / URLs). A neutral Material `SnackbarHost` remains the default host — projects can swap it. New `UiMessage` type + `userMessageFor` seam documented in `code-templates.md`; updates threaded through `data-patterns.md`, `analytics-patterns.md`, `deep-linking-patterns.md`, `test-patterns.md`, and the `scaffold-feature`/`add-screen`/`scaffold-tests`/`add-analytics`/`polish-ui` skills.
- **`cmp-quality`** — `audit-architecture` Check 7, `review-changes`, `validate-module`, `validate-tests` now enforce that generic contract (one-shot typed message stream + `userMessageFor`; no raw `throwable.message`) instead of hard-requiring `MutableSharedFlow<String>` + a *Material* host. The message host is now project-defined — a custom host satisfies the check.

### Fixed — cmp-quality internal contradictions
- **`validate-module`** §2.5 — DI check accepts `InMemory*`, `Room*`, or `Ktor*` as the `*LocalDataSource` binding (was hard-coded to `InMemory*`, false-failing modules migrated via `add-room-database`/`add-ktor-networking`).
- **`validate-module`** / **`dependency-audit`** — the `feature → data:api` direction is now domain-layer-aware and stated once: when a `domain` module exists the feature depends on `domain` (per `add-domain-layer`) and the direct `data:api` edge is optional; `dependency-audit` Check 8 supersedes Check 1 in that case. Resolves the Check 1↔8 contradiction and the `:core:feature` severity mismatch.
- **`validate-module`** Phase 4 — defers cross-module dependency/registration checks to `dependency-audit` (single source of truth) instead of duplicating them, mirroring `audit-architecture`.

### Fixed — cmp-design-bridge decoupling + staleness
- Removed project-coupling leaks from the (generic) skill bodies: `design-transform` no longer names StreakBank/Roborazzi/`recordRoborazziDebug`/`compose-canvas-dp.md` (now neutral `<project screenshot tool>` / `<gate-record command>` placeholders); `design-push` no longer names the retired `streakbank-ux` repo or a project memory file.
- Removed the dead `$CLAUDE_PLUGIN_ROOT/bin/*.mjs` fallback + in-plugin `npm link` from `design-pull`/`design-fidelity`/`design-transform` (the CLI is the published `cmp-design-bridge` npm package); install is now just `npm i -g cmp-design-bridge`.
- Added a `design-transform` ↔ `polish-ui` visual-authority boundary note (don't run both on one screen; `design-transform` owns screens with an authoritative design frame).

### Added — mechanized discipline (governance)
- **`scripts/check-coupling.sh`** — the coupling/manifest admission gate (ported + adapted from agent-marketplace): scans `skills/` **and** `agents/` for project-name / absolute-path / named-rule leaks, with carve-outs for the org name in repo-address lines + `author`/`owner` JSON and for the `cmp-design-bridge` npm install address. Shellcheck-clean.
- **`.github/workflows/ci.yml`** — CI admission gate: coupling check + shellcheck + JSON-manifest validation on every push/PR.
- **`CONTRIBUTING.md`** — the generic-core-only contribution discipline for this marketplace's three plugins.
- **`plugins/cmp-scaffold/PROVENANCE.md`** — declares the `references/` content authored-original (nothing vendored), satisfying the gate's provenance check honestly.

### Changed
- Plugin versions: `cmp-scaffold` 2.9.0 → 2.10.0, `cmp-quality` 2.9.0 → 2.10.0, `cmp-design-bridge` 0.1.0 → 0.2.0. Marketplace 2.10.0 → 2.11.0.

## 2.10.0 — 2026-06-30

### Added
- **`cmp-design-bridge`** plugin (v0.1.0) — thin skills wrapping the published `cmp-design-bridge` npm CLI: `design-pull`, `design-transform`, `design-fidelity` (steady-state Claude Design → Compose loop) + the gated `design-push` (one-time port INTO Claude Design). (This entry backfills the marketplace 2.10.0 bump, which shipped the plugin at commit `1a4d907` without a CHANGELOG line.)

## 2.9.0 — 2026-02-09

### Fixed
- **P1: `code-templates.md`** — removed deprecated `compose.components.resources` shorthand from build.gradle.kts template (only shows `libs.compose.components.resources` now)
- **P1: `adaptive-layouts-patterns.md`** — clarified that `1.10.0-alpha05` navigation-suite version is latest available, not a stable release
- **P1: `add-analytics`** — added `context: fork` (skill scans all ViewModels and screens across the project)

### Changed
- **`audit-architecture`** — merged Check 4/4b into single "Check 4: Design Token Compliance (dp + alpha)" with 4a/4b sub-items
- **`performance-audit`** — downgraded Check 1 (`@Stable`/`@Immutable` on data classes) from WARN to INFO (unnecessary but not harmful with strong skipping)
- **`accessibility-audit`** — clarified Check 6 (testTag) is for test automation, not screen reader accessibility
- **`validate-tests`** — downgraded mock framework usage from FAIL to WARN (MockK is acceptable for cross-platform)
- **`ui-recipes-loading.md`** — moved `ShimmerPlaceholderFraction` constant definition before its first use
- **`ui-recipes-cards.md`** — added inline comment explaining `Color.White` image overlay exception
- **`add-navigation-tab`** — updated description from "bottom navigation" to cover adaptive layouts (bottom bar, rail, drawer)
- **`compose-performance.md`** — fixed reduce motion reference to point directly to `ui-recipes-surfaces.md` instead of `ui-recipes.md`
- **6 skills** gained verification checklists: `add-screen`, `add-navigation-tab`, `add-remote-datasource`, `add-ktor-networking`, `add-image-loading`, `add-domain-layer`
- **All version numbers** aligned to 2.9.0 across `marketplace.json`, both `plugin.json` files, and CHANGELOG

## 2.8.0 — 2026-02-09

### Added
- **5 new skills:** `add-permissions` (runtime permission handling), `add-deep-linking` (URI schemes + App Links), `add-analytics` (Firebase Analytics + Crashlytics), `add-background-sync` (WorkManager + BGTaskScheduler), `add-push-notifications` (FCM + APNs)
- **5 new reference files:** `permissions-patterns.md`, `deep-linking-patterns.md`, `analytics-patterns.md`, `background-work-patterns.md`, `push-notifications-patterns.md`

### Changed
- **`add-remote-datasource`** — sharpened description to improve auto-invocation disambiguation with `add-ktor-networking`
- **`add-ktor-networking`** — sharpened description to clarify mock-replacement scope
- **`scaffold-tests`** — sharpened description to clarify post-business-logic timing
- **`polish-ui`** — strengthened selective recipe loading directive (do NOT load inapplicable recipe modules)
- **`add-remote-datasource`** — intro now references code-templates.md directly instead of routing through data-patterns.md (eliminates extra tool turn)
- **`convention-plugins-patterns.md`** — added explicit placeholder replacement note before `gradlePlugin` block
- **`adaptive-layouts-patterns.md`** — clarified alpha-stable status (alpha by semver, stable for production within CMP 1.10+)
- **`CLAUDE.md`** — added Skill Dependency Diagram showing intended workflow order and chaining; documented argument-hint convention (app-level skills omit it)
- **All version numbers** aligned to 2.8.0 across `marketplace.json`, both `plugin.json` files, and CHANGELOG

## 2.7.0 — 2026-02-09

### Fixed
- **P0: `domain-layer-patterns.md`** — convention plugin ID `kmp-library-convention` → `{package_base}.kmp.library` (was build-breaking mismatch with `convention-plugins-patterns.md`)
- **P0: `ui-recipes-cards.md`** — documented image overlay scrim exception for `Color.Black`/`Color.White` usage (was contradicting theming-patterns.md "never hardcode colors" rule)
- **P0: `scaffold-feature`** — added `Edit` to `allowed-tools` (Step 8 modifies existing files); removed unjustified `Bash`
- **P0: `theming-patterns.md`** — added image overlay scrim exception clause to the "no hardcoded colors" rule
- **P1: `ui-recipes-forms.md`** — replaced hardcoded English strings (`"Hide password"`, `"Name is required"`, `"Invalid email"`) with `stringResource()` / `getString()` calls
- **P1: `ui-recipes-lists.md`** — removed stale `@OptIn(ExperimentalFoundationApi::class)` on `stickyHeader` (stable in CMP 1.10)
- **P1: `code-templates.md`** — removed dead link to deleted `project-context.md`; fixed deprecated `compose.components.uiToolingPreview` and `compose.components.resources` shorthand to version catalog references
- **P1: `persistence-patterns.md`** — fixed `com.example` hardcoded imports to `{package_base}` placeholder (2 occurrences)
- **P1: `data-patterns.md`** — removed stale `Retrofit` mention (marketplace is Ktor-only)
- **P1: `compose-performance.md`** — fixed `com.example` in stability config example to `{package_base}`
- **P1: `adaptive-layouts-patterns.md`** — removed unused `NavigationSuiteScaffoldDefaults` import
- **P1: `add-navigation-tab`** — added `NavigationSuiteScaffold` detection guidance for adaptive layout compatibility
- **P1: `performance-audit`** — documented threshold difference vs `accessibility-audit` (500ms performance vs 150ms WCAG)

### Added
- **`theming-patterns.md`** — DataStore version catalog entries (`datastore-preferences` 1.1.7)
- **`code-templates.md`** — `kotlinx-serialization` plugin version catalog entry for `@Serializable` routes
- **`design-tokens.md`** — layout constraint breakpoint exception clause for raw dp rule
- **`ui-recipes-loading.md`** — named constants `ShimmerTranslateRange` and `ShimmerPlaceholderFraction` (replaced raw floats)
- **`ui-recipes-surfaces.md`** — named constant `ScaleOnPressFactor` (replaced raw `0.96f`)

### Changed
- **All 7 agents** — standardized verdict terminology to `PASS / NEEDS FIXES` across all agents (was VALID/PASS/GOOD inconsistency); added verdict line to `audit-architecture` and `performance-audit` (previously missing)
- **`audit-architecture`** — renumbered Check 13 → Check 10 (closed gap from v2.5.0 delegation of Checks 10-12)
- **`review-changes`** — standardized summary to use `PASS` instead of `Clean` for issue-free files
- **`cmp-scaffold/plugin.json`** — description updated to reflect all 16 skills (was listing only 9)
- **`cmp-quality/plugin.json`** — description updated to reflect all 7 agents (was listing only 5)
- **`README.md`** — fixed installation commands (`plugin marketplace add`, `@cmp-marketplace --scope project`); Quick Start step 13 now correctly triggers `validate-module` for single-module validation

## 2.5.0 — 2026-02-09

### Changed
- **`polish-ui`** — replaced verbose design assessment (10 lines of philosophy) with concise 2-line directive, moved optional enhancements (pull-to-refresh, reduce motion, connectivity) into compact gated section, renumbered core steps from 19 to 16
- **`audit-architecture`** — removed AnimatedContent contentKey check (deferred to `performance-audit`), renumbered checks to fill gap from previously removed Check 13, sharpened description for mutual exclusivity with `validate-module`
- **`review-changes`** — removed AnimatedContent contentKey check (deferred to `performance-audit`), added cross-reference notes to `performance-audit` and `accessibility-audit`
- **`validate-module`** — sharpened description to clarify single-module scope vs `audit-architecture` project-wide scope
- **`validate-tests`** — Phase 1 Check 1.5 now explicitly verifies Turbine is present in test dependencies (required by Phase 3)
- **`design-tokens.md`** — clarified `AnimDuration.long` (500ms) is for non-blocking background animations only, not screen transitions; added cross-reference to performance budgets
- **`compose-performance.md`** — clarified screen transition budget (≤ 300ms) and relationship to `AnimDuration.long`
- **`scaffold-feature`** — inlined project context detection (was referencing deleted `project-context.md`)
- **`scaffold-tests`** — moved `test-patterns.md` to shared `references/` directory; updated all reference paths

### Fixed
- **`extract-strings`** — added `Write` to `allowed-tools` (was unable to create `strings.xml` for modules that don't have one)

### Removed
- **`architecture-rules.md`** — deleted 135-line documentation-only file never referenced by any agent or skill (agents inline their own criteria)
- **`project-context.md`** — deleted 37-line file redundant with inline context detection in every skill

## 2.4.0 — 2026-02-09

### Added
- **2 new skills:** `add-theming` (Material 3 theming with color scheme, typography, dark mode toggle, DataStore persistence) and `add-adaptive-layout` (adaptive NavigationSuiteScaffold with WindowSizeClass detection)
- **1 new agent:** `performance-audit` — audits composable performance patterns including strong skipping compliance, reference stability, AnimatedContent contentKey, lazy list keys, image sizing, ViewModel init patterns, and performance budget indicators
- **1 new reference file:** `domain-layer-patterns.md` — use case patterns extracted from `add-domain-layer` skill (module structure, Flow/suspend use cases, DI module, ViewModel integration, dependency chain)
- **`ui-recipes.md`** — new Forms & Input section with validated text fields, password field with visibility toggle, form state in ViewModel (MutableStateFlow.update pattern), and form submission with loading spinner

### Changed
- **`add-domain-layer`** — replaced 4 inline code blocks with references to `domain-layer-patterns.md` (progressive disclosure alignment)
- **`audit-architecture`** — sharpened description to specify UiState patterns, DI bindings, design tokens, and navigation checks
- **`dependency-audit`** — sharpened description to specify layering violations, circular deps, and registration completeness
- **`validate-module`** — sharpened description to specify file structure, data layer, feature layer, and cross-module wiring
- **`data-patterns.md`** — trimmed Pattern 3 (Room) and Pattern 4 (Ktor) sections to single-line pointers to dedicated reference files (~60 lines removed)
- **`fix-imports`** — added next steps suggesting `extract-strings` and `review-changes`
- **`extract-strings`** — added next steps suggesting `fix-imports` and `review-changes`
- **`upgrade-dependencies`** — added next steps suggesting `dependency-audit` and build verification

### Fixed
- **`marketplace.json`** — version updated from stale `2.0.0` to `2.4.0`
- **`plugin.json`** — both plugins bumped from `2.3.0` to `2.4.0`
- **`add-navigation-tab`** — added `Write` to `allowed-tools` (needed for creating TopLevelDestination.kt on first tab)
- **README.md** — skill count updated from 14 to 16; agent count updated from 6 to 7; added new skill and agent rows; added theming and adaptive layout to Quick Start

## 2.3.0 — 2026-02-09

### Added
- **1 new skill:** `add-domain-layer` — scaffolds a domain/use-case layer for a feature module, extracting business logic from ViewModels into focused use case classes with proper DI wiring
- **1 new agent:** `validate-tests` — audits test quality, structure, and patterns (test doubles, Turbine usage, dispatcher setup, error path coverage)

### Changed
- **`polish-ui`** — now uses `context: fork` for isolated execution (scan-heavy, matches other fork skills)
- **`audit-architecture`** — removed partial accessibility check (Check 13); directs users to the dedicated `accessibility-audit` agent for comprehensive coverage
- **`validate-module`** — added networking (K1–K3) and persistence (P1–P3) pattern checks to Phase 2 data layer validation
- **`scaffold-feature`** — report now suggests `add-screen`, `add-remote-datasource`, `scaffold-tests`, and `polish-ui` as next steps
- **`add-remote-datasource`** — report now suggests `add-ktor-networking`, `scaffold-tests`, and `polish-ui` as next steps
- **`add-ktor-networking`** — report now suggests `add-room-database`, `scaffold-tests`, and `polish-ui` as next steps
- **`add-room-database`** — report now suggests `scaffold-tests` and `polish-ui` as next steps
- **`architecture-rules.md`** — reframed header to clarify documentation-only purpose (agents inline their own criteria)
- **`compose-performance.md`** — added reduce motion guidance to Key Rules
- **`theming-patterns.md`** — added `AnimDuration.kt` and `AnimEasing.kt` to design system file listing
- **`data-patterns.md`** — removed duplicate "When to Use Which" table; fixed `SqlDelight` reference to `Room`

### Fixed
- **README.md** — skill count updated from 12 to 14; added missing `polish-ui` and `add-domain-layer` to table; agent count updated to 6
- **`plugin.json`** — both plugins bumped from stale 2.0.0 to current version

## 2.2.0 — 2026-02-09

### Added
- **`design-tokens.md`** — new `AnimEasing` token catalog (standard, decelerate, accelerate, spring) with usage guidance
- **`ui-recipes.md`** — 4 new sections: Reduce Motion Support (expect/actual `rememberReduceMotion()` + conditional animation), Connectivity Indicator (`ConnectivityBanner` composable), Bottom Sheet & Dialog Patterns (ModalBottomSheet + ConfirmationDialog recipes), Gesture Convention Reference (platform gesture table)
- **`theming-patterns.md`** — Dark Mode Best Practices (tonal surfaces, no pure black, edge-to-edge) and Color Accessibility (WCAG AA minimums, semantic color pairs, Material Theme Builder reference)
- **`compose-performance.md`** — Performance Budgets section (cold launch, transitions, scroll, touch response targets) with lazy list `key` pattern and ViewModel init guidance
- **`adaptive-layouts-patterns.md`** — Orientation & State Preservation (rememberSaveable, scroll position, content width constraint)
- **`data-patterns.md`** — Offline UX Guidance in cache-first pattern (isRefreshing, lastRefreshed, connectivity-aware state)
- **`accessibility-audit.md`** — Check 13 (reduce motion compliance) and Check 14 (color contrast indicators)
- **`architecture-rules.md`** — rules A10 (reduce motion fallback), A11 (color contrast compliance), PERF-2 (lazy list key parameter)
- **`review-changes.md`** — 3 new Composable/UI checklist items (reduce motion, hardcoded colors, lazy list keys)

### Changed
- **`polish-ui`** — added Step 15 (reduce motion) and Step 16 (connectivity awareness); verify checklist now checks reduce motion fallback and named easing
- **`scaffold-feature`** — Step 5 now includes `key` parameter guidance for lazy lists; report verify list updated
- **`add-screen`** — Step 5 now includes `key` parameter note for lazy lists
- **`design-tokens.md`** — rule paragraph updated to flag raw easing as violation; import block includes `AnimEasing`
- **`ui-recipes.md`** — anti-patterns table expanded with `LinearEasing` and missing reduce motion entries; Key Rules updated with `AnimEasing.*`

## 2.1.1 — 2026-02-08

### Fixed
- **`polish-ui`** — skill now produces visible UI changes instead of only mechanical plumbing. Rewrites item composables with card recipe patterns, adds staggered list entry animations, adds scale-on-press micro-interactions, enforces 3+ typography styles per item, and applies ContentAlpha to secondary/tertiary text. Step 2 now globs all `.kt` files under the feature module to find item composables in `views/` subdirectories.
- **`ui-recipes.md`** — added explicit `import androidx.compose.material3.pulltorefresh.PullToRefreshBox` to Pull-to-Refresh recipe (was causing compile failure with wrong import path). Added Minimum Visual Upgrade Checklist section. Added `contentKey = { it::class }` to AnimatedContent recipe to prevent data-change flicker.

### Added
- **1 new reference file:** `compose-performance.md` — strong skipping mode, reference stability in ViewModel `combine` chains, stability annotation options, and `AnimatedContent` `contentKey` rule
- **`architecture-rules.md`** — new Performance section with rule PERF-1 (AnimatedContent contentKey)
- **`audit-architecture.md`** — Check 15: flags `AnimatedContent` without `contentKey` when used with sealed UiState
- **`review-changes.md`** — added `contentKey` check to Composable/UI checklist

## 2.1.0 — 2026-02-08

### Added
- **1 new skill:** `polish-ui` — enhances a feature's UI with shimmer loading, rich cards, animated state transitions, and visual hierarchy while preserving architecture (ViewModel/Repository/DI/Navigation untouched). Includes a design assessment step inspired by Anthropic's frontend-design plugin, adapted for Material 3 mobile constraints.
- **1 new reference file:** `ui-recipes.md` — opinionated, copy-paste-ready Compose Multiplatform component recipes covering shimmer loading, empty/error states, card compositions, list animations, state transitions, detail screens, search, surface patterns, and micro-interactions. Includes a Design Philosophy preamble with anti-pattern guidance.
- **`AnimDuration` design tokens** in `design-tokens.md` — `short` (150ms), `medium` (300ms), `long` (500ms) for consistent animation timing across all recipes

### Changed
- **`scaffold-feature`** — report step now suggests `polish-ui` as a follow-up for visual polish
- **`design-tokens.md`** — added `AnimDuration` token catalog, import, and rule coverage for raw animation duration literals

## 2.0.0 — 2026-02-08

### Added
- **5 new skills:** `add-ktor-networking` (Ktor 3.x HTTP client), `add-room-database` (Room KMP persistence), `add-image-loading` (Coil 3 image loading), `add-convention-plugins` (build-logic included build), `upgrade-dependencies` (version catalog updates)
- **1 new agent:** `accessibility-audit` — audits composables for contentDescription, touch targets, heading semantics, testTag coverage, role semantics, and live regions
- **5 new reference files:** `networking-patterns.md` (Ktor 3.x), `persistence-patterns.md` (Room KMP), `theming-patterns.md` (Material 3), `image-loading-patterns.md` (Coil 3), `convention-plugins-patterns.md` (build-logic)

### Changed
- **`data-patterns.md`** — added Pattern 3 (Room persistence) and Pattern 4 (Ktor networking) with upgrade path diagrams
- **`code-templates.md`** — added `@Preview` section with commonMain unified annotation; added CMP 1.10+ deprecation note for `compose.xxx` shorthand accessors
- **`design-tokens.md`** — fixed `<package>` → `{package_base}` placeholder consistency; added TouchTarget section documenting 48dp minimum
- **`architecture-rules.md`** — added accessibility rules (A1–A5), image loading rules (I1–I3), networking rules (K1–K3), persistence rules (P1–P3)
- **`audit-architecture.md`** — added Check 13 (accessibility basics) and Check 14 (image loading)
- **`review-changes.md`** — added accessibility checks to composable/UI checklist; added networking/persistence checks to data layer checklist; updated implementation naming to include `Room*`

## 1.1.0 — 2026-02-08

### Changed
- **Agents now self-contained** — inlined all PASS/FAIL/WARN check criteria directly in agent system prompts instead of referencing external files. Agents no longer waste turns loading context.
- **Merged quality references** — combined `architecture-rules.md` and `audit-checklist.md` into a single `architecture-rules.md` (single source of truth)
- **Deleted scaffold `architecture-rules.md`** — skills use `code-templates.md` for patterns; quality owns rule enforcement
- **Simplified `marketplace.json`** — stripped to spec-compliant fields only (removed `$schema`, `keywords`, `category`, `license`, `repository`, `homepage`)
- **Removed redundant `skills`/`agents` paths from `plugin.json`** — Claude Code auto-discovers from default directories
- **Added `context: fork`** to `fix-imports` and `extract-strings` skills (scan-heavy, benefits from context isolation)
- **Wired `design-tokens.md`** into `scaffold-feature` and `add-screen` skills (was orphaned)
- **Inlined brief context detection** in skills instead of depending on `project-context.md` as execution reference
- **Trimmed report templates** across all skills — concise instructions instead of rigid markdown templates
- **Fixed `data-patterns.md`** — replaced skill invocation link with code-templates.md reference

## 1.0.0 — 2026-02-08

### Added
- Reference files for progressive disclosure: `project-context.md`, `code-templates.md`, `design-tokens.md`, `data-patterns.md`, `string-resources.md` (cmp-scaffold), `architecture-rules.md` (cmp-quality), `test-patterns.md` (scaffold-tests)
- `argument-hint` frontmatter to all skills that accept arguments
- README.md, CLAUDE.md, CHANGELOG.md

### Changed
- Slimmed all 7 SKILL.md files by extracting duplicated content into shared reference files
- Slimmed all 4 agent .md files by extracting check definitions into shared reference files
- Architecture rules consolidated from inline content across 11 skill/agent files
