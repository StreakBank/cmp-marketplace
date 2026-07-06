---
name: design-intake
description: Implement a Compose Multiplatform screen from a SCREENSHOT — an arbitrary raster with no DOM, no tokens, no text layer (a Figma export, a competitor screen, a photo of a mock). Sequences the deterministic intake CLI (normalize + provenance + evidence sidecars), the model-driven transform grounded in the raster, and the iterate-until-faithful loop against the imported reference. Use when the design source is an image rather than a Claude Design frame.
argument-hint: <stateId> --image <path> [--config <dir>]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# /design-intake — Screenshot → faithful Compose (imported-reference leg)

Two judgment inputs are YOURS before the CLI runs (they are declarations the CLI
applies as a pure function, never auto-detected): the **content-box** (which
pixels are the screen vs foreign status-bar/OS chrome) and the
**theme-translation** (is this a cross-theme input — e.g. a light design for a
dark-only estate?). Everything deterministic lives in the CLI.

**CLI:** the standalone `cmp-design-bridge` command (install: `npm i -g
cmp-design-bridge`; ≥0.2.0 for `intake`). Optional: `npm i tesseract.js` in the
consuming repo enables the OCR evidence sidecar.

## Steps

1. **Declare + intake.** `Read` the raster; declare the content-box (crop out
   foreign chrome — status bar, browser frame, device bezel) and whether a theme
   translation applies:
   ```
   cmp-design-bridge intake <stateId> --config <repo>/.design-bridge \
     --image <path> [--content-box x,y,w,h] [--module <id>] \
     [--theme-translation light-to-dark]
   ```
   The module resolves from the longest config `statePrefix` — pass `--module`
   when the stateId matches none. Heed the inventory warning: enroll the stateId
   in the project's state inventory so the burndown + backfill gate see it.

2. **Read the evidence.** `Read` the normalized `<cache>/out/<stateId>.reference.png`
   (canonical geometry — measure directly in dp via the `<stateId>.grid.png`
   overlay), the intake manifest (palette census + token pairing), and the OCR
   sidecar if present (ADVISORY — the raster is authoritative over it).

3. **Author the Compose — translate to the design system, don't pixel-clone.**
   Apply the consuming project's faithfulness policy (its shim rule) if one
   exists; the default hierarchy: **existing DS component > DS token > raw value**
   (raw only as a flagged escalation with a code comment). Colors map by ROLE;
   for a declared `light-to-dark` input the token pairing is suppressed because
   nearest-hex is systematically wrong cross-theme — re-theme by role
   (background/surface/text tiers/primary action/status). Structure, spacing,
   copy (verbatim), and icon SEMANTICS are preserved; foreign chrome is never
   reproduced. Non-happy-path states (Loading/Error/empty) are NOT derivable
   from one raster — author them per the project's screen-shape rules. The
   authoring discipline itself is `/design-transform`'s (sealed UiState, shell
   hoisting, DS composables over re-authored atoms).

4. **Record the capture.** Emit a self-regression screenshot for the new state
   via the project's screenshot tool — basename MUST equal `<stateId>` (the
   derived-capture rule), and the capture harness must match the project's
   existing conventions exactly (same qualifiers, same capture call shape as the
   sibling tests in that module — harness drift confounds the grade).

5. **The loop.** Build the packet and grade:
   ```
   cmp-design-bridge verify <stateId> --config <repo>/.design-bridge --reference imported
   ```
   Grade the montage per the packet's `gradeInstruction` (for cross-theme inputs
   the color axis is ROLE consistency, not hex). Send targeted fixes back into
   step 3; re-record; re-verify. **Convergence:** verdict ∈ {MATCH, MINOR_DRIFT
   with only accepted residuals} AND a re-grade emits no new fix items. For
   high-stakes screens use multiple independent raters — single-grader variance
   is real. Record each deliberate translation deviation as an allowlist entry
   scoped with `states: ["<stateId>"]` (class `translation`) so it never
   suppresses drift on other states.

6. **Close the design-SoT loop (backfill).** The screenshot was the reference
   for IMPLEMENTATION; the project's design source of truth still needs a frame.
   Author the per-state frame from the SHIPPED implementation source (the
   project's standard frame-authoring recipe — that grounding is proven; a
   raster is not), upload via the gated `/design-push`, re-run `pull` +
   `lint`. Wire `lint --fail-on-backfill` into CI so a captured-but-frameless
   enrolled state blocks merge rather than lingering.

## What this skill never does
No deterministic screenshot→Compose generation (that is a bad transpiler); no
auto chrome detection (the content-box is your judgment); no cross-framework
pixel-diffing (misleading by design — the deterministic gate remains the
project's own self-regression tool). The fidelity grade stays ADVISORY;
acceptance of a NEW screen is the stable grade + a human look at the final
montage.
