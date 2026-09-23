# UI and UX Audits

A UI/UX audit identifies evidence-backed defects without changing the implementation. Its output is a reviewable report that separates broken behaviour, accessibility failures, structural drift, and preferences.

Use this workflow for an existing interface that may contain inconsistent styling, duplicated components, inaccessible interactions, weak responsive behaviour, or unowned design decisions. For implementation guidance after triage, use [Design-System Refactors](design-system-refactors.md).

## Define the deliverable

Produce one report containing:

- **Scope:** surfaces, routes, states, viewports, appearance modes, locales, and platforms included or excluded.
- **Findings:** one evidence-backed defect per entry, ordered by impact.
- **Method:** tools, queries, measurements, and interaction paths needed to reproduce the audit.
- **Cleared concerns:** suspected problems that inspection or measurement disproved.
- **Residual risk:** relevant areas the audit could not reach or verify.

Do not fix findings during an audit unless the task explicitly includes implementation. Even an obvious correction changes the evidence, bypasses triage, and expands the requested scope.

## Apply strict evidence rules

Every finding must satisfy these rules:

1. **Name the evidence.** Cite a file and symbol or selector, plus a measurement, reproducible interaction, or violated rule. "This looks inconsistent" is not evidence.
2. **State the consequence.** Explain what a user, assistive-technology user, translator, operator, or maintainer experiences. A preference without a consequence is not a defect.
3. **Prove reachability.** Trace the input or state producer. A nullable type, fallback branch, or theoretical value does not prove that users can encounter the problem.
4. **Separate conformance from quality.** A standards exemption may still indicate usability debt, but report it accurately rather than calling it a conformance failure.
5. **Classify visibility.** State whether a fix would change rendered output or interaction behaviour. A silent structural correction and a visible redesign require different review.
6. **Keep findings concise.** Start with one line. Add only the evidence and explanation needed to support the claim.

Prefer runtime evidence for computed styles, contrast, reflow, focus order, target sizes, loading states, animation, and responsive layout. Source inspection alone cannot establish rendered behaviour.

## Use a falsifiable system test

For design-system structure, ask:

> Could each site-wide visual decision—colour, type, spacing, borders, corners, shadows, icons, and motion—be changed once and reach every applicable surface?

Test the claim where practical by temporarily changing one owner, observing which surfaces follow, and restoring the original value. Record the method and every applicable surface that did not follow. Treat the temporary change as an experiment: preserve unrelated work and verify its complete reversal.

## Audit system ownership

Build counted inventories rather than reviewing representative files.

### Tokens and visual values

Inventory colours, typography, spacing, borders, radii, shadows, opacity, motion, stacking levels, breakpoints, and assets. Record:

- raw site-wide values outside their intended owner;
- values duplicated beside an existing token;
- near-matches that indicate drift;
- tokens named by appearance rather than meaning;
- semantic roles used for unrelated concepts;
- unused tokens and undeclared token references;
- opacity used where a stable semantic colour is required; and
- fallbacks that conceal missing configuration or undeclared tokens.

Do not classify every literal or fallback as defective. Local geometry and deliberate resilience can be valid. Require a clear owner and rationale when the value affects a shared visual decision.

### Duplication and component ownership

Search by rendered structure and behaviour, not only by names. Compare property groups, markup shapes, labels, data inputs, and interaction patterns. For each duplicated recipe, record the copies and where they disagree.

Inspect recurring concepts such as controls, panels, rows, badges, status indicators, disclosures, tooltips, menus, skeletons, empty states, and error states. Flag:

- independent implementations of the same product concept;
- feature code bypassing an established primitive without a reason;
- callers overriding a primitive's internal markup or state styling;
- style or class injection that exposes decisions the primitive should own;
- third-party UI components entering feature code outside the integration boundary; and
- primitives reused for the wrong semantic meaning because their appearance matches.

A native element is not automatically a bypass. Report it only when an applicable shared contract exists and the element loses required styling, behaviour, accessibility, or system ownership.

## Audit the running interface

Exercise every in-scope route and every reachable state. Automate repeated measurements where possible, but inspect the rendered result directly.

### Contrast and colour

Measure rendered foregrounds against their effective backgrounds, including alpha compositing and state effects. Use the accessibility standard and conformance level selected by the product. Record the criterion, threshold, measured ratio, element, state, and any applicable exception.

Do not assume every visible boundary must meet the non-text contrast threshold or that an inactive control is covered by the same requirement. The [WCAG contrast guidance](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast) defines scope and exceptions. Usability concerns outside conformance can still be reported as such.

### Pointer targets

Measure the interactive target, not only its visible icon. Apply the product's stated target-size policy or the selected accessibility standard, including spacing and other exceptions. WCAG documents [minimum](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum) and [enhanced](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced) criteria separately.

Also identify interactions available only through hover, precision pointing, or an undiscoverable gesture.

### Keyboard and focus

Traverse each interaction path by keyboard and record:

- focus order and visible focus indication;
- accessible name and role;
- keyboard operation for the interaction pattern;
- focus entry, containment, restoration, and dismissal for overlays; and
- interactive elements missing from the focus order or inert elements included in it.

