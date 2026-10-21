# ORCH-28 — Spike: GitHub branch-protection detection and task-branch push

**Suggested agent:** Codex · **Model:** GPT-6 Luna (reasoning: medium)
**Phase / priority:** P0 / P1
**Depends on (merged):** —

## Context

Read `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §5.8 ("Git credentials") and
§8.5, and `docs/orchestrator/risks.md` row S2 and the approval coverage matrix.

Decision D-005: workers may push their own task branches and open PRs; the owner merges on GitHub.
Because agents keep Git credentials, GitHub branch protection on `main` is the real guard against an
agent merging. Orchestrator Mode's preflight should check for it and warn when it's missing. We
need to know which GitHub API calls can tell us that without admin rights, and what this fork's
`main` currently has.

Repository: `colincooley/emdash` (remote `origin`). Upstream `generalaction/emdash` is read-only for
this task.

If `gh` isn't on PATH, look for it under
`$env:LOCALAPPDATA\Microsoft\WinGet\Packages\GitHub.cli_*\bin\gh.exe`.

## Goal

Know exactly how to detect "the default branch requires a reviewed PR to merge", and confirm that a
task branch push works with the project's normal Git credentials.

## Scope

- In:
  - Try, read-only: `gh api repos/colincooley/emdash/branches/main` (`protected` field),
    `gh api repos/colincooley/emdash/rules/branches/main` (rulesets), and
    `gh api repos/colincooley/emdash/branches/main/protection` (classic protection; may need admin).
    Record status codes and the fields that matter (required reviews, who can bypass, force-push,
    deletions). Explain which works for a non-admin collaborator.
  - Report the fork's current `main` protection state, and recommend the minimal settings the owner
    should enable (the owner does that by hand; don't change settings).
  - Find how Emdash already calls GitHub (look under `apps/emdash-desktop/src/core/features/github/`
    and `packages/plugins/src/` pull-request/repository plugins) and say where a preflight check
    should live and which client/account resolution it should reuse (read only).
  - Push your own branch (you must, to deliver the report) and record which credential path Git
    used: show `git config --show-origin --get-all credential.helper` and the remote URL form (no
    secrets).
- Out (do not do): merging anything; changing repository or branch-protection settings; pushing to
  any branch but your own; touching `upstream`; product code.

## Ownership

- Owns (may create/edit): `docs/pm/reports/ORCH-28.md`
- Read only: everything else
- Must not touch: any other repo file, repository settings, other branches

## Deliverables

1. Table: endpoint → status code as this user → relevant fields → usable for preflight (yes/no).
2. Current protection state of `colincooley/emdash` `main`, and recommended settings.
3. Proposed preflight rule (code-free): inputs, calls, result states (`protected`, `unprotected`,
   `unknown`), and the message shown for each.
4. Where in the codebase the check should live and what it reuses.
5. Push evidence: credential helper chain and push output (redacted).

## Acceptance criteria

- [ ] All deliverables answered with evidence.
- [ ] No setting changed, nothing merged, only your own branch pushed.
- [ ] No repo file changed except `docs/pm/reports/ORCH-28.md` (`git status` shown in the report).

## Report

Write `docs/pm/reports/ORCH-28.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, and stop. Don't merge.
