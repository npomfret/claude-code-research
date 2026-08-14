# Workflows and Configuration Maintenance

### The default workflow must be audit -> refactor -> implement -> verify

Anthropic's [Best Practices](https://code.claude.com/docs/en/best-practices) rightly emphasizes planning, explicit success criteria, and verification. The missing hardening step is that planning must include readiness refactoring by default.

Explicit success criteria are a force multiplier. Claude generally performs better when the task is framed as a verifiable outcome instead of a vague imperative. Success criteria alone do not guarantee a coherent implementation, however. For long-lived projects, combine them with the stronger workflow in this guide: audit first, refactor for readiness second, implement third, then verify against explicit criteria.

For any non-trivial task, Claude should follow this sequence:

1. **Audit**: inspect the existing code paths, abstractions, tests, and conventions.
2. **Refactor for readiness**: fix weak abstractions, duplication, naming drift, helper creep, or structural issues first.
3. **Implement**: add the feature on top of the prepared structure.
4. **Verify**: run the targeted checks, inspect output, and confirm the task against the stated success criteria.
5. **Update config**: if the work established an approved new convention or clarified an existing one, update the Claude config in the same change.

That sequence should be the default behavior, not an occasional act of discipline.

The key point is that "working with the smallest diff" is not the goal. Claude should assume the codebase is probably not ready for the new feature, decide what preparation is needed, do that refactor, and only then add behavior. Otherwise it will force the feature through the current shape and leave the area worse than it found it.

That includes resisting the lazy helper patch. If the first idea is "add another helper," Claude should stop and ask whether the real problem is missing encapsulation or an abstraction that needs to be reshaped first.

### Lead with the ideal solution

For every non-trivial change, Claude should deliberately answer this design question before proposing an implementation:

> If I were starting this area from scratch, knowing what I know now, what would the best approach be?

The answer is not a license to rewrite the world. It is a way to expose the shape the current feature actually wants: the right ownership boundary, a clear abstraction, encapsulated state, a coherent interface, and tests that describe the intended contract. Claude should then prepare the existing code toward that shape — refactor, extract, and encapsulate as far as the current task requires — before adding the new behavior.

When there is more than one viable approach, present the ideal solution first even if it is larger, more difficult, or needs approval. Then, if useful, present a lower-effort alternative with an explicit label such as “compromise” or “temporary path,” together with the debt, constraints, and future cleanup it creates. Claude must not lead with the shortest patch merely because it is easier to implement.

The audit step needs to be more explicit than "read the file you plan to edit." Claude should be instructed to search for nearby and non-local precedent before touching code:

- look upstream at who calls the code, constructs the object, or depends on the interface
- look downstream at implementations, side effects, persistence, transport, and consumers
- look laterally for similarly named files, classes, functions, hooks, services, handlers, and tests
- search for existing patterns, field names, error shapes, and helper usage across the repo before inventing a new variant

If this is not spelled out, Claude will often optimize for the local patch instead of the repository pattern. That is the mechanism behind most wheel-reinvention and a large share of long-term drift.

### Refactor for readiness, not speculative architecture

There is an important boundary here. This guide argues for aggressive readiness refactoring before feature work. That is not permission for Claude to invent broad frameworks "for the future."

The right counterweight is YAGNI: prepare the code for the current requirement without designing for hypothetical ones.

- refactor what the current task needs in order to become coherent
- do not build abstractions for hypothetical future use cases
- do not introduce a framework when a local extraction will do
- do not create a general-purpose layer until there is real duplication or a second concrete use case

The distinction matters because Claude drifts in both directions. Sometimes it patches too little. Sometimes it generalizes too much. A good setup tells it to prepare the ground for the current task, not to redesign the entire area around imagined future requirements.

### Preventing premature implementation

Claude skips straight to writing code when prompts are vague or time pressure is implied. Counteract that with prompt structure and skill structure.

Use prompts that demand the preparation phase explicitly:

> Add retry behavior to the existing workflow. First audit the current retry, error-handling, and orchestration patterns. Assume the current structure may need refactoring before the feature. Stop and ask before introducing new libraries or abstractions. Only implement after explaining the readiness refactor and verification plan.

This is better than:

> add retries

The official docs are correct that specificity matters. For long projects, specificity must include readiness expectations.

It should also include success criteria. In practice, the strongest prompt shape for non-trivial work is:

- describe the intended outcome
- require the audit
- require the readiness refactor if needed
- define the checks that prove the task is complete

That gives Claude the benefits of goal-driven execution without collapsing into smallest-diff thinking.

### Progressively disclose manual verification

When verification requires the user to perform several manual checks, Claude should not dump the complete procedure for every check into one message. That may be comprehensive, but it transfers the burden of sequencing, remembering, and reporting the work to the user.

Instead, Claude should first make the scope visible: state how many checks are needed and give a brief, one-line inventory of them. Then guide the user through one check at a time. Explain only the steps for the current check, ask for the result or reaction, and use that feedback before moving to the next one.

A good interaction looks like this:

> I need your help checking eight things. They cover sign-in, navigation, form validation, saving, error handling, responsive layout, keyboard use, and the final confirmation state.
>
> Let's start with sign-in. Open the login page, sign in with your test account, and tell me whether you reach the dashboard without seeing an error or unexpected delay.

This gives the user advance warning about the size and shape of the task without confronting them with eight sets of instructions at once. If the checks are independent, Claude can keep a short visible progress record as it goes. It should provide the full checklist up front only when the user asks for it, needs to delegate it, or explicitly wants to run the checks independently.

## Configuration Maintenance

### The configuration system should evolve with the codebase

This is an underused but high-leverage idea. Claude should not just consume `CLAUDE.md`, skills, agent definitions, and convention files. It should help maintain them.

Anthropic's skills and memory systems provide the mechanisms for durable repository guidance. The missing operational guidance is this: configuration updates should be part of normal development, not deferred cleanup.

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

