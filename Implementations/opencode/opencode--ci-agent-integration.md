---
type: implementation
harness: opencode
concept: ci-agent-integration
commit: ecc4916b5a
files: [github/action.yml:7-70, packages/opencode/src/cli/cmd/github.handler.ts:144-151, packages/opencode/src/cli/cmd/github.handler.ts:342-367, packages/opencode/src/cli/cmd/github.handler.ts:495-518, packages/opencode/src/cli/cmd/github.handler.ts:829-870, packages/opencode/src/cli/cmd/github.handler.ts:984-1010, packages/opencode/src/cli/cmd/github.handler.ts:1165-1184, packages/opencode/src/cli/cmd/github.handler.ts:1593-1597]
---
[[ci-agent-integration]] in [[opencode]].

## Mechanism
### Legacy runtime (`opencode github …`)
- **Install**: `opencode github install` writes `.github/workflows/opencode.yml`: triggers on `issue_comment`/`pull_request_review_comment` whose body contains ` /oc` or starts with `/oc` (and `/opencode`), with `permissions: id-token: write` (`packages/opencode/src/cli/cmd/github.handler.ts:342-367`); issues, PRs, schedule and `workflow_dispatch` are also supported (`1626341a4a`).
- **Action** `anomalyco/opencode/github` (`github/action.yml:7-70`): inputs `model`, `agent`, `share`, `prompt`, `use_github_token`, `mentions` (default `/opencode,/oc`), `variant`, `oidc_base_url`; installs via `curl -fsSL https://opencode.ai/install | bash` (cached by release tag) and runs `opencode github run`.
- **Token**: fetch the GitHub OIDC token, POST to `<oidc_base_url>/exchange_github_app_token` (or `…_with_pat`) for an opencode-agent App installation token (`packages/opencode/src/cli/cmd/github.handler.ts:984-1010`); configure git with it; `DELETE https://api.github.com/installation/token` at the end (`packages/opencode/src/cli/cmd/github.handler.ts:1593-1597`). `use_github_token: true` uses `GITHUB_TOKEN` instead.
- **Actor gate**: user events require collaborator permission `admin` or `write`, else "User X does not have write permissions" (`packages/opencode/src/cli/cmd/github.handler.ts:1165-1184`); the run adds an `eyes` reaction (`packages/opencode/src/cli/cmd/github.handler.ts:144`). Repo events (schedule, dispatch) skip the actor check (`packages/opencode/src/cli/cmd/github.handler.ts:495`).
- **Session**: created with `question: deny` only (`packages/opencode/src/cli/cmd/github.handler.ts:502-511`); shared when `share` is true, or unset and the repo is public (`packages/opencode/src/cli/cmd/github.handler.ts:515-518`) ([[session-export-share]]).
- **No permission responder**: the run subscribes to session events only to print tool and text parts (`packages/opencode/src/cli/cmd/github.handler.ts:829-870`); nothing answers `permission.asked`, so default `ask` rules (`external_directory`, `*.env` reads, `doom_loop`) would block until the job times out (observed from code; not reproduced) ([[approval-wait-without-responder]]).
- **Flows**: issue → branch `opencode/issue…`, run, if dirty summarize → commit → push → PR with "Closes #N" (`packages/opencode/src/cli/cmd/github.handler.ts:622`); PR → check out local or fork branch, push back, comment; schedule/dispatch → new branch + PR to default branch. If the agent switched branches itself, the infrastructure push is skipped. Comments carry a footer linking the share.

## Constants
| name | value | path:line |
|---|---|---|
| default mentions | `/opencode,/oc` | `packages/opencode/src/cli/cmd/github.handler.ts:747` |
| reaction | `eyes` | `packages/opencode/src/cli/cmd/github.handler.ts:144` |

## Evolution
- 2025-07-25 `3a7a2a838e` "wip: github actions".
- 2025-12-26 `1626341a4a` issues and `workflow_dispatch` events (#6157).
- 2026-06-03 `a3b97d9090` enforce existing git author identity (#30507); `github/index.ts` (older SDK-based implementation) last touched here.

## Quirks / drift
- `opencode run` has auto-reject and `--auto`; the GitHub path has neither ([[headless-rpc-mode]]).

pi contrast: no CI integration found in pi's notes.
