# UI and UX Audits

An audit produces evidence for triage. Keep implementation unchanged unless fixing is explicitly in scope, and distinguish confirmed defects from risks, preferences, and disproved concerns. Use [Design-System Refactors](design-system-refactors.md) for approved migrations.

## Deliverable and evidence

Report scope, findings ordered by impact, reproducible method, cleared concerns, and residual risk.

**Do:**

- Name surfaces, routes, states, viewports, appearances, locales, and platforms included and excluded.
- Support each finding with a file/symbol/selector and a measurement, reproducible interaction, or violated rule.
- State the consequence for users, assistive technology, translators, operators, or maintainers.
- Trace inputs or state producers to prove reachability.
- Classify fixes as visible, behavioural, or structural.
- Start findings with one concise line and add only supporting evidence.

**Don't:**

- Fix findings during a read-only audit.
- Treat nullable types or hypothetical inputs as proof of a reachable defect.
- Call a preference or intentional difference a defect.
- Equate standards conformance with overall usability.
- Claim rendered behaviour from source inspection alone.

## System ownership

Count in-scope implementations rather than reading representative samples. Test the system claim: can each site-wide colour, type, spacing, border, corner, shadow, icon, and motion decision change once and reach every applicable surface?

Where practical, temporarily change an owner and observe propagation. Preserve unrelated work, restore the experiment completely, and record exceptions.

| Inventory | Look for |
| --- | --- |
| Colours, type, spacing, borders, radii, shadows, opacity, motion, stacking, breakpoints, assets | Raw shared values, duplicate or near-match tokens, appearance-based names, semantic misuse, dead tokens, undeclared references, concealing fallbacks |
| Controls, panels, rows, badges, statuses, disclosures, tooltips, menus, skeletons, empty/error states | Duplicate recipes, missing owners, primitive bypasses, disagreements, external overrides, style injection, third-party UI outside its boundary |

Search by rendered structure, property groups, labels, inputs, and behaviour as well as names. Record copies and disagreements.

**Don't:** classify every literal, fallback, or native element as a defect. Local geometry and resilience can be valid; a native element is a bypass only where an applicable shared contract loses required ownership, styling, behaviour, or accessibility.

## Runtime checks

Exercise every in-scope route and reachable state. Automate repeatable measurements and inspect rendered results directly.

| Concern | Check |
| --- | --- |
| Contrast | Effective foreground/background after alpha and state effects; criterion, ratio, threshold, and applicable exceptions |
| Pointer targets | Interactive area, spacing exceptions, hover-only or precision-only access, and undiscoverable gestures |
| Keyboard and focus | Order, indication, names/roles, pattern keyboard maps, overlay entry/containment/restoration/dismissal, missing or spurious stops |
| Reflow | Supported viewport and text sizes, scrolling, clipping, overlap, fixed dimensions, safe areas, and unreachable controls |
| Consistency | Computed spacing, geometry, colours, and typography across surfaces sharing a rule |
| Layout | Composed flex/grid/table/overflow/containment/position/transform/stacking effects |
| Diagnostics | Console errors/warnings, failed or duplicate requests, repeated event registration, and interaction failures traced to owners |

Use the product's chosen accessibility standard and conformance level. Apply [non-text contrast scope and exceptions](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast) accurately; distinguish [minimum](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum) and [enhanced](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced) target policies. Report usability concerns outside conformance separately. A resized desktop browser may not reproduce device layout or input.

## States and meaning

**Do inspect:**

- Default, hover, focus, active, selected, and disabled states.
- Loading, empty, partial, success, failure, retry, cancellation, and timeout.
- Narrow/wide layouts, appearances, text sizes, locales, reduced motion, high contrast, and supported accessibility settings.
- Shared ownership of state styling and blocking overlays that leave covered controls operable.
- Localisation, plurals, numbers, currency, dates, and accessible names for controls, fields, images, regions, and scrollable areas.
- Correct elements, roles, relationships, and keyboard models.
- Text available only through images, icons, tooltips, or CSS-generated content.
- Testable business/formatting logic outside templates and rules duplicated across boundaries.

## Copy, icons, and information hierarchy

Read labels, buttons, headings, help text, and status messages in rendered context against the [UI copy conventions](engineering-conventions.md#ui-copy-icons-and-headings).

**Do inspect:**

- Poor word choice first: awkward, formal, abstract, or unfamiliar terms where natural task language would be clearer, even when concise and accurate.
- Internal operation names or implementation details that make users infer the action or outcome.
- Labels that misrepresent what happens, such as implying completion when work is only queued.
- Inconsistent names for the same concept.
- Text-heavy controls where an established icon would be clearer, and icon-only controls that are ambiguous or lack an accessible name.
- Redundant subheadings that repeat nearby content or fragment a simple task.

Record the exact wording, context, audience, and consequence, and what an alternative would clarify. Distinguish demonstrated confusion, misleading feedback, or a convention violation from an editorial preference.

## Validate and triage

Before reporting a defect, reproduce it or prove why a rule cannot take effect, confirm reachable state and applicable ownership/specification, quantify it where possible, and state fix visibility. Label unresolved reachability as risk and seek independent review when classification is contested.

**Example finding format:** adapt the fields to the report’s needs.

```md
### <Defect and consequence>

**Evidence:** <File/symbol/selector; reproduction or measurement.>
**Consequence:** <Who is affected and how.>
**Fix visibility:** <Visible, behavioural, or structural; reason.>
**Excluded:** <Adjacent work outside this finding.>
```

Use exact counts and one defect per entry. State what evidence would resolve uncertainty. Order findings by impact: broken behaviour, accessibility barriers, security/privacy/destructive risks, visible inconsistency, ownership drift, and untested logic. Keep observations, preferences, and cleared concerns separate.

Finish with inspected coverage, exclusions/unreachable cases, category counts, restored experiments, checks/runtime paths, and remaining uncertainty. Do not claim completeness from samples or report formatter-owned differences as defects.

## Automatically routed skill

Keep the reusable method in a focused skill and repository-specific routes, accounts, commands, outputs, matrices, and thresholds in owned references.

**Example skill:** adapt the scope, triggers, and reporting steps to the project.

```md
---
description: Use for read-only UI/UX audits, accessibility reviews, visual drift, responsive failures, and design-system bypasses. Excludes implementation-only work and isolated styling corrections.
---

# UI/UX Audit

1. Define scope, exclusions, output, and accessibility standard.
2. Count routes, states, shared owners, values, duplicates, and escapes.
3. Exercise in-scope runtime paths and capture reproducible evidence.
4. Verify states, semantics, localisation, copy, icon choices, and heading usefulness.
5. Validate reachability, consequence, semantics, and fix visibility.
6. Separate defects, risks, preferences, and cleared concerns.
7. Report impact-ordered findings, counts, method, and bounded coverage.
```
