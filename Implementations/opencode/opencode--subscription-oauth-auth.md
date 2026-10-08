---
type: implementation
harness: opencode
concept: subscription-oauth-auth
commit: ecc4916b5a
files: [packages/opencode/src/plugin/openai/codex.ts:12-14, packages/opencode/src/plugin/xai.ts:22-30, packages/opencode/src/plugin/github-copilot/copilot.ts:14, packages/opencode/src/account/account.ts:138, packages/opencode/src/plugin/index.ts:71-83]
---
[[subscription-oauth-auth]] in [[opencode]].

## Mechanism
- Built-in subscription logins: ChatGPT (Codex plugin, PKCE loopback on port 1455, endpoint `https://chatgpt.com/backend-api/codex/responses`, allow-listed models) (`packages/opencode/src/plugin/openai/codex.ts:12-14`), GitHub Copilot, GitLab Duo, xAI SuperGrok (device code) (`packages/opencode/src/plugin/index.ts:71-83`).
- **Claude Pro/Max is not built in** since `1ac1a0287c` 2026-03-19 ("anthropic legal requests"); docs: "Anthropic explicitly prohibits this … no longer the case as of 1.3.0" → [[opencode--provider-identity-shim|provider-identity-shim]].
- Device-code polling adds a 3 s safety margin to the server interval (Codex, xAI, Copilot) to avoid `slow_down`.
- opencode console account: eager refresh when < 5 min validity; concurrent refreshes coalesced (`packages/opencode/src/account/account.ts:138`).
- Credentials stored in `<data>/auth.json` mode 0600.

## Constants
| name | value | path:line |
|---|---|---|
| Codex `OAUTH_PORT` | 1455 | `packages/opencode/src/plugin/openai/codex.ts:13` |
| `OAUTH_POLLING_SAFETY_MARGIN_MS` | 3000 | `packages/opencode/src/plugin/openai/codex.ts:14`; `packages/opencode/src/plugin/xai.ts:26`; `packages/opencode/src/plugin/github-copilot/copilot.ts:14` |
| xAI device-code interval / min / slow-down / expiry | 5 s / 1 s / +5 s / 5 min | `packages/opencode/src/plugin/xai.ts:22-25` |
| xAI refresh skew | 120 s | `packages/opencode/src/plugin/xai.ts:30` |
| account eager refresh | 5 min | `packages/opencode/src/account/account.ts:138` |

## Evolution
- 2025-06-05 `35b03e4cb3` Claude OAuth support (with identity spoof); 2026-03-19 `1ac1a0287c` removed.
- 2026-01-09 `172bbdaced` Codex auth; 2026-01-22 `e4286ae7a3` Codex refresh token not persisted.
- 2026-02-16 `ef979ccfa8` GitLab mid-session token refresh; 2026-02-17 `fb79dd7bf8` MCP OAuth rejected credentials invalidated.
- 2026-04-01 `c619caefdd` coalesce concurrent console refreshes; `00d6841f84` refresh before expiry.
- 2026-05-20 `b32debb8a3` xAI Grok OAuth + device code.

Failures: [[credential-expires-mid-run]] · [[oauth-refresh-token-rotation-lost]].

Contrast: [[pi--subscription-oauth-auth|pi]] still ships Anthropic subscription OAuth with identity shims; opencode removed it under legal pressure and keeps ChatGPT, Copilot, GitLab and xAI.
