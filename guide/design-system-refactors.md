# Design-System Refactors

A design-system migration changes visual semantics, component boundaries, platform bindings, and verification. Count the current system, model its real variation, converge shared components, then migrate call sites. Use [UI and UX Audits](ui-ux-audits.md) for read-only assessment.

## Define the outcome and baseline

Choose a falsifiable product outcome. **Example:** changing accent colour, typography, corners, spacing, motion, and icons once each makes every applicable surface follow. Select the axes relevant to the product.

**Do:**

- Name the axes, surfaces, and completion checks before editing.
- Count raw values, constants, tokens, components, strings, formatters, assets, and bindings.
- Include skeletons, placeholders, previews, empty states, widgets, extensions, menu-bar roots, and web surfaces.
- Use baseline counts as acceptance criteria; document justified local literals.
- Audit excluded accessibility checks, disabled devices/orientations, unreachable screens, ignored targets, and conditional capture skips.

**Don't:** infer full coverage from representative files or call all confidence “verified.”

| Confidence | Evidence | Limit |
| --- | --- | --- |
| Build | Affected targets compile | Does not prove behaviour or rendering. |
| Behavioural | Tests or deterministic checks pin observable contracts | Does not prove visual fidelity. |
| Visual | Validated captures, measurements, or inspection of named surfaces/states | Does not cover unobserved cases. |

Keep residual gaps in status reports until evidence closes them. Generated-artifact synchronization proves freshness; escape checks prove routing through the token boundary. Neither proves complete propagation, coherent value combinations, component reuse, or readability.

## Model before migrating

**Do:**

- Decide whether variation is rebuild-time or runtime. Use runtime plumbing only for a current requirement; record future revisit conditions.
- Name semantic roles even when they have one value. Keep unrelated meanings distinct despite equal values.
- Model complete variation axes with named cases rather than ambiguous booleans or optional inputs.
- Return correlated geometry/style sets when radius, padding, fill, border, and shadow must agree.
- Prefer making invalid combinations unrepresentable, then component-owned lookup, then explicit runtime failure where richer modelling is disproportionate.
- Give each token demonstrated meaning and real consumers; delete it when its last consumer disappears.

**Don't:** invent plausible fallback designs, speculative scales, or platform-to-product mappings based on names alone.

## Migration sequence

1. Define minimum tokens by transcribing current rendered values.
2. Consolidate strings and formatters, pinning selected behaviour with tests.
3. Introduce semantic state colours as an isolated visible change.
4. Model the real variation axis before broad call-site work.
5. Converge genuinely shared components, one reviewable concept at a time.
6. Sweep remaining call sites by target or surface.
7. Reconcile asset catalogs and other platform bindings.
8. Reconcile web and cross-format bindings.
9. Publish the as-built specification.

**Do:** migrate all applicable callers and delete replaced paths in the same convergence commit, so one revert restores the old boundary.

**Don't:** sweep code that component convergence will delete, write an aspirational specification first, or call near-duplicate consolidation neutral before proving it.

## Component convergence

Read every candidate implementation. Extract stable common structure and keep genuinely different compositions separate.

**Do:**

- Let callers own domain content, formatting choices, and optional slots.
- Let components own structure, style, interaction, and accessibility.
- Resolve disagreements using semantics, accessibility, specification, and platform behaviour; use majority only as a lowest-churn fallback.
- Enumerate and migrate loaded, loading, empty, error, and small-screen paths.
- Trace every element using moved styles to its new owner.
- Drive states deliberately: hold requests open, fail them, and empty the data. Name unreachable states as coverage gaps.

**Don't:** create a shared component with a separate layout for every surface, require unrelated domain and formatting policies, or silently substitute one semantic category for another.

## Styling boundaries

Shared-component appearance must be controlled by its owner. Inventory overrides alongside tokens.

**Do:**

- Represent state and appearance through named semantic variants.
- Allow caller-owned layout inputs only where the boundary genuinely permits them.
- Use an intentional custom property for host-controlled size where appropriate, without competing component specificity.
- Let parents own spacing between children; keep outgoing margins out of reusable components.

**Don't:**

- Target a shared component with `!important`.
- Inject classes or styles to bypass state or appearance policy.
- Reach into internal markup with descendant selectors.
- Re-pack a primitive with feature-local media queries.
- Expose raw CSS colours, radii, pixel sizes, or durations as component design decisions.

Treat these escapes as API gaps and count remaining instances.

## Semantics, accessibility, and responsive behaviour

A token-compliant component can still have the wrong structure or interaction contract.

**Do:**

- Use semantic table, list, and control structures, valid relationships, and appropriate roles.
- Implement keyboard maps and state attributes for tabs, comboboxes, disclosures, sorting, and other interaction patterns at the shared owner.
- Check responsive behaviour in composed callers, including large text and narrow layouts.

**Don't:** imitate a semantic structure with generic containers, use roles that hide meaningful content, or call click-only markup a complete interaction pattern.

### Adaptive layout

**Do:**

