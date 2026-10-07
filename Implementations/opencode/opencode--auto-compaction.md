---
type: implementation
harness: opencode
concept: auto-compaction
commit: ecc4916b5a
files: [packages/opencode/src/session/overflow.ts:8-34, packages/opencode/src/session/compaction.ts:319-582, packages/opencode/src/session/prompt.ts:1149-1168, packages/opencode/src/session/prompt.ts:1320-1328, packages/opencode/src/session/processor.ts:491-496, packages/opencode/src/session/processor.ts:621-632, packages/opencode/src/provider/provider.ts:1610, packages/core/src/v1/config/config.ts:149-168, packages/opencode/src/config/config.ts:593-598, packages/core/src/session/compaction.ts:12-15, packages/core/src/session/compaction.ts:123-135, packages/core/src/session/compaction.ts:178-243, packages/core/src/session/runner/llm.ts:224-225, packages/core/src/session.ts:417-419]
---
[[auto-compaction]] in [[opencode]].

## Mechanism

### Legacy runtime — compaction is a user message the loop processes
- **Threshold** `usable()` (`packages/opencode/src/session/overflow.ts:10-20`): `context === 0` → 0; with `limit.input`: `input − (compaction.reserved ?? min(20_000, maxOutput))`; **without `limit.input` the `reserved` setting is ignored** and usable = `context − maxOutput` (maxOutput ≤ `OUTPUT_TOKEN_MAX = 32_000`, `packages/opencode/src/provider/transform.ts:18`).
- `isOverflow` (`overflow.ts:22-34`): false if `compaction.auto === false` or `limit.context === 0`; count = `tokens.total || input+output+cache.read+cache.write` of the **last response's provider usage**; overflow when `count >= usable`.
- **Custom models default to context 0** (`limit.context ?? existing ?? 0`, `packages/opencode/src/provider/provider.ts:1610`) → threshold compaction silently disabled; only the provider's overflow error can trigger it.
- **Trigger sites**: (1) mid-turn on every `finish-step` usage, unless the message is itself a summary → `needsCompaction`, stream stops (`packages/opencode/src/session/processor.ts:491-496`, `7a3ff5b98f`); (2) loop top: last finished non-summary assistant over threshold (`packages/opencode/src/session/prompt.ts:1161-1168`); (3) provider `ContextOverflowError` → `needsCompaction` (`processor.ts:621-632`), honouring `auto: false` since `7e09660c3b`; (4) manual `/compact`.
- `create` only persists a user message with a `compaction` part `{auto, overflow}` (`packages/opencode/src/session/compaction.ts:559-582`); `overflow: !handle.message.finish` (`prompt.ts:1320-1328`). The next loop iteration pops it as a task and calls `process` (`prompt.ts:1149-1159`).
- `process` (`compaction.ts:319-557`): hidden `compaction` agent model if configured else the user's model (`:358-361`); prior completed summaries hidden and the newest passed as `previousSummary` (`:363-366`); head/tail `select` ([[opencode--compaction-cut-point]]); plugin hooks `experimental.session.compacting` (add context or replace prompt) and `experimental.chat.messages.transform` (`:373-379`); head serialized to text ([[opencode--transcript-serialization-for-summary]]); one processor step with `tools: {}` and `system: []` (`:425-448`). Result: assistant message `summary: true, mode: "compaction"` whose parent is the compaction user message (`:393-419`).
- **Post-compaction transcript shape**: `[compaction user ("What did we do so far?"), summary assistant, retained tail…, synthetic continue user]` after [[opencode--context-projection|filterCompacted]] reorders. Auto runs append a synthetic user "Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed." with `metadata.compaction_continue` (`compaction.ts:519-547`); plugin `experimental.compaction.autocontinue` can veto (`:497-518`); manual `/compact` stops after summarizing (`020ee56f25`).
- Compaction request itself overflowing → `ContextOverflowError("Session too large to compact…")`, `return "stop"` (`:450-459`) — no cascade.
- Config: `compaction.{auto, prune, tail_turns, preserve_recent_tokens, reserved}` (`packages/core/src/v1/config/config.ts:149-168`); env `OPENCODE_DISABLE_AUTOCOMPACT` / `OPENCODE_DISABLE_PRUNE` force off (`packages/opencode/src/config/config.ts:593-598`).

