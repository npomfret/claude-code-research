# Claude Code Setup for Long-Running Projects

Last verified: **March 30, 2026**

This guide is for real codebases that will still matter in six months. It is not for demos, weekend prototypes, or "look what the agent can do" threads. The central problem is simple: Claude Code is useful, fast, and broad, but without structure it is a sloppy programmer. It duplicates logic, patches features into code that should have been refactored first, and slowly fills a codebase with pattern drift.

Anthropic's official documentation is the source of truth for what Claude Code can do: [Best Practices](https://code.claude.com/docs/en/best-practices), [Memory](https://code.claude.com/docs/en/memory), [Skills](https://code.claude.com/docs/en/skills), [Hooks](https://code.claude.com/docs/en/hooks), [MCP](https://code.claude.com/docs/en/mcp), [Commands](https://code.claude.com/docs/en/commands), [CLI Reference](https://code.claude.com/docs/en/cli-reference), [Settings](https://docs.anthropic.com/en/docs/claude-code/settings), and the [Claude Code changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md). The research reports in the reading-list index are useful as evidence of how people are using the tool in practice, but they are not authoritative. Several of them are good at surfacing mechanisms and bad at describing what scales.

## Common Claude Failure Modes

If the setup does not actively counter these, Claude will keep doing them:

- It copy-pastes locally convenient logic instead of finding or extracting the shared abstraction.
- It makes the smallest possible code change even when the surrounding structure is unready for the new requirement.
- It invents slight pattern variants because the first few files it read looked "close enough."
- It sometimes over-engineers in the opposite direction by introducing speculative abstractions that the current codebase does not actually need.
- It silently introduces new abstractions, dependencies, or file shapes unless explicitly told to stop and ask.
- It follows whatever context is most visible, which means bloated root memory and poorly routed guidance actively make it worse.
- It reaches for tools, MCPs, or browser automation before exhausting code-level investigation if those tools are available.
- It misses reusable workflow instructions when they are not designed to be automatically discoverable from the user's wording.

The rest of this guide exists to suppress those failure modes structurally rather than hoping Claude behaves better on its own.

## Source Posture

Before talking about setup, it is worth being explicit about the source material.

