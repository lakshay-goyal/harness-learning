---
type: harness
repo: https://github.com/openai/codex
commit: 622e9e3696
language: Rust
studied: 2026-10-08
aliases: [codex-cli, codex-rs, "OpenAI Codex CLI", "@openai/codex"]
---

OpenAI's coding-agent harness. Cargo workspace `codex-rs/` (~2.03M lines Rust in 5,172 `.rs` files, 157 `Cargo.toml` manifests), plus Python/TypeScript SDKs (`sdk/python`, `sdk/python-runtime`, `sdk/typescript`) and an npm wrapper (`codex-cli/`) that ships the Rust binary. 11,968 commits (2025-04-16 `59a180ddec` → 2026-10-07 `622e9e3696`), 663 author names, 1,470 tags (270 stable `rust-v0.x.y`; latest stable `rust-v0.161.0`, i.e. a stable release every 2–3 days). Apache-2.0 (`LICENSE:1-3`). Build: Cargo + Bazel (`MODULE.bazel`, `rbe.bzl`) + Nix (`flake.nix`) + `justfile`. Many commits carry `Co-authored-by: Codex` (e.g. `7a8407bbb6`): Codex is largely built with Codex. Digest: [[2026-10-08-codex]].

## Ideology
- **The model is the product; the harness serves one model family.** Responses API only (Chat Completions deleted `d2394a2494` 2026-02-03 → [[no-chat-completions-wire]]). The system prompt, tool set, truncation policy and context window come from the **model catalog**, not harness code (`a1abd53b6a` 2026-02-09 removed per-family prompt fallbacks) → [[per-model-system-prompt]], [[model-catalog]], [[tool-wire-kinds]]. Tool names and formats match what the models were trained on (`apply_patch`, `update_plan`, `shell`) → [[patch-envelope-edit]], [[foreign-harness-tool-hallucination]].
- **Contain the agent, don't trust it.** Every model-launched process runs in a kernel sandbox (Seatbelt / bubblewrap+seccomp / Windows restricted token), network off by default. Approval policy decides when a human or an **LLM reviewer** ("guardian") is asked. Escalation means re-running outside the sandbox only after approval. Exact opposite of pi → [[os-level-sandbox]], [[approval-policy-modes]], [[sandbox-escalation-retry]], [[llm-approval-reviewer]], [[isolation-strategy]].
- **Shell is the universal tool.** No read/grep/ls tools: the model uses shell (PTY-backed "unified exec" sessions) plus `apply_patch`. The UI recovers semantics by parsing commands. Experimental `read_file`/`list_dir`/`grep_files` were added in Oct 2025 and later removed → [[shell-execution]], [[shell-command-intent-parsing]], [[no-file-read-write-tools]], [[dedicated-vs-shell-tools]].
- **One core, many frontends, over a protocol.** Core = submission queue / event queue (`Op` in, `EventMsg` out). The TUI, `codex exec`, IDE extension and SDKs all talk JSON-RPC to the **app-server** (TUI moved onto it `db89b73a9c` 2026-03-16) → [[agent-event-stream]], [[client-server-session-split]], [[headless-rpc-mode]].

