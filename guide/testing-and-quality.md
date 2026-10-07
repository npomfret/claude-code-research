# Testing and Quality

Tests should express enduring behaviour and provide trustworthy evidence at reasonable cost. Test code deserves the same design care as production code. A reader should be able to understand a scenario, its important preconditions, and its expected outcome without tracing application plumbing.

The first sections cover scenario design, drivers, execution, and test maintenance. The later sections show how to route this guidance to coding agents and maintain wider engineering conventions.

## What to consider testing

Choose coverage according to the project's behaviour and risks. Consider the following where applicable; this is a checklist of possibilities, not a requirement to test everything:

- Normal workflows and their expected results.
- Boundary values, empty inputs, invalid or malformed data, and unusually large inputs.
- Important state transitions, domain invariants, and sequences of operations.
- Errors, unavailable dependencies, timeouts, cancellation, retries, and recovery.
- Concurrency, duplicate requests, ordering, and idempotency.
- Persistence, migrations, and compatibility with existing data or external contracts.
- Authentication, authorisation, and isolation between users or tenants.
- Relevant UI behaviour, accessibility, and loading, empty, and error states.
- Performance and resource consumption under representative workloads, in a dedicated suite.
- General properties across generated inputs where property-based testing adds useful coverage.

Check the quality of the evidence as well as the selection of cases: assertions should detect broken behaviour, substitutes should reflect the real contracts they represent, and tests should be deterministic and independent. Investigate flakiness rather than hiding it with retries. Coverage figures indicate what ran, not whether it was meaningfully checked; targeted mutation testing can help assess assertion strength.

## Coverage versus confidence

Chasing a code-coverage percentage is a fool's errand. Executing a line does not establish that its behaviour was meaningfully checked. Do not add low-value tests, preserve redundant ones, or weaken design merely to increase the number.

Coverage is never a goal in itself; protecting meaningful behaviour is. Do not write dedicated tests for completely trivial code merely to prove that it executes: straightforward field assignment and simple forwarding without additional logic are examples. Reserve tests for meaningful rules, validation, side effects, state transitions, and material risks rather than mechanically exercising boilerplate. Judge triviality by the absence of meaningful decisions or risk, not line count; a one-line calculation can still encode an important rule. Use compiler and static-analysis checks for the guarantees they can establish.

Use coverage as a diagnostic tool to find unexamined code and ask whether important behaviour lacks protection. Prioritise meaningful assertions, failure paths, boundaries, and material risks over blanket numerical targets. Targeted mutation testing can help check whether tests detect deliberate faults, but it too is evidence to interpret rather than another score to maximise.

## Design for testability

Consider how code will be tested while designing it: how will a test establish its starting state, supply dependencies, exercise behaviour, and observe the outcome? Good testability comes from clear responsibilities, explicit dependencies, and controlled lifetimes while preserving encapsulation.

**Do:**

- Pass required capabilities through constructors or function parameters so tests can supply in-memory substitutes for external systems.
- Give tests fresh instances with independent state rather than requiring global resets.
- Make time, randomness, and scheduling controllable where they affect behaviour.
- Verify state transitions through application-facing behaviour, returned results, or meaningful interactions with supplied dependencies.
- Let difficulty testing a behaviour prompt a review of the design and its responsibilities.

**Don't:**

- Hide dependencies in mutable static state, globals, or internally constructed external clients.
- Make fields public, add test-only getters, or bypass private access solely so tests can inspect internal state.
- Add public operations solely to let tests manipulate the implementation.
- Couple assertions to internal representation when observable behaviour establishes the contract.

