# Architecture Audit Skill - Design

**Date**: 2026-05-18
**Status**: Approved design, pre-implementation

## Purpose

A skill that performs a scoped, multi-agent architectural audit of a single subsystem within a large codebase. Acts as a team of senior architects reviewing one concern at a time. Outputs prioritized, TDD-ready refactoring tasks. Never changes application code.

Complements the existing `*-code-quality-audit` skills:

- The existing skills audit a whole codebase for code-quality issues (DRY, readability, structure, placement, tests).
- This skill audits a single subsystem for architectural issues (boundaries, coupling, data flow, dependency direction, testability, cognition) with multi-agent consensus.

## When to Use

- Before refactoring a large subsystem
- When a subsystem "feels architecturally off" but the user lacks a concrete diagnosis
- During onboarding to a complex piece of an unfamiliar codebase
- When deciding whether a piece is worth refactoring at all

## Hard Constraints

1. **No application code changes by the skill.** Output is findings and TDD-ready tasks. Implementation is the user's next step.
2. **No findings against files outside the user-confirmed scope.** Read-only peeks are allowed under defined rules; new findings against peeked files are not.
3. **No single-agent findings in the main report** except where severity is Critical and the agent provides strong evidence. Everything else needs at least two agents to corroborate.

## High-Level Pipeline

```
1. Subsystem intake     - user names the subsystem in plain English
2. Stack detection      - Agent 0 (Sonnet) detects stack and stack-specific concerns
   Scope discovery      - Discovery agent resolves subsystem to a file set
   (these run in parallel)
3. User confirmation    - user reviews/edits file list, sets severity threshold; blocks audit
4. Parallel audit       - 5 architect agents (Opus), each with a distinct lens
5. Consensus synthesis  - finding kept only if >=2 agents agree (or 1 + Critical)
6. Output               - prioritized findings with TDD plans, printed to conversation
7. Prioritization       - user chooses which findings to act on. Skill stops.
```

## Components

### Agent 0 - Stack Scout (Sonnet, fast)

**Why a separate fast agent**: stack detection is mechanical and cheap. Using Sonnet here avoids spending Opus tokens on file-extension counting and manifest parsing. Its output gates the prompts of the expensive parallel agents.

Reads in a single pass:

- Root manifests: `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `Gemfile`, `composer.json`, `pom.xml`, `build.gradle`, etc.
- Root config: `next.config.*`, `vite.config.*`, `tsconfig.json`, `tauri.conf.json`, framework markers
- File extension sampling across `src/` or repo root

Returns a structured block consumed by all later agents:

```
stack: "next.js" | "elysia" | "rust" | "vite-tauri" | "django" | "rails" | "unknown"
framework_version: "16.0.1" | null
stack_specific_concerns:
  - "auth checks belong in middleware.ts, not page components"
  - "RSC/client boundary leaks via client-only imports in server components"
  - "route handlers should delegate to a service layer"
