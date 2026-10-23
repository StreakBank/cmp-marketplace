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
npm i -g cmp-design-bridge@0.2.1
```

The CLI does the deterministic work (pull / render / lint / verify); the skills
sequence it and make the two irreducibly model-driven calls. Per-project config
lives in the consuming repo's `.design-bridge/`.

## Safe CLI help and argument handling

This wrapper targets CLI **0.2.1**; update the separate npm package deliberately,
never through a skill invocation. See [PROVENANCE.md](PROVENANCE.md) for the release
binding and verification status. The CLI rejects unknown/cross-command flags,
missing or malformed values, duplicate/conflicting flags, and extra positionals
before loading CONFIG or doing command work. Use its declared flags with separate
values (`--config <dir>`, not `--config=<dir>`).

Root `cmp-design-bridge --help` and `<command> --help` are side-effect-free in
0.2.1. **CLI 0.2.0 runs stamping lint for `lint --help`; use root help on that
version.** Neither this fix nor the wrapper changes lint's stamping default or
makes a successful lint prove that renders are fresh. The caveat below still applies.

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

**Why a stamping PASS is unreliable.** Both the stamp and the verify read the local
render output, never the design source. If `.cache/` is stale — frames pulled a while
ago, a state not re-rendered since the last pull, renders left over from another
branch — then:

- an `auto` row is pinned to the **stale** hash, and from then on reports
  "designContentHash verified (no drift)" forever, even though the authoritative frame
  has moved since;
- a state with **no** local render at all lands on the **PASS side** — a "no render
  yet" `ok` line, not a finding — so an unrendered state can never fail the lint, and
  a run over an empty cache passes the pin check having verified nothing.

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
