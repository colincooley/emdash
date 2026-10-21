# Orchestrator Mode — Phased Implementation Plan

> **Status: Design approved 2026-10-21 (decisions in [architecture](architecture.md#decisions-recorded-2026-10-21)).
> No implementation has started.**
> See [architecture](architecture.md), [risks](risks.md), [backlog](backlog.md).

Each phase ends in a mergeable, flag-off-safe state: with `orchestrator.enabled = false`, the app
must behave exactly as before. Every phase runs the local merge gate (`pnpm run format`, `lint`,
`typecheck`, `test`) before handoff. All testing uses `EMDASH_DB_FILE` scratch databases and
scratch repositories; no existing user project data is read or modified.

## Phase 0 — Spikes And Verification (no product code)

Goal: retire the unknowns that could invalidate the design.

- Verify that `claude-agent-acp` and `codex-acp` (versions pinned in the lockfile) accept
  `session/new.mcpServers` with `http` transport and expose the tools to the model; check
  `mcpCapabilities` advertised by each.
- Verify on Windows: loopback HTTP MCP and the stdio shim under `ELECTRON_RUN_AS_NODE`.
- Verify auth behavior: with `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` scrubbed, both CLIs use their
  subscription logins; confirm the auth-status probes distinguish login vs env-key auth.
- Identify the exact switch to disable claude.ai-hosted connectors for a Claude Code session.
- Identify native option ids for "edits allowed, commands ask" modes in both adapters.
- Confirm workers can push task branches with the project's normal credential mode, and how
  preflight can detect GitHub branch protection on the default branch.

Exit: a short spike report appended to `risks.md` with go/no-go per item. Throwaway code stays out
of `main`.

## Phase 1 — Foundations (flagged, invisible)

- ORCH-01 feature flag + settings contribution.
- ORCH-02 orchestrator durable store and schemas.
- ORCH-03 ACP session-scoped MCP servers (T1) — upstreamable commit.
- ORCH-04 conversation target extraction + `decorateAcpInput` hook (T2) — upstreamable commit.
- ORCH-05 session census accessor (T3).
- ORCH-06 Session Policy (env scrub, strict MCP, refused dangerous modes).

Exit: all foundations unit-tested; regression guard proves byte-identical ACP start input for
normal tasks; nothing user-visible.

## Phase 2 — Headless Worker Execution (no PM yet)

- ORCH-07 Execution Slot Governor.
- ORCH-08 Worker Launcher (createTask → provisionWorkspace → startSession).
- ORCH-09 Completion Monitor.
- ORCH-10 Scheduler with dependencies and caps.
- ORCH-11 Orchestrator Wire contract + controller (create run, add nodes manually, start, pause,
  cancel) — lets a human drive a DAG of workers before any PM exists.

Exit: a human can define 3 nodes (one depending on another) in a dev build, see them appear as
ordinary tasks with their own worktrees, observe at most N concurrent sessions, and see dependents
start after acceptance.

## Phase 3 — MCP Gateway And PM

- ORCH-12 MCP Gateway (transport, token scoping, Host checks, stdio shim).
- ORCH-13 PM tool set + worker `submit_result`.
- ORCH-14 PM Supervisor (launch PM task, event digests).
- ORCH-15 Initial plan review (drafts → queued) and non-blocking findings feed.

Exit: end-to-end run where the PM plans, the human reviews the plan, the PM dispatches, reads
results, and issues one follow-up — all on a scratch repo.

## Phase 4 — Persistence, Handoffs, Recovery

- ORCH-16 Handoff rotation (triggers, `write_handoff`, fresh generation).
- ORCH-17 Restart recovery (paused-on-boot, reconciliation).
- ORCH-18 Quota headroom gate.

Exit: a run survives an app restart and two PM rotations with no lost node state and no slot
overshoot.

## Phase 5 — Decisions, Approvals, And Review Handoff

- ORCH-19 Inbox: findings, decisions, approvals; per-node blocking; notifications.
- ORCH-20 Review handoff: accepted nodes move to `review` with branch/PR links; merged PRs observed
  through existing PR sync; branch-protection preflight warning.

Exit: an open decision blocks only its listed nodes while other nodes keep dispatching; every R7
category has a test proving it cannot happen through MCP; no code path in the slice performs a
merge.

## Phase 6 — UI Polish, Docs, Hardening

- ORCH-22 Orchestrator view (run graph, census, timeline) and task badges.
- ORCH-23 Windows validation pass (W1–W9) and long-path preflight.
- ORCH-24 Docs: promote this design into `agents/architecture/orchestrator.md`, update
  `AGENTS.md` pointers, mark ADR 0009 Accepted.

Exit: full merge gate green; manual Windows run; docs current.

## Later (not scheduled)

- Remote-host (SSH / workspace-server) orchestration.
- TUI workers via hook `stop` events and `sendInput`.
- Upstreaming T1–T3.

## Sequencing And Effort (rough)

| Phase | Depends on | Relative size |
| --- | --- | --- |
| 0 | — | S |
| 1 | 0 | M |
| 2 | 1 | L |
| 3 | 2 | L |
| 4 | 3 | M |
| 5 | 3 | M |
| 6 | 4, 5 | M |

Phases 4 and 5 can proceed in parallel once Phase 3 lands.
