# CMP Marketplace — Claude Code Context

Claude Code plugin marketplace for KMP/CMP (Kotlin Multiplatform / Compose Multiplatform) projects.

## Directory Structure

```
cmp-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace manifest (lists plugins)
├── plugins/
│   ├── cmp-scaffold/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json       # Plugin manifest (auto-discovers skills/)
│   │   ├── references/           # Shared reference files for skills
│   │   │   ├── code-templates.md
│   │   │   ├── design-tokens.md
│   │   │   ├── data-patterns.md
│   │   │   ├── string-resources.md
│   │   │   ├── networking-patterns.md
│   │   │   ├── persistence-patterns.md
│   │   │   ├── theming-patterns.md
│   │   │   ├── image-loading-patterns.md
│   │   │   ├── adaptive-layouts-patterns.md
│   │   │   ├── convention-plugins-patterns.md
│   │   │   ├── compose-performance.md
│   │   │   ├── domain-layer-patterns.md
│   │   │   ├── ui-recipes.md             # Index — links to recipe sub-files below
│   │   │   ├── ui-recipes-loading.md     # Shimmer, empty/error states, transitions
│   │   │   ├── ui-recipes-cards.md       # Card compositions, detail screens
│   │   │   ├── ui-recipes-lists.md       # Animated lists, search, pull-to-refresh
│   │   │   ├── ui-recipes-forms.md       # Validated fields, form state patterns
│   │   │   ├── ui-recipes-surfaces.md    # Tonal surfaces, micro-interactions, a11y
│   │   │   ├── test-patterns.md          # Test doubles, Turbine, dispatcher setup
│   │   │   ├── permissions-patterns.md  # Runtime permissions, expect/actual handlers
│   │   │   ├── deep-linking-patterns.md # Deep links, App Links, URI schemes
│   │   │   ├── analytics-patterns.md    # Firebase Analytics, Crashlytics, event naming
│   │   │   ├── background-work-patterns.md  # WorkManager, BGTaskScheduler, sync
│   │   │   └── push-notifications-patterns.md # FCM, APNs, notification channels
│   │   └── skills/               # Auto-discovered by Claude Code
│   │       ├── scaffold-feature/SKILL.md
│   │       ├── add-screen/SKILL.md
│   │       ├── add-navigation-tab/SKILL.md
│   │       ├── add-remote-datasource/SKILL.md
│   │       ├── add-ktor-networking/SKILL.md
│   │       ├── add-room-database/SKILL.md
│   │       ├── add-image-loading/SKILL.md
│   │       ├── add-convention-plugins/SKILL.md   (context: fork)
│   │       ├── upgrade-dependencies/SKILL.md     (context: fork)
│   │       ├── scaffold-tests/SKILL.md
│   │       ├── extract-strings/SKILL.md   (context: fork)
│   │       ├── fix-imports/SKILL.md       (context: fork)
│   │       ├── polish-ui/SKILL.md         (context: fork)
│   │       ├── add-domain-layer/SKILL.md
│   │       ├── add-theming/SKILL.md
│   │       ├── add-adaptive-layout/SKILL.md
│   │       ├── add-permissions/SKILL.md
│   │       ├── add-deep-linking/SKILL.md
│   │       ├── add-analytics/SKILL.md
│   │       ├── add-background-sync/SKILL.md
│   │       └── add-push-notifications/SKILL.md
│   └── cmp-quality/
│       ├── .claude-plugin/
│       │   └── plugin.json       # Plugin manifest (auto-discovers agents/)
│       └── agents/               # Auto-discovered by Claude Code
│           ├── audit-architecture.md
│           ├── dependency-audit.md
│           ├── validate-module.md
│           ├── review-changes.md
│           ├── accessibility-audit.md
│           ├── validate-tests.md
│           └── performance-audit.md
├── README.md
├── CLAUDE.md
└── CHANGELOG.md
```

## Skills vs Agents

