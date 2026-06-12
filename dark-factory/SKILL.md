---
name: dark-factory
description: Use when you want to drive an entire Linear epic (or task) to merged-on-main autonomously - decomposing it into child tickets and running a swarm of implement/self-review/fix sub-agents with a background QA agent, while the orchestrator holds only a compact ledger and never reads raw code itself. Each child ticket lands as its own PR; the orchestrator merges them in topological order. Takes a Linear task ID; the orchestrator worktree is already on the epic branch (user sets this up). Changes application code and can merge PRs to main.
---

# Dark Factory

## Overview

You are the **Orchestrator**. You take a single Linear task ID (an epic), decompose it into its child tickets, and drive a swarm of sub-agents to implement, review, fix, and merge the **entire epic** to `main` - with a parallel background QA agent continuously smoke-testing the work and filing bug reports that fold back into the queue.

Your one non-negotiable job is **context discipline**. You are a dispatcher and a bookkeeper, not an implementer. You must finish the whole epic without ever loading enough raw material into your own context to degrade. Everything heavy happens inside sub-agents; you hold only a compact ledger.

**The merge model:** Each child ticket lands as its **own PR** directly to `TARGET_BRANCH`. The orchestrator merges those PRs individually in topological order via `gh pr merge`. There is no integration branch. The orchestrator worktree is never modified by any of this - it is bookkeeping space only.

**Non-negotiable constraints:**
1. **Context firewall (section 3).** You never read file bodies, full diffs, full logs, or full ticket descriptions. Sub-agents read; you consume bounded reports.
2. **No red merges, no force-push to `main`, no permission/settings changes.** Ever.
3. **Never manually merge ticket branches into the orchestrator worktree.** Ticket work stays on ticket branches. PRs are merged via `gh pr merge`.
4. **No silent scope creep.** Decisions are escalated (`NEEDS_DECISION`) or, when simple, solved comprehensively; work that grows past a ticket's scope becomes a Linear `follow-up` ticket, never silent inline expansion.

## When to Use

- You have a Linear epic with child tickets and want it shipped end-to-end with minimal supervision.
- You have one larger Linear task you want implemented, reviewed, and merged through a disciplined pipeline.
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
| `TARGET_BRANCH` | `main` | Where each child ticket PR is merged. |
| `MAX_PARALLEL` | `6` | Cap on concurrent sub-agents. Match your worktree/VM pool. |
| `MAX_REVIEW_ROUNDS` | `4` | Review fix/re-review iterations before escalating. |
| `AUTO_MERGE` | `false` | `true` -> merge each child PR when clean. `false` -> open PRs, ping user, stop. Default safe; opt in explicitly. |
| `QA_FEEDBACK` | `true` | `true` -> drain QA-filed bug tickets into the work queue before epic closure. |
| `WORKTREE_MODE` | `local` | `local` (git worktree) or `vm` (Conductor `{worktree_id: vm_ip}`). |
| `REVIEW_TOOL` | `pr-review-toolkit` | The review skill/agent run inside review sub-agents. |
| `REVIEW_MODULES` | `[code, tests, errors, security, types, perf]` | Confirm against the actual toolkit before relying on names. |

> **Merge gate:** if `AUTO_MERGE=false`, stop after opening all PRs and report their URLs. **Never** force-push to `main`; **never** merge a red build under any setting.

## 2. Pipeline

