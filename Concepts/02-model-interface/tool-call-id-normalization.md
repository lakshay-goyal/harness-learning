---
type: concept
stage: messages
tier: candidate
aliases: [normalizeToolCallId, toolCallIdMap, toolCallCounter, tool-call-id-synthesis, requiresToolCallId, MISTRAL_TOOL_CALL_ID_LENGTH, "call_id|item_id", "fc_<hash>", shortHash]
harnesses: [pi, opencode]
---
Rewrite or synthesize tool-call ids so they satisfy the target provider's character set, length, prefix and uniqueness rules, while the pairing between each call and its result is kept.

## Why
- Id grammars differ by provider:
  - OpenAI Responses ids are `call_id|item_id` and can be 450+ characters with `|`.
  - Anthropic requires `^[a-zA-Z0-9_-]+$` and at most 64 characters.
  - Chat Completions allows at most 40 characters.
  - Mistral requires exactly 9 alphanumeric characters.
  - Codex rejects a trailing `_`.

  Switching providers mid-session fails without rewriting ([[cross-provider-tool-call-id-normalization]]).
- Lossy compression such as truncation creates collisions. Providers also omit or duplicate ids ([[tool-call-id-collision]]).
- Id requirements change across model generations ([[tool-call-id-requirement-drift]]).
- Typed item ids carry server-side pairing state ([[responses-reasoning-item-pairing]]).
- Streams key tool-call fragments by id even when the id is unstable ([[streamed-tool-call-fragmentation]]).

## Design space
- **Who normalizes**
  - A global sanitizer.
  - An adapter-supplied `normalizeToolCallId(id, model, source)`, applied only when the source is not the same model. A shared id map then rewrites the later tool results. *pi chose this:* 2c7c23b86.
- **Compression**
  - Truncate. *pi did this at first;* it caused collisions.
  - Sanitize, then join the parts, then hash when over the limit (`prefix_<8-char hash>`). *pi chose this:* d9f7f8147.
  - Mistral: hash to 9 characters, with bidirectional maps and reseeding on collision.
- **Foreign typed ids**
  - Hash into the target's prefix, e.g. `fc_<hash>`.
  - Drop the optional id when its prefix does not match the target type.
- **Missing or duplicate ids from the provider**
  - Synthesize `name_ts_counter`. *pi does this for Gemini.*
  - Derive the id from the stream index. *pi does this for Mistral.*
- **When ids are required**
  - Gate by provider.
  - Gate by parsed model version, e.g. Gemini ≥3. *pi chose this:* cbaca6038.
- **Truncate-and-pad**: Mistral ids stripped to alphanumerics and cut/padded to 9 chars without hashing (opencode).

## Implementations
- [[pi--tool-call-id-normalization|pi]] — per-adapter normalizers (Anthropic and Bedrock ≤64 alphanumeric, Completions ≤40 with hash, Responses `fc_` hashing and trailing `_` strip, Mistral 9-char hash, Google when `requiresToolCallId`) plus the `toolCallIdMap` inside `transformMessages`.
- [[opencode--tool-call-id-normalization|opencode]] — Claude ids scrubbed to `[a-zA-Z0-9_-]`, Mistral family 9 chars + `"Done."` bridge; v2 Gemini ids synthesized `tool_N`.

## Failures
- [[cross-provider-tool-call-id-normalization]]
- [[tool-call-id-collision]]
- [[tool-call-id-requirement-drift]]
- [[responses-reasoning-item-pairing]]
- [[streamed-tool-call-fragmentation]]

## Related
[[cross-provider-handoff]] · [[transcript-replay-repair]] · [[unified-provider-api]] · [[signed-reasoning-replay]]
