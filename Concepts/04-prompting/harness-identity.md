---
type: concept
stage: messages
tier: candidate
aliases: ["operating inside pi, a coding agent harness", "You are actually not Claude, you are Pi", PI_STATIC_INSTRUCTIONS, pi-codex-bridge, "You are a coding agent running in the Codex CLI", "You are Codex, based on GPT-5", "Within this context, Codex refers to the open-source agentic coding interface"]
harnesses: [pi, codex]
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
- **Harness identity + disambiguation from a same-named older product** ("Codex refers to the open-source agentic coding interface (not the old Codex language model built by OpenAI)") ✔ codex.
- **Model identity stated in a per-model prompt** ("You are GPT-5.1 running in the Codex CLI", "You are Codex, based on GPT-5.", "You are Codex, an agent based on GPT-6.") ✔ codex — safe only because the prompt is per-model catalog data ([[per-model-system-prompt]]).
- Rich persona as identity ("You have a vivid inner life as Codex: intelligent, playful, curious, and deeply present.", gpt-5.5) ✔ codex → [[personality-variants]].

## Implementations
- [[pi--harness-identity|pi]] — one harness-identity sentence; native model identity; Claude Code identity only for Anthropic OAuth.
- [[codex--harness-identity|codex]] — "You are a coding agent running in the Codex CLI…" fallback; per-model prompts name the model and give Codex a persona.

## Failures
- [[forced-model-identity-override]]
- [[foreign-harness-tool-hallucination]]

## Related
[[minimal-system-prompt]] · [[provider-identity-shim]] · [[subscription-oauth-auth]] · [[self-documentation-pointer]] · [[per-model-system-prompt]] · [[personality-variants]] · [[prompt-ownership]]
