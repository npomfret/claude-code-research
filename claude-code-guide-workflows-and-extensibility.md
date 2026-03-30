# Claude Code Guide: Workflows and Extensibility

Last verified: **March 30, 2026**

This section covers the reusable and composable parts of Claude Code: skills, subagents, hooks, MCP servers, and plugins.

## Skills

Current docs treat skills as the right way to create your own reusable workflows and custom commands.

Use a skill when you need:

- a structured checklist
- reference docs
- scripts or templates
- tool restrictions
- a model override
- a clean context boundary

Do not use a skill for:

- always-on repo memory
- a hard guardrail
- a one-off prompt you will never reuse

### Where Skills Live

- project skills: `.claude/skills/`
- personal skills: `~/.claude/skills/`
- plugins can also ship skills

### Invocation Modes

A skill can be:

- model-invocable
- user-invocable
- both

Important frontmatter details to know:

- `description` helps Claude route the skill automatically
- `user-invocable: true` exposes it for direct use
- `disable-model-invocation: true` removes it from automatic routing
- `allowed-tools` constrains the skill's tool surface
- `model` lets you pin a model
- `context: fork` lets the skill run in a fresh context

### Best Practices For Reliable Routing

If you want Claude to auto-invoke a skill reliably:

- write a concrete description
- say when to use it
- say when not to use it if there is overlap
- keep `SKILL.md` focused and move bulk into references or scripts
- prefer a few strong skills over many overlapping tiny ones

## Subagents

Subagents are specialized Claude instances with their own instructions, tools, permissions, and context.

They help when the task benefits from isolation:

- broad codebase research
- parallel investigation of separate subsystems
- planning before implementation
- independent reviews from different perspectives
- specialist tasks with distinct tool permissions

They usually hurt when:

- one file or one function is changing
- the work depends on tight shared reasoning in the same files
- the delegation overhead is larger than the task

Good defaults:

- keep the specialization narrow and obvious
- give the subagent only the context it actually needs
- consider a cheaper model for research or review agents

## Hooks

Hooks are the enforcement layer.

They can run:

- shell commands
- HTTP endpoints
- LLM-based checks

They trigger on Claude Code lifecycle events and can inspect tool inputs, block actions, modify behavior, or attach context.

### Best Uses

- block dangerous shell patterns
- run targeted checks after edits
- attach short repo reminders at session start
- enforce branch or environment constraints
- send notifications when long-running work completes

### Best Practices

- keep hooks fast and deterministic
- make blocking hooks narrow and well-tested
- use `async` for non-blocking background work
- prefer hooks for enforcement, not semantic routing
- make hook output visible enough that Claude can react to it

### What Not To Do

- do not move complex reasoning into hook classifiers
- do not attach heavyweight checks to every event
- do not let hooks silently mutate files

The clearest mental model is:

- memory explains the repo
- skills package repeatable workflows
- hooks enforce what must actually happen

## MCP Servers

Use MCP when the missing capability is "Claude needs access to a system."

Good examples:

- version-accurate docs
- GitHub, Jira, and issue trackers
- databases and internal systems
- search or retrieval systems

MCP is about tool access, not repo memory.

## Plugins

Use a plugin when the missing capability is "we want to share and version a setup."

Plugins can package:

- skills
- subagents
- hooks
- MCP servers
- commands
- LSP-backed code intelligence
- default settings

Practical details that matter:

- local plugin development works via `--plugin-dir`
- `/reload-plugins` speeds up iteration
- plugins are a clean way to distribute team-standard Claude Code setup

## Workflow Design Patterns

Patterns that tend to hold up well:

- use skills for repeated debug, review, migration, and release flows
- use subagents for research and separation, not for tiny edits
- use hooks when you keep reminding Claude to do the same thing
- use MCP for live systems and source-of-truth docs
- use plugins only after the basic local setup is already working

## Sources

- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Anthropic Engineering: Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Anthropic: How to Configure Hooks](https://www.anthropic.com/news/how-to-configure-hooks)
