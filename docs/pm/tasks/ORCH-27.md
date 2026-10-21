# ORCH-27 — Spike: subscription-only auth, billing env vars, and claude.ai connectors

**Suggested agent:** Claude Code · **Model:** Opus 5.5 (effort: high)
**Phase / priority:** P0 / P0
**Depends on (merged):** —

## Context

Read `docs/pm/PROCESS.md`, then `docs/orchestrator/architecture.md` §5.8 (Session Policy) and
`docs/orchestrator/risks.md` rows S3, S5, P1–P4, P6.

Requirement: orchestrated agents must use the owner's existing Claude and ChatGPT subscription
logins, never API billing. Today Emdash passes billing-related variables through to agents when
they are set in the user's shell or project env (allowlist in
`packages/core/src/primitives/agent-env/api/index.ts`), and the auth probes report
"authenticated" as soon as an API-key variable is present
(`packages/plugins/src/agents/helpers/auth.ts` `authenticatedFromEnv`, used by
`packages/plugins/src/agents/impl/claude/auth.ts` and `codex/index.ts`).

The design also needs orchestrated Claude sessions to *not* see claude.ai-hosted MCP connectors
(some can spend money), and a quota gate fed by the existing usage probes
(`packages/plugins/src/agents/impl/claude/usage.ts`, `codex/usage.ts`).

## Goal

Produce the exact env scrub list, an auth preflight rule that proves subscription login, the switch
that disables claude.ai connectors, and a judgment on whether the usage probes can drive a quota
gate.

## Scope

- In:
  - Read the allowlist and list every variable that can route billing or a different backend
    (Anthropic, Bedrock, Vertex, OpenAI, Azure, base-URL overrides, auth tokens, etc.).
  - Claude: show what `claude auth status` (JSON) reports for a subscription login; which field
    distinguishes subscription vs API key. To show precedence **without billing**, run a one-line
    `claude -p` with a deliberately invalid dummy key (e.g. `ANTHROPIC_API_KEY=sk-ant-invalid-test`)
    and record whether it fails on the key (meaning the key would take priority).
  - Codex: same for `codex login status` and a tiny `codex exec` with a dummy invalid
    `OPENAI_API_KEY`.
  - Find and verify the setting or env var that stops Claude Code from loading claude.ai
    connectors, and what `--strict-mcp-config` does. Verify by asking a cheap model to list its MCP
    server names (names only).
  - Read the usage probes; run them if they are read-only, and report what they return (no
    account identifiers in the report).
  - Propose: the scrub list, the preflight algorithm, and the connector switch, as short code-free
    specs the ORCH-06 task can implement.
- Out (do not do): setting real API keys; logging out or changing logins; editing `~/.claude*`,
  `~/.codex/*` or any global config; changing repo files other than your report; product code.

## Ownership

- Owns (may create/edit): `docs/pm/reports/ORCH-27.md`; scratch folder
  `%TEMP%\emdash-spikes\ORCH-27\` (outside the repo)
- Read only: everything else in the repo
- Must not touch: any other repo file, any other project, Emdash app data, credential files

## Deliverables

1. Table of billing-routing variables in the allowlist, with what each does.
2. Claude and Codex auth-status outputs (redacted) and the field that proves subscription login.
3. Precedence results with dummy keys (verbatim error lines).
4. The claude.ai connector switch, with before/after evidence; `--strict-mcp-config` behavior.
5. What the usage probes return and whether they are suitable for a quota gate (latency, failure
   modes).
6. Specs for ORCH-06: scrub list, preflight algorithm, connector switch.

## Acceptance criteria

- [ ] Every deliverable answered with evidence, or marked "could not determine" with why.
- [ ] No real API key used; no login changed; no global config changed.
- [ ] No tokens, emails, or account ids in the report.
- [ ] No repo file changed except `docs/pm/reports/ORCH-27.md` (`git status` shown in the report).

## Report

Write `docs/pm/reports/ORCH-27.md` using the report template in `docs/pm/PROCESS.md`, commit it on
your branch, push the branch, and stop. Don't merge.
