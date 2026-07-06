# Architecture Migration Recipes

Five proven, project-agnostic recipes for architecture-boundary and state-lifecycle
migrations in a KMP / Compose Multiplatform module graph — the structural siblings of
`compose-recomposition-migration-recipes.md` (which covers the recomposition-performance
classes). All five were validated by real executions on a production multi-module
codebase before being written down here.

Each recipe is written against the **`migration-harness`** plugin's recipe contract
(agent-marketplace, `references/RECIPE-CONTRACT.md`): target invariant → site
discovery → **classification table** (which sites are mechanical, which must
batch-escalate to a human) → transform → **per-site verify declarations** → completion
gate → known failure modes. The harness supplies the machinery (ledger, partitioned
fan-out, the block-until-signed-off gate); these recipes supply the domain content.
The classification discriminator, per site: *"can the completion gate tell right from
wrong?"* — if the gate is blind to a wrong-but-compiling choice, the site escalates.

Deterministic completion gates referenced below live in the **`cmp-arch-gates`** CLI
(sibling plugin): module-direction, data-api purity, datasource visibility,
transport-free presentation.

---

## Recipe A — Visibility-`internal` sweep

**Target invariant.** Feature-module `Screen` / `ViewModel` / `UiState` / view
composables and `data/impl` datasource types are `internal`; only genuine cross-module
entry points (graph routes, `navigateTo*` extensions, DI-consumed factories) are
public.

**Site discovery.** Grep default-public (no-modifier) top-level declarations under
`*/feature/src/commonMain` and `*/data/impl/src/commonMain`; subtract the project's
public-entry allowlist. False positive: a declaration that *looks* module-private but
has a cross-module consumer — confirm by grepping the symbol's imports across modules
before classifying.

**Classification table.**

| Mostly | Escalation site(s) |
|---|---|
| mechanical (add `internal`) | a candidate that satisfies the lint as `internal` but is a genuine deep-link route, reflection/DI target, or published test-fixture that must stay public — the compile gate can't distinguish "nobody imports it yet" from "external consumers exist outside this build" |

**Transform.** Add `internal`; one word per site. The only judgment is the escalation
row above.

**Per-site verify.**

```sh
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
```

**Completion gate.** The visibility lint (datasource-visibility gate + any
feature-visibility check the project wires) + full build. Green lint is the invariant;
this is the recipe with the tightest gate-to-invariant fit.

**Known failure modes.** Cross-module consumers hiding in platform source sets
(`androidMain`/`iosMain`) or test fixtures that the commonMain grep missed; screenshot
tests that render screens directly need `internal` (not `private`) — over-tightening
to `private` breaks them.

---

## Recipe B — Dead-code / dead-edge removal

**Target invariant.** No unreferenced composables/functions ship in production source;
no Gradle `project(":…")` edge remains whose symbols are never imported by the
declaring module.

**Site discovery.** Three feeds: (1) zero-caller greps for `private`/module-internal
functions; (2) a declared-vs-imported module-dependency diff (parse `project(":…")`
edges, grep the module's source for imports from each — the module-direction gate's
extractor is the reference parser); (3) the compiler's unused-param report for dead
parameters. Inventory each with the *evidence of deadness* (zero call sites, zero
imports), not just the location.

**Classification table.**

| Mostly | Escalation site(s) |
|---|---|
| gate-caught (build + tests + screenshot zero-drift prove the delete was truly dead) | a delete the gate cannot prove dead: reflectively-referenced or DI-string-referenced symbols, entries referenced from resources/manifests, and "dead vs dormant" product judgments (a parameter whose consuming feature is planned, not retired) |

**Transform.** Delete the symbol/edge *and its stale KDoc/comments*; a dead edge's
justification comment is part of the site.

**Per-site verify.**

```sh
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
node "$ML" add-verify --id residue --type grep-absent --pattern "<deleted symbol name(s)>" --paths <src-roots>
```

**Completion gate.** Full build + full tests + screenshot self-regression at **zero
drift** (pixels moving means the delete was not dead) + the dependency-graph gates
still green.