```dot
digraph dark_factory {
  rankdir=LR;
  intake   [label="Step 0\nConfirm epic branch\n+ fetch metadata" shape=doublecircle];
  gate     [label="Step 1\nParam gate\n(AUTO_MERGE?)" shape=diamond];
  plan     [label="PLAN\nPlanner agent\n-> DAG + waves"];
  swarm    [label="SWARM\nimplement +\nself-review +\nopen PR"];
  review   [label="PR REVIEW\nper ticket\n-> FIX -> re-review"];
  merge    [label="PR MERGE\n(topo order via\ngh pr merge)"];
  close    [label="EPIC CLOSE\nfinal re-poll\n-> close Linear epic"];
  cleanup  [label="CLEANUP\nsub-worktrees\n+ ticket branches" shape=box];
  qa       [label="QA AGENT\n(background, continuous)\nsmoke -> file bugs" shape=box style=dashed];
  backfill [label="EPIC RE-POLL\nfollow-up + qa tickets\n-> new waves" shape=box style=dashed];

  intake -> gate -> plan -> swarm -> review -> merge -> close -> cleanup;
  review -> review [label="loop <= MAX_REVIEW_ROUNDS"];
  swarm -> backfill [style=dashed];
  qa -> backfill [label="qa-found tickets" style=dashed];
  backfill -> swarm [label="schedule new\ntickets as waves" style=dashed];
}
```

## Step 0 - Confirm the epic branch and fetch metadata

This step is the only place the orchestrator touches Linear and git directly. Keep it tight.

1. **Confirm the current branch** with `git rev-parse --abbrev-ref HEAD`. This is the orchestrator's branch; it is already set to the epic's Linear branch name by the user. Note it in the ledger as the context anchor. **Do not rename, create, or check out any branch here.**
2. **Fetch the task** with the Linear MCP: `get_issue` with `id = LINEAR_TASK_ID`, `includeRelations: true`. Read **only** these fields into the ledger: identifier, title (truncate to <=6 words). Do **not** read the description body - that is the Planner agent's job (section 5.1).
3. **Confirm** to the user: resolved task identifier + title, and the current branch. Then proceed to the parameter gate.

> The orchestrator worktree branch is fixed by the user before invoking the skill. This step only verifies it. After Step 0 the orchestrator branch is never touched by work or merge operations; all work happens in sub-worktrees on ticket branches.

## Step 1 - Parameter gate (blocking)

Before any code-changing or merge-capable phase, confirm the high-consequence parameters with the user (use `AskUserQuestion`):

1. **`AUTO_MERGE`** - `true` (merge each child PR to `main` when clean) vs `false` (open PRs and ping). Default `false`. This decides whether the run writes to `main`; confirm it explicitly.
2. **`MAX_PARALLEL`** - confirm the swarm width matches the available worktree/VM pool.
3. **Linear team** - the team under which the QA agent and work agents file `qa-found` and `follow-up` tickets (resolve via `list_issue_labels`/team metadata if not obvious from the task).

Do not start planning until these are confirmed.

## 3. The Context Firewall (read this twice)

This is the whole point. Violating it is the only way this fails.

**You MUST NOT:**
- Read file bodies, full diffs, full test output, full build logs, or full ticket descriptions into your own context.
- Implement, edit, or review code yourself.
- Merge ticket branches into the orchestrator worktree - not via `git merge`, not via cherry-pick, not via any other mechanism.
- Echo a sub-agent's verbose output back into your reasoning. Summaries only.

**You MUST:**
- Delegate *all* heavy reading (deep ticket reading, codebase exploration) to sub-agents. Even epic decomposition's deep dive goes to a Planner agent - you keep only the DAG.
- Require every sub-agent to return a **bounded, fixed-schema report** (section 4). A prose dump is a protocol violation: discard and re-dispatch with the schema restated.
- Keep your entire working state in one **Ledger** (section 4 table). When it grows, compress completed rows to a single status glyph.
- Refer to everything large by **handle**: branch name, worktree id, Linear ticket id, PR number, a path on disk, or a Linear comment URL. Never by content.

If you ever feel the urge to "just look at the file to be sure" - don't. Dispatch a one-line verification sub-agent (`model: haiku`) and consume its boolean.

## 4. The Ledger and report schema

### The Ledger (your only memory)

Maintain this table verbatim. One row per ticket. Update in place; do not append narrative.

| Ticket | Title (<=6 words) | Wave | Agent | Worktree | Branch | PR | Deps | Status | Residual risk (<=12 words) |
|--------|-------------------|------|-------|----------|--------|----|------|--------|----------------------------|

