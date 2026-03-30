# Claude Code Guide: CLI and Operations

> Superseded by `claude-code-guide.md`, which is now the canonical guide. Keep this file only as source material or supporting notes.

Last verified: **March 30, 2026**

This section covers the operational surface of Claude Code: built-in commands, CLI usage, permissions, sandboxing, and the workflow patterns that consistently improve results.

## Built-In Commands That Matter

Built-in slash commands are still essential for interactive work.

### Session and context

- `/compact`: compress noisy context
- `/clear`: start fresh for a new task
- `/rewind`: branch from an earlier checkpoint
- `/resume`: return to an older session
- `/btw`: attach side context without derailing the main flow
- `/context`: inspect current context usage

### Setup and memory

- `/init`: generate an initial `CLAUDE.md`
- `/memory`: edit memory files

### Delegation and extensibility

- `/plan`: plan before acting
- `/agents`: manage custom subagents
- `/skills`: browse or manage skills
- `/hooks`: manage hooks
- `/mcp`: manage MCP servers
- `/plugin`: manage plugins
- `/reload-plugins`: reload local plugin changes during development

### Safety and visibility

- `/permissions`: review or adjust allowed actions
- `/sandbox`: inspect or configure sandboxing
- `/usage`: inspect usage
- `/cost`: inspect API spend
- `/release-notes`: open current release notes
- `/doctor`: troubleshoot installation or environment issues
- `/model`: switch models

One detail worth remembering: the official docs mark `/review` as deprecated in favor of the separate review feature or `claude --review`.

## CLI Details That Matter

The CLI is where many high-leverage workflows live.

### Core flags

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

### Multi-repo and environment flags

- `--worktree`: create or use git worktrees
- `--remote`: run in a remote environment
- `--remote-control`: connect to a running Claude Code instance
- `--plugin-dir`: load plugins from a local path during development
- `--strict-mcp-config`: fail fast on invalid MCP configuration
- `--agents`: point to extra agent definition directories

### Practical CLI patterns

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

Useful details:

- `--max-budget-usd` only applies in print mode
- `--permission-mode plan` is one of the cleanest ways to get a plan before edits
- remote and remote-control flows are powerful, but add operational complexity

## Permissions Reality

The documented permission model is cleaner than the day-to-day experience many users report, especially for Bash.

What the docs say:

- permission rules support tool-specific patterns such as `Bash(...)`
- Claude Code understands shell operators, so allowing one command should not automatically allow a chained second command
- allowlists, auto mode, and sandboxing are the main friction-control tools

What the public issue tracker suggests:

- saved permission rules have repeatedly failed to match later commands in some cases
- compound Bash commands are a common source of extra prompts and matching bugs
- Anthropic has shipped repeated fixes here, which is helpful, but also shows this surface has been unstable

The practical conclusion is:

- do not rely on narrow Bash allow patterns for compound commands
- prefer broad command-family rules over clever exact-match strings
- prefer native Claude tools over shell pipelines where possible
- use sandboxing to reduce approval churn
- move real policy enforcement into hooks

For teams dealing with repeated approvals across many repos, the most reliable pattern appears to be:

- stable rules in `~/.claude/settings.json`
- broad and boring allowlists
- sandboxing enabled
- hooks for the things that truly matter

## Best-Practice Workflow Patterns

Patterns most worth institutionalizing:

### Plan before heavy edits

Use `/plan` or `--permission-mode plan` when the task is architectural, risky, or broad.

### Keep acceptance criteria in the prompt

Ask for:

- the exact files or areas to change
- constraints
- tests to run
- what "done" means

Weak:

> add auth

Strong:

> Add password-reset support in the existing auth flow. Reuse the current email provider. Do not change session semantics. Update API handlers, UI states, and tests. Finish only when targeted tests pass.

### Attach ground truth

Anthropic's docs explicitly support attaching files, images, and URLs. Use that instead of relying on recollection.

### Use skills for repeated work, not memory

If the same workflow keeps reappearing, convert it into a skill instead of inflating `CLAUDE.md`.

### Use subagents for research and separation

If the task is broad or benefits from specialization, delegate. If it is tight and local, keep it in the main thread.

### Turn checks into hooks

If you keep reminding Claude to do the same thing, that is a signal to automate it.

### Reset before context rot wins

Anthropic explicitly recommends compacting, clearing, rewinding, resuming, and using checkpoints when appropriate.

## Common Failure Patterns

These mistakes degrade output quality quickly:

- overstuffed memory
- vague prompts
- soft rules for hard requirements
- using the wrong layer for the problem
- trying to keep one endless session alive forever

## Sources

- [Claude Code Commands](https://code.claude.com/docs/en/commands)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Code Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Issue #1271: repeated prompts for combined commands](https://github.com/anthropics/claude-code/issues/1271)
- [Issue #4787: broad Bash permissions behaving inconsistently](https://github.com/anthropics/claude-code/issues/4787)
- [Issue #4956: chained commands bypassing or mismatching permissions](https://github.com/anthropics/claude-code/issues/4956)
- [Issue #6850: existing saved rules still prompting again](https://github.com/anthropics/claude-code/issues/6850)
- [Issue #9875: "Don't ask again" overwriting saved rules](https://github.com/anthropics/claude-code/issues/9875)
