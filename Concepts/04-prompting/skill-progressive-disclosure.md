---
type: concept
stage: context
tier: candidate
aliases: [skills, SKILL.md, "<available_skills>", formatSkillsForPrompt, "/skill:name", disable-model-invocation, "<skills_instructions>", "$SkillName", skills.read, skills.list, codex-skills, dynamic_skill_selector, SkillScope]
harnesses: [pi, codex]
---
Only each skill's name, description and file path sit in the prompt; the model reads the skill body with a file tool when a task matches.

## Why
- Inlining every skill's instructions bloats every request; listing metadata costs ~4 lines per skill.
- If the hint names a tool that isn't available, skills silently vanish or become unloadable ([[skills-hidden-when-read-tool-absent]]).
- Skill-relative paths get resolved against cwd unless the rule is stated ([[instruction-relative-paths-resolved-from-cwd]]).

## Design space
- Inline full skill bodies (rejected; cost).
- **Metadata list + model-initiated read** (**pi chose**, Agent Skills XML format).
- Skills as tools (one tool per skill) vs file reads (pi: file reads via read/bash).
- User-only skills hidden from model (`disable-model-invocation`, invocable as `/skill:name` — **pi**).
- Explicit invocation inlines body into user message (**pi** `/skill:` → `<skill name location>` block).
- Validation strictness: hard-fail vs **warn-and-load** (**pi**; only missing description blocks).
- Trust: project/ancestor `.agents/skills` gated by project trust (**pi**).
- **Catalog budget** with graceful degradation: ≤2 % of window / 8,000 chars default, descriptions ≤1,024 chars, then *all* descriptions dropped with a warning; long paths compressed to root aliases ✔ codex.
- Read depth: "Read only enough to follow the workflow" (codex until 2026-06) → **read the chosen SKILL.md completely, to EOF; progressive disclosure selects files, not how much of one** ✔ codex ([[partial-file-read-acted-on]]); never delegate skill interpretation to a subagent ✔ codex.
- Turn scope: skill applies to the turn it was triggered; "Do not carry skills across turns unless re-mentioned" ✔ codex.
- Dedicated `skills.list` / `skills.read` tools for packages not on the host filesystem ✔ codex vs file reads ✔ pi.
- Retrieval pruning of the catalog (lexical BM25 / n-gram selector) — codex runs it as shadow metrics only (`c100109280`).
- Catalog as a world-state section (diffed, not re-sent) ✔ codex → [[world-state-diff-injection]].

## Implementations
- [[pi--skill-progressive-disclosure|pi]] — `<skills>` section, reader-aware hint (read/bash/indirect), spec limits 64/1024, collision diagnostics.
- [[codex--skill-progressive-disclosure|codex]] — `<skills_instructions>` developer block with trigger rules, 2 %-of-window budget, root-alias paths, read-to-EOF rule, System/Admin/Repo/User roots, plugin skills.

## Failures
- [[skills-hidden-when-read-tool-absent]]
- [[instruction-relative-paths-resolved-from-cwd]]
- [[self-reference-not-routed-to-docs]]
- Cross-group: [[partial-file-read-acted-on]] (03-tools)

## Related
[[prompt-template-expansion]] · [[context-file-hierarchy]] · [[dynamic-tool-guidelines]] · [[deferred-tool-loading]] · [[project-trust-gate]] · [[harness-package-distribution]] · [[self-documentation-pointer]] · [[world-state-diff-injection]] · [[agent-roles]]
