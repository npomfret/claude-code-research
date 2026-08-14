# Code Intelligence

### Prefer tools that expose facts about code

The most useful additions to Claude Code are not more personas, planning rituals, or prompt packs. They are tools that answer engineering questions with repository evidence:

- Where is this symbol defined and used?
- What calls it, and what does it call?
- Which execution paths pass through it?
- What is the blast radius of changing it?
- Which imports violate the intended architecture?
- Which files, exports, and dependencies are actually unused?
- Can this repeated code shape be found or rewritten safely across the repository?

These tools make Claude a better programmer because they reduce guessing. They do not replace reading the implementation, understanding runtime behavior, or running tests.

### Use the cheapest reliable tool first

Use this order:

1. **Compiler and language server** for types, definitions, references, rename operations, and diagnostics. Language-native tooling has the best understanding of the language's actual semantics.
2. **`rg` and repository search** for exact names, literals, configuration, tests, logs, and conventions. Text search is transparent, fast, and often sufficient.
3. **Code graph or structural index** when the question spans many files, indirect callers, inheritance, or execution paths.
4. **AST-aware search and rewriting** when text patterns are too fragile.
5. **Static checks in the build** when a discovered rule should remain enforced after the current session ends.

Do not add a heavyweight index merely to answer questions that the compiler or `rg` already answers well. Do not ask Claude to infer a repository-wide relationship from a handful of search results when a graph or language tool can enumerate it.

### GitNexus for repository structure and blast radius

[GitNexus](https://github.com/nxpatterns/gitnexus) indexes a local repository into a knowledge graph derived from Tree-sitter parsing and language-aware relationship resolution. Its CLI and MCP tools expose symbol context, incoming and outgoing calls, imports, inheritance, execution flows, paths between symbols, diff impact, and upstream blast radius. That directly supports the audit step this guide requires.

High-value uses:

- orienting in an unfamiliar or large repository;
- finding callers and downstream dependencies before a refactor;
- tracing a bug through a multi-file execution path;
- checking which processes a diff may affect;
- identifying related tests and modules before editing;
- and verifying that a rename or extraction covers more than the locally obvious references.

Use it as an index, not an oracle. Static call graphs have blind spots around reflection, dynamic dispatch, generated code, runtime registration, framework conventions, and unresolved language features. Confirm high-risk findings with language-server references, targeted searches, source inspection, and tests. Re-index after meaningful changes and check index staleness before relying on impact results.

There are two operational cautions:

- `gitnexus analyze` can install skills, register hooks, and write context into `CLAUDE.md` or `AGENTS.md`. Review those changes instead of accepting generated repository instructions blindly. Prefer a narrow or read-only integration when graph queries are all you need.
- The current repository uses the PolyForm Noncommercial 1.0.0 license. Personal and other qualifying noncommercial use is permitted; commercial teams must review the license or obtain appropriate terms before adoption.

### ast-grep for structural search and codemods

[ast-grep](https://ast-grep.github.io/) searches parsed syntax trees rather than raw text. Use it when formatting, variable names, or superficial syntax vary but the code shape is the same.

Good uses include:

- finding every call with a deprecated argument shape;
- locating nested `try/catch`, unsafe assertions, or framework anti-patterns;
- building a one-off, reviewable codemod;
- and turning a recurring structural convention into a checked rule.

Start with search-only output, inspect representative matches and edge cases, then run rewrites on a clean branch and review the diff. AST matching is more precise than regex, but a syntactic match is not proof of equivalent runtime semantics.

### dependency-cruiser for executable architecture

For JavaScript and TypeScript repositories, [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) can validate import relationships against checked-in rules. It can detect cycles, orphans, undeclared dependencies, production code importing test code, and forbidden layer crossings.

This is stronger than telling Claude "respect the architecture." Once the allowed import directions are encoded, the same rule applies to Claude, human programmers, and CI. Use its graph output to investigate the current structure; use its rule output to prevent regression.

### Knip for dead-code and dependency cleanup

For JavaScript and TypeScript, [Knip](https://knip.dev/) builds a project graph from entry points and framework-aware plugins, then reports unused files, exports, dependencies, unresolved imports, and optional cycle checks. It is valuable after refactors because Claude frequently leaves superseded code behind.

Treat findings as leads until the project graph is correctly configured. Dynamic imports, generated entry points, framework conventions, and missing plugin configuration can create false positives. Fix entry-point and plugin gaps before adding broad ignores, and review automatic fixes before committing them.

### Make discoveries durable

The best outcome is not merely that Claude used a tool once. Convert stable findings into repository enforcement:

- an architectural boundary becomes a dependency rule;
- a forbidden code shape becomes an AST lint rule;
- dead-code detection becomes a repeatable check;
- and canonical compiler, graph, search, and analysis commands become part of the repository's documented verification surface.

That is the dividing line between programmer tooling and Claude theatre: useful tools produce inspectable evidence and leave the codebase easier to verify without the current conversation.

