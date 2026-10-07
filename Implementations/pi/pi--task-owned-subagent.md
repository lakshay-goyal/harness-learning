---
type: implementation
harness: pi
concept: task-owned-subagent
commit: b30a6dd77
files: [packages/durable/docs/spec.md:2089, packages/durable/README.md:414, packages/coding-agent/src/experimental/durable/subagent.ts:24, packages/coding-agent/src/experimental/vacation/vacation.ts:65, packages/durable/test/examples/22-subagent-foreground.ts:1, packages/durable/test/examples/23-subagent-background.ts:1, packages/chord/src/services/provider.ts:183]
---
[[task-owned-subagent]] in [[pi]] — exists only in `packages/durable` (experimental, "API changes without notice", `packages/durable/README.md:3`); stable pi has none ([[no-subagents-core]]).

## Mechanism
- **No built-in subagent tool in the kernel**; subagents are composed from primitives: task-owned conversations, `api.conversation(id).submit()`, `requestId` idempotence (`packages/durable/README.md:414-459`; examples `test/examples/22-subagent-foreground.ts`, `23-subagent-background.ts`). Plan mode likewise only an example (`27-plan-mode.ts`).
- **Ownership tree** (`packages/durable/docs/spec.md:2089-2176`, "5.5 Structured concurrency"): every task names its owner at creation — its conversation (`{kind:"conversation"}`) or a task (`{kind:"task", taskId}`); every conversation is ownerless or task-owned; owner edges immutable; only owned conversations cross conversation boundaries.
  - **Abort flows down**: owner's abort mark or held non-`completed` outcome marks its ordinary owned work; background tasks are boundaries unless aborted directly / `Conversation.abort(ctx,{background:true})`.
  - **Owned work does not outlive its owner's finish**: a terminal commit while owned work is live is stored as `{status:"completing", outcome}`; scheduler writes final `terminal` once owned work drained (re-evaluated after every commit and at open); held outcome is final; writes split at the hold (tool result entry lands at hold; terminal state/waiters deferred) (`spec.md:2110-2140`).
  - New owned work needs a live owner (reject owner that is completing/terminal/abort-marked); conversations stay usable after their owner finished — e.g. to **interrogate a finished subagent** (`spec.md:2142-2148`).
  - **Waiting**: `{status:"waiting", checkpoint, on, policy}`; `failFast` gives every other live task in `on` an abort mark on first non-completed outcome; `allSettled` required for tasks not owned (`spec.md:2150-2160`).
  - **Abort order bottom-up**: abort handler runs only once owned work no longer live, so it sees final outcomes; abort handlers can't create owned children (compensate inline or create background conversation-owned tasks) (`spec.md:2162-2176`; Checkout/Payment example).
- **Foreground subagent tool** (README pattern = experimental TUI tool `packages/coding-agent/src/experimental/durable/subagent.ts:24-55`, installed `durable/runtime.ts:134`): description "Delegate a self-contained task to a subagent with the same tools and get its answer back. Give it everything it needs to know; it does not see this conversation." (`:29-30`); `replay:"safe"` because a rerun finds the child via `tx.scanConversations({ownerTaskId: api.taskId})` (`:33-37`); else `tx.createConversation({ownership:{kind:"task", taskId}})` — starts as copy of parent's agent (model, extensions, tools, cwd) — and `configure(..., {extensions:{remove:[Subagent]}})` so it **cannot recurse** (`:38-40`; README variant also downgrades `model: haiku`); `api.details({conversationId})` lets UI attach to the child (`:43`); submit `{type:"input", content: task, requestId:"subagent:<taskId>"}` and `wait` (`:45-46`); non-`done` → throw (→ error result); answer text read from the committed `AssistantEntry` (`:13-17`, `:47-51`).
- Semantics (`packages/durable/README.md:452-454`): aborting the call aborts the child — so does `execute()` throwing or a crash interrupting a non-replay-safe call; **parent idle only once child idle**; `{background:true}` task = boundary (survives parent abort, doesn't keep parent busy).
- **Background subagent that reports back** (`23-subagent-background.ts`; vacation demo `packages/coding-agent/src/experimental/vacation/vacation.ts:60-127`, `395315f48`): `research` tool creates a background conversation-owned task + child conversation owned by that task, returns immediately "Research started in the background." (`vacation.ts:105-127`); task `deliver` phase submits to child with `requestId: research:<task.id>` (`:79`), checkpoints `{phase:"report", report}`; `report` phase submits `[research report] …` to main as `whenBusy:"followUp"` with `requestId: research-report:<task.id>` (`:94-98`) → [[pi--follow-up-queue|follow-up-queue]]. Example 23: one `subagent` tool spawns, messages (steer/follow-up), stops, lists named subagents; name→conversation map in a document; each child owned by a background anchor task so parent Esc/idle never reach it; replies delivered by background reporter tasks; survives Harness close/reopen.
- **Accounting footgun** (contract, not guarded): a tool running an owned conversation must not report the child's usage — child's `pi.usage` already counts it; subtree sums would double count (`spec.md:4634-4636`). Owned foreground work extends the calling tool's hold, and with it the run (`spec.md:4637-4640`).
- **Process model**: experimental session worker stays alive while any task (incl. background) is live (`packages/coding-agent/src/experimental/session-worker.ts:591`); services cover only the root conversation — "Exposing subagent conversations needs keyed service instances per conversation" (`src/experimental/services/README.md:44`) → Chord keyed services addressed `(serviceId, key, generation)`, monotonic generation per key, stale callers get `service_stale_instance` (`packages/chord/src/services/provider.ts:183-212,377-410`) → [[client-server-session-split]].

## Constants
| name | value | path:line |
|---|---|---|
| child `requestId` | `subagent:${taskId}` | `experimental/durable/subagent.ts:45` |
| research `requestId`s | `research:${id}`, `research-report:${id}` | `experimental/vacation/vacation.ts:79,98` |
| tool replay policy | `"safe"` (foreground) | `subagent.ts:33` |
| join policies | `failFast`, `allSettled` | `spec.md:1616` |

## Evolution
- 2026-09-29 `9f1013506` spec "ownership and subagents" (Package 18) → `2532a0bef` conversation abort, ownership cascades, subagent handles (+661-line `harness-ownership.test.ts`; examples 22/23) → `1b347794e` ownership cancellation races (+209 test lines).
- 2026-09-30 `03180653c` structured concurrency: task ownership, waiting, completing holds, bottom-up abort (Package 19); `37c9d0d20` harden persistent subagent example (no wait, not rerun after crash, report each answer once).
- 2026-10-01 `5609b0d6c` experimental TUI coding agent on pi-durable ships `subagent` tool; `395315f48` vacation planner demo.

## Evidence commits
`9f1013506` `2532a0bef` `1b347794e` `03180653c` `37c9d0d20` `5609b0d6c` `395315f48`

## Quirks
- "No subagents" is a coding-agent **product** stance, not a runtime one: the durable kernel's ownership model is first-class (`08-absences`; `packages/coding-agent/README.md:19` vs `packages/durable/README.md:414-426`).
- Replay safety is per-operation: foreground spawn+wait is `safe` (idempotent via ownership index + requestId); background **stop** is not (a repeated stop could stop newer work) (`37c9d0d20`).
- Built-in durable coding tools default `replay:"unsafe"` (none declare `replay`, `harness/tool.ts:88`) → [[crash-safe-tool-replay]].

## Failures
[[ownership-cancellation-races]] · [[non-idempotent-tool-replayed-after-crash]]
