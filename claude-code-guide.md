# Claude Code Guide for Long-Running Projects

This guide is for maintained codebases, not demos or disposable prototypes. Without structure, Claude Code duplicates logic, patches features into code that first needs refactoring, and introduces pattern drift.

Anthropic's official documentation, collected at the end of this guide, is the source of truth for Claude Code capabilities.

## Operating Ethos

Give Claude room to explore, refactor, investigate, and use tools while constraining choices that create drift. It should inspect the codebase, improve weak areas, run checks, and follow evidence, but not invent architecture, coding styles, dependencies, testing patterns, or permission policies for local convenience.

A good setup makes preferred paths easy to find and weak paths hard to take. Skills, rules, agents, references, and commands help only when Claude can discover them from normal task wording. An orphaned Markdown file is not an effective convention or workflow.

Use skills and scoped rules to provide technical context without bloating always-on memory. Route them automatically when the task implies their use; users should not need to know a skill's name or request it explicitly.

Claude often takes the locally convenient route: patching weak structure, duplicating nearby patterns, preserving accidental behavior, weakening types, skipping tests, or using output-focused hacks. Counter this with explicit conventions, verification, approval gates, and mechanical formatting.

The setup needs three properties:

- **bounded freedom**: let Claude investigate, refactor for readiness, and use tools inside approved areas;
- **automatic discoverability**: make every important workflow, skill, rule, and specialist agent routable without special prompting;
- **tight conventions**: define the codebase's allowed shapes so Claude cannot quietly create new ones.

Freedom without conventions creates drift. Conventions without discoverability are ignored. Discoverability without freedom creates a bottleneck. Keep all three in force.

## Common Claude Failure Modes

If the setup does not actively counter these, Claude will keep doing them:

- It copy-pastes locally convenient logic instead of finding or extracting the shared abstraction.
- It moves code into helpers or interfaces without creating a meaningful abstraction, and exposes state for callers to coordinate instead of preserving encapsulation inside the owning object or module.
- It makes the smallest possible code change even when the surrounding structure is unready for the new requirement.
- It invents slight pattern variants because the first few files it read looked "close enough."
- It does not reliably look around for existing patterns before starting work, so it reinvents the wheel unless explicitly told to search upstream, downstream, and laterally.
- It sometimes over-engineers in the opposite direction by introducing speculative abstractions that the current codebase does not actually need.
- It avoids refactoring and test-first discipline unless forced to do them.
- During difficult bug investigations, it leaves unsuccessful instrumentation and speculative fixes in place, then stacks later conjectures on top until neither the evidence nor the final diff has a trustworthy baseline.
- It keeps every failing bug reproducer permanently, even when it records obsolete behavior, duplicates stronger coverage, or costs more than the enduring risk justifies.
- It buries the behaviour of integration and end-to-end tests under procedural setup, navigation, synchronization, and cleanup code instead of extracting those mechanics behind readable application drivers or equivalent test harnesses.
- It hides dependencies by constructing clients, repositories, clocks, configuration, or other external capabilities inside behavior code, making units difficult to instantiate and test in isolation.
- Even when it injects an external client, it lets vendor SDK methods, types, errors, and usage patterns spread through application code instead of containing them behind a narrow application-owned adapter.
- It reaches for mutable static or global state, singletons, and shared instances, creating hidden coupling between callers and tests whose outcomes depend on execution order.
- It litters otherwise clear code with narration, headings, and explanatory comments instead of trusting good names, types, abstractions, and control flow.
- It is bad at keeping code formatting consistent unless formatting is handled mechanically.
- It silently introduces new abstractions, dependencies, or file shapes unless explicitly told to stop and ask.
- It follows whatever context is most visible, which means bloated root instructions and poorly scoped guidance actively make it worse.
- It reaches for tools, MCPs, or browser automation before exhausting code-level investigation if those tools are available.
- It misses reusable workflow instructions when they are not designed to be automatically discoverable from the user's wording.
- It answers broad review questions from representative samples, then sounds more comprehensive than the evidence supports.
- It overwhelms the user by presenting every manual check, question, and instruction at once instead of guiding them through the work in manageable stages.
- It over-explains routine work and buries the outcome in implementation detail, making the user read more than is necessary to act or verify the result.
- It checks whether values are reused without checking whether names carry stable semantic meaning.
- It treats UI code like prototype presentation work instead of durable product architecture with contracts, naming semantics, and long-term maintenance cost.
- It treats visible styling as "consistent enough" while missing drift across containers, typography, spacing, borders, corners, shadows, icons, and feedback states.
- It writes a bare numeric or colour literal whenever styling a new element, because the value is locally obvious and no rule made it look for an owner, which is how one border width ends up authored in over a hundred places.
- When a shared component resists, it overrules the component from outside — a priority flag, an injected class name, a selector reaching into the component's markup — instead of treating the resistance as a missing variant in the component's own API.
- It reproduces a shared component's appearance by hand rather than finding it, and will even write a comment saying the copy matches the original instead of reading that as proof the copy should not exist.
- It draws a semantic structure out of generic containers because styling reaches the visual result faster, producing a table, list, or control that only looks like one.
- It implements the appearance of a known interaction pattern without its keyboard map or state attributes, and calls the component finished.
- It chooses a colour, spacing value, or token because the rendered result looks right, rather than because the name matches the meaning, which quietly couples two decisions that later need to diverge.
- It judges text contrast by eye, and reaches for opacity to make text look secondary, which changes contrast against whatever happens to be behind it and is invisible to any token check.
- During a consolidation it migrates the state it can see and leaves the loading, empty, and error paths of the same component behind.
- It reports a refactor as visually neutral on the strength of having read the diff, when every defect that class of work produces is one a diff cannot show.

