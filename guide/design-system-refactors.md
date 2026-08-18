# Design-System Refactors

Design-system work is not a styling sweep. It is a cross-cutting refactor of the product's visual semantics, shared component boundaries, platform bindings, and verification model. Claude is especially likely to get this work wrong because a locally plausible token substitution can look like progress while leaving the repository with two competing UI languages.

This chapter is derived from a real multi-target application that retrofitted a token layer and shared component boundary across several user-facing surfaces without an existing UI snapshot suite. Its conclusions were checked against another application whose semantic token table was already mature but whose adaptive layouts and rendered-evidence system were under active pressure. The detailed [failure catalogue](https://github.com/npomfret/no-spoilers/blob/main/FACTORING-A-DESIGN-SYSTEM.md) remains the living evidence base; this chapter extracts the reusable operating rules.

## Start with a falsifiable outcome

"Introduce a design system" is not a completion criterion. Define the smallest set of one-edit tests that the finished system must pass. For example:

> Change the accent colour, type scale, corner radii, spacing rhythm, motion, and icon set once each, and have every applicable surface follow.

The exact axes depend on the product. The important properties are that the statement:

- can be scored before, during, and after the work;
- scopes the work by product outcome rather than by an assumed file list;
- names every surface expected to follow;
- and makes unrelated token or component work visibly out of scope.

Do not begin from a representative reading of the likely files. Count first. Inventory raw values, centralised constants, semantic tokens, shared components, strings, formatters, assets, and every rendering surface. Include the places that are easiest to miss: loading skeletons, placeholders, previews, empty states, widgets, extensions, menu-bar roots, and web bindings.

Record the baseline as numbers. The initial counts become acceptance criteria and expose categories that prose inspection misses. Keep a documented-literal escape hatch for values that should remain local, but require the code to say why they are not tokens.

An escape checker is valuable, but score it against the claim it actually proves. A rule that rejects raw colours, fonts, spacing, or radii outside the token layer proves that call sites route through the intended boundary. It does not prove that changing a token reaches every applicable surface, that correlated values are used together, that a shared component has not been reimplemented, or that the resolved result is readable. Keep routing compliance and rendered outcome as separate acceptance criteria.

## Establish the confidence baseline

Before changing the structure, record three kinds of confidence separately:

| Confidence | What establishes it | What it does not establish |
| --- | --- | --- |
| Build | Every affected target compiles | Correct behaviour or rendering |
| Behavioural | Tests or deterministic checks pin observable behaviour | Visual fidelity |
| Visual | Validated captures, snapshots, measurements, or manual inspection cover named surfaces and states | Uncovered targets or states |

Never collapse these into one word such as "verified." If a surface has no capture path, state that as a residual risk every time status is summarised. Improved coverage mitigates the gap; it does not retroactively eliminate it.

Audit the verification system itself before trusting its green result. Record excluded accessibility checks, disabled device or orientation legs, unreachable screens, simulator-only assumptions, ignored targets, and capture helpers that skip work under some configurations. A suite described as an accessibility or visual audit may legitimately exclude noisy checks, but those exclusions become named coverage gaps rather than disappearing from the confidence statement.

## Model the target before migrating call sites

### Decide rebuild-time versus runtime variation

Ask whether the product changes its appearance at build time or while running. A rebuild-time reskin usually does not need environment injection, root modifiers, or runtime theme plumbing. Those mechanisms introduce silent defaults at every rendering root and become particularly fragile across widgets, extensions, and separately hosted UI.

Use the simplest model that implements the current requirement. If runtime selection may be needed later, record that revisit condition and the expected cost; do not build speculative machinery now.

### Name semantic roles before adding variants

Semantic roles are useful even when each role has only one value. Naming `textPrimary`, `surfacePanel`, or a domain state now prevents platform defaults and palette literals from remaining independent visual decisions. Add light/dark or brand variants only when the product actually needs them.

Name by meaning rather than appearance. Two roles may resolve to the same value today and still require separate names because they can diverge later. Conversely, similarly named roles in a platform vocabulary and a product vocabulary are not automatically equivalent; compare what they render, not what they are called.

### Model complete axes

Do not use a boolean when the domain has more than two cases. `compact: Bool` makes `false` and unspecified intent look identical and cannot express several distinct surfaces that happen to share dimensions today. Prefer a named enum whose cases reflect the real variation axis.

Optional parameters can reveal an underspecified axis. If callers compute whether a value should exist because the theme cannot distinguish two surfaces, refine the axis before making more component inputs optional.

When values must move together, return a correlated set rather than exposing independent lookups. Card radius, padding, fill, border, and shadow may form one geometry decision. Make invalid or incoherent combinations difficult to express.

Prefer, in order:

1. make invalid combinations unrepresentable in the type or component API;
2. keep a lookup inside the component that knows which cases are valid;
3. fail loudly at runtime only when modelling the subset would cost more than the invariant is worth.

Never invent a plausible fallback value for a case that has no design. Silent defaults turn accidental call sites into product decisions.

### Do not create speculative or dead tokens

A token needs demonstrated product meaning and real call sites. Do not manufacture a complete-looking scale from hypothetical future needs. Delete a token in the same change that removes its final consumer; unlike dead functions, dead constants frequently evade ordinary checks.

## Sequence the refactor to avoid doing work twice

A reliable default sequence is:

1. Add the minimum token definitions by transcribing what the product renders today.
2. Consolidate duplicated strings and formatters, with tests that pin any chosen behaviour.
3. Introduce semantic state colours as an isolated, explicitly visible change.
4. Merge the real variation axis before broad call-site work.
5. Converge genuinely duplicated components, one independently reviewable concept at a time.
6. Sweep remaining call sites onto tokens, one target or surface at a time.
7. Reconcile asset catalogs and other platform bindings that cannot reference code.
8. Reconcile web or other cross-format bindings.
9. Publish the as-built specification last, so every documented contract has an implementation.

The ordering is load-bearing. Sweeping call sites before component convergence edits code that may later be deleted. Writing the specification first encourages speculative tokens and undocumented gaps between aspiration and implementation.

Do not create a prolonged migration period. A convergence change should move every applicable caller and delete what it replaces in the same commit. If the new boundary is wrong, one revert should restore the previous state. Coexisting old and new paths teach future work that both remain valid.

Treat consolidation of near-duplicates as a behaviour or visual change until proven otherwise. Slightly different formatters, labels, colours, or spacing values may encode an intentional decision or an old accident; the refactor must discover which before selecting one.

## Converge components by structure, not resemblance

Read every implementation before designing the shared one. Components that display the same information may have fundamentally different structures. A shared implementation whose body switches into a separate layout for every surface has usually added indirection without creating a useful primitive.

Extract only the stable common structure. Leave genuinely distinct compositions separate.

The default content boundary should be:

- the caller owns domain content, formatting choices, and optional content slots;
- the component owns structure, style, interaction behaviour, and accessibility behaviour.

A component that requires the domain model, a surface enum, and a formatting policy is probably owning too much. Hidden fallbacks such as `countdown ?? location` are another warning: substituting one semantic category for another is not graceful degradation. Make absence expressible instead.

Do not let majority vote overrule semantics, accessibility, an existing specification, or platform behaviour. Majority is a useful lowest-churn fallback only when the disagreement carries no stronger meaning. Record which rule resolved each convergence.

## Avoid adaptive-UI anti-patterns that bypass the system

Design-system refactors often expose layout and accessibility failures that a token inventory cannot see. Watch for these shapes while reading candidate components and their callers.

### Do not feed a measurement back into the tree that produced it

A geometry read becomes dangerous when it writes state used by spacing, padding, sizing, or candidate selection inside the measured subtree. During resize, rotation, platform text-scaling changes, or animation, layout output then becomes input to the next layout while the first is still settling. The result may oscillate, re-enter layout, or hang at high CPU even though every individual value is valid.

Prefer a layout primitive that decides arrangement and leftover distribution from one proposal in one pass. If measurement is unavoidable, keep the value out of the subtree whose fit it can change, and prove convergence under the most disruptive supported resize—not only at a stable viewport.

### Do not nest adaptive candidate ladders casually

Candidate-based layout tools are appealing component boundaries because each component can list its preferred arrangements. Nested, their costs multiply: an outer component may measure every inner candidate for every outer candidate, for every repeated row. A locally cheap extracted component can therefore make its real composition many times slower.

Inventory adaptive boundaries as well as shared components. Benchmark the deepest realistic composition, with the largest supported collection and text size, before and after convergence. Prefer arithmetic or a single custom layout where the inputs are measurable and the arrangement is one coherent decision.

### Do not let local fitting cancel global semantics

Routing text through a semantic type role is not enough if a call site later applies a severe minimum scale factor, one-line truncation, or a fixed frame. Under pressure, the local modifier silently undoes the user's requested text size while the token checker remains green. Restack or wrap before shrinking glance-critical content; if shrinking is genuinely acceptable, make that part of the component contract and test its floor.

The same problem occurs with colour and state. A descendant may correctly choose a warning or selection colour only for an ancestor's grayscale, opacity, blend mode, or disabled treatment to erase it. In many UI frameworks, effects applied by an ancestor cannot be undone reliably by a descendant. Apply state treatments at the smallest boundary that owns the meaning, or return a correlated style that says which content retains identity and which recedes.

### Keep a reusable component inside its proposed bounds

A child that sizes itself from a distant container can bypass padding and frames introduced by its reusable parent. The arithmetic may return the same nominal width and still change which ancestor the width was measured against, producing overflow only in the composed screen. Prefer the immediate layout proposal. If a component deliberately escapes an intermediate content constraint on large canvases, model that as an explicit capability with a true no-op below its threshold, not as a container-sized fallback that still overrides intermediate constraints.

### Visual occlusion is not accessibility occlusion

An opaque overlay, launch curtain, skeleton, or transition does not automatically hide the accessibility elements underneath it. Hiding the overlay itself can leave invisible controls focusable and actionable. Treat accessibility presentation as part of the shared component boundary. If the overlay is purely decorative and does not block interaction, keep it out of the accessibility tree. If it blocks interaction, hide or make the covered content inert and expose an accessible progress, status, or dismissal control as appropriate.

## Verify neutral and visible changes differently

Split commits along the verifiability line. A mechanically neutral substitution and a deliberate pixel change should not share a commit merely because they touch one target. Reviewers should be able to prove or revert one without losing the other.

Useful deterministic checks include:

- **Value-multiset proof:** extract removed literals, resolve added tokens, and compare the multisets when the claim is pure renaming.
- **Resolved-rule comparison:** parse CSS or equivalent bindings, resolve variables transitively, and compare declarations before and after.
- **Exact arithmetic:** compute contrast, alpha compositing, dimensions, or timing relationships instead of estimating them visually.
- **Cross-format checks:** compare code, CSS, and asset-catalog copies that cannot share a compiled source of truth.

Visual verification also needs discipline:

1. Capture the same screen twice before the change to establish the tool's noise floor.
2. Name the surfaces, viewports, states, and appearance modes being covered.
3. Predict the expected pixel or geometry change numerically before capturing the result.
4. Validate that the application is present and in the intended state; a successful command and a PNG file do not prove a valid capture.
5. Compare the result with the prediction, accounting only for measured baseline noise.
6. Inspect captures directly even when automated checks pass.

"I opened it and nothing moved" is not visual evidence. One viewport, one state, and human memory do not establish neutrality.

Treat capture infrastructure as evidence-producing code. A screenshot named for a state must contain that state; existence of the target element below the fold is not enough. Wait for transitions to settle, derive orientation labels from the rendered canvas rather than from the requested device state, and prefer no capture to a confidently mislabelled one. If one pathological surface forces a device, orientation, appearance, or accessibility leg to be disabled globally, make restoring, replacing, or explicitly retiring that leg part of the blocking defect's completion criteria. Otherwise one local bug silently erases visual confidence for unrelated surfaces.

Use a risk-based surface matrix rather than the full Cartesian product of screens, states, viewports, appearances, languages, and accessibility settings. Each selected case should expose a distinct rule. Pairwise coverage can be a useful minimum where interactions are understood; add targeted combinations for known high-risk interactions. The matrix should still name the unselected combinations, so economy does not become accidental omission.

## Resolve divergent bindings with evidence

When Swift, CSS, asset catalogs, or other bindings disagree, do not begin by declaring one source authoritative. First determine whether they represent the same semantic element.

Use this order of evidence:

1. existing product semantics and accessibility requirements;
2. an element both bindings actually render;
3. measurements such as contrast, resolved colour, geometry, or timing;
4. an existing reviewed specification;
5. platform convention where it applies to the same intent;
6. majority only as a lowest-churn fallback.

Mapping by name is insufficient. A platform's `secondary` label may render closer to the product's tertiary role on the actual background. Composite and compare the values that users see.

Do not settle a two-value divergence by casually inventing a third value. Measurement should be allowed to change the available options—for example, keeping semantic colour on an accent and retaining primary text—rather than merely selecting one of the original candidates.

## Preserve the reasoning after the task ends

A task ledger is valuable working memory. Keep inventories, phase status, changed decisions, verification evidence, residual gaps, and open questions there while the refactor is active. Before deleting or archiving it, promote durable information into the layer that will be encountered when it matters:

- token values and invariants beside their definitions;
- observable behaviour in tests;
- the cross-platform contract in the as-built specification;
- non-obvious trade-offs in the relevant commit message or decision record;
- unavoidable cross-format duplication documented at both ends;
- and rejected approaches recorded with a concrete revisit condition.

"We decided not to" is not enough. Record why, what would need to change, and what the acceptable implementation should look like if the condition is later met.

Do not let the manual-review ledger become a permanent second backlog. Every entry should name why automation cannot settle it, the cheapest evidence that can, and the destination of any durable conclusion. Promote stable layout, contrast, state, and accessibility invariants into deterministic checks or named capture stories; keep subjective motion quality, hardware-only behaviour, and product judgement manual. Delete an entry only when its replacement evidence exists.

## Recommended automatically routed skill

Projects undertaking this work should create a focused skill rather than placing the whole failure catalogue in `CLAUDE.md`.

```text
.claude/skills/design-system-refactor/
  SKILL.md
  references/
    inventory.md
    token-modelling.md
    component-convergence.md
    visual-verification.md
    decision-record.md
```

The entry skill should be automatically discoverable from normal requests about design systems, theme or token migrations, reskins, UI consistency, semantic colours, component convergence, adaptive UI, or cross-platform visual alignment.

```md
---
description: Use automatically for design-system refactors, theme or token migrations, reskins, UI consistency work, semantic-colour changes, component convergence, and cross-platform visual alignment. Do not use for an isolated styling fix that does not change shared UI infrastructure.
---

# Design-System Refactor

1. Define and score the falsifiable one-edit outcome.
2. Inventory and count every category, surface, state, and binding.
3. Record build, behavioural, and visual confidence separately; audit exclusions and disabled coverage legs.
4. Decide rebuild-time versus runtime variation.
5. Model semantic roles, complete variation axes, and correlated values.
6. Read every candidate implementation and trace layout, effects, and accessibility across its composed callers.
7. Converge components before sweeping remaining call sites.
8. Delete replaced paths and newly dead tokens in the same change.
9. Split provably neutral substitutions from visible design decisions.
10. Verify with tests, arithmetic, structural comparisons, predicted visual diffs, and state-validated captures.
11. Name residual coverage gaps without overstating completion.
12. Restore, replace, or explicitly retire any coverage leg disabled by a blocking defect.
13. Publish the as-built specification and promote durable decisions.
```

The reference files should contain project-specific commands, paths, surface inventories, token semantics, screenshot procedures, and verification thresholds. The skill supplies the stable workflow; the project supplies the current facts.

## Keeping this guidance current

The source case study is a living document. Do not copy every narrative edit into this chapter. Reconcile only new or materially revised transferable conclusions, then update the provenance record below. If a new lesson changes the operational workflow, update the example skill in the same change so discoverability and guidance do not drift apart.

### Provenance

- Living source: [`no-spoilers/FACTORING-A-DESIGN-SYSTEM.md`](https://github.com/npomfret/no-spoilers/blob/main/FACTORING-A-DESIGN-SYSTEM.md)
- Source commit reviewed: `7b8dbdfddd4b67de6c327d01fe94b4183ba49632`
- Reconciled: 18 August 2026
