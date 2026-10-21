# ORCH-02 — Orchestrator durable store and state machine

**Suggested agent:** Codex · **Model:** GPT-6.1 Sol (reasoning: high)
**Phase / priority:** P1 / P0
**Depends on (merged):** —

## Context

Read `AGENTS.md`, `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §5.1 (store
tables), §8.1 (node states) and §9 (handoffs), and the ORCH-02 acceptance criteria in
`docs/orchestrator/backlog.md`. Read `agents/risky-areas/database.md` too.

Decision D-004: orchestrator state lives in its **own** SQLite file, `orchestrator.db`, never in the
app DB. Don't touch `apps/emdash-desktop/src/core/services/app-db/node/schema.ts` or
`apps/emdash-desktop/drizzle/`.

Follow the existing separate-store pattern exactly:

- `packages/core/src/runtimes/automations/node/persistence/store.ts`: `defineDurableSqliteStore`
  with `betterSqlite3Driver`, generated migrations, `backup: { retain: 2 }`,
  `assertSqliteStoreInvariants`.
- `packages/core/drizzle-automations.config.ts` and the `db:generate:automations` script in
  `packages/core/package.json`: drizzle-kit generate + `scripts/bundle-drizzle-migrations.ts`.
- Versioned JSON columns: `defineVersionedSchema()` (see `agents/conventions/versioned-schemas.md`).

Placement (architecture §4): `apps/emdash-desktop/src/core/features/orchestrator/node/store/`.
If the desktop app's Drizzle versions or tooling can't run the bundle step for a store there,
stop and say so in the report rather than moving the store elsewhere; the PM will decide.

Another task (ORCH-01) is adding `features/orchestrator/contributions/` and
`features/orchestrator/node/orchestrator-enabled.ts` at the same time. Stay out of those.

## Goal

A tested `orchestratorStore` with the §5.1 tables and an explicit, tested node state machine that
later tasks (scheduler, launcher, monitor) build on. Nothing opens it yet.

## Scope

- In:
  - Schema for `orch_runs`, `orch_nodes`, `orch_edges`, `orch_events`, `orch_inbox`,
    `orch_pm_sessions`, `orch_handoffs`, with the columns in §5.1. `orch_nodes` includes `brief`,
    `provider`, `model`, `effort`, `choiceReason`, `owns`, `readOnly`, `mustNotTouch`.
  - Generated migrations and the bundled `migrations.generated.ts`, produced by the tool, never
    hand-written.
  - `store.ts` defining the store; a path helper honoring `EMDASH_ORCHESTRATOR_DB_FILE`.
  - Repository functions only as far as tests need: create run, add node, add edge (reject cycles),
    transition node, append event, list nodes by run.
  - `node-transitions.ts`: the node state transition table from §8.1 as data, plus
    `assertTransition(from, to)`. Illegal transitions return an error `Result`.
  - Tests: store opens and migrates on a temp file; invariants pass; every legal transition is
    allowed and a sample of illegal ones is rejected; cycle rejection; `EMDASH_ORCHESTRATOR_DB_FILE`
    override.
- Out: opening the store at app boot, Wire contracts, scheduler logic, UI, new dependencies.

## Ownership

- Owns (may create/edit): `apps/emdash-desktop/src/core/features/orchestrator/node/store/`,
  `apps/emdash-desktop/drizzle-orchestrator.config.ts`, and **one new script line**
  (`db:generate:orchestrator`) in `apps/emdash-desktop/package.json` (hotspot, this task only;
  no dependency changes).
- Read only: everything else.
- Must not touch: `features/orchestrator/contributions/` and `node/orchestrator-enabled.ts`
  (ORCH-01), `pnpm-lock.yaml`, dependencies in any `package.json`, app DB schema, `drizzle/`,
  `drizzle/meta/`, `AGENTS.md`.

## Deliverables

1. Schema, migrations, store, transition table, minimal repository functions.
2. Tests as above.
3. In the report: the final transition table, and any column you added or changed versus §5.1
   and why.

## Acceptance criteria

- [ ] `git diff main -- apps/emdash-desktop/src/core/services/app-db apps/emdash-desktop/drizzle`
      is empty.
- [ ] `pnpm-lock.yaml` unchanged.
- [ ] Migrations regenerate identically from the script.
- [ ] `pnpm run format`, `lint`, `typecheck` pass; the new tests pass (list the commands).
- [ ] Only owned files changed (`git status` in the report).

## Report

Write `docs/pm/reports/ORCH-02.md` using the report template in `docs/pm/PROCESS.md`, commit on
your branch, push it, open a PR to `main` on `colincooley/emdash`, and stop. Don't merge.
