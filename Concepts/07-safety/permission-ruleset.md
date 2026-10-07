---
type: concept
stage: permissions
tier: variant
aliases: [permission, PermissionV1, PermissionV2, Permission.evaluate, Permission.ask, "once/always/reject", "allow/ask/deny", last-match-wins, OPENCODE_PERMISSION, permission.asked, DeniedError, BlockedError, CorrectedError, "--auto", "--dangerously-skip-permissions", PermissionSaved]
harnesses: [opencode]
---
Ordered allow/ask/deny rules over (permission, wildcard pattern) pairs, evaluated last-match-wins and layered from defaults, agent, config and session. "ask" suspends the call until the user replies once, always or reject.

## Why
- Built-in approvals need a declarative policy that users can override per tool, per command prefix and per path; hard-coded refusals cannot be relaxed or tightened (opencode moved its `.env` block into rules, `3611260405`).
- One ruleset drives two things: whether a tool is advertised at all (`*` deny hides it) and whether a single call needs a human.
- Without it a harness either asks for everything or for nothing. pi chose nothing ([[no-permission-prompts]]).
- The ruleset is a UX layer, not containment: opencode says so explicitly ("it is not designed to provide security isolation", `SECURITY.md:17`). See [[tool-only-isolation]].

## Design space
- **Evaluation order**: last match wins across flattened layers (opencode legacy and v2) vs first match vs most-specific match.
- **Rule key**: (permission, pattern) with a per-tool notion of pattern: path for read/edit, command source for bash, subagent name for task, skill name for skill (opencode).
- **Default when no rule matches**: `ask` (opencode both runtimes) vs allow vs deny.
- **Shipping default**: `*: allow` plus targeted asks (`external_directory`, `*.env` reads, repeated-call detection) (opencode build agent) vs ask-by-default (Claude Code style).
- **Config representation**: object map (loses key order: [[permission-rule-order-lost-in-object-config]]) vs ordered array of layers (opencode since `65368f609d`; v2 spec `specs/v2/config.md`).
- **"always" scope**: in-memory per instance (opencode legacy) vs persisted per project (opencode v2 `PermissionSaved`). Whether a saved approval may override a configured deny: yes by construction (opencode legacy, inferred) vs never (opencode v2).
- **Reject semantics**: reject cascades to every pending ask in the session (opencode); reject stops the run vs continues as a tool error (opencode `experimental.continue_loop_on_deny`); reject-with-message continues with feedback to the model (opencode legacy) or is collapsed to a generic tool error (opencode v2, [[permission-feedback-lost-in-tool-error]]).
- **Headless**: auto-reject every ask (opencode `run`) vs auto-approve non-denied (`--auto`) vs no responder ([[approval-wait-without-responder]]).
- **Subagent inheritance**: child gets parent session denies only (opencode HEAD) vs parent agent denies too (opencode 2026-05, reverted). See [[read-only-mode-bypass-via-subagent]].
- **Separate non-interactive policy**: IAM-like allow/deny statements without `ask` for provider use (opencode v2 `Policy`).
- **Hook override**: plugin hook may rewrite the decision (opencode `permission.ask`, dead since `2fc06c5a17`: [[dead-hook-in-public-api]]).

## Implementations
- [[opencode--permission-ruleset|opencode]] — legacy `Permission.evaluate` = `findLast` over flattened rulesets, default ask, in-memory "always"; v2 `PermissionV2` checks configured denies first, then configured + project-saved approvals; agent rulesets merged defaults → agent → user config → `OPENCODE_PERMISSION`.

## Failures
- [[alternate-edit-tool-skips-edit-pipeline]]
- [[permission-rule-order-lost-in-object-config]]
- [[declined-action-retried]]
- [[permission-feedback-lost-in-tool-error]]
- [[approval-wait-without-responder]]
- [[secret-guard-bypassed-by-other-tools]]
- [[read-only-mode-bypass-via-subagent]]
- [[dead-hook-in-public-api]]
- [[error-message-leaks-filtered-catalog]]

## Tradeoffs
- [[permission-prompts-vs-none]]
- [[cwd-confinement-vs-none]]

## Related
[[tool-call-gate]] · [[shell-command-permission-parsing]] · [[workspace-boundary-check]] · [[plan-mode]] · [[agent-profiles]] · [[secret-handling]] · [[tool-only-isolation]] · [[headless-rpc-mode]] · [[repeated-tool-call-detection]] · [[no-permission-prompts]] · [[no-sandbox]] · [[opencode]]
