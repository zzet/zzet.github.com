---
title: "Find usages without grep false positives"
description: "Find usages without grep false positives: why grep matches comments and strings, the ast-grep/LSP/code-graph ladder, and zero-false-positive references."
date: 2026-06-01
lastmod: 2026-06-06
draft: false
slug: "find-usages-without-grep-false-positives"
keywords: ["find usages without grep false positives", "find references without false positives", "grep false positives comments strings", "find all references vs grep", "ast-grep vs grep", "find usages mcp", "semantic find references"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You ask your agent to rename `process`. It runs `rg process`, gets 240 hits, and starts editing. Forty of those hits are the word *process* in comments. A dozen are in string literals and log messages. A handful are a different `process` — a local variable in an unrelated module that happens to share the name. The agent doesn't know which is which, because grep doesn't either.

This is the core problem when you **find usages without grep false positives**: grep matches *text*, not *meaning*. For an engineer skimming results, the noise is annoying but survivable — you eyeball it. For an AI coding agent that acts on every hit, each false positive is a candidate wrong edit, and each missed real reference is a silent break that ships.

> **Short answer:** grep returns false positives because it matches characters, not resolved symbols — so a name in a comment, a string, a different scope, or a same-named-but-different symbol all look identical to it. To find references with zero false positives you need *semantic resolution*: an editor LSP (precise, but one language and one editor), or a resolved code graph (precise, cross-file, batchable, agent-facing). Gortex's `find_usages` answers from graph reference edges, so every hit is a real reference, each tagged with a typed context you can filter on.

## Why grep returns false positives (and how to find usages without grep false positives)

Grep, ripgrep, and friends are line-oriented text scanners. They are fast and correct *at what they do* — finding lines that match a pattern. The trouble is that "lines that contain the string `process`" is not the same question as "places that reference the symbol `process`." Grep has no model of scope, type, or imports, so it cannot tell those questions apart.

That collapses into four classes of false positive:

| False-positive class | Example | What grep sees |
|---|---|---|
| **Comments** | `// process the queue here` | a match |
| **String literals** | `log.Info("process started")` | a match |
| **Wrong / different scope** | a local `process` in an unrelated function | a match |
| **Same name, different symbol** | `user.process` (a field) vs `process()` (a function) | both match |

A practitioner [survey of why coding agents still use grep](https://yage.ai/share/why-coding-agents-still-use-grep-en-20260327.html) makes the honest counter-argument: these false positives are *benign* when you're exploring, because the LLM filters the noise as it reads. That's true — and it's exactly the line where the danger starts.

### Benign in exploration, dangerous in writes

The benign-vs-dangerous distinction is the whole game. When the task is *read and understand*, extra hits cost a few tokens and some attention. When the task is *rename / delete / edit every reference*, the calculus inverts:

- A **false positive** that survives review becomes a wrong edit — you mangle a comment, a string, or an unrelated symbol.
- A **false negative** — a real reference grep never matched (a re-exported alias, a method resolved through an interface, a call across a dynamic dispatch) — is worse: it's silent. The code compiles, the agent reports success, and the bug ships.

Grep's failure mode is loud (too many hits). The failure mode that actually breaks production is quiet (a missed hit). Any serious "find all references" answer has to address both.

## The truth ladder: text → structure → semantics → graph

There's no single "right" tool here — there's a ladder, and each rung fixes a failure class while adding a cost. Pick the rung your task actually needs.

| Rung | Tool | What it matches | Kills which false positives | Cost it adds |
|---|---|---|---|---|
| 1. Text | grep / ripgrep | characters on a line | none | nothing — works everywhere, no index |
| 2. Structure | ast-grep | syntax nodes (AST) | comments, strings | per-pattern; no cross-file resolution |
| 3. Semantics | LSP find-references | resolved symbols in one language | comments, strings, wrong scope, same-name | editor-bound, one server per language, stateful |
| 4. Graph | resolved reference edges | symbols across the whole repo | all four classes | needs a built index / daemon |

### Rung 2 — ast-grep: structural, ignores comments and strings

[ast-grep](https://github.com/ast-grep/ast-grep) is a Rust CLI (MIT-licensed as of 2026) that matches *tree-sitter syntax patterns* instead of raw text. Because it works on the concrete syntax tree, a pattern like `$X.process($$$)` will not match the word *process* inside a comment or a string literal. That alone removes the two noisiest false-positive classes, and it makes ast-grep excellent at what it's actually for: structural search and safe, pattern-based rewrites (codemods). It's fast, the DX is good, and it's the right tool when you want to *transform* code by shape.

But it stops short of find-all-references. ast-grep's own [FAQ is admirably blunt](https://ast-grep.github.io/advanced/faq.html): it has *no scope analysis, no type information, no control-flow or data-flow, no taint analysis, and no constant propagation.* It can't find "variables of a specific type" or distinguish two same-named symbols across files. So `$X.process()` still can't tell *your* `process` from an unrelated one with the same name in another file — that's a semantic question, and ast-grep is a structural tool. Reach for it for [pattern-based rewrites, not for symbol resolution](https://understandingdata.com/posts/ast-grep-for-precision/).

### Rung 3 — LSP find-references: as precise as your compiler

[LSP's `textDocument/references`](https://amirteymoori.com/lsp-language-server-protocol-ai-coding-tools/) is the precise answer inside an editor. The language server resolves symbols the way the compiler does, so it cleanly distinguishes `process` the function from `process` the variable from `process` in a comment. Within one editor and one language, this is as accurate as it gets — and it's a fair concession that for that scope, **LSP is exactly as precise as a code graph.**

The catch is the shape, not the accuracy. LSP is stateful and one-server-per-language, which is fine in a live editor and awkward for an agent, a batch job, or a CI step. [Sourcegraph frames the same trade-off](https://sourcegraph.com/docs/code-search/code-navigation/search_based_code_navigation) as *search-based* (convenient, no index, lower accuracy) versus *precise* (compiler-accurate, cross-repo, requires an index). And there's a specific agent trap: [naive MCP-to-LSP bridges cold-start the language server on every request](https://blog.blackwell-systems.com/posts/agent-lsp/) — no warm session, no cross-file index — so `get_references` can return nothing or only partial results. That's the worst outcome of all: a silent false negative dressed up as a clean answer.

### Rung 4 — resolved graph edges

The top rung resolves the whole repository into a graph once and answers reference queries from stored edges. Sourcegraph precise navigation does this at enterprise scale (cross-repo, indexed). [codanna](https://github.com/bartolli/codanna) is a closer same-category peer for agents — a local code-intelligence MCP server with call graphs, fast lookups, and local embeddings, no API key. It's a legitimate tool; as of 2026-06-06 it is Apache-2.0 licensed and covers roughly fifteen languages (verify both against its repo before quoting). Gortex sits on this rung too, and the rest of this piece is about what the graph rung buys you that the others can't.

This is the same idea that underpins a [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/): once references are resolved edges rather than text matches, "find usages" stops being a guess.

## How Gortex finds usages without grep false positives

[Gortex](https://github.com/zzet/gortex) indexes a repository into an in-memory code graph and exposes it over MCP, CLI, and HTTP. Its `find_usages` tool answers from resolved reference edges, so it never returns a comment, a string, a wrong-scope local, or a same-named-different-symbol hit. There is no text-matching step to produce noise in the first place.

```bash
# CLI
gortex query usages process

# MCP (what your agent calls)
find_usages(symbol: "process")
```

Four properties make this an agent-grade answer rather than just "grep but accurate":

**1. Zero false positives by construction.** Each result is a real reference edge the resolver built — not a line that happened to contain the characters. The four false-positive classes never appear because the tool never looks at raw text to begin with.

**2. A typed reference context per usage, filterable.** Every usage carries a `context` describing *how* the symbol is used: `call`, `parameter_type`, `return_type`, `field`, `value`, `type`, or `attribute`. That's information neither grep, ast-grep, nor a raw LSP references list gives you. You can ask "show me only the call sites" or "only places used as a parameter type" — useful before a signature change, where call sites and type positions need different handling.

**3. No silent false negatives.** When `find_usages` returns zero edges, it doesn't just say "0." It labels the result: *likely unused* versus *possible extraction / resolution gap*. That directly answers the LSP-bridge cold-start trap — an empty answer is never presented as a confident "no references," so an agent won't delete a symbol on the strength of a query that actually failed to resolve.

**4. Built for agents, not editors.** `find_usages` runs over MCP, CLI, and HTTP; it's cross-file and cross-repo; it covers **257 languages**; and on a warm daemon its measured **p95 latency is 0.01ms** (gortex repos, ~71k nodes, M3 Max — a single-machine warm-daemon number; absolute timings vary by hardware). It is not editor-bound and not one-server-per-language, so an agent can fire many reference queries in a batch without spinning up a server per request.

### Before / after

| | `rg process` | `find_usages(process)` |
|---|---|---|
| Comment hits | yes | none |
| String-literal hits | yes | none |
| Wrong-scope / same-name hits | yes | none |
| Per-usage typed context | no | `call`, `parameter_type`, `return_type`, `field`, `value`, … |
| Zero-result meaning | ambiguous | labeled (unused vs resolution gap) |
| Languages | n/a (text) | 257 |
| Agent surface | shell only | MCP / CLI / HTTP |

Once references are reliable, the tools built on them follow: [call graphs that answer "who calls this?" across 14+ languages](/gortex/call-graph-for-ai-coding-assistant/) and [graph-verified refactoring — safe rename, dead-code detection, and cycle checks](/gortex/graph-verified-refactoring/) all stand on the same resolved edges. `find_usages` is one tool in the broader [grep replacement for AI agents that spans lexical, structural, and graph search](/gortex/grep-replacement-for-ai-agents/).

### Honest limits

Gortex is not free of trade-offs. Like Sourcegraph precise navigation, it needs a built index and a running daemon — grep needs nothing, so for a one-off literal search in an unindexed directory, grep is still the faster reach. Resolution quality is excellent on exact-symbol queries but, like any resolver, depends on the language tier and on imports being resolvable. And ast-grep remains the better choice when your goal is a *pattern-based rewrite* rather than a symbol lookup; LSP remains exactly as precise within its one editor and one language. The graph's win is the combination — cross-file, cross-language, batchable, typed context, and a no-silent-false-negative contract — not a claim of being "more correct than the compiler" for a single language.

## FAQ

### Why does grep return false positives when finding usages?

Grep matches text, not meaning. It has no model of scope, type, or imports, so it can't tell a real reference from the same name inside a comment, a string literal, an unrelated local variable, or a different symbol that happens to share the name. Those four classes are the usual false positives.

### Does ast-grep eliminate find-references false positives?

Partly. ast-grep matches the AST via tree-sitter, so it ignores comments and string literals — a real improvement over grep. But its own FAQ states it has no scope analysis, no type information, and no cross-file resolution, so it can't distinguish two same-named symbols across files or do a true find-all-references by symbol. It's ideal for structural search and safe rewrites, not symbol resolution.

### Is LSP "find references" the same as grep find usages?

No. LSP is semantically accurate — as precise as your compiler within one editor and one language, distinguishing function from variable from comment. The catch for agents: it's stateful and one-server-per-language, and naive MCP-LSP bridges cold-start the server per request, returning empty or partial results — silent false negatives — with no native batch or multi-symbol surface.

### How do I find usages with zero false positives in an AI coding agent?

Use a tool that resolves symbols into a graph and answers from reference edges instead of text — Sourcegraph precise navigation, codanna, or Gortex's `find_usages` over MCP. Gortex returns every real reference with zero false positives, tags each with a typed context (`call` / `parameter_type` / `return_type` / `field` / `value`) you can filter on, and labels zero-result cases as "likely unused" vs "possible resolution gap" so you never get a silent false negative — at about 0.01ms p95 on a warm daemon, across 257 languages.

### ast-grep vs grep — which should I use?

Use grep/ripgrep for fast, exploratory text foraging where speed beats precision. Use ast-grep when you need structural precision or a safe codemod, since it ignores comments and strings and can rewrite by shape. But to find *all references to a specific symbol* across files you need semantic resolution, which ast-grep can't do — reach for LSP inside an editor, or a resolved code graph (Gortex, Sourcegraph precise, codanna) for agents, CI, or cross-repo work.

## Bottom line

Grep's false positives are tolerable when you're reading and dangerous when an agent is writing — and its false negatives are silent. The fix is a ladder: ast-grep removes comment and string noise but can't resolve symbols across files; LSP is compiler-precise but editor-bound and one-language-at-a-time; a resolved code graph answers from reference edges. Gortex's `find_usages` sits on that top rung — zero false positives, a filterable typed reference context per usage, an explicit no-silent-false-negative contract, 0.01ms p95 warm, 257 languages, over MCP — with the honest caveat that, unlike grep, it needs an index and a daemon. If your agent acts on what it finds, that's the trade worth making.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