## Organ map
| Organ | Files | Notes |
|---|---|---|
| Loop | `codex-rs/core/src/session/turn.rs:150-837` (`run_turn` outer loop), `codex-rs/core/src/session/turn.rs:2513-3191` (`try_run_sampling_request` stream loop), `codex-rs/core/src/tasks/mod.rs` (task slot), `codex-rs/core/src/session/handlers.rs` (submission loop) | Turn continues while any tool call was dispatched or input is pending (`turn.rs:563`). Tools start executing during streaming (`FuturesOrdered`). No step cap: "as long as compaction works well … we shouldn't worry about being in an infinite loop" (`turn.rs:588`) → [[turn-loop]], [[single-active-task-slot]], [[no-turn-cap]] |
| Message builder | `codex-rs/core/src/context_manager/history.rs`, `normalize.rs`, `updates.rs`; `codex-rs/context-fragments/`; `codex-rs/core/src/context/**` (world state) | History = list of Responses `ResponseItem`s. Call/output pairing normalized. Harness text in tagged fragments with fixed roles. Environment/permissions re-sent as diffs → [[message-role-layering]], [[xml-prompt-boundaries]], [[world-state-diff-injection]], [[transcript-replay-repair]] |
| System prompt | model catalog `models.json` (bundled + remote via `codex-rs/models-manager/`), `codex-rs/protocol/src/prompts/base_instructions/default.md`, `codex-rs/prompts/templates/**`, `codex-rs/core/src/agents_md.rs` | Per-model base prompt. Since 2026-10-05 it is sent as a developer message, not `instructions`. AGENTS.md hierarchy, permission fragments, plan/persistent/personality fragments → [[per-model-system-prompt]], [[context-file-hierarchy]], [[permission-state-prompt]], [[plan-mode]] |
| Tools | `codex-rs/core/src/tools/**` (spec plan, router, registry, handlers), `codex-rs/tools/`, `codex-rs/apply-patch/`, `codex-rs/core/src/unified_exec/**`, `codex-rs/codex-mcp/`, `codex-rs/rmcp-client/`, `codex-rs/code-mode*/` | Model-/feature-gated tool plan. Function, freeform (Lark grammar) and hosted tools. Parallel calls when the tool allows it. MCP with deferred `tool_search`. V8 code mode → [[minimal-default-toolset]], [[tool-wire-kinds]], [[parallel-tool-execution]], [[deferred-tool-loading]], [[code-mode]] |
| Provider | `codex-rs/core/src/client.rs`, `codex-rs/codex-api/`, `codex-rs/codex-client/`, `codex-rs/model-provider/`, `codex-rs/model-provider-info/`, `codex-rs/login/` | Stateless requests (`store:false`, full input, encrypted reasoning replayed, `client.rs:961,996-1012`, `591cb6149a`). `prompt_cache_key` = session id. Responses-over-WebSocket with incremental append. ChatGPT OAuth + API key + Bedrock → [[unified-provider-api]], [[signed-reasoning-replay]], [[session-affinity-cache-routing]], [[subscription-oauth-auth]] |
| Session store | `codex-rs/rollout/` (`$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl`, `codex-rs/rollout/src/lib.rs:86-87`), `codex-rs/thread-store/`, `codex-rs/state/` (SQLite) | Linear JSONL is the source of truth. SQLite indexes threads. Resume = reverse replay to the newest surviving compaction. Fork = copy history → [[session-tree]], [[sqlite-session-index]], [[context-projection]], [[session-fork]], [[session-log-shape]] |
| Context mgmt | `codex-rs/core/src/compact.rs`, `compact_remote*.rs`, `codex-rs/core/src/session/turn.rs:1434` (`run_auto_compact`), `codex-rs/prompts/templates/compact/*` | Token count = last usage + 4 bytes/token estimate. Auto-compaction at pre-turn / mid-turn / post-turn. **Remote** compaction via `/responses/compact` (default since `cac0a6a29d` 2025-11-18) or local handoff summary. Model-requested context reset → [[auto-compaction]], [[token-estimation]], [[model-requested-context-reset]], [[compaction-locus]] |
| Safety | `codex-rs/sandboxing/`, `codex-rs/linux-sandbox/`, `codex-rs/windows-sandbox-rs/`, `codex-rs/execpolicy/`, `codex-rs/core/src/guardian/**`, `codex-rs/network-proxy/`, `codex-rs/process-hardening/` | Sandbox modes × approval policies. Execpolicy prefix rules. Guardian LLM reviewer. Egress proxy. Self-hardening → [[os-level-sandbox]], [[command-rule-policy]], [[egress-policy-proxy]], [[harness-process-hardening]] |
| Sub-agents | `codex-rs/core/src/agent/**`, `codex-rs/core/src/tools/handlers/multi_agents*`, `codex-rs/core/src/codex_delegate.rs`, `codex-rs/ext/agent*`, `codex-rs/ext/goal` | Child threads in the same process, results delivered to a mailbox, roles, concurrency caps. Review = locked-down child session. Goal mode auto-continues → [[in-process-subagent-threads]], [[subagent-result-mailbox]], [[review-subagent]], [[persistent-goal-continuation]] |
| Cancellation | `Op::Interrupt` → `abort_all_tasks(Interrupted)` (`codex-rs/core/src/session/mod.rs:5006-5013`) → cancellation-token tree → 100 ms grace (`codex-rs/core/src/tasks/mod.rs:925-1029`) → hard abort → synthesized `aborted by user` tool outputs + `<turn_aborted>` marker. Background terminals deliberately survive (`ba463a9dc7`) | [[abort-propagation]], [[partial-message-persistence]], [[interrupted-turn-invisible-to-model]] |

