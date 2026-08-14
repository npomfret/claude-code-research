# Parallel Work

### Subagent parallelism is cheaper than it looks

Subagents are now background by default. This is useful for parallel investigation, audit, review, and clearly separated implementation tasks because the main agent can continue while they run. A background agent’s permission prompt appears in the main session, and its result remains visible through `/tasks` after completion.

Do not infer more than the product promises: subagents return results to the parent, but they are not a shared live-reasoning team. Give each one a bounded question, a clear ownership boundary, and an expected artifact or conclusion. Reserve nested delegation and dynamic workflows for work that is genuinely decomposable; more agents do not repair an ambiguous task.

### Choose the isolation model deliberately

Worktree behavior depends on the surface. Desktop creates a worktree for each new session, and agent view moves a dispatched background session into a worktree when it needs to edit files. An ordinary subagent starts in the current checkout unless its definition sets `isolation: worktree` or the task explicitly requests worktree isolation. Use `claude --worktree` for a separately operated terminal session.

Worktrees branch from the remote default branch by default; set `worktree.baseRef` to `"head"` when isolated work must include local commits or feature-branch state. A `.worktreeinclude` file can copy required gitignored files such as a development `.env`. Review that file carefully because every matching secret is copied into each new isolated checkout.

For several independently managed interactive sessions, either worktrees or sibling clones can be appropriate. Multiple clones remain a simple option when you want complete environment separation; worktrees are lighter when the work shares a repository and you understand the Git workflow. The key decision is ownership, not the directory mechanism.

### How to avoid merge pain

Parallelism is only worth the overhead when tasks are actually separable.

Good candidates:

- isolated subsystems,
- long-running investigations,
- one implementation plus one read-only review thread,
- large refactors with clearly divided modules.

Bad candidates:

- multiple sessions editing the same feature area,
- small changes that would finish before synchronization overhead pays off,
- tasks that depend on constant shared reasoning across the same files.

Give each writer a separate checkout and a non-overlapping ownership boundary. Sync long-running branches with the integration branch before their changes diverge substantially.