- The official Anthropic docs are strong on product capabilities and weak on long-project governance. They correctly emphasize concise memory, specificity, skills, hooks, verification, and current commands, but they do not by themselves solve code quality drift.
- The report on the `claude-code-best-practice` repository is useful because it surfaces recurring tactics such as planning first, concise `CLAUDE.md`, verification loops, and parallel work. It is weak where it turns a pile of tips into an implied architecture. Long-lived projects need a tighter system than "86 good ideas." Source: [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html).
- The Boris/`CLAUDE.md` memory-system report correctly identifies memory, process discipline, and feedback loops as the real leverage. It becomes dangerous if interpreted as "keep adding more to root memory forever." That is exactly how you waste context. Source: [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html).
- The hidden-features report is good for capability discovery and weak as a setup philosophy. It surfaces `/loop`, hooks, mobile, teleportation, browser access, and slash commands, but feature discovery is not the same as codebase governance. Source: [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html).
- The gstack/Superpowers/Compound comparison is good at separating planning, evaluation, workflow discipline, and knowledge compounding. It is less useful if taken as a required stack for every team. This guide borrows the separation-of-concerns insight and rejects the meta-framework sprawl. Source: [I Compared gstack, Superpowers, and Compound Engineering](https://npomfret.github.io/reading-list-researcher/e502b3549aff92cb.html).
- The Superpowers repository itself is a useful primary source for three ideas this guide agrees with: common workflows should auto-route instead of relying on user memory, testing discipline should be explicit, and speculative complexity should be resisted through principles like TDD, YAGNI, and DRY. It is less useful where it prescribes a larger workflow stack and git worktree-centric execution model than many teams need. Sources: [Superpowers repo](https://github.com/obra/superpowers), [Best GitHub repos for Claude code that will 10x your next project](https://npomfret.github.io/reading-list-researcher/5187cc080607fbc8.html).
- The three "best GitHub repos for Claude Code" reports are ecosystem signals, not setup doctrine. They help identify recurring needs such as persistent memory, design guidance, automation, and curated skills, but curation lists are not architecture. Sources: [Kshitij Mishra's list](https://npomfret.github.io/reading-list-researcher/e6f4d1afbd729e56.html), [Hasan Toor's list](https://npomfret.github.io/reading-list-researcher/5187cc080607fbc8.html), [Jahir Sheikh's list](https://npomfret.github.io/reading-list-researcher/c1aa58026580b10e.html).
- The Google Stitch workflow report is a valid example of high-value task-specific context. It is still narrow, frontend-heavy, and token-expensive. It should influence design-task setup, not the architecture of the whole Claude Code system. Source: [Claude Code + Google Stitch 2.0](https://npomfret.github.io/reading-list-researcher/2aa3ed775a7a775e.html).
- The dev-browser report is useful as evidence that browser tooling can be valuable. It is not evidence that browser tooling should be a default reflex. Source: [Sawyer Hood Announces dev-browser Hits 4k GitHub Stars](https://npomfret.github.io/reading-list-researcher/b4cbb3b860869a88.html).
- The changelog is essential because it proves which features actually exist now, including skill hot-reload, forked skill contexts, new hook events, memory behavior changes, and operational fixes. It is descriptive, not prescriptive. Source: [Claude Code CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md).

The recurring consensus across the sources is real: keep memory concise, plan before broad edits, verify aggressively, and package repeated workflows. The missing piece is that none of those habits alone will stop Claude from introducing sloppiness. The setup must be built around preventing drift.

## 1. The Problem Statement

### Claude's default behavior is locally convenient and globally damaging

Claude Code does not naturally optimize for long-term codebase coherence. It optimizes for completing the current task with the least resistance. That produces three predictable pathologies:

1. It copy-pastes logic instead of abstracting shared behavior.
2. It duplicates an existing pattern because it did not search broadly enough before writing.
3. It introduces a slightly different way of doing something because the local context made it look reasonable.

The official [Best Practices](https://code.claude.com/docs/en/best-practices) doc is right to stress specificity, context, and verification. The Boris memory-system report is right that process discipline matters more than "prompt tricks." The best-practices-repo report is right that many teams underuse planning and reusable workflows. But most public advice still understates the main issue: the failure mode is not just "Claude occasionally makes mistakes." The failure mode is ongoing structural degradation.

Pattern drift compounds. One new error-handling style becomes three. A second API-call shape appears because the agent patched one endpoint quickly. One feature uses a shared helper, the next writes the logic inline, and a third invents a wrapper. Six weeks later the project has inconsistent behavior, partial abstractions, and bugs that only exist in one branch of duplicated code.

### Claude has a minimum-change bias

This is the most damaging bias for mature projects. When asked to add a feature, Claude usually assumes the current structure is valid and makes the minimum possible change to fit the feature in. That is fast in the moment and disastrous over time.

The hidden-features report and the Boris workflow reports correctly push planning and verification, but they do not go far enough on feature readiness. A world-class setup must encode a different default:

1. Audit the relevant code.
2. Assume it is not ready.
3. Refactor the area into a coherent shape.
4. Only then add the feature.

If that sequence is not enforced, Claude will happily bolt new behavior onto weak foundations forever.

### Context rots, and naive configuration makes it worse

Anthropic's [Memory](https://code.claude.com/docs/en/memory) guidance and [Best Practices](https://code.claude.com/docs/en/best-practices) guidance are correct that project memory should be concise and specific. The most damaging bad advice in the ecosystem is the repeated suggestion to "just add it to `CLAUDE.md`." That advice works on tiny projects because the entire repo is tiny. On a real project, `CLAUDE.md` is always-on context. Every unnecessary paragraph steals budget from the actual task.

The Boris/`CLAUDE.md` report contains a valid core idea: institutional memory matters. The invalid extension is treating the root memory file as the place to store all institutional memory. It is not. Long-lived projects need discoverable, on-demand context instead.

### Most public advice is optimized for the wrong use case

The ecosystem reports are full of novel features, tool lists, and throughput stories. Some of that is useful. Much of it is optimized for:

- demos,
- short-lived repos,
- personal experimentation,
- impressive screenshots,
- or loosely governed side projects.

That is why this guide rejects several common recommendations. Worktrees may be productive for some users, but they are not the best default for interactive agent collaboration. Browser tools are powerful, but using them before reading the source is context waste. Hook-based command blocking sounds safe, but it creates brittle friction while leaving the underlying governance problem unsolved.

The right question for every technique is: does this make the codebase more coherent after six months, or does it just help the current session look clever?

### Further reading

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)
- [Superpowers repo](https://github.com/obra/superpowers)

## 2. `CLAUDE.md` Architecture

### What `CLAUDE.md` is for

`CLAUDE.md` is the root routing layer. The official docs support using `/init` to create it and keeping it concise and repo-specific. The best-practices report, the Boris memory-system report, and Anthropic's own memory guidance all point in the same direction: the root file should contain information Claude cannot reliably infer and rules that are always in force.

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
  2. assume the area is not ready
  3. refactor for readiness before adding behavior
  4. explain the plan before broad or risky edits
- Never introduce a new dependency, pattern, abstraction, file layout, or naming scheme without explicit approval.
- Before writing code, identify the applicable convention skill(s). If no convention exists, stop and ask.
- Prefer code inspection and existing tests before using MCPs, browser tools, or external automation.
- If a human-approved convention changes, update the Claude config files in the same change.

## Commands
- Test: `pnpm test`
- Lint: `pnpm lint`
- Typecheck: `pnpm typecheck`
- Format: `pnpm format`

## Architecture Map
- API conventions: `.claude/skills/api-conventions`
- Frontend conventions: `.claude/skills/frontend-conventions`
- Feature workflow: `.claude/skills/feature-workflow`
- Config maintenance: `.claude/skills/config-maintenance`

## Dangerous Areas
- Do not edit generated files in `src/generated/**`
- Ask before touching deployment, billing, or auth foundation files
```

Why this works:

- The non-negotiables solve the real failure modes directly.
- The commands define completion.
- The architecture map points Claude to on-demand detail.
- The dangerous-areas section surfaces risks without bloating context.

### Keep the root file short on purpose

The official [Memory](https://code.claude.com/docs/en/memory) model supports multiple memory layers, including project memory, user memory, and more local memory. Use that. Child `CLAUDE.md` files are appropriate only when a subtree genuinely works differently. Do not create a forest of local memory files unless the repo really has distinct local rules.

One important nuance: community examples often use `.claude/rules/`. That may be a fine organizational convention, but it is not as strongly documented as `CLAUDE.md`, skills, hooks, settings, and permissions. Do not make your core architecture depend on undocumented assumptions about automatic rule loading. If you use `.claude/rules/`, treat it as a repository organization choice, not a magical runtime feature.

### Further reading

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)

## 3. Skills and Rules Architecture

### Define the layers clearly

The ecosystem uses the word "rules" too loosely. For a long-lived project, separate the layers:

- `CLAUDE.md`: always-on operating contract and routing.
- Skills: reusable, on-demand workflows and scoped instruction packages.
- Reference files: detailed conventions, subsystem notes, and examples used by skills.
- Hooks and tooling: enforcement, logging, and side effects.

The official [Skills](https://code.claude.com/docs/en/skills) docs are the strongest source here. Anthropic treats skills as the right way to package workflows and reusable context. The changelog confirms this surface is active and improving, including skill hot-reload and forked execution contexts in `2.1.0`.

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

This design keeps the root file short while still making detailed guidance available. It also avoids relying on `.claude/rules/` semantics that may not be stable or well-documented.

### Design for self-discovery

Assume the user will often forget which skill, rule, or agent exists. The setup should still work.

That means common workflows must be designed so Claude can discover and route to them automatically from ordinary task wording. If a recurring workflow only works when the human remembers a specific slash command or exact skill name, the setup is underspecified.

Use these rules:

- Give skills names and `description` fields that match real task language such as "feature workflow", "API conventions", or "bug investigation", not internal jargon.
- State both when to use the skill and when not to use it. Overlapping skills reduce routing reliability.
- Keep high-frequency skills narrow and obvious so Claude can confidently auto-select them.
- Keep rare or heavy workflows explicit. Automatic routing should cover common cases, not every possible case.
- Put routing hints in root `CLAUDE.md` so Claude is told to locate the applicable convention or workflow skill before editing.
- Write reference docs to support a skill, not to act as orphaned markdown that Claude might never load.
- If you use custom agents or subagents, define them around distinct jobs Claude can infer, such as `repo-audit`, `ui-review`, or `migration-check`, not vague labels like `engineer` or `helper`.
- If Claude repeatedly misses a relevant skill, fix the skill metadata, split overlapping skills, or rename the skill. Do not solve repeated routing failures by telling the user to remember more commands.

The design target is simple: for common task types, the user should be able to ask for the work naturally and Claude should pull in the right workflow guidance without being hand-held.

### How to write a skill

A good skill is concise, narrow, and action-oriented. The official skills docs are right that routing quality depends heavily on the `description`, invocation settings, and tool boundaries. The changelog is also relevant here because it confirms skills can hot-reload and use `context: fork`, which makes them more practical for iterative team use.

Example:

```md
---
description: Use for any non-trivial feature or bugfix. Enforces audit -> refactor -> implement -> verify. Do not use for typo-only or formatting-only edits.
user-invocable: true
---

# Feature Workflow

1. Load the applicable convention skills before writing code.
2. Audit the touched code paths and identify the current canonical patterns.
3. Assume the area is not ready for the new feature until proven otherwise.
4. Refactor for readiness if abstractions, naming, structure, or tests are weak.
5. If introducing a new dependency, pattern, abstraction, or file structure, stop and ask for approval.
6. Implement only after the structure is coherent.
7. Run targeted verification.
8. If the task established a new approved convention, update the config files in the same change.
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

The official skill frontmatter supports this distinction. Use it. Keep the auto-routed surface small and sharp.

### What a "rule" should mean in practice

In this guide, a rule is not a random text file. A rule is a load-bearing instruction with a clear home:

- always-on and universal: root `CLAUDE.md`,
- scoped and discoverable: skill plus reference file,
- mechanically enforced: hook, script, or test.

That definition matters because it prevents the common failure where teams scatter instructions across markdown files and hope Claude infers the right one at the right time.

### Further reading

- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [I Compared gstack, Superpowers, and Compound Engineering](https://npomfret.github.io/reading-list-researcher/e502b3549aff92cb.html)
- [Superpowers repo](https://github.com/obra/superpowers)

## 4. Coding Conventions as Infrastructure

### Conventions are the anti-drift system

This is the most important section in the guide.

If Claude is sloppy by default, then coding conventions are not style preferences. They are the infrastructure that prevents sloppiness from accumulating. The source material points in this direction but usually stops short of saying it bluntly. The best-practices repo report, the Boris memory-system report, and the ecosystem tooling lists all imply that structure matters. This guide makes it explicit: conventions are the primary quality mechanism.

### What must be specified

Your convention system must cover every area where Claude can invent a new local solution:

- naming conventions,
- file and module layout,
- import direction and boundary rules,
- shared-vs-local abstraction rules,
- error handling,
- result and exception patterns,
- async flows, retries, cancellation, and concurrency,
- state management,
- data fetching patterns,
- API client and server shapes,
- validation,
- logging,
- test placement and style,
- mocking rules,
- migration patterns,
- generated-code boundaries,
- and any domain-specific invariants that must stay consistent.

If a topic can drift, it needs a convention.

### Global conventions vs module-local conventions

Not every convention belongs at the same scope.

Global conventions should cover:

- naming,
- import style,
- file organization,
- error handling shape,
- testing expectations,
- "never introduce without approval" rules.

Module-local conventions should cover:

- subsystem-specific patterns,
- local directory organization,
- local API shapes,
- frontend component state rules,
- database-access rules,
- and any place where the module really does work differently.

This is why the "router plus on-demand skills" model matters. Global guidance stays short. Local detail is discovered when relevant.

### Testing discipline and TDD

The guide already argues for aggressive verification. In practice, many teams should go one step further and encode test-first behavior for at least non-trivial feature work and reproducible bug fixes.

The strongest source here is the [Superpowers repo](https://github.com/obra/superpowers), which explicitly emphasizes true red/green TDD, plus the Boris memory-system report's emphasis on never trusting "done" without rigorous verification. That combination holds up well on long-running projects because it attacks two Claude failure modes at once:

- Claude writes plausible code before it has pinned down observable behavior.
- Claude declares success too early because the code "looks right."

If your team uses TDD, write it down as a convention, not a vibe. Be explicit about when it applies:

- bug fix: reproduce the bug with a failing test first whenever the failure is testable
- feature work: add or update the smallest failing test that proves the intended behavior
- implementation: write the minimum code to get green
- refactor: clean up only after green
- completion: do not claim success without showing the relevant tests passing

Do not claim to use TDD if the repo does not actually work that way. False TDD language is worse than no TDD language because Claude will satisfy the words and miss the real workflow. But when the team does want test-first discipline, Claude should be told exactly what that means.

### Recommended convention document format

Do not write vague convention prose. Write documents that make drift obvious.

Use this shape:

```md
# API Error Handling Convention

Applies to:
- `apps/api/**`

Canonical pattern:
- Services throw typed domain errors.
- Route handlers translate domain errors into HTTP responses.
- Shared helpers format error payloads.

Required rules:
- Do not return ad-hoc `{ ok: false }` objects from services.
- Do not map domain errors to HTTP inside lower-level helpers.
- New endpoints must use the shared error translator.

Forbidden alternatives:
- Inline `try/catch` response shaping in each route
- Mixed throw-and-return error signaling
- Custom error payload shapes per endpoint

If no existing pattern fits:
- Stop and ask for a convention decision before introducing a new one.
```

This format is strict for a reason. Claude needs to know:

- what the canonical pattern is,
- what alternatives are forbidden,
- and what to do when the convention does not cover the case.

### The "stop and ask" rule

This rule must be encoded in more than one place:

- in the root `CLAUDE.md`,
- in the feature-workflow skill,
- and in convention files themselves.

The language should be explicit:

> Never introduce a new dependency, design pattern, abstraction layer, file structure, naming convention, or solution style without explicit approval. If the codebase does not already establish a pattern for the problem, stop, describe the gap, propose options, and wait for a decision.

That is stronger than "prefer existing patterns." It forces Claude to notice conceptual expansion before it happens.

### Claude must check conventions before writing, not after

A convention system only works if Claude is instructed to locate the applicable convention set before editing. The official docs are good on skills and concise memory but do not make this workflow rigid enough for long projects. Your setup should.

The correct sequence is:

1. identify the touched subsystem,
2. load the applicable convention skills,
3. audit the existing code to confirm the canonical pattern,
4. write only after the pattern is clear.

Do not accept "Claude will pick it up from the code." Sometimes it will. Often it will infer the wrong local pattern from whichever files it happened to inspect first.

### How to establish a new convention

When Claude finds a real gap:

1. it stops,
2. it describes the missing decision,
3. the human chooses the pattern,
4. Claude updates the convention docs and, if needed, the relevant skill,
5. then Claude implements against the new standard.

This is the basis of a self-maintaining configuration system. The human owns the conceptual surface area. Claude owns keeping the recorded instructions up to date.

### How to audit an existing codebase for drift

Before trusting conventions, you may need to discover them the hard way.

Run focused audits by concern:

- error handling,
- API call structure,
- async orchestration,
- test layout,
- frontend state,
- service abstraction boundaries.

For each concern:

1. search the repo for all implementations,
2. cluster them into distinct patterns,
3. choose the canonical one,
4. refactor major divergences,
5. write the convention down,
6. then make future work follow it.

This is the only reliable way to stop a large existing project from getting worse.

### Further reading

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [Superpowers repo](https://github.com/obra/superpowers)

## 5. Task Planning and the Preparation Phase

### The default workflow must be audit -> refactor -> implement -> verify

Anthropic's [Best Practices](https://code.claude.com/docs/en/best-practices) and the Boris-related reports are right to emphasize planning, explicit success criteria, and verification. The missing hardening step is that planning must include readiness refactoring by default.

For any non-trivial task, Claude should follow this sequence:

1. **Audit**: inspect the existing code paths, abstractions, tests, and conventions.
2. **Refactor for readiness**: fix weak abstractions, duplication, naming drift, or structural issues first.
3. **Implement**: add the feature on top of the prepared structure.
4. **Verify**: run the targeted checks, inspect output, and confirm the task against the stated success criteria.
5. **Update config**: if the work established an approved new convention or clarified an existing one, update the Claude config in the same change.

That sequence should be the default behavior, not an occasional act of discipline.

### Refactor for readiness, not speculative architecture

There is an important boundary here. This guide argues for aggressive readiness refactoring before feature work. That is not permission for Claude to invent broad frameworks "for the future."

The right counterweight is the simplicity discipline emphasized in the [Superpowers repo](https://github.com/obra/superpowers) through YAGNI and in the Boris memory-system report through "avoiding over-engineering." These ideas complement each other rather than conflict:

- refactor what the current task needs in order to become coherent
- do not build abstractions for hypothetical future use cases
- do not introduce a framework when a local extraction will do
- do not create a general-purpose layer until there is real duplication or a second concrete use case

The distinction matters because Claude drifts in both directions. Sometimes it patches too little. Sometimes it generalizes too much. A good setup tells it to prepare the ground for the current task, not to redesign the entire area around imagined future requirements.

### Encode the workflow structurally

Do not rely on remembering to ask Claude nicely. Encode this workflow in:

- root `CLAUDE.md`,
- the feature-workflow skill,
- and review/check skills if you use them.

The feature-workflow skill should force Claude to answer these questions before implementation:

- What existing pattern is already present?
- Is the current structure ready for the change?
- What must be refactored first?
- What would count as introducing a new pattern?
- Which decisions require approval?
- What tests and checks define done?

### Preventing premature implementation

Claude skips straight to writing code when prompts are vague or time pressure is implied. Counteract that with prompt structure and skill structure.

Use prompts that demand the preparation phase explicitly:

> Add billing retries to the existing payment flow. First audit the current retry, error-handling, and job orchestration patterns. Assume the current structure may need refactoring before the feature. Stop and ask before introducing new libraries or abstractions. Only implement after explaining the readiness refactor and verification plan.

This is better than:

> add billing retries

The official docs are correct that specificity matters. For long projects, specificity must include readiness expectations.

### How Claude should communicate the plan

For any task that is not obviously tiny, Claude should present:

- the code paths inspected,
- the current pattern it found,
- the readiness problems,
- the proposed refactor boundary,
- the implementation steps,
- the verification steps,
- and any approval gates.

That is not bureaucracy. It is the mechanism that catches hidden drift before the edit starts.

### Handling unclear scope

When the task is underspecified, Claude must ask instead of assuming. The best-practices report is good on using questions up front. This guide sharpens the rule: if scope ambiguity could change abstractions, dependencies, or pattern choice, Claude should stop and clarify. It must not silently choose the smaller local solution just because it is available.

### Breaking large tasks into sub-tasks

Large tasks should be sequenced by dependency:

1. audit and readiness refactor,
2. core implementation,
3. verification and cleanup,
4. config updates,
5. optional follow-on refactors.

Do not split a large task into parallel sub-tasks until the ownership boundaries are explicit. Parallelism without boundaries is just faster drift.

### Further reading

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)
- [I Compared gstack, Superpowers, and Compound Engineering](https://npomfret.github.io/reading-list-researcher/e502b3549aff92cb.html)
- [Superpowers repo](https://github.com/obra/superpowers)

## 6. Self-Maintaining Configuration

### The configuration system should evolve with the codebase

This is an underused but high-leverage idea. Claude should not just consume `CLAUDE.md`, skills, agent definitions, and convention files. It should help maintain them.

The source material gets part of the way there. The Boris memory-system report points toward compounding institutional knowledge. The gstack/Superpowers/Compound comparison points toward knowledge compounding and reusable workflows. Anthropic's skills and memory systems provide the mechanisms. The missing operational guidance is this: configuration updates should be part of normal development, not deferred cleanup.

Treat this as controlled self-improvement. When Claude encounters a recurring failure mode, a missing convention, stale routing, or an unclear workflow, it should not just work around the problem for the current task. It should propose or make the smallest approved config improvement that helps it do a better job next time.

### What Claude can update autonomously

Claude can safely update:

- stale command lists,
- outdated file paths in skills,
- outdated file paths or descriptions in agent definitions,
- examples inside convention documents,
- wording improvements that do not change policy,
- notes that record a human-approved decision,
- skill and agent metadata that improves discoverability without changing policy,
- and the skill/reference links that keep the routing layer accurate.

### What requires human approval

Claude must ask before:

- creating a new top-level convention,
- changing the meaning of an existing convention,
- adding or removing a major skill family,
- adding a new agent role or changing the responsibility of an existing one,
- changing hook behavior in a way that affects workflow semantics,
- adding or removing an MCP server,
- introducing a new dependency,
- or changing architecture boundaries.

The rule is simple: Claude may maintain the map, but the human owns the territory.

### Add a config-maintenance skill

Create a dedicated skill for this workflow.

Example:

```md
---
description: Use when a task establishes or clarifies an approved convention, or when Claude config files appear stale or contradictory.
user-invocable: true
---

# Config Maintenance

1. Identify which instruction file is affected: root memory, skill, or reference doc.
2. If the issue is workflow routing, also check whether an agent definition or skill description should be updated.
3. Confirm whether the change is documentation-only or a policy change.
4. If it is a policy change, stop and ask for approval.
5. Update the smallest correct file.
6. Keep routing files short; move detail into reference docs.
7. Ensure the updated instructions match the codebase as it exists now.
```

This keeps the configuration system alive without turning every task into a documentation exercise.

### Version-control the config with the code

Do not treat Claude config as personal local clutter if the project is team-owned. Project-level skills, agent definitions, conventions, and root memory belong in version control so the codebase and the agent instructions evolve together.

The practical benefit is enormous:

- convention changes are reviewed,
- instruction drift is visible,
- and new contributors inherit the same operating model.

### Audit config for staleness

A stale convention file is worse than no convention file. It gives false confidence.

Use three triggers to audit config:

- after a refactor that changes the canonical pattern,
- when Claude repeatedly asks the same question,
- when the codebase starts diverging from written instructions.

If the docs say one thing and the code does another, resolve it immediately. Do not let the contradiction sit.

### Further reading

- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)
- [I Compared gstack, Superpowers, and Compound Engineering](https://npomfret.github.io/reading-list-researcher/e502b3549aff92cb.html)

## 7. MCP Strategy

### MCP is for missing system access, not for thinking

Anthropic's [MCP](https://code.claude.com/docs/en/mcp) docs frame MCP correctly: it gives Claude access to tools and systems. That is valuable. The common misuse is letting MCP stand in for analysis that should have happened in the code first.

High-value MCP categories in a coding workflow:

- version-accurate docs,
- database inspection,
- GitHub and CI/CD state,
- issue trackers,
- internal service APIs,
- and carefully chosen browser/runtime tools when source inspection is not enough.

Low-value or overused cases:

- opening a browser before reading the component code,
- querying external systems before confirming the repo cannot answer the question,
- attaching heavyweight tools to every session whether needed or not.

The hidden-features report and the dev-browser report are useful because they show what is possible. They are weak if read as "always use the browser." The Stitch workflow report is valid for design-heavy work because it attaches real design-system context, but it is still task-specific and expensive.

### The code-first rule

Encode this directly:

1. read the code,
2. read the tests,
3. read the config,
4. inspect local logs or outputs,
5. only then reach for MCP if the answer depends on external truth or runtime state you cannot infer locally.

This saves context, time, and confusion.

### When an MCP is justified

Use an MCP when at least one of these is true:

- the source of truth is outside the repo,
- you need live system state,
- you need version-accurate external documentation,
- or you need to perform a remote action that cannot be simulated locally.

If none of those are true, stay in the codebase.

### Scope MCP availability

Do not make every MCP globally available all the time just because it exists. If the environment allows it, keep the default MCP surface small and add specialized MCPs only for sessions that need them.

A practical baseline:

- always-on: only the few MCPs that represent common external truth,
- task-specific: browser, design, deployment, or rare internal systems.

The cost is not just latency. It is also conceptual distraction. Claude will use tools that exist.

### Further reading

- [Claude Code MCP](https://code.claude.com/docs/en/mcp)
- [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html)
- [Claude Code + Google Stitch 2.0](https://npomfret.github.io/reading-list-researcher/2aa3ed775a7a775e.html)
- [Sawyer Hood Announces dev-browser Hits 4k GitHub Stars](https://npomfret.github.io/reading-list-researcher/b4cbb3b860869a88.html)

## 8. Hooks Strategy

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

### Hooks are not the right place to block normal development actions

Some guides recommend blocking `rm`, blocking certain writes, or turning hooks into a safety cage. Reject that.

Files legitimately need to be deleted. Directories legitimately need to be replaced. Blocking common actions at the hook layer creates three bad outcomes:

1. Claude fights the environment instead of solving the task.
2. Humans start working around the hook system.
3. The real problem, poor instructions and poor task governance, remains unsolved.

If you do not trust Claude to delete a file safely, the answer is not a generic blocking hook. The answer is better task instructions, better approval policy, better sandboxing, and stricter operating rules.

### High-value hook examples

Good hooks for this setup:

- `SessionStart`: print a short reminder to use convention skills and the feature workflow for non-trivial changes.
- `PostToolUse`: append command and file-change logs to an audit file.
- post-edit side effect: run a targeted formatter or linter on touched files.
- `Stop`: write a short task summary or emit a notification.
- multi-agent events from recent changelog additions: record task completion or teammate idle state if you use multi-agent workflows.

Keep them fast, deterministic, and visible.

### Why blocking hooks are worse than they look

The official docs allow blocking. This guide still recommends against making that your primary safety mechanism. Operational safety surfaces are already complicated: permissions, sandboxing, hooks, and human approvals all interact. The more brittle blocking logic you add, the more likely you are to create false positives and developer friction while still not solving conceptual drift.

Use hooks to observe and enhance. Do not use them to paper over bad governance.

### Further reading

- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html)

## 9. Parallelisation

### Use multiple clones for interactive work

The CLI and ecosystem sources surface worktrees and high-throughput parallel setups. This guide rejects worktrees as the default for interactive Claude Code use. Even though the CLI supports `--worktree`, and even though some public workflows use worktrees heavily, multiple clones are a better default for most teams.

Why:

- each clone is a fully independent workspace,
- the mental model is simple,
- there is less git-specific subtlety,
- and it is easier to reason about one Claude session per directory.

Set up sibling clones like this:

```text
~/src/project-main
~/src/project-api-refactor
~/src/project-ui-fix
~/src/project-review
```

One clone, one task, one Claude session.

### How to manage multiple clones

Use this operating model:

1. keep one integration clone as the main line,
2. create a new clone only for genuinely independent work,
3. assign explicit ownership per clone,
4. fetch and rebase or merge frequently from the main line,
5. never let two clones edit the same conceptual area without deciding ownership first.

The public throughput stories about running many Claude sessions are directionally useful. The part to keep is specialization and separation. The part to reject is unnecessary git cleverness.

### How to avoid merge pain

Parallelism is only worth the overhead when tasks are actually separable.

Good candidates:

- isolated subsystems,
- long-running investigations,
- one implementation plus one read-only review thread,
- large refactors with clearly divided modules.

Bad candidates:

- multiple sessions editing the same feature area,
- small changes that would finish before synchronization overhead pays off,
- tasks that depend on constant shared reasoning across the same files.

Parallelism without discipline produces parallel drift.

### Further reading

- [Claude Code Commands](https://code.claude.com/docs/en/commands)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html)

## 10. Anti-Patterns

### 1. Bloating `CLAUDE.md`

This is the most common large-project mistake. It comes from good intentions and bad scale assumptions. Official docs and the stronger research sources support concise memory. Community interpretations frequently break that advice by treating root memory as a dumping ground. Do not do that.

### 2. Using hooks to block normal commands

This is brittle, annoying, and conceptually lazy. Hooks are for observation and enhancement. Blocking should be rare and narrow, not the main safety mechanism.

### 3. Treating every markdown file as a "rule"

Scattered markdown is not a rules system. If Claude cannot reliably know when to load the instruction, it is just documentation. Use the explicit layer model: root memory, skills, references, hooks, tooling.

### 4. Letting Claude implement before it audits

This is how minimum-change debt accumulates. Audit first. Refactor for readiness second. Implement third. Anything else is just faster decay.

### 5. Soft conventions

Statements like "prefer consistency" or "reuse existing patterns when possible" are not enough. Conventions must say what the canonical pattern is, what alternatives are forbidden, and when Claude must stop and ask.

### 6. Silent introduction of new dependencies or abstractions

This is one of the main ways long-lived codebases become conceptually bloated. Every new dependency and abstraction expands the maintenance surface. Claude must never do this silently.

### 7. Speculative abstractions and premature frameworks

This is the mirror-image failure mode of minimum-change patching. Claude sees a messy area and jumps straight to a big reusable system that the current codebase has not earned yet. That is how you get abstraction layers with one caller, configuration systems for one case, and framework-shaped code around a single current requirement. Refactor for readiness. Do not build for imaginary future complexity.

### 8. Indiscriminate MCP use

If Claude can answer from code and tests, it should stay there. Browser-first and tool-first debugging are easy ways to waste context and time.

### 9. Treating repo or tool lists as architecture guidance

The ecosystem curation posts are useful for discovering tools. They are not substitutes for a setup philosophy. A list of five good repos does not tell you how to govern a growing codebase.

### 10. Using worktrees and `--bare` as the default interactive model

Those may be useful for specific power-user flows. This guide is not about batch pipelines or agent farms. For interactive coding on long-running projects, multiple clones are simpler and more robust.

### 11. Keeping one endless session alive

The official docs are right about compaction, reset, and session management. Long sessions rot. Use fresh sessions, clear boundaries, and explicit workflow skills instead of trying to carry all project state in one conversation forever.

### Further reading

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html)
- [Superpowers repo](https://github.com/obra/superpowers)

## Recommended Baseline

If you want a practical default setup, use this:

1. A short root `CLAUDE.md` that encodes non-negotiables and routes to deeper guidance.
2. A small skill set:
   - `conventions-global`
   - `feature-workflow`
   - one skill per subsystem with genuinely distinct conventions
   - `config-maintenance`
3. Reference documents for detailed conventions, kept outside root memory.
4. Hooks for audit logs, lightweight reminders, notifications, and targeted side effects.
5. A code-first MCP policy.
6. A hard stop-and-ask rule for any new dependency, pattern, abstraction, or convention gap.
7. Multiple clones, not worktrees, when parallel sessions are justified.

That is the world-class setup for long-running projects: not the most feature-rich setup, not the cleverest setup, and not the most impressive screenshot. The best setup is the one that keeps Claude useful while making drift, duplication, and sloppy local choices hard to introduce.

## Primary Sources

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code Commands](https://code.claude.com/docs/en/commands)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Code Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Claude Code CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [Someone Finally Documented How to Actually Use Claude Code](https://npomfret.github.io/reading-list-researcher/e3f151d5e47a73c7.html)
- [How Boris Cherny Trains AI at Anthropic: The CLAUDE.md Memory System](https://npomfret.github.io/reading-list-researcher/c641144769e8837a.html)
- [15 Hidden and Under-Utilized Features in Claude Code](https://npomfret.github.io/reading-list-researcher/08a08a991d5e8161.html)
- [I Compared gstack, Superpowers, and Compound Engineering](https://npomfret.github.io/reading-list-researcher/e502b3549aff92cb.html)
- [Best GitHub Repos for Claude Code That Will 10x Your Next Project in 2026](https://npomfret.github.io/reading-list-researcher/e6f4d1afbd729e56.html)
- [Best GitHub repos for Claude code that will 10x your next project](https://npomfret.github.io/reading-list-researcher/5187cc080607fbc8.html)
- [Best GitHub repos for Claude Code that will 10x your next project](https://npomfret.github.io/reading-list-researcher/c1aa58026580b10e.html)
- [Claude Code + Google Stitch 2.0](https://npomfret.github.io/reading-list-researcher/2aa3ed775a7a775e.html)
- [Sawyer Hood Announces dev-browser Hits 4k GitHub Stars](https://npomfret.github.io/reading-list-researcher/b4cbb3b860869a88.html)
- [Superpowers repo](https://github.com/obra/superpowers)

Use the official docs to verify what Claude Code supports. Use the research reports to understand how people are actually using it. Trust neither blindly.
