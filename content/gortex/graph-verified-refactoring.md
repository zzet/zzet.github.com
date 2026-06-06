---
title: "Rename a function across files safely: graph-verified refactoring"
description: "Rename a function across files safely, find dead code, and detect circular dependencies with a code graph instead of grep-sed or a guessing AI agent."
date: 2026-06-03
lastmod: 2026-06-06
draft: false
slug: "graph-verified-refactoring"
keywords: ["rename function across files safely", "find dead code in codebase", "detect circular dependencies", "find unused functions go", "safe refactoring with a code graph", "cross language dead code"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You ask your agent to rename `getUser` to `fetchUser`. It edits the definition and three obvious call sites, reports success, and moves on. What it missed: a re-exported alias two packages over, a method that satisfies an interface, a string in a route table, and the same name on an unrelated struct. The build is green in the file it touched and broken everywhere it didn't look. You find out in CI, or worse, in production.

To **rename a function across files safely** you need something that knows which `getUser` is *this* `getUser` — across every file, in every language in the repo. Grep doesn't. `sed` doesn't. And a standalone LLM, it turns out, doesn't either.

> **Short answer:** grep-sed renames are unsafe because they match text, so a name gets hit inside comments, strings, and unrelated same-named symbols; AI agents are unsafe because they pattern-match instead of resolving symbols and miss cross-file references after the first file. A graph-verified rename resolves every real reference through a code graph (or an editor's LSP) and updates exactly those — and lets you preview the edit before committing it. Gortex's `rename_symbol` updates resolved references, not text; `verify_change` and `preview_edit` let you check first; `safe_delete_symbol` refuses to delete code that test-only or cross-workspace callers still use.

## Why grep-sed rename is structurally unsafe

`grep | sed` renaming is the original sin of refactoring. It works on characters, and code has structure, so the same name matches in places that have nothing to do with your symbol:

| What `sed s/getUser/fetchUser/g` hits | Should it? |
|---|---|
| `// getUser returns the cached user` (comment) | No |
| `log.Info("getUser failed")` (string literal) | No |
| `account.getUser` — a method on a *different* type | No |
| a local variable named `getUser` in an unrelated scope | No |
| the actual function and its real callers | Yes — but only these |

A regex can't tell these apart, because telling them apart requires scope, types, and import resolution — none of which live in the text. Every tightening of the pattern trades one failure mode for another: too loose and you mangle strings; too strict and you miss a real call. And the deeper problem isn't the false positives you can see — it's the false negatives you can't, the real reference the pattern never matched, which compiles fine and breaks silently.

## Why AI agents do refactoring badly

The obvious fix — "let the agent do it" — fails for a related reason. LLMs excel at generating plausible code through pattern matching, but refactoring demands precision over plausibility, as the team behind [Kiro argues in *Refactoring made right*](https://kiro.dev/blog/refactoring-made-right/). Their illustration is mundane and damning: an agent asked to rename `get_loose_identifier` across four files and eight references gets the first file and quietly drops the rest.

The numbers back this up. A study of 1,752 Extract-Method scenarios found that [up to 76.3% of an LLM's suggestions were hallucinations](https://arxiv.org/abs/2401.15298) — 57.4% invalid (would not compile) and 18.9% not useful. The same study's fix is the point of this article: pairing the LLM with IDE static analysis raised recall to 53.4%, versus 39.4% for the prior best tool. The model wasn't replaced; it was *grounded*. Separate work on [multi-agent coordinated rename refactoring](https://arxiv.org/abs/2601.00482) reaches the same qualitative conclusion — standalone models routinely miss remaining cross-file references after renaming the first file.

The lesson isn't "don't use agents." It's that an agent doing a rename, a delete, or a cycle check should be calling a deterministic tool, not improvising the edit from memory. This is the same argument as [finding usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/): the agent needs resolved references, not a text match it has to second-guess.

## The honest solution menu (most of it isn't Gortex)

There is no single tool that owns cross-language safe-rename + dead-code + cycles. There *are* excellent per-language and per-layer specialists, and for a single-language repo one of them is very likely your best answer. Lead with these:

### Your IDE's LSP rename

The least exotic option is the best one inside an editor. An LSP-backed rename ([`textDocument/rename` in the LSP 3.17 spec](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)) is as precise as your compiler — it resolves symbols, distinguishes a function from a same-named field, and updates every real reference. The limits are structural: one stateful server per language, living inside the editor, which makes it awkward to drive from an agent, across repos, or in CI. Naive MCP-to-LSP bridges that cold-start a server per request tend to return partial results — silent false negatives.

### Dead code: Knip (JS/TS) and vulture (Python)

For JavaScript and TypeScript, [Knip](https://knip.dev/) is the standard. It finds unused files, dependencies, and exports by following imports from your entry points to build a dependency graph, then reports unused exports, missing deps, duplicate exports, and unused class members. It ships `--fix` auto-fix, a VS Code extension, a language server, an [MCP server (`@knip/mcp`)](https://github.com/webpro-nl/knip), and ~150 framework plugins. On JS/TS dead-export and dependency hygiene, Knip is more thorough than Gortex, and it's the right tool for that job.

For Python, [vulture](https://github.com/jendrikseipp/vulture) is the standard. It uses Python's `ast` module to record defined-versus-used names and reports unused functions, classes, variables, imports, and unreachable code — with a confidence score (100% for unreachable code and unused arguments, 90% for imports, 60% for everything else) and a `--make-whitelist` workflow. Crucially, vulture is honest about its blind spot, and you should be too: *"Due to Python's dynamic nature, static code analyzers like Vulture are likely to miss some dead code. Also, code that is only called implicitly may be reported as unused."* Every static dead-code tool, Gortex included, shares this caveat.

### Cycles: madge (JS)

For JavaScript module cycles, [madge](https://github.com/pahen/madge) builds a module dependency graph and `--circular` returns the modules in a cycle, with a CI-friendly exit code and optional SVG/DOT rendering (Graphviz is only needed for images). Its scope is JS (AMD/CommonJS/ESM) plus CSS preprocessors, at the module-file level — not symbol-level, no rename, no dead code. Within that scope it's the obvious pick.

### Structural rewrites: ast-grep

For pattern-based codemods, [ast-grep](https://ast-grep.github.io/) matches tree-sitter AST nodes with `$VAR` metavariables, so it ignores comments and strings and won't match inside a string literal the way `sed` does. It's fast, polyglot (20+ languages), and great for shape-based rewrites. But by design it is *syntactic, not semantic*: no cross-file reference tracking, no scope or type awareness, no symbol graph. It still can't tell a real call from a same-named method on another type — that's a semantic question, and ast-grep is a structural tool. See [the grep replacement ladder](/gortex/grep-replacement-for-ai-agents/) for where structural search ends and graph resolution begins.

| Tool | Owns | Scope | License (as of 2026) | Not for |
|---|---|---|---|---|
| **Knip** | JS/TS dead code, deps, exports + auto-fix | JS/TS | ISC | other languages; rename; cycles |
| **vulture** | Python dead code, confidence scoring | Python | MIT | other languages; rename; cycles |
| **madge** | JS module cycles + visual graph | JS/CSS, file-level | MIT | rename; dead code; symbol-level |
| **ast-grep** | fast structural search + codemods | 20+ langs, syntactic | MIT | cross-file symbol resolution; rename |
| **LSP rename** | compiler-precise rename | one lang/editor | varies | cross-repo; agent/CI driving |

Licenses, plugin counts, and feature sets above reflect each project as of 2026; check the linked repos for current terms. If your repo is one language and you live in an editor, stop here — one of these is your answer. The gap they leave is the polyglot, agent-driven one: a repo with Go *and* TypeScript *and* Python where you'd otherwise stitch four tools together, and where the actor doing the edit is an AI agent that needs a deterministic tool to call.

## How a code graph lets you rename a function across files safely

A code graph resolves the repository into nodes (symbols) and typed edges (calls, implements, references) once, then answers structural questions by walking edges instead of scanning text. That's the substrate every safe refactor needs — the [code knowledge graph](/gortex/code-knowledge-graph-for-llms/) turns "which `getUser` is this?" from a guess into a lookup. Gortex builds this graph across **257 languages** and exposes **100+ MCP tools** over it, Apache 2.0 and fully open-source, so the agent writing your code can call graph-verified operations directly.

### Safe rename across files

`rename_symbol` renames by resolved reference, not by text. It updates the definition and every real reference — across files, across the symbol's actual call and `implements` edges — and leaves the same name on an unrelated type alone, because that's a different node in the graph. Before you commit, two speculative tools let you check:

- `verify_change` checks a proposed signature change against every caller and interface implementor — it's the "what breaks?" gate, the same machinery described in [what breaks if I change this function](/gortex/impact-analysis-what-breaks-if-i-change-this/).
- `preview_edit` / `simulate_chain` answer "what would change if I applied this edit?" without touching disk.

```text
# Preview, verify, then apply — MCP tool calls the agent makes, instead of guessing
verify_change(symbol: "getUser", new_name: "fetchUser")   # all callers + implementors
preview_edit(rename: "getUser:fetchUser")                 # speculative, no disk write
rename_symbol(from: "getUser", to: "fetchUser")           # updates every resolved reference
```

The honest limit: `rename_symbol` and `safe_delete_symbol` work across languages, but `move_symbol` and `inline_symbol` are **Go-only for now**. If you need to move a symbol across files in TypeScript or Python today, that's still your IDE's job.

### Dead code, across languages, with a destructive-edit gate

`analyze dead_code` reports unreferenced symbols across the whole graph — Go, TypeScript, and Python in one pass, instead of running Knip plus vulture plus a third tool and reconciling the outputs. `find_clones` (MinHash + LSH near-duplicate detection) adds a `dead_only` mode that surfaces dead duplicates of live code — copies you can delete because a living version already exists.

The blunt caveat, the same one vulture states: static analysis can't see dynamic dispatch, reflection, or string-keyed dispatch. Treat dead-code results as *likely* unused, not certain. Which is exactly why the deletion step is gated:

```text
# MCP tool calls the agent makes
analyze(kind: "dead_code")              # cross-language unreferenced symbols
find_clones(dead_only: true)           # dead duplicates of live code
safe_delete_symbol(name: "oldHelper")  # refuses if real callers still exist
```

`safe_delete_symbol` is the difference between a graph-aware gate and a blind `--fix`. A blanket auto-fix removes code that test-only or cross-workspace callers still depend on, silently breaking the build; `safe_delete_symbol` runs an orphan-cascade check and refuses the delete when such callers exist. You can't out-detect Knip on TS dead exports or vulture on Python with Gortex — the edge here is one pass over every language plus a gate that won't let a confident-but-wrong deletion through.

### Circular dependencies, detected and prevented

Cycles aren't a style nit. They [cause tight coupling so changes ripple, make modules hard to unit-test in isolation, and create build problems](https://www.techtarget.com/searchapparchitecture/tip/The-vicious-cycle-of-circular-dependencies-in-microservices) — bundlers warn or error, and a cycle can break tree-shaking. `analyze cycles` finds existing cycles across the graph; `analyze would_create_cycle` is a pre-flight you run *before* applying an edit, so you never merge a new cycle in the first place.

```text
# MCP tool calls the agent makes
analyze(kind: "cycles")                       # existing cycles, cross-language
analyze(kind: "would_create_cycle", edit: …) # pre-flight: block before merge
```

That pre-flight is the part madge and friends don't give you — not "here are your cycles after the fact," but "this change would introduce one, don't apply it."

### The cross-language story, and the token math

The reason to reach for this over four specialists is breadth plus the agent surface. One engine holds Go, TypeScript, and Python in a single graph; one set of MCP tools answers rename, dead-code, clone, and cycle questions; and an agent calls them directly instead of improvising edits it pattern-matched. Because the agent reads resolved answers instead of raw files, Gortex reports **3–50× fewer tokens per response** versus naive file reads (the ~50× peak is for identifier lookups, not a flat figure). The default embeddings are lightweight baked GloVe-50d with zero setup, with MiniLM or OpenAI as an opt-in for stronger semantic recall — but rename, dead-code, and cycle analysis ride deterministic graph edges, not embeddings, so they don't depend on that choice.

What this isn't: a claim that Gortex beats Knip on TypeScript dead exports, vulture on Python, madge on JS cycles, or ast-grep on fast codemods. Each owns its turf. The claim is narrower and verifiable — cross-language coverage, a deterministic graph, and an agent-callable surface, with destructive edits gated instead of trusted.

## FAQ

### How do I rename a function across many files safely?

Use a tool that resolves references through a code graph or LSP, not grep/sed. Grep matches the name inside string literals, comments, docs, and unrelated same-named symbols because it works on text, not structure. A graph-verified rename (Gortex `rename_symbol`, or your IDE's LSP rename) updates only the real, resolved references, and lets you preview the change with `preview_edit` before applying it.

### Why shouldn't I let my AI agent do the rename directly?

Standalone LLMs pattern-match rather than resolve symbols. A study of 1,752 Extract-Method cases found up to 76.3% of LLM suggestions were hallucinations, and agents routinely miss references after renaming the first file. Back the agent with a graph or LSP tool — for example Gortex `rename_symbol` plus `verify_change` — so the edit is verified, not guessed.

### How do I find dead code across multiple languages?

Per-language specialists cover one language each — Knip for JavaScript/TypeScript, vulture for Python. For a polyglot repo you'd otherwise run several tools; a cross-language graph engine like Gortex runs one `dead_code` analysis over all languages and adds `find_clones --dead-only` for dead duplicates. All static dead-code tools have a blind spot for dynamic dispatch and reflection, so treat results as "likely unused," not certain.

### How do I detect and prevent circular dependencies?

For JavaScript/TypeScript, `madge --circular` lists modules in a cycle. For a cross-language repo, use a graph engine's `cycles` analysis. To stop regressions, run a `would_create_cycle` pre-flight before applying an edit so you never merge a new cycle — cycles cause tight coupling, hard-to-isolate tests, and broken builds and tree-shaking.

### Is it safe to delete an unused function?

Only if nothing real still depends on it. A blind auto-fix can remove code that test-only or cross-workspace callers use, silently breaking the build. A graph-aware gate like Gortex `safe_delete_symbol` refuses the delete when such callers exist, so you remove dead code without shipping a hidden break.

## Bottom line

Refactoring demands precision over plausibility, and the tools that give you precision work on structure, not text. For a single-language repo in an editor, your LSP rename, Knip, vulture, madge, or ast-grep is likely the right answer — each is the strongest tool on its home turf. The gap they leave is the polyglot, agent-driven one. There a code graph earns its keep: `rename_symbol` updates resolved references instead of text, `verify_change` and `preview_edit` let you check before committing, `safe_delete_symbol` gates a deletion behind an orphan-cascade check, and `analyze cycles` / `would_create_cycle` catch a bad merge before it lands — across 257 languages, as MCP tools the agent calls directly. The caveats stand: `move_symbol` and `inline_symbol` are Go-only for now, static dead-code analysis can't see reflection, and Gortex won't out-detect the specialists on their own language. Breadth, determinism, and a gated edit surface are the trade — and when an agent is doing the refactor, it's the trade worth making.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
