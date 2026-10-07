---
type: harness
repo: https://github.com/earendil-works/pi
commit: b30a6dd77
language: TypeScript
studied: 2026-10-07
aliases: [pi-coding-agent, "@earendil-works/pi-coding-agent", pi-mono]
---

Monorepo, 14 packages, 6804 commits (2025-08-09 → 2026-10-07), HEAD `b30a6dd77`. Node ≥ 22.19 (`README.md`). Digest: [[2026-10-07-pi]].

## Ideology
- **Minimal core, extend everything.** "Pi is a minimal, extensible agent harness that you can make your own… skips features like sub-agents and plan mode" (`README.md:15-19`). Missing features ship as example extensions / packages, not core → [[replaceable-builtin-extension]], [[extension-event-hooks]].
- **Tiny prompt, tiny toolset, full trust.** 4 default tools (read/bash/edit/write) → [[minimal-default-toolset]]; system prompt ~175 tok at v0 (`ffc9be886`) → ~680 tok at HEAD → [[minimal-system-prompt]]; no permission prompts, no sandbox ("Security theater", `3424550d2`) → [[no-permission-prompts]], [[no-sandbox]].
- **The log is the truth.** Session = append-only JSONL tree; model context re-projected from the log before every request (`agent-session.ts:793-799`, `466db0fec`); system prompt + tool set stored *in the transcript* as deltas (`9e05370b2`) → [[session-tree]], [[context-projection]], [[transcript-carried-system-prompt]]. Taken to its end in `packages/durable` (crash-resumable tasks) → [[durable-execution]].

## Organ map
| Organ | Files | Notes |
|---|---|---|
| Loop | `packages/agent/src/agent-loop.ts:163-321` (`runLoop`), `packages/agent/src/agent.ts`; post-run driver `packages/coding-agent/src/core/agent-session.ts:1821-1914` | 3 nested loops: inner turn loop, outer follow-up loop, session driver (retry/compaction/settle). No turn cap → [[turn-loop]], [[run-settlement]], [[no-turn-cap]] |
| Message builder | `packages/agent/src/types.ts` (`AgentMessage`, `convertToLlm`), `packages/coding-agent/src/core/messages.ts`, `packages/ai/src/api/transform-messages.ts:64-235`, `session-manager.ts:390-583` | App messages → LLM messages at request boundary; replay repair + cross-provider handoff → [[message-conversion-layer]], [[transcript-replay-repair]], [[cross-provider-handoff]] |
| System prompt | `packages/coding-agent/src/core/system-prompt.ts`, `resource-loader.ts`, `skills.ts` | Sections, per-tool snippets/guidelines, AGENTS.md hierarchy, skills list, docs pointer; no date → [[dynamic-tool-guidelines]], [[context-file-hierarchy]], [[skill-progressive-disclosure]] |
| Tools | `packages/coding-agent/src/core/tools/*` (read, bash, powershell, edit, write, grep, find, ls), `src/extensions/{tool-search,mcp,codemode}` | Parallel by default + per-file mutation queue; truncation 2000 lines / 50KB with spill file → [[parallel-tool-execution]], [[search-replace-edit]], [[tool-output-truncation]] |
| Provider | `packages/ai/**` (pi-ai: ~20 API adapters, OAuth, generated catalog), `coding-agent/src/core/model-*.ts` | Unified event stream; errors as stream events; thinking-level abstraction; overflow regex zoo → [[unified-provider-api]], [[thinking-level-abstraction]], [[context-overflow-detection]] |
| Session store | `packages/coding-agent/src/core/session-manager.ts` (JSONL v3), `~/.pi/agent/sessions/` | id/parentId tree, `/tree` `/fork` `/clone`, compaction + context_edit entries → [[session-tree]], [[session-fork]], [[context-edit-overlay]] |
| Context mgmt | `packages/coding-agent/src/core/compaction/*` | reserve 16384 / keep 20000 tokens; structured iterative summary; file-op tracking; overflow → compact+retry once → [[auto-compaction]], [[overflow-recovery]] |
| Extensibility | `packages/coding-agent/src/core/extensions/*`, `docs/extensions.md` | 41 events, jiti TS loading, packages via npm/git, RPC/JSON/SDK modes → [[extension-event-hooks]], [[headless-rpc-mode]], [[sdk-embedding]] |
| Cancellation | Esc → `AgentSession.abort()` (`agent-session.ts:2433-2443`) → one `AbortController`/run → provider stream + tools (bash kills process tree) → aborted partial persisted, skipped on replay | [[abort-propagation]], [[partial-message-persistence]] |

## Packages
| Package | Role |
|---|---|
| `agent` (pi-agent-core) | Loop, Agent state, steering/follow-up queues, tool execution |
| `ai` (pi-ai) | Unified LLM API, adapters, OAuth, model catalog, overflow/retry classifiers |
| `coding-agent` | CLI: session, tools, compaction, prompts, extensions, TUI/print/json/rpc modes, SDK |
| `tui` (pi-tui) | Differential-rendering terminal UI → [[differential-tui-rendering]] |
| `durable` (Pico5) | Second, crash-resumable harness: tasks, SQLite/JSONL storage, subagents → [[durable-execution]], [[task-owned-subagent]] |
| `env` | SSH-deployed Rust daemon for remote tool execution → [[remote-execution-env]] |
| `chord`, `protocol`, `client`, `server` | Replicated state, CBOR framing, experimental client/server split → [[replicated-state]], [[client-server-session-split]] |
| `mcp`, `codemode` | MCP client; QuickJS-WASM sandbox where only capability is calling tools → [[mcp-integration]], [[code-mode]] |
| `telemetry`, `evals` | Vendor-neutral telemetry contracts; paired A/B eval harness → [[install-telemetry]], [[harness-evals]] |

