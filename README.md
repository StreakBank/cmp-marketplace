# CMP Marketplace

A Claude Code plugin marketplace for **Kotlin Multiplatform (KMP) Compose Multiplatform** projects. Provides scaffolding skills for generating feature modules and quality agents for architecture auditing.

## Philosophy

This marketplace enforces a consistent, opinionated architecture for KMP/CMP apps:

- **Multi-module feature isolation** — each feature has `data/api`, `data/impl`, and `feature` sub-modules
- **Interface-based data sources** — swap InMemory for Room/Ktor without changing consumers
- **Reactive state** — `stateIn()` pattern in ViewModels, `Flow` reads, `Result` writes
- **Design system tokens** — no raw `dp` values, all sizing through `Spacing.*`, `IconSize.*`, etc.
- **String resources everywhere** — no hardcoded user-facing text
- **Type-safe navigation** — `@Serializable` routes (Navigation 2.x)
- **Real networking** — Ktor 3.x HTTP client with DTOs and safe API calls
- **Persistent storage** — Room KMP with entity/domain model separation
- **Image loading** — Coil 3 with cross-platform caching
- **Accessibility** — contentDescription, touch targets, heading semantics, testTag coverage
- **Theming** — Material 3 color schemes, typography, dark/light mode
- **Build consistency** — Convention plugins via build-logic included build

## Installation

```bash
# Add the marketplace
claude plugin marketplace add /path/to/cmp-marketplace

# Install individual plugins
claude plugin install cmp-scaffold@cmp-marketplace --scope project
claude plugin install cmp-quality@cmp-marketplace --scope project
```

## Plugins

### cmp-scaffold

Scaffolding skills for generating architecture-compliant code (21 skills).

| Skill | Command | Description |
|-------|---------|-------------|
| scaffold-feature | `/cmp-scaffold:scaffold-feature <name>` | Generate a complete feature module |
| add-screen | `/cmp-scaffold:add-screen <module> <screen>` | Add a screen to an existing module |
| add-navigation-tab | `/cmp-scaffold:add-navigation-tab <name>` | Wire a module into bottom navigation |
| add-remote-datasource | `/cmp-scaffold:add-remote-datasource <name>` | Upgrade local-only to local+remote |
| add-ktor-networking | `/cmp-scaffold:add-ktor-networking <name>` | Replace mock remote DS with Ktor HTTP client |
| add-room-database | `/cmp-scaffold:add-room-database <name>` | Replace in-memory local DS with Room KMP |
| add-image-loading | `/cmp-scaffold:add-image-loading <name>` | Add Coil 3 image loading to a feature |
| polish-ui | `/cmp-scaffold:polish-ui <name>` | Enhance UI with rich cards, shimmer loading, animations |
| add-domain-layer | `/cmp-scaffold:add-domain-layer <name>` | Add domain/use-case layer to a feature |
| add-convention-plugins | `/cmp-scaffold:add-convention-plugins` | Set up build-logic convention plugins |
| scaffold-tests | `/cmp-scaffold:scaffold-tests <name>` | Generate unit test scaffolding |
| extract-strings | `/cmp-scaffold:extract-strings <name-or-path>` | Extract hardcoded strings to resources |
| fix-imports | `/cmp-scaffold:fix-imports` | Fix CMP resource import prefixes |
| add-theming | `/cmp-scaffold:add-theming` | Set up Material 3 theming with dark mode and persistence |
| add-adaptive-layout | `/cmp-scaffold:add-adaptive-layout` | Upgrade to adaptive NavigationSuiteScaffold |
| add-permissions | `/cmp-scaffold:add-permissions <type>` | Add runtime permission handling (camera, location, etc.) |
| add-deep-linking | `/cmp-scaffold:add-deep-linking <name>` | Add deep link support to feature navigation routes |
| add-analytics | `/cmp-scaffold:add-analytics` | Add Firebase analytics and crash reporting |
| add-background-sync | `/cmp-scaffold:add-background-sync <name>` | Add background sync with WorkManager/BGTaskScheduler |
| add-push-notifications | `/cmp-scaffold:add-push-notifications` | Add push notifications with FCM and APNs |
| upgrade-dependencies | `/cmp-scaffold:upgrade-dependencies` | Check and update dependency versions |

### cmp-quality

Architecture auditing and validation agents (7 agents).

| Agent | Description |
|-------|-------------|
| audit-architecture | Full architecture compliance audit across all modules |
| dependency-audit | Cross-module dependency validation |
| validate-module | Deep validation of a single feature module |
| review-changes | Review uncommitted changes against architecture rules |
| accessibility-audit | Audit composable screens for accessibility compliance |
| validate-tests | Audit test quality, structure, and patterns |
| performance-audit | Audit composable performance patterns and recomposition efficiency |

## Quick Start

```bash
# 1. Scaffold a new feature module
/cmp-scaffold:scaffold-feature orders

# 2. Wire it into bottom navigation
/cmp-scaffold:add-navigation-tab orders

# 3. Add a detail screen
/cmp-scaffold:add-screen orders detail

# 4. Add remote data source (mock)
/cmp-scaffold:add-remote-datasource orders

# 5. Replace mock with real Ktor networking
/cmp-scaffold:add-ktor-networking orders

# 6. Add persistent storage with Room
/cmp-scaffold:add-room-database orders

# 7. Add image loading
/cmp-scaffold:add-image-loading orders

# 8. Set up theming
/cmp-scaffold:add-theming

# 9. Add adaptive navigation
/cmp-scaffold:add-adaptive-layout

# 10. Generate tests
/cmp-scaffold:scaffold-tests orders

# 11. Set up convention plugins (once per project)
/cmp-scaffold:add-convention-plugins

# 12. Add runtime permissions (if needed)
/cmp-scaffold:add-permissions camera

# 13. Add deep linking to the feature
/cmp-scaffold:add-deep-linking orders

# 14. Add analytics and crash reporting (once per project)
/cmp-scaffold:add-analytics

# 15. Add background sync
/cmp-scaffold:add-background-sync orders

# 16. Add push notifications (once per project)
/cmp-scaffold:add-push-notifications

# 17. Check for dependency updates
/cmp-scaffold:upgrade-dependencies

# 18. Validate the module
# (triggers validate-module agent for single-module deep check)
Validate the orders module

# 19. Check accessibility
# (triggers accessibility-audit agent)
Audit accessibility of the orders module

# 20. Check performance
# (triggers performance-audit agent)
Audit performance of the orders module

# 21. Review before committing
# (triggers review-changes agent)
Review my changes
```

## Navigation

These plugins target **Navigation 2.x** with `@Serializable` type-safe routes. Navigation 3.x support is planned for a future release.

## Companion Project

The architecture patterns and reference files in this marketplace are adapted from the [example-cmp](https://github.com/LadderPicks/example-cmp) companion project, which provides a working reference implementation.

## License

MIT
