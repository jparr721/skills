# skills

A collection of Claude Code skills. Each lives in its own directory with a `SKILL.md`.

## Skills

| Skill | Use it when |
|-------|-------------|
| [`dark-factory`](dark-factory/SKILL.md) | You want to drive an entire Linear epic (or task) to merged-on-main autonomously - a swarm of implement/review/fix/integrate sub-agents plus a background QA agent, orchestrated under a strict context firewall. Takes a Linear task ID, renames the branch to Linear's `gitBranchName`. Changes code and can merge to `main`. |
| [`architecture-audit`](architecture-audit/SKILL.md) | You want a scoped, multi-agent architectural audit of one subsystem - coupling, boundaries, data flow, dependency direction, testability, complexity. Outputs prioritized, TDD-ready tasks. Read-only. |
| [`nextjs-code-quality-audit`](nextjs-code-quality-audit/SKILL.md) | You want a thorough code-quality audit of a Next.js codebase - refactoring opportunities, misplaced concerns, DRY violations, missing tests, structural issues. Read-only. |
| [`elysia-code-quality-audit`](elysia-code-quality-audit/SKILL.md) | You want a thorough code-quality audit of an Elysia (Bun) backend - plugin/scope misuse, missing schema validation, DRY violations, security, tests. Tuned for `apps/` monorepos with Drizzle, Better Auth, pg-boss. Read-only. |
| [`vite-tauri-code-quality-audit`](vite-tauri-code-quality-audit/SKILL.md) | You want a thorough code-quality audit of a Vite + Tauri codebase - IPC boundary issues, misplaced concerns, DRY violations, bundle/build problems, Tauri security misconfig. Read-only. |

## Install (link a skill)

There is no install script. Open Claude Code in this repo and ask it to link the skill you want:

```
Link the dark-factory skill to my global registry.
```

Or, to pick from the list:

```
Which skills can I install from this repo? Link the ones I choose.
```

Claude Code does the work: it symlinks the chosen skill directory into your global registry at `~/.claude/skills/`, so the skill is available in every project without copying files.

## What that does under the hood

For each skill you pick, the link is a single symlink:

```bash
ln -s "$PWD/<skill-name>" ~/.claude/skills/<skill-name>
```

Because it is a symlink (not a copy), editing the skill here updates the linked version everywhere immediately.

- **Verify a link:** `ls -la ~/.claude/skills/<skill-name>`
- **Unlink a skill:** `rm ~/.claude/skills/<skill-name>` (removes only the symlink, never the source)
