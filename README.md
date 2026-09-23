# Claude Code Research

This repository contains a practical guide to configuring Claude Code for long-running software projects.

The main document, [claude-code-guide.md](claude-code-guide.md), explains how to structure `CLAUDE.md`, skills, rules, hooks, conventions, MCP usage, permissions, and parallel workflows so Claude Code produces maintainable work without accumulating drift.

It addresses common failure modes: local pattern copying, avoided refactors, inconsistent abstractions, leaked vendor APIs, tool overuse, weak type and database design, and disposable UI code.

## Suggested workflow

Start Codex and use this prompt:

```
This repository uses Claude Code. Read the Claude Code guide at <guide-url> and its linked material to understand the product's capabilities and limitations.

Inspect the repository, its configuration, and its code.

Configure Claude Code to plan, implement, test, and improve the codebase with minimal supervision while surfacing meaningful decisions and risks.

Create a concise, rigorous, and professional Claude Code environment for this repository.

Create or update the repository's Claude Code configuration. Put only crucial, repository-wide facts that Claude cannot infer in root `CLAUDE.md`. Do not use it to index `.claude/`; make skills, rules, agents, and references discoverable through their metadata, scope, placement, and ownership. Put scoped or procedural guidance in a mechanism that controls when it loads. Include repository-specific guidance only when evidence shows that omitting it would reduce reliability. Remove existing material that does not meet this standard.
```
