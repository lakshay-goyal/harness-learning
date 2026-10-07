---
type: failure
concepts: [provider-identity-shim]
harnesses: [pi]
---
**Symptom** — With Anthropic subscription OAuth ("stealth mode"), tool names were mapped to Claude Code names and the round trip failed. pi's `find` mapped to `Glob`, so the model's `Glob` call came back under a name pi could not resolve, and the tool call failed.

**Root cause** — A static bidirectional pi→CC name map is not invertible when several pi tools map onto one CC tool, or when the tool set changes at runtime.

**Fix · [[pi]]**
- `ec83d9147` 2026-01-10 — resolve OAuth tool names back via the **current tool set** in context.
- `a5f1016da` 2026-01-17 — replaced the hardcoded map with a single canonical CC list (`Read, Write, Edit, Bash, Grep, Glob, AskUserQuestion, …` sourced from cchistory, `packages/ai/src/api/anthropic-messages.ts:98-119`) and dropped the broken `find→Glob` mapping:
  - `toClaudeCodeName` does a case-insensitive match to CC casing, else returns the name unchanged (`:121-124`). It is applied to tool definitions (`:1603`), replayed `tool_use` (`:1443`) and `tool_removal` (`:1347`).
  - `fromClaudeCodeName(name, currentTools)` does a case-insensitive match against live tools (`:125-132,727-729`).
- Tests: `packages/ai/test/anthropic-tool-name-normalization.test.ts:28,70,111,164`.

**Lesson** — Name mangling for an upstream convention must be invertible against the live tool set. Rename only by case-folding, never many-to-one.

Related: [[provider-identity-shim]] · [[pi--provider-identity-shim|pi]] · [[subscription-oauth-auth]]
