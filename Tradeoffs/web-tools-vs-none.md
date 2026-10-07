---
type: tradeoff
concepts: [web-tools, mcp-integration, skill-progressive-disclosure]
harnesses: [pi, opencode]
---
# web-tools-vs-none

**Axis**: does the harness ship built-in web fetch/search tools, or leave web access to bash, CLIs and MCP?

| option | pi | opencode | evidence |
|---|---|---|---|
| None built in; `curl` via bash, CLI + README skills, MCP | ✅ | — | pi: "By default, pi has no web search or fetch tool. However, it can use `curl`" (`b172beb92`); exa-search CLI example `42e0cbd4f` → [[no-web-tools]] |
| Built-in fetch with format conversion and caps | — | ✅ `webfetch` always on: markdown via Turndown, 5 MB cap, 30 s default / 120 s max timeout | `packages/opencode/src/tool/webfetch.ts:9-11,39-125` → [[web-tools]] |
| Built-in search | — | `websearch` via hosted MCP (Exa or Parallel), only for opencode providers or flags; per-session A/B by `checksum(sessionID) % 2` | `packages/opencode/src/tool/registry.ts:58-65`; `packages/opencode/src/tool/websearch.ts:27-37,66-92` |
| Browser UA spoofing | — | Chrome UA; on Cloudflare challenge retry with honest UA `opencode` | `b978ca11da` 2026-01-24 → [[bot-detection-blocks-fetch]] |
| Injection defense on fetched content | ❌ | ❌ | both → [[no-prompt-injection-defense]] |

**When each wins**
- **None (pi)**: smallest injection surface by default; token-cheap (a README beats a tool schema, per the pre-MCP argument in `60e4fcf01`); users pick their own search backend.
- **Built-in (opencode)**: documentation lookups "just work", output is size-capped and converted to markdown instead of raw HTML through bash. Costs: the largest prompt-injection channel is on by default with no marking of untrusted content, and search depends on a hosted vendor.

Related: [[no-web-tools]] · [[mcp-builtin-vs-extension]] · [[minimal-vs-rich-toolset]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
