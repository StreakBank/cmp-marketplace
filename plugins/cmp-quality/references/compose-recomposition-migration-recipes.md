# Compose Recomposition Migration Recipes

Two proven, project-agnostic recipes for remediating the recomposition-performance
findings the `performance-audit` agent reports. Both are **behavior-preserving** — the
completion gate for each is a screenshot self-regression showing **zero pixel drift**
(a perf fix that moves a pixel is a design change, not a perf fix).

Read the strong-skipping stability calculus in the `performance-audit` agent first — it
defines the finding bar these recipes remediate (report-unstable param + fresh
per-recomposition allocation + hot/wide subtree).

---

## Recipe 1 — Deferred-reads migration (per-frame composition reads)

**Symptom.** A composable reads an animated / infinite value (`rememberInfiniteTransition`,
`animate*AsState`, an `Animatable`) **in its composition body** and passes the resulting
`Float` / `Color` / `Brush` down as a plain parameter — or applies it through a
value-variant modifier. Every animation frame invalidates the whole subtree at ~60fps for
the duration of the animation (a loading shimmer, a toggle, an entry transition). Strong
skipping cannot help: the param genuinely changes every frame.

**The fix in one line:** move the animated read out of the **composition** phase into the
**layout** or **draw** phase, where a value change re-runs only layout / draw — not
recomposition.

### Step 1 — Site discovery (greps)

Find composition-phase animated reads:

```bash
# infinite / animated value READ in composition (the value is used as a param or arg,
# not deferred into a lambda) — the ".value" read or a destructured read is the tell
grep -rnE 'rememberInfiniteTransition|animate[A-Za-z]*AsState' --include=*.kt
grep -rnE 'Brush\.(linear|radial|sweep)Gradient' --include=*.kt      # per-frame Brush rebuild?

# value-variant modifiers fed by an animated arg (the eager overloads read in composition):
grep -rnE 'Modifier\.(offset|alpha|scale)\(' --include=*.kt          # value variant, not lambda
```

Triage each hit: is the animated value read **in the composition body** (assigned to a
`val`, passed as a param, or used to build a `Brush` / `Modifier` value)? If it is already
read inside a `drawWithCache` / `onDraw*` / `graphicsLayer {}` / `offset {}` lambda, it is
already deferred — skip it.

### Step 2 — Transform to a draw / layout-phase read

| Composition-phase read (before) | Deferred read (after) |
|---|---|
| Build a `Brush` from an animated offset in the body, pass it as a param | `Modifier.drawWithCache { onDrawBehind { /* build the Brush from the animated State here */ } }` — the cache lambda re-runs on size change; the draw lambda re-runs per frame with no recomposition |
| Pass an animated `Float` down N children as a plain param | Pass the `State<Float>` (not its `.value`) down; each child reads `.value` **inside its own draw lambda** |
| `Modifier.offset(x = animatedDp)` (value variant) | `Modifier.offset { IntOffset(animatedPx.roundToInt(), 0) }` (lambda variant — layout phase) |
| `Modifier.alpha(animatedFloat)` / `Modifier.scale(...)` (value variants) | `Modifier.graphicsLayer { alpha = animatedFloat; scaleX = ...; scaleY = ... }` (draw phase) |

The rule: **a value that changes every frame must be read in the lowest phase that consumes
it.** Scale / alpha / translation → `graphicsLayer` (draw). Position / size → the lambda
modifier variants (`offset {}`, layout). Per-frame gradients → `drawWithCache { onDrawBehind
{ } }`.

### Step 3 — The `@Composable`-getter trap

Design-system tokens (theme colors, typography, font families) are frequently exposed as
`@Composable get()` properties — they can only be read in a composable context, **not**
inside a `draw` / `layout` / `remember` lambda. When you defer an animated read into a draw
lambda that also needs a token:

1. Read the token into a **local `val` in the composition body** first (where the
   `@Composable` getter is legal).
2. Pass that local into the draw / layout lambda.
3. If the lambda is inside `drawWithCache` or `remember`, add the local as a **key** so the
   cache invalidates when the theme value changes.

```kotlin
@Composable
fun ShimmerBar(progress: State<Float>) {
    val base = Tokens.surface        // @Composable getter → read HERE, in composition
    val hi   = Tokens.highlight      // ditto
    Box(
        Modifier.drawWithCache(base, hi) {           // tokens as keys
            onDrawBehind {
                val x = progress.value               // per-frame read, draw phase
                drawRect(
                    brush = Brush.linearGradient(
                        listOf(base, hi, base), startX = x, endX = x + size.width,
                    ),
                )
            }
        },
    )
}
```

Forgetting step 1 is the classic failure — a token read attempted inside the draw lambda
won't compile (the getter is `@Composable`), which tempts a "just capture the raw color
literal" workaround that then breaks theming.

### Step 4 — Completion gate

Regenerate the module's screenshot self-regression comparison. **The migration is done only
when the diff is zero pixels.** A correctly-deferred read renders identically to the
composition-phase read — any drift means the transform changed behavior (wrong phase, a
dropped cache key, a mis-mapped modifier variant). Non-negotiable: a per-frame-read fix that
isn't pixel-identical is a regression, not a fix.

> **KDoc-must-match-code.** If a composable's KDoc *claims* draw-phase discipline ("reads
> inside a `drawWithCache` lambda…") while its code passes the animated value as a plain
> param, the claim is stale and actively misleading — the prose said the right thing and the
> code drifted away from it. Treat a doc/code mismatch here as a finding in its own right,
> and update the KDoc in the same change.