- Prefer a layout primitive that decides arrangement in one pass.
- If measurement is unavoidable, keep it from changing the measured subtree's fit and prove convergence under disruptive resize, rotation, text scaling, and animation.
- Inventory nested adaptive candidate ladders and benchmark the deepest realistic composition with large collections and text sizes.
- Prefer arithmetic or one coherent custom layout where inputs are measurable.
- Restack or wrap before shrinking important text; test any permitted scaling floor.
- Size reusable children from their immediate proposal; model deliberate constraint escape with a true no-op below its threshold.
- Apply opacity, grayscale, blending, and disabled treatments at the smallest boundary owning the meaning, or use correlated styles that preserve identity where required.

**Don't:** feed layout output back into its own inputs, casually multiply candidate measurements, let fixed frames/truncation undo semantic text sizing, or let ancestor effects erase meaningful descendant state colours.

### Pending states and overlays

**Do:**

- Acknowledge actions immediately and keep input, animation, navigation, and accessibility responsive.
- Use indeterminate progress, measurable progress, or skeletons according to the operation.
- Model loading, empty, partial, success, failure, retry, cancellation, and timeout as explicit states.
- Preserve context, prevent duplicate submissions, and offer cancellation where safe and useful.
- Announce important progress and completion to assistive technology.
- Keep decorative overlays out of the accessibility tree; make covered content inert when an overlay blocks interaction and expose appropriate status or dismissal.
- Move I/O, decoding, database work, and expensive computation off the UI thread. Make async thread ownership explicit and test slow/failed paths with responsiveness monitoring.

**Don't:** assume visual occlusion hides underlying accessible controls or make waiting resemble a frozen interface.

## Verification

Separate provably neutral substitutions from visible decisions in independently reviewable commits. Quantify even small value changes.

**Deterministic checks:**

- Compare removed-literal and resolved-token multisets for pure renaming.
- Parse bindings and compare transitively resolved declarations.
- Calculate contrast, alpha compositing, dimensions, and timing exactly.
- Compare unavoidable code/CSS/asset-catalog copies across formats.

**Rendered and behavioural checks:**

1. Capture the baseline twice to establish noise.
2. Name selected surfaces, states, viewports, appearances, and accessibility settings.
3. Predict pixel or geometry changes numerically.
4. Validate the running application and actual captured state; wait for transitions and derive orientation from the canvas.
5. Compare against predictions and measured noise, then inspect captures directly.
6. Drive baseline and changed builds side by side with the same script for keyboard order, focus, targets, announced states, and input sizing.

**Don't:** infer neutrality from a mechanical diff, an existing PNG, one viewport, or memory of how the screen looked. Explain count changes rather than treating every change as regression or improvement.

Use a risk-based matrix: select cases exposing distinct rules, use pairwise coverage where interactions are understood, and add known high-risk combinations. Name unselected cases. Restore, replace, or explicitly retire coverage legs disabled by blocking defects.

## Divergent platform bindings

First prove the bindings represent the same semantic element, then use this evidence order:

1. Product semantics and accessibility requirements.
2. An element both bindings actually render.
3. Resolved contrast, colour, geometry, or timing measurements.
4. Reviewed specification.
5. Applicable platform convention.
6. Majority as the lowest-churn fallback.

**Don't:** declare one language authoritative, map by name alone, or invent a third value merely to settle a disagreement. Measurements may justify a different arrangement, such as keeping primary text and moving semantic colour to an accent.

## Preserve durable decisions

Keep a working ledger of inventory, phases, decisions, evidence, gaps, and open questions. Before archiving it, move durable knowledge to its maintained home:

- Values and invariants beside definitions.
- Behaviour in tests and cross-platform contracts in the as-built specification.
- Trade-offs in decision records or useful commit messages.
- Unavoidable cross-format duplication documented at both ends.
- Rejected approaches with reasons, concrete revisit conditions, and the acceptable future shape.

Keep out-of-scope findings explicitly unapproved and do not act on them. Each manual-review item should name why automation cannot settle it, the cheapest evidence that can, and its eventual destination. Promote stable layout, contrast, state, and accessibility checks into automation; retain subjective motion, hardware-only behaviour, and product judgement for manual review. Remove ledger entries only when replacement evidence exists.

## Automatically routed skill

Package this workflow outside root `AGENTS.md`, with repository-specific commands, inventories, capture procedures, and thresholds in owned references.

**Example layout:** the skill name and reference files illustrate one arrangement; use only the references the project needs.

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

**Example skill:** adapt the triggers and migration steps to the approved scope.

```md
---
description: Use for design-system refactors, token/theme migrations, reskins, UI consistency, semantic colours, component convergence, adaptive UI, and cross-platform alignment. Excludes isolated styling fixes that do not change shared infrastructure.
---

# Design-System Refactor

1. Define the falsifiable outcome and counted baseline.
2. Audit verification coverage and record separate confidence levels.
3. Model current variation, semantic roles, and correlated values.
4. Read candidate implementations, callers, states, and ownership escapes.
5. Converge components before sweeping tokens; remove replaced paths.
6. Verify neutral and visible changes separately, including runtime states.
7. Report coverage gaps and keep adjacent work explicitly unapproved.
8. Publish the as-built contract and preserve durable decisions.
```
