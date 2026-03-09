# Claude Code: The Practical Guide

Claude Code is Anthropic's terminal-based agentic coding tool. It reads your codebase, edits files, runs commands, and iterates — less like autocomplete and more like a tireless but amnesia-prone junior developer who needs clear direction and guardrails. Available in the terminal, VS Code, JetBrains, as a desktop app, and in-browser.

This guide covers **when and how to use** each extensibility feature, based on real developer experience and current official docs. The biggest shift since late 2025: Claude Code now draws a clean line between explicit **slash commands** and model-invoked **Agent Skills**.

---

## Quick Decision Framework

Before diving in, orient yourself. Each tool serves a distinct purpose — confusion comes from using one where another belongs.

| You need... | Use | Not |
|---|---|---|
| Behavior that must happen every time, no exceptions | **Hook** | CLAUDE.md (advisory, can be ignored) |
| A repeatable workflow Claude should discover automatically | **Agent Skill** | Slash command (explicit only) |
| An explicit shortcut like `/review-pr` or `/commit` | **Custom slash command** | Agent Skill (not directly invoked) |
| Context Claude should always have | **CLAUDE.md** | Skill (on-demand only) |
| Parallel work across many files/domains | **Sub-Agent** | Main thread (context will overflow) |
| Blocking dangerous commands | **Hook** (PreToolUse) | CLAUDE.md (unenforceable) |
| A quick one-off task | **Inline prompt** | Skill/Hook (overhead for one-shot work) |
| Notification when Claude finishes | **Hook** (Stop) | Nothing else can trigger on completion |
| External tool integration (Jira, DB, search) | **MCP Server** | Not a skill/command problem |
| Claude in CI/CD | **CLI with `-p`** | Interactive mode |
| Claude to plan before acting | **`--permission-mode plan`** | — |

**The mental model:** Slash commands are **shortcuts**. Skills are **capabilities**. Sub-Agents are **contractors**. Hooks are **guardrails**. CLAUDE.md is **standing memory**.

### The Layered Architecture

The most successful setups keep the always-on layer thin and push detail into on-demand layers:

```
Always-on (small):
  CLAUDE.md          → Routing table + non-negotiables (60-80 lines)
  Skill metadata     → Names + descriptions used for discovery
  Command metadata   → Names + descriptions for SlashCommand routing
  Hooks              → Guaranteed automation (format, lint, notify, block)

On-demand (loaded when matched):
  .claude/skills/    → Multi-file capabilities (SKILL.md + resources)
  .claude/commands/  → Explicit slash-command prompts
  .claude/agents/    → Specialist delegation (own context window)
  .claude/rules/     → Topic-specific context (loaded by path/relevance)
  MCP Servers        → External tools (lazy-loaded via Tool Search)
  Plugins            → Distribute commands, skills, hooks, agents, MCP

Session-level:
  CLI flags          → Model, permissions, output format
```

The guiding principle from Anthropic's context engineering research: **["Find the smallest set of high-signal tokens that maximize the likelihood of your desired outcome."](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)**

---

## Start Here: CLAUDE.md

CLAUDE.md is not a "feature" like the others, but **it's the single most impactful configuration** for getting good results. It's a Markdown file at your project root that Claude reads at the start of every conversation.

### The Instruction Budget

Claude's system prompt already contains ~50 instructions. Research shows frontier LLMs follow approximately 150-200 instructions with reasonable consistency, with [degradation as count increases](https://www.humanlayer.dev/blog/writing-a-good-claude-md). Your CLAUDE.md is competing for attention in a limited budget.

