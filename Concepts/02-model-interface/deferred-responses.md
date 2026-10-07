---
type: concept
stage: model-interface
tier: variant
aliases: [DeferredHandle, streamDeferred, fetchDeferred, cancelDeferred, "stopReason: deferred", durable poll phase, deferred-response-polling, pollAfterMs]
harnesses: [pi]
---
A provider generation that runs in the background or as a batch job. The harness gets back a durable handle (id, expiry, poll hint) instead of a live stream, persists it, polls for the result later (even after a restart), and can cancel it best-effort.

## Why
- Long or cheap batch-priced generations should not tie up a live connection.
- A process crash should not lose a paid-for generation.
- Without a handle type and a dedicated stop reason, the loop cannot tell "come back later" apart from "done" or "error".

## Design space
- **Request**
  - A per-request flag plus an optional window (15m/1h/24h). The response ends with `stopReason: deferred` and a handle.
- **Retrieval**
  - A single status check.
  - A long-poll `wait`.
  - A durable scheduler phase that polls after `pollAfterMs`, with a default.
- **Support**
  - Capability flags on lazily loaded adapters. Only some APIs implement it: in pi, Anthropic and the faux test provider.
- **Replay**
  - Deferred assistant messages are excluded from requests until they resolve (durable context derivation).
- codex: absent. `ResponsesApiRequest` has no `background` field (`codex-rs/codex-api/src/common.rs:279-304`). The nearest analogue is server-held continuation state over one WebSocket (`previous_response_id`). It is per connection and needs a full-resend fallback ([[server-side-state-missing-on-continuation]], [[session-affinity-cache-routing]]).

## Implementations
- [[pi--deferred-responses|pi]] — pi-ai `SimpleStreamOptions.deferred`, `DeferredHandle`, and `streamDeferred`/`fetchDeferred`/`cancelDeferred`. The durable `pi.generation` `poll` phase uses `pollAfterMs ?? 5000`.

## Failures
- None recorded for pi.
- [[server-side-state-missing-on-continuation]]

## Related
[[durable-execution]] · [[unified-provider-api]] · [[transcript-replay-repair]] · [[turn-loop]]
