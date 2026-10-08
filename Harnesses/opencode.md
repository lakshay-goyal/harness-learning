---
type: harness
repo: https://github.com/anomalyco/opencode
commit: ecc4916b5a
language: TypeScript
studied: 2026-10-08
aliases: [OpenCode, opencode-ai, "@opencode-ai/opencode", sst/opencode]
---

Bun/TypeScript monorepo, 32 top-level `packages/` dirs (36 `package.json` files at depth ≤3), 15866 commits (2025-03-21 `4b0ea68d7a` → 2026-10-06), HEAD `ecc4916b5a` on `dev`, CLI version `1.18.35` (`packages/opencode/package.json:3`). "The open source AI coding agent" (`README.md:10`). Digest: [[2026-10-08-opencode]].

**Two runtimes coexist.** Every opencode note says which one it means.
- **Legacy runtime** (`packages/opencode/src/**`, ~77k LOC): the shipping CLI. It owns the loop today.
- **v2 runtime** (`packages/core/src/**` + `packages/llm/src/**`): built spec-first from `specs/v2/**` and the `CONTEXT.md` glossary, starting at `76ee87ead8` (2026-06-03). It is mounted in the same server process (`packages/opencode/src/server/routes/instance/httpapi/server.ts:298-303`).
- **Unmerged branch** `origin/v2`: carries a third prompt architecture. It is cited as `origin/v2:<path>`.

## Ideology
- **Batteries included, provider-agnostic.** The harness ships a rich default toolset:
  - shell, read, glob, grep, edit, write, apply_patch, task, todowrite, webfetch, websearch, skill, question, plan, lsp, and code-mode `execute` (`packages/opencode/src/tool/registry.ts:121-253`)
  - built-in agents build, plan, general, explore, compaction, title, summary (`packages/opencode/src/agent/agent.ts`)
  - permission prompts, LSP, formatters, MCP, plugins, skills, share, and undo via snapshots.
  - Contrast: pi's minimal core, [[minimal-default-toolset]].
  - See [[agent-profiles]] · [[permission-ruleset]] · [[plan-mode]] · [[web-tools]] · [[task-list-tool]] · [[lsp-diagnostics-feedback]] · [[workspace-snapshots]].
- **Fit the model, not one prompt.**
  - The base system prompt is picked per model family by model-id substring: muse → meta, gpt-4/o1/o3 → beast, gpt-6 → gpt-astra, gpt+codex → codex, other gpt → gpt, gemini, claude → anthropic, trinity, kimi/moonshot → kimi, else default (`packages/opencode/src/session/system.ts:28-50`). See [[per-model-system-prompt]].
  - GPT-family models (`gpt-` ids except `oss` and `gpt-4`) get `apply_patch` instead of edit/write (`packages/opencode/src/tool/registry.ts:297-300`). See [[model-specific-toolset]] · [[patch-envelope-edit]].
  - A per-provider transform layer is the #1 fix hotspot: `packages/opencode/src/provider/transform.ts` (162 fix commits) + `packages/opencode/src/provider/provider.ts` (129). See [[transcript-replay-repair]] · [[tool-schema-lowering]].
- **The database is the conversation; the server is the product.**
  - There is no in-memory transcript. Every step re-reads SQLite and rebuilds the request (`packages/opencode/src/session/prompt.ts:1081-1341`).
  - Writes are events projected into rows (`packages/core/src/session/projector.ts:260-330`). v2 makes the store fully event-sourced. See [[context-projection]] · [[event-sourced-session-store]].
  - The TUI, desktop app, web app, ACP bridge, GitHub agent and SDK are all clients of one HTTP + SSE server. See [[client-server-session-split]] · [[location-scoped-runtime]].

