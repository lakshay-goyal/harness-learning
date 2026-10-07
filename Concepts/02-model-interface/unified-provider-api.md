---
type: concept
stage: model-interface
tier: candidate
aliases: [pi-ai, "stream()", AssistantMessageEventStream, StreamFunction, AssistantMessageEvent, unified-assistant-event-stream, owned-sse-decoder, output-index-slotting, lazy-provider-loading, harness-native-wire-protocol, pi-messages, mapStopReason, rawStopReason, LLMEvent, "@opencode-ai/llm", Route.make, OPENCODE_EXPERIMENTAL_NATIVE_LLM, ProviderTransform, patchedDependencies]
harnesses: [pi, opencode]
---
One provider-neutral streaming API and event protocol (start → text/thinking/toolcall start/delta/end → done|error, carrying a live partial message and a normalized stop reason) that sits on top of many vendor APIs, so the loop never sees vendor wire formats.

## Why
- Without it, every vendor quirk leaks into the loop. Each adapter re-learns stop-reason mapping ([[stop-reason-mapping-gaps]]), stream assembly ([[streamed-tool-call-fragmentation]], [[stream-delta-assembly-errors]]), SSE framing ([[sse-framing-errors]]) and payload hygiene ([[empty-payload-rejections]], [[endpoint-rejects-request-field]]).
- History built by one model must be replayable to another, which needs one canonical message shape. When the wire shape is not canonical, models misread or mirror it ([[assistant-content-shape-misread]]).
- The event stream is a hot path and a widely implemented type, so it needs efficient draining and structural typing that does not break implementers ([[quadratic-event-queue-drain]], [[private-fields-break-duck-typed-streams]]).
- Provider SDKs are heavy and often Node-only. Loading every SDK eagerly breaks browser and bundled builds ([[node-only-imports-break-browser-bundle]]).

## Design space
- **Abstraction level**
  - Thin passthrough of each vendor SDK's types.
  - Neutral event protocol with a live accumulated partial. *pi chose this:* `AssistantMessageEvent` and `partial` is a shared accumulator, not a snapshot.
  - Harness-native wire protocol, where the gateway does the vendor translation server-side. *pi has this as an additional API:* `pi-messages` / Radius.
- **Stream parsing**
  - Trust the vendor SDK's stream helper.
  - Own the SSE decoder, with an event whitelist and JSON repair. *pi chose this for Anthropic,* but only after a same-day revert and reapply: 4b926a30a → fc9220d2d → e58d631c8.
- **Item demultiplexing**
  - A single "current block" cursor. *pi did this originally.*
  - Slots keyed by output index and by id. *pi chose this:* 8c9dbffa3, 01509156b.
- **Stop reasons**
  - Partial mapping.
  - Total mapping, where unknown reasons become errors and the raw reason is kept. *pi chose this.*
  - Tool presence promotes the stop reason to toolUse only when the stop was `stop`, never when it was length.
- **Capability gating**
  - Sniff the provider id or URL at runtime.
  - Per-model compat flags from generated metadata. *pi moved here:* 6184307c3, 890f92088.
- **Loading**
  - Import every SDK eagerly.
  - Return the stream synchronously and lazy-import the adapter behind it. *pi chose this:* `.lazy.ts`, `lazyStream`.
- **Extensibility**
  - Closed set of APIs.
  - Open `Api` string, so plugins can implement their own stream functions. *pi chose this.*
- **Vendor SDK as substrate**: wrap Vercel AI SDK `streamText` and patch SDKs in place (`patches/*`) when they lag providers (opencode legacy) · own route-first protocols, Route = Protocol × Endpoint × Auth × Framing, one provider turn per call (opencode v2 `packages/llm`).
- **Quirk placement**: one provider-transform middleware hotspot over the final prompt (opencode `ProviderTransform`, ~1900 lines).
- **Unsupported routes**: fail loudly, never downgrade (opencode v2).

## Implementations
- [[pi--unified-provider-api|pi]] — pi-ai `Models.stream/streamSimple` over 10 chat APIs and 42 providers. It has a typed event protocol, lazily loaded adapters, compat flags generated per model, and its own SSE/JSON parsing.
- [[opencode--unified-provider-api|opencode]] — legacy AI SDK + `ProviderTransform` middleware + vendored SDK patches behind `LLMEvent`; v2 `packages/llm` own protocols, three routes accepted by the runner.

## Failures
- [[cache-marker-namespace-mismatch]]
- [[placeholder-tool-gets-called]]
- [[stop-reason-mapping-gaps]]
- [[streamed-tool-call-fragmentation]]
- [[stream-delta-assembly-errors]]
- [[sse-framing-errors]]
- [[empty-payload-rejections]]
- [[endpoint-rejects-request-field]]
- [[assistant-content-shape-misread]]
- [[missing-optional-fields-crash-replay]]
- [[quadratic-event-queue-drain]]
- [[private-fields-break-duck-typed-streams]]
- [[node-only-imports-break-browser-bundle]]
- [[truncated-stream-accepted-as-success]]
- [[sdk-enum-lags-provider-options]]

## Related
[[errors-as-stream-events]] · [[cross-provider-handoff]] · [[streaming-json-repair]] · [[model-catalog]] · [[http-transport-hardening]] · [[custom-provider-registration]] · [[terminal-event-required]] · [[truncated-tool-call-guard]] · [[agent-event-stream]] · [[extension-event-hooks]] · [[partial-message-persistence]]
