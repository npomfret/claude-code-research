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

Anthropic's [Hooks](https://code.claude.com/docs/en/hooks) documentation makes it clear that hooks can do a lot, and the changelog shows the hook surface is expanding. The official capability is broader than the recommendation in this guide.

For long-running interactive projects, the right use of hooks is:

- logging what Claude did,
- attaching lightweight reminders,
- running targeted post-edit checks,
- triggering notifications,
- updating state for audit or workflow systems,
- and recording useful metadata.

That is where hooks shine.

Natural-language-detecting hooks are usually a bad idea. If a hook has to infer user intent from vague task wording, classify whether a request "sounds like" a bug fix, or guess which workflow should apply from free text, it will be brittle. Use skills, agent descriptions, and root routing rules for semantic routing. Claude is already the language model in the loop, so it is usually better to rely on its own routing intelligence than to bolt on a second, cruder natural-language classifier in hooks. If semantic routing works almost all of the time and failures are corrected by better descriptions, narrower scope, and clearer routing hints, that is good enough. Use hooks for deterministic events and deterministic side effects.

### Hooks are not the right place to block normal development actions

Some guides recommend blocking `rm`, blocking certain writes, or turning hooks into a safety cage. Reject that.

Files legitimately need to be deleted. Directories legitimately need to be replaced. Blocking common actions at the hook layer creates three bad outcomes:

1. Claude fights the environment instead of solving the task.
2. Humans start working around the hook system.
3. The real problem, poor instructions and poor task governance, remains unsolved.

If you do not trust Claude to delete a file safely, the answer is not a generic blocking hook. The answer is better task instructions, better approval policy, better sandboxing, and stricter operating rules.

This is worth stating more bluntly: blocking hooks are often overused because they feel like control, but they are usually a brittle form of pseudo-governance. They are easy to circumvent, easy to get wrong, and often too prescriptive about the exact command shape instead of the real engineering intent. They create friction for legitimate work while doing little to address the deeper problem.

If the goal is to allow or deny classes of behavior, `settings.json` is the correct place to express that policy. Hooks are the wrong tool for access control. Use hooks for deterministic side effects and auditability; use settings and sandbox policy for permissions.

### High-value hook examples

Good hooks for this setup:

- `SessionStart`: print a short reminder to use convention skills and the feature workflow for non-trivial changes.
- `PostToolUse`: append command and file-change logs to an audit file.
- post-edit side effect: run a targeted formatter or linter on touched files.
- `Stop`: write a short task summary or emit a notification.
- multi-agent events from recent changelog additions: record task completion or teammate idle state if you use multi-agent workflows.

Keep them fast, deterministic, and visible.

### Permissions reality

Claude Code has a much more explicit permission surface than older versions: `allow`, `ask`, and `deny` rules, seven documented default modes (`default`/`manual`, `acceptEdits`, `plan`, `auto`, `dontAsk`, and `bypassPermissions`), sandbox controls, and a `PermissionRequest` hook. Rules use `Tool` or `Tool(specifier)` syntax and are evaluated deny first, then ask, then allow; the first matching rule wins. Write policies for the action you mean, test them with `/status` and a safe representative command, and do not assume a broad Bash rule is an airtight security boundary.

The practical recommendation is:

- put team-shared, deterministic `deny` rules for secrets and high-risk commands in `.claude/settings.json`;
- put personal convenience allows in `.claude/settings.local.json` or `~/.claude/settings.json`, not in a committed repository file;
- use sandbox filesystem, network, and credential restrictions when isolation matters, because they apply at the subprocess boundary;
- use a `PreToolUse` or `PermissionRequest` hook only for narrow, deterministic policy or workflow handling; and
- use `CLAUDE.md`, skills, tests, and review for behavior that cannot be expressed as a deterministic access rule.

Auto mode is now a useful middle ground for a personal or managed environment: the classifier can allow routine work while escalating risky actions. A repository cannot opt a user into it: `auto` and its prose-based `autoMode` policy are ignored in project and local settings. If you enable it, keep hard prohibitions in regular deny rules or sandbox policy and treat the classifier as a convenience layer, not an authorization model.

`bypassPermissions` remains deliberately dangerous. It is appropriate only for an isolated, disposable environment where the user has consciously accepted unrestricted execution; organizations can disable it with `disableBypassPermissionsMode`. The example in `examples/settings.json` is therefore an opt-in personal configuration, not a recommended checked-in default.

