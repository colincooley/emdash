# Backlog

Maintained by the PM task only. Process, status values and templates are in `PROCESS.md`.
Acceptance criteria for ORCH-01 to ORCH-24 are in `docs/orchestrator/backlog.md`; briefs in
`tasks/` are the paste-ready prompts.

## Board

Slots: **1 / 5 used** (PM).

| Running | Agent · model | Branch | Started |
| --- | --- | --- | --- |
| PM | Claude Code · Opus 5.5 | `emdash/project-manager-1wp2u` | 2026-10-21 |

**Before launching anything:** merge the PM PR for `emdash/project-manager-1wp2u`. Task branches
start from `main`, and the briefs point at `docs/pm/` and `docs/orchestrator/`, which only exist on
the PM branch until it's merged.

**Ready now (launch all four together → 5 / 5):**

| Task | Prompt (paste the whole file) | Agent | Model (effort) |
| --- | --- | --- | --- |
| ORCH-25 | `docs/pm/tasks/ORCH-25.md` — Claude ACP: per-session MCP, config options, Windows | Claude Code | Opus 5.5 (high) |
| ORCH-26 | `docs/pm/tasks/ORCH-26.md` — Codex ACP: per-session MCP, config options, Windows | Codex | GPT-6.1 Sol (high) |
| ORCH-27 | `docs/pm/tasks/ORCH-27.md` — Subscription-only auth, billing env vars, claude.ai connectors | Claude Code | Opus 5.5 (high) |
| ORCH-28 | `docs/pm/tasks/ORCH-28.md` — GitHub branch-protection detection, task-branch push | Codex | GPT-6 Luna (medium) |

No ownership overlap: each owns only its own report and a scratch folder outside the repo.

**Queued (launch order once Phase 0 reports are merged and the PM has folded them in):**

1. ORCH-02 (store) and ORCH-01 (flag) — no Phase 0 dependency on findings, but held so slots stay
   free for the spikes
2. ORCH-03 (after ORCH-25, ORCH-26)
3. ORCH-06 (after ORCH-27)
4. ORCH-04, ORCH-05

**Human:** after ORCH-28, enable the recommended branch protection on the fork's `main`.

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
| ORCH-25 | Claude ACP: per-session MCP (HTTP/stdio), config option ids, Windows loopback | P0 | Ready | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-25.md` |
| ORCH-26 | Codex ACP: per-session MCP (HTTP/stdio), config option ids, Windows sandbox | P0 | Ready | — | Codex · GPT-6.1 Sol (high) | `docs/pm/reports/ORCH-26.md` |
| ORCH-27 | Subscription-only auth, billing env scrub list, claude.ai connector switch, usage probes | P0 | Ready | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-27.md` |
| ORCH-28 | Branch-protection detection, task-branch push evidence | P1 | Ready | — | Codex · GPT-6 Luna (medium) | `docs/pm/reports/ORCH-28.md` |

### P1 — Foundations

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-01 | Feature flag and settings contribution | P0 | Backlog | — | Codex · GPT-6.1 Sol (medium) | `features/orchestrator/contributions/settings.ts`, its test |
| ORCH-02 | Orchestrator durable store and schemas | P0 | Backlog | — | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/store/` |
| ORCH-03 | ACP session-scoped MCP servers (T1) | P0 | Backlog | ORCH-25, ORCH-26 | Claude Code · Opus 5.5 (xhigh) | `packages/core/src/runtimes/acp/api/schemas.ts`, `…/node/runtime/session-materializer.ts`, tests |
| ORCH-04 | Conversation target extraction + `decorateAcpInput` hook (T2) | P0 | Backlog | — | Claude Code · Opus 5.5 (high) | `features/conversations/node/wire-controller.ts` (+ new extracted file) |
| ORCH-05 | Session census accessor (T3) | P1 | Backlog | — | Codex · GPT-6.1 Sol (medium) | `src/main/core/acp/agent-status-bridge.ts`, its test |
| ORCH-06 | Session Policy (env scrub, preflight, strict MCP, refused modes) | P0 | Backlog | ORCH-27 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/session-policy*` |

### P2 — Headless workers

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-07 | Execution Slot Governor | P0 | Backlog | ORCH-05 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/governor*` |
| ORCH-08 | Worker Launcher | P0 | Backlog | ORCH-02, ORCH-04, ORCH-06, ORCH-07 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/launcher*` |
| ORCH-09 | Completion Monitor | P0 | Backlog | ORCH-08 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/monitor*` |
| ORCH-10 | Scheduler with dependencies and caps | P0 | Backlog | ORCH-02, ORCH-07, Q-2 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/scheduler*` |
| ORCH-30 | Ownership and hotspot gate (no two running nodes own overlapping paths) | P1 | Backlog | ORCH-10 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/ownership*` |
| ORCH-11 | Orchestrator Wire contract and controller | P0 | Backlog | ORCH-02 | Codex · GPT-6.1 Sol (medium) | `features/orchestrator/api/`, `node/wire-controller.ts`, manifests (hotspot) |

### P3 — MCP gateway and PM

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-12 | MCP Gateway (loopback, tokens, Host checks, stdio shim) | P0 | Backlog | ORCH-03, ORCH-25 | Claude Code · Opus 5.5 (xhigh) | `features/orchestrator/node/mcp-gateway/` |
| ORCH-13 | PM and worker tools | P0 | Backlog | ORCH-12 | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/mcp-tools/` |
| ORCH-14 | PM Supervisor (PM task, digests) | P0 | Backlog | ORCH-09, ORCH-13 | Claude Code · Opus 5.5 (xhigh) | `features/orchestrator/node/pm-supervisor*` |
| ORCH-15 | Initial plan review and findings feed | P0 | Backlog | ORCH-14, Q-1 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/plan*`, browser plan view |
| ORCH-29 | "Next up" list: each node's prompt, agent, model, effort, owns; editable before dispatch | P0 | Backlog | ORCH-11, Q-1 | Claude Code · Opus 5.5 (high) | `features/orchestrator/browser/next-up*` |

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
