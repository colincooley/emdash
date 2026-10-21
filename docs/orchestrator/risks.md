# Orchestrator Mode — Risks, Compatibility, And Upstream Conflicts

> **Status: Design approved — implementation not started.** See [architecture](architecture.md).

## 1. Security Risks

| # | Risk | Severity | Mitigation in the design | Residual risk |
| --- | --- | --- | --- | --- |
| S1 | **Prompt injection through worker output or repository content** steers the PM into creating harmful tasks or misleading approval requests | High | PM tools cannot merge, push, delete, change settings, or decide approvals (§7.2). Worker reports are delivered as delimited data in digests. Per-run caps (nodes, dispatch rate, follow-ups). Plan approval before drafts queue (default). Approval cards show the canonical action, not the PM's description of it | A persuaded PM can still waste quota and create noisy tasks up to the caps |
| S2 | **Agents act outside Emdash** with same-user credentials (`gh` CLI token, SSH agent, Emdash's Git credential helper, cloud CLIs) — e.g. a worker runs `gh pr merge` or `git push --force` | High | Workers are allowed to push their own task branches (owner decision), so Git credentials stay available. Provider permission modes prompt for shell commands; dangerous modes refused; worker instructions forbid merge/force-push. **GitHub branch protection on the default branch (required review) is the merge gate**; preflight checks for it and warns when absent | Emdash is not an OS sandbox; without branch protection a misbehaving worker could merge |
| S3 | **Purchases / financial side effects via MCP connectors** inherited from the user's global Claude/Codex config or claude.ai account connectors (payments, brokerage, cloud consoles) | High | Strict MCP set for orchestrated sessions (`mcpServerMode: 'only'`); disable claude.ai-hosted connectors in orchestrated Claude sessions; `spend` approval kind. Preflight lists MCP servers that would have been inherited so the user can see what was excluded | Provider-side features outside MCP (e.g. web tools) are governed only by provider permissions |
| S4 | **Loopback MCP endpoint abuse** by other local processes or a browser (DNS rebinding) | Medium | 127.0.0.1 bind, ephemeral port, 256-bit per-session `Secret` tokens, constant-time compare, strict `Host` check, no CORS, listener only while a run is active | A same-user process that can read the agent process environment/arguments can steal a token; same trust boundary as the existing TUI hook server |
| S5 | **Token or secret leakage** via logs, transcripts, persisted intents, or telemetry | Medium | Tokens are `Secret`-wrapped; never in `orch_*` tables; ACP intents already exclude MCP credentials — add a test for the new field; reuse log redaction; Authorization header carried, not URL query | Provider CLIs may log MCP config in their own debug logs |
| S6 | **Approval spoofing / TOCTOU**: approved action differs from executed action | Medium | Approval bound to canonical `actionSpec` hash; re-hash before execution; decisions only through the renderer Wire procedure; MCP has no decide tool | — |
| S7 | **Subscription-quota exhaustion (runaway loops)** | Medium | Quota headroom gate from `usageLimits` probes; caps on attempts/follow-ups/dispatch rate; run goes `needs-human` instead of looping | Quota probes are advisory and may lag |
| S8 | **Repository-supplied lifecycle scripts** (`.emdash.json` `scripts.prepare/setup`) run automatically for every PM-created worktree | Medium | Existing behavior, but multiplied; preflight shows the resolved scripts; `autoRunSetup`/`autoRunRun` personal config respected | Same as manual task creation |
| S9 | **Shell/path injection via PM-supplied names** | Low | Task names and branch names go through existing validators (`resolveTaskBranchName`, Task name rules); paths only from `WorkspacePlacementResolver`; no string-built commands | — |
| S10 | **Stale or forged completion** (late events from an old session generation) | Low | Monitor checks conversation id + lease generation; ACP session/connection generation fencing already exists | — |

### Approval coverage matrix (R7)

| Category | Emdash-mediated path | Agent-side path | Enforcement strength |
| --- | --- | --- | --- |
| Merge | No MCP tool, no approval kind; the user merges on GitHub | Shell `git merge` into default branch + push / `gh pr merge` | Strong with GitHub branch protection; otherwise instruction-only |
| Push / publish | Allowed for the worker's own task branch (owner decision) | Pushes to other branches / force-push | Instruction + permission prompts; branch protection covers the default branch |
| Destructive (delete task/worktree/branch, force ops) | No MCP tool; approval-gated | Provider permission prompts for shell | Medium |
| Purchases / spend | No tool; `spend` approval kind | Strict MCP set; connectors disabled | Medium–strong |
| System changes (installs, global config, PATH, provider config) | No tool; `system-change` approval | Provider permission prompts | Medium |

## 2. Provider Compatibility Restrictions

| # | Topic | Finding | Consequence |
| --- | --- | --- | --- |
| P1 | Claude auth path | Claude ACP runs `@agentclientprotocol/claude-agent-acp` under Electron-as-Node with `CLAUDE_CODE_EXECUTABLE` pointing at the user's `claude` binary, so it uses the CLI's own login | Subscription auth works **only if** no API-key env reaches the process: Claude Code prefers `ANTHROPIC_API_KEY` when present → Session Policy scrub + auth preflight |
| P2 | Codex auth path | Codex ACP runs `@agentclientprotocol/codex-acp` with `CODEX_PATH`; auth comes from the Codex CLI login (`~/.codex`, `CODEX_HOME`) | Same scrub for `OPENAI_API_KEY` / base URL; preflight via `codex login status` |
| P3 | Terms of service | Running the official CLIs with the user's own subscription is the supported path; automated multi-session use may be subject to plan usage policies and rate limits | Owner confirmed (2026-10-21) that their plans allow this use. Emdash never extracts, proxies, or reuses subscription tokens |
| P4 | Plan rate limits | Subscription plans have rolling usage windows; 5 parallel sessions consume them quickly | Quota headroom gate; PM on a high-capability model, workers selectable per node (e.g. Sonnet for routine work) |
| P5 | Per-session MCP over ACP | ACP `session/new` supports `mcpServers`; HTTP support is advertised per agent (`mcpCapabilities.http`) | Use HTTP when advertised, else stdio shim. Verify both adapters at the pinned versions in Phase 0 |
| P6 | claude.ai connectors | Claude Code can surface claude.ai account connectors in CLI sessions | Disable for orchestrated sessions via the CLI's documented env/setting; **verify the exact switch** against the installed CLI version in Phase 0 |
| P7 | Permission requests | Emdash never auto-approves ACP permission requests (`acp-runtime.md` §Permissions) | Workers that hit prompts go `needs-human`; autonomy comes from native provider modes, not from Emdash approving |
| P8 | Native modes | Option ids differ per provider (Claude modes vs Codex approval/sandbox and `collaboration_mode`) | Policy stores native option ids; UI picks from discovered catalogs (`ProviderSettingsService`), no synthetic ids |
| P9 | Session restore | Codex reports missing rollouts as `-32603` recognized by `isSessionNotFound`; Claude uses deterministic session ids | Monitor maps `session_not_found` to `failed` with a resumable/fresh choice for the human |
| P10 | TUI workers | TUI completion relies on provider hooks (`stop`) and has no structured `sendPrompt` | Deferred; `pty-only` prompt-delivery providers are excluded |

## 3. Windows Compatibility

| # | Topic | Plan |
| --- | --- | --- |
| W1 | Process spawning | MCP stdio shim uses `process.execPath` + `ELECTRON_RUN_AS_NODE=1`, the pattern already used for both ACP adapters on Windows. No shell, no `.cmd` wrapping |
| W2 | Loopback server | `127.0.0.1` bind (as the TUI hook server does) avoids firewall prompts that wildcard binds trigger |
| W3 | Paths | All paths via `HostFileRef`/`parseNativeAbsolute`/placement resolver; no `/` string joins; tests run with Windows path fixtures |
| W4 | Long worktree paths | Deep worktree roots can exceed `MAX_PATH`; preflight warns when `core.longpaths` is off and the resolved worktree root is long |
| W5 | tmux | Not used (ACP workers; tmux is already disabled for local Windows sessions) |
| W6 | Claude Code on Windows | Requires Git for Windows (bash) for its shell tool; preflight surfaces the CLI's own error |
| W7 | Codex on Windows | Codex sandboxing on native Windows is less mature than on macOS/Linux; default Codex workers to approval-prompting modes on Windows and document it |
| W8 | SQLite store | `better-sqlite3` file store in `%APPDATA%\emdash*\` via the existing durable store (backups, invariants); no file locking beyond SQLite's |
| W9 | Line endings | Handoff memos and digests are stored as strings in SQLite; no files written into worktrees by the orchestrator |

## 4. Conflicts With Upstream Emdash Development

Upstream (`generalaction/emdash`) is very active: ~780 commits in the last 60 days, ~100 of them in
the ACP runtime, tasks slice, automations runtime, or app DB that this feature touches.

| # | Conflict source | Likelihood | Strategy |
| --- | --- | --- | --- |
| U1 | **App DB migrations** (sequential `drizzle/00NN_*.sql` + `meta/` journal) | Very high if we add tables | **Avoided**: separate `orchestrator.db` store; zero app-DB schema changes; tasks remain `type: 'task'` |
| U2 | ACP runtime start input / materializer (T1) | High | Keep the change to one optional schema field + one merge call; isolate in a separate commit; contribute upstream as a generic "session-scoped MCP servers" feature (it is useful beyond orchestration) |
| U3 | Conversations controller refactor (T2) | Medium | Pure extraction with an optional hook; upstream PR candidate |
| U4 | Wire protocol versioning for workspace-server | Medium | T1 is optional/additive = minor; v1 is local-host only, so no remote compatibility promise yet |
| U5 | Manifests and `services.ts` bootstrap (T4, T5) | Medium (frequent edits, trivial conflicts) | Append-only, clearly delimited blocks |
| U6 | Upstream ships its own multi-agent / orchestration feature | Medium | Keep orchestration logic in one slice behind a flag so it can be retired or reconciled; prefer upstreaming T1–T3 so the fork's diff shrinks to one slice |
| U7 | Automations runtime evolution | Low | Patterns reused, code not imported |
| U8 | Renames in task/conversation vocabulary (`CONTEXT.md` churn) | Medium | Depend only on public slice APIs and contracts; contract tests in the orchestrator slice catch drift on rebase |

Recommended branch hygiene: one long-lived `feat/orchestrator` branch rebased weekly onto
`upstream/main`; T1–T3 kept as separate, upstreamable commits at the base of the branch.

## 5. Phase 0 Findings (2026-10-21)

From `docs/pm/reports/ORCH-25.md` to `ORCH-28.md`. Decisions D-016 to D-024 are in
`docs/pm/DECISIONS.md`.

| Row | Result | Verdict |
| --- | --- | --- |
| P1 | Confirmed: `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` (even invalid) win over the claude.ai login and also disable claude.ai connectors. `claude auth status` JSON `authMethod` tells login types apart | Go, with scrub + preflight (D-018, D-019) |
| P2 | `OPENAI_API_KEY` and `OPENAI_BASE_URL` are ignored while logged in with ChatGPT, but **`CODEX_API_KEY` wins** and isn't in today's allowlist | Go, with scrub + preflight; `CODEX_API_KEY` is on the scrub list |
| P4 | Usage probes are read-only, about 1 s, and return percent used and reset time per window for both providers; both use experimental APIs | Go as a coarse gate that fails closed |
| P4a (new) | Claude "extra usage" overage can bill beyond the plan; no per-session switch found | Open: Q-3 |
| P5 (Codex) | HTTP MCP (bearer header arrives) and the Electron stdio shim both work on Windows; tools survive `session/load` when servers are passed again | Go for transport |
| P5a (new, Codex) | `session/new.mcpServers` is **added to** the user's Codex MCP config, not a replacement | No-go for strict MCP until ORCH-31 (D-016) |
| P5 (Claude) | ORCH-25: adapter 0.81.2 advertises `http`, `sse`, `loadSession`. HTTP MCP with bearer header works (header on every request, token not in logs or `~/.claude`); Electron stdio shim works, including paths with spaces; tools return after `session/load` with a new port and token | Go; HTTP primary |
| P6 | Through ACP, `_meta.claudeCode.options.strictMcpConfig: true` limits the session to exactly the passed servers (drops `.mcp.json`, user servers and claude.ai connectors); `ENABLE_CLAUDEAI_MCP_SERVERS=false` drops only connectors | Go (D-021) |
| P8 (Claude) | Option ids `mode` (`default`, `acceptEdits`, `plan`, `auto`, `bypassPermissions`), `model`, `effort` (`thought_level`), `fast`. MCP tool calls ask permission even in `acceptEdits`; `allowedTools` pre-approves exact names | Workers use `acceptEdits`; refuse `auto` and `bypassPermissions`; pre-approve orchestrator tools (D-022, D-023) |
| P5b (new) | The ACP runtime sends only `{ cwd, mcpServers }`; orchestrated sessions need provider `_meta` too | ORCH-03 adds it (D-024) |
| W1 | `127.0.0.1` listener under `electron.exe` raised no firewall prompt and created no rule | Go; owner confirms by eye in the real app |
| P8 (Codex) | Option ids `mode` (`read-only`, `agent`, `agent-full-access`), `model` (matches `codex debug models`), `reasoning_effort` (values depend on model), `collaboration_mode`, `fast-mode`. No exact "edits allowed, commands ask" preset | Workers use `agent`; `agent-full-access` refused (D-017) |
| P9 | A missing Codex session returns `-32603` "no rollout found", matching `isCodexSessionNotFound` | Confirmed |
| W7 (Codex) | Commands run under `agent` on native Windows; sandbox enforcement outside the workspace not yet shown | Pending ORCH-31 |
| S2 | Fork `main` is unprotected today; the rules endpoint works for detection. A worker using the owner's login can merge PRs or edit rulesets, and GitHub can't tell it apart from the owner | Owner adds a ruleset; Q-4 on worker identity (D-020) |