The following practices address these failure modes structurally.

## Relevant Claude Code Capabilities

Persistence and parallel-work features require clear repository boundaries:

- **Auto memory is enabled by default.** It stores Claude's repository learnings separately from team-authored `CLAUDE.md` instructions. Use `/memory` to inspect or disable it; keep team policy in version-controlled instructions.
- **Subagents run in the background by default, and forked subagents inherit the full conversation and prompt cache by default.** Permission prompts surface in the main session, and completed tasks remain visible in `/tasks`. Forks have task context but no shared live working memory; give parallel agents bounded outputs and explicit ownership.
- **Nested delegation and dynamic workflows have limits.** Subagents nest to three layers by default. A session defaults to 200 total subagent spawns and 20 concurrent agents; dynamic workflows advise fewer than 15 agents. Reserve these tools for genuinely decomposable work.
- **Auto mode is a permission posture, not a project setting.** Select it through user, managed, or CLI settings; checked-in project settings cannot enable it. Its prose-based safety policy complements deterministic deny rules and sandboxing but does not replace them.
- **Session controls support recovery and movement.** `/cd` changes the working directory, `/rewind` can return to before `/clear`, and `/doctor` (also `/checkup`) diagnoses configuration and can propose repairs. These controls do not justify endless, unfocused sessions.
- **Conversation controls serve distinct purposes.** `/fork` copies the conversation into a background session, `/subtask` creates an in-session fork that reports back, and `/branch` switches the current session to a copied conversation branch.
- **Forked skills and built-in reviews require explicit routing.** Skills with `context: fork` run in the background unless they set `background: false`. `/code-review`, `/verify`, and `/deep-research` run only when invoked.
- **Sessions can coordinate directly on macOS and Linux.** One session can discover and message another, including by `@`-mention. Treat this as a coordination channel, not shared reasoning.
- **Browser and computer-use tools support runtime verification.** Desktop includes an in-app browser, Claude in Chrome is available on direct plans, and computer use is a research-preview capability. Use these tools after source and test inspection, not instead of them.
- **Model and effort controls are configurable.** The `opus` alias can resolve differently across providers. Settings support ordered `fallbackModel` chains, while `/effort` controls reasoning effort. Use family aliases for upgrades and full model IDs for reproducibility.
- **Sandbox network policy can fail closed.** `sandbox.network.strictAllowlist` denies non-allowlisted hosts for sandboxed commands instead of falling back to a prompt. Use it when network egress must be deterministic, alongside filesystem isolation and regular permission rules.

