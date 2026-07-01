# cmp-design-bridge (skills)

Claude Code skills for the [`cmp-design-bridge`](https://github.com/StreakBank/cmp-design-bridge)
Claude Design → Compose bridge. They sequence the deterministic CLI and invoke
the two model-driven legs:

- **design-pull** — pull a design's per-state frames from Claude Design and stage them.
- **design-transform** — render a frame and re-author the idiomatic Compose screen to match.
- **design-fidelity** — grade the rendered design against the app's screenshot (advisory verdict).
- **design-push** — publish authored/updated frames back to Claude Design.

## Prerequisite

These skills call the standalone CLI (a separate npm package):

```bash
npm i -g cmp-design-bridge
```

The CLI does the deterministic work (pull / render / lint / verify); the skills
sequence it and make the two irreducibly model-driven calls. Per-project config
lives in the consuming repo's `.design-bridge/`.
