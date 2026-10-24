# cmp-design-bridge (skills)

Claude Code skills for the [`cmp-design-bridge`](https://github.com/StreakBank/cmp-design-bridge)
Claude Design → Compose bridge. They sequence the deterministic CLI and invoke
the two model-driven legs:

- **design-pull** — pull a design's per-state frames from Claude Design and stage them.
- **design-transform** — render a frame and re-author the idiomatic Compose screen to match.
- **design-fidelity** — grade the rendered design against the app's screenshot (advisory verdict).
- **design-push** — publish authored/updated frames back to Claude Design.

## Prerequisite

These skills require the separately installed standalone CLI 0.3.1:

```bash
npm i -g cmp-design-bridge@0.3.1
```

Installing this wrapper does not install or upgrade the CLI. Update the npm package
deliberately outside a skill invocation before using its 0.3.0 configuration and
evidence contract. Per-project config lives in the consuming repo's `.design-bridge/`.
The native Design MCP fetches files into that config's staged frame cache; CLI
`pull` validates already staged bytes and does not download canvas files. The skills
sequence those operations and the two model-driven legs.

## Safe CLI help and argument handling

This wrapper targets CLI **0.3.1**; update the separate npm package deliberately,
never through a skill invocation. See [PROVENANCE.md](PROVENANCE.md) for the release
binding and verification status. The CLI rejects unknown/cross-command flags,
missing or malformed values, duplicate/conflicting flags, and extra positionals
before loading CONFIG or doing command work. Use its declared flags with separate
values (`--config <dir>`, not `--config=<dir>`).

Root `cmp-design-bridge --help` and `<command> --help` are side-effect-free since
0.2.1. **CLI 0.2.0 runs stamping lint for `lint --help`; use root help on that
version.** Neither this fix nor the wrapper changes lint's stamping default or
makes a successful lint prove that renders are fresh. The caveat below still applies.

## Exact-state recipes and render-evidence refresh

CLI 0.3.0 supports `render-recipe.json`'s `stateOverrides` map, keyed by exact IDs
already enrolled in the join manifest or state inventory. Each entry may override
`viewport.logicalWidthCss`, `viewport.dpr`, `clipHeight` and `settleMs`. Omitted
values inherit the base recipe; `clipHeight: null` selects intrinsic height. There
is no prefix or wildcard matching. Unknown IDs/keys and invalid dimensions fail
during config loading. Ordinary `render`/`verify` use the same canonical CONFIG;
do not shrink a component frame or copy manifests from another config to pass the
width gate.

Render evidence records `effectiveRecipe` and `recipeHash` for normalized width,
DPR, crop and settle values, without machine paths. Missing or stale evidence,
including predecessor manifests, makes rendered `verify` refresh the state and
makes `lint` refuse its pin (`STALE_RENDER_RECIPE`). Refresh every state whose
local render evidence will be consumed when adopting 0.3.0. A recipe-only change
requires new render evidence even when the HTML/JSX content pin stays unchanged.

Imported intake uses the same effective geometry. Imported `verify` fails
`recipeCurrent`/`notStale` when recipe evidence is missing or stale; repeat intake
with the intended source and crop before grading. It cannot re-render a raster
automatically. Fractional CSS widths and DPR are accepted: imported physical
width/crop dimensions round to the nearest whole pixel, positive half ties up,
with a one-pixel minimum; intake parameters and imported verification use the same
rule. The recipe fingerprint retains the original effective values.

## Caveat — a stamping `lint` re-pins from your LOCAL render cache

**What `lint` re-pins.** Each join-manifest row carries a `designContentHash`. A row
whose hash is `auto` (or absent) is **STAMPED** with the `sourceContentHash` recorded
in that state's `<cache>/out/<stateId>.pull-manifest.json` — the manifest written by
the **last local `render`** of that state — and the join manifest is rewritten in
place. A row that already carries a real pin is instead **VERIFIED** against that same
local artifact (a mismatch is reported as `CONTENT_HASH_DRIFT`). Stamping is the
default; `--no-stamp` makes the command read-only **with respect to the join
manifest** — no row rewritten, the manifest not written back. It is not
side-effect-free: `lint` always (re)creates its out dir and writes its own
`join-lint-report.json` there.

`<cache>/out` is the *default* out dir and `--out <dir>` overrides it — for `render`
and for `lint` alike, so the two must agree. A `render --out X` followed by a bare
`lint` reads a different directory and reports every state as "no render yet".

**Why a stamping PASS is unreliable.** Both the stamp and the pin check read local
render output, never the remote design source. Even with current recipe evidence,
if `.cache/` is stale — frames pulled a while
ago, a state not re-rendered since the last pull, renders left over from another
branch — then:

- an `auto` row is pinned to the **stale** hash, and from then on reports
  "designContentHash verified (no drift)" forever, even though the authoritative frame
  has moved since;
- a state with **no** local render at all lands on the **PASS side** — a "no render
  yet" `ok` line, not a finding — so an unrendered state can never fail the lint, and
  a run over an empty cache passes the pin check having verified nothing.

So a stamping lint's PASS is only as fresh as `.cache/out`, and it silently converts
cache staleness into a durable, wrong pin. It refuses missing/stale recipe evidence
(`STALE_RENDER_RECIPE`) and failed renders (`STAMP_FROM_FAILED_RENDER`); those
checks do not fetch current canvas content or prove complete render coverage.

**Recommended sequence** (the only order that produces true pins):

1. Refresh/stage the affected frames through the native Design MCP, including
   **every actual same-folder `_*.jsx` helper**, then run `pull` to validate the
   staged bytes. CLI `pull` performs no download.
2. Preserve curated manifest rows. Add only reviewed new rows; `--write-manifest`
   regenerates the whole manifest, so never use it as a routine incremental update.
   If derivation is needed, run it against an isolated scratch CONFIG/manifest and
   take only the reviewed additions. `--out` does not redirect that manifest write.
3. `render` **every affected state** — a shared helper changes the content hash for
   every frame in its folder, even when an HTML file does not explicitly import it.
   Repopulate the agreed output directory from the exact refreshed sources.
4. `lint --no-stamp` — the integrity check without manifest writes. **This is the commit gate.**
5. `lint` **once**, stamping, so reviewed `auto` rows pin to the fresh hashes.
6. Commit the manifest with those pins and preserve prior curated row identities.

**`--no-stamp` is the safe default in CI and any automation.** A stamping lint *writes*
the join manifest, so an unattended run can produce (or dirty) pins nobody reviewed —
and on a partial or stale render cache those pins are wrong rather than merely missing.
Gate merges on `lint --no-stamp` (add `--fail-on-backfill` where the backfill gate
applies) and keep stamping a deliberate, local step performed after a full pull +
render.

Recipe evidence does not broaden content hashing: it still covers the frame plus
all same-folder `_*.jsx` siblings, not cross-folder dependencies. The runtime pin
`dsRuntimeHash` covers `_ds_bundle.js` alone; it does not prove arbitrary runtime
parts, CSS, fonts or uploaded bytes unchanged. Preserve the consuming project's
runtime-file validation and served-byte checks when refreshing those inputs.

## Wanted in the CLI — NOT IMPLEMENTED (recorded 2026-08-26)

Gaps observed in real use, recorded here so they are visible upstream. **None of these
exist today** — do not write config or CI against them.

- **`extraContentGlobs` (config) — NOT IMPLEMENTED.** A per-module glob list folded
  into the render/pull content hash, so files a frame depends on beyond
  `<stateId>.html` + that module folder's `_*.jsx` siblings participate in pinning.
- **`componentCaptures` (config) — NOT IMPLEMENTED.** A declared class of
  component-isolation captures. Today every `<statePrefix>*.png` in a module's capture
  dir is read as a screen state, so component-level captures land in the backfill queue
  or the stale-orphan list and skew the coverage burndown.
- **Shell-scoped state mapping (a non-module `shell-*` prefix) — NOT IMPLEMENTED.**
  Every state binds to a **module entry** (its frames folder, its `captureDir`, its
  `statePrefix`), so cross-cutting app-shell/chrome states belonging to no feature
  module cannot be mapped to a capture dir without inventing a placeholder module
  entry.
- **Cross-folder hash blindness — NOT IMPLEMENTED (open defect).** The content hash
  covers the frame plus the **same-folder** `_*.jsx` siblings only; a frame that depends
  on a file in another folder pins and re-verifies clean while its real source has
  changed. `extraContentGlobs` is the candidate fix.

## Returning tool learnings

Generic CLI argument or command defects belong in the
[CLI repository](https://github.com/StreakBank/cmp-design-bridge); sequencing and
skill guidance belong in the
[wrapper repository](https://github.com/StreakBank/cmp-marketplace). Return the
fix with regression evidence, a version/changelog update, and this plugin's
provenance. Keep project-specific policy in the consuming CONFIG or shim.
