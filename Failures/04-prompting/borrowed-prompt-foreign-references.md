---
type: failure
concepts: [per-model-system-prompt, tool-description-design]
harnesses: [opencode]
---
**Symptom** — Prompts lifted from other harnesses reference commands, files, tools, agents and paths opencode lacks; several were fixed only after models followed them:
- gemini.txt (gemini-cli): "Most tool calls … will first require confirmation" (l.58), "/bug command" (l.62).
- beast.txt (VS Code Copilot Beast Mode): memory file `.github/instructions/memory.instruction.md` with `applyTo` front matter (l.114), markdown todo lists.
- copilot-gpt-5.txt (VS Code Copilot `<gptAgentInstructions>`), routed for three weeks then orphaned.
- plan-reminder-anthropic.txt (Claude Code): literal `/Users/aidencline/.claude/plans/happy-waddling-feigenbaum.md` (l.10); never imported. plan-mode.txt asks for a "Plan agent" that does not exist.
- apply_patch description "FREEFORM tool" (Codex); bash `run_in_background` (Claude Code); task.txt example agents taken for real ones (Claude Code).
- `anthropic-20250930.txt`: 166-line verbatim Claude Code prompt ("/help: Get help with using Claude Code"), never imported.

**Root cause** — Prompts copied wholesale from gemini-cli, Copilot, Claude Code and Codex without auditing against the local tool/command/agent roster.

**Fix · [[opencode]]**
- `790e9947bd` 2025-08-12 "fix: task tool prompt": "NOTE: The agents below are fictional examples for illustration only"; examples deleted in `548648a3d9` 2026-05-16.
- `751899eeec` 2025-12-17 `run_in_background` removed; `dd0906be8c` 2026-01-19 FREEFORM line removed.
- `1ac1a0287c` 2026-03-19 "anthropic legal requests": `anthropic-20250930.txt` deleted.
- Unfixed at HEAD: gemini.txt, beast.txt, copilot-gpt-5.txt, plan-reminder-anthropic.txt, plan-mode.txt.

**Lesson** — Audit every borrowed prompt against the local roster before shipping; delete orphaned prompt files.

Related: [[per-model-system-prompt]] · [[foreign-harness-tool-hallucination]] · [[tool-description-drifts-from-implementation]] · [[prompt-names-unavailable-tools]] · [[provider-identity-shim]] · [[opencode--per-model-system-prompt|opencode]]
