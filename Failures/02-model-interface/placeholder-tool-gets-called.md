---
type: failure
concepts: [transcript-replay-repair, unified-provider-api]
harnesses: [opencode]
---
**Symptom** — Copilot and LiteLLM-Anthropic proxies reject requests whose history contains tool calls but which send no `tools` field (compaction, title). The harness added a placeholder `_noop` tool to satisfy them, and models then **called** the placeholder.

**Fix · [[opencode]]**
- `196a03caff` 2026-03-30 "discourage _noop tool call during LiteLLM compaction": description hardened to "Do not call this tool. It exists only for API compatibility and must never be invoked." (`packages/opencode/src/session/llm/request.ts:159-175`).
- `f9d99f044d` 2026-04-15 kept Copilot compaction requests valid.
- `7f7eb2e7f8` 2026-05-14 removed the LiteLLM workarounds once ported upstream (requires LiteLLM v1.85.0-rc.2+); Copilot keeps `_noop`.

**Lesson** — Compatibility shims are visible to the model: describe them as forbidden, keep them argument-light, and delete them as soon as upstream is fixed.

Related: [[transcript-replay-repair]] · [[empty-payload-rejections]] · [[opencode--transcript-replay-repair|opencode]]
