---
type: failure
concepts: [extension-event-hooks, permission-ruleset]
harnesses: [opencode]
---
**Symptom** — (Observed at `ecc4916b5a`.) Plugins implementing `permission.ask` to auto-allow or auto-deny permission prompts silently stopped having any effect: the hook is still declared in the public plugin API but is never called.

**Root cause** — The only trigger lived in the legacy permission module, deleted in `2fc06c5a17` 2026-03-14 "delete legacy permission module" (#17534); the declaration `"permission.ask"?: (input, output: {status: "ask" | "deny" | "allow"})` stayed (`packages/plugin/src/index.ts:261`). `grep -rn '"permission.ask"' packages/opencode/src packages/core/src` finds no trigger.

**Fix · [[opencode]]** — none at HEAD.

**Lesson** — Removing a hook's call site must remove or deprecate the hook in the public type; a declared-but-dead hook fails silently for every plugin that relied on it.

Related: [[extension-event-hooks]] · [[permission-ruleset]] · [[tool-call-gate]] · [[opencode--extension-event-hooks|opencode]]
