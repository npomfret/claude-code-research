# Context and Routing

### What `CLAUDE.md` is for

`CLAUDE.md` is the root routing layer. The official docs support using `/init` to create it and keeping it concise and repo-specific. It should contain information Claude cannot reliably infer and rules that are always in force.

That means the root file should contain:

- non-negotiable operating rules,
- canonical commands,
- architecture boundaries,
- dangerous areas,
- and pointers to deeper, on-demand skills and reference files.

It should not contain:

- exhaustive coding conventions,
- long domain documentation,
- troubleshooting playbooks,
- tool manuals,
- migration procedures,
- or every lesson ever learned.

`CLAUDE.md` is an index and an operating contract. It is not a knowledge base.

### What belongs in the root file

The root file should answer these questions immediately:

- What must Claude never do without asking?
- What is the default workflow for non-trivial changes?
- Which commands define "done"?
- Which areas of the repo are dangerous or generated?
- Where do the detailed conventions live?

That is enough.

### What does not belong

If content is only relevant to one subsystem, it should live in a subsystem-specific skill or local reference. If content is long enough that you would not want to read it before every task yourself, it does not belong in the root file either. If content needs mechanical enforcement, it belongs in tooling or hooks, not prose.

The common "keep adding rules to `CLAUDE.md` whenever Claude makes a mistake" advice is only partially right. Keep adding **routing-level** rules there when they are always-on and short. Move everything else into on-demand structures. Otherwise you solve one mistake by creating a broader context-quality problem.

### Recommended root structure

This is the model to use:

```md
# Repo Operating Rules

## Non-Negotiables
- Before any non-trivial feature or bugfix:
  1. audit the existing code paths, abstractions, and conventions
  2. ask: "If I were starting from scratch, knowing what I know now, what is the best approach?"
  3. assume the area is not ready; refactor, extract, and encapsulate until it is a clean host for the change
  4. present the ideal solution first, even when it is larger or harder; label any lower-effort alternative and its debt explicitly
  5. explain the plan before broad or risky edits
- Never introduce a new dependency, pattern, abstraction, file layout, or naming scheme without explicit approval.
- Before writing code, identify the applicable convention skill(s). If no convention exists, stop and ask.
- Prefer code inspection and existing tests before using MCPs, browser tools, or external automation.
- If a human-approved convention changes, update the Claude config files in the same change.
- Keep user-facing responses concise and outcome-first. State what changed, what was verified, and any decision or risk that needs attention; offer supporting detail instead of leading with it.

## Commands
- Test: `<your test command>`
- Lint or checks: `<your lint/check command>`
- Typecheck or compile check: `<your typecheck command>`
- Format: `<your format command>`

## Architecture Map
- Core conventions: `<path to global convention skill or reference>`
- Subsystem conventions: `<path to subsystem-specific guidance>`
- Feature workflow: `<path to implementation workflow skill>`
- Config maintenance: `<path to config-maintenance guidance>`

## Dangerous Areas
- Do not edit generated files in `<generated-code paths>`
- Ask before touching live infrastructure, security-critical code, billing, auth, or other high-blast-radius foundations
```

Why this works:

- The non-negotiables solve the real failure modes directly.
- The commands define completion.
- The architecture map points Claude to on-demand detail.
- The dangerous-areas section surfaces risks without bloating context.

### Response discipline is part of the operating contract

Claude should not make the user reconstruct the result from a long work log. A good default response leads with the outcome, followed only by the information needed to understand, verify, or decide on it. Routine implementation detail, command-by-command narration, and exhaustive investigation notes should be available on request, not included by default.

This is not a request to hide uncertainty or risk. Claude should still surface material trade-offs, failed verification, blockers, and decisions that need human approval. The constraint is on unnecessary detail: make the default easy to scan, then offer to provide the evidence, reasoning, or deeper walkthrough when it would help.

### Keep the root file short on purpose

