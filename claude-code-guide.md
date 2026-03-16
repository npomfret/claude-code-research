# Claude Code: Current Practical Guide

Claude Code changes quickly. This guide intentionally prioritizes Anthropic's current documentation, official release notes, and the official `anthropics/claude-code` repo over community lore. Where the product surface is still evolving, the guidance below favors stable patterns that materially improve results.

Last verified: **March 16, 2026**

---

## What Actually Gets Better Results

If your goal is to get the best work out of Claude Code, the highest-leverage improvements are not obscure flags. They are:

1. **Give Claude the right context, not more context.**
2. **Keep project memory concise and current.**
3. **Use the right execution layer for the job: memory, skill, subagent, hook, MCP, or CLI.**
4. **Force verification with tests, checks, and explicit acceptance criteria.**
5. **Manage context aggressively in long sessions.**

Anthropic's own best-practices docs repeatedly emphasize a few operational habits:

- be specific about the task, constraints, and success criteria
- attach the exact files, images, or URLs Claude needs
- break large work into steps or use `/plan`
- create concise project memory with `/init`
- use hooks for deterministic guardrails
- use skills and subagents to keep specialized knowledge off the always-on path
- use `/compact`, `/clear`, `/btw`, `/rewind`, and resume flows instead of dragging one session forever

That is the core playbook. Everything else is an optimization on top.

---

## The Current Mental Model

Use this decision table when choosing a Claude Code feature.

| You need... | Use | Why |
|---|---|---|
| Always-on project instructions | `CLAUDE.md` and `.claude/rules/` | Persistent memory layer |
| A reusable workflow Claude can invoke or you can call by name | **Skill** | Skills now cover both model-invoked and user-invoked custom workflows |
| A specialist with isolated context | **Subagent** | Better focus, lower context pollution |
| Deterministic enforcement or automation | **Hook** | Guardrails and lifecycle automation |
| External tools or live data | **MCP server** | Integrates APIs, docs, systems |
| Share setup across teams/repos | **Plugin** | Packages skills, agents, hooks, MCP, commands, settings |
| Non-interactive automation | `claude -p` | CI, scripts, batch workflows |

The main product change to internalize:

- **Built-in slash commands** still exist.
- **Custom commands are now implemented through skills.**
- A skill can be **model-invocable**, **user-invocable**, or both.

That means the old "custom slash commands vs skills" framing is now out of date.

---

## Start With Memory

`CLAUDE.md` is still the highest-leverage customization point, but the fine details matter.

### What Claude Code Loads

Claude Code supports multiple memory layers:

- project memory in `./CLAUDE.md`
- user memory in `~/.claude/CLAUDE.md`
- enterprise or managed memory, if configured
- imported memory files
- additional memory directories via environment configuration
- auto-generated memory
- on-demand memory in `.claude/rules/`

Anthropic's current memory docs describe `CLAUDE.md` as the place for repo-specific standards, commands, and workflows, while `.claude/rules/` is for more targeted rules that should load on demand.

### Keep Memory Short

Anthropic's current guidance is explicit:

- **keep each `CLAUDE.md` under roughly 200 lines**
- use **imports** to split large memory into smaller files
- use **child `CLAUDE.md` files** or `.claude/rules/` for more local context
- keep guidance **specific, actionable, and repo-specific**

This is the right way to think about instruction budget:

- `CLAUDE.md` should not try to teach programming
- it should tell Claude how *this repository* works
- if something is long, niche, or only occasionally relevant, move it off the always-on path

### What Belongs in `CLAUDE.md`

- build, test, lint, and formatting commands
- where major code lives
- repo-specific architecture constraints
- "never touch these files" rules
- local workflow requirements that matter on almost every task
- links or imports to more detailed memory only when needed

### What Does Not Belong There

- generic coding advice
- long style guides already enforced by tools
- rare one-off workflows
- deep domain references better stored in a skill or rule file
- anything that must be enforced mechanically rather than requested

### Fine Details That Matter

- Child `CLAUDE.md` files load when Claude enters that directory subtree.
- `.claude/rules/` exists specifically to reduce always-on prompt bloat.
- Auto-memory exists, but you should still treat hand-written memory as the authoritative layer.
- You can exclude imported memory from child directories with `claudeMdExcludes` if a local subtree should not inherit it.

### Good Memory Pattern

