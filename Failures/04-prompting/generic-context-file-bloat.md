---
type: failure
concepts: [context-file-hierarchy]
harnesses: [opencode]
---
**Symptom** — `/init` produced AGENTS.md files full of generic style advice and file trees ("Make it about 150 lines long"), loaded into every request.

**Root cause** — The generator prompt asked for coverage (build commands + code style guidelines) and a length target rather than non-obvious facts.

**Fix · [[opencode]]** — `897d83c589` 2026-04-01 "tighten AGENTS guidance": "Every line should answer: 'Would an agent likely miss this without help?' If not, leave it out"; prefer executable sources of truth; exclude generic advice and file trees; ask via `question` at most once; improve existing files in place (`packages/opencode/src/command/template/initialize.txt`; old text `897d83c589^:packages/opencode/src/command/template/initialize.txt`).

**Lesson** — Context files should hold only facts an agent cannot infer from the repo.

Related: [[context-file-hierarchy]] · [[prompt-template-expansion]] · [[opencode--context-file-hierarchy|opencode]]
