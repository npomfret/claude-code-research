# Claude Code Guide for Long-Running Projects

This guide helps keep maintained codebases coherent as Claude Code works on them. Explicit project decisions, discoverable guidance, and repeatable checks remain useful as models improve; the setup should address observed risks and make engineering constraints explicit.

Use this page as the overview and the focused chapters as practical checklists. Blocks labelled **Example** illustrate the guidance; adapt their names, paths, commands, and policy choices to the repository rather than treating them as required configuration. Anthropic's official documentation is the source of truth for product capabilities.

## Operating Ethos

Give Claude freedom to investigate and improve the code within clear project boundaries. Be brutally strict about implementation shortcuts: fewer mistakes do not justify relaxing the conventions that prevent hacks. “Be careful” is insufficient; name forbidden patterns, define legitimate exceptions, and reject violations even when the happy path works. Keep instructions short, load detail when relevant, and enforce deterministic conventions with tooling.

In particular, forbid catch-and-carry-on, sloppy logging, speculative fallbacks, blind retries, and defaults that conceal missing required values. Fix the violated contract or owning abstraction. Recovery is acceptable only when it follows an explicit policy and preserves valid state; see [Failure handling and forbidden shortcuts](guide/engineering-conventions.md#failure-handling-and-forbidden-shortcuts).

**Do:**

- Define the intended outcome and the checks that prove it.
- Search callers, implementations, and similar code before editing.
- Reuse canonical patterns; refactor weak structure when the current task needs it.
- Consider solutions creatively, present the best first regardless of cost or complexity, and explain the trade-offs of cheaper alternatives.
- Keep state, invariants, and external-service contracts behind clear ownership boundaries.
- Make skills and scoped rules discoverable from ordinary task wording and touched paths.
- Verify claims with source, tests, and runtime evidence appropriate to the change.
- Report the outcome, verification, and material decisions concisely.
- After any significant change, before commits, spawn an agent to do a non-pedantic code review - tell it to skip the minor and trivial stuff and focus on real problems.
- For research and reading, spawn an agent on a fast cheap model and have it report back.

**Don't:**

- Patch around weak structure or preserve accidental behaviour without a real compatibility requirement.
- Add speculative abstractions, duplicate implementations, or new conventions without approval.
- Turn root instructions into a knowledge base or a configuration directory.
- Use tools merely because they are available, or substitute a sample for a comprehensive audit.
- Change code during a read-only audit or retain unsupported debugging experiments.
- Claim behavioural or visual correctness from a clean build alone.

## Relevant Claude Code Capabilities

Persistence and parallel-work features require clear repository boundaries:

- **Auto memory is enabled by default.** It stores Claude's repository learnings separately from team-authored `AGENTS.md` instructions. Use `/memory` to inspect or disable it; keep team policy in version-controlled instructions.
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

## Guide Map

Detailed guidance lives in focused chapters that can be read, maintained, and reused independently.

- [Context and Routing](guide/context-and-routing.md) — `AGENTS.md (was CLAUDE.md)`, rules, skills, memory, and discoverability.
- [Engineering Conventions](guide/engineering-conventions.md) — type safety, abstractions, encapsulation, replaceable external-service adapters, explicit construction and dependency boundaries, duplication, UI architecture, logging, APIs, exceptions, and formatting.
- [Design-System Refactors](guide/design-system-refactors.md) — evidence-derived guidance for inventorying, modelling, sequencing, and verifying cross-surface UI-system migrations.
- [UI and UX Audits](guide/ui-ux-audits.md) — read-only, evidence-backed audits of interface defects, accessibility, runtime behaviour, visual drift, state coverage, and design-system ownership.
- [Database Correctness and Scale](guide/database.md) — normalization, transactions, constraints, indexes, and safe denormalization decisions.
- [Testing and Quality](guide/testing-and-quality.md) — readable driver-backed tests, controlled bug investigation, deliberate regression-test retention, TDD, convention design, stop-and-ask rules, and drift audits.
- [Code Intelligence](guide/code-intelligence.md) — repository search, JetBrains MCP, GitNexus, ast-grep, dependency-cruiser, and Knip.
- [Workflows and Configuration Maintenance](guide/workflows-and-maintenance.md) — audit → refactor → implement → verify, Git policy, progressive validation, and keeping Claude configuration current.
- [Integrations, Hooks, and Permissions](guide/integrations-and-permissions.md) — MCP strategy, hooks, settings, sandboxing, and permission posture.
- [Parallel Work](guide/parallel-work.md) — subagents, worktrees, ownership, and merge avoidance.

## Recommended Baseline

If you want a practical default setup, use this:

1. A short root `AGENTS.md` containing only crucial repository-wide facts and instructions; scoped Claude configuration must be independently discoverable rather than indexed from this file.
2. A small rules set for always-on global and path-scoped standing instructions.
3. A small skill set covering the relevant workflows. Example names, not required identifiers:
   - `conventions-global`
   - `feature-workflow`
   - `testing-conventions`
   - `bug-investigation`
   - `ui-ux-audit` where the product has a user interface
   - one skill per subsystem with genuinely distinct conventions
   - `config-maintenance`
4. Reference documents for detailed conventions, kept outside root `AGENTS.md` instructions and owned by the rule or skill that uses them.
5. Advisory hooks for reminders, diagnostics, audit logs, notifications, and targeted side effects; blocking hooks only for narrow, tested requirements.
6. `settings.json` for allow/deny behavior and permission posture.
7. Compiler, language-server, test, and repository-search commands as the first code-investigation layer, with JetBrains MCP preferred for supported IDE semantic operations.
8. A code graph such as GitNexus only when repository scale and relationship questions justify it and the IDE integration does not already provide the needed evidence.
9. Structural and static checks such as ast-grep, dependency-cruiser, or Knip where they fit the language and recurring failure modes.
10. A code-first MCP policy.
11. A hard stop-and-ask rule for any new dependency, pattern, abstraction, or convention gap.
12. An isolated worktree or clone for each parallel write task, with explicit ownership.

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

- [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) — an empirical study of 253 public `AGENTS.md` files, useful for distinguishing common content patterns from isolated template advice.
- [Agent READMEs](https://arxiv.org/abs/2511.12884) — a broader empirical study of repository-level agent context files and the instructions developers prioritize in practice.
- [JetBrains MCP Server](https://www.jetbrains.com/help/idea/mcp-server.html) — IDE-backed symbol information, call analysis, inspections, refactoring, and build/run tools for coding agents.
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
