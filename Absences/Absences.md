---
type: group
group: absences
---
# Absences

Deliberate non-features and removed designs, with the rationale and the opt-in path. pi ([[pi]], `b30a6dd77`) is the reference harness; opencode ([[opencode]], `ecc4916b5a`) is compared in [[#opencode stance]]. Design axes where the two diverge → [[Tradeoffs]].

**Stance at HEAD**
- "Pi ships with powerful defaults but skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package that does it your way." (`README.md:19`, wording `de7e675de`, 2026-10-02).
- Governance: "**pi's core is minimal**. If your feature does not belong in the core, it should be an extension. PRs that bloat the core will likely be rejected." (`CONTRIBUTING.md:7-9`). Even hook points "should be well considered … to avoid unmaintainable bloat" (`CONTRIBUTING.md:11`).

**Manifesto lineage**
1. **Nov 2025**: separate README sections, all 2025-11-12 by Mario Zechner. Each used "pi does not and will not …": sub-agents `e9935beb5`, background bash `271810c80`, to-dos `9066f58ca`, YOLO/security `b172beb92`, MCP `60e4fcf01`.
2. **Dec 2025**: "Philosophy" section (`3424550d2`, 2025-12-17). Six one-liners, including "No permission popups. Security theater."
3. **Sep 2026**: Philosophy text at `25cc5c7bf^:packages/coding-agent/README.md:533-549`, softened ("or build your own with extensions, or install a package").
4. **2026-09-22**: the section was **deleted** in docs refresh `25cc5c7bf` (#9898, Christian Klotz), one week before MCP shipped (`8562bcf66`). Whether the deletion was editorial or a softening of stance is unverified.

## Notes
| Absence | Status at HEAD | Opt-in path |
|---|---|---|
| [[no-subagents-core]] | absent in stable; **present in experimental durable** (`2532a0bef`) | `examples/extensions/subagent/` |
| [[no-plan-mode]] | absent | `examples/extensions/plan-mode/` |
| [[no-permission-prompts]] | absent (YOLO) | `tool_call` gate + `permission-gate.ts`, `protected-paths.ts` |
| [[no-sandbox]] | absent | Docker / Docker Sandboxes / gondolin / `sandbox/` / `ssh.ts` |
| [[no-todo-tool]] | absent | `todo.ts`, TODO.md |
| [[no-background-bash]] | absent | tmux, `interactive-shell.ts` |
| [[no-web-tools]] | absent | CLI + README skills, MCP |
| [[no-turn-cap]] | absent | `finishTurn` / `turn_end` hook |
| [[no-cwd-confinement]] | absent | `protected-paths.ts`, container |
| [[no-date-in-prompt]] | absent (removed `f4e9ca746`) | `APPEND_SYSTEM.md`, bash `date` |
| [[no-builtin-mcp-reversed]] | **reversed**: replaceable built-in since `8562bcf66` | `-builtin:mcp` to opt out |
| [[no-lsp]] | absent (omission) | none shipped |
| [[no-codebase-index]] | absent (omission) | context files, `claude-rules.ts`, MCP |
| [[no-checkpoints-undo]] | absent | `git-checkpoint.ts`, `auto-commit-on-exit.ts` |
| [[no-prompt-injection-defense]] | out of scope (`SECURITY.md:19-22`) | containment, gates |
| [[no-bash-default-timeout]] | absent (removed `29900ce64`) | model-passed `timeout`, spawn hook |
| [[no-auto-hot-reload]] | absent (manual `/reload`) | `reload-runtime.ts` |
| [[no-binary-detection-in-read]] | observed gap | `tool-override.ts` |

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

## Open questions
- Why opencode deleted the batch tool and the read-before-write guard (no commit bodies).
- Why prune and LSP/formatters went default-off in April 2026 (cache stability and process cost are plausible, unverified).
- Why MCP flipped (issue #10040 not fetched; the author changed from Mario Zechner to Armin Ronacher).
- Whether durable subagents will reach stable pi (`5609b0d6c` experimental TUI on pi-durable).
- The `sandbox/` example reads `.pi/sandbox.json` without a trust check (see [[no-sandbox]]).
- Third-party `pi install` runs lifecycle scripts (`packages/coding-agent/src/core/package-manager.ts:1844-1865`, no `--ignore-scripts`), while pi's own installs forbid them → [[supply-chain-pinning]].

Related: [[replaceable-builtin-extension]] · [[extension-event-hooks]] · [[minimal-default-toolset]] · [[Constants]]
