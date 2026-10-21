# ORCH-28 report — GitHub branch-protection detection and task-branch push

**Branch:** `emdash/28-l32rb`
**Status:** Done

## Summary

Queried the three requested GitHub REST endpoints for `colincooley/emdash` `main` on 2026-10-21. The branch is not protected: `protected` is `false`, the effective branch rules response is empty, and classic branch protection returned 404 `Branch not protected`. A task preflight can reliably check for the active `pull_request` rule with at least one required approval through the effective-rules endpoint; the branch `protected` field is a useful preliminary signal but does not identify which rule is active.

The task branch push uses HTTPS and Git Credential Manager (`manager`). No app code or repository settings were changed. The report is the only changed file.

## Changed files

- `docs/pm/reports/ORCH-28.md` — records the API results, detection recommendation, codebase seams, and push evidence requested by the brief.

## Verification

Read-only GitHub API calls, run with `gh api --include` while authenticated as `colincooley`:

| Endpoint | Status | Relevant fields | Usable for preflight? |
| --- | --- | --- | --- |
| `GET /repos/colincooley/emdash/branches/main` | 200 | `name: main`; `protected: false`; `protection.enabled: false`; status checks enforcement `off` | Partly. `protected: false` establishes no active branch protection/ruleset rules apply. `true` only says some protection applies; it does not prove reviewed PRs are required. |
| `GET /repos/colincooley/emdash/rules/branches/main` | 200 | `[]` | Yes. Returns the active rules that apply to the named branch, across repository and organization rulesets. Inspect for `type: pull_request` and `parameters.required_approving_review_count >= 1`. |
| `GET /repos/colincooley/emdash/branches/main/protection` | 404 | `{"message":"Branch not protected"}` | Yes for classic protection details when the caller can read them; a successful response exposes `required_pull_request_reviews`, review count and restrictions. It requires repository Administration read permission for fine-grained tokens, so it is not a dependable non-admin check. Here the branch endpoint and rules endpoint independently confirm that the 404 means unprotected. |

