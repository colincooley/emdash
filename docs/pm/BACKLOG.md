# Backlog

Maintained by the PM task only. Process, status values and templates are in `PROCESS.md`.
Acceptance criteria for ORCH-01 to ORCH-24 are in `docs/orchestrator/backlog.md`; briefs in
`tasks/` are the paste-ready prompts.

## Board

Slots: **2 / 5 used** (PM, ORCH-25).

| Running | Agent · model | Branch | Started |
| --- | --- | --- | --- |
| PM | Claude Code · Opus 5.5 | `emdash/project-manager-1wp2u` | 2026-10-21 |
| ORCH-25 | Claude Code · Opus 5.5 | `emdash/25-m0ps4` | 2026-10-21 |

In review (merge on GitHub): ORCH-26, ORCH-27, ORCH-28 report PRs.

**Ready now** (launch all three → 5 / 5). Paste the whole prompt file into a new Emdash task.
Merge the ORCH-26 PR before launching ORCH-31; it starts from `main` and reads that report.

```text
ORCH-01 — Orchestrator Mode setting (off by default)
Prompt: docs/pm/tasks/ORCH-01.md
Agent:  Codex
Model:  GPT-6.1 Sol
Effort: medium
```

```text
ORCH-02 — Orchestrator durable store and state machine
Prompt: docs/pm/tasks/ORCH-02.md
Agent:  Codex
Model:  GPT-6.1 Sol
Effort: high
```

```text
ORCH-31 — Codex MCP isolation and Windows sandbox boundary
Prompt: docs/pm/tasks/ORCH-31.md
Agent:  Codex
Model:  GPT-6.1 Sol
Effort: high
```

No ownership overlap: ORCH-01 owns `features/orchestrator/contributions/`, the enabled accessor,
the settings manifest and one settings page; ORCH-02 owns `features/orchestrator/node/store/` and
one desktop `package.json` script line; ORCH-31 owns only its report.

**Queued (in launch order):**

1. ORCH-06 Session Policy: after ORCH-25 reports (whether the Claude ACP adapter forwards
   `strictMcpConfig` and env) and ORCH-27 is merged
2. ORCH-03 session-scoped MCP: after ORCH-25 and ORCH-31
3. ORCH-04, ORCH-05

**Human:**

- Merge the ORCH-26, ORCH-27 and ORCH-28 report PRs.
- Add a branch ruleset on the fork's `main` (Settings → Rules → Rulesets): target
  `refs/heads/main`, **Require a pull request before merging** with 0 required approvals (you're
  the only collaborator and GitHub won't let you approve your own PR), **Block force pushes**,
  **Restrict deletions**, no bypass list. This stops direct pushes to `main`; see Q-4 for what it
  can't stop.
- Answer Q-3 and Q-4 in `DECISIONS.md`.

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
| ORCH-25 | Claude ACP: per-session MCP (HTTP/stdio), config option ids, Windows loopback | P0 | Running | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-25.md` |
| ORCH-26 | Codex ACP: per-session MCP (HTTP/stdio), config option ids, Windows sandbox | P0 | In review | — | Codex · GPT-6.1 Sol (high) | `docs/pm/reports/ORCH-26.md` |
| ORCH-27 | Subscription-only auth, billing env scrub list, claude.ai connector switch, usage probes | P0 | In review | — | Claude Code · Opus 5.5 (high) | `docs/pm/reports/ORCH-27.md` |
| ORCH-28 | Branch-protection detection, task-branch push evidence | P1 | In review | — | Codex · GPT-6 Luna (medium) | `docs/pm/reports/ORCH-28.md` |
| ORCH-31 | Codex MCP isolation and Windows sandbox boundary (follow-up to ORCH-26) | P0 | Ready | ORCH-26 | Codex · GPT-6.1 Sol (high) | `docs/pm/reports/ORCH-31.md` |

### P1 — Foundations

| ID | Task | Pri | Status | Depends on | Agent · model (effort) | Owns (planned) |
| --- | --- | --- | --- | --- | --- | --- |
| ORCH-01 | Orchestrator Mode setting (off by default) | P0 | Ready | — | Codex · GPT-6.1 Sol (medium) | `features/orchestrator/contributions/`, `node/orchestrator-enabled.ts`, settings manifest (hotspot), one settings page |
| ORCH-02 | Orchestrator durable store and state machine | P0 | Ready | — | Codex · GPT-6.1 Sol (high) | `features/orchestrator/node/store/`, `drizzle-orchestrator.config.ts`, one desktop `package.json` script line (hotspot) |
| ORCH-03 | ACP session-scoped MCP servers (T1) | P0 | Backlog | ORCH-25, ORCH-31 | Claude Code · Opus 5.5 (xhigh) | `packages/core/src/runtimes/acp/api/schemas.ts`, `…/node/runtime/session-materializer.ts`, tests |
| ORCH-04 | Conversation target extraction + `decorateAcpInput` hook (T2) | P0 | Backlog | — | Claude Code · Opus 5.5 (high) | `features/conversations/node/wire-controller.ts` (+ new extracted file) |
| ORCH-05 | Session census accessor (T3) | P1 | Backlog | — | Codex · GPT-6.1 Sol (medium) | `src/main/core/acp/agent-status-bridge.ts`, its test |
| ORCH-06 | Session Policy (env scrub, preflight, strict MCP, refused modes) | P0 | Backlog | ORCH-25, ORCH-27 | Claude Code · Opus 5.5 (high) | `features/orchestrator/node/session-policy*` |

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
