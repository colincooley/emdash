# Backlog

Maintained by the PM task only. Process, status values and templates are in `PROCESS.md`.
Acceptance criteria for ORCH-01 to ORCH-24 are in `docs/orchestrator/backlog.md`; briefs in
`tasks/` are the paste-ready prompts.

## Board

Slots: up to **4 / 5** once ORCH-01, ORCH-02 and ORCH-31 are running (PM + 3), so one slot is
free for ORCH-06.

| Running (if launched) | Agent · model |
| --- | --- |
| PM | Claude Code · Opus 5.5 |
| ORCH-01 | Codex · GPT-6.1 Sol (medium) |
| ORCH-02 | Codex · GPT-6.1 Sol (high) |
| ORCH-31 | Codex · GPT-6.1 Sol (high) |

In review: ORCH-25 report PR. Merge it and the PM docs PR before launching ORCH-06.

**Ready now** (fills the 5th slot):

```text
ORCH-06 — Session Policy (env scrub, auth preflight, strict MCP options, refused modes)
Prompt: docs/pm/tasks/ORCH-06.md
Agent:  Claude Code
Model:  Opus 5.5
Effort: high
```

If ORCH-01, ORCH-02 or ORCH-31 haven't been launched yet, they're still ready: briefs in
`tasks/`, agent and model in the task table below.

No ownership overlap: ORCH-06 owns only `features/orchestrator/node/session-policy/`.

**Queued (in launch order):**

1. ORCH-03 session-scoped MCP + provider `_meta` (D-024): after ORCH-31
2. ORCH-04, ORCH-05
3. ORCH-07 governor (after ORCH-05)

**Human:**

- Merge the ORCH-25 report PR and the PM docs PR.
- Add the `main` ruleset if not done yet (require a PR with 0 approvals, block force pushes,
  restrict deletions, no bypass list).
- Answer Q-3, Q-4 and Q-5 in `DECISIONS.md` (none blocks current work).

## Parallel and sequential work

- **Wave 1 (parallel):** ORCH-25, ORCH-26, ORCH-27, ORCH-28.
- **Wave 2 (parallel):** ORCH-01 (owns settings contribution only), ORCH-02 (new `orchestrator`
  slice `node/store/` only), ORCH-03 (ACP runtime hotspot), ORCH-06 (new slice
  `node/session-policy*` only). ORCH-01 and ORCH-02 both create the new slice folder; the PM will
  split ownership by subfolder in their briefs.
- **Gate:** ORCH-04 owns the conversations wire-controller hotspot; it can't run alongside any task
  touching that file.
- **Sequential chain:** ORCH-07 → ORCH-08 → ORCH-09 → ORCH-10; ORCH-12 → ORCH-13 → ORCH-14.

## Tasks

Priority: **P0** blocks the phase, **P1** needed for the phase, **P2** nice to have.

### P0 — Spikes

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-00 | Phase 0 spikes (split into ORCH-25 to ORCH-28, D-009) | — | Superseded | — | — | — |
| ORCH-25 | Claude ACP: per-session MCP (HTTP/stdio), config option ids, Windows loopback | P0 | In review | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-25.md` |
| ORCH-26 | Codex ACP: per-session MCP (HTTP/stdio), config option ids, Windows sandbox | P0 | Done | — | Codex · GPT-6.1 Sol (high) | `docs/pm/reports/ORCH-26.md` |
| ORCH-27 | Subscription-only auth, billing env scrub list, claude.ai connector switch, usage probes | P0 | Done | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-27.md` |
| ORCH-28 | Branch-protection detection, task-branch push evidence | P1 | Done | — | Codex · GPT-6 Luna (medium) | `docs/pm/reports/ORCH-28.md` |
| ORCH-31 | Codex MCP isolation and Windows sandbox boundary (follow-up to ORCH-26) | P0 | Ready | ORCH-26 | Codex · GPT-6.1 Sol (high) | `docs/pm/reports/ORCH-31.md` |

