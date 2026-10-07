---
type: implementation
harness: opencode
concept: provider-identity-shim
commit: ecc4916b5a
files: [94dd0a8dbe^:packages/opencode/src/session/prompt/anthropic_spoof.txt, 1ac1a0287c^:packages/opencode/src/session/prompt/anthropic-20250930.txt, 1ac1a0287c^:packages/opencode/src/session/llm.ts:207-221, packages/opencode/src/plugin/index.ts:71-83]
---
[[provider-identity-shim]] in [[opencode]].

## Mechanism — **removed** (historical)
- `anthropic_spoof.txt` = one line "You are Claude Code, Anthropic's official CLI for Claude." prepended to the system prompt for Anthropic providers via `SystemPrompt.header(providerID)` (added `35b03e4cb3` 2025-06-05 "claude oauth support"; `94dd0a8dbe^:packages/opencode/src/session/prompt/anthropic_spoof.txt`).
- `anthropic-20250930.txt`: 166-line verbatim Claude Code system prompt (incl. "/help: Get help with using Claude Code"), added `5a507023a6` 2025-09-30, never imported, deleted with the legal commit.
- Headers before removal: `anthropic-beta: claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14` set in the Anthropic provider loader (`1ac1a0287c^:packages/opencode/src/provider/provider.ts:154`); `User-Agent: opencode/<ver>` sent to every provider **except** `anthropic` (`1ac1a0287c^:packages/opencode/src/session/llm.ts:207-221`) — opencode suppressed its own identity toward Anthropic.
- Bundled `opencode-anthropic-auth@0.0.13` plugin supplied the Claude Pro/Max OAuth token.

## Removal
- 2026-01-25 `94dd0a8dbe` "rm spoof": `SystemPrompt.header` replaced by an empty system array plus plugin hook `experimental.chat.system.transform` (the spoof moved into the external auth plugin).
- 2026-03-19 `1ac1a0287c` "anthropic legal requests (#18186)": removed the built-in auth plugin, the 166-line prompt, the `claude-code-20250219` beta, and the provider-conditional `User-Agent`; docs now say "Anthropic explicitly prohibits this. Previous versions of OpenCode came bundled with these plugins but that is no longer the case as of 1.3.0" (`packages/web/src/content/docs/providers.mdx`).

## Current residue
- Codex plugin targets the ChatGPT Codex endpoint with an allow-list of models (`packages/opencode/src/plugin/openai/codex.ts:12-15`) — subscription access, not a prompt impersonation (unverified whether Codex-identifying headers are sent).

## Constants
none at HEAD.

## Evolution
- 2025-06-05 `35b03e4cb3` → 2025-09-30 `5a507023a6` → 2026-01-25 `94dd0a8dbe` → 2026-03-19 `1ac1a0287c`.

## Quirks / drift
- Identity shims are a business/legal risk, not only a technical one.

Failures: [[vendor-prompt-copy-legal-exposure]].

Contrast: [[pi--provider-identity-shim|pi]] still runs an Anthropic OAuth "stealth mode" with Claude Code tool names; opencode removed its equivalent after a legal request.
