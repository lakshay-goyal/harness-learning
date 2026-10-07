---
type: concept
stage: messages
tier: candidate
aliases: [thinkingSignature, redacted thinking, redacted_thinking, encrypted_content, "store:false", thoughtSignature, textSignature, reasoning_details, block_binding, "prefix_mismatch_behavior: drop_block", providerThinkingLevel, stale-reasoning-drop, signed-empty-block-preservation, stateless-store-false-replay, per-turn-effort-markers, allowEmptySignature, ANTHROPIC_BLOCK_BINDING, INCLUDE_ENCRYPTED_REASONING, interleaved.field, hasSignedReasoning]
harnesses: [pi, opencode]
---
Send provider-signed or encrypted reasoning back verbatim to the model that produced it. For any other model, drop it or convert it. Handle three edge cases: blocks that are signed but empty, signatures that have gone stale because the prompt changed, and per-turn effort settings that are bound to the signature.

## Why
- Reasoning continuity across turns, and with stateless APIs (`store:false`) the encrypted reasoning, depends on echoing opaque payloads exactly. When those payloads are dropped, multi-turn quality degrades or the request is rejected ([[opaque-reasoning-payload-lost]], [[reasoning-not-replayed-degrades-tool-args]]).
- The signature is the replay payload, not the visible text:
  - Signed blocks whose text is empty must still be sent ([[signed-empty-reasoning-dropped]]).
  - Endpoints that call themselves "compatible" disagree about empty signatures ([[empty-signature-semantics-vary]]).
- Signatures are scoped to one provider and model. Foreign or partial signatures cause a 400 ([[foreign-reasoning-signature-replayed]], [[aborted-reasoning-signature-invalid]], [[model-relabel-breaks-same-model-check]]).
- Signed reasoning is bound to the system prompt, tools and effort it was produced under. Changing any of them mid-session leaves persistent 400s ([[stale-thinking-signature-after-prefix-change]]).
- Some providers require signatures on tool calls in thinking mode ([[gemini-unsigned-tool-call-replay]]).
- Formatting reasoning as tagged text teaches the model the tags ([[thinking-tag-mimicry]]).

## Design space
- **Replay storage**
  - Server-side conversation state (`previous_response_id`, `store:true`).
  - Stateless replay of the encrypted item. *pi chose this:* `store:false` plus `include: reasoning.encrypted_content`.
  - Codex WebSocket continuation via `previous_response_id` while `store` stays false.
- **Replay key**
  - Text presence.
  - Signature presence. *pi chose this:* 6731a0ba9, 6138f5a07, b4e7d5c44, which reverted 8fc2b7682.
- **Scope check**
  - Same provider.
  - Same provider + API + model, with a format check such as base64 for TYPE_BYTES fields. *pi chose this:* 934e7e470.
- **Stale signatures after a prefix change**
  - Fail the request.
  - Strip locally.
  - Ask the server to drop mismatched blocks (`block_binding: drop_block`) and replay historical effort markers so the prefix stays exact. *pi chose this:* 4e69b0c28, 69f0be6f0.
- **Opaque streaming payloads**
  - Accumulate natively and serialize once when the block ends.
  - Back-fill from the terminal event when per-item events omit the payload (Azure).
  - Flush buffered bytes on every terminal path.
- **Empty signatures**
  - Downgrade to text. *This is the default.*
  - Replay `signature:""` behind a per-model compat flag.
- **Distinguishing reasoning from carriers of reasoning state**
  - Gemini signatures can appear on any part. Only `thought: true` marks reasoning.
- **Where ids are stripped**: in the message transform before serialization and request signing (opencode `a86ecf3bba`); a fetch-hook body rewrite broke signed Bedrock requests.
- **Required empty field**: some APIs need a reasoning field on every assistant message, even empty (DeepSeek V4, opencode `86715fecc4`).

## Implementations
- [[pi--signed-reasoning-replay|pi]] — `thinkingSignature`, `textSignature` and `thoughtSignature` fields, kept only when the model is the same. Per-adapter replay:
  - Anthropic `thinking` and `redacted_thinking`, managed-effort markers.
  - Bedrock `reasoningContent`.
  - Responses: the whole reasoning item as JSON.
  - Completions: `reasoning_details` and `reasoning_content`.
  - Gemini: `thoughtSignature` on any part.
  - Mistral: native thinking chunks.
- [[opencode--signed-reasoning-replay|opencode]] — `store:false` + encrypted reasoning with item ids stripped, Anthropic `drop_block` binding via SDK patch, DeepSeek/interleaved reasoning padding; re-implemented in `packages/llm`.

## Failures
- [[thinking-tag-mimicry]]
- [[gemini-unsigned-tool-call-replay]]
- [[signed-empty-reasoning-dropped]]
- [[empty-signature-semantics-vary]]
- [[aborted-reasoning-signature-invalid]]
- [[foreign-reasoning-signature-replayed]]
- [[model-relabel-breaks-same-model-check]]
- [[stale-thinking-signature-after-prefix-change]]
- [[opaque-reasoning-payload-lost]]
- [[reasoning-not-replayed-degrades-tool-args]]

## Related
[[cross-provider-handoff]] · [[thinking-level-abstraction]] · [[transcript-replay-repair]] · [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[unified-provider-api]]
