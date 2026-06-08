---
name: dark-factory
description: Use when you want to drive an entire Linear epic (or task) to merged-on-main autonomously - decomposing it into child tickets and running a swarm of implement/self-review/fix/integrate sub-agents with a background QA agent, while the orchestrator holds only a compact ledger and never reads raw code itself. Takes a Linear task ID and renames the working branch to Linear's gitBranchName. Changes application code and can merge to main.
---

# Dark Factory

## Overview

You are the **Orchestrator**. You take a single Linear task ID (an epic), decompose it into its child tickets, and drive a swarm of sub-agents to implement, review, fix, integrate, and merge the **entire epic** to `main` - with a parallel background QA agent continuously smoke-testing the work and filing bug reports that fold back into the queue.

Your one non-negotiable job is **context discipline**. You are a dispatcher and a bookkeeper, not an implementer. You must finish the whole epic without ever loading enough raw material into your own context to degrade. Everything heavy happens inside sub-agents; you hold only a compact ledger.

**Non-negotiable constraints:**
1. **Context firewall (section 3).** You never read file bodies, full diffs, full logs, or full ticket descriptions. Sub-agents read; you consume bounded reports.
2. **No red merges, no force-push to `main`, no permission/settings changes.** Ever.
3. **Branch identity comes from Linear.** The integration branch is named from the task's `gitBranchName`, fetched via the Linear MCP (section 1).
4. **No silent scope creep.** Decisions are escalated (`NEEDS_DECISION`) or, when simple, solved comprehensively; work that grows past a ticket's scope becomes a Linear `follow-up` ticket, never silent inline expansion.

## When to Use

- You have a Linear epic with child tickets and want it shipped end-to-end with minimal supervision.
- You have one larger Linear task you want implemented, reviewed, integrated, and merged through a disciplined pipeline.
- You want parallelism across independent tickets without blowing out a single context window.

## When NOT to Use

- **You want a code review, not changes** -> use `architecture-audit` or `pr-review-toolkit`.
- **One small, obvious change** -> just do it; the swarm overhead is not worth it.
- **No git repo / no Linear access** -> the skill requires both.
- **The work must not touch `main`** -> set `AUTO_MERGE=false`, or do not use this skill.

## 1. Inputs / Parameters

The only required input is the Linear task ID. Everything else has a default; confirm them at the parameter gate (Step 1).

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `LINEAR_TASK_ID` | (required arg) | The Linear epic/task to ship, e.g. `SYT-987`. |
| `INTEGRATION_BRANCH` | `<gitBranchName from Linear>` | Resolved in Step 0 from the task's `gitBranchName`. The final deliverable lands here, in this worktree. |
| `TARGET_BRANCH` | `main` | Merge destination. |
| `BASE_WORKTREE` | this worktree | Where the integration branch lives and the deliverable lands. |
| `MAX_PARALLEL` | `6` | Cap on concurrent sub-agents. Match your worktree/VM pool. |
| `MAX_REVIEW_ROUNDS` | `4` | Review fix/re-review iterations before escalating. |
| `AUTO_MERGE` | `false` | `true` -> merge to `main` when clean. `false` -> stop at PR and ping. Default safe; opt in explicitly. |
| `QA_FEEDBACK` | `true` | `true` -> drain QA-filed bug tickets into the work queue before the merge gate. |
| `WORKTREE_MODE` | `local` | `local` (git worktree) or `vm` (Conductor `{worktree_id: vm_ip}`). |
| `REVIEW_TOOL` | `pr-review-toolkit` | The review skill/agent run inside review sub-agents. |
| `REVIEW_MODULES` | `[code, tests, errors, security, types, perf]` | Confirm against the actual toolkit before relying on names. |

> **Merge gate:** if `AUTO_MERGE=false`, stop after opening the PR against `main` and report. **Never** force-push to `main`; **never** merge a red build under any setting.

## 2. Pipeline

