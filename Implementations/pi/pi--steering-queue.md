---
type: implementation
harness: pi
concept: steering-queue
commit: b30a6dd77
files: [packages/agent/src/agent.ts:143, packages/agent/src/agent.ts:299, packages/agent/src/agent-loop.ts:176, packages/agent/src/agent-loop.ts:295, packages/coding-agent/src/core/agent-session.ts:2012, packages/coding-agent/src/modes/interactive/interactive-mode.ts:4721, packages/durable/src/harness/inbox.ts:16]
---
[[steering-queue]] in [[pi]].

## Mechanism
- Two `PendingMessageQueue`s on `Agent` (`packages/agent/src/agent.ts:143-174`, constructed `:247-248`). Mode `"all"` drains everything; `"one-at-a-time"` drains only the oldest (`peek`/`drain`, `agent.ts:159-169`; `types.ts:49-55`). Default `"one-at-a-time"` for both (`agent.ts:247-248`; settings `settings-manager.ts:853-865`).
- `steer(msg)` — "Queue a message to be injected after the current assistant turn finishes" (`agent.ts:298-301`).
- **Poll points** (`getSteeringMessages`): run start (`agent-loop.ts:175-176`); after `prepareNextTurn` only if prior poll empty — "otherwise one-at-a-time mode would deliver two messages in this turn" (`:201-206`); after each turn's tool batch (`:295`). Contract: "Tool calls from the current assistant message are not skipped" (`types.ts:285-293`); must not throw.
- Injection = normal messages appended before next provider request with `message_start/end` → persisted. Role `user` from AgentSession (`agent-session.ts:2237-2262`) or `custom` via `sendCustomMessage` (`:2305-2312`).
- `Agent.continue()` with assistant tail drains one steering batch (`skipInitialSteeringPoll` so it isn't double-polled, `agent.ts:394-399`, `467-503`), else one follow-up batch, else throws "Cannot continue from message role: assistant" (`:407`). `b050c582a` (#1312).
- `peekQueuedMessages()` steering first (`agent.ts:330-333`); `hasQueuedMessages()` (`:325-327`); `clearAllQueues()`.
- AgentSession: `prompt()` while streaming requires `streamingBehavior: "steer" | "followUp"` else throws "Agent is already processing. Specify streamingBehavior…" (`agent-session.ts:2012-2025`). Queued text runs extension `input` handlers + skill/template expansion (`:2172-2199`, `faa9863cb`); extension slash commands cannot be queued (`:2267-2277`).
- UI mirrors: string arrays `_steeringMessages`/`_followUpMessages` (`agent-session.ts:393-396`), removed on user `message_start` by **text match**, steering checked first (`:1118-1136`); `queue_update{steering,followUp}` event (`:196-237`).
- **Esc while streaming**: `restoreQueuedMessagesToEditor({abort:true})` → `clearAllQueues()` → queued text back into editor → `session.abort()` (`modes/interactive/interactive-mode.ts:3050-3052,4721-4740`). Docs: "Aborting stops the current run and returns queued messages to the editor" (`docs/how-pi-works.md:13`).
- Side phases (compaction, branch summary, tree navigation) own their own input queue and drain after: prompts rejected during manual compaction (`agent-session.ts:1985-1989`, `8eda4f5b2`), queued ones sent after (`3852cb2b8`); branch-summary queue (`5c61d6bc9` #1803); flush after tree nav (`3166dae72` #3091); preserve steer vs follow-up semantics through compaction (`35a0d5d62` #6730: first queued prompt sent with `streamingBehavior: firstPrompt.mode`, `interactive-mode.ts:4815-4820` in `flushCompactionQueue` `:4760`).
- Mid-run extension messages with `triggerTurn` steer/followUp the active run; `triggerTurn:false` buffered in `_pendingCustomMessages` until `turn_end` (`agent-session.ts:2292-2328`, `1191-1194`; `240eb29c4` #8537; `47b5119d0` #8022) → [[out-of-band-message-deferral]].

## Constants
| name | value | path:line |
|---|---|---|
| `steeringMode` default | `"one-at-a-time"` | `packages/coding-agent/src/core/settings-manager.ts:854`; `packages/agent/src/agent.ts:247` |
| durable `steeringMode` default | `one-at-a-time` | `packages/durable/src/harness/agent.ts:56-73` |

## Evolution
- 2025-12-09 `0119d7610` AgentSession queue mode (`queueMode`).
- 2025-12-20 `117af076c` "interrupt tool batch on queued messages": queue polled after each tool, remaining calls skipped with error "Skipped due to queued user message." (feature "Queued message steering", `packages/ai/CHANGELOG.md:2155`, 0.25.1).
- 2026-01-02 `d0a4c3702` single `queueMessage()` split into `steer()`/`followUp()` (`packages/agent/CHANGELOG.md:727-731`, #403); setting migration `queueMode`→`steeringMode` (`settings-manager.ts:528-530`).
- 2026-03-16 `208a2cc12` "defer steering until after tool execution" — per-tool poll removed; CHANGELOG: "Fixed steering messages to wait until the current assistant message's tool-call batch fully finishes instead of skipping pending tool calls" (`packages/agent/CHANGELOG.md:497`). Docs `8a8e2a804` (#2330).
- 2026-09-08 `faa9863cb` input handlers run for queued messages.

## Evidence commits
`0119d7610` `117af076c` `d0a4c3702` `208a2cc12` `8a8e2a804` `b050c582a` `faa9863cb` `5c61d6bc9` `3166dae72` `35a0d5d62` `8eda4f5b2` `3852cb2b8` `240eb29c4` `47b5119d0`

## Quirks
- Duplicate texts across queues resolve to steering in the UI mirror; queued custom messages not mirrored; image-only (empty text) message never removed from mirror (`agent-session.ts:1118-1136`) (unverified behavior).
- Steering waits for the **whole** tool batch, so a slow tool delays the interrupt; hard cut only via abort ([[abort-propagation]]).

## Durable variant (packages/durable)
- Steer/follow-up are **durable submissions**: busy conversation → item in `pi.inbox` doc (`packages/durable/src/harness/inbox.ts:16-17`); `submit({whenBusy: steer|followUp|reject})`, default followUp; `reject` throws `ConversationBusy` (`packages/durable/README.md:306-323`).
- `requestId` dedupes within a conversation so a retry after restart returns the same submission (`packages/durable/README.md:115-124`; `harness/submissions.ts:155-160`).
- Boundaries: `postTools` places writes + steers; `final` places writes, steers, follow-ups; selection deterministic by item ID with mode first/all (`spec.md:2253-2256`, boundary table). Writes placed before user items, so a user item queued before a reset/compaction lands after it.
- Failed run leaves inbox alone; next submission's boundary places queued items oldest first (`spec.md:2298-2302`; `packages/durable/README.md:323`). `submission.abort()` withdraws only queued; `Conversation.abort()` withdraws queued steer/follow-ups but keeps writes.
- Queued `reset` placed during a tool round ends the run `unanswered/reset` (`generation.ts:633-637`).

## Failures
[[steering-skips-pending-tool-calls]] · [[side-phase-input-lost]] · [[queued-messages-stranded-at-run-end]] · [[reentrant-prompt-corrupts-state]]
