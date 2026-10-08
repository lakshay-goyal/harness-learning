---
type: group
group: absences
---
# Absences

Deliberate non-features and removed designs, with the rationale and the opt-in path. pi ([[pi]], `b30a6dd77`) is the reference harness; opencode ([[opencode]], `ecc4916b5a`) is compared in [[#opencode stance]]; codex ([[codex]], `622e9e3696`) in the codex column/sections. A note's `harnesses:` lists only harnesses where the absence holds; other harnesses are discussed in a `**codex**` / pi / opencode block inside the note. Design axes where harnesses diverge → [[Tradeoffs]].

**Stance at HEAD**
- "Pi ships with powerful defaults but skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package that does it your way." (`README.md:19`, wording `de7e675de`, 2026-10-02).
- Governance: "**pi's core is minimal**. If your feature does not belong in the core, it should be an extension. PRs that bloat the core will likely be rejected." (`CONTRIBUTING.md:7-9`). Even hook points "should be well considered … to avoid unmaintainable bloat" (`CONTRIBUTING.md:11`).

**Manifesto lineage**
1. **Nov 2025**: separate README sections, all 2025-11-12 by Mario Zechner. Each used "pi does not and will not …": sub-agents `e9935beb5`, background bash `271810c80`, to-dos `9066f58ca`, YOLO/security `b172beb92`, MCP `60e4fcf01`.
2. **Dec 2025**: "Philosophy" section (`3424550d2`, 2025-12-17). Six one-liners, including "No permission popups. Security theater."
3. **Sep 2026**: Philosophy text at `25cc5c7bf^:packages/coding-agent/README.md:533-549`, softened ("or build your own with extensions, or install a package").
4. **2026-09-22**: the section was **deleted** in docs refresh `25cc5c7bf` (#9898, Christian Klotz), one week before MCP shipped (`8562bcf66`). Whether the deletion was editorial or a softening of stance is unverified.

## Notes
| Absence | pi status at HEAD | pi opt-in path | codex (`622e9e3696`) |
|---|---|---|---|
| [[no-subagents-core]] | absent in stable; **present in experimental durable** (`2532a0bef`) | `examples/extensions/subagent/` | present — in-process threads, `multi_agent` Stable default on |
| [[no-plan-mode]] | absent | `examples/extensions/plan-mode/` | present — Plan collaboration mode + `update_plan` |
| [[no-permission-prompts]] | absent (YOLO) | `tool_call` gate + `permission-gate.ts`, `protected-paths.ts` | present — approval policies, execpolicy, Guardian reviewer |
| [[no-sandbox]] | absent | Docker / Docker Sandboxes / gondolin / `sandbox/` / `ssh.ts` | present — OS sandbox (Seatbelt/Landlock+bwrap/Windows) |
| [[no-todo-tool]] | absent | `todo.ts`, TODO.md | present, opt-in since `a9519cbcdd` 2026-08-31 |
| [[no-background-bash]] | absent | tmux, `interactive-shell.ts` | present — unified exec PTY sessions (reversed) |
| [[no-web-tools]] | absent | CLI + README skills, MCP | partial — `web_search`; no URL fetch (unverified) |
| [[no-turn-cap]] | absent | `finishTurn` / `turn_end` hook | **absent** — token budgets / goal breakers instead |
| [[no-cwd-confinement]] | absent | `protected-paths.ts`, container | present — writable roots + protected metadata |
| [[no-date-in-prompt]] | absent (removed `f4e9ca746`) | `APPEND_SYSTEM.md`, bash `date` | present since `90cc4e79a2` (world state, not base prompt) |
| [[no-builtin-mcp-reversed]] | **reversed**: replaceable built-in since `8562bcf66` | `-builtin:mcp` to opt out | present since 2025-05; built-in MCPs dropped `32b1ae7099` |
| [[no-lsp]] | absent (omission) | none shipped | **absent** |
| [[no-codebase-index]] | absent (omission) | context files, `claude-rules.ts`, MCP | **absent** |
| [[no-checkpoints-undo]] | absent | `git-checkpoint.ts`, `auto-commit-on-exit.ts` | **absent by removal** (`/undo` un-shipped `7a8407bbb6`) |
| [[no-prompt-injection-defense]] | out of scope (`SECURITY.md:19-22`) | containment, gates | partial — Guardian + authorization provenance |
| [[no-bash-default-timeout]] | absent (removed `29900ce64`) | model-passed `timeout`, spawn hook | **partial** — 10 s one-shot exec; no kill timeout in unified exec |
| [[no-auto-hot-reload]] | absent (manual `/reload`) | `reload-runtime.ts` | partial — watches skills/fs, reloads config (data only) |
| [[no-binary-detection-in-read]] | observed gap | `tool-override.ts` | n/a — no read tool |

Reading the table:
- **Stated** (manifesto): subagents, plan mode, permission prompts, sandbox, todos, background bash, MCP (reversed), web tools.
- **Design-implied**: turn cap, cwd confinement, bash timeout, prompt-injection defense, date (removed for cache), hot reload.
- **By omission**: LSP, codebase index, checkpoints, binary detection.

## Reversals (absent → present)
| Feature | Absent | Present | Note |
|---|---|---|---|
| MCP | `60e4fcf01` 2025-11-12 "pi does not support MCP" | `8562bcf66` 2026-09-29 replaceable built-in | token-cost objection answered by codemode/deferred exposure → [[no-builtin-mcp-reversed]] |
| Sub-agents | `e9935beb5` "will not" | `2532a0bef` 2026-09-29 durable subagent handles; `experimental/durable/subagent.ts` tool | experimental only → [[no-subagents-core]] |
| Auto-compaction | "Planned Features" in `e9935beb5`: "watch the context percentage … When it approaches 80%, ask the agent to write a summary .md file" | `6c2360af2` 2025-12-04 (#92) | → [[auto-compaction]] |
| Local models | "Planned Features" in `e9935beb5` | llama.cpp built-in extension (`packages/coding-agent/src/extensions/index.ts:8`) | → [[model-catalog]] |
| AGENTS.md trust gating | ungated | gated `89a92207f` 2026-06-05 → **ungated again** `5cb4f597f` 2026-06-09 | present → absent → [[no-prompt-injection-defense]] |
| Date in prompt | present `b1c2c32e2` 2025-11-12 | removed `f4e9ca746` 2026-07-14 | present → absent → [[no-date-in-prompt]] |
| bash 30s timeout | present `ffc9be886` | removed `29900ce64` 2025-11-12 | present → absent → [[no-bash-default-timeout]] |

No built-in agent tool has ever been deleted. `git log --diff-filter=D` on `packages/coding-agent/src/**/tools/` shows only the hooks/custom-tools merge.

## Removed features & packages
| Hash | Date | What | Why (from commit/changelog) |
|---|---|---|---|
| `aa005d062` | 2025-10-06 | `packages/browser-extension` removed | "migrated to separate sitegeist repo" |
| `92bad8619` | 2025-11-10 | `packages/agent-old` | "Removed agent-old" (no rationale) |
| `29900ce64` | 2025-11-12 | bash 30s default timeout | "make bash tool timeout optional" → [[no-bash-default-timeout]] |
| `8b1cca827` | 2025-11-27 | prompt identity "You are actually not Claude, you are Pi." | "Models now use their native identity" (#73) → [[harness-identity]] |
| `0f98decf6` | 2025-12-28 | `packages/proxy` | "proxy functionality is now handled by web-ui's createStreamFn with external proxy servers" |
| `c6fc08453` | 2026-01-05 | `core/hooks/*` + `core/custom-tools/*` deleted | merged into unified extensions system (#454) → [[extension-event-hooks]] |
| `e3dd4f21d` | 2026-01-08 | prompt rule "You are in READ-ONLY mode…" | no message; extension tool overrides made it invalid (inferred) → [[no-plan-mode]] |
| `d9464383e` | 2026-01-16 | example `load-file.ts` | test extension |
| `4068bc556` | 2026-01-17 | `pi-internal://` doc scheme (added `012319e15` the day before) and `PI_STATIC_INSTRUCTIONS` | replaced by absolute doc paths in `<docs>` → [[self-documentation-pointer]] |
| `b846a4bfc` | 2026-01-20 | prompt rule "Use bash ONLY for read-only operations…" | silent (#645) |
| `866d21c25` | 2026-01-22 | example `pi-dosbox` | moved to github.com/badlogic/pi-dosbox |
| `c5c515f56` | 2026-01-23 | example chalk-logger | "breaks TUI by using console.log directly" |
| `235b247f1` | 2026-03-22 | prompt rule "When summarizing your actions, output plain text directly - do NOT use cat or bash…" | silent refactor |
| `0ed0d4343` | 2026-04-30 | `packages/mom` (Slack bot) + `packages/pods` | "People should check out pi-chat … or use an older commit for mom and fork" |
| `fe66edd94` | 2026-04-30 | Google Gemini CLI + Antigravity providers/OAuth | changelog only states removal (`packages/ai/CHANGELOG.md:1119`; `packages/coding-agent/CHANGELOG.md:2119`); reason unverified |
| `8db0d2838` | 2026-05-01 | example `custom-provider-qwen-cli` | no rationale |
| `b141e1fa2` | 2026-05-20 | `packages/web-ui` workspace | "chore: remove web-ui workspace" (no rationale) |
| `1ab289980` | 2026-05-28 | prompt rule "Prefer grep/find/ls tools over bash…" | preferred unavailable tools (#5132) → [[dynamic-tool-guidelines]] |
| `5cb4f597f` | 2026-06-09 | trust gating of AGENTS.md/CLAUDE.md | "Project trust is only an input-loading guard" → [[no-prompt-injection-defense]] |
| `f4e9ca746` | 2026-07-14 | `Current date:` in system prompt | cache invalidation across dates (#6621) → [[no-date-in-prompt]] |
| `b70c0f5b4` | 2026-08-02 | reverted switchable terminal renderers (#7440) | revert (#7473) |
| `9e05370b2` | 2026-09-16 | example `kimi-deferred-tools.ts` | replaced by mid-conversation system messages (#9548) (inferred) → [[transcript-carried-system-prompt]] |
| `25cc5c7bf` | 2026-09-22 | README "Philosophy" manifesto (No MCP/sub-agents/…) | docs refresh #9898; one week before MCP landed |
| `7fd478a2e` | 2026-10-01 | experimental harness in pi-agent-core (sessions, pico3, compaction, skills, telemetry schemas), `packages/session-backends`, experimental mini/micro frontends | "Durable sessions live in @earendil-works/pi-durable" → [[durable-execution]] |
| `1c1e9c0ef` | 2026-10-02 | daxnuts easter egg | — |

## Absent feature → shipped example
- sub-agents → `subagent/`
- plan mode → `plan-mode/`
- approvals → `permission-gate.ts`, `protected-paths.ts`, `confirm-destructive.ts`, `dirty-repo-guard.ts`
- sandbox → `sandbox/`, `gondolin/`, `ssh.ts`
- todos → `todo.ts`
- checkpoints → `git-checkpoint.ts`, `auto-commit-on-exit.ts`
- model routing → `jev-router.ts`, `preset.ts` ([[virtual-model-router]])
- ask-user → `question.ts`, `questionnaire.ts`
- long processes → `interactive-shell.ts`
- compaction alternatives → `handoff.ts`, `custom-compaction.ts`, `trigger-compact.ts`
- terminating output → `structured-output.ts`

All under `packages/coding-agent/examples/extensions/`. **No example** for web search/fetch, LSP, embeddings/RAG, or background bash.

## Other extension-shaped (not absent, not core)
- Model routing/fallback: core has the virtual-model mechanism (`docs/virtual-models.md:3-5`, `540e174c7` #10035); the router exists only as the `jev-router.ts` example. There is no client-side cross-provider failover, only agent retry ([[auto-retry-backoff]]) and Anthropic server-side fallback metadata `compat.allowedFallbackModels` (`b03a367a4`; `packages/ai/src/types.ts:975`) → [[server-side-refusal-fallback]].
- Ask-user tool (AskUserQuestion analogue): only the `question.ts` / `questionnaire.ts` examples.

## codex

**Stance at HEAD** — no manifesto of non-features; absences are mostly *removals* recorded as `Stage::Removed` no-op flags (41 of 167 flags, `codex-rs/features/src/lib.rs`; [[feature-flag-stages]]) and as deleted crates. Governance: "**We do not accept external code contributions or pull requests.**" (`docs/contributing.md:5`) → [[no-external-code-contributions]]. Inverse of pi: safety, sub-agents, plan mode, background processes are core; *third-party code* is what is absent ([[no-executable-plugins]], [[extensibility-model]]).

### codex absence notes
| Absence | Group | Kind | Key evidence |
|---|---|---|---|
| [[no-prompt-and-agent-hooks]] | loop / platform | not yet (schema only) | `codex-rs/hooks/src/engine/discovery.rs:637-656` |
| [[no-steer-into-review-or-compact]] | loop | design | `codex-rs/core/src/session/turn_input.rs:824-835` |
| [[no-chat-completions-wire]] | model | removed `d2394a2494` 2026-02-03 | `codex-rs/model-provider-info/src/lib.rs:100-133` |
| [[no-server-stored-conversation]] | model / state | design `591cb6149a` | `codex-rs/core/src/client.rs:961` |
| [[no-output-token-cap]] | model | reverted attempt `c9e149fd5c`/`bce030ddb5` | `codex-rs/codex-api/src/common.rs:279-304` |
| [[no-client-price-table]] | model / cost | design | `codex-rs/app-server/src/turn_cost_worker.rs` |
| [[no-cache-miss-detection]] | caching | omission (unverified) | `codex-rs/otel/src/events/session_telemetry.rs:1139` |
| [[no-plugin-providers]] | model | stated | `codex-rs/model-provider-info/src/lib.rs:667-670` |
| [[no-file-read-write-tools]] | tools | removed `14c35a16a8`, `178c3b15b4`, `70807730f5` | `codex-rs/core/src/tools/spec_plan.rs:1082-1372` |
| [[removed-legacy-shell-tools]] | tools | removed `83decfa300`, `e783341b70`, `8a40095ea3` | unified exec only |
| [[no-kill-timeout-in-unified-exec]] | tools | design; experiment reverted `9719dc502c` | `codex-rs/core/src/session/handlers.rs:305-312` |
| [[no-transactional-multi-file-patch]] | tools | observed (weak) | `codex-rs/apply-patch/src/lib.rs:438-465` |
| [[no-per-file-mutation-queue]] | tools | removed gating `862b2122ee` | `e95abcdf49:codex-rs/core/src/tools/parallel.rs:207-217` |
| [[no-safe-command-allowlist]] | safety | removed `942af8447b` 2026-08-19 | 536 + 503 lines deleted |
| [[no-on-failure-approval-mode]] | safety | removed `2cf2a6a844` 2026-06-23 | `codex-rs/protocol/src/protocol.rs:1036` |
| [[no-cli-process-hardening]] | safety | removed `d3ff668f68` 2026-01-08 | `codex-rs/process-hardening/src/lib.rs:12-130` |
| [[no-user-selectable-personality]] | prompting | retired `132c739171` 2026-09-12 | `d63a9b8344` |
| [[no-offline-per-model-prompts]] | prompting | removed `a1abd53b6a` 2026-02-09 | orphaned `codex-rs/core/gpt*_prompt.md` |
| [[no-collaboration-styles]] | prompting | removed `31415ebfcf`, `d509df676b` | Default + Plan only |
| [[no-top-level-instructions-field]] | prompting / caching | changed `c9253c4977` 2026-10-05 | `codex-rs/core/src/client.rs:996-1011` |
| [[no-per-file-context-labels]] | prompting / context | design | `codex-rs/core/src/agents_md.rs:386-419` |
| [[no-structured-compaction-template]] | context | dropped `ea225df22e` 2025-09-12 | `codex-rs/prompts/templates/compact/prompt.md:1-9` |
| [[no-standalone-patch-format-doc]] | tools / prompting | deleted `8d637ae398` 2026-08-13 | grammar only |
| [[no-in-turn-overflow-retry]] | context | hardening reverted `15e79f3c26`→`69f3183a8e` | `codex-rs/core/src/session/turn.rs:1688-1691` |
| [[no-summary-validation-local]] | context | design | `codex-rs/core/src/compact.rs:752-756` |
| [[no-tool-output-spill-file]] | context | design | `codex-rs/core/src/context_manager/history.rs:514-515` |
| [[no-partial-history-fork]] | state / subagents | removed `6221a217e2` 2026-10-06 | `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302` |
| [[no-subagent-depth-limit-v2]] | subagents | design `70ac0f123c` | `codex-rs/core/src/tools/spec_plan.rs:726-729` |
| [[no-subagent-workspace-isolation]] | subagents | design | `codex-rs/core/src/agent/child_config.rs` |
| [[no-csv-fanout-jobs]] | subagents | removed `687f05cb94` 2026-07-20 | `codex-rs/features/src/lib.rs:1466` |
| [[no-awaiter-role]] | subagents | temp removed `fe439afb81` | `codex-rs/core/src/agent/role.rs:386` |
| [[no-delegate-approvals]] | subagents / safety | design | `codex-rs/core/src/codex_delegate.rs:64-73` |
| [[no-executable-plugins]] | platform | design | `codex-rs/plugin/src/manifest.rs:8-58` |
| [[no-strict-jsonrpc]] | platform | inherited from MCP | `codex-rs/app-server-protocol/src/rpc.rs:1-2` |
| [[no-typescript-cli]] | platform | removed `408c7ca142` 2025-08-08 | `codex-cli/` = launcher only |
| [[no-local-tokenizer]] | context | removed `52d0ec4cd8` 2025-11-20 | `codex-rs/utils/string/src/truncate.rs:4` |
| [[no-in-repo-user-docs]] | platform | design | `docs/*.md` ≈ 206 lines of pointers |
| [[no-external-code-contributions]] | governance | stated `31f23b6022` | `docs/contributing.md:5-11` |

Shared with pi (absent in both): [[no-turn-cap]] · [[no-lsp]] · [[no-codebase-index]] · [[no-checkpoints-undo]] (codex by removal) · [[no-bash-default-timeout]] (codex partial).

Folded (no separate note): general URL fetch — only `open_page` as a web-search action, no `web_fetch` handler (unverified beyond grep) → [[no-web-tools]].

### codex removed designs (no separate note)
| Hash | Date | What | Why (from commit) |
|---|---|---|---|
| `cca1122ddc` / `c432d9ef81` | 2025-04-28 / 04-30 | `interactive` crate and REPL subcommand | Rust TUI became the default |
| `c4af707e09` | 2025-12-10 | LLM command risk assessment (40 files, 703 deletions) | "received lukewarm reception during internal testing"; successor Guardian `e84ee33cc0` 2026-03-07 → [[llm-approval-reviewer]] |
| `7a8407bbb6` | 2025-12-22 | ghost-commit `/undo` | after `--staged` data-loss bug → [[no-checkpoints-undo]] |
| `a489b64cb5` | 2026-01-21 | tui2 alternative frontend (`0c8828c5e2` 2025-12-09) | retired; TUI rebuilt on app-server `db89b73a9c` → [[terminal-scrollback-tui]] |
| `38c442ca7f` → `ba5b94287e` / `77b0c75267` | 2026-02-13 → 03-11 | Apps-only `search_tool_bm25` | replaced by `tool_suggest` + Responses "bring your own" tool search → [[deferred-tool-loading]] |
| `58ac2a8773` | 2026-03-18 | live memory editing | "nit: disable live memory edition" → [[cross-session-memory]] |
| `2322e49549` / `b00a05c785` / `6dcac41d53` | 2026-03-04 → 03-26 | artifacts (presentation/spreadsheet tools, crates born 2026-03-03) | dropped |
| `930e5adb7e` | 2026-04-10 | usage-limit "notify workspace owner" (`account` crate) | reverted next day |
| `05c5829923` → `e3c2acb9cd` | 2026-04-13 → 04-18 | mailbox delivered only at request boundaries | reverted (#18325); re-offered opt-in `DeferMailboxPreemption` `766d2377a8` 2026-09-24 (`codex-rs/features/src/lib.rs:1446-1451`, default off) → [[subagent-result-mailbox]] |
| `8a559e7938` | 2026-04-24 | js_repl persistent Node REPL tool (`42e22f3bde` 2026-02-11; 63 files, 9,261 deletions) | superseded by V8 code mode `e4eedd6170`; `JsRepl` no-op flag (`codex-rs/features/src/lib.rs:429-430`) → [[code-mode]] |
| `4e05f3053c` | 2026-04-27 | ghost snapshots in Responses API surface | "make undo a no-op that reports the feature is unavailable" |
| `32b1ae7099` / `d579dafb70` | 2026-05-11 / 05-26 | built-in MCPs (`ca257b6ce5` 2026-05-06) / memories MCP | "Drop something that was never used" → [[no-builtin-mcp-reversed]] |
| `fd72e99384` | 2026-05-22 | legacy `[profiles.*]` config tables | profiles v2 = separate files → [[layered-settings]] |
| `656a2d0905` | 2026-07-10 | legacy execpolicy engine | v2 rules → [[command-rule-policy]] |
| `8431dc590a` | 2026-07-20 | invalid-image auto-repair retry ("Invalid image" text + retry) | now a hard bad-request error (`codex-rs/core/src/session/turn.rs:793-816`) → [[image-normalization]] |
| `86b1123ff6` | 2026-08-14 | per-model `supports_parallel_tool_calls` | parallelism always requested; harness lock constrains → [[parallel-tool-execution]] |
| `3052bbcf8c` | 2026-09-11 | `thread/rollback` in-place `ThreadRolledBack` markers (`8b7ec31ba7` 2026-01-06) | replaced by `thread/revert` (new rollout file per revert); markers still replayed → [[session-fork]] |
| `531f3836a1` | 2026-09-05 | `codex mcp-server` (Codex as an MCP server) | superseded by app-server → [[client-server-session-split]] |
| side branch `edd46c6347` `99b78c9bb2` `e39e0c4332` `e67726a994` `71163530a4` | — | compaction-prompt experiments (incl. strict-JSON summary) | never merged to mainline (rejected experiments) → [[no-structured-compaction-template]] |

Crate deaths (complete, `--diff-filter=D` on Cargo.toml, M8): interactive, repl, mcp-client (legacy stdio), git-apply (merged), protocol-ts, utils/tokenizer, tui2, mcp-types, network-proxy-cli, common (split), exec-server v1 (reborn `81996fcde6`), artifacts ×3 + package-manager, legacy tui `d65deec617`, account, instructions `4c2e730488`, device-key `e64a8979b0`, builtin-mcps, memories/mcp, debug-client `fc8c723553`, execpolicy-legacy, realtime-webrtc (first, `b93dcf341c`), core-skills `45f8cafa4e`, ext/guardian `e741cd9ace`, mcp-server.

### codex reversals
| Feature | First | Then | Note |
|---|---|---|---|
| Chat Completions | added `e924070cee` 2025-05-08 | removed `d2394a2494` 2026-02-03 | → [[no-chat-completions-wire]] |
| `/undo` checkpoints | default-on `052b052832` 2025-11-11 | un-shipped `7a8407bbb6` 2025-12-22 | → [[no-checkpoints-undo]] |
| read/grep/list tools | experimental 2025-10 | deleted 2026-03 / 2026-05 | → [[no-file-read-write-tools]] |
| personality selection | `714151eb4e` 2026-01-20 | retired `132c739171` 2026-09-12 | → [[no-user-selectable-personality]] |
| `update_plan` default | prompted heavily 2025-07-31 | opt-in `a9519cbcdd` 2026-08-31 | → [[no-todo-tool]] |
| date in context | absent | added `90cc4e79a2` 2026-02-26 | → [[no-date-in-prompt]] |
| auto-trust of undecided projects | `1e59dc5bda` 2026-08-04 | explicit prompt `17801b4206` same day | → [[project-trust-gate]] |
| CLI process hardening | `d61dea6fe6` 2025-09-25 | removed `d3ff668f68` 2026-01-08 | → [[no-cli-process-hardening]] |

## Open questions
- Why opencode deleted the batch tool and the read-before-write guard (no commit bodies).
- Why prune and LSP/formatters went default-off in April 2026 (cache stability and process cost are plausible, unverified).
- Why MCP flipped (issue #10040 not fetched; the author changed from Mario Zechner to Armin Ronacher).
- Whether durable subagents will reach stable pi (`5609b0d6c` experimental TUI on pi-durable).
- The `sandbox/` example reads `.pi/sandbox.json` without a trust check (see [[no-sandbox]]).
- Third-party `pi install` runs lifecycle scripts (`packages/coding-agent/src/core/package-manager.ts:1844-1865`, no `--ignore-scripts`), while pi's own installs forbid them → [[supply-chain-pinning]].

Related: [[replaceable-builtin-extension]] · [[extension-event-hooks]] · [[minimal-default-toolset]] · [[feature-flag-stages]] · [[extensibility-model]] · [[Constants]]

## opencode stance

**Stance at HEAD** ([[opencode]], `ecc4916b5a`)
- Rich by default, pruned by usage: ~14 built-in tools, built-in subagents, plan mode, todos, web, MCP, LSP (opt-in), snapshots/undo, approvals. Its absences are mostly **removals** (tools that were redundant, unused or harmful) and **v2 refusals to over-promise**, not manifesto lines.
- Governance: readily merged are "Bug fixes, Additional LSPs / Formatters, Improvements to LLM performance, Support for new providers…"; "However, any UI or core product feature must go through a design review with the core team before implementation." (`CONTRIBUTING.md:3-13`). Net-new functionality starts as an issue and waits for core-team approval before a PR (`CONTRIBUTING.md:251-253`); "All PRs must reference an existing issue" (`CONTRIBUTING.md:180-182`); "Long, AI-generated PR descriptions and issues are not acceptable and may be ignored" (`CONTRIBUTING.md:204-206`). Net effect: features are gated by maintainers, not by a written non-goals list — most absences below are decisions recorded in commits and specs.
- Security: "OpenCode does **not** sandbox the agent. The permission system exists as a UX feature" (`SECURITY.md:15-19`); out of scope: sandbox escapes, provider data handling, MCP server behavior, malicious config (`SECURITY.md:25-33`). "We do not accept AI generated security reports … automatic ban" (`SECURITY.md:3-7`, `e4b548fa76` 2026-02-17).

**pi's absences, checked against opencode**
| pi absence | opencode | evidence |
|---|---|---|
| [[no-sandbox]] | **also absent** (stated) | `SECURITY.md:15-19` |
| [[no-background-bash]] | **also absent**; removed again in v2 | `specs/v2/schema-changelog.md:697` |
| [[no-codebase-index]] | **also absent** | grep; `explore` subagent instead |
| [[no-prompt-injection-defense]] | **also absent** (implicit) | `SECURITY.md:31-33` |
| [[no-auto-hot-reload]] | **also absent** in legacy (unverified completeness); v2 goal | `specs/v2/instructions.md:13` |
| [[no-subagents-core]] | implements → [[task-owned-subagent]], [[agent-profiles]] | `packages/opencode/src/tool/task.ts` |
| [[no-plan-mode]] | implements → [[plan-mode]] | `packages/opencode/src/agent/agent.ts:156-180` |
| [[no-permission-prompts]] | implements → [[permission-ruleset]] | `packages/opencode/src/permission/index.ts:28-163` |
| [[no-todo-tool]] | implements → [[task-list-tool]] | `packages/opencode/src/tool/todo.ts` |
| [[no-web-tools]] | implements → [[web-tools]] | `packages/opencode/src/tool/registry.ts:58-65` |
| [[no-turn-cap]] | implements, opt-in → [[step-budget-limit]], [[repeated-tool-call-detection]] | `packages/opencode/src/session/prompt.ts:1178` |
| [[no-cwd-confinement]] | soft version → [[workspace-boundary-check]] | `packages/opencode/src/tool/external-directory.ts:13-44` |
| [[no-date-in-prompt]] | opposite (date in `<env>`; v2 appends date changes) | `packages/opencode/src/session/system.ts:83` |
| [[no-builtin-mcp-reversed]] | built-in since `37c34fd39c` 2025-06-03 → [[mcp-integration]] | `packages/opencode/src/mcp/index.ts` |
| [[no-lsp]] | implements, opt-in → [[lsp-diagnostics-feedback]] | `packages/opencode/src/lsp/lsp.ts:151` |
| [[no-checkpoints-undo]] | implements → [[workspace-snapshots]] | `packages/opencode/src/snapshot/index.ts` |
| [[no-bash-default-timeout]] | implements (2 min) → [[shell-execution]] | `packages/opencode/src/tool/shell.ts:347` |
| [[no-binary-detection-in-read]] | implements → [[file-read-tool]] | `packages/opencode/src/tool/read.ts:182-226` |

5 of 18 shared. Both harnesses agree on: no sandbox, no index, no injection defense, no model-facing background shell.

**opencode's own absences**
| Absence | Kind | Key evidence |
|---|---|---|
| [[no-project-trust-gate]] | stated (threat model) | `SECURITY.md:33`; `packages/opencode/src/tool/registry.ts:183-197` |
| [[no-claude-subscription-auth]] | removed (legal) | `1ac1a0287c` 2026-03-19; `94dd0a8dbe` |
| [[no-model-initiated-plan-entry]] | removed (failure) | `fa559b0385` 2026-02-24 |
| [[no-background-task-polling]] | removed (failure) | `dabf2dc013` 2026-05-25 |
| [[no-batch-tool]] | removed (no reason) | `463318486f` 2026-04-07 |
| [[no-read-before-write-guard]] | removed (no reason); pi never had it | `76a141090e` 2026-04-16 |
| [[removed-builtin-tools]] | index of removals (list, multiedit, todoread, lsp-*, codesearch, scout, patch) | see note |
| [[no-lsp-formatters-by-default]] | reversed default | `220e3e9a2b` 2026-04-16 |
| [[no-hardcoded-secret-refusal]] | replaced by rules | `3611260405` 2026-01-04 |
| [[no-codemode-default-limits]] | stated (design doc) | `packages/codemode/codemode.md:130-143` |
| [[v2-rejected-designs]] | stated (specs/v2) | `specs/v2/config.md:48-50`; `specs/v2/session.md:153-173` |

**Reverts as signal** (runtime, `git log --grep=^revert`)
- Reverted then re-added at HEAD: MCP OAuth redirect URI (`33290c54cd` 2026-01-16; `redirectUri` at `packages/opencode/src/mcp/oauth-provider.ts:19`), optional mDNS (`505068d5a6` 2025-12-26; `packages/opencode/src/server/mdns.ts`), global `~/.claude/skills` (`ef8388f0ee` 2025-12-29; `packages/opencode/src/skill/index.ts:186-193`), Trinity prompt (`b5a4671c64` 2026-02-03; `trinity.txt` present).
- Reverted and still absent: provider-level `store` option (`16cac69a72` 2026-01-14), signed-thinking reorder (`a763a14d44` 2026-06-02), git-backed review modes (`1b028d0632` 2026-03-26, 308 lines).
- Same-day revert of experimental code mode (`cb93114424` → `379adee35c` 2026-07-02), re-landed `2409c7a3d5` the next day.
