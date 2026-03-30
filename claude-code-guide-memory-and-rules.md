# Claude Code Guide: Memory and Rules

> Superseded by `claude-code-guide.md`, which is now the canonical guide. Keep this file only as source material or supporting notes.

Last verified: **March 30, 2026**

This section covers the part of Claude Code that most teams loosely call "rules." The important nuance is that not all rules are the same.

## What "Rules" Actually Means

In practice there are three different categories:

- advisory instructions: `CLAUDE.md`, imported memory, child `CLAUDE.md`, and in some setups `.claude/rules/`
- workflow instructions: skills
- enforceable policy: hooks, sandboxing, and permissions

If you write everything into memory and expect perfect compliance, you are using the wrong layer.

## Start With `CLAUDE.md`

`CLAUDE.md` is still the highest-leverage customization point.

The official best-practices guidance is consistent here:

- put repo-specific commands, workflow constraints, and architecture rules in `CLAUDE.md`
- keep it concise and specific
- focus on information Claude cannot infer reliably from the codebase alone
- use `/init` as a starting point, then aggressively prune it

The best root file is usually a router, not a handbook.

## What Claude Code Loads

Claude Code can draw from multiple memory layers, including:

- project memory in `./CLAUDE.md`
- user memory in `~/.claude/CLAUDE.md`
- child `CLAUDE.md` files when Claude works inside that subtree
- imported memory via `@path/to/file`
- additional managed or enterprise memory, where configured
- auto-generated memory

Practical implication:

- keep the root file small
- use imports for deeper reference material
- use child `CLAUDE.md` files only where subtree behavior actually differs

## Keep Memory Short

The local guide's original advice still holds: memory quality matters more than memory volume.

Good memory is:

- specific
- repo-specific
- actionable
- short enough that Claude keeps following it

Bad memory is:

- generic coding advice
- long style guides already enforced by tooling
- rare workflows that should be skills
- requirements that really need hooks or tests

## What Belongs In The Root File

Put these in the root `CLAUDE.md`:

- canonical test, lint, typecheck, and formatting commands
- major architecture boundaries
- generated-file and secrets warnings
- "never touch these files" rules
- what "done" means in this repo
- links or imports to deeper material only when needed

Keep these out of the root file:

- detailed migration procedures
- long troubleshooting playbooks
- domain references that only matter occasionally
- checklists you repeat often
- anything that must be enforced mechanically

## A Good Pattern

Use the root file as a routing layer:

```md
# Project Memory

## Commands
- Test: `pnpm test`
- Lint: `pnpm lint`
- Typecheck: `pnpm typecheck`

## Architecture
- API handlers live in `apps/api/src/routes`
- Shared domain types live in `packages/domain`

## Hard Rules
- Do not edit generated files in `src/generated/`
- Run targeted tests for touched packages before finishing

## More Context
- Frontend conventions: @./docs/frontend.md
- Database workflow: @./docs/database.md
```

That gives Claude a high-signal map instead of an instruction swamp.

## Advisory Versus Enforceable Rules

This is the most important nuance from the additional research.

Advisory rules work well for:

- naming conventions
- architectural boundaries
- which commands to run
- repo-specific "how we usually do this"

Enforceable rules should not stay as prose alone. If something must happen, move it into:

- a hook
- a test or script
- a permission policy
- a sandbox restriction

When Claude violates a rule, the fix is often not "write a stronger paragraph." It is:

- shrink the prompt
- create a skill
- add a hook
- improve verification

## The `.claude/rules/` Nuance

The original guide treated `.claude/rules/` as an on-demand or targeted rule layer. That is a sensible organizational pattern and it matches how many community examples structure larger Claude Code setups.

The nuance from the current public docs is that `CLAUDE.md`, child `CLAUDE.md`, imports, skills, hooks, and permissions are much more prominently documented than a dedicated `.claude/rules/` reference page.

My practical recommendation:

- use `.claude/rules/` only if it fits your repo organization well
- treat it as a convenience layer, not a magical new enforcement system
- verify exact behavior in your installed version before standardizing advanced assumptions

Examples of assumptions worth verifying before you depend on them:

- whether a rule file loads eagerly or lazily
- how it interacts with imported memory
- precedence between user-level memory and project-level rule files

## When To Move Something Out Of Memory

Move content out of memory when it is:

- repeated often enough to deserve a skill
- mandatory enough to deserve a hook
- local enough to belong in a child `CLAUDE.md`
- long enough to dilute the root file

The wrong move is turning every problem into more root memory.

## Practical Setup Ideas

### Small team

- keep the root `CLAUDE.md` under real pressure to stay short
- add one or two child `CLAUDE.md` files only where conventions differ
- convert repeated review, debug, and release tasks into skills
- use hooks for verification and guardrails

### Monorepo

- keep the root file tiny
- move package-specific guidance into child `CLAUDE.md` files
- avoid loading frontend, backend, infrastructure, and release detail into every session

### Review-heavy team

- define what a good review means in memory
- turn the actual review checklist into a skill
- enforce the non-negotiables with hooks or scripts

## Sources

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Community auto-updated Claude Code guide](https://github.com/Cranot/claude-code-guide)
- [Community rules framework example](https://github.com/sahin/ai-rules)

Use the community examples for ideas, not as the source of truth.
