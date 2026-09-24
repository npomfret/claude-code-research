# Engineering Conventions

Conventions keep independently made changes coherent. Be strict: specify the choices that matter, name forbidden shortcuts, show the canonical pattern, and enforce what tooling can check. A working happy path does not excuse concealed failures or invalid state. Keep global policy short and load subsystem detail where it applies.

## Convention coverage

Cover naming and file layout; import direction; construction and external-resource boundaries; shared versus local abstractions; errors; async flows, retries, cancellation, and concurrency; state and data fetching; APIs and validation; logging; formatting; tests and mocks; migrations; generated code; and domain invariants.

Add language and framework guidance where the repository needs it: TypeScript boundary validation and error types, React state and effects, Go package and error conventions, or Python typing and side-effect boundaries.

## Type safety

Use types to express contracts and prevent invalid states, with precision carried across file boundaries.

**Do:**

- Tighten boundary types and propagate them through the call graph.
- Make invalid states unrepresentable where practical.
- Use the compiler to guide design and verify changes.
- Encapsulate everything!

**Don't:**

- Introduce loose shapes, partial typing, or `any`-style escapes without a deliberate, accepted reason.
- Work around type errors by weakening a meaningful contract.
- Pass around primitive types.

## Abstractions and encapsulation

A useful abstraction hides knowledge and owns a decision or invariant. Review it by asking: **what do callers no longer need to know?** Moving code alone does not establish a better boundary.

**Do:**

- Organise around cohesive domain or application concepts.
- Keep state and the behaviour maintaining its invariants together.
- Expose narrow, intent-based operations; keep representation, sequencing, and intermediate state private.
- Let consumer needs shape collaborator contracts.
- Move repeated knowledge into the unit that should own it.
- Inspect the owning abstraction before adding a helper; refactor it when the behaviour lacks a coherent home.
- Test public behaviour and observable collaborations.

**Don't:**

- Expose mutable internals or make callers coordinate an owner's private workflow.
- Treat getters, setters, forwarding interfaces, or miscellaneous `Helper`/`Utils` classes as sufficient abstraction.
- Expose internals solely for tests.
- Create interfaces or layers mechanically when no stable concept or substitution need exists.

External systems already create substitution and failure boundaries on first use; they do not need a second provider to justify isolation.

## Treat every external system, API, library etc as replaceable

Each external API, SDK, or service belongs behind a narrow application-owned adapter. Injection makes a dependency visible; the adapter contains its provider contract. Always assume we will want to replace it and that job should be easy.

**Do:**

- Define operations from application needs using application-owned values and errors.
- Keep vendor imports, client construction, request/response types, identifiers, and quirks in adapters and composition code.
- Centralise authentication, retries, pagination, serialization, and rate-limit policy where callers need uniform behaviour.
- Translate failures once while preserving diagnostic context.
- Substitute small application-contract fakes in ordinary tests.
- Verify provider mapping with focused adapter integration or contract tests.
- Enforce import boundaries where possible and fix leakage in touched paths before adding calls.

**Don't:**

- Inject a raw SDK client throughout functionality code.
- Create a one-for-one wrapper that preserves the provider's whole surface.
- Assume a fake proves the production adapter works.

**Replacement test:** changing provider should primarily affect its adapter, wiring, configuration, and genuinely provider-specific product behaviour. Coordinated changes across use cases, handlers, view models, and application tests indicate leakage.

## Construct at the edges; pass capabilities inward

Separate system assembly from behaviour. A unit should be constructible directly with explicit fakes and callable without booting the application or mutating global state.

| Boundary | Responsibility |
| --- | --- |
| Entry point | Read and validate arguments, environment, configuration, and framework state. |
| Composition root | Choose adapters, construct collaborators, and own their lifetimes. |
| Functionality | Use the typed values and capabilities it receives. |

**Do:**

- Pass stable collaborators through constructors or explicit function arguments, and per-call data through method arguments.
- Make constructed objects ready to use without later property injection or hidden initialization.
- Inject narrow application-owned capabilities for network, databases, filesystems, time, randomness, IDs, scheduling, notifications, analytics, and configuration.
- Convert raw configuration into typed values at a boundary.
- Keep framework callbacks and types in adapters where practical.
- Resolve dependency-injection containers only in the composition layer; manual wiring is often sufficient.
- Inject a specific factory when runtime data determines later construction. Assemble that factory with its own collaborators at the edge.
- Give stateful stores, caches, sessions, and schedulers explicit owners and lifetimes; give tests fresh instances.
- Treat long constructor lists as feedback about responsibility size.

**Don't:**

- Construct external clients, read configuration, or perform hidden I/O inside functionality code.
- Obtain collaborators from singletons, globals, application delegates, registries, or service locators.
- Store mutable application state in module variables, static properties, shared instances, or global caches.
- Hide that state behind static accessors or singleton protocols.
- Pass a general container or undifferentiated dependency bag inward.
- Use constructor defaults that silently create production dependencies.
- Require global reset hooks, execution order, sleeps, or a full framework boot for unit tests.

