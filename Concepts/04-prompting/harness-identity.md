---
type: concept
stage: messages
tier: candidate
aliases: ["operating inside pi, a coding agent harness", "You are actually not Claude, you are Pi", PI_STATIC_INSTRUCTIONS, pi-codex-bridge]
harnesses: [pi]
---
Tell the model which harness it runs in (and therefore which tools/conventions apply) without overriding the model's own identity.

## Why
- Overriding identity ("you are not Claude, you are X") confuses models and was reverted in pi within 11 days ([[forced-model-identity-override]]).
- Models trained inside another harness reach for that harness's tools (`apply_patch`, `update_plan`) unless told where they are ([[foreign-harness-tool-hallucination]]).
- Provider subscription paths may *require* a specific identity sentence — that is a provider shim, not prompt design ([[provider-identity-shim]]).

## Design space
- Model identity override (pi Nov 2025; removed).
- No identity at all (pi v0).
- **Harness identity only: "operating inside <harness>, a coding agent harness"** (**pi chose**, `4068bc556`).
- Harness-specific bridge prompt per provider listing non-existent tools (pi Codex bridge, Jan 2026; removed).
- Provider-allowlisted frozen preamble (pi `PI_STATIC_INSTRUCTIONS`, 1 day).
- Provider-mandated identity block prepended only on that transport (pi Anthropic OAuth "You are Claude Code…").

## Implementations
- [[pi--harness-identity|pi]] — one harness-identity sentence; native model identity; Claude Code identity only for Anthropic OAuth.

## Failures
- [[forced-model-identity-override]]
- [[foreign-harness-tool-hallucination]]

## Related
[[minimal-system-prompt]] · [[provider-identity-shim]] · [[subscription-oauth-auth]] · [[self-documentation-pointer]]