**Known failure modes.** An agent's file deletion silently denied by its sandbox but
reported done — the lead re-verifies deletions with `find`/`ls`. Blast radius beyond
`.kt`: API specs, docs, scripts, committed generated bundles, test-harness fixtures —
sweep all file types and use the VCS-tracked grep for the authoritative residual.

---

## Recipe C — Transport → domain-error boundary translation

**Target invariant.** Transport/HTTP exceptions translate to a sealed domain-error
type at the data boundary; presentation modules import no transport type; the central
user-message sanitizer switches over the domain cases.

**Site discovery.** (1) Every reference to the transport error type / transport
error-detection helpers outside the transport module; (2) the boundary chokepoint(s)
where translation will live (the shared safe-call wrapper, per-source mappers); (3)
the **consumed-distinction inventory** — for every consumer, which accessor, which
status-code branch, which copy string it actually depends on. This inventory is
**mandatory before design**: the sealed case set is *derived* from consumed
distinctions, never invented. Splitting or merging cases without it silently changes
consumer behavior in ways every gate is blind to.

**Classification table.**

| Mostly | Escalation site(s) — batch these as ONE decision round |
|---|---|
| heavily semantic (this recipe is approval-gated by nature) | the sealed case set itself (which distinctions become cases); each consumer branch's mapping (present as a per-consumer mapping table); whether sibling domain-error types fold into the new type or stay as refinements (folding can silently change user-facing copy); the sanitizer's signature and copy-precedence rules |

**Transform.** **Foundation inline:** the sealed type + the chokepoint translation +
its tests are one semantic unit, lead-authored before any fan-out. **Consumers fan
out** file-partitioned once the boundary contract is fixed — with the decided mapping
table in each agent's brief. Copy-sensitive branches (payments, submissions) must be
byte-identical unless a decision says otherwise.

**Per-site verify.**

```sh
# The compile probe MUST resolve the classpath per touched module — a consumer that
# previously got the transport type transitively can lose it when edges are severed,
# and transform agents have self-attested past exactly this break in a real run.
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
node "$ML" add-verify --id no-transport --type grep-absent --pattern "<transport package import prefix>" --paths <presentation-src-roots>
```

Plus adversarial verification the ledger can't automate: a per-consumer
**copy-equivalence read** (old branch → new case → same user-facing string), a
residual sweep, and a completeness critic.

**Completion gate.** The `cmp-arch-gates` `transport-free-presentation` gate (import
prefix + Gradle-edge layers) + full build + full tests + platform compile canary
(iOS) — error types are commonMain-wide.

**Known failure modes.** The transitive-classpath break above (the reason per-site
verify is mechanical); copy drift hidden behind "equivalent" mappings — byte-compare
the strings; consumers enumerating the old taxonomy twice for different audiences
(user copy vs analytics tokens) — both enumerations are sites, and their partitions
must be preserved cell-for-cell unless a decision says otherwise.

---

## Recipe D — Threaded-param removal sweep

**Target invariant.** No production render path threads a retired
instrumentation/test-harness parameter; the retired primitive, its id constants, and
its documentation are gone.

**Site discovery.** Grep the parameter/primitive names across production source sets;
separate **API-surface declarations** (component signatures carrying the param) from
**pass-sites** (callers threading it). Then **verify the premise**: confirm the
consumer that justified the instrumentation is truly retired by locating (or failing
to locate) the consuming tool itself — in a real run, discovery proved the brief's
"preserve a sidecar for the surviving consumer" premise false (the consumer read a
different artifact entirely), flipping the plan to delete-all *before* any transform.

**Classification table.**

| Mostly | Escalation site(s) |
|---|---|
| mechanical (drop the param + pass-sites, delete the primitive) | what to preserve for a still-live adjacent consumer (a sidecar, a field subset another tool reads) — the gate can't know which outputs external tooling depends on |

**Transform.** Remove the param from each component signature; drop every pass-site;
delete the primitive files, orphaned id-constant holders, dedicated test files, and
the contract doc. Mechanically uniform; partition by module.

**Per-site verify.**

```sh
node "$ML" add-verify --id residue --type grep-absent --pattern "<param/primitive name regex>" --paths <src-roots>
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
```

