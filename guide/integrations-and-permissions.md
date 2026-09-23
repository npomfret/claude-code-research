# Integrations, Hooks, and Permissions

### MCP is for missing access or structured evidence, not for thinking

Anthropic's [MCP](https://code.claude.com/docs/en/mcp) docs frame MCP correctly: it gives Claude access to tools and systems. That includes external systems and local analysis engines such as a code graph. The common misuse is letting MCP stand in for analysis that should have happened in the code first.

High-value MCP categories in a coding workflow:

- version-accurate docs,
- locally indexed code relationships when ordinary search cannot reliably enumerate them,
- database inspection,
- GitHub and CI/CD state,
- issue trackers,
- internal service APIs,
- and carefully chosen browser/runtime tools when source inspection is not enough.

Low-value or overused cases:

- opening a browser before reading the component code,
- querying external systems before confirming the repo cannot answer the question,
- attaching heavyweight tools to every session whether needed or not.

### The code-first rule

Encode this directly:

1. read the code,
2. read the tests,
3. read the config,
4. inspect local logs or outputs,
5. use a local code-intelligence MCP when the question requires graph-wide relationships;
6. use an external MCP when the answer depends on remote truth or runtime state you cannot infer locally.

This saves context, time, and confusion.

### When an MCP is justified

Use an MCP when at least one of these is true:

- the source of truth is outside the repo,
- a local analysis engine can answer a structural question more completely than manual search,
- you need live system state,
- you need version-accurate external documentation,
- or you need to perform a remote action that cannot be simulated locally.

If none of those are true, stay with the compiler, language server, tests, and repository search.

### Scope MCP availability

Do not make every MCP globally available all the time just because it exists. If the environment allows it, keep the default MCP surface small and add specialized MCPs only for sessions that need them.

A practical baseline:

- always-on: only the few MCPs that provide frequently needed code intelligence or external truth,
- task-specific: database, browser, deployment, or rare internal systems.

The cost is not just latency. It is also conceptual distraction. Claude will use tools that exist.

## Hooks and Permissions

### Hooks are for logging, side effects, and auditability

Claude Code hooks support many actions, but their useful role in this setup is deliberately narrow.

For long-running interactive projects, the right use of hooks is:

- logging what Claude did,
- attaching lightweight reminders,
- running targeted post-edit checks,
- triggering notifications,
- updating state for audit or workflow systems,
- and recording useful metadata.

Avoid hooks that classify natural-language intent. Inferring whether a request sounds like a bug fix or which workflow applies is brittle. Route semantically through precise skill and agent descriptions, path-scoped rules, and local placement. Correct failures with better metadata, narrower scope, and clearer ownership. Reserve hooks for deterministic events and side effects.

### Hooks are not the right place to block normal development actions

Some guides recommend blocking `rm`, blocking certain writes, or turning hooks into a safety cage. Reject that.

Files legitimately need to be deleted. Directories legitimately need to be replaced. Blocking common actions at the hook layer creates three bad outcomes:

1. Claude fights the environment instead of solving the task.
2. Humans start working around the hook system.
3. The real problem, poor instructions and poor task governance, remains unsolved.

Use task instructions, approval policy, sandboxing, and operating rules when file deletion needs control; do not rely on a generic blocking hook.

Blocking hooks are brittle, easy to circumvent, and often constrain command syntax rather than engineering intent. They impede legitimate work without addressing weak instructions or governance.

If the goal is to allow or deny classes of behavior, `settings.json` is the correct place to express that policy. Hooks are the wrong tool for access control. Use hooks for deterministic side effects and auditability; use settings and sandbox policy for permissions.

### High-value hook examples

Good hooks for this setup:

- `SessionStart`: print a short reminder to use convention skills and the feature workflow for non-trivial changes.
- `PostToolUse`: append command and file-change logs to an audit file.
- post-edit side effect: run a targeted formatter or linter on touched files.
- `Stop`: write a short task summary or emit a notification.
- multi-agent events: record task completion or teammate idle state when using multi-agent workflows.

Keep them fast, deterministic, and visible.

### Permissions reality

Claude Code provides `allow`, `ask`, and `deny` rules; seven documented modes (`default`/`manual`, `acceptEdits`, `plan`, `auto`, `dontAsk`, and `bypassPermissions`); sandbox controls; and a `PermissionRequest` hook. Rules use `Tool` or `Tool(specifier)` syntax and are evaluated deny first, then ask, then allow; the first match wins. Write policies for the intended action, test them with `/status` and a safe representative command, and do not treat a broad Bash rule as an airtight security boundary.

Recommended policy:

- put team-shared, deterministic `deny` rules for secrets and high-risk commands in `.claude/settings.json`;
- put personal convenience allows in `.claude/settings.local.json` or `~/.claude/settings.json`, not in a committed repository file;
- use sandbox filesystem, network, and credential restrictions when isolation matters, because they apply at the subprocess boundary;
- use a `PreToolUse` or `PermissionRequest` hook only for narrow, deterministic policy or workflow handling; and
- use `CLAUDE.md`, skills, tests, and review for behavior that cannot be expressed as a deterministic access rule.

Auto mode can allow routine work while escalating risky actions in personal or managed environments. A repository cannot opt a user into it: project and local settings ignore `auto` and its prose-based `autoMode` policy. Keep hard prohibitions in deny rules or sandbox policy; the classifier is a convenience layer, not an authorization model.

`bypassPermissions` remains deliberately dangerous. It is appropriate only for an isolated, disposable environment where the user has consciously accepted unrestricted execution; organizations can disable it with `disableBypassPermissionsMode`. The example in `examples/settings.json` is therefore an opt-in personal configuration, not a recommended checked-in default.
