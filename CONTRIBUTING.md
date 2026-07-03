# Contributing — the core/shim discipline

Every plugin in this marketplace is built for **cross-project reuse** across any
Kotlin Multiplatform / Compose Multiplatform codebase. The rules below are the
admission gate; a plugin that fails them doesn't merge.

This marketplace currently hosts three plugins:

- **cmp-scaffold** — generation skills (scaffold a feature module, add a screen,
  wire networking/persistence/theming/navigation, etc.).
- **cmp-quality** — audit agents (architecture compliance, dependency wiring,
  accessibility, performance, test quality).
- **cmp-design-bridge** — design→Compose skills that wrap the published
  `cmp-design-bridge` npm CLI (pull a Claude Design frame, transform it into
  idiomatic Compose, grade cross-framework fidelity against a Roborazzi capture).

## 1. Generic core only

A plugin contains tool mechanics, protocols, command references, orchestration
patterns, and vendored+pinned upstream references — nothing else. Project policy
(which module layout a specific app chose, device/AVD/instance names, QA-layer
rulings, severity mappings, output-path conventions) lives in a **thin shim** in the
consuming project's `.claude/` estate, never here.

**The grep test** — run before every merge; all three must return nothing:

```bash
grep -riE '<any project name>' plugins/<name>/          # zero project names
grep -rE '/Users/|/home/|~/Projects' plugins/<name>/    # zero absolute project paths
grep -rE '\.claude/(rules|skills)/[a-z0-9._-]+' plugins/<name>/  # zero NAMED project-rule/skill refs
```

This is mechanized in `scripts/check-coupling.sh` (see §"The grep test is mechanized"
below) — don't hand-run the three greps above and call it done; run the script.

