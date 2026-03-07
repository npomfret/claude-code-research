# Claude Code: Practical Examples

Copy-paste-ready examples of the most useful skills, hooks, agent patterns, CLAUDE.md configs, and CLI workflows. Companion to `claude-code-guide.md`.

**Ground rules for this file:** Every example here either comes from official docs, a verifiable open-source repo, or a pattern independently confirmed by multiple sources. No inflated stats. No hallucinated star counts.

---

## Agent Skills

Agent Skills are not the same thing as custom slash commands.

- **Skills** live in `.claude/skills/<skill-name>/SKILL.md` and are invoked by Claude when the request matches the description
- **Slash commands** live in `.claude/commands/*.md` and are invoked explicitly by typing `/command-name`
- **Plugins** can bundle both

If you want Claude Code to "know how we do code review here," build a skill. If you want a deterministic shortcut like `/review-pr`, build a slash command.

### `code-review` Skill with Supporting Files

Good example of the new model: one capability, one folder, real supporting artifacts.

`.claude/skills/code-review/SKILL.md`:
```markdown
---
name: code-review
description: Review diffs and pull requests for correctness, security, performance, and missing tests. Use when asked to review code, inspect a PR, or audit recent changes.
---

# Code Review Skill

Use the checklist in `checklist.md`.
Use `scripts/collect-diff.sh` to gather the exact diff context before reviewing.

Process:
1. Collect the diff and changed files
2. Review against the checklist
3. Report findings by severity: Critical, Warning, Suggestion
4. Prefer concrete fixes over generic advice

## Version History
- v1.2.0: Added explicit test-gap and migration-risk checks
```

`.claude/skills/code-review/checklist.md`:
```markdown
# Review Checklist

## Correctness
- Does the implementation satisfy the intended behavior?
- Are edge cases handled?

## Security
- Are inputs validated?
- Are secrets or tokens exposed?

## Performance
- Any N+1 queries, repeated parsing, or avoidable allocations?

## Testing
- Are new branches covered?
- Are failure modes tested?
```

`.claude/skills/code-review/scripts/collect-diff.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

git diff --name-only HEAD~1
printf '\n---\n'
git diff HEAD~1
```

This is where skills shine: the workflow is no longer trapped in one prompt file.

---

### `debug-workflow` Skill with Sub-Agent Delegation

Use one coarse skill instead of five narrow ones.

`.claude/skills/debug-workflow/SKILL.md`:
```markdown
---
name: debug-workflow
description: Investigate and fix runtime bugs, test failures, build errors, and regressions. Use when something is broken or behaving unexpectedly.
---

# Debug Workflow

Process:
1. Reproduce the issue
2. Trace the failing code path
3. Delegate search-heavy work to subagents
4. Form a root-cause hypothesis
5. Implement the minimal fix
6. Verify with focused tests

When delegating:
- Use one subagent to inspect recent git changes
- Use one subagent to search for related failures across the codebase

Output:
- Root cause
- Fix
- Regression risk
- Prevention
```

The important part is the `description`, not clever prose in the body. Discovery quality rises or falls with the description.

---

### Pair a Skill with a Thin Slash Command

This is the cleanest hybrid pattern.

`.claude/commands/review-pr.md`:
```markdown
---
description: Review the current branch or a named PR using the project's review workflow
argument-hint: [pr number or diff target]
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(gh pr view:*)
disable-model-invocation: true
---

Review $ARGUMENTS using the `code-review` skill if it is available.
If no argument is provided, review the current branch diff against `HEAD~1`.
```

Use this when you want both:

- A discoverable reusable skill
- A deterministic entry point for humans

`disable-model-invocation: true` is important here. It stops Claude from treating the slash command itself as an automatic tool choice.

---

### Structured Slash Command for Commits

Slash commands still matter for highly explicit workflows.

`.claude/commands/commit.md`:
```markdown
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
argument-hint: [message]
description: Commit staged changes with sanity checks
model: haiku
disable-model-invocation: true
---

Before committing, check changed files for:
- TODO comments
- console.log or print statements
- Commented-out code blocks
- Test flags left enabled such as `.only` or `.skip`

If clean, commit with message: $ARGUMENTS
If issues are found, stop and list them.
```

This is a better fit as a slash command than a skill because the user is explicitly asking to commit right now.

