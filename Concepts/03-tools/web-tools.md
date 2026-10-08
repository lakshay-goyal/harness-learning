---
type: concept
stage: tools
tier: must-have
aliases: [webfetch, websearch, codesearch, Exa, Parallel search, TurndownService, cf-mitigated, web_search, "web.run", WebSearchMode, "Cached/Indexed/Live/Disabled", hosted web search, standalone web search, external_web_access, web-search-tool]
harnesses: [opencode, codex]
---
Built-in fetch and search tools that pull web content into context, with format conversion and size limits.

## Why
- Without them the model shells out to `curl` and dumps raw HTML into context, or guesses library APIs from stale training data.
- Search queries are formed from the model's training-era sense of "now" unless the harness supplies the date ([[model-assumes-training-year]]).
- Sites block non-browser clients; the fetch path needs a policy for bot detection ([[bot-detection-blocks-fetch]]).
- Coding tasks need current docs, release notes and error lookups that the model's training data lacks.
- Live internet access is an exfiltration/injection channel, so admins need a cached or disabled mode and a policy ceiling ([[egress-policy-proxy]], [[no-prompt-injection-defense]]).
- Hosted tools only exist on some providers/APIs (not on codex "Responses Lite"), so a client-executed fallback is needed for portability.

## Design space
- **Fetch**: URL → content-negotiated markdown/text/html, HTML→markdown conversion, byte cap, timeout cap, images as attachments (opencode).
- User-Agent: browser impersonation with honest-UA retry on a Cloudflare challenge (opencode legacy) vs honest UA only.
- **Search backend**: hosted MCP search endpoints (Exa, Parallel) called as JSON-RPC (opencode) vs provider-native search tools vs none.
- Rollout: deterministic per-session A/B between backends by session-id hash (opencode).
- Gating: only on the harness's own provider or behind env flags (opencode) vs always on.
- Code-specific search (Exa Code API, opencode `codesearch`; removed as broken).
- None; use shell + skills (pi, [[no-web-tools]]).
- **None; use shell `curl` or MCP** (pi: [[no-web-tools]]).
- **Provider-hosted search tool** (codex `web_search`, emitted when the provider advertises it) vs **client-executed search tool proxied to a backend** (codex `web.run` extension) — never both (codex: standalone replaces hosted).
- Access modes: cached index only / indexed-live / live / disabled (codex `WebSearchMode`, default Cached), with requirement allow-lists (codex `allowed_web_search_modes`, [[layered-settings]]).
- Result types: text vs text + image (codex per-model `web_search_tool_type`).
- Command set of a client tool: search only vs browse (`open`/`click`/`find`/`screenshot`) + verticals (finance/weather/sports/time) (codex `web.run`).
- Disabled for side agents (codex review sub-agent runs with web search off → [[review-subagent]]).
- Generic URL fetch tool: absent in codex (only an `open_page` search action; unverified beyond grep, M8).
- (folded from `web-search-tool`, codex framing) Internet search exposed to the model either as a provider-hosted tool (executed server-side during sampling) or as a client-executed tool that the harness proxies to a search backend — with an access mode (cached index vs live fetch) chosen by config/policy.

## Implementations
- [[opencode--web-tools|opencode]] — `webfetch` (5 MB, 30 s/120 s, Turndown, Chrome UA → `opencode` UA on `cf-mitigated`) + `websearch` over Exa/Parallel MCP, provider/flag-gated.
- [[codex--web-tools|codex]] — hosted Responses `web_search` (mode → `external_web_access`/`indexed_web_access`) or `web.run` standalone extension (7.5 KB description), selected by provider capability + `Feature::StandaloneWebSearch` / Responses Lite.

## Failures
- [[bot-detection-blocks-fetch]]
- [[model-assumes-training-year]]
- [[tool-description-drifts-from-implementation]]
- (02) [[strict-tool-schema-rejections]] (schema compaction stripped `web.run` field guidance, `9fe55d68e6`)

## Tradeoffs
- [[web-tools-vs-none]]

## Related
[[no-web-tools]] · [[mcp-integration]] · [[tool-description-design]] · [[tool-output-truncation]] · [[image-normalization]] · [[cache-stable-prompt-prefix]]
[[no-web-tools]] · [[tool-wire-kinds]] · [[tool-schema-lowering]] · [[egress-policy-proxy]] · [[replaceable-builtin-extension]] · [[minimal-default-toolset]] · [[model-catalog]]
