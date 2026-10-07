---
type: concept
stage: context
tier: candidate
aliases: [PI_MODEL, PI_SESSION_ID, PI_PROVIDER, PI_SESSION_FILE, PI_REASONING_LEVEL, exposeSessionEnvironment, PI_CODING_AGENT]
harnesses: [pi]
---
Expose volatile session facts (model, provider, session id/file, reasoning level) as environment variables of tool subprocesses instead of prompt text; the prompt only says they exist.

## Why
- Volatile facts in the prompt bust the cache prefix every time they change ([[volatile-system-prompt-prefix]]).
- Scripts the agent writes (and child processes) need machine-readable session identity anyway.
- An imperative pointer to them gets over-executed ([[imperative-guideline-over-compliance]]).

## Design space
- Facts in system prompt (rejected for cache; date removed).
- Facts as a dedicated tool (not in pi).
- **Env vars set fresh per command, inherited values scrubbed, pointer guideline only when enabled** (**pi chose**).
- Facts in a per-turn user/system delta (possible via section patches; not used for these).

## Implementations
- [[pi--env-vars-as-context|pi]] — 5 `PI_*` vars per bash/PowerShell command; `exposeSessionEnvironment` default true.

## Failures
- [[imperative-guideline-over-compliance]]

## Related
[[cache-stable-prompt-prefix]] · [[guideline-softening]] · [[shell-execution]] · [[no-date-in-prompt]]