### P1 — Foundations

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-01 | Orchestrator Mode setting (off by default) | P0 | Ready | — | Codex · GPT-6.1 Sol (medium) | `features/orchestrator/contributions/`, `node/orchestrator-enabled.ts`, settings manifest (hotspot), one settings page |
| ORCH-02 | Orchestrator durable store and state machine | P0 | Ready | — | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/store/`, `drizzle-orchestrator.config.ts`, one desktop `package.json` script line (hotspot) |
| ORCH-03 | ACP session-scoped MCP servers + provider `_meta` (T1, D-024) | P0 | Backlog | ORCH-25, ORCH-31 | Claude Code · Opus 5.5 (xhigh) | `packages/core/src/runtimes/acp/api/schemas.ts`, `…/node/runtime/session-materializer.ts`, tests |
| ORCH-04 | Conversation target extraction + `decorateAcpInput` hook (T2) | P0 | Backlog | — | Claude Code · Opus 5.5 (high) | `features/conversations/node/wire-controller.ts` (+ new extracted file) |
| ORCH-05 | Session census accessor (T3) | P1 | Backlog | — | Codex · GPT-6.1 Sol (medium) | `src/main/core/acp/agent-status-bridge.ts`, its test |
| ORCH-06 | Session Policy (env scrub, preflight, strict MCP, refused modes) | P0 | Ready | ORCH-25, ORCH-27 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/session-policy/` |

### P2 — Headless workers

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-07 | Execution Slot Governor | P0 | Backlog | ORCH-05 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/governor*` |
| ORCH-08 | Worker Launcher | P0 | Backlog | ORCH-02, ORCH-04, ORCH-06, ORCH-07 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/launcher*` |
| ORCH-09 | Completion Monitor | P0 | Backlog | ORCH-08 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/monitor*` |
| ORCH-10 | Scheduler with dependencies and caps | P0 | Backlog | ORCH-02, ORCH-07 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/scheduler*` |
| ORCH-30 | Ownership and hotspot gate (no two running nodes own overlapping paths) | P1 | Backlog | ORCH-10 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/ownership*` |
| ORCH-11 | Orchestrator Wire contract and controller | P0 | Backlog | ORCH-02 | Codex · GPT-6.1 Sol (medium) | `features/orchestrator/api/`, `node/wire-controller.ts`, manifests (hotspot) |

### P3 — MCP gateway and PM

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-12 | MCP Gateway (loopback, tokens, Host checks, stdio shim) | P0 | Backlog | ORCH-03, ORCH-25 | Claude Code · Opus 5.5 (xhigh) | `features/orchestrator/node/mcp-gateway/` |
| ORCH-13 | PM and worker tools | P0 | Backlog | ORCH-12 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/mcp-tools/` |
| ORCH-14 | PM Supervisor (PM task, digests) | P0 | Backlog | ORCH-09, ORCH-13 | Claude Code · Opus 5.5 (xhigh) | `features/orchestrator/node/pm-supervisor*` |
| ORCH-15 | Initial plan review and findings feed | P0 | Backlog | ORCH-14 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/plan*`, browser plan view |
| ORCH-29 | "Next up" list: each node's prompt, agent, model, effort, owns; editable before dispatch | P0 | Backlog | ORCH-11 | Claude Code · Opus 5.5 (high) | `features/orchestrator/browser/next-up*` |

### P4 to P6

| ID | Task | Pri | Status | Depends on | Agent · model (effort) |
| --- | --- | --- | --- | --- | --- |
| ORCH-16 | Handoff rotation | P0 | Backlog | ORCH-14 | Claude Code · Opus 5.5 (high) |
| ORCH-17 | Restart recovery | P0 | Backlog | ORCH-09, ORCH-14 | Claude Code · Opus 5.5 (high) |
| ORCH-18 | Quota headroom gate | P1 | Backlog | ORCH-10, ORCH-27 | Codex · GPT-6.1 Sol (medium) |
| ORCH-19 | Inbox: findings, decisions, approvals; per-node blocking | P0 | Backlog | ORCH-13 | Claude Code · Opus 5.5 (high) |
| ORCH-20 | Review handoff to GitHub; branch-protection preflight | P1 | Backlog | ORCH-09, ORCH-28 | Codex · GPT-6.1 Sol (medium) |
| ORCH-22 | Orchestrator view (graph, census, timeline) and task badges | P1 | Backlog | ORCH-11, ORCH-29 | Claude Code · Opus 5.5 (high) |
| ORCH-23 | Windows validation pass | P0 | Backlog | all P1–P5 | Human + Claude Code · Sonnet 5.5 (high) |
| ORCH-24 | Promote docs into `agents/`; ADR 0009 Accepted | P1 | Backlog | ORCH-23 | Claude Code · Sonnet 5.5 (medium) |

ORCH-21 is retired (D-005). Task IDs are never reused.
