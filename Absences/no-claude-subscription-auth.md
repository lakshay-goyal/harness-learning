---
type: absence
harnesses: [opencode]
---
# no-claude-subscription-auth

Removed: bundled Claude Pro/Max login and the "You are Claude Code" identity spoof.

**What's missing**
- No built-in Anthropic subscription OAuth. Built-in subscription auth remains for ChatGPT (Codex plugin), GitHub Copilot and GitLab Duo (`packages/opencode/src/plugin/index.ts:12-17`).
- No system-prompt line impersonating Claude Code.

**Evidence of decision**
- `94dd0a8dbe` (2026-01-25, "rm spoof") deleted "You are Claude Code, Anthropic's official CLI for Claude." (`94dd0a8dbe^:packages/opencode/src/session/prompt/anthropic_spoof.txt`).
- `1ac1a0287c` (2026-03-19, "anthropic legal requests", #18186):
  - removed the built-in `opencode-anthropic-auth` plugin from `packages/opencode/src/plugin/index.ts`;
  - deleted the 166-line `packages/opencode/src/session/prompt/anthropic-20250930.txt`;
  - docs now say: "There are plugins that allow you to use your Claude Pro/Max models with OpenCode. Anthropic explicitly prohibits this." (`packages/web/src/content/docs/providers.mdx:358`).
- Rationale is legal (commit subject), not technical.

**Contrast**
- pi still ships Anthropic OAuth (`packages/ai/src/auth/oauth/anthropic.ts:20`, callback port 53692, `92882dc4c`) plus a "stealth mode" that maps tool names to Claude Code casing → [[pi--subscription-oauth-auth|pi]], [[provider-identity-shim]].

**Implication**
- Subscription auth is a business/ToS surface, not just plumbing. A harness that impersonates a vendor client can be forced to drop it; third-party plugins keep the capability alive outside the core.

Related: [[subscription-oauth-auth]] · [[provider-identity-shim]] · [[harness-identity]] · [[credential-resolution]] · [[opencode]] · [[Absences]]
