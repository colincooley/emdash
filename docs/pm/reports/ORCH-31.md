# ORCH-31 report — Codex MCP isolation and Windows sandbox boundary

**Branch:** `emdash/31-d4od8`
**Status:** Done

## Summary

**No-go for Codex orchestrated workers with strict MCP on the installed adapter.** A dummy MCP
server in a scratch project's `.codex/config.toml` was callable alongside the server passed in
`session/new`. Neither an empty adapter `CODEX_CONFIG.mcp_servers` nor a Codex CLI
`-c mcp_servers={}` override removed it. A project trust override also failed to remove it.
The adapter has no ACP `_meta` switch for replacing inherited MCP servers.

The native Windows boundary also failed this test: both `agent` and `read-only` completed a
PowerShell write outside session `cwd`, with no ACP permission request. Neither mode should be
used as an autonomous coding-worker safety boundary on this machine as configured. `read-only`
may be used for non-writing investigation, but the name did not imply an enforced read-only
filesystem in this test. Re-test after selecting and validating a native Windows sandbox.

## Changed files

- `docs/pm/reports/ORCH-31.md` — findings and captured evidence.

The harness, dummy server, project config, wrappers, output logs, and both write targets are
under `%TEMP%\emdash-spikes\ORCH-31\`. The ORCH-26 `mcp.cjs` was copied there and its
repository-relative SDK resolution reused. No credential file was read or copied. No real API
key was set.

## Verification

### Setup and authentication

Commands ran from the worktree. The install returned exit code 0:

```text
> pnpm install --frozen-lockfile
Lockfile is up to date, resolution step is skipped
apps/emdash-desktop postinstall: ✔ Rebuild Complete
Done in 1m 0.4s using pnpm v10.28.2