## Distinctive choices
- Steering waits for whole tool batch; "skip remaining tools" tried and reverted (`117af076c` → `208a2cc12`) → [[steering-queue]].
- Tool calls from length-truncated responses never execute (`351efc828`) → [[truncated-tool-call-guard]].
- Failed attempts hidden via append-only `context_edit`, not deleted → [[context-edit-overlay]].
- Cache warmer with expected-value gate (≥ $0.05 saving, 0.15 idle-continuation prior) → [[cache-warming]].
- MCP tools hidden by default behind codemode / BM25 tool search, descriptions kept static for cache → [[deferred-tool-loading]], [[code-mode]].
- Subscription OAuth impersonates vendor CLIs ("stealth mode") → [[provider-identity-shim]].
- AGENTS.md loads even when project untrusted; prompt injection declared out of scope (`5cb4f597f`, `SECURITY.md`) → [[project-trust-gate]], [[no-prompt-injection-defense]].
- Second harness (`packages/durable`) built spec-first by agents in ~6 weeks, replaced earlier in-agent harness (`7fd478a2e`) → [[spec-driven-agentic-development]].

## Absences
[[no-subagents-core]] · [[no-plan-mode]] · [[no-permission-prompts]] · [[no-sandbox]] · [[no-todo-tool]] · [[no-background-bash]] · [[no-web-tools]] · [[no-turn-cap]] · [[no-cwd-confinement]] · [[no-date-in-prompt]] · [[no-builtin-mcp-reversed]] · [[no-lsp]] · [[no-codebase-index]] · [[no-checkpoints-undo]] · [[no-prompt-injection-defense]] · [[no-bash-default-timeout]] · [[no-auto-hot-reload]] · [[no-binary-detection-in-read]] — overview: [[Absences]].

## Fix hot-spots (from `06-fixes` mining)
| Theme | fix commits | changelog Fixed |
|---|---|---|
| reasoning / thinking / signatures | 151 | 267 |
| terminal / TUI | 86 (+244 `fix(tui)`) | 203 |
| tool-call/result assembly & replay | 62 | 129 |
| overflow / context window | 74 | 127 |
| auth / OAuth | 98 | 116 |
| compaction / summaries | 87 | 107 |
| prompt cache / affinity | 41 | 88 |
| abort / cancel / races | 63 | 81 |
Fix-commit split: coding-agent 884, ai 518, tui 244, agent 108 (`git log --grep='^fix'`).

## Repo & development practices
How the pi team uses agents (mostly pi itself) to build pi. Root `AGENTS.md` is auto-loaded by any agent run from the repo root; `CONTRIBUTING.md:19` requires contributors' agents to run there and follow it.

**Agent rules (`AGENTS.md`)**
| Rule | Evidence | Why |
|---|---|---|
| Several pi sessions may edit the same cwd at once → commit only your own files, stage explicit paths, `git status` before commit | `AGENTS.md:54-62`; added `5c59caee4` 2026-01-16 "critical git rules for parallel agent work" | Shared worktree, no isolation → [[shared-worktree-agents-clobber-each-other]] |
| Banned: `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git add -A/.`, `commit --no-verify`; never force-push; on rebase conflict in a file you didn't touch → abort and ask | `AGENTS.md:64-72` | Each destroys another session's unstaged work or bypasses hooks |
| PR review without moving the worktree (`gh pr diff`, `git show <ref>:<path>`, no `gh pr checkout`) | `AGENTS.md:78-82`; same rule in `.pi/prompts/pr.md:4` | Switching branches would stomp parallel sessions |
| `npm run check` after code changes; never `npm run build` / `npm test` unasked; never full vitest (e2e tests auto-activate when API-key env vars exist) → `./test.sh` | `AGENTS.md:32-36` | Cost + accidental paid calls |
| Suite tests use `test/suite/harness.ts` + faux provider, no real keys/tokens; regression tests carry the GitHub issue number | `AGENTS.md:38-39`; `packages/coding-agent/test/suite/README.md`; 81 files in `test/suite/regressions/`; faux added `ef6af5ebb` | Deterministic CI-safe agent tests → [[pi--unified-provider-api\|faux provider]] |
| Ad-hoc scripts: `write` to temp file, run, delete; never multi-line scripts inside `bash` | `AGENTS.md:40` | Quoting breakage, reviewability |
| Erasable TypeScript only (no `enum`, `namespace`, parameter properties…) so Node strip-only mode runs sources directly | `AGENTS.md:24`; `tsconfig.base.json:7` `erasableSyntaxOnly`; `06c6c324d` 2026-05-19 | `pi-test.sh` / `auto-pi` run `.ts` without a build |
| No inline/dynamic imports; no `any`; inline single-call helpers; configurable keys only via `DEFAULT_*_KEYBINDINGS`; never hand-edit `models.generated.ts` (edit `generate-models.ts`) | `AGENTS.md:18-28` | |
| Dependency hygiene: exact pins, `--ignore-scripts`, read `undici` changelog before bump, lockfile commits blocked unless `PI_ALLOW_LOCKFILE_CHANGE=1` | `AGENTS.md:43-50` | → [[pi--supply-chain-pinning\|supply-chain-pinning]] |
| Style: no emojis, no filler, "problem → concrete trace → solution"; answer the question before editing; say agree/disagree before acting on feedback | `AGENTS.md:3-13` | |
| AI-posted issue/PR comments go via `--body-file` and end with a fixed disclaimer (`This comment is AI-generated by \`/wr\``) | `AGENTS.md:88-92`; `.pi/prompts/wr.md:19-23` | Provenance of agent output |
| User-override clause: conflicting user instruction → ask for explicit confirmation first | `AGENTS.md:123-125` | |
| Provider checklist + interactive-testing + release moved out of AGENTS.md into on-demand skills | `7426ce977` 2026-05-22, `be26e3270` 2026-09-09; `AGENTS.md:100,121` | Keep always-loaded context small → [[skill-progressive-disclosure]] |

