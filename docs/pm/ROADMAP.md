# Roadmap

Phases come from `docs/orchestrator/implementation-plan.md`. Task details are in `BACKLOG.md`.

## P0 — Spikes

Retire the unknowns that could invalidate the design: per-session MCP over ACP for Claude and
Codex, subscription auth with billing env vars removed, claude.ai connectors, native permission /
model / effort option ids, Windows loopback, branch protection detection.

**Done when:** ORCH-25 to ORCH-28 reports are merged and the PM has folded the findings into
`docs/orchestrator/risks.md` with a go / no-go per item.

## P1 — Foundations (flagged, invisible)

Feature flag, orchestrator store, ACP session-scoped MCP servers, conversation target extraction,
session census, session policy (ORCH-01 to ORCH-06).

**Done when:** all merged; with the flag off, a normal task's ACP start input is byte-identical to
before.

## P2 — Headless workers

Governor, launcher, monitor, scheduler with ownership checks, Wire contract (ORCH-07 to ORCH-11,
ORCH-30).

**Done when:** a human-defined 3-node plan runs as ordinary tasks with at most 5 live sessions.

## P3 — MCP gateway and PM

Gateway, tools, PM supervisor, plan review, next-up list (ORCH-12 to ORCH-15, ORCH-29).

## P4 — Persistence and handoffs

Handoff rotation, restart recovery, quota gate (ORCH-16 to ORCH-18).

## P5 — Decisions and review handoff

Inbox, per-node blocking, review handoff to GitHub (ORCH-19, ORCH-20).

## P6 — UI, Windows, docs

Orchestrator view, Windows validation, docs promotion (ORCH-22 to ORCH-24).
