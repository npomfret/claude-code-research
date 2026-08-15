# Claude Code Guide for Non-Trivial & Long-Running Projects

This guide is for real codebases that will still matter in six months. It is not for demos, weekend prototypes, or "look what the agent can do" threads. The central problem is simple: Claude Code is useful, fast, and broad, but without structure it is a sloppy programmer. It duplicates logic, patches features into code that should have been refactored first, and slowly fills a codebase with pattern drift.

Anthropic's official documentation and changelog, collected at the end of this guide, are the source of truth for what Claude Code can do.

> **Verification date: 14 August 2026.** The latest weekly digest available during this review covers v2.1.220–v2.1.224 (3–7 August), and the changelog reaches v2.1.232. Claude Code changes quickly; use the linked weekly digest and changelog to check later releases and fixes.

## Operating Ethos

The trick to getting Claude Code working well is not to make it obedient everywhere. It is to give it room to move in the places where exploration, refactoring, investigation, and tool use are valuable, while sharply constraining the places where drift is expensive. Claude needs enough freedom to inspect the codebase, reshape weak areas, run checks, and follow the evidence. It should not have freedom to invent a new architecture, coding style, dependency, testing pattern, or permission posture just because that is the shortest path through the current task.

That balance is difficult because the boundaries have to be designed, not wished into existence. A good setup makes the right paths easy to find and the wrong paths hard to take. Skills, rules, agents, references, and commands only help when Claude can discover them from normal task wording and route itself to them without the human remembering a magic invocation. If an important convention or workflow lives in an orphaned markdown file, it may as well not exist.

The other half of the system is constraint. Claude will often take the locally convenient route: patch around a weak structure, duplicate a nearby pattern, preserve accidental behavior, weaken types, skip tests, or use a hack that gets the immediate output looking right. This is not malice; it is the default shape of task completion under pressure. The answer is not softer advice. The answer is harsh, explicit, highly visible coding conventions backed by verification, approval gates, and mechanical formatting. The setup should make the correct engineering path more obvious than the shortcut.

In practice, this guide argues for three simultaneous properties:

- **bounded freedom**: let Claude investigate, refactor for readiness, and use tools inside approved areas;
- **automatic discoverability**: make every important workflow, skill, rule, and specialist agent routable without special prompting;
- **tight conventions**: define the codebase's allowed shapes so Claude cannot quietly create new ones.

If one of those is missing, the system degrades. Freedom without conventions becomes chaos. Conventions without discoverability are ignored. Discoverability without freedom creates a well-documented bottleneck. The point of the architecture below is to keep all three in force at the same time.

## Common Claude Failure Modes

If the setup does not actively counter these, Claude will keep doing them:

- It copy-pastes locally convenient logic instead of finding or extracting the shared abstraction.
- It makes the smallest possible code change even when the surrounding structure is unready for the new requirement.
- It invents slight pattern variants because the first few files it read looked "close enough."
- It does not reliably look around for existing patterns before starting work, so it reinvents the wheel unless explicitly told to search upstream, downstream, and laterally.
- It sometimes over-engineers in the opposite direction by introducing speculative abstractions that the current codebase does not actually need.
- It avoids refactoring and test-first discipline unless forced to do them.
- It is bad at keeping code formatting consistent unless formatting is handled mechanically.
- It silently introduces new abstractions, dependencies, or file shapes unless explicitly told to stop and ask.
- It follows whatever context is most visible, which means bloated root memory and poorly routed guidance actively make it worse.
- It reaches for tools, MCPs, or browser automation before exhausting code-level investigation if those tools are available.
- It misses reusable workflow instructions when they are not designed to be automatically discoverable from the user's wording.
- It answers broad review questions from representative samples, then sounds more comprehensive than the evidence supports.
- It overwhelms the user by presenting every manual check, question, and instruction at once instead of guiding them through the work in manageable stages.
- It over-explains routine work and buries the outcome in implementation detail, making the user read more than is necessary to act or verify the result.
- It checks whether values are reused without checking whether names carry stable semantic meaning.
- It treats UI code like prototype presentation work instead of durable product architecture with contracts, naming semantics, and long-term maintenance cost.
- It treats visible styling as "consistent enough" while missing drift across containers, typography, spacing, borders, corners, shadows, icons, and feedback states.

The rest of this guide exists to suppress those failure modes structurally rather than hoping Claude behaves better on its own.

## Current Product Surface That Changes the Setup

The recent releases make Claude Code much more capable at persistence and parallel work. They do not remove the need for a controlled repository setup; they make the boundaries more important.

