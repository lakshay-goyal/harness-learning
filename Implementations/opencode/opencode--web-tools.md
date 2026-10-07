---
type: implementation
harness: opencode
concept: web-tools
commit: ecc4916b5a
files: [packages/opencode/src/tool/webfetch.ts:9-11, packages/opencode/src/tool/webfetch.ts:50, packages/opencode/src/tool/webfetch.ts:67-92, packages/opencode/src/tool/webfetch.ts:97-102, packages/opencode/src/tool/websearch.ts:30-36, packages/opencode/src/tool/websearch.ts:91-95, packages/opencode/src/tool/mcp-websearch.ts:4-7, packages/opencode/src/tool/registry.ts:58-62, packages/core/src/tool/webfetch.ts:17-19, packages/core/src/tool/websearch.ts:22-24, packages/opencode/src/tool/webfetch.txt:7-13]
---
[[web-tools]] in [[opencode]].

## Mechanism
### Legacy runtime
- **webfetch**: `format` markdown (default) | text | html sets a q-weighted `Accept` header; HTML converted with Turndown (`packages/opencode/src/tool/webfetch.ts:5,183`).
- Timeout `min(param ?? 30 s, 120 s)` (`webfetch.ts:50`).
- UA impersonates Chrome 143 (`:71`); on 403 with `cf-mitigated: challenge` retried once with `User-Agent: opencode` (`:84-88`) → [[bot-detection-blocks-fetch]].
- Size cap checked on `content-length` and on the actual buffer (`:97,102`); images returned as attachments.
- Description defers to better tools ("if another tool is present that offers better web fetching capabilities… prefer using that tool") and promises HTTP→HTTPS upgrade and that "Results may be summarized if the content is very large" (`packages/opencode/src/tool/webfetch.txt:7-13`); `webfetch.ts` has no summarization code, only the size cap (description/code drift).
- **websearch**: MCP JSON-RPC to Exa (`https://mcp.exa.ai/mcp`, optional `EXA_API_KEY`) or Parallel (`https://search.parallel.ai/mcp`) (`packages/opencode/src/tool/mcp-websearch.ts:4-7`). Not a provider-native tool.
- Backend choice: `OPENCODE_WEBSEARCH_PROVIDER` override, then flags, else `checksum(sessionID) % 2` → per-session A/B (`packages/opencode/src/tool/websearch.ts:30-36`).
- Defaults `numResults || 8`, `livecrawl || "fallback"`, timeout `"25 seconds"` (`websearch.ts:78,91-95`).
- Exposure: only for providers `opencode`/`opencode-go` or when Exa/Parallel flags are set (`packages/opencode/src/tool/registry.ts:58-62,293-295`).
- Description injects `{{year}}`: "The current year is {{year}}. You MUST use this year when searching…" (`packages/opencode/src/tool/websearch.txt:13-14`).
### v2 runtime
- `packages/core/src/tool/webfetch.ts:17-19` same 5 MiB / 30 s / 120 s; `packages/core/src/tool/websearch.ts:22-24` `MAX_NUM_RESULTS = 20`, `MAX_CONTEXT_CHARACTERS = 50_000`, `MAX_RESPONSE_BYTES = 256 KiB`. Bodies read through a bounded collector with a content-length precheck (`69f1ec22e3` 2026-06-21).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_RESPONSE_SIZE` | 5 MB | `packages/opencode/src/tool/webfetch.ts:9` |
| `DEFAULT_TIMEOUT` / `MAX_TIMEOUT` | 30 s / 120 s | `packages/opencode/src/tool/webfetch.ts:10-11` |
| websearch `numResults` default | 8 | `packages/opencode/src/tool/websearch.ts:91` |
| v2 `MAX_NUM_RESULTS` | 20 | `packages/core/src/tool/websearch.ts:22` |

## Evolution
- 2025-11-10 `3f5acc3dff` web + `codesearch` (Exa Code API) tools added.
- 2025-12-11 `9c126c5b64` removed the false "self-cleaning 15-minute cache" claim from the webfetch description.
- 2025-12-29 `fd973d242e` `format` optional, markdown default.
- 2026-01-15 `1a43e5fe87` websearch description: emphasize it "ISNT 2024" → [[model-assumes-training-year]]; 2026-02-13 `179c40749d` full date reduced to year to stop daily cache busts.
- 2026-01-24 `b978ca11da` retry with simple UA on 403.
- 2026-04-29 `6aa8e894b1` "rm broken codesearch tool"; re-added by the scout branch (`40d5ea1cf1`), removed again 2026-05-12 `2481dde36d`.
- 2026-05-08 `a43d3e0e1e` Parallel provider rollout.

## Quirks / drift
- websearch description promises "Domain filtering and advanced search options available" (`websearch.txt:11`); the schema has no domain parameter → [[tool-description-drifts-from-implementation]].
- Year in the description changes once a year → small, deliberate cache-bust window ([[volatile-system-prompt-prefix]]).

Contrast: pi ships no web tools by stance → [[no-web-tools]].
