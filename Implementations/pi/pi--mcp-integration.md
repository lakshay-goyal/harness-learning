---
type: implementation
harness: pi
concept: mcp-integration
commit: b30a6dd77
files: [packages/coding-agent/src/extensions/mcp/index.ts:1-27, packages/coding-agent/src/extensions/mcp/index.ts:93, packages/coding-agent/src/extensions/mcp/index.ts:156-231, packages/coding-agent/src/extensions/mcp/index.ts:405-519, packages/coding-agent/src/extensions/mcp/tools.ts:40-254, packages/coding-agent/src/extensions/mcp/runtime.ts:48-51, packages/coding-agent/src/extensions/mcp/config.ts:25-26, packages/coding-agent/src/core/mcp-servers.ts:12-76, packages/mcp/src/protocol/types.ts:4-9, packages/mcp/src/transports/stdio.ts:7-11, packages/mcp/src/transports/streamable-http.ts:166-168, packages/mcp/src/client.ts:38, packages/coding-agent/docs/mcp.md]
---
[[mcp-integration]] in [[pi]].

## Mechanism
- **Packaging**: standalone client `packages/mcp` (no official SDK dependency, `packages/mcp/README.md:1-5`) + built-in extension `packages/coding-agent/src/extensions/mcp/*` (`CA` = `packages/coding-agent/src`) + `packages/coding-agent/src/core/mcp-servers.ts`; `replaceable: true` — an extension registering `/mcp` takes over (`packages/coding-agent/src/extensions/index.ts:7-14`; [[replaceable-builtin-extension]]). Opt-out `--no-mcp`, `"extensions": ["-builtin:mcp"]` (`docs/mcp.md:228,266`). SDK sessions don't load built-ins; `examples/sdk/14-codemode-mcp.ts` wires them manually (`docs/mcp.md:272`).
- **Config**: `~/.pi/agent/mcp.json` + project `.pi/mcp.json` — project file only after [[project-trust-gate]]; same-name project entry replaces user entry; entry without command/url/type only overrides `enabled/exposure/toolExposure` (`docs/mcp.md:30-42`; `1387af7b4`). `env`/`headers` support `${VAR}` and whole-value `!command`. SSE transport rejected (`docs/mcp.md:64,78-79`). Provider-token auth (`"auth": {"provider": …}`) only in global config "so a repository cannot pick where the credential goes" (`packages/coding-agent/src/extensions/mcp/config.ts:25-26,133-135`). Extensions can `registerMcpServer/unregisterMcpServer` (`mcp_servers_change` event); `mcp.json` wins over a registered server of the same name (`mcp/index.ts:4-6`).
- **Exposure** (`packages/coding-agent/src/core/mcp-servers.ts:43-57`), default `codemode` (`mcp/index.ts:135-137`):
  - `codemode`: "callable from codemode scripts but neither declared to the model nor listed in the codemode description, which lists only the server's namespace. Scripts find them with `searchTools()`." Alias `codemode-deferred`.
  - `deferred`: not declared until `tool_search` loads them; model then calls directly.
  - `direct`: declared like built-ins. `hidden`: unreachable.
  - `toolExposure` per-tool overrides with `*` patterns (exact wins, else first matching pattern); `"exposure":"hidden"` + a few overrides = allowlist (`mcp-servers.ts:70-73`). Pattern matcher `*`→`.*` anchored (`mcp-servers.ts:12-25`).
  - Internally MCP `codemode` → ToolExposure `deferred` (`mcp/tools.ts:40-46`) — see [[pi--deferred-tool-loading|pi deferred-tool-loading]].
