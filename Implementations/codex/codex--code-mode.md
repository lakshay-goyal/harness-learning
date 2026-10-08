---
type: implementation
harness: codex
concept: code-mode
commit: 622e9e3696
files: [codex-rs/core/src/tools/code_mode/execute_spec.rs:5-27, codex-rs/code-mode-protocol/src/description.rs:16-134, codex-rs/code-mode-protocol/src/lib.rs:54-55, codex-rs/code-mode-protocol/src/runtime.rs:15-17, codex-rs/code-mode-runtime/src/service.rs:29-30, codex-rs/code-mode-runtime/src/runtime/mod.rs:192, codex-rs/code-mode-runtime/src/cell_actor/mod.rs:627, codex-rs/code-mode-host/src/main.rs:20-27, codex-rs/code-mode-host/src/lib.rs:55-60, codex-rs/core/src/tools/spec_plan.rs:657-671, codex-rs/core/src/tools/spec_plan.rs:838-861, e95abcdf49:codex-rs/core/src/tools/parallel.rs:181-185, codex-rs/core/src/thread_manager.rs:570-576, codex-rs/Cargo.toml:545]
---
[[code-mode]] in [[codex]] — V8 isolates in a separate host process; freeform `exec` (raw JS) + `wait` for long-running cells. Still UnderDevelopment (off by default).

