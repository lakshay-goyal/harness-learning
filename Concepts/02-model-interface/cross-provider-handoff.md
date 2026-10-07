---
type: concept
stage: messages
tier: candidate
aliases: [transformMessages, transform-messages.ts, isSameModel, cross-provider-transcript-handoff, foreign-reasoning-as-plain-text, unsigned-tool-call-handoff, non-vision-image-placeholder, synthetic-bridge-message, text-phase-signature, typed-item-id-prefix, requested-vs-response-model-identity, empty-content-placeholders, responseModel, TextSignatureV1, model_switch, maybe_run_previous_model_inline_compact]
harnesses: [pi, codex]
---
At request time, rewrite stored conversation history so a different provider, API or model will accept it in the middle of a session. Foreign reasoning is demoted, foreign opaque tokens are dropped, ids are reshaped, unsupported media gets placeholders, and role-order rules are satisfied. Raw history stays untouched.

## Why
- Each vendor's history contains things only that vendor accepts: signed reasoning, typed item ids, tool-call ids in vendor-specific formats. Replaying them verbatim to another model gets a 400 ([[foreign-reasoning-signature-replayed]], [[responses-reasoning-item-pairing]], [[tool-call-id-requirement-drift]]).
- The format chosen for foreign reasoning acts as a few-shot example, and the new model imitates it ([[thinking-tag-mimicry]]).
- Some vendors demand signed tool calls that history from other vendors does not have ([[gemini-unsigned-tool-call-replay]]).
- When content is dropped or placeholdered silently, the model hallucinates ([[placeholder-text-misleads-model]]).
- Which model counts as "same" decides whether reasoning is kept. If identity follows the model name the server echoes back, relays break that check ([[model-relabel-breaks-same-model-check]]).
- Dropping reasoning that open-weight chat templates expect changes model behavior ([[reasoning-not-replayed-degrades-tool-args]]).

## Design space
- **When to rewrite**
  - Rewrite the stored history once, at switch time.
  - Rewrite at the provider boundary on every request, leaving raw history untouched. *pi chose this:* `transformMessages`, called by every adapter.
- **Same-model test**
  - Provider only.
  - Provider + API + model id, using the *requested* id rather than the response model. *pi chose this:* 1283afd0d.
- **Foreign reasoning**
  - Drop it.
  - Wrap it in `<thinking>` tags. *pi tried this first* (46b5800d3); it caused mimicry, fixed in 29379ea0a and 16e142ef7.
  - Plain untagged text. *pi chose this.*
- **Unsigned foreign tool calls (signature-requiring targets)**
  - Convert to text. *pi tried this:* b18f401d9.
  - Text plus an anti-mimicry note. *pi tried this:* 5d3e7d5aa.
  - Vendor sentinel signature. *pi tried this:* a0d839ce8; Vertex rejected it.
  - Send unsigned. *pi chose this:* f7df47408.
- **Typed item ids**
  - Coerce them to the target type.
  - Drop optional ids whose prefix does not match the target type. *pi chose this:* bc2d8dc1c, d327b9c76.
- **Unsupported media**
  - Silently drop.
  - Explicit, true placeholder text, deduplicated for consecutive images. *pi chose this.*
- **Role-order rules**
  - Synthetic bridge messages, e.g. an assistant "I have processed the tool results." turn, or a follow-up user message carrying tool-result images.
- **Message phase and id metadata**
  - Versioned text signature `{v, id, phase}`, kept same-model only.
- **Within one vendor family** (codex)
  - Project history onto the *receiving model's capabilities* per request: modalities → text placeholder; image `detail: original` → `high`; Lite strips detail. ✔ codex
  - Summarize with the model that wrote the history when the compaction-compatibility hash changes or the window shrinks, falling back to the current model if the old one is rejected. ✔ codex
  - Persist the originating truncation budget on each tool output, so replay under a new model is deterministic. ✔ codex (`aa88a0333c`)

## Implementations
- [[pi--cross-provider-handoff|pi]] — `transformMessages` (packages/ai/src/api/transform-messages.ts) with an adapter-supplied id normalizer and id map. It demotes reasoning to plain text, strips signatures, adds image placeholders, and runs per-adapter replay rules.
- [[codex--cross-provider-handoff|codex]] — no vendor-format crossing. Per-model capability projection (modalities, image detail), a `<model_switch>` notice, previous-model compaction with fallback; third-party providers lose only `encrypted_function_args`.

## Failures
- [[thinking-tag-mimicry]]
- [[gemini-unsigned-tool-call-replay]]
- [[tool-call-id-requirement-drift]]
- [[foreign-reasoning-signature-replayed]]
- [[model-relabel-breaks-same-model-check]]
- [[responses-reasoning-item-pairing]]
- [[reasoning-not-replayed-degrades-tool-args]]
- [[placeholder-text-misleads-model]]
- [[assistant-content-shape-misread]]
- [[tool-result-image-routing]]
- [[model-switch-replays-unsupported-content]]
- [[compaction-pinned-to-unavailable-model]]
- [[truncation-budget-drift-on-replay]]

## Related
[[signed-reasoning-replay]] · [[tool-call-id-normalization]] · [[transcript-replay-repair]] · [[unified-provider-api]] · [[image-normalization]] · [[message-conversion-layer]] · [[model-resolution]] · [[virtual-model-router]] · [[provider-breadth]]