See the official [Memory](https://code.claude.com/docs/en/memory), [Subagents](https://code.claude.com/docs/en/sub-agents), and [Settings](https://code.claude.com/docs/en/settings) documentation.

## The Problem Statement

### Claude's default behavior is locally convenient and globally damaging

Claude Code does not naturally optimize for long-term codebase coherence. It optimizes for completing the current task with the least resistance. That produces three predictable pathologies:

1. It copy-pastes logic instead of abstracting shared behavior.
2. It duplicates an existing pattern because it did not search broadly enough before writing.
3. It introduces a slightly different way of doing something because the local context made it look reasonable.

The official [Best Practices](https://code.claude.com/docs/en/best-practices) emphasizes specificity, context, and verification. The larger risk is ongoing structural degradation, not isolated mistakes.

Pattern drift compounds. One error-handling style becomes three. One feature uses a shared helper, another writes the logic inline, and a third invents a wrapper. The result is inconsistent behavior, partial abstractions, and bugs confined to duplicated branches.

Pattern drift often begins when Claude reads only the nearest files before coding. Require a wider search: upstream for callers and shared abstractions, downstream for implementations and consumers, and laterally for similar files, classes, functions, and tests. This prevents reinvention of existing patterns.

Operationally, you should treat Claude like a lazy, inexperienced, tasteless developer with a strong bias against refactoring and TDD. That sounds harsh, but it produces the correct setup instincts. You do not give that developer vague guidance and broad freedom. You give them explicit constraints, clear examples, narrow workflow rules, and approval gates around anything that expands the codebase's conceptual surface area.

### Claude has a minimum-change bias

When adding a feature, Claude often assumes the existing structure is valid and makes the smallest change that fits. This creates long-term structural damage.

Claude often inserts behavior at the nearest plausible point and patches until tests pass. Preparing the codebase first avoids conditionals in the wrong layer, duplicated branches, awkward parameter growth, and embedded special cases.

It also shows up as false reverence for the existing code. Claude often behaves as if the current implementation must be production-hardened, backward compatible, and preserved at all costs even when the project is brand new, has never shipped, or is obviously still in flux. That leads it to add fallbacks, compatibility layers, default values, and defensive branches that the codebase has not earned. In many early or actively evolving projects, the correct move is to change the shape cleanly rather than preserve a nonexistent legacy contract.

Encode a different default:

1. Audit the relevant code.
2. Assume it is not ready.
3. Refactor the area into a coherent shape.
4. Only then add the feature.

If that sequence is not enforced, Claude will happily bolt new behavior onto weak foundations forever.

Require readiness refactoring, not speculative redesign or framework invention. Prepare the affected area to host the change, then implement it.

Claude should propose a cleaner, larger change when justified. It should identify when code needs reshaping, simplification, or replacement instead of preserving accidental behavior with compatibility hacks.

### Context rots, and naive configuration makes it worse

Anthropic's [Memory](https://code.claude.com/docs/en/memory) and [Best Practices](https://code.claude.com/docs/en/best-practices) guidance recommends concise, specific project memory. Do not put every instruction in `CLAUDE.md`: it is always-on context, so unnecessary content consumes task budget.

Institutional memory matters, but root `CLAUDE.md` is not the place to store all of it. Long-lived projects need discoverable, on-demand context instead.

Root `CLAUDE.md` is for non-obvious commands, repository-wide verification expectations, architectural decisions, conventions, repository etiquette, approval boundaries, and dangerous or generated areas. Include an item only when omitting it would predictably reduce reliability. Do not use the file to index `.claude/`; scoped configuration must be discoverable through metadata, path scope, placement, and reference ownership.

## Guide Map

Detailed guidance lives in focused chapters that can be read, maintained, and reused independently.

- [Context and Routing](guide/context-and-routing.md) — `CLAUDE.md`, rules, skills, memory, and discoverability.
- [Engineering Conventions](guide/engineering-conventions.md) — type safety, abstractions, encapsulation, replaceable external-service adapters, explicit construction and dependency boundaries, duplication, UI architecture, logging, APIs, exceptions, and formatting.
- [Design-System Refactors](guide/design-system-refactors.md) — evidence-derived guidance for inventorying, modelling, sequencing, and verifying cross-surface UI-system migrations.
- [Database Correctness and Scale](guide/database.md) — normalization, transactions, constraints, indexes, and safe denormalization decisions.
- [Testing and Quality](guide/testing-and-quality.md) — readable driver-backed tests, controlled bug investigation, deliberate regression-test retention, TDD, convention design, stop-and-ask rules, and drift audits.
- [Code Intelligence](guide/code-intelligence.md) — repository search, GitNexus, ast-grep, dependency-cruiser, and Knip.
- [Workflows and Configuration Maintenance](guide/workflows-and-maintenance.md) — audit → refactor → implement → verify, progressive validation, and keeping Claude configuration current.
- [Integrations, Hooks, and Permissions](guide/integrations-and-permissions.md) — MCP strategy, hooks, settings, sandboxing, and permission posture.
- [Parallel Work](guide/parallel-work.md) — subagents, worktrees, ownership, and merge avoidance.

## Recommended Baseline

If you want a practical default setup, use this:

1. A short root `CLAUDE.md` containing only crucial repository-wide facts and instructions; scoped Claude configuration must be independently discoverable rather than indexed from this file.
2. A small rules set for always-on global and path-scoped standing instructions.
3. A small skill set:
   - `conventions-global`
   - `feature-workflow`
   - `testing-conventions`
   - `bug-investigation`
   - one skill per subsystem with genuinely distinct conventions
   - `config-maintenance`
4. Reference documents for detailed conventions, kept outside root `CLAUDE.md` instructions and owned by the rule or skill that uses them.
5. Hooks for audit logs, lightweight reminders, notifications, and targeted side effects.
6. `settings.json` for allow/deny behavior and permission posture.
7. Compiler, language-server, test, and repository-search commands as the first code-investigation layer.
8. A code graph such as GitNexus only when repository scale and relationship questions justify it.
9. Structural and static checks such as ast-grep, dependency-cruiser, or Knip where they fit the language and recurring failure modes.
10. A code-first MCP policy.
11. A hard stop-and-ask rule for any new dependency, pattern, abstraction, or convention gap.
12. An isolated worktree or clone for each parallel write task, with explicit ownership.

A strong setup keeps Claude useful while making drift, duplication, and weak local choices difficult to introduce.

## Official Sources

Use official documentation to verify Claude Code capabilities. Use each code tool's primary documentation to verify behavior, language coverage, configuration, and license. Check generated indexes and static-analysis findings against source, compiler, runtime, and test evidence.

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code Commands](https://code.claude.com/docs/en/commands)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Code Settings](https://code.claude.com/docs/en/settings)
- [Claude Code Model Configuration](https://code.claude.com/docs/en/model-config)
- [Claude Code Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Parallel Agents](https://code.claude.com/docs/en/agents)
- [Claude Code Worktrees](https://code.claude.com/docs/en/worktrees)

### Further Reading

- [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) — an empirical study of 253 public `Claude.md` files, useful for distinguishing common content patterns from isolated template advice.
- [Agent READMEs](https://arxiv.org/abs/2511.12884) — a broader empirical study of repository-level agent context files and the instructions developers prioritize in practice.
- [GitNexus](https://github.com/nxpatterns/gitnexus) — a repository-intelligence and code-graph tool for exploring dependencies, execution flows, symbols, and the likely blast radius of a change.
- [ast-grep](https://github.com/ast-grep/ast-grep) — a structural search, linting, and codemod tool that matches syntax trees rather than relying on fragile text patterns.
- [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) — a dependency-analysis tool for JavaScript and TypeScript that can visualize module relationships and enforce architectural boundaries in CI.
- [Knip](https://github.com/webpro-nl/knip) — a JavaScript and TypeScript project-analysis tool for finding unused files, exports, dependencies, and configuration entries.
- [Growing Object-Oriented Software, Guided by Tests](https://growing-object-oriented-software.com/) — Steve Freeman and Nat Pryce's practical account of using tests and object collaboration to grow coherent, maintainable software; particularly valuable for understanding how testability exposes and improves design.
- **Composition Root** — Mark Seemann's pattern for keeping object-graph construction near an application's entry point and injecting dependencies into application code.
- [Anthropic Agent Skills](https://github.com/anthropics/skills) — Anthropic's official reference implementations for portable, automatically discovered skills; its `frontend-design` skill is a particularly strong example of grounding visual direction in the subject, audience, content, and deliberate critique rather than generic AI defaults.
- [Impeccable](https://github.com/pbakaus/impeccable) — a design language and skill set for planning, building, critiquing, auditing, and polishing production interfaces, with deterministic checks for common AI-generated UI defects.
- [Taste Skill](https://github.com/Leonxlnx/taste-skill) — an opinionated, framework-neutral collection for art direction, redesign audits, image-to-code work, and avoiding repetitive AI aesthetics; useful for marketing sites and portfolios, but its default v2 skill is experimental and its strong stylistic rules should be selected to fit the brief rather than adopted wholesale.
- [UI UX Pro Max](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) — a searchable UI/UX knowledge base and design-system generator covering accessibility, palettes, typography, product categories, and stack-specific guidance including SwiftUI; use its generated recommendations as candidates to validate against the product's real design system and primary platform guidance.
- [Vercel Agent Skills](https://github.com/vercel-labs/agent-skills) — Vercel's official agent skills, including detailed web-interface guidance and React/Next.js performance rules suitable for adapting into project-scoped UI conventions.
- [Figma MCP Server Guide](https://github.com/figma/mcp-server-guide) — Figma's official MCP configuration, skills, and design-to-code rules for retrieving structured design context, variables, components, assets, and screenshots.
- [Playwright MCP](https://github.com/microsoft/playwright-mcp) — Microsoft's browser-automation integration for accessibility-tree-driven UI inspection and testing; its CLI and accompanying skills are often the more context-efficient choice for coding agents.
- [SwiftUI Agent Skill](https://github.com/twostraws/SwiftUI-Agent-Skill) — Paul Hudson's focused review skill for modern SwiftUI APIs, data flow, navigation, performance, accessibility, and Apple Human Interface Guidelines.
- [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) — a CLI, MCP server, and agent skills for building, testing, launching, debugging, and inspecting iOS and macOS projects through Xcode.
- [Mobile MCP](https://github.com/mobile-next/mobile-mcp) — native iOS and Android simulator, emulator, and device automation using structured accessibility information and screenshots.
- [Peekaboo](https://github.com/steipete/Peekaboo) — a macOS CLI and MCP server for high-fidelity screenshots and accessibility-driven automation of applications, menus, windows, and controls.
