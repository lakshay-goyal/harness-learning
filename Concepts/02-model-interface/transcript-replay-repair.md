---
type: concept
stage: messages
tier: candidate
aliases: ["No result provided", "skip errored/aborted", drop-failed-turns-on-replay, orphaned-tool-call-repair, deferred-system-message-placement, "Tool result unavailable: history ends before this call completed.", failInterruptedTools, unsupportedParts, _noop]
harnesses: [pi, opencode]
---
At the provider boundary, normalize replayed history so it satisfies API invariants:
- Turns that failed or were aborted are excluded.
- Tool calls left without a result get a synthesized error result.
- Messages that would split a tool call from its result are moved after the result.
- Missing optional fields are filled in.

## Why
- Interruptions (Esc, 429/500 in the middle of tool execution, session switches) leave tool calls without results. Every provider rejects that ([[orphaned-tool-calls-and-results]]).
- Partial or errored assistant turns contain reasoning without a following item, or empty content. Replaying them fails ([[failed-turns-replayed]], [[aborted-reasoning-signature-invalid]]).
- Deleting tool calls to restore pairing invalidates the signed reasoning that covers them.
- Untyped plugins and old session files carry `null` content or undefined arguments, which crashes replay ([[missing-optional-fields-crash-replay]]).

## Design space
- **Where to repair**
  - Persist a repaired history.
  - Repair at one choke point on every request and keep the raw log intact. *pi chose this:* `transformMessages`.
  - Durable variant: repair during context derivation.
- **Orphaned tool calls**
  - Delete the call. *pi did this in v1 (51f5448a5);* it broke signatures.
  - Synthesize an `isError` result. *pi chose this:* fb1fdb600, plus the trailing-call case a23fab469.
- **Failed turns**
  - Convert unsigned thinking to text. *pi tried this:* 387cc97ba.
  - Filter only empty error messages: fbb74bb29.
  - Skip every errored or aborted assistant message. *pi chose this:* 2d27a2c72, which also deleted the `strictResponsesPairing` compat flag.
- **System or out-of-band messages placed between a call and its result**
  - Hold them back until the results are emitted. *pi chose this:* 9e05370b2.
  - The loop also buffers them to the turn boundary (see [[out-of-band-message-deferral]]).
- **Layering**
  - Each adapter repairs independently. This diverged in pi: 0f3a0f78b, where Codex dropped calls that the shared pass had given results.
  - One shared pass plus minimal adapter rules.
- **Orphaned tool calls (durable)**: fail every `pending|running` call before the next drain, never replay it (opencode v2) · synthesize an error at replay (opencode legacy).
- **Interrupted partial output**: replay as a successful result (opencode legacy, shell).
- **History requires a tools field**: inject a no-op placeholder tool when history has tool calls but no tools are enabled (opencode, Copilot/LiteLLM).
- **Unsupported modality**: replace the part with `ERROR: Cannot read … Inform the user.` text (opencode).

## Implementations
- [[pi--transcript-replay-repair|pi]] — `transformMessages` pass 0 normalizes null content and non-vision images; pass 2 skips errored/aborted messages, synthesizes "No result provided" results and defers system messages. The durable variant synthesizes "Tool result unavailable…" and also excludes `deferred` messages.
- [[opencode--transcript-replay-repair|opencode]] — `toModelMessages` skips errored turns and closes dangling calls; `ProviderTransform.message` filters empty blocks per SDK; v2 fails interrupted tools durably.

## Failures
- [[side-channel-message-splits-tool-pair]]
- [[session-switch-leaves-dangling-tool-calls]]
- [[orphaned-tool-calls-and-results]]
- [[failed-turns-replayed]]
- [[aborted-reasoning-signature-invalid]]
- [[missing-optional-fields-crash-replay]]
- [[placeholder-tool-gets-called]]
- [[empty-payload-rejections]]
- [[signed-empty-reasoning-dropped]]

## Related
[[cross-provider-handoff]] · [[tool-call-id-normalization]] · [[signed-reasoning-replay]] · [[partial-message-persistence]] · [[context-projection]] · [[context-edit-overlay]] · [[out-of-band-message-deferral]] · [[abort-propagation]] · [[truncated-tool-call-guard]]
