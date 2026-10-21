# ADR 0009: Optional Orchestrator Mode As A Desktop Slice Over Existing Task Services

## Status

Approved by the fork owner on 2026-10-21. Not implemented. Becomes Accepted when Phase 6 ships.

## Context

We want an optional mode where a Claude Code PM agent creates, dispatches, and supervises Claude
Code and Codex worker tasks through Emdash, with a global cap of five concurrent agent executions,
task dependencies, isolated worktrees, human approval gates, subscription-only authentication,
Windows support, and no regressions when the mode is off.

Emdash already has: desktop-owned task creation (`TaskService.createTask`), main-side workspace
activation (`provisionWorkspace`), ACP session start/prompt/terminate clients, structured session
summaries and agent-status hooks, a durable SQLite store primitive used by the automations
runtime, and per-session `mcpServers` in the ACP handshake (currently fed only from user-global
config).

## Decision

1. Implement orchestration as a **desktop feature slice** (`src/core/features/orchestrator/`) in
   the main process, not as a `packages/core` runtime, because tasks are desktop entities and the
   slice must call `TaskService` and conversation clients directly.
2. **PM and workers are ordinary tasks.** The orchestrator stores only orchestration metadata
   (nodes, edges, journal, approvals, PM generations) keyed by task/conversation ids.
3. Orchestration metadata lives in a **separate durable SQLite store**, not the app DB, to avoid
   app-DB migration conflicts with upstream.
4. The PM's tools are an **Emdash-hosted MCP server on loopback**, delivered per session through
   ACP `mcpServers` with scoped, in-memory capability tokens. No tool can merge, push, delete,
   change settings, resolve permissions, or decide approvals.
5. v1 workers and PM use **ACP** sessions; completion is proved from session summaries, never from
   timers.
6. The concurrency cap is a fixed 5 and counts all live agent sessions; the orchestrator never
   blocks sessions the user starts.
7. Orchestrated sessions get a **Session Policy**: billing env vars scrubbed, strict MCP set,
   dangerous modes refused.
8. Workers may push their own task branches and open PRs; **the user merges on GitHub**. Emdash has
   no merge path for orchestrated work.
9. The user reviews the initial plan; afterwards findings and decisions are surfaced as they arise,
   and an open decision blocks only the nodes that depend on it.
10. The mode is **off by default** behind `orchestrator.enabled`; shared-code changes are additive
   and inert unless orchestrator inputs are supplied.

## Consequences

- Small, additive changes to the ACP start input and conversations target resolution are required
  (high-risk area; kept as separate upstreamable commits).
- v1 is local-host only; remote orchestration needs the dispatcher and gateway placement revisited.
- Emdash cannot sandbox agents that run as the user; merge safety ultimately depends on
  remote branch protection. This is documented, not hidden.
- See `docs/orchestrator/` for architecture, risks, plan, and backlog.
