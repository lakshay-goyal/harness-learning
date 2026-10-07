---
type: failure
concepts: [turn-lifecycle-hooks, transcript-carried-system-prompt]
harnesses: [pi, opencode, codex]
---
**Symptom** — Extension tool changes made during a run were not applied to the next request in the same run, and a forced/run system prompt was dropped when tools were refreshed (#6162).

**Root cause** — Tool loadout and system-prompt options were computed once per user prompt (at prompt start), not per request; the refresh path rebuilt options from the base prompt.

**Fix · [[pi]]** — `e547bb9f4` 2026-06-30 "refresh session state before next turn" + `fd6659dd5` same day "preserve run prompt during tool refresh": `prepareNextTurnWithContext` recomputes prompt options from `_runSystemPromptOptions ?? _baseSystemPromptOptions` with current active tools and emits a sections/tools delta system message before the next turn (`packages/coding-agent/src/core/agent-session.ts:898-932`).

**Fix · [[codex]]**
- Structural: a fresh `StepContext` (tools, MCP binding, settings, AGENTS.md) is captured before every sampling request and re-captured when new input arrived (`codex-rs/core/src/session/turn.rs:460-495`) → [[mid-turn-settings-switch]].
- Symptom (delta variant): with `IncrementalTools` on Responses Lite (only added/changed tool definitions appended to history, `6326163b9a` 2026-10-03), a partial namespace re-declaration was ambiguous (are omitted tools gone?) and namespace removals listed every member. `402f5b6fdf` / `93f8e79fd2` 2026-10-05: explicit text "This is an incremental namespace update. Previously declared tools remain available for direct calls unless explicitly marked unavailable…" / "The following tools are no longer available. Do not call them:" / separate namespace-removal notice (`codex-rs/core/src/context/world_state/top_level_tools.rs:21-23`).
**Fix · [[opencode]]** — avoided by design rather than by a fix commit. Legacy resolves tools inside the step loop on every iteration (`packages/opencode/src/session/prompt.ts:1088`, `:1226` `SessionTools.resolve`). v2 materializes the registry per step (`packages/core/src/session/runner/llm.ts:203`); scoped registrations stack per name and closing a scope removes exactly that registration (`packages/core/src/tool/registry.ts:84-102`); each advertised definition carries an `identity` token and settlement rejects a call whose current registration differs with `Stale tool call: <name>` (`packages/core/src/tool/registry.ts:50-61`, `:105-120`). No dated regression found.

**Lesson** — Recompute loadout and prompt per request inside the loop, carry run-scoped overrides through refreshes, and when tool changes are sent as deltas state the merge semantics to the model.

Related: [[turn-lifecycle-hooks]] · [[transcript-carried-system-prompt]] · [[dynamic-tool-guidelines]] · [[system-prompt-override]] · [[deferred-tools-lost-on-resume]] · [[pi--turn-lifecycle-hooks|pi]] · [[world-state-diff-injection]] · [[codex--turn-lifecycle-hooks|codex]]
