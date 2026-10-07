---
type: tradeoff
concepts: [cache-breakpoint-placement, cache-retention-control, cache-stable-prompt-prefix, transcript-carried-system-prompt, cache-warming, cache-miss-accounting, session-affinity-cache-routing, cache-preserving-config-update]
---
**Axis:** where the harness invests to make prompt caching work.
- **Explicit cache control:** markers, retention, keep-alive economics and miss detection, across many providers.
- **Implicit vendor caching:** byte-determinism, affinity keys, server-side session state, and in-band settings changes, for one vendor's automatic prefix cache.

| dimension | pi | codex |
|---|---|---|
| cache model | explicit breakpoints per adapter: system, last tool, last message ([[pi--cache-breakpoint-placement]]) plus implicit for OpenAI-style | implicit OpenAI prefix caching only; no `cache_control` in `codex-rs` ([[cache-breakpoint-placement]]) |
| retention | neutral `none/short/long` knob mapped to provider TTLs; `none` for summaries ([[pi--cache-retention-control]]) | none; there is no retention field in `ResponsesApiRequest` (`codex-rs/codex-api/src/common.rs:279-304`) |
| key / affinity | session id as `prompt_cache_key` plus per-provider headers; **fresh** id for side requests and forks ([[pi--session-affinity-cache-routing]]) | session id **shared** across root, subagents and ephemeral forks; `session-id` header for ChatGPT routing; per-turn `x-codex-turn-state` sticky token ([[codex--session-affinity-cache-routing]]; [[cache-key-scoped-to-wrong-identity]]) |
| prefix stability | remove volatile facts, placeholder tool for Anthropic scaffolding, byte-stable meta-tool descriptions ([[pi--cache-stable-prompt-prefix]]) | deterministic regeneration: UUIDv5 ids for prefix items, sorted tool lists, date in a user-role context message ([[codex--cache-stable-prompt-prefix]]) |
| prompt and setting changes | transcript-carried system message with section patches and tool deltas ([[pi--transcript-carried-system-prompt]]) | typed world-state section diffs ([[world-state-diff-injection]]); Lite incremental tool catalog; request effort pinned per window plus an appended `ConfigurationUpdate` ([[codex--cache-preserving-config-update]]) |
| server-side state | client of the Codex WebSocket: delta plus `previous_response_id`, 55 min rotation | reference implementation: exhaustive request-equality check, strict prefix extension, full-resend fallback ([[incremental-request-diverges-from-history]], [[server-side-state-missing-on-continuation]]) |
| keep-alive | EV-gated TTL refresh with a 1-token output; Anthropic direct only ([[pi--cache-warming]]) | `generate=false` prewarm at turn or thread start, optionally with history; latency-oriented, no TTL model ([[codex--cache-warming]]) |
| detection | per-turn miss accounting with a noise floor and idle attribution ([[pi--cache-miss-accounting]]) | none found (grep-based): cached tokens go to OTEL only (`codex-rs/otel/src/events/session_telemetry.rs:1139`) ([[cache-miss-accounting]]) |
| compaction and cache | local summary; side requests opt out of caching | compaction reuses one model session, so sticky routing and WebSocket state survive its retries (`codex-rs/core/src/compact.rs:282-284`); remote v2 compaction is a normal streamed `/responses` request (`codex-rs/core/src/compact_remote_v2_attempt.rs:84-103`); local overflow trimming drops the *oldest* item "to preserve cache" (`codex-rs/core/src/compact.rs:330-341`) |

**When each wins**
- **Explicit control (pi):**
  - many providers with different cache semantics (Anthropic markers and TTL write premiums, Bedrock cache points);
  - money visible per message;
  - side requests that would pollute the cache.

  Breakpoints, retention and warming are worth it when writes are billed at a premium and TTLs are short.
- **Implicit determinism (codex):**
  - one vendor with automatic prefix caching and no write premium in the client's view (pricing is server-side);
  - control over the server protocol.

  Investment goes into never changing the prefix (stable ids, sorted tools, pinned parameters) and into routing (cache key granularity, sticky turn state, WebSocket deltas). Detection matters less when prevention is structural, and the vendor sees hit rates server-side.
- **Shared lesson.** Anything serialized into the prefix needs deterministic order and content ([[nondeterministic-tool-order-breaks-cache]], [[volatile-system-prompt-prefix]], [[late-tool-change-rewrites-cache]]). Changes should be appended, not rewritten ([[transcript-carried-system-prompt]]).
- **Divergence on side requests:**
  - pi isolates them, with a fresh id and no cache ([[side-request-cache-pollution]]).
  - codex shares the family key, because its side requests (subagents, forks, compaction) reuse the parent's prefix.

  Both are right for their prefix topology.

Related: [[provider-breadth]] · [[compaction-locus]] · [[session-affinity-cache-routing]] · [[cache-stable-prompt-prefix]] · [[cache-preserving-config-update]]
