---
type: implementation
harness: opencode
concept: message-conversion-layer
commit: ecc4916b5a
files: [packages/opencode/src/session/message-v2.ts:131-428, packages/opencode/src/session/message-v2.ts:238-266, packages/opencode/src/session/message-v2.ts:300-373, packages/opencode/src/session/prompt.ts:1255-1286, packages/opencode/src/session/llm/request.ts:56-208, packages/core/src/session/runner/to-llm-message.ts:115-167, packages/core/src/session/runner/llm.ts:197-223]
---
[[message-conversion-layer]] in [[opencode]].

## Mechanism

### Legacy runtime — stored parts → AI SDK UI messages → ModelMessages
- Storage schema (`SessionV1` in `@opencode-ai/core/v1/session`): user `{agent, model, format, tools, system}`; assistant `{parentID, mode, agent, path, cost, tokens{input,output,reasoning,cache}, finish, error, summary}`; parts `text{synthetic, ignored}`, `reasoning`, `tool{callID, state: pending|running|completed|error}`, `file`, `step-start{snapshot}`, `step-finish{snapshot,tokens,cost}`, `patch{hash,files}`, `compaction{auto,overflow,tail_start_id}`, `subtask`, `agent`.
- Tool-part state machine written by the processor: `pending` (input streaming) → `running` (input parsed, `time.start`) → `completed` (`output`, `title`, `metadata`, `attachments`, later `time.compacted`) | `error` (`error`, `metadata.interrupted`) (`packages/opencode/src/session/processor.ts:160-205`, `:216-253`, `:331-351`).
- `MessageV2.toModelMessagesEffect` (`packages/opencode/src/session/message-v2.ts:131-428`): user text parts (non-ignored), files except text/plain and directories, `compaction` → "What did we do so far?", `subtask` → "The following tool was executed by the user" (`:238-249`).
- Assistant messages with an error are **skipped**, except `AbortedError` with some non-step-start/non-reasoning part (`:258-266`) → [[opencode--partial-message-persistence]].
- Tool parts: completed → output (or "[Old tool result content cleared]" when pruned, `:302-305`); pending/running → `output-error` "[Tool execution was interrupted]" because Anthropic needs every tool_use answered (`:362-373`); step-start kept as boundaries so the AI SDK splits multi-step assistants.
- Reasoning/provider metadata dropped when the stored message came from a different model (`differentModel`, `:255`).
- Request assembly per step (`packages/opencode/src/session/prompt.ts:1255-1286`, `packages/opencode/src/session/llm/request.ts:56-208`): system = ONE joined string [agent or provider prompt, env, `<available_references>`, instructions, MCP instructions, skills, `user.system`]; history; trailing assistant `MAX_STEPS_PROMPT` on the last step; tools sorted by name (`request.ts:184`).

### v2 runtime — projected `SessionMessage` → `@opencode-ai/llm` messages
- `toLLMMessage` (`packages/core/src/session/runner/to-llm-message.ts:115-167`): `agent-switched`/`model-switched` dropped; `user` text + media; `synthetic` → user; `system` → `Message.system` (chronological update, [[opencode--transcript-carried-system-prompt]]); `shell` → user "Shell command: …"; `compaction` → user `<conversation-checkpoint>`.
- `LLM.request` (`packages/core/src/session/runner/llm.ts:197-223`): `system: [agent prompt, epoch baseline]`, messages, materialized tools, session headers, `promptCacheKey`. Provider lowering is in `packages/llm/src/protocols/*`.
- Gaps tracked in the V1 parity table (`specs/v2/session.md:123-151`): provider-family base prompts, plugin transforms, reminders, structured output missing.

## Constants
| name | value | path:line |
|---|---|---|
| compaction placeholder text | "What did we do so far?" | `packages/opencode/src/session/message-v2.ts:241` |
| interrupted tool text | "[Tool execution was interrupted]" | `packages/opencode/src/session/message-v2.ts:370` |

## Evolution
- 2025-09-17 `ff6a93f355` keep aborted messages only with substantive parts.
- 2026-03-27 `c33d9996f0` AI SDK v6.
- 2026-06-03 `76ee87ead8` v2 runtime with its own message model and native protocols.

## Quirks / drift
- Legacy goes through two conversions (stored → `UIMessage` → `ModelMessage` via `convertToModelMessages`), so provider-specific fixes live in `ProviderTransform.message` after conversion.
- v2 drops the synthetic tool-call shape for user shell runs ([[opencode--synthetic-tool-call-injection]]).

Contrast: [[pi--message-conversion-layer|pi]] keeps an `AgentMessage` union with custom roles converted to user text by `convertToLlm`; opencode legacy stores parts and renders placeholders at conversion, v2 projects typed session messages and lowers system updates per provider.
