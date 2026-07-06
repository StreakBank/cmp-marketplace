# Provenance — cmp-quality

## references/

All content under `references/` (`compose-recomposition-migration-recipes.md`) is
**authored-original** for this plugin.

- No upstream repo is vendored.
- No third-party LICENSE applies.
- Nothing here is copied from another project's skill, documentation, or codebase —
  the recipes describe generic Compose Multiplatform recomposition-remediation
  mechanics (deferred-phase state reads; stability-config reconciliation), written
  directly to accompany the `performance-audit` agent's strong-skipping calculus.
- There is nothing to attribute.

This file exists to satisfy `scripts/check-coupling.sh` check #5 (a plugin with a
`references/` dir must carry a `PROVENANCE.md`, since a `references/` dir is otherwise
a heuristic signal for vendored content) — here, the honest answer is that the
heuristic doesn't apply.

**If a future `references/` addition vendors upstream material**, record it here
instead of leaving this file stale:

- Upstream repo (URL)
- Commit hash pinned
- Exact file(s) taken
- Every pruning/edit applied (so refresh diffs stay possible)
- Confirmation the upstream LICENSE was added adjacent to the vendored content
