# ORCH-06 — Session Policy (env scrub, auth preflight, strict MCP options, refused modes)

**Suggested agent:** Claude Code · **Model:** Opus 5.5 (effort: high)
**Phase / priority:** P1 / P0
**Depends on (merged):** ORCH-25, ORCH-27

## Context

Read `AGENTS.md`, `docs/pm/PROCESS.md`, then:

- `docs/orchestrator/architecture.md` §5.8 (Session Policy);
- `docs/pm/reports/ORCH-27.md` §6 (the specs you implement) and `docs/pm/reports/ORCH-25.md` §6
  and §8 (Claude option ids and `_meta` options);
- `docs/pm/reports/ORCH-26.md` §6 (Codex option ids);
- decisions D-017 to D-019 and D-021 to D-023 in `docs/pm/DECISIONS.md`.

The Session Policy is the code that makes an orchestrated session safe before it starts: no API
billing, only allowed MCP servers, no dangerous permission modes. This task builds it as a **pure,
tested module**. Nothing calls it yet. ORCH-03 adds the ACP runtime plumbing that will carry its
output, and ORCH-08 calls it. Don't touch the ACP runtime or the conversations controller.

Existing code to read (don't change it):

- `packages/core/src/primitives/agent-env/api/index.ts`: `AGENT_ENV_VARS` allowlist.
- `packages/plugins/src/agents/impl/claude/auth.ts` and `packages/plugins/src/agents/helpers/auth.ts`:
  today's `claudeAuthStatus` reports any exit-0 `auth status` as authenticated, including API-key
  logins, and `authenticatedFromEnv` trusts env keys. **Don't change their behavior**; normal
  Emdash flows rely on it (ORCH-27 §6.2). Build stricter checks in the policy module.
- `packages/plugins/src/agents/impl/codex/index.ts`: how Codex login status is read.

Other tasks running at the same time own `features/orchestrator/contributions/`,
`node/orchestrator-enabled.ts` (ORCH-01) and `node/store/` (ORCH-02). Stay out of those.

## Goal

One module that, given a provider, the final launch env, and requested session options, returns
either a safe launch plan or a typed refusal, plus an auth preflight that proves subscription login.

## Scope

All paths are relative to `apps/emdash-desktop/src/core/features/orchestrator/node/`.

- In:
  - `session-policy/env-scrub.ts`: `scrubOrchestratedEnv(env)` implementing ORCH-27 §6.1 and
    D-018. Case-insensitive names on Windows; prefix patterns (`ANTHROPIC_FOUNDRY_*`,
    `ANTHROPIC_BEDROCK_*`, `ANTHROPIC_VERTEX_*`, `AZURE_OPENAI_*`). Proxy variables pass through.
    Returns the new env plus the list of removed **names** (never values) for the event journal.
    Adds `ENABLE_CLAUDEAI_MCP_SERVERS=false` for Claude.
  - `session-policy/env-scrub.test.ts`: includes a guard test that fails when `AGENT_ENV_VARS`
    gains a name matching `*_API_KEY`, `*_AUTH_TOKEN`, `*_BASE_URL` or `*_OAUTH_TOKEN` that isn't
    classified as scrubbed or explicitly allowed in this module.
  - `session-policy/auth-preflight.ts`: `preflightClaudeAuth(exec, env)` and
    `preflightCodexAuth(exec, env)` per ORCH-27 §6.2 and D-019, with an injected `exec` and a 5 s
    timeout. Results: `{ ok: true, plan }` or `{ ok: false, reason }`, where `reason` is one of
    `not-subscription-auth` (with the method name and `apiKeySource` name only), `signed-out`,
    `auth-unknown`. Unknown always blocks. Never include emails, org ids or tokens in results.
  - `session-policy/session-options.ts`: per provider, validate requested native options and
    produce the session settings:
    - Claude: `mode` must be `acceptEdits`, `default` or `plan`; refuse `bypassPermissions` and
      `auto` (D-022). Produce `_meta.claudeCode.options` with `strictMcpConfig: true` and
      `allowedTools` set to the exact orchestrator tool names for the role (D-021, D-023).
    - Codex: `mode` must be `agent` or `read-only`; refuse `agent-full-access` (D-017). Mark the
      result `strictMcp: 'unavailable'` until ORCH-31 lands (D-016).
    - `model` and `effort` / `reasoning_effort` pass through as native ids; validating them
      against the discovered catalog is out of scope.
  - Tests for each file with fake `exec` outputs taken from the reports (subscription, `api_key`,
    `oauth_token`, `third_party`, logged out, non-JSON, timeout).
- Out: calling the module from anywhere, ACP runtime changes, settings, UI, the MCP gateway,
  changing existing auth probes, new dependencies.

## Ownership

- Owns (may create/edit): `apps/emdash-desktop/src/core/features/orchestrator/node/session-policy/`
- Read only: everything else
- Must not touch: `features/orchestrator/contributions/`, `node/orchestrator-enabled.ts`,
  `node/store/`, `packages/core/src/runtimes/acp/**`, `packages/plugins/**`, any `package.json`,
  `pnpm-lock.yaml`, `AGENTS.md`

## Deliverables

1. The three modules and their tests.
2. In the report: the final scrub list as implemented, and any spec point from ORCH-27 §6 you
   couldn't implement as written, with why.

## Acceptance criteria

- [ ] An env containing every variable in ORCH-27 §6.1 comes out without any of them, in any
      letter case; proxy variables remain; removed names are reported, values never are.
- [ ] Auth preflight passes only the subscription cases; every other fixture blocks with the
      right reason.
- [ ] `bypassPermissions`, `auto` and `agent-full-access` are refused with a typed error.
- [ ] Claude output always contains `strictMcpConfig: true`.
- [ ] `pnpm run format`, `lint`, `typecheck` pass; the new tests pass (list the commands).
- [ ] Only owned files changed (`git status` in the report).

## Report

Write `docs/pm/reports/ORCH-06.md` using the report template in `docs/pm/PROCESS.md`, commit on
your branch, push it, open a PR to `main` on `colincooley/emdash`, and stop. Don't merge.
