# Engineering Conventions

### Conventions are the anti-drift system

Coding conventions are infrastructure, not style preferences. They are the primary mechanism for preventing local shortcuts from accumulating into drift.

### What must be specified

Your convention system must cover every area where Claude can invent a new local solution:

- naming conventions,
- file and module layout,
- import direction and boundary rules,
- application assembly, constructor injection, and external-resource boundaries,
- shared-vs-local abstraction rules,
- error handling,
- result and exception patterns,
- async flows, retries, cancellation, and concurrency,
- state management,
- data fetching patterns,
- API client and server shapes,
- external API, SDK, and service adapter boundaries,
- validation,
- logging,
- formatting expectations,
- test placement and style,
- mocking rules,
- migration patterns,
- generated-code boundaries,
- and any domain-specific invariants that must stay consistent.

If a topic can drift, it needs a convention.

Language- and framework-specific rules prevent locally adequate code from violating repository-wide idioms.

Examples:

- TypeScript: when to use unions vs classes, where runtime validation happens, how async errors are represented, and which import style is canonical
- React: state ownership, effect usage, data-fetching shape, and component boundary rules
- Go: package layout, error wrapping, interface usage, and when helpers should stay local
- Python: module structure, typing expectations, exception boundaries, and how side effects are isolated

#### Type Safety as a Design Tool

Claude often solves local problems with weaker typing than a codebase should tolerate: loose shapes, partial typing, untyped boundaries, `any`-style escapes, or needlessly vague values.

In typed codebases, strong type information controls drift, ambiguity, and bugs. Precision should propagate through surrounding code rather than stop at one file boundary.

The convention should push Claude toward that posture:

- add types aggressively in typed languages instead of treating them as optional polish
- tighten types at boundaries and let that precision propagate through the call graph
- prefer making invalid states unrepresentable when the language and design make that practical
- use the compiler as a design assistant and verification tool, not as an obstacle to work around
- avoid lazy escape hatches unless there is a clear, deliberate reason and the project accepts it

Use the compiler to carry as much of the correctness burden as practical. Avoid the ceremony of typed code without its safety.

#### Abstractions, Encapsulation, and the Helper Smell

Claude does not naturally create strong abstractions or preserve encapsulation. It often produces procedural code spread across loosely related services, helpers, and mutable data structures. Even when it extracts an interface or function, the result may only relocate code without giving the concept a coherent boundary. Extraction is not automatically abstraction, and an interface is not automatically good design.

A useful abstraction represents a stable concept or role while hiding details that callers should not need to know. Good abstractions reduce what the rest of the system must understand: they expose a small behavioral contract, use the language of the problem, and leave room for the implementation to change without coordinated edits across callers. Poor abstractions merely rename mechanics, forward every method of another type, or collect unrelated operations under vague names such as `Helper`, `Manager`, `Utils`, or `Service`.

Encapsulation is the ownership side of the same design. The object or module that owns state should also own the rules that keep that state valid. Callers should ask it to perform meaningful operations, not retrieve its internals, manipulate them elsewhere, and push the result back. Avoid public mutable fields, bags of getters and setters, leaked persistence or transport representations, and orchestration that forces callers to understand a callee's private workflow.

Required rules:

- Design around cohesive responsibilities and domain or application language, not around technical miscellany.
- Put data and the behavior that maintains its invariants in the same object or module where the language permits it.
- Expose the smallest API that lets callers express intent. Keep representation, sequencing, intermediate state, and implementation choices private.
- Tell an object or module what outcome is required rather than asking for its state and implementing its behavior in the caller.
- Make invalid state difficult or impossible to construct. Validate at the boundary, then preserve the invariant internally.
- Prefer narrow role-based collaborator contracts over broad interfaces that reveal an implementation's entire surface.
- Let the consumer's need shape a collaborator abstraction. Do not mechanically create an interface mirroring every concrete class.
- Test through public behavior and observable collaborations. Do not weaken encapsulation or expose internals merely to make tests reach them.
- When a change requires several callers to repeat the same knowledge, look for the concept that should own that knowledge and move the behavior there.
- When no stable concept or second implementation pressure exists, keep an internal concrete implementation. Do not add interfaces, layers, or wrappers as ceremony. External systems are different: replacement, failure, test substitution, and a contract outside the application's control create boundary pressure from the first integration.

