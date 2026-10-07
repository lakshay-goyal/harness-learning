---
type: concept
stage: architecture
tier: candidate
aliases: ["opencode github install", "opencode github run", "/oc", "/opencode mention", "anomalyco/opencode/github", exchange_github_app_token, use_github_token, opencode-agent GitHub App]
harnesses: [opencode]
---
Run the harness inside CI on repository events. It checks the actor and posts results back.

## Why
- Lets maintainers delegate issue triage, fixes and PR edits by comment, without a local session.
- CI runs with repository write tokens: who may trigger the agent must be checked, or any commenter on a public repo drives a write-capable agent.
- The run is headless: every interactive surface (questions, approvals) must be denied or answered, or the job hangs to its timeout ([[approval-wait-without-responder]]).

## Design space
- **Trigger**: comment mention (`/oc`, `/opencode`) on issues and PR review comments, plus issues, PRs, schedule, workflow_dispatch (opencode generated workflow).
- **Actor gate**: collaborator permission `admin`/`write` for user events; none for repo events (schedule, dispatch) (opencode).
- **Credentials**: OIDC token exchanged at the vendor for a GitHub App installation token, revoked at the end (opencode default) vs the workflow's `GITHUB_TOKEN` (opencode `use_github_token`).
- **Output**: branch + commit + PR for issues; push to the PR branch for PRs; comment with a footer (opencode).
- **Transcript visibility**: share link by default only on public repos (opencode).
- **Interactive tools**: `question` denied on the session (opencode); permission asks have no responder (opencode, observed gap).

## Implementations
- [[opencode--ci-agent-integration|opencode]] — `github/action.yml` installs the CLI and runs `opencode github run` (`packages/opencode/src/cli/cmd/github.handler.ts`); `opencode github install` writes the workflow.

## Failures
- [[approval-wait-without-responder]]

## Related
[[headless-rpc-mode]] · [[permission-ruleset]] · [[session-export-share]] · [[secret-handling]] · [[agent-profiles]] · [[opencode]]
