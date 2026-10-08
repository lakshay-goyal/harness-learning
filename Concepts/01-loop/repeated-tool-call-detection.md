---
type: concept
stage: loop
tier: variant
aliases: [repeated-tool-call-guard, repeated-call-detection, repetition-loop-detection, doom loop, doom_loop, DOOM_LOOP_THRESHOLD, "Possible doom loop"]
harnesses: [opencode]
---
Detect the model re-issuing the same tool call with identical input N times in a row, then interrupt it by asking the user or stopping, so it cannot loop.

## Why
- Models get stuck re-running the same failing call; without a guard the loop runs until a step cap, abort or budget ([[identical-tool-call-loop]]).
- Ambiguous tool errors make repeats more likely ([[ambiguous-tool-error-causes-retry-loop]]); the guard is the mechanical backstop to prompt rules.

## Design space
- **Equality**: exact serialized input (`JSON.stringify`, opencode) · normalized / semantic similarity · same tool regardless of input.
- **Window**: last N parts of one assistant message (opencode, N = 3) · last N calls across steps · sliding count per run.
- **Action**: ask the user via the permission channel, "always" per tool (opencode) · hard stop · inject a warning to the model · deny.
- **Configurability**: rule in the permission ruleset (`doom_loop: ask|allow|deny`, opencode) · fixed.
- **Absent**: pi (no guard, no cap: [[no-turn-cap]]); opencode v2 runner (checklist item open).

## Implementations
- [[opencode--repeated-tool-call-detection|opencode]] — legacy `DOOM_LOOP_THRESHOLD = 3` identical calls → `permission.ask("doom_loop")`, default `ask`; not in v2.

## Failures
- [[identical-tool-call-loop]]

## Tradeoffs
- [[turn-cap-vs-none]]

## Related
[[step-budget-limit]] · [[turn-loop]] · [[permission-ruleset]] · [[no-turn-cap]] · [[tool-error-as-result]]