`Status` is one of `QUEUED | RUNNING | PR_OPEN | REVIEW_CLEAN | REVIEW_FAIL | PR_MERGED | BLOCKED`.

Also keep three short lists:
- **Open findings** - `{id, ticket, module, severity, file:line, fix-status}`, one line each.
- **Rebase conflicts** - `{pr, branch, status}` - PRs that conflicted on merge and need a rebase agent.
- **QA bugs** - `{linear_id, severity, area, triaged?}`.
- **Backfill queue** - `{linear_id, source (follow-up | qa-found), scheduled?}`, tickets filed mid-run that must be drained before epic closure.

When a row hits `PR_MERGED`, collapse it to `[done] <ticket>` and drop its risk note.

### Sub-agent Report Schema (enforce strictly)

Every dispatched agent must end its turn with **exactly** this block and nothing verbose before it:

```
=== AGENT REPORT ===
ticket:          SYT-xxx
status:          DONE | BLOCKED | NEEDS_DECISION
branch:          <branch>
worktree:        <id/path>
pr:              <PR URL - REQUIRED; "none" is a protocol violation>
surface:         <files/dirs touched, globbed if many>
verify:          typecheck=PASS lint=PASS unit=PASS build=PASS    # or FAIL + 1-line reason
toolkit_review:  rounds=N findings_fixed=N remaining=0           # or remaining=N + reason if MAX_REVIEW_ROUNDS hit
remaining:       <=2 lines: anything not fixed and why
blocked_on:      <ticket/decision, or none>
followups:       <linear ids filed for out-of-scope work, or none>
notes:           <=2 lines: ordering hints, shared-file warnings for merge order
=== END ===
```

**A report without a real PR URL is a protocol violation.** The PR must be open before `pr-review-toolkit` can run; any agent that skips PR creation cannot have run the toolkit. Discard and re-dispatch with the protocol restated.

Reject and re-dispatch anything that does not conform. You read **only** this block.

## 5. Phase details

### 5.1 INTAKE + PLAN

1. **Fetch children** with `list_issues` using `parentId = LINEAR_TASK_ID`. Keep only metadata in the ledger (ids, titles, labels). If the task has **no children**, it is a single work unit: skip decomposition, create one ticket row, and run a single-ticket swarm wave (the rest of the pipeline is unchanged).
2. **Dispatch one Planner agent** (`model: sonnet` - planning is extraction and topological sorting, not synthesis; a missed soft dep degrades to a rebase conflict the pipeline already recovers from) to deep-read each child (via `get_issue` per child for body + `gitBranchName` + relations) and the relevant code, and return:
   - per-ticket acceptance criteria distilled to <=3 bullets,
   - per-ticket `gitBranchName` (the work branch for that ticket),
   - a **dependency DAG** (hard deps from Linear `blocks`/`blockedBy` + soft deps from predicted shared-file contention),
   - a **wave assignment** (topological layers) honoring `MAX_PARALLEL`.
   The Planner returns this as a compact structured block - not prose. You ingest **only** the DAG + waves + branch names.
3. Populate the ledger.

### 5.2 SCHEDULE + SWARM

For each wave, in dependency order:
- Spin up to `MAX_PARALLEL` work agents, **one isolated worktree each** (`git worktree add` branched from `TARGET_BRANCH`; or a Conductor VM in `vm` mode). Use each ticket's `gitBranchName` for its work branch.
- Give each agent the **Work Agent Brief** (section 6). Same-wave agents run concurrently and must stay within their declared surface to minimize conflicts.
- Collect reports. Update the ledger with `REVIEW_CLEAN` status and the PR URL. A ticket is not `REVIEW_CLEAN` until the agent reports `toolkit_review: remaining=0`. Do not start a dependent wave until its blockers are `REVIEW_CLEAN` or `PR_MERGED`.
- `BLOCKED` / `NEEDS_DECISION` -> park the ticket, surface the one-liner to the user, continue with the rest.

