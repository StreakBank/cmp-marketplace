# SHIM TEMPLATE — copy into your project's `.claude/rules/`

Copy the block below into `<your-project>/.claude/rules/arch-gates.md` and replace
every `<placeholder>`. This is a **path-keyed** rule — the `paths:` frontmatter
loads it only when matching files are touched. Do **not** force-load it (don't
`@`-import it from a top-level instructions file); the gates are a tooling concern,
not always-on context.

Keep this file's project facts in sync with `<your-project>/.arch-gates/config.json`
— the config is what the CLI reads; this rule is what a coding agent reads.

---

```markdown
---
paths:
  - "<kmp-module-root>/**/build.gradle.kts"
  - "<kmp-module-root>/**/data/api/**"
  - "<kmp-module-root>/**/data/impl/**"
  - "<kmp-module-root>/.arch-gates/**"
---

# arch-gates — <project> shim

The generic core is the `cmp-arch-gates` plugin skill + CLI. It owns the mechanics
(the layered-DAG + boundary/visibility gates, the `.arch-gates/config.json` seam,
`init`). This rule adds only what the plugin deliberately doesn't know about this
project.

## Project facts (mirror of `.arch-gates/config.json`)
- Transport module(s): `<:core:network>` — implementation-only, never api().
- Cross-owner data:api allowlist: `<owner/data/api> -> <:dep:data:api>` — <why it's allowed>.
- Banned data:api imports: <defaults, or this project's overrides>.
- Architecture doc: <path/to/your/arch-doc.md>.

## How it runs here
- CI: `npx cmp-arch-gates@<pinned-version> check` (see `<ci-workflow-file>`).
- Local: `cmp-arch-gates check` from the module root, or as a pre-commit hook.
- <Any gate kept as a project-local script instead of in the CLI, and why — e.g.
  a platform-specific visibility check with no place in the generic module graph.>

## Policy
- New cross-owner `data:api` `api()` edge → add it to the allowlist WITH a reason,
  or remove the edge. Never widen the banned-import list to make a leak pass.
- A NEW gate idea that is generic (true for any KMP repo) → contribute it to the
  `cmp-arch-gates` CLI, not here. A fact true only for this project → it stays here
  / in `.arch-gates/config.json`. (The core/shim discipline: generic → plugin,
  project facts → shim.)
```
