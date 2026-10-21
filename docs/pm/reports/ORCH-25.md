# ORCH-25 report — Spike: per-session MCP servers and config options over Claude ACP (Windows)

**Branch:** emdash/25-m0ps4
**Status:** Done

## Summary

Per-session MCP works through Emdash's Claude ACP adapter on Windows, over both HTTP and stdio.
The adapter is `@agentclientprotocol/claude-agent-acp` 0.81.2 (Claude Agent SDK 0.3.280), driving
Claude Code 2.1.307. It advertises `mcpCapabilities.http` and `sse` and `loadSession: true`.

- With an HTTP server on `127.0.0.1`, the model saw `mcp__orch__ping` and called it. The server
  got the `Authorization: Bearer` header on every request.
- The token never showed up in adapter stderr or anywhere under `~/.claude` / `~/.claude.json`.
- A stdio shim run as `electron.exe` with `ELECTRON_RUN_AS_NODE=1` also worked, including with
  spaces in both the executable path and the script path.
- `session/load` in a fresh adapter process, with a re-minted port and token, brought the tools
  back.
- Isolation: `_meta.claudeCode.options.strictMcpConfig: true` on `session/new` / `session/load`
  limits the session to exactly the servers passed. It drops both project `.mcp.json` servers and
  claude.ai connectors. `ENABLE_CLAUDEAI_MCP_SERVERS=false` drops only the claude.ai connectors.
- Native config option ids:
  - permission mode: `mode` (`acceptEdits`)
  - model: `model`
  - effort: `effort` (category `thought_level`)
- `acceptEdits` still asks permission for MCP tools. `allowedTools` in `_meta` pre-approves them.

Recommendation: **Go** for architecture §7.1 with Claude, using HTTP as the primary transport.
There are a few design changes, listed in §8.

## Setup

- `pnpm install --frozen-lockfile` in the worktree. It finished; the only side effect was a
  line-ending-only rewrite of `package.json` (`git diff` showed no content change), which I
  reverted with `git checkout -- package.json`.
- The scratch harness is in `C:\Users\colin\AppData\Local\Temp\emdash-spikes\ORCH-25\`:
  - `harness.mjs` spawns the adapter the same way `acp.buildSpawn` does:
    `node_modules\electron\dist\electron.exe` + adapter `dist/index.js`, `ELECTRON_RUN_AS_NODE=1`,
    and `CLAUDE_CODE_EXECUTABLE=C:\Users\colin\.local\bin\claude.exe`.
  - It talks ACP using the repo's `@agentclientprotocol/sdk` 1.1.0 (`ClientSideConnection` +
    `ndJsonStream`, the same as `connectStdioAcp`). Its `initialize` params are identical to
    `initializeAgent` in `acp-agent-connection.ts`.
  - Before spawning, it strips this nested session's `CLAUDECODE` / `CLAUDE_CODE_*` markers and
    any `ANTHROPIC_*` variables. None were set, and no API key was used; it ran on the
    subscription login.
- `mcp-core.mjs`, `mcp-http.mjs`, `mcp-stdio.mjs`: a hand-rolled MCP server with one tool, `ping`,
  that returns `pong-orch25-7f3a`. Nothing was installed.
  - The HTTP server checks the bearer token in constant time and checks `Host`. It logs only
    whether the header matched, never the token value.
- Every session used cwd `...\ORCH-25\cwd` with model `haiku` (set through `session/set_config_option`
  `model=haiku`) and a single short prompt. There were 12 prompt turns in total.

## Deliverables

### 1. `initialize`

`node harness.mjs init` (abridged to the asked fields; verbatim JSON):

```json
"protocolVersion": 1,
"agentCapabilities": {
  "mcpCapabilities": { "http": true, "sse": true },
  "loadSession": true,
  "sessionCapabilities": { "additionalDirectories": {}, "close": {}, "delete": {}, "fork": {},
                           "list": {}, "resume": {}, "subagents": {} },
  "promptCapabilities": { "image": true, "embeddedContext": true }, ...
},
"agentInfo": { "name": "@agentclientprotocol/claude-agent-acp", "title": "Claude Agent",
               "version": "0.81.2" },