### Reflow and responsive layout

Test the supported viewport and text-size matrix. Record horizontal page scrolling, clipped or overlapping content, fixed dimensions that prevent reflow, unsafe-area failures, and controls that become unreachable. Confirm behaviour in a representative runtime; a resized desktop viewport may not reproduce device layout, input, or browser behaviour.

### Layout and rendered consistency

Compare computed values for surfaces expected to share a rule. Measure small differences in spacing, geometry, colour, and typography rather than judging them by eye.

Inspect layout declarations in context. Flex, grid, table, overflow, containment, positioning, transforms, and stacking contexts can change the meaning of otherwise valid declarations. Demonstrate the consequence in the composed interface before reporting a defect.

### Runtime diagnostics

Record console errors, warnings, failed requests, duplicate requests, repeated event registration, and interaction failures. Trace each symptom to its owner before assigning a cause.

## Audit states and meaning

For every interactive control and data-bearing region, inspect applicable states:

- default, hover, focus, active, selected, and disabled;
- loading, empty, partial, success, failure, retry, cancellation, and timeout;
- narrow and wide layouts;
- supported appearance modes, text sizes, and locales; and
- reduced-motion, high-contrast, and other supported accessibility settings.

Verify that shared owners define these states and that callers do not reimplement them with independent values. Confirm that loading and blocking overlays expose correct accessibility state and do not leave covered controls operable.

Audit meaning as well as rendering:

- user-visible text uses the localisation system and appropriate plural, number, currency, and date formatting;
- controls, images, regions, fields, and scrollable areas have appropriate accessible names;
- semantic structures use the correct elements, roles, relationships, and keyboard models;
- visible text does not exist only in an image, icon, tooltip, or CSS-generated content;
- business and formatting logic is testable outside presentation templates; and
- the client does not independently recreate rules owned by another boundary.

## Validate each finding

Before including a finding:

1. Reproduce it in the running interface or identify the exact rule that cannot take effect.
2. Prove the state is reachable, or label it as an unverified risk.
3. Check the applicable specification, convention, and code ownership before calling an intentional difference a defect.
4. Quantify the result with a ratio, dimension, count, duration, or call-site total where possible.
5. State whether fixing it changes visible or interactive behaviour.
6. Seek an independent review when the evidence or classification is contested.

## Write findings for triage

Use this shape:

```md
### <Sentence naming the defect and consequence>

**Evidence:** <File and symbol or selector; reproduction or measurement.>

**Consequence:** <Who or what is affected, and how.>

**Fix visibility:** <Visible, behavioural, or structural, with a short reason.>

**Excluded:** <Adjacent work deliberately outside this finding.>
```

Name exact counts rather than "several." Keep separate defects in separate entries. If uncertainty remains, state what evidence would resolve it.

Order findings by impact:

1. broken or unreachable behaviour;
2. accessibility barriers;
3. security, privacy, or destructive interaction risks;
4. inconsistencies already visible to users;
5. duplicated or missing ownership likely to create drift;
6. untested presentation or business logic; and
7. observations, preferences, and cleared concerns.

Keep the final category visibly separate so preferences do not dilute confirmed defects.

## Finish with bounded claims

The report must state:

- what was inspected and measured;
- what was excluded or unreachable;
- the count produced by each audit category;
- which temporary experiments were restored;
- which automated checks and runtime paths were used; and
- what uncertainty remains.

Do not claim a complete audit from sampled files, routes, states, or viewports. Do not report formatter-owned differences, consistently applied conventions that merely differ from personal preference, unreachable hypothetical inputs, or issues created only to justify a preferred redesign.

## Recommended automatically routed skill

Package this workflow as a focused, model-invocable skill rather than placing it in root `CLAUDE.md`.

```md
---
description: Use automatically when asked to audit, review, assess, or catalogue UI/UX defects, accessibility gaps, visual drift, responsive failures, or design-system bypasses without implementing fixes. Do not use for an implementation-only task or an isolated styling correction.
---

# UI/UX Audit

1. Confirm the audit scope, exclusions, output location, and applicable accessibility standard.
2. Keep the audit read-only unless implementation is explicitly requested.
3. Inventory routes, states, viewports, appearance modes, locales, and shared UI owners.
4. Count tokens, raw values, duplicated recipes, primitive bypasses, and ownership escapes.
5. Exercise every in-scope runtime path and collect computed, interaction, accessibility, responsive, console, and network evidence.
6. Verify state coverage, semantic structure, localisation, and presentation-layer logic.
7. Accept a finding only when it has evidence, reachability, and a concrete consequence.
8. State whether each fix would be visible, behavioural, or structural.
9. Separate confirmed defects, unverified risks, preferences, and cleared concerns.
10. Order findings by user impact, then structural risk.
11. Report the method, counts, exclusions, restored experiments, and residual uncertainty.
```

Repository-specific commands, routes, test accounts, output paths, target matrices, and thresholds belong in skill-owned references. The skill should contain only the reusable audit method.
