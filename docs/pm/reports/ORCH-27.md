# ORCH-27 report — Spike: subscription-only auth, billing env vars, and claude.ai connectors

**Branch:** emdash/27-5h6cl
**Status:** Done

Tested with Claude Code 2.1.307, codex-cli 0.163.0, `@anthropic-ai/claude-agent-sdk` (latest from npm,
installed in the scratch folder only; the repo pins 0.3.280 and has no `node_modules` in this
worktree), Windows 11. Account: Claude Pro, ChatGPT Plus (plan names only; no identifiers below).

## Summary

- Both CLIs can be held to subscription logins, but **Emdash today does not do it**: the allowlist
  forwards ~15 billing/backend-routing variables, and the auth probes treat the mere presence of
  `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` as "authenticated".
- Claude: an API key (even an invalid one) **takes priority over the subscription login** and
  silently disables claude.ai connectors. `claude auth status` JSON `authMethod` distinguishes the
  cases; `"claude.ai"` + `subscriptionType` proves a subscription login.
- Codex: `OPENAI_API_KEY` and `OPENAI_BASE_URL` were **ignored** while logged in with ChatGPT, but
  `CODEX_API_KEY` **takes priority** (and is not in the allowlist today). `codex login status`
  prints `Logged in using ChatGPT` for a subscription login.
- claude.ai connectors: `ENABLE_CLAUDEAI_MCP_SERVERS=false` removes them; so does
  `--strict-mcp-config` on its own (verified, 4 connectors -> 0). The existing Claude usage probe
  already sets the env var.
- Usage probes work, are read-only, and return in ~0.8–1.1 s. Suitable for a quota gate with caveats
  (see 5).

## 1. Billing-routing variables in the allowlist

Source: `packages/core/src/primitives/agent-env/api/index.ts` (`AGENT_ENV_VARS`). Listed: variables
that can change which account pays, which backend is called, or which login store is read. The
claude/codex columns say whether the behaviour was observed (O) or is from reading/knowledge (R).

| Variable | Effect | Evidence |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | Claude uses API-key billing; wins over subscription login | O (section 3) |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token auth; wins over login | O (section 3) |
| `ANTHROPIC_BASE_URL` | Redirects all Claude API traffic to another host (billing proxy, gateway, or an untrusted server that would receive the credential). `auth status` still reports `claude.ai` | O (auth status) / R (traffic) |
| `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX` | Switch Claude to AWS / Google billing | O (`authMethod: third_party`) |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_PROFILE`, `AWS_REGION`, `AWS_DEFAULT_REGION` | Credentials/region for Bedrock | R |
| `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_VERTEX_BASE_URL` | Credentials/project for Vertex | R |
| `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_{HAIKU,SONNET,OPUS}_MODEL`, `ANTHROPIC_SMALL_FAST_MODEL`, `CLAUDE_CODE_SUBAGENT_MODEL` | Change model (not billing path, but can pick a model the plan lacks or a Bedrock/Vertex ID) | R |
| `CLAUDE_CONFIG_DIR` | Reads a different Claude login/settings store | R |
| `OPENAI_API_KEY` | Ignored by Codex when logged in via ChatGPT | O |
| `OPENAI_BASE_URL`, `OPENAI_ORGANIZATION`, `OPENAI_PROJECT`, `OPENAI_MODEL` | Ignored by Codex when logged in via ChatGPT (BASE_URL tested) | O (BASE_URL) |
| `AZURE_OPENAI_API_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_KEY` | Azure OpenAI backend for tools that read them | R |
| `CODEX_HOME` | Reads a different Codex login store | R |
| `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` | Route traffic through a proxy; not billing, but should be an explicit policy decision | R |
| Other providers' keys (`OPENROUTER_*`, `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `GITHUB_TOKEN`, `GH_TOKEN`, etc.) | Irrelevant to Claude/Codex sessions; are paid-API keys for other agents | R |

**Not in the allowlist but dangerous if added or injected by another layer** (project env, settings):
`CODEX_API_KEY` (O: takes priority over ChatGPT login), `CLAUDE_CODE_OAUTH_TOKEN` (O: switches
`authMethod` to `oauth_token`), `ANTHROPIC_FOUNDRY_*` / `CLAUDE_CODE_USE_FOUNDRY` (R), and the
`apiKeyHelper` setting in Claude settings files (R). I did not trace whether project env or other
layers can add these after the allowlist runs; that belongs to ORCH-06.

