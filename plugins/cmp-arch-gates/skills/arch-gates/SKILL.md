---
name: arch-gates
description: Run or configure the KMP / Compose Multiplatform architecture gates (a layered-module-DAG + boundary/visibility linter) via the cmp-arch-gates CLI. Use when wiring architecture enforcement into CI, checking module-graph layering locally, adding a new gate, or writing a project's .arch-gates/config.json. Triggers — "architecture gate", "module graph lint", "layered DAG check", "data:api purity", "cross-owner data:api re-export", "datasource visibility".
argument-hint: "[check|list] [--root <dir>] [--only <gate>]"
allowed-tools:
  - Bash
  - Read
  - Glob
---

# arch-gates

Deterministic architecture gates for KMP / Compose Multiplatform codebases. They
fail the build when the Gradle module graph or its layer boundaries drift, so an
accumulated-drift class of regression (a `core → feature` inversion, a transport
type leaking into a `data:api` contract, a data-source going public, an
undocumented cross-owner `data:api` re-export) can't silently return.

This skill is a thin wrapper: all logic lives in the **`cmp-arch-gates` CLI**
(deterministic, unit-tested, zero-dependency). Project facts live in the consuming
repo's `.arch-gates/config.json`; the CLI and this skill hardcode none.

## Prerequisite

```sh
npm i -g cmp-arch-gates        # or run ad-hoc with: npx cmp-arch-gates@<version>
```

Requires Node ≥ 18.

## Run

```sh
cmp-arch-gates check                       # scan cwd; config from ./.arch-gates/config.json
cmp-arch-gates check --root <repo-root>     # scan a specific root
cmp-arch-gates check --only module-direction
cmp-arch-gates list                         # print the gates
```

Exit `0` = all pass, `1` = a gate failed, `2` = usage/config error. Fast enough
(sub-second on a large repo) to run on every push and as a local pre-commit check.

## Onboard a new repo

```sh
cmp-arch-gates init --detect
```

Scaffolds a starter `.arch-gates/config.json` and scans the repo for **candidate**
facts — `:core:*` modules that depend on Ktor (transport candidates), existing
cross-owner `data:api` `api()` edges, and whether the config-independent gates
already pass — printing them for the human to confirm. It never auto-writes a
module name or allowlists an existing edge (silently allowlisting current
violations would make the gate green by hiding drift). Fill in the confirmed
facts, add the CI line, copy the shim template (below). `init` won't overwrite an
existing config without `--force`.

## The gates

| Gate | Fails when | Config key |
|------|-----------|-----------|
| `module-direction` | a `core → feature/data/domain`, `data:api → data:impl`, or `feature → feature` edge exists | — |
| `implementation-only-modules` | a configured module is `api()`-exported instead of `implementation()` | `transportModules` |
| `data-api-purity` | a `*/data/api` source imports a banned transport/serialization prefix | `dataApiBannedImports` |
| `datasource-visibility` | a `*(Remote\|Local)DataSource` type in `*/data/impl` isn't `internal`/`private` | — |
| `cross-owner-dataapi` | a `*/data/api` `api()`-re-exports another owner's `data:api` and it isn't allowlisted | `crossOwnerDataApiAllowlist` |

Layer is inferred from the Gradle-notation suffix; `build/`, `build-logic/`,
VCS/IDE caches, and `test-fixtures` are excluded; `datasource-visibility` also
skips every `*Test*` source set.

## Configure — `.arch-gates/config.json` (in the consuming repo)

```json
{
  "$schema": "https://raw.githubusercontent.com/StreakBank/cmp-arch-gates/main/schema/config.schema.json",
  "transportModules": [":core:network"],
  "dataApiBannedImports": ["io.ktor", "kotlinx.serialization"],
  "crossOwnerDataApiAllowlist": [
    { "owner": "catalog/data/api", "dep": ":pricing:data:api", "reason": "documented carve-out" }
  ],
  "docsRef": "docs/ARCHITECTURE.md"
}
```

All keys optional. Omit `transportModules` → that gate is a no-op; omit
`dataApiBannedImports` → the `io.ktor` / `kotlinx.serialization` default; omit
`crossOwnerDataApiAllowlist` → zero tolerance; `docsRef` → no pointer in failure
output. The values above are examples — supply your own module names, allowlist,
and doc pointer.

**Non-default layouts.** Repos that don't follow the suffix-style convention
configure `layerRules` (map a Gradle notation to a layer by prefix/suffix/regex —
this is how a prefix-style `:feature:home` repo opts in), `dataSourceTypeSuffixes`
(e.g. `NetworkDataSource`), and `dataApiDir` / `dataImplDir`. Nothing is hardcoded
in the core. If the configured `layerRules` classify no feature/data module,
`module-direction` fails loudly instead of greenwashing; a gate that scans zero
candidate files prints a `warn`, not a silent `ok`. See the CLI README for the full
schema.

## Wire into CI

Pin an exact version; the CLI runs on any CI runner (no plugin install needed):

```yaml
- run: npx cmp-arch-gates@<version> check
```

## Add a gate

Gates live in the CLI repo as pure `run(root, config) → { ok, violations }`
modules under `lib/gates/`, each with a fixture test. Add the module, register it
in `lib/gates/registry.mjs`, add a test, bump the CLI's semver. Keep every project
fact in config, never in the gate.

## Project policy stays in the consuming repo

This skill and the CLI are project-agnostic. Which modules are transport, which
cross-owner edges are allowed, and which architecture doc to cite are per-project
facts — they belong in that repo's `.arch-gates/config.json` and its own `.claude/`
estate (a thin path-keyed rule pointing here), never in this skill. Copy
`SHIM-TEMPLATE.md` (shipped in this plugin) into the consuming repo's
`.claude/rules/` and fill the placeholders to create that rule.
