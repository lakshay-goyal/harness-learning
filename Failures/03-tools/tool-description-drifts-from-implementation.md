---
type: failure
concepts: [tool-description-design, search-replace-edit, shell-execution, web-tools]
harnesses: [opencode]
---
**Symptom** — Tool descriptions promised parameters, checks or semantics the implementation did not have, and the model acted on the promise:
- bash description pasted from Claude Code advertised `run_in_background`, which the tool did not accept.
- `apply_patch` description (from Codex) said "This is a FREEFORM tool, so do not wrap the patch in JSON" while opencode exposes it as a JSON function tool.
- read description claimed image support the tool lacked (copied text).
- webfetch claimed a "self-cleaning 15-minute cache".
- Still at HEAD: bash "Executes a given bash command in a persistent shell session" though every call is a fresh process (`packages/opencode/src/tool/shell/prompt.ts:259`); `edit.txt:4`/`write.txt:5` say the tool errors without a prior read, but that check was deleted in `76a141090e`; websearch "Domain filtering and advanced search options available" with no such parameter (`packages/opencode/src/tool/websearch.txt:11`); `MAX_STEPS_PROMPT` "Tools are disabled" while legacy still sends tools; grep.txt routes counting to bash `rg`, contradicting the shell mapping.

**Root cause** — Descriptions copied from other harnesses or left behind when code changed; nothing ties description claims to code.

**Fix · [[opencode]]**
- `7d54f893c9` 2025-08-13 read description stops claiming image support (real support `225adc46ba` 2025-10-09).
- `9c126c5b64` 2025-12-11 webfetch cache claim removed.
- `751899eeec` 2025-12-17 "remove unsupported parameter from bash tool description" (added by `de8460cb99` 2025-12-10).
- `dd0906be8c` 2026-01-19 apply_patch FREEFORM line removed.
- Unfixed at HEAD: persistent-shell claim (since `904061c243` 2025-03-25), read-before-edit claims, websearch domain filtering, max-steps text in legacy.

**Lesson** — Generate description facts from code (constants, schema) or test that every quoted string and promised parameter exists; diff borrowed descriptions against the local schema.

Related: [[tool-description-design]] · [[tool-description-lies-about-async]] · [[borrowed-prompt-foreign-references]] · [[search-replace-edit]] · [[shell-execution]] · [[web-tools]] · [[step-budget-limit]] · [[opencode--tool-description-design|opencode]]