(Telling the consumer "write a shim in your project's `.claude/rules/`" is fine —
that's the discipline. Citing a *named* rule file is coupling.)

**Exempt from the grep test:** the hosting org's name in repo-address lines of
install docs (`claude plugin marketplace add <org>/cmp-marketplace` must name the
org) and `author`/`owner` attribution fields in `.claude-plugin/*.json`. Those are
distribution metadata, not content. The `cmp-design-bridge` plugin's own install
lines (`npm i -g cmp-design-bridge`, `npm install cmp-design-bridge`) are likewise a
legitimate install address for the wrapped CLI, not a project-name leak — see the
`ADDR_RE` carve-out in `scripts/check-coupling.sh`. Everything else the model reads
as instructions — SKILL.md, agent `.md` bodies, scripts, references, PROVENANCE —
gets zero exemptions.

Instance identifiers are the subtle case: an emulator serial, an applicationId, a
package name chosen by a specific app are project facts even when they look like
tool arguments. The core writes `<applicationId>` / `<module-name>`; the caller
supplies values.

The core also never *references* a shim — it must be complete without one. Shims
narrow; they don't complete.

## 2. Parameterization order

Prefer the earliest mechanism that fits: (1) environment detection when unambiguous
(e.g. detecting an existing module layout by reading `settings.gradle.kts`);
(2) invocation arguments for per-call facts (a feature name, a screen name);
(3) a project config file for durable machine-readable facts consumed by
deterministic tooling (e.g. `.design-bridge/config.json` for cmp-design-bridge);
(4) the project shim for policy. Data flows through detection/args/config; policy
flows through shims; nothing project-shaped flows through the core.

## 3. Naming and versioning

- Capability-based kebab-case, named for what it does, not what it wraps
  (`cmp-design-bridge`, not `claude-canvas-cli`). Scope prefixes only when the scope
  is real — the `cmp-` prefix here signals "targets Compose Multiplatform," not a
  specific consuming app.
- Semver in each plugin's `plugin.json`, starting `0.1.0`. Bump the marketplace's
  `.claude-plugin/marketplace.json` `metadata.version` on any plugin change. Every
  version bump gets a `CHANGELOG.md` line — the changelog is the load-bearing
  artifact; Claude Code does not enforce semver.

## 4. Wrapped binaries pin

A plugin wrapping an external binary or published CLI records the exact validated
version and installs from a version-pinned source with checksum verification when
the vendor supports it. `cmp-design-bridge` wraps the published `cmp-design-bridge`
npm package — pin the exact npm version validated against in the plugin's
PROVENANCE.md (if/when one is added) rather than floating on `latest`. Skills never
auto-update their wrapped binary; bumping a pin = edit the pinned version, re-validate,
commit.

## 5. Provenance and licensing

Vendored upstream content (reference specs, guidelines, skill fragments copied from
someone else's repo) requires:

- the upstream LICENSE file adjacent to the vendored content;
- a `PROVENANCE.md` in the plugin recording the upstream repo, **commit hash**, exact
  files taken, and every pruning/edit applied (this is what makes refresh diffs
  possible);
- no live-fetching of unpinned upstream content at runtime, ever.

Authored-original content (written for this plugin, not copied from anywhere) does
NOT need a PROVENANCE.md just because it lives under a `references/` directory —
but if a plugin has a `references/` dir, `scripts/check-coupling.sh` check #5 still
requires a PROVENANCE.md to exist, specifically so the file can state plainly
"nothing here is vendored" rather than leaving the provenance question unanswered.
See `plugins/cmp-scaffold/PROVENANCE.md` for the reference shape of that statement.

## 6. Third-party vetting

Any third-party skill content gets scanned with
[`cisco-ai-skill-scanner`](https://github.com/cisco-ai-defense/ai-skill-scanner)
before it enters this repo, and re-scanned on every refresh.

## 7. Skill and agent quality bar

- Skills are thin: the SKILL.md documents mechanics and sequencing; deterministic
  logic beyond a screenful belongs in a script (or a standalone CLI the skill calls,
  as cmp-design-bridge's skills do).
- Agents are self-contained: all execution criteria are inline in the agent's `.md`
  body — an agent has no conversation history and no access to a separate rules
  file, so nothing it needs can live behind indirection.
- Frontmatter: `name`, a `description` written for model invocation (triggers, not
  marketing), `allowed-tools`/`tools` as narrow as practical.
- The README quickstart must make sense to a stranger with none of the authoring
  project's context.

## 8. Feeding learnings back (the compounding loop)

Plugins improve by *use*, not just at authoring time — this is the return edge that
makes learnings compound across projects instead of dying in whatever project
happened to hit the gotcha. When a session (in ANY consuming project) discovers a
**generic** fact about a plugin's tool — a new CLI flag, a behavior the docs don't
mention, an installer bug, a better protocol for driving the wrapped CLI — it belongs
in the **plugin**, not in the consuming project's notes:

1. Clone the marketplace if it isn't already local: `gh repo clone StreakBank/cmp-marketplace`.
2. Edit the owning plugin's `SKILL.md`/agent `.md` (behavior/protocol), `scripts/`
   (installer/tool bug), or `PROVENANCE.md` (validation/version facts).
3. Bump the plugin's `plugin.json` semver + add a `CHANGELOG.md` line.
4. Run `scripts/check-coupling.sh <plugin>` and push. CI re-gates on merge.

The discriminator is the same core/shim test (§1): if the learning is true for any
KMP/CMP project using the tool, it's generic → upstream to the plugin. If it's true
only for one project (which module layout it chose, a specific app's screen names,
an ownership ruling), it's a **shim** fact — it stays in the consuming project's
`.claude/rules/` and never comes here.

A consuming project that captures agent learnings in its own always-loaded knowledge
file (e.g. a `.claude/agent-learnings.md`) should carry a carve-out telling sessions
to route generic tool learnings here instead of into the project silo — otherwise the
force-loaded "log it locally" instruction silently swallows every generic discovery.

## 9. The grep test is mechanized

`scripts/check-coupling.sh` is the single canonical implementation of the three
greps in §1 — plus manifest/provenance checks CI also gates on:

1. project names (`PROJECT_NAMES`, default `"streakbank ladderpicks"`) absent from
   model-facing content, minus the repo-address/npm-install carve-outs;
2. zero absolute project paths (`/Users/`, `/home/`, `~/Projects`);
3. zero named `.claude/(rules|skills)/<file>` references outside `*TEMPLATE*` files;
4. every plugin has a `.claude-plugin/plugin.json`;
5. every plugin with a `references/` dir has a `PROVENANCE.md`;
6. `.claude-plugin/marketplace.json` is valid JSON, lists every plugin on disk, and
   every listed plugin's `plugin.json` parses.

Run it before every push:

```bash
sh scripts/check-coupling.sh              # all plugins
sh scripts/check-coupling.sh cmp-scaffold # one plugin
```

CI (`.github/workflows/ci.yml`) runs the same script on every push/PR, plus
shellcheck on every `*.sh` and a JSON-validity pass over every `plugin.json` /
`marketplace.json`. A plugin that only passes locally because a maintainer forgot to
run the script will still be caught here.

## 10. Validation before merge

- Run the grep test / `scripts/check-coupling.sh` (§9).
- Exercise every documented command against a real target (a scratch KMP project,
  the `cmp-design-bridge` CLI against a real Claude Design canvas — whatever the
  plugin drives) and note the validation date + tool version in `PROVENANCE.md` when
  one exists.
- Confirm the authoring project still works with the plugin installed globally and
  its local copy (if any) deleted — extraction that breaks the origin is a
  regression.
