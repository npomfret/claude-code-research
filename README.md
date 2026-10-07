# Claude Code Research

This repository contains a practical guide to configuring Claude Code for long-running software projects.

The main document, [claude-code-guide.md](claude-code-guide.md), provides short rationales, practical do’s and don’ts, and workflows for `AGENTS.md`, skills, rules, hooks, conventions, MCP usage, permissions, and parallel workflows so Claude Code produces maintainable work without accumulating drift.

It focuses on maintainable architecture, discoverable instructions, proportionate verification, and coherent project conventions as models improve.

## Suggested workflow

Start Codex (yes Codex) with a prompt such as this:

```
This repository uses Claude Code. Read this Claude Code guide at https://raw.githubusercontent.com/npomfret/claude-code-research/refs/heads/main/claude-code-guide.md and its linked material to understand the product's capabilities and limitations.

Inspect this repository, its configuration, its code and recent commit history.

Configure Claude Code to plan, implement, test, and improve the codebase with minimal supervision while surfacing meaningful decisions and risks.

Create a concise, rigorous, and professional Claude Code environment for this repository.

Create or update the repository's Claude Code configuration. Put only crucial, repository-wide facts that Claude cannot infer in root `AGENTS.md`. Do not use it to index `.claude/`; make skills, rules, agents, and references discoverable through their metadata, scope, placement, and ownership. Put scoped or procedural guidance in a mechanism that controls when it loads. Include repository-specific guidance only when evidence shows that omitting it would reduce reliability. Remove existing material that does not meet this standard.
```
