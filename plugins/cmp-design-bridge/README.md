# cmp-design-bridge (skills)

Claude Code skills for the [`cmp-design-bridge`](https://github.com/StreakBank/cmp-design-bridge)
Claude Design → Compose bridge. They sequence the deterministic CLI and invoke
the two model-driven legs:

- **design-pull** — pull a design's per-state frames from Claude Design and stage them.
- **design-transform** — render a frame and re-author the idiomatic Compose screen to match.
- **design-fidelity** — grade the rendered design against the app's screenshot (advisory verdict).
- **design-push** — publish authored/updated frames back to Claude Design.

## Prerequisite

These skills call the standalone CLI (a separate npm package):

```bash
npm i -g cmp-design-bridge
```

The CLI does the deterministic work (pull / render / lint / verify); the skills
sequence it and make the two irreducibly model-driven calls. Per-project config
lives in the consuming repo's `.design-bridge/`.

## Caveat — a stamping `lint` re-pins from your LOCAL render cache

**What `lint` re-pins.** Each join-manifest row carries a `designContentHash`. A row
whose hash is `auto` (or absent) is **STAMPED** with the `sourceContentHash` recorded
in that state's `<cache>/out/<stateId>.pull-manifest.json` — the manifest written by
the **last local `render`** of that state — and the join manifest is rewritten in
place. A row that already carries a real pin is instead **VERIFIED** against that same
local artifact (a mismatch is reported as `CONTENT_HASH_DRIFT`). Stamping is the
default; `--no-stamp` makes the whole command read-only (no row rewritten, no manifest
written).

**Why a stamping PASS is unreliable.** Both the stamp and the verify read the local
render output, never the design source. If `.cache/` is stale — frames pulled a while
ago, a state not re-rendered since the last pull, renders left over from another
branch — then:

- an `auto` row is pinned to the **stale** hash, and from then on reports
  "designContentHash verified (no drift)" forever, even though the authoritative frame
  has moved since;
- a state with **no** local render at all is skipped with an informational "no render
  yet" line and contributes nothing to the PASS.

So a stamping lint's PASS is only as fresh as `.cache/out`, and it silently converts
cache staleness into a durable, wrong pin. The one thing it refuses is pinning from a
render that failed its own gate (`STAMP_FROM_FAILED_RENDER`) — everything else about
freshness is on you.

**Recommended sequence** (the only order that produces true pins):

1. `pull --write-manifest` — refresh every frame from the design source and regenerate
   the rows.
2. `render` **every state in scope** — repopulate `<cache>/out/` so the hashes describe
   the frames you just pulled.
3. `lint --no-stamp` — the read-only integrity check. **This is the commit gate.**
4. `lint` **once**, stamping, so the `auto` rows pin to the fresh hashes.
5. Commit the manifest with those pins.

**`--no-stamp` is the safe default in CI and any automation.** A stamping lint *writes*
the join manifest, so an unattended run can produce (or dirty) pins nobody reviewed —
and on a partial or stale render cache those pins are wrong rather than merely missing.
Gate merges on `lint --no-stamp` (add `--fail-on-backfill` where the backfill gate
applies) and keep stamping a deliberate, local step performed after a full pull +
render.

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
  Module resolution is purely `statePrefix` → module, so cross-cutting app-shell/chrome
  states whose prefix belongs to no feature module cannot be mapped to a capture dir
  without inventing a placeholder module entry.
- **Cross-folder hash blindness — NOT IMPLEMENTED (open defect).** The content hash
  covers the frame plus the **same-folder** `_*.jsx` siblings only; a frame that depends
  on a file in another folder pins and re-verifies clean while its real source has
  changed. `extraContentGlobs` is the candidate fix.