## Eras (`git log`, crate births/deaths)
| Era | Span | Defining commits |
|---|---|---|
| 0 TypeScript CLI | 2025-04-16 → 2025-08-08 | `59a180ddec` Node/Ink CLI. `31d0d7a305` Rust import (2025-04-24). `3104d81b7b` AGENTS.md. `408c7ca142` TS code deleted → [[no-typescript-cli]] |
| 1 Rust + GPT-5 tooling | 2025-08 → 2025-11 | `236c4f76a6` freeform apply_patch. `c09ed74a16` unified exec. `90a0fd342f` review mode. `d9dbf48828` app-server. `dc3c6bf62a` parallel tools. `a941ae7632` execpolicy v2. `838531d3e4` remote compaction. `e92c4f6561` ghost-commit undo (un-shipped `7a8407bbb6`) |
| 2 Platform + agents | 2025-12 → 2026-03 | `a8d5ad37b8` skills. `d2394a2494` Chat Completions removed. `86f81ca010` multi-agent tools. `d544adf71a` plan mode. `3878c3dc7c` SQLite state. `4922b3e571` memories. `3b54fd7336` hooks. `e84ee33cc0` guardian. `da616136cc` code mode. `db89b73a9c` TUI on app-server |
| 3 Decomposition + extensions | 2026-03 → 2026-10 | Mass crate extraction from core. `dae56994da` ThreadStore. `cefcfe43b9` Bedrock. `0ee737cea6` goals. `d2c3ebac1f` typed extension API (`codex-rs/ext/*`). `abbdde95b5` agent message board. `531f3836a1` `codex mcp-server` removed |

