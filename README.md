# Claude Code Research

This repository contains a practical guide for designing Claude Code setups for serious, long-running software projects.

The main document, [claude-code-guide.md](claude-code-guide.md), is not a basic feature tour. It explains how to structure `CLAUDE.md`, skills, rules, hooks, conventions, MCP usage, permissions, and parallel workflows so Claude Code produces maintainable work instead of accumulating drift.

The guide focuses especially on Claude's common failure modes: copying patterns locally, avoiding necessary refactors, inventing inconsistent abstractions, overusing tools, weakening type and database design, and treating UI code as disposable prototype work... to name but a few.

## Suggested workflow

Start `codex`—yes, Codex—and paste the following prompt:

```
This project uses Claude Code. Read [this guide](https://raw.githubusercontent.com/npomfret/claude-code-research/refs/heads/main/claude-code-guide.md) and the material it links to so you understand Claude Code's capabilities and limitations.

Then inspect the project and its recent commit history.

I need a world-class Claude Code environment for this project.

Create or update the `.claude` setup to support that goal. Focus on reusable, project-appropriate guidance; do not add product-specific rules or skills unless they are clearly justified. If you identify useful product-specific additions, propose them to the user at the end rather than adding them automatically.
```