- **Naming** (`createMcpToolName`, `mcp/tools.ts:78-93`): `mcp__<server>__<tool>`, every non-`[A-Za-z0-9_]` → `_` "Like Codex … so the name is also the identifier codemode scripts call it by"; max 64 chars ("Provider tool names are limited to 64 characters", `tools.ts:48-49`); sha256(`server\0tool`)[0:8] suffix when too long or taken; all colliding names get the suffix so assignment is order-independent (`b29db895c`).
- **Background connect** (`mcp/index.ts:1-14,1122-1145`): all enabled servers connect in background at session start; first prompt waits up to `DEFAULT_STARTUP_WAIT_MS = 10_000` (`index.ts:93`) **only for servers with `direct` tools**; codemode scripts wait only for servers they name — `scriptNeedsServer` regex: names the namespace, or uses `searchTools|describeNamespace|describeTool|ALL_TOOLS` → wait for all (`index.ts:227-231`); `tool_search` and resource tools wait for all (`e029c3ed0`). Runtime module loads lazily (`mcp/runtime.lazy.ts`). HTTP connect retries `CONNECT_RETRY_DELAYS_MS = [250, 1000]` for URL servers only (`runtime.ts:51,362`). `registerToolRenderer` lets MCP calls render before their server connects (`11449730c`).
- **Discovery activation** (`ensureDiscoveryActive`, `index.ts:481-519`): decided from **config** before connect — activate `codemode` if any `codemode` exposure and `autoEnableCodemode !== false`; activate `tool_search` for `deferred`; warn once otherwise: "MCP tools are only reachable from the codemode or tool_search tool, but neither is active[ (autoEnableCodemode is false)]; they cannot be called." (`index.ts:514-516`).
- **`mcp_servers` system-prompt section** (`index.ts:156-225`): intro "MCP servers whose tools are not declared to you." + " Call the tools of `codemode` servers from codemode scripts." / " Load the tools of `tool_search` servers with `tool_search`."; per line `- mcp__name (codemode|tool_search): <first line of description or server instructions>`; `MAX_SERVER_DESCRIPTION_CHARS = 250` ("as Codex allows for deferred namespaces"), whole section `MAX_SERVERS_SECTION_CHARS = 4096`, trailing servers dropped with "- … N more servers; find their tools with searchTools()" (`index.ts:158,163,205-219`). Recomputed in `before_agent_start` → `systemPromptOptions.sections` (`index.ts:1142-1148`) → appended as a section patch "so earlier messages stay cached" (`docs/mcp.md:202`) — [[transcript-carried-system-prompt]], [[cache-stable-prompt-prefix]].
- **Tool list changes**: tools can't be unregistered → dropped tools re-registered `exposure:"hidden"` (`index.ts:433-437`; disabled server `:440-447`). `getClient()` resolves the connection at execution time: "MCP server X is disabled." / "is still starting." / "MCP tool s/t is no longer available." (`index.ts:411-427`).
- **Execution**: every MCP call runs through pi's tool pipeline (`tool_call`/`tool_result` hooks, permission gates) (`index.ts:20-21`); nested calls from codemode carry `parentToolCallId` (`docs/mcp.md:252`). Per-request timeout default `DEFAULT_TIMEOUT_SECONDS = 60` per server, reset by progress notifications (`runtime.ts:48,230`; `mcp-servers.ts:75-76`); client lib default 30 s (`packages/mcp/src/client.ts:38`). **Tool calls never retried** — "server may already have performed them" (`docs/mcp.md:248`).
- **Result conversion** (`mcp/tools.ts:121-230`): text > `MCP_OUTPUT_MAX_BYTES = 20 KiB` → `truncateMiddle` with Codex-format header `Warning: truncated output (original token count: N)\nTotal output lines: M` + `[Full output: path (read it with offset/limit)]` (`tools.ts:129-140`; spill `pi-mcp-*.txt` via `writeOutputFile`, `tools.ts:68-70`); images kept; `resource_link` → text naming `read_mcp_resource`; non-image binary embedded resources → temp files; text-ish MIME blobs decoded; empty `isError` → "MCP tool s/t returned an error" (`tools.ts:220`). `isError` → error result for the model, but `structuredContent` = full `CallToolResult` minus `_meta`, **untruncated**, for scripts ([[structured-tool-output]]).
- **Schema fixups**: default `type:"object"`, add empty `properties` (`tools.ts:232-242`). Annotations `readOnlyHint/destructiveHint/idempotentHint/openWorldHint` passed through for permission extensions (`tools.ts:244-254`; `docs/mcp.md:254`) — [[tool-safety-annotations]].
- **Resources**: Codex-named `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` (`packages/coding-agent/src/core/mcp-servers.ts:28-30`, `packages/coding-agent/src/extensions/mcp/resources.ts`), registered only when some server has resources, exposure = widest of those servers; descriptions say "Prefer resources over web search when possible" (`resources.ts:257,281`).
- **Transport** (`packages/mcp`): protocol `LATEST_PROTOCOL_VERSION = "2025-11-25"`, accepts `2025-06-18`, `2025-03-26`, `2024-11-05` (`protocol/types.ts:4-9`); streamable HTTP retries stream (re)open on 408/429/≥500 (`streamable-http.ts:166-168`), 401 / 403 `insufficient_scope` → auth (`:160-164`). Stdio shutdown per spec: close stdin → after `STDIN_CLOSE_GRACE_MS = 500` SIGTERM → after `DEFAULT_CLOSE_TIMEOUT_MS = 2000` SIGKILL, to the **process group** (kills npx/uvx children), groups tracked and killed if host exits (`stdio.ts:7-14,164-177`) — [[process-tree-kill]]. stderr kept 64 KiB (`stdio.ts:7`), 2,000-char tail shown in errors (`runtime.ts:49`).
- **OAuth**: DCR or CIMD, RFC 9728/8414/9207, tokens per server name+URL in `~/.pi/agent/mcp-auth.json`, cross-process refresh lock (`1d74741e1`) (`packages/coding-agent/src/extensions/mcp/oauth.ts`). `/login` offers Radius sign-in + MCP setup (`ed8b3bcc1`).
- Logs `~/.pi/agent/mcp.log`, rotate at 5 MB (`docs/mcp.md:98`). `/mcp` manager: sign in, reconnect, enable/disable, change exposure (saved to defining `mcp.json`) (`index.ts:23-26`).
- **Client surface (`packages/mcp/src/client.ts`)**: methods only for `ping`, `tools/list`+`tools/call`, `resources/list`/`templates/list`/`read` (`client.ts:290-386`); `prompts`, `sampling`, `elicitation`, `completions` exist only as capability *types* (`protocol/types.ts:26-36`) — no prompt/sampling/elicitation support. Servers with only prompts/resources (no `tools` capability) are not asked for `tools/list` (`packages/coding-agent/src/extensions/mcp/runtime.ts:407-411`). Client serves server→client `ping` and `roots/list` (`client.ts:173-177`); pi advertises the session cwd as the single root (`runtime.ts:383`). Generic `setRequestHandler`/`onNotification` for anything else (`client.ts:262-269`).
- **Cancellation both ways**: caller `AbortSignal` rejects the pending request and sends `notifications/cancelled {requestId, reason}` (`client.ts:401-435,555-569`); incoming `notifications/cancelled` aborts the handler serving that server request (`client.ts:545`); close aborts all (`:590-595`). → [[abort-propagation]].
- **`toLlmContent`**: text/images pass through, embedded text/image resources unwrapped, audio/resource links/binary → short placeholders, content-less result with `structuredContent` → its JSON (`packages/mcp/README.md:31`). OAuth subset adapted from MIT MCP TypeScript SDK v1.29.0 (`README.md:112`, `LICENSES/`), otherwise no SDK dependency. In-memory transport for tests (`src/transports/in-memory.ts`, `src/testing/`).
- **Conformance CI**: official `@modelcontextprotocol/conformance@0.2.0-alpha.11` run for each negotiated version (`2025-03-26`, `2025-06-18`, `2025-11-25`) against the real `McpServerConnection` + `signInMcpServer` with a simulated browser (re-sign-in up to 3× for scope step-up); fails on regressions vs committed `baseline.json`, newly passing checks reported (`packages/coding-agent/test/mcp-conformance/README.md`; `.github/workflows/ci.yml:47-65`; `a4715ec9b` 2026-10-01).
- **Selection interplay**: `--tools` allowlist does NOT remove MCP tools unless an entry starts with `mcp__` (`04b97ef00`; `docs/mcp.md:228`, `docs/cli.md:132`) — [[minimal-default-toolset]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_STARTUP_WAIT_MS` | 10_000 ms | `packages/coding-agent/src/extensions/mcp/index.ts:93` |
| `MAX_SERVER_DESCRIPTION_CHARS` | 250 | `mcp/index.ts:158` |
| `MAX_SERVERS_SECTION_CHARS` | 4096 | `mcp/index.ts:163` |
| `MAX_TOOL_NAME_LENGTH` | 64 | `mcp/tools.ts:49` |
| `MCP_OUTPUT_MAX_BYTES` | 20 KiB (middle cut) | `mcp/tools.ts:51` |
| `OUTPUT_PREVIEW_LINES` (TUI) | 5 | `mcp/tools.ts:53` |
| `DEFAULT_TIMEOUT_SECONDS` | 60 | `mcp/runtime.ts:48` |
| `STDERR_TAIL_CHARS` | 2_000 | `mcp/runtime.ts:49` |
| `CONNECT_RETRY_DELAYS_MS` | [250, 1000] | `mcp/runtime.ts:51` |
| client `DEFAULT_REQUEST_TIMEOUT_MS` | 30_000 | `packages/mcp/src/client.ts:38` |
| `MAX_LIST_PAGES` | 1_000 | `packages/mcp/src/client.ts:39` |
| `DEFAULT_MAX_STDERR_BYTES` | 64 KiB | `packages/mcp/src/transports/stdio.ts:7` |
| `STDIN_CLOSE_GRACE_MS` / `DEFAULT_CLOSE_TIMEOUT_MS` | 500 / 2000 ms | `stdio.ts:8-10` |
| log rotation | 5 MB | `docs/mcp.md:98` |

## Evolution
- Pre-2025-11-12: README rationale "Token efficient: A 225-token README beats a 13,000-token MCP server description"; "Composable: Chain tools with bash pipes"; "No overhead" (removed lines in `60e4fcf01`).
- 2025-11-12 `60e4fcf01` — "**pi does not support MCP.** Instead, it relies on the four built-in tools above and assumes the agent can invoke pre-existing CLI tools or write them on the fly." 2025-12-17 `3424550d2` Philosophy "No MCP. Build CLI tools with READMEs (see Skills), or build an extension that adds MCP support."; community `pi-mcp-adapter` (`docs/mcp.md:266`). Philosophy removed `25cc5c7bf` (2026-09-22).
- 2026-09-29 `8562bcf66` (v0.99.0, #10040) — built-in MCP + codemode + tool_search; token objection answered by default `codemode` exposure. `1d74741e1` same day: serialize OAuth refreshes across processes.
- 2026-09-30 `e029c3ed0` stop waiting for servers on first prompt; `1c7e7df76` (#10212) codemode lists servers instead of tools; `b29db895c` names aligned with codemode identifiers (`_` like Codex).
- 2026-10-01 `c662ec7e3` tools loaded by `tool_search` restored on resume/reload.
- 2026-10-02 `1387af7b4` project `mcp.json` may override `enabled`/`exposure` of global servers.
- 2026-10-03 `11449730c` render MCP calls before server connects.
- 2026-10-05 `04b97ef00` keep MCP tools with `--tools`, add tool patterns and `--no-mcp`.

## Evidence commits
`60e4fcf01`, `3424550d2`, `25cc5c7bf`, `8562bcf66`, `1d74741e1`, `e029c3ed0`, `1c7e7df76`, `b29db895c`, `c662ec7e3`, `1387af7b4`, `11449730c`, `04b97ef00`, `ed8b3bcc1`.

## Quirks
- Reversal rationale not stated in `8562bcf66`; author change (Mario Zechner → Armin Ronacher); inferred that codemode exposure made MCP cheap in context `(unverified)` — [[no-builtin-mcp-reversed]].
- Model-facing MCP cap (20 KiB) is smaller than built-in 50 KiB and cuts the **middle**, unlike head (read/search) or tail (bash) — [[tool-output-truncation]].
- Discovery tool activation uses config, not connected state, so a server that later exposes zero tools still activates codemode.
- Tools for a removed server linger as `hidden` registrations for the session lifetime.

## Failures
- [[mcp-tool-name-collision]]
- [[oauth-refresh-token-rotation-lost]]
- [[mcp-startup-blocks-and-description-churn]]
- [[tool-allowlist-hides-mcp-tools]]
- [[deferred-tools-lost-on-resume]]