## 2. Auth status outputs (redacted)

`claude auth status` (subscription login):

```json
{ "loggedIn": true, "authMethod": "claude.ai", "apiProvider": "firstParty",
  "email": "<email>", "orgId": "<uuid>", "orgName": "<redacted>", "subscriptionType": "pro" }
```

`claude auth status --text`: `Login method: Claude Pro account`.

`codex login status`: `Logged in using ChatGPT` (exit 0). Text only, no JSON mode. Unchanged with a
dummy `OPENAI_API_KEY` set.

**Field that proves subscription login:**

- Claude: `authMethod === "claude.ai"` AND `apiProvider === "firstParty"` AND a non-empty
  `subscriptionType`. Other values seen: `api_key` (with `apiKeySource: "ANTHROPIC_API_KEY"`),
  `oauth_token` (from `ANTHROPIC_AUTH_TOKEN` or `CLAUDE_CODE_OAUTH_TOKEN`), `third_party` (with
  `apiProvider: bedrock|vertex`).
- Codex: the `codex login status` text must match `Logged in using ChatGPT`. A key login prints a
  different string (not tested: would need a real key). The app-server `account/read` returns
  `account.type === "chatgpt"` (and `planType`), which is a structured alternative.

## 3. Precedence results (dummy keys, no billing)

Run from a scratch folder, `--model haiku` for Claude. All invalid values; no real key used.

| Command | Result |
| --- | --- |
| `ANTHROPIC_API_KEY=sk-ant-invalid-test claude -p "say ok"` | `⚠ claude.ai connectors are disabled because ANTHROPIC_API_KEY or another auth source is set and takes precedence over your claude.ai login · Unset it to load your organization's connectors` then `Invalid API key · Fix external API key`, exit 1 |
| `ANTHROPIC_AUTH_TOKEN=invalid-test claude -p "say ok"` | same warning, then `Failed to authenticate. API Error: 401 Invalid bearer token`, exit 1 |
| `ANTHROPIC_API_KEY=… claude auth status` | `"authMethod": "api_key"`, `"apiKeySource": "ANTHROPIC_API_KEY"`, no `subscriptionType`, still `"loggedIn": true` |
| `OPENAI_API_KEY=sk-invalid-test codex exec "say ok"` | Succeeded (`ok`). Key ignored. Used ~2.5k subscription tokens |
| `OPENAI_BASE_URL=http://127.0.0.1:9/v1 codex exec "say ok"` | Succeeded (`ok`). Base URL ignored |
| `CODEX_API_KEY=sk-invalid-test codex exec "say ok"` | `ERROR … unexpected status 401 Unauthorized: Incorrect API key provided: sk-inval***test …`, exit 1. Key took priority |

Conclusion: for Claude an inherited key silently wins and is billed; for Codex only `CODEX_API_KEY`
matters. Note also that the Claude warning itself says connectors are disabled when a key is set, so a
key leak changes both billing and tool surface.

Side note: the Codex runs printed `hook: SessionStart Failed` / `hook: Stop Failed`; those come from
the user's own Codex hooks and are unrelated.

## 4. claude.ai connectors

Measured with `claude -p "say ok" --model haiku --output-format stream-json --verbose`, reading the
`system/init` event (server names and status only):

| Run | `mcp_servers` | MCP tools |
| --- | --- | --- |
| Baseline | 4 claude.ai connectors, all `connected` | 121 |
| `ENABLE_CLAUDEAI_MCP_SERVERS=false` | none | 0 |
| `--strict-mcp-config` only | none | 0 |
| Both | none | 0 |

- Switch: env `ENABLE_CLAUDEAI_MCP_SERVERS=false`. The existing usage probe
  (`probeClaudeUsage`) already sets it, so the repo already relies on it.
- `--strict-mcp-config` makes Claude load only servers passed via `--mcp-config`; with none passed it
  also dropped the account connectors. For ACP, the equivalent is the SDK option `strictMcpConfig`
  (used in `claude/usage.ts`). Whether the ACP adapter forwards it is not verified here.
- Recommend both: the env var as the explicit switch, strict MCP config as the allowlist mechanism for
  the orchestrator's own MCP server.
