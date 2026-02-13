# Claude Code: Practical Examples

Copy-paste-ready examples of the most useful skills, hooks, agent patterns, CLAUDE.md configs, and CLI workflows. Companion to `claude-code-guide.md`.

**Ground rules for this file:** Every example here either comes from official docs, a verifiable open-source repo, or a pattern independently confirmed by multiple sources. No inflated stats. No hallucinated star counts.

---

## Skills (Custom Slash Commands)

Skills live in `.claude/commands/` as markdown files. They become `/project:<name>` slash commands. The pattern: if you've typed the same instruction 3+ times, it should be a skill.

### /commit — Auto-Commit with Sanity Checks

The most common first skill teams build. Catches the embarrassing stuff before it hits the repo.

`.claude/commands/commit.md`:
```markdown
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
argument-hint: [message]
description: Commit with sanity checks
model: haiku
---

Before committing, check changed files for:
- TODO comments
- console.log or print statements
- Commented-out code blocks
- Test flags left enabled (e.g. .only, .skip)

If clean, commit with message: $ARGUMENTS
If issues found, list them and ask whether to proceed.
```

Uses Haiku (fast, cheap) because the task is simple pattern-matching.

---

### /review — Code Review Checklist

Encapsulates your team's review standards so nothing gets missed.

`.claude/commands/review.md`:
```markdown
---
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
description: Comprehensive PR review
---

## Changed Files
!`git diff --name-only HEAD~1`

## Detailed Changes
!`git diff HEAD~1`

## Review Checklist
For each changed file, evaluate:

1. **Correctness** — Does the logic do what it claims?
2. **Security** — SQL injection, XSS, secrets in code, input validation
3. **Performance** — N+1 queries, unnecessary allocations, missing indexes
4. **Error handling** — Are failures handled gracefully? Are errors swallowed silently?
5. **Test coverage** — Are new code paths tested? Are edge cases covered?
6. **Naming** — Are variables/functions descriptive? Would a new team member understand?

Provide specific, actionable feedback. Organize by priority: Critical > Warning > Suggestion.
For each issue, show the problematic code and a concrete fix.
```

The `` !` `` backtick syntax injects command output directly into the prompt — documented in the [official skills docs](https://code.claude.com/docs/en/skills).

---

### /investigate — Structured Bug Investigation

Forces systematic debugging instead of random poking.

`.claude/commands/investigate.md`:
```markdown
---
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(git diff:*)
argument-hint: <bug description>
description: Structured bug investigation
---

Investigate this bug: $ARGUMENTS

Follow this process:
1. **Reproduce** — Find the code path that triggers the bug
2. **Trace** — Follow the data flow from input to failure point
3. **Isolate** — Identify the smallest code change that would fix it
4. **Verify** — Check if the fix introduces regressions
5. **Root cause** — Explain *why* the bug exists (not just *what* is wrong)

Output format:
- **Summary:** One-sentence description
- **Root cause:** What went wrong and why
- **Fix:** Specific code change with diff
- **Risk:** What could break if we apply this fix
- **Prevention:** How to prevent similar bugs (test, lint rule, type change)
```

The value is in forcing Claude to diagnose before prescribing — without this structure, it tends to jump to a "fix" that addresses symptoms, not causes.

---

### /test — Generate Tests for a Module

`.claude/commands/test.md`:
```markdown
---
allowed-tools: Read, Write, Bash, Grep, Glob
argument-hint: <file or module path>
description: Write comprehensive tests
---

Write tests for: $ARGUMENTS

1. Read the file and understand its public API
2. Detect the test framework already in use (Jest, pytest, Vitest, etc.)
3. Follow existing test patterns in this project (check nearby test files)
4. Write tests covering:
   - Happy path for each public function
   - Edge cases (empty input, null, boundary values)
   - Error conditions (invalid input, network failures)
   - Integration points (does it work with its real dependencies?)

Do NOT write mock-heavy tests that test implementation details.
Test behavior, not implementation.
```

The instruction to detect the existing framework and follow project patterns is critical — without it, Claude often generates tests in the wrong style or framework.

---

### /explain — Onboarding Deep-Dive

`.claude/commands/explain.md`:
```markdown
---
allowed-tools: Read, Grep, Glob
argument-hint: <module or file path>
description: Deep explanation of a code area for onboarding
---

Explain this code area to someone joining the team: $ARGUMENTS

1. Start with an analogy comparing it to something from everyday life
2. Draw an ASCII diagram showing the flow or structure
3. Walk through the code step-by-step, explaining *why* each decision was made
4. Highlight common gotchas or misconceptions
5. List the most important files and their roles
6. Explain how this module connects to the rest of the system
```

---

### /commit-push-pr — One-Command Ship

`.claude/commands/commit-push-pr.md`:
```markdown
---
allowed-tools: Bash(git:*), Bash(gh:*)
argument-hint: [description of changes]
description: Commit, push, and open PR in one step
---

