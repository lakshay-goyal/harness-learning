---
type: failure
concepts: [guideline-softening, env-vars-as-context, dynamic-tool-guidelines]
harnesses: [pi]
---
**Symptom** — After adding "Inspect PI_* environment variables for current model and session details." to the always-on rules, models ran unnecessary inspection commands (#7128; exact commands not in repo — e.g. `env | grep PI_` is illustrative, unverified).

**Root cause** — An imperative sentence in an always-present rule reads as an instruction to perform every turn, not as availability information.

**Fix · [[pi]]**
- `bb3d7d399` 2026-07-22 (#6967): env vars + imperative guideline introduced.
- `4e64de695` 2026-08-06 (#7128) "soften PI environment guideline": → "You can inspect PI_* environment variables for current model and session details." — CHANGELOG `:759` "Softened … in an attempt to reduce unnecessary inspection commands" (HEAD `packages/coding-agent/src/core/tools/bash.ts:47`, `powershell.ts:20`). Commit body: "Attempt to address #7128 without closing the issue." — effect not measured (unverified).

**Lesson** — Phrase always-on capability hints descriptively ("You can…"); imperative mood in a standing rule becomes an action every turn.

Related: [[guideline-softening]] · [[env-vars-as-context]] · [[dynamic-tool-guidelines]] · [[pi--guideline-softening|pi]]