## Mechanism
### Tool surface
- `exec`: **freeform** tool whose Lark grammar is an optional first-line pragma `// @exec: {json}` + raw JS source (`codex-rs/core/src/tools/code_mode/execute_spec.rs:5-27`, `codex-rs/code-mode-protocol/src/description.rs:134`) → [[tool-wire-kinds]], [[constrained-tool-sampling]].
- `wait(cell_id, yield_time_ms, max_tokens, terminate)` function tool (`description.rs:48-56`); names `exec` / `wait` (`codex-rs/code-mode-protocol/src/lib.rs:54-55`); `wait` description/params catalog-overridable ("Invalid catalog tool parameters; using bundled parameters", `codex-rs/core/src/tools/code_mode/wait_spec.rs:53`).
- Description contract (`description.rs:23-47`): "Run JavaScript code to orchestrate/compose tool calls"; "Evaluates the provided JavaScript code in a fresh V8 isolate as an async module"; "All nested tools are available on the global `tools` object, for example `await tools.exec_command(...)`. Tool names are exposed as normalized JavaScript identifiers, for example `await tools.mcp__ologs__get_profile(...)`"; "Runs raw JavaScript -- no Node, no file system, no network access, no console"; "Accepts raw JavaScript source text, not JSON, quoted strings, or markdown code fences"; pragma `yield_time_ms` / `max_output_tokens`; "When the JS code is fully evaluated, the isolate's lifetime ends and unawaited promises are silently discarded."
- Helpers: `exit()`, `text()`, `image()`, `audio()`, `generatedImage()`, `store(key, value)` / `load(key)` (session KV across `exec` calls), `notify()` (inject an extra `custom_tool_call_output` immediately), `setTimeout` / `clearTimeout` (pending timeouts don't keep exec alive), `ALL_TOOLS` (`{name, description}`), `yield_control()` (yield accumulated output while the script keeps running).
- Nested tool typing: each enabled tool rendered as a TypeScript declaration `declare const tools: { name(args: …): Promise<…> }` from input + output JSON schema, with an MCP `CallToolResult` preamble (`description.rs:57-133,437`); schema rendering budget `code_mode.tool_input_schema_max_bytes` (`codex-rs/core/src/tools/spec_plan.rs:657-671`) → [[tool-schema-lowering]]; this is why `exec_command` declares an `output_schema` ([[structured-tool-output]]).
- Deferred tools: omitted from the `exec` description but callable on `tools` / listed in `ALL_TOOLS`; discovery `await tools.tool_search({query, limit: 8})` → names + full TS declarations (`description.rs:16-20`; `4d15794336`) → [[deferred-tool-loading]].

### Modes / exposure
- `CodeMode` = `exec` alongside direct tools; `CodeModeOnly` = nested tools hidden from direct exposure, only `exec`/`wait` (+ `DirectModelOnly` tools such as `request_user_input`, `new_context`) visible (`spec_plan.rs:838-850`).
- Per-namespace: `code_mode.excluded_tool_namespaces` / `direct_only_tool_namespaces` (`spec_plan.rs:852-861,257-266`); strict 3P mode keeps third-party tools deferred inside exec (`58ae3ba611` 2026-10-03).
- Prompt (catalog, code-mode era): "Batch independent searches and reads in one functions.exec using await Promise.allSettled([...])"; "`JSON.stringify()` is not shell escaping" (`ed391d4dd2` / `49e95cc73f`, models.json) → [[parallel-tool-execution]].

### Execution model (mirrors unified exec)
- Default exec yield 10_000 ms, wait yield 10_000 ms, 10_000 output tokens per exec call (`codex-rs/code-mode-protocol/src/runtime.rs:15-17`); still-running script returns "Script running with cell ID …" and is resumed with `wait`; 1 s yield grace only when yield ≥ 10 s (`codex-rs/code-mode-runtime/src/service.rs:29-30`). Truncation is flagged clearly to the model (`952656356a`).
- Nested calls re-enter the **same router** as model calls with `ToolCallSource::CodeMode{cell_id, runtime_tool_call_id}` — same parallel gate, PreToolUse/PostToolUse hooks, approvals (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:181-185`; `codex-rs/core/src/tools/context.rs:56-66`) → [[nested-tool-calls]]; blocking PostToolUse respected in code mode (`d7f298fe20`).
- Interrupt: active cells interrupted with their turn (`509565820f`; `Feature::CodeModeInterrupt` in `handle_task_abort`, `codex-rs/core/src/tasks/mod.rs:942-952`) → [[abort-propagation]].

### Isolation
- V8 pinned `v8 = "=150.4.0"` (`codex-rs/Cargo.toml:545`); one isolate per cell (`codex-rs/code-mode-runtime/src/runtime/mod.rs:192`); CPU-bound scripts stopped via `terminate_execution` (`codex-rs/code-mode-runtime/src/cell_actor/mod.rs:627`).
- Runs in a **separate process** `codex-code-mode-host` speaking stdio or gRPC (`codex-rs/code-mode-host/src/main.rs:20-27`, `transport.rs:9`); host caps: 256 in-flight requests, 128 active cells, 4096 recent request / session ids, 128-slot outgoing channel, 5 s shutdown (`codex-rs/code-mode-host/src/lib.rs:55-60`). "Transport or runtime failure closes the connection and relies on process replacement" (`da78d5fdc5`).
- `Feature::CodeModeHost` (Stable, on) selects the process-owned provider (`codex-rs/core/src/thread_manager.rs:570-576`); `Feature::CodeMode` / `CodeModeOnly` UnderDevelopment, off (`codex-rs/features/src/lib.rs:1124-1178`).
- **Session client crate** `codex-rs/code-mode` (`codex_code_mode`): `ProcessOwnedCodeModeSessionProvider` = one lazily spawned local host per provider; `DisabledCodeModeSessionProvider` rejects sessions; `GrpcCodeModeSessionProvider` (`1e557a554e` 2026-08-11) talks to a remote host over HTTP/2 gRPC (`http://`, `https://`, `unix://`), with reconnect + host *generations* (replacing a host changes public cell ids), `TRANSPORT_TIMEOUT` 60 s, W3C `traceparent` propagation (`codex-rs/code-mode/src/remote_session.rs:35-50`; `codex-rs/code-mode/src/grpc_session/mod.rs:55-80`; `codex-rs/code-mode/src/grpc_session/deadline.rs:7`). App-server picks it via `CodeModeHostTransport::Grpc(url)`, erroring unless `code_mode_host` is enabled (`codex-rs/app-server/src/lib.rs:602-616`) → [[remote-execution-env]].
- No explicit V8 heap limit found (`CreateParams::default()`) — whether the host process has OS-level limits is unverified.
- `codex-rs/v8-poc`: placeholder "Bazel-wired proof-of-concept crate reserved for future V8 experiments" (`codex-rs/v8-poc/src/lib.rs:1`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_EXEC_YIELD_TIME_MS` / `DEFAULT_WAIT_YIELD_TIME_MS` | 10_000 / 10_000 ms | `codex-rs/code-mode-protocol/src/runtime.rs:15-16` |
| `DEFAULT_MAX_OUTPUT_TOKENS_PER_EXEC_CALL` | 10_000 | `codex-rs/code-mode-protocol/src/runtime.rs:17` |
| `YIELD_GRACE_PERIOD` / `MIN_YIELD_TIME_FOR_GRACE` | 1 s / 10 s | `codex-rs/code-mode-runtime/src/service.rs:29-30` |
| `MAX_IN_FLIGHT_REQUESTS` / `MAX_ACTIVE_CELLS` | 256 / 128 | `codex-rs/code-mode-host/src/lib.rs:55-56` |
| host `SHUTDOWN_TIMEOUT` | 5 s | `codex-rs/code-mode-host/src/lib.rs:59` |
| V8 version pin | `=150.4.0` | `codex-rs/Cargo.toml:545` |
| heap limit | none explicit (unverified) | `CreateParams::default()` |

## Evolution
- 2026-02-11 `42e22f3bde` js_repl — persistent Node-based freeform JS REPL; 2026-02-17 `77f74a5c17` "race in js repl", `846464e869` reset hang; 2026-02-20 `73fd939296` grammar blocks wrapped payload prefixes; 2026-03-12 `d9a403a8c0` hard-stop js_repl on interrupt; `f35d46002a` U+2028/2029 broke JSONL framing ([[line-separator-breaks-jsonl-framing]]).
- 2026-03-09 `da616136cc` "Add code_mode experimental feature — A much narrower and more isolated (no node features) version of js_repl"; 2026-03-12 `d1b03f0d7f` default yield; 2026-03-20 `e4eedd6170` "Code mode on v8".
- 2026-04-24 `8a559e7938` js_repl removed (63 files, 9,261 deletions; no-op flag `JsRepl` kept).
- 2026-06-11 `aa46f2debf` protocol extracted + host crate; 2026-06-25 `da78d5fdc5` standalone process host, `ab16046c88` process-owned session client; 2026-07-30 `97576b1794` exclusively through the host.
- 2026-06-15 `d7f298fe20` blocking PostToolUse in code mode; 2026-06-16 `952656356a` clear truncation warning.
- 2026-07-30 `c126f206da` normalized tool-name collisions in code mode resolved ([[mcp-tool-name-collision]]).
- 2026-08-07 `509565820f` interrupt active cells with their turn.
- 2026-09-15 `aaa2cabfbc` "Disable V8 optimization paths affected by array sort bugs".
- 2026-09-24 `339e981ba7` configurable code-mode schema budget. 2026-10-03 `58ca099b03`, `58ae3ba611`; 2026-10-06 `4d15794336` ranked discovery.

## Versus pi
pi: QuickJS-WASM per call inside the host process, 256 MB heap, output cap with spill, `codemode({code})` JSON arg + Lark-constrained source ([[pi--code-mode]]); its failure [[codemode-sandbox-escape-to-host]] (print loops OOM'd the host) is the motivating hazard codex avoids by process isolation + replacement. codex adds a resumable `wait` (yield model) instead of a script timeout.
