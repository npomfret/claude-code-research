# Integrations, Hooks, and Permissions

MCP provides access and structured evidence. Use it when it adds something source, tests, and local commands cannot adequately establish. See the official [MCP documentation](https://code.claude.com/docs/en/mcp).

## Code-first investigation

Start with code, tests, configuration, and local outputs. Use semantic tools during investigation where they answer the question better; use external tools when remote truth or runtime state is required.

**Do use MCP for:**

- Version-accurate documentation.
- [JetBrains IDE analysis and refactoring](code-intelligence.md#jetbrains-mcp-for-ide-backed-code-intelligence).
- Indexed relationships ordinary search cannot reliably enumerate.
- Database inspection, GitHub/CI state, issues, internal APIs, and required remote actions.
- Browser/runtime evidence when source cannot establish rendered behaviour.

**Don't:** open a browser before understanding the relevant code, query remote systems for locally available answers, or attach heavyweight tools to every session.

Keep frequently useful integrations available; scope database, deployment, browser, and rare systems to the tasks needing them.

## Hooks

Prefer advisory hooks: reminders, actionable diagnostics, logging, and notifications that let work continue. Even slightly ambiguous wording or a faulty blocking condition can repeatedly reject legitimate actions, trap Claude in retries or workarounds, and become a huge productivity drain. Route task intent through skill/agent descriptions, path scope, and local placement.

**Do:**

- Log commands and edits through `PostToolUse`.
- Provide short `SessionStart` reminders.
- Format touched files or report targeted check failures with an actionable next step; avoid blocking each edit while work is incomplete.
- Notify or record summaries at `Stop`.
- Record completion/idle events for configured multi-agent work.
- Keep hooks fast, deterministic, and visible; keep advisory failures non-blocking.

**Don't:** classify natural-language intent with hooks, turn engineering advice into command vetoes, or block legitimate deletion and replacement with broad matching. Put access restrictions in settings and sandbox policy, and completion checks in tests or CI.

Use a blocking hook only for a narrow, explicitly chosen requirement that those mechanisms cannot express. Test legitimate and prohibited cases, keep its decision deterministic, and provide a precise reason and remedy. Repeated false positives are a hook defect to fix, rather than a reason to make Claude fight the hook.

## Permissions

Claude Code provides `allow`, `ask`, and `deny` rules; seven documented modes (`default`/`manual`, `acceptEdits`, `plan`, `auto`, `dontAsk`, and `bypassPermissions`); sandbox controls; and a `PermissionRequest` hook. Rules use `Tool` or `Tool(specifier)` syntax and are evaluated deny first, then ask, then allow; the first match wins. Write policies for the intended action, test them with `/status` and a safe representative command, and do not treat a broad Bash rule as an airtight security boundary.

Recommended policy:

- put team-shared, deterministic `deny` rules for secrets and high-risk commands in `.claude/settings.json`;
- put personal convenience allows in `.claude/settings.local.json` or `~/.claude/settings.json`, not in a committed repository file;
- use sandbox filesystem, network, and credential restrictions when isolation matters, because they apply at the subprocess boundary;
- prefer advisory hooks; use blocking `PreToolUse` or `PermissionRequest` handling only for a narrow, tested requirement that settings and sandbox policy cannot express; and
- use `AGENTS.md`, skills, tests, and review for behavior that cannot be expressed as a deterministic access rule.

Auto mode can allow routine work while escalating risky actions in personal or managed environments. A repository cannot opt a user into it: project and local settings ignore `auto` and its prose-based `autoMode` policy. Keep hard prohibitions in deny rules or sandbox policy; the classifier is a convenience layer, not an authorization model.

`bypassPermissions` remains deliberately dangerous. It is appropriate only for an isolated, disposable environment where the user has consciously accepted unrestricted execution; organizations can disable it with `disableBypassPermissionsMode`. The example in `examples/settings.json` is therefore an opt-in personal configuration, not a recommended checked-in default.
