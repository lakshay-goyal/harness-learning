---
type: failure
concepts: [harness-identity]
harnesses: [pi]
---
**Symptom** — The prompt told models "You are actually not Claude, you are Pi." — overriding model identity; reverted 11 days later (#73). (Specific misbehavior in #73 not fetched — unverified.)

**Root cause** — Conflating "which harness am I in" with "who am I"; identity overrides fight model training.

**Fix · [[pi]]**
- `0c5cbd006` 2025-11-16: identity override added (with self-docs pointer).
- `8b1cca827` 2025-11-27 (#73) "remove identity override from system prompt": "Models now use their native identity instead of being told they are Pi."
- `4068bc556` 2026-01-17: harness identity without model identity — "You are an expert coding assistant operating inside pi, a coding agent harness." (HEAD `packages/coding-agent/src/core/system-prompt.ts:155-156`).
- Only remaining identity sentence is provider-mandated (Anthropic OAuth "You are Claude Code, Anthropic's official CLI for Claude.", `packages/ai/src/api/anthropic-messages.ts:1167-1182`) → [[provider-identity-shim]].

**Lesson** — Tell the model where it runs, never who it is.

Related: [[harness-identity]] · [[provider-identity-shim]] · [[minimal-system-prompt]] · [[pi--harness-identity|pi]]
