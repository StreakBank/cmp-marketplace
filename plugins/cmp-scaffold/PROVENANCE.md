# Provenance — cmp-scaffold

## references/

All content under `references/` (24 files: `code-templates.md`, `design-tokens.md`,
`data-patterns.md`, `string-resources.md`, `networking-patterns.md`,
`persistence-patterns.md`, `theming-patterns.md`, `image-loading-patterns.md`,
`adaptive-layouts-patterns.md`, `convention-plugins-patterns.md`,
`compose-performance.md`, `domain-layer-patterns.md`, `permissions-patterns.md`,
`deep-linking-patterns.md`, `analytics-patterns.md`, `background-work-patterns.md`,
`push-notifications-patterns.md`, `test-patterns.md`, `ui-recipes.md`, and the
`ui-recipes-*.md` sub-files) is **authored-original** for this plugin.

- No upstream repo is vendored.
- No third-party LICENSE applies.
- Nothing here is copied from another project's skill, documentation, or codebase —
  the patterns describe this marketplace's own KMP/Compose Multiplatform
  conventions, written directly for these skills.
- There is nothing to attribute.

This file exists to satisfy `scripts/check-coupling.sh` check #5 (a plugin with a
`references/` dir must carry a `PROVENANCE.md`, since a `references/` dir is
otherwise a heuristic signal for vendored content) — here, the honest answer is
that the heuristic doesn't apply.

**If a future `references/` addition vendors upstream material**, record it here
instead of leaving this file stale:

- Upstream repo (URL)
- Commit hash pinned
- Exact file(s) taken
- Every pruning/edit applied (so refresh diffs stay possible)
- Confirmation the upstream LICENSE was added adjacent to the vendored content
