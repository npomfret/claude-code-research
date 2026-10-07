# Testing and Quality

Tests should express enduring behaviour and provide trustworthy evidence at reasonable cost. Use the repository's chosen test-first discipline, readable scenarios, and controlled debugging experiments.

## Test-first discipline

Where the team uses TDD:

1. Reproduce a testable bug or specify a feature with the smallest practical failing test.
2. Implement enough to pass.
3. Refactor after green.
4. Show relevant passing checks before claiming the behaviour works.
5. TDD is hard. Don't get hung up on it.

**Don't:** claim TDD for a workflow that does not follow it, or automatically retain every diagnostic reproducer.

Run fast targeted tests before small commits. Use the continuous integration server for full builds; report pending or failed CI separately from successful local checks. See the [Git workflow](workflows-and-maintenance.md#git-workflow).

## Readable behavioural tests

The test owns the meaningful starting state, action, and expectations. **Application drivers** own the mechanics of controlling and observing the application. Page Objects, robots, harnesses, builders, and API test clients can serve that boundary.

A test that is more complex than the code it is testing is simply not valid.


**Do:**

- A project should define "types" of tests (unit, integration etc) and clearly define the difference.
- Your build or test suite should enforce strict time limits which can be per test or more wide-ranging.
- Write integration and end-to-end scenarios in application or domain language.
- Keep scenario-defining values, important preconditions, intermediate checks, and final expectations visible.
- Use blank lines to make the stages of a test case (setup, sanity check, execute, verify etc) clear and separated
- Extract non-trivial setup, navigation, request construction, selectors, synchronization, diagnostics, and cleanup into focused drivers.
- Give test drivers intent-based operations, deterministic waiting, useful failure evidence, and isolated lifetimes.
- Test drivers should abstract some of the details of the underlying system, like waiting for asynchronous behavior to complete.
- Improve the driver before adding a scenario that would otherwise accumulate procedural machinery.
- Keep expected results independent of the production calculation under test.
- "Helpers" are an antipattern - instead, endevour to model state and behaviour.

**Don't:**

- Create a one-for-one wrapper around selectors or SDK calls, or one god-driver for the whole product.
- Hide intent in global fixtures or generic helpers.
- Spread sleeps, polling, screenshots, and retries through test bodies (use a driver).
- Introduce shared mutable state, execution-order dependence, or hidden cleanup.
- Extract every literal or add ceremony to a simple unit test.

**Example test:** the driver names and scenario are illustrative.

```ts
const account = await accounts.create({ plan: "trial" });
await app.signIn(account);
await dashboard.expectReady();

await subscriptions.cancelCurrent();

await subscriptions.expectCancelled();
```

A product-aware reader should understand this scenario without knowing its framework or application plumbing. Names and spacing should make phases clear without `Arrange`/`Act`/`Assert` narration.

## Controlled bug investigation

Each speculative change is a reversible experiment against a known baseline.

1. Record the working tree and preserve pre-existing user changes.
2. Define the observed failure and smallest reliable reproducer.
3. State one hypothesis and the observation that would support or falsify it.
4. Make one minimal reversible experiment and run the reproducer.
5. Retain only changes producing predicted or clearly useful evidence.
6. Reverse negative, irrelevant, or ambiguous experiments completely before the next hypothesis.
7. Inspect the diff to confirm it contains only the baseline and justified changes.
8. Verify the supported fix and remove temporary instrumentation without a deliberate final role.

**Don't:** stack failed conjectures, call ambiguity success, or use a broad reset that discards unrelated work. This applies to logging, probes, flags, timing changes, speculative fixes, and test modifications alike.

## Reproducer retention

A reproducer supports diagnosis; a permanent test protects an enduring contract or material risk. Decide after the fix:

| Decision | When |
| --- | --- |
| Keep or rewrite | It covers a meaningful boundary, invariant, transition, or failure risk at acceptable cost. |
| Consolidate | A clearer, cheaper, or better-layered test can protect the same contract. |
| Remove | It records obsolete implementation, duplicates stronger coverage, protects an impossible or retired failure, is unstable, or costs more than its remaining value. |

Consider runtime, fixtures, flakiness, maintenance, cognitive load, and resistance to legitimate refactoring. Maintain the suite as code: proportionately refactor, consolidate, or remove nearby tests while preserving valuable coverage.

**Do:** substantiate claimed redundancy where practical with a targeted mutation or temporary reintroduction of the fault that makes the retained suite fail.

**Don't:** keep a test solely because it found a bug, or delete it solely because the bug is fixed.

## Automatically routed skills

Use task-matching descriptions and local scope so guidance loads before tests or experiments are written. Path-scoped test rules may add concise local policy without copying the whole skill.

### Testing conventions

**Example skill:** adapt its triggers and process to the repository’s testing conventions.

```md
---
description: Use when creating, changing, reviewing, or refactoring tests and test infrastructure. Excludes running an unchanged suite.
---

# Testing Conventions

1. Identify the behaviour and test layer; inspect canonical tests and drivers.
2. Keep scenario state, actions, and expectations visible and independent.
3. Put non-trivial application-control mechanics behind focused, isolated drivers.
4. Refactor test infrastructure for readability before adding procedural scenarios.
5. Maintain nearby tests proportionately without losing valuable protection.
6. Run appropriate checks and inspect failure diagnostics.
```

### Bug investigation

**Example skill:** adapt the triggers and verification steps to the repository.

```md
---
description: Use when investigating, reproducing, diagnosing, or fixing observed bugs, intermittent failures, unexplained runtime behaviour, or failing tests. Excludes feature requests without an observed failure.
---

# Bug Investigation

1. Load subsystem and testing conventions; inspect code, tests, and logs.
2. Define the failure and add a small failing test when practical.
3. Preserve the baseline and test one explicit hypothesis at a time.
4. Retain useful evidence; reverse unsupported experiments before continuing.
5. Implement the supported fix and verify the failure and nearby behaviour.
6. Remove temporary scaffolding and decide the reproducer's permanent role.
7. Report diagnostic evidence, verification, test retention, and uncertainty.
```

## Convention documents and approval

Each convention needs scope, a canonical pattern, required rules, forbidden alternatives, and a procedure for gaps.

**Example convention:** the error policy below illustrates the document shape; use the repository’s approved error model.

```md
# Subsystem Error Handling

Applies to: `<paths>`

Do:
- Throw typed domain errors from services.
- Translate them at boundary handlers using the shared error translator.

Don't:
- Return ad-hoc error objects or invent endpoint-specific payloads.
- Mix throw-and-return signalling or map transport responses in lower layers.

If no pattern fits: describe the gap, propose options, and wait for a decision.
```

Give the repository's approval policy one authoritative home at its broadest applicable scope. The proposed policy is to ask before new dependencies, design patterns, abstraction layers, file structures, naming conventions, or solution styles. When a gap appears, explain it, present options, obtain the human decision, record it, and then implement.

**Do:** load conventions before editing, inspect code to confirm the canonical pattern, and update approved decisions and routing together.

**Don't:** duplicate approval policy across configuration for emphasis, rely on nearby code alone to establish precedent, or use a root configuration index to conceal routing defects.

## Drift audits

Audit by concern: errors, APIs, provider leakage, async orchestration, tests, frontend state, semantic naming, design-system coverage, and ownership boundaries.

1. Find all in-scope implementations and group distinct patterns.
2. Establish the canonical pattern through the appropriate decision process.
3. Refactor approved divergences and record the convention.
4. Make future work encounter it and enforce deterministic rules mechanically.

**External integrations:** inspect vendor imports and types, adapter error/protocol mapping, application fakes, adapter tests, and the file set a provider replacement would affect. Injection alone does not establish isolation.

**UI:** count containers, panels, typography, spacing, borders, surfaces, icons, controls, colours, and loading/empty/error paths. Distinguish raw values, centralised constants, semantic tokens, and shared components. Names must carry stable intent.

Use [UI and UX Audits](ui-ux-audits.md) for read-only runtime and accessibility assessment, and [Design-System Refactors](design-system-refactors.md) for migration baselines and separate build, behavioural, and visual confidence. State the coverage of any sample explicitly.
