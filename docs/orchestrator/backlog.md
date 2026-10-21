# Orchestrator Mode — Initial Backlog

> **Status: Design approved 2026-10-21; implementation not started.** Item ids are referenced by the
> [implementation plan](implementation-plan.md). "Flag-off invariant" means: with
> `orchestrator.enabled = false` the change has no observable effect.

---

### ORCH-00 · Phase 0 spikes

Verify P1, P2, P5, P6, P8 and W1/W2 from [risks](risks.md).

- [ ] Report states, per adapter, whether HTTP and stdio session MCP servers work and expose tools.
- [ ] Report confirms subscription login is used when billing env vars are absent (Claude, Codex).
- [ ] Report names the switch that disables claude.ai connectors in a Claude Code session.
- [ ] Report lists native option ids for the default worker permission modes.
- [ ] Windows loopback + stdio shim verified on Windows 11.
- [ ] Report confirms workers can push task branches and describes branch-protection detection.

### ORCH-01 · Feature flag and settings

- [ ] `orchestrator.enabled` setting (default `false`) contributed via the slice's settings
      contribution; visible under an "Experimental" settings group.
- [ ] When false: no store file is created, no port is opened, view/commands hidden.
- [ ] Toggling off with active runs pauses them and closes the gateway; no tasks are deleted.
- [ ] Test asserts flag-off boot creates no `orchestrator.db` and opens no listener.

### ORCH-02 · Orchestrator durable store

- [ ] Store defined with `defineDurableSqliteStore` (backups, `assertSqliteStoreInvariants`).
- [ ] Tables per architecture §5.1; versioned JSON columns via `defineVersionedSchema()`.
- [ ] `EMDASH_ORCHESTRATOR_DB_FILE` override for tests.
- [ ] Node transition table is explicit and unit-tested; illegal transitions are rejected.
- [ ] **No** changes to `src/core/services/app-db/node/schema.ts` or `drizzle/`.

### ORCH-03 · ACP session-scoped MCP servers (T1)

- [ ] Optional `sessionMcpServers`, `mcpServerMode` and provider `sessionMeta` (D-024) on the ACP
      start input, re-sent on `session/load`; absent = today's behavior exactly.
- [ ] `mode: 'only'` excludes user-global MCP servers; `'merge'` appends.
- [ ] Headers/env of session MCP servers are never written to persisted intents (test reads the
      intent file after start).
- [ ] Respects `mcpCapabilities` (HTTP dropped when unsupported, with a logged reason).
- [ ] Protocol change documented as minor in `agents/architecture/acp-runtime.md`.

### ORCH-04 · Conversation target extraction (T2)

- [ ] `resolveConversationRuntimeTarget` callable from main services without the Wire controller.
- [ ] Optional `decorateAcpInput(target, input)` hook; default identity.
- [ ] Existing conversations controller tests pass unchanged; new test proves identical input for a
      normal task.

### ORCH-05 · Session census accessor (T3)

- [ ] Read-only accessor returning live (non-suspended) ACP sessions and running TUI agent
      sessions for the local host.
- [ ] Updates within one summary event of a session starting/ending; unit-tested with fake
      summaries.

### ORCH-06 · Session Policy

- [ ] Scrubs the billing-routing env vars listed in architecture §5.8 from orchestrated sessions
      only; normal sessions untouched (test).
- [ ] Auth preflight blocks a run when auth is env-key or unknown, with an actionable message.
- [ ] Orchestrated sessions use `mcpServerMode: 'only'` with the gateway + per-run allowlist.
- [ ] Dangerous provider modes are rejected at policy validation: Claude `bypassPermissions`, `auto`;
      Codex `agent-full-access`.
- [ ] Brief: `docs/pm/tasks/ORCH-06.md` (pure module; wiring happens in ORCH-03 and ORCH-08).

### ORCH-07 · Execution Slot Governor

