---
type: group
group: 06-caching
---
Scope: provider prompt-cache economics — where cache markers go, how long entries live, keeping the cached prefix byte-stable while prompts/tools change, routing requests to the warm replica, keeping caches alive, and detecting when a turn re-billed what should have been a cache read.

## Concepts
- [[cache-breakpoint-placement]] — Where explicit cache markers go (system, last tool, last message) per provider.
- [[cache-retention-control]] — Neutral none/short/long retention mapped to provider TTL fields; no cache writes for one-off side requests.
- [[cache-stable-prompt-prefix]] — Keep volatile data (date, counts, server lists) out of system prompt and tool descriptions; preseed placeholders.
- [[transcript-carried-system-prompt]] — System prompt sections and tool declarations stored as transcript deltas and replayed, so changes append instead of rewriting the cached prefix.
- [[cache-preserving-config-update]] — Pin request-level sampling parameters (e.g. reasoning effort) per context window, and append a harness-authored configuration item to history when the user changes them.
- [[cache-warming]] — Cost-aware keep-alive replay of last request (1 output token) before TTL expiry.
- [[cache-miss-accounting]] — Per-turn detection of prompt tokens re-billed instead of cache-read.
- [[session-affinity-cache-routing]] — Send stable session id as cache key / affinity header so requests hit the same cached replica.

Failures: [[Caching Failures]] · Neighbors: [[Prompting]] (what goes in the prefix), [[Model Interface]] (usage/cost normalization, transports), [[Context]] (compaction resets the prefix).
