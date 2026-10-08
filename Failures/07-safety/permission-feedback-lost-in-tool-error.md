---
type: failure
concepts: [permission-ruleset, tool-error-as-result]
harnesses: [opencode]
---
**Symptom** — (Latent, inferred from code at `ecc4916b5a`; no issue or test found.) In the v2 runtime, when a user rejects a permission with a correction message, or a configured rule blocks the call, the model sees only a generic failure such as "Unable to execute command: <cmd>" — never the user's feedback or the fact that policy denied it.

**Root cause** — `PermissionV2` raises typed `CorrectedError{feedback}` and `BlockedError{rules}` (`packages/core/src/permission.ts:59-68`), but leaf tools wrap their whole body in `Effect.mapError(() => new ToolFailure({message: "Unable to …"}))` (`packages/core/src/tool/bash.ts:123-196`; same pattern in grep, write, edit). The legacy runtime had model-facing messages for both cases ("The user rejected permission to use this specific tool call with the following feedback: …", "The user has specified a rule which prevents you…", `packages/core/src/v1/permission.ts:7-26`).

**Fix · [[opencode]]** — none at HEAD `ecc4916b5a`.

**Lesson** — Permission outcomes are messages for the model: map decline-with-feedback and policy blocks to explicit text before any generic error wrapper.

Related: [[permission-ruleset]] · [[tool-error-as-result]] · [[declined-action-retried]] · [[opencode--permission-ruleset|opencode]]