GitHub documents `protected` as covering either branch protections or rulesets, and documents the rules-for-branch endpoint as returning all active rules applicable at repository or organization level. The latter requires only Metadata read on fine-grained tokens, and can be used unauthenticated for public repositories. The classic protection endpoint requires Administration read on fine-grained tokens. See [Get a branch](https://docs.github.com/en/rest/branches/branches), [Get rules for a branch](https://docs.github.com/en/rest/repos/rules), and [Get branch protection](https://docs.github.com/en/rest/branches/branch-protection).

The authenticated identity was `colincooley`; this check did not simulate a separate non-admin collaborator. The rules endpoint's documentation says a public repository can be queried without authentication; for private repositories, a read-capable collaborator needs the documented metadata permission. Errors, permission failures, and malformed or incomplete responses should map to `unknown`, never to `unprotected`.

Current state: `main` has no active protection or applicable rules requiring a reviewed PR. The branch head observed by the endpoint was `d50bdf9b455fd815b233429120282947e4e8b86f`.

Recommended owner-managed setting: an active branch ruleset matching only `refs/heads/main`, with **Require a pull request before merging**, at least one required approval, and no bypass actors. Require review resolution before merge as a useful companion. Keep force-push and deletion disallowed. If the intended threat model includes a worker running with the owner's repository-admin credentials, GitHub branch rules are not an immutable guard: that identity can change the ruleset or its bypass configuration. Protecting against that stronger threat requires workers to use an identity that cannot administer or bypass the ruleset.

## Proposed preflight rule

Inputs: the repository's canonical GitHub host/owner/repository and its resolved default branch (do not assume the name is `main`). Reuse the repository/account context already resolved for the project. Call `GET /repos/{owner}/{repo}/branches/{branch}` to confirm branch existence and obtain `protected`, then `GET /repos/{owner}/{repo}/rules/branches/{branch}` to inspect effective active rules. A `protected: false` result can short-circuit to `unprotected`; for `true`, the effective rules distinguish a reviewed-PR requirement from unrelated protection. Treat 404 for the rules endpoint, 403/5xx, network errors, malformed payloads, or an unsupported provider as `unknown`; classic-protection 404 alone is ambiguous outside the corroborated case above.

| Result | Condition | Suggested preflight message |
| --- | --- | --- |
| `protected` | An active `pull_request` rule applies and requires at least one approval; no configured bypass actor defeats the intended protection. | “Default branch requires an approved pull request before merge.” |
| `unprotected` | The branch exists and `protected` is false, or a complete effective-rules response contains no qualifying pull-request rule. | “Default branch does not require a reviewed pull request. An agent with Git credentials may be able to merge directly; ask the repository owner to enable protection.” |
| `unknown` | Any check cannot establish the effective rules (authorization, network, API, parsing, or provider limitation). | “Could not verify default-branch protection. Check repository access or settings before starting this run.” |

The result should report required approval count and whether bypass is configured, so the user can distinguish a one-approval rule from a rule with bypass. A `protected` result should not be interpreted as protection against a worker that can edit repository settings or act as an administrator.

## Where the check should live and what it reuses

Keep the check in the Orchestrator main-process preflight (`apps/emdash-desktop/src/core/features/orchestrator/node/`), alongside the design's other run-start checks. It is a safety gate for orchestration, not a generic GitHub feature needed by all task workflows.

Reuse the shared project integration account resolution (`ProjectIntegrationAccountResolver` / `resolveProjectAccount`) to select the account for the project's effective GitHub repository context, then read that exact account's typed credentials with `createGitHubCredentialReader`. Construct the request client using the stored `apiBaseUrl`; do not choose an account by login or silently fall back to another account. The existing `getOctokit` helper in `apps/emdash-desktop/src/core/features/github/node/services/octokit-provider.ts` shows the same explicit account ID and host pattern, though it is currently repository-service oriented. An isolated read-only Octokit or REST request helper can be extracted/reused if needed.

The pull-request integration already selects and routes per repository: `apps/emdash-desktop/src/core/services/pull-requests/node/engine/pull-request-engine.ts` resolves the integration and credentials through `GitPlatformAuthContract`; `packages/plugins/src/pull-requests/impl/github/client.ts` consumes those credentials. The Git credential service at `apps/emdash-desktop/src/core/features/github/api/node/services/git-credentials-service.ts` separately provides the task/terminal HTTPS credential channel. A protection preflight should reuse account selection and credential reading, not rely on the worker's shell `gh` executable or on the PR worker's long-lived client.

## Tests

No tests were added or run; this is a Phase 0 investigation spike with no product-code changes.

## How to check it by hand

Run the three `gh api` commands from the brief while authenticated to the fork. On the date of this report they returned, in order: HTTP 200 with `protected: false`; HTTP 200 with `[]`; and HTTP 404 with `Branch not protected`.

## Deviations from the brief

None. The branch endpoint and effective rules endpoint were both used to disambiguate the classic-protection 404. The report adds a threat-model limitation: protection does not constrain an agent that can administer the same repository settings.

## Blockers and open questions

- The documented branch-rules endpoint is sufficient for a read-only detection of the active `pull_request` rule. It does not return ruleset bypass actors. If preflight must validate bypass actors as well, enumerate applicable rulesets and inspect each ruleset; the API only returns `bypass_actors` to users with write access to that ruleset. For an account without that visibility, report `unknown` for bypass assurance.
- The API check used the owner's account, not a separate non-admin account. Public-repository access is documented as unauthenticated for the effective-rules endpoint, but the current repo is public; private-repository collaborator behavior was not independently exercised.

## Suggested follow-ups

- Owner: enable and verify an active reviewed-PR rule for `main`.
- Design: decide whether Orchestrator Mode supports a trusted local owner/admin account as its worker identity, given that such an identity may be able to change or bypass branch protection.

## Push evidence

Credential helper and remote URL captured before pushing:

```text
file:C:/Program Files/Git/etc/gitconfig manager
https://github.com/colincooley/emdash.git
```

The remote uses HTTPS. Git Credential Manager (`manager`) is the configured Git credential helper. `gh auth status` showed the GitHub CLI logged into `colincooley` with token contents masked; no token or credential data is included here. The task branch push output is recorded after the commit below.

Final `git status --short --branch` is recorded after commit and push; the report is the only repository file changed.
