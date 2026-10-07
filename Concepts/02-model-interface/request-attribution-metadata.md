---
type: concept
stage: model-interface
tier: candidate
aliases: [client_metadata, x-codex-turn-metadata, x-codex-installation-id, x-codex-window-id, x-codex-parent-thread-id, x-openai-subagent, x-codex-beta-features, ResponseDebugContext, MAX_MCP_ATTRIBUTION_BYTES]
harnesses: [codex]
---
Attach the harness's identity and call-site context to every model request, as headers and/or body metadata, so the provider can route, attribute, bill and debug it. The context covers installation, session, thread, turn, context window, parent/fork lineage, subagent kind and request kind. None of this metadata goes into the model-visible prompt.

## Why
- One user action fans out into many requests: sampling, compaction, review, memory and subagents. Without per-request kind and lineage, the backend cannot attribute cost, apply per-kind routing or quotas, or reconstruct a session for debugging.
- Cache and routing affinity need stable ids, which must be separate from what the model sees ([[session-affinity-cache-routing]]).
- Metadata grows: MCP attribution, tool metadata, trace context. It needs caps and must be shed before it breaks request size limits.
- Internal metadata must not leak to third-party providers.

## Design space
- **Carrier**
  - Headers only.
  - Body metadata map with header mirrors for HTTP and in-body only for WebSockets. ✔ codex
- **Granularity**
  - Session id only. pi sends session affinity headers per provider.
  - Full lineage per request: installation, session, thread, turn, window, parent/root, subagent kind, request kind, trigger. ✔ codex
- **Third-party providers**
  - Send everything.
  - Strip internal metadata for non-vendor providers (`include_internal_metadata`). ✔ codex
- **Size control**
  - None.
  - A per-field cap (MCP attribution 16 KiB) plus shedding optional metadata from the wire copy above a request-size threshold (15 MiB). ✔ codex
- **Gating**
  - Always on (codex, vendor client).
  - Only when telemetry is enabled (pi attribution headers; [[install-telemetry]]).
- **Response side**
  - Capture request ids and error JSON headers into a debug context. ✔ codex

## Implementations
- [[codex--request-attribution-metadata|codex]] — `client_metadata` map plus `x-codex-*`/`x-openai-subagent`/`session-id`/`thread-id` headers. MCP attribution cap, 15 MiB wire shedding, `ResponseDebugContext`.

## Failures
- (none recorded)

## Related
[[session-affinity-cache-routing]] · [[http-transport-hardening]] · [[provider-identity-shim]] · [[harness-identity]] · [[install-telemetry]] · [[errors-as-stream-events]]