Keep the root file as a routing layer:

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
- Frontend conventions: import `./.claude/rules/frontend.md`
- Database workflow: import `./.claude/rules/db.md`
```

That gives Claude a small, high-signal map instead of an instruction swamp.

---

## Skills: The Current Model

Skills are now the main mechanism for reusable custom workflows in Claude Code.

### What Changed

Current docs state that:

- **custom slash commands have been merged into the skill system**
- skills can be **model-invocable**, **user-invocable**, or both
- built-in slash commands remain separate
- existing `.claude/commands/` content still works, but **skills take precedence** over commands with the same name

If you want a reusable workflow in 2026 Claude Code, start with a skill.

### Where Skills Live

- project skills: `.claude/skills/`
- personal skills: `~/.claude/skills/`
- nested directories are supported and auto-discovered
- plugins can also provide skills

### Invocation Modes

A skill can be:

- **model-invocable**: Claude chooses it automatically when relevant
- **user-invocable**: you run it directly, typically as `/<skill-name>`
- **both**: often the best default

Important frontmatter details from the current docs:

- `description` is used for automatic invocation
- `user-invocable: true` exposes it for direct use
- `disable-model-invocation: true` removes it from automatic routing
- `allowed-tools` constrains what the skill can use
- `model` lets you pin a model for that skill
- `context: fork` lets a skill run in a fresh context, optionally with an `agent`

### Why This Matters

The best custom workflows usually need one of these:

- a structured checklist
- reference docs
- scripts or templates
- strict tool restrictions
- a model override
- a clean context boundary

That is exactly what skills are built for now.

### Best Practices For Reliable Skill Routing

If you want Claude to auto-invoke a skill reliably:

- write a concrete `description` that names the task, artifacts, and triggers
- include explicit "use when..." language
- include "do not use for..." when nearby skills overlap
- keep the main `SKILL.md` focused and move bulky detail into references or scripts
- prefer one strong skill per recurring workflow instead of many tiny overlapping ones

Example:

```yaml
---
description: Investigate failing tests, CI regressions, or runtime errors. Use when the user asks to debug a failure, triage a flaky test, or explain why a command or suite is breaking. Do not use for greenfield feature work.
user-invocable: true
allowed-tools: Read,Grep,Glob,Bash(npm test:*),Bash(pnpm test:*)
---
```

### Fine Details Worth Knowing

- `!` commands inside a skill invoke shell commands from the skill body.
- `$ARGUMENTS` is available for user-supplied arguments.
- Skills can reference hooks and subagents.
- Setting `disable-model-invocation: true` also means the description is not included for routing.
- If a skill name collides with a built-in command, the slash form resolves to the built-in. Anthropic's docs recommend a prefix such as `sc-` for direct-invoke custom skills.

### Practical Advice

Use skills for:

- code review checklists
- debugging and triage flows
- release verification
- migration runbooks
- repetitive repo-specific changes

Do not use a skill for something that should be:

- always-on memory
- a hard guardrail
- just a single one-off prompt

---

## Subagents

Subagents are still one of the best ways to improve output quality on complex work because they reduce context collision.

### What They Are

Subagents are specialized Claude instances with their own instructions, tools, permissions, and context. Claude Code includes built-in subagents such as `Explore`, `Plan`, and a general-purpose agent, and you can define custom subagents in `.claude/agents/` or `~/.claude/agents/`.

Current docs highlight a few capabilities that matter in practice:

- subagents have separate context windows
- they can have their own `allowed-tools`, `model`, and `permission-mode`
- they can preload relevant skills
- a resumed subagent keeps its prior context
- the current SDK also exposes `Task()` for custom agent orchestration

### When Subagents Improve Results

Use subagents when the task benefits from isolation:

- broad codebase research
- independent reviews from different perspectives
- planning before implementation
- parallel investigation of separate subsystems
- specialist flows with distinct tool permissions

### When They Usually Hurt

Avoid unnecessary delegation when:

- one file or one function is changing
- the work depends on tight shared reasoning across the same files
- the overhead of explaining context is larger than the task itself

### Fine Details That Matter

- A subagent only works with the context you give it and the tools it is allowed to use.
- A good subagent description is as important as a good skill description.
- If you want predictable delegation, make the specialization narrow and obvious.
- For cost and speed control, it is often worth keeping the main session on a stronger model and simpler research/review agents on a cheaper one.

### Good Uses

- `Explore`: find all call sites, data flow, or migration scope
- `Plan`: generate a plan you can approve before edits
- custom reviewer agents: security, performance, accessibility, schema review

Anthropic's best-practices doc also recommends running multiple Claude Code sessions or worktrees in parallel for larger efforts. Subagents are one part of that broader pattern.

---

## Hooks: Use Them For Guarantees

Hooks are the right tool when behavior must be automated or enforced.

### What They Do

Hooks can run:

- shell commands
- HTTP endpoints
- LLM-based checks

They trigger on Claude Code lifecycle events and can inspect tool inputs, block actions, modify behavior, or attach context.

### Current Hook Surface

Anthropic's current hooks docs list many events, including:

- `SessionStart`
- `UserPromptSubmit`
- `PreToolUse`
- `PermissionRequest`
- `PostToolUse`
- `PostToolUseFailure`
- `Notification`
- `SubagentStart`
- `SubagentStop`
- `Stop`
- `InstructionsLoaded`
- `ConfigChange`
- `PreCompact`
- `PostCompact`
- `SessionEnd`

The hook surface is broader now than older guides suggest, so avoid hardcoding an old event count.

### Best Uses

- block dangerous Bash commands
- run formatters or targeted checks after edits
- inject compact repo state or reminders at session start
- send notifications when a task finishes
- enforce branch protections or policy checks

### Best Practices

- keep hooks fast and deterministic
- make blocking hooks narrow and well-tested
- use `async` for non-blocking background work
- prefer hooks for enforcement, not semantic routing
- review hook output so Claude sees what changed

### Fine Details Worth Knowing

- Hooks are available globally, per project, and inside skills and agents.
- You can inspect and manage them with `/hooks`.
- Hook handlers receive JSON input and can return structured decisions.
- Project hooks are snapshotted when Claude Code launches; use `/hooks` to review and apply changes after editing them externally.

### What Not To Do

- do not move complex reasoning into shell-based hook classifiers
- do not attach heavyweight checks to every lifecycle event
- do not let silent code-changing hooks mutate files without clear output

The strongest pattern is simple:

- memory tells Claude how the repo works
- skills and subagents provide reusable expertise
- hooks enforce what must actually happen

---

## Built-In Commands That Matter

Built-in slash commands are still essential for interactive work. The specific command surface can change, but these are the ones worth knowing well.

### Session and Context Management

- `/compact`: compress the conversation when context is getting noisy
- `/clear`: start fresh for a new task
- `/rewind`: revert to a checkpoint and branch from there
- `/resume`: return to an earlier session
- `/btw`: attach "by the way" context without derailing the main flow
- `/context`: inspect current context usage

### Setup and Memory

- `/init`: generate an initial `CLAUDE.md`
- `/memory`: edit memory files

### Delegation and Extensibility

- `/plan`: plan before acting
- `/agents`: manage custom subagents
- `/skills`: browse or manage skills
- `/hooks`: manage hooks
- `/mcp`: manage MCP servers
- `/plugin`: manage plugins
- `/reload-plugins`: reload local plugin changes during development

### Safety, Tooling, and Visibility

- `/permissions`: review or adjust allowed actions
- `/sandbox`: inspect or configure sandboxing
- `/usage`: inspect usage
- `/cost`: inspect API spend
- `/release-notes`: open current release notes
- `/doctor`: troubleshoot installation or environment issues
- `/model`: switch models

### One Important Deprecation

Anthropic's current slash-command docs explicitly mark **`/review` as deprecated** in favor of the separate review feature or `claude --review`.

---

## CLI Details That Matter

The CLI is where many of the best high-leverage workflows live.

### Core Flags

- `-p`, `--print`: non-interactive mode
- `-c`, `--continue`: continue the most recent conversation
- `-r`, `--resume`: resume a session by id
- `--session-id`: target a specific session directly
- `--output-format`: structured output such as JSON or streaming JSON
- `--model`: choose the model
- `--effort`: set reasoning effort
- `--permission-mode`: permission behavior such as `acceptEdits`, `bypassPermissions`, or `plan`
- `--allowedTools`: allowlist specific tools or patterns
- `--tools`: MCP tool allowlist
- `--max-turns`: bound autonomous runs
- `--max-budget-usd`: cap spend in print mode

### Multi-Repo and Environment Flags

- `--worktree`: create or use git worktrees
- `--remote`: run in a remote environment
- `--remote-control`: connect to a running Claude Code instance
- `--plugin-dir`: load plugins from a local path during development
- `--strict-mcp-config`: fail fast on invalid MCP configuration
- `--agents`: point to extra agent definition directories

### Practical CLI Patterns

```bash
# Review changes non-interactively
claude -p "Review the current diff for correctness, regressions, and missing tests" \
  --output-format json \
  --max-turns 3

