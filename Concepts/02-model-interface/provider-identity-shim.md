---
type: concept
stage: model-interface
tier: candidate
aliases: [Claude Code stealth mode, toClaudeCodeName, fromClaudeCodeName, Codex instructions, provider-identity-shim, harness-impersonation, subscription-client-impersonation, invertible-tool-name-mapping, client-impersonation-headers, sk-ant-oat, "x-app: cli", originator, anthropic_spoof.txt, claude-code-20250219, SystemPrompt.header]
harnesses: [pi, opencode]
---
When a harness uses a subscription token issued for a vendor's own client, it reshapes requests to look like that client:
- a mandated system-prompt prefix,
- client user-agent and app headers,
- beta flags,
- canonical tool-name casing.

Tool names are mapped back on the way in.

## Why
- Subscription endpoints accept or price requests based on the client's identity. Without the shim, consumer tokens are unusable.
- Renaming tools must be invertible against the tools that are live right now. A static two-way map breaks round-trips, so the model calls a tool the harness cannot find ([[tool-name-mapping-not-invertible]]).

## Design space
- **Scope**
  - Off, API keys only.
  - On for OAuth tokens only, detected from the token shape (e.g. `sk-ant-oat`). *pi chose this.*
- **Prompt**
  - Replace the harness prompt.
  - Prepend the vendor identity line as its own cached block, then the harness prompt. *pi chose this.*
- **Tool names**
  - A static bidirectional map. *pi tried this,* including `find→Glob`; it broke and was removed in a5f1016da.
  - A canonical list matched case-insensitively outbound, and inbound resolution against the current tool set. *pi chose this:* ec83d9147.
- **Headers**
  - Client UA and version string, bumped repeatedly.
  - App and originator headers.
  - Copilot editor-integration headers.
  - Codex `originator` / `chatgpt-account-id`.
- **Policy risk**
  - Removing the feature: f5e6bcac1 removed it and 19b566334 restored it the same day; the reason is unverified.
- **Removal under legal pressure**: identity line, copied vendor prompt, beta header and bundled auth plugin deleted (opencode `1ac1a0287c` 2026-03-19).

## Implementations
- [[pi--provider-identity-shim|pi]] — the Anthropic OAuth "stealth mode" in `anthropic-messages.ts`: Claude Code identity block, `claude-cli` UA, `claude-code-20250219` and `oauth` betas, CC tool-name casing. Also Copilot VS Code headers and Codex `originator`.
- [[opencode--provider-identity-shim|opencode]] — removed: "You are Claude Code…" prefix, `claude-code-20250219` beta, UA suppressed for Anthropic; deleted after Anthropic legal requests.

## Failures
- [[foreign-harness-tool-hallucination]]
- [[tool-name-mapping-not-invertible]]
- [[vendor-prompt-copy-legal-exposure]]

## Related
[[subscription-oauth-auth]] · [[harness-identity]] · [[http-transport-hardening]] · [[credential-resolution]]
