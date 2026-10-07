---
type: failure
concepts: [provider-identity-shim, subscription-oauth-auth]
harnesses: [opencode]
---
**Symptom** — To use Claude Pro/Max subscription tokens, the harness impersonated the vendor's own client: a "You are Claude Code, Anthropic's official CLI for Claude." system-prompt line, the `claude-code-20250219` beta header, its own `User-Agent` suppressed only toward Anthropic, a bundled `opencode-anthropic-auth` plugin, and a 166-line verbatim copy of the vendor's system prompt in the repo. This drew legal requests from the vendor.

**Fix · [[opencode]]**
- `94dd0a8dbe` 2026-01-25 "rm spoof": identity line moved out of core into the auth plugin via `experimental.chat.system.transform` (`94dd0a8dbe^:packages/opencode/src/session/prompt/anthropic_spoof.txt`).
- `1ac1a0287c` 2026-03-19 "anthropic legal requests (#18186)": removed the bundled auth plugin, `anthropic-20250930.txt` (added `5a507023a6` 2025-09-30, never imported), the `claude-code-20250219` beta (`1ac1a0287c^:packages/opencode/src/provider/provider.ts:154`) and the provider-conditional `User-Agent` (`1ac1a0287c^:packages/opencode/src/session/llm.ts:207-221`); docs: "Anthropic explicitly prohibits this … no longer the case as of 1.3.0".

**Lesson** — Identity shims that ride a consumer subscription are a business and legal risk, not only a technical one; vendor prompt text copied into the repo is evidence even if unused.

Related: [[provider-identity-shim]] · [[subscription-oauth-auth]] · [[harness-identity]] · [[opencode--provider-identity-shim|opencode]] · [[pi--provider-identity-shim|pi]]
