# Claude Code: The Best Examples

Copy-paste-ready examples of the most popular skills, hooks, agent patterns, CLAUDE.md configurations, and CLI workflows used by real teams. Companion to `claude-code-guide.md`.

---

## Skills (Custom Slash Commands)

### The Highest-ROI Skills

Teams report **35-50% productivity gains** from replacing repetitive prompts with skills. The pattern: if you've typed the same kind of instruction 3+ times, it should be a skill.

#### /commit — Auto-Commit with Sanity Checks

The most widely adopted custom command. Catches common mistakes before they hit the repo.

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

**Why it's popular:** Catches the embarrassing stuff. Uses Haiku (fast, cheap) because the task is simple. Teams report it prevents ~30% of "oops" commits.

**Source:** [10 Claude Code Commands That Cut My Dev Time 60%](https://alirezarezvani.medium.com/10-claude-code-commands-that-cut-my-dev-time-60-a-practical-guide-60036faed17f)

---

#### /review — Code Review Checklist

The single most popular skill across the community. Encapsulates your team's review standards.

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

**Why it's popular:** Eliminates "I forgot to check for X" during reviews. Teams report **58% reduction in code review time**. The `!` backtick syntax injects command output directly into the prompt.

**Source:** [Alireza Rezvani](https://alirezarezvani.medium.com/10-claude-code-commands-that-cut-my-dev-time-60-a-practical-guide-60036faed17f) | [Builder.io](https://www.builder.io/blog/claude-code)

---

#### /investigate — Structured Bug Investigation

Turns chaotic debugging into a systematic process.

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

**Why it's popular:** Forces systematic debugging instead of random poking. New team members debug like seniors because the process is encoded.

---

#### /test — Generate Tests for a Module

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

**Why it's popular:** Test generation is the task where AI assistants show the clearest ROI. The instruction to detect the existing framework and follow project patterns prevents the common failure of generating tests in the wrong style.

---

#### /explain — Onboarding Deep-Dive

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

**Why it's popular:** New hires get consistent, thorough onboarding. The analogy-first approach produces explanations that actually stick.

---

#### /commit-push-pr — The One-Command Ship

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

**Why it's popular:** Collapses 5+ manual steps into one command. The "stop and ask" safety valve prevents autopilot mistakes.

---

### Built-In Skills Worth Knowing

Anthropic ships official skills in [github.com/anthropics/skills](https://github.com/anthropics/skills):

| Skill | What It Does | When It Activates |
|---|---|---|
| **PDF** | Extract text/tables, merge, split, watermark, OCR, encrypt | User mentions PDF or references .pdf file |
| **DOCX** | Create/edit Word docs with tracked changes | User mentions report, memo, letter, template |
| **PPTX** | Create/edit slide decks | User mentions deck, slides, presentation |
| **XLSX** | Create/edit spreadsheets, format conversion | Spreadsheet is primary input or output |
| **Playwright** | Browser automation and E2E testing on-the-fly | User says "test the homepage", "verify signup flow" |

The Playwright skill ([lackeyjb/playwright-skill](https://github.com/lackeyjb/playwright-skill)) is the most popular QA skill — it auto-detects running dev servers and returns screenshots with console output.

---

### Skill Collections

| Collection | Size | Focus |
|---|---|---|
| [Anthropic Official Skills](https://github.com/anthropics/skills) | Core set | Document processing, built-in capabilities |
| [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | 148+ commands, 54 agents | Full-stack: review, testing, deployment, business |
| [wshobson/commands](https://github.com/wshobson/commands) | Multiple collections | Full-stack, security, data/ML, infrastructure |
| [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | Curated hub | Skills, hooks, agents, plugins |
| [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | Curated | Claude Code-specific skills |
| [claude-code-skill-factory](https://github.com/alirezarezvani/claude-code-skill-factory) | Toolkit | Build and deploy production skills at scale |
| [36 Skills from 23 Creators](https://aiblewmymind.substack.com/p/claude-skills-36-examples) | 36 examples | Thinking, writing, debugging, workflows |

---

## Hooks

### The 5 Hooks That Actually Matter

Most teams need only 2-3 hooks. These are the ones with proven ROI, in priority order.

#### 1. Notification on Completion (Stop)

The easiest win. Zero risk, immediate productivity gain. You stop checking the terminal every 30 seconds.

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

**Why #1:** Near-universal adoption among power users. Lets you work in other apps without context-switching anxiety.

---

#### 2. Auto-Format After Edits (PostToolUse)

The most popular hook overall. Eliminates style inconsistencies without friction.

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

**Python (Black or Ruff):**
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

**Community wisdom:** Format on commit, not on every edit. Aggressive per-edit formatting floods the context window with system reminders about file changes. If you do format per-edit, accept the context cost.

---

#### 3. Block Dangerous Commands (PreToolUse)

The most requested safety feature. Prevents `rm -rf`, `sudo`, force-push, and database drops.

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

Hook configuration:
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

**Why it works:** Claude receives the denial reason and self-corrects to a safer approach. Exit code 0 + JSON `deny` is the correct pattern (not exit code 2 as some guides suggest — see [#21988](https://github.com/anthropics/claude-code/issues/21988)).

---

#### 4. Protect Sensitive Files (PreToolUse)

Prevents reading or editing `.env`, secrets, credentials, and lock files.

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

**Why it matters:** Teams report zero credential leaks after enabling. Essential for compliance-sensitive environments.

---

#### 5. Context Re-Injection After Compaction (SessionStart)

When auto-compact fires, Claude loses project context. This re-injects the essentials.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'IMPORTANT REMINDERS:\\n- Use Bun, not npm\\n- Run tests before every commit\\n- All API endpoints need TypeScript types\\n- Current focus: auth module refactor'"
          }
        ]
      }
    ]
  }
}
```

**Dynamic version (injects recent git state):**
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Recent commits:' && git log --oneline -5 && echo '\\nModified files:' && git status --short"
          }
        ]
      }
    ]
  }
}
```

**Why it matters:** Compaction is the #2 community pain point (after usage limits). This doesn't solve it completely, but prevents the worst "amnesia" failures.

---

### Advanced Hook Patterns

#### The Gatekeeper — Tests Before Commit

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

**Why it's effective:** Creates a tight feedback loop. Claude writes code, tries to commit, tests fail, Claude reads the failure, fixes the code, tries again. No human intervention needed.

#### The Complete Quality Gate

Format + lint + test, applied at the right moments:

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
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "npm test 2>&1 | tail -20",
            "timeout": 120
          },
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude is done\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

---

### Hook Sources

- [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) — Community hook collection and patterns
- [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) — Step-by-step with examples
- [Hooks Reference](https://code.claude.com/docs/en/hooks) | [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) — Official docs
- [Production Hooks Report](https://alirezarezvani.medium.com/the-production-ready-claude-code-hooks-guide-7-hooks-that-actually-matter-823587f9fc61) — 6-month production experience

---

## Sub-Agent Patterns

### Built-In Agents

Claude Code ships with three agent types. Understand these before building custom ones.

| Agent | Model | Tools | Use For |
|---|---|---|---|
| **Explore** | Haiku (fast/cheap) | Read, Glob, Grep only | File discovery, codebase search, understanding structure |
| **Plan** | Inherits from main | Read-only | Research before planning, architecture analysis |
| **General-Purpose** | Inherits from main | All tools | Complex multi-step work, implementation |

**When to use built-in vs custom:** Built-ins cover 80% of delegation needs. Build custom agents only when you need specialized behavior, restricted tools, or domain-specific prompts.

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

**Key detail:** The `description` field says "use proactively" — this tells Claude to automatically delegate reviews without being asked.

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

#### Read-Only Database Analyst

An agent with **hook-enforced restrictions** — can only run SELECT queries.

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

**Why it's powerful:** The hook makes the read-only constraint *mechanical*, not advisory. The agent literally cannot run write queries.

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

**When to use:** Gathering information from independent sources. The golden rule: **parallel only works when agents touch different files.**

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

**When to use:** Code that touches multiple concerns. Catches issues that a single reviewer would miss.

---

#### The Handoff Document (Sequential Agent Chains)

When Agent A finishes and Agent B picks up, use a structured handoff:

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

**Why it matters:** Sub-agents can't talk to each other. File-based handoff is the only reliable coordination mechanism. The "Failed Approaches" section is the highest-value part — it prevents the next agent from wasting tokens on dead ends.

**Source:** [Anthropic: Building a C Compiler](https://www.anthropic.com/engineering/building-c-compiler)

---

#### The Self-Organizing Swarm (Anthropic's C Compiler Pattern)

16 Claude agents built a 100,000-line C compiler by self-organizing:

- **No centralized orchestrator.** Each agent claims tasks by creating lock files in `current_tasks/`
- **Git as coordination.** Agents pull, work, push. Merge conflicts force different task selection.
- **Emergent specialization.** One focused on deduplication, another on optimization, another on documentation — not pre-assigned, but natural.
- **Shared test suite as the feedback signal.** Agents knew if their work was correct immediately.

**Cost:** $20,000 in API calls. **Result:** Fully functional compiler, 99% pass rate on GCC torture tests.

**Key insight:** You don't need complex orchestration. Give agents clear success criteria (tests), a simple coordination mechanism (lock files), and let them self-organize.

**Source:** [Anthropic Engineering Blog](https://www.anthropic.com/engineering/building-c-compiler) | [GitHub](https://github.com/anthropics/claudes-c-compiler)

---

### Agent Collections

| Collection | Description |
|---|---|
| [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | 100+ community templates |
| [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | 54 pre-built agents |
| [Claude Code System Prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | All built-in prompts documented |

---

## CLAUDE.md Examples

### The Anatomy of a Good CLAUDE.md

Community consensus: **under 300 lines, opinionated, explains the "why."** For each line, ask: "Would removing this cause Claude to make mistakes?"

#### Minimal Effective Example (Next.js)

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

**Why this works:** Short. Every line prevents a specific mistake Claude would otherwise make. The "Never" section catches the most expensive errors.

**Source:** [Builder.io: CLAUDE.md Guide](https://www.builder.io/blog/claude-md-guide) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)

---

#### Monorepo Example

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

#### Modular CLAUDE.md (Large Projects)

For projects that exceed 300 lines, use `.claude/rules/` for topic-specific context:

```
.claude/
  rules/
    code-style.md     — formatting, naming, patterns
    testing.md        — test framework, coverage requirements
    security.md       — auth patterns, input validation rules
    api-conventions.md — endpoint naming, response shapes, error codes
```

The main `CLAUDE.md` stays short and references these for deeper context.

**Source:** [Gend.co: Claude Skills and CLAUDE.md Guide](https://www.gend.co/blog/claude-skills-claude-md-guide) | [Dometrain: Creating the Perfect CLAUDE.md](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/)

---

## CLI Patterns

### Headless Mode (CI/CD)

#### One-Shot Code Review in CI

```bash
claude -p "Review this PR for security issues, performance problems, and test coverage gaps. Be specific." \
  --output-format json \
  --max-budget-usd 2.00
```

#### GitHub Actions Integration

Use the official action: `anthropics/claude-code-action@v1`

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

Popular CI use cases:
- Auto-review every PR
- Monthly documentation sync
- Weekly code quality audits
- Test failure auto-fix

**Source:** [GitHub: anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) | [Skywork: CI/CD Integration Guide](https://skywork.ai/blog/how-to-integrate-claude-code-ci-cd-guide-2025/)

---

#### Pipe Logs for Analysis

```bash
tail -f app.log | claude -p "Alert me if you see anomalies, errors, or unusual patterns"
```

#### Batch Processing Changed Files

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

---

### Session Management

#### Continue Last Conversation
```bash
claude -c  # Resume most recent session
```

#### Resume Specific Session
```bash
claude -r "auth refactor"  # Search past sessions by keyword
```

#### Jump to PR Session
```bash
claude --from-pr 142  # Resume the session that created PR #142
```

#### Fork a Conversation
```bash
claude --fork-session  # Branch off without mutating original
```

---

### Permission Patterns

#### Plan Mode (Analysis Only)
```bash
claude --permission-mode plan  # Claude plans but never executes
```

#### Selective Tool Access
```bash
# Only allow git commands and file reading
claude --allowedTools "Bash(git:*) Read Grep Glob"

# Block destructive commands
claude --disallowed-tools "Bash(rm:*) Bash(sudo:*)"
```

#### Shift+Tab Cycling

In interactive mode, press **Shift+Tab** to cycle: Normal > Auto-Accept > Plan > Normal. The fastest way to switch modes mid-session.

---

### Structured Output for Scripting

```bash
# JSON output for parsing
claude -p "List all TODO comments in src/" --output-format json

# With schema validation
claude -p "Extract function signatures from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}}}'

# Stream for real-time processing
claude -p "Explain this codebase" --output-format stream-json --verbose | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

---

## MCP Servers

### The Most Useful Integrations

| Server | What It Does | Install |
|---|---|---|
| **GitHub MCP** | PRs, issues, workflows, commits | `npx -y @github/github-mcp-server@latest` |
| **Sequential Thinking** | Breaks down complex reasoning | Improves multi-step task quality |
| **Perplexity** | Real-time web search | Research and fact-checking |
| **Context7** | Live, version-accurate library docs | Prevents stale training data hallucinations |
| **Supabase** | Database operations via natural language | Direct DB interaction |
| **Figma** | Design component extraction | Design-to-code workflows |

### Configuration

Add to `~/.claude.json`:
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

**Critical tip:** MCP servers bloat context with tool definitions. Enable Tool Search for lazy loading (up to 95% context reduction). Not all servers are production-ready — if one crashes, Claude loses access mid-session.

**Source:** [MCPcat: Best MCP Servers for Claude Code](https://mcpcat.io/guides/best-mcp-servers-for-claude-code/) | [Claude Code Docs: MCP](https://code.claude.com/docs/en/mcp)

---

## Putting It All Together

### Starter Setup (Day 1)

1. Run `/init` to create CLAUDE.md
2. Add a Stop hook for notifications
3. Create one skill for your most common task

### Intermediate Setup (Week 2)

4. Add auto-format hook (PostToolUse)
5. Add block-dangerous-commands hook (PreToolUse)
6. Create 2-3 more skills (/review, /test, /investigate)

### Advanced Setup (Month 2)

7. Build custom agents for specialized roles
8. Configure MCP servers for external integrations
9. Set up CI/CD with headless mode
10. Add context re-injection hook for compaction

### What Not to Do

- Don't stack 5+ synchronous hooks (180-second delays reported)
- Don't build custom agents for tasks built-ins handle fine
- Don't create skills for one-off tasks
- Don't put generic programming advice in CLAUDE.md
- Don't format on every edit if context preservation matters more than style

---

## All Sources

### Official
- [Claude Code Docs](https://code.claude.com/docs/en/overview) | [Skills](https://code.claude.com/docs/en/skills) | [Hooks](https://code.claude.com/docs/en/hooks) | [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) | [Sub-Agents](https://code.claude.com/docs/en/sub-agents) | [CLI Reference](https://code.claude.com/docs/en/cli-reference) | [MCP](https://code.claude.com/docs/en/mcp) | [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Anthropic: Building a C Compiler](https://www.anthropic.com/engineering/building-c-compiler) | [Equipping Agents with Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | [Building Agents with Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)

### Community Collections
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | [Claude Command Suite](https://github.com/qdhenry/Claude-Command-Suite) | [wshobson/commands](https://github.com/wshobson/commands) | [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) | [claude-code-skill-factory](https://github.com/alirezarezvani/claude-code-skill-factory) | [Anthropic Official Skills](https://github.com/anthropics/skills)

### Developer Guides
- [Builder.io: CLAUDE.md Guide](https://www.builder.io/blog/claude-md-guide) | [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | [Dometrain: Perfect CLAUDE.md](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/) | [Gend.co: Skills & CLAUDE.md Guide](https://www.gend.co/blog/claude-skills-claude-md-guide) | [Sid Bharath: Complete Guide](https://www.siddharthbharath.com/claude-code-the-complete-guide/)
- [10 Commands That Cut Dev Time 60%](https://alirezarezvani.medium.com/10-claude-code-commands-that-cut-my-dev-time-60-a-practical-guide-60036faed17f) | [36 Skills from 23 Creators](https://aiblewmymind.substack.com/p/claude-skills-36-examples) | [Pulumi: Top 8 DevOps Skills](https://www.pulumi.com/blog/top-8-claude-skills-devops-2026/) | [Production Hooks Report](https://alirezarezvani.medium.com/the-production-ready-claude-code-hooks-guide-7-hooks-that-actually-matter-823587f9fc61)
- [sshh.io: Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) | [InfoQ: Claude Code Creator's Workflow](https://www.infoq.com/news/2026/01/claude-code-creator-workflow/) | [Faros AI: Measuring Claude Code ROI](https://www.faros.ai/blog/how-to-measure-claude-code-roi-developer-productivity-insights-with-faros-ai/)

### Tutorials
- [DataCamp: Hooks Tutorial](https://www.datacamp.com/tutorial/claude-code-hooks) | [Stackademic: 10 Sub-Agent Examples](https://blog.stackademic.com/10-claude-code-sub-agent-examples-with-templates-you-can-use-immediately-a1159437ee2a) | [MCPcat: Best MCP Servers](https://mcpcat.io/guides/best-mcp-servers-for-claude-code/)

### GitHub Actions & CI/CD
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) | [Skywork: CI/CD Guide](https://skywork.ai/blog/how-to-integrate-claude-code-ci-cd-guide-2025/) | [Angelo Lima: CI/CD & Headless Mode](https://angelo-lima.fr/en/claude-code-cicd-headless-en/)

---

*Compiled February 2026. Verify examples against [current docs](https://code.claude.com/docs/en/overview) — Claude Code evolves rapidly.*
