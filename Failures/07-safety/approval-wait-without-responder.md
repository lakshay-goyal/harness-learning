---
type: failure
concepts: [permission-ruleset, headless-rpc-mode, ci-agent-integration]
harnesses: [opencode]
---
**Symptom** — A run blocked forever on a permission prompt that no one could answer: headless `opencode run` sessions hung; later only the root session was answered, so subagent asks still hung; v2 agent-less embedded sessions waited for an approval surface the host did not expose.

**Root cause** — `ask` suspends a call until a client replies; each non-interactive entry point must supply a responder (or a deny default) for every session in the tree, and the evaluated ruleset must belong to the identity that actually executes.

**Fix · [[opencode]]**
- `936f4cb0c6` 2025-07-31 "permission state hangs".
- `a3ba740de4` 2025-10-31 "resolve hanging permission prompts in headless mode" (#3522): `run` auto-rejects every `permission.asked` with a warning, or replies `once` under `--auto` (`packages/opencode/src/cli/cmd/run.ts:801-820`).
- `08faeb3893` 2026-08-20 "answer subagent permissions in run" (#43675): responder covers every session of the run.
- v2: "Agent-less embedded Sessions previously executed as `build` while evaluating an empty permission ruleset, so the first local tool could wait forever" (`specs/v2/schema-changelog.md:635`) → agent-less sessions evaluate as the default `build` agent (`specs/v2/session.md:191`).
- Open (observed at HEAD, not reproduced): `opencode github run` subscribes only to part events and never answers `permission.asked` (`packages/opencode/src/cli/cmd/github.handler.ts:829-870`), so default `ask` rules (`external_directory`, `*.env` reads, `doom_loop`) would stall the CI job until timeout.

**Lesson** — Every approval channel needs a responder or a deny default for each session in the delegation tree, per entry point; and authorize with the same identity that executes.

Related: [[permission-ruleset]] · [[headless-rpc-mode]] · [[ci-agent-integration]] · [[task-owned-subagent]] · [[opencode--headless-rpc-mode|opencode]] · [[opencode--ci-agent-integration|opencode CI]]
