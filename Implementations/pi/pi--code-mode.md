---
type: implementation
harness: pi
concept: code-mode
commit: b30a6dd77
files: [packages/coding-agent/src/extensions/codemode/tool.ts:89-158, packages/coding-agent/src/extensions/codemode/tool.ts:216-398, packages/coding-agent/src/extensions/codemode/execute.ts:47-56, packages/coding-agent/src/extensions/codemode/execute.ts:232-537, packages/codemode/src/runtime/host.ts:22, packages/codemode/src/runtime/host.ts:120-124, packages/codemode/src/runtime/worker.ts:26-148, packages/codemode/src/runtime/prelude-source.ts:22-41, packages/codemode/src/source.ts:19-22, packages/coding-agent/docs/codemode.md:3-7]
---
[[code-mode]] in [[pi]].

## Mechanism
- **Why** (`packages/coding-agent/docs/codemode.md:3`): "Only the script's output reaches the model, so a script can run calls in parallel and filter large results before the model sees them." Also keeps MCP tools out of declarations → stable, cacheable prompt (`extensions/mcp/index.ts:9-14`). "Compatible with Codex's exec tool, and codemode.mode works like Codex's tool modes (on or only)" (`8562bcf66`).
- **Schema** `{code: string}` — "Raw JavaScript source." (`packages/coding-agent/src/extensions/codemode/tool.ts:89-93`). `constrainedSampling: {type:"grammar", variants:{openai_lark: CODEMODE_SOURCE_GRAMMAR}}` — "Capable models write the script as raw text instead of a JSON-escaped string" (`tool.ts:394-395`; grammar `packages/codemode/src/source.ts:19-22`); falls back to a normal function tool where grammar tools unsupported ([[constrained-tool-sampling]]).
- **Description** = `DESCRIPTION_INTRO` (`tool.ts:137-139`): "Run JavaScript that calls other tools. The input is raw JavaScript (not JSON, no code fence), run as an async function body in a QuickJS sandbox: top-level `await` and `return` work. No Node, file system, network, or timers. - `await tools.<name>({ ...args })` resolves to a string, or an object if the tool's declaration says so, and rejects with an Error on failure. Calls still running when the script ends are cancelled. - Optional first line: `// @options: {"max_output_tokens": 10000, "timeout_ms": 60000}`" + `Globals:` one line per global (`tool.ts:142-153`: `text()/image()/console.log/return/exit()`, `store/load`, `ALL_TOOLS/searchTools/describeTool/describeNamespace`, optional `models` "Read docs/codemode.md first") + optional "Shared MCP Types" TS preamble (`tool.ts:276`) + "Nested tools:" TS declarations grouped by namespace (`tool.ts:280`). Full reference offloaded to `docs/codemode.md` (`CODEMODE_DOCS_PATH`, `tool.ts:134`) — [[tool-description-design]].
- Snippet "Run JavaScript that calls other tools"; guideline "Use codemode to batch independent tool calls (Promise.allSettled), chain them, or filter large output, instead of many separate calls." (`tool.ts:127-132`). Listed tools' `promptGuidelines` travel inside codemode's description because system-prompt rules cover only declared tools (`tool.ts:330-336`; `c30840c2e`).
- `exposure: "model-only"` — "Scripts must not start other scripts" (`tool.ts:391-392`).
- **Catalog budget**: `DEFAULT_CODEMODE_INLINE_BUDGET = 3000` est. tokens at chars/4 (`tool.ts:156-158`), setting `codemode.inlineBudget`; round-robin `selectCatalog` across namespace groups, cheapest-first (`tool.ts:216-239`); deferred tools never listed (`tool.ts:241-247`). Detail in [[pi--deferred-tool-loading|pi deferred-tool-loading]].
- **Modes** (`prepareCodemodeLoadout`, `tool.ts:330-378`), setting `codemode.mode`:
  - `on` (default): declared tools keep their declaration, description gets appended `Codemode: \`tools.x(args)\` resolves to …` (`tool.ts:327,348-352`); codemode lists only callable non-`direct` tools.
  - `only`: codemode lists every callable tool; active `direct` declarations hidden via `hiddenDeclarations` (`tool.ts:353,373-376`).
