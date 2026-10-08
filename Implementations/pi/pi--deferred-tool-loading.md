---
type: implementation
harness: pi
concept: deferred-tool-loading
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/types.ts:504-516, packages/coding-agent/src/core/extensions/types.ts:546-568, packages/coding-agent/src/extensions/tool-search/tool.ts:1-246, packages/coding-agent/src/extensions/tool-search/index.ts:14, packages/coding-agent/src/core/agent-session.ts:1555-1575, packages/coding-agent/src/extensions/mcp/tools.ts:40-46, packages/coding-agent/src/extensions/codemode/tool.ts:156-247]
---
[[deferred-tool-loading]] in [[pi]].

## Mechanism
- **Five exposure tiers** — `ToolExposure = "direct" | "model-only" | "codemode" | "deferred" | "hidden"` (`packages/coding-agent/src/core/extensions/types.ts:516`), doc-comment semantics (`types.ts:504-515`):
  - `direct`: declared while active, callable while active.
  - `model-only`: declared while active, **never callable** by scripts/nested calls — "Use it for orchestrating or interactive tools" (codemode and `tool_search` themselves).
  - `codemode`: callable whenever registered; not declared unless explicitly activated; codemode description lists it.
  - `deferred`: like `codemode` but codemode does **not** list it; `tool_search` can find it.
  - `hidden`: registered but unreachable; activating has no effect (how pi "unregisters" — tools can't be removed, `packages/coding-agent/docs/extensions.md:160`).
  - `direct`/`model-only` auto-activate on registration; others don't. **Active set == declared set** (`types.ts:513-514`).
- Callable set for nested calls = active `direct` + every registered `codemode`/`deferred` tool (`packages/coding-agent/src/core/agent-session.ts:1557-1566`) — see [[nested-tool-calls]].
- **`prepareLoadout(loadout) → {descriptions, hiddenDeclarations}`** (`types.ts:546-568,624`): an active orchestrating tool sees `declared/callable/registered` + `getExposure/getNamespace/getPromptGuidelines` and may rewrite declared tools' descriptions or hide declarations. Hidden declarations "stay active and callable, and the transcript still declares them, so the active set survives `/tree` and resume" (`types.ts:563-567`). Request-side projection strips them in `transformContext` (`agent-session.ts:1766-1784`, per 01-loop findings). Hidden tools also excluded from `<tools>`/`<rules>` (`028c0ec56` 2026-09-30, `c30840c2e` 2026-10-05) — see [[dynamic-tool-guidelines]].
- **`tool_search` tool** (`packages/coding-agent/src/extensions/tool-search/tool.ts`):
  - Schema: `query: string` ("Search query for deferred tools."), `limit?` default `DEFAULT_TOOL_SEARCH_LIMIT = 8` (`tool.ts:21,159-164`); empty query / non-positive-integer limit throw (`tool.ts:234-236`).
  - Description constant `TOOL_SEARCH_DESCRIPTION` (`tool.ts:220`): "# Tool discovery … Searches over deferred tool metadata with BM25 and exposes matching tools for the next model call. … Some of the tools, such as tools of MCP servers, may not have been provided to you upfront, and you should use this tool (`tool_search`) to search for the required tools. For MCP tool discovery, always use `tool_search`." Deliberately lists **no** tools/namespaces "so it stays the same while tools are registered, for example when MCP servers connect" (`tool.ts:216-219`).
  - promptSnippet "Search for tools that are not loaded yet and load the matches"; `exposure: "model-only"` — "Searching is not something scripts need; it changes what the model sees" (`tool.ts:226-232`).
  - `searchAndLoad` (`tool.ts:200-214`): candidates = registered tools with `codemode|deferred` exposure not already active → `Bm25Ranker.rank` → `setActiveTools([...active, ...matches])` → declared from the **next** model call. Result text `Loaded N tools. They are available from your next call:\n- name: <first line of description>`; `details.loaded` (`tool.ts:238-244`); none → "No matching tools found.".
  - Loads go through the active set, so they are **recorded in the transcript as tool changes** and survive `/tree`, resume, fork on that branch (`tool.ts:1-9`; `docs/mcp.md:226`) — via `SystemMessage.toolsAdded/Removed` ([[transcript-carried-system-prompt]]).
  - Off by default: registered `defaultActive:false` (`tool-search/index.ts:14`); enable `"defaultTools": ["+tool_search"]` (`docs/cli.md:182`) or auto-activated by MCP `deferred` exposure ([[mcp-integration]]).
- **BM25 ranker** (`tool.ts:119-157`): Okapi BM25, `k1=1.2`, `b=0.75`, idf `log(1+(N-df+0.5)/(df+0.5))`; query terms deduped; ties keep document order; score>0 only. Tokenizer (`tool.ts:39-80`): splits camelCase and non-alnum, lowercases, removes 21 stop words, naive plural stemming. Interface `ToolRanker` — "BM25 today; a hybrid ranker with embeddings can replace it" (`tool.ts:34-37`).
- **Search document** (`createToolSearchDocument`, `tool.ts:86-116`): name, name with `_`→space, description, schema descriptions + property names (recursive into `properties`, `items`, `anyOf/oneOf/allOf`), namespace name/description/instructions.
- Same ranker powers codemode's `searchTools()` (`tool.ts:1-3`; [[code-mode]]).
- **MCP mapping**: MCP `codemode` exposure maps to ToolExposure `deferred` — "`codemode` and `deferred` both leave tools out of the codemode description; they differ only in which tool the MCP extension activates" (`packages/coding-agent/src/extensions/mcp/tools.ts:40-46`).
- **Catalog budget (codemode as the other discovery surface)**: codemode description lists callable non-deferred tools within `DEFAULT_CODEMODE_INLINE_BUDGET = 3000` est. tokens (chars/4) (`codemode/tool.ts:156-158`); `selectCatalog` round-robin "like OpenCode's catalog": each round every group (no-namespace first, then namespaces by name) places its cheapest remaining tool; a group whose next tool doesn't fit drops out — "Every namespace is represented before any namespace is complete" (`codemode/tool.ts:216-239`). Deferred tools never listed and "do not affect the description at all" (`codemode/tool.ts:241-247`). Listing by exposure, not active set, keeps codemode's description unchanged when `tool_search` loads a tool (`codemode/tool.ts:339-341`).
- Volatile server data lives in the patchable `mcp_servers` system-prompt section instead of tool descriptions ([[cache-stable-prompt-prefix]], CHANGELOG `:198` #10212).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_TOOL_SEARCH_LIMIT` | 8 | `packages/coding-agent/src/extensions/tool-search/tool.ts:21` |
| BM25 `k1` / `b` | 1.2 / 0.75 | `tool-search/tool.ts:124-125` |
| stop words | 21 | `tool-search/tool.ts:39-61` |
| `DEFAULT_CODEMODE_INLINE_BUDGET` | 3000 est. tokens | `packages/coding-agent/src/extensions/codemode/tool.ts:156` |
| `CHARS_PER_TOKEN` (catalog cost) | 4 | `codemode/tool.ts:158` |

## Evolution
- 2026-07-10 `3d8f74357` (#6474) — predecessor `packages/ai/src/utils/deferred-tools.ts` (historical: added here, deleted in `9e05370b2`): "message-anchored tool loading" via `addedToolNames` on tool results so Anthropic/OpenAI Responses caching survived new tools.
- 2026-09-16 `9e05370b2` (#9548) — deferred-tools.ts deleted; system prompt text and tool changes become first-class transcript entries (`SystemMessage.toolsAdded/Removed`), making "load tool" just an active-set change.
- 2026-09-29 `8562bcf66` — exposure tiers, `tool_search` and codemode land as general core mechanisms + replaceable built-in extensions ([[replaceable-builtin-extension]]).
- 2026-09-30 `e029c3ed0` / `1c7e7df76` (#10212) — stop waiting for MCP servers on first prompt; codemode lists servers not tools; descriptions no longer include deferred tools/tool counts/MCP instructions "so it no longer changes when MCP servers connect".
- 2026-09-30 `028c0ec56` (#10192) — codemode-hidden tools not listed in system prompt; 2026-10-05 `c30840c2e` (#10343) hidden tools out of rules/skills hint.
- 2026-10-01 `c662ec7e3` — tools loaded by `tool_search` restored on resume/reload: pending restored names activate when their tool registers (MCP servers reconnect later).

## Evidence commits
`3d8f74357`, `9e05370b2`, `8562bcf66`, `e029c3ed0`, `1c7e7df76`, `028c0ec56`, `c30840c2e`, `c662ec7e3`, `fd6659dd5` (#6162 apply tool changes before next request), `bc2fa8d6d` (#1720 dynamic registration refresh).

## Quirks
- Loads are additive; there is no "unload" path in `tool_search` — active set only grows until user/extension `setActiveTools` (observed in `searchAndLoad`, `tool.ts:200-214`).
- Search excludes already-active tools, so re-searching never returns them (`tool.ts:208`).
- Tokenizer is lexical; synonyms miss (`unverified` real-world recall).
- Loading a tool changes the declared set → new tool declaration appended as transcript delta; on Anthropic needs inline-tools beta / placeholder tool to avoid a full cache miss ([[late-tool-change-rewrites-cache]]).

## Failures
- [[deferred-tools-lost-on-resume]]
- [[mcp-startup-blocks-and-description-churn]]
- [[late-tool-change-rewrites-cache]]
- [[prompt-names-unavailable-tools]]