## Timeline (dated)
- 2025-04-16 `59a180ddec` Node/TypeScript `codex-cli`: Ink TUI, loop in `75febbdefa:codex-cli/src/utils/agent/agent-loop.ts`, Responses API, approval modes suggest / auto-edit / full-auto. 2025-04-18 `9a948836bf` manual `/compact` already in the TS era.
- 2025-04-24 `31d0d7a305` Rust import into `codex-rs/` (crates ansi-escape, apply-patch, cli, core, exec, interactive, repl, tui). Same day `58f0e5ab74` execpolicy crate ("safe" commands).
- 2025-04-28 `cca1122ddc` TUI becomes the default interactive CLI; 2025-04-30 `c432d9ef81` REPL crate removed.
- 2025-05-02 → 05-05 MCP in Rust (`83961e0299` mcp-types, `21cd953dbd` mcp-server, `2cf7aeeeb6` mcp-client).
- 2025-05-08 `b940adae8e` Responses API working in Rust; same day `e924070cee` Chat Completions added as second wire.
- 2025-05-10 `3104d81b7b` migrate to AGENTS.md (from codex.md / instructions.md); `2b122da087` in Rust. 2025-05-12 `73fe1381aa` npm package starts shipping the Rust binary (`--native`).
- 2025-06-04 `515b6331bd` "add support for login with ChatGPT (#1212)" → [[subscription-oauth-auth]].
- 2025-07-29 `8828f6f082` experimental plan tool (`update_plan`) → [[plan-checklist-tool]].
- 2025-08-07 `107d2ce4e7` default model → "gpt-5". 2025-08-08 `408c7ca142` TypeScript code removed, Rust-only from v0.20.0 → [[no-typescript-cli]].
- 2025-08-13 `08ed618f72` ConversationManager; 2025-08-15 `d262244725` codex-protocol crate (shared wire types).
- 2025-08-22 `236c4f76a6` freeform (grammar) apply_patch; disabled by default 2 days later `4157788310` → [[tool-wire-kinds]].
- 2025-08-23 `363636f5eb` hosted web search → [[web-search-tool]].
- 2025-09-03 `234c0a0469` resume picker (`--resume` / `--continue`).
- 2025-09-10 `c09ed74a16` unified exec (PTY sessions) → [[shell-execution]].
- 2025-09-12 `90a0fd342f` review mode; 2025-10-29 `13e1d0362d` review delegated to a sub-Codex → [[review-subagent]].
- 2025-09-23 `2451b19d13` auto-compaction on for gpt-5-codex; `b90eeabd74` threshold 250k → [[auto-compaction]].
- 2025-09-26 `c549481513` responses-api-proxy; `e555a36c6a` rmcp-client.
- 2025-09-30 `d9dbf48828` app-server crate (split from `codex mcp`); same day `5b038135de` cloud tasks → [[cloud-task-delegation]].
- 2025-10-03 `33d3ecbccc` tool registry refactor; `e0b38bd7a2` per-model tool lists; 2025-10-05 `dc3c6bf62a` parallel tool calls (on `f5d9939cda` 2025-11-18).
- 2025-10-07/08/09 experimental `list_dir` `226215f36d`, `grep_files` `f52320be86`, `read_file` `0026b12615` (later removed → [[no-file-read-write-tools]]).
- 2025-10-22 `fd0673e457` local tokenizer; deleted `52d0ec4cd8` 2025-11-20 → [[no-local-tokenizer]].
- 2025-10-27 `e92c4f6561` / `afc4eaab8b` ghost commits + /undo; default on `052b052832` 2025-11-11; un-shipped `7a8407bbb6` 2025-12-22 → [[no-checkpoints-undo]].
- 2025-10-30 `87cce88f48` Windows sandbox alpha.
- 2025-11-17 `a941ae7632` execpolicy v2; cutover `fb9849e1e3` 2025-11-19; legacy engine removed `656a2d0905` 2026-07-10 → [[command-rule-policy]].
- 2025-11-18 `838531d3e4` remote compaction, on by default `cac0a6a29d`; API-key users `b3ddd50eee` 2025-12-12.
- 2025-11-25 `4502b1b263` codex-api + codex-client extracted from core.
- 2025-12-01 `a8d5ad37b8` skills. 2025-12-09 `0c8828c5e2` tui2 experiment (retired `a489b64cb5` 2026-01-21).
- 2025-12-11 `43e6e75317` `wire_api = "chat"` deprecation; 2026-02-03 `d2394a2494` Chat Completions removed, `88598b9402` wire_api dropped.
- 2026-01-06 `1dd1355df3` agent controller; 2026-01-12 `86f81ca010` / `623707ab58` / `9659583559` collab tools (spawn/wait/close); renamed multi_agent 2026-02-16 `e41536944e` / `beb5cb4f48`; roles `e47045c806`; CSV agent jobs `dcab40123f` 2026-02-24.
- 2026-01-12 `d75626ad99` / `e726a82c8a` Responses over WebSocket.
- 2026-01-19 `d544adf71a` plan mode / collaboration modes + `request_user_input` → [[plan-mode]].
- 2026-01-22 `a2c829a808` connectors via MCP; 2026-01-23 `77222492f9` network sandbox proxy → [[egress-policy-proxy]].
- 2026-01-28 `3878c3dc7c` SQLite state (landed `eace7c6610` 2026-02-23) → [[sqlite-session-index]].
- 2026-02-04 `4922b3e571` memories phase-1 DB; extraction + consolidation `6049ff02a0` / `e57892b211` 2026-02-10 → [[cross-session-memory]].
- 2026-02-05 `3b54fd7336` hooks; 2026-02-10 `d735df1f50` hooks crate; PreToolUse `73bbb07ba8` 2026-03-23; PostToolUse `c4d9887f9a` 2026-03-25 → [[turn-lifecycle-hooks]].
- 2026-02-11 `42e22f3bde` js_repl (removed `8a559e7938` 2026-04-24, superseded by code mode).
- 2026-02-17/20 realtime voice WebSocket (`03ce01e71f`, `6817f0be8a`) → [[voice-frontend-delegation]].
- 2026-03-01 `752402c4fe` plugins. 2026-03-07 `e84ee33cc0` guardian approval MVP → [[llm-approval-reviewer]].
- 2026-03-08 `da3689f0ef` in-process app-server for exec. 2026-03-09 `da616136cc` code mode; V8 `e4eedd6170` 2026-03-20; standalone host `aa46f2debf` 2026-06-11; host-only `97576b1794` 2026-07-30 → [[code-mode]].
- 2026-03-12 `bc48b9289a` tool search (custom MCPs `d7f99b0fa6` 2026-04-09) → [[deferred-tool-loading]].
- 2026-03-16 `db89b73a9c` TUI on top of app-server; legacy split removed `d65deec617` 2026-03-27.
- 2026-03-19 → 2026-04-02 crate extraction from core (e.g. `2e03d8b4d2` rollout, `59b68f5519` codex-mcp, `6fff9955f1` models-manager).
- 2026-04-14 `dae56994da` ThreadStore; 2026-04-16 `a803790a10` model-provider runtime; 2026-04-20 `cefcfe43b9` Bedrock; 2026-04-21 `1cd3ad1f49` SigV4.
- 2026-04-24 `0ee737cea6` goals → [[persistent-goal-continuation]]. 2026-04-27 `bb83eec825` memories read/write split; memories MCP dropped `d579dafb70` 2026-05-26; built-in MCPs dropped `32b1ae7099` 2026-05-11.
- 2026-05-08 `0c8d42525e` app-server daemon lifecycle. 2026-05-11 `d2c3ebac1f` typed extension API (`codex-rs/ext/*`): guardian `d996f5366f`, memories `8ba6749932`, goal `a80f07ec4a`, web-search `a22706dfae`, image-generation `ecb41fcb64`, skills `2d385e166c`, mcp `4ec3b8eeea`, agent `6629e08702`, queue `bc8b25ea02`, guardian-v2 `fe614a6304` (consolidated `e741cd9ace` 2026-08-19) → [[extensibility-model]].
- 2026-08-21 `daa48072f4` history and notes tools for token-budget sessions → [[session-token-budget]]. 2026-09-05 `531f3836a1` `codex mcp-server` removed. 2026-09-21 `abbdde95b5` agent message board → [[agent-message-board]]. 2026-10-01 `b707714ae4` gRPC cloud thread resume/attach.

