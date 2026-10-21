# ORCH-26 — Spike: per-session MCP servers and config options over Codex ACP (Windows)

**Suggested agent:** Codex · **Model:** GPT-6.1 Sol (reasoning: high)
**Phase / priority:** P0 / P0
**Depends on (merged):** —

## Context

Read `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §5.8 and §7, and
`docs/orchestrator/risks.md` rows P2, P5, P8, P9, W1, W7.

Orchestrator Mode will run Codex workers through Emdash's Codex ACP adapter and give each worker a
small Emdash-hosted MCP server (one tool, `submit_result`) per session through ACP
`session/new.mcpServers`. It also needs to set model, reasoning effort, and permission mode per
worker. None of this has been checked against the Codex adapter yet.

How Emdash runs Codex over ACP today:

- `packages/plugins/src/agents/impl/codex/index.ts` — `acp.buildSpawn`: `process.execPath` (the
  Electron binary) with `ELECTRON_RUN_AS_NODE=1`, the adapter script, and `CODEX_PATH=<path to
  codex>`.
- `packages/plugins/src/agents/impl/codex/adapter.ts` — adapter is
  `@agentclientprotocol/codex-acp/dist/index.js` (`@openai/codex` external).
- `packages/plugins/src/agents/impl/codex/acp-errors.ts` — `isCodexSessionNotFound`.
- `packages/core/src/runtimes/acp/node/connection/acp-agent-connection.ts` — `initialize` and
  `mcpCapabilities`.
- `packages/core/src/runtimes/acp/node/runtime/session-materializer.ts` and `mcp-servers.ts` — how
  `mcpServers` are built today.

## Goal

Find out, with evidence, whether per-session MCP servers (HTTP and stdio) work through the Codex
ACP adapter on Windows, and which native config option ids control model, reasoning effort,
permission/sandbox mode, and collaboration mode.

## Scope

- In:
  - Run `pnpm install --frozen-lockfile` in your worktree.
  - Write a throwaway Node harness **outside the repo** in `%TEMP%\emdash-spikes\ORCH-26\` that
    spawns the adapter the way `buildSpawn` does and talks ACP with the `@agentclientprotocol/sdk`
    package already in the repo's `node_modules`. Don't add dependencies anywhere.
  - Write a minimal MCP server in the same folder exposing one tool (`submit_result` that accepts
    `{ summary: string }` and returns `ok`).
  - Answer every question under Deliverables. Keep prompts tiny; use GPT-6 Luna in the session
    where possible to save quota.
- Out (do not do): changing any repo file other than your report; editing `~/.codex/config.toml`
  or other global config; setting a real `OPENAI_API_KEY`; product code.

## Ownership

- Owns (may create/edit): `docs/pm/reports/ORCH-26.md`; the scratch folder
  `%TEMP%\emdash-spikes\ORCH-26\` (outside the repo)
- Read only: everything else in the repo
- Must not touch: any other repo file, `pnpm-lock.yaml`, any other project, Emdash app data

## Deliverables

Answer each with commands run and verbatim output (redact any token):

1. What `initialize` returns: `agentCapabilities.mcpCapabilities` (`http`, `sse`), `loadSession`,
   protocol version, adapter version, codex-cli version.
2. HTTP MCP on `127.0.0.1:<port>` with an `Authorization: Bearer <random>` header: does the model
   see and call `submit_result`? Does the header arrive? Does the token appear in logs?
3. Stdio MCP via a shim started as the Electron binary with `ELECTRON_RUN_AS_NODE=1`: works on
   Windows?
4. Do MCP servers from `~/.codex/config.toml` also appear in the session? List names only. Is there
   a way to make the session see only the servers passed in `session/new`?
5. `session/load` with `mcpServers` passed again: tools still available? What does a missing
   session return (confirm `isCodexSessionNotFound` matches it)?
6. The `configOptions` the session reports: every option id, category, and values. Identify the
   ids/values for (a) model, (b) reasoning effort, (c) a mode where file edits in the workspace are
   allowed but commands outside it ask, (d) `collaboration_mode`.
7. Compare the model values with `codex debug models` (entries with `"visibility": "list"`).
8. Windows: how does that permission/sandbox mode behave on native Windows (does the sandbox run,
   degrade, or refuse)? Any other Windows-specific issues.
9. Go / no-go recommendation for Codex workers as designed, and suggested design changes.

## Acceptance criteria

- [ ] Every deliverable question answered with evidence, or marked "could not determine" with why.
- [ ] No repo file changed except `docs/pm/reports/ORCH-26.md` (`git status` shown in the report).
- [ ] No tokens or credential contents in the report.

## Report

Write `docs/pm/reports/ORCH-26.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, and stop. Don't merge.