- **Auto memory is now enabled by default.** It stores Claude’s own repository learnings separately from team-authored `CLAUDE.md` instructions. Use `/memory` to inspect or disable it; keep team policy in version-controlled instructions, not in auto memory.
- **Subagents run in the background by default, and forked subagents now inherit the full conversation and prompt cache by default.** Permission prompts from a background agent surface in the main session, and a completed task remains visible in `/tasks`. A fork gives the agent much better task context, but not a shared live working memory; give parallel agents independent, bounded outputs and explicit ownership.
- **Nested delegation and dynamic workflows exist, with limits.** Subagents currently nest to three layers by default. A session defaults to 200 total subagent spawns and 20 running concurrently, while dynamic workflows now carry an advisory default of fewer than 15 agents. Treat all of these as escalation tools for genuinely decomposable work, not as a default for small edits.
- **Auto mode is a real permission posture, not a project setting.** It uses a safety classifier and must be selected from user, managed, or CLI settings; checked-in project settings cannot enable it. It becomes the default mode for new Pro, Max, and Team sessions from 14 August 2026. Its policy is prose-based, so it complements rather than replaces deterministic deny rules and sandboxing.
- **Session recovery is better.** `/cd` can move a session’s working directory, `/rewind` can return to before `/clear`, and `/doctor` (also `/checkup`) diagnoses configuration and can propose repairs. These reduce operational friction; they do not make an endless, unfocused session a good idea.
- **Fork terminology changed.** `/fork` now copies the conversation into a separate background session. Use `/subtask` for the in-session fork that reports back into the current conversation, and `/branch` when you want to switch the current session onto a copied conversation branch.
- **Forked skills and built-in reviews are more explicit.** Skills with `context: fork` run in the background by default unless they set `background: false`. `/code-review`, `/verify`, and `/deep-research` run only when explicitly invoked; do not write routing guidance that assumes Claude will start them automatically.
- **Sessions can now coordinate directly.** On macOS and Linux, one session can discover another and send it a message (including by `@`-mentioning its name). This is useful for passing decisions or findings between separately run sessions, but it is a coordination channel, not shared reasoning; retain explicit task boundaries and ownership.
- **Browser and computer-use surfaces are broader.** Desktop now has an in-app browser, Claude in Chrome is generally available on direct plans, and computer use remains a research-preview capability. These are useful for verifying UI or consulting external truth; they should still follow source and test inspection rather than replace it.
- **Model and effort controls changed again.** Opus 5 is now the current `opus` alias on the Anthropic API and several supported providers; provider aliases can still resolve differently. Current settings support ordered `fallbackModel` chains, while `/effort` controls reasoning effort. Prefer family aliases when you want upgrades and full model IDs when reproducibility matters.
- **Sandbox network policy can fail closed.** `sandbox.network.strictAllowlist` denies non-allowlisted hosts for sandboxed commands instead of falling back to a prompt. Use it when network egress must be deterministic, alongside filesystem isolation and regular permission rules.