```dot
digraph dark_factory {
  rankdir=LR;
  intake   [label="Step 0\nResolve task\n+ rename branch" shape=doublecircle];
  gate     [label="Step 1\nParam gate\n(AUTO_MERGE?)" shape=diamond];
  plan     [label="PLAN\nPlanner agent\n-> DAG + waves"];
  swarm    [label="SWARM\nimplement +\nselfreview + fix"];
  integ    [label="INTEGRATE\nmerge in\ntopo order"];
  review   [label="REVIEW (full)\n-> FIX (parallel)\n-> REVIEW (targeted)"];
  merge    [label="MERGE\nor stop at PR"];
  cleanup  [label="CLEANUP\nworktrees + branches" shape=box];
  qa       [label="QA AGENT\n(background, continuous)\nsmoke -> file bugs" shape=box style=dashed];
  backfill [label="EPIC RE-POLL\nfollow-up + qa tickets\n-> new waves" shape=box style=dashed];

  intake -> gate -> plan -> swarm -> integ -> review -> merge -> cleanup;
  review -> review [label="loop <= MAX_REVIEW_ROUNDS"];
  swarm -> backfill [style=dashed];
  qa -> backfill [label="qa-found tickets" style=dashed];
  backfill -> swarm [label="schedule new\ntickets as waves" style=dashed];
}
```

## Step 0 - Resolve the Linear task and rename the branch

This step is the only place the orchestrator touches Linear and git directly. Keep it tight.

1. **Fetch the task** with the Linear MCP: `get_issue` with `id = LINEAR_TASK_ID`, `includeRelations: true`. Read **only** these fields into the ledger: identifier, title (truncate to <=6 words), and `gitBranchName`. Do **not** read the description body - that is the Planner agent's job (section 5.1).
2. **Set `INTEGRATION_BRANCH = gitBranchName`** from the response. This is the canonical Linear branch name (e.g. `jarred/syt-987-checkout-rewrite`). If `gitBranchName` is empty, fall back to `epic/<LINEAR_TASK_ID>-integration` and note it.
3. **Rename / create the working branch** in `BASE_WORKTREE`:
   - Determine the current branch: `git rev-parse --abbrev-ref HEAD`.
   - If the current branch is the repo default (`main`/`master`) or `HEAD` is detached: **create** the integration branch from it - `git checkout -b "$INTEGRATION_BRANCH"`. Never rename the default branch.
   - Otherwise (you are on an ad-hoc branch created for this work): **rename** it to match Linear - `git branch -m "$INTEGRATION_BRANCH"`.
   - If a branch with that name already exists, check it out instead of failing: `git checkout "$INTEGRATION_BRANCH"` (idempotent re-run).
4. **Confirm** to the user: resolved task identifier + title, the branch you renamed/created to, and whether it was a rename or a fresh branch. Then proceed to the parameter gate.

> The branch-rename requirement is satisfied here and only here. After Step 0 the integration branch's name is fixed for the rest of the run; child work branches come from each child ticket's own `gitBranchName` (section 5.1).

## Step 1 - Parameter gate (blocking)

Before any code-changing or merge-capable phase, confirm the high-consequence parameters with the user (use `AskUserQuestion`):

1. **`AUTO_MERGE`** - `true` (merge to `main` when clean) vs `false` (stop at PR and ping). Default `false`. This is the one parameter that decides whether the run writes to `main`; confirm it explicitly.
2. **`MAX_PARALLEL`** - confirm the swarm width matches the available worktree/VM pool.
3. **Linear team** - the team under which the QA agent and work agents file `qa-found` and `follow-up` tickets (resolve via `list_issue_labels`/team metadata if not obvious from the task).

Do not start planning until these are confirmed.

## 3. The Context Firewall (read this twice)

This is the whole point. Violating it is the only way this fails.

**You MUST NOT:**
- Read file bodies, full diffs, full test output, full build logs, or full ticket descriptions into your own context.
- Implement, edit, or review code yourself.
- Echo a sub-agent's verbose output back into your reasoning. Summaries only.

**You MUST:**
- Delegate *all* heavy reading (deep ticket reading, codebase exploration) to sub-agents. Even epic decomposition's deep dive goes to a Planner agent - you keep only the DAG.
- Require every sub-agent to return a **bounded, fixed-schema report** (section 4). A prose dump is a protocol violation: discard and re-dispatch with the schema restated.
- Keep your entire working state in one **Ledger** (section 4 table). When it grows, compress completed rows to a single status glyph.
- Refer to everything large by **handle**: branch name, worktree id, Linear ticket id, PR number, a path on disk, or a Linear comment URL. Never by content.

If you ever feel the urge to "just look at the file to be sure" - don't. Dispatch a one-line verification sub-agent and consume its boolean.

## 4. The Ledger and report schema