- Not tested: the setting through `@agentclientprotocol/claude-agent-acp` (adapter not installed in
  this worktree). ORCH-06 should re-run the init-event check through the real ACP path.

## 5. Usage probes

Run by replaying the probe protocol from the scratch folder (the repo's `node_modules` are absent).
Read-only: no prompts are sent, no tokens consumed.

**Claude** (`probeClaudeUsage`, SDK `initializationResult` + experimental `usage` control call):

- Returned `subscriptionType`, `apiProvider: firstParty`, `rate_limits_available: true`, windows
  `five_hour` (50% used) and `seven_day` (39% used) with reset timestamps, plus `extra_usage`
  (`utilization: null`), `spend`, `seven_day_breakdown` (all empty here). Init ~0.8 s, usage ~1.1 s total.
- With a dummy `ANTHROPIC_API_KEY` the probe gets `apiKeySource: ANTHROPIC_API_KEY`, no subscription
  type, `rate_limits_available: false`; the code maps that to `unsupported`. So a leaked key makes
  the gate go dark rather than wrong; treat `unsupported` as "block".
- The usage method name is `…_EXPERIMENTAL_MAY_CHANGE_DO_NOT_RELY_ON_THIS_API_YET`, so it can break
  on SDK upgrades. The `catch` returns `unavailable`.

**Codex** (`probeCodexUsage`, `codex app-server` JSON-RPC `account/read` + `account/rateLimits/read`):

- Returned `account.type: chatgpt`, `planType: plus`, primary window 300 min (13% used) and secondary
  10080 min (2% used), reset times in epoch seconds, one bucket `codex`. ~0.8 s total.
- Uses the experimental API capability; same fragility concern.

**Quota gate verdict: suitable as a coarse gate.** Percent-used per window plus reset time is exactly
what a headroom gate needs, and latency is low. Caveats:

1. Data is a point-in-time snapshot; 5 parallel sessions can burn the 5-hour window faster than the
   probe interval, so the gate needs a margin and re-checks before each dispatch.
2. Both APIs are experimental/unversioned; the gate must fail closed (`unavailable`, `unsupported`,
   `signed-out` all block dispatch) and the failure must be visible.
3. Claude's `extra_usage` / `spend` fields indicate pay-per-use overage exists on some plans. A
   subscription-only policy should also require extra usage to be off or at least not draw from it;
   whether overage can be disabled per session is **could not determine**.
4. Claude Pro showed a 5-hour window already at 50%; with the owner's plan the practical concurrency
   is a product question for the owner, not a probe limitation.

## 6. Specs for ORCH-06

### 6.1 Env scrub list

Applied to the final resolved launch env (after allowlist, project env and user overrides; the scrub
must run last). Match names case-insensitively on Windows. Remove for **all** orchestrated sessions:

- Anthropic: `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`,
  `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`,
  `CLAUDE_CODE_USE_FOUNDRY`, and any `ANTHROPIC_FOUNDRY_*`, `ANTHROPIC_BEDROCK_*`, `ANTHROPIC_VERTEX_*`.
- Cloud creds used by those backends: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
  `AWS_SESSION_TOKEN`, `AWS_PROFILE`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_VERTEX_BASE_URL`.
  Optional: `AWS_REGION`, `GOOGLE_CLOUD_*` (harmless without the above).
- OpenAI/Codex: `CODEX_API_KEY`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_ORGANIZATION`,
  `OPENAI_PROJECT`, `AZURE_OPENAI_*`.
- Login-store redirects: `CLAUDE_CONFIG_DIR`, `CODEX_HOME`, unless the owner has set them explicitly
  in Emdash settings for this purpose (default: scrub, so the default login is used).
- Model overrides: `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_*_MODEL`, `ANTHROPIC_SMALL_FAST_MODEL`,
  `CLAUDE_CODE_SUBAGENT_MODEL`, `OPENAI_MODEL`. Model choice should come from the node/session config.
- Proxy variables: leave as-is by default (corporate networks may need them); flag in settings.
- Add a test that fails if a new key-like variable (`*_API_KEY`, `*_AUTH_TOKEN`, `*_BASE_URL`) is added
  to `AGENT_ENV_VARS` without being classified as scrubbed or explicitly allowed.

Also do not rely on env only: the Claude `apiKeyHelper` setting and `.claude/settings*.json` `env`
blocks can reintroduce keys. Preflight (6.2) is the backstop.

### 6.2 Auth preflight algorithm

Per provider, run after the scrub, with the same env the session will get:

Claude:
1. Run `claude auth status` with the scrubbed env, 5 s timeout. Parse stdout as JSON; non-JSON,
   timeout or non-zero with no JSON -> `blocked: auth-unknown`.
2. Require `loggedIn === true`, `authMethod === "claude.ai"`, `apiProvider === "firstParty"`, and
   non-empty `subscriptionType`. Any other `authMethod` (`api_key`, `oauth_token`, `third_party`) ->
   `blocked: not-subscription-auth`, naming the method and, if present, `apiKeySource` (never the key).
3. Optionally cross-check with the usage probe: status `ready` confirms the subscription.

Codex:
1. Run `codex login status` with the scrubbed env, 5 s timeout. Require output matching
   `Logged in using ChatGPT`. Anything else (including key logins, unknown text, timeout) ->
   `blocked`.
2. Cross-check with usage probe `account.type === "chatgpt"`.

Both: run once per session start (cheap, <1 s) plus whenever the login changes. Unknown always blocks.
Do **not** reuse `authenticatedFromEnv` for orchestrated sessions: after the scrub it would return
`unknown` anyway, but the existing function would report an unscrubbed key as authenticated. The
existing `claudeAuthStatus` returns `authenticated` on any exit-0 `auth status`, including
`api_key`; ORCH-06 should add a stricter variant or extra check rather than change the default
behaviour of the normal Emdash flow.

### 6.3 Connector switch

For every orchestrated Claude session set `ENABLE_CLAUDEAI_MCP_SERVERS=false` in the launch env and
pass strict MCP config with only the Orchestrator MCP server plus the per-run allowlist. Verify in
the session init event that `mcp_servers` contains only the expected names; refuse to dispatch
otherwise. Codex has no claude.ai connectors; its MCP set comes from `~/.codex/config.toml`, so the
strict-set requirement needs a separate Codex check (**not investigated here**).

## Changed files

- `docs/pm/reports/ORCH-27.md` — this report.

## Verification

`git status --short` in the repo at the end of the spike showed only this report:

```
?? docs/pm/reports/ORCH-27.md
```

Scratch work (probe scripts, an npm install of the Agent SDK) lives in
`%TEMP%\emdash-spikes\ORCH-27\`, outside the repo. No real API key was used, no login changed, no
global config edited; the credential files were not read.

## Tests

None (spike).

## How to check it by hand

Run, from a folder outside the repo: `claude auth status`; `ANTHROPIC_API_KEY=sk-ant-invalid-test
claude -p "say ok" --model haiku`; `claude -p "say ok" --model haiku --output-format stream-json
--verbose` with and without `ENABLE_CLAUDEAI_MCP_SERVERS=false`, and read `mcp_servers` in the
`system`/`init` line.

## Deviations from the brief

- Two Codex runs and a few Claude runs sent a real "say ok" prompt on the owner's subscription
  (small token cost): `OPENAI_API_KEY` run (~2.5k tokens), `OPENAI_BASE_URL` run, and four Haiku
  connector-check runs. The brief asked for this style of test; flagging for transparency.
- The usage probes were replayed from the scratch folder, not run through the repo code, because the
  worktree has no installed dependencies.

## Blockers and open questions

- Can Claude "extra usage" overage be disabled or detected per session? (could not determine)
- Does the ACP adapter forward `strictMcpConfig` and the env var? Verify in ORCH-06 via the init event.
- Codex MCP restriction mechanism for orchestrated sessions: not investigated.
- Should `CLAUDE_CONFIG_DIR` / `CODEX_HOME` be scrubbed by default? (spec says yes; owner may want
  a multi-account setup later)
- Proxy variables: scrub or pass through?

## Suggested follow-ups

- ORCH-06 implements 6.1–6.3 in `session-policy.ts`, with tests for the unscrubbed-key and
  `authMethod` cases above.
- Consider a separate small task to fix the normal-flow auth probe so an invalid/unsubscribed key is
  not reported as "authenticated" (outside orchestrator scope; product decision).