## Organ map
| Organ | Legacy runtime | v2 runtime | Concepts |
|---|---|---|---|
| Loop | `SessionPrompt.runLoop`, `packages/opencode/src/session/prompt.ts:1081-1341`. `while(true)`: reload history, run exit test, run one processor step (`packages/opencode/src/session/processor.ts`). The exit test is "finished, finish ∉ {tool-calls, unknown}, no pending tool parts, answers latest user" (`packages/opencode/src/session/prompt.ts:1100-1129`). Steps cap at `agent.steps ?? Infinity` (`packages/opencode/src/session/prompt.ts:1178`). The doom loop trips at 3 identical calls and then asks the user (`packages/opencode/src/session/processor.ts:29,356-373`). | Session Drain in `packages/core/src/session/runner/llm.ts:392-415`. The inner loop continues while local tool calls or steers exist. The outer loop promotes one queued input at a time. Tools execute eagerly while the stream is still open. There is no doom-loop check and no session retry (TODO `packages/core/src/session/runner/llm.ts:55`). | [[turn-loop]] · [[step-budget-limit]] · [[repeated-tool-call-detection]] · [[steering-queue]] · [[follow-up-queue]] · [[auto-retry-backoff]] |
| Message builder | `packages/opencode/src/session/llm/request.ts` + `MessageV2.toModelMessagesEffect` (`packages/opencode/src/session/message-v2.ts`). System = per-model prompt + environment block (has the date) + instruction files + skills/MCP instructions. Mode reminders go into the latest user message. `ProviderTransform.message` normalizes per provider. | System Context = typed **Context Sources**. A **Context Epoch** baseline is immutable for cache. Changes arrive as **Mid-Conversation System Messages** at the Safe Provider-Turn Boundary (`CONTEXT.md:7-37`, `packages/core/src/system-context/index.ts`). | [[message-conversion-layer]] · [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[ephemeral-reminder-injection]] · [[context-file-hierarchy]] · [[mention-expansion]] |
| Tools | `ToolRegistry` (`packages/opencode/src/tool/registry.ts`). Built-ins + `tool/*.ts` from config dirs + plugin tools. Filtering is permission-driven at request time. The AI SDK owns dispatch. Unknown tools go to the `invalid` tool. Output truncates at 2000 lines / 50 KB and spills to a file. | One opaque Tool type with one executor. Tool output is bounded at one settlement boundary with managed spill files (`packages/core/src/tool-output-store.ts`). Leaf tools own their permission requests. | [[tool-output-truncation]] · [[tool-output-spill]] · [[tool-error-as-result]] · [[search-replace-edit]] · [[fuzzy-edit-matching]] · [[plugin-tools]] · [[mcp-integration]] · [[code-mode]] |
| Provider | Vercel AI SDK `streamText`, plus an opt-in native runtime behind `OPENCODE_EXPERIMENTAL_NATIVE_LLM` (`packages/opencode/src/session/llm.ts:85-381`). models.dev catalog with a 5-min disk TTL. Vendored SDK patches in `patches/`. | `packages/llm` Route = Protocol × Endpoint × Auth × Framing. The runner supports only OpenAI Responses, Anthropic Messages and OpenAI-compatible Chat (`packages/core/src/session/runner/model.ts:142-179`). Cache policy is "auto" (`packages/llm/src/cache-policy.ts`). | [[unified-provider-api]] · [[model-catalog]] · [[signed-reasoning-replay]] · [[cross-provider-handoff]] · [[sampling-parameter-defaults]] · [[cache-breakpoint-placement]] |
| Session store | One global SQLite DB (WAL) with JSON blobs per message and part (`packages/core/src/session/sql.ts:22-98`). Every write is an event publish that a projector upserts. Text deltas are never persisted. Legacy JSON storage is migrated. | Typed, versioned durable events per session aggregate, written in one immediate transaction that also runs the projectors (`packages/core/src/event.ts:205-358`). Ordering is by seq, not timestamp. | [[session-migration]] · [[partial-message-persistence]] · [[event-sourced-session-store]] · [[session-fork]] |
| Context mgmt | Compaction is a user message with a `compaction` part, processed by the loop. It uses an anchored summary template. The trigger is `limit.input − min(20k, maxOutput)` or `context − maxOutput` (`packages/opencode/src/session/overflow.ts:8-33`). Overflow → compact → replay the last user turn. Tool-output pruning is OFF by default since 2026-04. | Runs before every provider turn. The checkpoint is a structured summary + serialized recent text, `DEFAULT_KEEP_TOKENS = 8_000` (`packages/core/src/session/compaction.ts:12-15`). One physical retry on overflow. | [[auto-compaction]] · [[overflow-recovery]] · [[iterative-summary-update]] · [[structured-compaction-summary]] · [[tool-output-pruning]] |
| Undo | A shadow git repo per project/worktree, snapshotted at each step. Revert is a soft marker, committed on the next prompt. | Same shadow-git design. | [[workspace-snapshots]] |
| Cancellation | `POST /session/:id/abort` → `SessionRunState.cancel` → cancels background jobs transitively (subagents) → `Fiber.interrupt` → scoped `AbortController` aborts the fetch and tools. The shell gets force-killed after 3 s. `cleanup()` marks running tools interrupted. Partial shell output is replayed to the model **as a successful result** (`packages/opencode/src/session/message-v2.ts:338-349`). | Effect interruption only, with no AbortController plumbing (`specs/v2/tools.md:52`). Buffered fragments are flushed as durable `*.ended` events. | [[abort-propagation]] · [[process-tree-kill]] · [[partial-message-persistence]] |
| Safety | Permission rules (allow / ask / deny, last match wins). Defaults: `*: allow`, `doom_loop: ask`, `external_directory: ask`, `*.env` read asks (`packages/opencode/src/agent/agent.ts:119-135`). Bash is parsed with tree-sitter for per-command patterns. No sandbox. | `Policy` statements; the build agent ruleset starts with `*:*:allow` (`packages/core/src/plugin/agent.ts:109`). Rejection feedback is collapsed into a generic tool error. | [[permission-ruleset]] · [[shell-command-permission-parsing]] · [[workspace-boundary-check]] · [[project-trust-gate]] · [[secret-handling]] |
| Delegation | The `task` tool spawns a child session with a subagent profile. Background jobs are keyed by parent session. The plan agent may not spawn `general`. | v2 not yet. | [[task-owned-subagent]] · [[agent-profiles]] · [[plan-mode]] |

