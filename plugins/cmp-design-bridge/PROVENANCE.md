# CLI provenance and release check

This plugin contains skill wrappers. The standalone CLI is a separate npm
package; invoking a skill does not update that package.

## 0.3.1 release evidence

- CLI source: https://github.com/StreakBank/cmp-design-bridge
- Tested source commit: `ab27c3173f8b735b75d6a9eee18b57c9f8fc804d`.
- CLI release **0.3.1**; wrapper **0.3.5**, marketplace **2.17.5**.
- Scope: README learning return and version/changelog metadata only. CLI runtime,
  schemas, dependencies and the 0.3.0 recipe-evidence contract are unchanged.
  The wrapper adds upload-byte guidance and updates its existing prerequisite pins;
  skill permissions and command workflows are unchanged.
- Prepared npm tarball SHA-256: `0751d5f6aaacab1125e07bd52da18cc3563e90d179d98c7edc6373fe671cda58`.
- Prepared npm integrity: `sha512-01DkvDZv3ms6/0P+oqZQxHN02/Xj0Z3f+VB6DsKyO2krpqNHd238shckrNTEjgZ+hpwgnUZ5P6cdVX8CkFeCfQ==`.
- Validation 2026-10-23: **59 tests passed, zero skipped/cancelled**, including
  actual Chromium render and imported-intake cases. All **19 packaged files**
  equal the tested source. Existing wrapper coupling, shellcheck and JSON gates,
  scoped configuration audit and independent content review passed.
- The test command succeeded; its first receipt parser assumed TAP output while
  Node emitted the spec reporter. The retained output was reconciled without a
  test rerun. The initial package attempt was denied access to the user cache;
  packaging with a task-owned cache succeeded. No permissions were changed.
- The CLI repository has no configured Actions workflow; no CLI CI pass is claimed.
  Wrapper CI, publication and consumer adoption are separate delivery receipts.
- Before adoption, verify npm's exact 0.3.1 integrity against the value above,
  install the published package with lifecycle scripts disabled, and compare its
  packaged file hashes with this commit. Verify the installed 0.3.5 wrapper against
  its published commit and retain a real consumer's `lint --no-stamp` result.

## Prior 0.3.0 release evidence

- CLI source: https://github.com/StreakBank/cmp-design-bridge
- Tested source commit: `56c0a016bb8fd6045cdac1359d4b2620443853bf`.
- CLI release: **0.3.0**; wrapper **0.3.4**, marketplace **2.17.4**.
- CLI entry-point SHA-256: `d85b590fdfaf59a82c0df6d8c51ac4b2dc256c6d0a1056a0bc02d37f222450ef`.
- Recipe implementation SHA-256: `c56d2d158276c337542bce8e9101c1b7e3b13cacc2a27d914f077317e659dad9`.
- Prepared npm tarball SHA-256: `3fa2e9528e5c697f2a39a8c7991c8a822f49f785c5b34e795cc6a33ecacc26c1`.
- Prepared npm integrity: `sha512-9Qx8iguDIdk2UbuhjclETlEI5Z+w6EZSuO8OeYKA87zcneFZGCac4t2QndDhNG0cJCU1jae/pO4agdL3nxCFBg==`.
- Validation 2026-10-23: **59 tests passed, zero skipped/cancelled**, including
  actual Chromium render, imported intake, recipe-drift verification and fractional
  geometry cases. Playwright/core remain 1.58.2; dependency versions are unchanged.
  All **19 packaged files** match the tested source bytes. Independent review passed.
- The upstream CLI repository had no configured Actions workflows at release
  preparation. The local test run is the executed evidence; no CI pass is claimed.

The regression evidence preserves the predecessor's fractional intake viewport
failure and the first correction's fractional montage viewport failure. The final
implementation normalizes imported physical pixels consistently and rounds only
browser viewport bounds for the montage. Fractional CSS dimensions and display
scales remain valid inputs.

### Consumer migration

Version 0.3.0 adds exact enrolled-state recipe overrides and requires matching
portable recipe evidence on existing rendered/intake artifacts. Pre-0.3.0 renders
must be refreshed; imported references require fresh intake. Preserve curated
manifest rows, refresh authoritative frames and all same-folder helper inputs,
render every affected state, and gate on `lint --no-stamp` before deliberate
stamping. A state with no render still has no visual evidence; lint does not
establish canvas synchronization or visual fidelity.

### Publication and adoption evidence

At package preparation, the tarball above was tested locally and the source was
published to its task branch. Registry publication and global consumer adoption
are separate delivery steps, not established by the package or test receipts.
Before adopting the 0.3.4 wrapper, verify that npm's exact 0.3.0 integrity matches the
value above, install the published package with lifecycle scripts disabled, and
compare its 19 packaged file hashes to the reviewed source. Record the installed
package path/version and a real consuming configuration's fresh render and
`lint --no-stamp` results in the release record.

Revalidate these bindings on the next CLI pin bump or command-contract change.