A good review question is: **what knowledge does this unit hide, and what invariant or decision does it own?** If the answer is "none; it just forwards calls" or "callers assemble the behavior themselves," the boundary is probably not earning its existence.

Claude often patches design problems with another helper, wrapper, or utility instead of improving the owning abstraction.

That usually means one of two things:

- the design was not ready for the new requirement and should have been refactored first
- the abstraction boundary is wrong and Claude chose a convenience patch instead of fixing it

Helpers are valid when justified, but helper-first thinking often signals weak encapsulation.

The convention should push Claude toward the harder-looking but usually better move:

- look at the owning abstraction before adding a helper beside it
- improve encapsulation at the real boundary instead of scattering utility logic around the edges
- refactor the structure until the new behavior has an obvious home
- treat a new helper as a choice that needs justification, not as the automatic safe option

Inspect and refactor code for readiness before implementing a new requirement. A helper must not substitute for that work.

The acceptance test is not the number of classes or the absence of duplication. After the refactor, callers should know less, the owning unit should control more of its own validity and behavior, and a future implementation change should affect fewer places. If those properties did not improve, Claude probably moved code rather than improving the abstraction.

#### Treat Every External System as Replaceable

Injection alone is not enough. Claude may accept a vendor client as a constructor parameter and still spread that vendor's methods, request objects, response objects, errors, pagination model, and terminology throughout the application. The dependency is now visible and testable, but the application remains coupled to an API it does not own.

The stronger default is:

> Every external API, SDK, or service sits behind a narrow, application-owned boundary. Only its adapter knows the provider contract. The rest of the application speaks in its own capabilities, values, and errors.

This applies to remote APIs and separately deployed internal services as well as payment, authentication, analytics, feature-flag, messaging, notification, storage, search, database, operating-system, and framework services when application behavior depends on them. The boundary need not be a class or a large interface; depending on the language it may be a small protocol, trait, function, module, or closure. What matters is ownership and dependency direction, not ceremony.

Design the boundary from what the application needs, not by copying the provider's entire surface:

- expose intent-based operations and omit provider features the application does not use
- accept and return application-owned values rather than leaking vendor request, response, identifier, or error types
- keep authentication, retries, pagination, serialization, rate-limit handling, and provider quirks inside the adapter when those concerns should be uniform for callers
- translate provider failures once into the application's error model, preserving useful diagnostic context without making callers understand the provider
- keep provider imports and concrete client construction inside the adapter and composition layer
- give tests a small fake implementation of the application-owned contract; do not make ordinary application tests mock a vendor SDK
- test the adapter itself at the boundary with focused integration or contract tests, because a fake proves application behavior but not that the provider mapping is correct

Do not build a one-for-one forwarding wrapper that reproduces every vendor method and type. That merely adds a file while preserving the coupling. A boundary earns its place by reducing what callers need to know, constraining how the service is used, centralizing policy, and containing change.

Use the replacement test during design and review:

> If this provider were replaced, would the change be concentrated in one adapter, its composition wiring and configuration, plus genuinely provider-specific product behavior?

If replacing a provider would require coordinated edits across use cases, domain objects, view models, handlers, or tests, its contract has leaked. Fix leakage in the touched path before adding calls. External contracts are concrete sources of variability, so isolate them before their types and usage patterns spread.

General advice about "good architecture" is insufficient. Put a concise repository-wide rule in root `CLAUDE.md`, detailed guidance in a discoverable convention, and integration-leakage checks in the feature workflow. Where possible, enforce import boundaries so vendor packages appear only in adapters and composition code. Review the application-owned contract and replacement boundary, not merely whether the client was injected.

#### Construct at the Edges; Pass Capabilities Inward

Code becomes difficult to test when useful behavior hides object construction or external access. A service may create its HTTP client, a view model may reach for a singleton, or a repository may open its own database connection. These dependencies are absent from the public construction contract, forcing tests to use global mutation, framework bootstrapping, monkey-patching, real I/O, or private implementation knowledge.

The governing rule should be stated plainly:

> Construct objects at the application's edges. Pass required collaborators through constructors. Pass request-specific data through method or function arguments. Functionality code must not discover or create its own external dependencies.

"Dependency injection" is the established industry term, but the useful practice is simpler than the term suggests. Separate configuring and building the system from using it. A typical executable has three responsibilities:

1. **Entry point**: read command-line arguments, environment, configuration, and framework state; validate and convert them into explicit values.
2. **Construction layer or composition root**: choose concrete adapters, construct the object graph, manage lifetimes, and wire collaborators together. Builders or factories can keep this assembly readable.
3. **Functionality code**: run application and domain behavior using only the values and capabilities it was given.

The control flow should remain obvious:

```text
entry point -> read and validate configuration
            -> build application object graph
            -> start application or invoke use case

unit test   -> construct the same functionality with fakes
            -> invoke behavior directly
```

Dependency direction matters more than directory names or layer counts. Entry points and construction layers know concrete infrastructure. Functionality code receives narrow capabilities without knowing how production implementations are found or assembled. Use constructor parameters for stable collaborators, function parameters or explicit context values for functional code, and method parameters for per-call data.

Ban mutable static and global state: language globals, file-scoped mutable variables, singleton instances, static mutable properties, global registries, shared caches, and convenience APIs such as `shared`, `current`, or `default` when they conceal process-wide state. These constructs hide dependencies and lifetimes, couple unrelated code, leak between tests, and make parallel execution unsafe.

Do not work around the ban by placing mutable state behind static accessors or a singleton protocol. That changes the syntax, not the architecture. The state must belong to an explicitly constructed object with a deliberate lifetime, and that object—or a narrow capability backed by it—must be passed to its consumers.

The ban is on **state**, not on every use of type-level syntax. Immutable compile-time constants, stateless pure functions, and namespaced constructors or factory methods are acceptable because they do not retain mutable process-wide information. When an operating system or framework exposes unavoidable global state, access it only in a boundary adapter; inject that adapter into functionality code so tests can supply an isolated implementation.

Required rules:

- Make every required, long-lived collaborator a constructor parameter. A successfully constructed object should be ready to use and should not need later property injection or hidden initialization.
- Keep the composition root near the process, application, scene, command, request-handler, or framework entry point. Manual wiring is often sufficient. If the project uses a dependency-injection container, resolve from it only in this construction layer; never pass the container into functionality code.
- Treat network clients, databases, filesystems, clocks, random and identifier generators, schedulers, notification systems, analytics, process state, and configuration sources as external capabilities. Pass them in when behavior depends on them.
- Inject application-owned capability contracts into functionality code, not raw vendor clients. Keep each external API, SDK, or service behind the adapter that translates between the provider contract and application concepts.
- Read environment variables, user defaults, files, secrets, and remote configuration at an explicit boundary. Convert raw configuration into typed values before passing it inward.
- Keep framework-owned types and callbacks at adapters where practical. Translate them into application-level values and calls rather than making the whole object graph depend on the framework.
- Use a narrow factory or builder when objects genuinely must be created later from runtime data. Inject that factory into the caller, and give the factory its own dependencies when it is assembled.
- Keep factories specific to the object or subgraph they create. They must not expose a generic container, registry, resolver, or `getService` API; that merely hides dependencies behind a service locator.
- Do not give constructors default arguments that silently create production clients or touch external state. Convenience production construction belongs in the composition root or an explicitly named boundary factory.
- Do not introduce an interface for every class mechanically. Add a role boundary where callers need a substitutable capability, where external infrastructure must be adapted, or where tests need to observe a collaboration. Plain values and self-contained implementation details do not need ceremonial interfaces.
- Treat a long constructor dependency list as design feedback. It may reveal that a class owns too many responsibilities; do not conceal the problem inside a dependency bag or service locator.
- Give caches, stores, sessions, clocks, schedulers, and other stateful services explicit owners and lifetimes in the composition layer. Tests receive fresh instances; they must never depend on global reset hooks or execution order.

Forbidden alternatives in functionality code:

- constructing a concrete network, database, filesystem, analytics, clock, or similar client at the point of use
- importing vendor SDKs or allowing provider request, response, identifier, or error types to escape into functionality code
- reading global configuration, environment, disk, keychains, user defaults, or network state without an injected boundary
- reaching through singletons, global registries, application delegates, static mutable state, or service locators to obtain collaborators
- storing application, session, request, cache, test, or feature state in global variables, module-level mutable values, static properties, or shared instances
- making tests mutate or reset process-wide state before or after execution
- accepting a general container or undifferentiated dependency bag and pulling services from it on demand
- requiring a full application, framework, database, or network boot merely to unit test a domain or application behavior

There are legitimate local constructions. Functionality code may freely create values and private implementation objects that are deterministic, side-effect-free, cheap, and not independently variable. The prohibition is against hidden **collaborators and external capabilities**, not every use of a constructor. A domain-level factory may also create domain objects as part of the behavior it represents. The test is whether creation conceals a dependency or side effect that a caller or test needs to control.

