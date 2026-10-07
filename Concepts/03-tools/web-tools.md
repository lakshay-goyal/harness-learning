---
type: concept
stage: tools
tier: candidate
aliases: [webfetch, websearch, codesearch, Exa, Parallel search, TurndownService, cf-mitigated]
harnesses: [opencode]
---
Built-in fetch and search tools that pull web content into context, with format conversion and size limits.

## Why
- Without them the model shells out to `curl` and dumps raw HTML into context, or guesses library APIs from stale training data.
- Search queries are formed from the model's training-era sense of "now" unless the harness supplies the date ([[model-assumes-training-year]]).
- Sites block non-browser clients; the fetch path needs a policy for bot detection ([[bot-detection-blocks-fetch]]).

## Design space
- **Fetch**: URL → content-negotiated markdown/text/html, HTML→markdown conversion, byte cap, timeout cap, images as attachments (opencode).
- User-Agent: browser impersonation with honest-UA retry on a Cloudflare challenge (opencode legacy) vs honest UA only.
- **Search backend**: hosted MCP search endpoints (Exa, Parallel) called as JSON-RPC (opencode) vs provider-native search tools vs none.
- Rollout: deterministic per-session A/B between backends by session-id hash (opencode).
- Gating: only on the harness's own provider or behind env flags (opencode) vs always on.
- Code-specific search (Exa Code API, opencode `codesearch`; removed as broken).
- None; use shell + skills (pi, [[no-web-tools]]).

## Implementations
- [[opencode--web-tools|opencode]] — `webfetch` (5 MB, 30 s/120 s, Turndown, Chrome UA → `opencode` UA on `cf-mitigated`) + `websearch` over Exa/Parallel MCP, provider/flag-gated.

## Failures
- [[bot-detection-blocks-fetch]]
- [[model-assumes-training-year]]
- [[tool-description-drifts-from-implementation]]

## Tradeoffs
- [[web-tools-vs-none]]

## Related
[[no-web-tools]] · [[mcp-integration]] · [[tool-description-design]] · [[tool-output-truncation]] · [[image-normalization]] · [[cache-stable-prompt-prefix]]