---

## Recipe 2 — Stability-config reconciliation

**Symptom.** A stability configuration file (declaring pure-Kotlin `data/api` types stable
without a Compose import) has **drifted** from the codebase: entries name classes that were
moved or deleted (dead FQNs), and newer domain types that ARE used as composable params never
got registered — so the report marks them unstable and their subtrees don't
`.equals()`-skip. Manual config maintenance is un-holdable; this recipe makes drift a **CI
failure** instead of an audit finding.

### Step 1 — Regenerate the reports

Per the `performance-audit` evidence step: enable `metricsDestination` / `reportsDestination`
and force generation with `--rerun-tasks` (the options don't invalidate up-to-date compile
tasks). You need fresh `*-classes.txt` (per-class inference) and `*-composables.txt`
(per-param stability) for every module.

### Step 2 — Diff report-unstable domain types against the config

From the reports, collect every class the compiler marks `unstable` that is **used as a
composable parameter** (or as a type argument of one — see the generics note). From the
config file, collect every declared FQN. Two directions:

- **Config entry → no such class** (dead FQN): the class was renamed / moved / deleted.
  Remove or correct the entry. Dead entries are silent — the config parser doesn't error on
  an FQN that resolves to nothing, so these rot invisibly.
- **Report-unstable param type → no config entry and no `@Immutable`**: an unregistered type.
  It's costing `.equals()`-skipping wherever it's a param.

**Generics propagation.** A registered container (`Resource<T>`, `Cached<T>`, `List<T>`) is
still unstable when `T` is unstable — the report shows the *instantiated* param unstable even
though the container is listed. Registering the container does nothing; you must register (or
`@Immutable`) the **type argument**. Never paper over it with a star-projection (`Foo<*>`)
entry — that masks which argument is the real problem.

### Step 3 — Register or annotate

For each unregistered unstable type used as a param:

- **`data/api` pure-Kotlin type** (no Compose dependency wanted): add its FQN to the
  stability config. This is the default for wire / domain models.
- **A type the module owns and can annotate**: `@Immutable` at the source is fine and more
  local — module-local UiState / models don't need the config at all.

Do **not** blanket-register everything the report lists — only types that are actually
composable params (or their type arguments). A domain type that never crosses into a
composable costs nothing.

### Step 4 — Regenerate to prove the flips

Re-run Step 1 with `--rerun-tasks` and confirm each previously-unstable param now reads
**stable** in `*-composables.txt`. This is the proof the fix worked — "I added the entry" is
not evidence; "the param flipped to stable in the regenerated report" is. If a param is still
unstable after registering its type, its type argument is probably the culprit (the generics
note above).

### Step 5 — Guard with a config-FQN existence gate

Drift recurs the next time a class moves. Close the loop with a **deterministic gate**: every
FQN in the stability config must resolve to a real class in the source tree; a config entry
that resolves to nothing **fails CI**. This kills the dead-FQN recurrence class outright — a
package move that orphans an entry breaks the build instead of silently degrading skipping
until the next audit.

The gate is a pure existence / drift check over `(config FQNs) × (declared classes)` — it
belongs alongside the other deterministic module-boundary gates in the **`cmp-arch-gates`
linter** (sibling plugin in this marketplace), not in an ad-hoc project script. The linter's
config seam is where the config-file path and any project type lists live; the gate mechanism
itself is generic. A deeper, report-driven variant — flag any report-unstable type that
appears as a composable param and has neither a config entry nor `@Immutable` — is expensive
(it needs report generation) and fits a `--full` / nightly verification tier rather than the
fast pre-commit gate.

---

## Running these under the migration-harness (classification + per-site verify)

Both recipes are runnable as staged migrations under the **`migration-harness`**
plugin (agent-marketplace) — the ledger/fan-out/escalation machinery lives there;
this section supplies the two contract slots the recipes above don't state.

**Classification table** (the discriminator: can the completion gate tell right from
wrong?):

| Recipe | Mostly | Escalation site(s) |
|---|---|---|
| 1 — deferred-reads | mechanical (the zero-drift gate catches a wrong transform) | any restructure whose equivalence argument must reason about **emission order**, not just values — e.g. hoisting a derivation across reactive-flow topology. A value-identical hop can reorder arrivals and leak a new intermediate state that only sequence-asserting tests catch; if you can't argue the ordering, escalate or skip. Also: resolving a KDoc-vs-code contradiction (which one is the intent?) |
| 2 — stability-config | mechanical (the regenerated report is the proof) | a type where registering-vs-annotating is a policy fork — adding `@Immutable` would put a Compose dependency on a deliberately Compose-free module, or the type is shared by consumers with different stability expectations |

**Per-site verify declarations** (templates come from the project shim):

```sh
node "$ML" add-verify --id compile --type command --template "<per-module compile task for {module}>" --granularity module
# Recipe 1: the zero-drift screenshot compare is the completion gate (whole-module);
# Recipe 2: the per-site proof is the regenerated report flip — declare it as a
# command probe if the project exposes a per-module report task, else it stays a
# completion-gate obligation.
```

---

## Where the project-specific half lives

Both recipes are generic mechanics. The **facts** stay in the consuming project's shim:

- which types are registered in the stability config, and the config file's path;
- the exact Gradle property that gates report generation, and the module compile-task names;
- the screenshot-tool invocation that provides the zero-drift completion gate;
- the reference composables in the codebase that already model deferred-phase reads.

Keep those facts in the project's `.claude/` estate; keep these recipes tool-generic.
