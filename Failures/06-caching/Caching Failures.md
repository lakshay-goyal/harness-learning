---
type: group
group: 06-caching
---
Failures whose primary concept is in [[Caching]].

## Prefix stability
- [[volatile-system-prompt-prefix]] — date/time in the system prompt missed cache every reload/resume/day (3-step fix, then removed).
- [[late-tool-change-rewrites-cache]] — tool add/remove/redefine (and durable storage order) rewrote the head → full cache miss.

## Markers, retention, routing
- [[cache-breakpoints-miss-stable-segments]] — tool schemas, string user messages, OpenRouter tool results never cached.
- [[side-request-cache-pollution]] — compaction/branch summaries wrote cache under the session's identity.
- [[cache-affinity-lost-across-reopen]] — durable conversations lost provider session id on reopen/reset/compaction.

## Keep-alive
- [[late-cache-warming-is-a-full-write]] — delayed warm timers refreshed already-expired caches at full write price.

## See also (primary concept in other groups)
- [[mcp-startup-blocks-and-description-churn]] — meta-tool descriptions changed as MCP servers connected — [[Tools Failures]].
- [[server-limited-identifier-rejected]] — cache key/session-id length & header-shape rejections — [[Model Interface Failures]].
- [[capability-sniffing-misses-opaque-ids]] — Bedrock cache points on non-Claude models / missed on ARNs — [[Model Interface Failures]].
- [[connection-cache-shared-across-accounts]], [[persistent-connection-lifetime-exceeded]], [[transport-fallback-after-partial-output]] — Codex WS server-side session cache lifecycle — [[Model Interface Failures]].
- [[stale-thinking-signature-after-prefix-change]] — signed reasoning bound to old prompt/tools — [[Model Interface Failures]].
- [[usage-double-counting]], [[usage-priced-at-wrong-rate]] — cache read/write accounting & 1h write pricing — [[Model Interface Failures]].
- [[forced-system-prompt-applied-as-late-update]] — [[Prompting Failures]].