---

### Skill Layout for Runbooks and Templates

One of the most useful new patterns is putting reference material next to the skill.

```text
.claude/
  skills/
    release-checks/
      SKILL.md
      runbook.md
      rollback.md
      templates/
        release-summary.md
      scripts/
        verify-release.sh
```

That structure is much easier to maintain than an enormous `CLAUDE.md`.

---

### Built-In and Plugin-Provided Skills

Claude Code now has both bundled skills and plugin-delivered skills. Official docs call out bundled skills such as:

- `/batch`
- `/debug`
- `/loop`
- `/simplify`
- `/claude-api`

Treat these as built-in accelerators, not replacements for repo-specific skills. The moment your workflow depends on local conventions, add your own project skill or slash command.

---

### Skill Collections

| Collection | Notes |
|---|---|
| [Anthropic Official Skills](https://github.com/anthropics/skills) | Official reference repo for skill structure, resources, and packaging patterns |
| [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | Large command/agent collection; useful for pattern mining, but curate aggressively |
| [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | Broad community index of commands, hooks, agents, plugins, and templates |
| [wshobson/commands](https://github.com/wshobson/commands) | Good source of explicit slash-command patterns |
| [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | Curated skill collection |
| [claude-code-skill-factory](https://github.com/alirezarezvani/claude-code-skill-factory) | Useful if you want a repeatable internal skill-authoring workflow |

---

## Hooks

Hooks are the only mechanism that **guarantees** behavior. CLAUDE.md is advisory — Claude can ignore it. Hooks are mechanical — they execute regardless of what Claude "thinks."

Configure in `.claude/settings.json` (project-level) or `~/.claude/settings.json` (global).

### 1. Notification on Completion (Stop)

The easiest win. You stop checking the terminal every 30 seconds.

**macOS:**
```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code is done\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

**Linux:**
```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'Claude is done'"
          }
        ]
      }
    ]
  }
}
```

---

### 2. Auto-Format After Edits (PostToolUse)

Eliminates style inconsistencies. Claude is bad at whitespace — let your formatter handle it.

**JavaScript/TypeScript (Prettier):**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Python (Ruff):**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path | select(endswith(\".py\"))' | xargs ruff check --fix && jq -r '.tool_input.file_path | select(endswith(\".py\"))' | xargs ruff format",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Trade-off:** Per-edit formatting floods the context with "file changed" system reminders. Some teams prefer formatting only at commit time (via a pre-commit hook) to keep context cleaner. Pick one approach and be consistent.

---

### 3. Block Dangerous Commands (PreToolUse)

Prevents `rm -rf`, `sudo`, force-push, database drops.

`.claude/hooks/block-dangerous.sh`:
```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -qE '(rm\s+-rf|drop\s+table|git\s+reset\s+--hard|git\s+push\s+--force|sudo|chmod\s+777)'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked. Explain your intent and use a safer alternative."
    }
  }' >&1
  exit 0
fi

exit 0
```

Hook config:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/block-dangerous.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**Important:** The correct deny pattern is `exit 0` + JSON with `permissionDecision: "deny"`. Some older guides suggest `exit 2` — that signals a hook error, not a denial. Claude receives the `permissionDecisionReason` and self-corrects to a safer approach.

---

### 4. Protect Sensitive Files (PreToolUse)

Prevents reading or editing `.env`, secrets, credentials.

`.claude/hooks/protect-files.sh`:
```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED=(".env" ".env.local" "secrets" "credentials" "private_key" "api_key" ".ssh/")

for pattern in "${PROTECTED[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    jq -n '{
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: "Protected file: '"$FILE_PATH"'. Cannot access files matching sensitive patterns."
      }
    }' >&1
    exit 0
  fi
done

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Read",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/protect-files.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

---

### 5. Inject Git Context on Session Start (SessionStart)

