---
type: tradeoff
concepts: [shell-execution, process-tree-kill, abort-propagation]
harnesses: [pi, opencode]
---
# bash-timeout-default-vs-none

**Axis**: does a shell command get a default timeout, and is the model allowed to raise it without limit?

| dimension | pi | opencode legacy | opencode v2 | evidence |
|---|---|---|---|---|
| Default | none: "commands run until completion unless specified" | 2 min (env-overridable) | 2 min | pi `packages/coding-agent/src/core/tools/bash.ts:42`, `29900ce64` 2025-11-12 → [[no-bash-default-timeout]]; opencode `packages/opencode/src/tool/shell.ts:347`; `packages/core/src/tool/bash.ts:19` |
| Max a model may request | int32 ms (validated) | none (10-min max dropped in `75a4dcbce8`) | 10 min (schema) | pi `packages/coding-agent/src/core/tools/bash.ts:22`; v2 `packages/core/src/tool/bash.ts:20,28` |
| History | 30 s → none (`29900ce64`) | 1 min (`f3da73553c`) → 2 min, max removed (`75a4dcbce8` 2025-12-06) | max re-added (`76ee87ead8` 2026-06-03) | [[Constants]] 03-tools |
| On expiry | kill process tree, return partial output + "Command timed out" | kill (3 s grace), hint to retry with a larger timeout | kill (3 s grace) | opencode `packages/opencode/src/tool/shell.ts:550-563` → [[process-tree-kill]] |
| Background escape hatch | none (tmux) | none for bash | removed | → [[no-background-bash]] |

**When each wins**
- **No default (pi)**: builds and test suites that legitimately run long never die at an arbitrary mark; the human's Esc is the timeout. Cost: `npm run dev` hangs the turn until someone notices; unattended runs need their own budget.
- **Default 2 min (opencode)**: unattended and subagent runs can't wedge on a watcher; the timeout message teaches the model to ask for more. Cost: false kills of slow builds, and the legacy "no max" means the model can still pick hours.

Related: [[shell-execution]] · [[turn-cap-vs-none]] · [[no-bash-default-timeout]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
