---
type: implementation
harness: pi
concept: provider-identity-shim
commit: b30a6dd77
files: [packages/ai/src/api/anthropic-messages.ts:95, packages/ai/src/api/anthropic-messages.ts:121, packages/ai/src/api/anthropic-messages.ts:1014, packages/ai/src/api/anthropic-messages.ts:1167, packages/ai/src/auth/oauth/github-copilot.ts:13, packages/ai/src/api/github-copilot-headers.ts:23, packages/ai/src/api/openai-codex-responses.ts:1640]
---
[[provider-identity-shim]] in [[pi]].

## Mechanism

### Anthropic "stealth mode" (Claude Pro/Max OAuth tokens)
- Trigger: OAuth credential's `toAuth → {apiKey: access}`; adapter detects OAuth by substring `sk-ant-oat` (`packages/ai/src/api/anthropic-messages.ts:978-980`) — so env `ANTHROPIC_OAUTH_TOKEN` also gets shaped (test `anthropic-auth-token.test.ts:114,167`). Injected `options.client` (e.g. `AnthropicVertex`) forces `isOAuth=false` (`:607-609`; `0acb65fe1`).
- Client: `authToken`, `user-agent: claude-cli/${claudeCodeVersion}`, `x-app: cli` (`:1014-1034`); `claudeCodeVersion = "2.1.280"` (`:95-96`), bumped repeatedly (`92882dc4c` 2.1.62→2.1.75, `96317e50b`, `3a624b82d` "update reported Claude Code version").
- Betas for OAuth: `claude-code-20250219`, `oauth-2025-04-20` (`:1106`; computed in `getBetaFeatures`, see [[pi--http-transport-hardening|http-transport-hardening]]).
- System prompt: first block literally `"You are Claude Code, Anthropic's official CLI for Claude."`, pi's system text as second block, both with `cache_control` (`:1167-1182`) → 2 of Anthropic's ~4 breakpoints spent on system ([[cache-breakpoint-placement]]). Contrast: pi's own [[harness-identity]] line says "pi" — the shim prepends a vendor identity only on the subscription path.
- **Tool renaming** ("Stealth mode: Mimic Claude Code's tool naming exactly"): canonical CC list `Read, Write, Edit, Bash, Grep, Glob, AskUserQuestion, EnterPlanMode, ExitPlanMode, KillShell, NotebookEdit, Skill, Task, TaskOutput, TodoWrite, WebFetch, WebSearch` sourced from cchistory (`:98-119`) — includes tools pi does not ship ([[no-todo-tool]], [[no-plan-mode]], [[no-web-tools]]).
  - Outgoing `toClaudeCodeName`: case-insensitive match → CC casing, else unchanged (`:121-124`); applied to tool defs (`:1603`), replayed `tool_use` names (`:1443`), `tool_removal` refs (`:1347`).
  - Incoming `fromClaudeCodeName(name, currentTools)`: case-insensitive match against the **current tool set** restores pi's name (`:125-132, 727-729`). `ec83d9147` resolve via context (old static reverse map); `a5f1016da` replaced hardcoded pi→CC map with single list, removed broken `find→Glob` ("round-trip failed"). Tests `anthropic-tool-name-normalization.test.ts:28,70,111,164`.
- `PiAnthropic` subclass disables SDK default credential chain (`_shouldResolveDefaultCredentials() → false`) so pi's resolver is sole authority (`:329-339`).

### GitHub Copilot (VS Code Copilot Chat impersonation)
- OAuth client id base64 `Iv1.b507a08c87ecfe98` (VS Code Copilot app); headers `GitHubCopilotChat/0.35.0`, `vscode/1.107.0`, `copilot-chat/0.35.0`, `Copilot-Integration-Id: vscode-chat`; `X-GitHub-Api-Version: 2026-06-01` (`packages/ai/src/auth/oauth/github-copilot.ts:10-19, 185`).
- Per-request dynamic headers (`packages/ai/src/api/github-copilot-headers.ts:1-37`): `X-Initiator = "agent"` if last message role ≠ user else `"user"` (`575dcb267`; premium-request billing semantics unverified); `Openai-Intent: conversation-edits`; `Copilot-Vision-Request: true` when any user/toolResult has an image (`0dbc1065a`). Applied in completions, responses and anthropic clients (`openai-completions.ts:767-774`, `openai-responses.ts:268-275`, `anthropic-messages.ts:992-1011` with `anthropic-dangerous-direct-browser-access`).
- Copilot Claude routed via Anthropic Messages (`0a132a30a`); GPT via Responses.

