# Workflows and Configuration Maintenance

Use a repeatable workflow for non-trivial changes, with preparation and verification proportionate to the task. The aim is coherent structure and proven behaviour, without speculative redesign. See Anthropic's [Best Practices](https://code.claude.com/docs/en/best-practices).

## Default workflow

1. **Audit:** load applicable conventions and inspect code, tests, configuration, and external boundaries.
2. **Refactor for readiness:** correct the structure the current requirement depends on.
3. **Implement:** add behaviour to the prepared structure.
4. **Verify:** run appropriate checks and assess the stated success criteria.
5. **Update configuration:** record approved convention changes in the same change.

**Do:**

- Ask: “If starting this area from scratch, what would the best approach be?” Present that solution first; label lower-effort alternatives with their debt and constraints.
- Search upstream for callers and construction, downstream for implementations and side effects, and laterally for similar code and tests.
- Find canonical names, error shapes, and helpers before creating variants.
- Trace provider clients and types into the application; check adapter ownership and replacement impact.
- Check touched code for global configuration, scattered environment reads, and `.env` dependencies; move configuration loading to the boundary and inject validated settings.
- Treat readiness as unproven until the audit establishes it; refactor where needed, including encapsulation and hidden dependencies.
- Define the intended outcome, preparation, and completion checks in the task or workflow skill.
- Review failure paths against the [forbidden-shortcut rules](engineering-conventions.md#failure-handling-and-forbidden-shortcuts), even when the happy path and tests pass.

**Don't:**

- Optimise solely for the smallest diff or assume existing structure must be preserved.
- Use another helper to avoid correcting the owning abstraction.
- Build frameworks or general-purpose layers for hypothetical future use cases.
- Preserve nonexistent legacy contracts with defensive compatibility branches.

External-service isolation is justified on first use by its independent contract, failures, and test-substitution needs. Other abstractions need a current requirement or concrete reuse pressure.

**Example task prompt:** adapt the feature and checks to the task.

> Add retry behaviour. First inspect retry, error-handling, and orchestration patterns. Prepare the affected structure where needed, identify any approval decisions, and define checks before implementing.

Use [Design-System Refactors](design-system-refactors.md) for migrations requiring component convergence before broad token substitution.

## Manual verification

Keep the scope visible and the user's next action manageable.

**Do:** give a short inventory of the checks, guide one check at a time, use feedback before continuing, and keep a brief progress record.

**Don't:** present every detailed procedure at once unless the user requests a complete checklist or needs it for independent execution or delegation.

### Git workflow

Put the Git policy in root `AGENTS.md` or an always-loaded rule.

**Do:**

- **Work on `main`** unless explicitly told to use a branch, or a temporary branch is needed for worktree isolation.
- **Commit little and often:** small, coherent changes; stage only task-owned files or hunks.
- **Run fast targeted tests before committing.** Rely on the continuous integration server for full builds; distinguish local checks from pending or completed CI.
- **Regularly pull with `git pull --rebase`:** before starting, at sensible checkpoints, and before handoff. Inspect and preserve uncommitted work first.
- **Use worktrees when conflicts are expected or likely,** with explicit writer ownership. Rebase isolation branches onto current `main` and integrate without merge commits.
- Inspect conflict resolutions and rerun affected checks; address CI failures before claiming full verification.

**Don't:** merge, create merge commits, discard unrelated work to synchronize, or rewrite shared published history or force-push without explicit authorisation. Rebase local unpublished commits.

## Configuration maintenance

Version-control team-owned instructions alongside the code. Keep the configuration accurate as canonical patterns evolve; humans own policy decisions.

| Update directly within established policy | Ask before changing policy |
| --- | --- |
| Stale commands, paths, examples, and wording | Top-level conventions or their meaning |
| Records of human-approved decisions | Major skill families or agent responsibilities |
| Routing metadata and owned references | Hook workflow semantics or MCP servers |
| Descriptions that clarify existing scope | Dependencies or architecture boundaries |

**Do:**

- Update the smallest instruction surface that owns the guidance.
- Audit after canonical-pattern refactors, repeated questions, missed routing, or divergence between docs and code.
- Resolve contradictions promptly.
- Keep always-loaded instructions short and local detail with its rule or skill.

**Don't:** turn every task into documentation work, treat team configuration as personal clutter, or change policy under the guise of maintenance.

### Config-maintenance skill

**Example skill:** adapt its scope and approval handling to the repository’s policy.

```md
---
description: Use when approved conventions change or Claude configuration is stale, contradictory, or fails to route relevant guidance.
user-invocable: true
---

# Config Maintenance

1. Find the owning instruction surface and any affected routing metadata.
2. Distinguish documentation maintenance from policy change; ask before policy changes.
3. Update the smallest correct file and confirm it matches the codebase.
4. Keep scoped detail out of always-loaded instructions.
```