The official [Memory](https://code.claude.com/docs/en/memory) guidance targets fewer than 200 lines per `CLAUDE.md`. Child files are appropriate only when a subtree genuinely works differently. Path-scoped rules reduce startup context; splitting content into `@path` imports only reorganizes it because imports still load with the parent file.

### Context and content stability

Keep root memory stable because it is loaded into every session, not because of undocumented caching internals. Claude Code documents prompt caching separately, but its safe operational guidance is simpler: put durable, always-relevant instructions in `CLAUDE.md`; put multi-step or local guidance in skills or path-scoped rules; put transient task state in the conversation, issue, or plan. A project-root `CLAUDE.md` is re-read after `/compact`, while nested files reload when Claude next reads in that subtree. Use `/context` to confirm which memory files loaded, `/memory` to inspect auto memory, and `/doctor` to identify a checked-in root file that needs trimming.

One important nuance: `.claude/rules/` is now part of the official memory and rules surface, so it is a legitimate tool for modularizing always-on or path-scoped instructions. The same architectural caution still applies: do not treat a rules folder as the whole system. Use it as one layer alongside root `CLAUDE.md`, skills, hooks, settings, and reference files. The important question is still whether each instruction lives in the correct layer and is loaded when needed.

## 3. Skills and Rules Architecture

### Define the layers clearly

The word "rules" is often used too loosely. For a long-lived project, separate the layers:

- `CLAUDE.md`: always-on operating contract and routing.
- `.claude/rules/`: persistent always-on or path-scoped standing instructions.
- Skills: reusable, on-demand workflows and scoped instruction packages.
- Reference files: detailed conventions, subsystem notes, and examples used by skills.
- Hooks and tooling: enforcement, logging, and side effects.
- `settings.json`: allow/deny behavior, permission posture, and related access policy.

The official [Skills](https://code.claude.com/docs/en/skills) docs are the strongest source here. Anthropic treats skills as the right way to package workflows and reusable context. Skills can be project-, personal-, enterprise-, or plugin-scoped; nested project skills become available when Claude reads or edits in that subtree. They can also run in a forked context when a bounded research or review task deserves its own agent.

Rules deserve first-class treatment in this architecture. They are the right home for standing instructions that should load automatically all the time or automatically for a subtree. Skills are different: they are better for task-shaped workflows, investigation flows, and reusable procedures that Claude should invoke based on intent. Reference files are different again: they hold detail that supports a rule or skill without needing to load constantly.

### What a skill should do

A skill should answer one question cleanly: "When this task type appears, what exact process and constraints should Claude follow?"

Good skill categories:

- global coding conventions,
- subsystem conventions,
- feature implementation workflow,
- bug investigation workflow,
- review workflow,
- release workflow,
- config-maintenance workflow.

Bad skill categories:

- giant grab-bag "backend skill",
- vague "good coding practices",
- one-off project notes that are never reused,
- rules that should just live in the root file.

### Recommended repository layout

Use a structure like this:

```text
.claude/
  rules/
    global.md
    api.md
    frontend.md
  skills/
    feature-workflow/
      SKILL.md
    conventions-global/
      SKILL.md
    api-conventions/
      SKILL.md
    frontend-conventions/
      SKILL.md
    config-maintenance/
      SKILL.md
  references/
    architecture-map.md
    conventions-global.md
    api-conventions.md
    frontend-conventions.md
  hooks/
    command-log.sh
    post-edit-check.sh
```

This design keeps the root file short while still making detailed guidance available. It also avoids collapsing the whole architecture into either one giant root file or one giant rules folder when different instruction types belong in different layers.

### Design for self-discovery

Assume the user will often forget which skill, rule, or agent exists. The setup should still work.

That means common workflows must be designed so Claude can discover and route to them automatically from ordinary task wording. If a recurring workflow only works when the human remembers a specific slash command or exact skill name, the setup is underspecified.

This needs to be stated plainly: if you want Claude to use any part of the Claude-side file surface unprompted — root `CLAUDE.md`, local `CLAUDE.md` files, skills, agent definitions, reference documents, settings-adjacent guidance, or other repo-owned Claude config files — it is not enough for those files to merely exist somewhere in the repository. They have to be discoverable. In practice that means each important file needs a clear routing path through root `CLAUDE.md`, skill metadata, agent descriptions, local scope boundaries, or another explicit entry point Claude can infer from normal task wording. Orphaned markdown is not a discoverability strategy.

Rules fit into this discoverability model too. A rule is discoverable when Claude can load it automatically because of its always-on scope or its path scope. That is exactly why rules are useful: they make standing guidance discoverable without requiring the user to remember an invocation step.

Use these rules:

- Give skills names and `description` fields that match real task language such as "feature workflow", "API conventions", or "bug investigation", not internal jargon.
- Put likely trigger phrases and exclusions directly in the `description`; it is the routing metadata Claude sees before loading the full skill.
- State both when to use the skill and when not to use it. Overlapping skills reduce routing reliability.
- Keep high-frequency skills narrow and obvious so Claude can confidently auto-select them.
- Keep rare or heavy workflows explicit. Automatic routing should cover common cases, not every possible case.
- Put routing hints in root `CLAUDE.md` so Claude is told to locate the applicable convention or workflow skill before editing.
- Put genuinely local skills under the relevant nested `.claude/skills/` directory; use path-scoped rules for standing guidance by file type or subtree.
- Write reference docs to support a skill, not to act as orphaned markdown that Claude might never load.
- If you use custom agents or subagents, define them around distinct jobs Claude can infer, such as `repo-audit`, `ui-review`, or `migration-check`, not vague labels like `engineer` or `helper`.
- If Claude repeatedly misses a relevant skill, fix the skill metadata, split overlapping skills, or rename the skill. Do not solve repeated routing failures by telling the user to remember more commands.
- Test automatic routing with several natural prompts a programmer might actually type. Treat repeated missed invocations as a metadata, naming, or scope bug.

The design target is simple: for common task types, the user should be able to ask for the work naturally and Claude should pull in the right workflow guidance without being hand-held.

### How to write a skill

A good skill is concise, narrow, and action-oriented. The official skills docs are right that routing quality depends heavily on its `description`, invocation settings, directory scope, and tool boundaries. Skills may include supporting files, dynamic context injection, and `context: fork` with a suitable agent for bounded work.

Example:

```md
---
description: Use for any non-trivial feature or bugfix. Enforces audit -> refactor -> implement -> verify. Do not use for typo-only or formatting-only edits.
user-invocable: true
---

# Feature Workflow

1. Load the applicable convention skills before writing code.
2. Audit the touched code paths and identify the current canonical patterns.
3. Ask: "If I were starting from scratch, knowing what I know now, what is the best approach?"
4. Assume the area is not ready for the new feature until proven otherwise. Refactor, extract, and encapsulate until it is a clean host for the change.
5. Present the ideal solution first. If a smaller or faster alternative exists, label it as a compromise and state the debt, constraint, or risk it accepts.
6. If the ideal solution introduces a new dependency, pattern, abstraction, or file structure, stop and ask for approval.
7. Implement only after the structure is coherent.
8. Run targeted verification.
9. If the task established a new approved convention, update the config files in the same change.
```

The point is not elegance. The point is to make Claude's default operating behavior hostile to drift.

### Auto-loaded vs explicitly loaded skills

Not every skill should be model-invocable by default.

Use automatic routing for:

- frequent workflow skills,
- narrow subsystem conventions,
- recurring bugfix or review flows,
- and any guidance the user is likely to forget to invoke manually.

Require explicit invocation for:

- heavy reference material,
- low-frequency release tasks,
- migration playbooks,
- experimental or risky workflows.

The official skill frontmatter supports this distinction through `disable-model-invocation` and `user-invocable`; `context: fork` changes execution into a forked agent context and now runs in the background by default unless the skill sets `background: false`. Edits from a backgrounded fork fall outside the parent session's checkpoints, so `/rewind` does not undo them; use git to review and revert them. Use nested directories or path-scoped rules for local applicability. Keep the auto-routed surface small and sharp.

### Put each rule in its enforceable layer

A rule is a load-bearing instruction with a clear home, not an arbitrary markdown file:

- Root rule in `CLAUDE.md`:
  - "Never introduce a new dependency or abstraction without explicit approval."
- Path-scoped standing rule in `.claude/rules/`:
  - "In `<subsystem path>`, use the shared error translation pattern and do not invent local response shapes."
- Scoped rule in a convention doc or skill:
  - "For this task type, audit -> refactor -> implement -> verify."
- Permission rule in `settings.json`:
  - "This command family is allowed, this one must ask, and this one is denied."
- Enforced rule in tooling:
  - "Touched API packages must pass their targeted test command before the task is complete."

Reference documents may support any of these layers, but documentation alone does not route or enforce a rule. The important test is whether Claude reliably encounters the instruction and whether a deterministic rule can be moved into tooling.