**Epic re-poll (continuous backfill).** Work agents and the QA agent file new child tickets under the epic mid-run; pick them up without being told:
- **When:** at every phase boundary and every time an agent reports `DONE` or a PR is merged. Cheap - metadata only.
- **How:** `list_issues` with `parentId = LINEAR_TASK_ID`; diff the returned ids against the ledger. Any id not already in the ledger is new work. An agent report's `followups` field is the fast path - schedule those immediately rather than waiting for the next poll.
- **Then:** add each new ticket as a ledger row (and to the Backfill queue), assign it to a fresh wave, and run it through the same SWARM -> PR REVIEW -> PR MERGE pipeline. Honor `blocks`/`blockedBy` so a backfill ticket blocked by in-flight work waits for its blockers.
- **Gate coupling:** epic closure (5.5) cannot proceed while any un-scheduled or in-flight `follow-up`/`qa-found` ticket exists. Drain the epic to empty first.

### 5.3 PR REVIEW + FIX (owned entirely by the work agent)

The orchestrator does **not** run reviews, accumulate findings, or dispatch fix agents. That is the work agent's job. See Work Agent Brief (section 6) for the full loop; the orchestrator only verifies the outcome:

- Agent report has `pr: <URL>` (real URL, not "none")
- Agent report has `toolkit_review: remaining=0`
- CI is green on the PR (check via `gh pr checks <PR>`)

If `toolkit_review: remaining=N` with N > 0, it means `MAX_REVIEW_ROUNDS` was hit. Mark the ticket `REVIEW_FAIL` in the ledger and escalate to the user with the PR URL and the agent's `remaining:` note. Do not dispatch fix agents yourself.

If CI is red after the agent reports DONE (a race with a post-push check): dispatch a **targeted Fix agent** on that sub-worktree with the specific failing CI output (handle only - no raw logs). The fix agent re-runs `pr-review-toolkit` on the PR after fixing, then reports back. The orchestrator only reads the fix agent's report block.

### 5.4 PR MERGE (topological order)

Merge ticket PRs in topological order (no PR is merged before all its `blocks`/`blockedBy` deps are `PR_MERGED`):

1. For each eligible PR: run `gh pr merge <PR> --squash --delete-branch` (or `--merge` per repo convention). Verify CI is green before merging; abort and surface the failure if red.
2. **Conflict on merge:** if `gh pr merge` fails due to conflict (the PR branch is behind `TARGET_BRANCH` after previous merges):
   - Dispatch a **Rebase agent** (`model: sonnet` - the rebase is mechanical and structural conflicts escalate anyway) on that ticket's sub-worktree: `git fetch origin {{TARGET_BRANCH}} && git rebase origin/{{TARGET_BRANCH}}`, resolve conflicts, run typecheck/lint/tests, force-push the branch.
   - The Rebase agent **must** re-run `pr-review-toolkit` on the PR after force-pushing. Fix any new findings. Return a standard Agent Report block with `toolkit_review: remaining=0` before the orchestrator retries the merge.
   - Structural/ambiguous conflict -> stop, escalate to the user.
3. Mark the ticket `PR_MERGED` in the ledger. Collapse the row.
4. **Sync the orchestrator branch with `TARGET_BRANCH`** after each merge: `git fetch origin && git rebase origin/{{TARGET_BRANCH}}`. Resolve any conflicts directly - do not dispatch an agent for this. The orchestrator branch carries no code changes, so conflicts are rare and always local (e.g. a merge commit touching a file the orchestrator branch also touched). Resolve, then continue.
5. After each merge, re-poll the epic (5.2) for new backfill tickets.

### 5.5 EPIC CLOSE

Runs only when all ledger rows are `PR_MERGED` and the Backfill queue is empty.

1. **Final re-poll:** run `list_issues` with `parentId = LINEAR_TASK_ID` one last time. Any new ticket found resets to SWARM phase for that ticket - do not skip it. Only proceed if the re-poll returns nothing new.
2. **Close the epic** in Linear: `save_issue` with `id = LINEAR_TASK_ID`, `stateId = <done state id>` (resolve via `list_issue_statuses` for the team). Add a comment via `save_comment` summarizing what shipped.
3. If `AUTO_MERGE=false` (PRs opened but not merged), skip epic closure and instead ping the user with all PR URLs for manual review and merge.

