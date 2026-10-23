# CLI provenance and release check

This plugin contains authored skill wrappers, not a vendored CLI. The binary is
installed separately from the published npm package; skills do not update it.

- CLI source: https://github.com/StreakBank/cmp-design-bridge
- Tested source commit: `b25ead16ee3f453685be2113449fd4000e2cb64d`.
- CLI release: **0.2.1**; wrapper **0.3.3**, marketplace **2.17.3**.
- CLI entry-point SHA-256: `d85b590fdfaf59a82c0df6d8c51ac4b2dc256c6d0a1056a0bc02d37f222450ef`.
- Argument-test SHA-256: `05b7144ec8bf9c07b918e5a662a22313c374548d3228b8a93687a4d64907b6ba`.
- Prepared npm tarball SHA-256: `a2d2dbd326ffc3829e9c0ca395d2c0b4d589eeb32fa0b1ee459846a9cce2d400`.
- Prepared npm integrity: `sha512-dtVfA9ttITJxFxKJhBni2M4TVbXJxvBm0ufA4TdPsdwiIUS5FPGTKg7f+vhFzjrST2QNlFqWHBeLwZgXQki7Jw==`.
- Validation 2026-10-22: all 47 CLI argument and existing smoke tests passed,
  zero skipped/cancelled; Playwright/core 1.58.2. All 18 packaged files match
  the tested source bytes. Real consuming CONFIG `lint --no-stamp` passed
  without changing any CONFIG files. This is structural evidence only;
  no fresh canvas synchronization or render-fidelity claim is made.
- Scope: per-command argument validation before CONFIG/dispatch, safe root and
  command help, and regression coverage. Command implementations, dependencies,
  and default stamping are unchanged. README documents local-render freshness.

Registry release and CLI consumer adoption verified 2026-10-23: npm version
`0.2.1` reports the exact integrity above. The global published-package install
in the global npm package directory, reached by the installed CLI entry point,
matches all 18 reviewed packaged files.
Root help succeeds; unknown commands and flags fail with exit 2 before config
loading. The installed consumer passes `lint --no-stamp` against a real consuming
configuration without changing its metadata. This is structural validation,
not a new render or canvas synchronization claim.

The release was installed from npm at the exact version, with lifecycle scripts
disabled. The release record retains registry integrity, the 18-file byte
comparison, command checks and the no-stamp lint report.

Revalidate on the next CLI pin bump or a change to its argument/command contract.
