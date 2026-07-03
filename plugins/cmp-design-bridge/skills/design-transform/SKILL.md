---
name: design-transform
description: Transform a pulled Claude Design frame into idiomatic Compose Multiplatform under the project's declarative-stack rules. Use to implement a NEW design state or update an existing screen from a changed design. This is a judgment task (re-authoring, not transpiling) — the skill sequences the deterministic CLI inputs and you do the Compose authoring.
argument-hint: <stateId> [--config <dir>]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# /design-transform — Design → idiomatic Compose (TRANSFORM leg)

The TRANSFORM is irreducibly model-driven: re-authoring a design under the
declarative-stack rules is a judgment task, NOT a transpile. Forcing it into a
deterministic generator produces a bad transpiler. So the CLI prepares the
inputs (the rendered frame, the source JSX, the conventions, the token maps) and
YOU author the Compose.

**CLI:** the skill calls the standalone `cmp-design-bridge` command (install:
`npm i -g cmp-design-bridge`).

## Steps

1. **Ensure the frame is pulled + rendered** (so you can see the target):
   ```
   cmp-design-bridge render <stateId> --config <repo>/.design-bridge
   ```
   `Read` the rendered `<cache>/out/<stateId>.design.png`.

2. **Read the design source + the conventions.** The frame's `_<module>-screen.jsx`
   (the structural source) + `_<module>-data.jsx` (the fixture) under
   `<cache>/canvas/<module>/`, and `<repo>/.design-bridge/conventions.md`
   (token map, the cited declarative-stack rules, the component seam).

3. **Author the Compose under the declarative-stack rules** (cited in
   `conventions.md`): sealed `Error | Active(Loading | Loaded)` UiState;
   hoist the shell above the screen-root `when`; one slot per state-transition;
   client-owned selection; consume `core/designsystem` composables over
   re-authoring atoms. Use `cmp-scaffold` generators so the module's layout, DI,
   navigation, and UiState shape match hand-written code. Map tokens per
   `conventions.md` (px→dp 1:1, 600→SemiBold, accepted deviations → the CMP value).

4. **Emit a self-regression screenshot stub** for the new state via
   <the project's screenshot/self-regression tool> so the Tier-2 gate has a
   baseline to record. The capture basename MUST equal the `<stateId>` (the
   §9.1 derived-capture rule).

5. **Verify** with `/design-fidelity <stateId>` (advisory) and record the Tier-2
   baseline via <the project's gate-record command> — the deterministic gate.

## Honest ceiling
SVG→DrawScope charts (sparklines, distribution viz) are the least-automatable
part — vision can tolerate but not *verify* their internals. Author these by
hand against the project's canvas-dp rules (numeric Canvas dimensions must be
dp/sp converted, never raw px).

## Visual authority boundary
`design-transform` and the `cmp-scaffold` `polish-ui` skill hold opposite
theories of visual authority — do NOT run both on the same screen. When a
Claude Design canvas frame is the source of truth for a screen, `design-transform`
owns its visuals end-to-end; skip `polish-ui` for that screen entirely.
