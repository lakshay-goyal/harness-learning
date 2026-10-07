---
type: concept
stage: context
tier: candidate
aliases: [skills, SKILL.md, "<available_skills>", formatSkillsForPrompt, "/skill:name", disable-model-invocation, "<skill_content>", OPENCODE_DISABLE_CLAUDE_CODE_SKILLS]
harnesses: [pi, opencode]
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
- Dedicated skill tool returning body + sampled sibling files (opencode) vs file read (pi).
- Remote skill indexes pulled from URLs and cached (opencode).
- Read other harnesses' skill directories for compatibility (opencode: `~/.claude/skills`, `.agents/skills`).

## Implementations
- [[pi--skill-progressive-disclosure|pi]] — `<skills>` section, reader-aware hint (read/bash/indirect), spec limits 64/1024, collision diagnostics.
- [[opencode--skill-progressive-disclosure|opencode]] — verbose `<available_skills>` (name, description, location) in the system prompt, omitted when `skill` is denied; dedicated `skill` tool returns body + ≤10 sampled files; scans `.claude`/`.agents` skill dirs and remote indexes.

## Failures
- [[error-message-leaks-filtered-catalog]]
- [[skills-hidden-when-read-tool-absent]]
- [[instruction-relative-paths-resolved-from-cwd]]
- [[duplicated-catalog-in-prompt]]

## Tradeoffs
- [[web-tools-vs-none]]

## Related
[[prompt-template-expansion]] · [[context-file-hierarchy]] · [[dynamic-tool-guidelines]] · [[deferred-tool-loading]] · [[project-trust-gate]] · [[harness-package-distribution]]