**Repo-local pi setup (`.pi/`, dogfooding)**
- Prompts (`/cl` changelog audit, `/is` issue analysis — "Do not trust analysis written in the issue… Ignore any root cause analysis", `/pr` Good/Bad/Ugly review, `/wr` wrap: changelog → comment → commit own files → push main → close issue, `/sa` security advisory, `/deslop` simplify) (`.pi/prompts/is.md:15-19`, `wr.md:17-30`) → [[pi--prompt-template-expansion|prompt-template-expansion]].
- Skills: `add-llm-provider.md` (types → provider → lazy registration in `register-builtins.ts` → `generate-models.ts` → mandatory test matrix: `stream`, `tokens`, `abort`, `empty`, `context-overflow`, `unicode-surrogate`, `tool-call-without-result`, `image-tool-result`, `total-tokens`, `cross-provider-handoff` with ≥1 pair per model family) (`.pi/skills/add-llm-provider.md:30-43`); `interactive-testing.md` (drive TUI in `tmux new-session -x 80 -y 24` + `capture-pane`, `.pi/skills/interactive-testing.md:10-17`); `release.md`.
- Extensions: `import-repro.ts` (`/ir <gist>` imports a CI-analysis session, rewrites recorded high-entropy cwd to local checkout, `:5-15,197`); `prompt-url-widget.ts` (widget linking the PR/issue/advisory when `/pr`/`/is`/`/sa` prompt detected, `:7-9,173`); `tps.ts` (tok/s + cache r/w notify on `agent_end`); `redraws.ts` (`/tui` shows `tui.fullRedraws`) → [[pi--differential-tui-rendering|full-redraw counter]].
- Dev runners: `pi-test.sh` runs `src/experimental/cli.ts` from source via `--import source-resolver.ts`, `--no-env` unsets ~35 provider credential vars (`pi-test.sh:17-59`, `d2f3b42de`); `scripts/auto-pi.sh` symlinked as `pi` → latest local build with `PI_EXPERIMENTAL=1`, `--stable`/`pi update` fall through to the installed pi (`:4-13,56-58`; `5cd93f688` 2026-08-20).

**Testing & checks**
- `test.sh`: `env -i` empty environment, temp `HOME`/`TMPDIR`/XDG/npm config, `LANG=C TZ=UTC`, git prompts/askpass disabled, `PI_NO_LOCAL_LLM=1`, `AWS_EC2_METADATA_DISABLED=true`; cleanup deletes only a dir carrying `.pi-test-owned` marker (`test.sh:4-37,39-79`) — tests can't see real keys or `~/.pi`.
- `npm run check` = biome (`--error-on-warnings`) + pinned-deps + runtime-deps + ts-relative-imports + entry-graphs + install-lock `--check` + `tsc --noEmit` + browser-smoke (`package.json:20`). Pre-commit: lockfile guard → check → browser smoke if `packages/ai/*` or lockfiles staged → re-stage formatted files (`.husky/pre-commit:6-43`).
- Entry-graph budgets: "Entry points are cost contracts" — walks value-import graph per `exports` entry, per-entry `maxFiles`/`forbid` (e.g. `pi-ai/models` ≤15 files, no `providers/`) (`scripts/check-entry-graphs.mjs:2-40`); added `5507d76ee` after `import { type X }` (not type-only) pulled whole barrels: server 112 files/98 MB → 28/69 MB.
- Custom GritQL lint: forbids `model.type === "chat"` (chat models may omit `type`) (`scripts/biome/model-type-comparison.grit`).
- CI: build → check → build `pi-env` daemon → test; separate job runs pi's MCP client against pinned official conformance suite vs `test/mcp-conformance/baseline.json` (`.github/workflows/ci.yml:32-65`) → [[pi--mcp-integration|mcp-integration]]. Daily `npm audit` + `audit signatures` (`npm-audit.yml`). All actions pinned by SHA.
- Session-mining scripts — tool design driven by real transcripts: `edit-tool-stats.mjs` classifies edit failures per model/extension from `~/.pi/agent/sessions` (`classifyErrorKind`, `:243`; `235b247f1`), `read-tool-stats.mjs` (`classifyRead`, `:182`; `392455299`), `tool-stats.ts` (`6d4d2e928`), `session-context-stats.mjs`, `cost.ts`; `session-transcripts.ts --analyze` splits transcripts into 100k-char files and spawns pi subagents to find patterns (`scripts/session-transcripts.ts:2-20`; `92486e026` 2026-01-11).