## Packages (harness-relevant)
| Package | Role |
|---|---|
| `opencode` | Legacy CLI runtime: session loop, tools, provider transforms, agents, permission, LSP, MCP, plugins, server, TUI (`src/cli/cmd/tui`), ACP, GitHub agent, share, snapshots, worktree |
| `core` | v2 runtime: session runner and projector, system-context, tool runtime and output store, policy, catalog, database, events, plus v1 schemas shared with legacy |
| `llm` | v2 provider layer: routes, protocols, cache policy, provider errors |
| `codemode` | Own interpreter (no QuickJS/V8) exposed as one `execute` tool; legacy passes no limits (`packages/opencode/src/tool/code-mode.ts:239-260`) → [[code-mode]] |
| `server`, `protocol`, `schema`, `client`, `sdk`, `sdk-next` | HttpApi contract, event schemas, generated clients → [[client-server-session-split]] · [[sdk-embedding]] · [[agent-event-stream]] |
| `plugin` | `@opencode-ai/plugin` hook types: `chat.params`, `tool.execute.before/after`, `experimental.chat.system.transform`, … → [[extension-event-hooks]] |
| `tui`, `app`, `desktop`, `web`, `console`, `ui`, `session-ui` | Clients and hosted gateway (out of scope except where they shape runs) |