When you start or resume a session, inject current project state so Claude knows where things stand.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Current branch:' && git branch --show-current && echo '\\nRecent commits:' && git log --oneline -5 && echo '\\nModified files:' && git status --short"
          }
        ]
      }
    ]
  }
}
```

Useful for picking up where you left off. Can also include project-specific reminders:

```json
{
  "command": "echo 'REMINDERS:\\n- Use Bun, not npm\\n- Run tests before every commit\\n- Current focus: auth module refactor' && git log --oneline -3 && git status --short"
}
```

---

### 6. Context Enrichment on Every Prompt (UserPromptSubmit)

SessionStart context gets pushed back and "forgotten" in long sessions. UserPromptSubmit fires on every prompt submission — whatever your script writes to stdout gets injected alongside the user's prompt as additional context.

**Basic: Inject project routing hints**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'PROJECT CONTEXT:\\n- TypeScript monorepo (pnpm)\\n- Use /project:review for code reviews\\n- Use /project:test for writing tests\\n- Use /project:debug for bug investigation\\n- Always run pnpm test before committing'"
          }
        ]
      }
    ]
  }
}
```

This is soft routing — it reminds Claude which skills exist and when to use them. It's not deterministic (Claude can still ignore it), but it's re-injected every prompt so it survives long sessions where CLAUDE.md context fades.

**Advanced: Auto-refresh context every N prompts**

Based on [John Lindquist's pattern](https://gist.github.com/johnlindquist/23fac87f6bc589ddf354582837ec4ecc) — tracks prompt count and re-injects heavier context periodically:

`.claude/hooks/refresh-context.sh`:
```bash
#!/bin/bash
COUNT_FILE="/tmp/claude-prompt-count"
COUNT=$(cat "$COUNT_FILE" 2>/dev/null || echo 0)
COUNT=$((COUNT + 1))
echo "$COUNT" > "$COUNT_FILE"

# Always inject lightweight routing
echo "ROUTING: /project:review (code review) | /project:debug (bugs) | /project:test (tests)"

# Every 10 prompts, re-inject heavier project context
if [ $((COUNT % 10)) -eq 0 ]; then
  echo ""
  echo "PROJECT STATE:"
  git branch --show-current
  git status --short
  echo ""
  echo "RECENT CHANGES:"
  git log --oneline -3
fi
```

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/refresh-context.sh"
          }
        ]
      }
    ]
  }
}
```

**Why this matters:** In long sessions, Claude stops using tools and skills that were introduced at the start because the context has been pushed back or compacted. This pattern keeps routing and project state fresh.

---

### Advanced: Test Gate Before Commit

Run tests before any commit. If they fail, Claude must fix them before proceeding.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); CMD=$(echo \"$INPUT\" | jq -r \".tool_input.command // empty\"); if echo \"$CMD\" | grep -q \"git commit\"; then npm test 2>&1 || { echo \"Tests failed. Fix before committing.\" >&2; exit 2; }; fi'"
          }
        ]
      }
    ]
  }
}
```

Creates a tight feedback loop: Claude writes code → tries to commit → tests fail → Claude reads failure → fixes code → tries again. No human intervention needed.

---

### Hook Resources

