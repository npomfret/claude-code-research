# Testing and Quality

## Use Test-First Discipline Deliberately

The guide already argues for aggressive verification. In practice, many teams should go one step further and encode test-first behavior for at least non-trivial feature work and reproducible bug fixes.

Test-first work is useful here because it attacks two Claude failure modes at once:

- Claude writes plausible code before it has pinned down observable behavior.
- Claude declares success too early because the code "looks right."

If your team uses TDD, write it down as a convention, not a vibe. Be explicit about when it applies:

- bug fix: reproduce the bug with the smallest practical failing test first whenever the failure is testable; decide separately whether that reproducer belongs in the permanent suite
- feature work: add or update the smallest failing test that proves the intended behavior
- implementation: write the minimum code to get green
- refactor: clean up only after green
- completion: do not claim success without showing the relevant tests passing

Do not claim to use TDD if the repo does not actually work that way. False TDD language is worse than no TDD language because Claude will satisfy the words and miss the real workflow. But when the team does want test-first discipline, Claude should be told exactly what that means.

## Tests Must Make Behaviour Easy to Read

A test is useful only when a reader can quickly identify the meaningful starting state, the action, and the expected result. Where an intermediate state matters, make that verification equally visible. The setup, exercise, and verification phases should be apparent from the semantic operations and spacing in the test; they should not need `Arrange`, `Act`, and `Assert` narration to become understandable.

This matters most in integration, system, and end-to-end tests. Such tests may need substantial machinery to launch the application, seed data, authenticate, navigate, locate controls, construct requests, wait for asynchronous work, observe state, capture diagnostics, and clean up. Those mechanics are necessary, but they are not the behaviour the test is trying to communicate. Hundreds of lines of procedural setup in a test class are evidence of missing test infrastructure, not an unavoidable property of comprehensive testing.

Use **application drivers** as the generic name for abstractions that control and observe the system under test. Depending on the layer and local vocabulary, a project may call them test harnesses, test DSLs, Page Objects, component objects, robots, fixture builders, API test clients, or Screenplay actors and tasks. The name is less important than the ownership boundary:

> The test owns the scenario, behaviourally meaningful inputs, actions, and expectations. Drivers own the mechanics required to control and observe the application.

Required rules:

- Integration and end-to-end test bodies must read in application or domain language rather than transport, framework, selector, or infrastructure language.
- Extract non-trivial application control, environment setup, navigation, request construction, synchronization, observation, and cleanup mechanics from the test class into focused drivers or equivalent harness abstractions.
- Keep values and state transitions that explain the scenario visible in the test. Do not hide the reason for the test inside a generic fixture, global setup hook, or opaque helper.
- Make important precondition checks and final expectations explicit in the scenario, either as ordinary assertions over values returned by a driver or as clearly named semantic verification operations.
- Give drivers intent-based operations. Do not create a thin layer that merely renames selectors, button presses, HTTP endpoints, framework calls, or vendor SDK methods one for one.
- Prefer several cohesive capability or surface drivers over one god-driver that knows the entire product.
- Centralize deterministic synchronization and useful failure diagnostics in the driver. Do not spread arbitrary sleeps, polling loops, screenshot plumbing, or retry behaviour through test bodies.
- Do not calculate expected results with the same production logic being tested. A driver may observe and translate application state, but the test's expectation must remain independently meaningful.
- Give each test an isolated, explicit starting state. Driver reuse must not introduce shared mutable state, order dependence, or hidden cleanup requirements.
- When adding a test exposes substantial procedural code in the test class, perform a readiness refactor: extract or improve the driver first, then write the short behavioural scenario against it.

For example, prefer a test shaped like this:

```ts
const account = await accounts.create({ plan: "trial" });
await app.signIn(account);
await dashboard.expectReady();

await subscriptions.cancelCurrent();

await subscriptions.expectCancelled();
```

The driver layer owns how accounts are created, how sign-in and navigation work, which controls or endpoints implement cancellation, how readiness is detected, and which diagnostics accompany a failure. The test retains the facts that matter to the scenario: the account plan, the action, the meaningful intermediate state, and the expected outcome.