### OpenAI Codex / ChatGPT
- Codex: `originator: pi` default header, pi UA; caller may override originator/UA (`0cf65d2bf` #10429) (`packages/ai/src/api/openai-codex-responses.ts:1640-1661`); SSE adds `OpenAI-Beta: responses=experimental`; Bearer JWT + `chatgpt-account-id` from claim `https://api.openai.com/auth`.`chatgpt_account_id` (`:53, 1627-1638, 1658-1659`). Body `instructions` must be non-empty → `"You are a helpful assistant."` fallback (`15aa31350` #4184), `text.verbosity:"low"` default (`953f89fbe`, rationale undocumented).
- Login extras: `originator=<agentName|pi>`, `codex_cli_simplified_flow=true` (`auth/oauth/openai-codex.ts:289-308`); callback port 1455 shared with Codex CLI.
- Sign in with ChatGPT: registers pi honestly as a dynamic agent client (`dynamic_agent_client`, `agent_name_hint`) — not an impersonation; but token-sharing endpoint rejects `prompt_cache_retention`, `prompt_cache_options`, `max_output_tokens`, `temperature` → stripped (`openai-responses.ts:328-346`).
- Other first-party client ids reused: Meta Muse Code CLI `1031625952748946`, xAI `grok-cli:access` scope with `referrer: pi` (see [[pi--subscription-oauth-auth|subscription-oauth-auth]]).

## Constants
| name | value | path:line |
|---|---|---|
| `claudeCodeVersion` | 2.1.280 | packages/ai/src/api/anthropic-messages.ts:95-96 |
| OAuth betas | claude-code-20250219, oauth-2025-04-20 | packages/ai/src/api/anthropic-messages.ts:1106 |
| CC identity line | "You are Claude Code, Anthropic's official CLI for Claude." | packages/ai/src/api/anthropic-messages.ts:1167-1182 |
| CC tool list | 17 names | packages/ai/src/api/anthropic-messages.ts:98-119 |
| Copilot client headers | GitHubCopilotChat/0.35.0, vscode/1.107.0, vscode-chat | packages/ai/src/auth/oauth/github-copilot.ts:13-18 |
| `X-GitHub-Api-Version` | 2026-06-01 | packages/ai/src/auth/oauth/github-copilot.ts:19 |
| Codex default instructions | "You are a helpful assistant." | packages/ai/src/api/openai-codex-responses.ts:527-564 |

## Evolution
- 2025-12-19 `575dcb267` X-Initiator logic; `0dbc1065a` Copilot-Vision-Request.
- 2026-01-09 `f5e6bcac1` remove Anthropic OAuth → reverted `19b566334` same day (reason unverified).
- 2026-01-10 `ec83d9147` incoming names resolved via current tools; 2026-01-17 `a5f1016da` case-insensitive single CC list.
- 2026-02-06 `0a132a30a` Copilot Claude via Anthropic Messages.
- 2026-03-09 `0acb65fe1` injected client bypasses OAuth shaping.
- 2026-03-13 `92882dc4c`, 2026-09-02 `96317e50b`, 2026-09-22 `3a624b82d` reported CC version bumps.
- 2026-10-06 `0cf65d2bf` caller headers override Codex originator/UA.

## Evidence commits
575dcb267, 0dbc1065a, f5e6bcac1, 19b566334, ec83d9147, a5f1016da, 0a132a30a, 0acb65fe1, 92882dc4c, 96317e50b, 3a624b82d, 0cf65d2bf, 15aa31350, 953f89fbe

## Quirks
- The shim must be kept in lock-step with the vendor client (version string, beta list, tool list) — a maintenance treadmill evidenced by repeated version bumps.
- Name mapping is lossy if two pi tools differ only by case; resolution depends on the live tool set.
- `X-Initiator` heuristic semantics undocumented (unverified).
- Why Codex defaults verbosity low (`953f89fbe`) — quality trade-off not documented (unverified).

## Failures
- [[tool-name-mapping-not-invertible]]
- [[empty-payload-rejections]] (Codex empty instructions)
- [[endpoint-rejects-request-field]] (ChatGPT token field stripping)
