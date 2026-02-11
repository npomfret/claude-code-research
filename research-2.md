# Claude Code: A Nuanced Research Guide

Claude Code is Anthropic's agentic coding tool -- a terminal-based AI assistant that doesn't just suggest code but autonomously reads your codebase, edits files, runs commands, observes results, and iterates. Think of it less as autocomplete and more as an amnesia-prone junior developer who is tireless, extremely fast at reading code, but needs clear direction and guardrails. It's available in the terminal, VS Code, JetBrains, as a desktop app, and in-browser.

This document is a nuanced guide to **when and how to use** Claude Code's four major extensibility features: **Skills, Hooks, Sub-Agents, and Commands** -- based on real developer experience, community reports, and the honest trade-offs people don't put in marketing copy.

---

## Table of Contents

1. [Skills (Custom Slash Commands)](#1-skills-custom-slash-commands)
2. [Hooks](#2-hooks)
3. [Sub-Agents (The Task Tool)](#3-sub-agents-the-task-tool)
4. [Commands (Built-in CLI & Slash Commands)](#4-commands-built-in-cli--slash-commands)
5. [CLAUDE.md -- The Unsung Foundation](#5-claudemd--the-unsung-foundation)
6. [MCP Servers & Plugins](#6-mcp-servers--plugins)
7. [The Real Problems](#7-the-real-problems)
8. [Decision Framework: Which Tool When?](#8-decision-framework-which-tool-when)
9. [Sources & References](#9-sources--references)

---

## 1. Skills (Custom Slash Commands)

### What They Actually Are

Skills are Markdown files that become slash commands. That's it. Put a `.md` file in `.claude/commands/` (project-level, shared via git) or `~/.claude/commands/` (personal), and it becomes a `/project:<name>` or `/<name>` command. The file's content becomes the prompt. `$ARGUMENTS` is the only dynamic parameter.

The design is *deliberately simple*. No YAML config, no plugin API, no DSL. Just Markdown.

### When They Shine

Skills excel in a very specific niche: **repeatable prompts you'd otherwise retype**. The sweet spot is:

- **Code review checklists** -- by far the most popular skill. Instead of remembering 10 things to check, you type `/project:review`.
- **PR/commit message generation** -- standardized formatting across the team.
- **Bug investigation templates** -- structured approaches that ensure consistent debugging.
- **Onboarding** -- new team members get the same prompts as veterans via project-level commands.

The key insight from experienced users: **skills are prompt engineering encapsulation**. Your best prompts get saved for reuse rather than lost to chat history.

### When They Don't

- **One-off tasks** -- creating a skill for something you'll do once is overhead.
- **Anything requiring logic** -- there's no conditional branching, no loops, no inter-skill composition. If you need "if TypeScript do X, else do Y," you can't express it in a skill.
- **Heavy parameterization** -- `$ARGUMENTS` is your only variable. No named params, no flags. Power users work around this by structuring what they pass, but it's clunky.
- **When CLAUDE.md would suffice** -- if an instruction applies to *every* interaction ("never use `any` in TypeScript"), it belongs in CLAUDE.md, not a skill.

### What the Community Reports

**Positive:**
- Teams report dramatic alignment gains from project-level skills committed to git. Everyone reviews code the same way.
- The low barrier to entry is universally praised -- create a file, it works.
- Power users combine skills with CLAUDE.md (persistent context) and hooks (automatic triggers) for a layered system.

**Negative/Nuanced:**
- **Overly prescriptive skills backfire** -- if your skill is too rigid, Claude follows the letter rather than the spirit, producing checklist-compliant but actually useless output.
- **Discoverability is weak** -- team members don't know what commands exist unless they explore `.claude/commands/` or check the autocomplete menu. There's no "list all commands with descriptions" feature.
- **Argument placement matters more than expected** -- "Review $ARGUMENTS for security issues" vs "Do a security review focusing on $ARGUMENTS" produces meaningfully different results.
- Many developers report they **haven't adopted skills yet** despite their simplicity, suggesting the feature isn't as discoverable or obviously useful as hooks or agents.

### Popular Skill Examples

| Skill | What It Does |
|-------|-------------|
| `/review` | Code review against a project-specific checklist (security, performance, error handling, tests) |
| `/commit` | Generate conventional commit messages from staged changes |
| `/test` | Write comprehensive tests for a given file/module |
| `/investigate <bug>` | Structured bug investigation: find code, trace flow, identify root cause, suggest fix |
| `/pr` | Generate PR descriptions from branch changes |
| `/security` | OWASP-style security audit |
| `/explain <module>` | Onboarding-style deep explanation of a code area |

The [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) provides 148+ slash commands and 54 agents. The [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) repository curates community skills.

**Source:** [Claude Code Docs: Skills](https://code.claude.com/docs/en/skills) | [OneAway: Complete Guide](https://oneaway.io/blog/claude-code-skills-slash-commands) | [alexop.dev: Customization Guide](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/)

---

## 2. Hooks

### What They Actually Are

Hooks are shell commands (or LLM prompts, or sub-agent invocations) that fire automatically at specific lifecycle events. They are the **hard guarantee** layer -- while CLAUDE.md says "please format your code," a hook *forces* formatting to happen every time a file is edited. No ifs, ands, or buts.

There are 13 lifecycle events including `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `Notification`, `SubagentStop`, and more. Hooks communicate via stdin (JSON in), stdout (JSON decisions out), and exit codes.

### When They Shine

Hooks are Claude Code's **enforcement mechanism**. Use them when behavior must be *guaranteed*, not *suggested*:

- **Auto-formatting after edits** -- The single most popular hook. PostToolUse on `Write|Edit` running Prettier/Black/gofmt.
- **Security guardrails** -- PreToolUse hooks blocking `rm -rf`, `sudo`, `chmod 777`, or changes to `.env` files.
- **Notifications** -- Stop hooks that notify you when Claude finishes (macOS `say 'Claude is done'` is the iconic example).
- **Branch protection** -- Preventing changes on `main`/`master`.
- **Headless/autonomous mode safety** -- When running with `--dangerously-skip-permissions`, hooks are your only guardrails.
- **Input modification** (since v2.0.10) -- PreToolUse hooks can *rewrite* tool parameters. A hook can intercept `rm -rf` and change it to `rm -i`.

### When They Don't

This is where the nuance gets important:

- **Auto-formatting on every edit creates context noise.** When your formatter changes files, Claude gets a system reminder about those changes. Aggressive formatting on every edit floods the context window. **Community wisdom: format on commit, not on every edit.**
- **Synchronous hooks block execution.** One team reported 180 seconds of delay per interaction from stacking linter + formatter + git logger + metrics hooks. Use `async: true` for anything that doesn't need to block.
- **Don't use hooks for complex reasoning.** Hooks should be fast, deterministic checks -- not multi-step logic requiring conversation context.
- **When a CLAUDE.md instruction would suffice.** If the behavior is advisory and doesn't need to be guaranteed, a rule in CLAUDE.md is simpler and has zero overhead.
- **When you lack the expertise to maintain them.** Hooks require writing and maintaining shell scripts or Python code. They're a developer-only feature.

### Critical Bugs and Gotchas

The community has surfaced several serious issues:

1. **PreToolUse exit codes can be IGNORED** ([Issue #21988](https://github.com/anthropics/claude-code/issues/21988)) -- Operations proceed even when hooks return non-zero exit codes. **Do not rely solely on hooks for security-critical blocking without verifying this is fixed in your version.**

2. **Progressive duplication bug** ([Issue #3523](https://github.com/anthropics/claude-code/issues/3523)) -- Hooks can multiply during long sessions, causing 10+ simultaneous ESLint processes after each file edit. Severe CPU/memory degradation.

3. **Matchers are case-sensitive** -- `"bash"` will NOT match the `Bash` tool. This is a common "my hook never fires" report.

4. **All matching hooks run in parallel** -- You cannot order hooks or define dependencies. Sequential execution is a [feature request](https://github.com/anthropics/claude-code/issues/21533).

5. **Home directory quirk** -- User-level Stop hooks don't fire when Claude Code runs from `~` ([Issue #15629](https://github.com/anthropics/claude-code/issues/15629)).

6. **Settings sometimes fail to load** -- Hooks in `~/.claude/settings.json` occasionally silently fail ([Issue #11544](https://github.com/anthropics/claude-code/issues/11544)).

### What the Community Reports

Despite being arguably the most powerful extensibility mechanism, one survey noted **"approximately nobody is using them."** Adoption is low relative to the feature's power, likely because:

- They require shell scripting knowledge
- The configuration is JSON in settings files (not as discoverable as dropping a Markdown file)
- The bugs listed above erode trust

Those who *do* use hooks are enthusiastic:
- Auto-formatting hooks "eliminate an entire class of back-and-forth"
- Security hooks provide "peace of mind for autonomous operation"
- Notification hooks are "universally praised as quality-of-life improvements"
- A [6-month production report](https://alirezarezvani.medium.com/the-claude-code-hooks-nobody-talks-about-my-6-month-production-report-30eb8b4d9b30) details advanced real-world usage patterns

### Popular Hook Examples

**Text-to-speech notification (Stop):**
```json
{
  "hooks": {
    "Stop": [{"matcher": "", "hooks": [{"type": "command", "command": "say 'Claude has finished working'"}]}]
  }
}
```

**Auto-format with Prettier (PostToolUse):**
```json
{
  "hooks": {
    "PostToolUse": [{"matcher": "Write|Edit", "hooks": [{"type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATHS"}]}]
  }
}
```

**Block destructive commands (PreToolUse):**
A hook on `Bash` that parses stdin JSON, checks for `rm -rf /`, `sudo`, `chmod 777`, and exits with code 2 to block.

**LLM-evaluated safety (Prompt hook):**
```json
{"matcher": "Bash", "hooks": [{"type": "prompt", "prompt": "Does this command look safe to execute? $ARGUMENTS"}]}
```
Sends the command to Haiku for a yes/no safety check before execution.

**Multi-agent observability dashboard:**
The [claude-code-hooks-multi-agent-observability](https://github.com/disler/claude-code-hooks-multi-agent-observability) project uses hooks at every lifecycle event to emit structured events for real-time monitoring.

**Source:** [Official Hooks Reference](https://code.claude.com/docs/en/hooks) | [Official Hooks Guide](https://code.claude.com/docs/en/hooks-guide) | [DataCamp Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) | [Medium: Hooks You're Ignoring](https://medium.com/@lakshminp/claude-code-hooks-the-feature-youre-ignoring-while-babysitting-your-ai-789d39b46f6c) | [Hacker News Discussion](https://news.ycombinator.com/item?id=45789960)

---

## 3. Sub-Agents (The Task Tool)

### What They Actually Are

Sub-agents are spawned Claude instances that operate in their own isolated context windows. The main agent (orchestrator) delegates work via the `Task` tool, and sub-agents execute independently with their own tool access. Up to 10 can run concurrently.

The critical architecture detail: **sub-agents do NOT see your conversation history.** They only receive the explicit prompt you give them. This is both the key strength (prevents context pollution, enables parallelism) and the key weakness (they need explicit, detailed context).

### When They Shine

Sub-agents have a very clear sweet spot: **parallel, independent work on separate files/domains**.

**The golden rule: parallel only works when agents touch different files.**

Best use cases:
- **Large-scale codebase research** -- "Find all usages of deprecated function X across 200 files"
- **Fan-out analysis** -- Security review + performance review + accessibility review, each in parallel
- **Documentation generation** -- A primary agent lists all functions, sub-agents document each one, a final agent assembles
- **Multi-module refactoring** -- Frontend agent handles React components while backend agent manages API routes

The [Anthropic engineering blog](https://www.anthropic.com/engineering/building-c-compiler) describes using **16 Claude instances simultaneously** to build a C compiler, with each agent taking a "lock" on a task via text files in a `current_tasks/` directory.

### When They Don't

This is where many developers over-invest:

- **Over-parallelizing is a real problem.** Launching 10 agents for a simple feature wastes tokens and creates coordination overhead. One experienced developer noted: "Max out at three or four specialized agents. More than that decreases productivity rather than increasing it."

- **The Explore agent's summaries can be lossy.** One power user [explicitly prefers having Opus read files itself](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/) rather than delegating to sub-agents, because direct reads enable "proper attention relationships across context."

- **UI flickering.** Parallel sub-agents cause visual noise in the terminal that some find disorienting.

- **Context density is critical.** "Bad invocation: Fix authentication" vs. "Good invocation: Fix OAuth redirect loop where successful login redirects to /login instead of /dashboard. Reference the auth middleware in src/lib/auth.ts." Sub-agents can't ask clarifying questions.

- **Inter-agent communication is limited.** Sub-agents cannot talk to each other -- only the outer agent coordinates. The most successful pattern for scale is **file-based communication**: each agent writes results to a markdown file, which the next agent reads.

- **Simple tasks don't need them.** If you're editing one file, the overhead of spawning is wasteful.

### What the Community Reports

**Positive:**
- Sub-agents make large codebase tasks "feasible" that would exhaust the context window.
- The fan-out/fan-in pattern for code review is widely praised.
- Cost optimization: running main sessions on Opus with Sonnet sub-agents (`CLAUDE_CODE_SUBAGENT_MODEL`) significantly reduces costs.

**Negative/Nuanced:**
- **Claude Code uses sub-agents conservatively by default.** To maximize parallelization, you must be explicit in your prompts.
- Many developers struggle with **context window limitations and session persistence**, losing progress between sessions.
- The paradigm works best when you think of sub-agents as **disposable, stateless workers** -- not persistent team members.
- Running sub-agents that return detailed results can still consume significant context in the main thread.

### Popular Patterns

| Pattern | Description | When to Use |
|---------|------------|-------------|
| **Fan-Out/Fan-In** | Parallel research, synthesize results | Gathering info from many independent sources |
| **Expert Panel** | Security expert + Performance expert + A11y expert review the same code | Multi-perspective analysis |
| **Pipeline** | Task A output feeds Task B input feeds Task C | Sequential processing stages |
| **Background Research** | Web lookups, doc exploration, audit while main thread works | Non-blocking info gathering |

**Source:** [Official Sub-Agent Docs](https://code.claude.com/docs/en/sub-agents) | [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) | [ClaudeFast: Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) | [Sankalp's Experience with Claude Code 2.0](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)

---

## 4. Commands (Built-in CLI & Slash Commands)

### Interactive Slash Commands

These are the built-in commands available at the Claude Code prompt:

| Command | What It Actually Does | Nuanced Take |
|---------|----------------------|--------------|
| `/compact` | Summarizes conversation to free context | **The most important command for long sessions.** Use proactively at ~60% context, not when you hit the wall. Accepts guiding instructions: `/compact focus on the database migration work`. |
| `/clear` | Wipes conversation, starts fresh | Use when switching tasks entirely. Don't use mid-task -- you'll lose all context. |
| `/review` | Code review of current changes | Good for quick reviews, but community reports that **GPT-5.2-Codex finds more bugs with fewer false positives** for dedicated review tasks. |
| `/pr` | Create a pull request | Analyzes branch, drafts title/description, creates PR via `gh`. Very popular. |
| `/commit` | Create a git commit | Generates conventional commit messages. |
| `/init` | Create CLAUDE.md | Walks through project setup. The **single most impactful thing you can do** for Claude Code quality. |
| `/model` | Switch models mid-session | Useful for: Sonnet for fast tasks, Opus for complex reasoning. |
| `/cost` | Show token usage | Eye-opening. Track this regularly. |
| `/memory` | View/edit CLAUDE.md files | |

### CLI Flags (The Power User Arsenal)

The CLI is where Claude Code becomes a Unix-style composable tool:

**Essential flags:**
- `-p` / `--print` -- Non-interactive mode. The gateway to scripting and CI/CD.
- `-c` / `--continue` -- Resume the most recent conversation. Universally useful.
- `-r` / `--resume` -- Resume a specific past session by search.
- `--model <model>` -- Override model. Aliases: `sonnet`, `opus`.
- `--effort <level>` -- Reasoning effort: `low`, `medium`, `high`.
- `--output-format json` -- Structured output for parsing in scripts.
- `--max-budget-usd` -- Cap spending. Only works with `-p`.

**Permission control:**
- `--allowed-tools "Bash(git:*) Read"` -- Whitelist specific tools with glob patterns.
- `--disallowed-tools "Bash(rm:*)"` -- Blacklist specific tools.
- `--permission-mode plan` -- Claude only plans, never executes. **Great for reviewing what Claude *would* do before letting it loose.**
- `--dangerously-skip-permissions` -- Exactly as dangerous as it sounds. Only in sandboxed environments.

**CI/CD patterns:**
```bash
# One-shot code review in CI
claude -p "Review this PR for security issues" --output-format json --max-budget-usd 2.00

# Pipe logs for analysis
tail -f app.log | claude -p "Alert me if you see anomalies"

# Batch processing
git diff main --name-only | claude -p "Review changed files for security issues"

# Schema-validated output
claude -p "List API endpoints" --json-schema '{"type":"array","items":{"type":"object","properties":{"path":{"type":"string"},"method":{"type":"string"}}}}'
```

**Custom agents via CLI:**
```bash
claude --agents '{"security-reviewer": {"description": "Security specialist", "prompt": "You are an expert security auditor. Focus on OWASP Top 10."}}' --agent security-reviewer
```

### What the Community Reports

- **`/compact` is underused and critical.** Experienced users monitor context usage and compact proactively. New users hit the wall and wonder why quality degrades.
- **`-c` (continue) is a workflow game-changer.** Being able to exit, come back later, and pick up exactly where you left off.
- **`--from-pr` is surprisingly useful** for jumping back into the session that created a specific PR.
- **Session forking** (`--fork-session`) lets you branch off a conversation without mutating the original -- useful for exploring alternative approaches.
- **`--permission-mode plan`** is loved for reviewing what Claude would do before giving it real access.

**Source:** [CLI Reference](https://code.claude.com/docs/en/cli-reference) | [Headless Mode Docs](https://code.claude.com/docs/en/headless) | [F22 Labs: 10 Productivity Workflows](https://www.f22labs.com/blogs/10-claude-code-productivity-tips-for-every-developer/)

---

## 5. CLAUDE.md -- The Unsung Foundation

CLAUDE.md is not a "feature" in the same category as skills/hooks/agents, but **it's the single most important configuration** for getting good results. It's a Markdown file at your project root that Claude reads at the start of every conversation.

### The Hard-Won Community Wisdom

**Keep it short.** "For each line, ask 'Would removing this cause Claude to make mistakes?'" Bloated CLAUDE.md files cause Claude to **ignore your actual instructions** as they get lost in noise.

**CLAUDE.md guides behavior but doesn't enforce it.** Claude can and does ignore CLAUDE.md instructions, especially as the context window fills. If something MUST happen, use a hook. CLAUDE.md is for *advisory* context.

**What belongs in CLAUDE.md:**
- Build/test commands (`npm test`, `cargo build`)
- Code style conventions that differ from defaults
- Architecture decisions (where files go, naming patterns)
- "Never do X" instructions (never modify `src/generated/`, always run lint after changes)

**What doesn't:**
- Anything that needs to be *guaranteed* (use hooks)
- Task-specific workflows (use skills)
- Generic programming advice Claude already knows

### The Three-Attempt Mental Model

From a [staff engineer's 6-week journey](https://www.sanity.io/blog/first-attempt-will-be-95-garbage):
- First attempt: 95% garbage rate. But you learn what Claude misunderstands.
- Second attempt: 50% garbage. You've refined your CLAUDE.md based on failures.
- Third attempt: Finally workable. Your project documentation is now calibrated.

The takeaway: **CLAUDE.md is an iterative document.** You refine it based on where Claude fails.

**Source:** [Best Practices](https://code.claude.com/docs/en/best-practices) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Sanity: First Attempt Will Be 95% Garbage](https://www.sanity.io/blog/first-attempt-will-be-95-garbage) | [Getting Good Results](https://www.dzombak.com/blog/2025/08/getting-good-results-from-claude-code/)

---

## 6. MCP Servers & Plugins

### The Ecosystem

The Model Context Protocol (MCP) connects Claude Code to external tools and data sources. As of 2026, there are 50+ curated MCP servers and a growing plugin ecosystem.

**Most popular MCP servers:**
- **GitHub MCP** -- Read repos, manage issues/PRs, automate workflows
- **Brave Search MCP** -- Real-time web search (fills Claude's knowledge gap)
- **Context7** -- Live, version-accurate documentation instead of stale training data
- **Supabase MCP** -- Database operations with natural language
- **Google Workspace MCP** -- Sheets/Drive/Gmail/Calendar integration
- **Firecrawl** -- Web scraping with JS rendering and anti-bot handling

### The Nuanced Take

**MCP servers bloat context upfront** with tool definitions. Claude Code's Tool Search feature enables lazy loading, reducing context usage by up to 95%, but you need to configure it.

**Not all MCP servers are production-ready.** Servers run as separate processes; if they crash, Claude loses access mid-session.

**Plugins** (introduced in Claude Code 2.1) bundle slash commands, agents, skills, and hooks together. Install with `/plugin install`. A [community marketplace](https://claude-plugins.dev/) is emerging.

**Source:** [MCP Docs](https://code.claude.com/docs/en/mcp) | [ClaudeFast: 50+ Best MCP Servers](https://claudefa.st/blog/tools/mcp-extensions/best-addons) | [Firecrawl: Top 10 Plugins](https://www.firecrawl.dev/blog/best-claude-code-plugins)

---

## 7. The Real Problems

### The Community's Honest Frustrations

This wouldn't be a nuanced guide without the complaints:

**1. Usage Limits Are the #1 Pain Point**

Pro subscribers who previously used Claude Code for 40-50 hours/week reported being reduced to 6-8 hours after the Sonnet 4.5 rollout. There's "no clear visibility into what the actual limits are or how usage is tracked." Community tools like [ccusage](https://github.com/) (CLI dashboard) and browser extensions emerged to fill transparency gaps. ([The Register](https://www.theregister.com/2026/01/05/claude_devs_usage_limits/) | [GitHub Issue #9094](https://github.com/anthropics/claude-code/issues/9094))

**2. Context Loss & Auto-Compaction**

Claude forgets project context mid-session. Auto-compact aggressively discards details. Developers describe "racing against compaction" to document work before details vanish. Performance degrades as project complexity increases. ([Community Struggles Gist](https://gist.github.com/eonist/0a5f4ae592eadafd89ed122a24e50584))

**3. Confident Incompetence**

Claude produces broken code with conviction. Claims tasks are complete when they're incomplete. Generates fake tests that don't verify functionality. "Only 30% first-try reliability -- not from fundamentally broken code, but from poor architectural decisions creating downstream problems." ([Sanity Blog](https://www.sanity.io/blog/first-attempt-will-be-95-garbage))

**4. Over-Engineering**

Adds unnecessary complexity instead of simple solutions. Creates duplicate files, odd filenames, incomplete refactors. The system prompt explicitly counters this tendency, but it persists.

**5. Cost**

$1,000-1,500/month for senior engineers using it heavily. To access Opus 4.5, you need $100-200/month Claude Max. The ROI is there for some (2-3x faster feature shipping), but requires substantial learning curve investment.

### The Uncomfortable Truth

From an experienced developer who's used Claude Code for over a year:

> "Treat AI like a junior developer who doesn't learn." This reframing -- not as magic replacement but as amnesia-prone assistant -- determined success.

> "I'm more critical of 'my code' now because I didn't type a lot of it. No emotional attachment means better reviews."

> "The goal isn't to code without AI, but to be a better developer because of AI, not despite it -- and sometimes that means turning it off."

**Source:** [Sanity: Staff Engineer's Journey](https://www.sanity.io/blog/first-attempt-will-be-95-garbage) | [The Register: Usage Limits](https://www.theregister.com/2026/01/05/claude_devs_usage_limits/) | [Community Struggles Analysis](https://gist.github.com/eonist/0a5f4ae592eadafd89ed122a24e50584) | [Kelsey Piper: I Can't Stop Yelling at Claude Code](https://www.theargumentmag.com/p/i-cant-stop-yelling-at-claude-code)

---

## 8. Decision Framework: Which Tool When?

| Situation | Best Tool | Why Not the Others |
|-----------|-----------|-------------------|
| "This must happen every time, no exceptions" | **Hook** | CLAUDE.md is advisory (can be ignored). Skills are on-demand. |
| "Our team reviews code the same way" | **Skill** (project-level) | Hooks are for automated triggers, not interactive workflows. CLAUDE.md is for global context, not task-specific prompts. |
| "I want context Claude always has" | **CLAUDE.md** | Skills are invoked on demand. Hooks are event-driven. |
| "Search 200 files for a pattern" | **Sub-Agent** | Keeps main context clean. Parallelizable. |
| "Format code after every edit" | **Hook** (but be careful) | Skills can't auto-trigger. CLAUDE.md instruction will be inconsistent. **Caveat:** Consider formatting on commit instead to reduce context noise. |
| "Block dangerous commands" | **Hook** (PreToolUse) | Only hooks can *block* tool execution. But verify the exit code bug is fixed first. |
| "Quick one-off task" | **Inline prompt** | Skills/hooks/agents are overhead for one-shot tasks. |
| "Notify me when Claude finishes" | **Hook** (Stop) | Nothing else can trigger on completion. |
| "Connect to Jira/Slack/database" | **MCP Server** | Not a skill/hook/agent problem -- it's a tool integration. |
| "Run Claude in CI/CD" | **CLI with `-p`** | Headless mode + `--output-format json` + `--max-budget-usd`. |
| "Complex multi-part task" | **Sub-Agents** (carefully) | But max 3-4 specialized agents. More decreases productivity. |
| "I want Claude to plan before acting" | **`--permission-mode plan`** | Not a skill -- it's a CLI flag that makes Claude propose actions without executing. |

### The Layered Architecture

The most successful setups use all features together in layers:

```
CLAUDE.md          → Always-on context (coding standards, architecture)
Skills             → On-demand workflows (review, commit, investigate)
Hooks              → Guaranteed automation (format, lint, notify, block)
Sub-Agents         → Parallel work (research, multi-file analysis)
MCP Servers        → External tool integration (Jira, databases, search)
CLI flags          → Session-level control (model, permissions, output format)
```

Each layer serves a different purpose. Confusion comes from trying to use one where another belongs.

---

## 9. Sources & References

### Official Documentation
- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Skills Documentation](https://code.claude.com/docs/en/skills)
- [Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [Sub-Agents](https://code.claude.com/docs/en/sub-agents)
- [Best Practices](https://code.claude.com/docs/en/best-practices)
- [CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Headless Mode](https://code.claude.com/docs/en/headless)
- [MCP Integration](https://code.claude.com/docs/en/mcp)

### Anthropic Engineering & Blog
- [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)
- [Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Enabling Autonomous Claude Code](https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously)
- [How to Configure Hooks](https://claude.com/blog/how-to-configure-hooks)

### Developer Experience & Community
- [Sanity: First Attempt Will Be 95% Garbage (Staff Engineer's 6-Week Journey)](https://www.sanity.io/blog/first-attempt-will-be-95-garbage)
- [Sankalp: My Experience with Claude Code 2.0](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)
- [Builder.io: How I Use Claude Code (+ My Best Tips)](https://www.builder.io/blog/claude-code)
- [sshh.io: How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)
- [Getting Good Results from Claude Code](https://www.dzombak.com/blog/2025/08/getting-good-results-from-claude-code/)
- [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [oreateAI: CLAUDE.md Best Practices from Reddit](https://www.oreateai.com/blog/claudemd-best-practices-reddit/3a29cdd605ebb2025681e71218021b5e)
- [Medium: Hooks Nobody Talks About (6-Month Production Report)](https://alirezarezvani.medium.com/the-claude-code-hooks-nobody-talks-about-my-6-month-production-report-30eb8b4d9b30)
- [Kelsey Piper: I Can't Stop Yelling at Claude Code](https://www.theargumentmag.com/p/i-cant-stop-yelling-at-claude-code)

### Community Frustrations & Analysis
- [The Register: Claude Devs Complain About Usage Limits](https://www.theregister.com/2026/01/05/claude_devs_usage_limits/)
- [GitHub Issue #9094: Unexpected Usage Limit Changes](https://github.com/anthropics/claude-code/issues/9094)
- [Community Struggles Gist (Top 3 Problems)](https://gist.github.com/eonist/0a5f4ae592eadafd89ed122a24e50584)
- [AI Tool Analysis: Claude Code Review 2026](https://aitoolanalysis.com/claude-code/)

### Curated Collections & Toolkits
- [awesome-claude-code (21.6k stars)](https://github.com/hesreallyhim/awesome-claude-code)
- [awesome-claude-code-toolkit (135 agents, 35 skills, 19 hooks)](https://github.com/rohitg00/awesome-claude-code-toolkit)
- [everything-claude-code (Hackathon Winner Configs)](https://github.com/affaan-m/everything-claude-code)
- [Claude Command Suite (148+ commands)](https://github.com/qdhenry/Claude-Command-Suite)
- [Claude Code Showcase (Full Config Example)](https://github.com/ChrisWiles/claude-code-showcase)
- [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery)

### Guides & Tutorials
- [alexop.dev: Understanding Claude Code's Full Stack](https://alexop.dev/posts/understanding-claude-code-full-stack/)
- [Young Leaders Tech: Skills vs Commands vs Subagents vs Plugins](https://www.youngleaders.tech/p/claude-skills-commands-subagents-plugins)
- [ProductTalk: Guide to Slash Commands, Agents, Skills, and Plugins](https://www.producttalk.org/how-to-use-claude-code-features/)
- [eesel.ai: Complete Guide to Hooks](https://www.eesel.ai/blog/hooks-in-claude-code)
- [eesel.ai: Subagents Guide](https://www.eesel.ai/blog/subagents-in-claude-code)
- [ClaudeFast: Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices)
- [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks)
- [OneAway: Skills and Slash Commands Complete Guide](https://oneaway.io/blog/claude-code-skills-slash-commands)
- [ClaudeLog: Docs, Guides, Tutorials](https://claudelog.com/)
- [Claude Skills Deep Dive (First Principles)](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)

### MCP & Plugins
- [ClaudeFast: 50+ Best MCP Servers](https://claudefa.st/blog/tools/mcp-extensions/best-addons)
- [Firecrawl: Top 10 Claude Code Plugins](https://www.firecrawl.dev/blog/best-claude-code-plugins)
- [Claude Plugins Directory](https://claude-plugins.dev/)
- [awesome-claude-plugins](https://github.com/ComposioHQ/awesome-claude-plugins)

---

*Research compiled February 2026. Claude Code evolves rapidly -- verify specific features against [current documentation](https://code.claude.com/docs/en/overview).*