**Contribution gate (anti-AI-slop)**
- New contributors' issues *and* PRs auto-closed (`issue-gate.yml`, `pr-gate.yml` on `pull_request_target`); bypass via `.github/APPROVED_CONTRIBUTORS` (`<user> issue|pr`, ~400 lines) or write access; trusted bots skip (`issue-gate.yml:18-25,83-96`). Closed as `not_planned` + `untriaged` label (`:109-128`).
- Maintainer reply `lgtmi` (issues) / `lgtm` (issues+PRs) at start or end of a comment → `approve-contributor.yml` regex-matches (`:33-43`), commits the allowlist update and pushes (`:176-183`).
- Timeline: PR gate `3eded2c14` 2026-01-18 → "OSS weekend" issue gating `572876be1` 2026-03-14 → permanent gate `d62d22173` 2026-04-14. Rationale: agent-generated tracker spam; "AI can help… It is not trusted to make final maintainer decisions" (`CONTRIBUTING.md:13-17,79-93`); automation spam → permanent block (`:52-54`).
- Security model: user account = trust boundary; out of scope: sandboxing, prompt injection, untrusted repos/extensions, anything needing local write access to `~/.pi`, `AGENTS.md`, `models.json` (`SECURITY.md:6-22,48-68`; `e30b1b18d` 2026-06-02) → [[no-sandbox]], [[no-prompt-injection-defense]].

**Agent-in-CI issue triage (`issue-analysis.yml`, `abe9c9d9f` 2026-07-06)**
- Trigger: `pi-analyze` label or `@issuron analyze [#run-on-linux|windows|mac]` from an `earendil-works/staff` member (`:1-19`). Runs `node packages/coding-agent/src/cli.ts -p --approve --session-dir … --model openai-codex/gpt-5.5 --thinking high "/is <issue-url>"` (`:264-265,382-393`), 45-min timeout (`:259`).
- Checkout dir `pi-ci-<32 hex>` so the recorded cwd is unique and rewritable (`:267-273`); session exported to HTML+JSONL gist, linked as `pi.dev/session/#<id>` and `pi "/ir <id>"` repro command (`:488-569,632`) → [[pi--session-export-share|session-export-share]].
- Subscription OAuth in CI: `PI_AUTH_JSON` secret; refreshed `auth.json` written back to the environment secret, refusing malformed/no-refresh-token files (`:420-452`). Must be a dedicated login: Codex rotates refresh tokens, shared auth.json with a dev machine invalidates the second refresher (`:34-37`) → [[oauth-refresh-token-rotation-lost]].

**Release & distribution**
- Lockstep versioning: all packages one version; `patch` = fixes+additions, `minor` = breaking, no majors (`.pi/skills/release.md:10`). `/cl` audit first; `release:local` smoke from `/tmp` for both Node and Bun binaries incl. a real prompt in tmux — "startup alone is not a passing smoke test" (`release.md:12-35`; `interactive-testing.md:19`).
- `release.mjs` bumps, refreshes Nix model-catalog pin, commits/tags, pushes; tag → `build-binaries.yml` (Bun 1.3.14 standalone binaries, smoke matrix, npm trusted publishing via OIDC + provenance, then `announce-pi-dev-release` verifies every package resolves before writing the R2 marker read by `pi.dev/api/latest-version`) (`release.md:37-47`; `build-binaries.yml:27-45,312-389`; binaries since `c4a65ad8b` 2025-12-02).
- Nix flake (`c71a496ff` 2026-10-02): tag builds Linux+macOS, then fast-forwards `stable` branch (`nix run github:earendil-works/pi/stable`); nightly cron catches packaging drift (`nix.yml:11-38,135-142`).
- Changelog discipline: per-package `## [Unreleased]`, released sections immutable, contributors never edit CHANGELOG (`AGENTS.md:102-117`; `CONTRIBUTING.md:69`). No `install.sh`/`install.ps1` in repo (git ls-files).

## Coverage index
Checklist audit of `packages/coding-agent` at `b30a6dd77`: every user-facing knob with default and evidence. `SM` = `packages/coding-agent/src/core/settings-manager.ts`, `A` = `src/cli/args.ts`, `IM` = `src/modes/interactive/interactive-mode.ts`, `ST` = `docs/settings.md`.

