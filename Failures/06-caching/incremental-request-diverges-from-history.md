---
type: failure
concepts: [session-affinity-cache-routing]
harnesses: [codex]
---
**Symptom** — WebSocket incremental requests (`previous_response_id` plus new items) did not match the real conversation:
- They reused `previous_response_id` even when tools had changed mid-turn, so the server continued under stale request properties.
- They resent output items that were already in server context, duplicating them.

**Root cause** — Delta validity was checked on the input alone, and the baseline left out the server's own output items. Request properties that changed outside `input` (tools, instructions, effort) were invisible to the check.

**Fix · [[codex]]**
- `0639c33892` 2026-02-10: compare all request properties ("Tools can dynamically change mid-turn now").
- `4473147985` 2026-02-10: don't resend output items. The baseline is previous input plus previous output.
- `responses_request_properties_match` destructures `ResponsesApiRequest` exhaustively, so the compiler forces a decision for any new field (fail-closed) (`codex-rs/core/src/client.rs:341-393`).
- A strict prefix-extension check, item by item and ignoring internal metadata; a late tool-result metadata change forces a full resend (`codex-rs/core/src/client.rs:395-414,1401-1434`).
- `c9253c4977` 2026-10-05: changed base instructions force a full request.
- `17d552fb4d` 2026-05-18: compaction no longer resets WebSocket state externally, because the strict-extension check already catches it.
- A full request records its reset reason: `incremental|other|restored_history|no_previous_request` (`codex-rs/core/src/client.rs:1979-1994`).

**Lesson** — A delta request is valid only if every non-input parameter is equal and the new input strictly extends (previous input + previous output). Make the equality check fail closed for new fields.

Related: [[session-affinity-cache-routing]] · [[cache-preserving-config-update]] · [[server-side-state-missing-on-continuation]] · [[codex--session-affinity-cache-routing|codex]]