### 5.6 CLEANUP (always, even on partial failure)

- Remove every **sub-worktree** the swarm created (`git worktree remove <path> --force`). Do **not** touch the orchestrator worktree - the user manages that.
- Delete merged ticket branches: `git branch -d <branch>` locally (the remote branch is deleted by `--delete-branch` at merge time).
- Verify `git worktree list` shows only the orchestrator worktree and the tree is clean.
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
> 4. **Verify** locally - discover the repo's scripts (package.json/turbo.json/Makefile) and run typecheck, lint, unit tests, and build. All must pass before opening the PR.
> 5. **Open a PR first.** Push your branch and run `gh pr create --base {{TARGET_BRANCH}} --title "{{TICKET}}: <short title>" --body "<acceptance criteria + test plan>"`. The PR URL is required in your report. `pr-review-toolkit` cannot run without an open PR - do not skip this step.
> 6. **Run `pr-review-toolkit` on the PR.** Use the PR URL from step 5. Fix **every** finding yourself - no finding may remain. After each fix batch: re-run typecheck/lint/tests, force-push to update the PR, then re-run `pr-review-toolkit`. Repeat until findings = 0. Count your rounds.
>    - If you hit `{{MAX_REVIEW_ROUNDS}}` with findings still remaining: stop the loop, note what is left and why in `remaining:`, report `BLOCKED`.
>    - Never skip or abbreviate `pr-review-toolkit`. Every finding must be addressed.
> 7. Do **not** merge the PR yourself. Do **not** touch `main` or the orchestrator branch. Do **not** rebase onto sibling ticket branches.
> 8. End with the **Agent Report** block (section 4) and nothing more. `pr:` must be the real PR URL. `toolkit_review:` must reflect actual rounds run and remaining count. Keep prose minimal - only the block is read.

## 7. Background QA Agent (separate, continuous, off the critical path)

Runs in its own loop, independent of the orchestrator's phase machine. Dispatch with `model: sonnet` - it runs for the entire epic, so it is the largest cumulative spend, and it never edits application code:
1. Pull the latest `TARGET_BRANCH`, stand up the app, run smoke tests + exploratory QA flows.
2. For each defect, **file a Linear ticket** with `save_issue`: `parentId = LINEAR_TASK_ID`, `team = <confirmed team>`, `labels = ["qa-found", <severity>]`, `description = <repro steps + failing commit>`. Confirm labels exist first via `list_issue_labels`; create missing ones via `create_issue_label`.
3. Loop continuously; never block the swarm.

When `QA_FEEDBACK=true`, `qa-found` tickets are drained through the same epic re-poll/backfill mechanism as agent follow-ups (5.2): scheduled as a new wave before epic closure. Epic closure cannot proceed while untriaged `qa-found` tickets exist.

## 8. Guardrails

- **Parallelism:** never exceed `MAX_PARALLEL` live agents; never two agents on the same file.
- **Model tiers:** clerical agents run on cheaper models - Planner, QA, and Rebase agents on `sonnet`; verification one-liners on `haiku`. Work agents and CI Fix agents inherit the session default - never downgrade an agent that writes application code.
- **Idempotency:** agents must be resumable - re-running on a partially-done branch is safe. Step 0's branch confirmation is idempotent.
- **No silent scope creep:** unclear ticket -> `NEEDS_DECISION`, not improvisation.
- **Comprehensive over minimal:** within scope, agents take the fullest correct solution, never a band-aid. Scope-expanding ideas become Linear `follow-up` tickets under the epic, never silent inline expansion.
- **Backfill discipline:** re-poll the epic at every phase boundary and on every PR merge; schedule new `follow-up`/`qa-found` tickets as fresh waves before epic closure.
- **No red merges, no force-push to `main`, no permission/settings changes.**
- **No manual branch merges:** ticket branches land via `gh pr merge` only - the orchestrator never runs `git merge` on ticket branches.
- **Escalation:** anything still failing after `MAX_REVIEW_ROUNDS`, or any structural rebase conflict, stops and pings the user with a one-line summary (post via `save_comment` on the epic if useful) - do not grind.
- **Context:** if any sub-agent report exceeds the schema, discard and re-dispatch. Protect your context.

