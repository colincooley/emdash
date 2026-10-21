# ORCH-31 — Spike: Codex MCP isolation and Windows sandbox boundary

**Suggested agent:** Codex · **Model:** GPT-6.1 Sol (reasoning: high)
**Phase / priority:** P0 / P0
**Depends on (merged):** ORCH-26

## Context

Read `docs/pm/PROCESS.md`, then `docs/pm/reports/ORCH-26.md` (all of it) and
`docs/orchestrator/architecture.md` §5.8 (Session Policy).

ORCH-26 found that the Codex ACP adapter (`@agentclientprotocol/codex-acp` 1.13.1) **adds** the
servers passed in `session/new.mcpServers` to whatever MCP servers the effective Codex config
already has. It does not replace them. This machine has no global Codex MCP servers, so nobody has
seen a leak yet, but the design requires orchestrated workers to see only the Emdash gateway plus a
per-run allowlist. ORCH-26 also could not show whether the `agent` mode (`workspace-write` +
`on-request`) actually stops writes outside the workspace on native Windows, or only relies on
approval.

## Goal

Find a supported way to start a Codex ACP session that sees **only** the MCP servers Emdash passes,
while still using the owner's ChatGPT login, and find out whether `workspace-write` is enforced on
native Windows.

## Scope

- In:
  - Run `pnpm install --frozen-lockfile` in your worktree. Reuse or copy the ORCH-26 harness from
    `%TEMP%\emdash-spikes\ORCH-26\` into `%TEMP%\emdash-spikes\ORCH-31\`.
  - **Leak test without touching global config.** Make a dummy MCP server visible to Codex through
    a non-global layer only, for example a project-scoped `.codex/config.toml` in a scratch working
    directory, or `-c`/`--config` overrides if the adapter or `CODEX_PATH` launch can pass them.
    Confirm whether the dummy server shows up in an ACP session started the way Emdash starts one.
  - **Isolation options.** Test each candidate and record which one hides the dummy server while
    keeping the passed server and the ChatGPT login. Candidates: config overrides that clear
    `mcp_servers` (for example `-c mcp_servers={}`), adapter `_meta` or env options (read the
    adapter source in `node_modules/@agentclientprotocol/codex-acp/dist/`), project-trust settings.
    A separate `CODEX_HOME` is **out** if it needs copying `auth.json` (see Out).
  - **Sandbox boundary.** In `mode=agent`, ask the model to write a file in a second scratch folder
    outside the session `cwd` (both folders under `%TEMP%\emdash-spikes\ORCH-31\`). Record whether
    the write is blocked by the sandbox, asks for permission, or succeeds. Repeat with
    `mode=read-only`. Answer every permission request with `cancel`.
  - Keep prompts tiny and use GPT-6 Luna in the sessions.
- Out (do not do): editing `~/.codex/config.toml` or anything under `~/.codex`; running
  `codex mcp add`; copying, reading or printing `auth.json` or other credential files; setting a
  real `OPENAI_API_KEY` or `CODEX_API_KEY`; writing anywhere outside
  `%TEMP%\emdash-spikes\ORCH-31\`; changing repo files other than your report; product code.

## Ownership

- Owns (may create/edit): `docs/pm/reports/ORCH-31.md`; the scratch folder
  `%TEMP%\emdash-spikes\ORCH-31\` (outside the repo)
- Read only: everything else in the repo
- Must not touch: any other repo file, `pnpm-lock.yaml`, `~/.codex`, any other project, Emdash app
  data

## Deliverables

Answer each with commands run and verbatim output (redact any token):

1. Which non-global layer made a dummy MCP server appear in an Emdash-style Codex ACP session, with
   evidence (server names only).
2. For each isolation candidate: did it hide the dummy server, keep the passed server, and keep
   `Logged in using ChatGPT` behavior? A table is fine.
3. The recommended isolation mechanism and exactly how Emdash would apply it (spawn args, env, or
   `session/new` fields), or "none found" with what an upstream adapter change would need.
4. Sandbox results for `agent` and `read-only`: outside-`cwd` write blocked, asked, or succeeded,
   with the ACP updates shown.
5. Go / no-go for Codex workers with strict MCP, and the worker `mode` you recommend.

## Acceptance criteria

- [ ] Every deliverable answered with evidence, or marked "could not determine" with why.
- [ ] Nothing under `~/.codex` changed or read beyond `codex login status` and `codex mcp list`.
- [ ] No repo file changed except `docs/pm/reports/ORCH-31.md` (`git status` shown in the report).
- [ ] No tokens or credential contents in the report.

## Report

Write `docs/pm/reports/ORCH-31.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, open a PR to `main` on `colincooley/emdash`, and stop. Don't merge.