A public interface should exist because the application needs it. Tests should exercise that interface without weakening it. An explicit constructor is useful for supplying dependencies in object-oriented code, but its absence alone does not make a class untestable. See [Abstractions and encapsulation](engineering-conventions.md#abstractions-and-encapsulation) and [Construct at the edges; pass capabilities inward](engineering-conventions.md#construct-at-the-edges-pass-capabilities-inward).

### Configuration without global state

Mutable static or global state hides dependencies and undermines isolation. Reading environment variables throughout application code has the same problem: a component silently depends on process-wide state, and tests must mutate that state to control it.

Supply the application's required configuration at startup, validate it, and encapsulate it in explicit typed configuration objects. Pass the relevant configuration to the components that need it; do not let them discover values through globals, static accessors, or scattered environment reads. If environment variables are a deployment input, read them only at the startup boundary.

Tests should construct configuration directly in memory and pass it in, without changing the process environment or depending on a developer's configuration files. Test configuration loading and validation separately at their boundary. See [Globals, environment variables, and configuration](engineering-conventions.md#globals-environment-variables-and-configuration).

## Readable behavioural tests

Treat test code with the same respect as production code: design focused abstractions, encapsulate mechanics, and refactor for clarity. Hundreds of lines of setup, polling, and waiting make the test itself difficult to trust and make failures harder to diagnose. Keep the test case simple to read, even when exercising it requires complex machinery.

The test owns the meaningful starting state, action, and expectations. Test drivers own the mechanics of controlling and observing the application; their design is described below.

Use names and whitespace to make setup, sanity checks, execution, and verification immediately recognisable. Encapsulation should remove distracting detail while keeping the behaviour being tested explicit.

**Do:**

- Write integration and end-to-end scenarios in application or domain language.
- Keep scenario-defining values, important preconditions, intermediate checks, and final expectations visible.
- Use blank lines to separate setup, sanity checks, execution, and verification.
- Keep expected results independent of the production calculation under test.

**Don't:**

- Hide intent in global fixtures or generic helpers.
- Spread sleeps, polling, screenshots, and retries through test bodies (use a driver).
- Extract every literal or add ceremony to a simple unit test.

**Example test:** the driver names and scenario are illustrative.

```ts
const account = await accounts.create({ username: "alice", password: "correct-password" });
await login.open();

await login.expectReady();

await login.fillUsername(account.username);
await login.fillPassword("incorrect-password");
await login.submit();

await login.expectError("Invalid username or password");
```

A product-aware reader should understand this scenario without knowing its framework or application plumbing. The four groups show setup, a sanity check, execution, and verification without needing `Arrange`/`Act`/`Assert` narration. The driver encapsulates selectors and bounded waiting for readiness or an error; the test states which outcome it expects. A missing error must fail with useful diagnostics when the wait expires.

## Meaningful assertions and sanity checks

Every behavioural test should assert a meaningful outcome. Merely executing code without crashing is weak evidence unless successful completion itself is the explicit contract being tested. Assertions may live in clearly named driver operations such as `expectError`, but the expected outcome must remain apparent in the scenario.

When testing a state change, sanity-check the relevant starting state so the final assertion cannot pass simply because it was already true. For example:

```ts
const subscription = subscriptions.build({ status: "active" });

expect(subscription.canAccessPaidFeatures()).toBe(true);

subscription.cancel();

expect(subscription.canAccessPaidFeatures()).toBe(false);
```

In this illustrative domain, cancellation removes paid access. The sanity check establishes that the action must change something to satisfy the final assertion. Use this where the starting condition matters; do not mechanically duplicate every final assertion with its inverse. Test unchanged state or idempotency with the preconditions appropriate to that behaviour.

Ask whether the test would fail if the action did nothing or produced the wrong result. Check the outcome that matters, keep expected results independent of the production calculation, and avoid assertions that merely confirm incidental setup. Observe state through the application's legitimate public interface rather than exposing internals for assertions.

## Test data construction

Encapsulate test-data construction in recognisable patterns such as builders and factories. Do not fill test bodies with object graphs, repetitive field assignments, or persistence setup that obscures the scenario. Builders should produce valid objects by default and offer explicit variations; a factory can create a ready-to-use value or account with a clear purpose.

Invalid data is sometimes the point of a test. Use a clearly named invalid-data builder or an explicit option that makes the violated rule apparent. Construct malformed input at the boundary being tested rather than weakening production types or bypassing domain invariants to manufacture an impossible internal object.

Keep only scenario-defining parameters at the call site; let the builder supply incidental data:

```ts
const account = accounts.build({ plan: "trial", status: "expired" });
```

The test states the relevant plan and status; the builder owns unrelated valid defaults. Use the same care for these abstractions as for drivers: model meaningful concepts, avoid shared mutable data, and make external effects apparent. For example, distinguish an in-memory `build` operation from a `create` operation that persists data.

Every explicit override should earn its place: changing it so that the intended precondition no longer holds should invalidate the scenario and make the test fail. If an override can be removed without affecting the scenario, move it into the builder's defaults. This is a relevance check, not a requirement that every alternative value fail; different values may represent the same behavioural case.

**Don't:** hide a scenario's important preconditions in defaults, replace construction code with a miscellaneous helper collection, or let a unit-test builder silently access external systems. Simple literals can stay inline when an abstraction would add ceremony rather than improve clarity.

## Test drivers and encapsulation

**Test drivers** provide a focused interface for controlling and observing the application. Other names for this role include Page Objects (often organised into a Page Object Model for web tests), robots, harnesses, and API test clients. Builders can encapsulate the construction of scenario state. The terminology varies; the important boundary is between the readable scenario and the mechanics needed to execute it.

Design these abstractions around meaningful state and behaviour. A driver should own knowledge that the scenario does not need, such as how to locate a login field or observe completion of an asynchronous operation. Merely moving procedural code into a generic `helpers` file does not create that boundary.

For tests beyond simple unit cases, use drivers wherever controlling or observing the system introduces non-trivial mechanics. They keep that complexity out of the scenario and provide one place to maintain it.

**Do:**

- Encapsulate non-trivial setup, navigation, request construction, selectors, synchronisation, diagnostics, and cleanup in focused abstractions.
- Give drivers clear operations such as `fillUsername`, `submit`, and `expectError`; make their effects and expectations apparent from their names.
- Keep scenario-defining choices explicit in the test, even when a builder or driver performs the setup.
- Encapsulate asynchronous waiting with bounded deadlines and useful failure evidence.
- Give stateful drivers explicit, isolated lifetimes and ownership of their resources.
- Improve the driver before adding a scenario that would otherwise accumulate procedural machinery.
- Review and refactor drivers with the same care as production code.

**Don't:**

- Create a one-for-one wrapper around selectors or SDK calls without hiding meaningful knowledge.
- Create one driver responsible for the whole product.
- Hide scenario actions or assertions inside unrelated setup operations.
- Add abstractions that make a simple test harder to understand.

### Waiting for asynchronous state

Never sleep for an arbitrary duration and assume that an asynchronous update has completed. A fixed delay makes tests unnecessarily slow when the system is fast and unreliable when it is slow.

Put condition-based waiting in the driver. Poll until the expected state is observed, or use a reliable event or framework waiting mechanism that establishes the same condition. Return control to the test only when that condition holds; fail when a bounded deadline expires, reporting the expected condition and last observed state. Any polling interval belongs inside the driver, not in the scenario.

```ts
await orders.submit(order);

await orders.waitForStatus(order.id, "confirmed");
await orders.expectConfirmation(order.id);
```

The test states the action and the state it needs next. The driver owns how to observe that state, the polling or synchronisation mechanism, the deadline, and failure diagnostics. Waiting for state should observe it without repeating the action that initiated the update.

## Test types and suites

Every project should define its test vocabulary and the boundaries of each type. Names such as "unit test" are overloaded; document what they mean in this repository. The following definitions are a suggested starting point, not a requirement to implement every type.

| Type | Boundary and purpose |
| --- | --- |
| Unit | Executes application code entirely in memory, without network, database, filesystem, or external-process access. Pass in fakes, stubs, or mocks for external dependencies, through constructors or function parameters, and check behaviour and meaningful interactions. |
| Integration | Exercises code against real boundaries such as a filesystem, database, network service, or another process. Define which components are real and which are substituted. These tests usually require more setup and run more slowly. |
| End-to-end | Exercises deployed, running software through its external interfaces, using a separate test client rather than calling application internals. The deployment may run locally or on a remote server. |
| Smoke | Runs a small, non-destructive selection of checks against the deployed system to confirm essential behaviour after a staging or production deployment. This describes a suite's purpose and may reuse end-to-end scenarios. |
| Provider | Checks that a third-party provider behaves as the application's integration expects. State whether it uses a sandbox or live service, and distinguish provider failures from application failures. |
| Performance | Measures latency, throughput, or resource consumption under a defined workload. Keep it separate from ordinary behavioural suites, with explicit measurement conditions, baselines, and acceptance criteria. |

Under this suggested definition, a unit test need not isolate one class or function. It can construct a group of real objects and exercise their collaboration entirely in memory. Twenty collaborating objects do not turn it into an integration test; accessing the network, a third-party service, a database, the filesystem, or another external process crosses that boundary. Substitute external dependencies rather than automatically mocking every internal collaborator. In-memory execution enables fast tests, but their runtime must still meet the project's unit-test budget.

Lean on unit tests for broad behavioural coverage: they are fast, cheap, and easy to run frequently. Add tests at real boundaries where in-memory substitutes cannot establish the required confidence. A project may need only unit tests; choose the types that protect its actual risks.

Choose the right suite for the purpose and make the distinctions explicit. These categories can overlap: a smoke suite may reuse end-to-end scenarios, and a performance suite may exercise an in-memory unit or a deployed system. Sharing a framework does not make their execution needs the same; a performance test written with JUnit should not be selected as an ordinary unit test.

Clearly name and label suites, and make them independently runnable when their purpose, dependencies, or execution needs differ. For each suite, define:

- Its scope, selection rules, and run command.
- Required services, data, credentials, and any cleanup or retention policy.
- Expected runtime and enforced time limits.
- Where and when it runs, including any special hardware or deployment requirements.

Running the unit suite should never silently start services or require access to the outside world.

## Choosing substitutes and testing approaches

Choose testing approaches by the confidence they provide relative to their complexity and cost. Speed matters, but so do readability, fidelity to the real system, maintenance, and useful failure diagnostics. Each project should decide these tradeoffs rather than automatically choosing the fastest or most isolated technique.

Two examples to consider:

- **Mocking library versus an explicit stub or fake:** does reflection or interception machinery require verbose setup, obscure the scenario, or produce unhelpful errors? A small substitute with a focused driver may be easier to read, control, and diagnose. Use a mocking library when it earns its complexity; assert interactions that matter to the contract rather than reproducing every internal call and its order.
- **React tests with a simulated DOM versus a real browser:** does the in-memory environment provide convincing evidence and understandable failures for the behaviour in question? Loading the page in a browser and interacting through Playwright may be slower and more expensive but easier to inspect and trust, especially for browser-dependent behaviour. A simulated environment may still suit focused cases; choose according to the evidence required.

Substitutes must reflect the contracts they stand in for. Keep real-boundary coverage where a fake or mock cannot establish that the integration works.

Do not equate speed or isolation with greater value. A slower approach can be worthwhile when it supplies clearer, more trustworthy evidence.

## Reliable execution and verification

**Do:**

- Give each test independent state; keep failures from contaminating later tests.
- Make timeout failures explain the expected condition and the observed state.

**Don't:** introduce shared mutable state, execution-order dependence, or hidden cleanup, or treat retries as evidence that an intermittent failure is fixed.

### Isolation and starting state

Every test is responsible for establishing its own preconditions and isolating its state from other activity. It must not rely on execution order, another test's setup or cleanup, or an assumption that tests run sequentially or in parallel. Shared infrastructure may manage this isolation, but each test must receive an independent starting state.

For example, an integration test against a real database must not assume the database is empty or contains records left by a previous test. Create the required data and use an appropriate isolation mechanism, such as a dedicated database or schema, a unique tenant or namespace, or a transaction whose effects remain isolated from other tests. Choose a mechanism that fits the application, including any background workers or separate connections.

Cleanup is a project decision, separate from isolation. Properly isolated data may not need per-test removal, especially in disposable environments; retaining it can help diagnose failures. Decide whether resources are removed after each test, disposed of with the environment, or retained under a defined policy. Account for accumulated storage or cost, and release resources such as locks or processes when retaining them would cause interference or exhaustion. When cleanup is used, handle failure paths and remove only resources owned by the test.

Cleanup alone does not prevent interference while tests run: avoid shared identifiers, unscoped queries, and global resets that can affect concurrent activity. A test should produce the same result when run alone, in a different order, or alongside other tests within the suite's supported resource limits.

### Execution concurrency

Parallel execution is a project decision, not an automatic goal. Independent tests allow the team to choose sequential or parallel execution without changing their meaning. Set concurrency according to CPU, memory, shared-service capacity, and the need to keep a developer's machine usable while tests run; shortest elapsed time is not always the priority.

Local and CI environments may use different worker limits. Document sensible defaults and how to override them. Isolation remains important even when the suite runs sequentially.

### Enforced time limits

Every project should choose and enforce strict completion deadlines appropriate to each test type. Large suites, expensive computations, and slow external operations must not leave builds waiting indefinitely with no way to distinguish progress from a hang.

Set limits for individual test cases and, where appropriate, test classes or groups, whole suites, and CI jobs. Include setup and teardown in those limits or give them their own deadlines. A timeout must fail the check and stop the stalled work; an outer suite or job deadline should catch hangs that the test runner cannot interrupt.

Unit tests should be very fast. A project might allow at most one second per unit test, already a generous allowance for many in-memory cases; this is an example, not a universal threshold. Integration and end-to-end tests may need longer limits reflecting their dependencies and execution environment. Define these budgets per project rather than inheriting unlimited waits or unexplained framework defaults.

Report which test or phase timed out, the elapsed time and configured limit, and useful available diagnostics. Investigate unexpected slowness rather than automatically increasing the timeout.

### Flaky tests

Flaky tests damage productivity and trust in the suite. Treat recurring intermittent failures as defects requiring prompt investigation, not as an accepted background condition. Where supported, use CI's test history and flaky-test tracking to identify patterns, prioritise work, and assess whether a fix improved reliability.

Investigate whether the cause is application behaviour, test logic, isolation, synchronisation, resource contention, or infrastructure. Preserve the original failure and diagnostics when retrying; a passing retry does not prove the defect is fixed. If temporary quarantine is necessary, make the lost coverage visible and assign responsibility for restoring it.

Aim for a dependable suite rather than an unattainable guarantee of zero failures. An occasional infrastructure slowdown may exceed a reasonable timeout even in a well-designed test. Distinguish isolated incidents from recurring patterns, use evidence to guide effort, and avoid both normalising unreliable tests and spending disproportionate effort chasing perfection.

### Local checks and continuous integration

In a non-trivial project, the full suite may be expensive or require services, hardware, or capacity unavailable on a developer's machine. A recommended approach is to rely heavily on the continuous integration server for comprehensive verification. Each project should decide its required local checks, push requirements, and CI coverage explicitly.

Suggested workflow, subject to the project's policy:

1. Run the tests appropriate to the change locally, within the machine's capabilities. The project may allow pushing without running the full suite locally.
2. Push the change so CI builds it and runs the full required suite in the configured environments.
3. Monitor the CI run for that revision, inspect failures, and fix or investigate them before treating the change as verified.
4. Report local results and CI results separately. Pending, skipped, or failed required checks do not establish success.

Document which checks CI runs and any separate deployment or scheduled suites. Passing CI provides evidence for the behaviour those checks cover. See the [Git workflow](workflows-and-maintenance.md#git-workflow).

## Test-first discipline

Where the team uses TDD:

1. Reproduce a testable bug or specify a feature with the smallest practical failing test.
2. Implement enough to pass.
3. Refactor both production and test code after green.
4. Show relevant passing checks before claiming the behaviour works.

Use the repository's chosen discipline pragmatically. If a failing automated test is impractical, explain the exception and identify the evidence used to verify the change.

**Don't:** claim TDD for a workflow that does not follow it, or automatically retain every diagnostic reproducer.

## Controlled bug investigation

The ideal bug-fixing workflow starts by writing an automated test that exposes the bug before changing production code. Observe it fail for the expected reason, implement the fix, and observe it pass. This applies even when the team does not use TDD for feature development: it provides evidence that the change addresses the reported failure. When an automated reproducer is impractical, explain why and use the smallest reliable alternative.

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

## Maintaining the test suite

Regularly review, refactor, consolidate, and delete tests. This is healthy, encouraged maintenance: a test is not permanently valuable merely because someone wrote it. Behaviour can be retired, assumptions can become obsolete, and stronger coverage elsewhere can make a test redundant.

Preserve useful protection rather than test count. Before deleting a test, establish whether its behaviour still matters and whether retained coverage protects it adequately. Similar-looking tests may cover different boundaries or failure modes. Removing obsolete or redundant tests should make the suite clearer, faster, and easier to trust, without concealing unresolved failures.

### Reproducer retention

**Open question for the team:** after fixing a bug, should its reproducing test remain in the permanent suite? If the bug had never existed, would we have written this test? What value does it provide now that the failure is fixed?

A reproducer supports diagnosis; a permanent test can protect an enduring contract or guard against recurrence. Fixing the bug does not eliminate that possible value, but discovering a bug does not automatically justify keeping its test forever. Teams should decide their retention policy explicitly, considering both future protection and ongoing cost.

The following options support that discussion rather than prescribe a universal answer:

One useful outcome is to replace the diagnostic reproducer with a clear test of the required behaviour that the bug revealed was insufficiently protected. Express the contract in application or domain terms rather than preserving the accident of how the bug was discovered. This may mean adding a missing scenario or strengthening an existing test for an overlooked input or state. Verify that the replacement would fail if the original fault returned.

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
2. Keep scenario state, actions, and expectations visible; separate phases with whitespace.
3. Put non-trivial application-control mechanics and bounded waiting behind focused, isolated drivers.
4. Give test code the same design care as production code; refactor drivers before adding procedural scenarios.
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