**Settings keys** (global `~/.pi/agent/settings.json` + project `.pi/settings.json`; merge rules → [[pi--layered-settings|layered-settings]])
| key | default | evidence | note |
|---|---|---|---|
| `defaultProvider` / `defaultModel` | auto | `SM:825-831` | [[pi--model-resolution\|model-resolution]] |
| `defaultThinkingLevel` | `"medium"` | `SM:890`; `core/defaults.ts:3` | [[pi--thinking-level-abstraction\|thinking-level]] |
| `modelThinkingLevels` | none (`"provider/id"` → level) | `SM:900-916` | [[pi--thinking-level-abstraction\|thinking-level]] |
| `thinkingBudgets` | built-in 1024/2048/8192/16384 | `SM:1297` | [[pi--thinking-level-abstraction\|thinking-level]] |
| `hideThinkingBlock` | false (TUI only) | `SM:1069-1071` | [[pi--thinking-level-abstraction\|thinking-level]] |
| `enabledModels` | all (Ctrl+P cycle, `--models` format) | `SM:1452-1454` | [[pi--model-resolution\|model-resolution]] |
| `transport` | `"auto"` (`sse`/`websocket`/`websocket-cached`) | `SM:927-929`; `ST:121` | [[pi--unified-provider-api\|unified-provider-api]] |
| `cacheWarming` | `"streaming"` (global-only) | `SM:1046-1049` | [[pi--cache-warming\|cache-warming]] |
| `showCacheMissNotices` | false | `SM:1073-1075` | [[pi--cache-miss-accounting\|cache-miss-accounting]] |
| `steeringMode` / `followUpMode` | `"one-at-a-time"` | `SM:853-865` | [[pi--steering-queue\|steering]] / [[pi--follow-up-queue\|follow-up]] |
| `externalEditor` | `$VISUAL` → `$EDITOR` → nano/notepad | `SM:1077-1087` | [[pi--layered-settings\|layered-settings]] |
| `doubleEscapeAction` | `"tree"` (`fork`/`none`) | `SM:1469-1471`; `IM:3059-3074` | [[pi--session-tree\|session-tree]] |
| `treeFilterMode` | `"default"` | `SM:1479-1483` | [[pi--session-tree\|session-tree]] |
| `defaultProjectTrust` | `"ask"` (global-only) | `SM:1123-1126` | [[pi--project-trust-gate\|project-trust-gate]] |
| `defaultTools` | read, bash, edit, write; `+x`/`-x` modifiers | `SM:215,1457-1461` | [[pi--minimal-default-toolset\|minimal-default-toolset]] |
| `codemode.mode` / `codemode.inlineBudget` | `"on"` / 3000 est. tokens | `SM:101-108`; `extensions/codemode/tool.ts:156` | [[pi--code-mode\|code-mode]] |
| `sessionDir` | agent session dir (`PI_CODING_AGENT_SESSION_DIR`, `--session-dir` override) | `SM:820-823`; `ST:64` | [[pi--session-tree\|session-tree]] |
| `compaction.enabled` / `.reserveTokens` / `.keepRecentTokens` / `.modelOverrides` | true / 16384 / 20000 / none | `SM:22-25,937-998` | [[pi--auto-compaction\|auto-compaction]] |
| `branchSummary.reserveTokens` / `.skipPrompt` | 16384 / false | `SM:999-1008` | [[pi--branch-summary\|branch-summary]] |
| `retry.enabled` / `.maxRetries` / `.baseDelayMs` / `.maxAgentDelayMs` | true / 3 / 2000 / 60000 | `SM:1010-1030`; `packages/ai/src/utils/retry.ts:126` | [[pi--auto-retry-backoff\|auto-retry-backoff]] |
| `retry.provider.timeoutMs` / `.maxRetries` / `.maxRetryDelayMs` | `httpIdleTimeoutMs` / 0 / 60000 | `SM:1057-1063`; `ST:129-133` | [[pi--auto-retry-backoff\|auto-retry-backoff]] |
| `httpProxy` | none (global-only; sets `HTTP_PROXY`/`HTTPS_PROXY` if unset) | `core/http-dispatcher.ts:45-50`; `main.ts:594,872` | [[pi--http-transport-hardening\|http-transport-hardening]] |
| `httpIdleTimeoutMs` | 300000 (0 disables) | `SM:1032-1034`; `core/http-dispatcher.ts:4` | [[pi--http-transport-hardening\|http-transport-hardening]] |
| `websocketConnectTimeoutMs` | 15000 | `SM:1065-1067`; `packages/ai/src/api/openai-codex-responses.ts:57` | [[pi--http-transport-hardening\|http-transport-hardening]] |
| `shellPath` / `shellCommandPrefix` | platform default / none | `SM:1101-1104,1134-1136` | [[pi--shell-execution\|shell-execution]] |
| `npmCommand` | `npm` | `SM:1144-1146` | [[pi--harness-package-distribution\|package-distribution]] |
| `packages` / `extensions` / `skills` / `prompts` / `themes` | `[]` (combined across layers; `!glob`, `+path`, `-path`, `-builtin:x`) | `SM:1207-1285`; `ST:151-160` | [[pi--harness-package-distribution\|package-distribution]], [[pi--replaceable-builtin-extension\|builtin-extension]] |
| `enableSkillCommands` | true (`/skill:name`) | `SM:1287-1289` | [[pi--skill-progressive-disclosure\|skills]] |
| `images.autoResize` / `images.blockImages` | true / false | `SM:1426-1441` | [[pi--image-normalization\|image-normalization]] |
| `theme` | `"system"` (name with `/` ignored) | `SM:873-882`; `ST:92` | — (cosmetic) |
| `quietStartup` | false (`true` / `"header"`); `--verbose` overrides | `SM:1112-1115` | — |
| `tuiMode` | `"fullscreen"` (`regular`) | `SM:1371-1373` | [[pi--differential-tui-rendering\|tui]] |
| `fullscreenExitOutput` / `fullscreenScrollbar` / `fullscreenCopyOnSelect` / `fullscreenWheelScrollLines` | `transcript` / `auto` / true / `auto` (1-100) | `SM:1381-1420` | [[pi--differential-tui-rendering\|tui]] |
| `editorPaddingX` / `outputPad` / `autocompleteMaxVisible` / `showHardwareCursor` | 0 / 1 / 5 / false (`PI_HARDWARE_CURSOR=1`) | `SM:1491-1523` | [[pi--differential-tui-rendering\|tui]] |
| `terminal.showImages` / `.imageWidthCells` / `.clearOnShrink` / `.showTerminalProgress` | true / 60 / false (`PI_CLEAR_ON_SHRINK=1`) / false (OSC 9;4) | `SM:1311-1360` | [[pi--differential-tui-rendering\|tui]] |
| `terminal.hyperlinks` / `.images` / `.trueColor` | `"auto"` | `SM:1301-1309`; `packages/tui/src/terminal-image.ts:146-165` | [[pi--differential-tui-rendering\|tui]] |
| `markdown.codeBlockIndent` / `markdown.mermaid` | `"  "` / `"streaming"` (`off`/`final`) | `SM:1531-1538` | — (cosmetic) |
| `collapseChangelog` | false | `SM:1154-1156` | — |
| `enableInstallTelemetry` / `enableAnalytics` | true / false | `SM:1164-1176` | [[pi--install-telemetry\|install-telemetry]] |
| `warnings.anthropicExtraUsage` | true | `SM:1547-1549`; `IM:5243` | [[pi--subscription-oauth-auth\|subscription-oauth]] |
| internal: `lastChangelogVersion`, `trackingId`, `deviceId` (global-only) | written by pi | `SM:810,1178,1198-1205` | [[pi--install-telemetry\|install-telemetry]] |

