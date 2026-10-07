# Context and Routing

Instructions work when Claude encounters the right guidance before acting. Keep the repository contract concise, route local knowledge on demand, and put deterministic enforcement in tooling.

## Root `AGENTS.md`

Include only crucial repository-wide instructions or facts Claude cannot reliably infer. Use `/init` as a starting point and review the result.

**Do include:**

- Only important and non-obvious build, run, test, lint, typecheck, and format commands.
- Repository-wide completion checks, workflow, Git etiquette, and approval boundaries.
- Project-specific architecture, invariants, and departures from standard conventions.
- Dangerous or generated areas.
- Project purpose only when it materially changes engineering decisions.
- Links to documentation.

**Don't include:**

- Exhaustive conventions, domain documentation, tool manuals, or playbooks.
- Local procedures, transient task state, or every past lesson.
- An index of `.claude/` files to compensate for poor routing.
- Artificial priority tiers such as “Non-Negotiables”; every retained instruction should earn its always-on attention.

Keep exceptions beside the instruction they qualify. Put subtree-specific guidance in local skills or path-scoped rules, and mechanically enforce what can be checked.

### Example root template

**Example:** adapt the sections, commands, and policy choices to the repository. This is not a required checklist or a ready-to-install configuration.

```md
# Repo Operating Rules

- For non-trivial work: load conventions, audit, refactor for readiness,
  implement, and verify. Present the best solution first.
- Ask before introducing a new dependency, pattern, abstraction, file layout,
  or naming scheme. Explain plans before broad or risky edits.
- Keep external systems behind narrow application-owned adapters.
- Construct dependencies at the edges and pass capabilities inward.
  Ban mutable globals, static state, and singletons; give state explicit owners.
- Prefer an explicitly supplied config file, validated into an immutable typed object.
  Inject needed settings; keep environment reads at the entry boundary, out of behaviour code.
- Preserve encapsulation and use intent-based APIs.
- Add comments only for non-obvious public contracts or unavoidable constraints.
- Do not conceal failures with catch-and-continue, guessed defaults, speculative
  fallbacks, or blind retries. Recovery must follow approved policy and preserve valid state.
- Start investigations with source and tests; use tools for evidence they add.
- Follow the repository's Git policy and record approved convention changes.
- Report outcomes, verification, and material decisions concisely.

## Commands

- Test: `<targeted test command>`
- Checks: `<lint/typecheck command>`
- Format: `<format command>`
- Full build: `<CI job>`

## Dangerous Areas

- Do not edit generated files in `<paths>`.
- Ask before touching `<project-specific high-blast-radius areas>`.
```

Include these instructions only where they represent chosen project policy. Keep detailed workflows in their owning skills.

### Context stability

The official [Memory](https://code.claude.com/docs/en/memory) guidance targets fewer than 200 lines per file. `@path` imports reorganise content but still load with their parent; they do not reduce startup context.

**Do:** keep root guidance durable, use child files only where a subtree genuinely differs, and keep temporary state in the conversation, issue, or plan. Use `/context` to inspect loaded memory, `/memory` for auto memory, and `/doctor` for configuration issues. Root instructions reload after `/compact`; nested files reload when that subtree is read.

**Don't:** add a root rule for every isolated mistake or store team policy only in auto memory.

See [Best Practices](https://code.claude.com/docs/en/best-practices) and the [study of public agent manifests](https://arxiv.org/abs/2509.14744) for the narrow operating-contract approach.

## Instruction layers

| Surface | Owns |
| --- | --- |
| `AGENTS.md` | Crucial repository-wide operating contract |
| `.claude/rules/` | Always-on or path-scoped standing guidance |
| Skills | Task-shaped workflows and reusable context |
| References | Detail and examples owned by a rule or skill |
| Hooks | Advisory reminders, diagnostics, logging, and deterministic side effects |
| Tooling and CI | Repeatable checks and deterministic enforcement |
| `settings.json` | Permissions and access policy |

Use the layer that controls when the instruction applies. Rules govern standing behaviour; skills govern procedures; references supply facts and detail. See the official [Skills documentation](https://code.claude.com/docs/en/skills).

### Example repository layout

**Example:** this tree illustrates the instruction layers. The filenames, skill names, and supporting-file arrangement are illustrative; create only what the repository needs and follow the tool’s required discovery locations.

```text
.claude/
  rules/
    global.md
    api.md
    frontend.md
  skills/
    feature-workflow/
      SKILL.md
    testing-conventions/
      SKILL.md
    bug-investigation/
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

## Design for self-discovery

The user should be able to describe the desired outcome naturally. Routine correctness should not require remembering skill names or slash commands.

**Do:**

- Match skill names and descriptions to real requests, with trigger phrases and exclusions.
- Keep workflows narrow: feature work, tests, bugs, reviews, releases, configuration, and distinct subsystem conventions.
- Use nested placement and path-scoped rules for local guidance.
- Give reference documents an owning rule or skill.
- Describe agents by bounded jobs such as repository audit or migration review.
- Test routing with several natural prompts; fix metadata, overlap, placement, or ownership when it fails.

**Don't:**

- Create grab-bag “backend” or “good coding” skills.
- Leave important guidance in orphaned Markdown.
- Fix missed routing by expanding the root index or making the user remember more commands.

Skills can be project-, personal-, enterprise-, or plugin-scoped. Nested project skills become available when Claude reads or edits that subtree.

## Automatic versus explicit invocation

Automatically route safe guidance when its relevance is clear: common workflows, local conventions, bug and review methods, and deep reference-backed expertise. Progressive disclosure keeps unused detail out of context.

Require explicit invocation when activation is itself a user decision or side effect, or the workflow is destructive, privileged, experimental, unusually costly, or unsafe to select from ambiguous wording.

Use `disable-model-invocation` and `user-invocable` to express that distinction. `context: fork` uses a background fork unless `background: false` is set. Fork edits fall outside parent checkpoints; `/rewind` does not undo them, so use Git for review or recovery.

### Feature-workflow skill

Keep the entry workflow short; let references supply repository-specific detail.

**Example skill:** adapt the name, triggers, and steps to the repository’s approved workflow.

```md
---
description: Use for non-trivial features and bugfixes requiring audit, readiness refactoring, implementation, and verification. Excludes typo-only and formatting-only edits.
user-invocable: true
---

# Feature Workflow

1. Load applicable conventions and search callers, implementations, and peers.
2. Inspect ownership, invariants, hidden dependencies, mutable global state,
   and provider-contract leakage in touched paths.
3. Identify the best design for the requirement and any approval decisions.
4. Refactor only what the current requirement needs; then implement.
5. Review the diff and comments; run appropriate verification.
6. Record any approved convention changes in their owning configuration.
```

## Response discipline

Lead with the outcome, relevant verification, and decisions or risks requiring attention. Keep routine command logs and exhaustive investigation notes available on request. Report failures, blockers, and material trade-offs directly.
