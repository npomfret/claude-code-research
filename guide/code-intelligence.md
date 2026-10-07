# Code Intelligence

Use the cheapest tool that can answer the question completely. Language and graph tools reveal relationships; source and runtime checks establish what those relationships mean.

## Tool order

1. Compiler, language server, and IDE tools for types, definitions, references, renames, and diagnostics; prefer JetBrains MCP for supported IDE operations.
2. `rg` for names, literals, configuration, tests, logs, and conventions.
3. Graph/index queries for indirect callers, inheritance, and multi-file paths.
4. AST-aware search and rewriting when text matching is fragile.
5. Build-integrated static checks for durable enforcement.

**Don't:** add a heavy index for a question ordinary search answers well, or infer repository-wide relationships from a handful of matches.

### JetBrains MCP for IDE-backed code intelligence

The [JetBrains MCP server](https://www.jetbrains.com/help/idea/mcp-server.html) exposes IDE capabilities to Claude Code. IntelliJ IDEA includes it from version 2025.2; enable it under **Settings → Tools → MCP Server** and use client auto-configuration or copy the connection configuration.

The current documentation includes:

- `search_symbol` and `get_symbol_info` for locating and understanding symbols;
- `analyze_calls` for incoming and outgoing call hierarchies;
- `get_file_problems` and `lint_files` for inspections;
- `rename_refactoring` for semantic renames;
- build and run-configuration tools for verification.

Prefer these operations where supported: they let Claude use the IDE's understanding of the project. Confirm availability in the installed IDE's exposed-tool list; support depends on version, language, plugins, and project indexing. Review refactoring diffs and run relevant checks.

JetBrains MCP is the preferred direction for projects using a supported JetBrains IDE. As its semantic and analysis coverage grows, it may supersede GitNexus for those projects. That is an expectation, not a claim of complete feature parity today. Keep GitNexus only where it adds useful graph, execution-flow, or diff-impact evidence the IDE integration does not provide.

### GitNexus for additional repository structure and blast radius

[GitNexus](https://github.com/nxpatterns/gitnexus) builds a local code graph from Tree-sitter parsing and relationship resolution. Use it for symbol context, calls, imports, inheritance, execution paths, diff impact, and upstream blast radius where IDE coverage is insufficient.

**Do:**

- Use graph queries to orient, trace bugs, find callers/tests, and scope refactors.
- Check staleness and re-index after meaningful changes.
- Confirm high-risk findings with language references, source, targeted search, and tests.
- Review instruction, skill, and hook changes made by `gitnexus analyze`; prefer narrow/read-only integration when only queries are needed.
- Review its PolyForm Noncommercial 1.0.0 license and obtain appropriate terms for commercial adoption.

**Don't:** treat static graphs as complete around reflection, dynamic dispatch, generated code, runtime registration, framework conventions, or unresolved language features.

### ast-grep for structural search and codemods

[ast-grep](https://ast-grep.github.io/) matches parsed code shapes. Use it for deprecated call forms, nested catches, unsafe assertions, framework anti-patterns, codemods, and structural lint rules.

**Do:** search first, inspect matches and edge cases, rewrite in a clean working tree, and review the diff under the [Git workflow](workflows-and-maintenance.md#git-workflow).

**Don't:** equate a syntactic match with equivalent runtime semantics.

### dependency-cruiser for executable architecture

[dependency-cruiser](https://github.com/sverweij/dependency-cruiser) checks JavaScript/TypeScript imports for cycles, orphans, undeclared dependencies, test imports in production, and forbidden boundaries. Use graph output for investigation and checked-in rules for enforcement.

### Knip for dead-code and dependency cleanup

[Knip](https://knip.dev/) reports unused files, exports, dependencies, unresolved imports, and optional cycles from a configured JavaScript/TypeScript project graph.

**Do:** configure entry points and framework plugins, investigate dynamic/generated paths, and review fixes before committing.

**Don't:** delete every reported item blindly or hide configuration gaps with broad ignores.

## Make discoveries durable

Turn stable boundaries into dependency rules, forbidden shapes into AST checks, and dead-code detection into repeatable verification. Document canonical analysis commands so the repository remains verifiable after the conversation ends.
