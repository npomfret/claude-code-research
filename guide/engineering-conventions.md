# Engineering Conventions

### Conventions are the anti-drift system

This is the most important section in the guide.

If Claude is sloppy by default, then coding conventions are not style preferences. They are the infrastructure that prevents sloppiness from accumulating. Anthropic's best-practices and memory guidance point in this direction but stop short of making it a complete engineering system. This guide makes it explicit: conventions are the primary quality mechanism.

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
- formatting expectations,
- test placement and style,
- mocking rules,
- migration patterns,
- generated-code boundaries,
- and any domain-specific invariants that must stay consistent.

If a topic can drift, it needs a convention.

Language-specific and framework-specific rules are especially high leverage. Claude's laziness often shows up as "good enough for this file" code that violates the idioms of the language or framework the rest of the codebase is using. Explicit language-level rules are a strong way to constrain that lazy but enthusiastic behavior before it spreads.

Examples:

- TypeScript: when to use unions vs classes, where runtime validation happens, how async errors are represented, and which import style is canonical
- React: state ownership, effect usage, data-fetching shape, and component boundary rules
- Go: package layout, error wrapping, interface usage, and when helpers should stay local
- Python: module structure, typing expectations, exception boundaries, and how side effects are isolated

#### Type Safety as a Design Tool

Claude is also too lazy about type safety in languages and codebases that support it well. Its default behavior is often to satisfy the immediate local problem with weaker typing than the codebase should tolerate: loose shapes, partial typing, untyped boundaries, `any`-style escapes, or values that could have been made precise but were left vague to get the task over the line.

That is backwards. In typed codebases, strong type information is one of the main tools that keeps drift, ambiguity, and bugs under control. Good teams treat type safety like a useful virus: once a part of the system is made precise, that precision should spread outward through the surrounding code instead of stopping at one file boundary.

The convention should push Claude toward that posture:

- add types aggressively in typed languages instead of treating them as optional polish
- tighten types at boundaries and let that precision propagate through the call graph
- prefer making invalid states unrepresentable when the language and design make that practical
- use the compiler as a design assistant and verification tool, not as an obstacle to work around
- avoid lazy escape hatches unless there is a clear, deliberate reason and the project accepts it

The important mindset is that the compiler is helping. If a type-safe language or codebase gives you a chance to let the type system carry more of the correctness burden, Claude should normally take that chance. Otherwise it leaves the project with the worst of both worlds: the ceremony of a typed codebase and the safety level of an untyped one.

#### Abstractions, Encapsulation, and the Helper Smell

Claude is also too eager to throw "helpers" at a design problem. That is one of its laziest habits. Instead of stepping back, examining the shape of the code, and improving the owning abstraction, it will often patch over the discomfort by adding another helper function, wrapper, or utility file near the problem.

That usually means one of two things:

- the design was not ready for the new requirement and should have been refactored first
- the abstraction boundary is wrong and Claude chose a convenience patch instead of fixing it

Helpers are not forbidden. The problem is helper-first thinking. When helpers become the default response, they are often a sign that corners were cut and encapsulation was not taken seriously enough.

The convention should push Claude toward the harder-looking but usually better move:

- look at the owning abstraction before adding a helper beside it
- improve encapsulation at the real boundary instead of scattering utility logic around the edges
- refactor the structure until the new behavior has an obvious home
- treat a new helper as a choice that needs justification, not as the automatic safe option

In practice, code is almost never "already ready" for the next requirement. It has to be grown into readiness through repeated inspection and refactoring. Claude should be trained to look, consider, design, and refactor before it writes the next line of implementation code. Throwing a helper at the problem is often just a way to avoid that work.

#### Duplication, Redundancy, and the Code Tax

Claude also needs explicit pressure against leaving extra code behind. It often misses an existing implementation and re-creates it, or it completes a refactor but leaves the old path, wrapper, branch, helper, or partially superseded code lying around "just in case."

That is not harmless. Every line of code carries a maintenance cost. Every duplicate branch, redundant wrapper, stale helper, and unused file increases the code tax the team has to pay forever after.

The convention should be simple:

- prefer less code when less code preserves the intended behavior
- when replacing a code path, remove the old one unless there is a real compatibility reason not to
- do not leave dead code, unused helpers, redundant branches, or superseded implementations behind after a change
- if Claude finds existing code that already solves the problem, it should reuse or consolidate it instead of reimplementing it nearby

Almost always, less code is better than more code that does the same job. The burden of proof should be on keeping the extra code, not on deleting it.

#### Comments Are an Exception, Not a Substitute for Clear Code

Claude often adds comments that merely narrate the code immediately below them. Those comments do not improve understanding; they add text to maintain, drift out of date, and make a well-structured implementation harder to scan.

The default should be self-explanatory code. Prefer clear names, small coherent functions, direct control flow, and well-chosen types and abstractions over comments that translate syntax into prose. Do not add comments just because a file or function was changed.

Comments are appropriate when they preserve information the code cannot express clearly on its own, such as:

- a non-obvious constraint, invariant, or external-system quirk
- an intentionally unusual implementation chosen for a documented trade-off
- a surprising edge case, workaround, or performance/security concern
- a decision that would otherwise invite a future contributor to "simplify" the code incorrectly

When a comment is justified, write the reason and the constraint, not a line-by-line description of the mechanism. If the code is unclear and no such context exists, refactor it until it is clear instead of explaining unclear code with a comment.

#### Frontend File Boundaries

Frontend code deserves an explicit warning because Claude is particularly bad here. Left unguided, it will happily pour HTML, CSS, TSX, local state, helper functions, and one-off subviews into a single large component file no matter how complex the screen becomes. That is one of its default failure modes.

The deeper issue is that Claude often does not respect UI code as architecture. It treats frontend work like prototype presentation glue: get the screen to look roughly right, put the CSS near the component that needs it, and move on. That is the wrong posture for a real product. UI code has contracts just like backend code: component APIs, design tokens, semantic naming, accessibility behavior, responsive layout rules, interaction states, loading and error states, and brand consistency. A local CSS value or one-off TSX shape is not harmless just because it is visual. It becomes part of the product's language.

A serious frontend convention should push in the opposite direction:

- assume a non-trivial component may contain reusable or independently understandable parts
- extract meaningful subcomponents, styles, helpers, and view-model logic into dedicated files when complexity starts to rise
- prefer file shapes that make important UI pieces more discoverable elsewhere in the repo
- treat extraction as a readability and maintainability tool, not just a reuse optimization
- treat styling, layout, and interaction patterns as durable product infrastructure, not as disposable prototype code

The key point is not "split everything aggressively." The key point is that a growing UI file should not be Claude's resting state. Extraction has multiple benefits even before reuse happens: smaller files are easier to review, patterns are easier to discover with search, and future work is less likely to pile more logic into one oversized TSX file. If a component is becoming hard to scan, that is already enough reason to consider decomposition.

#### Application Primitives

Claude is reluctant to create application primitives. It often treats a repeated UI shape as "just styling" and patches the current screen locally instead of extracting the shared product concept. A serious setup should push the opposite direction: when containers, panels, buttons, icon buttons, modals, drawers, separators, warnings, empty states, loading states, clickable rows, tabs, chips, or badges recur across the app, Claude should consider whether the right move is a named primitive with a semantic API.

These primitives are not premature abstraction when the product already repeats the concept. They are how the application preserves UI/UX consistency, accessibility behavior, responsive behavior, interaction states, and future design flexibility. A local button style or one-off warning box may look cheaper in the current diff, but it teaches Claude and future contributors that the product language is optional.

The setup should name this explicitly. Use terms like `application primitives`, `product UI primitives`, or `design-system primitives`, and define where they live. The important part is not the label. The important part is that repeated UI behavior gets a stable home instead of being recreated screen by screen.

#### Frontend Semantic Tokens

Frontend semantics deserve another explicit rule: Claude is too eager to reuse visual styles by superficial appearance instead of by meaning.

This usually shows up in token and class reuse. Claude sees an existing style or token with the right visual output and reuses it even when the semantic meaning is different. For example, a codebase may have a red `danger` token used for errors or destructive actions. Later, another feature may need something visually red for a completely different domain meaning. Claude will often reuse the `danger` token because the color matches, even though the meaning does not. That is the wrong abstraction.

The convention should be semantic tokens first, implementation second:

- name tokens and styles for what they mean, not just how they look
- do not reuse an error or danger token for an unrelated domain concept just because the current color is similar
- allow two semantic tokens to resolve to the same presentational value when appropriate, but keep the semantic names distinct
- prefer domain-language naming in domain features, even when the current visual treatment overlaps with an existing utility

This matters because visual coincidence is not semantic equivalence. Two concepts may share a presentational value today and need different treatments later. If the code collapses both concepts into one token, future design changes become harder and the current code becomes less legible. Claude needs explicit guidance here because its default instinct is "reuse the style that looks right," not "preserve the meaning of the UI state in the naming layer."

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

#### Structured Logging

Logging deserves to be called out explicitly because Claude is reliably bad at it. This is not just a style issue. It is a data-quality issue.

Claude is trained on huge volumes of internet code, and internet logging examples are full of obsolete habits: interpolated strings, inconsistent field naming, multiline dumps, and messages that are half prose and half data. That style produces log files that are hard to filter, hard to aggregate, and unpleasant to query. A serious Claude setup should counter this with an explicit logging convention.

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

This one change is disproportionately valuable. Once the pattern is applied consistently, log files stop being messy text blobs and start acting like an audit stream of application activity. They can be filtered by label, aggregated by field, and queried without brittle string parsing.

If the codebase cares about observability, do not leave logging style to Claude's judgment. Write down the event-label-plus-JSON rule as a convention, add examples, and enforce it in review.

#### HTTP and API Behavior

APIs deserve another explicit warning. Claude often treats API work as "return the right JSON and move on," which means it forgets or postpones standard HTTP concerns that should usually be considered up front.

That includes things like:

- content negotiation and `Accept` handling where relevant
- compression such as gzip or whatever transport conventions the stack already uses
- HTTP response headers beyond the bare minimum
- cache behavior, cache-control policy, and freshness rules
- validators such as `ETag` and related conditional request support where appropriate

The exact choices depend on the product, traffic pattern, and infrastructure, so the guide should not prescribe one universal header set. The rule is more general: when building or changing an API, Claude should not assume the job ends at the response body shape. It should either build in the relevant protocol-level behavior or explicitly ask which HTTP conventions the project expects. Otherwise it will ship narrow endpoint logic that works functionally but ignores basic industry-standard concerns until later, when they are more awkward to retrofit.

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

The practical rule is simple: if something is broken, let it break loudly enough that it can be found and fixed. Silent recovery and local log-and-continue behavior often make systems less reliable, not more.

#### Exception Logging and Context

Logging exceptions has its own convention. Claude often logs only the error message and discards the stack trace. That is almost always the wrong tradeoff. When an exception is logged, the stack trace is usually the most valuable part because it tells you where the problem actually happened. A message without a stack trace is often just a complaint with no location data.

The triggering state is often just as important. Parameters, identifiers, and other relevant runtime context can be the difference between a fixable production failure and an untraceable mystery. When the language and platform allow it, Claude should preserve that context with the exception rather than discarding it or reducing it to a vague log line.

So the convention should say:

- do not strip stack traces when logging exceptions unless there is a very specific reason
- prefer passing the original error object through structured logging or the platform's native exception logging path
- attach the relevant runtime context or input state to the exception or structured log payload when the platform supports it
- include enough context to reproduce or diagnose the failure, but not so much that logs become a data dump or a security problem
- avoid logging the same exception repeatedly at multiple layers as it bubbles outward
- log once at the boundary that is responsible for reporting, handling, or terminating the failure

This is another place where Claude needs an explicit rule because its default instinct is to catch early, log a string, and keep going. That looks defensive in a diff and is often disastrous in production behavior.

#### Mechanical Formatting

Formatting deserves special treatment. Claude is not reliable at preserving exact formatting conventions over time, especially in mixed-language repos or codebases with very specific style requirements. Do not rely on prose alone here. Put formatting under mechanical control.

This is a practical recurring failure mode. Claude leaves behind small inconsistencies everywhere: quote style drifts, spacing changes from file to file, line wrapping becomes arbitrary, and the repo slowly accumulates formatting noise that has nothing to do with the task. Left alone, this creates review churn and makes real code changes harder to see.

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

This is why the "router plus on-demand skills" model matters. Global guidance stays short. Local detail is discovered when relevant.
