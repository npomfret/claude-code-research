# Claude Code Research

This repository contains a practical guide for designing Claude Code setups for serious, long-running software projects.

The main document, [claude-code-guide.md](claude-code-guide.md), is not a basic feature tour. It explains how to structure `CLAUDE.md`, skills, rules, hooks, conventions, MCP usage, permissions, and parallel workflows so Claude Code produces maintainable work instead of accumulating drift.

The guide focuses especially on Claude's common failure modes: copying patterns locally, avoiding necessary refactors, inventing inconsistent abstractions, leaking vendor APIs through application code instead of encapsulating them, overusing tools, weakening type and database design, and treating UI code as disposable prototype work... to name but a few.

## Suggested workflow

Start `codex` - yes, Codex - and paste the following prompt:

```
This project uses Claude Code. Read [this guide](https://raw.githubusercontent.com/npomfret/claude-code-research/refs/heads/main/claude-code-guide.md) and the material it links to so you understand Claude Code's capabilities and limitations.

Then inspect the project and its recent commit history.

The goal is to turn Claude Code from a capable programmer into a world class programming team: one that can plan, implement, test, and improve the codebase with minimal supervision, while still surfacing meaningful decisions and risks.

I need a brutally efficient and professional, Claude Code environment for this project.

Create or update the project's Claude Code configuration, including a root `CLAUDE.md` only where the repository has crucial, broadly applicable facts or instructions that Claude cannot reliably infer. Do not use `CLAUDE.md` to index or link the contents of `.claude/`; make skills, rules, agents, and their supporting references discoverable through their own metadata, scope, placement, and ownership. Put scoped, procedural, or nuanced guidance in the mechanism that can express when it applies. Add project-specific guidance only when repository evidence shows that omitting it would predictably make Claude less reliable. You are free to update or remove existing sections that do not meet that standard.
```
