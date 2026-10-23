---
name: design-fidelity
description: Verify a Claude Design frame against the shipped Compose app's self-regression (screenshot) baseline — cross-framework fidelity grade. Use after pulling/updating a design frame, or to check that a Compose screen still matches its design. The deterministic gate is the app's own self-regression gate; this skill adds the advisory cross-framework grade the old pixel-parity-vs-prototype gate used to provide.
argument-hint: <stateId> [--config <dir>]
allowed-tools:
  - Bash
  - Read
  - Glob
---

# /design-fidelity — cross-framework fidelity grade (VERIFY leg)

This skill is a THIN on-ramp. The deterministic mechanism (render the REFERENCE
design side, locate the SUBJECT app baseline, build the comparison packet +
montage, run the mechanical gates) is the `cmp-design-bridge verify` CLI. The
GRADE itself — "do these two engine-different renders depict the same screen at
acceptable fidelity?" — is the irreducibly model-driven leg, and YOU do it.

**CLI:** requires the separately installed `cmp-design-bridge` 0.3.0 command
(`npm i -g cmp-design-bridge@0.3.0`). The wrapper does not upgrade the CLI;
arrange that outside this skill before consuming the new evidence contract.

## Steps

1. **Build the fidelity packet** (deterministic). Refresh the actual frame and
   every same-folder `_*.jsx` helper through `/design-pull` first; CLI `pull`
   validates staged bytes and does not fetch canvas content. Then:
   ```
   cmp-design-bridge verify <stateId> --config <repo>/.design-bridge --render
   ```
   This writes `<cache>/out/<stateId>.{design.png, montage.png, fidelity-packet.json}` (imported-reference mode: the reference artifact is `<stateId>.reference.png`; the packet's `designPng` field aliases it, so read the packet's paths rather than assuming basenames)
   and prints the mechanical-gate result (render console-clean, width-ok,
   subject capture exists). If the mechanical gates FAIL, stop and fix the render
   before grading.
   The canonical config resolves exact enrolled `stateOverrides`, inheriting
   omitted values and treating null clip height as intrinsic height. Rendered
   `verify` also refreshes missing/stale recipe evidence without `--render`;
   inspect `effectiveRecipe`, `recipeHash` and `recipeCurrent` in the evidence.
   Imported mode cannot re-render a raster: missing/stale recipe evidence requires
   a fresh intake with the intended source/crop before grading.
   A new recipe does not close cross-folder or runtime-asset hash gaps; verify
   those dependencies under the project's own source/asset checks. If a shared
   helper changed, refresh all affected states before stamping pins and gate with
   `lint --no-stamp` (which writes a report, not the join manifest).

2. **Read the packet + both images.** `Read` the `designPng`, the `cmpPng`, and
   the `fidelity-packet.json` (it carries the allowlist + the per-state
   presence-contract inline).

3. **Grade, applying the rules in the packet's `gradeInstruction`:**
   - IGNORE engine differences (Skia line-box ~1.4× CSS, elevation vs shadow,
     AA edges, M3 padding) — never findings.
   - TOLERATE fixture differences (different streak/rank/accuracy values, avatar
     glyph) — classify "fixture", never "real-drift".
   - APPLY the `allowlist` (pair by rendered hex/geometry, not token name).
   - USE the `presenceContract`: absent-but-in-`present` → candidate real
     finding; absent-and-in-`absentByGating` → suppressed.
   - Verdict ∈ {MATCH, MINOR_DRIFT, MISMATCH}; list only REAL drifts, each with a
     recommendation (fix-CMP / fix-design / fixture-align / needs-human).

4. **For thoroughness on a high-stakes screen** (or when the user asks for a
   rigorous grade), run the multi-rater + adversarial workflow instead of a
   single-pass grade: 3 independent raters → synthesis → skeptic. See the
   reference workflow in the framework docs (`design-pipeline/`); cap image
   agents at ≤3 concurrent (the 429 finding).

5. **Report**: the verdict + the real-drift list. State plainly that this grade
   is ADVISORY; the deterministic gate is the Tier-2 self-regression gate
   (`packet.selfRegressionGate`), which is unchanged and prototype-independent.

## What this skill does NOT do
- It does not pixel-diff (pixel diffs mislead across Skia-vs-CSS).
- It does not gate a merge by itself — the app's self-regression gate is the gate.
- It does not author Compose — that's `/design-transform`.