Deferred construction sometimes repeats the pattern at a smaller boundary. For example, a running job may receive data that determines which worker subgraph is required. The job should receive a narrow `WorkerFactory`; the composition layer constructs that factory with its network and persistence adapters, and the factory constructs workers from the runtime data. The job still does not reach into a global container or instantiate production adapters itself.

This structure makes unit testability a design property rather than a testing trick. Before accepting new functionality code, Claude should be able to answer yes to all of these:

- Can the unit be constructed in a test with in-memory values, stubs, or fakes and without starting the application framework?
- Are all operations that can perform I/O, observe time or randomness, or mutate external state visible in the constructor or call signature?
- Can the test invoke the behavior without disk, network, environment mutation, sleeps, or global cleanup?
- Does production wiring live in an obvious construction boundary that can be inspected separately?
- Would a reader know the unit's required capabilities from its public API rather than searching its method bodies?

Load this convention automatically. Put only its crucial repository-wide rule in root `CLAUDE.md`; otherwise scope it to relevant files or tasks and expose the detail through the owning rule or skill. Require workflows to inspect touched code for hidden construction and I/O. Enforce dependency direction with architecture tests or import rules where possible. Tests should construct units directly with explicit fakes; awkward construction is evidence that the boundary needs improvement.

See [Growing Object-Oriented Software, Guided by Tests](https://growing-object-oriented-software.com/) for the testability pressure behind this design, Mark Seemann's Composition Root for the assembly boundary, and Martin Fowler on [separating service configuration from use](https://martinfowler.com/articles/injection.html#SeparatingConfigurationFromUse).

#### Duplication, Redundancy, and the Code Tax

Claude also needs explicit pressure against leaving extra code behind. It often misses an existing implementation and re-creates it, or it completes a refactor but leaves the old path, wrapper, branch, helper, or partially superseded code lying around "just in case."

Every duplicate branch, redundant wrapper, stale helper, and unused file adds maintenance cost.

The convention should be simple:

- prefer less code when less code preserves the intended behavior
- when replacing a code path, remove the old one unless there is a real compatibility reason not to
- do not leave dead code, unused helpers, redundant branches, or superseded implementations behind after a change
- if Claude finds existing code that already solves the problem, it should reuse or consolidate it instead of reimplementing it nearby

Prefer less code when behavior is equivalent. Extra code must justify its maintenance cost.

#### Comments Are an Exception, Not a Substitute for Clear Code

Claude loves to write comments, including comments that make elegant code worse. It narrates the next statement, labels short blocks, restates names in prose, adds section banners inside small files, and leaves tutorial-style explanations for ordinary language features. These comments interrupt reading, duplicate the implementation, become stale, and train future changes to preserve noise rather than clarity.

The rule should therefore be strict:

> Do not add a code comment by default. First make the code explain itself through names, types, cohesive abstractions, encapsulation, and simple control flow. A comment is allowed only when it records essential information that the code cannot express.

Allowed exceptions are narrow:

- **Public API documentation** when an externally consumed contract needs to describe semantics the signature cannot express, such as lifecycle, units, errors, threading, ordering, compatibility, or security requirements. Public visibility alone does not require a comment, and documentation must not merely restate the symbol name or signature.
- **Unavoidable non-obvious code** forced by an external quirk, platform defect, performance constraint, protocol requirement, migration, compatibility concern, or deliberate hack. The comment must explain why the surprising code exists, the constraint that prevents the obvious implementation, and—where useful—the evidence or condition under which it can be removed.

Everything else should be expressed in code or removed. In particular, Claude must not add:

- comments that describe what the next line, branch, loop, or function already says
- headings or divider comments used to organize a function that should instead be decomposed
- comments that repeat type information, parameter names, return values, or test assertions
- `Arrange`, `Act`, and `Assert` labels around already readable tests
- comments commemorating a change, fix, refactor, or superseded implementation; version control already records the change
- commented-out code, speculative TODOs, conversational notes, or explanations addressed to the reviewer
- comments used to excuse confusing names, oversized functions, leaky abstractions, tangled control flow, or missing types

When code appears to need an explanatory comment, Claude should first try, in order:

1. improve the names and types;
2. simplify the control flow;
3. extract a cohesive operation or value;
4. move the behavior behind the abstraction that owns it;
5. remove unnecessary cleverness.

Only if the essential information still cannot live in the code should a comment remain. Write the shortest durable explanation of **why**, not a description of **what** or **how**. During refactoring, review existing comments too: remove those made redundant by clearer code, and update the exceptional comments whose underlying constraints changed.

#### Frontend File Boundaries

Without explicit boundaries, Claude tends to combine markup, styling, state, helpers, and subviews in oversized component files.

UI code is architecture, not presentation glue. Its contracts include component APIs, design tokens, semantic naming, accessibility behavior, responsive layout, interaction states, loading and error states, and brand consistency. Local CSS values and one-off component shapes become part of the product language.

A serious frontend convention should push in the opposite direction:

- assume a non-trivial component may contain reusable or independently understandable parts
- extract meaningful subcomponents, styles, helpers, and view-model logic into dedicated files when complexity starts to rise
- prefer file shapes that make important UI pieces more discoverable elsewhere in the repo
- treat extraction as a readability and maintainability tool, not just a reuse optimization
- treat styling, layout, and interaction patterns as durable product infrastructure, not as disposable prototype code

Do not split everything mechanically, but do not let growing UI files become the default. Smaller files are easier to review and search, and they discourage further accumulation. A component that is hard to scan is a candidate for decomposition even before reuse appears.

#### Application Primitives

Claude is reluctant to create application primitives. It often treats a repeated UI shape as "just styling" and patches the current screen locally instead of extracting the shared product concept. A serious setup should push the opposite direction: when containers, panels, buttons, icon buttons, modals, drawers, separators, warnings, empty states, loading states, clickable rows, tabs, chips, or badges recur across the app, Claude should consider whether the right move is a named primitive with a semantic API.

These primitives are not premature abstraction when the product already repeats the concept. They are how the application preserves UI/UX consistency, accessibility behavior, responsive behavior, interaction states, and future design flexibility. A local button style or one-off warning box may look cheaper in the current diff, but it teaches Claude and future contributors that the product language is optional.

Name these concepts `application primitives`, `product UI primitives`, or `design-system primitives`, and define where they live. Repeated UI behavior needs a stable home.

#### Frontend Semantic Tokens

Frontend semantics deserve another explicit rule: Claude is too eager to reuse visual styles by superficial appearance instead of by meaning.

This often appears in token and class reuse. For example, a red `danger` token for errors or destructive actions does not represent an unrelated domain concept merely because both render red.

The convention should be semantic tokens first, implementation second:

- name tokens and styles for what they mean, not just how they look
- do not reuse an error or danger token for an unrelated domain concept just because the current color is similar
- allow two semantic tokens to resolve to the same presentational value when appropriate, but keep the semantic names distinct
- prefer domain-language naming in domain features, even when the current visual treatment overlaps with an existing utility

Visual coincidence is not semantic equivalence. Separate concepts may share a value while needing independent names and future treatments. Collapsing them makes code less legible and design changes harder.

#### Site-Wide Semantic Constants

Claude is also bad at centralizing site-wide UI semantics. Even when a product clearly has repeated layout and presentation rules, it tends to scatter them across components instead of establishing a single semantic layer that the rest of the interface can depend on.

This problem is much broader than colors and fonts. The centralized layer often needs to cover:

- semantic typography
- spacing and layout rhythm
- borders, strokes, and separators
- corner radii and surface treatments
- elevation and shadow rules
- icon and emoji semantics
- motion, animation, and reaction patterns
- reusable state styles
- and any other visual primitive that should remain consistent across the site or app

The convention should be to define these things in one intentional place whenever they are site-wide concerns. If they are scattered across individual components, even a straightforward reskin or brand refresh becomes expensive because the semantics were never separated from the local implementation.

Done correctly, this pays off twice:

- day-to-day UI work becomes more consistent because Claude has one canonical place to follow
- large-scale visual changes become much cheaper because the semantic layer can change without rewriting every component by hand

Claude needs explicit pressure here because its default instinct is local convenience. It will happily inline spacing values, duplicate border styles, pick ad hoc icon treatments, and repeat animation choices file by file unless the project establishes a central semantic system and tells it to use it.

Creating or retrofitting that system needs a stricter workflow than ordinary component work. See [Design-System Refactors](design-system-refactors.md) for the inventory, token-modelling, component-convergence, sequencing, and cross-surface verification rules. In particular, do not mistake tokenising literals for completing a design system, and do not converge components merely because they look similar.

#### Structured Logging

Logging is a data-quality concern, not just a style concern.

Interpolated strings, inconsistent field names, multiline dumps, and prose-data mixtures produce logs that are difficult to filter, aggregate, and query. Define an explicit logging convention.

The convention should be simple:

- treat the log message as a stable event label, not a sentence template
- never parameterize the message string with runtime values
- put runtime values in the second argument as a structured JSON object
- ensure the logger emits the label and the JSON payload on one line

Bad:

```ts
logger.debug(`i noticed item ${foo} changing {count} times!!`)
```

Good:

```ts
logger.debug(`item change observed`, { item: foo, count })
```

Applied consistently, this pattern creates an event stream that can be filtered by label, aggregated by field, and queried without brittle string parsing.

If the codebase cares about observability, do not leave logging style to Claude's judgment. Write down the event-label-plus-JSON rule as a convention, add examples, and enforce it in review.

#### HTTP and API Behavior

API work must account for protocol behavior, not only response JSON.

That includes things like:

- content negotiation and `Accept` handling where relevant
- compression such as gzip or whatever transport conventions the stack already uses
- HTTP response headers beyond the bare minimum
- cache behavior, cache-control policy, and freshness rules
- validators such as `ETag` and related conditional request support where appropriate

Header choices depend on the product, traffic, and infrastructure. API work does not end at the response body: implement relevant protocol behavior or ask which HTTP conventions apply. Do not defer basic protocol concerns until they are costly to retrofit.

#### Exceptions and Fail-Fast Behavior

Error handling deserves equally explicit guidance. Claude is unusually bad at exception discipline because its training data over-represents defensive local `try/catch` blocks that catch, log, and continue in places that should simply fail.

That produces several bad outcomes:

- broken state is allowed to limp forward
- execution becomes harder to reason about
- logs become noisy and duplicated
- the real failure site becomes harder to locate

Almost always, the safer default is fail fast:

- let errors and exceptions bubble out of the guts of the system by default
- catch exceptions at clear application boundaries, not at every available call site
- only catch locally when the code can actually recover, translate the error meaningfully, or perform required cleanup
- if the code cannot restore a valid state, do not catch-and-carry
- if the project wants a different exception policy, Claude should ask rather than invent one

Let unrecoverable failures surface clearly. Silent recovery and local log-and-continue behavior reduce reliability.

#### Exception Logging and Context

Do not log only an exception message and discard its stack trace. The stack trace identifies the failure location.

The triggering state is often just as important. Parameters, identifiers, and other relevant runtime context can be the difference between a fixable production failure and an untraceable mystery. When the language and platform allow it, Claude should preserve that context with the exception rather than discarding it or reducing it to a vague log line.

So the convention should say:

- do not strip stack traces when logging exceptions unless there is a very specific reason
- prefer passing the original error object through structured logging or the platform's native exception logging path
- attach the relevant runtime context or input state to the exception or structured log payload when the platform supports it
- include enough context to reproduce or diagnose the failure, but not so much that logs become a data dump or a security problem
- avoid logging the same exception repeatedly at multiple layers as it bubbles outward
- log once at the boundary that is responsible for reporting, handling, or terminating the failure

Explicitly prohibit early catch-log-and-continue behavior unless the code can recover.

#### Mechanical Formatting

Formatting deserves special treatment. Claude is not reliable at preserving exact formatting conventions over time, especially in mixed-language repos or codebases with very specific style requirements. Do not rely on prose alone here. Put formatting under mechanical control.

Uncontrolled quote style, spacing, and line wrapping create review churn and obscure substantive changes.

For many teams, [dprint](https://dprint.dev/) is a strong default because it is fast, multi-language, and configuration-driven. The practical pattern is:

The exact command surface depends on the stack. Examples in this guide may use `npm run ...` because it is familiar shorthand, but the point is the shape of the workflow, not npm specifically.

- define formatting policy in formatter config, not in prose
- expose a simple formatter command or script that applies fixes directly
- run formatting automatically on touched files after edits
- use checks in CI or review flows to catch anything that still slips through

For example, `dprint.json` can define the policy, one command can apply it, and another can check it in CI. A narrow post-edit hook may format only touched files, avoiding a noisy repository-wide rewrite. The repository, not the model, should decide the final formatting.

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

Keep global guidance short and load local detail when relevant.
