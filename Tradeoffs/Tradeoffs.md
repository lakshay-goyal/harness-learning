---
type: group
group: tradeoffs
---
# Tradeoffs

Design axes where studied harnesses choose differently. Each note: the axis, a table of options × harnesses with evidence, and when each option wins. Harnesses so far: [[pi]] (`b30a6dd77`), [[opencode]] (`ecc4916b5a`), [[codex]] (`622e9e3696`). Axes first written pi-vs-opencode carry codex in an `## Also: [[codex]]` section folded in from the codex run. Many axes mirror a pi absence ([[Absences]]); values live in [[Constants]].

## Safety and control
- [[permission-prompts-vs-none]] — YOLO + hooks (pi) vs allow/ask/deny ruleset (opencode); neither claims it is security.
- [[cwd-confinement-vs-none]] — no path boundary (pi) vs `external_directory` ask (opencode).
- [[turn-cap-vs-none]] — no cap (pi) vs opt-in step budget + 3-identical-call guard (opencode); neither caps by default.
- [[bash-timeout-default-vs-none]] — no default (pi) vs 2 min, legacy no max, v2 10 min max (opencode).
- [[undo-vs-none]] — git is the undo (pi) vs shadow-git snapshots per step (opencode).

- [[isolation-strategy]] — no sandbox (pi) vs OS sandbox + approval policies + LLM reviewer + egress proxy (codex).
## Toolset
- [[minimal-vs-rich-toolset]] — 4 default tools (pi) vs ~14, pruned by usage (opencode). · codex: shell + `apply_patch`, no read/grep tools, model-gated plan
- [[edit-tool-variants]] — multi-edit + light normalization (pi) vs fuzzy cascade + `apply_patch` for GPT (opencode legacy) vs exact-only (opencode v2). · codex: patch envelope only
- [[web-tools-vs-none]] — curl/CLI/MCP (pi) vs built-in `webfetch`/`websearch` (opencode). · codex: hosted web search
- [[todo-tool-vs-none]] — TODO.md (pi) vs `todowrite` (opencode). · codex: `update_plan` (opt-in since 2026-08-31)
- [[lsp-feedback-vs-none]] — none (pi) vs post-edit diagnostics, opt-in (opencode).
- [[mcp-builtin-vs-extension]] — replaceable extension, deferred exposure (pi) vs core, direct declaration (opencode).

## Agents and modes
- [[builtin-subagents-vs-none]] — human orchestrates (pi) vs `task` tool with `general`/`explore` (opencode). · codex: in-process threads + mailbox
- [[plan-mode-vs-none]] — plans in files (pi) vs `plan` agent + approval exit (opencode). · codex: collaboration mode Plan

## Model interface
- [[provider-breadth]] — ~20 adapters (pi) vs Responses-API-only + model catalog (codex).

## Prompt and context
- [[single-vs-per-model-system-prompt]] — one tiny prompt (pi) vs ~10 per-family prompts (opencode). · codex: per-model prompt owned by the model catalog
- [[prompt-cache-strategy]] — transcript-carried prompt (pi) vs per-request rebuild (opencode legacy) vs Context Epochs (opencode v2). · codex: stateless full resend + `prompt_cache_key` + WebSocket append
- [[compaction-design]] — fixed headroom/keep (both), scaled keep window and optional pruning (opencode). · codex: local handoff summary or server `/responses/compact`
- [[mid-run-user-input]] — in-memory steering/follow-up queues (pi) vs persist-then-pick-up (opencode legacy) vs durable inbox (opencode v2).

## State and platform
- [[session-store-format]] — JSONL tree per session (pi) vs SQLite + event-sourced projections (opencode). · codex: linear rollout JSONL + SQLite index
- [[client-server-vs-single-process]] — single process + RPC (pi) vs server-first with TUI/web/ACP clients (opencode).

- [[tui-rendering-strategy]] — differential redraw (pi) vs native scrollback (codex).
- [[extensibility-model]] — runtime TS extensions (pi) vs compile-time ext crates + declarative plugins/skills/hooks/MCP (codex).
## Patterns across axes
- **Convergence under pressure**: both bound recovery loops but not the work loop; both refuse to call approvals "security"; opencode v2's Context Epoch re-derives pi's append-only system prompt; opencode turned LSP, formatters and pruning default-off, moving toward pi's defaults.
- **Opposite failure logs**: pi's absences are stated up front and later softened (MCP reversed). opencode's are learned by removal (batch, plan_enter, task_status, read-before-write guard) → [[removed-builtin-tools]].
- **Two halves of safety**: pi gates inputs (project trust), opencode gates actions (ruleset) → [[no-project-trust-gate]], [[no-permission-prompts]].
