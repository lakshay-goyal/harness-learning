---
type: failure
concepts: [repeated-tool-call-detection, turn-loop]
harnesses: [opencode]
---
**Symptom** — The model re-issues the same tool call with the same arguments again and again (same search, same failing edit), burning steps and tokens until the user aborts.

**Fix · [[opencode]]** `d983b9485d` 2025-10-30 "add doom loop detection" (#3445): when the last 3 parts of the current assistant message are the same tool with `JSON.stringify`-equal input and none pending, the processor raises a `doom_loop` permission ask (`packages/opencode/src/session/processor.ts:29,353-379`); default `ask` (`packages/opencode/src/agent/agent.ts:121`). `4e549b1c05` 2025-11-09 made it configurable (`allow`/`deny`); `47bfae52c0` 2025-11-18 fixed the permission check. Prompt-side complement: `95ad6758af` 2026-02-06 (#12144) Trinity prompt "Avoid repeating the same tool with the same parameters once you have useful results… do not search again in a loop" (`packages/opencode/src/session/prompt/trinity.txt:86`; see [[per-model-system-prompt]]). Not ported to the v2 runner (`packages/core/src/session/runner/llm.ts:55`).

**Lesson** — Exact-argument equality over a short window is a cheap, low-false-positive guard; route the decision to the human-approval channel rather than hard-stopping, and remember that within-step windows miss cross-step repeats.

Related: [[repeated-tool-call-detection]] · [[step-budget-limit]] · [[no-turn-cap]] · [[opencode--repeated-tool-call-detection|opencode]]