# Plan only, no execution
claude -p "Plan the migration from Jest to Vitest" \
  --permission-mode plan

# Continue yesterday's work
claude --continue
```

### Fine Details

- `--max-budget-usd` only applies in print mode.
- `--permission-mode plan` is one of the best ways to get a high-quality plan before code changes.
- `--remote-control` and remote environments make it easier to keep heavy work off your local machine, but they add another layer of operational complexity.

---

## MCP Servers and Plugins

These are related but not the same thing.

### MCP

MCP servers expose external tools and data sources to Claude Code. Good uses include:

- version-accurate docs
- GitHub, Jira, and issue trackers
- databases and internal systems
- search or retrieval systems

Use MCP when the missing capability is "Claude needs access to a system."

### Plugins

Plugins are the packaging layer for reusable Claude Code extensions. Current docs describe plugins as able to bundle:

- skills
- subagents
- hooks
- MCP servers
- commands
- LSP-based code intelligence
- default settings

Use a plugin when the missing capability is "we want to share and version a setup."

### Fine Details That Matter

- local plugin development is supported via `--plugin-dir`
- `/reload-plugins` exists for fast iteration
- plugins can act as the clean distribution layer for team-standard skills and agents
- LSP-backed plugins can improve diagnostics and navigation, not just add prompts

---

## Best-Practice Workflow Patterns

These are the patterns most worth institutionalizing.

### 1. Plan Before Heavy Edits

Use `/plan` or `--permission-mode plan` when the task is architectural, risky, or broad. Review the plan before allowing edits.

### 2. Keep Acceptance Criteria In The Prompt

Ask for:

- the exact files or areas to change
- constraints
- tests to run
- what "done" means

Weak:

> add auth

Strong:

> Add password-reset support in the existing auth flow. Reuse the current email provider. Do not change session semantics. Update API handlers, UI states, and tests. Finish only when targeted tests pass.

### 3. Attach Ground Truth

Anthropic's docs explicitly support attaching:

- files or folders with `@`
- images by drag/drop or paste
- URLs for docs or tickets

Do that instead of relying on recollection.

### 4. Use Skills For Repeated Work, Not Memory

If the same workflow keeps reappearing, convert it into a skill instead of inflating `CLAUDE.md`.

### 5. Use Subagents For Research And Separation

If the task is broad or benefits from specialization, delegate. If it is tight and local, keep it in the main thread.

### 6. Turn Checks Into Hooks

If you keep reminding Claude to do the same thing, that is a signal to automate it.

### 7. Reset Before Context Rot Wins

Anthropic's best-practices docs explicitly recommend:

- compacting
- clearing
- rewinding
- resuming
- using checkpoints
- running multiple sessions when appropriate

Long, overloaded sessions are a quality problem, not just a token problem.

---

## Common Failure Patterns

These are the mistakes most likely to degrade Claude Code output quality.

### Overstuffed Memory

Too much always-on instruction creates dilution. Keep memory focused and move detail elsewhere.

### Vague Prompts

"Fix this" is much worse than "Investigate this failing test, explain root cause, then patch only the minimal code path and rerun the targeted suite."

### Soft Rules For Hard Requirements

If something must happen, do not trust memory alone. Use a hook, a restricted tool policy, or a scripted check.

### Wrong Layer

If you solve every problem with `CLAUDE.md`, you will end up with poor routing and lower compliance. Use the right layer instead.

### One Endless Session

Claude Code works better when you deliberately restart, compact, rewind, or split work into multiple threads.

---

## Good Defaults For A New Repo

If you are setting up Claude Code from scratch, this is the order that tends to pay off fastest:

1. Run `/init`, then prune `CLAUDE.md` down to the smallest useful version.
2. Add `.claude/rules/` for domain-specific or subtree-specific guidance.
3. Create 2-3 skills for your most repeated workflows.
4. Add a few narrow hooks for safety and verification.
5. Add subagents only where specialization or isolation genuinely helps.
6. Add MCP servers for live systems and plugins for shared packaging once the basics are stable.

That sequence usually beats jumping straight into a large plugin or hook stack.

---

## What I Removed From The Previous Guide

The older version mixed in a lot of fast-moving or weakly sourced material. I removed or de-emphasized:

- the outdated "custom slash commands vs skills" split
- brittle feature counts such as fixed hook event totals
- speculative or community-only claims about cost, usage limits, and adoption
- issue-driven claims that were not central to getting better results
- opinionated heuristics that were not clearly grounded in current Anthropic docs

Those sections age badly and distract from the practices that consistently improve output.

---

## References

### Official Docs

- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Best Practices](https://code.claude.com/docs/en/best-practices)
- [Memory](https://code.claude.com/docs/en/memory)
- [Slash Commands](https://code.claude.com/docs/en/slash-commands)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [Skills](https://code.claude.com/docs/en/skills)
- [CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Plugins](https://code.claude.com/docs/en/plugins)
- [MCP](https://code.claude.com/docs/en/mcp)

### Official Release Tracking

- [Claude release notes overview](https://docs.claude.com/en/release-notes/overview)
- [anthropics/claude-code releases](https://github.com/anthropics/claude-code/releases)
- [anthropics/claude-code changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

### Anthropic Engineering

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [How to Configure Hooks](https://www.anthropic.com/news/how-to-configure-hooks)
- [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)

---

When in doubt, prefer the official docs and release notes over any third-party guide, including this one.
