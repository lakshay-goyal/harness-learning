---
type: absence
harnesses: [pi]
---
# no-builtin-mcp-reversed

The headline reversal: MCP was absent on principle from 2025-11 to 2026-09-28, then shipped on 2026-09-29 as a *replaceable built-in extension*. It is **no longer absent** at HEAD; this note keeps the rationale and the shape of the reversal.

**What was missing (phase 1, 2025-11-12 → 2026-09-28)**
- No MCP client and no MCP config. The agent was expected to use "pre-existing CLI tools or write them on the fly".

**Evidence of the original decision**
- Pre-`60e4fcf01` README (lines removed in that diff): "You don't need MCP to extend pi's capabilities." Arguments:
  - "**Token efficient**: A 225-token README beats a 13,000-token MCP server description".
  - "**Composable**: Chain tools with bash pipes, save outputs to files, process results with code".
  - "**No overhead**: No server processes, no protocol complexity, just executables".
- `60e4fcf01` (2025-11-12, Mario Zechner): "**pi does not support MCP.** Instead, it relies on the four built-in tools above and assumes the agent can invoke pre-existing CLI tools or write them on the fly as needed." The recipe: CLI tool + README.md + "Tell the agent to read that README" (e.g. `~/agent-tools/screenshot/`).
- `3424550d2` (2025-12-17): "**No MCP.** Build CLI tools with READMEs (see Skills). The agent reads them on demand." Links the essay "What if you don't need MCP at all?" (mariozechner.at, 2025-11-02; contents unverified).
- `25cc5c7bf^:packages/coding-agent/README.md:537`: "**No MCP.** Build CLI tools with READMEs (see Skills), or build an extension that adds MCP support."
- Community adapter `pi-mcp-adapter` filled the gap (named at `packages/coding-agent/docs/mcp.md:266`).

**Phase 2: shipped (2026-09-29, v0.99.0)**
- `8562bcf66` "feat(coding-agent): codemode and MCP" (Armin Ronacher, closes #10040). Key sentence from the commit body: "The core only gets general mechanisms; codemode, tool_search and MCP are built-in extensions that use them. Other extensions can replace these built-ins."
- Registered `replaceable: true` (`packages/coding-agent/src/extensions/index.ts:9-13`). An extension registering `codemode`, `tool_search` or `/mcp` "takes over instead of running alongside" → [[replaceable-builtin-extension]].
- New `packages/mcp`: standalone client for stdio + streamable HTTP + OAuth, no official SDK dependency (`packages/mcp/README.md:1-5`). Changelog `packages/coding-agent/CHANGELOG.md:236,248`.

**How the original objection (token cost) was answered**
- Default exposure is `codemode`. MCP tools are "neither declared to the model nor listed in the codemode description"; scripts find them with `searchTools()`. Alternatives are `deferred` (via `tool_search`), `direct` and `hidden` (`docs/mcp.md:191-198`) → [[code-mode]], [[deferred-tool-loading]].
- `tool_search` ranks with BM25 ("a hybrid ranker with embeddings can replace it", `src/extensions/tool-search/tool.ts:2,34`).
- The server list lives in an `mcp_servers` system-prompt section, appended as a new message when it changes "so earlier messages stay cached" (`docs/mcp.md:202`; CHANGELOG `:197-199`, #10212). Capped at `MAX_SERVERS_SECTION_CHARS = 4096` with 250-char descriptions (`extensions/mcp/index.ts:158,163`, `e029c3ed0`) → [[cache-stable-prompt-prefix]].
- Output cap `MCP_OUTPUT_MAX_BYTES = 20 KiB`, middle cut (`extensions/mcp/tools.ts:51`). That is smaller than the built-in 50KB.

**Opt-out / opt-in today**
- `--no-mcp`, `"extensions": ["-builtin:mcp"]`, or install an extension that registers `/mcp` (`docs/mcp.md:228,266`).
- SDK sessions do not load built-in extensions. `examples/sdk/14-codemode-mcp.ts` wires MCP manually (`docs/mcp.md:272`).
- Safety: project `.pi/mcp.json` is loaded only after project trust (`docs/mcp.md:32`; `trust-manager.ts:32`) → [[project-trust-gate]]. Provider-token auth is allowed only in global config, "so a repository cannot pick where the credential goes" (`src/extensions/mcp/config.ts:25-26,133-135`).

**History**
| Date | Hash | Event |
|---|---|---|
| 2025-11-12 | `60e4fcf01` | "pi does not support MCP" |
| 2025-12-17 | `3424550d2` | Philosophy: "No MCP" |
| 2026-09-22 | `25cc5c7bf` | Philosophy section deleted, one week before MCP lands |
| 2026-09-29 | `8562bcf66` | codemode + tool_search + MCP as replaceable built-ins (#10040) |
| 2026-09-30 | `e029c3ed0` | MCP lazy connect: first prompt waits ≤10s only for direct-tool servers |
| 2026-10-02 | `de7e675de` | README stance names only sub-agents + plan mode as skipped |

**Why the reversal**: the commit gives no rationale. Inferred (unverified): codemode/deferred exposure removed the context-cost objection, and the "replaceable built-in extension" packaging keeps the core minimal in principle. The author also changed (Mario Zechner → Armin Ronacher). Issue #10040 text was not fetched.

**Implication**
- Ideological "no X" can flip once a mechanism neutralizes the original cost. Here that was declaration cost → deferred/codemode exposure.
- pi's pattern for adding a feature without "bloating core": ship it in-box on the public extension API, disable-able by name.

**opencode contrast**: implements it since `37c34fd39c` (2025-06-03, "mcp support"): stdio + StreamableHTTP/SSE + OAuth, tools declared directly as `<server>_<tool>` (code mode optional); MCP is "outside our trust boundary" (`SECURITY.md:32`) — see [[mcp-integration]] / [[mcp-builtin-vs-extension]].

Related: [[mcp-integration]] · [[code-mode]] · [[deferred-tool-loading]] · [[replaceable-builtin-extension]] · [[skill-progressive-disclosure]] · [[cache-stable-prompt-prefix]] · [[no-web-tools]] · [[Absences]]
