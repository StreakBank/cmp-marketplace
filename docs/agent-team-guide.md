# Agent Team Feedback Loop Guide

Validate CMP marketplace plugins against a reference project using Claude Code agent teams.

## Prerequisites

1. **Enable agent teams** in `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "NODE_OPTIONS": "--max-old-space-size=8192"
  }
}
```
> **Why NODE_OPTIONS?** Agent teams spawn multiple Node.js processes. Without this, the default ~4GB heap limit can cause OOM crashes when teammates invoke skills/agents that read many files.

2. **Install plugins** in the target project:
```bash
cd /path/to/example-project
claude plugin marketplace add /path/to/cmp-marketplace
claude plugin install cmp-scaffold@cmp-marketplace --scope project
claude plugin install cmp-quality@cmp-marketplace --scope project
```

3. **Add team protocol** to the target project's `CLAUDE.md` (see [example-cmp template](#claudemd-template) below).

## Launch Prompt

Start Claude Code from the **target project** directory, then paste:

```
Create an agent team to validate our CMP marketplace plugins against this
reference project. The marketplace lives at ../cmp-marketplace/ and contains
16 scaffold skills and 7 quality agents for KMP/CMP architecture.

IMPORTANT: Spawn only ONE teammate at a time to avoid OOM crashes. Shut down
each teammate before spawning the next. Work through these phases sequentially:

Phase 1 — Spawn **Auditor** (read-only, general-purpose agent):
Run the 4 quality agents one at a time against this project:
  1. audit-architecture
  2. dependency-audit
  3. validate-module products
  4. accessibility-audit
Report structured findings: total checks, PASS/FAIL/WARN counts, and every
FAIL with file path and description. Do NOT modify any files.

After Phase 1: I'll review findings before continuing.

Phase 2 — Spawn **Plugin Fixer** (writes only to ../cmp-marketplace/):
Based on the audit findings I share, fix plugin files (SKILL.md, agent .md,
reference .md). Report every file changed and what was fixed.

Phase 3 — Spawn **Scaffolder** (writes only to this project):
Run scaffold skills to fill gaps (scaffold-tests products,
add-convention-plugins, etc.). Report what was generated and any issues.

Phase 4 — Spawn **Verifier** (runs commands only, no file writes):
Run ./gradlew :composeApp:assembleDebug and any generated tests. Report
pass/fail with error details.

Phase 5 — Spawn **Auditor** again for re-audit:
Re-run all 4 quality agents. Compare against Phase 1 baseline. Report delta.

Use delegate mode — coordinate but don't implement yourself. I want to
review findings between phases before moving on.
```

## Phases

| Phase | Teammate | Action | Success Criteria |
|-------|----------|--------|-----------------|
| 1 | Auditor | Run all 4 quality agents | Structured PASS/FAIL/WARN counts reported |
| 2 | Plugin Fixer | Fix marketplace files based on findings | All targeted files updated, changes reported |
| 3 | Scaffolder | Run scaffold skills to fill gaps | Generated code reported, no skill errors |
| 4 | Verifier | Build and test | `assembleDebug` passes |
| 5 | Auditor | Re-run all audits | FAIL count decreased, no new FAILs |

### Convergence Criteria (Phase 5)
- FAIL count decreased from Phase 1 baseline
- No new FAILs introduced
- If FAILs persist, lead decides: iterate (back to Phase 2) or flag for human review

## Team Design

### Why the split avoids file conflicts
| Teammate | Writes to | Reads from |
|----------|-----------|------------|
| Auditor | Nothing (read-only) | example-cmp |
| Plugin Fixer | ../cmp-marketplace/ only | Both repos |
| Scaffolder | example-cmp only | Both repos |
| Verifier | Nothing (commands only) | example-cmp |

No two teammates write to the same repo simultaneously.

### Lead behavior
- Uses **delegate mode** (Shift+Tab) — coordinates but doesn't implement
- Reviews findings between phases before advancing
- Creates targeted tasks for Plugin Fixer and Scaffolder based on audit results

## CLAUDE.md Template

Add this to the target project's `CLAUDE.md`:

```markdown
## Agent Team Protocol

This project is being validated by an agent team coordinating between two repos:
- **This repo** (example-cmp): KMP reference implementation
- **../cmp-marketplace/**: Claude Code plugin marketplace

### Teammate roles
- **Auditor**: Read-only analysis of this project. Do not modify any files.
- **Plugin Fixer**: Only modify files in ../cmp-marketplace/. Do not touch this repo.
- **Scaffolder**: Only modify files in this repo. Do not touch ../cmp-marketplace/.
- **Verifier**: Only run build/test commands. Do not modify any files.

### Key project values for context detection
- rootProject.name = "Example" → resource_prefix = "example"
- namespace = "com.example.app" → package_base = "com.example"
- Feature modules: products, cart, favorites, settings
- composeApp DI: composeApp/src/commonMain/kotlin/com/example/app/di/AppModule.kt
- NavHost: composeApp/src/commonMain/kotlin/com/example/app/navigation/ExampleNavHost.kt
```

## Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Teammates edit same files | Strict role separation: each teammate owns one repo or is read-only |
| Lead implements instead of delegating | Use delegate mode (Shift+Tab) |
| Auditor uses stale plugin logic | Plugins installed from local path, so updates are picked up on next invocation |
| Build failures block progress | Verifier reports errors, lead creates fix tasks for Scaffolder |
| Agent team feature is experimental | Known limitations: no session resume, task status can lag. Monitor and steer. |
| Node.js OOM with multiple teammates | Set `NODE_OPTIONS=--max-old-space-size=8192`. Spawn only 1 teammate at a time; shut down before spawning next. |
| Token cost | Keep team to 4 teammates max. Use Sonnet for teammates where possible |

## Post-Run Verification

After the team completes:
1. Check `../cmp-marketplace/` for updated plugin files
2. Check example-cmp for new generated code (tests, convention plugins, etc.)
3. Build passes: `./gradlew :composeApp:assembleDebug`
4. Re-audit FAIL count is lower than initial audit
5. Review git diff in both repos to ensure changes are sensible
