---
type: implementation
harness: opencode
concept: turn-loop
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:1081-1341, packages/opencode/src/session/message-v2.ts:594-611, packages/opencode/src/session/processor.ts:641-697, packages/opencode/src/session/llm.ts:280-323, packages/core/src/session/runner/llm.ts:392-416, packages/core/src/session/runner/llm.ts:174-355]
---
[[turn-loop]] in [[opencode]].

## Mechanism

### Legacy runtime (`packages/opencode/src/session/prompt.ts`)
- `SessionPrompt.runLoop` (span `SessionPrompt.run`) is a `while (true)` over **SQLite, not memory**: every iteration sets status busy, re-reads history with `MessageV2.filterCompactedEffect(sessionID)`, then `MessageV2.latest(msgs)` picks newest user / newest assistant / newest *finished* assistant / pending `compaction`+`subtask` parts by time, not array position (`packages/opencode/src/session/prompt.ts:1088-1096`; `packages/opencode/src/session/message-v2.ts:590-611`). No in-memory transcript survives between steps; each step is a fresh read + `toModelMessagesEffect`.
- One assistant message **per model step** (not per user turn), all chained to the same `parentID: lastUser.id`.
- Per-iteration order: exit test → `step++` → title side-call forked at step 1 → `tasks.pop()` subtask → compaction task → pre-request overflow check (`compaction.isOverflow`, `prompt.ts:1161-1167`) → agent lookup → step cap (`prompt.ts:1178-1179`, see [[opencode--step-budget-limit|step-budget-limit]]) → reminders → `experimental.chat.messages.transform` hook (`prompt.ts:1255`) → processor.
- **Exit test** (`prompt.ts:1106-1130`): break only if `lastAssistant.finish ∉ {"tool-calls","unknown"}` AND the last assistant has no tool part (excluding `providerExecuted` and cleanup-marked interrupted orphans, `isOrphanedInterruptedTool` `prompt.ts:96-100`) AND `lastAssistant.parentID === lastUser.id`. The parent check is what makes a newly persisted user message force another step ([[opencode--steering-queue|steering]]).
- "`stop` with tool calls" continues (OpenAI-compatible providers send `stop` after tool calls); `unknown` finish continues instead of ending.
- **Processor = exactly one model step**: `SessionProcessor.process` streams `llm.stream(input)` → `Stream.tap(handleEvent)` → `Stream.takeUntil(needsCompaction)` → returns `"compact" | "stop" | "continue"` (`packages/opencode/src/session/processor.ts:30,641-697`). AI SDK `streamText` is called with no `stopWhen`, so one call = one step; the outer loop is the multi-step driver (`packages/opencode/src/session/llm.ts:280`).
- Tools execute **inside** `streamText` (AI SDK owns dispatch; parallel calls as the SDK schedules them); the processor persists `tool-result`/`tool-error` (`packages/opencode/src/session/tools.ts:41-134`).
- Outcome mapping (`prompt.ts:1288-1328`): structured output captured → break; `content-filter` finish → persisted `ContentFilterError` + `Session.Event.Error`, break; `json_schema` requested but text-only → `StructuredOutputError`, break; `stop` → break; `compact` → `compaction.create({auto:true, overflow: !finish})`, continue.
- After the loop: `compaction.prune` forked in the background (`prompt.ts:1338`).

### v2 runtime (`packages/core/src/session/runner/llm.ts`) — "Session Drain"
- Two nested loops in `SessionRunner.run` (`packages/core/src/session/runner/llm.ts:392-416`): inner `while (needsContinuation)` runs one **Provider Turn**; outer loop promotes one queued input and restarts the inner loop with `step = 1`.
- `needsContinuation` = turn issued a complete local tool call and no provider error (`llm.ts:354`), OR a steer is pending (`llm.ts:410`). After the first turn `promotion = "steer"`.
- One explicit `llm.stream(request)` per provider turn (`llm.ts:241`); history reloaded from the SQLite projection every turn via `SessionHistory.entriesForRunner` (`llm.ts:200`) — again no in-memory conversation. Spec: `specs/v2/session.md:50`.
- Each complete local tool call is durably recorded, then started eagerly in a `FiberSet`; all fibers are awaited after stream close (`llm.ts:256-280,305`).
- Before the first turn, tools left `pending|running` by a dead process are failed "Tool execution interrupted" (`llm.ts:118-138,399`).
- Implementation checklist lives in the file header: steps `[x]`, retries + identical-call guard `[ ]` (`llm.ts:43-91`).

## Constants
| name | value | path:line |
|---|---|---|
| finishes that continue | `"tool-calls"`, `"unknown"` | `packages/opencode/src/session/prompt.ts:1113` |
| AI SDK `maxRetries` (main turn) | `input.retries ?? 0` | `packages/opencode/src/session/llm.ts:323` |
| v2 initial step per drain/promotion | `1` | `packages/core/src/session/runner/llm.ts:404,195` |

## Evolution
- 2025-06-14 `fa1266263d` AI SDK `maxSteps: 1000` (SDK-owned multi-step loop) → 2025-08-09 `e2fac991dc` custom `stopWhen` → later one-step `streamText` + harness loop.
- 2026-04-01 `733a3bd031` continue when provider says `stop` but emitted tool calls.
- 2026-05-25 `748fcb7ebd` ignore cleanup-marked interrupted tools in the exit test.
- 2026-06-11 `e2527db3c7` content-filter finish surfaced as visible error.
- 2026-06-03 `76ee87ead8` v2 embedded runtime; 2026-06-20 `4f1a9d7aef` removed the invented `MAX_STEPS = 25` cap.
- 2026-08-21 `57fa34f235` continue on `unknown` finish.

## Quirks / drift
- Continuation is decided from transcript content, not the provider finish reason alone — the lesson of [[stop-reason-mapping-gaps]].
- Legacy re-reads the whole session from SQLite each step: simple crash/resume semantics, O(history) per step.
- v2 has no doom-loop guard yet ([[opencode--repeated-tool-call-detection|repeated-tool-call-detection]]) and no runner-level retry ([[opencode--auto-retry-backoff|auto-retry-backoff]]).
- `step` is per run in legacy; queued messages share the counter. v2 resets it on promotion.

Contrast: [[pi--turn-loop|pi]] keeps an in-memory messages array with inner turn + outer follow-up loop; opencode rebuilds state from the store every step.
