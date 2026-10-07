---
type: failure
concepts: [transcript-replay-repair]
harnesses: [pi]
---
**Symptom** — After interruptions or errors, providers rejected the replayed history:
- Anthropic and Claude broke the tool_use→tool_result chain.
- OpenAI: `No tool call found for function call output with call_id …`.
- Codex errored on synthetic results whose calls the converter had dropped.
- Direct replay of a transcript ending in tool calls was rejected.

**Root cause** — Interrupted flows leave tool calls without results, or leave results whose calls were dropped by another layer. Two layers each "fixing" the transcript independently diverged.

**Fix · [[pi]]**
1. `51f5448a5` (2025-10-01), v1: deleted tool calls that had no results. This invalidated thinking signatures covering those calls, and broke OpenAI's reasoning→item pairing.
2. `fb1fdb600` (2025-12-20): keep the calls and insert a synthetic `toolResult{isError:true, text:"No result provided"}` (HEAD `packages/ai/src/api/transform-messages.ts:167-186`).
3. `0f3a0f78b` (2026-01-17), #812: don't track calls from `stopReason:"error"` messages, which produced orphan *results* when Codex dropped those calls.
4. `b92062211` (2026-04-15): extracted the synthetic-result helper.
5. `a23fab469` (2026-04-22), #3555: also synthesize results for *trailing* unresolved calls (`:222-232`), CHANGELOG `packages/ai/CHANGELOG.md:1224`.

Mid-transcript system messages between a call and its results are held back until after the results (`:163-166,216-221`; 9e05370b2).

Open quirk (unverified): a toolResult whose assistant message was skipped still passes through (`:213-215`).

Durable variant: "Tool result unavailable: history ends before this call completed." (`packages/durable/src/harness/context.ts:10`).

**Lesson** — Run one central transcript-repair pass before every request. Repair by *adding* the missing counterpart, never by mutating signed content. Run the invariant at end of sequence, not only at transitions.

Related: [[transcript-replay-repair]] · [[failed-turns-replayed]] · [[session-switch-leaves-dangling-tool-calls]] · [[pi--transcript-replay-repair|pi]]
