# ORCH-26 report — Codex ACP per-session MCP spike on Windows

**Branch:** `emdash/26-lo52w`
**Status:** Done

## Summary

Codex ACP 1.13.1 with codex-cli 0.163.0 accepted both HTTP and stdio MCP servers in
`session/new` on native Windows. GPT-6 Luna called `submit_result` through each transport,
and the tool remained available after `session/load` with the server supplied again. HTTP
requests carried the bearer header. A missing session matched `isCodexSessionNotFound`.

**Go for transport integration; no-go for the worker policy as currently written.** The
adapter combines requested MCP servers with inherited Codex config. This machine has no
globally configured MCP servers, so strict isolation was not validated live and cannot be
assumed. The adapter exposes approval presets, not separate sandbox and approval options;
the requested “edits allowed, commands ask” policy has no exact preset. Windows sandbox
enforcement outside the workspace was not established by this spike.

## Changed files

- `docs/pm/reports/ORCH-26.md` — evidence and recommendation.

Scratch scripts and output are in `%TEMP%\emdash-spikes\ORCH-26\` (`harness.cjs`,
`mcp.cjs`, and captured output). They are outside the repository. The harness starts
`node_modules/electron/dist/electron.exe` with `ELECTRON_RUN_AS_NODE=1`, the installed
`@agentclientprotocol/codex-acp/dist/index.js`, and `CODEX_PATH` pointing to the
installed native `codex.exe`. It uses the repository's ACP and MCP SDKs. No API key was set.

## Verification

Commands below ran in the worktree unless a scratch path is shown. The evidence projector
reads the captured harness output and emits selected fields. Its complete output below is
verbatim; later excerpts show the relevant parts in context. No bearer token value is
included. Duplicate completed tool notifications in the projection arise from replay on load.

```text
> node "$env:TEMP\emdash-spikes\ORCH-26\evidence.cjs"
HTTP_INIT {"protocolVersion":1,"agentInfo":{"name":"@agentclientprotocol/codex-acp","title":"Codex","version":"1.13.1"},"loadSession":true,"mcpCapabilities":{"acp":false,"http":true,"sse":false}}
HTTP_OPTIONS [{"id":"mode","category":"mode","type":"select","values":["read-only","agent","agent-full-access"]},{"id":"collaboration_mode","category":"collaboration_mode","type":"select","values":["default","plan"]},{"id":"model","category":"model","type":"select","values":["gpt-6.1-sol","gpt-6-astra","gpt-6-sol","gpt-6-luna","gpt-5.6-sol","gpt-5.6-terra","gpt-5.6-luna"]},{"id":"reasoning_effort","category":"thought_level","type":"select","values":["low","medium","high","xhigh","max","ultra"]},{"id":"fast-mode","category":"model_config","type":"boolean","values":[]}]
HTTP_TOOL_CALLS [{"name":"orch26_http","summary":"orch26 probe","result":"ok"},{"name":"orch26_http","summary":"orch26 probe","result":"ok"},{"name":"orch26_http","summary":"orch26 loaded","result":"ok"}]
HTTP_CALLS "http request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp response status=202\nhttp request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp submit_result orch26 probe\nhttp response status=200\nhttp request auth=true method=POST\nhttp submit_result orch26 loaded\nhttp response status=200\n"
HTTP_MISSING {"code":-32603,"message":"Internal error","data":{"details":"no rollout found for thread id 00000000-0000-4000-8000-000000000000"}}
HTTP_STDERR ""
STDIO_INIT {"protocolVersion":1,"agentInfo":{"name":"@agentclientprotocol/codex-acp","title":"Codex","version":"1.13.1"},"loadSession":true,"mcpCapabilities":{"acp":false,"http":true,"sse":false}}
STDIO_OPTIONS [{"id":"mode","category":"mode","type":"select","values":["read-only","agent","agent-full-access"]},{"id":"collaboration_mode","category":"collaboration_mode","type":"select","values":["default","plan"]},{"id":"model","category":"model","type":"select","values":["gpt-6.1-sol","gpt-6-astra","gpt-6-sol","gpt-6-luna","gpt-5.6-sol","gpt-5.6-terra","gpt-5.6-luna"]},{"id":"reasoning_effort","category":"thought_level","type":"select","values":["low","medium","high","xhigh","max","ultra"]},{"id":"fast-mode","category":"model_config","type":"boolean","values":[]}]
STDIO_TOOL_CALLS [{"name":"orch26_stdio","summary":"orch26 probe","result":"ok"},{"name":"orch26_stdio","summary":"orch26 probe","result":"ok"},{"name":"orch26_stdio","summary":"orch26 loaded","result":"ok"}]
STDIO_CALLS "stdio submit_result orch26 probe\nstdio submit_result orch26 loaded\n"
STDIO_MISSING {"code":-32603,"message":"Internal error","data":{"details":"no rollout found for thread id 00000000-0000-4000-8000-000000000000"}}
STDIO_STDERR ""
READ_ONLY_PERMISSION [["allow_once","accept_execpolicy_amendment","cancel"]]
READ_ONLY_COMMAND [{"status":"failed","output":"","exitCode":null}]
READ_ONLY_STOP "cancelled"
AGENT_PERMISSION []
AGENT_COMMAND [{"status":"completed","output":"sandbox-ok\r\n","exitCode":0}]
AGENT_STOP "end_turn"
```

### Setup and versions

```text
> pnpm install --frozen-lockfile
Lockfile is up to date, resolution step is skipped
Done in 41.6s using pnpm v10.28.2
exit code: 0