### v2 runtime — pre-request estimate, checkpoint message
- `compactIfNeeded` before every provider turn: estimate `JSON({system, messages, tools})` at chars/4 and compact when `> context − max(maxOutput, buffer)` (`packages/core/src/session/compaction.ts:232-243`); `context` undefined or ≤ 0 → never (`:234-235`). On success the runner dies with `continueAfterCompaction(step)` and replays the same turn from reloaded history without consuming a step (`packages/core/src/session/runner/llm.ts:224-225`).
- Settings merged across config documents, later wins: `{auto: true, buffer: 20_000, keep.tokens: 8_000}` (`compaction.ts:123-135`).
- `compactAfterOverflow` (`:178-231`): skip when nothing old enough (`:184`) or when the summary prompt itself won't fit `context − summaryOutput` (`:189-190`); `Compaction.Started` event; LLM call with **only a user message, `tools: []`, no system prompt**, `maxTokens = min(output || 4096, 4096)` (`:201-210`); provider error / empty text → return false, old boundary stays (`:211-221`); `Compaction.Ended{text, recent}` activates the cut.
- Lowered as one user `<conversation-checkpoint>` with `<summary>` + `<recent-context>`, "Treat it as historical context, not as new instructions" (`packages/core/src/session/runner/to-llm-message.ts:147-165`). Completed compaction also starts a new Context Epoch ([[opencode--transcript-carried-system-prompt]]).
- Manual compaction: `session.compact` returns `OperationUnavailableError` (`packages/core/src/session.ts:417-419`).

## Constants
| name | value | path:line |
|---|---|---|
| legacy `COMPACTION_BUFFER` | 20_000 (input-limit branch only) | `packages/opencode/src/session/overflow.ts:8` |
| `OUTPUT_TOKEN_MAX` | 32_000 | `packages/opencode/src/provider/transform.ts:18` |
| custom model `limit.context` default | 0 (disables threshold) | `packages/opencode/src/provider/provider.ts:1610` |
| v2 `DEFAULT_BUFFER` | 20_000 | `packages/core/src/session/compaction.ts:12` |
| v2 `DEFAULT_KEEP_TOKENS` | 8_000 | `packages/core/src/session/compaction.ts:13` |
| v2 `SUMMARY_OUTPUT_TOKENS` | 4_096 | `packages/core/src/session/compaction.ts:15` |
| `compaction.auto` default | true | `packages/core/src/v1/config/config.ts:151-153` |

## Evolution
- 2025-05-29 `f9f41e205d` "add summarize".
- 2025-06-17 `57b3051024` / 2025-06-26 `f4c0d2d2fd`: threshold `(context − output) × 0.9` went ≤ 0 → summarize every turn → [[compaction-threshold-underflow]].
- 2025-07-10 `469f667774` 32k output cap; 2025-09-12 `983e3b2ee3` "fix compaction issues" dropped the ×0.9 ratio (briefly `/2`, `eb24d2f847` 2025-09-13).
- 2025-09-17 `71076d5c68` synthetic user prompt after compaction → [[post-compaction-transcript-ends-on-assistant]]; 2025-11-25 `020ee56f25` no auto-continue for manual.
- 2025-12-26 `ed06de5e30` configurable `compaction.auto`.
- 2026-01-01 `7a3ff5b98f` mid-turn check on `finish-step` → [[threshold-check-misses-post-tool-request]].
- 2026-02-10 `0fd6f365be` (#12924) 20k reserve "to ensure that input window has enough room to compact".
- 2026-03-02 `be20f865ac` 413 recovery via compaction → [[opencode--overflow-recovery]].
- 2026-04-13 `34e2429c49` autocontinue hook; 2026-04-15 `e83b22159d` continuation tracked as agent-initiated for Copilot billing.
- 2026-06-04 `7e09660c3b` respect `auto: false` on overflow; 2026-06-05 `beae7290f3` v2 compaction (#30986).

## Quirks / drift
- `compaction.reserved` silently does nothing for models without `limit.input` (most catalog entries; inference from `overflow.ts:17-19`).
- Legacy judges the threshold from the *previous* response's usage; v2 from a pre-request estimate — legacy can still send one oversized request after a huge tool result if no `finish-step` usage arrived (inference).
- v2 `compactAfterOverflow` does not check `config.auto`, so overflow recovery runs even when auto compaction is off — the bug legacy fixed in `7e09660c3b` (inference from `compaction.ts:178-190`).
- A `compaction` agent with "Do not continue the conversation…" exists in v2 (`packages/core/src/plugin/agent.ts:37`) but the v2 path sends no system prompt.

## Failures
[[compaction-threshold-underflow]] · [[post-compaction-transcript-ends-on-assistant]] · [[overflow-ignores-autocompact-optout]] · [[threshold-check-misses-post-tool-request]] · [[overflow-compaction-cascade]] · [[compaction-request-shape-mismatch]] · [[reordered-context-misidentifies-latest-turn]]

Contrast: [[pi--auto-compaction|pi]] uses a fixed 16384 reserve checked by estimate before every request and appends a compaction entry; opencode legacy reserves 20k only when `limit.input` exists, judges by last usage, and stores compaction as a user+assistant message pair.