The official [What’s new](https://code.claude.com/docs/en/whats-new), [Memory](https://code.claude.com/docs/en/memory), [Subagents](https://code.claude.com/docs/en/sub-agents), and [Settings](https://code.claude.com/docs/en/settings) pages are the ongoing references for this section.

## 1. The Problem Statement

### Claude's default behavior is locally convenient and globally damaging

Claude Code does not naturally optimize for long-term codebase coherence. It optimizes for completing the current task with the least resistance. That produces three predictable pathologies:

1. It copy-pastes logic instead of abstracting shared behavior.
2. It duplicates an existing pattern because it did not search broadly enough before writing.
3. It introduces a slightly different way of doing something because the local context made it look reasonable.

The official [Best Practices](https://code.claude.com/docs/en/best-practices) doc is right to stress specificity, context, and verification. But most public advice still understates the main issue: the failure mode is not just "Claude occasionally makes mistakes." The failure mode is ongoing structural degradation.

Pattern drift compounds. One new error-handling style becomes three. A second API-call shape appears because the agent patched one endpoint quickly. One feature uses a shared helper, the next writes the logic inline, and a third invents a wrapper. Six weeks later the project has inconsistent behavior, partial abstractions, and bugs that only exist in one branch of duplicated code.

One practical cause of pattern drift is that Claude often does not "look around" before writing. It reads the nearest file or two, sees something that looks approximately right, and starts coding. That is not enough. A serious setup has to force a wider search before edits begin: look upstream at callers and shared abstractions, look downstream at implementations and consumers, and look laterally for similarly named files, classes, functions, and tests elsewhere in the repo. If Claude does not search broadly first, it will keep reinventing patterns that already exist.

Operationally, you should treat Claude like a lazy, inexperienced, tasteless developer with a strong bias against refactoring and TDD. That sounds harsh, but it produces the correct setup instincts. You do not give that developer vague guidance and broad freedom. You give them explicit constraints, clear examples, narrow workflow rules, and approval gates around anything that expands the codebase's conceptual surface area.

### Claude has a minimum-change bias

This is the most damaging bias for mature projects. When asked to add a feature, Claude usually assumes the current structure is valid and makes the minimum possible change to fit the feature in. That is fast in the moment and disastrous over time.

This often shows up as shoehorning. Claude sees a request, finds the nearest place the new behavior could be squeezed in, and patches until the tests pass or the output looks plausible. That is not the same as preparing the codebase for the requirement. The result is usually a hack: more conditionals in the wrong layer, duplicated branching, awkward parameter growth, and one more special case embedded in a structure that was already bending.

It also shows up as false reverence for the existing code. Claude often behaves as if the current implementation must be production-hardened, backward compatible, and preserved at all costs even when the project is brand new, has never shipped, or is obviously still in flux. That leads it to add fallbacks, compatibility layers, default values, and defensive branches that the codebase has not earned. In many early or actively evolving projects, the correct move is to change the shape cleanly rather than preserve a nonexistent legacy contract.

A serious setup must encode a different default:

1. Audit the relevant code.
2. Assume it is not ready.
3. Refactor the area into a coherent shape.
4. Only then add the feature.

If that sequence is not enforced, Claude will happily bolt new behavior onto weak foundations forever.

The required posture is aggressive readiness refactoring. Not speculative redesign. Not framework invention. Refactor the current area until it is actually a clean host for the change, then implement the change. If the setup does not push Claude toward that sequence, it will keep choosing the smaller patch over the better design.

Just as important, Claude should be encouraged to offer the cleaner, larger change when that is the right engineering answer. The setup should not train it to worship the smallest patch. It should train it to identify when the existing code should simply be reshaped, simplified, or replaced instead of being padded with hacks to preserve accidental behavior.

### Context rots, and naive configuration makes it worse

Anthropic's [Memory](https://code.claude.com/docs/en/memory) guidance and [Best Practices](https://code.claude.com/docs/en/best-practices) guidance are correct that project memory should be concise and specific. The most damaging bad advice in the ecosystem is the repeated suggestion to "just add it to `CLAUDE.md`." That advice works on tiny projects because the entire repo is tiny. On a real project, `CLAUDE.md` is always-on context. Every unnecessary paragraph steals budget from the actual task.

Institutional memory matters, but the root memory file is not the place to store all of it. Long-lived projects need discoverable, on-demand context instead.

## Guide Map

This overview keeps the operating model and current-product summary in one place. The detailed guidance now lives in focused chapters so it can be read, maintained, and reused independently.

- [Context and Routing](guide/context-and-routing.md) — `CLAUDE.md`, rules, skills, memory, and discoverability.
- [Engineering Conventions](guide/engineering-conventions.md) — type safety, abstractions, duplication, UI architecture, logging, APIs, exceptions, and formatting.
- [Database Correctness and Scale](guide/database.md) — normalization, transactions, constraints, indexes, and safe denormalization decisions.
- [Testing and Quality](guide/testing-and-quality.md) — TDD, convention design, stop-and-ask rules, and drift audits.
- [Code Intelligence](guide/code-intelligence.md) — repository search, GitNexus, ast-grep, dependency-cruiser, and Knip.
- [Workflows and Configuration Maintenance](guide/workflows-and-maintenance.md) — audit → refactor → implement → verify, progressive validation, and keeping Claude configuration current.
- [Integrations, Hooks, and Permissions](guide/integrations-and-permissions.md) — MCP strategy, hooks, settings, sandboxing, and permission posture.
- [Parallel Work](guide/parallel-work.md) — subagents, worktrees, ownership, and merge avoidance.

## Recommended Baseline

If you want a practical default setup, use this:

1. A short root `CLAUDE.md` that encodes non-negotiables and routes to deeper guidance.
2. A small rules set for always-on global and path-scoped standing instructions.
3. A small skill set:
   - `conventions-global`
   - `feature-workflow`
   - one skill per subsystem with genuinely distinct conventions
   - `config-maintenance`
4. Reference documents for detailed conventions, kept outside root memory.
5. Hooks for audit logs, lightweight reminders, notifications, and targeted side effects.
6. `settings.json` for allow/deny behavior and permission posture.
7. Compiler, language-server, test, and repository-search commands as the first code-investigation layer.
8. A code graph such as GitNexus only when repository scale and relationship questions justify it.
9. Structural and static checks such as ast-grep, dependency-cruiser, or Knip where they fit the language and recurring failure modes.
10. A code-first MCP policy.
11. A hard stop-and-ask rule for any new dependency, pattern, abstraction, or convention gap.
12. An isolated worktree or clone for each parallel write task, with explicit ownership.

That is the world-class setup for long-running projects: not the most feature-rich setup, not the cleverest setup, and not the most impressive screenshot. The best setup is the one that keeps Claude useful while making drift, duplication, and sloppy local choices hard to introduce.

## Official Sources and Further Reading

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
- [Claude Code What’s New](https://code.claude.com/docs/en/whats-new)
- [Claude Code CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [GitNexus repository](https://github.com/nxpatterns/gitnexus)
- [GitNexus license](https://github.com/nxpatterns/gitnexus/blob/main/LICENSE)
- [ast-grep documentation](https://ast-grep.github.io/)
- [dependency-cruiser repository](https://github.com/sverweij/dependency-cruiser)
- [Knip documentation](https://knip.dev/)

Use the official docs and release notes to verify what Claude Code supports. Use each code tool's primary documentation to verify its current behavior, language coverage, configuration, and license. Treat generated indexes and static-analysis findings as evidence to check against the source, compiler, runtime, and tests.
