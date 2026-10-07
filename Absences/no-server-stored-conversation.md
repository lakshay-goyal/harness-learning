---
type: absence
harnesses: [codex]
---
# no-server-stored-conversation

Every HTTP request is `store: false` with the full input and encrypted reasoning; the harness, not the provider, owns conversation state.

**What's missing**
- `store: false`, full `input` resent, `include: ["reasoning.encrypted_content"]` (`codex-rs/core/src/client.rs:961`, `:996-1012`).
- `previous_response_id` used only inside one WebSocket connection as a bandwidth optimization, when request properties are identical and the input is a strict extension (`codex-rs/core/src/client.rs:337-414`); `previous_response_not_found` → full resend (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:166-168`).

**Evidence of decision**
- `591cb6149a` 2025-07-23 "Always send entire request context (#1641)".

**Implication**
- Zero-data-retention compatible; resume/fork/compaction work from local rollouts ([[session-tree]]); cost is resending full context each request, mitigated by prompt caching ([[cache-strategy]]).
- Server-side continuation failures remain a failure class ([[server-side-state-missing-on-continuation]]).

Related: [[deferred-responses]] · [[signed-reasoning-replay]] · [[session-tree]] · [[cache-strategy]] · [[Absences]]
