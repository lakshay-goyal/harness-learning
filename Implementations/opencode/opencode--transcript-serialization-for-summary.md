---
type: implementation
harness: opencode
concept: transcript-serialization-for-summary
commit: ecc4916b5a
files: [packages/opencode/src/session/compaction.ts:30, packages/opencode/src/session/compaction.ts:51-85, packages/opencode/src/session/compaction.ts:378-391, packages/opencode/src/session/compaction.ts:425-448, packages/core/src/session/compaction.ts:85-121, packages/core/src/session/compaction.ts:160-174, packages/core/src/session/runner/to-llm-message.ts:147-165]
---
[[transcript-serialization-for-summary]] in [[opencode]].

## Mechanism

### Legacy runtime
- `serialize` (`packages/opencode/src/session/compaction.ts:54-85`): `[User]: …` + `[Attached mime: file]`; `[Assistant]:`, `[Assistant reasoning]:`, `[Assistant tool call]: name(json)`, `[Tool result]: …` (≤ 2000 chars + `\n[truncated]`, `:51-52`), `[Tool error]: …`; pruned outputs as "[Old tool result content cleared]".
- Joined with blank lines inside `<conversation>` (`buildPrompt`); sent as **one user message**, `system: []` (agent prompt `compaction.txt` added by request prep), `tools: {}` (`:425-448`). If a plugin replaced the prompt, the serialized history is appended after "The following is the conversation history:" (`:439`).

### v2 runtime
- Same tags plus `[System update]:`, `[Synthetic context]:`, `[Shell]: cmd\n<output ≤2000 chars>` (`packages/core/src/session/compaction.ts:95-121`); `[Attached mime: name]` for tool media (`:88-93`).
- The kept recent part is also serialized text, re-entering context inside `<conversation-checkpoint><summary>…</summary><recent-context>…</recent-context>` as a user message "historical context, not as new instructions" (`packages/core/src/session/runner/to-llm-message.ts:147-165`).

## Constants
| name | value | path:line |
|---|---|---|
| `TOOL_OUTPUT_MAX_CHARS` | 2_000 + `[truncated]` | `packages/opencode/src/session/compaction.ts:30`; `packages/core/src/session/compaction.ts:14` |

## Evolution
- Pre-2026-08: head sent as real model messages with `stripMedia`/`toolOutputMaxChars` (then JSON-serialized messages) → invalid shapes on orphaned history.
- 2026-04-22 `574b2c2170` 2000-char tool output cap in summarizer input.
- 2026-05-12 `ca28dd02ec` stop serializing the retained tail into the summary prompt.
- 2026-08-06 `b7f9363393` (#40800) "serialize orphaned compaction history" — flat tagged text.

## Quirks / drift
- Tool-call arguments are serialized in full JSON (no cap), so a huge `write` payload still enters the summarizer input (inference from `:71`).

## Failures
[[summarizer-continues-conversation]] · [[summarization-request-overflows]] · [[compaction-request-shape-mismatch]]

Contrast: [[pi--transcript-serialization-for-summary|pi]] has the same `<conversation>` + 2000-char design; opencode adopted it later (2026-04 → 2026-08) and in v2 also serializes the kept tail so no provider-native message crosses the compaction boundary.