**Environment variables** (`docs/environment-variables.md`; undocumented ones marked †)
| var | effect | evidence | note |
|---|---|---|---|
| `PI_CODING_AGENT=true`, `AI_AGENT=pi` (set by pi) | child-process markers | `cli/setup.ts:6-7`; `rpc-entry.ts:7-8` | [[pi--env-vars-as-context\|env-vars-as-context]] |
| `PI_SESSION_ID`, `PI_SESSION_FILE`, `PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL` (set by pi) | per-command session facts in bash/powershell tools | `core/tools/bash.ts:186-212` | [[pi--env-vars-as-context\|env-vars-as-context]] |
| `PI_CODING_AGENT_DIR` | config dir (name derives from `APP_NAME`) | `config.ts:584-606` | [[pi--harness-identity\|harness-identity]] |
| `PI_CODING_AGENT_SESSION_DIR` | session dir; `--session-dir` wins | `config.ts:586` | [[pi--session-tree\|session-tree]] |
| `PI_PACKAGE_DIR` | package dir override (Nix/Guix) | `config.ts:393-397` | [[pi--harness-package-distribution\|package-distribution]] |
| `PI_OFFLINE` | no startup network, no catalog refresh; implies `PI_SKIP_VERSION_CHECK` | `main.ts:576-580` | [[pi--model-catalog\|model-catalog]] |
| `PI_SKIP_VERSION_CHECK` | skip pi.dev latest-version request | `utils/version-check.ts:98` | [[pi--install-telemetry\|install-telemetry]] |
| `PI_TELEMETRY` | override install telemetry + attribution headers | `core/telemetry.ts:10` | [[pi--install-telemetry\|install-telemetry]] |
| `PI_CACHE_RETENTION=long` | extended prompt caching | `packages/ai/src/api/anthropic-messages.ts:67-73` | [[pi--cache-retention-control\|cache-retention]] |
| `PI_SHARE_VIEWER_URL` | `/share` viewer base | `config.ts:596` | [[pi--session-export-share\|export-share]] |
| `PI_RADIUS_GATEWAY` | Radius origin for `/bug` + relay | `core/radius.ts:4-8` | [[pi--install-telemetry\|install-telemetry]] |
| `PI_HARDWARE_CURSOR`, `PI_HYPERLINKS`, `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR`, `PI_CLEAR_ON_SHRINK`† | terminal capability overrides | `SM:1346,1492`; `packages/tui/src/terminal-image.ts:146-158` | [[pi--differential-tui-rendering\|tui]] |
| `PI_TUI_ESC_TIMEOUT` | lone-ESC wait (10 ms; 100 ms over SSH) | `packages/tui/src/terminal.ts:115-131` | [[pi--differential-tui-rendering\|tui]] |
| `PI_TIMING`†, `PI_STARTUP_BENCHMARK`† | startup profiling | `core/timings.ts:6`; `main.ts:932-975` | [[pi--differential-tui-rendering\|tui]] |
| `PI_MANAGED_INSTALL_ROOT`†, `PI_INSTALLER_API_BASE`† | managed self-update root / release API | `package-manager-cli.ts:50-79,209` | [[pi--supply-chain-pinning\|supply-chain-pinning]] |
| `PI_EXPERIMENTAL=1`† | enables experimental client/server CLI | `core/experimental.ts:2` | [[pi--client-server-session-split\|client-server]] |
| `PI_SERVER_DIR`, `PI_SERVER_ID`, `PI_SESSION_WORKER_{CONTROL_ADDRESS,CONTROL_TOKEN,SESSION_KEY_BASE64,PEER_ID}`† | experimental server profile dir (`~/.pi/server`) / session-worker handshake | `experimental/server.ts:51-55`; `experimental/session-worker.ts:64-67` | [[pi--client-server-session-split\|client-server]] |
| `LLAMA_BASE_URL`, `LLAMA_API_KEY`, `HF_TOKEN`(+paths) | llama.cpp provider / HF search | `extensions/llama/provider.ts:25-35`; `huggingface.ts:46-61` | [[pi--custom-provider-registration\|custom-provider]] |
| `VISUAL`, `EDITOR`, `HTTP_PROXY`, `HTTPS_PROXY` | editor fallback / proxy | `SM:1083`; `core/http-dispatcher.ts:48` | [[pi--layered-settings\|layered-settings]] |

