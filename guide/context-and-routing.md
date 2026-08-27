# Context and Routing

### What `CLAUDE.md` is for

`CLAUDE.md` is the repository-wide operating contract. The official docs support using `/init` to create it and keeping it concise and repo-specific. Every item in it should be crucial: information Claude cannot reliably infer or instructions that apply broadly enough to justify loading for every task.

The strongest recurring content categories are:

- exact build, run, test, lint, typecheck, and formatting commands that are not obvious from the repository,
- repository-wide testing and verification expectations,
- project-specific architectural decisions, boundaries, and invariants,
- project-specific conventions that differ from the language or framework defaults,
- repository etiquette and approval boundaries that apply across task types,
- and dangerous or generated areas.

These are candidates, not required template sections. Include an item only when omitting it would predictably make Claude less reliable. A short statement of the project's purpose can also be worthwhile when that purpose materially changes engineering decisions and is not already obvious from the repository; otherwise it is orientation prose competing with instructions.

It should not contain:

- exhaustive coding conventions,
- long domain documentation,
- troubleshooting playbooks,
- tool manuals,
- migration procedures,
- an index of files under `.claude/`,
- or every lesson ever learned.

`CLAUDE.md` is an operating contract. It is not a knowledge base or a directory of Claude configuration. Skills, rules, agents, and their supporting references should make themselves discoverable through their native metadata, scope, and ownership.

### What belongs in the root file

The root file should answer these questions immediately:

- What must Claude never do without asking?
- What is the default workflow for non-trivial changes?
- Which commands define "done"?
- Which project-specific decisions or conventions would Claude otherwise miss?
- Which areas of the repo are dangerous or generated?

That is enough.

### What does not belong

If content is only relevant to one subsystem, it should live in a subsystem-specific skill, path-scoped rule, or local reference owned by one of those mechanisms. If content is long enough that you would not want to read it before every task yourself, it does not belong in the root file either. If content needs mechanical enforcement, it belongs in tooling or hooks, not prose.

The common "keep adding rules to `CLAUDE.md` whenever Claude makes a mistake" advice is only partially right. Add an instruction only when it is crucial and broadly applicable. Move scoped, procedural, or nuanced guidance into the mechanism that can express when it applies. Otherwise you solve one mistake by creating a broader context-quality problem.

Do not create an artificial priority tier inside the file. A heading such as "Non-Negotiables" implies that the remaining content matters less. Everything in `CLAUDE.md` should earn the same scarce, always-on attention. When an instruction genuinely allows discretion or has exceptions, state that scope and nuance explicitly beside it.

### Recommended root structure

This is the model to use:

```md
# Repo Operating Rules

## Operating Instructions
- Before any non-trivial feature or bugfix:
  1. audit the existing code paths, abstractions, and conventions
  2. ask: "If I were starting from scratch, knowing what I know now, what is the best approach?"
  3. assume the area is not ready; refactor, extract, and encapsulate until it is a clean host for the change
  4. present the ideal solution first, even when it is larger or harder; label any lower-effort alternative and its debt explicitly
  5. explain the plan before broad or risky edits
- Never introduce a new dependency, pattern, abstraction, file layout, or naming scheme without explicit approval.
- Construct objects and select concrete external adapters only at explicit application boundaries. Functionality code receives required collaborators through constructors or function arguments; it must not discover them through globals, service locators, or hidden I/O.
- Mutable static or global state is banned, including singletons, shared instances, global registries, module-level mutable values, and global caches. Give state an explicitly constructed owner and lifetime, then pass it to consumers. Immutable constants and stateless pure functions are not state.
- Preserve encapsulation: give each object or module ownership of its state, invariants, and behavior; expose narrow intent-based APIs, and do not substitute helpers, forwarding interfaces, or mutable data access for a real abstraction.
- Do not add code comments unless documenting a non-obvious public API contract or an unavoidable external constraint, quirk, or hack. First make the code self-explanatory; permitted comments explain why, never narrate what.
- Prefer code inspection and existing tests before using MCPs, browser tools, or external automation.
- If a human-approved convention changes, update the Claude config files in the same change.
- Keep user-facing responses concise and outcome-first. State what changed, what was verified, and any decision or risk that needs attention; offer supporting detail instead of leading with it.

## Commands
- Test: `<your test command>`
- Lint or checks: `<your lint/check command>`
- Typecheck or compile check: `<your typecheck command>`
- Format: `<your format command>`

## Dangerous Areas
- Do not edit generated files in `<generated-code paths>`
- Ask before touching live infrastructure, security-critical code, billing, auth, or other high-blast-radius foundations
```

