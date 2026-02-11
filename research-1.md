# Claude Code: Strategic Usage & Nuance

> **Executive Summary:** Success with Claude Code requires treating it less like a smarter autocomplete and more like a **junior engineer** that you manage. The key is knowing when to give it a "recipe" (Skill), when to give it a "job" (Agent), and when to set "boundaries" (Hooks).

---

## 1. Skills vs. Agents: The "Recipe" vs. The "Job"

The most common confusion is between Skills and Agents. They seem similar but serve fundamentally different management needs.

| Feature | Metaphor | Core Function | Best For... | Anti-Pattern |
| :--- | :--- | :--- | :--- | :--- |
| **Skill** | **The Cookbook** | Teaches Claude *HOW* to do a specific task. | **Repeatable, defined processes.**<br>• "Create a React Component using our design tokens."<br>• "Write a migration following our DB schema rules." | Trying to make a Skill "think" (e.g., "Analyze the architecture"). Skills should be deterministic instructions. |
| **Agent** | **The Contractor** | Takes a high-level goal and figures out *WHAT* to do. | **Open-ended exploration.**<br>• "Map out the authentication flow."<br>• "Find the root cause of this race condition."<br>• "Plan a refactor of the billing module." | Using an Agent for simple tasks. It's slower and more expensive. Don't hire a contractor to change a lightbulb. |

### The "Sweet Spot" Strategy
*   **Build Skills for "Style":** Use skills to enforce *how* code looks and functions (boilerplate, patterns, testing standards). This effectively expands your `CLAUDE.md` without polluting the global context.
*   **Use Agents for "Scope":** Use agents when you don't know the full scope of the work. Let the `Explore` agent build the mental map so you don't have to context-dump manually.

---

## 2. Hooks: The "Guardrails" (Not the Logic)

Hooks are your safety net, but they are dangerous if overused.

### Strategic Usage
*   **✅ The "Gatekeeper" Pattern:** Use `block-at-submit` hooks.
    *   *Example:* "Run `npm test` before committing." If it fails, Claude must fix it. This creates a tight feedback loop where Claude self-corrects without your intervention.
*   **✅ The "Context Injector":**
    *   *Example:* A hook that runs `git status` or `ls -R` at the start of a session. This ensures Claude always knows the "ground truth" state of the repo immediately.

### Critical Anti-Patterns
*   **❌ The "Backseat Driver" (Mid-Write Hooks):** interrupting Claude *while* it's writing code (e.g., linting every file save). This breaks the model's "chain of thought," leads to frustration, and wastes tokens. Validating *after* the work is done is much more effective.
*   **❌ The "Silent Assassin":** Hooks that modify code (like auto-formatters) without telling Claude. Claude will get confused why the file changed. Always let Claude know (via tool output) if a hook changed something.

---

## 3. Workflow Archetypes

### A. The "Architect-Builder" Loop (For New Features)
1.  **Phase 1 (Agent):** Use the `Plan` agent. "Research the Stripe API and plan the subscription checkout flow."
    *   *Output:* A markdown file (`plan.md`) detailed enough for a human to read.
2.  **Phase 2 (Human):** **Review the plan.** This is your highest-leverage intervention point. Correct the logic here, not in the code.
3.  **Phase 3 (Skill):** "Implement step 1 of `plan.md` using the `/new-component` skill."

### B. The "Refactor Sweeper" (For Tech Debt)
1.  **Setup:** Create a temporary `REF_SKILL.md` that defines the *exact* transformation (e.g., "Replace `moment.js` with `date-fns`").
2.  **Execution:** "Find all usages of `moment()` in `src/utils` and apply the Refactor Skill."
3.  **Safety:** Rely on a `pre-commit` hook to run tests. If they fail, Claude iterates.

---

## 4. The "Quiet Dangers"

### The "Confidence Trap"
Claude Code is exceptionally good at making code *look* correct. It will confidently introduce a subtle bug (like a race condition) while perfectly matching your indentation style.
*   **Mitigation:** ask Claude to "Explain the potential edge cases in this change" *before* it writes the code.

### The Context cliff
In long sessions (>30 turns), Claude Code *will* forget early instructions or hallucinate file contents.
*   **Nuance:** The `/compact` command helps, but it's lossy.
*   **Rule of Thumb:** If a task takes more than 20 turns, **stop**. Commit the work. Restart the session. Treat sessions as "work days" — clear the slate regularly.

---

## 5. Summary Recommendation

*   **Start Simple:** Don't write custom Agents yet.
*   **Invest in `CLAUDE.md`:** It's the highest ROI configuration.
*   **Build 1-2 Skills:** Focus on your most painful, repetitive task (e.g., "New API Endpoint").
*   **Use Hooks for Safety Only:** prevent commits that break the build.
