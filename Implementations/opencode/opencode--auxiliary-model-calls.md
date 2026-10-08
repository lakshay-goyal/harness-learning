---
type: implementation
harness: opencode
concept: auxiliary-model-calls
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:193-253, packages/opencode/src/session/prompt.ts:1132-1139, packages/opencode/src/agent/agent.ts:234-262, packages/opencode/src/agent/prompt/title.txt:1-44, packages/opencode/src/agent/prompt/summary.txt:1-11, packages/opencode/src/session/summary.ts:102-127, packages/opencode/src/provider/provider.ts:1961-2028, packages/opencode/src/session/llm/request.ts:80-91]
---
[[auxiliary-model-calls]] in [[opencode]].

## Mechanism (legacy runtime)

### Session title (live)
- `SessionPrompt.ensureTitle` runs only for root sessions with the default title and **exactly one** non-synthetic user message; forked at step 1 into the layer scope (not the run fiber, so user abort does not cancel it) (`packages/opencode/src/session/prompt.ts:193-206,1132-1139`).
- Model: title agent's model → provider **small model** (`getSmallModel`: `small_model` config → plugin hook → family priority `gemini-flash`, `gpt-nano`, `claude-haiku`) → the user's model (`prompt.ts:216-221`; `packages/opencode/src/provider/provider.ts:1961-2028`).
- Request: hidden `title` agent, `temperature: 0.5`, permission `"*": "deny"`, `tools: {}`, `small: true`, `retries: 2`, user preamble "Generate a title for this conversation:\n" + first-turn history (or subtask prompts) (`prompt.ts:222-236`; `packages/opencode/src/agent/agent.ts:234-249`).
- `small: true` → user's thinking variant skipped, `smallOptions` (lowest effort / thinking off) used (`packages/opencode/src/session/llm/request.ts:80-91`).
- Output post-processing: strip `<think>…</think>`, first non-empty line, cap 100 chars (`97 + "..."`) (`prompt.ts:243-249`).
- Prompt `title.txt`: XML `<task>/<rules>/<examples>`, "≤50 characters", "NEVER respond to questions", "DO NOT SAY YOU CANNOT GENERATE A TITLE OR COMPLAIN ABOUT THE INPUT", "Never include tool names", "Vary your phrasing", same language as the user (`packages/opencode/src/agent/prompt/title.txt:1-44`).

### Turn summary (dead)
- `summary` agent ("Write like a pull request description … 2-3 sentences … preserve that exact question") still defined (`packages/opencode/src/agent/agent.ts:251-262`; `packages/opencode/src/agent/prompt/summary.txt:1-11`; inlined `packages/core/src/plugin/agent.ts:84`) but its caller was removed: `SessionSummary.summarize` now computes file diffs only (`packages/opencode/src/session/summary.ts:102-127`).

### v2 runtime
- Not implemented: "[ ] Update title, summaries, compaction state, and cleanup in bounded background work" (`packages/core/src/session/runner/llm.ts:83`).

## Constants
| name | value | path:line |
|---|---|---|
| title temperature | 0.5 | `packages/opencode/src/agent/agent.ts:240` |
| title SDK retries | 2 | `packages/opencode/src/session/prompt.ts:234` |
| title cap (code) | 100 chars | `packages/opencode/src/session/prompt.ts:249` |
| title cap (prompt) | ≤50 chars | `packages/opencode/src/agent/prompt/title.txt:10` |

## Evolution (title prompt: 24-commit history)
- 2025-06-21 `137e964131` "EXACTLY one line"; 2025-07-13 `bb28b70700` "NEVER reply to the user's message"; 2025-07-16 `70229b150c` "needs to change due to small models"; 2025-07-16 `a4664e2344` title uses the same options as its model; 2025-07-26 `ad8a4bc744` strip `<think>`.
- 2025-08-06 `49aa48ce58` no regeneration on auto-compact; 2025-08-31 `e9826e8a22` "You are a title generator. You output ONLY a thread title."
- 2025-09-06 `564143071e` shell-first sessions got no title; 2025-11-28 `17e8322c29` "ignore tool execution messages" reverted same day `0e280017e6`.
- 2025-11-07 `dabb1aa719` refusal rule; 2025-11-23 `7413c2715c` "Always output something meaningful".
- 2026-01-04 `554572bc39` main-model variant leaked into small model; 2026-01-06 `5db78f20e9` subtask-only first message; 2026-01-07 `fe57d7bb38` "Analyzing …" monotony → rules removed, temperature 0.5; 2026-01-21 `fd77d31b49` user's language.
- 2026-01-12 `71a7ad1a4e` per-message summary caller removed; 2026-02-16 `45fa5e7199` per-message title LLM calls removed.
- 2025-11-19 `3b72857124`, 2026-02-02 `bd9d7b3221` OpenAI reasoning effort low/minimal for titles.

## Quirks / drift
- Side calls run outside the cancel scope and outside the step budget.
- Small models misbehave in every way a summarizer does: answer, refuse, think aloud, echo tools, drift language.

Failures: [[title-from-tool-narration]] · [[imperative-guideline-over-compliance]] · [[small-model-format-noncompliance]] · [[summary-drops-pending-user-request]] · [[summarizer-continues-conversation]] · [[summarizer-refusal]] · [[side-call-language-drift]].

Contrast: pi has summarization side calls ([[split-turn-summary]], [[branch-summary]]) but no title generation; opencode routes titles to a separate small model with heavy post-processing.
