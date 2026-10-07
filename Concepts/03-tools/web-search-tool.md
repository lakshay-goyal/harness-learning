---
type: concept
stage: tools
tier: candidate
aliases: [web_search, "web.run", WebSearchMode, "Cached/Indexed/Live/Disabled", hosted web search, standalone web search, external_web_access]
harnesses: [codex]
---
Internet search exposed to the model either as a provider-hosted tool (executed server-side during sampling) or as a client-executed tool that the harness proxies to a search backend — with an access mode (cached index vs live fetch) chosen by config/policy.

## Why
- Coding tasks need current docs, release notes and error lookups that the model's training data lacks.
- Live internet access is an exfiltration/injection channel, so admins need a cached or disabled mode and a policy ceiling ([[egress-policy-proxy]], [[no-prompt-injection-defense]]).
- Hosted tools only exist on some providers/APIs (not on codex "Responses Lite"), so a client-executed fallback is needed for portability.

## Design space
- **None; use shell `curl` or MCP** (pi: [[no-web-tools]]).
- **Provider-hosted search tool** (codex `web_search`, emitted when the provider advertises it) vs **client-executed search tool proxied to a backend** (codex `web.run` extension) — never both (codex: standalone replaces hosted).
- Access modes: cached index only / indexed-live / live / disabled (codex `WebSearchMode`, default Cached), with requirement allow-lists (codex `allowed_web_search_modes`, [[layered-settings]]).
- Result types: text vs text + image (codex per-model `web_search_tool_type`).
- Command set of a client tool: search only vs browse (`open`/`click`/`find`/`screenshot`) + verticals (finance/weather/sports/time) (codex `web.run`).
- Disabled for side agents (codex review sub-agent runs with web search off → [[review-subagent]]).
- Generic URL fetch tool: absent in codex (only an `open_page` search action; unverified beyond grep, M8).

## Implementations
- [[codex--web-search-tool|codex]] — hosted Responses `web_search` (mode → `external_web_access`/`indexed_web_access`) or `web.run` standalone extension (7.5 KB description), selected by provider capability + `Feature::StandaloneWebSearch` / Responses Lite.

## Failures
- (02) [[strict-tool-schema-rejections]] (schema compaction stripped `web.run` field guidance, `9fe55d68e6`)

## Related
[[no-web-tools]] · [[tool-wire-kinds]] · [[tool-schema-normalization]] · [[egress-policy-proxy]] · [[replaceable-builtin-extension]] · [[minimal-default-toolset]] · [[model-catalog]]