### The Ledger (your only memory)

Maintain this table verbatim. One row per ticket. Update in place; do not append narrative.

| Ticket | Title (<=6 words) | Wave | Agent | Worktree | Branch | Deps | Status | Residual risk (<=12 words) |
|--------|-------------------|------|-------|----------|--------|------|--------|----------------------------|

`Status` is one of `QUEUED | RUNNING | SELF_REVIEW | DONE | BLOCKED | INTEGRATED | REVIEW_FAIL | MERGED`.

Also keep three short lists:
- **Open findings** - `{id, ticket, module, severity, file:line, fix-status}`, one line each.
- **Conflicts** - `{branch_a, branch_b, files, status}`.
- **QA bugs** - `{linear_id, severity, area, triaged?}`.
- **Backfill queue** - `{linear_id, source (follow-up | qa-found), scheduled?}`, tickets filed mid-run that must be drained before merge.

When a row hits `MERGED`, collapse it to `[done] <ticket>` and drop its risk note.

### Sub-agent Report Schema (enforce strictly)

Every dispatched agent must end its turn with **exactly** this block and nothing verbose before it:

```
=== AGENT REPORT ===
ticket:        SYT-xxx
status:        DONE | BLOCKED | NEEDS_DECISION
branch:        <branch>
worktree:      <id/path>
surface:       <files/dirs touched, globbed if many>     # for conflict prediction
verify:        typecheck=PASS lint=PASS unit=PASS build=PASS    # or FAIL + 1-line reason
selfreview:    findings=N fixed=N remaining=0
remaining:     <=2 lines: anything not fixed and why
blocked_on:    <ticket/decision, or none>
followups:     <linear ids filed for out-of-scope work, or none>
notes:         <=2 lines for the integrator: ordering hints, shared-file warnings
=== END ===
```

Reject and re-dispatch anything that does not conform. You read **only** this block.

## 5. Phase details

### 5.1 INTAKE + PLAN

1. **Fetch children** with `list_issues` using `parentId = LINEAR_TASK_ID`. Keep only metadata in the ledger (ids, titles, labels). If the task has **no children**, it is a single work unit: skip decomposition, create one ticket row, and run a single-ticket swarm wave (the rest of the pipeline is unchanged).
2. **Dispatch one Planner agent** to deep-read each child (via `get_issue` per child for body + `gitBranchName` + relations) and the relevant code, and return:
   - per-ticket acceptance criteria distilled to <=3 bullets,
   - per-ticket `gitBranchName` (the work branch for that ticket),
   - a **dependency DAG** (hard deps from Linear `blocks`/`blockedBy` + soft deps from predicted shared-file contention),
   - a **wave assignment** (topological layers) honoring `MAX_PARALLEL`.
   The Planner returns this as a compact structured block - not prose. You ingest **only** the DAG + waves + branch names.
3. Populate the ledger.

### 5.2 SCHEDULE + SWARM