Do not extract every literal or create ceremony around a small unit test. The boundary is mechanical application-control knowledge: once that knowledge is non-trivial, repeated, or capable of obscuring the scenario, it does not belong in the test class. The acceptance test for the abstraction is whether a product-aware reader can understand the behaviour without first understanding the test framework or application plumbing.

## Bug Investigation Is a Sequence of Controlled Experiments

Claude's normal instinct during a difficult investigation is often reasonable: form a conjecture, add instrumentation or a speculative fix, run the reproducer, and inspect what happens. The damaging failure comes after an unsuccessful attempt. Claude frequently leaves that change in place, adds another experiment on top, and eventually reasons from a working tree containing several unsupported ideas. Later observations no longer have a clean baseline, and an apparent fix may depend accidentally on abandoned code.

Treat every speculative change as a reversible experiment:

1. Record the current working-tree baseline and preserve any pre-existing human changes.
2. State one hypothesis and the observation that would support or falsify it.
3. Make the smallest reversible change needed to test it. This includes temporary logging, probes, feature flags, timing changes, test modifications, and speculative fixes.
4. Run the specific reproducer and classify the result.
5. If the attempt produces the predicted result or clearly identified useful evidence, retain only the changes whose value is now understood and continue deliberately.
6. If the result is negative, irrelevant, or ambiguous, completely reverse that experiment before testing the next hypothesis.
7. Inspect the diff before the next attempt and confirm that it contains only the original baseline plus changes justified by evidence.

Ambiguity is not success. If Claude cannot name the useful evidence an experiment produced, the experiment failed and must be removed. Never stack a new conjecture on top of an unsuccessful one. Reverse only the current experiment; do not use a broad reset or checkout that would discard the user's existing work.

Temporary instrumentation is still temporary even when it helps locate the bug. Remove it when the investigation no longer needs it unless the team deliberately chooses to retain it as supported product instrumentation. The final change should contain the fix and the deliberately selected verification, not the archaeological remains of the search.

## A Bug Reproducer Is Not Automatically a Permanent Test

A failing reproducer and a permanent regression test serve different purposes. The reproducer is diagnostic scaffolding: it proves the failure, constrains the investigation, and demonstrates that the proposed fix changes the observed outcome. A permanent test is executable specification: it protects an enduring behavioural contract or material risk at an acceptable ongoing cost.

After the fix, deliberately retain, rewrite, consolidate, or remove the reproducer:

- **Keep or rewrite it** when the bug exposed a meaningful previously uncovered boundary, invariant, state transition, interaction, or failure mode whose regression risk justifies permanent coverage.
- **Consolidate it** when an existing or more canonical test can express the same contract more clearly, cheaply, or at a more appropriate layer.
- **Remove it** when it merely records the old implementation, duplicates stronger coverage, protects behaviour that is no longer required, exercises a failure class the new design makes impossible, is unstable, or costs more to run and maintain than the remaining risk warrants.

Test cost includes execution time, fixture and environment setup, flakiness, maintenance, cognitive load, and unnecessary resistance to legitimate refactoring. A bug having occurred once is evidence about risk, not an automatic claim on permanent suite capacity.

Treat the test suite as maintained code, not append-only history. When working in a test area, refactor, rewrite, consolidate, or remove nearby tests when doing so improves clarity, eliminates redundant or obsolete coverage, reduces unjustified cost, or better expresses the enduring behavioural contract. Keep this cleanup proportionate to the task and preserve coverage whose value still exceeds its cost.

When removing a reproducer because other tests already protect the behaviour, verify that claim where practical: temporarily reintroduce the faulty behaviour or make an equivalent targeted mutation and confirm that the retained suite fails. This mutation check is strong evidence of redundancy, not a universal rule that every historical bug must retain equivalent coverage forever.

Do not keep a test merely because it helped find a bug, and do not remove it merely because the bug is fixed. Keep the smallest, clearest set of tests whose enduring behavioural and risk-reduction value exceeds their continuing cost.