> codex --version
codex-cli 0.163.0

> codex login status
Logged in using ChatGPT

> node -p "require('./node_modules/@agentclientprotocol/codex-acp/package.json').version"
1.13.1
```

`pnpm install` briefly changed `package.json` line endings in the working tree without a
content diff; `git restore --worktree -- package.json` returned it to the initial clean
state. The lockfile was not changed.

### 1. ACP initialize

The harness called `initialize` with protocol version 1 and Emdash-like client capabilities.

```json
{"protocolVersion":1,"agentInfo":{"name":"@agentclientprotocol/codex-acp","title":"Codex","version":"1.13.1"},"agentCapabilities":{"auth":{"logout":{}},"providers":{},"loadSession":true,"promptCapabilities":{"embeddedContext":true,"image":true},"sessionCapabilities":{"resume":{},"list":{},"close":{},"delete":{},"fork":{},"additionalDirectories":{},"subagents":{}},"mcpCapabilities":{"acp":false,"http":true,"sse":false},"_meta":{"authStatus":{}}}}
```

### 2. HTTP MCP and bearer header

The harness hosted a stateless Streamable HTTP MCP server on `127.0.0.1:<ephemeral port>`
and passed `Authorization: Bearer <random 256-bit token>` in `session/new.mcpServers`.

```text
> $env:ORCH26_CODEX=(Get-ChildItem C:\Users\colin\AppData\Roaming\npm\node_modules\@openai\codex -Recurse -Filter codex.exe | Select-Object -First 1 -ExpandProperty FullName)
> node "$env:TEMP\emdash-spikes\ORCH-26\harness.cjs" (Get-Location).Path http
UPDATE {"title":"mcp.orch26_http.submit_result","kind":"execute","status":"in_progress","rawInput":{"server":"orch26_http","tool":"submit_result","arguments":{"summary":"orch26 probe"}},"_meta":{"is_mcp_tool_call":true},"sessionUpdate":"tool_call"}
UPDATE {"status":"completed","rawInput":{"server":"orch26_http","tool":"submit_result","arguments":{"summary":"orch26 probe"}},"rawOutput":{"result":{"content":[{"type":"text","text":"ok"}],"structuredContent":null,"_meta":null},"error":null},"sessionUpdate":"tool_call_update"}
ANSWER "OK"
CALLS "http request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp response status=202\nhttp request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp submit_result orch26 probe\nhttp response status=200\n"
STDERR ""
```

The server records only whether the header matched, not the token. The token did not appear
in captured harness output or adapter stderr. Other Codex diagnostic logs were not inspected,
so absence there could not be determined. The first HTTP test server reused one stateless
transport and returned 500 after initialization; creating a fresh server and transport per
request yielded the successful evidence above. This was a harness error, not an adapter
failure.

### 3. Stdio MCP through Electron

The stdio server's ACP registration used the Electron executable as `command`, the scratch
`mcp.cjs` as its argument, and `ELECTRON_RUN_AS_NODE=1` in its environment.

```text
> node "$env:TEMP\emdash-spikes\ORCH-26\harness.cjs" (Get-Location).Path stdio
UPDATE {"title":"mcp.orch26_stdio.submit_result","kind":"execute","status":"in_progress","rawInput":{"server":"orch26_stdio","tool":"submit_result","arguments":{"summary":"orch26 probe"}},"_meta":{"is_mcp_tool_call":true},"sessionUpdate":"tool_call"}
ANSWER "OK"
CALLS "stdio submit_result orch26 probe\n"
CALLS_AFTER_LOAD "stdio submit_result orch26 probe\nstdio submit_result orch26 loaded\n"
STDERR ""
```

### 4. Inherited MCP servers and isolation

```text
> codex mcp list --json
[]
```

No global MCP names were available on this machine. The installed adapter's
`createSessionConfig` starts with the effective Codex config and adds requested servers
under `mcp_servers`; it does not replace inherited servers. It also checks existing config
names for deduplication. This is source evidence that global servers can be inherited, but
their actual appearance in this account's session could not be tested. There is no
per-session ACP option observed to suppress inherited servers. An isolated Codex config
home with subscription authentication preserved, or an upstream adapter change that
constructs an isolated MCP set, needs a separate validation before strict MCP policy ships.

### 5. Session load and missing session

The harness passed the same `mcpServers` to `loadSession`, then prompted again.

```text
LOAD_PROMPT_RESULT {"stopReason":"end_turn"}
CALLS_AFTER_LOAD "http request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp response status=202\nhttp request auth=true method=POST\nhttp response status=200\nhttp request auth=true method=POST\nhttp submit_result orch26 probe\nhttp response status=200\nhttp request auth=true method=POST\nhttp submit_result orch26 loaded\nhttp response status=200\n"
MISSING_SESSION {"code":-32603,"message":"Internal error","data":{"details":"no rollout found for thread id 00000000-0000-4000-8000-000000000000"}}
```

The same `MISSING_SESSION` response occurred in the stdio run. Its code and exact `data.details`
match `packages/plugins/src/agents/impl/codex/acp-errors.ts`.

### 6. Native config options

`session/new.configOptions` reported the following ids, categories, and selectable values:

```text
mode|mode|read-only,agent,agent-full-access
collaboration_mode|collaboration_mode|default,plan
model|model|gpt-6.1-sol,gpt-6-astra,gpt-6-sol,gpt-6-luna,gpt-5.6-sol,gpt-5.6-terra,gpt-5.6-luna
reasoning_effort|thought_level|low,medium,high,xhigh,max,ultra
fast-mode|model_config|boolean (false/true)
```

`mode` is the permission/sandbox preset. The installed adapter maps both `read-only`
(human reviewer) and `agent` (automatic reviewer) to `workspace-write` plus `on-request`;
`agent-full-access` maps to `danger-full-access` plus `never`. The labels do not describe
independent edit and command permissions. The `model`, `reasoning_effort`, and
`collaboration_mode` ids are direct. The available effort values depend on the model;
GPT-6 Luna offers through `max`, not `ultra`. The successful prompt runs used GPT-6 Luna.

### 7. Model catalog comparison

```text
> (codex debug models | ConvertFrom-Json).models | Where-Object visibility -eq 'list' | ForEach-Object slug
gpt-6.1-sol
gpt-6-astra
gpt-6-sol
gpt-6-luna
gpt-5.6-sol
gpt-5.6-terra
gpt-5.6-luna
```

The seven `model` option values exactly match the seven `visibility: "list"` slugs.

### 8. Windows permission and sandbox behavior

The harness selected `mode=read-only` and asked for `cmd /c echo sandbox-ok` in the scratch
working directory. It received an ACP permission request even for this harmless command;
the harness returned `cancelled` rather than authorizing it.

```text
PERMISSION {"options":[{"optionId":"allow_once","name":"Yes, proceed","kind":"allow_once"},{"optionId":"accept_execpolicy_amendment","name":"Yes, and don't ask again for commands that start with `cmd /c echo sandbox-ok`","kind":"allow_always"},{"optionId":"cancel","name":"No, and tell Codex what to do differently","kind":"reject_once"}]}
UPDATE {"status":"failed","rawOutput":{"formatted_output":"","exit_code":null},"sessionUpdate":"tool_call_update"}
SANDBOX_PROMPT {"stopReason":"cancelled"}
```

With `mode=agent`, automatic review approved the same command and it ran on native Windows:

```text
UPDATE {"status":"completed","rawOutput":{"formatted_output":"sandbox-ok\r\n","exit_code":0},"sessionUpdate":"tool_call_update"}
SANDBOX_PROMPT {"stopReason":"end_turn"}
ANSWER "sandbox-ok"
```

These observations show the adapter neither refused Windows execution nor silently bypassed
its ACP approval flow. They do **not** prove Windows filesystem sandbox enforcement or how
an outside-workspace write would be handled. No outside-workspace write was attempted.
`codex sandbox -C <scratch> cmd /c echo sandbox-ok` could not serve as a direct test:
the CLI returned `the following required arguments were not provided: --permission-profile <NAME>`.
`codex sandbox -P default -C <scratch> ...` returned
`Error: default_permissions requires a [permissions] table`.

### 9. Recommendation

Use HTTP MCP for Codex when `mcpCapabilities.http` is true; retain the Electron stdio shim
as a validated fallback. Always resupply servers on `session/load`. Surface and set the
adapter's actual `model`, `reasoning_effort`, `mode`, and `collaboration_mode` options.
For a human approval path on Windows, `read-only` gives ACP permission requests, including
for a harmless command. `agent` uses automatic review. Neither is a verified exact
“edits allowed, commands ask” mode.

Block strict orchestrator MCP isolation until a test with an inherited global MCP server
and an isolated Codex configuration proves that workers see only the allowlisted servers.
Block any claim of enforced Windows workspace boundaries until a scratch-only permission
test distinguishes restricted sandbox execution from approval-only behavior. Do not treat
`session/new.mcpServers` as an exclusive list with this adapter.

## Tests

No repository tests were added or changed. The throwaway harness completed HTTP, stdio,
load, missing-session, config-catalog, and Windows approval probes. No product code changed,
so the full merge gate was not run.

## How to check it by hand

Run the two scratch harness modes (`http` and `stdio`) with the repository path argument
and `ORCH26_CODEX` set to the native `codex.exe`. Check the `CALLS` and `CALLS_AFTER_LOAD`
lines. Run `codex mcp list --json` and `codex debug models` for the local account catalog.
The scratch harness is deliberately temporary and is not committed.

## Deviations from the brief

None. The report is the only repository file changed. Final `git status --short` was:

```text
?? docs/pm/reports/ORCH-26.md
```

## Blockers and open questions

- Can the Codex adapter be given a subscription-authenticated, isolated config home without
  exposing global MCP servers? This account's global MCP list is empty, so no live negative
  isolation test was possible.
- Does native Windows `workspace-write` enforce filesystem boundaries for Codex ACP on this
  host, and can approval be configured separately from the three exposed presets?

## Suggested follow-ups

- Test an inherited MCP server with a disposable Codex configuration and verify the
  proposed isolation path before implementing strict MCP policy.
- Add a scratch-only Windows sandbox boundary test and document the exact approval behavior
  required for worker sessions.