- **Execution** (`packages/coding-agent/src/extensions/codemode/execute.ts:416-537`): parse `// @options` (only `max_output_tokens`, `timeout_ms`; line replaced by empty line to keep line numbers, `packages/codemode/README.md` "Source format"); callable = `ctx.tools` minus codemode; each `tools.x()` → `ctx.executeTool(name,args,{signal})` = full validation, `tool_call`/`tool_result` hooks, permission gates, id `<parent>/<n>` ([[nested-tool-calls]]).
  - **Value to script** (`toScriptValue`, `execute.ts:400-410`): `structuredContent` if the tool declares `outputSchema` (even for `isError` results, e.g. MCP) else joined text; `isError` without structured → reject with text ([[structured-tool-output]]).
  - Timeout: `timeoutMs: sourceOptions.timeoutMs ?? Number.POSITIVE_INFINITY` — **no default** (`execute.ts:484`); sandbox lib default `DEFAULT_TIMEOUT_MS = 300_000` unused by pi (`packages/codemode/src/runtime/host.ts:22`). Abort via parent signal.
  - Memory: `CODEMODE_MEMORY_LIMIT_BYTES = 256 MiB` — worker shares pi's process; without it a runaway script could grow to wasm32's 4 GiB (`execute.ts:51-56,485`).
  - **Output**: budget `DEFAULT_MAX_OUTPUT_TOKENS = 10_000` × 4 chars (`execute.ts:246,522`); overflow → head/tail halves + `…N tokens truncated…`, full text spilled to `pi-codemode-*.txt` with `[Full output: path (read with offset/limit)]` (`execute.ts:373-396`). Header `Script completed|Script failed\nWall time X seconds\nOutput:\n` (`execute.ts:528`); multiple text items get `==> text N/M <==`; console lines grouped in `<console_output>` (`eb326d265`). Failures list nested calls made before failure "(they are not undone)" (`execute.ts:298-314`). Middle truncation — see [[tool-output-truncation]].
  - In-VM hard caps `MAX_OUTPUT_CHARS = 16 Mi`, `MAX_OUTPUT_ITEMS = 100000` → RangeError (`packages/codemode/src/runtime/prelude-source.ts:40-41`; `319fecb89`).
  - `image()` validates base64 + magic signature (`d2931ad3d` #10215) and saves each image to a temp file with a path label (`execute.ts:333-366`; `d677d0ee7`) — [[image-normalization]].
  - **store/load**: JSON KV across calls; writes kept only if script succeeds; persisted as `codemode-store` custom session entries applied root→leaf → branch-aware (`execute.ts:232-243,503-507`); limits 256 KiB/value, 1 MiB total (`prelude-source.ts:32-33`) — [[branch-scoped-extension-state]].
  - **models.\***: catalog list, `classify()`, `generateImages()`; resolved by provider+id only — "A script-supplied baseUrl or headers must never receive the credentials" (`execute.ts:638-641`); catalog strips `headers` (`execute.ts:90-95`); `MAX_CONCURRENT_MODEL_CALLS = 4` (`execute.ts:50,635`); usage added to session cost (`9a100c7cc`) — [[structured-classifier-api]].
  - Discovery globals: `searchTools()` (same BM25 as `tool_search`), `describeTool()`, `describeNamespace()` (`execute.ts:554-620`); namespace aliases `mcp__dev-radius|mcp__dev_radius|dev-radius|dev_radius` (`execute.ts:543-548`); unknown tool → "Did you mean …" ≤5 close matches (`prelude-source.ts:238`).
- **Sandbox** (`packages/codemode/src/runtime/{host,worker}.ts`):
  - QuickJS → WASM (`quickjs-wasi`); **fresh worker thread + fresh VM per execution**: "keeps termination simple: a runaway script, including one that only spins the microtask queue, is killed with `terminate()` and cannot poison a later run" (`host.ts:120-124`).
  - Capabilities only via injected tools/globals through a `bridge` passing primitives/JSON strings (`worker.ts:64-102`); no timers/fetch/process/require/modules/WebAssembly; `eval`/`Function` work but stay in-VM (`packages/codemode/README.md`).
  - Interrupt: `SharedArrayBuffer` flag polled by QuickJS `interruptHandler` (`worker.ts:60`, `host.ts:129`); `maxStackSize` so deep recursion → RangeError not trap (`worker.ts:57-59`); QuickJS fd 1/2 discarded (`worker.ts:26-44`).
  - Stall detection: awaiting a promise nothing can settle (no pending host call, no timers) fails immediately (`worker.ts:120-124`; `prelude-source.ts:22`).
  - Hardening `b223082bb` (#10444): built-ins frozen + globals read-only before script; host validates every worker payload (`BridgeError` "The script may have modified built-ins such as a prototype's toJSON") (`host.ts:49-95,200-213`).
  - Wrapper `(async (tools, console) => {<code>\n})` on the script's first line so stack line numbers match (`worker.ts:144-148`).
  - Lazy: worker + wasm load on first call (`tool.ts:396-398`, `execute.lazy.ts`); bundled builds pass explicit worker entry (`4a42f8faf`).
- **Tools shown to the model as TypeScript, not JSON Schema** (`packages/codemode/src/declarations.ts`): each tool rendered as description + ```` ```ts declare const tools: { name(args: T): Promise<R>; } ```` sample (`renderToolSample`, `declarations.ts:153-159`) — used in the codemode description listing (`packages/coding-agent/src/extensions/codemode/tool.ts:202`) and `ALL_TOOLS`/describe results (`execute.ts:445`). `schemaToType` converts JSON Schema → TS type; input types > `DEFAULT_INPUT_SCHEMA_MAX_CHARS = 16_000` chars collapse to `unknown`, local `$ref` expansion capped at `MAX_REF_EXPANSIONS = 32` and stops at recursive refs (`declarations.ts:10-12,228-231`). MCP output schemas detected structurally (`content` array + boolean `isError` + object `_meta`) → `Promise<CallToolResult<T>>` with a TS preamble of MCP result types (`declarations.ts:18,166-181`). Identifiers: invalid JS chars → `_` (`my-tool` → `my_tool`) (`identifier.ts:1-12`). QuickJS wasm compiled once per path, cached; path overridable for Bun-compiled binaries; failed load retried (`wasm.ts:14-20`).
- **Nested-call record** on the parent result: 256 calls, 8 KiB args/call, 32 KiB total, 500 error chars (`packages/coding-agent/src/core/nested-tool-calls.ts:26-31`).
- **Activation**: registered `defaultActive:false` (`codemode/index.ts:43`); `"defaultTools": ["+codemode"]` or auto when an MCP server with `codemode` exposure is configured and `autoEnableCodemode !== false` (`docs/mcp.md:204,230`; `mcp/index.ts:481-519`). `replaceable: true` built-in (`extensions/index.ts:7-14`).
- Not a security boundary for effects: "isolates script logic, not tool effects" — every nested call is a normal tool call (`docs/codemode.md:7`) — [[no-sandbox]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_CODEMODE_INLINE_BUDGET` | 3000 est. tokens | `packages/coding-agent/src/extensions/codemode/tool.ts:156` |
| `CODEMODE_MEMORY_LIMIT_BYTES` | 256 MiB | `codemode/execute.ts:56` |
| `DEFAULT_MAX_OUTPUT_TOKENS` | 10_000 (×4 chars) | `codemode/execute.ts:246` |
| script timeout (pi) | ∞ unless `timeout_ms` | `codemode/execute.ts:484` |
| `DEFAULT_TIMEOUT_MS` (lib) | 300_000 | `packages/codemode/src/runtime/host.ts:22` |
| `MAX_CONCURRENT_MODEL_CALLS` | 4 | `codemode/execute.ts:50` |
| `ARGS_PREVIEW_CHARS` / `ERROR_PREVIEW_CHARS` | 200 / 500 | `codemode/execute.ts:47-48` |
| `MAX_STORE_VALUE_CHARS` / `MAX_STORE_TOTAL_CHARS` | 256 KiB / 1 MiB | `packages/codemode/src/runtime/prelude-source.ts:32-33` |
| `MAX_OUTPUT_CHARS` / `MAX_OUTPUT_ITEMS` | 16 MiB / 100_000 | `prelude-source.ts:40-41` |
| `COLLAPSED_ARGS_CHARS` (renderer) | 80 | `codemode/renderer.ts:21` |

## Evolution
- 2025-11 → 2026-09: no MCP, no code execution tool; "No MCP" stance ([[no-builtin-mcp-reversed]]).
- 2026-09-29 `8562bcf66` (closes #10040, Armin Ronacher) — codemode + MCP: "The core only gets general mechanisms; codemode, tool_search and MCP are built-in extensions that use them."
- 2026-09-29 `9a100c7cc` — classifier + nested tool usage into session cost. `1ff5b6fdd` — bash returns up to 1 MiB structured output to scripts.
- 2026-09-30 `1c7e7df76` (#10212) — list MCP servers not tools; `d2931ad3d` (#10215) validate `image()`; `4a42f8faf` Windows binary worker; `0582d9c11` previews limited to wrapped lines; `028c0ec56` (#10192) `only` mode no longer lists read/bash/edit/write in system prompt.
- 2026-10-01 `6f1072cc0` — "keep only one line per global in the tool description … Declared tools get a one-line note on how scripts call them"; errors "guide scripts back" (close-match suggestions) instead of documenting everything upfront; default GPT-5.6 request ~5,300 → ~3,300 tokens.
- 2026-10-02 `319fecb89` (#10283) cap output in VM. 2026-10-04 `d677d0ee7` images to temp files.
- 2026-10-05 `b223082bb` (#10444) freeze built-ins/validate payloads; `021eae60a` (#10251) codemode `read` on images → image blocks; `c30840c2e` guidelines travel with listing.
- 2026-10-06 `269121616` (#10555) description marks `searchTools/describeTool` as `await`-ed; `eb326d265` output item separators.

## Evidence commits
`8562bcf66`, `9a100c7cc`, `1ff5b6fdd`, `1c7e7df76`, `d2931ad3d`, `4a42f8faf`, `0582d9c11`, `028c0ec56`, `6f1072cc0`, `319fecb89`, `d677d0ee7`, `b223082bb`, `021eae60a`, `c30840c2e`, `269121616`, `eb326d265`.

## Quirks
- No default script timeout in pi despite the lib's 300 s default (`execute.ts:484` vs `host.ts:22`) — a hung nested tool blocks until user abort.
- Calls still running when the script returns are cancelled; nested side effects of a failed script are explicitly "not undone".
- `store()` writes discarded on failure — transactional per script.
- Script detection of needed MCP servers is a regex on source text (`scriptNeedsServer`, `mcp/index.ts:227-231`), so dynamic names via string concatenation only work because discovery globals force waiting for all servers.
- Grammar sampling only on OpenAI-family Responses with `supportsOpenAIGrammarTools`; elsewhere the model must JSON-escape JS (`unverified` error rate).

## Failures
- [[codemode-sandbox-escape-to-host]]
- [[tool-description-lies-about-async]]
- [[mcp-tool-name-collision]]
- [[image-content-poisoning]]
