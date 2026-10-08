---
type: implementation
harness: codex
concept: signed-reasoning-replay
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:942-953, codex-rs/core/src/client.rs:961, codex-rs/core/src/client.rs:1003]
---
[[signed-reasoning-replay]] in [[codex]].

## Mechanism
- **Stateless replay.** Every request sets `store: false` and `include: ["reasoning.encrypted_content"]` (`codex-rs/core/src/client.rs:961,1003`). Encrypted `Reasoning` items are kept in history and resent verbatim.
- **Server item ids are not sent back** (`5bcc9d8b77` 2025-09-09, "Do not send reasoning item IDs", issue #3292).
- **No foreign-reasoning conversion.** Every provider is Responses-shaped, so nothing is demoted or converted. The only cross-provider step is for non-OpenAI providers: internal chat-message metadata passthrough is stripped and `encrypted_function_args` on `FunctionCall` items is cleared (`codex-rs/core/src/client.rs:942-953`).
- **WebSocket continuation.** Within one WebSocket connection, `previous_response_id` refers to server-held state while `store` stays false ([[codex--session-affinity-cache-routing|codex WS]]).
- **Estimation.** Encrypted reasoning still counts against the window. Codex adds estimated tokens for reasoning items before the last user boundary unless the server signals `ServerReasoningIncluded` ([[token-estimation]]; `b519267d05` 2025-11-21).
- **Effort binding.** The effort a prefix was built with stays pinned per window, and changes are appended as configuration items ([[cache-preserving-config-update]]). This keeps the request shape constant.

## Evolution
- `591cb6149a` 2025-07-23: "Request encrypted COT when not storing Responses."
- `5bcc9d8b77` 2025-09-09: stop sending reasoning item ids.
- `414b8be8b6` 2025-09-12 (#3539): "Always request encrypted cot". Without it, follow-up stateless requests "will fail with 500".
- `dc2f26f7b5` 2025-11-04: `is_api_message` misclassified `ResponseItem::Reasoning`; fixed.
- `d2d00b6632` 2026-07-10 (#32206): "Always send reasoning parameters in Responses requests". The `supports_reasoning_summaries` model flag was removed.

## Versus pi
- [[pi--signed-reasoning-replay|pi]] handles signatures from six vendor formats, same-model scoping, stale-signature `block_binding`, and empty-signature compat.
- Codex has one format (`encrypted_content`) and one vendor family, so the problem reduces to *always request it, always replay it, never send server ids*.

## Failures
- [[opaque-reasoning-payload-lost]]
