---
type: failure
concepts: [model-catalog]
harnesses: [pi]
---
**Symptom**
- Bedrock prompt caching was enabled for non-Claude models that merely had cache pricing, which returned errors.
- Application inference profiles (opaque ARNs) got no caching and no adaptive thinking.
- Claude 5 Bedrock ids got no caching.

**Root cause** — Capability detection matched substrings of the model id. ARNs and inference-profile ids hide the model family, and new id shapes miss the patterns.

**Fix · [[pi]]**
- `d442bbcc1` 2026-01-13 — added Bedrock prompt caching for Claude.
- `1a4d153d7` 2026-03-14 — limited Bedrock caching to Claude, not "any model with cache pricing" (#2053).
- `31d59f851` 2026-03-18 — `AWS_BEDROCK_FORCE_CACHE=1` escape hatch for application inference profiles (#2346).
- `5a07d946e` 2026-04-26 — also check `model.name` for caching and adaptive thinking (#3527).
- `ed4bc7308` 2026-04-28 — normalize names with `[\s_.:]→-` (`packages/ai/src/api/bedrock-converse-stream.ts:758-778`).
- `114bacf34` 2026-07-02 — Claude 5 ids (#6235).
- HEAD `supportsPromptCaching` (`bedrock-converse-stream.ts:861-892`): requires "claude" in id or name, else FORCE_CACHE.

**Lesson** — Capability sniffing needs a user-controlled fallback (name metadata, env override). Prefer explicit catalog flags; see [[thinking-config-per-model-drift]].

Related: [[model-catalog]] · [[pi--model-catalog|pi]] · [[thinking-level-abstraction]]