- [ ] `census < cap` required to reserve; cap fixed at 5 (hard ceiling, not user-raisable).
- [ ] Lease taken before task creation; released on terminate/suspend observation or launch
      failure; generation-fenced.
- [ ] Manual sessions are counted, never blocked (default); test starts manual sessions at cap
      and asserts no dispatch and no manual-session failure.
- [ ] Property test: random event interleavings never produce census > cap from orchestrator
      dispatches.

### ORCH-08 · Worker Launcher

- [ ] Creates tasks through `TaskService.createTask` with `type: 'task'`; they appear in the normal
      task list and can be opened, renamed, archived, deleted like any task.
- [ ] Each worker gets its own new worktree; `baseFrom: node:<id>` branches from the upstream
      node's task branch.
- [ ] Provisions via `TaskService.provisionWorkspace` and starts the ACP session `fresh` without
      the renderer.
- [ ] Each failure step (`create`, `provision`, `start`) yields a node `failed` with step/code/message
      and releases the lease.

### ORCH-09 · Completion Monitor

- [ ] Uses `'agent:event'` as wake signal and the `SessionSummary` as proof; no timers decide
      outcomes.
- [ ] Implements the outcome table in architecture §5.5, including `needs-human` on pending
      permissions.
- [ ] User deleting/archiving a worker task cancels its node and blocks dependents.
- [ ] Stale-generation events are ignored (test).

### ORCH-10 · Scheduler

- [ ] Ready = queued + every upstream node's PR merged to the default branch (D-013); a per-run
      opt-in allows "accepted by the PM" (stacked branches). Cycles rejected on edge insert.
- [ ] Priority then FIFO; event-driven plus `Clock` backstop tick.
- [ ] Per-run caps (nodes, attempts, follow-ups, dispatch rate) move the run to `needs-human`.
- [ ] Manual-clock tests cover dependency chains, cap reached, and slot freed.

### ORCH-11 · Orchestrator Wire contract

- [ ] Contract in `features/orchestrator/api/`, controller in `node/`, registered through
      `desktop-wire-contract.ts` and `controllers.ts`.
- [ ] Live model for run state; procedures for create/pause/resume/cancel run and manual node
      editing.
- [ ] Plan review, decision answers, and approval decisions are only available here (not via
      MCP).

### ORCH-12 · MCP Gateway

- [ ] Listens on `127.0.0.1` ephemeral port only while enabled and a run is active.
- [ ] Rejects bad `Host`, missing/invalid token; constant-time compare; `Secret`-wrapped tokens.
- [ ] Tokens scoped by run/role/node/generation; revoked on terminate/rotation/quit.
- [ ] Stdio shim works for providers without HTTP MCP capability.
- [ ] Tool-by-role matrix test: every tool absent from a role returns "not permitted".

### ORCH-13 · PM and worker tools

- [ ] Implements the tool table in architecture §7.2 with zod-validated inputs.
- [ ] Every mutating call writes an `orch_events` row.
- [ ] No tool can merge, delete, archive, change settings, resolve permissions, or decide
      approvals/decisions (test enumerates the registry).
- [ ] Worker preamble instructs: push only your task branch, never merge or force-push.
- [ ] `submit_result` from a worker token can only affect its own node.

### ORCH-14 · PM Supervisor

- [ ] PM runs as an ordinary task `PM: <goal>` in its own worktree with a Claude ACP session.
- [ ] Event digests delivered with `sendPrompt({ placement: 'queue' })`, batched by window and size.
- [ ] PM slot held for the PM's active life; PM crash marks the run `needs-human`.

### ORCH-15 · Initial plan review and findings feed

- [ ] Nodes created before `submit_plan` is approved stay `draft`; the human can approve, edit, or
      reject individual nodes.
- [ ] After plan approval, newly added nodes queue immediately and appear in the findings feed as
      plan changes (unless blocked by an open decision).
- [ ] `record_finding` items appear in the inbox and never block work.

### ORCH-16 · Handoff rotation

