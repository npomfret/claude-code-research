# Claude Code: Popular Examples & Best Practices

> **Context:** This document compiles the most effective "real world" usage patterns for Claude Code, focusing on tools that solve specific friction points in the AI-assisted workflow.

---

## 1. Skills Hall of Fame (The "Cookbook")

Skills are your way of saving a perfect prompt so you don't have to type it again. The community has converged on a few high-value categories.

### A. The "Fullstack Developer" Suite
**Why it's popular:** Instead of generic coding, these skills enforce specific stack choices (e.g., "Use Tailwind, not CSS modules").
*   **`/skill react-component`**: Scaffolds a component + test + storybook file in one go.
*   **`/skill api-route`**: Creates a Next.js API route with Zod validation pre-configured.
*   **Why it's good:** It stops Claude from guessing your file structure. It forces consistency.

### B. The "Reviewer"
**Why it's popular:** Context-aware code reviews are better than generic ones.
*   **`/skill review`**: "Review the staged changes. Focus on: 1. Security vulnerabilities. 2. Performance bottlenecks. 3. Adherence to DRY. Do not comment on formatting."
*   **Why it's good:** It filters out the noise (linting errors) so you can focus on logic.

### C. The "Bug Hunter"
**Why it's popular:** Debugging often requires a rigorous scientific method that AI skips if not prompted.
*   **`/skill investigate`**: "1. Find the code handling X. 2. Add log statements to trace execution. 3. Run the reproduction script. 4. Analyze logs. 5. Propose a fix."
*   **Why it's good:** It forces Claude to *diagnose* before *prescribing*, reducing hallucinated fixes.

---

## 2. Agents: The "Persona" Pattern

While Claude Code has built-in agents (`Explore`, `Plan`), the community effectively creates "custom agents" by defining sub-agents with narrow scopes.

| Agent Persona | Responsibility | Why it works |
| :--- | :--- | :--- |
| **The "Archaeologist"** | `Explore` agent tasked with "Map the dependency graph of module X." | Prevents the main context from getting clogged with file reads. |
| **The "Security Auditor"** | Sub-agent with access *only* to read files and run `semgrep`/`snyk`. | Specialized focus finds things generalist agents miss. |
| **The "Frontend Specialist"** | Sub-agent told "You are an expert in CSS Grid and A11y. You cannot touch backend code." | Prevents "fullstack" refactors that accidentally break the API. |

**Best Practice:** Use the **Plan Agent** (`--permission-mode plan`) to scope out work before letting the **Execute Agent** (default) touch files.

---

## 3. Commands: The Power User Kit

The CLI is where the real power lies for integration.

### The "Time Machine": `/compact`
*   **Command:** `/compact` (or `/compact focus on X`)
*   **Why it's critical:** Claude's "IQ" drops as context fills. Compacting is like a garbage collection for its brain.
*   **Pro Tip:** Run this every time you switch tasks within a session (e.g., moving from "Backend" to "Frontend" work).

### The "Pipeline": `-p` (Print Mode)
*   **Command:** `git diff main | claude -p "Review these changes"`
*   **Why it's critical:** This turns Claude into a unix pipe. You can chain it into CI/CD, pre-commit hooks, or daily reporting scripts.
*   **Pro Tip:** Use `--output-format json` to make the output parseable by other tools (like posting to Slack).

---

## 4. Hooks: The "Guardrails"

Hooks are the only way to *guarantee* behavior.

### The "Janitor": Auto-Format
*   **Trigger:** `PostToolUse` (on `Write`)
*   **Command:** `npx prettier --write $CLAUDE_FILE_PATHS`
*   **Why it's good:** Claude is bad at whitespace. This stops it from wasting turns fixing lint errors. It separates "logic generation" (Claude) from "style enforcement" (Prettier).

### The "Bodyguard": Dangerous Command Blocker
*   **Trigger:** `PreToolUse` (on `Bash`)
*   **Command:** `grep -q "rm -rf" <<< "$INPUT" && exit 2`
*   **Why it's good:** It prevents catastrophe. Even if you trust Claude, you shouldn't trust an LLM with `rm -rf /` without a safety net.

### The "Gatekeeper": Test-on-Commit
*   **Trigger:** `UserPromptSubmit` (intercepting "commit")
*   **Command:** `npm test`
*   **Why it's good:** It forces Claude to prove its work. If tests fail, the commit is blocked, and Claude has to try again.

---

## 5. References & Community Resources

*   **Awesome Claude Code:** [github.com/hesreallyhim/awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) - The central hub for community skills.
*   **Claude Command Suite:** [github.com/qdhenry/Claude-Command-Suite](https://github.com/qdhenry/Claude-Command-Suite) - A large collection of ready-to-use slash commands.
*   **Pre-commit Hooks:** [github.com/aRustyDev/claude-code-hooks](https://github.com/aRustyDev/claude-code-hooks) - Security and formatting hooks.
