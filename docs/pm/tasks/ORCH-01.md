# ORCH-01 — Orchestrator Mode setting (off by default)

**Suggested agent:** Codex · **Model:** GPT-6.1 Sol (reasoning: medium)
**Phase / priority:** P1 / P0
**Depends on (merged):** —

## Context

Read `AGENTS.md`, `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §3 (placement) and
§10.1 (feature flag), and the ORCH-01 acceptance criteria in `docs/orchestrator/backlog.md`.

Orchestrator Mode is an optional feature in a new desktop slice,
`apps/emdash-desktop/src/core/features/orchestrator/`. Nothing in it exists yet. Every later task
checks one switch: an app setting `orchestrator` with `{ enabled: boolean }`, default `false`.

Use an **app setting**, not a telemetry feature flag. Feature flags here are telemetry-backed
(`src/renderer/lib/hooks/useFeatureFlag.ts`) and read as `false` when telemetry is off, which would
make the feature impossible to enable for users who disable telemetry.

Patterns to follow (paths relative to `apps/emdash-desktop/src/core/`):

- Contribution: `features/source-control/contributions/settings.ts` (`defineSettingsContribution`).
- Registration: `manifests/shared/settings-contributions.ts` (`appSettingsSchemaContributions`).
- Settings UI: `features/settings/contributions/settings-pages.ts` and
  `features/settings/browser/pages/`. Look at how an existing boolean setting is rendered and
  saved, and follow that.
- Node-side reads: find how a main-process service reads an app setting and reacts to changes
  (search for an existing `getSetting`/settings-service subscription), and follow that.

Another task (ORCH-02) is creating `features/orchestrator/node/store/` at the same time. Stay out
of that folder.

## Goal

The owner can turn Orchestrator Mode on and off in Settings; it's off by default; main-process
code has one accessor to read it and be told when it changes. With the setting off, nothing about
the app changes.

## Scope

- In:
  - `features/orchestrator/contributions/settings.ts`: `orchestrator` contribution, schema
    `{ enabled: boolean }`, default `{ enabled: false }`.
  - Register it in `manifests/shared/settings-contributions.ts` (one import, one entry).
  - A toggle labelled "Orchestrator Mode (experimental)" with a one-line description ("Let a PM
    agent plan, launch and supervise tasks. Off by default."). Put it on the most fitting existing
    settings page, in an "Experimental" section if one exists, otherwise add a small section at the
    bottom of the General page. Don't add a new settings page.
  - `features/orchestrator/node/orchestrator-enabled.ts`: read the current value and subscribe to
    changes, built on the existing settings service. No other behavior yet.
  - Tests: default is `false`; schema rejects bad values; the accessor reports changes. Update the
    existing settings manifest tests if they enumerate keys.
- Out: any other orchestrator behavior, Wire contracts, views, commands, the store, telemetry
  flags, new dependencies.

## Ownership

- Owns (may create/edit): `features/orchestrator/contributions/`,
  `features/orchestrator/node/orchestrator-enabled.ts` and its test,
  `manifests/shared/settings-contributions.ts` and its test (hotspot, this task only),
  the one settings page file where the toggle goes (name it in your report).
- Read only: everything else.
- Must not touch: `features/orchestrator/node/store/` (ORCH-02), `pnpm-lock.yaml`, any
  `package.json`, app DB schema or `drizzle/`, `AGENTS.md`.

## Deliverables

1. The setting, its registration, the toggle, and the node accessor.
2. Tests as above.
3. A screenshot of the toggle (run the app with `EMDASH_DB_FILE` pointing at a scratch DB; never
   the default DB) in the report, or a note on why you couldn't.

## Acceptance criteria

- [ ] Fresh install: setting reads `{ enabled: false }`.
- [ ] Toggling in the UI persists across restart (scratch DB).
- [ ] With the setting off, no new files, ports, views or commands appear.
- [ ] `pnpm run format`, `lint`, `typecheck` pass; focused tests pass (list the commands).
- [ ] Only owned files changed (`git status` in the report).

## Report

Write `docs/pm/reports/ORCH-01.md` using the report template in `docs/pm/PROCESS.md`, commit on
your branch, push it, open a PR to `main` on `colincooley/emdash`, and stop. Don't merge.
