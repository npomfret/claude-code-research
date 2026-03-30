# Claude Code: Current Practical Guide

Last verified: **March 30, 2026**

Claude Code changes quickly. This guide prioritizes Anthropic's current docs, release notes, and the official `anthropics/claude-code` repository over community lore.

The previous single-file version had grown too large to stay useful. It is now split into a short overview plus focused companion docs:

- [Memory and Rules](./claude-code-guide-memory-and-rules.md)
- [Workflows and Extensibility](./claude-code-guide-workflows-and-extensibility.md)
- [CLI and Operations](./claude-code-guide-cli-and-operations.md)

## What Actually Gets Better Results

If your goal is to get the best work out of Claude Code, the highest-leverage improvements are not obscure flags. They are:

1. Give Claude the right context, not more context.
2. Keep project memory concise and current.
3. Use the right execution layer for the job.
4. Force verification with tests, checks, and explicit acceptance criteria.
5. Manage context aggressively in long sessions.

Anthropic's current best-practices docs repeatedly emphasize a few habits:

- be specific about the task, constraints, and success criteria
- attach the exact files, images, or URLs Claude needs
- break large work into steps or use `/plan`
- create concise project memory with `/init`
- use hooks for deterministic guardrails
- use skills and subagents to keep specialized knowledge off the always-on path
- use `/compact`, `/clear`, `/rewind`, `/resume`, and checkpoints instead of dragging one session forever

That is the core playbook. Everything else is an optimization on top.

## The Current Mental Model

Use this decision table when choosing a Claude Code feature.

| You need... | Use | Why |
|---|---|---|
| Always-on repo instructions | `CLAUDE.md` | Persistent project memory |
| Narrow or local instructions | child `CLAUDE.md`, imports, and sometimes `.claude/rules/` | Keeps root memory small |
| A reusable workflow or custom command | skill | On-demand, structured, can show up as slash commands |
| A specialist with isolated context | subagent | Better focus, lower context pollution |
| Deterministic enforcement or automation | hook | Guardrails and lifecycle automation |
| External tools or live data | MCP server | Integrates APIs, docs, and systems |
| Share setup across teams or repos | plugin | Packages skills, agents, hooks, MCP, and settings |
| Non-interactive automation | `claude -p` | CI, scripts, batch workflows |

The biggest category mistake teams make is treating all of those as one "rules" bucket. They are different layers with different strengths.

## Read This Guide In Order

1. Start with [Memory and Rules](./claude-code-guide-memory-and-rules.md) if you are setting up `CLAUDE.md`, child memory, or a rules strategy.
2. Read [Workflows and Extensibility](./claude-code-guide-workflows-and-extensibility.md) if you need reusable skills, hooks, subagents, MCP servers, or plugins.
3. Read [CLI and Operations](./claude-code-guide-cli-and-operations.md) for slash commands, CLI flags, permissions, sandboxing, and day-to-day workflow patterns.

## Good Defaults For A New Repo

If you are setting up Claude Code from scratch, this is the order that usually pays off fastest:

1. Run `/init`, then prune `CLAUDE.md` down to the smallest useful version.
2. Add child `CLAUDE.md` files or imported memory only where local guidance genuinely differs.
3. Create 2-3 skills for repeated workflows.
4. Add a few narrow hooks for safety and verification.
5. Add subagents only where specialization or isolation genuinely helps.
6. Add MCP servers for live systems and plugins for shared packaging once the basics are stable.

That sequence usually beats jumping straight into a large plugin or hook stack.

## Key Sources

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Commands](https://code.claude.com/docs/en/commands)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Claude Code Changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

When in doubt, prefer the official docs and release notes over any third-party guide, including this one.
