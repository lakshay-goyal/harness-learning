---
type: implementation
harness: pi
concept: harness-identity
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:155-156, packages/ai/src/api/anthropic-messages.ts:1167-1192, packages/ai/src/api/openai-codex-responses.ts:557, packages/coding-agent/src/core/bug-report.ts:282-300]
---
[[harness-identity]] in [[pi]].

## Mechanism
- Preamble names the harness, never the model: "You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files." (`packages/coding-agent/src/core/system-prompt.ts:155-156`). Models keep their native identity (`8b1cca827` #73).
- The **only** model-identity sentence pi sends is provider-mandated: Anthropic OAuth (Claude Pro/Max subscription) — "For OAuth tokens, we MUST include Claude Code identity" → first system block "You are Claude Code, Anthropic's official CLI for Claude.", pi's prompt as second block, both with `cache_control` (`packages/ai/src/api/anthropic-messages.ts:1167-1182`); plus tool names mapped to Claude Code casing → [[provider-identity-shim]]. Example `examples/extensions/custom-provider-anthropic/index.ts:414` copies it.
- Codex (ChatGPT backend) requires non-empty `instructions`: leading system text or fallback "You are a helpful assistant." (`packages/ai/src/api/openai-codex-responses.ts:557`; `15aa31350` #4184).
- Harness identity is reused in side prompts: bug-report summarizer "You are helping a user file a bug report about pi, the coding agent they are talking to. … Do NOT continue the conversation…" (`bug-report.ts:282-300`; `3c75b2747`).
- Custom SYSTEM.md replaces the preamble → identity line gone unless the user restates it ([[system-prompt-override]]).
- **Rebrandable identity**: a source fork sets `package.json` `piConfig {name, configDir}` → `APP_NAME` (default `pi`), `APP_TITLE` (`π` unless renamed), `CONFIG_DIR_NAME` (default `.pi`), and env-var names derived from the app name, e.g. `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR` / `…_CODING_AGENT_SESSION_DIR` (`packages/coding-agent/src/config.ts:553-586`; `docs/cli-integration.md:82-95`). The system-prompt preamble is a string literal naming "pi", not `APP_NAME` (`system-prompt.ts:156`), so a rebranded fork still tells the model it runs inside pi unless it edits the prompt or ships a SYSTEM.md.

## Constants
| name | value | path:line |
|---|---|---|
| preamble | "…operating inside pi, a coding agent harness…" | `system-prompt.ts:156` |
| OAuth identity block | "You are Claude Code, Anthropic's official CLI for Claude." | `anthropic-messages.ts:1172` |
| Codex fallback instructions | "You are a helpful assistant." | `openai-codex-responses.ts:557` |
| `claudeCodeVersion` (reported) | 2.1.280 | `anthropic-messages.ts:95-96` |

## Evolution
- `ffc9be886` 2025-10-17: "You are an expert coding assistant." (no harness name).
- `0c5cbd006` 2025-11-16: "**You are actually not Claude, you are Pi.** You are an expert coding assistant…".
- `8b1cca827` 2025-11-27 (#73): removed — "Models now use their native identity instead of being told they are Pi." (11 days) → [[forced-model-identity-override]].
- **Codex detour (packages/ai)**:
  - `1650041a6` 2026-01-04: Codex OAuth provider shipped upstream `codex-instructions.md` (105 lines) + bridge `pi-codex-bridge.ts`: "# Codex Running in Pi … ## CRITICAL: Tool Replacements" / `<critical_rule priority="0">` "❌ APPLY_PATCH DOES NOT EXIST → ✅ USE "edit" INSTEAD — NEVER use: apply_patch, applyPatch — ALWAYS use: edit for ALL file modifications" / "❌ UPDATE_PLAN DOES NOT EXIST — NEVER use: update_plan, updatePlan, read_plan, readPlan, todowrite, todoread …" / "## Verification Checklist 1. Using edit, not apply_patch 2. No plan tools used 3. Only the tools listed above are called" → [[foreign-harness-tool-hallucination]].
  - `bb50738f7` 2026-01-05: pi system prompt appended after the bridge in a single bridge message.
  - `6dcb64565` 2026-01-10: "Prepare for alternative Codex harness certification".
  - `6484ae279` 2026-01-16 (#737): bridge + upstream prompts deleted; `PI_STATIC_INSTRUCTIONS` "You are pi, an expert coding assistant…" with "This string is whitelisted by OpenAI and must not change."; dynamic prompt sent as developer messages.
  - `4068bc556` 2026-01-17: static instructions deleted; new preamble "operating inside pi, a coding agent harness"; Codex uses normal system prompt as `instructions` (ai CHANGELOG "OpenAI Codex responses now use the context system prompt directly in the instructions field"; coding-agent CHANGELOG `:4110-4111`).
  - `15aa31350` 2026-05-05 (#4184): non-empty fallback instructions (CHANGELOG `:2044`).
- `f5e6bcac1` → `19b566334` 2026-01-09: "Remove Anthropic OAuth support" (with an accidental `prompt = "You are a helpful assistant. Be concise."` and OAuth test payloads) reverted same day — reason unverified.
- `3c75b2747` 2026-09-19: bug-report prompt names pi.

## Evidence commits
`ffc9be886`, `0c5cbd006`, `8b1cca827`, `1650041a6`, `bb50738f7`, `6dcb64565`, `6484ae279`, `4068bc556`, `15aa31350`, `f5e6bcac1`, `19b566334`, `3c75b2747`.

## Quirks
- Contradiction by design: harness says "use native identity" (`8b1cca827`) yet impersonates Claude Code for subscription tokens (provider requirement, not a prompt philosophy).
- The 1-day `PI_STATIC_INSTRUCTIONS` episode shows provider allowlisting can force a harness to freeze prompt text — pi chose to revert within 24h rather than couple.
- Whether OAuth path still renames tools to Claude Code names: yes, `toClaudeCodeName` on tool defs, replayed `tool_use`, `tool_removal` (`anthropic-messages.ts:121-132, 1347, 1443, 1603`).

## Failures
- [[forced-model-identity-override]]
- [[foreign-harness-tool-hallucination]]
