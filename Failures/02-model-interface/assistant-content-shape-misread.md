---
type: failure
concepts: [unified-provider-api, cross-provider-handoff]
harnesses: [pi]
---
**Symptom** — Three variants:
- Claude via GitHub Copilot re-answered every previous prompt on each turn.
- DeepSeek V3.2 via NVIDIA NIM produced recursively nested `[{'type':'text',...}]` output.
- `requiresThinkingAsText` replay crashed or garbled the message (#3387).

**Root cause**
- Assistant content was sent as a content-part array. Copilot's Claude translation mis-segmented the history, and DeepSeek mirrored the array structure in its own output.
- Thinking text was `unshift`ed onto content that had already become a string (inferred from the `1d488626d` diff).

**Fix · [[pi]]**
- `4894fa411` (2025-12-17, Release v0.23.2), #209: Copilot-only fix — joined string when `model.provider === "github-copilot"`, plus `Openai-Intent: conversation-edits` and any-assistant/tool `X-Initiator` (in `packages/ai/src/providers/openai-completions.ts` at that commit; moved to `src/api/` in `ba93da9a9`) (`packages/coding-agent/CHANGELOG.md:5603`).
- `23109b113` (2026-03-10), #2008: generalized to all providers — assistant text sent as a plain joined string (HEAD `packages/ai/src/api/openai-completions.ts:1330-1337`). Exception at HEAD: `requiresThinkingAsText` models still get a text-part array (`:1328`).
- `1d488626d` (2026-04-20), #3387: rebuild as `[thinkingText, ...textParts]` with no tags (`:1323-1328`).

**Lesson** — Send the most canonical wire form. Models learn from the transcript's shape, and translating adapters can silently mis-segment history. The symptom is behavioral, not an error. Keep intermediate representations typed until final serialization.

Related: [[unified-provider-api]] · [[cross-provider-handoff]] · [[thinking-tag-mimicry]] · [[pi--unified-provider-api|pi]]
