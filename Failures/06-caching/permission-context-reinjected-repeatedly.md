---
type: failure
concepts: [cache-stable-prompt-prefix, context-file-hierarchy]
harnesses: [codex]
---
**Symptom** — Context injections grew with every event or every environment:
- After each command approval (an exec-policy amendment), the full permissions instructions block was appended again (`1bbfb5cfad` 2026-08-03).
- A per-environment `project_doc_max_bytes` let the total AGENTS.md payload grow with the number of environments (`85e0661c3b` 2026-08-07).
- Full-history agent forks inherited the parent's multi-agent policy text (`663da53823` 2026-08-20).

**Root cause** — Injections re-rendered whole blocks on every change instead of deltas, and the budgets applied per source rather than globally.

**Fix · [[codex]]**
- `1bbfb5cfad` 2026-08-03 (#36800), "Avoid reinjecting permissions after command approvals": emit only the newly approved prefixes.
- `85e0661c3b` 2026-08-07 (#37424), "Cap project instructions across environments": a global cap.
- `663da53823` 2026-08-20 (#39641), "Sanitize developer context in full-history agent forks": filter developer content per item when forking.
- The general mechanism is typed world-state sections that emit nothing when unchanged and replace/remove notices otherwise ([[world-state-diff-injection]]).

**Lesson** — Context injections need delta semantics and global budgets. Otherwise they grow per event and per environment, bloating the context and churning the cached prefix.

Related: [[cache-stable-prompt-prefix]] · [[context-file-hierarchy]] · [[transcript-carried-system-prompt]] · [[world-state-diff-injection]] · [[stale-context-files-mid-session]] · [[codex--cache-stable-prompt-prefix|codex]]
