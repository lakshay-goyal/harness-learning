---
type: group
group: 02-model-interface
---
Scope: the boundary between the agent and model providers:
- the neutral streaming API;
- turning provider failures into stream results and classifying them;
- history handoff and replay repair;
- reasoning controls and replaying reasoning;
- the model catalog and how models are selected;
- credentials and subscription login;
- the transport layer;
- operations other than chat.

## Concepts
- [[unified-provider-api]] — One provider-neutral streaming API and event protocol (start/delta/end/done/error with a live partial message) over many vendor APIs.
- [[errors-as-stream-events]] — Once a stream is returned, failures never throw. The stream ends with an error message that carries the partial content and usage.
- [[cross-provider-handoff]] — Rewrite stored history so a different provider or model accepts it mid-session.
- [[transcript-replay-repair]] — Normalize replayed history at the provider boundary:
  - drop failed or aborted turns,
  - synthesize results for orphaned tool calls,
  - keep adjacency rules.
- [[tool-call-id-normalization]] — Rewrite or synthesize tool-call ids to meet the target provider's character set, length and uniqueness rules, keeping each call paired with its result.
- [[signed-reasoning-replay]] — Replay provider-signed or encrypted reasoning verbatim to the same model; drop or convert it for others; handle stale signatures.
- [[thinking-level-abstraction]] — A provider-neutral reasoning-effort scale, mapped per provider and model (effort vs token budget), clamped, with a reserve for answer tokens.
- [[context-overflow-detection]] — Classify as context overflow:
  - provider errors, via a regex catalogue with exclusions;
  - silent overflow (usage larger than the window);
  - recoverable length stops.
- [[max-tokens-context-clamp]] — Clamp requested output tokens to the window minus estimated input minus a safety margin.
- [[usage-cost-accounting]] — Normalize heterogeneous usage (input/output/cacheRead/cacheWrite/reasoning) and price each message, including tiers.
- [[model-catalog]] — Typed model metadata generated at build time (context window, cost, capabilities, compat flags), plus a remote overlay and user/extension layers.
- [[model-resolution]] — Turn a user string (provider/id, fuzzy, glob, :level) into a concrete model; startup selection cascade; scoped cycling.
- [[virtual-model-router]] — A selectable pseudo-model that dispatches each request to a physical model and effort.
- [[custom-provider-registration]] — A plugin API to add or override providers, OAuth, discovery and streaming at runtime.
- [[subscription-oauth-auth]] — Log in with a consumer subscription (PKCE loopback or device code) and use the token as an API credential, with locked refresh.
- [[provider-identity-shim]] — A provider-mandated prompt prefix and tool renaming that impersonate the vendor's own client for subscription tokens.
- [[credential-resolution]] — Precedence among runtime flag, stored credential, config value (literal/$ENV/!command) and env/ambient chains; a stored credential owns its provider.
- [[constrained-tool-sampling]] — Provider-side strict JSON schema or grammar enforcement for tool arguments, with a fallback when unsupported.
- [[streaming-json-repair]] — Tolerant parsing of partial or malformed tool-argument JSON while streaming.
- [[unicode-sanitization]] — Strip unpaired UTF-16 surrogates and invalid text before provider serialization.
- [[deferred-responses]] — Background or async provider generations persisted as a handle plus pollAt, surviving restarts.
- [[structured-classifier-api]] — A typed classify() operation (choice/score/bool) distinct from chat; logprob label classification.
- [[http-transport-hardening]] — HTTP/WS transport policy owned by the harness:
  - proxies and idle timeouts,
  - connection max-age,
  - sticky WS→SSE fallback,
  - body compression,
  - header rules.
- [[server-side-refusal-fallback]] — The provider retries refused requests on a fallback model approved in advance.

Failures: [[Model Interface Failures]]