**Completion gate.** Full build + screenshot self-regression at **zero drift** (a
semantics-tree/instrumentation param must not move pixels) + the surviving adjacent
consumer's own gate (e.g. a live e2e run) when the escalation decided something
survives.

**Known failure modes.** The wrong-premise case above (always premise-verify);
instrumentation references in docs, scripts, and harness fixtures beyond production
source; deleting the param but leaving the `@Suppress` / suppression comments that
referenced it.

---

## Recipe E — Flow start-mode sweep (`stateIn` Eagerly → WhileSubscribed)

**Target invariant.** No feature ViewModel uses `SharingStarted.Eagerly` for a UiState
`stateIn`; every derived flow uses the repo's uniform `WhileSubscribed(<timeout>)` so
upstream combines idle when nothing observes them.

**Site discovery.** Grep `SharingStarted.Eagerly` across `*/feature/src/*`. Separate
the `stateIn` sites (migrate) from plain `.asStateFlow()` / `MutableStateFlow`
widenings (NOT in scope — they have no sharing policy; `.value` is always live).
Inventory per site: what the combine derives from, whether the stateIn **initial**
value equals the first emission on fresh entry, and the VM's scoping (leaf-route vs
parent-graph multi-step — the classification hinges on it). Also inventory every test
reading a migrated flow's `.value`: stale counts from an old audit are likely a large
undercount — re-count.

**Classification table.**

| Mostly | Escalation site(s) |
|---|---|
| mechanical (initial == first emission; leaf-scoped single collector) | **cross-step pre-fill flash**: in a parent-graph-scoped multi-step VM, a later step's flow is COLD until its screen mounts, so a field entered on a prior step renders the initial (empty) value for the first frame before the combine emits. Eagerly masks this; every gate is blind to it (screenshot tests render hardcoded state; unit tests read `.value`). A human decides: accept the one-frame flash / repoint the display field to an always-live raw flow / keep Eagerly with a KDoc |

**Transform.** Swap the starter literal (match the repo's existing timeout convention
exactly). Then the test rework, which is most of the work: under WhileSubscribed a
stateIn'd flow's upstream NEVER runs without a collector, so every bare
`viewModel.flow.value` read stays frozen at the initial value. Add an active
background collector for the specific flow(s) each test reads (imitate an in-repo
precedent if one exists), including tests that would now FALSE-PASS because they
assert the initial value — those silently stop testing the combine otherwise. Never
weaken assertions; leave self-subscribing Turbine `.test{}` blocks alone.

**Per-site verify.**

```sh
node "$ML" add-verify --id no-eagerly --type grep-absent --pattern "SharingStarted\\.Eagerly" --paths <feature-src-roots>
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
node "$ML" add-verify --id tests --type command --template "<per-module test task for {module}>" --granularity module
```

**Completion gate.** Full build + full test suite + screenshot self-regression at
zero drift (starter changes must not move pixels) + platform compile canary
(commonMain-wide change).

**Known failure modes (observed in the validating run).**

- **Turbine sequence-test conflation:** a Turbine block over a previously-Eagerly
  flow starts the combine only at subscription; with a synchronous fake repository,
  the combine's source subscription races the submit and a transient emission
  (`Submitting`) conflates away — the sequence assertion fails. Fix by
  **pre-collecting** in a background scope before the Turbine block (this models the
  production subscriber, which is always collecting before the user can act) — do
  NOT weaken the sequence assertion; it is the gate that catches real emission-order
  changes.
- **Stale test-count premises:** an old audit's "N tests break" figure rots as tests
  accrue — re-inventory at discovery (the validating run found ~4–5× the deferred
  estimate).
- **Silent coverage loss:** tests whose assertion coincides with the initial value
  keep passing while testing nothing — they need collectors too, not just the
  hard-failing ones.

---

## Where the project-specific half lives

All four recipes are generic mechanics. The **facts** stay in the consuming project's
shim/config: the per-module compile-task and test-task templates; which visibility
allowlist entries exist and why; the transport module's package prefix; the screenshot
tool's zero-drift invocation; which platform canaries apply; the adjacent consumers
(e2e harnesses, external tools) whose survival Recipe D must check. Keep those in the
project's `.claude/` estate and `.arch-gates/` config; keep these recipes
tool-generic.
