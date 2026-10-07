---
type: failure
concepts: [system-prompt-override, transcript-carried-system-prompt]
harnesses: [pi]
---
**Symptom** — An extension's `before_agent_start` returning a full `systemPrompt` didn't actually replace the prompt on models that support mid-conversation system messages: they "kept the original prompt as their leading system prompt and received the forced one as a later update" — the original instructions stayed in charge.

**Root cause** — After `9e05370b2` moved the prompt into the transcript, a forced prompt was diffed/appended like any other change; a later system message is weaker than the leading one and doesn't remove it. Recording forced prompts also wrote a full prompt on every change.

**Fix · [[pi]]**
- `e4c75a732` 2026-09-16 "replace the system prompt when a handler forces it": forced prompt = opaque content with no sections (`buildSystemPromptState`, HEAD `packages/coding-agent/src/core/system-prompt.ts:199-205`).
- `16292398a` 2026-09-17 "send forced system prompts without recording them": transcript keeps structured sections; `_installAgentForcedPromptProjection` collapses all system messages into one head with the forced text + current tools at request time (`agent-session.ts:1786-1801`). Evals updated to validate the prompt the transform *sent* (findings 10).
- Earlier sibling: forced/run prompt dropped during tool refresh (#6162) → [[tool-loadout-stale-within-run]] (`e547bb9f4`, `fd6659dd5`).

**Lesson** — A full override must replace the head of the request, not append; keep the recorded transcript structured and project overrides at request time.

Related: [[system-prompt-override]] · [[transcript-carried-system-prompt]] · [[extension-event-hooks]] · [[pi--system-prompt-override|pi]]