"authMethods": []
```

The adapter depends on `@anthropic-ai/claude-agent-sdk` 0.3.280. `claude --version` reports
`2.1.307 (Claude Code)`. The source hard-codes `mcpCapabilities: { http: true, sse: true }` and
`loadSession: true` (`acp-agent.js` ~L1112), so these values don't depend on configuration.

### 2. HTTP MCP

`node harness.mjs http` sent
`{type:"http", name:"orch", url:"http://127.0.0.1:<port>/mcp", headers:[{name:"Authorization", value:"Bearer <REDACTED>"}]}`.

- **The model saw and called `ping`.** Tool calls were `["ToolSearch", "mcp__orch__ping"]`, and the
  assistant text was ``"1) `mcp__orch__ping` returned: `pong-orch25-7f3a`. ..."``. The MCP tool is
  deferred: the model first loads it with `ToolSearch`.
- The tool call went through ACP `session/request_permission` with options `allow-once`,
  `allow-with-updates` and `reject` (default mode).
- **The server received the header on every request.** Server log, one entry per request:
  `hasAuthHeader:true, authScheme:"Bearer", authOk:true, hostOk:true, host:"127.0.0.1:57294"`,
  `ua:"claude-code/2.1.307 (sdk-ts, agent-sdk/0.3.280)"`. The request sequence was:
  `POST server/discover` (probe, `mcp-protocol-version: 2026-07-28`) → our `-32601` → `POST initialize`
  → `POST notifications/initialized` → `GET` (SSE stream, we return 405) → `POST tools/list` →
  `POST tools/call ping`.
- **The token wasn't in the logs.** The harness printed `token occurrences in adapter stderr: 0` on
  every run. Adapter stderr contains only `[session/create] ... phase=...` and
  `[session/query] ... apiType=native baseUrl=native` lines.
  - I also grepped every run's token across `~/.claude` and `~/.claude.json`. That includes the CLI's
    session transcripts for the scratch cwd (`~/.claude/projects/C--Users-colin-AppData-Local-Temp-emdash-spikes-ORCH-25-cwd/`).
    There were **no hits**. The only hits were the harness's own unredacted run logs in the scratch
    folder.

### 3. Stdio MCP via the Electron binary

`node harness.mjs stdio` sent
`{name:"orchstdio", command:"<repo>\\node_modules\\electron\\dist\\electron.exe", args:["...\\ORCH-25\\mcp-stdio.mjs"], env:[{ELECTRON_RUN_AS_NODE:"1"}, ...]}`.

- **It works on Windows.** The shim logged
  `execPath: ...\electron.exe, electronRunAsNode: "1", versions: {node: "24.15.0", electron: "40.10.2"}`.
  RPCs were `server/discover, initialize, notifications/initialized, tools/list, tools/call ping`,
  and the model replied ``"`mcp__orchstdio__ping` returned: `pong-orch25-7f3a`"``.
- **Spaces in paths:** I repeated the run with `command` = `...\ORCH-25\electron dist\electron.exe`
  (a junction to the real `dist`) and the script in `...\ORCH-25\dir with space\mcp-stdio.mjs`.
  It worked: the shim's `argv` showed both paths intact and the tool returned the fixed string.
  No shell or `.cmd` wrapping is involved, so there was nothing to quote.
- **Path note:** the repo uses `node-linker=hoisted`, so Electron is at
  `<repo>\node_modules\electron\dist\electron.exe`, not under
  `apps/emdash-desktop/node_modules/electron/` as the brief says. At runtime this doesn't matter,
  because the shim would use `process.execPath`.

### 4. Other servers and isolation

Without any isolation (`node harness.mjs http --prompt list`), the model listed:
`claude_ai_Claude_Docs, claude_ai_Google_Drive, claude_ai_Robinhood, claude_ai_Strava, decoy, orch`.

- The `claude_ai_*` servers are claude.ai account connectors. The available commands also included
  `mcp:claude.ai Strava:*` and `mcp:claude.ai Robinhood:*` prompts.
- `~/.claude.json` has no user-global or project `mcpServers` entries (I checked key names only), so
  none of those appeared.
- `decoy` is a stdio server I put in the scratch cwd's `.mcp.json`, with
  `.claude/settings.local.json` `enableAllProjectMcpServers: true`. It stands in for non-session
  servers, because I couldn't add a global server without breaking the brief's rules. The adapter
  passes `settingSources: ["user", "project", "local"]`, so user-global servers load through the
  same path.

Results with each switch (session servers = `orch` only):

| Variant | Model-reported servers | claude.ai MCP prompts in commands | Decoy process spawned* |
| --- | --- | --- | --- |
| none | claude_ai ×4, decoy, orch | yes (8) | yes (1) |
| `--env ENABLE_CLAUDEAI_MCP_SERVERS=false` (adapter process env) | `decoy, orch` | none | — |
| `_meta.claudeCode.options.strictMcpConfig: true` | `orch` | none | **no (0)** |
| both | `orch` | none | — |

\*I ran with `--no-mcp` and the decoy writing a start-up log. It logged `strict: 0` and `base: 1`
start events, so the strict-mode result above doesn't rely only on what the model said.

**Answer:** yes. Pass `_meta: { claudeCode: { options: { strictMcpConfig: true } } }` on
`session/new` (and on `session/load`, see §5). The session then sees **only** the servers passed in
`mcpServers`: no `.mcp.json`/`~/.claude.json` servers and no claude.ai connectors.
`ENABLE_CLAUDEAI_MCP_SERVERS=false` is a narrower switch that only removes claude.ai connectors.
Both were tested through ACP.

### 5. `session/load` with `mcpServers` re-sent

I ran `node harness.mjs load --session 85d38b35-... --strict` in a new adapter process with a new
HTTP server, a new port and a new token. This mimics an app restart or a PM rotation with a
re-minted token.

- `loadSession` succeeded and replayed history. Its `configOptions` ids were `mode`, `model` and
  `effort`. `fast` was missing after load.
- After load, the model called `mcp__orch__ping` → `pong-orch25-7f3a`. The new server saw
  `initialize, tools/list, tools/call ping`, with `authOk: true` on every request.
- The strict flag held on load: decoy spawned 0 times and the model listed only `orch`.
- Because the original session was non-strict, the model added: "the resume notice says the
  claude.ai Docs, Google Drive, Robinhood, and Strava servers have disconnected". In other words,
  the transcript records which servers went away. This is harmless if sessions are strict from the
  start.

### 6. `configOptions` (from `session/new`)

| id | category | type | current | values |
| --- | --- | --- | --- | --- |
| `mode` | `mode` | select | `default` | `default` (Manual), `acceptEdits` (Accept edits), `plan`, `auto` (Claude handles permission decisions), `bypassPermissions` |
| `model` | `model` | select | `default` | `default`, `opus`, `sonnet`, `fable`, `haiku`, `claude-haiku-4-5-20251001`, `claude-sonnet-5`, `claude-opus-5`, `claude-fable-5`, `claude-opus-4-8`, `claude-opus-4-7`, `claude-opus-4-6`, `claude-sonnet-4-6` |
| `effort` | `thought_level` | select | `high` | `default`, `low`, `medium`, `high`, `xhigh`, `max` |
| `fast` | `model_config` | boolean | `false` | — |

`session/new` also returns legacy `modes` with the same five ids. It didn't return `models`.

- (a) "Edits allowed, commands ask": `mode` = `acceptEdits`. `session/set_config_option` accepted
  it. The `auto` mode is also present; like `bypassPermissions`, it should be refused for
  orchestrated sessions.
- (b) Model: `model`. The cheapest model is `haiku`.
- (c) Effort/thinking: `effort` (category `thought_level`).

**Permission behavior that affects the design:** in `acceptEdits`, the MCP call still raised
`session/request_permission` for `mcp__orch__ping`. Adding
`_meta.claudeCode.options.allowedTools: ["mcp__orch__ping"]` (together with `strictMcpConfig`) let
the call through without a permission request. I tested exact tool names only, not wildcards.

### 7. Windows

- **Firewall:** no prompt was observed, and no rule was created.
  - The harness HTTP server bound `127.0.0.1` from `node.exe`. That binary already had older
    "Node.js JavaScript Runtime" Allow rules, so on its own that run proves nothing.
  - I also ran a `127.0.0.1` listener under `electron.exe` with `ELECTRON_RUN_AS_NODE=1` and fetched
    from it. Afterwards
    `Get-NetFirewallRule -Direction Inbound | ... -match 'electron' | Measure-Object` returned `0`.
    A firewall prompt would have created a rule either way, so no prompt fired.
  - I can't see the screen, so the owner may want to confirm by eye in the real app.
- The stdio shim needs no `.cmd`/shell, and spaces in paths work (§3).
- Electron's Node is 24.15.0. The adapter process and the shim ran without problems.
- Each run left no orphaned processes. A `Win32_Process` check for `emdash-spikes` /
  `claude-agent-acp` came back empty.
- Running sessions creates transcripts under `~/.claude/projects/<scratch-cwd-slug>/`. That is
  normal CLI behavior and not a config change. No project entry for the scratch cwd was added to
  `~/.claude.json`.

### 8. Recommendation: Go

**Go** for §7.1 with Claude, with HTTP as the primary transport. Suggested design changes:

1. **Use `strictMcpConfig` for the strict MCP set (§5.8).** Send
   `_meta.claudeCode.options.strictMcpConfig: true` on every orchestrated `session/new` **and**
   `session/load`. This is stronger than the risks P6 candidate: it also removes user, project and
   `.mcp.json` servers. Keep `ENABLE_CLAUDEAI_MCP_SERVERS=false` in the adapter env as a second
   layer.
   - The adapter merges `_meta.claudeCode.options.env` into the CLI env per session, so the env
     switch could also be per session. I haven't tested that.
2. **The ACP runtime needs a per-session `_meta` field.** `SessionMaterializer.buildNewSessionRequest`
   / `buildLoadSessionRequest` currently send only `{ cwd, mcpServers }`, and `mcpServers` comes
   only from `agentHost.readMcpServers`.
   - Orchestrated starts need to add (a) the orchestrator server and (b) provider `_meta`
     (`strictMcpConfig`, `allowedTools`), and re-supply both on load.
   - This touches the `packages/core/src/runtimes/acp/**` hotspot. Like `mcpServers` credentials,
     the token and `_meta` must stay out of persisted intents.
3. **Pre-approve orchestrator tools.** Even in `acceptEdits`, every MCP tool call asks permission.
   Pass `allowedTools` with the role's exact tool names (`mcp__<serverName>__<tool>`), or the
   human gets a prompt on every `get_run` / `submit_result`. Keep the server name stable (for
   example `emdash_orch`), because tool ids are built from it.
4. **Gateway protocol details.** Claude 2.1.307 first sends a `server/discover` probe
   (`mcp-protocol-version: 2026-07-28`) and then opens a `GET` SSE stream. The gateway must answer
   unknown methods with JSON-RPC `-32601` and `GET` with `405` (both worked here), or use an MCP SDK
   transport that does. `Authorization` and `Host` arrive on every request, including `GET`, so
   the §7.1 header and Host checks work as designed.
5. **The stdio shim isn't needed for Claude.** It works, so keep it for Codex if needed (ORCH-26).
6. **Token rotation on reload works.** A re-minted token and port on `session/load` was enough.
   Nothing about the old server or token is persisted by the CLI.
7. **Permission-mode policy.** Refuse both `bypassPermissions` **and** `auto`, because `auto`
   delegates permission decisions to Claude.
8. **Also consider limiting `settingSources`.** I didn't test this. Orchestrated sessions still load
   user and project settings (hooks, permissions, plugins, skills). `settingSources` is a
   pass-through `_meta` option.

## Changed files

- `docs/pm/reports/ORCH-25.md` — this report. No other repo file changed.

## Verification

- `git status --short` before writing this report: empty (clean).
- `git status --short` after writing it, before commit: `?? docs/pm/reports/ORCH-25.md` only.
- No lint, typecheck or test runs, because this spike changes no code.

## Tests

None (spike). The scratch harness is outside the repo.

## How to check it by hand

From `C:\Users\colin\AppData\Local\Temp\emdash-spikes\ORCH-25\` with the repo installed:

- `node harness.mjs init`
- `node harness.mjs http`
- `node harness.mjs stdio`
- `node harness.mjs http --prompt list [--strict | --env ENABLE_CLAUDEAI_MCP_SERVERS=false]`
- `node harness.mjs load --session <id from an http run> --strict`
- `node harness.mjs http --mode acceptEdits --strict --options '{"allowedTools":["mcp__orch__ping"]}'`

Logs go to `logs\`. Delete `logs\*.token` and the `cwd\.mcp.json` decoy when done.

## Deviations from the brief

- The brief gives the Electron path as `apps/emdash-desktop/node_modules/electron/...`. The repo is
  hoisted, so I used `<repo>\node_modules\electron\dist\electron.exe`.
- I couldn't test user-global `~/.claude.json` servers directly, because there are none and the
  brief forbids adding any. I used a project-scope `.mcp.json` decoy in the scratch cwd instead;
  it loads through the same settings-source path.
- I added two checks the brief didn't ask for: MCP permission prompting under `acceptEdits`, and
  `allowedTools` pre-approval. Both bear directly on the §7.2 tool UX.

## Blockers and open questions

- The design needs to decide (§8.2) whether orchestrated sessions get a per-session `_meta` channel
  in the ACP runtime (hotspot `packages/core/src/runtimes/acp/**`), or whether isolation goes
  through a dedicated connection with its own env.
- Should orchestrated sessions also limit `settingSources` (user hooks and plugins)? This is a
  product decision.

## Suggested follow-ups

- Test `allowedTools` wildcards (`mcp__emdash_orch__*`) and per-session `_meta...options.env` with
  `ENABLE_CLAUDEAI_MCP_SERVERS=false`.
- Owner: confirm by eye that the real app shows no firewall prompt when the gateway binds.
- Add an ACP runtime test that the orchestrator token and `_meta` never reach persisted intents
  (ORCH-03).
