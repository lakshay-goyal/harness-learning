---
type: failure
concepts: [guideline-softening, env-vars-as-context, dynamic-tool-guidelines, auxiliary-model-calls]
harnesses: [pi, opencode]
---
**Symptom** — After adding "Inspect PI_* environment variables for current model and session details." to the always-on rules, models ran unnecessary inspection commands (#7128; exact commands not in repo — e.g. `env | grep PI_` is illustrative, unverified).

**Root cause** — An imperative sentence in an always-present rule reads as an instruction to perform every turn, not as availability information.

**Fix · [[pi]]**
- `bb3d7d399` 2026-07-22 (#6967): env vars + imperative guideline introduced.
- `4e64de695` 2026-08-06 (#7128) "soften PI environment guideline": → "You can inspect PI_* environment variables for current model and session details." — CHANGELOG `:759` "Softened … in an attempt to reduce unnecessary inspection commands" (HEAD `packages/coding-agent/src/core/tools/bash.ts:47`, `powershell.ts:20`). Commit body: "Attempt to address #7128 without closing the issue." — effect not measured (unverified).

**Lesson** — Phrase always-on capability hints descriptively ("You can…"); imperative mood in a standing rule becomes an action every turn.

Related: [[guideline-softening]] · [[env-vars-as-context]] · [[dynamic-tool-guidelines]] · [[pi--guideline-softening|pi]]

**Fix · [[opencode]]** — brevity variant: anthropic.txt "You MUST answer concisely with fewer than 4 lines" plus one-word answer examples (`dac1506680` 2025-08-11) produced over-terse answers for complex work; `5a507023a6` 2025-09-30 → "matching the level of detail … with the level of complexity of the user's query"; examples removed `795b845782` 2025-10-25. The fallback prompt still says "fewer than 4 lines" (`packages/opencode/src/session/prompt/default.txt:84`). See [[opencode--guideline-softening]].
- Title-generator variant: "Use -ing verbs for actions (Debugging, Implementing, Analyzing)" (added `e9826e8a22` 2025-08-31) made nearly every session title start "Analyzing …". `fe57d7bb38` 2026-01-07 "avoid repetative 'Analyzing ...' titles" deleted the rule, added "Vary your phrasing - avoid repetitive patterns like always starting with \"Analyzing\"" and "Never include tool names in the title", and set title temperature 0.5 (HEAD `packages/opencode/src/agent/prompt/title.txt`; `packages/opencode/src/agent/agent.ts:240`). Style examples in a rule list act as templates. See [[opencode--auxiliary-model-calls]].