**CLI flags** (`A:92-266`; `--help` text `A:299-345`)
| flag | effect | evidence | note |
|---|---|---|---|
| `--mode text\|json\|rpc`, `-p/--print` | output mode; non-TTY stdin/stdout or piped stdin ⇒ print | `A:96,177-183`; `main.ts:112-123,891-898` | [[pi--headless-rpc-mode\|headless-rpc]], [[pi--agent-event-stream\|event-stream]] |
| `-c/--continue`, `-r/--resume`, `--session <path\|id>`, `--session-id <id>` (exact, create if missing), `--fork <path\|id>`, `--no-session`, `--session-dir`, `-n/--name` | session selection | `A:111-141` | [[pi--session-tree\|session-tree]], [[pi--session-fork\|session-fork]] |
| `--provider`, `--model <pat[:thinking]>`, `--models <pats>`, `--thinking`, `--api-key`, `--list-models [search]` | model/thinking/credential | `A:115-119,142-146,167-176,218-224` | [[pi--model-resolution\|model-resolution]] |
| `--system-prompt`, `--append-system-prompt` (repeatable, text or file) | prompt override | `A:121-125` | [[pi--system-prompt-override\|system-prompt-override]] |
| `-t/--tools` (allowlist or `+x/-x`), `-xt/--exclude-tools`, `-nt/--no-tools`, `-nbt/--no-builtin-tools` | tool selection; `--tools` keeps MCP unless an entry starts `mcp__` | `A:147-166`; help `A:318-324` | [[pi--minimal-default-toolset\|minimal-default-toolset]] |
| `-e/--extension` (incl. `builtin:<name>`), `-ne/--no-extensions`, `--no-mcp` | extensions | `A:186-192` | [[pi--runtime-plugin-loading\|plugin-loading]], [[pi--mcp-integration\|mcp]] |
| `--skill`, `-ns/--no-skills`, `--prompt-template`, `-np/--no-prompt-templates`, `--theme`, `--use-theme`, `--no-themes`, `-nc/--no-context-files` | resource loading | `A:193-217` | [[pi--skill-progressive-disclosure\|skills]], [[pi--context-file-hierarchy\|context-files]] |
| `-a/--approve`, `-na/--no-approve` | trust project files for this run / ignore them | `A:241-244` | [[pi--project-trust-gate\|project-trust-gate]] |
| `--offline` | = `PI_OFFLINE=1` | `A:245-246`; `main.ts:576-580` | [[pi--model-catalog\|model-catalog]] |
| `--export <in.jsonl> [out.html]` | export and exit | `A:184` | [[pi--session-export-share\|export-share]] |
| `--tui-mode fullscreen\|regular`, `--verbose` | TUI mode / force verbose startup | `A:225-240` | [[pi--differential-tui-rendering\|tui]] |
| `@file`, `--`, unknown `--x` | file attachments / end of options / extension flags | `A:247-264` | [[pi--xml-prompt-boundaries\|xml-boundaries]], [[pi--headless-rpc-mode\|headless-rpc]] |
| subcommands `install/remove/uninstall/update [self]/list/config/auth/mcp` | package mgmt, credential export, MCP OAuth | help `A:289-298`; `cli/auth-command.ts:5-45` | [[pi--harness-package-distribution\|package-distribution]], [[pi--credential-resolution\|credential-resolution]] |