## 9. Final Report (to the user)

A single compact summary: task id, tickets shipped vs blocked, all PR URLs (merged or open), review rounds used per ticket, follow-up tickets filed/scheduled, QA bugs filed/fixed, epic closed in Linear (y/n), sub-worktrees cleaned (y/n), and any escalations needing a decision. No transcripts.

## Quick Reference

| Phase | Who runs | What the orchestrator keeps |
|-------|----------|-----------------------------|
| Step 0 confirm branch | Main thread (git + Linear MCP) | epic id, title, current branch |
| Step 1 param gate | Main thread + user | `AUTO_MERGE`, `MAX_PARALLEL`, team |
| PLAN | Planner agent (`sonnet`) | DAG + waves + per-ticket branches |
| SWARM | Work agents (parallel) | Agent Report blocks + PR URLs |
| Epic re-poll | Main thread (Linear MCP) | new ticket ids -> Backfill queue |
| PR REVIEW/FIX | Review + Fix agents (parallel per ticket) | Open findings list |
| PR MERGE | Main thread (`gh pr merge`, topo order) | PR merge status |
| EPIC CLOSE | Main thread (Linear MCP) | closure confirmation |
| CLEANUP | Main thread | sub-worktree-clean boolean |
| QA (background) | QA agent (continuous, `sonnet`) | QA bugs list |

## Common Mistakes

- **Reading code/diffs/logs into your own context.** The single failure mode. Delegate every heavy read; consume only report blocks.
- **Manually merging ticket branches into the orchestrator worktree.** Never run `git merge` on a ticket branch. All merges happen via `gh pr merge`. The orchestrator worktree's branch is never modified.
- **Dispatching fix agents from the orchestrator.** The orchestrator never accumulates findings or dispatches fix agents. The work agent owns the full review-fix loop. If findings remain, the ticket is `REVIEW_FAIL` and escalated to the user - not silently repaired by the orchestrator.
- **Accepting a report without a real PR URL.** A `pr: none` response means the agent skipped PR creation, which means `pr-review-toolkit` never ran. Discard the report, re-dispatch with the protocol restated.
- **Running `pr-review-toolkit` before the PR exists.** The skill requires an open PR. The work agent must push and create the PR before invoking the toolkit - not after.
- **Merging on `AUTO_MERGE=false`.** That mode stops at open PRs. Merging is a different, explicit setting.
- **Closing the epic before the final re-poll.** Always run one last `list_issues parentId=LINEAR_TASK_ID` and confirm it returns nothing new before calling `save_issue` to close.
- **Merging with untriaged `qa-found` tickets.** When `QA_FEEDBACK=true`, epic closure is blocked until the QA queue is drained.
- **Two agents on the same file.** Partition fix/work agents by file. Overlap creates conflicts you then pay to resolve.
- **Improvising on an unclear ticket.** Return `NEEDS_DECISION` and escalate. Scope creep is not yours to grant.
- **Shipping a band-aid.** Within scope the agent takes the most comprehensive correct fix; the only reason to go narrow is genuine scope expansion, which becomes a `follow-up` ticket, not a smaller patch.
- **Forgetting to re-poll the epic.** Agents file follow-up tickets mid-run; if the orchestrator does not re-poll `list_issues parentId=LINEAR_TASK_ID`, that work is silently dropped and the epic ships incomplete.
- **Deleting the orchestrator worktree.** Cleanup only removes sub-worktrees spawned by the swarm. The orchestrator worktree is the user's responsibility.
- **Skipping cleanup on failure.** Cleanup runs always, even on partial failure - leftover worktrees and branches pollute the next run.
