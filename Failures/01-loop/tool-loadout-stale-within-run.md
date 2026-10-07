---
type: failure
concepts: [turn-lifecycle-hooks, transcript-carried-system-prompt]
harnesses: [pi, opencode]
---
**Symptom** — Extension tool changes made during a run were not applied to the next request in the same run, and a forced/run system prompt was dropped when tools were refreshed (#6162).

**Root cause** — Tool loadout and system-prompt options were computed once per user prompt (at prompt start), not per request; the refresh path rebuilt options from the base prompt.

**Fix · [[pi]]** — `e547bb9f4` 2026-06-30 "refresh session state before next turn" + `fd6659dd5` same day "preserve run prompt during tool refresh": `prepareNextTurnWithContext` recomputes prompt options from `_runSystemPromptOptions ?? _baseSystemPromptOptions` with current active tools and emits a sections/tools delta system message before the next turn (`packages/coding-agent/src/core/agent-session.ts:898-932`).

**Fix · [[opencode]]** — avoided by design rather than by a fix commit. Legacy resolves tools inside the step loop on every iteration (`packages/opencode/src/session/prompt.ts:1088`, `:1226` `SessionTools.resolve`). v2 materializes the registry per step (`packages/core/src/session/runner/llm.ts:203`); scoped registrations stack per name and closing a scope removes exactly that registration (`packages/core/src/tool/registry.ts:84-102`); each advertised definition carries an `identity` token and settlement rejects a call whose current registration differs with `Stale tool call: <name>` (`packages/core/src/tool/registry.ts:50-61`, `:105-120`). No dated regression found.

**Lesson** — Recompute loadout and prompt per request inside the loop's turn hook, and carry run-scoped overrides through refreshes.

Related: [[turn-lifecycle-hooks]] · [[transcript-carried-system-prompt]] · [[dynamic-tool-guidelines]] · [[system-prompt-override]] · [[deferred-tools-lost-on-resume]] · [[pi--turn-lifecycle-hooks|pi]]