## Recommended Automatically Routed Skills

The principles above will not reliably affect Claude merely because they exist in a project document. Make them load before tests or experiments are written by using model-invocable skills whose descriptions match ordinary requests. A path-scoped rule can additionally enforce the key test-shape requirements for the repository's integration and end-to-end test directories, but should contain only the concise local policy rather than duplicate the full skill.

### Testing Conventions

```md
---
description: Use automatically when creating, changing, reviewing, or refactoring unit, integration, system, UI, or end-to-end tests and test infrastructure. Enforces readable behavioural scenarios, focused application drivers, test isolation, and deliberate test-suite maintenance. Do not use when only running an unchanged test suite.
---

# Testing Conventions

1. Identify the behaviour, boundary, and appropriate test layer before writing the test.
2. Inspect nearby tests, fixtures, drivers, harnesses, Page Objects, builders, and canonical assertion patterns.
3. Make the test body express the meaningful starting state, action, and expectation in application or domain language. Keep important intermediate verification explicit.
4. Put non-trivial setup, application control, navigation, transport, selectors, synchronization, observation, diagnostics, and cleanup behind focused drivers or equivalent harness abstractions. Do not accumulate those mechanics in the test class.
5. Keep scenario-defining values visible. Do not hide intent in global setup, opaque fixtures, or generic helpers.
6. Give drivers semantic intent-based operations, deterministic waiting, useful failure diagnostics, and isolated lifetimes. Do not create a one-for-one wrapper or a shared mutable god-driver.
7. Keep expectations independent from the production implementation; drivers may observe state but must not reproduce the logic that determines the expected result.
8. If the existing test infrastructure cannot express the scenario clearly, refactor it for readiness before adding the test. Stop and ask before introducing a genuinely new testing pattern or dependency.
9. Treat tests as maintained code. Refactor, rewrite, consolidate, or remove nearby tests when that proportionately improves clarity, coverage, reliability, or cost without losing valuable behavioural protection.
10. Run the narrow test, then the relevant broader suite, and review failure output for diagnostic quality as well as correctness.
```

### Bug Investigation

Keep this workflow in a focused, automatically discoverable skill rather than relying on the user to request it explicitly:

```md
---
description: Use automatically when investigating, reproducing, diagnosing, or fixing a bug, intermittent failure, unexplained runtime behaviour, or failing test. Do not use for a feature request with no observed failure.
---

# Bug Investigation

1. Load the applicable subsystem conventions and inspect the relevant code, tests, logs, and recent changes.
2. Define the observed failure and the smallest reliable reproducer.
3. Load the testing conventions and add the smallest practical failing test first when the behaviour is testable. Use or improve the application driver rather than putting non-trivial control mechanics in the test class.
4. Record the working-tree baseline and preserve pre-existing changes.
5. For each conjecture, state the hypothesis and expected observation, make one minimal reversible experiment, and run the reproducer.
6. Retain an experiment only when it produces predicted or clearly useful evidence. Otherwise reverse it completely before trying another conjecture. Never stack unsuccessful experiments.
7. Implement the supported fix and verify the original failure, nearby behaviour, and relevant broader checks.
8. Review the final diff and remove temporary logging, probes, flags, timing changes, speculative fixes, and test scaffolding that have no deliberate final role.
9. Decide separately whether to retain, rewrite, consolidate, or remove the reproducer according to its enduring behavioural value, regression risk, and ongoing cost. When claiming redundant coverage, use a targeted mutation check where practical.
10. Report the evidence for the diagnosis, what was verified, the permanent test decision, and any residual uncertainty.
```

## Recommended Convention Document Format

Do not write vague convention prose. Write documents that make drift obvious.

Use this shape:

```md
# Subsystem Error Handling Convention

Applies to:
- `<path or subsystem scope>`

Canonical pattern:
- Services throw typed domain errors.
- Boundary handlers translate domain errors into user-facing or transport-level responses.
- Shared helpers format any common error payload or reporting shape.

Required rules:
- Do not return ad-hoc `{ ok: false }` objects from services.
- Do not map domain errors to boundary-specific responses inside lower-level helpers.
- New entry points must use the shared error translator.

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

## The "Stop and Ask" Rule

Give this rule one authoritative home at the broadest scope where it is genuinely required. If it applies to every task in the repository, it earns a place in root `CLAUDE.md`. If it applies only to a task type or subsystem, put it in the corresponding skill or path-scoped rule. Do not duplicate it across root instructions, skills, and convention files merely for emphasis; duplicated policy drifts and obscures which version is authoritative.

The language should be explicit:

> Never introduce a new dependency, design pattern, abstraction layer, file structure, naming convention, or solution style without explicit approval. If the codebase does not already establish a pattern for the problem, stop, describe the gap, propose options, and wait for a decision.

That is stronger than "prefer existing patterns." It forces Claude to notice conceptual expansion before it happens.

## Claude Must Load Conventions Before Writing, Not After

A convention system only works if the applicable guidance loads before Claude edits. Make that happen through precise skill descriptions, path-scoped rules, local scope, and references owned by the mechanism that uses them. Requiring root `CLAUDE.md` to enumerate or locate conventions conceals routing defects in the configuration.

The correct sequence is:

1. identify the touched subsystem,
2. load the applicable convention skills,
3. audit the existing code to confirm the canonical pattern,
4. write only after the pattern is clear.

Do not accept "Claude will pick it up from the code." Sometimes it will. Often it will infer the wrong local pattern from whichever files it happened to inspect first.

## How to Establish a New Convention

When Claude finds a real gap:

1. it stops,
2. it describes the missing decision,
3. the human chooses the pattern,
4. Claude updates the convention docs and, if needed, the relevant skill,
5. then Claude implements against the new standard.

This is the basis of a self-maintaining configuration system. The human owns the conceptual surface area. Claude owns keeping the recorded instructions up to date.

## How to Audit an Existing Codebase for Drift

Before trusting conventions, you may need to discover them the hard way.

Run focused audits by concern:

- error handling,
- API call structure,
- external API, SDK, and service leakage,
- async orchestration,
- test layout,
- frontend state,
- design-system coverage,
- UI semantic naming,
- service abstraction boundaries.

For each concern:

1. search the repo for all implementations,
2. cluster them into distinct patterns,
3. choose the canonical one,
4. refactor major divergences,
5. write the convention down,
6. then make future work follow it.

For an external integration audit, check more than whether calls are injectable. Provider SDK imports and provider-owned request, response, identifier, and error types should stop at an application-owned adapter. Application tests should substitute the narrow capability without understanding the vendor; focused adapter tests should verify translation, error mapping, and any provider-specific protocol behavior. As a practical acceptance test, estimate the files that would change if the provider were replaced. Broad changes outside the adapter, composition wiring, configuration, and genuinely provider-specific product behavior reveal a leaky boundary.

This is the only reliable way to stop a large existing project from getting worse.

UI audits need special care because Claude is prone to doing a plausible sample pass and then overstating the result. For design-system questions, require category-complete scans before accepting a verdict. The audit should cover containers, boxes, panels, typography, spacing, borders, separators, corners, shadows, surfaces, icons, controls, colours, loading states, empty states, and error states. It should also distinguish four different things that Claude often blends together: raw values, centralised tokens, semantic tokens, and shared components. A value being tokenised is not enough. The token or component name should encode stable intent or domain meaning, not just current appearance. `danger`, `surface-panel`, `separator-subtle`, and `points-high` are semantic. `red`, `box2`, `border-light`, `canvas-2`, and `ink-2` are weaker unless the project has explicitly documented what they mean.

For a full design-system migration, extend this audit into a counted baseline and keep build, behavioural, and visual confidence separate. [Design-System Refactors](design-system-refactors.md) describes how to prove neutral substitutions, establish screenshot noise floors, predict visual diffs, and report uncovered surfaces without overstating completion.