**Target: 60-80 lines.** Above ~150 lines, multiple teams report Claude ignoring instructions outright ([#6120](https://github.com/anthropics/claude-code/issues/6120), [#15443](https://github.com/anthropics/claude-code/issues/15443), [#19471](https://github.com/anthropics/claude-code/issues/19471)). [Arize found](https://arize.com/blog/claude-md-best-practices-learned-from-optimizing-claude-code-with-prompt-learning/) that optimizing system prompt content alone yielded 5%+ gains in general coding and +11% for repo-specific work. HumanLayer's own CLAUDE.md is [under 60 lines](https://github.com/humanlayer/humanlayer/blob/main/CLAUDE.md).

For each line, ask: **"Would removing this cause Claude to make mistakes?"** If not, cut it.

### What It Is (and Isn't)

**It guides behavior but doesn't enforce it.** Claude can and does ignore CLAUDE.md instructions, especially as the context fills. If something MUST happen, use a hook. One developer's CLAUDE.md explicitly prohibited hardcoding API keys — [Claude did it anyway, costing $30,000](https://paddo.dev/blog/claude-code-hooks-guardrails/).

**Think of CLAUDE.md as a routing table**, not a manual. It should tell Claude: what the project is, how to build/test it, where things live, and what to never do. Everything else should live in skills (on-demand), hooks (enforced), or `.claude/rules/` (topic-specific).

**What belongs here:**
- Build/test commands (`npm test`, `cargo build`)
- Code style conventions that differ from defaults
- Architecture overview (where files go, module boundaries)
- Hard "never do X" rules (never modify `src/generated/`, never commit `.env`)
- Routing hints: which skills/agents to use for what

**What doesn't:**
- Anything needing guaranteed enforcement (use hooks)
- Task-specific workflows (use skills — they load on demand, ~100 tokens for metadata)
- Generic programming advice Claude already knows
- Code style rules a linter can enforce — ["never send an LLM to do a linter's job"](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- Personality instructions ("be a senior engineer") — wastes tokens, changes nothing

### Progressive Disclosure

For projects that outgrow 80 lines, use progressive disclosure instead of a massive CLAUDE.md:

```
CLAUDE.md              → Routing table + non-negotiables (60-80 lines)
.claude/rules/         → Topic-specific context
.claude/commands/      → Explicit slash-command workflows
.claude/skills/        → Model-invoked capabilities with resources
.claude/agents/        → Specialist delegation
Hooks                  → Guaranteed enforcement
```

Each layer loads only when needed. [Skills load at three levels](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/): metadata (~100 tokens always), instructions (<5K tokens when matched), resources (only during execution). This keeps the always-on context budget small while giving Claude access to deep knowledge on demand.

### The Three-Attempt Reality

From a [staff engineer's 6-week journey](https://www.sanity.io/blog/first-attempt-will-be-95-garbage):
- **First attempt:** 95% garbage. But you learn what Claude misunderstands.
- **Second attempt:** 50% garbage. You've refined CLAUDE.md based on failures.
- **Third attempt:** Finally workable. Your project documentation is now calibrated.

CLAUDE.md is an iterative document. You refine it based on where Claude fails. Even [Anthropic's own Claude Code team](https://paddo.dev/blog/how-boris-uses-claude-code/) contributes to their CLAUDE.md multiple times a week — and ruthlessly prunes it.

---

## Agent Skills

### What They Are

Agent Skills are **model-invoked capabilities** stored as directories containing a `SKILL.md` file plus optional scripts, templates, and reference docs. In Claude Code they live in `.claude/skills/<skill-name>/SKILL.md` for project skills or `~/.claude/skills/<skill-name>/SKILL.md` for personal skills. Installed plugins can bundle skills too.

This is a different feature from custom slash commands:

- **Slash commands** are explicit and live in `.claude/commands/*.md`
- **Skills** are automatic and live in `.claude/skills/<name>/SKILL.md`
- Both can coexist, and plugins can ship both

Anthropic launched Agent Skills on **October 16, 2025**. In Claude Code, only **custom** skills are supported; Anthropic's prebuilt document skills (PDF, Word, Excel, PowerPoint) are for Claude.ai and the API rather than local Claude Code sessions.

### The Routing Mechanism

Claude discovers skills from three sources:

- `~/.claude/skills/`
- `.claude/skills/`
- Installed plugin `skills/` directories

Each skill needs YAML frontmatter in `SKILL.md` with at least:

```yaml
---
name: code-review
description: Review code for correctness, security, performance, and test gaps. Use when reviewing diffs, PRs, or recent code changes.
---
```

The `description` is the routing surface. Claude decides whether to load the skill based on that description and your request. The practical implication is simple: **be specific about both what the skill does and when Claude should use it**.

Unlike slash commands, you do not type a skill name to trigger it. You test a skill by asking for the task naturally and seeing whether Claude selects it.

### Why This Matters

Skills are the right abstraction when one capability needs more than a single prompt file:

- A code-review capability with checklists, shell helpers, and reference docs
- A release skill with rollout instructions, runbooks, and verification scripts
- A data-migration skill with templates and project-specific gotchas
- A PDF/domain-processing skill with scripts and supporting documentation

### When They Shine

Skills are best when you need **organized, reusable expertise** rather than a snippet:

- **Multi-file workflows** — instructions plus scripts plus references
- **Team-standard capabilities** — shared via git or plugins
- **Automatic discovery** — Claude should reach for the workflow on its own
- **Long-lived domain knowledge** — runbooks, policies, templates, style guides

This is the strongest new Claude Code feature if you want a repo to "teach" Claude how your team works without bloating `CLAUDE.md`.

### Best Practices

Official guidance and field experience line up on a few patterns:

- **Keep them focused.** One capability per skill beats a giant "everything" skill.
- **Write trigger-rich descriptions.** Mention file types, task names, and situations.
- **Package the real artifacts.** Put checklists, templates, scripts, and examples alongside `SKILL.md`.
- **Version them in the file.** A short version-history section makes team changes understandable.
- **Share broadly via plugins when needed.** Anthropic explicitly recommends plugins for reusable distribution beyond a single repo.

### January 2026 Skill-Builder Guide

Anthropic published a more detailed skill-authoring guide in **January 2026**. The official PDF is here, and there is also a community Markdown conversion that is easier to skim in-browser:

- [The Complete Guide to Building Skills for Claude (official Anthropic PDF)](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf?hsLang=en)
- [Skill Builder Guide (community Markdown conversion of Anthropic PDF)](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a)
- [Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills)

High-signal takeaways from that guide:

- **Validate routing with real prompts.** Anthropic recommends testing both "should trigger" and "should not trigger" examples instead of trusting the first description draft.
- **Extract from successful conversations.** Their preferred workflow is to get a real task working in chat first, then turn the winning prompt/process into a skill.
- **Keep `SKILL.md` lean.** The guide suggests keeping the core file under roughly **5,000 words** and moving supporting detail into `references/` or scripts.
- **Use negative triggers when skills overlap.** A blunt "Do NOT use for..." clause is explicitly recommended when neighboring skills start colliding.
- **Bundle deterministic checks.** If a step matters, ship a script or template for it rather than hoping the model remembers the instruction every time.
- **Describe outcomes, not implementation trivia.** Users and models route better off task language ("review contracts", "triage CI failures") than off internal packaging details.

### Making Skills Auto-Invoke Reliably

If the goal is "Claude should reach for this on its own," the practical rules are:

- **Treat `description` as the routing API.** Put the task, the natural trigger phrases, and the scope boundary in that one field.
- **Mirror how humans ask.** Include the verbs, artifacts, file types, and domain terms users actually use in prompts.
- **Be narrow enough to be distinct.** "Process PDFs" is weak; "review PDF contracts for legal-risk issues and clause extraction" is routable.
- **Say what not to do.** Negative routing language helps prevent both false positives and skill collisions.
- **Front-load the execution path.** Once loaded, Claude should immediately see the workflow, constraints, and which supporting files/scripts to read next.

Example:

```yaml
---
name: contract-review
description: Review PDF contracts for legal-risk issues, clause extraction, and red-flag summaries. Use when the user asks to review contracts, extract clauses from agreements, or summarize legal terms. Do NOT use for general PDF extraction or invoice processing.
---
```

### When They Don't

- **If you need explicit control** — use a slash command instead.
- **If the workflow fits in one markdown file** — a slash command is lighter.
- **If the instruction should always be in force** — put it in `CLAUDE.md` or a hook.
- **If the skill is too vague** — Claude simply won't reach for it reliably.

### Operational Reality

- Skills in Claude Code are **filesystem-based**. Update the files, then restart Claude Code to reload them.
- Skills do **not** sync automatically across Claude products. Claude.ai uploads, API skills, and local Claude Code skills are separate deployment surfaces.
- Plugin skills are increasingly the cleanest team distribution model because they bundle versioning, marketplaces, and optional commands/hooks around the same capability.

### A Good Default

Start with 2-3 coarse skills that map to real recurring work:

| Skill | Purpose |
|---|---|
| `code-review` | Review diffs/PRs using your team's checklist and helper scripts |
| `debug-workflow` | Structured investigation for failing tests, runtime bugs, and regressions |
| `release-checks` | Release/runbook validation with project-specific scripts |

If you also want explicit entry points, pair each skill with a thin slash command that tells Claude to use it.

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

**The "Context Refresher":** A UserPromptSubmit hook that [re-injects important context every N prompts](https://gist.github.com/johnlindquist/23fac87f6bc589ddf354582837ec4ecc). SessionStart context gets pushed back and "forgotten" in long sessions — this pattern solves that. Whatever a UserPromptSubmit hook writes to stdout gets added to Claude's context alongside your prompt. Use this to inject project state, sprint priorities, or routing hints that influence which skills/agents Claude reaches for.

**A better routing pattern than prompt-time shell heuristics:** If your goal is "Claude should choose the right workflow on its own," do **not** build a fragile `UserPromptSubmit` classifier in Bash that pattern-matches user prompts and tells Claude what workflow to use. That pushes semantic routing into the least reliable layer. A stronger pattern is:

- keep workflow routing in **skill descriptions** and **sub-agent descriptions**
- keep the always-on memory layer short and explicit about routing precedence
- use hooks only for **hard policy** and **lightweight session context**
- if you want a reminder, prefer a single **SessionStart** routing summary over per-prompt classification

The practical rule is simple: **hooks should enforce, not interpret.** Let Claude do semantic routing natively through skills and agents; use hooks for blocking dangerous actions, formatting, notifications, or injecting a short session-level reminder.

### When They Don't

- **Auto-formatting on every edit creates context noise.** When your formatter changes files, Claude gets a system reminder about those changes. Aggressive formatting floods the context window. **Community wisdom: format on commit, not on every edit.**
- **Synchronous hooks block execution.** One team reported 180 seconds of delay per interaction from stacking linter + formatter + git logger + metrics hooks. Use `async: true` for anything that doesn't need to block.
- **Don't use hooks for complex reasoning.** Hooks should be fast, deterministic checks.
- **Don't turn `UserPromptSubmit` into a workflow router by keyword-matching prompts.** That is brittle, adds latency on every prompt, and can fight Claude's native skill/sub-agent routing. If routing is weak, fix the skill and agent `description` fields first.
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

Custom subagents can also be routed automatically. Anthropic's docs use the same basic advice as skills: write a tight description, make the specialization obvious, and let the main agent invoke them when the task matches that niche rather than telling Claude to "use agent X" every time.

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

If you want automatic delegation rather than manual prompting, the best pattern is:

- Give each agent a **single sharp specialty** ("React accessibility review", "Postgres query optimization", "log triage").
- Put the **trigger language in the description**, not buried deep in instructions.
- Keep tool access and scope aligned with that specialty so the agent feels safe to call.
- Avoid overlapping agents that all sound like "general engineering help" or Claude will route inconsistently.

### Patterns

| Pattern | Description | When to Use |
|---|---|---|
| **Fan-Out/Fan-In** | Parallel research, synthesize results | Gathering info from many independent sources |
| **Expert Panel** | Security + Performance + A11y experts review the same code | Multi-perspective analysis |
| **Pipeline** | Task A → Task B → Task C | Sequential processing stages |
| **Background Research** | Web lookups, doc exploration while main thread works | Non-blocking info gathering |

### Cost Optimization

Running main sessions on Opus with Sonnet sub-agents (`CLAUDE_CODE_SUBAGENT_MODEL`) significantly reduces costs while maintaining quality for research/review tasks.

### Agent Teams (Experimental, New in Feb 2026)

Agent Teams are different from regular sub-agents: they add a lead + teammate model where teammates can communicate with each other and share a task list.

**Enablement:** off by default. Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (env or settings).

**What appears true so far (confidence-ranked):**

- **High confidence (official):** Feature exists and is explicitly experimental, with known limitations (resume gaps, task status lag, slower shutdown, no nested teams, one team per lead session). Cost can be much higher than single-session flows (docs call out roughly ~7x in some plan-mode scenarios).
- **Medium confidence (named practitioners with hands-on reports):**
  - [Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/) reports strongest wins on parallelizable work (clear ownership boundaries, independent files/domains), and highlights cost/coordination overhead if decomposition is weak.
  - [Eric Buess](https://www.linkedin.com/posts/ebuess_claude-code-agent-teams-supervision-by-autonomous-activity-7335740096966541312-wm2x) describes an "autonomous supervision" workflow where lead reviews teammate output before integration.
  - [Matthew Hartman](https://www.linkedin.com/posts/mjhartman_enhance-your-development-workflow-with-agent-activity-7336953312526481408-Olpc) reports agent teams helped front-end + back-end split work in parallel on smaller feature tasks.
- **Lower confidence (anonymous but repeated):** HN/Reddit users consistently report two themes: big speedups for embarrassingly parallel tasks, and steep token burn/permission friction for tightly-coupled work.

**Practical heuristic (2026 reality):**

- Use Agent Teams for **parallel discovery/review/build lanes** with low file overlap.
- Prefer normal sub-agents (or single agent) for **sequential reasoning** or shared-file edits.
- Cap to **2-4 teammates** first; productivity and costs often worsen past that unless orchestration is mature.
- Treat teams as **throughput tool, not quality guarantee**. Keep tests and explicit acceptance criteria in the lead prompt.

---

## Commands & CLI

### Built-In Interactive Commands

| Command | What It Does | Nuanced Take |
|---|---|---|
| `/compact` | Summarizes conversation to free context | **The most important command for long sessions.** Use proactively at ~60% context, not when you hit the wall. Accepts instructions: `/compact focus on the database migration work`. |
| `/clear` | Wipes conversation, starts fresh | Use when switching tasks entirely. Don't use mid-task. |
| `/review` | Code review of current changes | Good for quick reviews. |
| `/pr` | Create a pull request | Analyzes branch, drafts title/description, creates PR via `gh`. Very popular. |
| `/commit` | Create a git commit | Generates conventional commit messages. |
| `/init` | Create CLAUDE.md | The **single most impactful thing you can do** for quality. |
| `/memory` | Edit memory files | Useful once your `CLAUDE.md` and imported memory layers become part of the workflow. |
| `/plugin` | Discover, install, and manage plugins | Important now that marketplaces and plugin-packaged skills are first-class. |
| `/agents` | Manage custom sub-agents | Good for specialist delegation as your setup matures. |
| `/mcp` | Manage MCP servers | Faster than editing config by hand once you have several integrations. |
| `/permissions` | Review or update allowed tools | Helpful when balancing speed vs safety. |
| `/doctor` | Diagnose install/environment problems | Worth remembering before blaming the model. |
| `/model` | Switch models mid-session | Sonnet for fast tasks, Opus for complex reasoning. |
| `/usage` | Show token usage and burn | Track this regularly. Eye-opening. |

### Custom Slash Commands

Custom slash commands still matter. They are the right tool when you want an explicit, typed entry point such as `/review-pr` or `/ship`.

- Store them in `.claude/commands/*.md` or `~/.claude/commands/*.md`
- Plugins can also bundle namespaced commands
- They support frontmatter such as `description`, `allowed-tools`, `argument-hint`, `model`, and `disable-model-invocation`
- Use them when you want deterministic invocation rather than skill discovery

The current Claude Code model is:

- **Slash command** = "run this exact workflow now"
- **Skill** = "Claude should know how to do this when relevant"

### CLI Flags (The Power User Arsenal)

**Essential:**
- `-p` / `--print` — Non-interactive mode. The gateway to scripting and CI/CD.
- `-c` / `--continue` — Resume the most recent conversation. Workflow game-changer.
- `-r` / `--resume` — Resume a specific past session by search.
- `--model <model>` — Override model. Aliases: `sonnet`, `opus`.
- `--output-format json` — Structured output for parsing in scripts.
- `--max-budget-usd` — Cap spending. Only works with `-p`.

**Permission control:**
- `--allowedTools "Bash(git:*) Read"` — Whitelist specific tools with glob patterns.
- `--disallowedTools "Bash(rm:*)"` — Blacklist specific tools.
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

The Model Context Protocol connects Claude Code to external tools and data sources. Plugins are now the packaging layer that can bundle **skills, slash commands, hooks, agents, and MCP servers** together.

**Most popular:**
- **GitHub MCP** — repos, issues, PRs, workflows
- **Brave Search MCP** — real-time web search
- **Context7** — live, version-accurate documentation instead of stale training data
- **Supabase MCP** — database operations via natural language
- **Firecrawl** — web scraping with JS rendering and anti-bot handling

**Gotchas:** MCP servers bloat context upfront with tool definitions. Claude Code's Tool Search enables lazy loading, but you need to configure it. Not all servers are production-ready — if they crash, Claude loses access mid-session.

### Why Plugins Matter Now

Plugins are where a lot of the new Claude Code surface area lands:

- **Marketplaces** — Claude Code can connect to plugin marketplaces and expose marketplace-installed tools and workflows inside the app
- **Team distribution** — a plugin is the cleanest way to share commands, skills, hooks, and MCP integrations across many repos
- **Code intelligence** — Anthropic documents LSP-based plugins that add diagnostics, jump-to-definition, hover info, and rename support

If you care about skills, plugins matter because they turn a local `.claude/skills/` experiment into something you can version, publish, and reuse.

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

1. **Run `/init`** to create your CLAUDE.md. Keep it under 80 lines — routing table + non-negotiables.
2. **Add a Stop hook** for notifications and a PreToolUse hook for dangerous command blocking — easy wins, zero risk.
3. **Add 1-2 explicit slash commands** in `.claude/commands/` for the workflows you invoke intentionally (`/review-pr`, `/ship`, `/triage`).
4. **Build 2-3 coarse skills** in `.claude/skills/` with strong descriptions and real supporting files. Let each delegate to sub-agents for specialized work.
5. **Iterate on CLAUDE.md** by pruning. When Claude ignores an instruction, the file is probably too long. Cut lines, don't add them.
6. **Only then** explore plugins, MCP servers, and UserPromptSubmit context injection.

Don't front-load everything into CLAUDE.md. Don't build 10 narrow skills. Don't confuse skills with slash commands. Don't stack 5 synchronous hooks. Start simple, add complexity as you understand the failure modes.

---

## References

### Official Documentation
- [Claude Code Overview](https://docs.claude.com/en/docs/claude-code/overview) | [Slash Commands](https://docs.claude.com/en/docs/claude-code/slash-commands) | [Hooks](https://docs.claude.com/en/docs/claude-code/hooks) | [Subagents](https://docs.claude.com/en/docs/claude-code/sub-agents) | [CLI Reference](https://docs.claude.com/en/docs/claude-code/cli-reference) | [MCP](https://docs.claude.com/en/docs/claude-code/mcp)
- [Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) | [Manage Skills in Claude Code](https://docs.claude.com/en/docs/claude-code/tutorials/manage-skills) | [The Complete Guide to Building Skills for Claude (official Anthropic PDF)](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf?hsLang=en) | [Skill Builder Guide (community Markdown conversion of Anthropic PDF)](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a)
- [Plugins Overview](https://docs.claude.com/en/docs/claude-code/plugins/overview) | [Create Plugins](https://docs.claude.com/en/docs/claude-code/plugins/create-plugins) | [Plugin Marketplaces](https://docs.claude.com/en/docs/claude-code/plugins/marketplaces) | [Code Intelligence Plugins](https://docs.claude.com/en/docs/claude-code/plugins/code-intelligence-plugins)
- [Release Notes](https://docs.claude.com/en/release-notes/overview)

### Anthropic Engineering
- [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) | [Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | [Enabling Autonomous Claude Code](https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously) | [How to Configure Hooks](https://claude.com/blog/how-to-configure-hooks)

### Developer Experience
- [Sanity: First Attempt Will Be 95% Garbage](https://www.sanity.io/blog/first-attempt-will-be-95-garbage) | [Sankalp: Claude Code 2.0 Experience](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/) | [Builder.io: How I Use Claude Code](https://www.builder.io/blog/claude-code) | [sshh.io: How I Use Every Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) | [Addy Osmani: Claude Code Agent Teams](https://addyosmani.com/blog/claude-code-agent-teams/) | [Getting Good Results](https://www.dzombak.com/blog/2025/08/getting-good-results-from-claude-code/) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Kelsey Piper: I Can't Stop Yelling at Claude Code](https://www.theargumentmag.com/p/i-cant-stop-yelling-at-claude-code) | [Medium: Hooks 6-Month Production Report](https://alirezarezvani.medium.com/the-claude-code-hooks-nobody-talks-about-my-6-month-production-report-30eb8b4d9b30)

### Architecture & Optimization
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | [Arize: CLAUDE.md Best Practices from Prompt Learning](https://arize.com/blog/claude-md-best-practices-learned-from-optimizing-claude-code-with-prompt-learning/) | [alexop.dev: Progressive Disclosure for AI Coding Tools](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/) | [paddo.dev: Skills Controllability Problem](https://paddo.dev/blog/claude-skills-controllability-problem/) | [paddo.dev: Hooks as Guardrails That Actually Work](https://paddo.dev/blog/claude-code-hooks-guardrails/) | [paddo.dev: How Boris Uses Claude Code](https://paddo.dev/blog/how-boris-uses-claude-code/) | [John Lindquist: Auto-Refresh Context Every N Prompts](https://gist.github.com/johnlindquist/23fac87f6bc589ddf354582837ec4ecc) | [GitButler: Automate Workflows with Hooks](https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks)

### Community & Issues
- [HN: Claude Code Agent Teams Thread](https://news.ycombinator.com/item?id=46902368) | [HN: Opus 4.6 Launch Thread](https://news.ycombinator.com/item?id=46902223) | [Reddit: Agent Teams Early Testing](https://www.reddit.com/r/ClaudeCode/comments/1qwxah9/anyone_testing_agent_teams/) | [Reddit: Agent Teams Discussion](https://www.reddit.com/r/ClaudeCode/comments/1r0n4ma/agent_teams/) | [The Register: Usage Limits](https://www.theregister.com/2026/01/05/claude_devs_usage_limits/) | [GitHub #9094: Usage Limits](https://github.com/anthropics/claude-code/issues/9094) | [Community Struggles Gist](https://gist.github.com/eonist/0a5f4ae592eadafd89ed122a24e50584) | [GitHub #21988: Hook Exit Codes](https://github.com/anthropics/claude-code/issues/21988) | [GitHub #3523: Hook Duplication](https://github.com/anthropics/claude-code/issues/3523)

### Collections & Toolkits
- [awesome-claude-code (21.6k stars)](https://github.com/hesreallyhim/awesome-claude-code) | [awesome-claude-code-toolkit](https://github.com/rohitg00/awesome-claude-code-toolkit) | [everything-claude-code](https://github.com/affaan-m/everything-claude-code) | [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) | [Claude Plugins Directory](https://claude-plugins.dev/)

### Tutorials
- [alexop.dev: Full Stack Guide](https://alexop.dev/posts/understanding-claude-code-full-stack/) | [OneAway: Skills Complete Guide](https://oneaway.io/blog/claude-code-skills-slash-commands) | [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) | [ClaudeFast: Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) | [F22 Labs: 10 Productivity Workflows](https://www.f22labs.com/blogs/10-claude-code-productivity-tips-for-every-developer/)

---

*Updated March 7, 2026. Claude Code evolves rapidly — verify against the current docs and release notes before treating any workflow as settled.*
