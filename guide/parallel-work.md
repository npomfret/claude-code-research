# Parallel Work

Parallelism pays off when tasks have independent questions and clear ownership. Extra agents do not resolve an ambiguous task or create shared live reasoning.

## Do

- Give each agent a bounded question, owned files or subsystem, and expected artifact or conclusion.
- Use parallel investigation, isolated subsystems, read-only review alongside implementation, or clearly divided refactors.
- Give concurrent writers separate checkouts and non-overlapping ownership.
- Follow the [Git workflow](workflows-and-maintenance.md#git-workflow): default to `main`, use temporary branches for worktree isolation, regularly rebase, and integrate without merge commits.

## Don't

- Put several writers in the same feature area.
- Parallelise small tasks whose synchronization costs exceed the work.
- Assume forks share live working memory or use nested delegation for tightly coupled reasoning.

## Isolation and product behaviour

Subagents run in the background by default, with permission prompts in the main session and results in `/tasks`.

| Surface | Isolation |
| --- | --- |
| Desktop | Each new session gets a worktree. |
| Agent view | A dispatched background session moves to a worktree when it needs to edit. |
| Ordinary subagent | Current checkout unless `isolation: worktree` or the task requests isolation. |
| Separate terminal session | Use `claude --worktree`. |

Worktrees default to the remote default branch. Set `worktree.baseRef` to `"head"` when isolated work needs local commits or feature-branch state. Review `.worktreeinclude` carefully: matching gitignored files, including secrets, are copied into every isolated checkout.

Sibling clones are also suitable for independently managed sessions needing complete environment separation. Choose the checkout mechanism around ownership and environment needs.