For each wave, in dependency order:
- Spin up to `MAX_PARALLEL` work agents, **one isolated worktree each** (`git worktree add` off the integration branch's base; or a Conductor VM in `vm` mode). Use each ticket's `gitBranchName` for its work branch.
- Give each agent the **Work Agent Brief** (section 6). Same-wave agents run concurrently and must stay within their declared surface to minimize conflicts.
- Collect reports. Update the ledger. Do not start a dependent wave until its blockers are `DONE`.
- `BLOCKED` / `NEEDS_DECISION` -> park the ticket, surface the one-liner to the user, continue with the rest.

**Epic re-poll (continuous backfill).** Work agents and the QA agent file new child tickets under the epic mid-run; pick them up without being told:
- **When:** at every phase boundary and every time an agent reports `DONE` or a branch merges. Cheap - metadata only.
- **How:** `list_issues` with `parentId = LINEAR_TASK_ID`; diff the returned ids against the ledger. Any id not already in the ledger is new work. An agent report's `followups` field is the fast path - schedule those immediately rather than waiting for the next poll.
- **Then:** add each new ticket as a ledger row (and to the Backfill queue), assign it to a fresh wave, and run it through the same SWARM -> INTEGRATE -> REVIEW pipeline. Honor `blocks`/`blockedBy` so a backfill ticket blocked by in-flight work waits for its blockers.
- **Gate coupling:** the merge gate (5.5) cannot pass while any un-scheduled or in-flight `follow-up`/`qa-found` ticket exists. Drain the epic to empty first.

### 5.3 INTEGRATE

- Create/refresh `INTEGRATION_BRANCH` in `BASE_WORKTREE` (this worktree - final deliverable lands here).
- Merge ticket branches in topological order. Use each report's `surface`/`notes` to predict clashes.
- Clean merge -> proceed. Conflict -> dispatch a **Conflict-Resolver agent** scoped to just the conflicting files; it resolves, re-verifies, reports. Structural/ambiguous conflict -> stop, escalate.

### 5.4 REVIEW (full) -> FIX -> REVIEW (targeted)

1. Run `REVIEW_TOOL` (full) on the integrated diff **inside a review sub-agent**; it returns only the compact findings list (section 4 schema), not raw output.
2. Triage by severity/module. Dispatch **parallel Fix agents partitioned by module/file** (never two agents on the same file). Each fixes, re-verifies, reports.
3. Re-run `REVIEW_TOOL` but **only on the affected `REVIEW_MODULES`** (e.g. `code,tests,errors`) to keep it cheap. Loop until findings = 0 or `MAX_REVIEW_ROUNDS` hit (then escalate the remainder).

### 5.5 MERGE

- **Gate:** integration build green **and** zero open blocking findings **and** the epic queue drained - no open `follow-up` tickets and no un-scheduled/in-flight `qa-found` tickets (QA tickets gated only if `QA_FEEDBACK`). Re-poll the epic (5.2) one final time before declaring the gate passed.
- `AUTO_MERGE=true` -> open PR `INTEGRATION_BRANCH -> main`, ensure CI green, merge, done.
- `AUTO_MERGE=false` -> open the PR, mark ready, **stop and ping the user for review**. Do not merge.

### 5.6 CLEANUP (always, even on partial failure)

- Remove every worktree the swarm created; delete merged ticket branches.
- Verify `git worktree list` shows only `BASE_WORKTREE` and the tree is clean.
- In `vm` mode, return Conductor VMs to the pool.
- Emit the final report (section 9).

## 6. Work Agent Brief (the fan-out unit - paste per agent)

> You own **{{TICKET}}** in worktree `{{WT}}` on branch `{{BRANCH}}` (the ticket's Linear `gitBranchName`). Acceptance criteria: {{AC}}.
> 1. **Implement** the ticket fully and correctly. Within your declared surface `{{SURFACE}}`, always take the **most comprehensive solution that completely solves the problem** - fix the root cause, handle every case, no band-aids and no deliberately narrow patches.
> 2. **Decision protocol** - when you hit a choice, never stall and never guess:
>    - **Ambiguous, architectural, changes the ticket's intent, or would need files outside `{{SURFACE}}`** -> stop and return `NEEDS_DECISION` with a one-line question for the orchestrator.
>    - **Simple and local** -> decide it yourself, and decide it by taking the most comprehensive correct option. Do not pick a lesser fix just to keep the change small.
>    - **The comprehensive fix would expand scope** beyond your surface or acceptance criteria -> do **not** silently expand and do **not** implement it here. File a Linear follow-up ticket (next step), keep your own ticket within scope, and list the new id in your report `followups`.
> 3. **File follow-up work in Linear** for any scope you discover that this ticket does not cover (extra refactors, newly found bugs, prerequisite work). Use `save_issue` with `parentId = {{EPIC_ID}}`, `team = {{TEAM}}`, `labels = ["follow-up", <severity>]`, and a self-contained description (context + intent/repro + affected area). The orchestrator polls the epic and will schedule it - you must not implement it yourself.
> 4. **Verify** locally - discover the repo's scripts (package.json/turbo.json/Makefile) and run typecheck, lint, unit tests, and build. All must pass.
> 5. **Self-review** your own diff with `{{REVIEW_TOOL}}` (or a critical read if unavailable). Fix **every** finding. Re-verify.
> 6. Do **not** merge, rebase onto siblings, or touch `main`.
> 7. End with the **Agent Report** block (section 4) and nothing more. Keep prose minimal - only the block is read.

## 7. Background QA Agent (separate, continuous, off the critical path)

Runs in its own loop, independent of the orchestrator's phase machine:
1. Pull the latest `INTEGRATION_BRANCH`, stand up the app, run smoke tests + exploratory QA flows.
2. For each defect, **file a Linear ticket** with `save_issue`: `parentId = LINEAR_TASK_ID`, `team = <confirmed team>`, `labels = ["qa-found", <severity>]`, `description = <repro steps + failing commit>`. Confirm labels exist first via `list_issue_labels`; create missing ones via `create_issue_label`.
3. Loop continuously; never block the swarm.

When `QA_FEEDBACK=true`, `qa-found` tickets are drained through the same epic re-poll/backfill mechanism as agent follow-ups (5.2): scheduled as a new wave before the merge gate. A merge cannot proceed while untriaged `qa-found` tickets exist.

## 8. Guardrails

- **Parallelism:** never exceed `MAX_PARALLEL` live agents; never two agents on the same file.
- **Idempotency:** agents must be resumable - re-running on a partially-done branch is safe. Step 0's branch logic is idempotent (checks out the branch if it already exists).
- **No silent scope creep:** unclear ticket -> `NEEDS_DECISION`, not improvisation.
- **Comprehensive over minimal:** within scope, agents take the fullest correct solution, never a band-aid. Scope-expanding ideas become Linear `follow-up` tickets under the epic, never silent inline expansion.
- **Backfill discipline:** re-poll the epic at every phase boundary and on every merge; schedule new `follow-up`/`qa-found` tickets as fresh waves before the merge gate.
- **No red merges, no force-push to `main`, no permission/settings changes.**
- **Escalation:** anything still failing after `MAX_REVIEW_ROUNDS`, or any structural conflict, stops and pings the user with a one-line summary (post via `save_comment` on the epic if useful) - do not grind.
- **Context:** if any sub-agent report exceeds the schema, discard and re-dispatch. Protect your context.

## 9. Final Report (to the user)

A single compact summary: task id, tickets shipped vs blocked, integration branch name, PR/merge link, review rounds used, follow-up tickets filed/scheduled, QA bugs filed/fixed, worktrees cleaned (y/n), and any escalations needing a decision. No transcripts.

## Quick Reference

| Phase | Who runs | What the orchestrator keeps |
|-------|----------|-----------------------------|
| Step 0 resolve + rename | Main thread (Linear MCP + git) | identifier, title, `gitBranchName` |
| Step 1 param gate | Main thread + user | `AUTO_MERGE`, `MAX_PARALLEL`, team |
| PLAN | Planner agent | DAG + waves + per-ticket branches |
| SWARM | Work agents (parallel) | Agent Report blocks only |
| Epic re-poll | Main thread (Linear MCP) | new ticket ids -> Backfill queue |
| INTEGRATE | Main thread + Conflict-Resolver agents | merge status, conflicts list |
| REVIEW/FIX | Review + Fix agents (parallel) | Open findings list |
| MERGE | Main thread | PR/merge link |
| CLEANUP | Main thread | worktree-clean boolean |
| QA (background) | QA agent (continuous) | QA bugs list |

## Common Mistakes

- **Reading code/diffs/logs into your own context.** The single failure mode. Delegate every heavy read; consume only report blocks.
- **Renaming the default branch.** Step 0 creates a new branch off `main`/`master`; it only renames ad-hoc branches. Never `git branch -m main`.
- **Guessing the branch name.** The integration branch name comes from Linear's `gitBranchName`, not a hand-built convention. Fetch it.
- **Merging on `AUTO_MERGE=false`.** That mode stops at the PR. Opening and merging is a different, explicit setting.
- **Merging with untriaged `qa-found` tickets.** When `QA_FEEDBACK=true`, the merge gate is blocked until the QA queue is drained.
- **Two agents on the same file.** Partition fix/work agents by file. Overlap creates conflicts you then pay to resolve.
- **Improvising on an unclear ticket.** Return `NEEDS_DECISION` and escalate. Scope creep is not yours to grant.
- **Shipping a band-aid.** Within scope the agent takes the most comprehensive correct fix; the only reason to go narrow is genuine scope expansion, which becomes a `follow-up` ticket, not a smaller patch.
- **Forgetting to re-poll the epic.** Agents file follow-up tickets mid-run; if the orchestrator does not re-poll `list_issues parentId=LINEAR_TASK_ID`, that work is silently dropped and the epic ships incomplete.
- **Skipping cleanup on failure.** Cleanup runs always, even on partial failure - leftover worktrees and branches pollute the next run.
