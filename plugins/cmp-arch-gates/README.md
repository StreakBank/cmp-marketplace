# cmp-arch-gates (plugin)

Thin Claude Code skill wrapping the [`cmp-arch-gates`](https://github.com/StreakBank/cmp-arch-gates)
CLI — deterministic architecture gates for KMP / Compose Multiplatform codebases (a
layered-module-DAG + boundary/visibility linter for CI or local runs).

## What it does

Fails the build when the Gradle module graph or its layer boundaries drift — a
`core → feature` inversion, a transport type leaking into a `data:api` contract, a
data-source going public, an undocumented cross-owner `data:api` re-export. Five
gates; all logic lives in the CLI, all project facts in the consuming repo's
`.arch-gates/config.json`.

## Install

```sh
npm i -g cmp-arch-gates        # the deterministic CLI (Node ≥ 18, zero deps)
```

Then invoke the `arch-gates` skill, or run the CLI directly.

## Onboard a repo

```sh
cmp-arch-gates init --detect   # scaffold .arch-gates/config.json + report candidate facts
```

Copy `SHIM-TEMPLATE.md` into the consuming repo's `.claude/rules/` for the
project-side pointer + facts.

## Files

- `skills/arch-gates/SKILL.md` — the skill (mechanics + config + CI wiring).
- `SHIM-TEMPLATE.md` — copy-paste path-keyed `.claude/rules/` pointer for a consumer.

The gate implementations, tests, and config schema live in the CLI repo, not here —
this plugin stays thin.
