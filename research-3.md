# Claude Code Research: Skills, Agents (Subagents), Hooks, Commands — Nuanced View

Claude Code is Anthropic’s agentic coding environment that runs in your terminal and can read, edit, and execute across your codebase while you steer it with natural language and explicit controls. It combines core LLM reasoning with a local toolchain so it can perform real software tasks, not just answer questions. For developers, it behaves like a programmable teammate: it can search the repo, edit files, run tests/commands, and follow project-specific rules via Skills, Hooks, and Subagents.

This file is intentionally not a rewrite of documentation. It focuses on *when these tools actually shine in practice* vs. when they introduce friction, plus pragmatic heuristics for choosing among them. Official docs and community resources are listed in References for completeness.

Latest syntax docs (authoritative):
- Skills `SKILL.md` format: `https://docs.claude.com/en/docs/claude-code/skills` citeturn2search2
- Subagents frontmatter format: `https://docs.claude.com/en/docs/claude-code/subagents` citeturn2search0
- Hooks settings JSON structure: `https://docs.claude.com/en/docs/claude-code/hooks` citeturn1search1
- Slash command syntax: `https://docs.claude.com/en/docs/claude-code/slash-commands` citeturn0search0

## Skills

**What they excel at**
- Repeatable workflows that need structure: templates, checklists, scripts, and reference files.
- Institutional memory that shouldn’t drift (e.g., compliance checks, code review standards, data processing rules).
- Tasks where *discovery* is valuable: Claude should decide to apply the skill when relevant.

**When they underperform**
- Very small or one-off tasks. A skill adds activation friction and context load.
- Highly novel or ambiguous tasks where you want open-ended reasoning, not a procedure.
- Broad “mega-skills” that try to cover too much. These often fail to activate or produce diluted results.

**Use them when**
- The workflow will be reused and benefits from guardrails or templates.
- You want consistent execution and outcomes across a team.

**Avoid when**
- The task is short, experimental, or better handled by a quick prompt or slash command.

**Practical heuristic**
If you find yourself re-typing a process *and* the process needs structured assets, make it a Skill. If it’s just a recurring prompt, consider a Slash Command instead.

## Agents (Subagents)

**What they excel at**
- Context isolation: long research, large code searches, or specialist reviews.
- Enforcing constraints: read-only review agents, test-only agents, or security-only agents.
- Parallelizable work: one agent explores, another reviews, another runs tests.

**When they underperform**
- Tasks needing continuous, shared reasoning with the main thread.
- Rapid back-and-forth where the overhead of coordination slows progress.

**Use them when**
- You want a specialist that can work independently and hand back results.
- You want tighter tool permissions or distinct model choices per task.

**Avoid when**
- The task requires rich, nuanced context from the main conversation at every step.

**Practical heuristic**
If the task is “research or review” rather than “decide or build,” use a subagent. If it needs unified reasoning or tight integration, keep it in the main agent.

## Hooks

**What they excel at**
- Deterministic automation: formatting, linting, validation, notifications.
- Guardrails: blocking edits to protected files, enforcing policies, logging.
- Consistent behavior that should *always* run, not “maybe” run.

**When they underperform**
- Anything requiring judgment or subjective decisions.
- Overly broad hooks that trigger expensive or risky commands too often.

**Use them when**
- You want guaranteed enforcement and repeatability.

**Avoid when**
- You want nuanced decisions or context-sensitive tradeoffs.

**Practical heuristic**
Hooks are for certainty, not intelligence. If you need a decision, don’t use a hook.

## Commands (Slash Commands)

**What they excel at**
- Explicit control: quick actions you want to run *now*.
- Short, repeatable prompts: “review,” “summarize,” “explain,” “status.”
- Interactive workflow controls: permissions, model switching, adding dirs.

**When they underperform**
- Workflows that require rich assets, templates, or scripts (use Skills instead).
- Automatic discovery (skills are designed for that).

**Use them when**
- You want precision and manual timing for small tasks.

**Avoid when**
- The workflow is multi-step or needs supporting files.

**Practical heuristic**
If it’s a short, frequently repeated prompt, use a slash command. If it needs assets or discovery, use a skill.

## How to choose quickly

**Decision cues**
- Need auto-discovery + assets? → **Skill**
- Need isolated specialist work? → **Subagent**
- Need deterministic enforcement? → **Hook**
- Need explicit one-off action? → **Slash Command**

**Combination patterns (high-signal setups)**
- Skills for repeatable workflows
- Subagents for reviews/tests/debug
- Hooks for enforcement and formatting
- Slash commands for manual control and quick prompts

## References

1. Claude Code product page — `https://claude.com/product/claude-code`
2. anthropics/claude-code (official repo) — `https://github.com/anthropics/claude-code`
3. Claude Code Skills documentation — `https://code.claude.com/docs/en/skills`
4. Claude Code Subagents documentation — `https://code.claude.com/docs/en/subagents`
5. Claude Code Hooks reference — `https://code.claude.com/docs/en/hooks`
6. Claude Code Slash Commands documentation — `https://docs.claude.com/en/docs/claude-code/slash-commands`
7. Claude Code Interactive Mode reference — `https://docs.claude.com/en/docs/claude-code/interactive-mode`
8. Plugins reference (component schemas) — `https://docs.claude.com/en/docs/claude-code/plugins-reference`
9. anthropics/skills (official skills repo) — `https://github.com/anthropics/skills`
10. davila7/claude-code-templates (community templates) — `https://github.com/davila7/claude-code-templates`
