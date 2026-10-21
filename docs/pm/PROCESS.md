# Process

How work on Orchestrator Mode (this Emdash fork) is planned, run, and reviewed. The PM task
coordinates; implementation happens in separate Emdash tasks that the owner launches by hand. This
is the same process used for the Toll project, adapted to this repo. Orchestrator Mode itself is
meant to automate this process later (see `docs/orchestrator/architecture.md` §11).

## Roles

| Role | Does | Doesn't |
| --- | --- | --- |
| Owner (Colin) | Approves plans and decisions, launches Emdash tasks, reviews and merges PRs on GitHub | — |
| PM task | Keeps `ROADMAP.md`, `BACKLOG.md`, `DECISIONS.md` and task briefs current; reviews reports; recommends what to launch next, with agent, model and effort | Implement features, merge, or discard work |
| Implementation task | Does one task brief on its own branch, writes its report | Edit PM docs (other than its own report), touch files outside its brief, merge |

## Concurrency

- **Hard maximum: 5 Emdash tasks running at once, including the PM task.** That leaves
  **4 implementation slots**. No exceptions.
- Human-only tasks don't use a slot.
- Work beyond the limit waits in the **Queued** list in `BACKLOG.md`, in launch order.
- Two running tasks never own overlapping files (see Ownership).

**Before recommending a launch, the PM checks:**

1. Running tasks + this one ≤ 5 (counting the PM).
2. The task ID is unique in `BACKLOG.md` and has never been used before.
3. Its owned paths don't overlap any running task's owned paths, or any hotspot another running
   task holds.
4. All its dependencies are merged to `main`, and no open question it depends on is unanswered.

## Decisions and open questions

The owner makes product decisions. When a brief or the design leaves something open:

- Implementation agents build the smallest neutral version (or leave a clearly named
  placeholder), and list the question under "Blockers and open questions" in their report.
- The PM adds it to Open questions in `DECISIONS.md` and marks only the tasks that truly depend on
  it as blocked. Unrelated work keeps going.

## Ownership and hotspots

Every brief lists the paths it **owns** (may create or edit) and paths it may only **read**. The
PM checks that running tasks don't overlap before marking a task Ready.

Single-owner hotspots (only one running task may touch each):

- `pnpm-lock.yaml` and any `package.json`
- `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `README.md`
- `apps/emdash-desktop/src/core/manifests/**`
- `apps/emdash-desktop/src/main/bootstrap/boot/phases/services.ts`
- `apps/emdash-desktop/src/core/features/conversations/node/wire-controller.ts`
- `packages/core/src/runtimes/acp/**` (high-risk area)
- `apps/emdash-desktop/src/core/services/app-db/**` and `drizzle/` (orchestrator work must not
  change these at all; see D-004)

If a task finds it needs a file it doesn't own, it stops and reports a blocker instead of editing
it.

## Branches and merging

- Each task works on its own Emdash branch, started from the current `main` of the fork
  (`origin`, `colincooley/emdash`).
- Before review, the branch is rebased on (or merged with) the latest `main` and passes its checks.
- Tasks push their own branch and may open a PR against the fork's `main`. **The owner merges on
  GitHub.** Nothing is merged, force-pushed, or deleted by an agent.
- A task's dependencies must be **merged to `main`** before it's launched, not just finished.
- Never open PRs against `upstream` (`generalaction/emdash`) unless the owner asks.

## Verification

Every code task must pass, and quote the result of, the checks its brief lists. The default set
from the repo root is:

```bash
pnpm install --frozen-lockfile
pnpm run format
pnpm run lint
pnpm run typecheck
pnpm run test
```

Run focused checks while iterating; run the full set before review. Spike tasks (Phase 0) list
their own checks instead.

Safety rules for every task:

- Never change `pnpm-lock.yaml` unless the brief owns it.
- Never run the Emdash app against the default database; use `EMDASH_DB_FILE` pointing at a
  scratch file.
- Never read or modify other projects (including Toll) or their Emdash data.
- Never set real `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` values; this project uses subscription
  logins only.
- Never print, log, or commit tokens, credentials, or the contents of `~/.claude` / `~/.codex`
  auth files.

## Choosing an agent

| Pick | When the task is |
| --- | --- |
| **Claude Code** | Architecture and high-risk areas (ACP runtime, PTY, SSH, session policy); integration across slices and processes; renderer UI; anything where the design needs interpreting; investigation spikes with open-ended findings |
| **Codex** | Tightly specified, self-contained logic with clear tests (stores, schedulers, governors, schemas, contracts); mechanical refactors; scripted checks with a precise expected output |
| **Human** | Account/plan settings, GitHub repository settings, manual Windows validation, product decisions |

Both run on the existing Claude and ChatGPT subscriptions. No API keys, paid APIs, or new services.

### Choosing a model

Each task in `BACKLOG.md` lists a suggested model and reasoning effort, e.g.
`Claude Code · Opus 5.5 (high)`. Pick both in Emdash when launching. Lineups as of 2026-10-21
(Claude Code 2.1.306, codex-cli 0.163.0):

| Agent | Model | Use for |
| --- | --- | --- |
| Claude Code | **Opus 5.5** (default) | Most Claude tasks. `high` effort; `xhigh` for high-risk or foundation tasks other work builds on |
| Claude Code | Sonnet 5.5 | Small, well-scoped follow-ups (review fixes, doc tweaks). `medium`–`high` effort |
| Claude Code | Fable 5.1 | Optional upgrade if an Opus run stalls on a hard problem. Uses subscription limits much faster; don't use by default |
| Codex | **GPT-6.1 Sol** (default) | Most Codex tasks. Set reasoning to `medium` (Sol defaults to `low`); `high` for logic with tricky invariants |
| Codex | GPT-6 Astra | The hardest Codex tasks. Capped allowance on Plus; use sparingly |
| Codex | GPT-6 Luna | Small, focused changes and scripted checks. `medium` effort |

To see what the Codex account offers, run `codex debug models` (entries with
`"visibility": "list"` are selectable). The PM re-checks it whenever a wave is prepared.

## Task status values

`Backlog` → `Ready` (brief written, dependencies merged) → `Queued` (waiting for a slot) →
`Running` → `In review` → `Done` (merged) — or `Changes requested` / `Blocked`.

## Task brief template

Briefs live in `docs/pm/tasks/ORCH-xx.md`. Paste the whole file into Emdash as the task prompt.

```markdown
# ORCH-xx — <title>

**Suggested agent:** Claude Code | Codex · **Model:** <model> (effort: <level>)
**Phase / priority:** P? / P?
**Depends on (merged):** ORCH-...

## Context
Why this task exists; relevant design sections; relevant decisions (D-...).

## Goal
One or two sentences.

## Scope
- In: ...
- Out (do not do): ...

## Ownership
- Owns (may create/edit): ...
- Read only: ...
- Must not touch: ...

## Deliverables
...

## Acceptance criteria
- [ ] ...

## Report
Write `docs/pm/reports/ORCH-xx.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, and stop. Don't merge.
```

## Completion report template

Every implementation task writes `docs/pm/reports/ORCH-xx.md`:

```markdown
# ORCH-xx report — <title>

**Branch:** <branch>
**Status:** Done | Partial | Blocked

## Summary
What was done, in a few sentences.

## Changed files
- `path` — why

## Verification
Commands run, exit codes, and relevant output (paste errors/warnings verbatim).

## Tests
Tests added or changed, and their results.

## How to check it by hand
Steps for the owner to see it working.

## Deviations from the brief
Anything done differently, and why.

## Blockers and open questions
...

## Suggested follow-ups
...
```
