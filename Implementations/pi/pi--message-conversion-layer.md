---
type: implementation
harness: pi
concept: message-conversion-layer
commit: b30a6dd77
files: [packages/agent/src/types.ts:351, packages/agent/src/types.ts:196, packages/agent/src/agent.ts:38, packages/agent/src/agent-loop.ts:381, packages/coding-agent/src/core/messages.ts:11, packages/coding-agent/src/core/messages.ts:148, packages/coding-agent/src/core/sdk.ts:296]
---
[[message-conversion-layer]] in [[pi]].

## Mechanism
- **Union type**: `AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages]`; `CustomAgentMessages` is empty and extended by declaration merging (`packages/agent/src/types.ts:351-374`).
- **Per-request pipeline** in `streamAssistantResponse` (`packages/agent/src/agent-loop.ts:381-407`): `context.messages` → `transformContext(messages, signal)` (AgentMessage→AgentMessage, [[pi--context-transform-hook]]) → `convertToLlm(messages)` (AgentMessage→Message) → `normalizeContext` (pi-ai) → `getApiKey(provider)` per call (expiring OAuth, `1167e8445` #223) → `streamFn`. README diagram `AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM` (`packages/agent/README.md:51-58`). Contract for both hooks: "must not throw or reject. Return a safe fallback value instead" (`packages/agent/src/types.ts:196-235`).
- **Default converter** keeps only `system`/`user`/`assistant`/`toolResult` (`packages/agent/src/agent.ts:38-46`).
- **Coding-agent custom roles** (declaration merge in `packages/coding-agent/src/core/messages.ts:69-77`): `bashExecution` (user `!cmd`), `custom` (extension message, `display` flag), `branchSummary`, `compactionSummary`.
- **`convertToLlm`** (`packages/coding-agent/src/core/messages.ts:148-196`), exhaustive `never` switch:
  - `bashExecution` → user text via `bashExecutionToText`: ``Ran `cmd` `` + fenced output or "(no output)" + "(command cancelled)" / "Command exited with code N" + "[Output truncated. Full output: <path>]"; skipped when `excludeFromContext` (`!!cmd`) (`packages/coding-agent/src/core/messages.ts:82-98`, `152-161`).
  - `custom` → user (string content wrapped into text block).
  - `branchSummary` → user `BRANCH_SUMMARY_PREFIX + summary + BRANCH_SUMMARY_SUFFIX` ("The following is a summary of a branch that this conversation came back from:\n\n<summary>\n…</summary>", `:19-24`).
  - `compactionSummary` → user `COMPACTION_SUMMARY_PREFIX + summary + COMPACTION_SUMMARY_SUFFIX` ("The conversation history before this point was compacted into the following summary:\n\n<summary>\n…\n</summary>", `:11-17`).
  - `system`/`user`/`assistant`/`toolResult` pass through (system messages carry prompt sections + tool deltas, [[transcript-carried-system-prompt]]).
- **Policy wrapper** in `packages/coding-agent/src/core/sdk.ts:296-331` `convertToLlmWithBlockImages`: if `images.blockImages` (checked dynamically per call), every image in user/toolResult content → text "Image reading is disabled.", consecutive placeholders deduped; history keeps images ([[pi--image-normalization]]).
- Same `convertToLlm` reused by compaction/branch summarization before serialization and exported for extensions.
- Downstream provider-boundary repair (drop error/aborted assistants, synthesize "No result provided", non-vision placeholders, cross-model thinking → text) lives in pi-ai `transformMessages` → [[transcript-replay-repair]], [[cross-provider-handoff]].
- Message events: AgentSession persists every `message_end` (custom roles included) so the session log stores the rich union, not wire messages.

## Evolution
- 2025-12-19 `1167e8445` (#223) `getApiKey` per LLM call inside the pipeline.
- 2026-01-06 `1fc2a912d` `blockImages` filter at the conversion layer.
- v2→v3 session migration renamed role `hookMessage` → `custom` (`migrateV2ToV3`, `packages/coding-agent/src/core/session-manager.ts:315-331`).
- 2026-09-16 `9e05370b2` (#9548) system messages become first-class transcript entries passing through conversion.

## Evidence commits
`1167e8445` `1fc2a912d` `9e05370b2` `8c0ccd14b`

## Quirks
- All custom roles become **user** messages — a plugin note or shell transcript is indistinguishable from user text except by wording (inferred).
- `8c0ccd14b`: null `content` from untyped JS extensions/hand-edited sessions normalized at ingestion boundaries (incl. `transformMessages`) rather than in every consumer.

## Failures
[[context-handler-drops-system-state]] · [[side-channel-message-splits-tool-pair]]
