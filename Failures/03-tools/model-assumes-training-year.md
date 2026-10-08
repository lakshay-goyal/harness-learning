---
type: failure
concepts: [web-tools]
harnesses: [opencode]
---
**Symptom** — For "latest X" searches the model added its training-era year to queries (e.g. "AI news 2024"), returning stale results.

**Root cause** — The model's sense of "now" comes from training data unless the harness states the date where queries are formed.

**Fix · [[opencode]]**
- `1a43e5fe87` 2026-01-15 "adjust websearch tool to emphasize that it ISNT 2024": current date injected into the websearch description.
- `179c40749d` 2026-02-13 date reduced to the year "to avoid cache busts": "The current year is {{year}}. You MUST use this year when searching … search for 'AI news 2026', NOT 'AI news 2025'" (`packages/opencode/src/tool/websearch.txt:13-14`).
- The env block in the system prompt also carries `Today's date` (`packages/opencode/src/session/system.ts:83`).

**Lesson** — Put the current date where queries are formed, quantized to the coarsest useful unit so caches survive.

Related: [[web-tools]] · [[volatile-system-prompt-prefix]] · [[no-date-in-prompt]] · [[opencode--web-tools|opencode]]