```

If stack is `unknown`, the audit proceeds with universal lenses only.

### Discovery Agent (runs in parallel with Agent 0)

Input: user's plain-English subsystem description.

Process:

1. Glob/grep for entry-point candidates (filenames, exports, route patterns matching the description).
2. Read entry points and walk imports one level deep.
3. Pull in associated test files.
4. Categorize results.

Output categories:

- **Core** - clearly part of the subsystem.
- **Adjacent / called-into** - dependencies of core that the subsystem relies on.
- **Tests** - test files exercising any of the above.
- **Possibly related but uncertain** - referenced by core, used elsewhere too, unclear if in-scope.

### User Confirmation Gate (blocking)

The skill presents the four-bucket file list and asks the user to:

- Confirm or edit the file list (drop, add, promote uncertain to core).
- Set a severity threshold (default: Moderate and above; Minor dropped).
- Optionally name focus areas to emphasize or skip.

No audit agent runs until the user confirms. This is the primary off-the-rails guard.

### Architect Agents (Opus, parallel, 5 lenses)

Each receives:

- The confirmed file list.
- Agent 0's `stack_specific_concerns` appended to its lens prompt.
- The severity threshold.

| # | Lens | Looks for |
|---|------|-----------|
| 1 | Boundaries & coupling | Layer violations, leaky abstractions, circular deps, oversized public surface, cross-cutting concerns leaking into core logic |
| 2 | Data flow & state | Where state lives vs. where it mutates, hidden side effects, implicit coupling via shared mutable state, response/request shape drift |
| 3 | Dependency direction | Inversion violations, god modules, high-incoming-edges hot spots, stable/volatile mismatch |
| 4 | Testability & seams | Untestable units, missing seams, hard-to-mock dependencies, branch-coverage gaps inside the subsystem, integration vs unit balance |
| 5 | Complexity & cognition | Parallel hierarchies, primitive obsession, feature envy across modules, abstractions that do not earn their weight |

Each agent returns findings with:

- ID, files + line ranges, severity (Critical / Moderate / Minor), the architectural principle violated, 1-3 evidence quotes, a proposed direction (not a diff), confidence (low/med/high).

#### Scope Contract for Architect Agents

- **Findings only on confirmed in-scope files.** Out-of-scope files cannot be the subject of a finding.
- **Read-only peek allowed** when an in-scope file references an out-of-scope file and the agent suspects it materially affects the analysis. The agent may open that file to confirm or refute the suspicion.
- **Peek budget**: 5 files per agent. Soft cap.
- **Peek reporting**: each agent reports `{file, why_peeked, did_it_change_analysis}` in a Scope Notes section.
- **Auto scope-expansion signal**: if >=2 agents peeked at the same out-of-scope file with a relevance reason, the synthesizer surfaces it as a "Consider re-running with expanded scope" note at the top of the output.

### Synthesis Pass

Runs after all 5 architect agents return.

Rules:

1. Findings that overlap by file/line/concern are merged into one with the union of evidence and the maximum severity.
2. A merged finding survives only if backed by >=2 distinct agents. Exception: a single-agent Critical finding survives but is tagged "single-agent".
3. Dropped findings are listed in a "Dropped Findings" section for transparency.
4. Findings below the user's severity threshold are dropped silently (already filtered).
5. Each surviving finding gets a TDD plan:
   - **Red**: name the failing test to write (location + assertion shape).
   - **Green**: the minimal change that makes it pass.
   - **Refactor**: any follow-up cleanup once green.

### Output Format

Printed directly to the conversation. No file written. Matches the convention of the existing `*-code-quality-audit` skills.

```markdown
# Architecture Audit - <Subsystem Name>

> No skill action changed application code. The implementor owns all edits.

## Summary
- Subsystem: <name>
- Stack: <detected stack + version>
- Files in scope: <N> (Core <a>, Adjacent <b>, Tests <c>)
- Findings: <N> (Critical <a>, Moderate <b>, Minor <c>)
- Scope expansion suggested: <list of files, or "none">
- Consensus stats: <kept> surviving / <raw> raw / <dropped> dropped as single-agent

## Critical
### [ARCH-001] <title>
- Files: <path:lines>, <path:lines>
- Flagged by: <lenses that agreed> (<n>/5)
- Principle: <which architectural rule is violated>
- Evidence:
  > <quote 1>
  > <quote 2>
- Direction: <proposed shape, no diff>
- Confidence: <low|med|high>

**TDD plan**:
1. Red: <failing test to write>
2. Green: <smallest change to pass>
3. Refactor: <cleanup once green>

## Moderate
(same shape)

## Minor
(same shape)

## Scope Notes
- <file> - peeked by <agents>. <relevance>. <recommendation>.

## Dropped Findings (single-agent, non-critical)
- <agent>: <one-line summary>. Not corroborated.
```

### Prioritization Step

After the output, the skill asks: "Which findings do you want to act on, and in what order?"

The user replies with selected IDs. The skill confirms back the prioritized list and then stops. It does not edit files. The user takes the list to their next session or invokes `writing-plans` themselves.

## What This Skill Is Not

- Not a whole-codebase auditor (use the existing `*-code-quality-audit` skills for that).
- Not an implementation tool (no edits, no commits).
- Not a code-style or linter replacement.
- Not a security review (use `security-review` for that).

## Open Questions

None outstanding at design approval time.

## File Layout

```
architecture-audit/
  SKILL.md          # frontmatter + full skill content
```

Single file, same shape as the existing audit skills in this repo.