Immutable constants, stateless functions, and deterministic, cheap, side-effect-free local value construction are valid. Unavoidable platform global state belongs behind an injected adapter. A domain factory may create domain objects; it must not conceal external capabilities.

**Review checks:** required capabilities are visible in the public API; production wiring has an obvious home; tests construct and invoke the unit with in-memory values and fakes. Load this convention for touched code and enforce dependency direction mechanically where possible.

See [Growing Object-Oriented Software, Guided by Tests](https://growing-object-oriented-software.com/), Mark Seemann's Composition Root, and Fowler on [separating configuration from use](https://martinfowler.com/articles/injection.html#SeparatingConfigurationFromUse).

## Globals, environment variables, and configuration

Globals hide dependencies and make tests difficult to isolate. Environment variables are process-wide configuration state too: reading `process.env` throughout the application forces tests to mutate shared state and can make results depend on execution order. A config file only improves this when it is loaded explicitly, rather than discovered globally by each consumer.

**Do:**

- Prefer an explicitly supplied config-file path at the application entry point.
- Load and validate configuration once; encapsulate it in a typed, immutable configuration object with meaningful fields.
- Pass relevant configuration through constructors or function arguments. Give each consumer the settings it needs rather than unrestricted access to the whole application config.
- Construct test configuration directly in memory; test file loading and validation separately at the boundary.
- If deployment requires environment variables or a secret source, read them only at the entry/composition boundary and translate them into explicit configuration before passing them inward.

**Don't:**

- Introduce `.env` files or new environment variables as the automatic solution to configuration needs.
- Read `process.env`, load dotenv, discover config files, or fetch global configuration inside behaviour code or at module import time.
- Export a global config singleton, even if its values are immutable; consumers must declare the configuration they depend on.
- Make tests edit environment variables, depend on a developer's `.env`, or reset shared configuration between runs.
- Default missing required settings to guessed values. Fail at startup with a precise validation error.

Keep secrets out of committed configuration. The source may vary by deployment; explicit ownership, validation, and injection remain the same.

## Duplication and redundant code

Every retained path has a maintenance cost.

**Do:** search for existing solutions, reuse or consolidate them, and delete replaced paths in the same change.

**Don't:** leave dead helpers, unused files, redundant wrappers, speculative branches, or superseded implementations without a real compatibility reason.

## Comments

Make code explain itself through names, types, cohesive operations, and clear control flow. Comments are reserved for essential information code cannot express.

Essentially: DO NOT COMMENT CODE!

**Do:**

- Document ONLY non-obvious public contract semantics: lifecycle, units, errors, ordering, threading, compatibility, or security.
- Explain unavoidable platform quirks, protocol requirements, performance constraints, migrations, or hacks with a short durable reason and useful removal condition.
- Review existing comments during refactoring and remove redundant ones.

**Don't:**

- Narrate statements, repeat signatures or assertions, or add tutorial explanations.
- Add section banners or `Arrange`/`Act`/`Assert` labels to readable code.
- Commemorate fixes, leave commented-out code, or add speculative TODOs and reviewer conversations.
- Use comments to excuse confusing names, missing types, tangled control flow, or weak boundaries.

Before commenting, improve names and types, simplify control flow, extract a cohesive operation, move behaviour to its owner, and remove unnecessary cleverness. Public visibility alone does not require documentation.

## Frontend boundaries and primitives

UI contracts include semantics, accessibility, responsive behaviour, and all interaction and data states. Shared product concepts need stable owners.

**Do:**

- Extract meaningful subcomponents, styles, and view-model logic as complexity grows, even before reuse appears.
- Give repeated controls, panels, modals, rows, tabs, badges, warnings, loading, and empty states named application primitives with semantic APIs.
- Keep pieces easy to discover and review.

**Don't:** split files mechanically, accumulate unrelated UI responsibilities in one file, or recreate an existing primitive's appearance locally.

## UI copy, icons, and headings

Claude's biggest UI writing weakness is word choice: literal, awkward, abstract, or technical words where a person expects natural language. Short, accurate text can still be a poor label; it often describes the mechanism instead of what the action means to the user.

**Do:**

- Write from the user's task and vocabulary: what they want to do, what will happen, and what they need to decide. Internal names are not product copy.
- Prefer familiar, concrete words ("Edit", "Share", "Sign out", "Refresh") over formal or implementation-derived ones ("Modify entry", "Configure access permissions", "Terminate session", "Invalidate cache").
- Label actions with concise verbs and, where needed, their object; use one name per concept across the product.
- Describe status as user progress and outcome ("Preparing your report…", "Couldn't upload the file. Try again."); wording must match actual behaviour, such as queued versus done.
- Choose deliberately between text, icon, and both; use a familiar icon from the product's set for established actions, and give icon-only controls an accessible name.
- Review copy in the rendered screen, including loading, empty, and error states, as part of the UI conventions or implementation skill rather than an optional polish pass.

**Don't:** surface internal operations or diagnostics as labels, rely on a tooltip to rescue an incomprehensible control, or add headings such as "Actions" or "Details" that repeat what the layout or content already makes obvious. Keep meaningful semantic heading structure and necessary instructions.

See [UI and UX Audits](ui-ux-audits.md#copy-icons-and-information-hierarchy) for reviewing existing interfaces.

## Semantic tokens and site-wide constants

Names encode meaning; values implement it. Separate roles may share a value today and diverge later.

**Do:**

- Name tokens for product intent and domain concepts.
- Centralise site-wide typography, spacing, layout rhythm, borders, separators, corners, surfaces, elevation, icons, motion, and state styles.
- Keep unrelated semantic roles distinct even when their colours match.

**Don't:** reuse `danger` for an unrelated red domain state, scatter shared decisions through components, or equate tokenising literals with completing a design system.

Use [Design-System Refactors](design-system-refactors.md) for inventory, component convergence, migration sequencing, and rendered verification.

## Structured logging (aka "auditing")

Logs should be queryable events with stable labels and structured runtime data.

Logs should be both human and machine-readable.

Think of your log files as a kind of append-only database of events, errors and state changes.

A log line should consist of a message (aka "a label") followed by some data (a JSON blob).

**Do:** put runtime values in a JSON payload, emit label and payload on one line, and use consistent field names.

**Example log call:** use the equivalent API provided by the repository’s logger.

```ts
logger.debug("item change observed", { item: foo, count });
```

**Don't:** interpolate runtime values into the message, mix prose and data, or emit multiline dumps.

## HTTP and APIs

Response correctness includes relevant protocol behaviour as well as the body.

**Do:** follow established content negotiation, compression, headers, cache-control, freshness, validators, and conditional-request conventions. Ask which apply when the product or infrastructure does not make them clear.

**Don't:** add every protocol feature mechanically or defer required behaviour merely because development requests return the right JSON.

## Exceptions and diagnostic context

Let unrecoverable failures surface. Catch at an application boundary or where code can genuinely recover, translate, or clean up.

**Do:** preserve the original error and stack, attach useful structured input context, and log once at the boundary responsible for reporting or terminating the failure.

**Don't:** catch-log-and-continue in invalid state, discard stacks, duplicate logs across layers, or dump sensitive or excessive data. Ask before inventing a different exception policy.

## Failure handling and forbidden shortcuts

Fail visibly when a required contract is broken. Resilience must implement a defined product requirement; it must not make incorrect behaviour appear successful.

**Don't:**

- Add broad `try/catch`, swallow errors, or log and continue without restoring a valid state.
- Replace failed operations with empty collections, fabricated results, or success-shaped responses.
- Default missing required values to `0`, `""`, a placeholder identifier, or guessed configuration. Avoid optional chaining and null coalescing that conceal a broken invariant.
- Add fallback providers, alternate paths, or compatibility branches merely to make a failure disappear.
- Retry blindly, indefinitely, or around the wrong failure class; repeat non-idempotent operations without duplicate protection.
- Use vague or interpolated logs, discard the original error/stack, or log the same failure at every layer.
- Weaken validation, types, assertions, or tests to make the workaround pass.

**Do:**

- Validate required inputs and configuration at their boundary; represent legitimate absence explicitly.
- Fix the source of a violated invariant or the abstraction that owns it.
- Catch only for supported recovery, boundary translation, or required cleanup; otherwise propagate the failure.
- Use defaults only for genuinely optional values with defined semantics.
- Use fallbacks only when the contract defines their trigger, acceptable result, and observability.
- Retry only identified transient failures under an established policy: bounded attempts/time, appropriate backoff, cancellation, and idempotency or duplicate protection. Surface final failure.
- Test relevant missing, malformed, unavailable, and exhausted-retry paths; check resulting state and diagnostics.
- Ask for a policy decision if no approved recovery behaviour exists.

Reject these shortcuts in review even when tests pass. Put the concise prohibition in the repository-wide contract and keep the detailed rules in the applicable convention.

## Mechanical formatting

The repository should determine formatting through configuration and repeatable commands. [dprint](https://dprint.dev/) is one option; use the formatter appropriate to the stack.

**Do:** configure the formatter, expose apply/check commands, format touched files after edits, and check in CI.

**Don't:** rely on prose for exact formatting or make a repository-wide rewrite as a side effect of a local edit.

## Global versus local conventions

Keep naming, imports, file organisation, error shape, testing expectations, and repository-wide approval rules global. Keep subsystem layouts, API shapes, component state, and database-access details local. Load each before the work it governs.
