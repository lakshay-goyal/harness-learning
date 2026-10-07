---
type: group
group: 06-caching
---
Failures whose primary concept is in [[Caching]].

## Prefix stability
- [[volatile-system-prompt-prefix]] — date/time in the system prompt missed cache every reload/resume/day (3-step fix, then removed).
- [[late-tool-change-rewrites-cache]] — tool add/remove/redefine (and durable storage order) rewrote the head → full cache miss.
- [[nondeterministic-tool-order-busts-cache]] — codex: hash-map MCP tool order changed between turns and broke prompt caching.
- [[permission-context-reinjected-repeatedly]] — codex: the full permissions block was re-appended after every approval; per-environment AGENTS.md budgets grew with environment count.
- [[ephemeral-history-rewrite-busts-cache]] — request-only wrapper on queued user messages changed already-sent bytes (opencode).

## Markers, retention, routing
- [[cache-breakpoints-miss-stable-segments]] — tool schemas, string user messages, OpenRouter tool results never cached.
- [[cache-marker-namespace-mismatch]] — markers under the wrong SDK key/level silently disabled caching (opencode).
- [[side-request-cache-pollution]] — compaction/branch summaries wrote cache under the session's identity.
- [[cache-affinity-lost-across-reopen]] — durable conversations lost provider session id on reopen/reset/compaction.
- [[cache-key-scoped-to-wrong-identity]] — codex: a thread-id cache key split the root and its subagents across shards; ephemeral forks lost the parent's `session-id` header affinity.
- [[incremental-request-diverges-from-history]] — codex: WebSocket delta requests reused `previous_response_id` after a mid-turn tool change, and resent output items.

## Keep-alive
- [[late-cache-warming-is-a-full-write]] — delayed warm timers refreshed already-expired caches at full write price.

## See also (primary concept in other groups)
- [[mcp-startup-blocks-and-description-churn]] — meta-tool descriptions changed as MCP servers connected — [[Tools Failures]].
- [[server-limited-identifier-rejected]] — cache key/session-id length & header-shape rejections — [[Model Interface Failures]].
- [[capability-sniffing-misses-opaque-ids]] — Bedrock cache points on non-Claude models / missed on ARNs — [[Model Interface Failures]].
- [[connection-cache-shared-across-accounts]], [[persistent-connection-lifetime-exceeded]], [[transport-fallback-after-partial-output]], [[server-side-state-missing-on-continuation]] — Codex WS server-side session cache lifecycle — [[Model Interface Failures]].
- [[prewarm-blocks-turn-start]] — codex WebSocket warmup gated turn start — [[Loop Failures]].
- [[stale-context-files-mid-session]] — appended AGENTS.md replacement notices — [[Prompting Failures]].
- [[stale-thinking-signature-after-prefix-change]] — signed reasoning bound to old prompt/tools — [[Model Interface Failures]].
- [[usage-double-counting]], [[usage-priced-at-wrong-rate]] — cache read/write accounting & 1h write pricing — [[Model Interface Failures]].
- [[forced-system-prompt-applied-as-late-update]] — [[Prompting Failures]].
