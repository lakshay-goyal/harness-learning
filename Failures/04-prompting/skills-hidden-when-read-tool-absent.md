---
type: failure
concepts: [skill-progressive-disclosure, dynamic-tool-guidelines]
harnesses: [pi]
---
**Symptom** — With `--tools bash` (no `read`), the `<available_skills>` list disappeared from the prompt entirely, so the model never learned skills existed (#8552); later, when `read` was hidden behind codemode/`prepareLoadout`, the hint named a tool the model couldn't call (#10343).

**Root cause** — Skill disclosure was gated on the `read` tool and the hint hard-coded "Use the read tool…", though bash (`cat`) or an indirect reader can load SKILL.md just as well.

**Fix · [[pi]]**
- `1d6dbf9e3` 2026-09-03 (#8552) "keep skills available with bash-only tools": "Use bash to load a skill's file when the task matches its description."
- `c30840c2e` 2026-10-05 (#10343): hidden reader → "indirect" hint "Load a skill's file when the task matches its description." naming no tool (HEAD `packages/coding-agent/src/core/system-prompt.ts:174-182`, `skills.ts:366-371`).

**Lesson** — Gate disclosure on *any* capability that can fetch the content, and phrase the hint in terms of the tool actually declared.

Related: [[skill-progressive-disclosure]] · [[dynamic-tool-guidelines]] · [[prompt-names-unavailable-tools]] · [[pi--skill-progressive-disclosure|pi]]