Why this works:

- Every instruction is important enough to justify always-on context.
- The commands define completion.
- The dangerous-areas section surfaces risks without bloating context.
- Scoped guidance remains independently discoverable instead of turning the root file into a configuration index.

### Response discipline is part of the operating contract

Claude should not make the user reconstruct the result from a long work log. A good default response leads with the outcome, followed only by the information needed to understand, verify, or decide on it. Routine implementation detail, command-by-command narration, and exhaustive investigation notes should be available on request, not included by default.

This is not a request to hide uncertainty or risk. Claude should still surface material trade-offs, failed verification, blockers, and decisions that need human approval. The constraint is on unnecessary detail: make the default easy to scan, then offer to provide the evidence, reasoning, or deeper walkthrough when it would help.

### Keep the root file short on purpose

The official [Memory](https://code.claude.com/docs/en/memory) guidance targets fewer than 200 lines per `CLAUDE.md`. Child files are appropriate only when a subtree genuinely works differently. Path-scoped rules reduce startup context; splitting content into `@path` imports only reorganizes it because imports still load with the parent file.

This content model is not merely a template convention. Anthropic's [Best Practices](https://code.claude.com/docs/en/best-practices) guidance names non-obvious commands, testing instructions, repository etiquette, project-specific architectural decisions, and code-style differences while excluding facts Claude can infer, standard conventions, volatile information, and tutorials. An [empirical study of 253 public `Claude.md` files](https://arxiv.org/abs/2509.14744) likewise found that build and run instructions, implementation guidance, architecture, and testing were the dominant categories. Prevalence does not prove effectiveness, but the agreement between official guidance, observed practice, and independent practitioner accounts makes these the clearest areas of broad consensus.

### Context and content stability

Keep root `CLAUDE.md` stable because it is loaded into every session, not because of undocumented caching internals. Claude Code documents prompt caching separately, but its safe operational guidance is simpler: put durable, always-relevant instructions in `CLAUDE.md`; put multi-step or local guidance in skills or path-scoped rules; put transient task state in the conversation, issue, or plan. A project-root `CLAUDE.md` is re-read after `/compact`, while nested files reload when Claude next reads in that subtree. Use `/context` to confirm which memory files loaded, `/memory` to inspect auto memory, and `/doctor` to identify a checked-in root file that needs trimming.

One important nuance: `.claude/rules/` is now part of the official memory and rules surface, so it is a legitimate tool for modularizing always-on or path-scoped instructions. The same architectural caution still applies: do not treat a rules folder as the whole system. Use it as one layer alongside root `CLAUDE.md`, skills, hooks, settings, and reference files. The important question is still whether each instruction lives in the correct layer and is loaded when needed.

## 3. Skills and Rules Architecture

### Define the layers clearly

The word "rules" is often used too loosely. For a long-lived project, separate the layers:

- `CLAUDE.md`: always-on repository operating contract.
- `.claude/rules/`: persistent always-on or path-scoped standing instructions.
- Skills: reusable, on-demand workflows and scoped instruction packages.
- Reference files: detailed conventions, subsystem notes, and examples used by skills.
- Hooks and tooling: enforcement, logging, and side effects.
- `settings.json`: allow/deny behavior, permission posture, and related access policy.

The official [Skills](https://code.claude.com/docs/en/skills) docs are the strongest source here. Anthropic treats skills as the right way to package workflows and reusable context. Skills can be project-, personal-, enterprise-, or plugin-scoped; nested project skills become available when Claude reads or edits in that subtree. They can also run in a forked context when a bounded research or review task deserves its own agent.

Rules deserve first-class treatment in this architecture. They are the right home for standing instructions that should load automatically all the time or automatically for a subtree. Skills are different: they are better for task-shaped workflows, investigation flows, and reusable procedures that Claude should invoke based on intent. Reference files are different again: they hold detail that supports a rule or skill without needing to load constantly.

Use these mechanisms aggressively. Highly technical guidance is valuable precisely because skills and path-scoped rules let it remain narrow: SwiftUI accessibility rules can appear for SwiftUI work, database transaction conventions for persistence work, and UI audit criteria for interface reviews without taxing every unrelated task. The goal is not less guidance. It is more relevant guidance, loaded only where and when it applies.

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
- crucial repository-wide instructions that should live in root `CLAUDE.md` instead.

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

The stronger target is zero ritual. The user describes the desired outcome; Claude identifies the task and touched subsystem, loads the applicable rules and skills, follows their references, and performs the work. Users may invoke a skill explicitly when they want to, but routine correctness must not depend on them knowing the configuration. If the user repeatedly has to say "use the UI skill" or "check the database rules," treat that as a routing defect in the skill description, rule scope, naming, or ownership of its supporting references.

This needs to be stated plainly: if you want Claude to use any part of the Claude-side file surface unprompted, it is not enough for those files to merely exist somewhere in the repository. Each mechanism must be discoverable in its native way: skills through precise metadata, rules and local `CLAUDE.md` files through appropriate scope, agents through clear descriptions, and reference documents through the skill or rule that owns them. Orphaned markdown is not a discoverability strategy.

Do not compensate for weak `.claude/` configuration by listing or linking it from root `CLAUDE.md`. That hides the defect while spending context on every task. Fix the skill metadata, rule scope, agent description, directory placement, or reference ownership so normal task wording and touched paths lead Claude to the right material directly.

Rules fit into this discoverability model too. A rule is discoverable when Claude can load it automatically because of its always-on scope or its path scope. That is exactly why rules are useful: they make standing guidance discoverable without requiring the user to remember an invocation step.

Use these rules:

- Give skills names and `description` fields that match real task language such as "feature workflow", "API conventions", or "bug investigation", not internal jargon.
- Put likely trigger phrases and exclusions directly in the `description`; it is the routing metadata Claude sees before loading the full skill.
- State both when to use the skill and when not to use it. Overlapping skills reduce routing reliability.
- Keep high-frequency skills narrow and obvious so Claude can confidently auto-select them.
- Keep rare or heavy workflows explicit. Automatic routing should cover common cases, not every possible case.
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
3. Inspect touched functionality for hidden construction, service location, mutable static or global state, configuration reads, or I/O. Remove mutable globals from the touched path; move state into explicitly constructed, lifetime-owned objects and pass required capabilities inward.
4. Identify which object or module should own the affected state, invariants, and decisions. Check that callers can use a narrow intent-based API without coordinating the owner's internals.
5. Ask: "If I were starting from scratch, knowing what I know now, what is the best approach?"
6. Assume the area is not ready for the new feature until proven otherwise. Refactor, extract, and encapsulate until it is a clean host for the change.
7. Present the ideal solution first. If a smaller or faster alternative exists, label it as a compromise and state the debt, constraint, or risk it accepts.
8. If the ideal solution introduces a new dependency, pattern, abstraction, or file structure, stop and ask for approval.
9. Implement only after the structure is coherent.
10. Review every added or retained code comment. Remove narration and comments made redundant by the refactor; keep only documented public contracts and unavoidable why-level constraints.
11. Run targeted verification, including a direct unit test constructed with explicit fakes where applicable.
12. If the task established a new approved convention, update the config files in the same change.
```

The point is not elegance. The point is to make Claude's default operating behavior hostile to drift.

### Auto-loaded vs explicitly loaded skills

Safe skills whose relevance can be inferred should normally be model-invocable. Progressive disclosure already keeps their bodies and supporting references out of context until needed, so a technically deep skill does not need to become always-on noise merely to be automatically discoverable.

Use automatic routing for:

- frequent workflow skills,
- narrow subsystem conventions,
- recurring bugfix or review flows,
- specialized technical guidance tied to detectable task language, file types, or paths,
- heavy reference-backed workflows whose entry skill can load the detail progressively,
- and any useful guidance the user should not have to remember to invoke manually.

Require explicit invocation for:

- workflows whose activation itself represents a user decision, external side effect, or approval boundary,
- destructive, privileged, experimental, or unusually costly operations,
- and genuinely ambiguous tasks where automatic selection would be unsafe.

The official skill frontmatter supports this distinction through `disable-model-invocation` and `user-invocable`; `context: fork` changes execution into a forked agent context and now runs in the background by default unless the skill sets `background: false`. Edits from a backgrounded fork fall outside the parent session's checkpoints, so `/rewind` does not undo them; use git to review and revert them. Use nested directories or path-scoped rules for local applicability. Keep routing metadata precise and non-overlapping, but do not hide safe, valuable expertise behind slash commands merely to keep the skill list short.

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