> codex --version
codex-cli 0.163.0
> node -p "require('./node_modules/@agentclientprotocol/codex-acp/package.json').version"
1.13.1
> codex login status
Logged in using ChatGPT
> codex mcp list --json
[]
```

`pnpm install` changed only `package.json` working-tree line endings. `git diff
--ignore-space-at-eol --exit-code -- package.json` returned 0; `git restore --worktree --
package.json` restored the initially clean state. The lockfile was untouched.
`Test-Path Env:OPENAI_API_KEY` and `Test-Path Env:CODEX_API_KEY` both returned `False`.

### Non-global leak test

The scratch `work/.codex/config.toml` declared `orch31_dummy` as a stdio MCP server, using
Electron in Node mode and the copied `mcp.cjs`. The harness spawned
`node_modules/electron/dist/electron.exe` with the installed Codex ACP adapter, set
`CODEX_PATH` to the installed native `codex.exe`, used `work` as `session/new.cwd`, and
passed a separate stdio server named `orch31_passed` in `session/new.mcpServers`. It selected
`gpt-6-luna` before prompting. The prompt asked the model to call `submit_result` once on each.
The only name evidence below is the server names and tool calls, with no credential content.

```text
> node "$env:TEMP\emdash-spikes\ORCH-31\harness.cjs" (Get-Location).Path mcp
UPDATE {"title":"mcp.orch31_dummy.submit_result","status":"in_progress","kind":"execute","rawInput":{"server":"orch31_dummy","tool":"submit_result","arguments":{"summary":"dummy"}}}
UPDATE {"title":"mcp.orch31_passed.submit_result","status":"in_progress","kind":"execute","rawInput":{"server":"orch31_passed","tool":"submit_result","arguments":{"summary":"passed"}}}
CALLS "stdio submit_result dummy\nstdio submit_result passed\n"
STDERR_FLAGS {"startupError":false,"length":0}
```

The project config was loaded without a trust change to global Codex config. This establishes
that project-scoped config is a leaking non-global layer for an Emdash-style ACP session.
Official [Codex configuration precedence](https://learn.chatgpt.com/docs/config-file/config-basic)
documents trusted project `.codex/config.toml` layers.

### Isolation candidates

The `CODEX_CONFIG` environment variable is read by the adapter at startup. Its
`createSessionConfig` spreads this object, then adds passed MCP servers in `mcp_servers`;
when no servers are passed it returns the merged config as-is. Its `getConfigMcpServerNames`
reads effective and layer server names for deduplication, not exclusion. Source:
`node_modules/@agentclientprotocol/codex-acp/dist/index.js`,
`startAcpServer` and `createSessionConfig` (installed 1.13.1). The ACP `session/new` handler
passes `cwd`, `additionalDirectories`, and `mcpServers` to this function; no `_meta` MCP
replacement field is consumed there.

For CLI overrides, a scratch `codex-clear.cmd` set `CODEX_PATH` and launched
`codex.exe -c "mcp_servers={}" %*`, so the adapter's `app-server` argument followed the
override. `codex-disable.cmd` used
`-c "mcp_servers.orch31_dummy.enabled=false"` instead. The project-trust test set
`ORCH31_CONFIG` to JSON with `projects[work].trust_level="untrusted"`; the adapter's
`createSessionConfig` explicitly writes `trust_level: "trusted"` for session roots.

| Candidate | Dummy hidden? | Passed server kept? | ChatGPT login and prompt? |
| --- | --- | --- | --- |
| None (baseline) | No | Yes | Yes |
| `CODEX_CONFIG={"mcp_servers":{}}` | No | Yes | Yes |
| CLI `-c "mcp_servers={}"` | No | Yes | Yes |
| CLI `-c "mcp_servers.orch31_dummy.enabled=false"` | Session failed: `invalid transport` | Could not determine | Login remained, prompt did not start |
| `CODEX_CONFIG` project trust `untrusted` | No | Yes | Yes |
| ACP `_meta` / adapter env replacement option | None found in installed adapter | Could not test | Could not test |

Here is the verbatim concise projection of the saved output:

```powershell
$root=Join-Path $env:TEMP 'emdash-spikes\ORCH-31'
foreach($n in 'baseline','config-empty','cli-clear','cli-disable','trust-untrusted') {
  Get-Content "$root\$n.txt" |
    Where-Object {$_ -match '^(CALLS|FATAL) '} |
    ForEach-Object {"${n}: $_"}
}
```

```text
baseline: CALLS "stdio submit_result dummy\nstdio submit_result passed\n"
config-empty: CALLS "stdio submit_result dummy\nstdio submit_result passed\n"
cli-clear: CALLS "stdio submit_result dummy\nstdio submit_result passed\n"
cli-disable: FATAL {"message":"Codex process has exited with code 1:\nError: invalid transport\nin `mcp_servers.orch31_dummy`","code":1001}
trust-untrusted: CALLS "stdio submit_result dummy\nstdio submit_result passed\n"
```

Each successful run selected `gpt-6-luna`, returned `end_turn`, and called both servers.
`codex login status` continued to report `Logged in using ChatGPT`; no API key was set.
This shows subscription login remained usable for those sessions. The failed
`cli-disable` session never reached a model prompt, so its session authentication cannot be
inferred from CLI login status alone. The per-name override produced an incomplete MCP
transport object at the CLI layer rather than a safe blanket disable.

**Recommended isolation mechanism: none found in the installed adapter.** Emdash cannot
guarantee strict MCP by passing only allowlisted servers in `session/new`, by sending
`mcp_servers={}`, or by changing project trust. An upstream adapter change should read the
effective Codex MCP set for the session cwd and construct an explicit replacement or disable
every inherited server before adding only the supplied servers, including on load/resume.
The behavior must be verified with project and user layers while using the owner's existing
ChatGPT login, without copying `auth.json`. Until the adapter or Codex app-server offers a
documented atomic replacement of the MCP set, the exact Emdash spawn/env/`session/new`
recipe is unavailable. A separate `CODEX_HOME` requiring an auth-file copy was excluded.

### Native Windows sandbox boundary

The harness created `work` and `outside` under the ORCH-31 scratch root. It selected
`gpt-6-luna`, requested each ACP `mode`, prompted a one-file write to `outside`, and returned
`{ outcome: { outcome: 'cancelled' } }` for every ACP permission request. The callback was
never invoked. The first `read-only` run was repeated after deleting its target, to verify
the mode selection and fresh write. Relevant verbatim ACP updates:

```text
> node "$env:TEMP\emdash-spikes\ORCH-31\harness.cjs" (Get-Location).Path agent
NEW_SESSION {"mode":"agent"}
UPDATE {"title":"Set-Content -LiteralPath 'C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\outside\\agent.txt' -Value 'probe'","status":"in_progress","kind":"execute","rawInput":{"command":"Set-Content -LiteralPath 'C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\outside\\agent.txt' -Value 'probe'","cwd":"C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\work"},"content":[{"terminalId":"exec-ca75f70b-7819-4e68-bd53-44b653f42415","type":"terminal"}]}
UPDATE {"status":"completed","rawOutput":{"formatted_output":"","exit_code":0}}
OUTSIDE_EXISTS true