1. Run `git status` to see what's changed
2. Stage all relevant changes (not .env, not .DS_Store)
3. Generate a conventional commit message from the diff
4. Commit, push to current branch
5. Create a PR using `gh pr create` with:
   - Clear title summarizing the change
   - Body describing what changed and why
   - Link to any related issues

If anything looks wrong at any step, stop and ask.
$ARGUMENTS
```

The "stop and ask" instruction is the safety valve — without it, this command is a footgun.

---

### Coarse Skill Pattern: /debug with Sub-Agent Delegation

Instead of separate skills for debug-api, debug-frontend, debug-database — one broad skill that delegates:

`.claude/commands/debug.md`:
```markdown
---
allowed-tools: Read, Grep, Glob, Bash, Edit, Task
argument-hint: <error or bug description>
description: Debug any issue. Use when encountering errors, test failures, or unexpected behavior. Investigates, diagnoses, and fixes.
---

Investigate and fix: $ARGUMENTS

## Process
1. **Classify** — Is this a runtime error, test failure, build error, or logic bug?
2. **Reproduce** — Find the code path that triggers the issue
3. **Delegate** — Use sub-agents for parallel investigation:
   - Spawn an Explore agent to search for related error patterns across the codebase
   - Spawn an Explore agent to check recent git changes that may have introduced the bug
4. **Diagnose** — Synthesize findings into a root cause
5. **Fix** — Implement the minimal change that resolves it
6. **Verify** — Run the relevant tests to confirm the fix

## Output
- **Root cause:** What went wrong and why
- **Fix:** The specific code change
- **Prevention:** How to prevent this class of bug
```

The `description` field is keyword-rich ("errors, test failures, unexpected behavior") so Claude routes here naturally. The skill itself delegates to Explore sub-agents for the research-heavy parts, keeping the main context focused on the fix.

This one skill replaces 3-4 narrow debug skills while routing better — Claude has one clear match instead of choosing between overlapping descriptions.

---

### Skill Collections

| Collection | Notes |
|---|---|
| [Anthropic Official Skills](https://github.com/anthropics/skills) | PDF, DOCX, PPTX, XLSX processing, Playwright browser automation |
| [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | 148+ commands, 54 agents. Large, structured, covers review/test/deploy |
| [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | Community hub aggregating skills, hooks, agents, CLAUDE.md templates |
| [wshobson/commands](https://github.com/wshobson/commands) | Full-stack, security, data/ML, infrastructure collections |
| [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | Curated skill collection |
| [claude-code-skill-factory](https://github.com/alirezarezvani/claude-code-skill-factory) | Toolkit for building and deploying skills at scale |

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

- [Official Hooks Docs](https://code.claude.com/docs/en/hooks) | [Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
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
claude --disallowed-tools "Bash(rm:*) Bash(sudo:*)"
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

Source: [MCPcat: Best MCP Servers for Claude Code](https://mcpcat.io/guides/best-mcp-servers-for-claude-code/) | [Claude Code Docs: MCP](https://code.claude.com/docs/en/mcp)

---

## Getting Started Progression

### Day 1 (Starter)
1. Run `/init` to create CLAUDE.md — keep it under 80 lines (routing table + non-negotiables)
2. Add the Stop hook for notifications
3. Add the block-dangerous-commands PreToolUse hook

### Week 2 (Intermediate)
4. Build 2-3 coarse skills (/review, /debug, /test) with explicit `description` fields
5. Add auto-format hook (PostToolUse) — pick: per-edit OR per-commit, not both
6. Learn `/compact` and Shift+Tab mode cycling
7. Add a UserPromptSubmit hook for basic routing context

### Month 2 (Advanced)
8. Build custom agents as specialists under your coarse skills
9. Configure MCP servers for external integrations
10. Set up CI/CD with headless mode (`-p` flag)
11. Add a context-refresher UserPromptSubmit hook for long sessions

### What Not to Do
- Don't bloat CLAUDE.md past 80 lines — prune, don't add. Move detail to skills and `.claude/rules/`
- Don't build 10 narrow skills — build 3-5 coarse skills and let them delegate to sub-agents
- Don't stack 5+ synchronous hooks (180-second delays reported)
- Don't build custom agents for tasks the built-ins handle fine
- Don't put code style rules in CLAUDE.md that a linter can enforce via hooks
- Don't format on every edit AND care about context preservation — pick one

---

## Sources

### Official
- [Claude Code Docs](https://code.claude.com/docs/en/overview) | [Skills](https://code.claude.com/docs/en/skills) | [Hooks](https://code.claude.com/docs/en/hooks) | [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) | [Sub-Agents](https://code.claude.com/docs/en/sub-agents) | [CLI Reference](https://code.claude.com/docs/en/cli-reference) | [MCP](https://code.claude.com/docs/en/mcp) | [GitHub Actions](https://code.claude.com/docs/en/github-actions)
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

*Compiled February 2026. Verify examples against [current docs](https://code.claude.com/docs/en/overview) — Claude Code evolves rapidly.*
