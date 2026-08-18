# Testing and Quality

The guide already argues for aggressive verification. In practice, many teams should go one step further and encode test-first behavior for at least non-trivial feature work and reproducible bug fixes.

Test-first work is useful here because it attacks two Claude failure modes at once:

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

This is the only reliable way to stop a large existing project from getting worse.

UI audits need special care because Claude is prone to doing a plausible sample pass and then overstating the result. For design-system questions, require category-complete scans before accepting a verdict. The audit should cover containers, boxes, panels, typography, spacing, borders, separators, corners, shadows, surfaces, icons, controls, colours, loading states, empty states, and error states. It should also distinguish four different things that Claude often blends together: raw values, centralised tokens, semantic tokens, and shared components. A value being tokenised is not enough. The token or component name should encode stable intent or domain meaning, not just current appearance. `danger`, `surface-panel`, `separator-subtle`, and `points-high` are semantic. `red`, `box2`, `border-light`, `canvas-2`, and `ink-2` are weaker unless the project has explicitly documented what they mean.

For a full design-system migration, extend this audit into a counted baseline and keep build, behavioural, and visual confidence separate. [Design-System Refactors](design-system-refactors.md) describes how to prove neutral substitutions, establish screenshot noise floors, predict visual diffs, and report uncovered surfaces without overstating completion.
