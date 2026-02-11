# Research: Popular Claude Code Agents, Skills, Commands (Feb 2026)

This note summarizes the most popular *publicly visible* agents, skills, and command packs in the Claude Code ecosystem. “Most popular” here is based on the only consistently public signals we can measure: GitHub stars/forks for open repos and install counts for official plugins.

## Method
- **Popularity signals**: GitHub stars/forks and Claude plugin install counts (from official plugin pages).
- **Scope**: Claude Code ecosystem (agents, skills, commands, plugin bundles, and collections).
- **Caveat**: There is no official, centralized “usage” leaderboard for skills/commands, so these are proxies.

## Agents & Agent Packs (by visible adoption)
1. **everything-claude-code (affaan-m)** — massive “all-in-one” bundle with agents/skills/hooks/commands; reported ~41.8k stars and 5.1k forks (community marketplace mirror). This size implies broad adoption and reuse for teams that want a comprehensive starting point rather than piecemeal configuration. Source: Claude Index marketplace metrics. citeturn4search6
2. **wshobson/agents** — ~28.4k stars, 3.1k forks. It’s popular because it ships a *large, modular agent catalog* plus workflows and skills in focused plugins, keeping context small while enabling breadth. Source: GitHub repo page. citeturn3view0
3. **awesome-claude-code (hesreallyhim)** — ~21.6k stars, 1.2k forks. It’s a community discovery hub that aggregates agents/commands/CLAUDE.md templates, making it the default starting point for many users. Source: ClaudeLog mirror of GitHub stats. citeturn4search0

## Skills & Plugins (by install count)
1. **Context7 (Upstash)** — ~104k installs. It’s popular because it delivers *live, version-accurate documentation* into context, which directly improves correctness and reduces hallucination risk for API/library usage. Source: official plugin page. citeturn0search4
2. **Code Review (Anthropic Verified)** — ~79.9k installs. High adoption reflects the consistent demand for standardized review workflows, which are high-leverage and low risk. Source: official plugin page (related plugins list). citeturn0search4
3. **GitHub MCP (Official)** — ~74.1k installs. It’s widely installed because it connects Claude Code to core dev workflows: issues, PRs, Actions, and repo search. Source: official plugin page. citeturn1search6
4. **Plugin Developer Toolkit (Anthropic)** — ~19.2k installs. Popular because it provides structured, reusable “skills for building skills” (hooks, MCP, commands, validation) with examples and guardrails. Source: official plugin page. citeturn0search4

## Commands & Command Packs
1. **Claude Command Suite (qdhenry)** — ~845 stars. A large, structured command pack (148+ commands) with ready-to-run workflows (review, test, deploy). Popular because it standardizes team workflows and reduces prompt-crafting overhead. Source: GitHub repo page. citeturn2search0
2. **awesome-claude-code** — includes a large slash command collection; widely referenced because it curates community-vetted commands in one place. Source: ClaudeLog mirror. citeturn4search0

## Why These Are “Good” (Signals + Practical Value)
- **Strong adoption signals** (stars/installs) imply broad real-world use and frequent updates, which usually correlates with better docs, troubleshooting, and community support. citeturn3view0turn4search0turn0search4
- **Focused, modular packaging** (e.g., wshobson/agents) reduces context bloat and keeps agent/tool loading minimal, which improves runtime performance and reduces hallucinations. citeturn3view0
- **Workflow standardization** (command suites and code review plugins) helps teams get consistent output quality and avoids the “one-off prompt” trap. citeturn2search0turn0search4
- **Live documentation integration** (Context7) reduces stale knowledge risks and aligns outputs to exact library versions. citeturn0search4

## Notes
- Popularity metrics change quickly; these values reflect the most recent public snapshots as of Feb 2026.
- If you want a narrower list (e.g., “top 5 by installs” only, or a “top 10 agents by stars”), say the word and I’ll refine.
