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
| D-013 | 2026-10-21 | In the product, a dependency gate defaults to "merged to `main`" (as in this process); "accepted by the PM" (stacked branches) is a per-run opt-in | Proposed |

## Open questions

Waiting on the owner. A question only blocks the tasks listed under "Blocks".

| ID | Question | Source | Blocks |
| --- | --- | --- | --- |
| Q-1 | Should the product auto-dispatch ready work after plan approval (current design), or show each next task for a one-click launch like this manual process? A per-run toggle is possible | D-012 | ORCH-15, ORCH-29 (design only; Phase 0 unaffected) |
| Q-2 | D-013: is "merged to main" the right default dependency gate for the product? | PM | ORCH-10 |
