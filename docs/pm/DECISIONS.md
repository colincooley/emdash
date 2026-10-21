# Decisions

Numbered log of project decisions for Orchestrator Mode. The owner makes product decisions; the
PM records technical and process decisions and marks them as such. Agents don't settle product
questions — they add them to Open questions and keep unrelated work moving.

Status: **Approved** (the owner agreed), **Proposed** (waiting on the owner), or **Superseded**.

| ID | Date | Decision | Status |
| --- | --- | --- | --- |
| D-001 | 2026-10-21 | PM docs live in `docs/pm/` (roadmap, backlog, decisions, process, task briefs, reports), same shape as Toll | Approved |
| D-002 | 2026-10-21 | Hard maximum of 5 running Emdash tasks including the PM, so 4 implementation slots; human tasks don't use a slot. The product's own cap is also a fixed 5 and counts manual sessions without blocking them | Approved |
| D-003 | 2026-10-21 | Orchestrator v1 is local-host only; SSH / workspace-server hosts later | Approved |
| D-004 | 2026-10-21 | Orchestrator state lives in its own SQLite store; no app-DB schema changes or Drizzle migrations (avoids upstream conflicts) | Approved |
| D-005 | 2026-10-21 | Workers may push their own task branches and open PRs against the fork; the owner merges on GitHub; Emdash has no merge path | Approved |
| D-006 | 2026-10-21 | The owner reviews the initial plan; after that, findings and decisions are surfaced as they arise and only block the tasks that depend on them | Approved |
| D-007 | 2026-10-21 | Owner confirmed Claude and ChatGPT plan terms allow automated parallel use with subscription logins | Approved |
| D-008 | 2026-10-21 | Each brief declares owned paths; running tasks never overlap; hotspot files have one owner at a time (Toll D-007) | Approved |
| D-009 | 2026-10-21 | Phase 0 is split into four parallel spikes (ORCH-25 to ORCH-28) instead of one ORCH-00 task, so they fit the 4 slots | Approved (PM, process) |
| D-010 | 2026-10-21 | Spike code stays outside the repo (a scratch folder); only the report is committed | Approved (PM, process) |
| D-011 | 2026-10-21 | PM process docs are not referenced from `AGENTS.md` (an upstream hotspot); briefs are self-contained and point at `docs/pm/PROCESS.md` | Approved (PM, process) |
| D-012 | 2026-10-21 | The product mirrors this manual process: each work item carries a paste-ready prompt, suggested agent, model, effort, owned paths, and dependencies; the owner sees them in a "next up" list (architecture §11) | Approved |
| D-013 | 2026-10-21 | In the product, a dependency gate defaults to "merged to `main`" (as in this process); "accepted by the PM" (stacked branches) is a per-run opt-in | Approved |
| D-014 | 2026-10-21 | In the product, ready work dispatches automatically after the owner approves the initial plan (no per-task Launch click). The owner can still Hold, Edit or Cancel any queued node, and a per-run "manual launch" switch stays available. Until the product's launcher exists (ORCH-08/ORCH-10), the owner launches tasks by hand, because Emdash has no external way to create tasks today | Approved |
| D-015 | 2026-10-21 | The PM presents each ready task as its own block (task, prompt file, agent, model, effort), never several tasks in one table, so each copies cleanly | Approved |
| D-016 | 2026-10-21 | Codex `session/new.mcpServers` is additive to the user's Codex config (ORCH-26), so strict MCP for Codex workers is blocked until ORCH-31 finds an isolation mechanism. Claude workers are unaffected | Approved (PM, technical) |
| D-017 | 2026-10-21 | Codex workers run with `mode=agent` (workspace-write + automatic review); `agent-full-access` is refused by policy. Native option ids: `mode`, `model`, `reasoning_effort`, `collaboration_mode` | Approved (PM, technical; revisit after ORCH-31 sandbox test) |
| D-018 | 2026-10-21 | Session Policy scrub list is ORCH-27 §6.1, applied last to the final launch env, case-insensitive on Windows, including `CODEX_API_KEY` and `CLAUDE_CODE_OAUTH_TOKEN`. `CLAUDE_CONFIG_DIR` and `CODEX_HOME` are scrubbed by default; proxy variables pass through. Claude sessions also get `ENABLE_CLAUDEAI_MCP_SERVERS=false` plus strict MCP config | Approved (PM, technical) |
| D-019 | 2026-10-21 | Auth preflight: Claude requires `claude auth status` JSON `authMethod=claude.ai`, `apiProvider=firstParty` and a non-empty `subscriptionType`; Codex requires `Logged in using ChatGPT`. Anything else, or unknown, blocks the run. Usage probes fail closed | Approved (PM, technical) |
| D-020 | 2026-10-21 | Branch-protection preflight uses `GET /repos/{o}/{r}/rules/branches/{default}` and looks for an active `pull_request` rule; errors map to `unknown`, never `unprotected` | Approved (PM, technical) |
| D-021 | 2026-10-21 | Claude workers get strict MCP through `_meta.claudeCode.options.strictMcpConfig: true` on both `session/new` and `session/load` (ORCH-25), with `ENABLE_CLAUDEAI_MCP_SERVERS=false` as a second layer | Approved (PM, technical) |
| D-022 | 2026-10-21 | Claude native option ids are `mode`, `model`, `effort`. Workers default to `mode=acceptEdits`; `bypassPermissions` and `auto` are refused | Approved (PM, technical) |
| D-023 | 2026-10-21 | Orchestrator tools are pre-approved per role with exact names in `allowedTools` (`mcp__emdash_orch__<tool>`); the MCP server name is fixed as `emdash_orch` | Approved (PM, technical) |
| D-024 | 2026-10-21 | ORCH-03 adds an optional per-session provider `_meta` field to the ACP start input alongside `sessionMcpServers`, re-sent on load and never persisted in intents | Approved (PM, technical) |
| D-025 | 2026-10-21 | Every brief ends with "push your branch and open a PR to `main`" so finished work always shows up as a PR | Approved (PM, process) |

## Open questions

Waiting on the owner. A question only blocks the tasks listed under "Blocks".

| ID | Question | Source | Blocks |
| --- | --- | --- | --- |
| Q-3 | Claude "extra usage" (pay-per-use overage) can bill beyond the subscription, and ORCH-27 found no per-session switch. Will you turn extra usage off in your claude.ai account settings? PM recommendation: yes, and the product warns when the usage probe shows it on | ORCH-27 | ORCH-06 (warning only) |
| Q-4 | Workers push with your own GitHub login. GitHub can't tell an agent using that login from you, so no ruleset can stop an agent from merging (`gh pr merge`) or editing the ruleset (ORCH-28). Options: (a) accept for v1 and rely on the worker instructions plus the orchestrator having no merge tool; (b) give workers a separate bot account with write access, then require 1 approval on `main` so only you can approve and merge. PM recommendation: (a) now, (b) before running Orchestrator Mode on other projects | ORCH-28 | ORCH-20 (design only) |

| Q-5 | Orchestrated Claude sessions currently load your user and project Claude settings (hooks, plugins, skills, `CLAUDE.md`). Keep that, or limit them to project settings only? PM recommendation: keep both for v1, because repo instructions and your hooks matter, and revisit if a hook or plugin misbehaves | ORCH-25 | — (nothing blocked) |

Answered: Q-1 by D-014 (auto-dispatch); Q-2 by D-013 (merged to `main`).
