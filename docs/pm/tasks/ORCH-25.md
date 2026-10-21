# ORCH-25 — Spike: per-session MCP servers and config options over Claude ACP (Windows)

**Suggested agent:** Claude Code · **Model:** Opus 5.5 (effort: high)
**Phase / priority:** P0 / P0
**Depends on (merged):** —

## Context

Read `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §5.8 and §7, and
`docs/orchestrator/risks.md` rows P1, P5, P6, P8, W1, W2.

Orchestrator Mode plans to give a Claude Code PM agent an Emdash-hosted MCP server, delivered per
session through ACP `session/new.mcpServers`, with a bearer token in an `Authorization` header.
Nobody has checked yet that Emdash's Claude ACP adapter accepts that, exposes the tools to the
model, or works on Windows.

How Emdash runs Claude over ACP today:

- `packages/plugins/src/agents/impl/claude/index.ts` — `acp.buildSpawn`: `process.execPath` (the
  Electron binary) with `ELECTRON_RUN_AS_NODE=1`, the adapter script, and
  `CLAUDE_CODE_EXECUTABLE=<path to claude>`.
- `packages/plugins/src/agents/impl/claude/adapter.ts` — adapter is
  `@agentclientprotocol/claude-agent-acp/dist/index.js`.
- `packages/core/src/runtimes/acp/node/connection/acp-agent-connection.ts` — `initialize` and how
  `mcpCapabilities` is read.
- `packages/core/src/runtimes/acp/node/runtime/session-materializer.ts` (`resolveSessionMcpServers`,
  `buildNewSessionRequest`) and `mcp-servers.ts` — how `mcpServers` are built today.

## Goal

Find out, with evidence, whether per-session MCP servers (HTTP and stdio) work through the Claude
ACP adapter on Windows, what the adapter advertises, and which native config option ids control
permission mode, model, and effort.

## Scope

- In:
  - Run `pnpm install --frozen-lockfile` in your worktree so the adapter is in `node_modules`.
  - Write a throwaway Node harness **outside the repo** in `%TEMP%\emdash-spikes\ORCH-25\` that
    spawns the adapter the way `buildSpawn` does and talks ACP to it with the
    `@agentclientprotocol/sdk` package already installed in the repo (load it from the repo's
    `node_modules`; don't add dependencies anywhere).
  - Write a minimal MCP server in the same scratch folder exposing one tool (`ping` returning a
    fixed string). Hand-roll it or use an MCP SDK that is already present transitively in
    `node_modules`; don't install anything.
  - Answer every question under Deliverables.
  - Keep test prompts tiny and pick the cheapest model the session offers, to save subscription
    quota.
- Out (do not do): changing any file in the repo other than your report; `claude mcp add` or any
  change to `~/.claude*` or global config; setting a real `ANTHROPIC_API_KEY`; implementing product
  code.

## Ownership

- Owns (may create/edit): `docs/pm/reports/ORCH-25.md`; the scratch folder
  `%TEMP%\emdash-spikes\ORCH-25\` (outside the repo)
- Read only: everything else in the repo
- Must not touch: any other repo file, `pnpm-lock.yaml`, any other project, Emdash app data

## Deliverables

Answer each with commands run and verbatim output (redact any token):

1. What `initialize` returns: `agentCapabilities.mcpCapabilities` (`http`, `sse`),
   `loadSession`, protocol version, adapter version.
2. HTTP MCP: `session/new` with an `http` server on `127.0.0.1:<port>` and an
   `Authorization: Bearer <random>` header. Does the model see and call `ping`? Does the server
   receive the header? Does the token show up in adapter stderr/logs?
3. Stdio MCP: the same tool through a stdio shim started as the Electron binary
   (`apps/emdash-desktop/node_modules/electron/...`) with `ELECTRON_RUN_AS_NODE=1`. Does it work on
   Windows? Any quoting or path issues?
4. Which other MCP servers/tools the model reports seeing in that session (user-global
   `~/.claude.json` servers? claude.ai connectors?). List names only. Is there a way to make the
   session see **only** the servers passed in `session/new`? If you find an env var or setting
   (candidate: `ENABLE_CLAUDEAI_MCP_SERVERS=false`), test it through ACP.
5. `session/load` of the same session with `mcpServers` passed again: are the tools available after
   reload (needed for PM rotation and restart)?
6. The `configOptions` the session reports: list every option id, its category, and its values.
   Identify the ids/values for (a) "edits allowed, commands ask" permission mode, (b) model,
   (c) effort/thinking level.
7. Windows: did binding `127.0.0.1` trigger a firewall prompt? Anything else Windows-specific.
8. Go / no-go recommendation for architecture §7.1 with Claude, and any design changes you suggest.

## Acceptance criteria

- [ ] Every deliverable question answered with evidence, or marked "could not determine" with why.
- [ ] No repo file changed except `docs/pm/reports/ORCH-25.md` (`git status` shown in the report).
- [ ] No tokens or credential contents in the report.

## Report

Write `docs/pm/reports/ORCH-25.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, and stop. Don't merge.
