# Claude Code: The Practical Guide

Claude Code is Anthropic's terminal-based agentic coding tool. It reads your codebase, edits files, runs commands, and iterates — less like autocomplete and more like a tireless but amnesia-prone junior developer who needs clear direction and guardrails. Available in the terminal, VS Code, JetBrains, as a desktop app, and in-browser.

This guide covers **when and how to use** each extensibility feature, based on real developer experience and honest trade-offs.

---

## Quick Decision Framework

Before diving in, orient yourself. Each tool serves a distinct purpose — confusion comes from using one where another belongs.

| You need... | Use | Not |
|---|---|---|
| Behavior that must happen every time, no exceptions | **Hook** | CLAUDE.md (advisory, can be ignored) |
| A repeatable team workflow (review, commit, investigate) | **Skill** | Hook (can't be invoked interactively) |
| Context Claude should always have | **CLAUDE.md** | Skill (on-demand only) |
| Parallel work across many files/domains | **Sub-Agent** | Main thread (context will overflow) |
| Blocking dangerous commands | **Hook** (PreToolUse) | CLAUDE.md (unenforceable) |
| A quick one-off task | **Inline prompt** | Skill/Hook (overhead for one-shot work) |
| Notification when Claude finishes | **Hook** (Stop) | Nothing else can trigger on completion |
| External tool integration (Jira, DB, search) | **MCP Server** | Not a skill/hook problem |
| Claude in CI/CD | **CLI with `-p`** | Interactive mode |
| Claude to plan before acting | **`--permission-mode plan`** | — |

**The mental model:** Skills are **recipes** (teach Claude *how* to do a specific task). Sub-Agents are **contractors** (take a goal and figure out *what* to do). Hooks are **guardrails** (enforce behavior mechanically). CLAUDE.md is **standing orders** (always-on context).

### The Layered Architecture

The most successful setups use all features together:

```
CLAUDE.md      → Always-on context (coding standards, architecture)
Skills         → On-demand workflows (review, commit, investigate)
Hooks          → Guaranteed automation (format, lint, notify, block)
Sub-Agents     → Parallel work (research, multi-file analysis)
MCP Servers    → External tool integration (Jira, databases, search)
CLI flags      → Session-level control (model, permissions, output format)
```

---

## Start Here: CLAUDE.md

CLAUDE.md is not a "feature" like the others, but **it's the single most impactful configuration** for getting good results. It's a Markdown file at your project root that Claude reads at the start of every conversation.

**Keep it short.** For each line, ask "Would removing this cause Claude to make mistakes?" Bloated files cause Claude to ignore your actual instructions as they get lost in noise.

**It guides behavior but doesn't enforce it.** Claude can and does ignore CLAUDE.md instructions, especially as the context fills. If something MUST happen, use a hook.

**What belongs here:**
- Build/test commands (`npm test`, `cargo build`)
- Code style conventions that differ from defaults
- Architecture decisions (where files go, naming patterns)
- "Never do X" instructions (never modify `src/generated/`, always run lint after changes)

**What doesn't:**
- Anything needing guaranteed enforcement (use hooks)
- Task-specific workflows (use skills)
- Generic programming advice Claude already knows

### The Three-Attempt Reality

From a [staff engineer's 6-week journey](https://www.sanity.io/blog/first-attempt-will-be-95-garbage):
- **First attempt:** 95% garbage. But you learn what Claude misunderstands.
- **Second attempt:** 50% garbage. You've refined CLAUDE.md based on failures.
- **Third attempt:** Finally workable. Your project documentation is now calibrated.

CLAUDE.md is an iterative document. You refine it based on where Claude fails.

---

## Skills (Custom Slash Commands)

### What They Are

Markdown files that become slash commands. Put a `.md` file in `.claude/commands/` (project-level, shared via git) or `~/.claude/commands/` (personal), and it becomes `/project:<name>` or `/<name>`. The file's content becomes the prompt. `$ARGUMENTS` is the only dynamic parameter.

The design is deliberately simple. No YAML, no plugin API, no DSL. Just Markdown.

### When They Shine

Skills are **prompt engineering encapsulation** — your best prompts saved for reuse instead of lost to chat history. The sweet spot is repeatable prompts you'd otherwise retype:

- **Code review checklists** — the most popular skill. Type `/project:review` instead of remembering 10 things to check.
- **PR/commit message generation** — standardized formatting across the team.
- **Bug investigation templates** — structured approaches ensuring consistent debugging.
- **Onboarding** — new team members get the same prompts as veterans.

Build skills for *style* — how code looks and functions (boilerplate, patterns, testing standards). This extends your CLAUDE.md without polluting global context.

### When They Don't

- **One-off tasks** — creating a skill for something you'll do once is overhead.
- **Anything requiring logic** — no conditional branching, no loops, no composition. If you need "if TypeScript do X, else do Y," you can't express it.
- **Heavy parameterization** — `$ARGUMENTS` is your only variable. No named params, no flags.
- **Broad "mega-skills"** — skills that try to cover too much often fail to activate or produce diluted results.
- **When CLAUDE.md would suffice** — if an instruction applies to every interaction, it belongs in CLAUDE.md, not a skill.

### What the Community Reports

**Positive:** Teams report dramatic alignment from project-level skills committed to git. The low barrier to entry is universally praised.

**Negative:** Overly prescriptive skills backfire — Claude follows the letter rather than the spirit. Discoverability is weak — team members don't know what commands exist unless they explore `.claude/commands/`. Argument placement matters more than expected ("Review $ARGUMENTS for security issues" vs "Do a security review focusing on $ARGUMENTS" produces meaningfully different results). Many developers haven't adopted skills despite their simplicity.

### Popular Examples

| Skill | Purpose |
|---|---|
| `/review` | Code review against a project-specific checklist |
| `/commit` | Generate conventional commit messages from staged changes |
| `/test` | Write comprehensive tests for a file/module |
| `/investigate <bug>` | Structured bug investigation: find code, trace flow, identify root cause |
| `/pr` | Generate PR descriptions from branch changes |
| `/security` | OWASP-style security audit |
| `/explain <module>` | Onboarding-style deep explanation of a code area |

Community collections: [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | [Claude Command Suite (148+ commands)](https://github.com/qdhenry/Claude-Command-Suite)

---

## Hooks

### What They Are

Shell commands (or LLM prompts, or sub-agent invocations) that fire automatically at lifecycle events. They are the **hard guarantee** layer — while CLAUDE.md says "please format your code," a hook *forces* it every time.

13 lifecycle events including `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `Notification`, `SubagentStop`. Hooks communicate via stdin (JSON in), stdout (JSON decisions out), and exit codes.

### When They Shine

Hooks are for **certainty, not intelligence.** Use them when behavior must be guaranteed:

- **Auto-formatting after edits** — the most popular hook. PostToolUse on `Write|Edit` running Prettier/Black/gofmt.
- **Security guardrails** — PreToolUse blocking `rm -rf`, `sudo`, `chmod 777`, or changes to `.env`.
- **Notifications** — Stop hooks notifying you when Claude finishes (`say 'Claude is done'` on macOS is the iconic example).
- **Branch protection** — preventing changes on `main`/`master`.
- **Headless safety** — when running with `--dangerously-skip-permissions`, hooks are your only guardrails.
- **Input modification** (since v2.0.10) — PreToolUse hooks can rewrite tool parameters (intercept `rm -rf` and change it to `rm -i`).

**The "Gatekeeper" pattern:** Run `npm test` before committing. If it fails, Claude must fix it. This creates a tight feedback loop where Claude self-corrects without your intervention.

**The "Context Injector":** A hook running `git status` or `ls -R` at session start, ensuring Claude always knows the ground truth state of the repo.

### When They Don't

- **Auto-formatting on every edit creates context noise.** When your formatter changes files, Claude gets a system reminder about those changes. Aggressive formatting floods the context window. **Community wisdom: format on commit, not on every edit.**
- **Synchronous hooks block execution.** One team reported 180 seconds of delay per interaction from stacking linter + formatter + git logger + metrics hooks. Use `async: true` for anything that doesn't need to block.
- **Don't use hooks for complex reasoning.** Hooks should be fast, deterministic checks.
- **The "Silent Assassin" anti-pattern:** Hooks that modify code (like auto-formatters) without telling Claude. Claude gets confused why the file changed. Always let Claude know via tool output if a hook changed something.
- **The "Backseat Driver" anti-pattern:** Interrupting Claude while it's writing code (linting every file save) breaks its chain of thought and wastes tokens. Validate *after* the work is done.

### Critical Bugs and Gotchas

1. **PreToolUse exit codes can be IGNORED** ([#21988](https://github.com/anthropics/claude-code/issues/21988)) — operations proceed despite non-zero exit codes. Don't rely solely on hooks for security-critical blocking without verifying this is fixed in your version.
2. **Progressive duplication bug** ([#3523](https://github.com/anthropics/claude-code/issues/3523)) — hooks multiply during long sessions, causing 10+ simultaneous ESLint processes. Severe CPU/memory degradation.
3. **Matchers are case-sensitive** — `"bash"` will NOT match the `Bash` tool.
4. **All matching hooks run in parallel** — no ordering or dependency definition. Sequential execution is a [feature request](https://github.com/anthropics/claude-code/issues/21533).
5. **Home directory quirk** — user-level Stop hooks don't fire from `~` ([#15629](https://github.com/anthropics/claude-code/issues/15629)).

### Adoption Reality

Despite being the most powerful extensibility mechanism, one survey noted **"approximately nobody is using them."** They require shell scripting knowledge, configuration is JSON in settings files (less discoverable than dropping a Markdown file), and the bugs above erode trust. Those who do use them are enthusiastic.

### Popular Examples

**Notification:**
```json
{"hooks": {"Stop": [{"matcher": "", "hooks": [{"type": "command", "command": "say 'Claude has finished working'"}]}]}}
```

**Auto-format:**
```json
{"hooks": {"PostToolUse": [{"matcher": "Write|Edit", "hooks": [{"type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATHS"}]}]}}
```

**Block destructive commands:** A PreToolUse hook on `Bash` that parses stdin JSON, checks for `rm -rf /`, `sudo`, `chmod 777`, and exits with code 2 to block.

**LLM safety check:**
```json
{"matcher": "Bash", "hooks": [{"type": "prompt", "prompt": "Does this command look safe to execute? $ARGUMENTS"}]}
```

---

## Sub-Agents (The Task Tool)

### What They Are

Spawned Claude instances with isolated context windows. The main agent delegates work via the `Task` tool; sub-agents execute independently with their own tool access. Up to 10 concurrent.

**Critical:** Sub-agents do NOT see your conversation history. They only receive the explicit prompt you give them. This is both the key strength (prevents context pollution, enables parallelism) and the key weakness (they need detailed, explicit context).

### When They Shine

Sub-agents have a clear sweet spot: **parallel, independent work on separate files/domains.** The golden rule: parallel only works when agents touch different files.

- **Large-scale codebase research** — "Find all usages of deprecated function X across 200 files"
- **Fan-out analysis** — security review + performance review + accessibility review, each in parallel
- **Documentation generation** — primary agent lists functions, sub-agents document each, final agent assembles
- **Multi-module refactoring** — frontend agent handles React components while backend agent manages API routes

Use agents for *scope* — when you don't know the full scope of the work. Let the Explore agent build the mental map so you don't have to context-dump manually.

Anthropic's engineering blog describes using [16 Claude instances simultaneously to build a C compiler](https://www.anthropic.com/engineering/building-c-compiler), with each agent taking a "lock" on a task via text files in a `current_tasks/` directory.

### When They Don't

- **Over-parallelizing wastes tokens.** "Max out at three or four specialized agents. More than that decreases productivity rather than increasing it."
- **Explore agent summaries can be lossy.** One power user [explicitly prefers having Opus read files itself](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/) because direct reads enable "proper attention relationships across context."
- **Context density is critical.** Bad: "Fix authentication." Good: "Fix OAuth redirect loop where successful login redirects to /login instead of /dashboard. Reference the auth middleware in src/lib/auth.ts."
- **Inter-agent communication is limited.** Sub-agents can't talk to each other. The most successful pattern at scale is **file-based communication**: each agent writes results to a markdown file, which the next agent reads.
- **Simple tasks don't need them.** If you're editing one file, the overhead of spawning is wasteful. Don't hire a contractor to change a lightbulb.
- **Tasks needing continuous shared reasoning** — rapid back-and-forth where coordination overhead slows progress.

### Practical Heuristic

If the task is "research or review" rather than "decide or build," use a sub-agent. If it needs unified reasoning or tight integration, keep it in the main agent.

### Patterns

| Pattern | Description | When to Use |
|---|---|---|
| **Fan-Out/Fan-In** | Parallel research, synthesize results | Gathering info from many independent sources |
| **Expert Panel** | Security + Performance + A11y experts review the same code | Multi-perspective analysis |
| **Pipeline** | Task A → Task B → Task C | Sequential processing stages |
| **Background Research** | Web lookups, doc exploration while main thread works | Non-blocking info gathering |

### Cost Optimization

Running main sessions on Opus with Sonnet sub-agents (`CLAUDE_CODE_SUBAGENT_MODEL`) significantly reduces costs while maintaining quality for research/review tasks.

---

## Commands & CLI

### Interactive Slash Commands

| Command | What It Does | Nuanced Take |
|---|---|---|
| `/compact` | Summarizes conversation to free context | **The most important command for long sessions.** Use proactively at ~60% context, not when you hit the wall. Accepts instructions: `/compact focus on the database migration work`. |
| `/clear` | Wipes conversation, starts fresh | Use when switching tasks entirely. Don't use mid-task. |
| `/review` | Code review of current changes | Good for quick reviews. |
| `/pr` | Create a pull request | Analyzes branch, drafts title/description, creates PR via `gh`. Very popular. |
| `/commit` | Create a git commit | Generates conventional commit messages. |
| `/init` | Create CLAUDE.md | The **single most impactful thing you can do** for quality. |
| `/model` | Switch models mid-session | Sonnet for fast tasks, Opus for complex reasoning. |
| `/cost` | Show token usage | Track this regularly. Eye-opening. |

### CLI Flags (The Power User Arsenal)

**Essential:**
- `-p` / `--print` — Non-interactive mode. The gateway to scripting and CI/CD.
- `-c` / `--continue` — Resume the most recent conversation. Workflow game-changer.
- `-r` / `--resume` — Resume a specific past session by search.
- `--model <model>` — Override model. Aliases: `sonnet`, `opus`.
- `--output-format json` — Structured output for parsing in scripts.
- `--max-budget-usd` — Cap spending. Only works with `-p`.

**Permission control:**
- `--allowed-tools "Bash(git:*) Read"` — Whitelist specific tools with glob patterns.
- `--disallowed-tools "Bash(rm:*)"` — Blacklist specific tools.
- `--permission-mode plan` — Claude only plans, never executes. Great for reviewing what Claude *would* do.
- `--dangerously-skip-permissions` — Only in sandboxed environments.

**CI/CD patterns:**
```bash
# One-shot code review
claude -p "Review this PR for security issues" --output-format json --max-budget-usd 2.00

# Pipe logs for analysis
tail -f app.log | claude -p "Alert me if you see anomalies"

# Batch processing
git diff main --name-only | claude -p "Review changed files for security issues"
```

**Session management:**
- `--from-pr` — Jump back into the session that created a specific PR.
- `--fork-session` — Branch off a conversation without mutating the original.

---

## MCP Servers & Plugins

The Model Context Protocol connects Claude Code to external tools and data sources. 50+ curated MCP servers exist as of 2026.

**Most popular:**
- **GitHub MCP** — repos, issues, PRs, workflows
- **Brave Search MCP** — real-time web search
- **Context7** — live, version-accurate documentation instead of stale training data
- **Supabase MCP** — database operations via natural language
- **Firecrawl** — web scraping with JS rendering and anti-bot handling

**Gotchas:** MCP servers bloat context upfront with tool definitions. Claude Code's Tool Search enables lazy loading (up to 95% context reduction), but you need to configure it. Not all servers are production-ready — if they crash, Claude loses access mid-session.

**Plugins** (Claude Code 2.1+) bundle slash commands, agents, skills, and hooks together. Install with `/plugin install`. A [community marketplace](https://claude-plugins.dev/) is emerging.

---

## Workflow Playbooks

### The Architect-Builder Loop (New Features)

1. **Plan (Agent):** Use the Plan agent. "Research the Stripe API and plan the subscription checkout flow."
   - Output: a markdown plan detailed enough for a human to review.
2. **Review (Human):** This is your highest-leverage intervention point. Correct the logic here, not in the code.
3. **Build (Skill):** "Implement step 1 of `plan.md` using the `/new-component` skill."

### The Refactor Sweeper (Tech Debt)

1. **Define:** Create a temporary skill that defines the exact transformation (e.g., "Replace `moment.js` with `date-fns`").
2. **Execute:** "Find all usages of `moment()` in `src/utils` and apply the refactor skill."
3. **Verify:** A pre-commit hook runs tests. If they fail, Claude iterates.

### The 20-Turn Rule

If a task takes more than 20 turns, **stop.** Commit the work. Restart the session. Treat sessions as "work days" — clear the slate regularly. `/compact` helps but it's lossy.

---

## The Hard Truths

### The Confidence Trap

Claude is exceptionally good at making code *look* correct. It will confidently introduce a subtle bug while perfectly matching your indentation style. **Mitigation:** Ask Claude to explain potential edge cases *before* it writes the code.

### Usage Limits

The #1 community pain point. Pro subscribers reported being reduced from 40-50 hours/week to 6-8 hours after model rollouts. No clear visibility into actual limits or tracking. Community tools like [ccusage](https://github.com/) emerged to fill transparency gaps.

### Context Loss & Auto-Compaction

Claude forgets project context mid-session. Auto-compact aggressively discards details. Developers describe "racing against compaction." Performance degrades as project complexity increases.

### Confident Incompetence

Claude produces broken code with conviction. Claims tasks are complete when they're incomplete. Generates fake tests that don't verify functionality. One staff engineer reported "only 30% first-try reliability — not from broken code, but from poor architectural decisions creating downstream problems."

### Over-Engineering

Adds unnecessary complexity instead of simple solutions. Creates duplicate files, odd filenames, incomplete refactors. The system prompt explicitly counters this tendency, but it persists.

### Cost

$1,000-1,500/month for senior engineers using it heavily. Opus access requires $100-200/month Claude Max. ROI is there for some (2-3x faster feature shipping), but requires a substantial learning curve.

### The Reframe That Determines Success

> "Treat AI like a junior developer who doesn't learn."

> "I'm more critical of 'my code' now because I didn't type a lot of it. No emotional attachment means better reviews."

> "The goal isn't to code without AI, but to be a better developer because of AI — and sometimes that means turning it off."

---

## Getting Started (Priority Order)

1. **Run `/init`** to create your CLAUDE.md. This is the highest-ROI action.
2. **Iterate on CLAUDE.md** based on where Claude fails. Expect 2-3 rounds.
3. **Build 1-2 skills** for your most painful, repetitive task (e.g., "New API Endpoint," "Code Review").
4. **Add a Stop hook** for notifications — easy win, zero risk.
5. **Only then** explore sub-agents, MCP servers, and advanced hooks.

Don't write custom agents yet. Don't stack 5 hooks on day one. Start simple, add complexity as you understand the failure modes.

---

## References

### Official Documentation
- [Overview](https://code.claude.com/docs/en/overview) | [Skills](https://code.claude.com/docs/en/skills) | [Hooks](https://code.claude.com/docs/en/hooks) | [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) | [Sub-Agents](https://code.claude.com/docs/en/sub-agents) | [Best Practices](https://code.claude.com/docs/en/best-practices) | [CLI Reference](https://code.claude.com/docs/en/cli-reference) | [Headless Mode](https://code.claude.com/docs/en/headless) | [MCP](https://code.claude.com/docs/en/mcp)

### Anthropic Engineering
- [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) | [Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | [Enabling Autonomous Claude Code](https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously) | [How to Configure Hooks](https://claude.com/blog/how-to-configure-hooks)

### Developer Experience
- [Sanity: First Attempt Will Be 95% Garbage](https://www.sanity.io/blog/first-attempt-will-be-95-garbage) | [Sankalp: Claude Code 2.0 Experience](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/) | [Builder.io: How I Use Claude Code](https://www.builder.io/blog/claude-code) | [sshh.io: How I Use Every Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) | [Getting Good Results](https://www.dzombak.com/blog/2025/08/getting-good-results-from-claude-code/) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Kelsey Piper: I Can't Stop Yelling at Claude Code](https://www.theargumentmag.com/p/i-cant-stop-yelling-at-claude-code) | [Medium: Hooks 6-Month Production Report](https://alirezarezvani.medium.com/the-claude-code-hooks-nobody-talks-about-my-6-month-production-report-30eb8b4d9b30)

### Community & Issues
- [The Register: Usage Limits](https://www.theregister.com/2026/01/05/claude_devs_usage_limits/) | [GitHub #9094: Usage Limits](https://github.com/anthropics/claude-code/issues/9094) | [Community Struggles Gist](https://gist.github.com/eonist/0a5f4ae592eadafd89ed122a24e50584) | [GitHub #21988: Hook Exit Codes](https://github.com/anthropics/claude-code/issues/21988) | [GitHub #3523: Hook Duplication](https://github.com/anthropics/claude-code/issues/3523)

### Collections & Toolkits
- [awesome-claude-code (21.6k stars)](https://github.com/hesreallyhim/awesome-claude-code) | [awesome-claude-code-toolkit](https://github.com/rohitg00/awesome-claude-code-toolkit) | [everything-claude-code](https://github.com/affaan-m/everything-claude-code) | [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) | [Claude Plugins Directory](https://claude-plugins.dev/)

### Tutorials
- [alexop.dev: Full Stack Guide](https://alexop.dev/posts/understanding-claude-code-full-stack/) | [OneAway: Skills Complete Guide](https://oneaway.io/blog/claude-code-skills-slash-commands) | [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) | [ClaudeFast: Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) | [F22 Labs: 10 Productivity Workflows](https://www.f22labs.com/blogs/10-claude-code-productivity-tips-for-every-developer/)

---

*Compiled February 2026. Claude Code evolves rapidly — verify against [current docs](https://code.claude.com/docs/en/overview).*
