---
type: concept
stage: tool-design
tier: must-have
aliases: [codemode, "tools.<name>()", QuickJS WASM, code-mode-tool-orchestration, wasm-sandbox-per-execution, "codemode.mode on/only", "execute tool", "@opencode-ai/codemode", "$codemode.search", OPENCODE_EXPERIMENTAL_CODE_MODE, "owned tree-walking interpreter", codex-code-mode-host, CodeModeOnly, js_repl]
harnesses: [pi, opencode, codex]
---
The model writes a script that calls other tools as async functions inside a sandbox; only the script's output (not each intermediate result) enters the context.

## Why
- Many small sequential tool calls each cost a round-trip and put raw output in context; a script can parallelize (`Promise.allSettled`), chain, and filter large results first.
- Lets large tool catalogs (MCP) stay undeclared and the prompt stable/cacheable.
- The sandbox is a new attack/failure surface: runaway output exhausts host memory, scripts patching built-ins crash the host bridge ([[codemode-sandbox-escape-to-host]]); the API description must be exact about async semantics ([[tool-description-lies-about-async]]).

## Design space
- Sandbox: fresh WASM VM in a fresh worker per execution (pi QuickJS) vs persistent REPL vs subprocess (Node/Python).
- Capabilities: tools only, no fs/net/timers (pi) vs full runtime.
- Nested calls go through the full tool pipeline (validation, hooks, permissions) (pi) → [[nested-tool-calls]].
- Input as raw source via grammar-constrained sampling (pi Lark grammar) vs JSON-escaped string → [[constrained-tool-sampling]].
- Mode: alongside direct tools ("on") vs only code-mode declared ("only", direct declarations hidden) (pi both).
- Catalog in description under a token budget vs search-only discovery (pi both: inline budget + `searchTools()`).
- Script-visible values: structured output when declared, else text (pi) → [[structured-tool-output]].
- Cross-call state: branch-scoped KV store persisted as session entries (pi) → [[branch-scoped-extension-state]].
- Limits: memory cap, output cap (head/tail + spill), optional timeout, stall detection.
- Sandbox: V8 isolate per cell in a **separate host process** (stdio/gRPC), failure → process replacement (✔ codex) vs in-process WASM (✔ pi).
- Long-running scripts: yield + resumable `wait(cell_id)` (✔ codex, 10 s default yield) vs script timeout (✔ pi 300 s).
- Input: freeform raw-JS tool with optional `// @exec: {json}` pragma line (✔ codex) vs JSON arg with grammar-constrained source (✔ pi).
- Nested tool typing as TypeScript declarations rendered from input + output schemas under a byte budget (✔ codex).
- Cells interrupted together with their turn (✔ codex `509565820f`).
- Predecessor: persistent Node REPL tool (codex `js_repl` 2026-02 → removed 2026-04-24; races, hangs, U+2028 framing bugs).
- Engine: owned tree-walking interpreter over a restricted JS subset, no third-party VM (opencode: "bounded tool orchestration, not arbitrary JavaScript").
- Limits as host policy with no library defaults; host passes none, only user abort stops a script (opencode legacy) vs harness-enforced caps (pi).
- Reachable tools: MCP only (opencode legacy) vs every tool (pi).

## Implementations
- [[pi--code-mode|pi]] — `codemode({code})` QuickJS-WASM per call, `tools.*`, `models.*`, `store/load`, discovery globals; 256MB heap, 10k-token output budget; off by default, auto with MCP codemode exposure.
- [[codex--code-mode|codex]] — freeform `exec` (raw JS, V8 `=150.4.0` isolate per cell in `codex-code-mode-host`) + `wait`; `tools.*` with TS declarations, `store/load`, `notify`, `yield_control`; CodeMode / CodeModeOnly (off by default).
- [[opencode--code-mode|opencode]] — experimental `execute` tool over MCP tools only; TypeScript stripped, Acorn-parsed, owned tree-walking interpreter; no default timeout/call/output limits; ≤8 concurrent calls; per-child-call hooks + permission.

## Failures
- [[codemode-sandbox-escape-to-host]]
- [[tool-description-lies-about-async]]
- [[mcp-tool-name-collision]]
- (02) [[line-separator-breaks-jsonl-framing]]

## Related
[[nested-tool-calls]] · [[deferred-tool-loading]] · [[mcp-integration]] · [[structured-tool-output]] · [[constrained-tool-sampling]] · [[branch-scoped-extension-state]] · [[parallel-tool-execution]] · [[tool-description-design]] · [[structured-classifier-api]] · [[tool-wire-kinds]] · [[tool-schema-lowering]]

## Tradeoffs
- [[mcp-builtin-vs-extension]]
