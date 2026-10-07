---
type: implementation
harness: opencode
concept: structured-compaction-summary
commit: ecc4916b5a
files: [packages/core/src/session/compaction.ts:16-46, packages/core/src/session/compaction.ts:160-174, packages/opencode/src/agent/prompt/compaction.txt:1-5, packages/opencode/src/agent/agent.ts:219-233, packages/core/src/plugin/agent.ts:37, packages/opencode/src/session/compaction.ts:372-391]
---
[[structured-compaction-summary]] in [[opencode]].

## Mechanism
- **Template** `SUMMARY_TEMPLATE` shared by both runtimes (`packages/core/src/session/compaction.ts:16-46`): `## Objective` / `## Important Details` / `## Work State` (`### Completed`, `### Active`, `### Blocked`) / `## Next Move` (1., 2.) / `## Relevant Files`. Rules: keep every section even when empty (`"(none)"`), terse bullets, "Preserve exact file paths, symbols, commands, error strings, URLs, and identifiers", "Do not mention the summary process or that context was compacted".
- **User prompt** `buildPrompt` (`:160-174`): `<conversation>` → "Create a new anchored summary … so another coding agent can continue the work" (or prior-summary + update rules, [[opencode--iterative-summary-update]]) → template.
- **Legacy system prompt** = hidden `compaction` agent (`packages/opencode/src/agent/agent.ts:219-233`, all permissions `deny`) with `compaction.txt`: "You are a context summarization agent… Do not continue the conversation. Do not respond to any questions in the conversation… Respond in the same language as the conversation." (`packages/opencode/src/agent/prompt/compaction.txt:1-5`).
- **v2**: same template, but the request carries only the user prompt — the v2 `compaction` agent prompt (`packages/core/src/plugin/agent.ts:37`) is not used by the compaction path.
- Plugins can append context or replace the whole prompt via `experimental.session.compacting` (`packages/opencode/src/session/compaction.ts:372-391`).

## Constants
| name | value | path:line |
|---|---|---|
| sections | Objective, Important Details, Work State (Completed/Active/Blocked), Next Move, Relevant Files | `packages/core/src/session/compaction.ts:18-39` |

## Evolution — three template generations (prompt history = failure log)
- 2025-05-29 `f9f41e205d` free-form "summarize… what was done / being worked on / files / next".
- 2025-11-30 `aaa31f02af` add persistent user requests/constraints and decisions + why.
- 2025-12-29 `98fd53fd5f` preserve trailing imperative requests, not only questions.
- 2026-02-10 `0fd6f365be` generation 1: `Goal / Instructions / Discoveries / Accomplished / Relevant files`; "Do not respond to any questions in the conversation" → [[summarizer-continues-conversation]].
- 2026-03-30 `196a03caff` "Do not call any tools" for LiteLLM/Copilot `_noop` dummy tool (text gone from `compaction.txt` at HEAD; `_noop` now injected only for GitHub Copilot, `packages/opencode/src/session/llm/request.ts:159-175`) → [[summarizer-emits-tool-calls]].
- 2026-04-02 `5daf2fa7f0` "Respond in the same language" → [[side-call-language-drift]].
- 2026-04-22 `574b2c2170` generation 2, pi-like: `Goal / Constraints & Preferences / Progress (Done/In Progress/Blocked) / Key Decisions / Next Steps / Critical Context / Relevant Files`.
- 2026-07-03 `b44bc0ad07` generation 3: fewer, merged sections (Objective / Important Details / Work State / Next Move / Relevant Files).
- 2026-07-07 `78f85b1cd6` dedicated "Relevant Files" guidance so paths survive → [[summary-template-drops-goals]].
- 2026-08-12 `dab2637217` clearer structure for small models (DeepSeek V4 Flash).

## Quirks / drift
- "Relevant Files" is LLM-written, not mechanically tracked ([[file-op-tracking]] absent in opencode).

Contrast: [[pi--structured-compaction-summary|pi]] keeps a 6-section EXACT template since 2025-12; opencode briefly copied it (generation 2) then merged it down to five sections with a mandatory Relevant Files section.