- [Official Hooks Docs](https://docs.claude.com/en/docs/claude-code/hooks)
- [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) — Community hook collection
- [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) — Step-by-step walkthrough

---

## Sub-Agent Patterns

### Built-In Agents

Know these before building custom ones — they cover 80% of delegation needs.

| Agent | Model | Tools | Use For |
|---|---|---|---|
| **Explore** | Haiku (fast/cheap) | Read, Glob, Grep only | File discovery, codebase search, understanding structure |
| **Plan** | Inherits from parent | Read-only | Architecture analysis, research before implementation |
| **General-Purpose** | Inherits from parent | All tools | Complex multi-step work, implementation |

Build custom agents only when you need: restricted tools, domain-specific prompts, or hook-enforced constraints.

---

### Custom Agent Templates

#### Code Reviewer Agent

`.claude/agents/code-reviewer.md`:
```markdown
---
name: code-reviewer
description: Expert code review specialist. Use proactively after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a senior code reviewer.

When invoked:
1. Run `git diff` to see recent changes
2. Review each changed file against this checklist:
   - Correctness: Does it do what it claims?
   - Security: Injection, XSS, secrets in code, input validation
   - Performance: N+1 queries, unnecessary allocations
   - Error handling: Are failures handled? Are errors swallowed?
   - Naming: Would a new team member understand this?
3. Organize findings by priority: Critical > Warning > Suggestion
4. For each issue, show the problematic code and a concrete fix
```

The `description` field says "use proactively" — this tells Claude to automatically delegate reviews without being asked.

---

#### Debugger Agent

`.claude/agents/debugger.md`:
```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger.

Process:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify the fix works by running the failing test/command

For each issue provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- How to prevent similar bugs
```

---

#### Read-Only Database Analyst (Hook-Enforced)

The most interesting pattern: an agent with **mechanical restrictions**, not just advisory ones.

`.claude/agents/db-reader.md`:
```markdown
---
name: db-reader
description: Execute read-only database queries for analysis and reporting.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access.
Execute SELECT queries only. You cannot modify data.
```

`./scripts/validate-readonly-query.sh`:
```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)\b' > /dev/null; then
  echo "Blocked: Write operations not allowed." >&2
  exit 2
fi
exit 0
```

The hook makes the read-only constraint *mechanical*. The agent literally cannot run write queries, regardless of how it's prompted. This is the model for any agent that handles sensitive operations.

---

### Orchestration Patterns

#### Fan-Out/Fan-In (Parallel Research)

The most common pattern. Spawn multiple agents for independent research, synthesize results.

```
Research these areas in parallel using separate sub-agents:

1. Authentication module — How does login work? What are the security boundaries?
2. Database layer — What queries are most expensive? Are there missing indexes?
3. API layer — Which endpoints are most called? Where are the bottlenecks?

Each agent: summarize findings in 2-3 paragraphs max.
Then synthesize into a single architecture overview.
```

Golden rule: **parallel only works when agents touch different files.** If two agents read the same file, fine. If they edit the same file, you get conflicts.

---

#### Expert Panel (Multi-Perspective Review)

Three specialists review the same code from different angles:

```
Review src/checkout/ using 3 parallel agents:

1. Security expert — OWASP top 10, input validation, auth issues
2. Performance expert — query efficiency, caching opportunities, memory usage
3. Accessibility expert — ARIA labels, keyboard navigation, screen reader support

Each returns a prioritized list of findings with specific fixes.
```

Catches issues that a single generalist reviewer would miss.

---

#### The Handoff Document (Sequential Agent Chains)

Sub-agents can't talk to each other. File-based handoff is the only reliable coordination mechanism.

```markdown
# Handoff: Auth Module Refactor

## Completed
- Migrated session storage from cookies to JWT
- Updated 12 route handlers
- Tests passing: 47/52

## Failed Approaches (saves the next agent from repeating mistakes)
- Tried storing refresh tokens in localStorage — XSS risk, reverted
- Attempted shared Redis session store — connection pooling issues in test env

## Failing Tests
- test/auth/refresh.test.ts:42 — TokenExpiredError (token TTL too short in test config)
- test/auth/logout.test.ts:18 — Session not cleared from Redis

## Next Steps
1. Fix the 5 failing tests
2. Add rate limiting to /auth/refresh endpoint
3. Update API documentation
```

The "Failed Approaches" section is the highest-value part. It prevents the next agent from wasting tokens on dead ends.

Source: [Anthropic: Building a C Compiler](https://www.anthropic.com/engineering/building-c-compiler)

---

#### The Self-Organizing Swarm (Anthropic's C Compiler Pattern)

16 Claude agents built a 100,000-line C compiler by self-organizing:

- **No centralized orchestrator.** Each agent claims tasks by creating lock files in `current_tasks/`
- **Git as coordination.** Agents pull, work, push. Merge conflicts force different task selection.
- **Shared test suite as the feedback signal.** Agents knew if their work was correct immediately.
- **Emergent specialization.** One focused on deduplication, another on optimization, another on documentation — not pre-assigned, but natural.

Cost: ~$20,000 in API calls. Result: Fully functional compiler, 99% pass rate on GCC torture tests.

Key insight: You don't need complex orchestration. Give agents clear success criteria (tests), a simple coordination mechanism (lock files), and let them self-organize.

Source: [Anthropic Engineering Blog](https://www.anthropic.com/engineering/building-c-compiler) | [GitHub](https://github.com/anthropics/claudes-c-compiler)

---

### Agent Collections

| Collection | Description |
|---|---|
| [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | Community templates |
| [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | 54 pre-built agents |

---

## CLAUDE.md Examples

Target: **60-80 lines.** Above ~150 lines, Claude starts ignoring instructions ([evidence](https://www.humanlayer.dev/blog/writing-a-good-claude-md)). For each line, ask: "Would removing this cause Claude to make mistakes?"

### Routing Table Style (Recommended)

The CLAUDE.md as a thin routing layer — tells Claude what the project is, how to build it, and where to go for more. Everything else lives in skills, hooks, or `.claude/rules/`.

```markdown
# Acme Dashboard

Next.js 14 (App Router), TypeScript strict, Prisma ORM, Tailwind CSS.

## Commands
- `pnpm dev` — dev server on :3000
- `pnpm test` — Vitest
- `pnpm db:migrate` — Prisma migrations

## Structure
- `src/app/` — pages and layouts
- `src/components/` — one component per file
- `src/lib/` — shared utils, db client, auth
- `src/server/` — API routes, background jobs
- `prisma/` — schema and migrations

## Skills
- `/project:review` — code review checklist
- `/project:debug` — structured bug investigation
- `/project:test` — write tests for a module

## Rules
- Named exports only (no default exports)
- Server Components by default. 'use client' only when needed.
- All DB queries through `src/lib/db.ts`
- `zod` for input validation at API boundaries

## Never
- Never modify `prisma/migrations/` directly
- Never commit `.env`
- Never use `any` — use `unknown` and narrow
```

~30 lines. The "Skills" section acts as a routing table — Claude knows what's available without needing the full skill content in context. Detailed review checklists, test patterns, and debug workflows live in the skill files themselves.

Source: [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Arize: CLAUDE.md Optimization](https://arize.com/blog/claude-md-best-practices-learned-from-optimizing-claude-code-with-prompt-learning/)

---

### Pre-Simplification Example (What Most Teams Start With)

This is what a CLAUDE.md looks like before teams learn to simplify. It works, but the length means instructions compete for attention:

```markdown
# Project: Acme Dashboard

Next.js 14 (App Router), TypeScript strict, Prisma ORM, Tailwind CSS.

## Commands
- `npm run dev` — starts dev server on :3000
- `npm test` — runs Vitest
- `npm run db:migrate` — runs Prisma migrations
- `npm run lint` — ESLint + Prettier check

## Architecture
- `src/app/` — Next.js App Router pages and layouts
- `src/components/` — React components. One component per file.
- `src/lib/` — Shared utilities, database client, auth helpers
- `src/server/` — Server-only code. API route handlers, background jobs.
- `prisma/` — Database schema and migrations

## Conventions
- Named exports only (no default exports)
- Server Components by default. Add 'use client' only when needed.
- All database queries go through `src/lib/db.ts`, never import Prisma directly
- Error responses follow `{ error: string, code: string }` shape
- Use `zod` for all input validation at API boundaries

## Never
- Never modify `prisma/migrations/` — always use `npm run db:migrate`
- Never commit `.env` files
- Never use `any` type — use `unknown` and narrow
```

Still reasonable at ~30 lines, but notice the difference: no routing hints to skills, no progressive disclosure. As projects grow, this pattern balloons to 150+ lines and Claude starts dropping instructions.

Source: [Builder.io: CLAUDE.md Guide](https://www.builder.io/blog/claude-md-guide)

---

### Monorepo Example

```markdown
# Acme Platform (Monorepo)

## Structure
- `/apps/web` — Next.js frontend
- `/apps/api` — Express backend
- `/packages/ui` — Shared React components
- `/packages/config` — Shared TypeScript/ESLint/Prettier configs
- `/packages/types` — Shared TypeScript types

## Commands (run from root)
- `pnpm dev` — starts all apps
- `pnpm test` — runs all tests
- `pnpm -F web test` — run tests for web app only
- `pnpm -F api test` — run tests for api only

## Rules
- Changes to `/packages/types` affect both apps. Run full test suite.
- The API and web app share types via `@acme/types`. Never duplicate type definitions.
- Use `pnpm` exclusively. Do not use npm or yarn.
```

---

### Modular CLAUDE.md (Large Projects)

For projects that exceed 300 lines, use `.claude/rules/` for topic-specific context:

```
.claude/
  rules/
    code-style.md      — formatting, naming, patterns
    testing.md         — test framework, coverage requirements
    security.md        — auth patterns, input validation rules
    api-conventions.md — endpoint naming, response shapes, error codes
```

The main `CLAUDE.md` stays short and references these for deeper context.

Source: [Gend.co: Claude Skills and CLAUDE.md Guide](https://www.gend.co/blog/claude-skills-claude-md-guide) | [Dometrain: Creating the Perfect CLAUDE.md](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/)

---

## CLI Patterns

### Headless Mode (CI/CD)

#### One-Shot Review
```bash
claude -p "Review this PR for security issues, performance problems, and test coverage gaps." \
  --output-format json \
  --max-budget-usd 2.00
```

#### Pipe Logs for Analysis
```bash
tail -f app.log | claude -p "Alert me if you see anomalies, errors, or unusual patterns"
```

#### Batch Review Changed Files
```bash
git diff main --name-only | claude -p "Review these changed files for security issues"
```

#### Cost-Controlled Automation
```bash
claude -p "Fix all linting errors in src/" \
  --max-budget-usd 5.00 \
  --max-turns 3 \
  --allowedTools "Bash(npm run lint:fix),Edit"
```

The `-p` flag turns Claude into a unix pipe. Chain it into CI/CD, pre-commit hooks, or reporting scripts. Add `--output-format json` when the output needs to be parsed by another tool.

---

### GitHub Actions Integration

Official action: `anthropics/claude-code-action@v1`

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Source: [github.com/anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)

---

### Session Management

```bash
claude -c                  # Resume most recent session
claude -r "auth refactor"  # Search past sessions by keyword
claude --from-pr 142       # Resume the session that created PR #142
claude --fork-session       # Branch off without mutating original
```

---

### Permission Patterns

```bash
# Plan mode: Claude analyzes but never executes
claude --permission-mode plan

# Selective tool access
claude --allowedTools "Bash(git:*) Read Grep Glob"

# Block destructive commands
claude --disallowedTools "Bash(rm:*) Bash(sudo:*)"
```

**Shift+Tab** in interactive mode cycles: Normal → Auto-Accept → Plan → Normal. Fastest way to switch modes mid-session.

---

### Structured Output

```bash
# JSON output for parsing
claude -p "List all TODO comments in src/" --output-format json

# Stream for real-time processing
claude -p "Explain this codebase" --output-format stream-json --verbose | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

---

### Context Management

**`/compact`** is garbage collection for Claude's brain. Its "IQ" drops as context fills. Run `/compact` (or `/compact focus on X`) every time you switch tasks within a session.

---

## MCP Servers

Model Context Protocol servers extend Claude Code with external tool access. Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server@latest"]
    }
  }
}
```

| Server | What It Does |
|---|---|
| **GitHub MCP** | PRs, issues, workflows, commits — the most broadly useful |
| **Context7** | Live, version-accurate library docs — prevents stale training data hallucinations |
| **Supabase** | Database operations via natural language |
| **Figma** | Design component extraction for design-to-code workflows |
| **Perplexity** | Real-time web search for research and fact-checking |

**Warning:** MCP servers bloat context with tool definitions. Enable Tool Search for lazy loading. Not all servers are production-ready — if one crashes, Claude loses access mid-session.

Source: [MCPcat: Best MCP Servers for Claude Code](https://mcpcat.io/guides/best-mcp-servers-for-claude-code/) | [Claude Code Docs: MCP](https://docs.claude.com/en/docs/claude-code/mcp)

---

## Getting Started Progression

### Day 1 (Starter)
1. Run `/init` to create CLAUDE.md — keep it under 80 lines (routing table + non-negotiables)
2. Add the Stop hook for notifications
3. Add the block-dangerous-commands PreToolUse hook

### Week 2 (Intermediate)
4. Add 1-2 explicit slash commands in `.claude/commands/` for workflows you intentionally invoke
5. Build 2-3 coarse skills in `.claude/skills/` with explicit `description` fields and supporting files
6. Add auto-format hook (PostToolUse) — pick: per-edit OR per-commit, not both
7. Learn `/compact` and Shift+Tab mode cycling
8. Add a UserPromptSubmit hook for basic routing context

### Month 2 (Advanced)
9. Build custom agents as specialists under your coarse skills
10. Configure MCP servers for external integrations
11. Package reusable workflows as plugins if multiple repos need them
12. Set up CI/CD with headless mode (`-p` flag)
13. Add a context-refresher UserPromptSubmit hook for long sessions

### What Not to Do
- Don't bloat CLAUDE.md past 80 lines — prune, don't add. Move detail to skills and `.claude/rules/`
- Don't confuse skills with slash commands — use both, but for different reasons
- Don't build 10 narrow skills — build 3-5 coarse skills and let them delegate to sub-agents
- Don't stack 5+ synchronous hooks (180-second delays reported)
- Don't build custom agents for tasks the built-ins handle fine
- Don't put code style rules in CLAUDE.md that a linter can enforce via hooks
- Don't format on every edit AND care about context preservation — pick one

---

## Sources

### Official
- [Claude Code Overview](https://docs.claude.com/en/docs/claude-code/overview) | [Slash Commands](https://docs.claude.com/en/docs/claude-code/slash-commands) | [Hooks](https://docs.claude.com/en/docs/claude-code/hooks) | [Subagents](https://docs.claude.com/en/docs/claude-code/sub-agents) | [CLI Reference](https://docs.claude.com/en/docs/claude-code/cli-reference) | [MCP](https://docs.claude.com/en/docs/claude-code/mcp) | [GitHub Actions](https://docs.claude.com/en/docs/claude-code/github-actions)
- [Agent Skills Overview](https://docs.claude.com/en/docs/agents-and-tools/claude-for-desktop-agent-sdk/agent-skills/overview) | [Using Skills in Claude](https://docs.claude.com/en/docs/agents-and-tools/claude-for-desktop-agent-sdk/agent-skills/use-skills) | [How to Create Custom Skills](https://docs.claude.com/en/docs/agents-and-tools/claude-for-desktop-agent-sdk/agent-skills/custom-skills) | [Manage Skills in Claude Code](https://docs.claude.com/en/docs/claude-code/tutorials/manage-skills)
- [Plugins Overview](https://docs.claude.com/en/docs/claude-code/plugins/overview) | [Create Plugins](https://docs.claude.com/en/docs/claude-code/plugins/create-plugins) | [Plugin Marketplaces](https://docs.claude.com/en/docs/claude-code/plugins/marketplaces) | [Code Intelligence Plugins](https://docs.claude.com/en/docs/claude-code/plugins/code-intelligence-plugins)
- [Release Notes](https://docs.claude.com/en/release-notes/overview)
- [Anthropic: Building a C Compiler](https://www.anthropic.com/engineering/building-c-compiler)

### Community Collections
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)
- [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | [wshobson/commands](https://github.com/wshobson/commands) | [Anthropic Official Skills](https://github.com/anthropics/skills)
- [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) | [claude-code-skill-factory](https://github.com/alirezarezvani/claude-code-skill-factory)

### Guides
- [Builder.io: CLAUDE.md Guide](https://www.builder.io/blog/claude-md-guide) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Dometrain: Perfect CLAUDE.md](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/) | [Gend.co: Skills & CLAUDE.md Guide](https://www.gend.co/blog/claude-skills-claude-md-guide)
- [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) | [MCPcat: Best MCP Servers](https://mcpcat.io/guides/best-mcp-servers-for-claude-code/)

### Architecture & Optimization
- [Anthropic: Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | [Arize: CLAUDE.md Optimization via Prompt Learning](https://arize.com/blog/claude-md-best-practices-learned-from-optimizing-claude-code-with-prompt-learning/) | [alexop.dev: Progressive Disclosure](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/) | [paddo.dev: Skills Controllability Problem](https://paddo.dev/blog/claude-skills-controllability-problem/) | [paddo.dev: Hooks as Guardrails](https://paddo.dev/blog/claude-code-hooks-guardrails/) | [paddo.dev: How Boris Uses Claude Code](https://paddo.dev/blog/how-boris-uses-claude-code/) | [John Lindquist: Auto-Refresh Context](https://gist.github.com/johnlindquist/23fac87f6bc589ddf354582837ec4ecc)

### GitHub Actions & CI/CD
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) | [Skywork: CI/CD Guide](https://skywork.ai/blog/how-to-integrate-claude-code-ci-cd-guide-2025/)

---

*Updated March 7, 2026. Verify examples against the current docs and release notes before treating any workflow as settled.*