## Distinctive choices
- Mid-run user input needs no queue object. It is persisted and picked up by the next step's history reload. A wrapper marking queued messages was removed because it broke the cache (`f092bafe88`). See [[steering-queue]].
- Repetition guard is a permission ask (`doom_loop`), not an abort (`d983b9485d`, 2025-10-30). See [[repeated-tool-call-detection]].
- The legacy step cap is enforced only by prompt: "CRITICAL - MAXIMUM STEPS REACHED … Tools are disabled", yet tools have stayed enabled since `fed4776451` (2025-12-14). v2 sets `toolChoice: "none"`. See [[step-budget-limit]].
- User-triggered shell commands and `/subtask` are recorded as synthetic tool calls. See [[synthetic-tool-call-injection]].
- Native mid-conversation `role:"system"` is used only for `claude-opus-4-8`. Other models get escaped `<system-update>` user text. See [[transcript-carried-system-prompt]].
- The Claude Code impersonation prompt and bundled Claude Pro/Max login were removed after "anthropic legal requests" (`1ac1a0287c`). See [[provider-identity-shim]] · [[subscription-oauth-auth]].
- Fuzzy edit block-anchor matching accepted similarity 0.0 until `236cfcbbc3` (2026-06-05), so it could replace whole regions. See [[fuzzy-edit-matching]].
- Numbers identical to pi: 2000 lines / 50 KB truncation, 2000 px image cap, JPEG ladder 80/85/70/55/40, OAuth port 1455. Shared lineage is (unverified). See [[Constants]].

## Fix hot-spots
| fixes | file |
|---|---|
| 162 | `packages/opencode/src/provider/transform.ts` |
| 129 | `packages/opencode/src/provider/provider.ts` |
| 123 | `packages/opencode/src/session/prompt.ts` |
| 93 | `packages/opencode/src/session/index.ts` (historical; deleted `f25f1485d5` 2026-04-27, session service now `packages/opencode/src/session/session.ts`) |
| 90 | `packages/opencode/src/config/config.ts` |
| 56 | `packages/opencode/src/session/message-v2.ts` |
| 51 | `packages/opencode/src/mcp/index.ts` |
| 38 | `packages/opencode/src/snapshot/index.ts` |

The most revert-prone class is replay shape: reasoning pairing `8c2aec43b8`→`3c3d6b65c2`, empty blocks `d8a15e7bc9`→`d69366b00c`, tool-attachment placement `8fd1b92e6e`→`f5a6a4af7f`, signed-thinking reorder `42173bca4b`→`a763a14d44`. Stream idle timeout: 2 min → 5 min → off (`d69962b0f7`) → 5 min (`4eb29a64f0`).

## Spec ↔ code drift (v2)
- Spill-write failure: `CONTEXT.md:194` says lossy success; `specs/v2/tools.md:157` and the code report it as a failure.
- Session move: the spec says "clear epoch"; `d8bf79225f` (2026-08-13) stopped clearing it (`packages/core/src/session/projector.ts:242-256`).
- `specs/v2/session.md:204` says "ask by default", but the build ruleset starts with `*:*:allow` (`packages/core/src/plugin/agent.ts:109`).
- Cost: `packages/llm/DESIGN.md:677-680` says "unavailable", but the code writes `cost: 0`.

## Absences
Shared with pi: [[no-sandbox]] · [[no-background-bash]] · [[no-codebase-index]] · [[no-prompt-injection-defense]] · [[no-auto-hot-reload]] · [[no-read-before-write-guard]].
Specific to opencode: [[no-project-trust-gate]] · [[no-claude-subscription-auth]] · [[no-batch-tool]] · [[no-model-initiated-plan-entry]] · [[no-background-task-polling]] · [[removed-builtin-tools]] · [[no-lsp-formatters-by-default]] · [[no-hardcoded-secret-refusal]] · [[no-codemode-default-limits]] · [[v2-rejected-designs]].
Index: [[Absences]].

## Tradeoffs vs pi
[[permission-prompts-vs-none]] · [[plan-mode-vs-none]] · [[builtin-subagents-vs-none]] · [[todo-tool-vs-none]] · [[lsp-feedback-vs-none]] · [[web-tools-vs-none]] · [[minimal-vs-rich-toolset]] · [[single-vs-per-model-system-prompt]] · [[session-store-format]] · [[undo-vs-none]] · [[cwd-confinement-vs-none]] · [[turn-cap-vs-none]] · [[bash-timeout-default-vs-none]] · [[mid-run-user-input]] · [[compaction-design]] · [[prompt-cache-strategy]] · [[mcp-builtin-vs-extension]] · [[edit-tool-variants]] · [[client-server-vs-single-process]]. Index: [[Tradeoffs]].
