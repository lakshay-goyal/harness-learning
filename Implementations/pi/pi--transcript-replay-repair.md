---
type: implementation
harness: pi
concept: transcript-replay-repair
commit: b30a6dd77
files: [packages/ai/src/api/transform-messages.ts:71, packages/ai/src/api/transform-messages.ts:158, packages/ai/src/api/transform-messages.ts:167, packages/ai/src/api/transform-messages.ts:195, packages/ai/src/api/transform-messages.ts:213, packages/ai/src/api/transform-messages.ts:231, packages/ai/src/api/anthropic-messages.ts:1319, packages/coding-agent/src/core/agent-session.ts:793, packages/durable/src/harness/context.ts:9]
---
[[transcript-replay-repair]] in [[pi]].

## Mechanism
- **One choke point at the provider boundary**: pass 2 of `transformMessages` (packages/ai/src/api/transform-messages.ts:158-232), run by every adapter before conversion. Raw history (JSONL) is never rewritten; repair is per request.
- **Failed/aborted turns skipped entirely**: assistant messages with `stopReason` `error`/`aborted` dropped — partial reasoning without its following item / incomplete tool calls break e.g. OpenAI "reasoning without following item" (:195-203; 2d27a2c72 2026-01-19 #838, deleted `strictResponsesPairing` compat). Predecessors: `76312ea7e` 2025-12-10 Mistral empty aborted assistants (#165); `fbb74bb29` 2026-01-16 empty error assistants filtered centrally; `387cc97ba` 2025-11-18 unsigned aborted thinking → text (superseded) → [[failed-turns-replayed]], [[aborted-reasoning-signature-invalid]].
- Consequence: "aborted messages may be appended to context and continued" (packages/ai/README.md:1162-1185) works because the partial is *not* replayed; persisted in session file only → [[partial-message-persistence]].
- **Orphaned tool calls** (no result before next assistant/user, or at end of list) get synthetic `toolResult{text:"No result provided", isError:true}` (:167-186, 222-232; helper extracted `b92062211` 2026-04-15). History: v1 `51f5448a5` 2025-10-01 *deleted* calls lacking results → invalidated thinking signatures covering them; `fb1fdb600` 2025-12-20 keep calls + synthesize results; `a23fab469` 2026-04-22 trailing case (#3555); `0f3a0f78b` 2026-01-17 don't track calls from errored messages (Codex converter dropped those calls → orphan *results*, #812) → [[orphaned-tool-calls-and-results]].
- Synthetic result timestamps `Date.now()` (:177) — request-time only.
- **Deferred system messages**: mid-transcript system messages that fall between a tool call and its results are held back and emitted after the results so call/result adjacency holds (:163-166, 216-221; 9e05370b2 2026-09-16). Anthropic adapter additionally holds `role:"system"` updates until right before the next assistant message (an update placed before a user message lands after it) (packages/ai/src/api/anthropic-messages.ts:1319-1328, 1333-1354).
- Pass 0 null-content normalization (`content == null → []`, :71-73; 8c0ccd14b 2026-07-06 #6343) also applied in agent loop + session load → [[missing-optional-fields-crash-replay]].
- Adapter-level complements: empty assistant skipped (Completions openai-completions.ts:1393-1404; Bedrock bedrock-converse-stream.ts:1010-1014); consecutive toolResults merged into one user message (Anthropic :1462-1478, Bedrock :954-966, Google google-shared.ts:320-330, Completions :1407-1432); `requiresAssistantAfterToolResult` synthetic bridge (openai-completions.ts:1239-1246).
- **Coding-agent side**: canonical projection rebuilt per request from session log (`prepareRequest` swaps `context.messages` for `buildSessionProjection().messages`, packages/coding-agent/src/core/agent-session.ts:787-844; 466db0fec) — failed retry/overflow attempts hidden via `context_edit` null while raw history kept (agent-session.ts:1236-1252, 3785-3786) → [[context-projection]], [[context-edit-overlay]], [[abandoned-attempts-left-in-context]].
- Out-of-band messages (extension `triggerTurn:false`, user `!` bash) buffered to turn end to keep adjacency (240eb29c4 #8537) → [[out-of-band-message-deferral]]. Session switch mid-turn aborts+persists first (cefa40ed8 #7022) → [[session-switch-leaves-dangling-tool-calls]].
- Abort during tool preflight: never-prepared calls get no result → synthesized "No result provided" at next request (packages/agent/src/agent-loop.ts:619-628).

## Durable variant (packages/durable)
- Context derived from immutable entries (packages/durable/src/harness/context.ts): active range from newest `head` marker; edits `omit`/`replace`; **tool results moved to directly follow their call**; missing results synthesized as `"Tool result unavailable: history ends before this call completed."` (context.ts:10); assistant messages with `aborted`/`error`/`deferred` excluded (context.ts:9); system message preceded only by user messages moved to the front (`leadWithSystem`, context.ts:171-176; 92216fa15 2026-10-06 — cache fix) (spec.md:267-290).
- Leftover committed partial converted into an aborted `pi.assistant` entry before the next request (spec.md:3522-3531).

## Constants
| name | value | path:line |
|---|---|---|
| orphan result text | `No result provided` (isError) | packages/ai/src/api/transform-messages.ts:175 |
| durable orphan text | `Tool result unavailable: history ends before this call completed.` | packages/durable/src/harness/context.ts:10 |

## Evolution
- 2025-10-01 `51f5448a5` remove calls without results (broke signatures).
- 2025-11-18 `387cc97ba` aborted unsigned thinking → text.
- 2025-12-10 `76312ea7e` Mistral: skip empty aborted assistants (#165).
- 2025-12-20 `fb1fdb600` synthesize results instead of deleting calls.
- 2026-01-16 `fbb74bb29` filter empty error assistants; 2026-01-17 `0f3a0f78b` don't track errored-message calls (#812); 2026-01-19 `2d27a2c72` skip all errored/aborted turns (#838).
- 2026-04-15 `b92062211` helper; 2026-04-22 `a23fab469` trailing orphans (#3555).
- 2026-07-06 `8c0ccd14b` null content.
- 2026-09-16 `9e05370b2` system messages held back for adjacency.
- 2026-09-21 `466db0fec` session projections authoritative.
- 2026-10-06 `92216fa15` durable leading-system normalization.

## Evidence commits
51f5448a5 · 387cc97ba · 76312ea7e · fb1fdb600 · fbb74bb29 · 0f3a0f78b · 2d27a2c72 · b92062211 · a23fab469 · 8c0ccd14b · 9e05370b2 · 466db0fec · 240eb29c4 · cefa40ed8 · 92216fa15

## Quirks
- A toolResult whose assistant was skipped (error/aborted) still passes through (transform-messages.ts:213-215); safety relies on the loop never executing tools of aborted turns — `0f3a0f78b` handled this historically (unverified at HEAD).
- Two "repair" layers (pi-ai pass + adapter converters) once diverged (#812) — lesson baked into keeping repair central.

## Failures
- [[orphaned-tool-calls-and-results]] · [[failed-turns-replayed]] · [[aborted-reasoning-signature-invalid]] · [[missing-optional-fields-crash-replay]] · [[abandoned-attempts-left-in-context]] · [[session-switch-leaves-dangling-tool-calls]]