**Slash commands** (registry `src/core/slash-commands.ts:20-43`; dispatch `IM:3185-3324`)
| command | does | note |
|---|---|---|
| `/settings`, `/scoped-models` (Ctrl+P cycle set), `/model [p/m]`, `/thinking [lvl]`, `/login [p]`, `/logout` | config | [[pi--model-resolution\|model-resolution]], [[pi--subscription-oauth-auth\|oauth]] |
| `/new`, `/resume`, `/name`, `/session`, `/tree`, `/fork`, `/clone`, `/import <jsonl>` (confirm, then replace session) | sessions | [[pi--session-tree\|session-tree]], [[pi--session-fork\|session-fork]] |
| `/compact [instructions]` | manual compaction | [[pi--auto-compaction\|auto-compaction]] |
| `/copy` (last assistant text → clipboard, `IM:6610-6650`), `/export [path]`, `/share`, `/bug [desc]` | export | [[pi--session-export-share\|export-share]], [[pi--install-telemetry\|install-telemetry]] |
| `/trust`, `/reload`, `/hotkeys`, `/changelog`, `/quit` | runtime | [[pi--project-trust-gate\|trust]], [[no-auto-hot-reload]] |
| `/llama` (built-in extension, `extensions/llama/index.ts:183`) | llama.cpp router models | [[pi--custom-provider-registration\|custom-provider]] |
| hidden: `/debug` (render dump, `IM:3300,6914`), `/arminsayshi`, `/dementedelves` (easter eggs, `IM:3305-3314`) | not in autocomplete | [[pi--differential-tui-rendering\|tui]] |
| resource commands: extension `registerCommand`, prompt templates by name, `/skill:name` | `docs/slash-commands.md:54-58` | [[pi--prompt-template-expansion\|templates]] |

**Examples not otherwise cited** (`packages/coding-agent/examples/`; all others are cited in [[pi--extension-event-hooks|extension-event-hooks]] and topic notes)
| example | demonstrates |
|---|---|
| `extensions/custom-provider-gitlab-duo/` | provider delegating to pi-ai's Anthropic/OpenAI stream impls via GitLab AI Gateway, `/login gitlab-duo` or `GITLAB_TOKEN` (`index.ts:1-10`) → [[pi--custom-provider-registration\|custom-provider]] |
| `extensions/built-in-tool-renderer.ts`, `minimal-mode.ts` | re-register read/bash/edit/write to change rendering only (collapsed tool display) → [[pi--extension-ui-primitives\|extension-ui-primitives]] |
| `extensions/qna.ts`, `summarize.ts` | "prompt generator" pattern: side LLM call (`ctx.modelRegistry.complete`) result loaded into editor / custom UI (`qna.ts:1-8`; `summarize.ts:146-181`) |
| `extensions/commands.ts`, `session-name.ts`, `system-prompt-header.ts` | `pi.getCommands()`, `setSessionName`, `ctx.getSystemPrompt()` |
| `extensions/with-deps/` | extension with own `node_modules` resolved by jiti → [[pi--runtime-plugin-loading\|plugin-loading]] |
| `extensions/github-issue-autocomplete.ts` | custom `#issue` autocomplete provider (preloads ≤100 issues via `gh`) |
| `extensions/custom-footer.ts`, `custom-header.ts`, `hidden-thinking-label.ts`, `message-renderer.ts`, `widget-placement.ts`, `border-status-editor.ts`, `modal-editor.ts`, `rainbow-editor.ts`, `mac-system-theme.ts`, `working-message-test.ts`, `overlay-test.ts`, `overlay-qa-tests.ts`, `doom-overlay/`, `snake.ts`, `space-invaders.ts` | UI surface demos (footer/header/editor replacement, overlays, Kitty key release) → [[pi--extension-ui-primitives\|extension-ui-primitives]] |
| `sdk/01-…14-*.ts` | one per embedding boundary (listed in [[pi--sdk-embedding\|sdk-embedding]]) |
| `plugins/pi-example-plugin` | experimental Chord plugin: `session` facet (remote greeting service) + `tui` facet (`/hello`), built by Chord into a server-owned cache and shipped to clients (`README.md:1-5`) → [[pi--client-server-session-split\|client-server]] |

**Source modules without a dedicated note** (all else in `src/` is cited by some note)
| module | role |
|---|---|
| `cli/session-picker.ts`, `cli/config-selector.ts` (`pi config`), `cli/startup-ui.ts`, `cli/list-models.ts` | interactive pickers / startup banner / `--list-models` (15 s abort, `main.ts:887`) |
| `modes/interactive/chat-viewport.ts`, `tui-renderer.ts`, `components/*` (selectors, dialogs, `mermaid.ts`, `diff.ts`, easter eggs) | TUI composition → [[pi--differential-tui-rendering\|tui]] |
| `utils/wsl.ts` | WSL detection (`WSL_DISTRO_NAME`/`WSLENV` or `/proc/version`, `:4-14`) so clipboard text/image copy falls back to Windows interop (`utils/clipboard.ts:116`, `clipboard-image.ts:214`) |
| `utils/deprecation.ts`, `changelog.ts`, `frontmatter.ts`, `git.ts`, `json.ts`, `zip.ts`, `open-browser.ts`, `syntax-highlight.ts`, `sleep.ts` | helpers (deprecation warnings deduped per message, `deprecation.ts:3-8`) |
| `experimental/{vacation,durable,services}/*`, `experimental/plugins/*` | experimental client/server + durable harness wiring → [[pi--client-server-session-split\|client-server]], [[pi--durable-execution\|durable-execution]] |

## Navigation
[[Home]] · [[Constants]] · [[Loop]] · [[Model Interface]] · [[Tools]] · [[Prompting]] · [[Context]] · [[Caching]] · [[Safety]] · [[State]] · [[Subagents]] · [[Platform]]
