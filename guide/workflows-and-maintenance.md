# Workflows and Configuration Maintenance

### The default workflow must be audit -> refactor -> implement -> verify

Anthropic's [Best Practices](https://code.claude.com/docs/en/best-practices) emphasizes planning, explicit success criteria, and verification. Add readiness refactoring: frame the task as a verifiable outcome, audit first, prepare the structure, implement, and verify.

For any non-trivial task, Claude should follow this sequence:

1. **Audit**: inspect the existing code paths, abstractions, tests, conventions, and external-system boundaries.
2. **Refactor for readiness**: fix weak abstractions, duplication, naming drift, helper creep, leaking provider contracts, or structural issues first.
3. **Implement**: add the feature on top of the prepared structure.
4. **Verify**: run the targeted checks, inspect output, and confirm the task against the stated success criteria.
5. **Update config**: if the work established an approved new convention or clarified an existing one, update the Claude config in the same change.

The smallest diff is not the goal. Determine whether the codebase is ready, make the necessary structural changes, and then add behavior.

That includes resisting the lazy helper patch. If the first idea is "add another helper," Claude should stop and ask whether the real problem is missing encapsulation or an abstraction that needs to be reshaped first.

### Lead with the ideal solution

For every non-trivial change, Claude should deliberately answer this design question before proposing an implementation:

> If I were starting this area from scratch, knowing what I know now, what would the best approach be?

This question exposes the ownership boundary, abstraction, state model, interface, and tests the feature needs. Refactor toward that shape only as far as the task requires.

When there is more than one viable approach, present the ideal solution first even if it is larger, more difficult, or needs approval. Then, if useful, present a lower-effort alternative with an explicit label such as “compromise” or “temporary path,” together with the debt, constraints, and future cleanup it creates. Claude must not lead with the shortest patch merely because it is easier to implement.

The audit step needs to be more explicit than "read the file you plan to edit." Claude should be instructed to search for nearby and non-local precedent before touching code:

- look upstream at who calls the code, constructs the object, or depends on the interface
- look downstream at implementations, side effects, persistence, transport, and consumers
- look laterally for similarly named files, classes, functions, hooks, services, handlers, and tests
- search for existing patterns, field names, error shapes, and helper usage across the repo before inventing a new variant
- for every external API, SDK, or service touched, trace where its client and types enter the application, whether an application-owned adapter already exists, and how many callers would change if the provider were replaced

Without this audit, Claude often optimizes for a local patch and reinvents repository patterns.

### Refactor for readiness, not speculative architecture

Readiness refactoring does not permit speculative frameworks. Apply YAGNI: prepare for the current requirement without designing for hypothetical ones.

- refactor what the current task needs in order to become coherent
- do not build abstractions for hypothetical future use cases
- do not introduce a framework when a local extraction will do
- do not create a general-purpose layer until there is real duplication or a second concrete use case
- do isolate an external system on first use; its independently changing contract, failure modes, and need for test substitution are already concrete reasons for a boundary

Prepare for the current task without either patching too little or generalizing beyond evidence.

### Preventing premature implementation

Claude skips straight to writing code when prompts are vague or time pressure is implied. Counteract that with prompt structure and skill structure.

Use prompts that demand the preparation phase explicitly:

> Add retry behavior to the existing workflow. First audit the current retry, error-handling, and orchestration patterns. Assume the current structure may need refactoring before the feature. Stop and ask before introducing new libraries or abstractions. Only implement after explaining the readiness refactor and verification plan.

Avoid vague prompts such as:

> add retries

For non-trivial work, specify readiness expectations and success criteria:

- describe the intended outcome
- require the audit
- require the readiness refactor if needed
- define the checks that prove the task is complete

Design-system migrations need a specialized version of this workflow because component convergence must precede the broad token sweep, and provably neutral substitutions should be separated from visible design decisions. See [Design-System Refactors](design-system-refactors.md) for that sequencing and its verification model.

### Progressively disclose manual verification

When verification requires several manual checks, do not present every detailed procedure at once. That transfers sequencing, memory, and reporting work to the user.

Instead, Claude should first make the scope visible: state how many checks are needed and give a brief, one-line inventory of them. Then guide the user through one check at a time. Explain only the steps for the current check, ask for the result or reaction, and use that feedback before moving to the next one.

A good interaction looks like this:

> I need your help checking eight things. They cover sign-in, navigation, form validation, saving, error handling, responsive layout, keyboard use, and the final confirmation state.
>
> Let's start with sign-in. Open the login page, sign in with your test account, and tell me whether you reach the dashboard without seeing an error or unexpected delay.

This shows the scope without overwhelming the user. Keep a short progress record when checks are independent. Provide the full checklist up front only when requested or needed for delegation or independent execution.

## Configuration Maintenance

### The configuration system should evolve with the codebase

Claude should maintain `CLAUDE.md`, skills, agent definitions, and convention files as part of normal development.

When Claude finds a recurring failure, missing convention, broken discovery path, or unclear workflow, it should propose or make the smallest approved configuration improvement.

### What Claude can update autonomously

Claude can safely update:

- stale command lists,
- outdated file paths in skills,
- outdated file paths or descriptions in agent definitions,
- examples inside convention documents,
- wording improvements that do not change policy,
- notes that record a human-approved decision,
- skill and agent metadata that improves discoverability without changing policy,
- and references owned by skills or rules that keep their on-demand guidance accurate.

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

Claude may maintain established configuration; humans own its policy.

### Add a config-maintenance skill

Create a dedicated skill for this workflow.

Example:

```md
---
description: Use when a task establishes or clarifies an approved convention, or when Claude config files appear stale or contradictory.
user-invocable: true
---

# Config Maintenance

1. Identify which instruction surface owns the guidance: root `CLAUDE.md`, rule, skill, agent definition, or supporting reference.
2. If the issue is workflow routing, also check whether an agent definition or skill description should be updated.
3. Confirm whether the change is documentation-only or a policy change.
4. If it is a policy change, stop and ask for approval.
5. Update the smallest correct file.
6. Keep always-loaded instructions short; keep scoped detail with the rule or skill that owns it.
7. Ensure the updated instructions match the codebase.
```

This maintains configuration without turning every task into documentation work.

### Version-control the config with the code

Do not treat Claude config as personal local clutter if the project is team-owned. Project-level skills, agent definitions, conventions, and root `CLAUDE.md` instructions belong in version control so the codebase and the agent instructions evolve together.

Benefits:

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