**Skills** run inline in the main conversation. They use progressive disclosure — the SKILL.md body loads when triggered, and reference files are read on demand via tool calls. Good for: code generation workflows.

**Agents** run in isolated subagent contexts with their own system prompt. Their `.md` body is their entire system prompt — they have no conversation history. All execution criteria must be inline. Good for: analysis tasks that produce verbose output.

## Adding/Modifying Skills

1. Create `skills/<skill-name>/SKILL.md` with frontmatter
2. Reference shared content from `../../references/` for templates and conventions
3. Skills in `skills/` are auto-discovered — no need to list them in `plugin.json`
4. Use `context: fork` for scan-heavy skills that produce verbose intermediate output

### Required SKILL.md Frontmatter

```yaml
---
name: skill-name
description: When to use this skill (Claude uses this for auto-invocation)
argument-hint: <placeholder>         # if skill accepts arguments
allowed-tools:                       # tools pre-approved without user prompt
  - Read
  - Write
---
```

## Adding/Modifying Agents

1. Create `agents/<agent-name>.md` with frontmatter
2. **Inline all execution criteria** — the agent body is the entire system prompt
3. Reference files are supplementary docs, not execution dependencies
4. Agents in `agents/` are auto-discovered — no need to list them in `plugin.json`

### Required Agent Frontmatter

```yaml
---
name: agent-name
description: When Claude should delegate to this agent
tools:                               # capability allowlist (restrictive, not additive)
  - Read
  - Glob
model: sonnet                        # sonnet | opus | haiku | inherit
---
```

## Skill Dependency Diagram

Skills are designed to be run in sequence. App-level skills (no arguments) are run once per project. Feature-level skills chain together:

```
scaffold-feature <name>
├── add-screen <name> <screen>         (additional screens)
├── add-navigation-tab <name>          (wire into bottom nav)
├── add-remote-datasource <name>       (upgrade to local+remote)
│   ├── add-ktor-networking <name>     (replace mock with real HTTP)
│   └── add-room-database <name>       (replace in-memory with Room)
├── add-domain-layer <name>            (extract use cases from ViewModel)
│   └── add-background-sync <name>     (WorkManager / BGTaskScheduler)
├── add-image-loading <name>           (Coil 3 setup + feature wiring)
├── add-deep-linking <name>            (URI scheme + App Links)
├── scaffold-tests <name>              (unit test scaffolding)
└── polish-ui <name>                   (visual design upgrade)

App-level (once per project):
├── add-theming                        (Material 3 + dark mode + DataStore)
├── add-adaptive-layout                (NavigationSuiteScaffold + WindowSizeClass)
├── add-convention-plugins             (build-logic included build)
├── add-analytics                      (Firebase Analytics + Crashlytics)
├── add-push-notifications             (FCM + APNs)
├── add-permissions <type>             (runtime permission handling)
└── upgrade-dependencies               (version catalog updates)

Utilities (run anytime):
├── extract-strings <name-or-path>     (hardcoded strings → resources)
└── fix-imports                        (CMP resource import prefixes)
```

## Key Design Decisions

- **Agents are self-contained** — criteria inline, not behind reference file indirection
- **Skills use progressive disclosure** — templates and conventions in reference files, loaded on demand
- **Nav 2.x only** — all patterns target Navigation 2.x; Nav 3.x noted for future
- **Auto-discovery** — `plugin.json` has no explicit skills/agents paths; Claude Code finds them in default directories
- **Agents are the source of truth** — quality agents inline their own check criteria; no separate rules file
- **Room KMP for persistence** — not SQLDelight
- **build-logic included build** — convention plugins for consistent multi-module config
- **App-level skills have no `argument-hint`** — skills that take no arguments (theming, adaptive layout, analytics, push notifications, convention plugins, fix-imports, upgrade-dependencies) omit `argument-hint` from frontmatter; feature-level skills include `argument-hint: <feature-name>`