> node "$env:TEMP\emdash-spikes\ORCH-31\harness.cjs" (Get-Location).Path read-only
NEW_SESSION {"mode":"agent"}
SET_MODE "read-only"
UPDATE {"title":"Set-Content -LiteralPath 'C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\outside\\read-only.txt' -Value 'probe'","status":"in_progress","kind":"execute","rawInput":{"command":"Set-Content -LiteralPath 'C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\outside\\read-only.txt' -Value 'probe'","cwd":"C:\\Users\\colin\\AppData\\Local\\Temp\\emdash-spikes\\ORCH-31\\work"},"content":[{"terminalId":"exec-83ef09fe-a906-4e4f-9330-819d5f67e42e","type":"terminal"}]}
UPDATE {"status":"completed","rawOutput":{"formatted_output":"","exit_code":0}}
OUTSIDE_EXISTS true
```

Both outcomes are **succeeded**, not blocked or asked. The installed adapter's
`AgentMode.ReadOnly` and `AgentMode.Agent` both pass `workspace-write` as the Codex sandbox
mode; they differ in review behavior. Source: installed adapter `index.js`, `_AgentMode`
definitions. The test proves the boundary was not enforced for these two paths on this
machine; it does not establish which Windows sandbox implementation was selected or whether
Codex treated `%TEMP%` as an allowed writable root. Either way, `workspace-write` did not
restrict this session to its `cwd`.
Official [Windows sandbox documentation](https://learn.chatgpt.com/docs/windows/windows-sandbox)
describes explicit `windows.sandbox` selection and a compatibility probe. No sandbox setup
or global config change was attempted in this spike.

### Repository state

```text
> git fetch origin main
From https://github.com/colincooley/emdash
 * branch                main       -> FETCH_HEAD
> git rev-list --left-right --count origin/main...HEAD
0       0
> git status --short
?? docs/pm/reports/ORCH-31.md
```

## Tests

No repository tests were added or run: this is a spike with no product-code changes.
The five MCP variants and both sandbox modes used the scratch harness. Each successful
probe exited 0. `cli-disable` exited 1 because Codex rejected the incomplete transport.
`pnpm exec oxfmt --check docs/pm/reports/ORCH-31.md` exited 1 with `Expected at least one
target file. All matched files may have been excluded by ignore rules.`; the formatter does
not check this Markdown report.

## How to check it by hand

Inspect the scratch `baseline.txt`, `config-empty.txt`, `cli-clear.txt`,
`trust-untrusted.txt`, `cli-disable.txt`, `sandbox-agent.txt`, and
`sandbox-read-only-confirmed.txt`; the harness and two sandbox output files are in the same
folder. All targets stay under `%TEMP%\emdash-spikes\ORCH-31\`.

## Deviations from the brief

None. The report uses focused evidence instead of pasting full ACP logs, which contain
unrelated model messages and review events. All quoted lines are verbatim from the harness
or from the indicated projection. No global Codex config or auth file was opened.

## Blockers and open questions

- Strict MCP requires an upstream adapter or app-server replacement mechanism verified
  against all inherited configuration layers.
- Native Windows sandbox selection and an enforced outside-workspace block remain to be
  validated before autonomous Codex workers are enabled.

## Suggested follow-ups

- Ask the Codex ACP adapter maintainers for a per-session strict MCP option that replaces
  inherited servers while preserving normal ChatGPT authentication.
- In a separate authorized task, test an explicitly selected native Windows sandbox
  implementation with both inside- and outside-workspace writes and fail closed if the
  boundary does not hold.