- [ ] Triggers: context ratio, digest count, generation age; evaluated only at quiescence.
- [ ] Old PM terminated before new one starts (no overshoot); same PM task, new conversation.
- [ ] New generation bootstraps from store snapshot + latest memo + events since memo.
- [ ] Timeout path writes a mechanical handoff from the journal.

### ORCH-17 · Restart recovery

- [ ] Runs return `paused` after restart; no provider is started at boot.
- [ ] Reconciles nodes against app-DB tasks and ACP session lists; deleted tasks → `cancelled`.
- [ ] Resume re-mints tokens and re-attaches the PM.

### ORCH-18 · Quota headroom gate

- [ ] Reads provider usage probes; dispatch pauses below a configurable headroom threshold.
- [ ] Run shows "waiting for quota" instead of failing.

### ORCH-19 · Inbox: findings, decisions, approvals

- [ ] One inbox for findings, decisions, and approvals in the Orchestrator view + OS notification
      through the existing notification service.
- [ ] Each decision/approval lists blocked node ids; only those nodes (and dependents) wait; test
      proves unrelated ready nodes keep dispatching while a decision is open.
- [ ] Approval records bound to canonical action hash; mismatch on execute → `failed`.
- [ ] Answers and decisions are attributed and journaled and reach the PM in its next digest.

### ORCH-20 · Review handoff

- [ ] Accepted nodes set the task to `review` and show branch name and PR link (if the worker
      opened one).
- [ ] Merged PRs are observed through the existing PR sync; the node shows "merged by user".
- [ ] Preflight warns when the default branch has no GitHub branch protection.
- [ ] The slice contains no merge code path (test/lint check).

### ~~ORCH-21~~ · Dropped

Workers keep the project's normal Git credentials (owner decision: workers may push).

### ORCH-22 · Orchestrator UI

- [ ] View contributed through `contributions/browser.ts` + `views.ts` and aggregated by the
      browser manifest.
- [ ] Shows run graph, node states, census (`n/5`), pending approvals, timeline.
- [ ] Orchestrated tasks show a small badge linking back to their run; otherwise unchanged.

### ORCH-23 · Windows validation

- [ ] W1–W9 checked on Windows 11 with Claude and Codex workers.
- [ ] Long-path preflight warning implemented.

### ORCH-24 · Documentation

- [ ] `agents/architecture/orchestrator.md` describes the shipped behavior; this folder marked
      historical.
- [ ] ADR 0009 status → Accepted with any deviations recorded.

---

## Added 2026-10-21 (see `docs/pm/BACKLOG.md` for agent, model and status)

### ORCH-25 to ORCH-28 · Phase 0 spikes

ORCH-00 is split into four parallel spikes (D-009). Briefs: `docs/pm/tasks/ORCH-25.md` to
`ORCH-28.md`; their acceptance criteria are in the briefs.

### ORCH-29 · Next up list

- [ ] Each ready/queued node shows its full brief, agent, model, effort (dropdowns from discovered
      catalogs), the PM's reason, owned paths, dependencies, and blocking decisions.
- [ ] Brief, agent, model, and effort are editable before dispatch; edits are journaled.
- [ ] Actions: Launch now, Hold, Edit, Cancel; plan-review mode adds Approve.
- [ ] Ready nodes auto-dispatch after plan approval (D-014); a per-run "manual launch" switch makes
      them wait for Launch now. Held nodes never auto-dispatch.

### ORCH-30 · Ownership and hotspot gate

- [ ] A node is not dispatched while any running node owns an overlapping path (glob-aware) or a
      hotspot it needs.
- [ ] The conflict is shown on the node's Next up card.
- [ ] Tests cover exact, prefix, and glob overlaps, Windows path spelling, and hotspot exclusivity.

### ORCH-31 · Codex MCP isolation and Windows sandbox boundary

Follow-up spike to ORCH-26 (D-016). Brief: `docs/pm/tasks/ORCH-31.md`; acceptance criteria are in
the brief.