**Crate deaths** (`--diff-filter=D` on `Cargo.toml`): interactive `cca1122ddc` · repl `c432d9ef81` · legacy mcp-client `4cd6b01494` (→ rmcp) · git-apply `fa92cd92fa` · protocol-ts `42683dadfb` · utils/tokenizer `52d0ec4cd8` · tui2 `a489b64cb5` · mcp-types `891ed87409` · network-proxy-cli `c2c6bc90f8` · common split `8b7f8af343` · exec-server v1 `38f84b6b29` (reborn `81996fcde6` as remote execution server) · artifacts / artifact-presentation / artifact-spreadsheet / package-manager `6dcac41d53` / `2322e49549` / `2e849703cd` · legacy tui `d65deec617` · account `930e5adb7e` · instructions `4c2e730488` · device-key `e64a8979b0` · builtin-mcps `32b1ae7099` · memories/mcp `d579dafb70` · debug-client `fc8c723553` · execpolicy-legacy `656a2d0905` · first realtime-webrtc `b93dcf341c` · core-skills `45f8cafa4e` · ext/guardian `e741cd9ace` · mcp-server `531f3836a1`.

## Distinctive choices
- Fully stateless requests despite server-side storage being available: full context + encrypted reasoning every call (`591cb6149a`), WebSocket incremental append for cache/latency → [[cache-strategy]], [[no-server-stored-conversation]].
- Two compaction engines: server `/responses/compact` (opaque encrypted item) and local handoff summary. Plus the model-callable `new_context_window` → [[compaction-locus]].
- Context state as typed **world-state** sections re-sent only when their snapshot changes → [[world-state-diff-injection]].
- Guardian: approvals routed to a sandboxed LLM reviewer that sees authorship-labelled evidence → [[llm-approval-reviewer]], [[delegated-authorization-provenance]].
- Per-exec interception: patched shell routes every `execve` to the harness for per-subcommand policy → [[per-exec-interception]].
- Cross-session memories pipeline (extract → consolidate → inject) → [[cross-session-memory]].
- Native-scrollback TUI instead of full-screen redraw → [[terminal-scrollback-tui]].
- Feature flags with lifecycle stages gate almost every subsystem → [[feature-flag-stages]].

## Absences (see [[Absences]])
[[no-turn-cap]] · [[no-codebase-index]] · [[no-lsp]] · [[no-checkpoints-undo]] (undo removed) · [[no-file-read-write-tools]] · [[no-chat-completions-wire]] · [[no-typescript-cli]] · [[no-local-tokenizer]] · [[no-safe-command-allowlist]] · [[no-executable-plugins]] · [[no-external-code-contributions]]

## Tradeoffs vs pi
[[isolation-strategy]] · [[edit-format]] · [[dedicated-vs-shell-tools]] · [[prompt-ownership]] · [[compaction-locus]] · [[provider-breadth]] · [[cache-strategy]] · [[subagent-hosting]] · [[session-log-shape]] · [[tui-rendering-strategy]] · [[extensibility-model]]
